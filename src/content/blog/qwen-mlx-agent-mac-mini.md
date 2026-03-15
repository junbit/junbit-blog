---
title: "Ollama보다 3배 빠른 MLX로 Qwen3.5 에이전트 만들기 (Mac Mini 24GB)"
description: "Mac Mini M4 24GB에서 MLX + Qwen3.5-35B-A3B 3bit로 49→59 tok/s 달성, 멀티모달 비전까지 지원하는 로컬 AI 에이전트 구축 과정"
pubDate: 2026-03-15
tags: ["llm", "mlx", "qwen", "mac-mini", "local-ai", "agent"]
---

이전 글에서 Ollama + Qwen3.5 27B로 로컬 에이전트를 만들었다. 잘 동작하긴 했지만, 솔직히 느렸다. 27B 모델이 8.5 tok/s밖에 안 나오니 간단한 질문에도 10초 이상 기다려야 했다.

레딧과 커뮤니티를 돌아다니다 보니 Apple Silicon에서는 **Ollama/llama.cpp보다 MLX가 훨씬 빠르다**는 후기가 반복적으로 올라오고 있었다. 직접 확인해보기로 했다.

---

## 모델 선정: 왜 35B-A3B인가

Qwen3.5에는 27B(Dense)와 35B-A3B(MoE)가 있다. 둘 다 24GB에 들어가는데, MoE 구조인 35B-A3B가 실제로 활성화하는 파라미터는 3B뿐이다. 즉 **전체 지식은 35B급인데 추론 속도는 3B급**이라는 뜻이다.

레딧에서 가장 많이 추천되는 MLX 4bit 모델은 `mlx-community/Qwen3.5-35B-A3B-4bit`였다. Hugging Face 기준 월 34.6k 다운로드로 압도적이었다.

---

## 환경 세팅

가상환경에서 작업한다.

```bash
cd ~/projects/agent
python3 -m venv mlx-env
source mlx-env/bin/activate
pip install mlx mlx-lm openai rich
```

- `mlx`, `mlx-lm`: Apple Silicon 전용 ML 프레임워크 + LLM 추론
- `openai`: mlx_lm.server가 제공하는 OpenAI 호환 API 호출용
- `rich`: 터미널 UI

---

## 첫 테스트: 4bit

```bash
python -m mlx_lm generate \
  --model mlx-community/Qwen3.5-35B-A3B-4bit \
  --prompt "Hello, tell me a short joke." \
  --max-tokens 100
```

결과:

```
Prompt: 18 tokens, 1.058 tokens-per-sec
Generation: 100 tokens, 49.608 tokens-per-sec
Peak memory: 19.562 GB
```

**49.6 tok/s.** 이전 Ollama + 27B에서 8.5 tok/s였으니 약 6배 빠르다. 다만 peak 메모리가 19.56GB로, 24GB 시스템에서 여유가 4.4GB밖에 안 남는다.

---

## 에이전트 변환: Ollama → MLX

이전에 만든 `agent.py`는 Ollama의 OpenAI 호환 API를 사용하고 있었다. MLX도 `mlx_lm.server`로 같은 형식의 API를 제공하므로, 핵심 구조는 유지하면서 서버 관리 부분만 바꿨다.

```
사용자 입력
    ↓
agent-mlx.py (에이전트 루프)
    ↓
mlx_lm.server (자동 시작, OpenAI 호환 API)
    ↓
Qwen3.5-35B-A3B (MLX 추론)
```

주요 변경점:

- **서버 자동 시작/종료**: 에이전트 실행 시 `mlx_lm.server`를 자동으로 띄우고, 종료 시 정리
- **기본 모델**: Ollama 모델명 대신 Hugging Face 모델 경로 사용
- **API 엔드포인트**: `localhost:11434/v1` → `localhost:8080/v1`

```python
# 서버 자동 시작
proc = subprocess.Popen(
    [python_cmd, "-m", "mlx_lm", "server",
     "--model", model,
     "--port", str(port)],
    stdout=_server_log_file,
    stderr=_server_log_file,
)

# OpenAI 호환 클라이언트
client = OpenAI(base_url="http://localhost:8080/v1", api_key="mlx")
```

도구 정의, 에이전트 루프, 실시간 스트리밍, think/no_think 전환 등은 이전 글의 구조를 그대로 사용한다.

---

## 문제 발생: 생성 중 크래시

실제로 쓰다 보니 **텍스트 생성 도중에 갑자기 모델이 꺼지는 현상**이 반복됐다.

```
> 다음 내용을 한글로 번역해줘 "Should I marry..."
── step 1/15 | ctx: 686/32,768 (2%) ──
, who's from a very traditional Indian family...
  (26 tokens, 49.0s, 0.5 tok/s)
```

0.5 tok/s로 급격히 느려지다가 서버가 죽어버렸다.

### 원인 조사

GitHub 이슈와 레딧을 뒤져본 결과, 원인은 복합적이었다.

**1. KV 캐시 메모리 누적**

MLX의 KV 캐시는 토큰 생성 시 메모리를 축적하면서 제대로 해제하지 않는다. [ml-explore/mlx-examples#724](https://github.com/ml-explore/mlx-examples/issues/724)에 따르면, 모델 로딩 후 24GB → 첫 생성 후 50GB → 두 번째 생성 후 90GB까지 캐시가 늘어나는 사례가 보고됐다.

**2. macOS GPU 메모리 한도**

macOS는 기본적으로 GPU에 전체 RAM의 65~75%만 할당한다. 24GB 시스템이면 약 16~18GB. 모델이 19.5GB를 쓰면 이 한도를 넘긴다.

**3. 도구 결과가 컨텍스트를 폭발시킴**

`curl`로 웹페이지를 가져오면 HTML 전체가 tool result로 들어가서 컨텍스트가 한 번에 5% → 26%로 점프한다. 이때 prefill 단계에서 메모리가 순간적으로 튀면서 크래시가 발생한다.

### 적용한 대책

**서버 옵션으로 메모리 제한:**

```python
proc = subprocess.Popen(
    [python_cmd, "-m", "mlx_lm", "server",
     "--model", model,
     "--port", str(port),
     "--prompt-cache-bytes", str(cache_bytes),  # KV 캐시 상한 (기본 4GB)
     "--prompt-cache-size", "1",                # 동시 캐시 1개로 제한
     "--decode-concurrency", "1",               # 병렬 디코딩 차단
     "--prompt-concurrency", "1",               # 병렬 프롬프트 차단
     "--prefill-step-size", "1024",             # prefill 메모리 피크 완화
     ],
)
```

| 설정 | Ollama 대응 | 역할 |
|------|-------------|------|
| `--prompt-cache-bytes` | `OLLAMA_KV_CACHE_TYPE` | KV 캐시 메모리 상한 |
| `--prompt-cache-size 1` | `OLLAMA_MAX_LOADED_MODELS=1` | 캐시 수 제한 |
| `--decode-concurrency 1` | `OLLAMA_NUM_PARALLEL=1` | 메모리 스파이크 방지 |
| `--prefill-step-size 1024` | - | prefill 단계 메모리 완화 |

**컨텍스트 자동 정리:**

```python
def trim_messages(messages, ctx_limit):
    """컨텍스트가 80%를 넘으면 오래된 메시지부터 제거."""
    used = estimate_tokens(messages)
    if used <= int(ctx_limit * 0.8):
        return messages
    # system 메시지는 보존, 오래된 것부터 삭제
    ...
```

**도구 출력 크기 제한:**

```python
def run_shell(command, max_output_chars=8000):
    ...
    if len(output) > max_output_chars:
        output = output[:max_output_chars] + "\n... (출력 잘림)"
    return output
```

이 세 가지로 크래시 빈도가 줄었지만, 근본적인 문제는 **모델이 19.5GB로 24GB에 너무 빡빡하다**는 점이었다.

---

## 3bit 양자화 전환

4bit(20GB)에서 메모리 여유가 4.4GB밖에 안 되니, KV 캐시가 조금만 쌓여도 터졌다. 3bit로 내리면 어떨까?

Hugging Face에서 `andrevp/Qwen3.5-35B-A3B-MLX-VLM-3bit`를 찾았다. 비전(VLM) 지원까지 포함된 3bit 변환본이다.

```bash
python -m mlx_lm generate \
  --model andrevp/Qwen3.5-35B-A3B-MLX-VLM-3bit \
  --prompt "Hello, tell me a short joke." \
  --max-tokens 100
```

결과:

```
Generation: 100 tokens, 59.491 tokens-per-sec
Peak memory: 15.239 GB
```

| 항목 | 4bit | 3bit | 변화 |
|------|------|------|------|
| 생성 속도 | 49.6 tok/s | **59.5 tok/s** | +20% |
| Peak 메모리 | 19.56 GB | **15.24 GB** | -4.3 GB |
| 24GB 기준 여유 | 4.4 GB | **8.8 GB** | 2배 |

3bit가 오히려 **더 빠르다.** 메모리 대역폭에 여유가 생기면서 GPU가 더 효율적으로 동작하기 때문이다. 품질 차이는 KL Divergence 기준으로 4bit 0.55 vs 3bit 0.95 정도인데, 실사용에서 체감 차이는 거의 없었다.

---

## 멀티모달(비전) 지원 추가

에이전트에서 "이 이미지 묘사해줘"라고 했더니 파일 메타데이터만 읽고 "시각적 내용을 분석할 수 없습니다"라고 답했다.

알고 보니 **Qwen3.5는 전 모델이 네이티브 멀티모달**이다. Qwen3까지는 텍스트(Qwen3)와 비전(Qwen3-VL)이 분리됐지만, 3.5부터는 모든 모델에 비전 인코더가 내장되어 있다.

문제는 `mlx-lm`이 텍스트 전용이라는 것. 비전 기능을 쓰려면 `mlx-vlm`이 필요하다.

```bash
pip install mlx-vlm torch torchvision
```

에이전트에 이미지 분석 도구를 추가했다:

```python
{
    "name": "analyze_image",
    "description": "이미지 파일을 분석하여 내용을 묘사합니다.",
    "parameters": {
        "properties": {
            "image_path": {"type": "string", "description": "이미지 파일 경로"},
            "prompt": {"type": "string", "description": "이미지에 대한 질문"},
        },
    },
}
```

내부적으로는 `mlx_vlm generate` 명령을 subprocess로 호출한다:

```python
result = subprocess.run(
    [python_cmd, "-m", "mlx_vlm", "generate",
     "--model", model,
     "--prompt", prompt,
     "--image", str(image_path),
     "--max-tokens", "1024"],
    capture_output=True, text=True, timeout=180,
)
```

테스트 결과:

```
Generation: 500 tokens, 55.334 tokens-per-sec
Peak memory: 17.606 GB
```

55.3 tok/s로 이미지를 묘사하면서도 메모리는 17.6GB, 여유 6.4GB. 3bit 덕분에 비전까지 써도 안정적이다.

---

## 안정성 강화

마지막으로 운영 안정성을 위한 장치들을 추가했다.

**서버 로그 파일 저장:**

```python
server_log = log_dir / "mlx-server.log"
_server_log_file = open(server_log, "w")
proc = subprocess.Popen(..., stdout=_server_log_file, stderr=_server_log_file)
```

크래시가 발생하면 `mlx-server.log`에서 원인을 추적할 수 있다.

**서버 크래시 감지 + 자동 재시작:**

```python
if _server_process and _server_process.poll() is not None:
    console.print(f"⚠ MLX 서버가 크래시됨! (exit code: {_server_process.returncode})")
    # 로그 출력 후 자동 재시작
    start_mlx_server(model, port)
```

**GPU 메모리 한도 안내:**

```python
# 시작 시 GPU 메모리 한도 확인
sysctl_result = subprocess.run(["sysctl", "iogpu.wired_limit_mb"], ...)
if sysctl_result.returncode != 0:
    console.print("GPU 메모리 한도 미설정 — 다음 명령 실행 권장:")
    console.print("  sudo sysctl iogpu.wired_limit_mb=20480")
```

`iogpu.wired_limit_mb`는 macOS가 GPU에 할당하는 메모리 한도를 올려주는 설정이다. 오버클럭이 아니라 소프트웨어 한도 조정이므로 하드웨어에 영향은 없다. 다만 3bit로 내린 이상 이 설정 없이도 충분히 안정적이라, 필수는 아니다.

---

## 사용법

```bash
cd ~/projects/agent
source mlx-env/bin/activate

# 기본 실행 (서버 자동 시작)
python agent-mlx.py

# thinking 비활성화 (더 빠른 응답)
python agent-mlx.py --no-think

# KV 캐시 제한 조정
python agent-mlx.py --cache-limit-gb 3

# 이미 서버를 따로 띄워놓은 경우
python agent-mlx.py --no-server
```

에이전트 내부 명령어:

| 명령어 | 동작 |
|--------|------|
| `quit` | 종료 (서버도 같이 종료) |
| `clear` | 대화 초기화 |
| `think` / `no_think` | 사고 모드 전환 |
| `/think <질문>` | 이번 질문만 think 모드 |

---

## Ollama vs MLX 비교

같은 Mac Mini M4 24GB에서 같은 계열 모델로 비교한 실측치다.

| 항목 | Ollama + 27B Q4 | MLX + 35B-A3B 3bit |
|------|-----------------|-------------------|
| 생성 속도 | 8.5 tok/s | **59.5 tok/s** |
| Peak 메모리 | ~22 GB | **15.2 GB** |
| 메모리 여유 | ~2 GB | **8.8 GB** |
| 비전(이미지) | 미지원 | **지원** |
| 안정성 | swap 위험 | 안정적 |
| Function Calling | 지원 | 지원 |

MLX가 약 7배 빠르면서 메모리도 적게 쓴다. MoE 구조(35B-A3B)와 MLX 최적화의 시너지 덕분이다.

---

## 정리

Mac Mini 24GB에서 로컬 LLM 에이전트를 쓰려면, 현시점 기준으로 **MLX + Qwen3.5-35B-A3B 3bit**가 가장 현실적인 조합이다.

핵심 교훈:

- **4bit는 24GB에서 빡빡하다.** 모델만 19.5GB를 먹으면 KV 캐시 여유가 없어서 크래시가 반복된다
- **3bit가 오히려 빠르다.** 메모리 대역폭 여유 → GPU 효율 증가 → 49.6 → 59.5 tok/s
- **Qwen3.5는 네이티브 멀티모달이다.** mlx-vlm만 추가하면 같은 모델로 이미지 분석까지 가능
- **서버 안정성은 직접 챙겨야 한다.** 로그 저장, 출력 크기 제한, 자동 재시작은 필수

다음에는 이 에이전트에 웹 검색, MCP 연동 등을 추가해볼 계획이다.

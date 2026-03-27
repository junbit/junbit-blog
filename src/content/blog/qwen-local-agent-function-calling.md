---
title: "Qwen3.5 Function Calling 구현 가이드 - Ollama 로컬 에이전트 만들기"
description: "Ollama + OpenAI 호환 API로 Qwen3.5의 Function Calling을 Python으로 직접 구현하는 방법. Codex CLI의 tool_calls 한계를 넘는 로컬 에이전트 구축."
pubDate: 2026-03-14
tags: ["llm", "ollama", "qwen", "agent", "function-calling"]
---

Mac Mini에서 Qwen3.5 모델을 Codex CLI로 쓰고 있었는데, 도구 호출(Function Calling) 기반의 에이전트 기능이 제대로 동작하지 않았다. 커뮤니티에서도 "Codex CLI + Ollama 조합에서 tool_calls가 안 나온다"는 보고가 꽤 있었고, 결국 직접 Python으로 에이전트를 만들기로 했다.

---

## 왜 Codex CLI로는 안 되는가

Codex CLI는 OpenAI 스타일의 구조화된 `tool_calls` / `function_call` JSON을 전제로 설계되어 있다. 문제는 Ollama + Qwen 조합이 이 프로토콜을 항상 정확히 맞춰주지 못한다는 점이다.

커뮤니티 사례를 보면:

- Codex CLI에 qwen2.5-coder:14b를 붙인 사례에서 "툴을 전혀 실행 못 한다"는 리포트가 있다
- FastAPI 래퍼로 `/v1/chat/completions`를 흉내냈지만, 모델이 구조화된 `tool_calls`를 안 뱉고 텍스트 설명만 하는 문제
- 결국 대부분 "직접 Python에서 tools 정의하고 파싱하는 커스텀 스크립트"로 귀결

코딩 보조(리팩터, 설명, 테스트 생성)는 도구 호출 없이도 되지만, **파일 시스템 수정, 셸 명령 실행 같은 자동 액션**을 하려면 Function Calling이 필수다.

---

## Qwen-Agent vs 직접 구현

Qwen 팀이 공식으로 제공하는 Qwen-Agent 프레임워크가 있다. Function Calling, MCP, Code Interpreter, RAG까지 포함된 풀 패키지다.

**Qwen-Agent가 유리한 경우:**
- MCP 서버 연동이 필요할 때
- Code Interpreter (샌드박스 Python 실행)
- RAG (문서 기반 질의응답)

**직접 구현이 유리한 경우:**
- Ollama의 OpenAI 호환 API에서 `tool_calls`가 이미 구조화되어 나올 때
- 커스텀 도구 정의가 자유로워야 할 때
- 의존성 최소화, 디버깅 용이

실제로 테스트해보니 Ollama + Qwen3.5 9B에서 `tool_calls`가 구조적으로 잘 반환된다:

```python
from openai import OpenAI

client = OpenAI(base_url='http://localhost:11434/v1', api_key='ollama')
resp = client.chat.completions.create(
    model='qwen3.5:9b',
    messages=[{'role': 'user', 'content': '현재 디렉토리 파일 목록 보여줘'}],
    tools=tools,
    tool_choice='auto',
    temperature=0.3,
)
print(resp.choices[0].finish_reason)  # 'tool_calls'
print(resp.choices[0].message.tool_calls)  # 구조화된 tool_call 객체
```

Qwen-Agent 레이어를 끼울 필요 없이 OpenAI SDK로 바로 동작하므로, 직접 구현을 선택했다.

---

## 구현 구조

```
사용자 입력
    ↓
에이전트 루프 (최대 15 스텝)
    ↓
Ollama API 호출 (스트리밍)
    ↓
┌─ tool_calls 반환 → 도구 실행 → 결과 피드백 → 루프 계속
└─ 텍스트 반환 → 최종 답변 출력 → 루프 종료
```

핵심은 **모델 호출 → tool_calls 확인 → 도구 실행 → 결과를 다시 모델에 피드백**하는 루프다.

---

## 가상환경 세팅

다른 프로젝트와 충돌을 피하기 위해 가상환경에서 작업한다.

```bash
cd ~/projects/qwen
python3 -m venv .venv
source .venv/bin/activate
pip install openai rich
```

- `openai`: Ollama의 OpenAI 호환 API 호출
- `rich`: 터미널 UI (스피너, 패널, 마크다운 렌더링)

---

## 도구 정의

OpenAI Function Calling 스펙에 맞춰 도구를 정의한다:

```python
TOOLS = [
    {
        "type": "function",
        "function": {
            "name": "run_shell",
            "description": "셸 명령어를 실행합니다.",
            "parameters": {
                "type": "object",
                "properties": {
                    "command": {"type": "string", "description": "실행할 셸 명령어"}
                },
                "required": ["command"],
            },
        },
    },
    # read_file, write_file, edit_file, list_directory, search_files ...
]
```

| 도구 | 설명 |
|------|------|
| `run_shell` | 셸 명령 실행 (ls, git, python 등) |
| `read_file` | 파일 읽기 |
| `write_file` | 파일 생성/쓰기 |
| `edit_file` | 파일 내 문자열 교체 |
| `list_directory` | 디렉토리 목록 |
| `search_files` | 파일 내용 검색 (grep) |

---

## 에이전트 루프 핵심 로직

```python
def agent_loop(client, model, messages, max_steps=15):
    for step in range(max_steps):
        content, tool_calls, thinking = call_model_streaming(client, model, messages, TOOLS)

        # 1) 구조화된 tool_calls → 도구 실행 → 결과 피드백
        if tool_calls:
            messages.append({"role": "assistant", "content": content or "", "tool_calls": tool_calls})
            for tc in tool_calls:
                fn_name = tc["function"]["name"]
                fn_args = json.loads(tc["function"]["arguments"])
                result = execute_tool(fn_name, fn_args)
                messages.append({"role": "tool", "tool_call_id": tc["id"], "content": result})
            continue

        # 2) 텍스트에서 tool_call 패턴 파싱 (Qwen fallback)
        if content:
            fn_name, fn_args = try_parse_tool_call_from_text(content)
            if fn_name:
                result = execute_tool(fn_name, fn_args)
                messages.append({"role": "assistant", "content": content})
                messages.append({"role": "user", "content": f"[도구 실행 결과]\n{fn_name}: {result}"})
                continue
            # 순수 텍스트 → 최종 답변
            messages.append({"role": "assistant", "content": content})
            return
```

Qwen이 가끔 `tool_calls` 대신 `<tool_call>` 태그를 텍스트로 뱉는 경우가 있어서, fallback 파서도 넣어두었다.

---

## 실시간 진행 표시

로컬 모델은 응답이 느릴 수 있어서, 사용자가 "멈춘 건가?" 하지 않도록 실시간 표시가 중요하다.

Rich 라이브러리의 `Live` + `Spinner`를 사용해서 3단계 표시를 구현했다:

```python
def show_spinner(message, stop_event):
    with Live(Spinner("dots", text=f" {message}"), console=console,
              refresh_per_second=10, transient=True) as live:
        start = time.time()
        while not stop_event.is_set():
            elapsed = time.time() - start
            live.update(Spinner("dots", text=f" {message} ({elapsed:.1f}s)"))
            stop_event.wait(0.1)
```

| 단계 | 표시 |
|------|------|
| 모델 응답 대기 | `⠋ 모델 응답 대기 중... (2.3s)` |
| 사고 중 (thinking) | `⠙ 사고 중... (5.1s)` |
| 도구 실행 중 | `⠹ list_directory 실행 중... (0.2s)` |

스피너는 별도 데몬 스레드에서 돌아가므로, 토큰이 안 들어오는 구간에서도 계속 회전한다.

텍스트 응답은 토큰 단위로 스트리밍 출력하고, 완료 후 성능 정보를 표시한다:

```
(126 tokens, 14.8s, 8.5 tok/s)
```

---

## Ollama 버전 주의사항

Qwen3.5 27B 모델을 쓸 때 **Ollama 0.17.6 미만 버전에서 tool calling이 완전히 고장나는 버그**가 있었다.

GitHub 이슈 #14493에서 보고된 세 가지 버그:

1. **`<think>` 태그 미닫힘**: thinking + tool_calls 조합 시 `</think>`가 누락되어 후속 턴이 전부 깨짐
2. **penalty sampling 미구현**: `repeat_penalty` 등이 API에서 받아도 무시됨
3. **generation prompt 누락**: 도구 호출 후 `<|im_end|>`가 안 나와서 루프가 끊김

v0.17.6에서 수정되었으므로 반드시 업데이트해야 한다.

```bash
# 버전 확인
ollama --version

# Ollama.app으로 설치한 경우 (brew가 아님)
curl -fsSL https://ollama.com/download/Ollama-darwin.zip -o /tmp/Ollama-darwin.zip
unzip -o /tmp/Ollama-darwin.zip -d /Applications/

# 서비스 재시작
launchctl stop com.ollama.server
launchctl start com.ollama.server
```

기존 모델, plist 환경변수, 자동 실행 설정 모두 영향 없다. 바이너리만 교체되는 구조.

---

## 사용법

```bash
cd ~/projects/qwen
source .venv/bin/activate

# 기본 (qwen3.5:9b)
python agent.py

# 27B 모델 사용
python agent.py --model "qwen3.5:27b-q4_K_M"

# thinking 비활성화 (더 빠른 응답)
python agent.py --no-think

# 대화 중 모델 변경
> model qwen3.5:27b-q4_K_M
```

| 명령어 | 동작 |
|--------|------|
| `quit` / `exit` | 종료 |
| `clear` | 대화 초기화 |
| `model <이름>` | 모델 실시간 변경 |

---

## 실행 예시

```
>>> Ollama 연결 성공 | 모델: qwen3.5:9b
╭──────────────────────────────────────────╮
│ Qwen 로컬 에이전트                         │
│ 모델: qwen3.5:9b | 최대 스텝: 15          │
╰──────────────────────────────────────────╯

> 현재 디렉토리 파일 목록 보여줘

── step 1/15 ──
╭─────────── 도구 호출 ───────────╮
│ list_directory({                 │
│   "path": "."                    │
│ })                               │
╰──────────────────────────────────╯
╭─────────── 실행 결과 ───────────╮
│ [D] .venv                        │
│ [F] agent.py (23KB)              │
│ [F] mk.png (1MB)                 │
╰──────────────────────────────────╯

── step 2/15 ──
현재 디렉토리에 있는 파일과 폴더 목록입니다:
- `.venv` (폴더)
- `agent.py` (23KB)
- `mk.png` (1MB)
  (126 tokens, 14.8s, 8.5 tok/s)
```

9B 모델이 `tool_calls`를 구조적으로 반환하고, 도구 실행 결과를 받아 자연어로 정리해준다.

---

## 정리

| 항목 | Codex CLI | 직접 구현 에이전트 |
|------|-----------|-------------------|
| 도구 호출 | 불안정 (Ollama 호환 문제) | 안정적 (OpenAI SDK 직접 호출) |
| 실시간 표시 | 없음 | 스피너 + 스트리밍 |
| 커스텀 도구 | 제한적 | 자유롭게 추가 가능 |
| 의존성 | Codex CLI + Node.js | Python + openai + rich |
| Qwen-Agent 필요 | - | 불필요 (Ollama API로 충분) |

Codex CLI는 "로컬 코딩 챗봇" 용도로는 여전히 쓸만하지만, 도구 호출 기반 에이전트로 확장하려면 직접 구현하는 게 현실적이다. Ollama 0.17.6 이상에서 Qwen3.5의 Function Calling이 안정화되면서, 복잡한 미들웨어 없이도 OpenAI SDK만으로 충분히 동작하는 환경이 됐다.

---

## 소스 코드

이 글에서 다룬 에이전트의 전체 소스 코드를 GitHub에 공개했다. 클론해서 바로 사용할 수 있다.

**GitHub**: [https://github.com/junbit/myqwenagent](https://github.com/junbit/myqwenagent)

```bash
git clone https://github.com/junbit/myqwenagent.git
cd myqwenagent
pip install openai rich
python agent.py
```

---
title: "Mac Mini 24GB에서 Qwen3.5 27B 로컬 LLM 최적화 (Ollama + Codex CLI)"
description: "Mac Mini 24GB 환경에서 Qwen3.5 27B 양자화 모델을 안정적으로 실행하기 위한 최적화 과정 정리"
pubDate: 2026-03-04
tags: ["llm", "ollama", "qwen", "mac-mini", "local-ai"]
---

최근 로컬 환경에서 대형 언어 모델을 직접 실행하는 사례가 많아지고 있다.
특히 Apple Silicon 환경에서는 unified memory 구조 덕분에 생각보다 큰 모델도 실행할 수 있다.

이번 글에서는 **Mac Mini 24GB 환경에서 Qwen3.5 27B 양자화 모델을 안정적으로 실행하기 위해 진행한 최적화 과정**을 정리한다.

목표는 다음과 같다.

- 27B 모델을 **swap 폭발 없이 안정적으로 실행**
- 로컬 LLM을 **개발 워크플로우에서 활용**
- **Codex CLI와 연결하여 코드 작업에 사용**

---

## 시스템 환경

```
Machine: Mac Mini (Apple Silicon M4)
RAM: 24GB unified memory
OS: macOS
Inference Engine: Ollama
Primary Model: Qwen3.5 27B (Q4_K_M quantization)
Context Length: 32K
```

로컬 모델 실행 구조:

```
Codex CLI
    ↓
OpenAI API compatible endpoint
    ↓
Ollama
    ↓
Qwen3.5 27B (local)
```

---

## Qwen3.5 27B 모델 선택

Mac 24GB 환경에서는 **모델 크기 선택이 매우 중요하다.**

| 모델 크기 | 안정성 |
|----------|--------|
| 7B | 매우 안정 |
| 13B | 안정 |
| 20B | 안정 |
| 27B | 한계 근처 |
| 35B | 사실상 불가능 |

선택한 모델: `qwen3.5:27b` (Q4_K_M 양자화)

예상 메모리 사용량:

```
Model weights  ≈ 17GB
KV cache       ≈ 1.5~2GB
macOS + system ≈ 3~4GB
────────────────────────
합계           ≈ 22~23GB
```

즉, **24GB 시스템에서는 거의 한계에 가까운 구성**이다.

---

## Ollama 설치 및 모델 다운로드

```bash
brew install ollama
```

```bash
ollama pull qwen3.5:27b
```

---

## 32K Context 커스텀 모델 생성

기본 모델 설정을 목적에 맞게 커스터마이징하기 위해 Modelfile을 사용해 별도 모델을 생성했다.

`Modelfile`:

```
FROM qwen3.5:27b

PARAMETER num_ctx 32768
```

모델 생성:

```bash
ollama create qwen3.5-27b-q4km-32k -f Modelfile
```

생성 확인:

```bash
ollama list
```

---

## Ollama 메모리 최적화

Mac 24GB 환경에서 안정적으로 실행하려면 환경 변수 설정이 중요하다.

```bash
OLLAMA_FLASH_ATTENTION=1
OLLAMA_KV_CACHE_TYPE=q8_0
OLLAMA_NUM_PARALLEL=1
OLLAMA_MAX_LOADED_MODELS=1
OLLAMA_KEEP_ALIVE=10m
OLLAMA_HOST=127.0.0.1:11434
```

| 옵션 | 설명 |
|------|------|
| `FLASH_ATTENTION=1` | Apple Silicon에서 attention 성능 개선 |
| `KV_CACHE_TYPE=q8_0` | KV cache 메모리 효율 개선 |
| `NUM_PARALLEL=1` | 병렬 추론으로 인한 메모리 스파이크 방지 |
| `MAX_LOADED_MODELS=1` | 모델 중복 로드 방지 |
| `KEEP_ALIVE=10m` | idle 시 모델 unload |
| `HOST=127.0.0.1` | 로컬 접근만 허용 |

---

## macOS 메모리 상태 확인

대형 모델을 실행할 때는 메모리 상태를 수시로 확인하는 게 좋다.

```bash
sudo memory_pressure
```

테스트 중 메모리 상태:

```
System-wide memory free percentage: 17~22%
```

이 범위에서는 시스템이 안정적으로 유지된다.

---

## Codex CLI와 로컬 Ollama 연결

로컬 모델을 코드 작업에 사용하기 위해 Codex CLI와 Ollama를 연결했다.

처음에는 다음 명령어를 시도했다.

```bash
ollama launch codex --model qwen3.5-27b-q4km-32k
```

**실패.** 이유는 `qwen3.5-27b-q4km-32k`가 Ollama registry 모델이 아니라 로컬에서 생성한 custom model이기 때문이다.

### 해결 방법

Codex CLI는 OpenAI API 호환 endpoint를 지원한다.
Ollama도 동일한 형식의 endpoint를 제공하므로 다음과 같이 연결할 수 있다.

```bash
OPENAI_API_KEY=ollama \
OPENAI_BASE_URL=http://127.0.0.1:11434/v1 \
codex -m qwen3.5-27b-q4km-32k
```

흐름:

```
Codex CLI → OpenAI API → Ollama (localhost:11434) → Qwen3.5 27B
```

---

## 실제 테스트 결과

| 작업 | 결과 |
|------|------|
| 긴 코드 분석 | 안정적 |
| 장문 reasoning (10k+ 토큰) | 가능 |
| 한국어 장문 생성 | 자연스러움 |
| 메모리 사용량 | 한계 근처 유지 |

---

## 운영 팁

Mac 24GB에서 27B 모델을 안정적으로 운영하려면 다음을 지켜야 한다.

**병렬 추론 금지**

```bash
OLLAMA_NUM_PARALLEL=1
```

**Context 길이 관리**
32K를 초과하면 swap 위험이 높아진다.

**모델 idle unload**

```bash
OLLAMA_KEEP_ALIVE=10m
```

---

## 결론

Mac Mini 24GB 환경에서 Qwen3.5 27B는 KV cache 최적화, 병렬 추론 제한, context 길이 관리 조건을 지키면 안정적으로 실행할 수 있다.

다만 응답 생성에 상당한 시간이 소요되기 때문에, **배치 처리나 야간 자동화처럼 대기 시간이 문제되지 않는 작업에 한해 실용적이다.** 실시간 대화나 빠른 응답이 필요한 용도라면 7B~9B 모델을 사용하는 게 현실적이다.

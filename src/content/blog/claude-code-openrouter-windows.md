---
title: "Claude Code + OpenRouter 윈도우 설정 가이드 - 무료 모델 연동 방법"
description: "윈도우에서 Claude Code에 OpenRouter를 연결해 Gemini 2.0 Flash, Mistral 등 무료 LLM 모델을 사용하는 설정 방법 단계별 안내."
pubDate: 2026-03-11
tags: ["claude-code", "openrouter", "windows", "llm"]
---

Claude Code는 성능이 뛰어나지만 기본적으로 유료 API 과금 체계다.
OpenRouter를 브릿지로 연결하면 Gemini 2.0 Flash, Mistral 등 무료 모델을 붙여서 비용 없이 사용할 수 있다.

윈도우 환경에서 이 설정을 끝내는 방법을 정리한다.

---

## 1. Claude Code 설치

터미널(CMD 또는 PowerShell)을 열고 설치:

```bash
curl -fsSL https://claude.ai/install.cmd -o install.cmd && install.cmd && del install.cmd
```

### PATH 추가 (중요)

설치 후 `claude` 명령어가 인식되지 않으면 아래 경로를 PATH에 추가해야 한다.

```
C:\Users\<사용자명>\.local\bin
```

**추가 방법:** 시스템 속성 → 환경 변수 → 사용자 변수 Path 편집 → 새로 만들기 → 위 경로 붙여넣기

---

## 2. OpenRouter 연동 환경 변수 설정

Claude Code가 Anthropic 서버 대신 OpenRouter를 바라보도록 환경 변수 4개를 설정한다.
`setx`를 사용하면 재부팅 없이 영구 저장된다.

> **주의:** 명령어 입력 후 반드시 터미널을 닫고 새로 열어야 적용된다.

```bat
:: 1. 베이스 URL 설정
setx ANTHROPIC_BASE_URL "https://openrouter.ai/api/v1"

:: 2. OpenRouter API 키 설정
setx ANTHROPIC_AUTH_TOKEN "여기에_OpenRouter_API_키_입력"

:: 3. Anthropic API 키 비우기 (OpenRouter 우선 인식을 위해 필수)
setx ANTHROPIC_API_KEY ""

:: 4. 사용할 모델 설정
setx ANTHROPIC_MODEL "openrouter/free"
```

OpenRouter API 키는 [openrouter.ai](https://openrouter.ai) 가입 후 발급받을 수 있다.

---

## 3. 실행 및 모델 확인

새 터미널에서 `claude` 실행 후 모델 확인:

```
/model
```

목록에 `openrouter/free ✓ Custom model`이 체크되어 있으면 성공.
선택되어 있지 않다면 방향키로 해당 항목을 선택한다.

---

## 주의사항

**PDF·대용량 파일 처리 시 토큰 주의**

무료 모델이라도 PDF 분석이나 대용량 파일을 컨텍스트에 포함하면 토큰 소모가 급격히 증가한다.
무료 플랜의 Rate Limit을 초과하거나 소액 과금이 발생할 수 있다.

**모델 우선순위**

환경 변수로 설정한 모델보다 실행 시 직접 지정하는 `--model` 플래그가 항상 우선 적용된다.

```bash
claude --model google/gemini-2.0-flash-exp:free
```

---

## 결론

환경 변수 4개만 설정하면 윈도우에서도 Claude Code를 무료로 사용할 수 있다.
OpenRouter에서 제공하는 다양한 무료 모델을 교체하면서 쓰기엔 충분한 구성이다.

다만 무료 모델 특성상 컨텍스트 제한이나 속도 제약이 있을 수 있으니, 복잡한 작업엔 유료 모델과 병행하는 게 현실적이다.

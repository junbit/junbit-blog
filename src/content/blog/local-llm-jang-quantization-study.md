---
title: "로컬 LLM 최신 양자화 공부 기록 - JANG, JANGTQ, Attention, Expert, Mamba 정리"
description: "MiniMax M2.7과 GLM-5.1 허깅페이스 모델 카드를 기준으로 JANG, JANGTQ, mixed-precision 양자화, attention과 expert 역할, Mamba가 어디에 속하는 개념인지 정리했다."
pubDate: 2026-04-17
tags: ["llm", "quantization", "mlx", "apple-silicon", "local-ai", "mamba"]
---

최근 로컬 LLM 쪽을 보다 보면 모델 이름이 점점 길어진다.

- `JANG_2L`
- `JANG_3L`
- `JANGTQ`
- `CRACK`
- `MXFP4`
- `NVFP4`

처음엔 전부 "양자화 이름 비슷한 것들"처럼 보였는데, 실제로는 서로 결이 다르다. 어떤 것은 **몇 비트로 압축했는지**를 말하고, 어떤 것은 **어떤 텐서를 보호했는지**를 말하고, 어떤 것은 **양자화가 아니라 탈정렬(abliteration)** 을 뜻한다.

이 글은 내가 MiniMax M2.7과 GLM-5.1 관련 허깅페이스 모델 카드를 직접 읽으면서 정리한 공부 기록이다. 결론부터 말하면, 내가 처음 이해했던 내용 중 맞는 부분도 있었지만, **`JANG_2L`과 `JANGTQ`를 섞어서 이해한 부분은 분리해서 봐야 했다.**

---

## 먼저 정정부터: `JANG_2L`과 `JANGTQ`는 같은 게 아니다

처음엔 `dealignai/MiniMax-M2.7-JANGTQ-CRACK`을 보고 "`8/6/2-bit mixed precision`이겠구나"라고 생각했는데, 실제 README를 다시 확인해보니 그건 **`JANG_2L` 쪽 설명**이었다.

허깅페이스 카드 기준으로 보면 MiniMax M2.7 계열은 적어도 이렇게 나뉜다.

| 모델 | 양자화 방식 | 크기 | 핵심 포인트 |
|------|-------------|------|-------------|
| [`dealignai/MiniMax-M2.7-JANG_2L-CRACK`](https://huggingface.co/dealignai/MiniMax-M2.7-JANG_2L-CRACK) | `JANG_2L` | 63 GB | `8/6/2-bit` affine mixed precision |
| [`dealignai/MiniMax-M2.7-JANG_3L-CRACK`](https://huggingface.co/dealignai/MiniMax-M2.7-JANG_3L-CRACK) | `JANG_3L` | 89 GB | `8/4/3-bit` affine mixed precision |
| [`dealignai/MiniMax-M2.7-JANGTQ-CRACK`](https://huggingface.co/dealignai/MiniMax-M2.7-JANGTQ-CRACK) | `JANGTQ` | 55 GB | 8-bit affine + 2-bit TurboQuant experts |

즉,

- `JANG_2L`은 **혼합 정밀도 affine 양자화**
- `JANGTQ`는 **코드북 + Hadamard 회전을 쓰는 TurboQuant 계열**

로 봐야 맞다.

이 차이를 모르고 보면 둘 다 "중요한 곳은 높게, 전문가는 낮게"처럼 설명돼서 같은 포맷처럼 느껴지는데, 실제 구현은 꽤 다르다.

---

## JANG이 먼저고, JANGTQ는 그다음이다

내가 이해한 기준으로 정리하면 이렇다.

### JANG

JANG은 Apple Silicon용 MLX 생태계에서 쓰는 **혼합 정밀도 양자화 포맷**이다.

핵심 아이디어는 단순하다.

- 모든 텐서를 한 비트폭으로 밀어버리지 않고
- 민감한 텐서는 높게 유지하고
- 덩치가 큰 텐서는 낮게 압축한다

예를 들어 `JANG_2L`은 MiniMax M2.7 카드에서 다음처럼 설명된다.

- attention: 8-bit
- embeddings: 6-bit
- experts: 2-bit

이건 "모델의 논리 회로는 보호하고, 용량 대부분을 차지하는 전문가 레이어에서 메모리를 회수한다"는 전략이다.

### JANGTQ

반면 `JANGTQ`는 여기서 한 단계 더 간다.

[`JANGQ-AI/MiniMax-M2.7-JANGTQ`](https://huggingface.co/JANGQ-AI/MiniMax-M2.7-JANGTQ)와 [`dealignai/MiniMax-M2.7-JANGTQ-CRACK`](https://huggingface.co/dealignai/MiniMax-M2.7-JANGTQ-CRACK) README 기준으로, JANGTQ는 routed expert 가중치를 단순 affine 2-bit로 줄이는 대신:

- **2-bit 코드북 양자화**
- **randomized Hadamard rotation**
- **커스텀 Metal 커널**

을 묶어서 쓴다.

정리하면 이런 그림이다.

- attention / embeddings / lm_head: 8-bit affine
- router gate, norm 같은 아주 민감한 부분: fp16 또는 높은 정밀도 유지
- routed experts: 2-bit TurboQuant

그리고 이 2-bit TurboQuant가 그냥 "거칠게 2비트로 깎은 것"이 아니라, **회전된 분포에 맞춰 학습된 작은 코드북으로 더 잘 근사하려는 방식**이라는 점이 핵심이다.

README에 나온 비교값만 보면 JANGTQ는 MiniMax M2.7에서:

- 디스크 크기: 약 55~56.5 GB
- 평균 비트: 약 2.15 bpw
- MMLU-200: 91.5% 또는 CRACK 버전 92.0%

로 표기돼 있다.

즉 "더 작고, 단순 affine 2-bit보다 덜 망가지는 2-bit 계열"을 노린 포맷이라고 보면 된다.

---

## 그럼 attention은 정확히 뭐고 왜 지켜야 할까

여기서 attention을 이해해야 왜 mixed precision이 먹히는지 감이 온다.

attention은 모델이 문장을 읽을 때 **"지금 이 토큰이 앞의 어떤 정보와 연결돼야 하는가"**를 계산하는 장치다.

예를 들어 이런 문장이 있다고 해보자.

> "동생이 사과를 먹었는데, 그것은 아주 맛있었다."

여기서 `"그것"`이 무엇인지 잡아내는 연결 회로가 attention이다. `"사과"`와 `"그것"`을 연결하는 힘이 흐트러지면 모델은 문맥을 잘못 잡고, 긴 글에서는 추론 뼈대 자체가 흔들린다.

그래서 배포자들이 attention을 8-bit 근처로 보호하려는 이유는 명확하다.

- 여기가 무너지면 문맥 연결이 깨지고
- 문맥 연결이 깨지면 결과적으로 모델이 멍청해진다

나는 이걸 **지휘부**라고 이해하는 쪽이 가장 쉬웠다.

- attention: 지휘부
- expert: 대규모 작업 부대

지휘부는 작지만 중요하고, 작업 부대는 엄청 크지만 일부 손실을 감당할 수 있다.

---

## expert는 뭐고, 왜 비트를 더 낮게 줘도 되나

MiniMax M2.7이나 GLM-5.1은 MoE, 즉 Mixture of Experts 계열이다.

이 구조에서는 모델 내부에 많은 전문가 블록이 있고, 토큰이 들어올 때마다 라우터가 그중 일부만 활성화한다.

쉽게 말해:

- 모델 전체에는 수많은 전문가가 대기하고 있고
- 실제 추론 때는 그중 몇 명만 호출된다

그래서 expert는 중요하지 않아서 낮게 주는 게 아니라, **가장 큰 메모리 덩어리이기 때문에 압축 효율이 압도적으로 좋다**는 쪽에 가깝다.

배포 카드들이 공통적으로 강조하는 포인트도 이거다.

- experts가 파라미터 대부분을 차지한다
- routed experts는 매 토큰마다 전체가 다 켜지지 않는다
- 따라서 이 부분을 낮은 비트로 줄여도 전체 체감 품질을 방어할 여지가 있다

특히 JANGTQ README는 MiniMax M2.7에서 routed expert MLP가 전체 파라미터의 약 98%라고 설명한다. 그러면 당연히 여기가 최우선 압축 타깃이 된다.

한마디로 요약하면:

- attention은 작지만 치명적이라 보호
- expert는 크기 때문에 압축 가치가 가장 큼

이다.

---

## `GLM-5.1-JANG_2S`를 보면서 더 선명해진 점

이번에 같이 찾아본 [`JANGQ-AI/GLM-5.1-JANG_2S`](https://huggingface.co/JANGQ-AI/GLM-5.1-JANG_2S) 카드도 이 철학을 꽤 분명하게 보여준다.

README 기준으로 이 모델은:

- GLM-5.1 744B MoE 기반
- `JANG_2S`
- `(critical=6, important=4, compress=2)` 비트 튜플
- 평균 2.10 bits/weight
- 디스크 크기 228 GB
- MLX Studio 전용

으로 적혀 있다.

여기서 중요한 건 `2S`가 "작은 장비용 가벼운 버전" 정도가 아니라, **critical / important / compress 계층을 나눠 비트폭을 다르게 배정하는 계열**이라는 점이다.

특히 카드에는 routed experts가 파라미터의 약 97%이고, 이 구간을 2-bit로 처리한다고 나와 있다.

즉 MiniMax에서 보던 생각과 거의 같다.

- critical path는 더 지키고
- 중요하지만 덜 민감한 구간은 중간 비트
- 전문가 대량 구간은 2-bit

MoE 거대 모델을 살리는 방법이 결국 여기로 수렴하는 느낌이었다.

다만 GLM 카드에서 한 가지 분명한 제약도 보였다.

- 아직 실험적 릴리스
- 정식 벤치마크는 미완료
- 표준 `mlx_lm`이 아니라 MLX Studio 전용 런타임 필요

즉 이런 최신 양자화 모델은 "숫자만 좋으면 아무 런타임에서나 잘 돈다"가 아니라, **포맷과 런타임이 같이 맞아야 한다**는 점도 같이 봐야 한다.

---

## 왜 일반 uniform quantization은 깨질 수 있을까

MiniMax 관련 카드들에서 가장 강한 문장이 하나 있었는데, 요지는 이거다.

> MLX uniform quantization은 MiniMax에서 사실상 망가진다.

이건 과장처럼 보일 수도 있지만, 왜 이런 말이 나오는지는 이해가 갔다.

uniform quantization은 말 그대로 **전체를 같은 방식으로 누르는 접근**이다. 그런데 거대 MoE 모델은:

- 라우터
- attention 경로
- embeddings
- experts

가 모두 같은 민감도를 갖지 않는다.

이럴 때 전부 똑같이 4-bit나 2-bit로 누르면, 가장 민감한 경로에서 먼저 붕괴가 난다. 반대로 JANG 계열은 **민감한 경로에 비트를 몰아주고**, 덩치 큰 experts에서 비트를 회수하는 식으로 균형을 맞춘다.

내가 이번에 정리하면서 배운 건, 최신 로컬 LLM 양자화는 더 이상 "4비트냐 아니냐" 수준의 얘기가 아니라는 점이다.

이제는 질문이 이렇게 바뀌었다.

- 어느 텐서를 몇 비트로 둘 건가
- 어떤 경로는 affine로 둘 건가
- 어떤 경로는 코드북으로 돌릴 건가
- 런타임이 그 포맷을 실제로 처리할 수 있나

즉 **양자화 포맷과 추론 엔진이 같이 설계되는 단계**로 들어간 느낌이다.

---

## 그럼 Mamba는 여기에 어디에 속하나

중간에 Mamba도 같이 공부했는데, 이건 양자화와는 축이 다르다.

Mamba는 "몇 비트로 압축하느냐"가 아니라, **아예 모델이 긴 문맥을 처리하는 구조 자체를 바꾸는 시도**에 가깝다.

Transformer는 attention 때문에 길이가 길어질수록 계산량과 캐시 부담이 급격히 커진다. Mamba 계열은 이 문제를 줄이기 위해 **state space model 기반으로 중요한 상태만 유지하는 방식**을 쓴다.

그래서 내 머릿속에서는 이렇게 정리됐다.

- JANG / JANGTQ: 같은 모델을 더 작게, 더 효율적으로 돌리는 기술
- Mamba: 긴 문맥을 다루는 구조 자체를 바꾸는 기술

둘은 경쟁하기도 하지만, 실제 최신 모델에서는 같이 섞일 수도 있다. 즉:

- 구조는 Mamba / hybrid
- 저장은 양자화
- 실행은 전용 런타임

이런 식으로 계층이 나뉜다.

---

## 내가 이번에 얻은 제일 중요한 기준

이번 공부를 하면서 제일 크게 바뀐 건, 모델 파일명을 보는 눈이었다.

예전에는 그냥:

- 큰 모델
- 4bit
- 돌아가면 됨

정도로 봤다.

지금은 다르게 본다.

1. 이건 uniform quant인지 mixed precision인지
2. attention과 router를 얼마나 보호했는지
3. experts를 어떤 방식으로 압축했는지
4. 이 포맷을 지원하는 런타임이 따로 필요한지
5. 모델 카드의 수치가 배포자 주장인지, 재현 가능한 벤치인지

특히 마지막이 중요했다. 허깅페이스 카드에는 흥미로운 수치가 많지만, 커뮤니티 체감이나 X/레딧 반응은 결국 참고 수준이다. **가장 먼저 봐야 할 것은 배포자가 정확히 어떤 포맷과 어떤 런타임 전제를 걸고 있는지**였다.

---

## 한 줄 정리

내 기준으로 이번 공부의 결론은 이렇다.

**로컬 LLM 최신 양자화의 핵심은 "모든 가중치를 똑같이 줄이는 것"이 아니라, attention과 routing 같은 급소는 지키고, 거대한 expert 덩어리는 더 똑똑하게 압축하는 것이다. 그리고 JANGTQ는 그 흐름의 가장 공격적인 Apple Silicon 특화 버전 중 하나다.**

---

## 참고한 원문

- [MiniMaxAI/MiniMax-M2.7](https://huggingface.co/MiniMaxAI/MiniMax-M2.7)
- [dealignai/MiniMax-M2.7-JANG_2L-CRACK](https://huggingface.co/dealignai/MiniMax-M2.7-JANG_2L-CRACK)
- [dealignai/MiniMax-M2.7-JANG_3L-CRACK](https://huggingface.co/dealignai/MiniMax-M2.7-JANG_3L-CRACK)
- [JANGQ-AI/MiniMax-M2.7-JANGTQ](https://huggingface.co/JANGQ-AI/MiniMax-M2.7-JANGTQ)
- [dealignai/MiniMax-M2.7-JANGTQ-CRACK](https://huggingface.co/dealignai/MiniMax-M2.7-JANGTQ-CRACK)
- [JANGQ-AI/GLM-5.1-JANG_2S](https://huggingface.co/JANGQ-AI/GLM-5.1-JANG_2S)
- [QuIP#: Even Better LLM Quantization with Hadamard Incoherence and Lattice Codebooks](https://arxiv.org/abs/2402.04396)
- [Mamba: Linear-Time Sequence Modeling with Selective State Spaces](https://arxiv.org/abs/2312.00752)

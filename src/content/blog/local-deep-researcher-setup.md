---
title: "local-deep-researcher 설치 가이드 - Qwen3.5 9B 로컬 리서치 에이전트"
description: "Mac Mini M4에서 Ollama + Qwen3.5 9B로 local-deep-researcher를 설치하고, 2분 만에 출처 포함 리서치 리포트를 자동 생성하는 방법."
pubDate: 2026-03-17
tags: ["llm", "ollama", "qwen", "langchain", "deep-researcher", "tavily"]
---

구글에서 뭔가 조사할 때 보통 이런 흐름을 따른다. 키워드 검색 → 링크 10개 열기 → 하나씩 읽기 → "이건 아니네" 닫기 → 다른 검색어로 재검색 → 반복. 주제가 좀만 복잡해지면 탭 20개에 30분은 금방 간다.

[local-deep-researcher](https://github.com/langchain-ai/local-deep-researcher)는 이 과정을 로컬 LLM이 대신 해주는 리서치 에이전트다. 주제 하나 던지면 AI가 검색어를 만들고, 웹에서 검색하고, 결과를 요약하고, "아직 부족한 정보가 뭐지?"를 스스로 판단해서 재검색한다. 이걸 설정한 횟수만큼 반복한 뒤 출처 포함 마크다운 리포트를 뽑아준다.

---

## 내 환경

- Mac Mini M4 24GB
- Ollama 0.17.7 (LaunchAgent로 항시 실행)
- qwen3.5:9b 모델
- Python 3.12

---

## 설치

```bash
git clone https://github.com/langchain-ai/local-deep-researcher.git
cd local-deep-researcher
python3.12 -m venv .venv
source .venv/bin/activate
pip install -e ".[dev]"
```

`.env.example`을 복사해서 `.env`를 만든다:

```
SEARCH_API=tavily
LLM_PROVIDER=ollama
LOCAL_LLM=qwen3.5:9b
OLLAMA_BASE_URL=http://localhost:11434
MAX_WEB_RESEARCH_LOOPS=2
FETCH_FULL_PAGE=false
STRIP_THINKING_TOKENS=true
USE_TOOL_CALLING=true
TAVILY_API_KEY=tvly-xxx...
```

여기까지는 README대로 하면 된다. 문제는 그 다음이다.

---

## 삽질 1: JSON mode + thinking 모델 = 무한 대기

처음에 `USE_TOOL_CALLING=false`(기본값, JSON mode)로 실행했더니 **10분이 넘도록 아무 응답이 없었다.**

원인을 찾기 위해 Ollama API를 직접 찔러봤다:

```bash
curl http://localhost:11434/api/generate -d '{
  "model": "qwen3.5:9b",
  "prompt": "Generate a search query. Reply JSON: {\"query\": \"...\"}",
  "format": "json",
  "stream": false
}'
```

응답:

```json
{
  "response": "",
  "thinking": "{\n\"Query\": \"Apple M4 chip benchmarks...\"\n}"
}
```

**`response`가 빈 문자열이고, JSON이 `thinking` 필드에 들어가 있었다.**

qwen3.5는 thinking 모델이라 모든 출력에 `<think>...</think>` 토큰을 먼저 생성한다. JSON format을 강제하면 thinking 영역에 JSON을 쓰고 실제 response는 비워버린다. 코드에서는 `result.content`(= response)를 파싱하니까 매번 빈 문자열 → JSON 디코드 실패 → fallback 쿼리로 빠지는 무한 루프였다.

**해결: `USE_TOOL_CALLING=true`로 전환.** Tool calling은 JSON 파싱이 아니라 모델의 tool_calls 구조체를 사용하므로 thinking 모델에서도 정상 동작한다.

---

## 삽질 2: thinking 토큰 = 극심한 속도 저하

tool calling으로 바꿨는데도 여전히 10분 넘게 걸렸다. 리서치 루프가 2회면 LLM 호출이 5~6번 발생하는데, **매 호출마다 수백 토큰의 thinking을 생성**하느라 실제 답변보다 생각하는 시간이 몇 배로 길었다.

`langchain-ollama`의 `ChatOllama` 소스를 뒤져보니 `reasoning`이라는 파라미터가 있었다. 이걸 `False`로 설정하면 Ollama API에 `think: false`를 전달해서 thinking을 완전히 끈다.

`graph.py`의 모든 `ChatOllama` 호출에 `reasoning=False`를 추가했다:

```python
# graph.py - get_llm 함수
return ChatOllama(
    base_url=configurable.ollama_base_url,
    model=configurable.local_llm,
    temperature=0,
    reasoning=False,  # thinking 모델 사용 시 필수!
)
```

요약 단계의 ChatOllama에도 동일하게 적용:

```python
# graph.py - summarize_sources 함수 내
llm = ChatOllama(
    base_url=configurable.ollama_base_url,
    model=configurable.local_llm,
    temperature=0,
    reasoning=False,
)
```

**결과: 10분+ → 2분으로 단축.**

---

## DuckDuckGo vs Tavily

같은 주제 "2025년 Apple M4 칩 성능 벤치마크 요약"으로 비교했다:

| 항목 | DuckDuckGo | Tavily |
|------|-----------|--------|
| 시간 | 1분 56초 | 2분 24초 |
| 소스 품질 | WEF 기사, 중국어 GPU 랭킹 (엉뚱함) | notebookcheck, GEEKOM 비교 (정확함) |
| 리포트 품질 | "벤치마크를 찾을 수 없음" | M4 멀티스레드 10%↑, GPU 15%↑ 구체적 수치 |

DuckDuckGo는 무료지만 리서치 쿼리에 대한 검색 품질이 많이 떨어진다. Tavily는 월 1000회 무료 티어가 있어서 개인 사용으로는 충분하다. **리포트 품질은 모델 크기보다 검색 소스 품질에 훨씬 더 의존한다.**

---

## CLI 한 줄로 실행하기

매번 코드를 수정하는 건 귀찮으니 CLI로 만들었다.

`cli.py`:

```python
"""CLI for local-deep-researcher."""
import sys, os
from dotenv import load_dotenv
load_dotenv()

def main():
    if len(sys.argv) < 2:
        print('Usage: research "연구 주제"')
        sys.exit(1)

    topic = " ".join(sys.argv[1:])
    print(f"\n🔍 리서치 시작: {topic}\n")

    from ollama_deep_researcher.graph import graph

    result = graph.invoke(
        {"research_topic": topic},
        config={"configurable": {
            "local_llm": os.getenv("LOCAL_LLM", "qwen3.5:9b"),
            "llm_provider": os.getenv("LLM_PROVIDER", "ollama"),
            "search_api": os.getenv("SEARCH_API", "tavily"),
            "max_web_research_loops": int(os.getenv("MAX_WEB_RESEARCH_LOOPS", "2")),
            "fetch_full_page": os.getenv("FETCH_FULL_PAGE", "false").lower() == "true",
            "ollama_base_url": os.getenv("OLLAMA_BASE_URL", "http://localhost:11434"),
            "strip_thinking_tokens": os.getenv("STRIP_THINKING_TOKENS", "true").lower() == "true",
            "use_tool_calling": os.getenv("USE_TOOL_CALLING", "true").lower() == "true",
        }}
    )

    report = result["running_summary"]
    print(report)

    # 리포트 자동 저장
    os.makedirs("reports", exist_ok=True)
    safe_name = topic[:50].replace("/", "_").replace(" ", "_")
    with open(f"reports/{safe_name}.md", "w") as f:
        f.write(f"# {topic}\n\n{report}\n")
    print(f"\n📄 리포트 저장됨")

if __name__ == "__main__":
    main()
```

`research` 쉘 스크립트:

```bash
#!/usr/bin/env bash
SCRIPT_DIR="$(cd "$(dirname "$(readlink -f "$0")")" && pwd)"
cd "$SCRIPT_DIR"
source .venv/bin/activate
python cli.py "$@"
```

`/opt/homebrew/bin/research`에 심링크를 걸면 어디서든 실행 가능:

```bash
chmod +x research
ln -sf $(pwd)/research /opt/homebrew/bin/research
```

이제 터미널에서 이렇게 쓰면 된다:

```bash
research "연구자 테자스 람다스와 마틴 웰스의 논문 선도 거래 신호 식별 방법 요약"
```

2분 뒤에 출처 포함 마크다운 리포트가 터미널에 출력되고, `reports/` 폴더에 자동 저장된다.

---

## 27B 모델은 필요 없다

qwen3.5:27b-q4_K_M도 있지만 쓸 이유가 없다:

- 이 리서처에서 LLM이 하는 일은 "검색어 생성 → 짧은 요약 → 지식 갭 파악" 수준이라 9B로 충분
- 27B는 ~4 tok/s라서 루프 2회 기준 10분 이상 소요 (9B는 2분)
- 리포트 품질 차이는 미미한데 시간만 5배

**9B + Tavily 조합이 이 용도에서의 sweet spot이다.**

---

## 정리

| 설정 | 값 |
|------|-----|
| 모델 | qwen3.5:9b |
| 검색 | Tavily (월 1000회 무료) |
| 핵심 수정 | `reasoning=False`, `USE_TOOL_CALLING=true` |
| 실행 시간 | ~2분 (루프 2회) |
| 사용법 | `research "주제"` |

thinking 모델 + JSON mode 충돌이라는 예상치 못한 삽질이 있었지만, 해결하고 나니 로컬 LLM의 실용적인 활용법을 하나 더 확보한 셈이다. 탭 20개 열어가며 30분 쓸 일을 터미널 한 줄로 2분에 끝낼 수 있다.

---

## 부록: 실제 리서치 결과물

아래는 `research "연구자 테자스 람다스와 마틴 웰스가 논문 선도 거래: 시장의 미래 가격 움직임을 예측하는 데 영향력 있는 거래의 특징 에서 이러한 신호를 식별하는 방법 요약"` 명령으로 생성된 리포트 원문이다. 약 2분 소요, 편집 없이 그대로 붙여넣었다.

> 연구자 테자스 람다스와 마틴 웰스의 논문 '선도 거래: 시장의 미래 가격 움직임을 예측하는 데 영향력 있는 거래의 특징'은 과거 데이터와 시장 요인을 기반으로 미래 가격 변화를 예측하는 핵심 개념을 다룹니다. 이 연구는 특정 거래 패턴이나 신호가 시장 참여자들의 행동을 반영하여 향후 가격 흐름을 선도할 수 있음을 강조합니다. 이를 식별하기 위해서는 먼저 과거 가격 데이터를 수집하고 전처리하는 과정이 필수적이며, 선 차트나 산점도 등을 활용한 탐색적 데이터 분석 (EDA) 을 통해 시간 경과에 따른 변동과 다양한 요인 간의 상관관계를 파악해야 합니다.
>
> 논문에서 제시된 신호를 효과적으로 포착하고 예측 모델을 구축하기 위해서는 단순한 통계적 분석을 넘어 머신러닝 및 AI 기법의 적용이 중요합니다. 예를 들어, LSTM(장단기 메모리) 과 같은 딥러닝 모델은 과거 가격 데이터뿐만 아니라 거래량과 뉴스 정서 등 비정형 데이터를 함께 학습하여 숨겨진 패턴을 찾아냅니다. 또한, 변동성 클러스터링을 설명하는 GARCH 모델이나 이동 평균, RSI 와 같은 기술적 지표를 결합한 앙상블 방법 (배깅, 부스팅, 스태킹) 을 사용하여 개별 모델의 약점을 보완하고 예측 정확도를 높일 수 있습니다. 이러한 접근법은 비트코인 등 암호화폐나 주식 시장에서 단기 추세를 예측하는 데 활용될 수 있으며, 최종적으로는 모델 성능을 평가하여 미래 가격 변화와 추세를 보다 정밀하게 예측하는 능력을 향상시킵니다.
>
> 최근 연구 동향을 살펴보면, 고빈도 주가 예측 분야에서 딥러닝 아키텍처의 성능 비교 결과가 주목받고 있습니다. 이러한 맥락에서 주의력 기반 (attention-based) 구조, 특히 트랜스포머 (Transformers) 가 순환 신경망이나 합성곱 신경망과 같은 기존 모델보다 일관되게 우수한 성능을 보인다는 연구 결과가 제시되고 있습니다. 이는 복잡한 시장 데이터 간의 장기 의존성을 효과적으로 포착할 수 있는 트랜스포머 아키텍처가 선도 거래 신호를 식별하고 미래 가격 움직임을 예측하는 데 있어 더 강력한 도구로 부상하고 있음을 시사합니다. 특히, 본 논문에서 언급된 신호 식별 및 예측 접근법에는 로컬 레벨 상태 공간 칼만 필터와 LSTM 신경망을 결합한 하이브리드 예보 방식을 포함하여 다양한 모델의 장점을 통합하려는 노력이 이루어지고 있습니다.

**Sources:**
- [가격 예측 분석 - FasterCapital](https://fastercapital.com/ko/content/%EA%B0%80%EA%B2%A9-%EC%98%88%EC%B8%A1-%EB%B6%84%EC%84%9D--%EA%B3%BC%EA%B1%B0-%EB%8D%B0%EC%9D%B4%ED%84%B0-%EB%B0%8F-%EC%8B%9C%EC%9E%A5-%EC%9A%94%EC%9D%B8%EC%9D%84-%EA%B8%B0%EB%B0%98%EC%9C%BC%EB%A1%9C-%EB%AF%B8%EB%9E%98-%EA%B0%80%EA%B2%A9-%EB%B3%80%ED%99%94-%EB%B0%8F-%EC%B6%94%EC%84%B8%EB%A5%BC-%EC%98%88%EC%B8%A1%ED%95%98%EB%8A%94-%EB%B0%A9%EB%B2%95.html)
- [Deep Learning Architectures for High-Frequency Stock Price Prediction - ResearchGate](https://www.researchgate.net/publication/395995775_Deep_Learning_Architectures_for_High-Frequency_Stock_Price_Prediction_An_Empirical_Evaluation)
- [Attention mechanisms of the Transformer architectures - ResearchGate](https://www.researchgate.net/figure/Attention-mechanisms-of-the-Transformer-architectures-used-in-this-study-a-Vanilla_fig1_389453974)

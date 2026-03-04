---
title: "로컬 LLM으로 주식 뉴스 자동화 파이프라인 구축하기"
description: "Mac Mini M4에서 Qwen 27B 모델로 매일 오전 8시 주식 뉴스 보고서를 자동 생성·발송하는 파이프라인을 구축한 과정"
pubDate: 2026-03-04
tags: ["ollama", "python", "automation", "llm", "stock"]
---

매일 오전 주식시장 개장 전에 주요 뉴스를 정리해서 이메일로 받아보고 싶었다.
유료 서비스 쓰기엔 아깝고, 직접 만들어보기로 했다.

## 목표

- 전날 오후 4시 ~ 당일 오전 7시 29분 뉴스 수집
- 평일 오전 7:30 자동 시작, 8:00 전 이메일 발송
- 비용 $0 (로컬 LLM 활용)

## 아키텍처

```
RSS 수집 (collector)
    ↓
키워드 필터링 (summarizer)
    ↓
27B 모델 보고서 생성 (reporter) — ~17분
    ↓
Gmail 발송 (mailer)
```

전체 소요 시간: 약 18분. 7:30 시작하면 7:48 완료 → 8:00 전 여유있게 도착.

## 기술 스택

| 항목 | 선택 |
|------|------|
| 뉴스 수집 | feedparser (RSS) |
| DB | PostgreSQL |
| 필터링 | 키워드 기반 (30+ 종목/경제 키워드) |
| 보고서 생성 | Ollama — Qwen 27B Q4_K_M |
| 이메일 | Gmail SMTP + 앱 비밀번호 |
| 자동화 | macOS LaunchAgent |

## RSS 소스

- 연합인포맥스
- 한국경제
- 매일경제
- Reuters (영문)

## 핵심 설계 결정: LLM 요약 대신 키워드 필터링

처음에는 9B 모델로 각 기사를 요약하려 했다.
결과: 기사 1건당 190~240초. 100건이면 6시간 이상. 완전히 비현실적.

원인 분석: Qwen 계열의 "thinking mode" — `/no_think` 플래그나 `think: false` 옵션을 줘도
모델 내부에서 체인오브소트를 돌리는 건 막을 수 없었다.

**해결**: 요약은 포기하고 키워드 기반 pre-filtering으로 전환.

```python
STOCK_KEYWORDS = [
    "주가", "증시", "코스피", "코스닥", "나스닥", "S&P",
    "금리", "달러", "환율", "채권", "인플레이션",
    "실적", "어닝", "매출", "영업이익",
    "반도체", "AI", "배터리", "전기차",
    # ...30+ 키워드
]

def is_relevant(text: str) -> bool:
    return any(kw in text for kw in STOCK_KEYWORDS)
```

0초에 100건 처리. 관련 기사만 DB에 저장 후 27B 모델이 한 번에 보고서 작성.

## 27B 모델 타임아웃

27B 모델은 17분 걸린다. 기본 타임아웃(2분)으론 당연히 실패.

```python
response = requests.post(
    f"{OLLAMA_BASE_URL}/api/generate",
    json={"model": REPORT_MODEL, "prompt": prompt, "stream": False},
    timeout=1800  # 30분
)
```

## KST 시간 처리

PostgreSQL은 UTC로 저장, Mac은 KST(+9). 쿼리에서 명시적 변환 필수:

```sql
WHERE collected_at >= (%s::date - INTERVAL '1 day' + INTERVAL '7 hours')
  AND collected_at <  (%s::date + INTERVAL '22 hours 30 minutes')
```

전날 16:00 KST = UTC 07:00, 당일 07:30 KST = UTC 전날 22:30.

## LaunchAgent 등록

```bash
cp scripts/com.stocknews.daily.plist ~/Library/LaunchAgents/
launchctl load ~/Library/LaunchAgents/com.stocknews.daily.plist
```

plist에서 평일 5개 각각 지정 (Weekday 1~5, Hour 7, Minute 30).
`ProcessType: Interactive`로 실행 중 Mac 슬립 방지.

## 결과

첫 발송 성공. 이메일에 날짜별 주요 뉴스 섹션 + 시장 요약이 Markdown → HTML로 변환되어 도착.

27B 모델이 생성하는 보고서 품질이 꽤 괜찮다. 단순 나열이 아니라 흐름을 읽어서 정리해준다.

## 소스코드

[github.com/junbit/hotdeal/stock_news](https://github.com/junbit)

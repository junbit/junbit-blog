# 개인 AI 개발 블로그 — PRD

## 개요

AI 모델을 활용한 개인 개발 프로젝트와 실험을 기록하는 블로그.
**운영 비용 $0 (도메인 제외)** 을 목표로, 정적 사이트 + 무료 호스팅 구조로 설계한다.

---

## 비용 설계 원칙

| 항목 | 선택 | 비용 |
|------|------|------|
| 사이트 생성 | Astro (정적 사이트) | 무료 |
| 호스팅 | Cloudflare Pages | 무료 (무제한 빌드) |
| 소스 저장 | GitHub | 무료 |
| 댓글 | giscus (GitHub Discussions) | 무료 |
| 이미지 | GitHub 저장소 내 포함 | 무료 |
| 검색 | Pagefind (정적 검색 라이브러리) | 무료 |
| 분석 | Cloudflare Web Analytics | 무료 |
| 도메인 | 선택사항 (없으면 *.pages.dev 사용) | $0 ~ $15/년 |

**총 운영비: $0/월** (도메인 구매 시 약 $1.2/월)

---

## 기술 스택

### 핵심
- **Astro** — 정적 사이트 생성기. Markdown/MDX 기본 지원, 빌드 결과물이 순수 HTML/CSS/JS라 빠름
- **Tailwind CSS** — 스타일링
- **GitHub** — 소스 + 콘텐츠 저장소
- **Cloudflare Pages** — GitHub push 시 자동 빌드·배포 (무료 플랜: 빌드 500회/월)

### 선택
- **giscus** — GitHub Discussions 기반 댓글 시스템 (로그인 필요 없고 스팸 없음)
- **Pagefind** — 빌드 시 검색 인덱스 자동 생성, 서버 없이 전문 검색 가능
- **Cloudflare Web Analytics** — 쿠키 없는 개인정보 친화적 방문자 분석

---

## 콘텐츠 구조

```
블로그 카테고리:

/posts
  /ai-tools        AI 도구 실험 및 리뷰 (Ollama, LLM 비교 등)
  /projects        실제 만든 프로젝트 빌드로그 (stock-news, hack-radar 등)
  /til             Today I Learned — 짧은 메모
  /setup           개발 환경 설정 기록

/about             소개 페이지
```

### 글 작성 방식
- 로컬에서 Markdown 파일로 작성
- `git push` 하면 Cloudflare Pages가 자동 빌드 → 배포 (약 1분 소요)
- 별도 CMS 없이 VSCode + GitHub로 완결

---

## 사이트 구조

```
blog/
├── src/
│   ├── content/
│   │   └── posts/          # .md / .mdx 포스트
│   ├── layouts/
│   │   ├── Base.astro
│   │   └── Post.astro
│   ├── pages/
│   │   ├── index.astro     # 홈 (최신 포스트 목록)
│   │   ├── posts/
│   │   │   └── [...slug].astro
│   │   ├── about.astro
│   │   └── tags/
│   │       └── [tag].astro
│   └── components/
│       ├── Header.astro
│       ├── Footer.astro
│       ├── PostCard.astro
│       └── TagList.astro
├── public/
│   └── images/
├── astro.config.mjs
├── tailwind.config.mjs
├── package.json
└── PRD.md
```

---

## 포스트 메타데이터 형식

```markdown
---
title: "로컬 LLM으로 주식 뉴스 자동화 만들기"
date: 2026-03-04
tags: ["ollama", "python", "automation"]
summary: "Mac Mini M4에서 Qwen 27B 모델로 매일 아침 주식 보고서를 자동 생성하는 파이프라인을 구축했다."
draft: false
---
```

---

## 배포 파이프라인

```
로컬 VSCode에서 .md 작성
        ↓
git push origin main
        ↓
Cloudflare Pages (자동 감지)
        ↓
astro build + pagefind 인덱싱
        ↓
전 세계 Cloudflare CDN 배포 (~1분)
```

### GitHub Actions 없이 운영
Cloudflare Pages가 GitHub 저장소를 직접 연결해서 빌드·배포를 처리하므로 CI/CD 설정이 별도로 불필요하다.

---

## 디자인 방향

- 코드 중심 블로그 → 코드 블록 가독성 최우선
- 다크모드 기본 지원 (시스템 설정 연동)
- 불필요한 애니메이션 없음 — 빠른 로딩 우선
- 모바일 대응

---

## v1 구현 순서

1. GitHub에 `blog` 저장소 생성 (public)
2. Astro + Tailwind 프로젝트 초기화
3. 기본 레이아웃 구현 (헤더, 포스트 목록, 포스트 페이지)
4. 태그 페이지 + Pagefind 검색 연동
5. giscus 댓글 설정
6. Cloudflare Pages 연결 (GitHub 저장소 연결 → 자동 배포)
7. 첫 포스트 작성 (stock-news 프로젝트 빌드로그)

---

## 운영 지속성을 위한 원칙

- **글감 부담 최소화**: TIL은 3줄도 OK. 완성도보다 기록 빈도 우선
- **빌드 자동화**: push 하나로 배포 완료, 관리 공수 없음
- **의존성 최소화**: 외부 서비스 의존 없이 GitHub + Cloudflare만으로 완결
- **이전 가능성**: 정적 HTML이라 어느 플랫폼으로도 이전 가능

---

## 참고: 비슷한 구조로 운영 중인 블로그 유형

- `username.pages.dev` (Cloudflare 무료 도메인)
- 또는 커스텀 도메인 연결 시 Cloudflare에서 DNS + SSL 무료 처리

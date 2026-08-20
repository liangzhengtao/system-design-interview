[English](README.md) | [中文](README.zh.md) | [日本語](README.ja.md) | [Français](README.fr.md) | [Español](README.es.md) | [العربية](README.ar.md) | [한국어](README.ko.md) | [Português](README.pt.md) | [Русский](README.ru.md) | [Deutsch](README.de.md)

<div align="center">

<img src=".banner.svg" width="100%" alt="banner">

</div>


# 시스템 디자인 인터뷰 가이드

**시스템 디자인 인터뷰를 완벽 준비하세요. 10가지 개념 + 5개 완전한 시스템 디자인과 ASCII 다이어그램.**

---

<p align="center">
  <a href="#개념">개념</a> •
  <a href="#시스템-디자인">시스템 디자인</a> •
  <a href="#사용-방법">사용 방법</a> •
  <a href="#기여하기">기여하기</a>
</p>

---

## 이 가이드를 선택해야 하는 이유

시스템 디자인 인터뷰는 대기업 기술 인터뷰에서 가장 도전적인 부분입니다. 이 가이드는 다음을 제공합니다:

- **구조화된 프레임워크** — 모든 설계 질문에 대한 단계별 접근법
- **ASCII 다이어그램** — 화이트보드에 그릴 수 있는 시각적 아키텍처
- **트레이드오프 분석** — 하나의 답이 아닌, 장단점을 포함한 여러 접근법
- **용량 추정** — 설계를 구체화하기 위한 실제 수치
- **이중 언어 콘텐츠** — 모든 파일에 완전한 중국어 버전 포함

---

## 목차

- [개념](#개념)
- [시스템 디자인](#시스템-디자인)
- [사용 방법](#사용-방법)
- [기여하기](#기여-하기)
- [라이선스](#라이선스)

---

## 개념

시스템 디자인 문제를 다루기 전에 알아야 할 기초 지식입니다.

| # | 개념 | 핵심 주제 |
|---|------|----------|
| 1 | [기초](concepts/fundamentals.md) | 확장성, CAP 정리, 일관성 모델, 대역폭 vs 처리량 |
| 2 | [로드 밸런싱](concepts/load-balancing.md) | L4/L7, 라운드 로빈, 일관된 해싱, 상태 확인 |
| 3 | [캐싱](concepts/caching.md) | Redis, CDN, 브라우저 캐시, 캐시 전략, 무효화 |
| 4 | [데이터베이스 설계](concepts/database-design.md) | SQL vs NoSQL, 인덱싱, 샤딩, 복제, ACID |
| 5 | [메시지 큐](concepts/message-queues.md) | Kafka, RabbitMQ, 이벤트 기반 아키텍처, 정확히 한 번 |

### 개념 맵

```
                        ┌─────────────────────────┐
                        │  System Design           │
                        │  Fundamentals            │
                        └────────────┬────────────┘
                                     │
                 ┌───────────────────┼───────────────────┐
                 │                   │                   │
          ┌──────▼──────┐    ┌──────▼──────┐    ┌──────▼──────┐
          │ Load Balancing│    │   Caching   │    │  Database   │
          └──────┬──────┘    └──────┬──────┘    └──────┬──────┘
                 │                   │                   │
                 └───────────────────┼───────────────────┘
                                     │
                             ┌───────▼───────┐
                             │ Message Queues │
                             └───────┬───────┘
                                     │
                        ┌────────────▼────────────┐
                        │    System Designs        │
                        └─────────────────────────┘
```

---

## 시스템 디자인

인기 있는 시스템 디자인 인터뷰 질문에 대한 완전한 end-to-end 솔루션입니다.

| # | 디자인 | 난이도 | 핵심 개념 |
|---|--------|--------|----------|
| 1 | [URL 단축기](designs/url-shortener.md) | ⭐⭐ | Base62 인코딩, 해싱, 리다이렉트, 분석 |
| 2 | [채팅 시스템](designs/chat-system.md) | ⭐⭐⭐ | WebSocket, 메시지 큐, 프레전스, 전달 |
| 3 | [뉴스 피드](designs/news-feed.md) | ⭐⭐⭐ | 팬아웃, 랭킹, 타임라인, 소셜 그래프 |
| 4 | [검색 엔진](designs/search-engine.md) | ⭐⭐⭐⭐ | 역인덱스, 크롤링, 랭킹, NLP |
| 5 | [분산 캐시](designs/distributed-cache.md) | ⭐⭐⭐⭐ | 일관된 해싱, 복제, 축출 |

---

## 사용 방법

### 인터뷰 준비용

```
Step 1: Read concepts/fundamentals.md to build your foundation
Step 2: Study each concept file (2-3 hours total)
Step 3: Practice system designs (1-2 hours each)
Step 4: Time yourself — aim for 45 minutes per design
Step 5: Review trade-offs and practice explaining decisions
```

### 인터뷰 당일

```
┌─────────────────────────────────────────────────┐
│           45-Minute Interview Structure          │
├─────────────────────────────────────────────────┤
│                                                 │
│  0-5 min    Requirements clarification           │
│                                                 │
│  5-10 min   High-level design                   │
│                                                 │
│  10-25 min  Deep dive into core components       │
│                                                 │
│  25-35 min  Scaling & trade-offs                │
│                                                 │
│  35-45 min  Monitoring & failure scenarios       │
│                                                 │
└─────────────────────────────────────────────────┘
```

### RESHADED 프레임워크

이 가이드의 각 디자인은 **RESHADED** 프레임워크를 따릅니다:

```
R - Requirements       Requirements analysis
E - Estimation          Capacity estimation
S - Storage design      Storage design
H - High-level design   High-level design
A - API design          API design
D - Detailed design     Detailed design
E - Evaluation          Trade-off evaluation
D - Distinctive component  Differentiating component
```

---

## 로드맵

- [x] 5개 핵심 개념 파일
- [x] 5개 완전한 시스템 디자인
- [x] 이중 언어 콘텐츠 (영어 + 중국어)
- [ ] 5개 추가 디자인 (Rate Limiter, Notification System, Video Streaming, Web Crawler, Key-Value Store)
- [ ] Anki 플래시카드 (간격 반복학습)
- [ ] 인터랙티브 다이어그램

---

## 기여하기

기여를 환영합니다! 자세한 내용은 [CONTRIBUTING.md](CONTRIBUTING.md)를 참조하세요.

---

## 스타 히스토리

이 가이드가 유용하다고 생각되시면, 스타를 눌러주세요!

---

## 라이선스

이 프로젝트는 MIT 라이선스에 따라 라이선스가 부여됩니다 - 자세한 내용은 [LICENSE](LICENSE) 파일을 참조하세요.

---

<p align="center">
  ❤️를 담아 <a href="https://github.com/liangzhengtao">liangzhengtao</a>가 만들었습니다
</p>

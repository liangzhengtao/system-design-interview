[English](README.md) | [中文](README.zh.md) | [日本語](README.ja.md) | [Français](README.fr.md) | [Español](README.es.md) | [العربية](README.ar.md) | [한국어](README.ko.md) | [Português](README.pt.md) | [Русский](README.ru.md) | [Deutsch](README.de.md)

<div align="center">

<img src=".banner.svg" width="100%" alt="banner">

</div>


# System Design Interview Leitfaden

**Meistern Sie Ihr System Design Interview. 10 Konzepte + 5 vollständige Systemdesigns mit ASCII-Diagrammen.**

---

<p align="center">
  <a href="#konzepte">Konzepte</a> •
  <a href="#systemdesigns">Systemdesigns</a> •
  <a href="#verwendung">Verwendung</a> •
  <a href="#mitwirken">Mitwirken</a>
</p>

---

## Warum dieser Leitfaden?

System-Design-Interviews sind der herausforderndste Teil technischer Interviews bei Top-Unternehmen. Dieser Leitfaden bietet:

- **Strukturierter Rahmen** — Schritt-für-Schritt-Ansatz für jede Designfrage
- **ASCII-Diagramme** — Visuelle Architektur, die Sie an die Tafel zeichnen können
- **Trade-off-Analyse** — Nicht nur eine Antwort, sondern mehrere Ansätze mit Vor- und Nachteilen
- **Kapazitätsschätzung** — Reale Zahlen für konkrete Designs
- **Zweisprachiger Inhalt** — Jede Datei enthält eine vollständige chinesische Version

---

## Inhaltsverzeichnis

- [Konzepte](#konzepte)
- [Systemdesigns](#systemdesigns)
- [Verwendung](#verwendung)
- [Mitwirken](#mitwirken)
- [Lizenz](#lizenz)

---

## Konzepte

Grundwissen, das Sie vor der Bearbeitung von System-Design-Fragen benötigen.

| # | Konzept | Kernthemen |
|---|---------|-----------|
| 1 | [Grundlagen](concepts/fundamentals.md) | Skalierbarkeit, CAP-Theorem, Konsistenzmodelle, Latenz vs Durchsatz |
| 2 | [Load Balancing](concepts/load-balancing.md) | L4/L7, Round-Robin, Konsistentes Hashing, Health Checks |
| 3 | [Caching](concepts/caching.md) | Redis, CDN, Browser-Cache, Cache-Strategien, Invalidierung |
| 4 | [Datenbankdesign](concepts/database-design.md) | SQL vs NoSQL, Indexierung, Sharding, Replikation, ACID |
| 5 | [Message Queues](concepts/message-queues.md) | Kafka, RabbitMQ, Event-Driven Architecture, Exactly-Once |

### Konzeptkarte

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

## Systemdesigns

Vollständige End-to-End-Lösungen für beliebte System-Design-Interviewfragen.

| # | Design | Schwierigkeit | Kernkonzepte |
|---|--------|---------------|-------------|
| 1 | [URL-Verkürzer](designs/url-shortener.md) | ⭐⭐ | Base62-Kodierung, Hashing, Redirect, Analytics |
| 2 | [Chat-System](designs/chat-system.md) | ⭐⭐⭐ | WebSocket, Message Queue, Presence, Delivery |
| 3 | [Newsfeed](designs/news-feed.md) | ⭐⭐⭐ | Fan-out, Ranking, Timeline, Social Graph |
| 4 | [Suchmaschine](designs/search-engine.md) | ⭐⭐⭐⭐ | Invertierter Index, Crawling, Ranking, NLP |
| 5 | [Verteilter Cache](designs/distributed-cache.md) | ⭐⭐⭐⭐ | Konsistentes Hashing, Replikation, Eviction |

---

## Verwendung

### Zur Interviewvorbereitung

```
Step 1: Read concepts/fundamentals.md to build your foundation
Step 2: Study each concept file (2-3 hours total)
Step 3: Practice system designs (1-2 hours each)
Step 4: Time yourself — aim for 45 minutes per design
Step 5: Review trade-offs and practice explaining decisions
```

### Am Interviewtag

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

### Das RESHADED-Framework

Jedes Design in diesem Leitfaden folgt dem **RESHADED**-Framework:

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

## Roadmap

- [x] 5 Kernkonzept-Dateien
- [x] 5 vollständige Systemdesigns
- [x] Zweisprachiger Inhalt (Englisch + Chinesisch)
- [ ] 5 weitere Designs (Rate Limiter, Benachrichtigungssystem, Video-Streaming, Web Crawler, Key-Value Store)
- [ ] Anki-Karteikarten für verteiltes Lernen
- [ ] Interaktive Diagramme

---

## Mitwirken

Wir begrüßen Beiträge! Siehe [CONTRIBUTING.md](CONTRIBUTING.md) für Details.

---

## Stern-Verlauf

Wenn Ihnen dieser Leitfaden geholfen hat, vergeben Sie bitte einen Stern!

---

## Lizenz

Dieses Projekt steht unter der MIT-Lizenz — siehe die Datei [LICENSE](LICENSE) für Details.

---

<p align="center">
  Mit ❤️ erstellt von <a href="https://github.com/liangzhengtao">liangzhengtao</a>
</p>

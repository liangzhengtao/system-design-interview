[English](README.md) | [中文](README.zh.md) | [日本語](README.ja.md) | [Français](README.fr.md) | [Español](README.es.md) | [العربية](README.ar.md) | [한국어](README.ko.md) | [Português](README.pt.md) | [Русский](README.ru.md) | [Deutsch](README.de.md)

<div align="center">

<img src=".banner.svg" width="100%" alt="banner">

</div>


# Guia de Entrevista de Design de Sistemas

**Arrase na sua entrevista de design de sistemas. 10 conceitos + 5 designs completos de sistemas com diagramas ASCII.**

---

<p align="center">
  <a href="#conceitos">Conceitos</a> •
  <a href="#designs-de-sistemas">Designs de Sistemas</a> •
  <a href="#como-usar">Como Usar</a> •
  <a href="#contribuir">Contribuir</a>
</p>

---

## Por que este Guia?

Entrevistas de design de sistemas são a parte mais desafiadora das entrevistas técnicas nas principais empresas. Este guia fornece:

- **Framework estruturado** — Abordagem passo a passo para qualquer questão de design
- **Diagramas ASCII** — Arquitetura visual que você pode desenhar no quadro branco
- **Análise de trade-offs** — Não apenas uma resposta, mas múltiplas abordagens com prós/contras
- **Estimativa de capacidade** — Números reais para tornar seu design concreto
- **Conteúdo bilíngue** — Cada arquivo possui uma versão completa em chinês

---

## Índice

- [Conceitos](#conceitos)
- [Designs de Sistemas](#designs-de-sistemas)
- [Como Usar](#como-usar)
- [Contribuir](#contribuir)
- [Licença](#licença)

---

## Conceitos

Conhecimento fundamental necessário antes de enfrentar questões de design de sistemas.

| # | Conceito | Tópicos-Chave |
|---|----------|---------------|
| 1 | [Fundamentos](concepts/fundamentals.md) | Escalabilidade, teorema CAP, modelos de consistência, latência vs throughput |
| 2 | [Load Balancing](concepts/load-balancing.md) | L4/L7, round-robin, hashing consistente, health checks |
| 3 | [Cache](concepts/caching.md) | Redis, CDN, cache do navegador, estratégias de cache, invalidação |
| 4 | [Design de Banco de Dados](concepts/database-design.md) | SQL vs NoSQL, indexação, sharding, replicação, ACID |
| 5 | [Filas de Mensagens](concepts/message-queues.md) | Kafka, RabbitMQ, arquitetura orientada a eventos, exatamente uma vez |

### Mapa de Conceitos

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

## Designs de Sistemas

Soluções completas end-to-end para questões populares de entrevista de design de sistemas.

| # | Design | Dificuldade | Conceitos-Chave |
|---|--------|-------------|-----------------|
| 1 | [Encurtador de URL](designs/url-shortener.md) | ⭐⭐ | Codificação Base62, hashing, redirecionamento, analytics |
| 2 | [Sistema de Chat](designs/chat-system.md) | ⭐⭐⭐ | WebSocket, fila de mensagens, presença, entrega |
| 3 | [Feed de Notícias](designs/news-feed.md) | ⭐⭐⭐ | Fan-out, ranking, timeline, grafo social |
| 4 | [Mecanismo de Busca](designs/search-engine.md) | ⭐⭐⭐⭐ | Índice invertido, crawling, ranking, NLP |
| 5 | [Cache Distribuído](designs/distributed-cache.md) | ⭐⭐⭐⭐ | Hashing consistente, replicação, eviction |

---

## Como Usar

### Para Preparação de Entrevista

```
Step 1: Read concepts/fundamentals.md to build your foundation
Step 2: Study each concept file (2-3 hours total)
Step 3: Practice system designs (1-2 hours each)
Step 4: Time yourself — aim for 45 minutes per design
Step 5: Review trade-offs and practice explaining decisions
```

### Para o Dia da Entrevista

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

### Framework RESHADED

Cada design neste guia segue o framework **RESHADED**:

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

- [x] 5 arquivos de conceitos fundamentais
- [x] 5 designs completos de sistemas
- [x] Conteúdo bilíngue (inglês + chinês)
- [ ] 5 designs adicionais (Rate Limiter, Sistema de Notificações, Streaming de Vídeo, Web Crawler, Key-Value Store)
- [ ] Flashcards Anki para repetição espaçada
- [ ] Diagramas interativos

---

## Contribuir

Aceitamos contribuições! Consulte [CONTRIBUTING.md](CONTRIBUTING.md) para detalhes.

---

## Histórico de Estrelas

Se este guia foi útil para você, por favor dê uma estrela!

---

## Licença

Este projeto está licenciado sob a licença MIT - veja o arquivo [LICENSE](LICENSE) para detalhes.

---

<p align="center">
  Feito com ❤️ por <a href="https://github.com/liangzhengtao">liangzhengtao</a>
</p>

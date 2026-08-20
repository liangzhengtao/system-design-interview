[English](README.md) | [中文](README.zh.md) | [日本語](README.ja.md) | [Français](README.fr.md) | [Español](README.es.md)

<div align="center">

<img src=".banner.svg" width="100%" alt="banner">

</div>


# Guía de Entrevista de Diseño de Sistemas

**Domina tu entrevista de diseño de sistemas. 10 conceptos + 5 diseños completos con diagramas ASCII.**

---

<p align="center">
  <a href="#conceptos">Conceptos</a> •
  <a href="#diseños-de-sistemas">Diseños de sistemas</a> •
  <a href="#cómo-usar">Cómo usar</a> •
  <a href="#contribuir">Contribuir</a>
</p>

---

## ¿Por qué esta guía?

Las entrevistas de diseño de sistemas son la parte más desafiante de las entrevistas técnicas en empresas top. Esta guía proporciona:

- **Un marco estructurado** — Un enfoque paso a paso para cualquier pregunta de diseño
- **Diagramas ASCII** — Arquitecturas visuales que puedes dibujar en una pizarra
- **Análisis de trade-offs** — No una sola respuesta, sino múltiples enfoques con pros/contras
- **Estimación de capacidad** — Números reales para hacer tu diseño concreto
- **Contenido bilingüe** — Cada archivo tiene una versión completa en chino

---

## Tabla de contenidos

- [Conceptos](#conceptos)
- [Diseños de sistemas](#diseños-de-sistemas)
- [Cómo usar](#cómo-usar)
- [Contribuir](#contribuir)
- [Licencia](#licencia)

---

## Conceptos

Conocimientos fundamentales que necesitas antes de abordar preguntas de diseño de sistemas.

| # | Concepto | Temas clave |
|---|---------|------------|
| 1 | [Fundamentos](concepts/fundamentals.md) | Escalabilidad, teorema CAP, modelos de consistencia, latencia vs throughput |
| 2 | [Balanceo de carga](concepts/load-balancing.md) | L4/L7, round-robin, hashing consistente, health checks |
| 3 | [Caché](concepts/caching.md) | Redis, CDN, caché del navegador, estrategias de caché, invalidación |
| 4 | [Diseño de base de datos](concepts/database-design.md) | SQL vs NoSQL, indexación, sharding, replicación, ACID |
| 5 | [Colas de mensajes](concepts/message-queues.md) | Kafka, RabbitMQ, arquitectura orientada a eventos, exactly-once |

### Mapa de conceptos

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

## Diseños de sistemas

Soluciones completas de extremo a extremo para preguntas populares de entrevista de diseño de sistemas.

| # | Diseño | Dificultad | Conceptos clave |
|---|--------|------------|--------------|
| 1 | [Acortador de URL](designs/url-shortener.md) | ⭐⭐ | Codificación Base62, hashing, redirección, analítica |
| 2 | [Sistema de chat](designs/chat-system.md) | ⭐⭐⭐ | WebSocket, cola de mensajes, presencia, entrega |
| 3 | [Noticias/feed](designs/news-feed.md) | ⭐⭐⭐ | Fan-out, ranking, línea temporal, grafo social |
| 4 | [Motor de búsqueda](designs/search-engine.md) | ⭐⭐⭐⭐ | Índice invertido, crawling, ranking, NLP |
| 5 | [Caché distribuido](designs/distributed-cache.md) | ⭐⭐⭐⭐ | Hashing consistente, replicación, evicción |

---

## Cómo usar

### Para preparación de entrevistas

```
Paso 1: Lee concepts/fundamentals.md para construir tu base
Paso 2: Estudia cada archivo de conceptos (2-3 horas en total)
Paso 3: Practica diseños de sistemas (1-2 horas cada uno)
Paso 4: Cronométrate — apunta a 45 minutos por diseño
Paso 5: Revisa los trade-offs y practica explicar tus decisiones
```

### El día de la entrevista

```
┌─────────────────────────────────────────────────┐
│        Estructura de la entrevista (45 min)      │
├─────────────────────────────────────────────────┤
│                                                 │
│  0-5 min    Clarificación de requisitos          │
│                                                 │
│  5-10 min   Diseño de alto nivel                 │
│                                                 │
│  10-25 min  Profundización en componentes clave  │
│                                                 │
│  25-35 min  Escalabilidad y trade-offs           │
│                                                 │
│  35-45 min  Monitoreo y escenarios de fallo      │
│                                                 │
└─────────────────────────────────────────────────┘
```

### El marco RESHADED

Cada diseño en esta guía sigue el marco **RESHADED**:

```
R - Requirements       Análisis de requisitos
E - Estimation          Estimación de capacidad
S - Storage design      Diseño de almacenamiento
H - High-level design   Diseño de alto nivel
A - API design          Diseño de API
D - Detailed design     Diseño detallado
E - Evaluation          Evaluación de trade-offs
D - Distinctive component  Componente diferenciador
```

---

## Hoja de ruta

- [x] 5 archivos de conceptos fundamentales
- [x] 5 diseños de sistemas completos
- [x] Contenido bilingüe (inglés + chino)
- [ ] 5 diseños más (limitador de tasa, sistema de notificaciones, streaming de video, web crawler, almacén clave-valor)
- [ ] Tarjetas Anki para repetición espaciada
- [ ] Diagramas interactivos

---

## Contribuir

¡Las contribuciones son bienvenidas! Consulta [CONTRIBUTING.md](CONTRIBUTING.md) para más detalles.

---

## Historial de estrellas

Si esta guía te resulta útil, ¡dale una estrella!

---

## Licencia

Este proyecto está licenciado bajo la Licencia MIT — consulta el archivo [LICENSE](LICENSE) para más detalles.

---

<p align="center">
  Made with ❤️ by <a href="https://github.com/liangzhengtao">liangzhengtao</a>
</p>

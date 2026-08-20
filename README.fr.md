[English](README.md) | [中文](README.zh.md) | [日本語](README.ja.md) | [Français](README.fr.md) | [Español](README.es.md) | [العربية](README.ar.md) | [한국어](README.ko.md) | [Português](README.pt.md) | [Русский](README.ru.md) | [Deutsch](README.de.md)

<div align="center">

<img src=".banner.svg" width="100%" alt="banner">

</div>


# Guide d'entretien de conception système

**Réussissez votre entretien de conception système. 10 concepts + 5 conceptions complètes avec des diagrammes ASCII.**

---

<p align="center">
  <a href="#concepts">Concepts</a> •
  <a href="#conceptions-système">Conceptions système</a> •
  <a href="#comment-lutiliser">Comment l'utiliser</a> •
  <a href="#contribuer">Contribuer</a>
</p>

---

## Pourquoi ce guide ?

Les entretiens de conception système sont la partie la plus difficile des entretiens techniques dans les grandes entreprises. Ce guide fournit :

- **Un cadre structuré** — Une approche pas à pas pour toute question de conception
- **Des diagrammes ASCII** — Des architectures visuelles que vous pouvez dessiner au tableau blanc
- **Analyse des compromis** — Pas une seule réponse, mais plusieurs approches avec leurs avantages/inconvénients
- **Estimation de capacité** — Des chiffres concrets pour rendre votre conception tangible
- **Contenu bilingue** — Chaque fichier dispose d'une version chinoise complète

---

## Table des matières

- [Concepts](#concepts)
- [Conceptions système](#conceptions-système)
- [Comment l'utiliser](#comment-lutiliser)
- [Contribuer](#contribuer)
- [Licence](#licence)

---

## Concepts

Connaissances fondamentales à maîtriser avant d'aborder les questions de conception système.

| # | Concept | Sujets clés |
|---|---------|------------|
| 1 | [Fondamentaux](concepts/fundamentals.md) | Évolutivité, théorème CAP, modèles de cohérence, latence vs débit |
| 2 | [Équilibrage de charge](concepts/load-balancing.md) | L4/L7, round-robin, hachage cohérent, health checks |
| 3 | [Mise en cache](concepts/caching.md) | Redis, CDN, cache navigateur, stratégies de cache, invalidation |
| 4 | [Conception de base de données](concepts/database-design.md) | SQL vs NoSQL, indexage, sharding, réplication, ACID |
| 5 | [Files de messages](concepts/message-queues.md) | Kafka, RabbitMQ, architecture événementielle, exactly-once |

### Carte des concepts

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

## Conceptions système

Solutions complètes de bout en bout pour les questions populaires d'entretien de conception système.

| # | Conception | Difficulté | Concepts clés |
|---|--------|------------|--------------|
| 1 | [Raccourcisseur d'URL](designs/url-shortener.md) | ⭐⭐ | Encodage Base62, hachage, redirection, analytique |
| 2 | [Système de chat](designs/chat-system.md) | ⭐⭐⭐ | WebSocket, file de messages, présence, livraison |
| 3 | [Fil d'actualités](designs/news-feed.md) | ⭐⭐⭐ | Fan-out, classement, chronologie, graphe social |
| 4 | [Moteur de recherche](designs/search-engine.md) | ⭐⭐⭐⭐ | Index inversé, crawling, classement, NLP |
| 5 | [Cache distribué](designs/distributed-cache.md) | ⭐⭐⭐⭐ | Hachage cohérent, réplication, eviction |

---

## Comment l'utiliser

### Pour la préparation d'entretien

```
Étape 1 : Lisez concepts/fundamentals.md pour construire vos bases
Étape 2 : Étudiez chaque fichier de concepts (2-3 heures au total)
Étape 3 : Entraînez-vous sur les conceptions système (1-2 heures chacune)
Étape 4 : Chronométrez-vous — visez 45 minutes par conception
Étape 5 : Révisez les compromis et entraînez-vous à expliquer vos décisions
```

### Le jour de l'entretien

```
┌─────────────────────────────────────────────────┐
│          Structure de l'entretien (45 min)       │
├─────────────────────────────────────────────────┤
│                                                 │
│  0-5 min    Clarification des exigences          │
│                                                 │
│  5-10 min   Conception de haut niveau            │
│                                                 │
│  10-25 min  Approfondissement des composants     │
│             clés                                 │
│  25-35 min  Mise à l'échelle et compromis        │
│                                                 │
│  35-45 min  Surveillance et scénarios de panne   │
│                                                 │
└─────────────────────────────────────────────────┘
```

### Le cadre RESHADED

Chaque conception de ce guide suit le cadre **RESHADED** :

```
R - Requirements       Analyse des exigences
E - Estimation          Estimation de la capacité
S - Storage design      Conception du stockage
H - High-level design   Conception de haut niveau
A - API design          Conception de l'API
D - Detailed design     Conception détaillée
E - Evaluation          Évaluation des compromis
D - Distinctive component  Composant différenciateur
```

---

## Feuille de route

- [x] 5 fichiers de concepts fondamentaux
- [x] 5 conceptions système complètes
- [x] Contenu bilingue (anglais + chinois)
- [ ] 5 conceptions supplémentaires (limiteur de débit, système de notifications, streaming vidéo, web crawler, stockage clé-valeur)
- [ ] Cartes Anki pour la répétition espacée
- [ ] Diagrammes interactifs

---

## Contribuer

Les contributions sont les bienvenues ! Voir [CONTRIBUTING.md](CONTRIBUTING.md) pour plus de détails.

---

## Historique des étoiles

Si vous trouvez ce guide utile, n'hésitez pas à lui donner une étoile !

---

## Licence

Ce projet est sous licence MIT — voir le fichier [LICENSE](LICENSE) pour les détails.

---

<p align="center">
  Made with ❤️ by <a href="https://github.com/liangzhengtao">liangzhengtao</a>
</p>

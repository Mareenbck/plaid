# CLAUDE.md — Plaid

Ce fichier donne le contexte du projet à Claude Code. À garder à jour quand une décision change.

## Portée de ce fichier

Plaid est un **projet perso**. Pour ce repo uniquement, ce fichier **prime** sur les instructions globales chargées automatiquement (`~/.claude/CLAUDE.md`, `~/AGENTS.md`), qui décrivent les conventions de mes projets pro.

- En cas de conflit, suivre ce fichier. Ne s'appliquent pas ici : MongoDB (→ PostgreSQL), Yarn (→ npm), Vitest (→ Jest), les commandes `yarn ...` (→ tableau « Installation et commandes »), le design system `@myjobglasses/dsm` (→ charte « Mix 1 »).
- Sur les points que ce fichier ne précise pas, les conventions globales restent valables tant qu'elles sont compatibles avec la stack ci-dessous.

## Le projet

Plaid est une app web perso de recommandation de films et de séries (hors cinéma, disponibles en France).

- Chaque semaine, un **top 10 films** et un **top 10 séries**, avec la plateforme où les regarder.
- Chaque titre a deux scores : **qualité globale** et **affinité personnelle**. Le classement combine les deux.
- Les votes de l'utilisateur (j'aime, pas pour moi, déjà vu) réajustent les recommandations suivantes.
- Un **mode à deux** recommande ce qui plaira aux deux personnes d'un duo.
- Autres features V1 : inscription en 4 étapes, section « Noter ce que j'ai vu », mode « Ce soir », Ma liste + alertes, Recap du mois, curseur découverte.

## Stack

| Brique          | Choix                                                                      |
| --------------- | -------------------------------------------------------------------------- |
| Backend         | Ruby on Rails 8, mode API                                                  |
| API             | GraphQL (graphql-ruby)                                                     |
| Front           | React + TypeScript + Apollo Client, build Vite                             |
| Base de données | PostgreSQL (extensions : `pg_trgm`, `unaccent` ; `pgvector` plus tard)     |
| Tâches de fond  | Solid Queue (tâches planifiées dans `config/recurring.yml`)                |
| Auth            | Générateur d'auth Rails 8 + OmniAuth (Google, Apple) + lien magique par e-mail |
| E-mails         | Action Mailer                                                              |
| Tests           | RSpec (API), Jest (web), Cypress (parcours clés)                           |
| CI              | CircleCI                                                                   |
| Déploiement     | Kamal                                                                      |

Rails 8 est nouveau pour moi : privilégie ses choix par défaut (Solid Queue, Solid Cache, générateur d'auth, Kamal) et explique brièvement ce qui change par rapport à Rails 7 quand tu les utilises.

## Structure du repo

```
plaid/
├── api/                 # Rails 8 (mode API)
├── web/                 # React + TypeScript + Vite
├── docker-compose.yml   # PostgreSQL en local
├── .mise.toml           # versions Ruby et Node
├── bin/setup            # installation complète
├── bin/dev              # lance API + web
└── CLAUDE.md
```

## Installation et commandes

Le projet doit s'installer sur n'importe quel Mac **sans accès extérieur** : seulement Git, mise et Docker.

```bash
mise install                  # installe les versions Ruby/Node du projet
docker compose up -d db       # PostgreSQL en conteneur
bin/setup                     # gems, paquets npm, base, données de démo
bin/dev                       # API + front en local
```

| Action        | Commande                            |
| ------------- | ----------------------------------- |
| Tests API     | `cd api && bundle exec rspec`       |
| Lint API      | `cd api && bundle exec rubocop`     |
| Tests front   | `cd web && npm test`                |
| Lint front    | `cd web && npm run lint`            |
| Types GraphQL | `cd web && npm run codegen`         |
| Jobs en local | lancés par `bin/dev` (Solid Queue)  |

Si une commande de ce tableau n'existe pas encore, crée-la plutôt que de contourner.

## Conventions

- **Code, noms de fichiers, commits et commentaires en anglais** ; textes de l'interface en **français**.
- Une branche et une PR par ticket Asana : `feat/1-2-tmdb-sync`, `fix/...`, `chore/...` (le numéro reprend celui du ticket, ex. 1.2).
- Commits courts au format Conventional Commits (`feat:`, `fix:`, `chore:`, `test:`).
- Toute logique métier (scores, affinité, duo) est **testée en RSpec**, y compris les cas limites.
- Pas de secret dans le code : clés API dans `rails credentials`, jamais dans `.env` commité.
- Préférer les solutions natives de Rails et de PostgreSQL à une nouvelle dépendance. Demander avant d'ajouter un service externe (Redis, moteur de recherche, etc.).
- Petites PR, faciles à relire.

## Sources de données

- **TMDB** : catalogue, détails, genres, mots-clés, tendances, plateformes en France (watch providers, région FR).
- **OMDb** : notes IMDb, Rotten Tomatoes et Metacritic, via l'`imdb_id`. **Quota gratuit : 1 000 requêtes par jour**, à ne jamais dépasser. Prioriser les titres candidats au top, rafraîchir au plus une fois par semaine.
- Stocker les réponses brutes des API en **JSONB** pour pouvoir recalculer sans rappeler les API.
- Rotten Tomatoes et Metacritic manquent souvent pour les séries : le calcul doit le gérer.
- Allociné et SensCritique ne sont **pas utilisés** (pas d'API publique, scraping interdit par leurs conditions d'utilisation).
- **Mentions obligatoires** dans l'interface : logo TMDB et mention indiquant que l'app utilise l'API TMDB sans être approuvée par TMDB ; crédit **JustWatch** pour les données de plateformes.

## Règles métier

### Score qualité (sur 100)

| Critère                         | Poids | Source                              |
| ------------------------------- | ----- | ----------------------------------- |
| Notes spectateurs               | 35 %  | IMDb, TMDB                          |
| Critiques professionnelles      | 25 %  | Rotten Tomatoes, Metacritic         |
| Qualité narrative et artistique | 20 %  | proxy V1, défini dans le ticket 2.1 |
| Popularité et réception récente | 10 %  | tendance TMDB                       |
| Originalité                     | 10 %  | proxy V1, défini dans le ticket 2.1 |

- Chaque source est ramenée sur 0-100.
- **Moyenne bayésienne** pour les titres avec peu d'avis : `(v / (v + m)) × R + (m / (v + m)) × C`, où R est la note du titre, v son nombre de votes, C la moyenne du catalogue et m un seuil de votes.
- Si une source manque, son poids est réparti sur les autres et un **indice de confiance** est enregistré.
- Un titre sans aucune note reçoit la moyenne du catalogue C.

### Notes de l'utilisateur

| Réponse       | Valeur                                                     |
| ------------- | ---------------------------------------------------------- |
| Adoré         | +2                                                         |
| Bien aimé     | +1                                                         |
| Vu, sans avis | 0                                                          |
| Pas aimé      | −2 (pèse plus fort, pour ne pas reproposer ce qui a déplu) |
| Pas vu        | aucune note enregistrée                                    |

### Affinité et classement

- Le profil de goût est calculé à partir des notes : genres, mots-clés TMDB, intrigue complexe, rythme, ambiance.
- **Rang final = 0,4 × qualité + 0,6 × affinité.**
- Un « Pas pour moi » sur le top 10 retire le titre au prochain rafraîchissement.
- **Curseur découverte** : un titre sur dix (réglable) est volontairement hors des habitudes, et signalé comme tel.
- L'affinité est recalculée en tâche de fond après chaque vote.

### Mode à deux (formule validée)

```
affinité duo = moyenne(A1, A2) − 0,3 × max(0, |A1 − A2| − 10)
```

- **Veto** : le titre est écarté si l'une des deux personnes a exclu le genre ou n'a pas aimé une saison précédente.
- Les votes faits en mode « à deux » alimentent les deux profils.
- La seconde personne a son propre profil (invitation par e-mail ou lien), ou un profil invité créé en notant 5 titres sur le même appareil.

## Tâches de fond (Solid Queue)

| Job               | Fréquence                  | Rôle                                              |
| ----------------- | -------------------------- | ------------------------------------------------- |
| `SyncTmdbJob`     | chaque jour                | nouveautés, tendances, plateformes FR             |
| `EnrichOmdbJob`   | chaque jour, dans le quota | IMDb, Rotten Tomatoes, Metacritic                 |
| `QualityScoreJob` | chaque nuit                | score qualité + indice de confiance               |
| `AffinityJob`     | à chaque vote              | profil de goût et affinités                       |
| `AlertsJob`       | chaque jour                | nouvelles saisons, arrivées et départs → e-mails  |

## Modèle de données (principales tables)

`users`, `duos`, `titles`, `title_sources`, `quality_scores`, `ratings`, `taste_profiles`, `platforms`, `availabilities`, `watchlist_items`, `alerts`.

## Front : charte « Mix 1 »

- Fond lavande `#F2F0FF`, cartes blanches très arrondies (20 à 28 px).
- Accent violet `#6C4CF5`, second accent citron vert `#E3F98A`, texte `#17142B`, texte secondaire `#5A5580`.
- Titres en **Unbounded**, texte en **DM Sans** (Google Fonts).
- Mise en page desktop : liste du top 10 à gauche ; à droite, cartes « Ce que Plaid a compris », « Mes plateformes », « Ma liste » et « Ce soir ».
- Accessibilité : zones cliquables d'au moins 44 px, contraste texte d'au moins 4,5:1, vrais `<button>` et `<a>`, `aria-label` sur les boutons qui n'affichent qu'une icône.
- Pas d'emoji dans l'interface : icônes SVG au trait.

## Liens

- Suivi des tickets (Asana) : <https://app.asana.com/1/117662020128811/project/1220677773763360>
- Décision d'architecture (Notion) : <https://app.notion.com/p/41cec71f9747810494c7f38f1643124e>
- Maquettes : <https://claude.ai/artifact/U8YfWZEDisDWAgJuhUQmNV>
- Repo : <https://github.com/Mareenbck/plaid>

Ces liens pointent pour l'instant vers des espaces professionnels : les mettre à jour quand le suivi sera déplacé vers des espaces perso.

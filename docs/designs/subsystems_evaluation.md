# Évaluation des sous-systèmes de rotki

Évaluation technique des sous-systèmes existants, au commit `a0fc8a5` (`develop`, version `1.44.1`).
Document complémentaire de `architecture_overview.md`. Il sert de référence pour prioriser
la dette technique et les tests ; il ne décrit pas de changement de code.

## 1. Méthode

Les indicateurs ci-dessous sont mesurés sur le dépôt. Ils ne remplacent pas une revue de code.

| Indicateur | Définition |
|------------|-----------|
| LOC source | lignes des fichiers source hors tests (`.py`, `.rs`, `.ts`, `.vue`) |
| Fichiers de test | fichiers `test_*.py` (backend), `*.spec.ts` (frontend), tests Rust |
| `Any` | occurrences de `Any` / `# type: ignore` (typage faible) |
| `noqa` | suppressions de linter ; `E501` (lignes longues) domine presque partout |
| TODO / FIXME | marqueurs de dette explicites |
| Couplage | imports inter-couches (ex. `chain/` important `api/`) |

Les tests ont été exécutés lorsque l'environnement le permettait (voir section 4).

## 2. Vue synthétique

| Sous-système | LOC source | Fichiers | Tests associés | `Any` | TODO | Verdict |
|--------------|-----------:|---------:|----------------|------:|-----:|---------|
| `chain/` (EVM, Ethereum, L2, Solana, BTC…) | 103 468 | 1 586 | nombreux (`tests/unit`, `integration`) | 188 | 29 | **Fort volume, dette de structure** |
| `api/` (REST, v1, services, WS) | 24 862 | 35 | 84 fichiers `tests/api` | 347 | 9 | **Fonctionnel, monolithes** |
| `db/` (DB utilisateur, filtres, migrations) | 30 166 | 70 | 21 fichiers `tests/db` | 84 | 6 | **Cœur critique, complexité élevée** |
| `exchanges/` | 17 933 | 28 | 25 fichiers | 39 | 3 | **Bien couvert, fichiers volumineux** |
| `externalapis/` | 11 952 | 28 | 22 fichiers | 46 | 2 | Correct |
| `globaldb/` | 7 997 | 38 | (tests transverses) | 5 | 6 | Solide, `handler.py` volumineux |
| `history/` | 5 648 | 30 | (tests transverses) | 31 | 1 | Correct |
| `accounting/` | 4 181 | 22 | tests dédiés (`tests/unit/accounting`, cost basis, PnL) | 35 | 5 | **Cœur métier, bien testé** |
| `tasks/` | 7 596 | 11 | 7 fichiers référencent les tâches | 17 | 0 | Correct, bien documenté |
| `concurrency/` | 363 | 3 | 16 fichiers référencent l'annulation | 9 | 0 | **Petit module propre** |
| `mcp/` | 2 454 | 18 | 10 fichiers `tests/mcp` | 20 | 0 | Récent, bien isolé |
| `premium/` | 1 637 | 3 | (transverses) | 7 | 0 | Compact |
| `data_import/` | 5 131 | 24 | (transverses) | 28 | 0 | Correct |
| Frontend `frontend/app` | ~117 000 (ts) + 947 `.vue` | ~2 000 | 1 222 `*.spec.ts`, e2e, contrats | 93 `any` | — | **Large, bien testé** |
| `colibri/` (Rust) | 6 680 | ~25 | 303 tests Rust (colibri + crates) | — | — | Sain, petit périmètre |
| `crates/starling*` (Rust) | — | — | tests de cycle de vie (`starling-core/tests`) | — | — | Bien testé |

## 3. Analyse par sous-système

### 3.1 Cœur de comptabilité (`accounting/`, `history/`, `inquirer.py`)

**Forces**
- Séparation claire : `accountant.py` orchestre, `pnl.py` calcule, `cost_basis/` implémente les méthodes,
  `export/` produit les rapports.
- Typage monétaire via `fval.py` (précision fixe), plutôt que des flottants.
- Tests dédiés (cost basis, PnL, PnL debug, import debug).

**Faiblesses**
- Dépendances montantes : `accounting/debugimporter/json.py` importe `api.v1.schemas`
  (`ModifiableSettingsSchema`), ce qui fait dépendre le moteur comptable de la couche HTTP.
- `accounting/pot.py` contient un import cyclique résolu localement (TODO « figure out a better way »).
- Une seule instance de compte (« single accounting pot ») est supportée ; la TODO `accountant.py:59`
  indique la généralisation prévue.
- `cost_basis/prefork.py` : TODO liée à l'issue #1610, à traiter avec les méthodes de coût.

**Recommandation** : déplacer `ModifiableSettingsSchema` (ou ses types) dans un module neutre,
puis supprimer l'import cyclique de `pot.py`.

### 3.2 API (`api/`)

**Forces**
- Séparation REST / services : `api/rest.py` délègue à `api/services/*` (20 références).
- Validation déclarative (marshmallow) et versionnage via `api/v1/`.
- Couverture de test importante (84 fichiers de tests API).
- Serveur ASGI optionnel (`asgi.py`) et WebSocket natif, sans changement côté frontend.

**Faiblesses**
- **Monolithes** : `api/v1/schemas.py` (5 534 lignes), `api/rest.py` (5 181), `api/v1/resources.py` (4 286).
  Ces fichiers concentrent une grande part des évolutions et des conflits de fusion.
- 347 occurrences de `Any` / `type: ignore`, surtout dans les schémas et les helpers de réponse.
- Authentification répartie : `api/session_token.py` (HMAC, `sid`), `session_store.py`
  (`session.db` lu par colibri), contrôle d'hôte côté MCP. La logique est bien documentée mais
  dispersée sur trois composants (core, colibri, starling-proxy).
- Couplage inverse : les types WebSocket (`api/websockets/typedefs.py`) et les types de filtrage
  (`api/v1/types.py`) sont importés par `chain/`, `db/`, `data_handler.py`, `banks/`, `assets/`.
  Une vingtaine de modules hors `api/` dépendent donc de la couche HTTP.

**Recommandation** : extraire les types partagés (`WSMessageType`, `ProgressUpdateSubType`,
`IncludeExcludeFilterData`, `HistoryEventsStep`) dans un module `rotkehlchen/types_ws.py` ou
`rotkehlchen/api_types/` sans dépendance HTTP ; découper `schemas.py` par domaine
(`schemas/balances.py`, `schemas/history.py`…).

### 3.3 Chaînes blockchain (`chain/`)

**Forces**
- Architecture à couches : `chain/manager.py` (abstractions), `chain/evm/` (logique EVM commune),
  `chain/ethereum/` (spécificités L1), chaînes dédiées. Ajouter une chaîne EVM réutilise `evm/`.
- Décodage par règles (`EventDecoderFunction`, `input_data_rules`, `post_processing_rules`),
  avec détection de doublons de règles (`ValueError` sur conflit).
- Tests par chaîne et par module (`test_arbitrum_one_inquirer.py`, `test_optimism_transactions.py`…),
  tests VCR (cassettes) pour rejouer les appels réseau.

**Faiblesses**
- **Volume** : 103 k lignes, dont ~45 k dans `chain/evm/` et ~26 k dans `chain/ethereum/`.
  Le risque principal est la duplication entre chaînes et protocoles DeFi.
- **Lignes longues** : ~3 000 suppressions `noqa: E501`. Le linter ignore les lignes longues, ce qui
  réduit la lisibilité des règles de décodage.
- Dépendance vers `api/websockets/typedefs.py` dans 9 modules `chain/` (progression des tâches).
  Les événements de progression sont une préoccupation transverse, pas HTTP.
- 29 marqueurs TODO, dont plusieurs sur la gestion des jetons et des événements internes.

**Recommandation** : sortir les types de progression dans un module partagé (voir 3.2), puis
documenter le contrat d'ajout d'une chaîne ou d'un décodeur (modèle à suivre, tests attendus).

### 3.4 Base de données (`db/`, `globaldb/`)

**Forces**
- Schéma versionné : 75 `CREATE TABLE` dans `db/schema.py`, migrations par version
  (`db/upgrades/`, `db/upgrade_manager.py`), vérifications d'intégrité (`db/checks.py`).
- Base globale séparée (`globaldb/`) partagée entre utilisateurs, avec son propre système de migrations.
- Pilote SQLite / SQLCipher isolé (`db/drivers/sqlite.py`) ; mode de planification `THREADING`
  (migration gevent).
- Tests de migration de données (`tests/data_migrations`, 7 fichiers).

**Faiblesses**
- `db/dbhandler.py` (5 079 lignes) et `db/filtering.py` (3 030) sont très volumineux.
  `filtering.py` est le point le plus complexe : TODO « Quite hacky » (ligne 449).
- `db/history_events.py` (2 817 lignes) et `globaldb/handler.py` (2 854) : même problème de taille.
- 751 `noqa` dans `db/` : l'essentiel vient du `E501` sur les requêtes SQL.
- Dépendance vers `api/v1/types.py` dans `db/filtering.py`, `db/history_events.py`, `db/eth2.py`.
- Code de contournement dans `db/updates.py` (« always force updating ») à revoir.

**Recommandation** : découper `dbhandler.py` par domaine (`db/settings.py` et `db/addressbook.py`
existent déjà, à étendre) ; déplacer les types de filtre hors de `api/`.

### 3.5 Exchanges et sources externes (`exchanges/`, `externalapis/`)

**Forces**
- Interface commune (`exchanges/exchange.py`) et gestionnaire (`exchanges/manager.py`).
- Bon ratio tests / fichiers pour les exchanges (25 fichiers de tests pour 28 modules).
- Clients HTTP unifiés (`externalapis/interface.py`), avec réessais et limitation de débit
  (`tests/unit/test_rate_limiter.py`).

**Faiblesses**
- Fichiers par exchange très volumineux : `binance.py` (1 774 lignes), `kraken.py` (1 684).
- Les intégrations dépendent de formats d'API qui changent : sans tests d'enregistrement récents,
  les régressions sont probables. Les cassettes VCR existent mais ne couvrent pas tout.

**Recommandation** : découper les exchanges volumineux par type d'endpoint (soldes, trades, dépôts)
ou par parseur de réponse.

### 3.6 Tâches de fond et concurrence (`tasks/`, `concurrency/`)

**Forces**
- Migration gevent → asyncio/threads bien tracée (`docs/designs/gevent_to_asyncio.md`, statuts « done »).
- Module `concurrency/` petit (363 lignes), API `Task` unique, annulation coopérative
  (`CancellationToken`, `checkpoint`), tests d'annulation (`test_cancellation.py`).
- gevent n'est plus importé dans le code source (vérifié par recherche des `import gevent`) et la
  règle ruff `TID251` l'interdit.
- Zéro TODO dans `tasks/` et `concurrency/`.

**Faiblesses**
- **Documentation de code périmée** : `concurrency/__init__.py`, `concurrency/cancellation.py` et
  `api/asgi.py` décrivent encore un « monkeypatching actif », alors que le design doc indique que la
  phase 6 (suppression du monkeypatching) est terminée. À corriger.
- `tasks/manager.py` (1 284 lignes) : une méthode `_maybe_*` par type de tâche, répétitif.
  Une table de planification déclarative réduirait la duplication.
- Les threads ne peuvent pas être tués ; l'arrêt dépend de checkpoints. Un tâche longue sans
  checkpoint retarde l'arrêt (choix documenté, mais à surveiller).

**Recommandation** : mettre à jour les docstrings ; envisager une table de planification.

### 3.7 MCP (`mcp/`) et supervision (`crates/starling*`)

**Forces**
- MCP isolé derrière le proxy : `/mcp` est routé par `starling-proxy` vers son propre port,
  avec signature à clé dérivée (`aud: mcp`) empêchant qu'un jeton MCP valide un cookie navigateur.
- Starling supprime l'en-tête interne des requêtes proxifiées (défense en profondeur).
- Tests de cycle de vie POSIX et Windows (`starling-core/tests`).

**Faiblesses**
- Le serveur MCP dépend du processus Python ; un redémarrage de core coupe les outils.
- Deux chemins de configuration (CLI et `ServiceLayout`) à garder synchronisés (`starling-core/src/config.rs`).

### 3.8 Colibri (`colibri/`, Rust)

**Forces**
- Périmètre restreint et clair : icônes, collections et mappings d'actifs, recherche d'actifs,
  prix via oracle, déverrouillage de la base utilisateur, configuration et santé.
- 303 tests Rust (colibri et crates), gestion d'erreurs typée (`database/errors.rs`).
- Dépendances épinglées exactement dans le workspace.

**Faiblesses**
- **Dualité de base de données** : colibri ouvre la base utilisateur (`database/user_db.rs`) et la base
  globale, tandis que Python conserve son propre `DBHandler`. Les schémas et les migrations doivent donc
  rester cohérents entre Python et Rust. Ce risque n'est pas documenté dans un seul endroit.
- Partage de la responsabilité des icônes (`icons.py` : « Partly moved to colibri ») : un état
  intermédiaire coexiste.

**Recommandation** : documenter explicitement quelles tables sont écrites par quel processus, et
ajouter un test de compatibilité de schéma entre Python et Rust.

### 3.9 Frontend (`frontend/app`)

**Forces**
- Structure claire : `pages/` (routes), `modules/` (domaines), `modules/core/` (socle transverse : API,
  tâches, notifications, tables, formulaires).
- Couverture de test large : 1 222 fichiers `*.spec.ts`, tests e2e et tests de contrat.
- Client API centralisé avec file de requêtes, réessais et annulation (`modules/core/api/`).

**Faiblesses**
- 93 occurrences de `any` / `as any` hors tests et 28 fichiers avec `eslint-disable`.
- Taille : ~117 000 lignes de TypeScript hors tests, dont des stores volumineux.
- Le typecheck (`vue-tsc --build`) demande plus de 2 Go de mémoire (voir section 4).

## 4. Exécution des tests dans cette session

| Périmètre | Résultat | Commentaire |
|-----------|----------|-------------|
| Backend Python (`pytest`) | **Non exécuté** | Le projet exige Python 3.14 (`requires-python = "==3.14.*"`). Le téléchargement de l'interpréteur depuis GitHub est bloqué dans cette session (erreur de certificat TLS). |
| Frontend unitaire (`vitest run`, `frontend/app`) | **1 243 fichiers réussis, 13 197 tests réussis, 1 todo, 1 fichier en échec** | Node 24 et pnpm 12 installés via le registre npm ; installation avec `--ignore-scripts` (le post-install d'Electron télécharge depuis electronjs.org, bloqué). Le seul échec, `electron/main/create-protocol.spec.ts`, est dû au binaire Electron absent (« Electron failed to install correctly »), donc environnemental et non lié au code. Durée : ~9,6 min sur 2 cœurs. |
| Frontend typecheck (`vue-tsc --build`) | **Non concluant** | Le processus Node est arrêté pour dépassement de mémoire (tas à 2,6 Go, RAM totale de 3 Go). Il faut une machine plus puissante ou un typecheck par projet. |
| Rust (`cargo test`) | **Non exécuté** | `cargo` absent de l'environnement. |

Lecture : le frontend est en bon état sur le plan des tests unitaires (aucun échec fonctionnel
observé). Le backend et le code Rust n'ont pas pu être vérifiés dans cette session.

## 5. Priorités recommandées

| Priorité | Action | Sous-système | Effort |
|----------|--------|--------------|--------|
| 1 | Supprimer les dépendances `chain/`, `db/`, `accounting/` vers `api/` (types partagés déplacés) | api / chain / db | Moyen |
| 2 | Corriger les docstrings qui décrivent un monkeypatching encore actif | concurrency, api/asgi | Faible |
| 3 | Découper `api/v1/schemas.py`, `api/rest.py`, `db/dbhandler.py` par domaine | api / db | Élevé |
| 4 | Lever la dérogation `E501` sur les règles de décodage et requêtes SQL | chain / db | Moyen |
| 5 | Test de compatibilité de schéma Python ↔ colibri | db / colibri | Moyen |
| 6 | Réduire les `any` du frontend et des schémas API | frontend / api | Élevé |
| 7 | Table de planification déclarative pour `tasks/manager.py` | tasks | Moyen |

## 6. Limites

- Les métriques sont des indicateurs de surface (taille, annotations, suppressions) ; elles ne mesurent
  ni la complexité cyclomatique ni la couverture de lignes.
- Les tests backend n'ont pas pu être exécutés dans cette session ; leur absence de résultat ne
  signifie pas qu'ils échouent.
- Les conclusions sur le couplage proviennent de l'analyse des imports statiques.

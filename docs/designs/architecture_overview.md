# Architecture de rotki (vue d'ensemble)

Document de référence décrivant la structure du dépôt `rotki/rotki` au commit
`a0fc8a5` (branche `develop`, version `1.44.1`). Il est produit à partir de
l'exploration du code source ; les chemins cités sont relatifs à la racine du dépôt.

## 1. Vue d'ensemble

rotki est un gestionnaire de portefeuille, de comptabilité et de fiscalité crypto,
auto-hébergé et orienté vie privée. Les données restent sur la machine de l'utilisateur
(base SQLCipher chiffrée). L'application combine :

| Couche | Technologie | Emplacement |
|--------|-------------|-------------|
| Backend métier et API REST | Python (Flask, webargs, marshmallow) | `rotkehlchen/` |
| Serveur ASGI / WebSocket (option) | uvicorn + pont WSGI | `rotkehlchen/api/asgi.py` |
| Frontend | Vue 3 / TypeScript, Electron ou navigateur | `frontend/app/` |
| Service Rust de performance | Rust, axum, reqwest | `colibri/` |
| Superviseur de processus | Rust | `crates/starling*` |
| Proxy de développement | TypeScript | `frontend/dev-proxy/` |

## 2. Vue des processus à l'exécution

```
Electron (main + preload)  ─┐
  ou navigateur (dev:web)   │
                            ▼
              starling (superviseur, NDJSON sur stdio)     [crates/starling]
                 │                         │
                 ▼                         ▼
        rotki-core (Python)         colibri (Rust)        [colibri/]
        API REST + WS               API locale, base globale,
        port REST / WS              cache d'icônes, prix, blockchain
```

- **starling** (`crates/starling/README.md`) démarre les services dans l'ordre de dépendance,
  attend leur disponibilité (`/api/1/ping` pour rotki-core, port pour colibri), les supervise
  et les arrête dans l'ordre inverse. Il verrouille le répertoire de données pour empêcher
  deux instances. Il est construit sur `starling-core` (moteur de cycle de vie, sans dépendance
  à la plateforme) et `starling-proxy`.
- Le processus Python est lancé par `rotkehlchen/server.py` (`RotkehlchenServer`), qui
  instancie `Rotkehlchen` puis `APIServer`.
- Les ports REST et WebSocket sont configurables (`--rest-api-port`, `--websockets-api-port`,
  voir `rotkehlchen/args.py`).

## 3. Backend Python (`rotkehlchen/`)

### 3.1 Cœur : la classe `Rotkehlchen`

`rotkehlchen/rotkehlchen.py` (classe `Rotkehlchen`, ~1 700 lignes) est le conteneur central.
Il assemble, dans son `__init__`, les composants partagés :

- `MessagesAggregator` (messages et avertissements vers l'utilisateur) et `RotkiNotifier`
  (diffusion temps réel) ;
- `TaskSupervisor` (exécution des tâches de fond) et `TaskManager` (planification) ;
- `ExchangeManager`, `BankManager` ;
- `DataHandler` (import / export de données) ;
- clients de données de prix et API externes : `Cryptocompare`, `Coingecko`, `Defillama`,
  `Alchemy`, `Moralis`, `Birdeye`, `Kraken` ;
- `IconManager`, `AssetsUpdater`, `DataMigrationManager` ;
- `threading.Event` de fermeture (`shutdown_event`).

### 3.2 API (`rotkehlchen/api/`)

| Module | Rôle |
|--------|------|
| `api/server.py` | `APIServer` et `RestAPI` : câblage Flask, CORS, sessions, WebSocket |
| `api/rest.py` (~5 200 lignes) | Ressources REST (point d'entrée fonctionnel de l'API) |
| `api/v1/` | Ressources, schémas marshmallow, champs et parseurs (`resources.py`, `schemas.py`, `wallet_resources.py`…) |
| `api/services/` | Logique métier appelée par les ressources : `accounting`, `balances`, `exchanges`, `history`, `history_events`, `transactions`, `user_data`, `settings`, `external_services`… |
| `api/rest_helpers/` | Helpers de réponse (téléchargements, historique, wrap) |
| `api/websockets/` | `RotkiNotifier` et types de messages |
| `api/session_store.py`, `session_token.py` | Sessions et jetons |
| `api/asgi.py` | Serveur uvicorn unique (REST via pont WSGI + WebSocket natif sur `/ws`) |

Le sous-système `rotkehlchen/mcp/` expose en outre des outils MCP (serveur `server.py`,
enregistrement d'outils dans `tools/`, garde premium, analytics d'usage), avec un contrôle
d'hôte loopback.

### 3.3 Persistance

- **Base utilisateur** : `rotkehlchen/db/` — `dbhandler.py` (`DBHandler`, ~5 000 lignes),
  `schema.py` (75 `CREATE TABLE`), `schema_transient.py`, `history_events.py`,
  `accounting_rules.py`, `addressbook.py`, `queried_addresses.py`, `snapshots.py`, etc.
  Les pilotes sont dans `db/drivers/` (`sqlite.py`). Les migrations de schéma sont dans
  `db/upgrades/` et le gestionnaire `db/upgrade_manager.py`.
- **Base globale (partagée, non chiffrée par utilisateur)** : `rotkehlchen/globaldb/` —
  catalogue d'actifs, mappings, cache, migrations (`globaldb/migrations`, `globaldb/upgrades`),
  gestion des mises à jour d'actifs (`asset_updates/`).
- **Données d'évaluation** : `rotkehlchen/data/`.

### 3.4 Blockchain (`rotkehlchen/chain/`)

- `chain/manager.py` : abstractions `ChainManager`, `ChainManagerWithTransactions`,
  `ChainManagerWithNodesMixin`.
- `chain/aggregator.py` : agrège les soldes et transactions sur toutes les chaînes.
- `chain/ethereum/` (couche spécifique à Ethereum, avec `defi/`, `decoding/`, `modules/`,
  `oracles/`, `interfaces/`, `airdrops.py`, `accountant.py`) et `chain/evm/` (logique commune
  EVM : `node_inquirer.py`, `nodes.py`, `decoding/`, `accounting/`, `l2_with_l1_fees/`,
  `proxies_inquirer.py`).
- Chaînes dédiées : `arbitrum_one/`, `avalanche/`, `base/`, `optimism/`, `polygon_pos/`,
  `gnosis/`, `scroll/`, `sonic/`, `monad/`, `ink/`, `binance_sc/`, `hyperliquid/`,
  `robinhood/`, `solana/`, `substrate/`, `bitcoin/`, `zksync_lite/`.
- Les décodeurs de transactions (`decoding/`) transforment les transactions brutes en
  événements d'historique (`db/history_events.py`).

### 3.5 Exchanges et sources externes

- `rotkehlchen/exchanges/` : une classe par plateforme (Binance, Kraken, Coinbase, OKX,
  Bybit, Gemini, Bitstamp, etc., ~30 fichiers) orchestrées par `exchanges/manager.py`
  (`ExchangeManager`). `exchanges/exchange.py` définit l'interface commune.
- `rotkehlchen/externalapis/` : clients HTTP vers les services tiers (Etherscan et dérivés,
  CoinGecko, CryptoCompare, DefiLlama, Alchemy, Moralis, Helius, Jupiter, Hyperliquid, Kraken,
  GitHub, Google Calendar…), tous basés sur `externalapis/interface.py`.
- `rotkehlchen/banks/` : intégrations bancaires.
- `rotkehlchen/data_import/` : import de CSV et de fichiers exchanges.

### 3.6 Historique, prix et comptabilité

- `rotkehlchen/history/` : `manager.py` (`EventsHistorian`), `price.py` et `price_oracles/`
  (sources de prix), `processing.py`, `deserialization.py`, `events/`, `data_issues/`.
- `rotkehlchen/oracles/` : oracles de prix et de conversion.
- `rotkehlchen/inquirer.py` : `Inquirer`, point d'accès aux prix courants.
- `rotkehlchen/accounting/` : moteur comptable.
  - `accountant.py` (orchestration), `pnl.py` (plus/moins-values), `pot.py` (pools de coûts),
  - `cost_basis/` (méthodes FIFO, LIFO, HIFO, etc.), `rules.py` (règles comptables),
  - `export/` (rapports), `structures/`, `debugimporter/`.
- `rotkehlchen/balances/` : soldes historiques et manuels.

### 3.7 Tâches de fond (`rotkehlchen/tasks/`)

- `tasks/manager.py` (`TaskManager`, ~1 300 lignes) : planifie des tâches périodiques
  (requêtes de transactions EVM, reçus de transactions, historique exchanges, décodage,
  dérivation xpub, requêtes CryptoCompare, traitement des soldes historiques, remédiation des
  anomalies de données, vérification du statut premium, sauvegarde de la base…).
- `tasks/supervisor.py` : exécution, annulation et arrêt propre des tâches.
- `rotkehlchen/concurrency/` : primitives de concurrence (Task, TaskSupervisor basés sur des
  threads, jetons d'annulation).

### 3.8 Modèle de concurrence

Le projet migre de gevent vers asyncio (`docs/designs/gevent_to_asyncio.md`), document de
décision à lire en premier pour comprendre le code de concurrence :

- Une boucle asyncio (thread principal) gère le serveur, les WebSockets et l'ordonnancement.
- La logique métier reste synchrone et s'exécute dans un pool de threads borné.
- Le pilote SQLite fonctionne en mode `THREADING` (plus de monkey-patching gevent).
- Les annulations passent par `CancellationToken` (remplacement de `greenlet.kill`).

### 3.9 Configuration et autres modules

- `args.py` (arguments CLI), `config.py`, `feature_flags.py`, `crypto.py` (chiffrement de la
  base), `logging.py`, `user_messages.py`, `usage_analytics.py`, `icons.py`, `sigil.py`,
  `fval.py` (valeurs à précision fixe), `types.py`, `interfaces.py`.
- `rotkehlchen/premium/` : gestion de l'abonnement premium et synchronisation
  (`premium.py`, `sync.py`).
- `rotkehlchen/errors/`, `rotkehlchen/serialization/` : erreurs typées et sérialisation.

## 4. Frontend (`frontend/`)

Monorepo pnpm (`pnpm-workspace.yaml`) avec trois paquets :

- `frontend/app/` : application Vue 3 / TypeScript, Vite, Pinia (stores), Vue Router,
  Tailwind. Points d'entrée : `src/main.ts`, `src/App.vue`.
- `frontend/common/` : code partagé.
- `frontend/dev-proxy/` : proxy de développement (composants premium locaux, mocks de
  requêtes asynchrones).

Organisation de `frontend/app/src/` :

| Dossier | Contenu |
|---------|---------|
| `pages/` | Pages routées : `accounts`, `balances`, `history`, `reports`, `staking`, `onchain`, `nfts`, `import`, `settings`, `price-manager`, `asset-manager`, etc. |
| `modules/` | Logique par domaine (stores, composables, composants) : `accounts`, `assets`, `balances`, `history`, `reports`, `settings`, `premium`, `task-center`, `wallet`… |
| `modules/core/` | Socle transverse : `api/` (client `rotki-api.ts`, file de requêtes, réessais, transformateurs), `tasks/`, `messaging/`, `notifications/`, `action-center/`, `form/`, `table/` |
| `router/` | Routage (`route-map.d.ts` pour le typage des routes) |
| `plugins/rui/` | Système de composants « rui » (UI) |
| `layouts/` | Mises en page |
| `locales/`, `i18n.ts` | Internationalisation |
| `electron/` | Processus principal Electron, preload, commandes IPC (`ipc-commands.ts`) |

### Flux de données frontend

1. Un store (Pinia) ou un composable appelle le client `rotki-api.ts` (HTTP vers l'API REST).
2. Les requêtes longues sont exécutées comme **tâches** : l'API renvoie un identifiant et le
   résultat arrive ensuite par WebSocket (`/ws`), géré dans `modules/core/tasks/`.
3. Les notifications et messages serveur (`RotkiNotifier`) alimentent le centre d'actions et
   les notifications.

## 5. Service Colibri (`colibri/`, Rust)

Service local complémentaire, démarré par starling :

| Module | Rôle |
|--------|------|
| `src/api/` | Endpoints axum : `assets`, `configuration`, `database`, `globaldb_endpoints/`, `health`, `icons`, `prices`, schémas JSON |
| `src/blockchain/` | Inquisiteurs de nœuds EVM (`node_inquirer.rs`), ABIs embarquées |
| `src/database/` | Accès SQLite de la base utilisateur (`user_db.rs`, `sql_utils.rs`) |
| `src/globaldb/` | Accès à la base globale |
| `src/coingecko.rs`, `icons.rs` | Prix et cache d'icônes |
| `src/session.rs`, `http.rs`, `logging.rs`, `args.rs` | Sessions, client HTTP, logs, CLI |

Colibri est un workspace Cargo avec `crates/starling`, `crates/starling-core` et
`crates/starling-proxy`. Les versions de dépendances sont épinglées exactement
(`Cargo.toml`, `[workspace.dependencies]`).

## 6. Outillage, build et qualité

- `package.py` et `rotkehlchen.spec` (PyInstaller) : packaging de l'application.
- `Dockerfile`, `packaging/` : images Docker et paquets.
- `pyproject.toml` / `uv.lock` : dépendances Python (gérées avec `uv`).
- `Cargo.toml` / `Cargo.lock` : workspace Rust.
- `Makefile` : cibles `lint` (ruff, mypy, pyright, pylint, vérifications de constantes et de
  f-strings de logs), `format`, `test-assets`, `create-cassettes` (enregistrements VCR).
- `tools/` : scripts (`scenarios/`, `bench/`, `profiling/`, `assets_database/`, `pylint/`…).
- `rotkehlchen_mock/` : données de mock. `stubs/` : stubs de typage.
- `docs/` : documentation Sphinx (`api.rst`, `dev_guide.rst`, `websockets.rst`,
  `changelog.rst`) et documents de conception (`docs/designs/`).
- `AGENTS.md` / `CLAUDE.md` : conventions de développement du projet.

## 7. Flux d'une requête typique

Exemple : affichage des soldes.

1. L'utilisateur ouvre la page `balances` ; le store appelle l'API REST.
2. `api/rest.py` valide la requête (`api/v1/schemas.py`) et délègue à `api/services/balances.py`.
3. Le service interroge `chain/aggregator.py` (soldes blockchain), `exchanges/manager.py`
   (soldes exchanges), `balances/manual.py` (soldes manuels) et `inquirer.py` (prix).
4. Les résultats sont convertis et renvoyés ; les tâches longues sont planifiées par
   `tasks/manager.py` et leur résultat est poussé via `/ws`.

## 8. Points d'entrée à lire en premier

| Sujet | Fichier |
|-------|---------|
| Démarrage backend | `rotkehlchen/server.py`, `rotkehlchen/__main__.py` |
| Conteneur principal | `rotkehlchen/rotkehlchen.py` |
| Routes REST | `rotkehlchen/api/rest.py`, `rotkehlchen/api/v1/` |
| Schéma base | `rotkehlchen/db/schema.py`, `rotkehlchen/db/dbhandler.py` |
| Concurrence | `docs/designs/gevent_to_asyncio.md`, `rotkehlchen/tasks/manager.py` |
| Supervision des processus | `crates/starling/README.md` |
| Client API frontend | `frontend/app/src/modules/core/api/rotki-api.ts` |
| Conventions de code | `AGENTS.md` |

## 9. Limites de ce document

- Document rédigé par lecture statique de la structure et de quelques fichiers clés ; il ne
  remplace pas la lecture du code pour les détails d'implémentation.
- Les compteurs (lignes, nombre de tables) sont des ordres de grandeur au commit `a0fc8a5`.
- L'outil « REA (Reverse Engineer Anything) » n'était pas disponible dans cette session ;
  l'analyse a été faite directement sur le dépôt.

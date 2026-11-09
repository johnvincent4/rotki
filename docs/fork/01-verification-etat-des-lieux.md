# rotki — Vérification factuelle du document de synthèse

**Objet** : confronter le document « Vision ensemble assistant B » (synthèse rédigée *sans* accès au dépôt) au code réel de `rotki/rotki`, puis en tirer les conséquences pour un fork de **pré-comptabilité française** (sociétés à l'IS détentrices de crypto-actifs).

**Commit analysé** : `9b4b0db9a02719678d8f1cdc93c019c82b36f1fc`, branche `develop`, 2026-11-09.
**Version applicative** : `1.44.2` (fallback SCM `pyproject.toml`, `Cargo.toml` workspace).

---

## 0. Méthode et légende

Tout ce qui suit a été **lu dans le code présent dans ce checkout**, pas déduit. Chaque affirmation est étiquetée :

| Marqueur | Signification |
|---|---|
| ✅ | Vérifié dans le code (chemin de fichier + ligne fournis) |
| ❌ | **Contredit** par le code : le document se trompe |
| 🟡 | Partiellement vrai / vrai mais avec une nuance importante |
| 🆕 | Absent du document : élément nouveau non couvert |

Volumétrie réelle mesurée (remplace les « ⛔ non déductible » du document) :

| Mesure | Valeur | Commande |
|---|---|---|
| Fichiers Python hors tests | **2 050** | `find rotkehlchen -name '*.py' -not -path '*/tests/*' \| wc -l` |
| Lignes Python hors tests | **258 681** | idem + `xargs cat \| wc -l` |
| Fichiers Python de test | 491 | `find rotkehlchen/tests -name 'test_*.py' \| wc -l` |
| Lignes de test | 267 149 | — |
| Frontend (`app/src` + `common`) | 4 025 fichiers / **483 022** lignes | — |
| Routes REST déclarées | **191** | `grep -c "^\s*('/" rotkehlchen/api/server.py` |
| Classes de ressources | **182** | `grep -c "^class .*Resource" rotkehlchen/api/v1/resources.py` |
| Méthodes de `RestAPI` | **350** | `grep -c "    def " rotkehlchen/api/rest.py` |
| Endpoints protégés par login | 232 | `grep -c "@require_loggedin_user"` |
| Décodeurs de protocoles EVM | **63** dossiers | `ls -d rotkehlchen/chain/evm/decoding/*/` |
| Exchanges supportés | **22** modules | `ls rotkehlchen/exchanges/*.py` (hors socle commun) |
| Chaînes / écosystèmes | **19** (dont 13 EVM) | `ls -d rotkehlchen/chain/*/` hors `evm`/`mixins`/`decoding` |
| Migrations de schéma user DB | **28** fichiers `v26→v54` + **30** migrations de données | `ls rotkehlchen/db/upgrades/v*.py` |
| Schémas : user DB **v54**, global DB **v19**, transient **v2** | — | `db/settings.py:49`, `globaldb/utils.py:23` |

---

## 1. Corrections majeures au document

### 1.1 ❌ Le backend n'utilise plus gevent — il tourne sur asyncio/uvicorn

C'est l'erreur la plus lourde du document, car elle contamine tout le chapitre « architecture » et le chapitre « risque ».

Le document affirme (ch. 2.2, 3, 4, 11.5, 15.4-R6) : « serveur WSGI gevent », « concurrence coopérative par greenlets », « monkey-patching global », « gevent = dépendance irréversible 🔴 extrême ».

Réalité :

- `gevent` **n'est plus une dépendance** : `pyproject.toml` ne le liste pas ; le serveur est `uvicorn==0.54.0` + `a2wsgi==1.10.10` + `websockets==17.3`.
- Il n'y a plus qu'**un seul serveur** : `rotkehlchen/api/asgi.py` (uvicorn, ASGI) sert le Flask REST via un pont WSGI→ASGI **et** le WebSocket natif sur `/ws`.
- Le monkey-patching a été supprimé : `rotkehlchen/server.py:33-42` explique même que le code règle `sys.setswitchinterval(0.0005)` *parce que* le GIL est de nouveau partagé entre threads préemptifs.
- `rotkehlchen/concurrency/` remplace les greenlets : `tasks.py`, `cancellation.py` (`CancellationToken` + checkpoints), `TaskSupervisor` (thread-backed).
- Driver DB : `rotkehlchen/db/drivers/sqlite.py` (plus de `drivers/gevent.py`).
- La migration est documentée et **terminée** : `docs/designs/gevent_to_asyncio.md` (phase 6 « The flip: remove monkey patching and gevent » = **done**).
- **Aucun import** de `gevent` ne subsiste. Les 44 fichiers non-test qui contiennent encore le mot « gevent » sont des commentaires historiques, des docstrings ou de la documentation de conception.

**Conséquences pour le fork** :
- Le risque « dépendance irréversible à gevent » **disparaît** du registre des risques.
- Le modèle réel est : **une boucle asyncio** (serveur, WS, planification) + **un pool borné de threads** pour toute la logique métier synchrone (dont la comptabilité). C'est plus facile à profiler, à débugger, et compatible avec le Python *free-threaded* — que rotki cible déjà : `requires-python = "==3.14.*"`, `sys._is_gil_enabled()` loggé au démarrage.
- Corollaire : le document parle de « greenlets, budget borné de greenlets » dans `TaskManager` ; la réalité est un `TaskSupervisor` à threads avec annulation coopérative explicite (`checkpoint()`), donc **une comptabilité interruptible proprement** — utile pour un fork qui doit générer des rapports longs.

### 1.2 ❌ Le « God object » et le « RestAPI obèse » ont été partiellement dégraissés

Le document décrit `api/rest.py` comme une façade monolithique unique. Réalité : il existe désormais une **couche services** par domaine — `rotkehlchen/api/services/` : `accounting.py`, `history.py`, `history_events.py`, `balances.py`, `exchanges.py`, `assets.py`, `accounts.py`, `transactions.py`, `settings.py`, `user_data.py`, `banks.py`, `integrations.py`, `locations.py`, `external_services.py`.

`rest.py` reste gros (5 470 lignes, 350 méthodes) mais il **délègue** désormais à ces services ; c'est le point d'extension naturel du fork : **écrire un `FrenchAccountingService` plutôt que modifier `rest.py`**.

Taille réelle : `rotkehlchen.py` **1 762** lignes (et non « god object tentaculaire »), `api/rest.py` 5 470, `db/dbhandler.py` 5 010, `api/v1/resources.py` 4 500. Le vrai point dur est `DBHandler` (5 010 lignes) et son absence d'abstraction, pas l'orchestrateur.

### 1.3 ❌ `Trade` et `LedgerAction` n'existent plus dans le code

Le document décrit (§2.4 faiblesse n°6, §15.2) une « dette de modélisation en cours de résorption : coexistence de `Trade`, `LedgerAction`, `AssetMovement` legacy et du nouveau modèle `HistoryEvent` ».

Vérifié : **aucune occurrence de `LedgerAction`** hors tests ; pas de table `trades` dans `rotkehlchen/db/schema.py`. L'historique est **exclusivement** en `history_events` (`DB_CREATE_HISTORY_EVENTS`, `schema.py:481`) avec héritage par table :

`HistoryBaseEntryType` (`history/events/structures/base.py:146`) = `HISTORY_EVENT`, `EVM_EVENT`, `ETH_WITHDRAWAL_EVENT`, `ETH_BLOCK_EVENT`, `ETH_DEPOSIT_EVENT`, `ASSET_MOVEMENT_EVENT`, `SWAP_EVENT`, `EVM_SWAP_EVENT`, `SOLANA_EVENT`, `SOLANA_SWAP_EVENT`, `BITCOIN_EVENT`, `BANK_TRANSACTION_EVENT`.

**Conséquence** : le fork part d'un modèle **unique et stabilisé** — très bonne nouvelle pour la pré-comptabilité, qui a besoin d'une source de vérité unique.

### 1.4 ❌ `event_identifier` s'appelle `group_identifier` ; la contrainte d'unicité est `(group_identifier, sequence_index)`

Le document cite partout `event_identifier` comme clé de regroupement. Le code utilise `group_identifier` (`schema.py:484`), avec `UNIQUE(group_identifier, sequence_index)` (`schema.py:497`). `event_identifier` n'existe que dans des tables de liaison (`bitcoin_events_addresses`, `eth_staking_events_info`).

Détail important : **il n'y a plus de colonne `counterparty` dans `history_events`** — elle est dans la table d'extension `chain_events_info` (`schema.py:502-507`, colonnes `tx_ref BLOB`, `counterparty`, `address`). Le document la place dans `EvmEvent` de façon générique : c'est vrai au niveau objet Python, faux au niveau schéma.

### 1.5 🟡 `global.db` : non chiffrée — confirmé ; mais trois connexions, pas trois bases « documentées »

Vérifié : `GlobalDBHandler` ouvre `global.db` **sans** `unlock_database()` (`globaldb/utils.py:89` `DBConnection(...)` puis `read_ctx`), contrairement à la base utilisateur (`db/dbhandler.py:614` `unlock_database(...)`). `global.db` est en clair, copiée depuis `rotkehlchen/data/global.db` au premier lancement.

Nuances à ajouter :
- Il existe en réalité **quatre** types de connexions : `DBConnectionType` = `USER`, `TRANSIENT`, `GLOBAL`, `PACKAGED_GLOBAL` (`drivers/sqlite.py:285`) — la dernière étant le `global.db` embarqué en lecture seule, pour comparer/restaurer.
- La base **transient est aussi chiffrée** : `_connect(conn_attribute='conn_transient')` appelle le même `unlock_database()` (`dbhandler.py:600`, `614`). Le document la présente comme « non chiffrée » : **faux**.
- Nom exact : `rotkehlchen_transient.db` (`dbhandler.py:229`), pas `*_transient.db`.

**Conséquence fork** : rien de sensible ne fuit via `global.db` ; mais attention, un fork « comptable » sera tenté d'y stocker des référentiels (plan de comptes, journaux, taux de TVA). **`global.db` est partagée entre tous les comptes utilisateurs et non chiffrée** — le plan de comptes doit aller en base **utilisateur**, pas en global.

### 1.6 ❌ `include_gas_costs` est un réglage mort (orphelin)

Le document présente `include_gas_costs` comme une règle métier active (« Frais de gas déductibles ou non »). Vérifié par grep exhaustif : le réglage est **déclaré, sérialisé, stocké dans le rapport, exposé dans l'UI**… mais **aucun code comptable ne le lit**. Les seules occurrences backend sont `db/settings.py`, `db/reports.py:119`, `accounting/export/{csv,report}.py`, `api/v1/schemas.py`. Aucune dans `accounting/pot.py`, `accounting/history_base_entries.py` ou `accounting/cost_basis/`.

Aujourd'hui le gas est traité comme un événement `SPEND/FEE` de counterparty `CPT_GAS` (`history/events/structures/evm_event.py:105`), piloté par les **règles comptables**, plus par le réglage.

**Conséquence fork** : ne pas s'appuyer sur ce réglage. La déductibilité des frais (gas, frais d'exchange) passera par `count_entire_amount_spend` / `count_cost_basis_pnl` dans `accounting_rules` — mécanisme, lui, bien vivant.

### 1.7 ❌ La faille #13393 est corrigée (et le document la présente comme non vérifiée, avec un « risque élevé »)

Le document en fait le risque **R2**, avec « statut de correction inconnu » et « contredit la promesse centrale du produit ».

Vérifié :
- L'issue GitHub **#13393 est fermée** depuis le **2026-10-22** (`gh api repos/rotki/rotki/issues/13393`).
- Le code utilise désormais **Argon2id** (params `time_cost=3`, `memory_kib=64*1024`, `lanes=4`) + dérivation HKDF à séparation de domaine, chiffrement **AES-256-GCM par chunks** avec AAD authentifiant l'en-tête, le numéro de chunk et le flag de fin : `rotkehlchen/crypto.py:36-144`, appelé par `data_handler.py:352` (`compress_and_encrypt_db`). Format `RKPBAK01`, rétro-compatibilité legacy gérée par `is_backup_format()`.

**Conséquence** : le risque R2 doit être **supprimé** du registre. Le chiffrement de la base locale reste du SQLCipher avec `KDF_ITER = 64000` (`db/constants.py:7`).

### 1.8 ❌ L'API locale n'est plus « protégée seulement par le loopback »

Le document (§5.3 point 7, §8.1, §11.4) affirme : « pas de token par requête », « surface d'attaque = binding loopback », et demande d'auditer.

Vérifié — il existe un mécanisme complet, opt-in via `ROTKI_SESSION_KEY` :
- `rotkehlchen/api/session_token.py` : cookie `rotki_session`, HttpOnly, HMAC-SHA256 sur payload `{u, iat, exp, sid}`, TTL 7 jours, **`sid` = session active unique** ⇒ un nouveau login révoque tous les cookies antérieurs (ferme l'issue #3156).
- `rotkehlchen/api/session_store.py` : `session.db` (à côté de `global.db`), session unique active, tokens MCP à séparation de domaine cryptographique (`aud: mcp`).
- `rotkehlchen/api/server.py:590-600` : **deny-by-default** par verbe HTTP dès qu'une clé de session est configurée ; allowlist limitée à `POST /users/<name>/authenticate`, création de compte, `/ping`, etc.
- `rotkehlchen/api/server.py:681-740` : validation du cookie sur chaque requête protégée + vérification `sid == active`.
- CORS : `--api-cors` par défaut `http://localhost:*/*`, compilé en regex **ancrée** pour éviter le contournement `http://localhost.evil.com` (`server.py:523-556`).

**Conséquence fork** : le mode « déployé sur un serveur » (Docker) est **déjà** sécurisé correctement ; le document sous-estimait largement ce point.

### 1.9 🆕 Le produit n'est plus « Python + Vue » : il y a deux services Rust

Le document ignore totalement l'architecture actuelle côté processus. Réalité :

| Composant | Rôle | Preuve |
|---|---|---|
| `rotki-core` (Python) | API REST + moteur comptable + blockchains | `rotkehlchen/` |
| `colibri` (Rust, `colibri/`) | Service axum : prix, icônes, globalDB, cache — sa propre API HTTP, sa propre connexion SQLCipher (`rusqlite` bundled-sqlcipher) | `colibri/src/{api.rs,globaldb,prices.rs,icons.rs}` |
| `starling` (Rust, `crates/starling*`) | **Superviseur de processus** : graphe de services, gating de readiness (`/api/1/ping` pour core, port pour colibri), terminaison de l'arbre de processus, canal de contrôle NDJSON sur stdio | `crates/starling/README.md` |
| Electron | N'a plus qu'**un** enfant à superviser : `starling` | idem |

C'est une **décomposition en 3 processus + frontend**, pas un « monolithe Python ». Impact direct pour le fork : toute nouvelle fonctionnalité comptable qui a besoin de données de `colibri` (prix, référentiel) passe par une API HTTP interne, et le démarrage doit respecter le graphe de `starling`.

### 1.10 🆕 Un serveur MCP est intégré

`rotkehlchen/mcp/` : serveur MCP (Model Context Protocol) complet — `server.py`, `auth.py` (tokens à audience `mcp`), `registry.py` (découverte d'outils), `tools/{analytics,assets,info,prices,taxonomy}.py`, avec modes de confidentialité (`mcp_privacy_mode`) et `validate_loopback_host()` forçant le loopback.

Pour un fork « pré-comptabilité », c'est un **canal d'exposition idéal** vers un agent/LLM ou vers un logiciel tiers : écrire un outil MCP `french_ledger_export` coûte bien moins cher qu'un endpoint REST + écran UI.

### 1.11 🆕 Les « locations » sont devenues un arbre en base, extensible par l'utilisateur

Le document décrit `Location` comme un **enum figé** (Kraken, Ethereum, Binance…). Faux depuis la v1.45 (user DB v54) :

- Table `locations(identifier, name, parent_identifier, is_builtin, is_active, icon, image)` — auto-référencée, profondeur arbitraire (`schema.py:3-16`).
- `location_aliases(alias, location_identifier)` : résolution des libellés de CSV importés (`schema.py:20-26`).
- API : `GET/POST /locations`, `PATCH/DELETE /locations/<id>`, `/locations/aliases`, `/locations/<id>/usage`, `/locations/<id>/image` (`server.py:423-430`, `resources.py:729-812`).
- Design : `docs/designs/hierarchical_locations.md`, avec la distinction **location** (où) vs **connector** (comment on obtient la donnée).
- Les snapshots suivent : `timed_location_data` référence `locations(identifier)`.

**Conséquence majeure pour le fork** : on peut modéliser des **emplacements de détention utiles à la comptabilité** sans toucher au schéma — par exemple une hiérarchie `Total > Trésorerie > Binance > Binance-Spot`, ou `Total > Immobilisations > Ledger`, ou des sous-comptes par entité. C'est un levier de personnalisation **gratuit** que le document jugeait impossible (il décrivait un enum).

### 1.12 🆕 Il existe une notion d'« issues de données » (data issues) et de conflits non résolus

`rotkehlchen/history/data_issues/` + `rotkehlchen/tasks/data_issues.py` + table `bucket_processing_stops` (`schema.py:988`), `rotkehlchen/db/unresolved_conflicts.py` + `unresolved_remote_conflicts` (`schema.py:657`) : rotki détecte et **expose à l'utilisateur** les problèmes de données (trous, divergences, conflits de règles comptables après mise à jour distante).

C'est directement réutilisable comme **« piste d'audit »** pour un fork comptable : une écriture comptable n'est fiable que si l'absence d'anomalie est *prouvée*.

### 1.13 🆕 Les comptes bancaires sont intégrés (dont Qonto)

`rotkehlchen/banks/` : `connector.py`, `manager.py`, `fints.py` (+ manifestes bancaires allemands), `qonto.py`, et `BankTransactionEvent` (`history/events/structures/bank_transaction.py`) qui est un `HistoryEvent` de `entry_type = BANK_TRANSACTION_EVENT`.

**Pourquoi c'est décisif pour la France** : une société à l'IS détentrice de crypto a **des flux fiat** (virements vers/depuis l'exchange, paiement des factures, encaissements clients). Un `FEC` ou un journal de trésorerie exige les **deux jambes**. Qonto est un acteur français majeur des PME : le fork peut, sans développement, obtenir les mouvements bancaires réels et **rapprocher** les virements des dépôts/retraits d'exchange (voir §4.3).

### 1.14 🆕 La couche « pretium » : prix manuels de première classe

Le document mentionne les oracles en passant. Réalité, importante pour l'ANC 2026-01 (obligation de documenter la « Value Date/Time » et la source) :
- `price_history(from_asset, to_asset, source_type, timestamp, price)` conserve **la source** de chaque prix (`globaldb/schema.py:263-272`, enum `price_history_source_types` : MANUAL, COINGECKO, CRYPTOCOMPARE, XRATESCOM, MANUAL_CURRENT, DEFILLAMA, UNISWAPV2/V3, ALCHEMY…).
- Prix manuels : `PUT /assets/prices/latest` (`add_manual_latest_price`), `PUT/PATCH/DELETE /assets/prices/historical` (`TimedHistoricalPriceSchema`), `GET /assets/prices/latest/all`.
- La précédence est explicite : « manual price entries always first » (`globaldb/handler.py:131`).

**Conséquence** : la traçabilité de la valorisation exigée par le régulateur est **déjà dans le schéma**. Il ne manque que l'exposition dans l'export.

### 1.15 🆕 Il existe un export CSV « tous les événements » avec valeur fiat

Le document ne cite que le CSV de PnL. Il existe aussi `POST|PUT /history/events/export` (`resources.py:3688-3713`) → `HistoryService.export_history_events` (`api/services/history.py:505`) qui :
- applique le filtre de la vue historique (période, actifs, types…),
- valorise **chaque événement** en devise principale via `serialize_for_csv(fiat_value=...)` (`history/events/structures/base.py:423-456`),
- ajoute `asset_symbol`, `amount`, `fiat_value`, `direction`, `location_path`,
- respecte la limite freemium `HISTORY_EVENTS` (1 000 en gratuit — voir §1.17).

C'est la **matière première naturelle** d'un export pré-comptable : la table `history_events` valorisée.

### 1.16 🟡 Limits freemium : exactes, mais incomplètes dans le document

Vérifié : `FREE_HISTORY_EVENTS_LIMIT = 1000`, `FREE_PNL_EVENTS_LIMIT = 1000`, `FREE_REPORTS_LOOKUP_LIMIT = 20`, `FREE_USER_NOTES_LIMIT = 10` (`constants/limits.py`, `accounting/constants.py:15-16`), `ETH_STAKED = 0` en gratuit (`premium/premium.py:158`).

Nuance absente du document : le plafond PnL **tronque le calcul lui-même**, pas seulement la restitution — `accountant.py:248` : `if not active_premium and count >= FREE_PNL_EVENTS_LIMIT: break`. Un rapport gratuit sur une société avec > 1 000 événements est donc **silencieusement partiel** (le champ `total_actions` du rapport révèle l'écart). Pour un fork destiné à des sociétés, **il faut retirer ces limites** — c'est un point bloquant, pas cosmétique.

Autre nuance : `pdf`/`csv` avec formules Excel — `pnl_csv_with_formulas` est **activé par défaut** (`db/settings.py:66`) : le CSV contient des formules `=SUMIF(...;";";...)` et `=IF(K3="";...)`, avec séparateur `;`. Un logiciel comptable tiers qui ingère ce fichier doit soit parser les formules, soit désactiver l'option.

### 1.17 ❌ Points secondaires erronés

| Affirmation du document | Réalité vérifiée |
|---|---|
| « Flask-RESTful pour les ressources » | Plus de `flask_restful` : `BaseMethodView(MethodView)` maison (`api/v1/common_resources.py:9`) + `webargs`/`marshmallow`. Flask 3.1.3 ✅. |
| « `trades` table / `Trade` legacy » | Absent (cf. §1.3). |
| « documentation VitePress dans `rotki/docs` » | Vrai pour le site, mais le dépôt contient aussi une doc **Sphinx** (`docs/{api,websockets,dev_guide,changelog}.rst`) et des **design docs** `docs/designs/`. |
| « pas de doc d'architecture / ADR » | Partiellement faux : `docs/designs/` contient `gevent_to_asyncio.md` (journal de décision complet), `hierarchical_locations.md`, `perf_and_integration_framework.md` ; `AGENTS.md`/`CLAUDE.md` (1 282 lignes) décrivent l'architecture et les commandes. |
| « énumérations `Location` figées » | Arbre en base, extensible (cf. §1.11). |
| « front i18n complet » | `fr.json` est à **30,4 %** de couverture (1 482 clés sur 4 877), dont les écrans comptables et rapports quasi absents. Fallback `en` (`src/i18n.ts:8`). Voir §4.7. |
| « `llms.txt` publié » | Aucun `llms.txt` dans ce dépôt (c'est côté `rotki/docs`). |

### 1.18 ✅ Ce que le document dit juste (et qui compte)

- Licence **AGPL-3.0** ✅ (`gh api` : `spdx_id: AGPL-3.0`), 4 085 ⭐ / 788 forks, branche par défaut `develop`, dernier push 2026-11-09 ✅.
- **SQLCipher AES-256**, clé dérivée du mot de passe (`rotkehlchen/db/utils.py:486-515`) ✅.
- **Aucun ORM**, SQL brut, `read_ctx()`/`write_ctx()` ✅.
- **Décimaux exacts** : `FVal` sur `Decimal`, précision réglée à `ceil(log10(2**256))` (≈ 78 chiffres), `float` interdit ✅ (`fval.py:1-30`). **Point capital pour la comptabilité.**
- **Settings du rapport figés** : `pnl_report_settings` stocke `profit_currency`, `taxfree_after_period`, `include_crypto2crypto`, `calculate_past_cost_basis`, `include_gas_costs`*, `cost_basis_method`, `eth_staking_taxable_after_withdrawal_enabled`, `include_fees_in_cost_basis` (`db/reports.py:110-121`). Reproductibilité ✅. (* mort, cf. §1.6.)
- **Règles comptables en données** (`accounting_rules` : type, subtype, counterparty, taxable, count_entire_amount_spend, count_cost_basis_pnl, accounting_treatment, is_event_specific) ✅ (`schema.py:623-635`), éditables à la main, **importables/exportables en JSON** (`/accounting/rules/import|export`) — donc **versionnables par le fork**.
- **Dégradation gracieuse** + transparence : `missing_acquisitions`, `missing_prices`, `events_skipped_no_rule`, `is_complete` sur le cost basis, exposés via `/history/actionable_items` (`api/services/history.py:211-227`) ✅.
- **Prix manuels** et cascade d'oracles ✅.
- **Electron durci** : `nodeIntegration:false`, `sandbox:true`, `contextIsolation:true` (`frontend/app/electron/main/window-config.ts:42-44`) ✅.
- **Migrations testées**, `--api-cors` restreint, DB backups avant upgrade ✅.
- Le **pipeline de décodage on-chain** et la conservation des données brutes (`evm_transactions`, `evmtx_receipts`, `evmtx_receipt_logs`) permettant le **re-décodage** ✅ — confirmé, et c'est effectivement un atout de premier ordre pour une piste d'audit.

---

## 2. Ce que fait réellement le moteur comptable (lecture du code)

Comprendre le moteur est indispensable : le fork doit **produire des écritures comptables françaises**, pas un PnL de particuliers. Voici son fonctionnement exact.

### 2.1 Chaîne de traitement

```
HistoryEvent (history_events)
   │  Accountant.process_history(start_ts, end_ts, events)
   ▼
AccountingRulesManager.get_event_settings(event)      → (taxable, count_entire_amount_spend,
   │                                                     count_cost_basis_pnl, accounting_treatment)
   ▼
EventsAccountant.process(event)  (accounting/history_base_entries.py)
   │  ├─ BASIS_TRANSFER  → transfer_basis() : les lots suivent l'actif
   │  ├─ SWAP            → _process_swap() : out + in + fees, prix dérivés
   │  └─ défaut          → add_in_event / add_out_event
   ▼
AccountingPot (accounting/pot.py)  + CostBasisCalculator (accounting/cost_basis/base.py)
   │  PnL = valeur_de_cession − coût_de_revient_des_lots_consommés
   ▼
ProcessedAccountingEvent[]  →  rotkehlchen_transient.db (pnl_events / pnl_reports / _totals / _settings)
   ▼
CSVExporter  →  all_events.csv (dans un zip)   |   API /reports/<id>/data   |   /reports/<id>/export
```

### 2.2 Ce que le moteur calcule, et ce qu'il ne calcule PAS

| Concept nécessaire à la comptabilité française | Présent ? | Détail |
|---|---|---|
| Coût d'acquisition (valeur d'entrée) | ✅ | `AssetAcquisitionEvent.rate` = prix en devise principale à la date |
| Consommation de lots (FIFO/CUMP) | ✅ | `FIFOCostBasisMethod`, `AverageCostBasisMethod` |
| Plus/moins-value de cession | ✅ | `ProcessedAccountingEvent.calculate_pnl()` (`structures/processed_event.py:193-231`) |
| Plus/moins-value **latente** à la clôture | ❌ | Aucune occurrence de `fair_value`, `revaluat`, `impair`, `unrealiz`, `depreciat` dans le backend |
| **Stock final** / inventaire à une date | ❌ | `CostBasisCalculator.get_events(asset).acquisitions_manager.get_acquisitions()` existe en mémoire (`cost_basis/base.py:152`) mais **aucun endpoint ne l'expose** (seuls des tests l'utilisent) |
| **Écritures en partie double** (débit/crédit) | ❌ | Modèle mono-ligne : un `ProcessedAccountingEvent` a un `direction` (`IN`/`OUT`/`NEUTRAL`) et des montants, pas de contrepartie |
| Plan de comptes / journaux | ❌ | Aucun champ `account`, `journal`, `ledger` |
| Exercice comptable / clôture / à-nouveaux | ❌ | Le rapport est une **plage de dates** (`start_ts`/`end_ts`), pas un exercice verrouillé |
| Immutable / séquence d'écritures figées | ❌ | `pnl_reports` est supprimable (`DELETE /reports/<id>`) ; les événements sont éditables (`PUT /history/events`) |
| FEC (18 colonnes, A47 A-1) | ❌ | Rien |
| TVA | ❌ | Rien |
| Rattachement à une **entité** (SIREN, société) | ❌ | Un « user » rotki = un profil local ; pas de notion de personne morale |

### 2.3 Trois « pièges » du moteur à connaître avant de forker

**Piège n°1 — `taxfree_after_period` est actif par défaut.**
`DEFAULT_TAXFREE_AFTER_PERIOD = YEAR_IN_SECONDS` (`db/settings.py:51`), c'est-à-dire la règle **allemande** d'exonération après un an. Le calcul l'applique dans `cost_basis/base.py:200-206` :

```python
at_taxfree_period = acquisition_event.timestamp + settings.taxfree_after_period < timestamp
```

En droit français, **il n'existe aucune exonération par durée de détention** pour une société à l'IS. Ce réglage doit être forcé à `None` dans le fork — sinon un tiers des cessions seront silencieusement classées « free » au lieu de « taxable », et le PnL « taxable » sera faux. C'est le défaut le plus dangereux du moteur pour un usage français.

Symétriquement, `include_crypto2crypto = True` par défaut est **correct** pour une société à l'IS (contrairement au particulier français) : les échanges crypto↔crypto y sont imposables. Bon point à conserver.

**Piège n°2 — le coût de revient est mutualisé par actif, tous emplacements confondus.**
`CostBasisCalculator.reset()` : `self._events: defaultdict[Asset, CostBasisEvents]` (`cost_basis/base.py:594`) ; la clé est l'actif (résolu éventuellement vers l'actif principal d'une « collection »). Il n'y a **pas** de granularité par emplacement/exchange/portefeuille.

Or l'ANC 2026-01 impose la FIFO ou le CUMP **par catégorie homogène** de crypto-actifs, et une société peut vouloir suivre des portefeuilles distincts (trading vs immobilisations). Le fork devra introduire une **clé de regroupement** (asset × bucket) — c'est une modification ciblée mais qui touche le cœur du moteur (`_resolve_bucket_asset`, `CostBasisEvents`).

Bonne nouvelle : `use_asset_collections_in_cost_basis` (défaut `True`) montre exactement le point d'extension à imiter — il redirige un actif vers un « bucket » (`cost_basis/base.py:600-628`). Un fork peut y brancher ses propres buckets (par location, par catégorie ANC).

**Piège n°3 — le prix d'entrée des acquisitions est un prix d'oracle, pas un prix de facture.**
`add_in_event` cherche `get_rate_in_profit_currency(asset, timestamp)`, et en cas d'échec **retombe à `ZERO_PRICE`** (`pot.py:248-257`). Un actif sans prix entre dans le calcul à coût nul ⇒ plus-value maximale. Le rapport signale le manque via `missing_prices`, mais **le calcul est faux, silencieusement**. Pour une comptabilité, ce comportement doit devenir **bloquant** (le fork doit refuser de produire un état).

De plus, la devise principale est par défaut **USD** (`DEFAULT_MAIN_CURRENCY = A_USD`, `db/settings.py:57`). À basculer en EUR, et le taux USD→EUR utilisé pour valoriser rétroactivement doit être documenté (cf. §3.3).

### 2.4 Le CSV de PnL : ce qu'il contient exactement

`CSVExporter` (`accounting/export/csv.py`) produit **un seul fichier** `all_events.csv` dans un `csv.zip`, colonnes (`to_exported_dict`, `processed_event.py:128-166` + CSV) :

`type, notes, location, timestamp, asset_identifier, asset, free_amount, taxable_amount, price, pnl_taxable, cost_basis_taxable, pnl_free, cost_basis_free, direction, location_path`

Plus, si `pnl_csv_have_summary` (désactivé par défaut, `db/settings.py:67`) : un bloc de synthèse par `AccountingEventType` (via `SUMIF`/`SUM` Excel), la version de rotki, puis le rappel des **5** réglages de `ACCOUNTING_SETTINGS` (`include_crypto2crypto`, `taxfree_after_period`, `include_gas_costs`, `calculate_past_cost_basis`, `cost_basis_method`) — alors que la base en fige **8** (`db/reports.py:110-121`, cf. §1.18).

Le fichier `skipped_external_events.csv` **ne fait pas partie de ce zip** : il est produit séparément par le débogage de l'historique (`history/skipped.py:64`, téléchargement via `rest.py:3713`).

Points notables pour l'ingestion par un logiciel tiers :
- `type` = `AccountingEventType` (`TRADE`, `FEE`, `ASSET_MOVEMENT`, `MARGIN_POSITION`, `LOAN`, `PREFORK_ACQUISITION`, `STAKING`, `HISTORY_EVENT`, `TRANSACTION_EVENT`) — **pas** un type comptable français.
- `cost_basis_taxable` / `cost_basis_free` sont des **chaînes lisibles** du type « 0.5 ETH acquired at 2021-03-01 12:00:00 with rate 1500 EUR/ETH », pas des montants exploitables (sauf en mode formules, où ce sont des formules Excel pointant vers d'autres lignes).
- Aucune colonne débit/crédit, compte, journal, ni pièce justificative. Aucun `hash` de transaction, sauf concaténé dans `notes` (URL d'explorer + notes).

**Conclusion** : le CSV de PnL **n'est pas** un format d'export pré-comptable ; c'est un rapport fiscal de particuliers. Le fork devra produire son propre export (voir §4).

---

## 3. Compatibilité avec les exigences comptables françaises

### 3.1 Le cadre à respecter (références)

- **Règlement ANC 2026-01 du 9 janvier 2026**, homologué par arrêté du 12 août 2026, publié au JO le 3 septembre 2026 : refonte de la section 619 du PCG sur les **crypto-actifs** (alignée sur MiCA), nouveau plan de comptes (compte **522** « crypto-actifs et assimilés détenus », 52221 prêtés, 524 empruntés, 513 jetons de monnaie électronique).
  - Application **obligatoire aux exercices ouverts à compter du 1er janvier 2027**, application anticipée possible.
  - Classement selon les **droits et obligations** attachés (immobilisation incorporelle « droits sur un bien » vs compte 522).
  - Évaluation des crypto-actifs du compte 522 à la **valeur vénale à chaque clôture**, variations en comptes transitoires (perte latente à l'actif, gain latent au passif) — art. 619-12.
  - Sorties : **FIFO (PEPS) ou coût moyen pondéré (CUMP)** au choix, appliqué **par catégorie homogène**, mentionné en annexe.
  - Dépréciation **ligne par ligne**, sans compensation entre lignes (art. 619-11/619-12 pour les actifs logés selon les droits ; 214-5/214-6 pour la mécanique de dépréciation).
  - Annexe : nombre et valeur des détentions, **modalités de détermination de la valeur vénale** (plateformes/sources, **horaire de valorisation**), méthode de valorisation retenue.
- **Fiscalité IS** : la connexion fiscalo-comptable (art. 38 CGI) s'applique ; **pas de régime autonome** pour les crypto-actifs des personnes morales à l'IS. Taux 15 % (fraction PME) puis 25 %. Aucune exonération par durée de détention. Les échanges crypto-crypto sont des opérations à comptabiliser.
- **FEC** : art. **A47 A-1 du LPF** (arrêté du 29 juillet 2013) — 18 colonnes dans un ordre strict, un fichier par exercice, numérotation séquentielle continue, ordre chronologique, écritures validées et figées, nommage `SIREN + FEC + date de clôture`.
- **Absence d'exonération de TVA** pour les cessions de crypto-actifs (exonération art. 261 C CGI depuis *Hedqvist*), mais les **commissions** de services sont taxables : à isoler côté fork si l'entité facture.

### 3.2 Ce qui est déjà compatible (à conserver tel quel)

| # | Atout du code | Pourquoi c'est conforme / utile | Preuve |
|---|---|---|---|
| 1 | **Arithmétique décimale exacte** (`FVal`/`Decimal`, précision ≈78 chiffres, `float` impossible) | Aucune erreur d'arrondi binaire : condition *sine qua non* d'une comptabilité. Les montants sont stockés en **TEXT** dans SQLite. | `fval.py:11-30` |
| 2 | **Modèle d'événement canonique unique** + `UNIQUE(group_identifier, sequence_index)` | Source de vérité unique et **idempotente** : re-synchroniser un exchange ne duplique pas l'écriture. Base d'une piste d'audit. | `schema.py:481-498` |
| 3 | **FIFO déjà implémenté** (+ LIFO/HIFO/ACB) | L'ANC 2026-01 exige PEPS **ou** CUMP. `FIFO` est le défaut (`CostBasisMethod.FIFO`). L'ACB existe et sert de base à un CUMP, avec une réserve de modèle à lever (§3.3 lettre S). | `cost_basis/base.py:303,318,333,346` |
| 4 | **Lots d'acquisition matérialisés** (`MatchedAcquisition`, `full_amount`, `rate`, `timestamp`, `index`) | Permet de reconstituer le **coût de revient par lot** : indispensable pour justifier une cession et pour l'annexe. | `cost_basis/base.py:470-505` |
| 5 | **Règles comptables externalisées en base**, éditables, **import/export JSON** | Permet de **versionner** une doctrine comptable française dans le dépôt du fork (plan de règles FR), sans patcher le code. | `schema.py:623-655`, `api/services/accounting.py:33-105` |
| 6 | **Réglages du rapport figés** dans `pnl_report_settings` | Reproductibilité : un état comptable doit pouvoir être rejoué à l'identique. | `db/reports.py:110-121` |
| 7 | **Transparence sur les données manquantes** (`missing_acquisitions`, `missing_prices`, `events_skipped_no_rule`, `is_complete`) | Correspond à l'exigence d'un dossier de travail fiable (et à l'esprit « piste d'audit fiable »). | `accounting/types.py`, `api/services/history.py:211-227` |
| 8 | **Provenance des prix conservée** (`price_history.source_type` + timestamps) | Répond à l'obligation ANC de documenter la source et l'horaire de valorisation. | `globaldb/schema.py:232-272` |
| 9 | **Prix manuels** (courants + historiques, prioritaires) | Permet de figer un cours de clôture validé par l'expert-comptable (valeur vénale du 31/12). | `globaldb/manual_price_oracles.py`, `api/v1/resources.py:2996-3040` |
| 10 | **Emplacements hiérarchiques personnalisables** | Structure de suivi par entité/compte/exchange, sans modifier le schéma. | `schema.py:3-26`, `server.py:423-430` |
| 11 | **Flux bancaires (FinTS, Qonto)** comme `HistoryEvent` | Fournit la **jambe fiat** nécessaire au rapprochement et au journal de trésorerie. | `banks/`, `history/events/structures/bank_transaction.py` |
| 12 | **Conservation des données brutes on-chain + re-décodage** | Si un décodeur est corrigé, l'historique est reconstruit : c'est la « piste d'audit » côté blockchain. | `schema.py:238-332`, `db/evmtx.py` |
| 13 | **`/history/actionable_items`** et **data issues** | Liste actionnable des anomalies à corriger avant de figer un exercice. | `api/services/history.py:211`, `history/data_issues/` |
| 14 | **Import CSV générique + mappeurs d'actifs par exchange** | Ingestion de relevés ; résolution des tickers propriétaires (`XXBT` → `BTC`). | `data_import/`, `location_asset_mappings` |
| 15 | **Sécurité locale (SQLCipher), API déployable (cookie/session), Electron durci** | Conservation des données de l'entité chez elle — argument fort face à un expert-comptable. | cf. §1.8 |
| 16 | **Export « tous les événements » valorisé** | Fournit la table de base (date, actif, quantité, contre-valeur EUR) à partir de laquelle le fork construit ses écritures. | `api/services/history.py:505` |
| 17 | **Devise principale paramétrable, EUR natif** (`A_EUR`) + oracle `XRATESCOM` + conversion EUR/USD gérée | Valorisation en euros sans dépendance à une source unique. | `constants/assets.py:6`, `externalapis/xratescom.py` |
| 18 | **`accounting_treatment = SWAP` et `BASIS_TRANSFER`** | Permet déjà de traiter correctement un échange crypto↔crypto comme une cession + acquisition (report du prix de revient) au lieu d'un simple flux. | `chain/evm/accounting/structures.py:31-34` |

### 3.3 Ce qui doit être adapté (écarts bloquants ou structurants)

| # | Écart | Gravité | Nature du travail |
|---|---|---|---|
| A | **`taxfree_after_period` actif par défaut (règle allemande)** — exonération après 1 an inexistante en France | 🔴 **Bloquant** | Forcer `None` ; interdire le réglage dans l'UI du fork. `settings.py:51`, `cost_basis/base.py:200-206` |
| B | **Pas de partie double** : un événement = une ligne + une direction | 🔴 **Bloquant** | Couche de **traduction** vers un modèle d'écritures équilibrées (débit/crédit), ou modèle d'écritures séparé alimenté par les `ProcessedAccountingEvent`. C'est **le** cœur du fork. |
| C | **Pas de notion d'exercice / clôture / à-nouveaux** ; rapport = plage de dates supprimable | 🔴 **Bloquant** | Ajouter exercice (date début/fin, statut ouvert/verrouillé), figer/signer l'export, empêcher la suppression. |
| D | **Pas de plan de comptes ni de journaux** | 🔴 **Bloquant** | Table de mapping `(event_type, event_subtype, counterparty, categorie) → (compte PCG, journal, sens)` — à stocker en **user DB** (pas `global.db`, cf. §1.5). |
| E | **Pas de FEC** (18 colonnes, ordre strict, numérotation séquentielle) | 🔴 **Bloquant** | Générer un FEC + un export « journal » ; séquence d'écritures immuable. |
| F | **Pas de valorisation à la clôture / dépréciation ligne par ligne** (ANC 619-12, 619-11) | 🔴 **Bloquant** | Nouveau calcul : inventaire à date `T` (quantité × valeur vénale) + comparaison à la valeur nette comptable, **par ligne**, sans compensation. Réutiliser `HistoricalBalancesManager` et le cache de prix. |
| G | **Aucun endpoint d'inventaire (stock final)** : les lots restants n'existent qu'en mémoire pendant le calcul | 🟠 Élevé | Exposer `CostBasisCalculator.get_acquisitions()` par actif/bucket à une date ; indispensable pour le bilan et pour le report des lots d'un exercice à l'autre. |
| H | **Pas de granularité de coût de revient par portefeuille/catégorie** (mutualisation par actif) | 🟠 Élevé | Introduire des « buckets » (asset × catégorie ANC × entité) via le point d'extension `_resolve_bucket_asset`. |
| I | **Prix manquant ⇒ `ZERO_PRICE` silencieux** | 🔴 **Bloquant (fiabilité)** | Rendre bloquant (ou explicitement provisionné) : un rapport ne doit pas être produit avec un coût d'entrée nul. `pot.py:248-257` |
| J | **Devise principale par défaut USD** ; conversion rétroactive implicite | 🟠 Élevé | Forcer EUR ; tracer le taux appliqué (conserver le couple (asset, timestamp, source) dans l'export). |
| K | **Limites freemium tronquent le calcul** (1 000 événements PnL) | 🟠 Élevé (si on garde le code tel quel) | Retirer/neutraliser `FREE_*_LIMIT` pour le fork ; sinon un rapport de société est partiel sans le dire clairement. `accountant.py:248` |
| L | **`include_gas_costs` est un réglage mort** | 🟡 Faible | Ne pas s'appuyer dessus ; le gas est un événement `SPEND/FEE` piloté par les règles. À documenter ou supprimer de l'UI du fork. |
| M | **Pas de TVA / facturation** | 🟠 Élevé (si activité de service) | Hors périmètre « pré-comptabilité », mais à prévoir : si la société facture en crypto, il faut produire la pièce. |
| N | **Export incomplet pour un tiers** : CSV de PnL non exploitable, pas de hash de tx structuré, pas de pièce jointe | 🟠 Élevé | Nouvel export (voir §4). |
| O | **Événements éditables/supprimables, rapport supprimable** | 🟠 Élevé | Verrouillage des périodes validées + piste d'audit (qui/quand/quoi) ; réutiliser `history_events_backup` et les data issues. |
| P | **Pas d'identification de l'entité** (SIREN, dénomination, adresse, régime) | 🟡 Moyen | Table de paramètres société (nécessaire à l'en-tête du FEC et au nommage du fichier). |
| Q | **Pas de lettrage / rapprochement persistant** | 🟡 Moyen | Le mécanisme de `history_event_links` (`ASSET_MOVEMENT_MATCH`, `BRIDGE_MATCH`) existe : à étendre à un lettrage comptable (`link_type` supplémentaire). |
| R | **Interface et messages non traduits en français** (`fr.json` à 30 %) | 🟡 Moyen | Compléter les clés comptables ; sinon les comptables finaux ne peuvent pas travailler. |
| S | **L'`ACB` n'est pas un CUMP français dans son état actuel** — trois écarts précis : (1) `AssetAcquisitionEvent.from_processed_event` ne transporte que `amount`, `timestamp`, **`price`** et `index` (`cost_basis/base.py:51-57`) : les frais d'acquisition ne sont donc intégrés au coût moyen **que si l'appelant les a déjà fondus dans `price`** (ce que fait `get_prices_for_swap` quand `include_fees_in_cost_basis=True`, `pot.py:496-525`) — il n'y a aucun champ de frais séparé ; (2) l'ACB retient un coût moyen **glissant, recalculé à chaque entrée** (`current_total_acb`/`current_amount`, `base.py:369-379`), alors que le CUMP français se calcule **périodiquement (à la clôture ou par période)** ; (3) les `matched_acquisitions` restent consommées dans l'ordre FIFO **même en ACB** (`calculate_spend_cost_basis` du parent, `base.py:199-210`), ce qui est cohérent pour un coût moyen mais rend le **détail des lots** non représentatif d'un inventaire au CUMP. | 🟡 **Moyen à élevé** | Ne pas « activer ACB » en croyant faire du CUMP. Écrire un `CumpCostBasisMethod` explicite (période de référence paramétrable, champ de frais séparé, sortie d'inventaire dédiée) et le valider avec l'expert-comptable. C'est le seul point du moteur où le fork doit **ajouter** une méthode, pas seulement la relabelliser. |

---

## 4. Architecture de fork recommandée (pré-comptabilité, pas comptabilité)

### 4.1 Principe directeur

> **rotki reste le moteur d'acquisition + normalisation + valorisation. Le fork ajoute une couche « traduction comptable » en sortie, sans réécrire le moteur.**

Trois raisons de ne pas transformer rotki en logiciel comptable :
1. Le modèle de données est **mono-ligne avec direction**, pas un journal en partie double : le transformer casserait tout le moteur PnL.
2. La comptabilité française exige **immutabilité et séquence** : c'est un modèle de données différent (écritures validées), pas un attribut supplémentaire.
3. L'objectif est de **préparer** les écritures pour un logiciel tiers : il suffit de produire un **format pivot** (journal, FEC partiel, CSV normé) que le cabinet importe.

### 4.2 Où brancher dans le code

Les points d'insertion exacts (services applicatifs, buckets de coût de revient, tables,
exports, verrouillage d'exercice) sont détaillés dans le document de conception :
[`02-blueprint-fork.md`](./02-blueprint-fork.md), §2 pour le modèle de données qui doit aller
en **base utilisateur** (jamais dans `global.db`, non chiffrée et partagée), §1 et §3 pour les
règles de passage comptable et les formats d'export.

Les deux seuls points qui relèvent de la *vérification* et non de la conception :

- **`accounting_rules` est réutilisable** : la clé `(type, subtype, counterparty)` existe déjà,
  l'écran d'édition et l'import/export JSON aussi (`/accounting/rules/import|export`). Le fork
  peut donc versionner sa doctrine française **en JSON dans son dépôt** plutôt qu'en code.
- **`CostBasisCalculator._resolve_bucket_asset`** (`cost_basis/base.py:599-628`) est le point
  d'extension déjà utilisé pour mutualiser un coût de revient par « collection » d'actifs :
  c'est là que se branchent les **catégories homogènes** exigées par l'ANC 2026-01.

### 4.3 Les trois briques à construire

Journal en partie double, inventaire/valorisation de clôture (ANC 619-11/619-12) et exports
(CSV pivot + FEC + annexe) : contenu détaillé, écritures par type d'événement, 18 colonnes du
FEC et plan de mise en œuvre en [`02-blueprint-fork.md`](./02-blueprint-fork.md) §1-§5.

### 4.4 Ce que le fork peut éviter d'écrire (déjà là)

| Besoin | Déjà fourni |
|---|---|
| Synchronisation exchanges (**22**) + chaînes (**19**) | `exchanges/`, `chain/` |
| Décodage sémantique DeFi (63 protocoles) | `chain/evm/decoding/` |
| Valorisation historique multi-oracles + cache + prix manuels | `inquirer.py`, `history/price.py`, `globaldb/` |
| Import CSV de relevés (CoinTracking, Kraken, Binance, etc.) + CSV générique | `data_import/importers/` |
| Rapprochement dépôt/retrait exchange ↔ transfert on-chain | `asset_movement_matching` + `history_event_links` |
| Rapprochement de bridges inter-chaînes | `bridge_match` |
| Déduplication/idempotence | `UNIQUE(group_identifier, sequence_index)` |
| Anomalies de données | data issues + `/history/actionable_items` |
| Comptes bancaires FR (Qonto) | `banks/qonto.py` |
| Chiffrement local + auth API déployée | SQLCipher + session cookie |
| Reproductibilité des rapports | `pnl_report_settings` |
| Interface web/desktop multilingue | Vue 3 + `fr.json` (à compléter) |
| Superviseur de processus | `starling` |
| Canal agent/LLM | serveur MCP |

### 4.5 Ce qu'il faut retirer ou neutraliser dans le fork

1. Premium/freemium : limites (K), endpoints premium-only pour l'historique de soldes, sync cloud (par défaut risqué pour des données d'entité).
2. Analytics : `usage_analytics.py` envoie OS/version à `rotki.com/api/1/usage_analytics` (désactivable par réglage, **activé par défaut** : `DEFAULT_SUBMIT_USAGE_ANALYTICS = True`, `settings.py:59`) ; `sigil.py` envoie des événements à `sigil.rotki.com`. **À supprimer** dans un fork destiné à des sociétés (une DPA/registre des traitements serait nécessaire).
3. Mise à jour distante des **règles comptables** depuis `rotki/data` (`db/updates.py:435`) : incompatible avec une doctrine figée — le fork doit **embarquer** ses règles FR et ne jamais les écraser.
4. Traductions/UX : vocabulaire fiscal allemand/canadien (`taxfree`, `ACB`) à masquer.
5. `include_gas_costs` (mort) et `taxfree_after_period` (K/A).

### 4.6 Licence : point juridique à traiter avant tout

**AGPL-3.0** : si le fork est distribué ou **exposé en réseau** (mode Docker/self-hosted, ce qui est le cas pour un cabinet ou un client), l'intégralité du code dérivé doit être publiée sous AGPL. Un fork « interne » non distribué n'a pas cette obligation, mais l'exposition en SaaS la déclenche. À arbitrer avec le conseil juridique : c'est structurant pour le modèle de distribution (binaire desktop vs service hébergé).

### 4.7 Interface : le français est à 30 %

`frontend/app/src/locales/fr.json` : 1 482 clés sur 4 877 (30,4 %). Manquent notamment :
`accounting_settings.*` (règles comptables), `profit_loss_report.actionable.*` (les avertissements de fiabilité, **précisément ceux qu'un comptable doit lire**), `cost_basis_table.*` partiellement, `profit_loss_overview.*`.

Un fork destiné à des cabinets doit traiter cela comme un livrable à part entière (les messages d'anomalie sont le cœur de la valeur).

---

## 5. Registre de risques corrigé

| # doc | Sujet | Verdict |
|---|---|---|
| R1 | Perte de données sur migration | **Maintenu**. Atténué (backups + tests), mais réel. Le fork ajoute des migrations ⇒ même discipline requise. |
| R2 | Dérivation faible de la sauvegarde premium (#13393) | **Supprimé.** Corrigé : Argon2id + HKDF + AES-256-GCM par chunks (`crypto.py`), issue fermée le 2026-10-22. |
| R3 | Erreur de décodage/règle silencieuse | **Maintenu et aggravé pour l'usage FR** : ajouter que `ZERO_PRICE` et `taxfree_after_period` produisent des résultats **faux mais plausibles**. |
| R4 | God object | **Réduit** : couche `api/services/` en place, `rotkehlchen.py` = 1 762 lignes. Le point dur restant est `DBHandler` (5 010). |
| R5 | Dépendance aux API tierces | **Maintenu.** |
| R6 | Fragilité du build natif (SQLCipher/PyInstaller/Electron) | **Maintenu**, et **complexifié** : il faut ajouter `colibri` + `starling` (Rust) au build et à la signature. |
| — | *(nouveau)* gevent | **Disparu** (migration terminée). |
| — | *(nouveau)* **Pas de FEC / journal / exercice** | Ce n'est pas un risque mais un **périmètre à construire** : cf. doc 02 §1-§3. Le risque associé est de sous-estimer la charge (partie double + verrouillage + FEC). |
| — | *(nouveau)* Architecture tri-processus | **Nouveau risque** : un fork doit maintenir 3 processus (Python, colibri, starling) et leur contrat. |
| — | *(nouveau)* Fork AGPL exposé en réseau | **Nouveau risque juridique** (§4.6). |
| — | *(nouveau)* Doctrine ANC 2026-01 en évolution | **Nouveau risque métier** : la fiscalité IS des crypto-actifs n'est pas stabilisée (art. 38 CGI par défaut) ; le fork doit versionner sa doctrine et rester paramétrable. |

---

## 6. Réponses directes aux incertitudes du document

| Incertitude du document | Réponse |
|---|---|
| « Aucune métrique chiffrée » | Fournies (§0). |
| « gevent ? » | Non. asyncio + threads, uvicorn. |
| « ORM ? » | Non, confirmé : SQL brut, `read_ctx`/`write_ctx`. |
| « trois bases ? » | Oui, mais **transient est chiffrée** aussi ; `global.db` non chiffrée + un `global.db` embarqué en lecture seule (`PACKAGED_GLOBAL`). |
| « events : `event_identifier` ? » | `group_identifier`, `UNIQUE(group_identifier, sequence_index)`. |
| « `Trade`/`LedgerAction` encore là ? » | Non, supprimés. |
| « `taxfree_after_period` = -1 pour désactiver ? » | C'est `None` (nullable), pas `-1`. Défaut = **1 an**. |
| « `accounting_treatment` taxables ? » | Enum réel : **`SWAP`** et **`BASIS_TRANSFER`** seulement (`chain/evm/accounting/structures.py:31`). Le document parlait d'un « taxable/…/treatment » générique : les 3 booléens sont à part (`taxable`, `count_entire_amount_spend`, `count_cost_basis_pnl`). |
| « API locale sans token ? » | Faux : cookie HMAC + session unique + deny-by-default quand `ROTKI_SESSION_KEY` est défini. |
| « pas d'OpenAPI ? » | Confirmé : pas de génération OpenAPI/apispec ; doc Sphinx (`docs/api.rst`) + schémas marshmallow. |
| « tests : VCR ? » | Confirmé (`vcrpy==8.5.0`, `pytest-vcr`), 491 fichiers de test. |
| « `include_gas_costs` actif ? » | Non : réglage orphelin. |
| « `location` = enum figé ? » | Non : arbre en base extensible (v1.45 / DB v54). |
| « Rust absent ? » | Non : `colibri` + `starling`. |
| « MCP ? » | Oui, serveur MCP complet. |
| « banques ? » | Oui, FinTS + Qonto, et ce sont des `HistoryEvent`. |

---

## 7. Synthèse

La synthèse en une page de l'ensemble de l'analyse (état du code, ce qui est réutilisable,
ce qui doit être construit, arbitrages à rendre) est le document
[`00-synthese.md`](./00-synthese.md). Elle n'est pas dupliquée ici pour éviter deux vérités
divergentes entre les documents.

Les écarts et les adaptations détaillés restent les §1 à §5 ci-dessus ; la traduction en
travaux concrets est en [`02-blueprint-fork.md`](./02-blueprint-fork.md).

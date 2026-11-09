# Pré-comptabilité crypto-actifs en droit français — analyse code-vérifiée de rotki

**Livrable de synthèse.** Trois questions posées, trois réponses. Les preuves techniques
sont dans les deux documents détaillés :

| Document | Contenu |
|---|---|
| [`01-verification-etat-des-lieux.md`](./01-verification-etat-des-lieux.md) | Vérification point par point du document de synthèse initial, corrections, réponses aux incertitudes, registre de risques corrigé |
| [`02-blueprint-fork.md`](./02-blueprint-fork.md) | Modèle de données à ajouter, règles de passage comptable, formats d'export (journal, FEC, annexe), plan de mise en œuvre, tests |

Base analysée : dépôt `rotki/rotki`, commit `9b4b0db9a02719678d8f1cdc93c019c82b36f1fc`
(`develop`, 2026-11-09), version applicative 1.44.2. **Tout ce qui suit a été lu dans le code
de ce commit**, pas déduit de la documentation marketing.

---

## 1. Le document initial : ce qui est faux, ce qui est juste

Le document a été rédigé sans accès au dépôt. Sur les points **structurants**, il se trompe
sur l'architecture, ce qui invalide une partie de son analyse de risque.

### 1.1 Les cinq erreurs qui changent les conclusions

| Affirmation du document | Réalité dans le code |
|---|---|
| « Backend WSGI gevent, greenlets, monkey-patching » — présenté comme un risque 🔴 **extrême et irréversible** | **Faux.** Plus aucune dépendance ni import de `gevent`. Le serveur est **uvicorn 0.54.0** + pont `a2wsgi` + WebSocket natif. Un seul serveur (`rotkehlchen/api/asgi.py`). Monkey-patching supprimé. Migration documentée et **terminée** (`docs/designs/gevent_to_asyncio.md`, phases 0→7 = done). Le risque disparaît. |
| « Modèle en transition : `Trade` + `LedgerAction` + `AssetMovement` coexistent avec `HistoryEvent` » | **Faux.** `LedgerAction` n'existe plus (0 occurrence hors tests), pas de table `trades`. Un **modèle unique** : `history_events` + `UNIQUE(group_identifier, sequence_index)`. C'est une **très bonne** nouvelle pour la pré-comptabilité. |
| « Énumération `Location` figée » | **Faux.** Depuis la user DB v54 : table `locations` **auto-référencée** (arbre de profondeur arbitraire) + `location_aliases`, avec un CRUD complet (`/locations`, `/locations/<id>/usage`, `/locations/aliases`). Le fork peut modéliser ses axes de suivi **sans migration**. |
| « API locale protégée seulement par le loopback, pas de token » | **Faux.** Cookie de session HMAC-SHA256 (`rotki_session`), **session unique active** (`sid` persisté dans `session.db`), deny-by-default par route+verbe dès que `ROTKI_SESSION_KEY` est défini, CORS compilé en regex ancrées. Le mode déployé est **déjà** correctement sécurisé. |
| « Sauvegarde premium à dérivation faible (issue #13393), statut inconnu, risque élevé » | **Corrigé.** Issue **fermée le 2026-10-22**. Le code utilise **Argon2id → HKDF → AES-256-GCM par chunks** avec AAD authentifiant l'en-tête, l'index de chunk et le flag final (`rotkehlchen/crypto.py`, format `RKPBAK01`). Le risque R2 doit être **supprimé**. |

### 1.2 Les omissions notables (le document ne les mentionne pas du tout)

- **Architecture à trois processus** : `rotki-core` (Python), **`colibri`** (Rust : prix, icônes, globalDB, cache — sa propre API HTTP et sa propre base SQLCipher) et **`starling`** (Rust : superviseur de processus avec graphe de services et gating de readiness). Electron ne supervise plus qu'un seul enfant. Conséquence : le build et la maintenance d'un fork portent sur 3 binaires, pas 1.
- **Serveur MCP intégré** (`rotkehlchen/mcp/`, outils `analytics`/`assets`/`info`/`prices`/`taxonomy`, tokens à audience `mcp`, modes de confidentialité). C'est le canal d'intégration le moins coûteux vers un agent ou un outil tiers.
- **Comptes bancaires réels** : `banks/qonto.py` (Qonto) et `banks/fints.py` (FinTS/PSD2), exposés comme `HistoryEvent` de type `BANK_TRANSACTION_EVENT`. Pour une société française, c'est la **jambe fiat** indispensable au rapprochement.
- **Couche « data issues »** : détecteurs d'anomalies (`history/data_issues/`), conflits non résolus (`unresolved_remote_conflicts`), arrêts de traitement (`bucket_processing_stops`), items actionnables (`/history/actionable_items`). Matière première d'une **piste d'audit**.
- **Export CSV de tous les événements valorisés** (`POST /history/events/export`), distinct du CSV de PnL : chaque événement avec `amount`, `fiat_value`, `asset_symbol`, `direction`, `location_path`.
- **Provenance des prix en base** : `price_history(from_asset, to_asset, source_type, timestamp, price)` — la source et l'horodatage sont conservés, ce qui répond directement à l'exigence d'annexe de l'ANC 2026-01.
- **Réglage mort** : `include_gas_costs` est déclaré, stocké, exporté… mais **aucun code comptable ne le lit**. Le document le présentait comme une règle métier active.

### 1.3 Ce que le document dit juste (et qui compte)

Licence **AGPL-3.0** ; **SQLCipher** AES-256 ; pas d'ORM (SQL brut, `read_ctx`/`write_ctx`) ;
**décimaux exacts** (`FVal` sur `Decimal`, précision ~78 chiffres, `float` interdit) ;
**réglages du rapport figés** en base (reproductibilité) ; **règles comptables en données**
éditables et importables/exportables en JSON ; **dégradation gracieuse** avec `is_complete`,
`missing_acquisitions`, `missing_prices` ; **Electron durci** (`nodeIntegration:false`,
`sandbox:true`, `contextIsolation:true`) ; migrations testées avec sauvegarde préalable ;
conservation des **données brutes on-chain** permettant le re-décodage (vraie piste d'audit).

### 1.4 Chiffres réels (le document n'en donnait aucun)

2 050 fichiers Python hors tests / 258 681 lignes · 491 fichiers de test / 267 149 lignes ·
4 025 fichiers frontend / 483 022 lignes · 191 routes REST · 182 classes de ressources ·
350 méthodes de `RestAPI` · **22 exchanges** · **19 chaînes** (13 EVM) · 63 décodeurs de
protocoles EVM · 28 migrations de schéma (user DB v54) + 30 migrations de données ·
global DB v19 · transient DB v2.
`rotkehlchen.py` = 1 762 lignes, `api/rest.py` = 5 470 (mais désormais **délégué** à 15
services dans `api/services/`), `db/dbhandler.py` = 5 010 — le vrai point dur est le
`DBHandler`, pas l'orchestrateur.

---

## 2. Ce qui est déjà utilisable tel quel pour la pré-comptabilité d'une société à l'IS

**Réponse courte : tout l'amont.** rotki couvre déjà l'essentiel de ce qui coûte cher dans un
tel outil — collecte, normalisation, valorisation, traçabilité. Les **18 atouts** détaillés sont
en `01` §3.2 ; les cinq qui portent réellement le projet :

1. **Arithmétique exacte de bout en bout.** `FVal` = `Decimal`, précision `ceil(log10(2**256))`,
   les `float` sont rejetés, les montants sont stockés en TEXT. Aucune dérive d'arrondi : pour
   une comptabilité, c'est la condition *sine qua non*, et elle est tenue.
2. **Un modèle d'événement canonique unique et idempotent.** `history_events` avec
   `UNIQUE(group_identifier, sequence_index)` : re-synchroniser un exchange ne duplique pas
   l'écriture. Source de vérité unique, base saine pour une piste d'audit.
3. **Les lots d'acquisition sont matérialisés.** `MatchedAcquisition` porte `amount`, `event`
   (`timestamp`, `full_amount`, `rate`, `index`) et `taxable`. Le coût de revient d'une cession
   est donc **justifiable lot par lot** — ce qu'exige l'annexe (méthode PEPS/CUMP).
4. **La doctrine comptable est une donnée, pas du code.** `accounting_rules`
   (`type`, `subtype`, `counterparty`, `taxable`, `count_entire_amount_spend`,
   `count_cost_basis_pnl`, `accounting_treatment`, `is_event_specific`) est éditable et
   **importable/exportable en JSON** (`/accounting/rules/import|export`). Un fork peut donc
   versionner son « pack de règles FR » dans son dépôt et le charger à l'installation.
5. **La traçabilité de la valorisation est déjà en base.** `price_history` conserve la
   **source** de chaque prix, et les prix manuels sont prioritaires et datés. L'obligation de
   documenter « plateformes/sources utilisées et horaire de valorisation » (ANC 2026-01,
   annexe) est donc satisfaite par le schéma — il ne reste qu'à la restituer.

À quoi s'ajoutent, sans développement : 22 exchanges, 19 chaînes, 63 décodeurs DeFi,
rapprochement dépôts/retraits exchange ↔ transferts on-chain (`history_event_links`,
`ASSET_MOVEMENT_MATCH`) et bridges (`BRIDGE_MATCH`), import CSV multi-sources, flux bancaires
Qonto/FinTS, prix manuels historiques, `fr.json` (30 % — voir réserve ci-dessous), et
l'export « tous les événements » valorisé qui constitue la **table de base** du fork.

**Trois réserves à lever avant de produire des chiffres opposables** (détail en `01` §2.3) :

| Réglage | Défaut actuel | Effet en France |
|---|---|---|
| `taxfree_after_period` | `YEAR_IN_SECONDS` (règle **allemande**) | Les cessions détenues > 1 an sont classées « exonérées ». En France, **aucune exonération de ce type** pour une société à l'IS ⇒ PnL faussé. |
| `main_currency` | **USD** | Valorisation dans la mauvaise devise ; la conversion USD→EUR doit être documentée. |
| Prix manquant | `ZERO_PRICE` **silencieux** (`pot.py:251,257`) | Un actif sans prix entre dans le coût de revient à zéro ⇒ plus-value maximale, sans erreur bloquante. |

Bonne nouvelle symétrique : `include_crypto2crypto = True` par défaut est **correct** pour une
société à l'IS (les échanges crypto↔crypto y sont imposables), contrairement au cas du
particulier français.

---

## 3. Ce qui doit être adapté dans le fork

**Réponse courte : tout l'aval.** Le moteur produit un **PnL de particuliers** ; il ne produit
ni écritures, ni inventaire, ni états. Quatre manques sont structurants :

| # | Manque | Conséquence | Preuve |
|---|---|---|---|
| 1 | **Pas de partie double.** Un `ProcessedAccountingEvent` = une ligne + une `direction` (`IN`/`OUT`/`NEUTRAL`), pas de contrepartie. | Il faut une **couche de traduction** vers des écritures équilibrées. C'est le cœur du fork. | `accounting/structures/processed_event.py` |
| 2 | **Aucune notion de plan de comptes, journal, exercice, clôture.** Le rapport est une plage de dates (`start_ts`/`end_ts`), supprimable (`DELETE /reports/<id>`). | Pas d'état opposable, pas de FEC, pas d'à-nouveaux. | `api/server.py:336-348`, `db/reports.py:227` |
| 3 | **Aucune valorisation à la clôture ni dépréciation.** 0 occurrence de `fair_value`, `revaluat`, `impair`, `unrealiz`, `depreciat` dans le backend. | L'article 619-12 de l'ANC 2026-01 (valeur vénale à chaque clôture, variations en comptes transitoires) et la dépréciation **ligne à ligne** sont intégralement à écrire. | grep exhaustif |
| 4 | **Aucun export comptable.** 0 occurrence de `FEC`, `journal_entry`, `tva`/`vat_`, `invoice`. Le CSV de PnL n'est pas ingérable par un logiciel comptable (types `AccountingEventType`, `cost_basis_*` en texte libre ou en formules Excel). | Journal + FEC + annexe à construire. | `accounting/export/csv.py` |

S'y ajoutent des écarts réels mais moins lourds : **aucun endpoint d'inventaire** (les lots
restants n'existent qu'en mémoire pendant le calcul — `BaseCostBasisMethod.get_acquisitions`,
`cost_basis/base.py:151`, utilisé seulement par les tests) ; **coût de revient mutualisé par
actif** tous emplacements confondus (clé `Asset`, `base.py:595`) alors que l'ANC raisonne par
**catégorie homogène** ; **limites freemium qui tronquent le calcul lui-même** (1 000
événements PnL, `accountant.py:248`) ; **pas d'identification de l'entité** (SIREN) ; **pas de
lettrage** ; **`fr.json` à 30,4 %** (1 482 clés sur 4 877), les messages d'anomalie comptable
(`profit_loss_report.actionable.*`) étant précisément parmi les absents.

**Le point technique le plus délicat** est le **CUMP**. `AverageCostBasisMethod` (ACB) existe,
mais ce n'est pas un CUMP français : le coût moyen est **glissant** (recalculé à chaque entrée),
les frais ne sont pris en compte que s'ils ont été fondus dans `price` par l'appelant (aucun
champ de frais dans `AssetAcquisitionEvent`, `base.py:51-57`), et les lots restent consommés en
ordre FIFO même en ACB. Le fork doit écrire un `CumpCostBasisMethod` explicite (période de
référence, frais séparés, inventaire dédié), pas activer `ACB` en le relabellisant.

**Point d'insertion privilégié** : `CostBasisCalculator._resolve_bucket_asset`
(`base.py:599-628`) sait déjà rediriger un actif vers un « bucket » (c'est le mécanisme de
`use_asset_collections_in_cost_basis`). C'est là qu'on branche les **catégories homogènes**
(asset × catégorie ANC × entité) sans réécrire le moteur.

### 3.1 Architecture recommandée

> **rotki reste le moteur d'acquisition et de valorisation ; le fork ajoute un traducteur comptable en sortie.**

Trois raisons de ne pas transformer rotki en logiciel comptable : (a) le modèle mono-ligne
n'est pas un journal et le convertir casserait tout le moteur PnL ; (b) la comptabilité exige
l'**immutabilité** et une **séquence** d'écritures, qui sont un modèle de données distinct ;
(c) l'objectif est de **préparer** les écritures pour un logiciel tiers, donc de produire un
**format pivot** — pas de remplacer le logiciel du cabinet.

Trois briques à construire (modèle de données SQL complet, règles de passage comptable par
type d'événement, et les 18 colonnes du FEC sont en `02` §1-§3) :

1. **Journal en partie double**, alimenté par les `ProcessedAccountingEvent` + lots, avec
   mapping `(type, subtype, counterparty) → (compte PCG, journal, sens)` **en données**.
2. **Inventaire et clôture** : quantités par ligne à la date de clôture, valeur vénale
   (prix manuel prioritaire, source conservée), variations en comptes transitoires,
   dépréciation ligne à ligne.
3. **Exports** : CSV pivot « journal » (intégration), **FEC** au format A47 A-1 pour
   l'archivage/contrôle, et les éléments d'annexe.

Tout le nouveau modèle va en **base utilisateur** (SQLCipher) : `global.db` est **non chiffrée**
et partagée entre tous les comptes — elle ne doit jamais recevoir un plan de comptes ni un SIREN.

### 3.2 Cadre réglementaire et fiscal à respecter

- **ANC 2026-01** (homologué le 12/08/2026, JO du 03/09/2026) : refonte de la section 619 du
  PCG. Comptes **513** (jetons de monnaie électronique), **522** / **52221** (crypto-actifs
  détenus / prêtés), **524** (empruntés). Évaluation des crypto-actifs du 522 à la **valeur
  vénale à chaque clôture**, variations en comptes transitoires (art. 619-12) ; dépréciation
  **ligne à ligne sans compensation** ; sorties en **PEPS ou CUMP par catégorie homogène** ;
  annexe sur les modalités et l'horaire de valorisation. Application **obligatoire aux
  exercices ouverts à compter du 1er janvier 2027**, anticipée possible.
- **Fiscalité** : pas de régime autonome pour les crypto-actifs des personnes morales à l'IS
  — connexion fiscalo-comptable (art. 38 CGI), taux **15 % jusqu'à 42 500 €** puis **25 %**,
  aucune exonération par durée de détention. La valorisation **comptable** est donc
  directement déterminante pour l'impôt, y compris le risque d'imposition de plus-values
  latentes via les comptes transitoires.
- **FEC** : art. **A47 A-1 du LPF**, 18 colonnes dans un ordre strict, séparateur tabulation,
  un fichier par exercice, numérotation séquentielle continue, écritures **figées**,
  nommage `SIREN + FEC + date de clôture`. C'est cette contrainte qui impose le
  **verrouillage d'exercice** (écart n°2 ci-dessus).

### 3.3 Ce qu'il faut retirer du fork

Limites freemium et gating premium (dont tous les endpoints `/balances/historical/*`, utiles
pour l'inventaire, protégés uniquement par décorateur `require_premium_user`) ; **analytics**
(`usage_analytics.py` → `rotki.com`, `sigil.py` → `sigil.rotki.com`, activés par défaut en
production : `DEFAULT_SUBMIT_USAGE_ANALYTICS = True`) ; **mise à jour distante des règles
comptables** depuis `github.com/rotki/data` (`db/updates.py:435`) — incompatible avec une
doctrine figée et versionnée ; vocabulaire fiscal allemand/canadien dans l'UI ; réglages morts
ou dangereux (`include_gas_costs`, `taxfree_after_period`).

**Point juridique à trancher avant la première ligne de code** : la licence est **AGPL-3.0**.
Un fork distribué — ou simplement **exposé en réseau** (mode Docker/self-hosted, cas d'un
cabinet ou d'un client) — doit publier son code dérivé sous AGPL. Cela conditionne le modèle de
distribution (binaire desktop vs service hébergé).

---

## 4. En une page

Avec **258 681 lignes de Python** (hors tests) et **483 022 lignes de frontend**, rotki 1.44
n'est pas ce que décrit le document : ce n'est plus un monolithe gevent, mais **trois processus**
(Python, Rust, Rust) exposant 191 routes REST, un WebSocket et un serveur MCP, sur une base
chiffrée, avec un modèle d'historique unique et stabilisé et une arithmétique décimale exacte.

Pour une **pré-comptabilité** française, **l'amont est déjà construit et de bon niveau** :
collecte (22 exchanges / 19 chaînes), normalisation canonique idempotente, décodage sémantique
(63 protocoles), valorisation multi-oracles avec provenance conservée, lots FIFO matérialisés,
doctrine comptable en données versionnables, transparence sur les anomalies, flux bancaires
français, prix manuels datés. Le fork n'a pas à refaire ce qui représente l'essentiel du coût.

**L'écart est entièrement en aval**, et il est réel : pas de partie double, pas de plan de
comptes/journaux/exercice, pas de valorisation de clôture ni de dépréciation (ANC 619-11/619-12),
pas d'export (journal, FEC) et pas d'inventaire exposé. Ce sont quatre chantiers identifiés,
chiffrables et sans dépendance cachée — plus un cinquième, méthodologique : écrire un vrai
`CumpCostBasisMethod` plutôt que réutiliser l'ACB canadien.

**Trois réglages à corriger avant toute production de chiffres** : `taxfree_after_period`
(exonération allemande à un an, active par défaut), devise principale USD, et prix manquant
silencieusement à zéro.

**Deux décisions à faire trancher hors code** : la licence AGPL-3.0 selon le mode d'exposition,
et la doctrine comptable (PEPS vs CUMP, catégories homogènes, classement ANC, horaire de
valorisation) — parce que l'ANC 2026-01 rend la valorisation comptable **directement
déterminante pour l'impôt sur les sociétés**, la fiscalité des crypto-actifs des personnes
morales n'ayant pas de régime autonome.

# Blueprint technique du fork « pré-comptabilité FR »

Document de conception, à lire après `01-verification-etat-des-lieux.md`.
Toutes les références de fichiers renvoient au commit `9b4b0db` de `rotki/rotki`.

---

## 0. Positionnement retenu

> **rotki = moteur d'acquisition et de valorisation. Le fork = traducteur comptable.**
> Aucune écriture n'est « inventée » : chaque ligne du journal est dérivable d'un
> `ProcessedAccountingEvent` + de ses lots d'acquisition, et reste traçable jusqu'à
> l'événement source (`history_events.identifier`) puis à la transaction on-chain.

Chaîne cible :

```
sources (22 exchanges, 19 chaînes, Qonto/FinTS, CSV, saisie manuelle)
        │  (existant)
        ▼
history_events  ── normalisation canonique, idempotente   (existant)
        │
        ├──► Accountant / CostBasisCalculator ──► ProcessedAccountingEvent  (existant)
        │                                              │
        │                                              ▼
        │                                   ★ COUCHE FR (à écrire)
        │                                     ├─ plan de comptes (PCG 2026)
        │                                     ├─ règles de mapping
        │                                     ├─ journal (partie double)
        │                                     ├─ inventaire & valeur vénale de clôture
        │                                     └─ dépréciations ligne à ligne
        │                                              │
        ▼                                              ▼
  /history/events/export (existant)          ★ exports (à écrire)
                                              ├─ CSV pivot (intégration logiciel tiers)
                                              ├─ FEC (A47 A-1)
                                              └─ annexe (sources de valorisation)
```

---

## 1. Modèle de données à ajouter (base **utilisateur** chiffrée)

> ⚠️ Tout ce qui concerne l'entité (SIREN, plan de comptes, exercices) va dans
> `rotkehlchen.db` (SQLCipher), **jamais** dans `global.db` (non chiffrée et partagée
> entre tous les comptes — cf. doc 01 §1.5).

### 1.1 Entité et paramètres comptables

```sql
-- Identité de la personne morale (en-tête FEC, nommage du fichier)
CREATE TABLE IF NOT EXISTS fr_entity (
    identifier INTEGER PRIMARY KEY,
    siren TEXT NOT NULL,                -- 9 chiffres, sert au nommage FEC
    legal_name TEXT NOT NULL,
    legal_form TEXT,                     -- SAS, SARL, SA…
    vat_number TEXT,
    fiscal_year_end TEXT NOT NULL,       -- 'MM-DD', ex. '12-31'
    currency TEXT NOT NULL DEFAULT 'EUR',
    created_ts INTEGER NOT NULL
);

-- Doctrine comptable versionnée (une ligne par version, jamais écrasée)
CREATE TABLE IF NOT EXISTS fr_accounting_doctrine (
    identifier INTEGER PRIMARY KEY,
    version INTEGER NOT NULL UNIQUE,
    label TEXT NOT NULL,                 -- 'ANC 2026-01 — CUMP — exercice 2027'
    valuation_method TEXT NOT NULL       -- 'FIFO' | 'CUMP'
    CHECK (valuation_method IN ('FIFO', 'CUMP')),
    classification TEXT NOT NULL         -- 'COMPTE_522' | 'IMMO_INCORPORELLE'
    CHECK (classification IN ('COMPTE_522', 'IMMO_INCORPORELLE')),
    value_date_policy TEXT NOT NULL,     -- 'CLOSE_UTC' | 'CLOSE_23:59:59_CET' | 'MANUAL'
    price_source_policy TEXT NOT NULL,   -- JSON: ordre des oracles + 'manual_only'
    effective_from INTEGER NOT NULL,     -- timestamp
    created_ts INTEGER NOT NULL
);
```

`valuation_method` : l'ANC 2026-01 (art. 619-12) n'autorise que **PEPS (FIFO)** ou
**CUMP**, appliqué **par catégorie homogène**. Toute autre méthode doit être refusée
par le fork (donc `HIFO`, `LIFO` et l'`ACB` canadien masqués).

### 1.2 Plan de comptes

```sql
CREATE TABLE IF NOT EXISTS fr_chart_of_accounts (
    account_number TEXT PRIMARY KEY,     -- '522100', '512', '627', '775'
    label TEXT NOT NULL,
    account_type TEXT NOT NULL           -- 'ACTIF_IMMO','ACTIF_CIRC','TRESORERIE',
    CHECK (account_type IN ('ACTIF_IMMO','ACTIF_CIRC','TRESORERIE','CHARGE',
                            'PRODUIT','TRANSITOIRE_ACTIF','TRANSITOIRE_PASSIF',
                            'DETTE','CREANCE')),
    is_pcg_builtin INTEGER NOT NULL DEFAULT 1 CHECK (is_pcg_builtin IN (0,1))
);
```

Socle PCG 2026 minimal à embarquer (comptes modifiés par ANC 2026-01) :

| Compte | Libellé | Usage |
|---|---|---|
| 522 | Crypto-actifs et assimilés détenus | détention (évaluation à la valeur vénale, art. 619-12) |
| 5221 / 5222 / … | sous-comptes par catégorie homogène | **catégorie = bucket de coût de revient** |
| 52221 | Crypto-actifs et assimilés détenus **prêtés** | prêts |
| 524 | Crypto-actifs et assimilés **empruntés** | emprunts |
| 513 | Jetons de monnaie électronique | e-money tokens MiCA |
| 512 / 512x | Banque (par compte Qonto/FinTS) | jambe fiat |
| 627 | Services bancaires et assimilés | frais d'exchange, frais réseau |
| 775 / 675 | Produits / charges sur cessions | résultat de cession |
| 76x / 66x | Produits / charges divers | rewards, gas non activable |
| 47x | Comptes transitoires | **variations de valeur vénale à la clôture** (art. 619-12) |
| 29x / 39x | Dépréciations | dépréciation ligne à ligne (214-5/214-6) |
| 401 / 411 | Fournisseurs / clients | flux d'exploitation |

### 1.3 Mapping événement → écriture

```sql
CREATE TABLE IF NOT EXISTS fr_posting_rules (
    identifier INTEGER PRIMARY KEY,
    -- clé d'identification de l'événement, alignée sur accounting_rules
    event_type TEXT NOT NULL,
    event_subtype TEXT NOT NULL,
    counterparty TEXT NOT NULL DEFAULT 'NONE',
    -- conditions optionnelles
    location_identifier TEXT REFERENCES locations(identifier),
    asset_category TEXT,                 -- bucket (voir §1.4)
    -- résultat : modèle d'écriture
    journal_code TEXT NOT NULL,          -- 'ACH','VTE','BNQ','OD','IMMO'
    debit_account TEXT REFERENCES fr_chart_of_accounts(account_number),
    credit_account TEXT REFERENCES fr_chart_of_accounts(account_number),
    amount_basis TEXT NOT NULL           -- quelle base : 'PRIX_VENAL','COUT_REVIENT',
    CHECK (amount_basis IN ('PRIX_VENAL','COUT_REVIENT','MONTANT_EUR','FRAIS_EUR')),
    priority INTEGER NOT NULL DEFAULT 100,
    doctrine_version INTEGER REFERENCES fr_accounting_doctrine(version)
);
```

> **Choix de conception** : réutiliser la clé `(event_type, event_subtype, counterparty)`
> d'`accounting_rules` permet de **réutiliser l'écran d'édition existant** et l'import/export
> JSON (`/accounting/rules/import`) pour versionner la doctrine FR dans le dépôt du fork.
> On ajoute seulement les colonnes de destination comptable.

### 1.4 Catégories homogènes (buckets de coût de revient)

```sql
CREATE TABLE IF NOT EXISTS fr_asset_bucket (
    identifier INTEGER PRIMARY KEY,
    label TEXT NOT NULL,                 -- 'BTC-immobilisation', 'ETH-trading'
    main_asset TEXT NOT NULL,            -- asset racine (ou collection)
    account_number TEXT NOT NULL REFERENCES fr_chart_of_accounts(account_number),
    doctrine_version INTEGER NOT NULL REFERENCES fr_accounting_doctrine(version),
    UNIQUE(label, main_asset, doctrine_version)
);

CREATE TABLE IF NOT EXISTS fr_asset_bucket_member (
    bucket_id INTEGER NOT NULL REFERENCES fr_asset_bucket(identifier) ON DELETE CASCADE,
    asset_identifier TEXT NOT NULL,      -- tout token/wrapped rattaché
    PRIMARY KEY (bucket_id, asset_identifier)
);
```

C'est la matérialisation de l'exigence « FIFO ou CUMP **par catégorie homogène** ».
Côté moteur, le point d'insertion est `CostBasisCalculator._resolve_bucket_asset`
(`accounting/cost_basis/base.py:600`), qui sait déjà rediriger un actif vers un bucket
(c'est ce que fait `use_asset_collections_in_cost_basis`).

### 1.5 Exercices et journal

```sql
CREATE TABLE IF NOT EXISTS fr_fiscal_period (
    identifier INTEGER PRIMARY KEY,
    label TEXT NOT NULL,                 -- 'Exercice 2027'
    start_ts INTEGER NOT NULL,
    end_ts INTEGER NOT NULL,
    status TEXT NOT NULL DEFAULT 'OPEN'  -- 'OPEN' | 'CLOSED' | 'FILED'
    CHECK (status IN ('OPEN','CLOSED','FILED')),
    doctrine_version INTEGER NOT NULL REFERENCES fr_accounting_doctrine(version),
    snapshot_pnl_report_id INTEGER,      -- rapport PnL figé au moment de la clôture
    closed_ts INTEGER,
    closing_hash TEXT,                   -- SHA-256 du FEC produit (scellement)
    UNIQUE(start_ts, end_ts)
);

-- Journal : une écriture = N lignes équilibrées
CREATE TABLE IF NOT EXISTS fr_journal_entry (
    identifier INTEGER PRIMARY KEY,
    period_id INTEGER NOT NULL REFERENCES fr_fiscal_period(identifier),
    entry_number INTEGER NOT NULL,       -- EcritureNum : séquentiel, continu, par journal
    journal_code TEXT NOT NULL,
    entry_date TEXT NOT NULL,            -- 'YYYYMMDD'
    label TEXT NOT NULL,
    piece_ref TEXT,                      -- hash de tx / id d'événement
    piece_date TEXT,
    validation_date TEXT,
    currency TEXT NOT NULL DEFAULT 'EUR',
    source_group_identifier TEXT,        -- history_events.group_identifier
    created_ts INTEGER NOT NULL,
    UNIQUE(period_id, journal_code, entry_number)
);

CREATE TABLE IF NOT EXISTS fr_journal_line (
    identifier INTEGER PRIMARY KEY,
    entry_id INTEGER NOT NULL REFERENCES fr_journal_entry(identifier) ON DELETE CASCADE,
    line_number INTEGER NOT NULL,
    account_number TEXT NOT NULL REFERENCES fr_chart_of_accounts(account_number),
    account_label TEXT NOT NULL,
    auxiliary_account TEXT,
    auxiliary_label TEXT,
    debit TEXT NOT NULL DEFAULT '0',     -- TEXT : décimaux exacts, comme partout dans rotki
    credit TEXT NOT NULL DEFAULT '0',
    currency_amount TEXT,                -- Montantdevise
    currency_code TEXT,                  -- Idevise
    lettering TEXT,                      -- EcritureLet
    lettering_date TEXT,
    -- traçabilité
    history_event_id INTEGER,
    asset_identifier TEXT,
    asset_amount TEXT,
    asset_price TEXT,
    price_source TEXT,
    UNIQUE(entry_id, line_number)
);
```

**Invariants à tester** (golden tests) :
1. `SUM(debit) == SUM(credit)` par écriture (à l'unité de la devise de tenue).
2. `entry_number` **strictement croissant sans trou** par `(period_id, journal_code)`.
3. Aucune écriture dont `period.status != 'OPEN'` ne peut être créée/modifiée/supprimée.
4. Toute ligne non-fiat possède `history_event_id` non nul (traçabilité obligatoire).

---

## 2. Règles de passage comptable (mapping initial)

Table de départ proposée, à valider par l'expert-comptable. Elle s'exprime avec les
énumérations **réelles** de rotki (`HistoryEventType` / `HistoryEventSubType`, cf.
`history/events/structures/types.py:33` et suivants).

| Événement rotki | Nature économique | Écriture (débit / crédit) | Journal |
|---|---|---|---|
| `TRADE` / `SPEND` (contre fiat), `accounting_treatment` NULL | Acquisition crypto | 522x **D** montant EUR · 512 **C** montant EUR · 627 **D** frais | ACH |
| `TRADE` / `SPEND` (contre crypto) | Cession + acquisition | 522y **D** valeur vénale reçue · 522x **C** coût de revient des lots · 775/675 écart | VTE |
| `TRADE` / `SPEND` contre fiat (sortie) | Cession crypto | 512 **D** produit net · 522x **C** coût de revient · 775/675 écart | VTE |
| `SPEND` / `FEE` (counterparty `CPT_GAS`) | Frais de réseau | 627 **D** · 522x **C** (si payé en crypto : cession accessoire) | OD |
| `RECEIVE` / `REWARD`, `STAKING` / `REWARD` | Produit (staking, intérêts) | 522x **D** valeur vénale du jour · 76x **C** | OD |
| `RECEIVE` / `AIRDROP` | Produit exceptionnel | 522x **D** · 778 **C** | OD |
| `DEPOSIT` / `DEPOSIT_ASSET` (vers protocole), sans perte de propriété | **Pas d'écriture** : transfert interne | — (uniquement du suivi quantité) | — |
| `WITHDRAWAL` / `REMOVE_ASSET` | Retour de protocole | — | — |
| `GENERATE_DEBT` / `PAYBACK_DEBT` | Emprunt DeFi | 522x **D** · 524/168 **C** (puis inverse au remboursement) | OD |
| `TRANSFER` / `NONE` entre deux comptes détenus | Transfert interne | — (aucun impact résultat) — **mais impact localisation** | — |
| `ASSET_MOVEMENT` (dépôt/retrait exchange) | Transfert interne + frais | 627 **D** frais · 522x **C** | OD |
| `BANK_TRANSACTION_EVENT` (Qonto/FinTS) | Flux bancaire | 512 **D**/**C** selon sens, contrepartie 4xx | BNQ |
| `LOSS` | Perte (hack, burn) | 667 **D** · 522x **C** | OD |
| `ADJUSTMENT` / `EXCHANGE_ADJUSTMENT` | Régularisation | 658/758 | OD |
| *Clôture* — variation de valeur vénale (619-12) | Gain latent | 522x **D** · 47x **C** (transitoire passif) | OD |
| *Clôture* — variation de valeur vénale (619-12) | Perte latente | 47x **D** (transitoire actif) · 522x **C** | OD |
| *Clôture* — dépréciation ligne à ligne | Dépréciation | 681/6817 **D** · 29x/39x **C** | OD |

> Trois règles **spécifiquement françaises** n'ont pas d'équivalent dans rotki et doivent
> être créées : les deux lignes de clôture « valeur vénale » et la dépréciation ligne à
> ligne. Le reste se déduit des événements déjà produits.

---

## 3. Export vers le logiciel comptable tiers

### 3.1 Format pivot « journal » (recommandé pour l'intégration)

CSV, UTF-8, séparateur `;`, une ligne = une jambe :

```csv
date;journal;ecriture_num;compte;compte_lib;auxiliaire;piece_ref;piece_date;libelle;debit;credit;devise;montant_devise;source_event_id;tx_ref;prix_source
20270315;VTE;VTE-2027-000042;512;Banque Qonto;;a1b2c3…;20270315;Cession 0,5 BTC;30000,00;0,00;EUR;;12034;0x9f…;COINGECKO@2027-03-15T14:00Z
20270315;VTE;VTE-2027-000042;522100;Crypto-actifs détenus - BTC;;a1b2c3…;20270315;Cession 0,5 BTC;0,00;21000,00;EUR;;12034;0x9f…;FIFO
20270315;VTE;VTE-2027-000042;775;Produits sur cessions;;a1b2c3…;20270315;Plus-value de cession;0,00;9000,00;EUR;;12034;0x9f…;
```

Champs obligatoires imposés par le fork (à afficher comme « non négociables ») :
`compte`, `debit`/`credit` (décimaux exacts, virgule), `piece_ref`, `source_event_id`,
`tx_ref`, `prix_source`.

### 3.2 FEC — article A47 A-1 du LPF

Les **18 colonnes**, dans cet ordre exact :

| # | Colonne | Contenu dans le fork |
|---|---|---|
| 1 | `JournalCode` | `ACH`, `VTE`, `BNQ`, `OD`, `IMMO` |
| 2 | `JournalLib` | « Achats crypto-actifs », … |
| 3 | `EcritureNum` | `{JournalCode}-{YYYY}-{NNNNNN}` séquentiel continu |
| 4 | `EcritureDate` | `YYYYMMDD` |
| 5 | `CompteNum` | compte PCG (≤ 13 car.) |
| 6 | `CompteLib` | libellé du plan de comptes |
| 7 | `CompAuxNum` | auxiliaire (compte bancaire, exchange) |
| 8 | `CompAuxLib` | libellé de l'auxiliaire |
| 9 | `PieceRef` | hash de transaction ou id d'événement |
| 10 | `PieceDate` | `YYYYMMDD` |
| 11 | `EcritureLib` | libellé de l'opération |
| 12 | `Debit` | décimal, virgule, 2 déc. |
| 13 | `Credit` | décimal, virgule, 2 déc. |
| 14 | `EcritureLet` | lettrage (rapprochement bancaire) |
| 15 | `DateLet` | `YYYYMMDD` |
| 16 | `ValidDate` | date de validation (= clôture) |
| 17 | `Montantdevise` | montant en devise |
| 18 | `Idevise` | `EUR` |

Contraintes à implémenter :
- **un fichier par exercice**, aucune fraction ;
- séparateur **tabulation** (le plus sûr) ou `|` ;
- nommage : `{SIREN}FEC{YYYYMMDD}` (date de clôture) ;
- numérotation **séquentielle et continue**, ordre chronologique ;
- **écritures figées** : généré uniquement sur un `fr_fiscal_period.status = 'CLOSED'` ;
- pas d'écriture non équilibrée : contrôle bloquant avant écriture du fichier ;
- encodage : UTF-8 (ou ASCII étendu), pas de BOM.

### 3.3 Annexe (art. 838-15 / 619-12)

Le fork doit produire automatiquement les éléments d'annexe exigés :

1. nombre et montant des crypto-actifs détenus, **par catégorie** ;
2. **modalités de détermination de la valeur vénale** : plateformes/sources utilisées,
   **horaire de valorisation** (d'où `value_date_policy` dans la doctrine) ;
3. méthode retenue : **PEPS ou CUMP** ;
4. dépréciations constatées ;
5. le cas échéant, motivations d'un transfert entre catégories.

Toutes ces informations sont **déjà en base** (provenance des prix + buckets + journal) :
il s'agit de les restituer, pas de les recalculer.

---

## 4. Contrat d'interface avec le logiciel comptable tiers

Trois modes possibles, du plus simple au plus intégré :

| Mode | Coût | Pour qui |
|---|---|---|
| **A. Export fichier** (journal CSV + FEC) | faible | cabinet qui importe manuellement |
| **B. Dépôt automatisé** (dossier surveillé, ou API REST du fork) | moyen | cabinet outillé |
| **C. Outil MCP** (`rotkehlchen/mcp/tools/french_ledger.py`) | faible | agent IA / copilote du cabinet |

Recommandation : livrer **A** en premier (c'est ce qui débloque la valeur), et **C** juste
après (le serveur MCP existe déjà, l'ajout d'un outil est marginal).

Endpoints REST à ajouter (convention existante : `ApiResult`, `OkResult`, `FailResult`) :

```
GET    /fr/periods                          → liste des exercices et statuts
PUT    /fr/periods                          → créer un exercice
POST   /fr/periods/{id}/close               → calculer clôture (inventaire + valeur vénale + dépréciation)
POST   /fr/periods/{id}/journal/generate    → générer les écritures
GET    /fr/periods/{id}/journal             → journal paginé (filtres réutilisés de db/filtering.py)
GET    /fr/periods/{id}/export/fec          → téléchargement FEC
GET    /fr/periods/{id}/export/journal      → téléchargement CSV pivot
GET    /fr/periods/{id}/annexe              → données d'annexe structurées
GET    /fr/inventory?date=…                 → inventaire (quantités + valeur) à une date
POST   /fr/doctrine                         → nouvelle version de doctrine (jamais d'écrasement)
```

---

## 5. Plan de mise en œuvre

| Lot | Contenu | Dépendances | Ordre de grandeur |
|---|---|---|---|
| **L0** | Neutralisations : `taxfree_after_period → None`, devise EUR, prix manquant **bloquant**, retrait freemium/analytics, embarquement des règles comptables FR | — | 1–2 j |
| **L1** | Modèle `fr_*` (migrations `v54→v55`) + doctrine + plan de comptes + buckets | L0 | 3–5 j |
| **L2** | Inventaire exposé (lots restants à une date, par bucket) | L1 | 3–5 j |
| **L3** | Génération du journal (partie double) + invariants d'équilibre | L1, L2 | 8–12 j |
| **L4** | Clôture : valeur vénale, comptes transitoires, dépréciation ligne à ligne | L3 | 5–8 j |
| **L5** | Exports CSV pivot + FEC + annexe | L3 | 4–6 j |
| **L6** | Verrouillage d'exercice et piste d'audit (scellement SHA-256, journal des modifications) | L3, L5 | 3 j |
| **L7** | UI : écrans exercice / journal / clôture + traduction FR des messages comptables | L3–L6 | 10–15 j |
| **L8** | Outil MCP + endpoints REST publics | L5 | 2–3 j |

Les fourchettes sont indicatives et supposent une personne connaissant le dépôt.
Les lots L1–L5 sont le chemin critique ; L7 peut être fait en parallèle.

---

## 6. Stratégie de test (à ne pas improviser)

Sur un outil comptable, la valeur est l'**exactitude reproductible**. Trois niveaux :

1. **Golden tests comptables** — un jeu de scénarios figés (achat, cession, échange
   crypto↔crypto, gas, staking, transfert interne, hack, clôture avec gain latent, clôture
   avec perte latente, dépréciation puis reprise) avec le **journal attendu ligne à ligne**.
   rotki a déjà cette culture : `rotkehlchen/tests/unit/accounting/test_basic.py`,
   `test_default_settings.py`.
2. **Invariants** (propriétés, exécutés sur des historiques générés) : équilibre
   débit/crédit, continuité de numérotation, conservation des quantités
   (`Σ entrées − Σ sorties == inventaire`), et **conservation du coût de revient** :
   `Σ coût des lots consommés + coût du stock final == Σ coûts d'entrée`.
3. **Tests de rejouabilité** : rejouer un exercice clos doit produire un FEC **bit à bit
   identique** (scellement SHA-256). C'est le meilleur test de non-régression du fork.

---

## 7. Décisions à faire trancher avant de coder

| # | Question | Impact |
|---|---|---|
| 1 | **Distribution** : desktop local, ou service hébergé pour le cabinet ? | Déclenche les obligations **AGPL-3.0** et le RGPD/DPA |
| 2 | **CUMP ou FIFO** par catégorie ? | choix doctrinal inscrit en annexe, irréversible en cours d'exercice |
| 3 | **Classement ANC** : compte 522 (valeur vénale) ou immobilisation incorporelle (art. 619-11) ? | change tout le modèle de clôture |
| 4 | **Politique d'horaire de valorisation** (clôture UTC, 23:59 CET, cours de fixing ?) | doit être documentée en annexe, donc figée par exercice |
| 5 | **Granularité des catégories homogènes** : par actif, par famille (BTC, ETH, stablecoins), par portefeuille ? | détermine les buckets et le plan de comptes |
| 6 | **Périmètre** : pré-comptabilité seule, ou génération du FEC (donc écritures figées) ? | le FEC exige le verrouillage d'exercice (L6) |
| 7 | **Traitement des frais** : activables au coût d'entrée ou charges ? | `include_fees_in_cost_basis` existe, mais son articulation avec la doctrine FR doit être validée |
| 8 | Qui **signe** la doctrine : le cabinet impose-t-il son plan de comptes ? | détermine si `fr_chart_of_accounts` est livré ou paramétré par client |

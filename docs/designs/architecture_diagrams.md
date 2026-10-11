# rotki en diagrammes

Vue pédagogique de l'architecture de rotki, destinée aux lecteurs non spécialistes.
Chaque diagramme montre **un plan différent** du même système. Les couleurs, les formes et les
traits ont la même signification d'un diagramme à l'autre (voir la légende ci-dessous).

Documents de référence détaillés : `architecture_overview.md` (architecture technique) et
`subsystems_evaluation.md` (évaluation des sous-systèmes).

---

## Légende

Lisez cette légende avant les diagrammes. Chaque couleur indique **dans quel langage ou dans quelle
couche** se trouve un élément. Chaque forme indique **son rôle**.

```mermaid
flowchart LR
    subgraph LEG ["Légende"]
        direction LR
        L1["Interface utilisateur<br/>(frontend, TypeScript / Vue)"]:::front
        L2["Logique métier<br/>(backend, Python)"]:::back
        L3["Service de performance ou de supervision<br/>(Rust)"]:::rust
        L4[("Données stockées sur le disque")]:::data
        L5{{"Service externe<br/>(blockchain, exchange, prix)"}}:::ext
        L6(["Personne"]):::user
        L7["Problème de conception<br/>(trait pointillé)"]:::back
    end
    L1 ~~~ L2 ~~~ L3 ~~~ L4 ~~~ L5 ~~~ L6 ~~~ L7

    classDef front fill:#dbeafe,stroke:#1d4ed8,color:#0f172a,stroke-width:2px
    classDef back fill:#dcfce7,stroke:#15803d,color:#0f172a,stroke-width:2px
    classDef rust fill:#ffedd5,stroke:#c2410c,color:#0f172a,stroke-width:2px
    classDef data fill:#f3e8ff,stroke:#7e22ce,color:#0f172a,stroke-width:2px
    classDef ext fill:#f1f5f9,stroke:#64748b,color:#0f172a,stroke-dasharray:4 3
    classDef user fill:#fef9c3,stroke:#a16207,color:#0f172a,stroke-width:2px
```

| Élément | Couleur | Forme | Signification |
|---------|---------|-------|---------------|
| Interface | bleu | rectangle | ce que l'utilisateur voit et manipule (écrit en TypeScript / Vue) |
| Logique métier | vert | rectangle | le « cerveau » de rotki (écrit en Python) |
| Performance / supervision | orange | rectangle | composants écrits en Rust |
| Données | violet | cylindre | informations stockées (bases de données) |
| Externe | gris | hexagone pointillé | services que rotki interroge mais ne contrôle pas |
| Utilisateur | jaune | pilule | la personne qui utilise rotki |
| Trait plein | — | flèche | « appelle » ou « envoie des données à » |
| Trait pointillé | — | flèche | dépendance **inattendue** (à corriger, voir plan 4) |

---

## Plan 1 : rotki en une image

Ce qu'est rotki, avec les acteurs principaux.

```mermaid
flowchart LR
    U(["Vous"]):::user
    subgraph APP ["rotki (sur votre ordinateur)"]
        direction LR
        UI["Écran de rotki<br/>(ce que vous voyez)"]:::front
        BR["Moteur de calcul<br/>(comptabilité, historique, soldes)"]:::back
        DB[("Vos données<br/>chiffrées localement")]:::data
    end
    subgraph WORLD ["Le monde extérieur"]
        direction TB
        BC{{"Blockchains<br/>Ethereum, Solana, Bitcoin…"}}:::ext
        EX{{"Plateformes d'échange<br/>Binance, Kraken…"}}:::ext
        PX{{"Sources de prix<br/>CoinGecko, CryptoCompare…"}}:::ext
    end
    U -->|"utilise"| UI
    UI -->|"demande et affiche"| BR
    BR -->|"lit et écrit"| DB
    BR -->|"lit les soldes et transactions"| BC
    BR -->|"lit l'historique"| EX
    BR -->|"lit les cours"| PX

    classDef front fill:#dbeafe,stroke:#1d4ed8,color:#0f172a,stroke-width:2px
    classDef back fill:#dcfce7,stroke:#15803d,color:#0f172a,stroke-width:2px
    classDef data fill:#f3e8ff,stroke:#7e22ce,color:#0f172a,stroke-width:2px
    classDef ext fill:#f1f5f9,stroke:#64748b,color:#0f172a,stroke-dasharray:4 3
    classDef user fill:#fef9c3,stroke:#a16207,color:#0f172a,stroke-width:2px
```

**À retenir** : rotki *lit* les informations de vos comptes et de la blockchain, puis calcule vos gains,
pertes et impôts localement. Vos données personnelles restent sur votre machine : rotki ne les envoie
pas à un serveur central, même s'il interroge les services ci-dessus.

---

## Plan 2 : quels langages, qui parle à qui

Chaque couleur est un langage ou une couche. Les flèches sont des modes de communication.

```mermaid
flowchart TB
    U(["Utilisateur"]):::user

    subgraph L_FRONT ["Langage TypeScript et Vue : l'interface"]
        direction LR
        F1["Pages<br/>(ce que vous voyez)"]:::front
        F2["Modules<br/>(logique d'écran)"]:::front
        F3["Client réseau<br/>(envoie les demandes)"]:::front
    end

    subgraph L_PY ["Langage Python : la logique métier"]
        direction LR
        P1["Interface de programmation<br/>(reçoit les demandes)"]:::back
        P2["Calculs<br/>(comptabilité, historique)"]:::back
        P3["Connexions externes<br/>(blockchains, exchanges)"]:::back
    end

    subgraph L_RUST ["Langage Rust : performance et supervision"]
        direction LR
        R1["Superviseur starling<br/>(démarre et surveille)"]:::rust
        R2["Colibri<br/>(icônes, prix, base locale)"]:::rust
        R3["Proxy<br/>(porte d'entrée unique)"]:::rust
    end

    subgraph L_DATA ["Stockage"]
        direction LR
        D1[("Base utilisateur<br/>chiffrée")]:::data
        D2[("Base commune<br/>des actifs")]:::data
    end

    U --> F1
    F1 --> F2 --> F3
    F3 -->|"HTTP : demande, réponse"| R3
    R3 -->|"transmet /api"| P1
    R3 -->|"transmet /colibri"| R2
    P1 --> P2 --> P3
    P2 --> D1
    R2 --> D1
    R2 --> D2
    P1 -.->|"WebSocket : avancement en direct"| F3
    R1 -.->|"démarre et surveille"| P1
    R1 -.->|"démarre et surveille"| R2

    classDef front fill:#dbeafe,stroke:#1d4ed8,color:#0f172a,stroke-width:2px
    classDef back fill:#dcfce7,stroke:#15803d,color:#0f172a,stroke-width:2px
    classDef rust fill:#ffedd5,stroke:#c2410c,color:#0f172a,stroke-width:2px
    classDef data fill:#f3e8ff,stroke:#7e22ce,color:#0f172a,stroke-width:2px
    classDef user fill:#fef9c3,stroke:#a16207,color:#0f172a,stroke-width:2px
```

**Lecture** : le bleu est tout ce qui s'affiche, le vert tout ce qui calcule, l'orange tout ce qui
démarre, protège ou accélère. Les flèches en pointillé sont des liens de surveillance ou de mise à jour
en direct, pas des demandes ponctuelles.

---

## Plan 3 : ce qui démarre quand vous ouvrez rotki

Quatre programmes collaborent. Le superviseur les lance dans le bon ordre et les arrête proprement.

```mermaid
flowchart TB
    E["Application de bureau<br/>(Electron) ou navigateur"]:::front
    S["Superviseur starling<br/>(chef d'orchestre)"]:::rust

    subgraph PROGS ["Les programmes lancés"]
        direction LR
        C["rotki-core<br/>le moteur Python"]:::back
        CO["colibri<br/>service Rust rapide"]:::rust
        M["serveur MCP<br/>pour assistants IA"]:::back
        X["proxy<br/>porte d'entrée unique"]:::rust
    end

    E -->|"lance et pilote"| S
    S -->|"démarre dans l'ordre"| C
    S -->|"démarre après le moteur"| CO
    S -->|"démarre si activé"| M
    S -->|"ouvre l'accès à l'application"| X
    X -->|"aiguille les demandes"| C
    X -->|"aiguille /colibri"| CO
    X -->|"aiguille /mcp"| M

    classDef front fill:#dbeafe,stroke:#1d4ed8,color:#0f172a,stroke-width:2px
    classDef back fill:#dcfce7,stroke:#15803d,color:#0f172a,stroke-width:2px
    classDef rust fill:#ffedd5,stroke:#c2410c,color:#0f172a,stroke-width:2px
```

**À retenir** : le superviseur évite qu'un programme reste allumé après la fermeture de l'application
et empêche d'ouvrir deux fois les mêmes données.

---

## Plan 4 : ensembles et sous-ensembles du moteur Python

Chaque grand bloc du moteur. Les flèches montrent qui utilise qui. Les **flèches pointillées marquées ⚠**
sont des dépendances à l'envers : une couche basse (calcul, base de données) dépend de l'interface
HTTP, ce qui n'est pas souhaité.

```mermaid
flowchart TB
    subgraph ENTREE ["Entrée : interface HTTP et temps réel"]
        API["api<br/>routes, schémas, services"]:::back
    end

    subgraph METIER ["Cœur métier : ce que rotki calcule"]
        direction LR
        ACC["accounting<br/>gains, pertes, impôts"]:::back
        HIS["history<br/>historique, prix"]:::back
        TSK["tasks<br/>travaux de fond"]:::back
        DATA["data_import<br/>imports de fichiers"]:::back
    end

    subgraph MONDE ["Connexions au monde extérieur"]
        direction LR
        CH["chain<br/>blockchains"]:::back
        EXC["exchanges<br/>plateformes d'échange"]:::back
        EXT["externalapis<br/>services de données"]:::back
    end

    subgraph STOCK ["Stockage"]
        direction LR
        DB[("db<br/>base utilisateur")]:::data
        GDB[("globaldb<br/>base commune")]:::data
    end

    API --> ACC
    API --> HIS
    API --> TSK
    API --> DATA
    ACC --> HIS
    HIS --> CH
    HIS --> EXC
    HIS --> EXT
    TSK --> CH
    ACC --> DB
    HIS --> DB
    CH --> GDB
    EXT --> GDB

    CH -.->|"⚠ dépendance inverse"| API
    DB -.->|"⚠ dépendance inverse"| API
    ACC -.->|"⚠ dépendance inverse"| API

    classDef back fill:#dcfce7,stroke:#15803d,color:#0f172a,stroke-width:2px
    classDef data fill:#f3e8ff,stroke:#7e22ce,color:#0f172a,stroke-width:2px
```

**Lecture** : on devrait pouvoir descendre seulement de haut en bas (de l'entrée vers le stockage).
Les trois flèches pointillées marquées ⚠ remontent vers l'entrée : c'est le principal point d'attention
technique identifié.

---

## Plan 5 : les blockchains, en poupées russes

Un ensemble contient des sous-ensembles. Les chaînes qui fonctionnent comme Ethereum partagent
la même logique de base (la « couche EVM »).

```mermaid
flowchart TB
    subgraph CHAIN ["chain : toutes les blockchains"]
        direction TB
        subgraph EVM ["Couche EVM : logique commune aux chaînes compatibles Ethereum"]
            direction LR
            ETH["Ethereum<br/>le réseau principal"]:::back
            L2A["Arbitrum"]:::back
            L2B["Base"]:::back
            L2C["Optimism"]:::back
            L2D["Gnosis, Polygon,<br/>Avalanche, Scroll…"]:::back
        end
        SOL["Solana"]:::back
        BTC["Bitcoin"]:::back
        OTH["Autres : Hyperliquid,<br/>Substrate, zkSync Lite"]:::back
    end
    ETH -.->|"fournit la logique de base"| EVM
    L2A --> EVM
    L2B --> EVM
    L2C --> EVM
    L2D --> EVM

    classDef back fill:#dcfce7,stroke:#15803d,color:#0f172a,stroke-width:2px
```

**Lecture** : ajouter une chaîne compatible Ethereum revient surtout à la brancher sur la couche EVM
commune. Solana et Bitcoin ont leur propre logique.

---

## Plan 6 : l'interface en sous-ensembles

La partie visible, du plus général (pages) au plus technique (socle commun).

```mermaid
flowchart TB
    subgraph PAGES ["Pages : ce que vous ouvrez"]
        direction LR
        P1["Soldes"]:::front
        P2["Historique"]:::front
        P3["Rapports fiscaux"]:::front
        P4["Comptes et adresses"]:::front
        P5["Paramètres"]:::front
    end
    subgraph MODULES ["Modules : la logique de chaque domaine"]
        direction LR
        M1["Soldes et actifs"]:::front
        M2["Transactions"]:::front
        M3["Rapports"]:::front
        M4["Comptes"]:::front
    end
    subgraph CORE ["Socle commun : utilisé partout"]
        direction LR
        K1["Client réseau<br/>et file d'attente"]:::front
        K2["Suivi des tâches<br/>en cours"]:::front
        K3["Notifications"]:::front
        K4["Tableaux et formulaires"]:::front
    end

    P1 --> M1
    P2 --> M2
    P3 --> M3
    P4 --> M4
    P5 --> K4
    M1 --> K1
    M2 --> K1
    M3 --> K2
    M4 --> K1
    M2 --> K3

    classDef front fill:#dbeafe,stroke:#1d4ed8,color:#0f172a,stroke-width:2px
```

**Lecture** : une page ne contient pas la logique : elle s'appuie sur un module, qui s'appuie sur le socle
commun. Les pages sont donc faciles à ajouter ou à modifier.

---

## Plan 7 : parcours d'une action (afficher ses soldes)

Ce qui se passe, étape par étape, quand vous ouvrez la page des soldes.

```mermaid
sequenceDiagram
    actor U as Vous
    participant UI as Écran (TypeScript)
    participant RX as Proxy (Rust)
    participant CORE as Moteur (Python)
    participant BC as Blockchains et plateformes
    participant DB as Vos données

    U->>UI: Ouvre la page Soldes
    UI->>RX: Demande les soldes (HTTP)
    RX->>CORE: Transmet la demande
    CORE->>DB: Lit vos comptes
    CORE->>BC: Interroge les soldes en ligne
    BC-->>CORE: Montants
    CORE->>CORE: Convertit en euros ou dollars
    CORE-->>RX: Résultat
    RX-->>UI: Résultat
    UI-->>U: Affiche vos soldes
    Note over CORE,UI: Pour les longs calculs, l'avancement arrive en direct (WebSocket)
```

---

## Plan 8 : sécurité simplifiée (la session)

Comment rotki sait que c'est bien vous, sans envoyer votre mot de passe à chaque action.

```mermaid
sequenceDiagram
    actor U as Vous
    participant UI as Écran
    participant CORE as Moteur Python
    participant CO as Colibri Rust

    U->>UI: Saisit votre mot de passe
    UI->>CORE: Authentification
    CORE->>CORE: Vérifie le mot de passe et crée un badge signé
    CORE-->>UI: Badge (cookie sécurisé)
    Note over UI: Le navigateur conserve le badge
    UI->>CORE: Action protégée + badge
    CORE->>CORE: Vérifie la signature et la durée
    UI->>CO: Action protégée + badge
    CO->>CO: Vérifie le même badge
    Note over CORE,CO: Une nouvelle connexion invalide les anciens badges
```

**Pourquoi deux vérifications ?** Le moteur et Colibri sont tous deux accessibles sur votre machine.
Chacun vérifie le badge, sans dépendre de l'autre.

---

## Plan 9 : état de santé des sous-systèmes

Évaluation qualitative issue de `subsystems_evaluation.md`. Vert : solide. Orange : à surveiller.
Rouge : dette prioritaire. La taille indique le volume de code de production.

```mermaid
flowchart LR
    subgraph BON ["Solide"]
        direction TB
        G1["accounting<br/>4 200 lignes<br/>bien testé"]:::good
        G2["concurrency<br/>363 lignes<br/>module propre"]:::good
        G3["exchanges<br/>18 000 lignes<br/>bien couvert"]:::good
        G4["Frontend<br/>1 243 fichiers de tests<br/>réussis"]:::good
        G5["colibri<br/>Rust, 303 tests"]:::good
    end
    subgraph SURV ["À surveiller"]
        direction TB
        O1["tasks<br/>7 600 lignes"]:::warn
        O2["externalapis<br/>12 000 lignes"]:::warn
        O3["globaldb<br/>8 000 lignes"]:::warn
        O4["mcp<br/>2 500 lignes, récent"]:::warn
    end
    subgraph DETTE ["Dette prioritaire"]
        direction TB
        R1["api<br/>25 000 lignes<br/>fichiers géants"]:::bad
        R2["chain<br/>104 000 lignes<br/>dépendances inverses"]:::bad
        R3["db<br/>30 000 lignes<br/>dbhandler 5 000 lignes"]:::bad
    end

    classDef good fill:#dcfce7,stroke:#15803d,color:#0f172a,stroke-width:2px
    classDef warn fill:#ffedd5,stroke:#c2410c,color:#0f172a,stroke-width:2px
    classDef bad fill:#fee2e2,stroke:#b91c1c,color:#0f172a,stroke-width:2px
```

**Limite** : les tests Python et Rust n'ont pas pu être exécutés dans cette session. Leur état
est donc « non vérifié », et non « bon » ni « mauvais ».

---

## Plan 10 : quelle est la part de chaque langage ?

Volume de code de production et de tests, en lignes.

```mermaid
pie showData
    title Code de production (≈ 480 000 lignes)
    "Python (moteur)" : 243859
    "TypeScript et Vue (interface)" : 215225
    "Rust (colibri et superviseur)" : 22596
```

```mermaid
pie showData
    title Code de tests (≈ 460 000 lignes)
    "Tests Python" : 246266
    "Tests TypeScript (interface)" : 212363
```

**Lecture** : rotki est autant une application d'interface qu'un moteur de calcul. Presque la moitié du code
est consacrée aux tests, ce qui est un bon signe de sérieux.

---

## Plan 11 : priorités de travail

Chaque action recommandée, selon son impact estimé et son effort. Positions indicatives, fondées sur
le jugement de l'évaluation.

```mermaid
quadrantChart
    title Priorités recommandées
    x-axis "Effort faible" --> "Effort élevé"
    y-axis "Impact faible" --> "Impact fort"
    quadrant-1 "Projets structurants"
    quadrant-2 "À faire en premier"
    quadrant-3 "Corrections rapides"
    quadrant-4 "Peu rentable"
    "1 Supprimer les dépendances inverses": [0.5, 0.9]
    "2 Corriger les commentaires périmés": [0.1, 0.3]
    "3 Découper les gros fichiers": [0.85, 0.8]
    "4 Réduire les exceptions de lint": [0.5, 0.4]
    "5 Test de cohérence Python et Rust": [0.5, 0.7]
    "6 Réduire les types génériques": [0.85, 0.5]
    "7 Planification déclarative des tâches": [0.5, 0.35]
```

---

## Plan 12 : ce que rotki permet de faire

```mermaid
mindmap
  root((rotki))
    Suivre
      Soldes sur blockchains
      Soldes sur plateformes
      Cours et valeurs
    Comprendre
      Historique des transactions
      Décodage des opérations
      Graphiques
    Calculer
      Plus ou moins-values
      Méthodes de coût FIFO et autres
      Rapports fiscaux
    Protéger
      Données chiffrées sur votre machine
      Aucun envoi vers un serveur rotki
      Sessions sécurisées
    Étendre
      Assistants IA via MCP
      Nouvelles chaînes et exchanges
```

---

## Glossaire rapide

| Terme | Explication simple |
|-------|--------------------|
| Blockchain | registre public des transactions (Ethereum, Bitcoin…) |
| Exchange | plateforme où l'on achète et vend des cryptos (Binance, Kraken…) |
| Frontend | la partie visible de l'application |
| Backend | la partie qui calcule, cachée derrière l'écran |
| API | la « porte » par laquelle l'écran demande des informations au moteur |
| WebSocket | canal qui envoie des nouvelles en direct (avancement d'un calcul) |
| Cookie de session | petit badge signé qui prouve que vous êtes connecté |
| Base chiffrée | fichier de données protégé par votre mot de passe |
| MCP | protocole permettant à un assistant IA d'utiliser rotki |
| Dépendance inverse | un composant bas qui dépend d'un composant haut (à éviter) |

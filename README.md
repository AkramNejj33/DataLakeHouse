# Spark Issue Triage Platform

## Description

Plateforme complète de triage automatique des incidents du projet Apache Spark, construite sur
une architecture en médaillon (Bronze → Argent → Or) hébergée sur Snowflake et orchestrée par dbt.
Le dataset source est le dump public JIRA Apache Spark de Kaggle (mars 2025, ~49 832 tickets SPARK).

Deux chemins de consommation sont exposés :

- **Chemin 1 — Inférence IA** : pipeline V6 Hybrid RCA basé sur des embeddings de phrases
  (`all-MiniLM-L6-v2`) et une recherche KNN cosinus (k=15). Prédit le type d'incident et la
  résolution probable d'un ticket en temps réel. Génère une analyse textuelle (via Anthropic
  claude-haiku si une clé API est configurée, sinon un template structuré).
- **Chemin 2 — Tableau de bord analytique** : dashboard Streamlit 5 pages explorant les
  volumes mensuels, la dynamique de résolution, la charge des assignataires et les liens entre tickets.

---

## Résultats obtenus

| Métrique | Valeur | Seuil cible |
|----------|--------|-------------|
| Accuracy issuetype | **75,32 %** | > 70 % |
| Accuracy résolution | **91,52 %** | > 75 % |
| Macro-F1 issuetype | 33,77 % | — |
| Macro-F1 résolution | 16,40 % | — |

> Le macro-F1 bas est attendu : le dataset est fortement déséquilibré (Bug + Improvement = 75 %
> des tickets). Les classes rares (Question, Documentation, Cannot Reproduce) ont peu de
> représentants en validation, ce qui tire la moyenne vers le bas. L'accuracy globale reste
> au-dessus du seuil cible.

**Jeu de données :**

| Table | Lignes |
|-------|--------|
| RAW.ISSUES | 1 149 321 |
| RAW.COMMENTS | 5 047 714 |
| RAW.CHANGELOG | 9 653 526 |
| RAW.ISSUELINKS | 390 063 |
| Tickets SPARK filtrés | 49 832 |
| MARTS_ML.MART_ML (train + val) | 42 083 |
| Tickets d'entraînement | 38 274 |
| Tickets de validation | 3 809 |

**Tests dbt :** PASS=44 WARN=2 ERROR=0 (les 2 warnings concernent les valeurs "Won't Fix"
dont l'apostrophe génère un warning SQL dans les tests `accepted_values`).

---

## Prérequis

- Python 3.12 (le dbt venv) + Python 3.11+ (scripts de chargement)
- Un compte Snowflake (trial ou payant — voir note ci-dessous)
- dbt-snowflake 1.11+ installé via `uv` dans `dbt_project/.venv/`
- Les 4 fichiers CSV dans `data/` (issues, comments, changelog, issuelinks)
- Docker Desktop (optionnel, pour lancer les apps via docker-compose)

> **Note Snowflake Cortex :** Les fonctions `SNOWFLAKE.CORTEX.COMPLETE` et
> `SNOWFLAKE.CORTEX.EMBED_TEXT_1024` sont bloquées sur les comptes trial. Le pipeline
> d'inférence utilise donc `sentence-transformers` en local (gratuit, ~80 MB).
> Les scripts `cortex/*.sql` sont fournis pour référence et fonctionneront sur un compte payant.

---

## Installation rapide (Docker)

La façon la plus simple de lancer les applications :

```bash
git clone https://github.com/Anas-elkhabbaz/DataLakeHouse_PFE.git
cd DataLakeHouse_PFE

# Configurer les identifiants Snowflake
cp .env.example .env
# Éditer .env avec vos valeurs SNOWFLAKE_*

# Lancer les deux applications
docker-compose up --build
```

- Application d'inférence : http://localhost:8501
- Tableau de bord analytique : http://localhost:8502

La première construction télécharge le modèle `all-MiniLM-L6-v2` (~80 MB). Les lancements
suivants démarrent en quelques secondes grâce au cache Docker.

---

## Installation manuelle

```bash
git clone https://github.com/Anas-elkhabbaz/DataLakeHouse_PFE.git
cd DataLakeHouse_PFE

# Installer les dépendances de l'app d'inférence
pip install -r apps/inference/requirements.txt

# Installer les dépendances du tableau de bord
pip install -r apps/analytics/requirements.txt

# Configurer les identifiants
cp .env.example .env
# Éditer .env

# Lancer les apps
streamlit run apps/inference/inference_app.py   # port 8501
streamlit run apps/analytics/analytics_app.py  # port 8502
```

---

## Configuration Snowflake (chargement initial des données)

### 1. Écrire le profil dbt

```bash
python load/write_profiles.py
```

Ce script lit `.env` et écrit `~/.dbt/profiles.yml` en gérant correctement l'encodage Unicode.

### 2. Créer la base de données, les schémas et le warehouse

```bash
python load/run_phase1.py
```

Crée : DATABASE `PFE_SPARK`, 6 schémas (RAW, STAGING, INTERMEDIATE, MARTS_ML,
MARTS_ANALYTICS, CORTEX), warehouse `PFE_WH`, stage interne `RAW.CSV_STAGE`.

### 3. Inspecter les en-têtes CSV

```bash
python load/inspect_headers.py
```

Affiche les positions exactes des colonnes dans chaque CSV. Vérifier que le mapping
dans `load/04_copy_into_raw.sql` correspond avant de passer à l'étape suivante.

### 4. Charger les CSV vers le stage puis dans les tables brutes

```bash
python load/03_put_files.py    # PUT vers @RAW.CSV_STAGE (~30 min selon la connexion)
python load/run_phase4.py      # COPY INTO + vérification des comptes
```

### 5. Pipeline dbt

```bash
cd dbt_project
.venv\Scripts\dbt deps         # Installer dbt-utils
.venv\Scripts\dbt seed         # Tables de mapping labels
.venv\Scripts\dbt run          # 11 modèles
.venv\Scripts\dbt test         # 46 tests (attendu : PASS=44, WARN=2)
```

### 6. Pipeline ML (embeddings + prédictions + évaluation)

```bash
python load/run_ml_pipeline.py
```

Génère les embeddings (`results/embeddings_cache.npz`, 57 MB), fait les prédictions
sur les 3 809 tickets de validation, évalue les performances et sauvegarde les résultats
dans `results/` et dans `CORTEX.MART_PREDICTIONS` sur Snowflake.

---

## Configuration optionnelle — Analyse LLM (Anthropic)

Pour activer l'analyse textuelle générée par IA dans l'application d'inférence,
ajouter la clé suivante dans `.env` :

```
ANTHROPIC_API_KEY=sk-ant-...
```

Obtenir une clé sur [console.anthropic.com](https://console.anthropic.com).
Le modèle utilisé est `claude-haiku-4-5` (~$0,25 / 1M tokens — coût négligeable par prédiction).
Sans cette clé, l'application génère un rapport structuré automatique.

---

## Structure du projet

```
DataLakeHouse_PFE/
├── .env.example                 # Template de configuration
├── .gitignore
├── .dockerignore
├── docker-compose.yml           # Lance inference + analytics
├── pyproject.toml
├── README.md
│
├── data/                        # CSVs source (non versionnés, ~8 GB total)
│   ├── issues.csv
│   ├── comments.csv
│   ├── changelog.csv
│   └── issuelinks.csv
│
├── load/                        # Scripts de chargement Bronze
│   ├── run_phase1.py            # Crée la base Snowflake
│   ├── inspect_headers.py       # Vérifie les positions CSV
│   ├── 03_put_files.py          # PUT vers le stage
│   ├── run_phase4.py            # COPY INTO + vérification
│   ├── run_ml_pipeline.py       # Embeddings + prédictions + évaluation
│   ├── write_profiles.py        # Génère ~/.dbt/profiles.yml
│   └── 04_copy_into_raw.sql     # DDL tables brutes + mapping $N
│
├── dbt_project/                 # Transformations Silver + Or
│   ├── models/
│   │   ├── staging/             # 4 vues (1:1 avec les sources)
│   │   ├── intermediate/        # 4 tables (NLP, features, split)
│   │   └── marts/
│   │       ├── ml/              # MART_ML (42 083 lignes)
│   │       └── analytics/       # MART_ANALYTICS_OPS + DEPS
│   ├── seeds/                   # issuetype_mapping.csv, resolution_mapping.csv
│   ├── macros/                  # clean_jira_text, generate_schema_name
│   └── .venv/                   # Python 3.12 venv pour dbt-snowflake
│
├── cortex/                      # Pipeline V6 Hybrid RCA (SQL Snowflake)
│   ├── 01_train_enriched.sql    # RCA via CORTEX.COMPLETE (compte payant requis)
│   ├── 02_train_embeddings.sql  # Embeddings via CORTEX.EMBED_TEXT_1024
│   ├── 03_predict.sql           # KNN + arbitrage LLM (194M comparaisons)
│   └── 04_evaluate.sql          # Accuracy, F1, matrice de confusion
│
├── apps/
│   ├── inference/               # Application de triage (Streamlit)
│   │   ├── Dockerfile
│   │   ├── inference_app.py     # UI professionnelle + KNN temps réel
│   │   └── requirements.txt
│   └── analytics/               # Tableau de bord 5 pages (Streamlit)
│       ├── Dockerfile
│       ├── analytics_app.py
│       ├── pages/
│       │   ├── 1_overview.py
│       │   ├── 2_resolution_dynamics.py
│       │   ├── 3_workload.py
│       │   └── 4_relationships.py
│       └── requirements.txt
│
├── results/                     # Artefacts de l'évaluation
│   ├── embeddings_cache.npz     # Embeddings pré-calculés (57 MB, versionné)
│   └── ...                      # Prédictions et métriques (non versionnées)
│
└── docs/
    ├── architecture.md
    ├── data_dictionary.md
    └── decisions_log.md
```

---

## Documentation

| Document | Contenu |
|----------|---------|
| [docs/architecture.md](docs/architecture.md) | Schéma medallion et description des couches |
| [docs/data_dictionary.md](docs/data_dictionary.md) | Description de chaque colonne de mart_ml |
| [docs/decisions_log.md](docs/decisions_log.md) | Choix architecturaux figés et leur justification |

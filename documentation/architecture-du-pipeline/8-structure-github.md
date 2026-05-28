# Structure du dépôt GitHub — Projet GoodAir

## Principe général

Le dépôt GitHub contient **tout ce qui se déploie et se versionne** : le code, la configuration, l'infrastructure et la documentation. Il ne contient jamais de données ni de secrets.

Ce qui vit ailleurs :

| Élément                        | Où                                  |
| ------------------------------ | ----------------------------------- |
| Données Bronze / Silver / Gold | ADLS Gen2                           |
| Référentiel des stations       | ADLS Gen2 `/reference/`             |
| Clés API, secrets, credentials | Azure Key Vault                     |
| State Terraform                | ADLS Gen2 (backend distant dédié)   |
| Dashboards Power BI            | Service Power BI (gestion manuelle) |

## Arborescence complète MVP

```text
goodair/
│
├── .github/
│   └── workflows/
│       ├── deploy-infra.yml
│       ├── deploy-functions.yml
│       ├── deploy-dbt.yml
│       ├── postgres-schedule.yml
│       └── quality-check.yml
│
├── infra/
│   ├── main.tf
│   ├── variables.tf
│   ├── outputs.tf
│   ├── backend.tf
│   └── modules/
│       ├── adls/
│       ├── functions/
│       ├── container-apps/
│       ├── postgresql/
│       ├── adf/
│       ├── keyvault/
│       ├── cognitive-search/
│       └── monitoring/
│
├── functions/
│   ├── extract_aqicn/
│   │   ├── __init__.py
│   │   └── function.json
│   ├── extract_openweather/
│   │   ├── __init__.py
│   │   └── function.json
│   ├── transform_bronze_to_silver/
│   │   ├── __init__.py
│   │   └── function.json
│   ├── shared/
│   │   ├── adls_client.py
│   │   ├── keyvault_client.py
│   │   ├── http_client.py
│   │   └── quality_checks.py
│   ├── requirements.txt
│   └── host.json
│
├── dbt/
│   ├── dbt_project.yml
│   ├── profiles.yml
│   ├── Dockerfile
│   ├── models/
│   │   ├── staging/
│   │   │   └── stg_air_quality.sql
│   │   └── gold/
│   │       ├── fact_air_quality.sql
│   │       ├── dim_station.sql
│   │       ├── dim_city.sql
│   │       ├── dim_time.sql
│   │       └── dim_pollutant.sql
│   ├── tests/
│   │   ├── assert_aqi_not_negative.sql
│   │   └── assert_no_duplicate_measurements.sql
│   └── macros/
│       └── generate_schema_name.sql
│
├── adf/
│   └── pipelines/
│       ├── pl_ingest_hourly.json
│       ├── pl_bronze_to_silver.json
│       └── pl_silver_to_gold.json
│
├── reference/
│   ├── bootstrap_stations.py
│   └── stations_schema.json
│
├── monitoring/
│   └── grafana/
│       ├── dashboards/
│       │   ├── pipeline_overview.json
│       │   └── data_quality.json
│       └── provisioning/
│           └── datasources.yml
│
├── tests/
│   ├── functions/
│   │   ├── test_extract_aqicn.py
│   │   ├── test_extract_openweather.py
│   │   └── test_transform_bronze_to_silver.py
│   └── integration/
│       └── test_pipeline_end_to_end.py
│
├── documentation/
│
├── .env.example
├── .gitignore
└── README.md
```

## Détail de chaque dossier

### `.github/workflows/`

Contient les pipelines GitHub Actions. Chaque fichier correspond à un périmètre précis.

| Fichier                 | Déclencheur          | Ce qu'il fait                                                          |
| ----------------------- | -------------------- | ---------------------------------------------------------------------- |
| `deploy-infra.yml`      | Push sur `main`      | Terraform plan + apply — provisionne toutes les ressources Azure       |
| `deploy-functions.yml`  | Push sur `main`      | Build et déploiement des Azure Functions (extraction + transformation)  |
| `deploy-dbt.yml`        | Push sur `main`      | Build de l'image Docker dbt Core et déploiement sur Azure Container Apps |
| `postgres-schedule.yml` | Cron 20h / 7h (lun–ven) | Stop/start automatique Azure Database for PostgreSQL pour réduire les coûts |
| `quality-check.yml`     | PR sur `develop`     | Tests unitaires, lint Python, validation des schémas dbt               |

Les secrets nécessaires aux workflows (credentials Azure, tenant ID, client ID) sont stockés dans les **GitHub Actions Secrets**, jamais dans le code.

### `infra/`

Code Terraform pour provisionner l'ensemble de l'infrastructure Azure.

| Fichier / dossier           | Contenu                                                           |
| --------------------------- | ----------------------------------------------------------------- |
| `main.tf`                   | Point d'entrée Terraform, appel des modules                       |
| `variables.tf`              | Variables paramétrables (région, noms, tailles)                   |
| `outputs.tf`                | Valeurs exportées après provisioning (URLs, IDs, connection strings) |
| `backend.tf`                | Configuration du backend distant (ADLS Gen2) pour le state        |
| `modules/adls/`             | Compte ADLS Gen2, containers, ACL                                 |
| `modules/functions/`        | Plan Azure Functions Consumption, application, identity managée   |
| `modules/container-apps/`   | Environnement Container Apps (dbt Core, Grafana)                  |
| `modules/postgresql/`       | Azure Database for PostgreSQL Flexible Server, base `goodair_gold` |
| `modules/adf/`              | Data Factory, linked services, triggers                           |
| `modules/keyvault/`         | Key Vault, politiques d'accès                                     |
| `modules/cognitive-search/` | Service Azure Cognitive Search, définition de l'index Bronze      |
| `modules/monitoring/`       | Container App Grafana, OpenTelemetry collector                    |

### `functions/`

Code Python des Azure Functions. Chaque fonction est isolée dans son propre dossier.

| Dossier                         | Rôle                                                                                                                                                         |
| ------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `extract_aqicn/`                | Lit les `station_id` actifs depuis `/reference/stations.json` sur ADLS, appelle `feed/@station_id` pour chaque station, écrit les JSON bruts sur ADLS Bronze |
| `extract_openweather/`          | Appelle l'API OpenWeatherMap pour chaque ville, écrit les JSON bruts sur ADLS Bronze                                                                         |
| `transform_bronze_to_silver/`   | Lit les JSON Bronze, parse et nettoie les données (`"-"` → NULL, normalisation timestamps, gestion polluants dynamiques), écrit en Parquet sur ADLS Silver    |
| `shared/adls_client.py`         | Client ADLS Gen2 réutilisable (authentification Managed Identity)                                                                                            |
| `shared/keyvault_client.py`     | Lecture des secrets depuis Key Vault                                                                                                                         |
| `shared/http_client.py`         | Client HTTP avec retry et gestion des quotas API                                                                                                             |
| `shared/quality_checks.py`      | Validation des données à l'entrée de la couche Silver (nullité, plages de valeurs, fraîcheur)                                                                |

La liste des `station_id` à collecter n'est **jamais codée en dur** dans les fonctions. Elle est lue dynamiquement depuis `/reference/stations.json` sur ADLS à chaque exécution.

### `dbt/`

Projet dbt Core pour les transformations Silver → Gold. dbt génère automatiquement la documentation, les tests de qualité et le graphe de lignage des données.

| Fichier / dossier                 | Rôle                                                                                        |
| --------------------------------- | ------------------------------------------------------------------------------------------- |
| `dbt_project.yml`                 | Configuration du projet dbt (nom, version, chemins des modèles)                             |
| `profiles.yml`                    | Connexion à Azure Database for PostgreSQL (lu depuis variables d'environnement)              |
| `Dockerfile`                      | Image Docker pour exécuter dbt Core dans Azure Container Apps                               |
| `models/staging/stg_air_quality.sql` | Vue de staging : sélection et typage des colonnes depuis la couche Silver                |
| `models/gold/fact_air_quality.sql`   | Table de faits : mesures horaires d'AQI par station, agrégations journalières            |
| `models/gold/dim_station.sql`     | Dimension stations : identifiant, nom, coordonnées                                          |
| `models/gold/dim_city.sql`        | Dimension villes : regroupement géographique des stations                                   |
| `models/gold/dim_time.sql`        | Dimension temps : décomposition heure / jour / semaine / mois                               |
| `models/gold/dim_pollutant.sql`   | Dimension polluants : liste des indicateurs mesurés (PM2.5, PM10, NO2, O3…)                 |
| `tests/`                          | Tests SQL personnalisés (AQI positif, absence de doublons sur la clé de fait)               |
| `macros/`                         | Macros SQL réutilisables (ex: génération dynamique des noms de schéma par environnement)    |

dbt s'exécute dans un **Azure Container App** déclenché par ADF après chaque transformation Bronze→Silver. Le container démarre, exécute `dbt run && dbt test`, puis s'arrête.

### `adf/`

Pipelines ADF exportés au format JSON et versionnés dans Git. ADF supporte nativement la synchronisation Git — les pipelines sont édités dans l'interface ADF et commités automatiquement dans ce dossier.

| Pipeline                   | Rôle                                                                                                                                                   |
| -------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `pl_ingest_hourly.json`    | Trigger horaire : déclenche en parallèle `extract_aqicn` et `extract_openweather`, attend la fin, enchaîne Bronze→Silver puis Silver→Gold               |
| `pl_bronze_to_silver.json` | Déclenche la Azure Function `transform_bronze_to_silver`, surveille l'exécution                                                                        |
| `pl_silver_to_gold.json`   | Déclenche le job dbt Core sur Azure Container Apps (`dbt run && dbt test`), surveille la fin avant de rendre la main                                   |

### `reference/`

Scripts de récupération du référentiel des stations. Ce dossier ne contient pas les données elles-mêmes (qui vivent sur ADLS) mais les scripts qui les produisent.

| Fichier                 | Rôle                                                                                                                                                                                    |
| ----------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `bootstrap_stations.py` | Appelle l'endpoint `search` de l'API WAQI, filtre les stations par critères géographiques et qualité, produit le fichier `stations.json` à uploader manuellement sur ADLS `/reference/` |
| `stations_schema.json`  | Schéma JSON attendu pour le fichier de référentiel (`station_id`, `status`, `city`, `coordinates`)                                                                                      |

Ce script est exécuté **une seule fois** au démarrage du projet, puis lors de mises à jour occasionnelles du référentiel des stations.

### `monitoring/`

Dashboards Grafana et configuration OpenTelemetry, versionnés dans Git.

| Fichier                                     | Rôle                                                                                            |
| ------------------------------------------- | ----------------------------------------------------------------------------------------------- |
| `grafana/dashboards/pipeline_overview.json` | Vue globale du pipeline : taux de succès, latence, dernière exécution par étape                 |
| `grafana/dashboards/data_quality.json`      | KPIs qualité : taux de nullité, doublons, fraîcheur, résultats des tests dbt                    |
| `grafana/provisioning/datasources.yml`      | Configuration des sources de données Grafana (Azure Monitor, PostgreSQL)                        |

Les dashboards Grafana étant des fichiers JSON, ils sont modifiables dans l'interface puis exportés et commités dans Git.

### `tests/`

Tests unitaires Python pour les Azure Functions, et tests d'intégration pour la pipeline complète.

| Dossier / fichier                              | Ce qui est testé                                                                                                       |
| ---------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------- |
| `tests/functions/test_extract_aqicn.py`        | Parsing des réponses API WAQI, gestion des erreurs (status != ok, timeout), construction des chemins ADLS              |
| `tests/functions/test_extract_openweather.py`  | Parsing des réponses API OpenWeather, gestion des cas limites                                                          |
| `tests/functions/test_transform_bronze_to_silver.py` | Nettoyage JSON (`"-"` → NULL), normalisation timestamps, déduplication, schéma Parquet attendu en sortie       |
| `tests/integration/test_pipeline_end_to_end.py` | Vérifie qu'un fichier JSON Bronze synthétique traverse correctement toute la pipeline jusqu'à PostgreSQL Gold         |

Les tests unitaires sont exécutés automatiquement par `quality-check.yml` à chaque Pull Request. Les tests dbt (`dbt test`) sont exécutés à chaque run Silver→Gold en production.

### `documentation/`

Documentation technique du projet, déjà structurée dans le dépôt.

### Fichiers racine

| Fichier        | Contenu                                                                                                                                     |
| -------------- | ------------------------------------------------------------------------------------------------------------------------------------------- |
| `.env.example` | Template des variables d'environnement nécessaires au développement local (noms des ressources Azure, région, connection string PostgreSQL) — jamais le `.env` réel |
| `.gitignore`   | Exclut `.env`, `__pycache__`, `.terraform/`, `dbt/target/`, `dbt/dbt_packages/`, fichiers de secrets                                       |
| `README.md`    | Présentation du projet, prérequis, guide de démarrage rapide                                                                                |

## Ce qui ne va jamais dans le dépôt

| Élément                                | Raison                                                   |
| -------------------------------------- | -------------------------------------------------------- |
| `.env`                                 | Contient des secrets                                     |
| `terraform.tfstate`                    | State Terraform — stocké sur ADLS Gen2 (backend distant) |
| `*.json` de données Bronze/Silver/Gold | Les données vivent sur ADLS, pas dans Git                |
| `dbt/target/`                          | Artefacts de compilation dbt — générés à l'exécution    |
| `dbt/dbt_packages/`                    | Dépendances dbt — installées via `dbt deps` au runtime  |
| Clés API, tokens, passwords            | Stockés dans Azure Key Vault                             |
| Fichiers `.pbix` Power BI              | Gérés manuellement via le service Power BI               |

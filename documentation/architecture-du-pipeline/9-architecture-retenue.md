# Architecture retenue — Synthèse des décisions GoodAir

Ce document résume l'ensemble des décisions architecturales prises au cours du projet GoodAir, les options écartées et les raisons de chaque choix. Il constitue la référence unique à destination du jury et de l'équipe.

---

## Contexte du projet

GoodAir est un projet de pipeline de données dont l'objectif est de collecter, transformer, stocker et restituer des données de qualité de l'air issues de deux APIs publiques : **AQICN** et **OpenWeatherMap**.

Le pipeline doit répondre aux exigences du Bloc 3 (collecte, data lake, data warehouse, ETL/ELT, sécurité) et du Bloc 5 (qualité des données, traçabilité, data visualisation, analyses statistiques) de la grille d'évaluation MSPR.

---

## Architecture finale retenue

```text
AQICN / OpenWeatherMap
          ↓
Azure Data Factory
(orchestration + déclenchement horaire)
          ↓
Azure Functions Python — Extraction
(appels API REST, écriture JSON sur ADLS Bronze)
          ↓
Azure Functions Python — Transformation Bronze → Silver
(parsing JSON, nettoyage, normalisation, écriture Parquet sur ADLS Silver)
          ↓
dbt Core sur Azure Container Apps — Transformation Silver → Gold
(agrégations, modèle en étoile, écriture dans PostgreSQL Gold)
          ↓
Azure Database for PostgreSQL — Flexible Server
(serving Gold, connecteur natif Power BI)
          ↓
Power BI
(dashboards, restitution métier)

────────────────────────────────────────────────────────────────
ADLS Gen2            (Data Lake — Reference / Bronze / Silver)
Azure Cognitive Search (moteur de recherche élastique — Bronze)
Microsoft Purview    (catalogue de données — gouvernance)
Azure Key Vault      (secrets et clés API)
Microsoft Entra ID   (identités, RBAC)
Grafana + OpenTelemetry (monitoring transversal — portable)
GitHub Actions       (CI/CD — portable)
Terraform            (IaC — multi-cloud)
```

---

## Décision 1 — Mode de traitement : Batch

**Choix retenu : traitement batch horaire**

**Options écartées :** streaming (Kafka, Spark Streaming, Azure Event Hubs)

**Raison :**

Les APIs AQICN et OpenWeatherMap ne fournissent pas de flux temps réel. Elles sont interrogeables uniquement par requêtes HTTP, avec des quotas et une fréquence de mise à jour de l'ordre de l'heure. Un traitement streaming aurait introduit une complexité technique élevée (Kafka, Event Hubs, connecteurs) sans aucun bénéfice fonctionnel pour les cas d'usage métier — analyses de tendances, rapports, comparaisons géographiques — qui n'exigent pas de latence inférieure à l'heure.

Le batch présente en outre l'avantage d'une architecture simple, robuste, maîtrisable et démontrable dans le cadre d'une évaluation pédagogique.

---

## Décision 2 — Stack technique : Azure directement en MVP

**Choix retenu : Azure dès le MVP**

**Option écartée :** stack open source locale (Airflow + MinIO + PostgreSQL + dbt + Metabase)

**Raison :**

Un benchmark initial a identifié les composants nécessaires au pipeline sur une stack open source locale (voir `3-benchmark-du-pipeline.md`). Cette analyse a servi de base de référence, mais l'équipe a décidé de passer directement à Azure pour trois raisons principales :

- **Disponibilité d'un abonnement Azure** dans le cadre du projet ;
- **Exigence de démonstration industrielle** : une architecture cloud managée est plus représentative d'une architecture de production qu'une stack locale ;
- **Élimination de la dette technique** : passer d'une stack locale à Azure après coup est une migration coûteuse. Partir directement sur Azure évite cette duplication.

La correspondance entre les composants open source analysés et les composants Azure initialement retenus est la suivante :

| Stack open source (référence) | Stack Azure (MVP initial) |
| ----------------------------- | ------------------------- |
| Apache Airflow                | Azure Data Factory        |
| Scripts Python                | Azure Functions (Python)  |
| MinIO                         | ADLS Gen2                 |
| Python (Bronze→Silver)        | Azure Databricks          |
| dbt + PostgreSQL (Gold)       | Azure Synapse SQL         |
| Metabase / Superset           | Power BI                  |
| Variables d'environnement     | Azure Key Vault           |
| Logs Airflow                  | Azure Monitor             |

---

## Décision 3 — Orchestration : Azure Data Factory

**Choix retenu : Azure Data Factory (ADF)**

**Options écartées :** Apache Airflow sur VM/AKS, Synapse Pipelines

**Raison :**

ADF est le service d'orchestration managé Azure. Il déclenche les Azure Functions d'extraction, enchaîne les transformations Bronze→Silver et Silver→Gold, et assure la supervision du pipeline sans code d'infrastructure. La facturation est à l'exécution (pas de ressource permanente). Son intégration native avec Azure Functions, Container Apps, ADLS Gen2 et Key Vault évite toute friction d'interopérabilité.

Apache Airflow aurait apporté plus de flexibilité (DAGs Python, portabilité) mais au prix d'une charge d'exploitation significative (maintenance de la VM ou du cluster AKS, mises à jour, sécurité).

---

## Décision 4 — Extraction des APIs : Azure Functions Python

**Choix retenu : Azure Functions (Python)**

**Options écartées :** Databricks notebooks, ADF Web Activity, Container App dédié

**Raison :**

L'extraction se résume à des appels HTTP REST et à l'écriture de JSON sur ADLS Gen2. Azure Functions en plan Consumption est idéal pour ce cas d'usage : serverless, facturation uniquement à l'exécution, déclenchement natif par ADF, logs centralisés dans Azure Monitor. Python est le langage adapté pour parser du JSON, appliquer des règles de validation et gérer les quotas API.

---

## Décision 5 — Data Lake : Azure Data Lake Storage Gen2

**Choix retenu : ADLS Gen2**

**Options écartées :** Azure Blob Storage, MinIO local, Amazon S3

**Raison :**

ADLS Gen2 est conçu pour les architectures data lake. Il combine le stockage objet d'Azure Blob Storage avec un système de fichiers hiérarchique, des ACL fines au niveau dossier/fichier, et une intégration native avec tous les services Azure de traitement (Databricks, Container Apps, ADF, Purview). La résidence des données en région France Centre garantit la conformité RGPD.

L'arborescence retenue :

```text
/reference/   — référentiel des stations (ID, ville, coordonnées)
/bronze/      — JSON bruts partitionnés par source/année/mois/jour/heure
/silver/      — Parquet nettoyés et normalisés
```

---

## Décision 6 — Transformation Bronze → Silver : Azure Functions Python

**Choix retenu : Azure Functions Python**

**Options écartées (successivement) :** Azure Databricks, dbt Core

**Raison :**

Cette transformation consiste à parser des JSON hétérogènes et partiellement invalides (valeurs `"-"`, champs dynamiques selon les polluants disponibles, timestamps non normalisés). C'est un problème de **code Python**, pas de SQL. Databricks (Spark) a d'abord été retenu pour cette tâche mais s'est révélé architecturalement et financièrement surdimensionné : le volume traité est de l'ordre de quelques Mo par heure, soit moins d'1 Go par mois — bien en dessous du seuil où Spark apporte une valeur réelle.

dbt a été écarté pour ce segment car il est orienté transformations SQL déclaratives, pas parsing de JSON bruts complexes.

Azure Functions Python résout ce problème proprement : même langage que l'extraction, timeout suffisant (10 min en plan Consumption, illimité en plan Premium), facturation à l'exécution.

Traitements réalisés :
- parsing des fichiers JSON AQICN et OpenWeatherMap ;
- nettoyage : `"-"` → `NULL`, gestion des champs absents ;
- normalisation des timestamps en UTC ;
- gestion des polluants dynamiques (champs variables selon les stations) ;
- déduplication et contrôles de cohérence ;
- écriture en Parquet sur ADLS Silver.

---

## Décision 7 — Transformation Silver → Gold : dbt Core sur Azure Container Apps

**Choix retenu : dbt Core sur Azure Container Apps**

**Options écartées (successivement) :** Azure Databricks, Azure Functions Python seules

**Raison :**

La transformation Silver→Gold est fondamentalement du **SQL analytique** : jointures entre tables de faits et dimensions, agrégations temporelles (moyennes horaires, journalières), construction du modèle en étoile. C'est exactement le domaine pour lequel dbt a été conçu.

Databricks a été écarté (voir décision 6) pour les mêmes raisons de coût et de surdimensionnement.

Azure Functions seules ont été étudiées mais écartées : écrire des transformations analytiques en Python impératif (pandas + psycopg2) est plus verbeux, moins maintenable, et perd les bénéfices clés de dbt : tests de qualité automatiques (`dbt test`), documentation générée (`dbt docs`), graphe de lignage des données.

dbt Core est open source, s'installe via `pip install dbt-postgres`, et s'exécute dans un container Docker minimal. Le Container App démarre à la demande d'ADF, exécute `dbt run && dbt test`, et s'arrête. Coût : quelques secondes de compute par run.

Modèle en étoile construit par dbt :

```text
fact_air_quality — mesures horaires d'AQI par station
dim_station      — référentiel des stations
dim_city         — regroupement géographique
dim_time         — décomposition temporelle (heure/jour/semaine/mois)
dim_pollutant    — liste des indicateurs mesurés
```

---

## Décision 8 — Serving Gold : Azure Database for PostgreSQL Flexible Server

**Choix retenu : Azure Database for PostgreSQL Flexible Server (B1ms)**

**Options écartées successivement :** Azure Synapse Serverless SQL, Azure SQL Database Serverless

**Raison :**

Azure Synapse Serverless SQL a d'abord été retenu comme serving layer. Il a été abandonné pour deux raisons : sa facturation à $5/To scanné devient imprévisible avec des refreshs Power BI fréquents ou des requêtes non filtrées, et son couplage fort à l'écosystème Databricks/Delta le rendait redondant une fois Databricks supprimé.

Azure SQL Database Serverless a été envisagé comme alternative, mais écarté au profit de PostgreSQL pour trois raisons :
- **Open source et portable** : PostgreSQL fonctionne à l'identique sur AWS RDS, GCP Cloud SQL, ou en local, sans modification du code dbt ni des connexions Power BI ;
- **Connecteur dbt natif** : `dbt-postgres` est le connecteur de référence de l'écosystème dbt, le plus mature et le mieux documenté ;
- **Cohérence avec le projet** : PostgreSQL était déjà retenu dans le benchmark open source initial (fichier 3), ce qui assure une continuité logique et pédagogique.

Le coût est maîtrisé via un stop/start automatique piloté par GitHub Actions (arrêt à 20h, démarrage à 7h en semaine), ramenant la facture mensuelle à **€3–6/mois** pour un tier B1ms.

---

## Décision 9 — Moteur de recherche élastique : Azure Cognitive Search

**Choix retenu : Azure Cognitive Search**

**Options écartées :** Elasticsearch autohébergé, Elastic Cloud sur Azure

**Raison :**

La grille d'évaluation MSPR demande explicitement un **moteur de recherche élastique de données non structurées ou semi-structurées**. Azure Cognitive Search répond à cette exigence en indexant directement les fichiers JSON bruts stockés dans ADLS Gen2 (couche Bronze), sans duplication ni déplacement de données.

Il permet la recherche full-text sur les noms de stations, de villes, les polluants et les attributions de sources — cas d'usage qui n'est pas couvert par le SQL analytique de dbt/PostgreSQL, qui opère sur des données déjà structurées.

Elasticsearch autohébergé aurait été plus portable mais aurait introduit une charge d'exploitation significative (déploiement, mises à jour, sécurité) sans gain fonctionnel pour le projet.

Positionnement dans le Data Lake :

| Besoin | Moteur | Couche |
| --- | --- | --- |
| Requêtes analytiques SQL structurées | dbt Core + PostgreSQL | Gold |
| Recherche full-text, exploration par station ou polluant | Azure Cognitive Search | Bronze |

---

## Décision 10 — Catalogue de données : Microsoft Purview

**Choix retenu : Microsoft Purview**

**Raison :**

Purview est le service de gouvernance des données d'Azure. Il assure le lignage automatique des données (traçabilité de la source jusqu'au tableau Power BI), la classification des données sensibles et la gestion du catalogue structuré. Il répond à l'exigence MSPR de catalogue de données structurées, en complément d'Azure Cognitive Search qui couvre les données semi-structurées.

---

## Décision 11 — Data visualisation : Power BI

**Choix retenu : Power BI**

**Options écartées :** Metabase, Apache Superset, Grafana

**Raison :**

Power BI dispose d'un connecteur natif PostgreSQL et s'authentifie via Microsoft Entra ID. Il est le standard en entreprise pour la restitution BI et répond directement à l'exigence MSPR d'outil d'affichage de données. Metabase et Superset ont été analysés dans le benchmark initial mais écartés car moins démonstratifs dans un contexte d'évaluation professionnelle. Grafana est conservé pour le **monitoring technique** (supervision du pipeline), pas pour la restitution métier.

---

## Décision 12 — Monitoring : Grafana + OpenTelemetry

**Choix retenu : Grafana + OpenTelemetry**

**Option écartée :** Azure Monitor + Log Analytics

**Raison :**

Azure Monitor et Log Analytics sont entièrement propriétaires Microsoft (requêtes KQL non portables, dashboards non exportables). À la demande du professeur encadrant, l'architecture adopte une solution de monitoring portable pour limiter le lock-in sur les composants transversaux.

OpenTelemetry est un standard ouvert CNCF adopté par tous les fournisseurs cloud — les traces et métriques émises par les Azure Functions et les Container Apps dbt sont collectées indépendamment d'Azure. Grafana est déployé sur Azure Container Apps et se connecte aux sources existantes (Azure Monitor, PostgreSQL) via ses connecteurs natifs. Les dashboards et règles d'alerte sont des fichiers JSON/YAML versionnés dans Git.

---

## Décision 13 — CI/CD : GitHub Actions

**Choix retenu : GitHub Actions**

**Option écartée :** Azure DevOps

**Raison :**

Azure DevOps est propriétaire Microsoft. GitHub Actions est indépendant du cloud cible — les workflows YAML fonctionnent quel que soit le provider. Des actions officielles existent pour déployer sur Azure (ADF, Terraform, Azure Functions, Container Apps) sans dépendre d'Azure DevOps. GitHub Actions gère également le stop/start automatique de PostgreSQL via des crons programmés.

---

## Décision 14 — IaC : Terraform

**Choix retenu : Terraform**

**Options écartées :** ARM, Bicep

**Raison :**

ARM et Bicep sont des langages de templating Azure-only. Terraform est multi-cloud : les mêmes patterns fonctionnent sur Azure, AWS et GCP. Il supporte nativement des modules officiels pour tous les services Azure utilisés dans le projet. `terraform plan` permet de visualiser les changements avant application, et `terraform state` permet le rollback d'infrastructure. Les workspaces Terraform gèrent nativement la séparation dev/test/prod.

---

## Décision 15 — Sécurité : Azure Key Vault + Microsoft Entra ID

**Choix retenu : Azure Key Vault + Microsoft Entra ID (RBAC)**

**Raison :**

Aucun secret (clé API, chaîne de connexion) ne figure dans le code ni dans le dépôt Git. Toutes les clés API sont stockées dans Key Vault et lues dynamiquement par les Azure Functions via Managed Identity — sans aucune credential en clair. Microsoft Entra ID assure le contrôle d'accès par rôle (RBAC) sur l'ensemble des ressources Azure, conformément au principe du moindre privilège exigé par le RGPD. La résidence des données en région France Centre est garantie par la configuration Azure.

---

## Composants abandonnés en cours de projet et raisons

| Composant | Raison de l'abandon | Remplacé par |
| --- | --- | --- |
| **Azure Databricks** | Coût de cluster Spark disproportionné pour quelques Mo/h ; moteur surdimensionné pour le volume réel | Azure Functions Python (Bronze→Silver) + dbt Core (Silver→Gold) |
| **Azure Synapse Serverless SQL** | Facturation imprévisible à $5/To scanné ; couplage fort au format Delta — redondant sans Databricks | Azure Database for PostgreSQL |
| **Azure Monitor + Log Analytics** | Propriétaire Microsoft, requêtes KQL non portables, lock-in sur les composants transversaux | Grafana + OpenTelemetry |
| **Stack open source locale** | Décision de passer directement à Azure pour le MVP (cf. Décision 2) | Architecture Azure complète |

---

## Synthèse des composants retenus

| Brique | Composant retenu | Statut lock-in |
| --- | --- | --- |
| Orchestration | Azure Data Factory | Lock-in accepté — remplaçable par Airflow |
| Extraction APIs | Azure Functions (Python) | Faible — logique Python standard |
| Data Lake | ADLS Gen2 | Lock-in accepté — format Parquet portable |
| Transformation Bronze→Silver | Azure Functions (Python) | Faible — logique Python standard |
| Transformation Silver→Gold | dbt Core sur Azure Container Apps | Faible — open source, multi-cloud |
| Serving Gold | Azure Database for PostgreSQL | Faible — open source, portable |
| Moteur de recherche élastique | Azure Cognitive Search | Lock-in accepté — remplaçable par Elasticsearch |
| Catalogue de données | Microsoft Purview | Lock-in accepté — requis MSPR |
| Data visualisation | Power BI | Lock-in accepté — connecteur PostgreSQL natif |
| Monitoring | Grafana + OpenTelemetry | Aucun — open source, standard CNCF |
| CI/CD | GitHub Actions | Aucun — indépendant du cloud |
| IaC | Terraform | Aucun — multi-cloud |
| Sécurité secrets | Azure Key Vault | Lock-in accepté — remplaçable par HashiCorp Vault |
| Identités et RBAC | Microsoft Entra ID | Lock-in accepté — standard entreprise |

---

## Couverture de la grille d'évaluation MSPR

| Exigence MSPR | Composant couvrant | Statut |
| --- | --- | --- |
| Serveur de données du Data Lake | ADLS Gen2 | ✅ |
| Catalogue de données structurées | Microsoft Purview | ✅ |
| Moteur de requêtes de données | dbt Core + PostgreSQL | ✅ |
| Moteur de recherche élastique | Azure Cognitive Search | ✅ |
| Outil d'affichage de données | Power BI | ✅ |
| Pipeline ETL/ELT | ADF + Azure Functions + dbt Core | ✅ |
| Data Lake (Bronze/Silver/Gold) | ADLS Gen2 | ✅ |
| Data Warehouse (modèle en étoile) | PostgreSQL — 5 tables | ✅ |
| Qualité des données | Azure Functions (checks Python) + dbt tests | ✅ |
| Traçabilité / Data Lineage | dbt docs + Purview + `collected_at/observed_at` | ✅ |
| Sécurité et RGPD | Key Vault + Entra ID + France Centre | ✅ |
| IaC et reproductibilité | Terraform | ✅ |
| CI/CD | GitHub Actions | ✅ |
| Monitoring et alerting | Grafana + OpenTelemetry | ✅ |

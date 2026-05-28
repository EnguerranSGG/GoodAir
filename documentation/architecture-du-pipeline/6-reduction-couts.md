# Recommandation — Remplacement d'Azure Databricks et Azure Synapse Analytics

## Contexte

L'architecture GoodAir retenue dans `4-benchmark-pipeline-azure.md` repose sur deux composants dont les coûts se sont révélés disproportionnés par rapport au volume de données réel du projet :

- **Azure Databricks** : facturation à l'heure de cluster (DBU), déclenchée même pour des jobs courts. Sur un pipeline horaire avec des données de l'ordre de quelques Mo, le coût de démarrage du cluster représente la majorité de la facture.
- **Azure Synapse Analytics Serverless SQL** : facturation à $5/To scanné. Avec des refreshs Power BI fréquents ou des requêtes non optimisées sur des fichiers Delta, la facture peut devenir significative.

Ce document propose des alternatives concrètes, classées par ordre de priorité selon le rapport coût/complexité/conformité MSPR.

---

## Analyse du problème par composant

### Pourquoi Databricks est cher pour GoodAir

Le projet collecte des données horaires pour un nombre limité de stations françaises. Le volume réel est de l'ordre de **quelques Mo par heure**, soit moins d'1 Go par mois. Spark (le moteur sous-jacent de Databricks) est conçu pour des volumes distribués de plusieurs dizaines de Go minimum. Utiliser Databricks pour traiter quelques fichiers JSON est architecturalement surdimensionné.

### Pourquoi Synapse Serverless SQL peut coûter cher

Synapse Serverless SQL facture au volume de données **scannées**. Si les tables Delta ne sont pas correctement partitionnées ou si Power BI déclenche des requêtes larges sans filtre, chaque requête peut scanner l'intégralité des fichiers Parquet. Sur un mois avec des refreshs fréquents, cela s'accumule.

---

## Option A — Recommandation principale : Azure Functions + dbt Core + Azure Database for PostgreSQL

C'est l'option la plus économique. Elle reste conforme aux exigences MSPR, conserve le pattern Bronze/Silver/Gold et réduit le lock-in en remplaçant les services propriétaires par des alternatives open source.

### Architecture révisée

```text
AQICN / OpenWeatherMap
          ↓
Azure Data Factory (orchestration — inchangé)
          ↓
Azure Functions Python — Extraction
(appels REST, écriture Bronze ADLS Gen2 — inchangé)
          ↓
Azure Functions Python — Transformation Bronze → Silver
(parsing JSON, nettoyage, normalisation, écriture Parquet Silver ADLS Gen2)
          ↓
dbt Core sur Azure Container Apps — Transformation Silver → Gold
(agrégations, modèle en étoile, écriture dans Azure Database for PostgreSQL)
          ↓
Azure Database for PostgreSQL — Flexible Server
(serving Gold, connecteur natif Power BI)
          ↓
Power BI (inchangé)
```

### Détail des remplacements

#### Azure Databricks → Azure Functions Python + dbt Core

| Tâche           | Ancien composant     | Nouveau composant           | Justification                                              |
| --------------- | -------------------- | --------------------------- | ---------------------------------------------------------- |
| Bronze → Silver | Databricks Spark job | Azure Functions Python      | Volume faible, Python pur suffit largement, pas de cluster |
| Silver → Gold   | Databricks Spark job | dbt Core sur Container Apps | SQL analytique, open source, versionnable, testable        |

**Azure Functions pour Bronze → Silver**

Les transformations techniques (parsing JSON, nettoyage, normalisation) ne nécessitent pas Spark. Elles sont réalisables en Python pur avec `pandas` ou même sans dépendance externe. Azure Functions supporte un timeout de 10 minutes (plan Consumption) à illimité (plan Premium), suffisant pour traiter quelques Mo de JSON par heure.

```python
# Exemple — ce qui était un job Spark devient une Function Python
import json, pandas as pd
from azure.storage.filedatalake import DataLakeServiceClient

def transform_bronze_to_silver(bronze_json: dict) -> pd.DataFrame:
    records = []
    for station_id, data in bronze_json.items():
        aqi = data.get("aqi")
        aqi = None if aqi == "-" else aqi
        records.append({
            "station_id":   station_id,
            "aqi":          aqi,
            "observed_at":  data.get("time", {}).get("iso"),
            "collected_at": pd.Timestamp.utcnow().isoformat(),
        })
    return pd.DataFrame(records)
```

**dbt Core pour Silver → Gold**

dbt Core est open source et s'exécute dans n'importe quel environnement Python. Il se connecte à Azure Database for PostgreSQL via le connecteur `dbt-postgres` — le connecteur de référence de dbt, le plus mature et le mieux documenté de l'écosystème.

```yaml
# profiles.yml — connexion dbt → Azure Database for PostgreSQL
goodair:
  target: prod
  outputs:
    prod:
      type: postgres
      host: goodair.postgres.database.azure.com
      port: 5432
      dbname: goodair_gold
      schema: public
      user: "{{ env_var('DBT_USER') }}"
      password: "{{ env_var('DBT_PASSWORD') }}"
      sslmode: require
```

dbt Core s'exécute dans un **Azure Container App** déclenché par ADF après la transformation Bronze→Silver. Le container démarre, exécute les modèles dbt, puis s'arrête. Facturation uniquement pendant l'exécution (~quelques secondes par run).

#### Azure Synapse Serverless SQL → Azure Database for PostgreSQL Flexible Server

Azure Database for PostgreSQL Flexible Server est un service managé PostgreSQL open source. Contrairement à Azure SQL Database (propriétaire T-SQL), PostgreSQL est un moteur libre et portable.

| Caractéristique           | Azure Synapse Serverless SQL   | Azure Database for PostgreSQL Flexible Server  |
| ------------------------- | ------------------------------ | ---------------------------------------------- |
| Modèle de facturation     | $5/To scanné                   | ~€12–15/mois (B1ms) avec stop/start automatisé |
| Moteur                    | Propriétaire Microsoft         | PostgreSQL — open source                       |
| Lock-in                   | Élevé                          | Faible — portable sur n'importe quelle infra   |
| Pause automatique         | Non                            | Non natif — stop/start via GitHub Actions      |
| Connecteur Power BI       | Natif                          | Natif                                          |
| Connecteur dbt            | `dbt-sqlserver` (moins mature) | `dbt-postgres` (connecteur de référence dbt)   |
| SQL standard              | T-SQL (dialecte Microsoft)     | PostgreSQL standard                            |
| Continuité avec le projet | Rupture                        | Cohérent avec le benchmark initial (fichier 3) |

**Paramétrage recommandé pour GoodAir :**

```text
Tier          : Burstable — B1ms (1 vCore, 2 Go RAM)
Stockage      : 32 Go (largement suffisant pour la couche Gold)
Région        : France Centre
Version       : PostgreSQL 16
Arrêt auto    : via GitHub Actions (cron stop 20h, start 8h en semaine)
```

**Automatisation du stop/start pour réduire les coûts :**

```yaml
# .github/workflows/postgres-schedule.yml
name: PostgreSQL stop/start
on:
  schedule:
    - cron: "0 20 * * 1-5" # stop à 20h du lundi au vendredi
    - cron: "0 7  * * 1-5" # start à 7h du lundi au vendredi

jobs:
  manage-postgres:
    runs-on: ubuntu-latest
    steps:
      - uses: azure/cli@v1
        with:
          inlineScript: |
            if [ "${{ github.event.schedule }}" = "0 20 * * 1-5" ]; then
              az postgres flexible-server stop \
                --resource-group goodair-rg --name goodair-postgres
            else
              az postgres flexible-server start \
                --resource-group goodair-rg --name goodair-postgres
            fi
```

Avec ce stop/start automatisé (12h actif par jour en semaine), le coût mensuel réel descend à **€3–6/mois**.

La couche Gold est chargée dans PostgreSQL par dbt Core après chaque transformation Silver→Gold. Power BI se connecte via son connecteur PostgreSQL natif.

### Estimation de coût mensuelle

| Composant                                            | Avant                                                     | Après           |
| ---------------------------------------------------- | --------------------------------------------------------- | --------------- |
| Azure Databricks                                     | ~€150–300/mois (cluster Standard_DS3_v2, 1h/jour minimum) | €0 (supprimé)   |
| Azure Synapse Serverless                             | Variable selon scans                                      | €0 (supprimé)   |
| Azure Functions (ajout transformation Bronze→Silver) | Inclus (plan Consumption existant)                        | ~€0–2/mois      |
| Azure Container Apps (dbt Core Silver→Gold)          | €0                                                        | ~€1–5/mois      |
| Azure Database for PostgreSQL B1ms (avec stop/start) | €0                                                        | ~€3–6/mois      |
| **Total estimé**                                     | **€150–300+/mois**                                        | **~€5–15/mois** |

## Impact sur la conformité MSPR

| Exigence MSPR          | Avant                  | Après (Option A)                         |
| ---------------------- | ---------------------- | ---------------------------------------- |
| Pipeline ETL/ELT       | Databricks             | Azure Functions + dbt Core ✅            |
| Data Lake              | ADLS Gen2              | ADLS Gen2 — inchangé ✅                  |
| Data Warehouse         | Synapse Serverless SQL | Azure Database for PostgreSQL ✅         |
| Modèle en étoile       | Oui                    | Oui — dbt construit le même modèle ✅    |
| Transformations Python | Databricks PySpark     | Azure Functions Python ✅                |
| Qualité des données    | Databricks + Delta     | Azure Functions + dbt tests ✅           |
| Data Lineage           | Delta time travel      | dbt docs + `collected_at/observed_at` ✅ |

Aucune exigence de la grille d'évaluation n'est perdue dans cette migration.

## Composants inchangés

Les composants suivants ne sont pas affectés par cette recommandation :

- Azure Data Factory (orchestration)
- Azure Functions Python (extraction)
- Azure Data Lake Storage Gen2 (stockage Bronze/Silver)
- Azure Cognitive Search (moteur de recherche élastique)
- Microsoft Purview (catalogue)
- Azure Key Vault + Microsoft Entra ID (sécurité)
- Grafana + OpenTelemetry (monitoring)
- GitHub Actions + Terraform (CI/CD et IaC)
- Power BI (data visualisation)

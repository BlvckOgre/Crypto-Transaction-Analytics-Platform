# Crypto Transaction Analytics Platform

An end-to-end data engineering project built during the **Data Engineering Zoomcamp 2025 Cohort**. It ingests cryptocurrency data (market trades + blockchain transactions) into a cloud-native pipeline, transforms it, and presents it through a dashboard for insights.

---

## Table of Contents

- [Problem Statement](#problem-statement)
- [Disclaimer](#disclaimer)
- [Data Sources](#data-sources)
- [Architecture Overview](#architecture-overview)
- [Infrastructure: Terraform](#infrastructure-terraform)
- [Orchestration: Kestra](#orchestration-kestra)
- [Transformation: PySpark](#transformation-pyspark)
- [Visualization: Looker Studio](#visualization-looker-studio)
- [Conclusion](#conclusion)

---

## Problem Statement
To better understand cryptocurrency behavior, this project investigates the relationship between **on-chain blockchain activity** and **market trade prices**. The aim is to explore whether blockchain transactions (similar to company sales) can be linked to market price movements.

## Disclaimer ⚠️
- The full dataset is ~4GB and covers 3 months with ~32 million rows.
- PySpark backfilling runs daily jobs for each day — resulting in **~90 jobs** and **potential costs** if reproduced.
- Follow the project steps in order for best results:
  > Terraform → Kestra → PySpark → Dashboard

---

## Data Sources 📊

- **Market Trade Prices**: [Bitcoin Historical Data (Binance, 2018–2025)](https://www.kaggle.com/datasets/novandraanugrah/bitcoin-historical-datasets-2018-2024/)
- **Blockchain Transactions**: [Google BigQuery: `crypto_bitcoin`](https://console.cloud.google.com/bigquery?p=bigquery-public-data&d=crypto_bitcoin&page=dataset)

---

## Architecture Overview 🛠️

![Pipeline Diagram](img/pipeline-diagram.png)

The system includes:
- **Terraform**: Infra provisioning on GCP
- **Kestra**: Job orchestration & scheduling
- **PySpark**: ETL and schema modeling
- **Looker Studio**: Dashboard visualization

---

## Infrastructure: Terraform

Terraform provisions all required GCP services:

### Resources Created
1. **Service Account** with BigQuery and GCS permissions
2. **IAM Roles** for data processing and job submission
3. **Firewall Rule** for Kestra VM access (IP locked)
4. **Compute VM Instance** labeled `kestra-vm` for orchestration
5. **GCS Bucket** for data lake and script storage
6. **BigQuery Dataset**: includes `Fact_Transactions` and `Dim_MarketPrice`

📌 *Graph generation*: `terraform graph | dot -Tpng > graph.png`

![Terraform Graph](img/tf-graph.png)

### Setup Instructions
```bash
# 1. Authenticate
$ gcloud init
$ gcloud auth application-default login

# 2. Plan & Apply
$ terraform plan
$ terraform apply
```

---

## Orchestration: Kestra

Kestra orchestrates daily DAGs at **3AM EST** to trigger extraction, transformation, and loading steps.

### VM Setup
1. SSH into `kestra-vm`
2. Install Docker & Docker Compose
3. Upload `docker-compose.yml` (edit `<YOUR PASSWORD>`)
4. Start Kestra using:
```bash
docker-compose up -d
```

Paste [etl-pipeline-trigger.yaml](src/flows/etl-pipeline-trigger.yaml) in Kestra UI under namespace: `de-zoomacamp-crypto`

### DAG Tasks Breakdown
- **Ingest Kaggle Market Data** (daily → Parquet → GCS)
- **Ingest Blockchain Transactions** (BQ export → GCS)
- **Submit PySpark Job** (Dataproc Serverless → BigQuery)

![GCS Screenshot](img/gcs.png)
![Kestra DAG View](img/etl-pipeline-trigger1.png)
![Kestra DAG View 2](img/etl-pipeline-trigger2.png)

### Secrets (KV Store)
Define the following KV pairs:
- `GCP_PROJECT_ID`
- `GCP_DATASET`
- `GCP_BUCKET_NAME`

![KV Store](img/kv-store.png)

### Backfill Setup ⏳
Backfill from **Jan 1, 2025** to today (approx. 8 hrs for 90 days). Include a 2-min delay between jobs to avoid GCP quotas.

---

## Transformation: PySpark

Used for ETL tasks. Jobs run on **Dataproc Serverless** with minimal overhead.

### Job Parameters
```bash
--input_date=YYYY-MM-DD
--projectid={{ kv('GCP_PROJECT_ID') }}
--bq_dataset={{ kv('GCP_DATASET') }}
--bucket={{ kv('GCP_BUCKET_NAME') }}
--frequency=15m|1h|4h|1d
```

### Partitioning & Clustering
- `Fact_Transactions` → Partitioned on `block_timestamp_month`
- `Dim_MarketPrice` → Partitioned on `timestamp_month`
- Both clustered on `cryptocurrency`

![Schema - Fact](img/schema_fact.png)
![Schema - Dim](img/schema_dim.png)

---

## Visualization: Looker Studio 📈

Dashboard showcases both **temporal** and **categorical** data using:
- Distribution of transaction types
- Price/volume trends across time

🔗 [View Dashboard](https://lookerstudio.google.com/reporting/674f8151-8f82-4d0e-8ef9-bc6f342bfd4c)

![Dashboard](img/dashboard.png)

---

## Conclusion ✅

This project explored the relationship between blockchain activity and trading activity:

1. **Count-based Metrics**: No strong correlation between trade and transaction counts.
2. **Volume-based Metrics**: Stronger correlation observed in total amounts (spikes and dips occur concurrently).

> Future enhancements may include real-time ingestion, multiple coins, and anomaly detection.

---

Feel free to clone this repo, follow the instructions, and tweak it for your use case. Contributions are welcome!


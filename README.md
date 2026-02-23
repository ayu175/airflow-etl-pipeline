# Airflow ETL Pipeline

An automated data pipeline built with Apache Airflow that stages raw event and song data from AWS S3, loads it into a Redshift data warehouse, and runs automated data quality checks — all orchestrated through a custom DAG with four purpose-built operators.

> This project was completed as part of the [Udacity Data Engineering Nanodegree](https://www.udacity.com/course/data-engineer-nanodegree--nd027). The starter template provided the SQL helper class, operators, and DAG configuration.

## Pipeline Overview

```
S3 (Raw JSON) → Stage to Redshift → Load Fact Table → Load Dimension Tables → Data Quality Checks
```

Raw data arrives as JSON files in S3 in two forms: **log data** (user activity events) and **song data** (track and artist metadata). The pipeline stages both into Redshift, transforms them into a star schema, and validates the output before the run is marked complete.

## DAG Configuration

The DAG (`final_project.py`) is configured with the following defaults:

- No dependencies on past runs
- Tasks retry 3 times on failure, with a 5-minute interval between retries
- Catchup disabled
- No email on retry

## Custom Operators

Four operators were built in the `plugins/operators/` directory:

**`stage_redshift.py` — Stage Operator**
Loads JSON-formatted files from S3 into Redshift staging tables using a dynamic `COPY` statement. Supports timestamped S3 paths for backfill runs.

**`load_fact.py` — Fact Table Operator**
Executes SQL transformations from the helper class to populate the fact table. Configured for append-only inserts to support incremental loads.

**`load_dimension.py` — Dimension Table Operator**
Loads dimension tables using a truncate-insert pattern, clearing the table before each load to ensure data freshness. Insert-only mode is also supported via a parameter.

**`data_quality.py` — Data Quality Operator**
Runs SQL-based test cases against loaded tables and compares results to expected values. Raises an exception and triggers task retry if any check fails, ensuring pipeline integrity before the run completes.

## Project Structure

```
airflow-etl-pipeline/
├── dags/
│   ├── final_project.py        # Main DAG with task definitions and dependencies
│   └── create_tables.py        # DAG to initialize staging and warehouse tables
├── plugins/
│   ├── operators/
│   │   ├── stage_redshift.py   # Stages S3 JSON data to Redshift
│   │   ├── load_fact.py        # Loads fact table from staging
│   │   ├── load_dimension.py   # Loads dimension tables from staging
│   │   └── data_quality.py     # Runs data quality checks
│   └── helpers/
│       └── sql_queries.py      # SQL transformation queries
├── create_tables.sql            # DDL for staging and analytics tables
├── docker-compose.yaml          # Spins up Airflow locally via Docker
└── README.md
```

## Tools & Technologies

Apache Airflow · AWS S3 · AWS Redshift · Python · SQL · Docker

## How to Run

### Prerequisites
- [Docker Desktop](https://www.docker.com/products/docker-desktop/) installed
- AWS account with an active Redshift cluster and S3 access

### 1. Start Airflow
```bash
git clone https://github.com/ayu175/airflow-etl-pipeline.git
cd airflow-etl-pipeline
docker-compose up -d
```
Navigate to `http://localhost:8080` — use `airflow` / `airflow` to log in.

### 2. Configure Connections
In the Airflow UI, go to **Admin > Connections** and add:
- `aws_credentials` — your AWS Access Key ID and Secret Access Key
- `redshift` — your Redshift cluster host, database, username, and password

### 3. Initialize Tables
Trigger the `create_tables` DAG first to set up all staging and analytics tables in Redshift.

### 4. Run the Pipeline
Trigger the `final_project` DAG. Monitor task progress in the Graph View — all tasks should complete green, with the data quality checks confirming successful loads.

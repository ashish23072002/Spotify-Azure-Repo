# Spotify-Azure-Repo


🥉 Bronze Layer — Raw Data Ingestion (Azure Data Factory)
Overview

The Bronze layer is responsible for ingesting raw data from the source application database (Azure SQL Database) into the Azure Data Lake Storage Gen2.

This layer focuses on data ingestion only, without applying transformations or business logic. The data is stored in its original form so that it can be processed later in the Silver and Gold layers.

The Bronze layer implements a metadata-driven incremental ingestion pipeline using Azure Data Factory (ADF).

Architecture Position

The Bronze layer is the first stage of the Medallion Architecture used in this project.

Application Database (Azure SQL)
        │
        ▼
Azure Data Factory (Ingestion Pipeline)
        │
        ▼
Azure Data Lake Storage Gen2
        │
        └── Bronze Layer (Raw Data)
Goals of the Bronze Layer

The Bronze layer is designed to:

Extract data from the source database

Load raw data into the Data Lake

Capture only incremental changes

Store ingestion checkpoints

Maintain raw data history

Support scalable ingestion for multiple tables

Key Technologies Used
Technology	Purpose
Azure Data Factory	Orchestrates ingestion pipeline
Azure SQL Database	Source application database
Azure Data Lake Storage Gen2	Raw data storage
Parquet Format	Efficient columnar storage
Dynamic SQL	Table-agnostic ingestion
Metadata-Driven Pipeline	Scalable ingestion logic
Metadata-Driven Ingestion

Instead of building separate pipelines for each table, this project uses a metadata-driven approach.

A pipeline parameter named loop_input defines the tables to ingest.

Example Parameter Configuration
[
{"schema":"dbo","table":"DimUser","cdc_col":"updated_at","from_date":""},
{"schema":"dbo","table":"DimTrack","cdc_col":"updated_at","from_date":""},
{"schema":"dbo","table":"DimDate","cdc_col":"date","from_date":""},
{"schema":"dbo","table":"DimArtist","cdc_col":"updated_at","from_date":""},
{"schema":"dbo","table":"FactStream","cdc_col":"stream_timestamp","from_date":""}
]
What Each Field Means
Field	Description
schema	Database schema
table	Table name
cdc_col	Column used for incremental loading
from_date	Optional override for backfilling

This metadata allows a single pipeline to ingest multiple tables dynamically.

Pipeline Execution Flow

The Bronze pipeline runs the following steps:

ForEach Table
      │
      ▼
Lookup last processed timestamp
      │
      ▼
Extract incremental data
      │
      ▼
Load data to Data Lake
      │
      ▼
Update checkpoint file
Step-by-Step Pipeline Explanation
1. ForEach Loop

The pipeline iterates through each table defined in the loop_input parameter.

Example iterations:

Iteration 1 → DimUser
Iteration 2 → DimTrack
Iteration 3 → DimDate
Iteration 4 → DimArtist
Iteration 5 → FactStream

This enables the pipeline to process multiple tables in a single execution.

2. Lookup Activity — Retrieve Last Processed Timestamp

The Lookup activity (last_cdc) retrieves the latest processed timestamp from a checkpoint file stored in the Data Lake.

Example checkpoint file:

bronze/DimUser_cdc/cdc.json

Example content:

{
 "cdc": "2024-03-01T10:00:00"
}

This value determines where the next ingestion should start.

3. Incremental Data Extraction

The pipeline dynamically generates SQL queries based on metadata.

Example dynamic query:

SELECT *
FROM dbo.DimUser
WHERE updated_at > '2024-03-01T10:00:00'

This ensures that only new or updated records are extracted.

Benefits:

Prevents reloading historical data

Improves pipeline performance

Reduces compute cost

4. Copy Activity — Load Data to Data Lake

The Copy activity (AzureSQLToLake) transfers the extracted data to the Bronze layer in the Data Lake.

Destination configuration:

Container: bronze
Folder: <table_name>
File: <table_name>_<timestamp>.parquet

Example output:

bronze/DimUser/DimUser_20250311.parquet
bronze/DimTrack/DimTrack_20250311.parquet

Data is stored in Parquet format for efficient storage and analytics.

5. Data Validation

After copying data, the pipeline checks whether any rows were ingested.

IF rows_copied > 0

If no new records are detected, the pipeline removes the empty output file to keep the Data Lake clean.

6. Update CDC Checkpoint

If new data is ingested, the pipeline calculates the latest timestamp using:

SELECT MAX(cdc_column)
FROM source_table

Example:

SELECT MAX(updated_at)
FROM dbo.DimUser

This value is stored in the checkpoint file.

Example:

bronze/DimUser_cdc/cdc.json

Updated content:

{
 "cdc": "2025-03-11T09:30:00"
}

This checkpoint will be used in the next pipeline run.

Bronze Data Lake Structure

After ingestion, the Data Lake contains:

bronze/
   ├── DimUser/
   │      DimUser_20250311.parquet
   │
   ├── DimUser_cdc/
   │      cdc.json
   │
   ├── DimTrack/
   │      DimTrack_20250311.parquet
   │
   ├── DimTrack_cdc/
   │      cdc.json
   │
   ├── DimDate/
   ├── DimArtist/
   └── FactStream/

Each table maintains its own checkpoint file for incremental ingestion.

Why the Bronze Layer Is Important

The Bronze layer provides:

Raw data storage

Data lineage

Historical ingestion records

Incremental loading capability

Reliable data foundation for downstream processing

Without the Bronze layer, downstream transformations could become unreliable or inconsistent.

Bronze Layer Output

The output of this layer is raw parquet datasets stored in the Data Lake.

These datasets are later consumed by the Silver layer (Databricks transformations).

Bronze Layer Output
        │
        ▼
Databricks Silver Layer
Key Features Implemented

✔ Metadata-driven ingestion
✔ Incremental loading (CDC based)
✔ Dynamic SQL generation
✔ Data Lake checkpointing
✔ Scalable pipeline design

Next Layer

The data stored in the Bronze layer is processed in the Silver layer, where it is cleaned and standardized using Databricks and PySpark.

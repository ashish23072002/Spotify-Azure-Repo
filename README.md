# Spotify-Azure-Repo

# 🥉 Bronze Layer — Raw Data Ingestion (Azure Data Factory)

## 📌 Overview

The **Bronze layer** is responsible for **ingesting raw data** from the **source application database (Azure SQL Database)** into **Azure Data Lake Storage Gen2**.

This layer focuses purely on **data ingestion**, without applying **transformations or business logic**.  
The data is stored in its **original raw format** so it can later be processed in the **Silver** and **Gold** layers.

The Bronze layer implements a **metadata-driven incremental ingestion pipeline** using **Azure Data Factory (ADF)**.

---

# 🏗️ Architecture Position

The **Bronze layer** is the **first stage of the Medallion Architecture** used in this project.
Application Database (Azure SQL)
│
▼
Azure Data Factory (Ingestion Pipeline)
│
▼
Azure Data Lake Storage Gen2
│
└── Bronze Layer (Raw Data)




---

# 🎯 Goals of the Bronze Layer

The Bronze layer is designed to:

- Extract data from the **source database**
- Load **raw data into the Data Lake**
- Capture **incremental changes only**
- Store **ingestion checkpoints**
- Maintain **raw data history**
- Support **scalable ingestion for multiple tables**

---

# 🛠️ Key Technologies Used

| Technology | Purpose |
|-------------|---------|
| **Azure Data Factory** | Orchestrates ingestion pipeline |
| **Azure SQL Database** | Source application database |
| **Azure Data Lake Storage Gen2** | Raw data storage |
| **Parquet Format** | Efficient columnar storage |
| **Dynamic SQL** | Table-agnostic ingestion |
| **Metadata-Driven Pipeline** | Scalable ingestion logic |

---

# ⚙️ Metadata-Driven Ingestion

Instead of building **separate pipelines for each table**, this project uses a **metadata-driven approach**.

A pipeline parameter called **`loop_input`** defines the tables to ingest.

## Example Parameter Configuration

```json
[
{"schema":"dbo","table":"DimUser","cdc_col":"updated_at","from_date":""},
{"schema":"dbo","table":"DimTrack","cdc_col":"updated_at","from_date":""},
{"schema":"dbo","table":"DimDate","cdc_col":"date","from_date":""},
{"schema":"dbo","table":"DimArtist","cdc_col":"updated_at","from_date":""},
{"schema":"dbo","table":"FactStream","cdc_col":"stream_timestamp","from_date":""}
]
```

## 📘 What Each Field Means

| Field | Description |
|------|-------------|
| **schema** | Database schema |
| **table** | Table name |
| **cdc_col** | Column used for incremental loading |
| **from_date** | Optional override for backfilling |

This metadata allows **a single pipeline** to ingest **multiple tables dynamically**.

---
## 🔄 Pipeline Execution Flow

The **Bronze pipeline** executes the following process:

```text
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
```
## 🧠 Step-by-Step Pipeline Explanation

---

### 1️⃣ ForEach Loop

The pipeline iterates through each table defined in the **`loop_input` parameter**.

#### Example Iterations

```text
Iteration 1 → DimUser
Iteration 2 → DimTrack
Iteration 3 → DimDate
Iteration 4 → DimArtist
Iteration 5 → FactStream
```

### 2️⃣ Lookup Activity — Retrieve Last Processed Timestamp

The **Lookup activity (`last_cdc`)** retrieves the **latest processed timestamp** from a **checkpoint file** stored in the **Data Lake**.

#### Example Checkpoint File

```text
bronze/DimUser_cdc/cdc.json
```
```json
{
  "cdc": "2024-03-01T10:00:00"
}
```

### 3️⃣ Incremental Data Extraction

The pipeline dynamically generates **SQL queries** based on metadata.

#### Example Dynamic Query

```sql
SELECT *
FROM dbo.DimUser
WHERE updated_at > '2024-03-01T10:00:00'
```

### 4️⃣ Copy Activity — Load Data to Data Lake

The **Copy activity (`AzureSQLToLake`)** transfers the extracted data into the **Bronze layer** of the **Data Lake**.

#### Destination Configuration

| Parameter | Value |
|-----------|------|
| **Container** | `bronze` |
| **Folder** | `<table_name>` |
| **File** | `<table_name>_<timestamp>.parquet` |

#### Example Output

```text
bronze/DimUser/DimUser_20250311.parquet
bronze/DimTrack/DimTrack_20250311.parquet
```

### 5️⃣ Data Validation

After copying data, the pipeline verifies whether **any rows were ingested**.

#### Validation Logic

```sql
IF rows_copied > 0
```

### 6️⃣ Update CDC Checkpoint

If new data is ingested, the pipeline calculates the **latest timestamp** using the following SQL query.

#### Generic Query

```sql
SELECT MAX(cdc_column)
FROM source_table
```

## 🗂️ Bronze Data Lake Structure

After ingestion, the **Data Lake structure** looks like this:

```text
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
```

## ⭐ Why the Bronze Layer Is Important

The **Bronze layer** provides the foundational raw data storage for the entire data platform.

### Key Benefits

- **Raw Data Storage**  
  Stores the **original source data** exactly as received from the application database.

- **Data Lineage**  
  Enables tracking of **where data originated** and how it moves through the pipeline.

- **Historical Ingestion Records**  
  Maintains historical raw files for **traceability and auditing**.

- **Incremental Loading Capability**  
  Uses **CDC checkpoints** to ensure only **new or updated records** are ingested.

- **Reliable Data Foundation**  
  Provides a **stable input layer** for downstream transformations.

Without the **Bronze layer**, downstream transformations in **Silver and Gold layers** could become **unreliable, inconsistent, or difficult to debug**.

---

## 📦 Bronze Layer Output

The **output of the Bronze layer** is a collection of **raw Parquet datasets** stored in **Azure Data Lake Storage Gen2**.

These datasets represent the **exact data extracted from the source system** with **no transformations applied**.

The raw datasets are then consumed by the **Silver Layer**, where data cleaning, validation, and standardization take place using **Databricks and PySpark**.

### Data Flow to the Next Layer

```text
Bronze Layer Output
        │
        ▼
Databricks Silver Layer
```

## 🔜 Next Layer

The data stored in the **Bronze layer** is processed in the **Silver layer**.

In the **Silver layer**, the raw datasets are:

- **Cleaned**
- **Validated**
- **Transformed**
- **Standardized**

These transformations are performed using **Databricks** and **PySpark**.

The goal of the **Silver layer** is to convert **raw Bronze data** into **structured and reliable datasets** that are ready for analytics and downstream consumption.

---

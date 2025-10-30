# End-to-End Azure Data Engineering Project (AdventureWorks)

This repository contains the complete code and configuration for an end-to-end data engineering project built on the Microsoft Azure platform.
The project follows a modern data warehouse (Medallion) architecture to ingest, process, transform, and serve the **AdventureWorks** sample dataset.

This project was built following the "Azure End-To-End Data Engineering Project (From Scratch!)" video by Ansh Lamba.

## 🏛️ Project Architecture

This project implements a **Medallion Architecture**, which is a best-practice pattern for organizing data in a data lake into three distinct layers:

* **Bronze (Raw Data):** The data is ingested from the source system and landed in its raw, unchanged format.
* **Silver (Transformed Data):** The raw data is cleaned, conformed, and enriched. This layer contains validated, queryable data.
* **Gold (Curated Data):** The silver-layer data is aggregated and modeled into business-centric tables (e.g., external tables or views) ready for analytics and reporting.

### Data Flow Diagram

[Source API (GitHub)] --> [Azure Data Factory (Dynamic Pipeline)] --> [ADLS Gen2 (Bronze)] --> [Azure Databricks (PySpark)]
--> [ADLS Gen2 (Silver - Parquet)] --> [Azure Synapse Analytics (Serverless SQL)] --> [ADLS Gen2 (Gold - Parquet)] --> [Power BI]


---

## 🛠️ Technologies Used

* **Data Ingestion:** Azure Data Factory (ADF)
* **Data Lake:** Azure Data Lake Storage (ADLS) Gen2
* **Data Transformation:** Azure Databricks (using PySpark)
* **Data Warehousing & Serving:** Azure Synapse Analytics (Serverless SQL Pool)
* **Security:** Azure Key Vault, Service Principals, and Managed Identity
* **Reporting:** Power BI

---

## Phase 1: Ingestion (Bronze Layer)

* **Tool:** Azure Data Factory
* **Source:** AdventureWorks CSV files hosted on GitHub, accessed via an **HTTP Linked Service**.
* **Key Feature:** This pipeline is **dynamic and metadata-driven**. Instead of building a separate pipeline for each table, this project uses:
    1.  A **Lookup Activity** to read a `get.json` configuration file. This file contains a list of all tables to be ingested (e.g., `calendar.csv`, `customers.csv`, etc.).
    2.  A **ForEach Activity** to loop over the array of tables from the Lookup.
    3.  Inside the loop, a **Copy Data Activity** dynamically copies each table from the source (GitHub) to the `bronze` container in ADLS Gen2, creating a separate folder for each table.

## Phase 2: Transformation (Silver Layer)

* **Tool:** Azure Databricks
* **Process:**
    1.  **Secure Connection:** The Databricks notebook connects to ADLS Gen2 securely using an **App Registration (Service Principal)** with credentials stored in Azure Key Vault.
    2.  **Read Data:** All raw CSV files are read from the `bronze` container into PySpark DataFrames.
    3.  **Transformations:** Various data cleaning and enrichment tasks are performed, including:
        * **Calendar:** Extracting `month` and `year` from the date column.
        * **Customers:** Concatenating `prefix`, `first_name`, and `last_name` into a `full_name` column using `concat_ws`.
        * **Products:** Using the `split` function to parse product codes from SKUs and extract the first word from product names.
        * **Sales:** Converting data types (e.g., to timestamp), replacing characters using `regexp_replace`, and performing column-based arithmetic.
    4.  **Analysis:** The notebook demonstrates performing `groupBy` aggregations and creating visualizations directly within Databricks to analyze the transformed data (e.g., total orders by day).
    5.  **Write Data:** The final, transformed DataFrames are written to the `silver` container in the efficient **Parquet format**.

## Phase 3: Serving (Gold Layer)

* **Tool:** Azure Synapse Analytics (Serverless SQL Pool)
* **Process:**
    1.  **Secure Connection:** Synapse connects to ADLS Gen2 using its **System-Managed Identity**, which has been granted "Storage Blob Data Contributor" permissions. This is a best-practice, password-less approach.
    2.  **Create Logical Warehouse:**
        * A new **Serverless SQL Database** (e.g., `aw_database`) is created.
        * A `gold` schema is created to organize the production-ready tables.
        * **SQL Views** are created using the `OPENROWSET` function to read the Silver Parquet files directly from the data lake. This provides a relational database layer over the data lake without moving or duplicating data.
    3.  **Materialize Data (CETAS):**
        * The project demonstrates using **`CREATE EXTERNAL TABLE AS SELECT` (CETAS)** to run a transformation (e.g., `SELECT * FROM gold.sales_view`) and physically save the results as a new Parquet file in the `gold` container. This external table (`gold.external_sales`) is now optimized for reporting.

## Phase 4: Visualization (Reporting)

* **Tool:** Power BI
* **Process:**
    1.  **Connect to Synapse:** Power BI connects directly to the **Serverless SQL Endpoint** of the Azure Synapse workspace.
    2.  **Authentication:** The connection is authenticated using the Synapse SQL admin credentials.
    3.  **Data Model:** The `gold.external_sales` table is loaded into the Power BI data model.
    4.  **Report:** A simple report is built, demonstrating how a business user can now easily analyze the curated data.

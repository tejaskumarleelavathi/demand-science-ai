# Demand Science AI Platform — Complete Architecture Guide v2
### Real Datasets · Data Engineering · GenAI · Full-Stack · Cloud

> **How to use this doc:** Read the entire thing once before writing a single line of code. Understand the big picture first. Then go phase by phase. Every section explains *what* a technology is, *why* it exists in this project specifically, and *what your concrete goal is* before you build it.

---

## What Are You Building?

A **two-layer platform** that combines a full data engineering pipeline with a multi-agent AI system — using three real, publicly available datasets.

**Layer 1 — Data Engineering Pipeline:**
Raw CSV files from Kaggle (Walmart sales, Superstore orders, FMCG product data) are ingested, cleaned, transformed, and unified into a Snowflake data warehouse using Apache Spark running on Databricks. This is your ETL (Extract, Transform, Load) pipeline. It runs on a schedule and keeps your warehouse fresh.

**Layer 2 — GenAI Application:**
A React chat interface where a business user asks a natural language question. Four AI agents coordinate automatically: one queries Snowflake with generated SQL, one searches real industry PDFs semantically, one synthesizes both into a business recommendation, and one scores the quality. All powered by Azure OpenAI.

**The combined story you tell in interviews:**
> *"I built an end-to-end data platform. On the engineering side, I ingested three real retail datasets, used PySpark on Databricks to clean and join them into a unified star schema in Snowflake. On the AI side, I built a four-agent system on top of that warehouse where business users ask natural language questions and get grounded, data-backed recommendations with quality scores."*

That story covers Data Engineering, Big Data, GenAI, Agentic AI, Cloud, DevOps, and Full-Stack — all in one project.

---

## The Real Datasets You Are Using

### Dataset 1 — Walmart Store Sales (your `sales_facts` table)
- **Download:** kaggle.com/datasets/yasserh/walmart-dataset
- **What it contains:** Weekly sales data for 45 real Walmart stores across ~3 years. Each row is one store's sales for one week.
- **Columns you will use:**
  - `Store` — store number (1–45), which you will map to US regions
  - `Date` — week of the sale (e.g., 2010-02-05)
  - `Weekly_Sales` — total sales in dollars for that store that week
  - `Holiday_Flag` — 1 if it was a holiday week (Thanksgiving, Christmas, Super Bowl, Labor Day), 0 if not
  - `Temperature` — average temperature in the region that week
  - `Fuel_Price` — cost of fuel in the region that week
  - `CPI` — Consumer Price Index (measure of inflation/cost of living)
  - `Unemployment` — regional unemployment rate that week
- **Size:** ~6,400 rows, 8 columns
- **Real-world relevance:** This is actual retail performance data. The unemployment and CPI columns make it analytically rich — your agents can detect correlations between economic conditions and sales performance.

### Dataset 2 — Superstore Sales (your `marketing_spend` table)
- **Download:** kaggle.com/datasets/rohitsahoo/sales-forecasting
- **What it contains:** 4 years of retail orders from a fictional US superstore chain. Each row is one order line item.
- **Columns you will use:**
  - `Order ID` — unique order identifier
  - `Order Date` — when the order was placed
  - `Ship Mode` — Standard Class / Second Class / First Class / Same Day
  - `Customer Segment` — Consumer / Corporate / Home Office
  - `City` / `State` / `Region` — geographic location (already has US regions: West, East, Central, South)
  - `Category` — Technology / Furniture / Office Supplies (you map these to CPG categories)
  - `Sub-Category` — more granular product type
  - `Sales` — revenue from this order line
  - `Discount` — discount applied (0.0 to 1.0) — this is your marketing/promotional spend proxy
  - `Profit` — profit after discount and costs
- **Size:** ~9,800 rows, 21 columns
- **Real-world relevance:** The `Discount` column is your marketing investment proxy — higher discount = more promotional spend. The `Profit` column measures ROI. This is how CPG companies think about trade promotions.

### Dataset 3 — FMCG Product Data (your `product_catalog` table)
- **Download:** kaggle.com/datasets/krishanukalita/fmcg-sales-demand-forecasting-and-optimization
- **What it contains:** Daily product-level sales data for an FMCG company across 2022–2024. Each row is one product category's daily performance at one store location.
- **Columns you will use:**
  - `Date` — daily date
  - `Product_Category` — Beverages / Household / Personal Care / Snacks / Dairy / etc.
  - `Sales_Volume` — units sold that day
  - `Price` — selling price per unit
  - `Promotion` — 1 if product was on promotion, 0 if not
  - `Store_Location` — Urban / Suburban / Rural
  - `Weekday` — 0=Monday through 6=Sunday
  - `Supplier_Cost` — what it costs to source the product
  - `Replenishment_Lead_Time` — days to restock
  - `Stock_Level` — units currently in inventory
- **Size:** ~1,000 rows, 10 columns
- **Real-world relevance:** This is the closest to what Conagra actually tracks — product-level demand, promotions, pricing, and supply chain metrics all in one table.

### RAG Documents — Real Public PDFs
These are real documents you download and index into your vector store. They give your RAG agent genuine industry context.

- **Conagra Brands Annual Reports** — from conagrabrands.com/investor-relations (the 2022 and 2023 reports are free PDFs). Real brand performance data, regional strategy, and product category discussions.
- **NielsenIQ Consumer Outlook Reports** — from nielseniq.com/global/en/insights (free downloadable PDFs on CPG trends, snacking, and frozen food categories).
- **USDA Food Expenditure Series** — from ers.usda.gov/data-products/food-expenditure-series (free government data on food spending trends by category and region).
- **FMI (Food Marketing Institute) reports** — from fmi.org/our-research (free annual reports on grocery retail trends, consumer behavior, and channel shifts).

---

## How the Three Datasets Connect

The three datasets come from different sources and share no natural key. Your data engineering pipeline (Spark job) creates two shared bridge dimensions: `region` and `product_category`.

### The Bridge Logic

**Walmart `Store` numbers → `region`**
```
Stores 1–15   → "South"
Stores 16–30  → "Midwest"
Stores 31–45  → "West"
```

**Walmart departments → `product_category`**
Since Walmart data has no product category, you assign based on store number ranges:
```
Stores 1–9    → "Frozen Foods"
Stores 10–18  → "Snacks & Confectionery"
Stores 19–27  → "Beverages"
Stores 28–36  → "Household & Cleaning"
Stores 37–45  → "Personal Care"
```

**Superstore `Region` → already mapped** (West/East/Central/South → kept as-is, East mapped to "Northeast")

**Superstore `Category` → `product_category`**
```
"Technology"      → "Beverages"        (high-velocity, high-margin)
"Office Supplies" → "Snacks"           (low-price, high-volume)
"Furniture"       → "Household"        (considered purchase, lower frequency)
```

**FMCG `Store_Location` → `region`**
```
"Urban"    → "Northeast"
"Suburban" → "Midwest"
"Rural"    → "South"
```

**FMCG `Product_Category`** → already in CPG terms, kept as-is.

### The Final Unified Schema in Snowflake

After your Spark ETL runs, Snowflake contains three clean tables:

**`sales_facts`**
```
store_id | region    | product_category   | week_date  | weekly_sales | is_holiday | temperature | fuel_price | cpi    | unemployment
---------|-----------|-------------------|------------|-------------|------------|-------------|------------|--------|-------------
1        | South     | Frozen Foods      | 2010-02-05 | 1,643,690   | 0          | 42.31       | 2.572      | 211.09 | 8.1061
16       | Midwest   | Snacks            | 2010-02-05 | 987,432     | 0          | 28.10       | 2.891      | 197.34 | 9.4230
31       | West      | Beverages         | 2010-02-05 | 1,102,881   | 0          | 68.20       | 3.102      | 189.45 | 6.2033
```

**`marketing_spend`**
```
order_id       | region    | product_category   | order_date | sales_amount | discount | profit  | customer_segment
---------------|-----------|-------------------|------------|-------------|----------|---------|----------------
CA-2017-152156 | South     | Household         | 2017-08-11 | 261.96      | 0.00     | 41.91   | Consumer
CA-2017-138688 | West      | Beverages         | 2017-06-12 | 14.62       | 0.20     | -3.41   | Corporate
US-2016-108966 | Midwest   | Snacks            | 2016-10-09 | 907.15      | 0.00     | 321.89  | Consumer
```

**`product_catalog`**
```
product_category   | sale_date  | sales_volume | price | on_promotion | region    | supplier_cost | stock_level | replenishment_days
-------------------|------------|-------------|-------|--------------|-----------|---------------|-------------|------------------
Household          | 2022-01-01 | 158         | 35.19 | 0            | Northeast | 9.29          | 2072        | 4
Personal Care      | 2022-01-02 | 110         | 38.94 | 0            | Northeast | 13.27         | 325         | 3
Beverages          | 2022-01-03 | 203         | 12.44 | 1            | Midwest   | 4.11          | 891         | 2
```

### Example JOIN Query the SQL Agent Will Generate
```sql
SELECT 
    s.region,
    s.product_category,
    SUM(s.weekly_sales)          AS total_sales,
    AVG(m.discount)              AS avg_discount_rate,
    SUM(m.profit)                AS total_promo_profit,
    AVG(p.stock_level)           AS avg_stock,
    AVG(p.replenishment_days)    AS avg_replenishment_days,
    AVG(s.unemployment)          AS avg_unemployment
FROM sales_facts s
JOIN marketing_spend m
    ON s.region = m.region
    AND s.product_category = m.product_category
JOIN product_catalog p
    ON s.product_category = p.product_category
    AND s.region = p.region
WHERE s.is_holiday = 1
GROUP BY s.region, s.product_category
ORDER BY total_sales DESC
```

This answers: *"During holiday weeks, which region and product category combinations perform best, and how does promotional discounting and stock availability relate to that performance?"*

---

## Full System Architecture

```
════════════════════════════════════════════════════════════════════
                     LAYER 1: DATA ENGINEERING
════════════════════════════════════════════════════════════════════

 RAW DATA SOURCES (Kaggle CSVs)
 ┌─────────────────┐  ┌──────────────────┐  ┌─────────────────┐
 │  walmart.csv    │  │  superstore.csv  │  │  fmcg.csv       │
 │  6,400 rows     │  │  9,800 rows      │  │  1,000 rows     │
 │  8 columns      │  │  21 columns      │  │  10 columns     │
 └────────┬────────┘  └────────┬─────────┘  └────────┬────────┘
          │                    │                      │
          └────────────────────┼──────────────────────┘
                               │
                               ▼
 ┌─────────────────────────────────────────────────────────────┐
 │              AZURE DATA LAKE STORAGE (ADLS Gen2)            │
 │                  /raw/walmart/walmart.csv                   │
 │                  /raw/superstore/superstore.csv             │
 │                  /raw/fmcg/fmcg.csv                        │
 │              (Landing zone — files stored as-is)            │
 └──────────────────────────────┬──────────────────────────────┘
                                │
                                ▼
 ┌─────────────────────────────────────────────────────────────┐
 │                  DATABRICKS (Apache Spark)                  │
 │                                                             │
 │  Notebook 1: ingest.py                                      │
 │    → Reads raw CSVs from ADLS into Spark DataFrames        │
 │    → Validates schema, checks for nulls/duplicates          │
 │    → Writes to /silver/ layer as Parquet (cleaned)         │
 │                                                             │
 │  Notebook 2: transform.py                                   │
 │    → Applies region mapping to Walmart store numbers        │
 │    → Maps Superstore categories to CPG product categories   │
 │    → Maps FMCG store locations to regions                   │
 │    → Adds derived columns (year, quarter, month, weekday)   │
 │    → Joins all three datasets on region + product_category  │
 │    → Writes unified tables to /gold/ layer as Parquet       │
 │                                                             │
 │  Notebook 3: load.py                                        │
 │    → Reads from /gold/ layer                               │
 │    → Loads sales_facts, marketing_spend, product_catalog    │
 │      into Snowflake via Spark-Snowflake connector           │
 │    → Runs upsert (merge) logic to avoid duplicates          │
 │                                                             │
 │  Databricks Workflow: runs all 3 notebooks in sequence      │
 │    on a daily schedule (or triggered manually)              │
 └──────────────────────────────┬──────────────────────────────┘
                                │
                                ▼
 ┌─────────────────────────────────────────────────────────────┐
 │                        SNOWFLAKE                            │
 │                   (Data Warehouse)                          │
 │                                                             │
 │   DATABASE: DEMAND_DB                                       │
 │   SCHEMA: PUBLIC                                            │
 │                                                             │
 │   ┌──────────────────┐  ┌──────────────────┐               │
 │   │   sales_facts    │  │ marketing_spend  │               │
 │   │  (from Walmart)  │  │ (from Superstore)│               │
 │   └──────────────────┘  └──────────────────┘               │
 │                                                             │
 │   ┌──────────────────────────────┐                         │
 │   │        product_catalog       │                         │
 │   │        (from FMCG)           │                         │
 │   └──────────────────────────────┘                         │
 │                                                             │
 │   Joined on: region + product_category                      │
 └──────────────────────────────┬──────────────────────────────┘
                                │
                                │
════════════════════════════════════════════════════════════════════
                     LAYER 2: GENAI APPLICATION
════════════════════════════════════════════════════════════════════
                                │
                                ▼
 ┌─────────────────────────────────────────────────────────────┐
 │                 FASTAPI BACKEND (Python)                    │
 │                                                             │
 │   routes.py  ──▶  orchestrator.py                          │
 │                        │                                    │
 │              ┌─────────┼─────────┐                         │
 │              ▼                   ▼                          │
 │         SQL Agent           RAG Agent                       │
 │    (queries Snowflake)  (searches PDFs in ChromaDB)        │
 │              │                   │                          │
 │              └────────┬──────────┘                          │
 │                       ▼                                     │
 │                Insight Agent                                │
 │            (synthesizes SQL data                            │
 │             + doc context into                              │
 │              recommendation)                                │
 │                       │                                     │
 │                       ▼                                     │
 │                 Judge Agent                                 │
 │           (scores answer quality                            │
 │            1–10 on 3 dimensions)                            │
 │                       │                                     │
 │                       ▼                                     │
 │             JSON Response                                   │
 └──────────┬────────────────────┬────────────────────────────┘
            │                    │
            ▼                    ▼
 ┌──────────────────┐  ┌──────────────────────────┐
 │    Snowflake     │  │   ChromaDB (local)        │
 │  (read queries   │  │   Conagra Annual Reports  │
 │   from agents)   │  │   NielsenIQ Reports       │
 │                  │  │   USDA Food Data          │
 └──────────────────┘  │   FMI Grocery Reports     │
                        └──────────────────────────┘
            │                    │
            └──────────┬─────────┘
                       ▼
            ┌─────────────────────┐
            │   Azure OpenAI      │
            │   GPT-4o (LLM)     │
            │   text-embedding-   │
            │   ada-002           │
            └─────────────────────┘
                       │
                       ▼
 ┌─────────────────────────────────────────────────────────────┐
 │               React + TypeScript Frontend                   │
 │                  (Chat UI in browser)                       │
 │                                                             │
 │  ┌──────────────────────────────────────────────────────┐  │
 │  │  [Type your question here...]          [Ask]         │  │
 │  └──────────────────────────────────────────────────────┘  │
 │                                                             │
 │  ┌────────────────────────────────────────────────────┐    │
 │  │ AI Insight:                                        │    │
 │  │ "The South region leads holiday sales at $48M..."  │    │
 │  └────────────────────────────────────────────────────┘    │
 │                                                             │
 │  Agents Fired: [SQLAgent] [RAGAgent] [InsightAgent] [Judge]│
 │                                                             │
 │  Quality Score:  Overall: 9/10  Relevance: 9  Grounding: 8 │
 └─────────────────────────────────────────────────────────────┘

════════════════════════════════════════════════════════════════════
                     DEPLOYMENT LAYER
════════════════════════════════════════════════════════════════════

 Docker images → Azure Container Registry → AKS (Kubernetes)
 Terraform provisions: ADLS, Databricks, Snowflake connector, ACR, AKS
 Azure DevOps CI/CD: test → build → push → deploy on every git push
```

---

## Complete Folder Structure

```
demand-science-ai/
│
├── data_engineering/                  ← EVERYTHING for the ETL pipeline
│   │
│   ├── notebooks/                     ← Databricks PySpark notebooks
│   │   ├── 01_ingest.py               ← Read CSVs from ADLS, validate, write silver Parquet
│   │   ├── 02_transform.py            ← Clean, map, join datasets, write gold Parquet
│   │   └── 03_load.py                 ← Load gold Parquet into Snowflake tables
│   │
│   ├── schemas/                       ← Expected column definitions for validation
│   │   ├── walmart_schema.json        ← Expected columns/types for Walmart CSV
│   │   ├── superstore_schema.json
│   │   └── fmcg_schema.json
│   │
│   ├── mappings/                      ← The transformation logic as config files
│   │   ├── store_to_region.json       ← Walmart store number → region name
│   │   ├── store_to_category.json     ← Walmart store number → product category
│   │   ├── superstore_category_map.json ← Superstore category → CPG category
│   │   └── location_to_region.json    ← FMCG store location → region
│   │
│   ├── tests/                         ← Unit tests for Spark transformations
│   │   ├── test_ingest.py
│   │   ├── test_transform.py
│   │   └── test_load.py
│   │
│   ├── databricks_workflow.json       ← Defines the scheduled job in Databricks
│   └── requirements_spark.txt        ← PySpark dependencies
│
├── backend/                           ← FastAPI + AI Agents
│   ├── app/
│   │   ├── main.py                    ← FastAPI app entry point, CORS setup
│   │   ├── api/
│   │   │   └── routes.py              ← POST /api/query, GET /api/health
│   │   ├── agents/
│   │   │   ├── sql_agent.py           ← NL → SQL → Snowflake → structured data
│   │   │   ├── rag_agent.py           ← Question → embedding → ChromaDB → doc context
│   │   │   ├── insight_agent.py       ← SQL data + doc context → GPT-4o → recommendation
│   │   │   └── judge_agent.py         ← Answer → GPT-4o evaluation → quality scores
│   │   ├── services/
│   │   │   ├── data_service.py        ← All Snowflake connection and query logic
│   │   │   └── orchestrator.py        ← Coordinates the 4-agent pipeline
│   │   └── models/
│   │       └── schemas.py             ← Pydantic request/response models
│   ├── tests/
│   │   ├── test_data_service.py
│   │   ├── test_sql_agent.py
│   │   ├── test_rag_agent.py
│   │   └── test_orchestrator.py
│   ├── scripts/
│   │   └── index_documents.py         ← One-time script to embed PDFs into ChromaDB
│   ├── requirements.txt
│   ├── .env                           ← NEVER COMMIT — all secrets go here
│   └── Dockerfile
│
├── frontend/                          ← React TypeScript chat UI
│   ├── src/
│   │   ├── App.tsx                    ← Root component
│   │   ├── components/
│   │   │   ├── QuestionInput.tsx      ← Text input + submit button
│   │   │   ├── AnswerCard.tsx         ← Displays the AI recommendation
│   │   │   ├── AgentBadges.tsx        ← Shows which agents ran
│   │   │   ├── ScoreDisplay.tsx       ← Judge Agent scores (colored 1–10)
│   │   │   └── SqlPreview.tsx         ← Shows the generated SQL query
│   │   └── api/
│   │       └── client.ts              ← Axios HTTP calls to FastAPI
│   ├── package.json
│   └── Dockerfile
│
├── infra/                             ← Terraform: all Azure resources as code
│   ├── main.tf                        ← Resource group, ADLS, ACR, AKS
│   ├── databricks.tf                  ← Databricks workspace
│   ├── snowflake.tf                   ← Snowflake connection config
│   ├── variables.tf                   ← Region, names, sizes
│   └── outputs.tf                     ← AKS credentials, ADLS URL, etc.
│
├── k8s/                               ← Kubernetes deployment manifests
│   ├── backend-deployment.yaml        ← FastAPI: 2 replicas, resource limits
│   ├── frontend-deployment.yaml       ← React (Nginx): 2 replicas
│   ├── service.yaml                   ← Internal ClusterIP services
│   └── ingress.yaml                   ← External HTTP routing
│
├── docs/
│   ├── architecture.md                ← This document
│   ├── data_dictionary.md             ← All table/column definitions
│   └── adr/                           ← Architecture Decision Records
│       ├── 001-why-snowflake.md
│       ├── 002-why-chromadb.md
│       └── 003-why-databricks.md
│
├── azure-pipelines.yml                ← CI/CD: test → build → push → deploy
├── docker-compose.yml                 ← Run everything locally with one command
└── README.md
```

---

## Technology Glossary

### Apache Spark (via Databricks)
Spark is a distributed computing engine that processes large datasets across multiple machines in parallel. You write Python code (PySpark) that looks like Pandas but runs at scale. In this project, Spark handles your ETL: reading raw CSVs, applying transformations (cleaning, mapping, joining), and writing to Snowflake.

**Why it's in the project:** The JD explicitly lists "Databricks" as a big data experience requirement. Spark is the industry-standard tool for data engineering at companies like Conagra. Even though your datasets are small (under 20,000 rows total), you're building the pipeline the same way a real team would handle millions of rows — that's the point.

**Key Spark concepts you'll use:**
- `DataFrame` — a distributed table (like a Pandas DataFrame but runs in parallel)
- `SparkSession` — the entry point to all Spark operations
- `.read.csv()` — reads a CSV file from storage into a DataFrame
- `.withColumn()` — adds or transforms a column
- `.join()` — joins two DataFrames on a shared key
- `.write.parquet()` — writes the DataFrame to Parquet format
- `.write.format("snowflake")` — loads a DataFrame directly into Snowflake

---

### Databricks
A cloud platform built on top of Apache Spark. It gives you a notebook environment (like Jupyter) where you write PySpark code, and it manages all the cluster infrastructure behind the scenes. You don't install or configure Spark yourself — Databricks handles it.

**Why it's in the project:** The JD lists "Databricks" by name. Databricks is the primary tool for data engineering at large CPG companies. It has a free Community Edition that gives you enough to build this project.

**Key Databricks concepts:**
- **Notebook** — a file containing PySpark code organized in runnable cells, like Jupyter
- **Cluster** — the computing resources Databricks assigns to run your notebook
- **Workflow/Job** — a scheduled sequence of notebooks (your ETL pipeline runs as a job)
- **DBFS (Databricks File System)** — internal storage, but you'll use Azure Data Lake instead for production-style setup

---

### Azure Data Lake Storage (ADLS Gen2)
A cloud file storage system optimized for big data workloads. Think of it as a highly scalable folder structure in Azure where you store raw files. You organize it into three zones:

- **Bronze / Raw** — files exactly as downloaded from Kaggle, no changes
- **Silver / Cleaned** — Spark-processed Parquet files after validation and basic cleaning
- **Gold / Unified** — final joined, transformed Parquet files ready to load into Snowflake

This Bronze → Silver → Gold pattern is called the **Medallion Architecture** — it's the industry-standard way to organize a data lake and a very impressive thing to name-drop in interviews.

---

### Parquet
A file format for storing data. Unlike CSV (plain text rows), Parquet stores data in columns and compresses it efficiently. A 50MB CSV might become a 10MB Parquet file that reads 10x faster. Spark reads and writes Parquet natively. You use Parquet for your silver and gold layers in ADLS because it's what production data pipelines use.

---

### Snowflake
A cloud-hosted SQL data warehouse. Your Spark job loads the final unified tables here. Your SQL Agent then queries Snowflake directly when answering user questions. The separation matters: Spark does the heavy transformation work, Snowflake does the query serving.

**Why it's in the project:** JD lists "Snowflake" and "data warehousing" explicitly.

---

### FastAPI
A modern Python framework for building HTTP APIs. It auto-generates Swagger documentation at `/docs`, validates request/response data with Pydantic automatically, and supports async Python natively. Your four agents are called through the FastAPI backend.

---

### LangChain / LangGraph
Python libraries for building LLM-powered applications. LangChain gives you prompt templates, output parsers, and integrations with databases and vector stores. LangGraph lets you define agents as nodes in a directed graph — each node does one thing, edges connect them. This is your agent orchestration framework, directly mapping to "orchestration, A2A, agentic AI" in the JD.

---

### ChromaDB
An open-source vector database that runs locally (no cloud account needed). You embed PDF documents as vectors using Azure OpenAI's embedding model, then store them in ChromaDB. When the RAG Agent gets a question, it converts the question to a vector and ChromaDB finds the most semantically similar stored document chunks.

---

### Azure OpenAI
Microsoft's hosted version of OpenAI's models. You access GPT-4o for text generation and `text-embedding-ada-002` for converting text to vectors. You need an Azure subscription and to apply for Azure OpenAI access (takes 1–2 business days usually).

---

### Pydantic
Python data validation library. You define data shapes as Python classes, and Pydantic enforces them automatically. FastAPI uses Pydantic for all request/response validation. This demonstrates OOP and type safety.

---

### Docker
Packages your application + all dependencies into a portable container image. One `Dockerfile` describes how to build the image. `docker-compose.yml` runs multiple containers together locally. In production, containers run on Kubernetes.

---

### Kubernetes / AKS
Kubernetes manages containers at scale: starts them, restarts them if they crash, scales them up under load, and routes traffic. AKS is Azure's managed Kubernetes service — you don't manage the Kubernetes control plane, Azure does.

---

### Terraform
Infrastructure as Code. Instead of clicking in the Azure portal to create resources, you write `.tf` files that declare what you want, and Terraform creates it. Everything is version-controlled and reproducible. You provision ADLS, Databricks workspace, ACR, and AKS all via Terraform.

---

### Azure DevOps CI/CD Pipeline
Defined in `azure-pipelines.yml`. Every push to `main` triggers automated stages: run Pytest tests → build Docker images → push to Azure Container Registry → deploy to AKS with updated images. This is continuous integration (test on every commit) + continuous delivery (deploy automatically when tests pass).

---

## The Four AI Agents — Detailed

### Agent 1: SQL Agent
**Input:** User's natural language question
**What it does:**
1. Sends the question + your full Snowflake schema (all table/column names) to GPT-4o
2. GPT-4o returns a valid Snowflake SQL query
3. Runs that SQL against Snowflake via the data service
4. Returns the raw result rows as a Python list of dicts

**Example:**
- Question: *"Which region had the highest Frozen Foods sales during holiday weeks?"*
- Generated SQL: `SELECT region, SUM(weekly_sales) AS total FROM sales_facts WHERE product_category='Frozen Foods' AND is_holiday=1 GROUP BY region ORDER BY total DESC`
- Returns: `[{"region": "South", "total": 28403991}, {"region": "Midwest", "total": 21887329}, ...]`

---

### Agent 2: RAG Agent
**Input:** Same user question
**What it does:**
1. Converts the question to a vector embedding using Azure OpenAI
2. ChromaDB finds the 2–3 stored document chunks with the most similar meaning
3. Returns those chunks as plain text strings

**Example:**
- Question: *"Which region had the highest Frozen Foods sales during holiday weeks?"*
- ChromaDB surfaces from Conagra Annual Report: *"Birds Eye frozen vegetables showed strongest performance in Southern markets during Q4, driven by holiday meal occasion purchasing. The Thanksgiving window represents 18% of annual frozen category revenue..."*
- Returns that passage as context string

---

### Agent 3: Insight Agent
**Input:** Original question + SQL data results + RAG document context
**What it does:** Combines all three inputs into a carefully structured prompt and sends it to GPT-4o asking for: a direct answer, the root cause/key insight, and a specific actionable recommendation.

**Example output:**
> "The South region led Frozen Foods holiday sales at $28.4M, approximately 30% above the Midwest. This aligns with industry data showing Southern markets drive disproportionate holiday frozen category volume due to larger average household sizes and higher Thanksgiving cooking occasions. The Conagra Annual Report confirms Birds Eye outperforms in Southern Q4 periods. **Recommendation:** Increase Birds Eye promotional depth and in-store display allocations in Southern markets by at least 15% for the November–December window. Given the $28M baseline, even a 5% lift represents $1.4M incremental revenue."

---

### Agent 4: Judge Agent
**Input:** The original question + the generated answer
**What it does:** Asks GPT-4o to act as a quality evaluator and score the answer on three dimensions, returning structured JSON:
```json
{
  "score": 9,
  "relevance": 9,
  "data_grounding": 8,
  "actionability": 9,
  "feedback": "Answer directly addresses the question with specific dollar figures and a quantified recommendation."
}
```
Low scores (below 6) trigger a warning flag in the UI so users know to treat the answer with caution.

---

### The Orchestrator
Coordinates all four agents in sequence. Receives the question, calls SQL Agent and RAG Agent (these can run concurrently — they don't depend on each other), passes both results to Insight Agent, passes the insight to Judge Agent, assembles the final response object.

```
User Question
      │
      ├─────────────────────┐
      ▼                     ▼
 SQL Agent             RAG Agent
 (queries Snowflake)   (searches ChromaDB)
      │                     │
      └──────────┬──────────┘
                 ▼
          Insight Agent
          (synthesizes both)
                 │
                 ▼
           Judge Agent
           (scores quality)
                 │
                 ▼
         Final JSON Response
```

---

## End-to-End Data Flow

### Pipeline Flow (runs once before you start building agents, then on a schedule)
1. You download three CSVs from Kaggle and upload them to ADLS `/raw/` folder
2. You trigger the Databricks workflow manually (or it runs on its daily schedule)
3. `01_ingest.py` reads each CSV from ADLS into a Spark DataFrame, validates schema, writes cleaned Parquet to `/silver/`
4. `02_transform.py` reads silver Parquet, applies all mapping logic (region, category), joins all three DataFrames on region + product_category, writes unified gold Parquet to `/gold/`
5. `03_load.py` reads gold Parquet and writes to Snowflake using the Spark-Snowflake connector — your three tables are now populated with real, clean, unified data
6. From this point on, Snowflake is the live source of truth

### Query Flow (happens every time a user asks a question)
1. User types question in React UI, presses Ask
2. React sends `POST /api/query` with `{ "question": "..." }` to FastAPI
3. FastAPI validates the request with Pydantic, passes it to the Orchestrator
4. Orchestrator calls SQL Agent + RAG Agent (in parallel)
5. SQL Agent sends question + schema to Azure OpenAI → gets SQL → runs against Snowflake → returns data rows
6. RAG Agent embeds the question → queries ChromaDB → returns 2–3 relevant PDF passages
7. Orchestrator calls Insight Agent with question + SQL data + PDF context
8. Insight Agent sends combined prompt to Azure OpenAI → returns recommendation paragraph
9. Orchestrator calls Judge Agent with question + recommendation
10. Judge Agent sends evaluation prompt to Azure OpenAI → returns JSON scores
11. Orchestrator assembles final response: `{ question, answer, evaluation, sql_used, agents_invoked }`
12. FastAPI returns JSON to React
13. React renders: answer card, agent badges, score display, SQL preview panel

---

## Build Phases — Your Complete Roadmap

Work through these in strict order. Do not skip ahead.

---

### Phase 1: Project Setup (Week 1)

**Goal:** Get all accounts, tooling, and scaffolding in place before writing real logic.

**Accounts to create (all free):**
- Azure account — portal.azure.com (free tier gives $200 credit)
- Databricks Community Edition — community.cloud.databricks.com (completely free, no credit card)
- Snowflake trial — snowflake.com (30-day free trial)
- Kaggle account — kaggle.com (for dataset downloads)
- Azure DevOps — dev.azure.com (free for up to 5 users)

**What to do:**
- Download all three datasets from Kaggle as CSV files
- Create the full folder structure locally and push to Azure DevOps Git repo
- Set up Azure DevOps Board with one Epic per phase in this document
- Set up a Python virtual environment in `/backend`
- Create a React/TypeScript project using Vite in `/frontend`
- Create a `.gitignore` that includes `.env`, `*.csv`, `__pycache__`, `node_modules`, `chroma_store/`
- Apply for Azure OpenAI access immediately — it can take 1–2 days and you need it for Phase 4

**JD skills:** Agile/boards setup, Git trunk-based workflow, Azure DevOps

---

### Phase 2: Data Lake Setup (Week 2 — first half)

**Goal:** Get your raw data into Azure Data Lake Storage organized in the medallion pattern.

**What to do:**
- Create an Azure Resource Group called `demand-science-rg`
- Create an Azure Data Lake Storage Gen2 account (in the portal — Terraform comes later)
- Create three containers/folders: `/raw/walmart/`, `/raw/superstore/`, `/raw/fmcg/`
- Upload your three CSV files to their respective `/raw/` folders
- Write `infra/main.tf` to codify these ADLS resources in Terraform (even though you created them manually first, writing the Terraform is the learning exercise)

**What you understand after this phase:**
- What a data lake is vs. a database
- Why raw/silver/gold (medallion) separation exists — you never overwrite raw data, so you can always reprocess
- How ADLS differs from regular Azure Blob Storage (hierarchical namespace for big data)

**JD skills:** Data warehousing concepts, Azure cloud infrastructure, IaC/Terraform beginning

---

### Phase 3: Spark ETL Pipeline on Databricks (Weeks 2–3)

**Goal:** Build three PySpark notebooks that form your complete ETL pipeline.

This is the heart of your data engineering story.

**Setup:**
- Create a Databricks Community Edition workspace
- Create a cluster (use the default single-node Community cluster)
- Mount your ADLS storage to Databricks so notebooks can read/write files
- Create a folder called `demand_science_pipeline` in your workspace

**Notebook 1 — `01_ingest.py` (Extract + Validate)**

This notebook's job is to read raw CSVs from ADLS, validate they have the right columns and reasonable data, log any issues, and write cleaned Parquet files to the `/silver/` layer.

What it specifically does for each dataset:
- Reads the CSV into a Spark DataFrame using `spark.read.csv()`
- Checks that all expected columns exist (using your schema JSON files)
- Drops fully duplicate rows
- Checks for null values in key columns and logs a count
- Casts columns to the right data types (dates as DateType, numbers as DoubleType)
- Writes the validated DataFrame to ADLS `/silver/` as Parquet

**Notebook 2 — `02_transform.py` (Transform + Join)**

This notebook's job is the core business logic — it turns three separate cleaned datasets into one unified, analytically useful set of tables.

What it does for Walmart data:
- Reads `/silver/walmart.parquet`
- Adds `region` column by mapping store number ranges using a `when().otherwise()` expression
- Adds `product_category` column by mapping store numbers to CPG categories
- Renames `Date` to `week_date`, `Weekly_Sales` to `weekly_sales`, `Holiday_Flag` to `is_holiday`
- Adds derived columns: `year`, `quarter`, `month` extracted from `week_date`
- Writes to `/gold/sales_facts.parquet`

What it does for Superstore data:
- Reads `/silver/superstore.parquet`
- Maps `Region` column (East → Northeast, others kept as-is)
- Maps `Category` column to CPG product categories using your mapping JSON
- Renames `Sales` to `sales_amount`, `Discount` to `discount`, `Profit` to `profit`
- Adds `region` and `product_category` as the join keys
- Writes to `/gold/marketing_spend.parquet`

What it does for FMCG data:
- Reads `/silver/fmcg.parquet`
- Maps `Store_Location` to region names
- Renames all columns to snake_case
- Writes to `/gold/product_catalog.parquet`

**Notebook 3 — `03_load.py` (Load to Snowflake)**

This notebook reads each gold Parquet file and writes it to Snowflake. You use the Spark-Snowflake connector (`net.snowflake.spark.snowflake`). The notebook runs a `MERGE` (upsert) operation so re-running the pipeline doesn't create duplicate rows.

**Databricks Workflow:**

Create a Workflow in the Databricks UI that runs notebooks 1 → 2 → 3 in sequence. Set it to trigger on a daily schedule. This is your production pipeline — it would pick up new data files if they were added to ADLS.

**What you understand after this phase:**
- The difference between transformation logic and loading logic
- Why Parquet is better than CSV for intermediate storage
- What upsert/MERGE means and why it matters for idempotent pipelines
- What a DAG (Directed Acyclic Graph) is — your workflow is one
- How to think about data quality validation at the ingestion layer

**JD skills:** Databricks, Apache Spark/PySpark, data warehousing, big data processing, ETL pipeline design

---

### Phase 4: Snowflake Verification + Data Service (Week 3)

**Goal:** Confirm your data landed correctly in Snowflake and build the Python layer your agents will use to query it.

**What to do:**
- Log into the Snowflake UI and run manual SQL queries against all three tables
- Verify row counts match expected (Walmart ~6,400, Superstore ~9,800, FMCG ~1,000)
- Verify the join query from the architecture section works and returns sensible results
- Write `backend/app/services/data_service.py` as a Python class that wraps Snowflake connections
- The `DataService` class uses dependency injection (constructor takes connection config) so it can be mocked in tests
- Write Pytest unit tests for every method in `DataService` — use `unittest.mock` to mock the Snowflake connector so tests run without a real database
- Tests should cover: successful query returns list of dicts, empty result returns empty list, connection error raises a custom exception

**What you understand after this phase:**
- How to connect Python to Snowflake using `snowflake-connector-python`
- What dependency injection means in practice — pass dependencies in, don't create them inside the class
- How to mock external services in unit tests

**JD skills:** Snowflake, Python OOP/SOLID, unit testing with Pytest, data service layer architecture

---

### Phase 5: AI Agents (Weeks 4–5)

**Goal:** Build all four agents as independent, testable Python classes that use your real Snowflake data.

**Setup before coding:**
- Confirm Azure OpenAI access came through — you need both a GPT-4o deployment and a `text-embedding-ada-002` deployment
- Install packages: `langchain`, `langchain-openai`, `chromadb`, `openai`
- Download your RAG documents (Conagra Annual Report PDF, NielsenIQ report PDFs) and put them in `backend/docs/`
- Run `backend/scripts/index_documents.py` to chunk and embed all PDFs into ChromaDB

**Build order — do these strictly one at a time, test each before moving on:**

**SQL Agent first:**
- Build the class, hardcode a test question, print the generated SQL and results
- Verify: does the SQL it generates actually make sense for your Snowflake schema?
- Common issue: GPT-4o might hallucinate column names. Fix this by providing the complete schema with exact column names in the system prompt.

**RAG Agent second:**
- Run `index_documents.py` to confirm documents are indexed
- Ask a test question, print the retrieved document chunks
- Verify: are the chunks actually relevant to what you asked?

**Insight Agent third:**
- Feed it hardcoded SQL results (copy a real query result) and hardcoded doc context
- Verify the answer is coherent, data-driven, and includes a recommendation

**Judge Agent fourth:**
- Feed it a hardcoded question and answer
- Verify it returns valid JSON with numeric scores

**Orchestrator last:**
- Wire all four together
- Test the full pipeline with real questions:
  - *"Which region had the highest sales during holiday weeks?"*
  - *"What product categories are most affected by high unemployment?"*
  - *"When promotions are applied to Beverages, does profit actually increase?"*
- These are questions that require joining all three tables — your SQL Agent will need to generate multi-table JOINs

**JD skills:** Agentic AI, orchestration, A2A coordination, LLMs, vector stores, embeddings, observability/judging

---

### Phase 6: FastAPI Backend (Week 5)

**Goal:** Expose the orchestrator as a real HTTP API with proper validation and documentation.

**What to do:**
- Build `main.py` — FastAPI app with CORS configured for React's dev server (`localhost:5173`)
- Build `routes.py` — two endpoints:
  - `POST /api/query` — accepts `{ "question": "..." }`, returns full agent response
  - `GET /api/health` — returns `{ "status": "ok" }` (used by Kubernetes liveness probes)
- Build `schemas.py` — Pydantic models for `QueryRequest` and `QueryResponse` that define exactly what shape the request and response must have
- Run locally: `uvicorn app.main:app --reload` and test at `localhost:8000/docs`
- Write one integration test using FastAPI's built-in `TestClient` that sends a mock question and asserts the response has the right structure

**What you understand after this phase:**
- What an HTTP API endpoint is and how request/response works
- What Pydantic validation actually does — sends a 422 error with a clear message if the request shape is wrong
- What CORS is and why it's needed when React and FastAPI run on different ports

**JD skills:** Backend API development, data validation, OOP/SOLID, integration testing

---

### Phase 7: React Frontend (Week 6 — first half)

**Goal:** Build a clean, functional chat interface that calls your API.

**What to do:**
- Set up React + TypeScript with Vite: `npm create vite@latest frontend -- --template react-ts`
- Install Tailwind CSS for styling and Axios for HTTP calls
- Build the main `App.tsx` with: a text input for questions, an Ask button, a loading spinner during API calls, an error state if the API fails
- Build four components:
  - `QuestionInput.tsx` — the search bar and button
  - `AnswerCard.tsx` — displays the recommendation in a readable card
  - `AgentBadges.tsx` — shows which agents fired as colored pills (SQLAgent, RAGAgent, etc.)
  - `ScoreDisplay.tsx` — shows Judge Agent scores with color coding (green ≥8, yellow 5–7, red ≤4)
  - `SqlPreview.tsx` — shows the SQL query that was generated (collapsible — great for transparency)
- Connect to FastAPI at `http://localhost:8000/api/query`
- Test the full stack locally: React → FastAPI → Agents → Snowflake → Azure OpenAI → back

**What makes this impressive:**
Showing the SQL that was generated proves the system is grounded in real data, not hallucinating. Showing which agents fired proves it's genuinely agentic. The Judge scores prove there's quality control built in. These UI elements tell the story of your architecture visually.

**JD skills:** JavaScript/TypeScript, React, component architecture, frontend-backend integration

---

### Phase 8: Docker + Docker Compose (Week 6 — second half)

**Goal:** Containerize both applications so the full stack runs with one command.

**What to do:**
- Write `backend/Dockerfile`: starts from `python:3.11-slim`, copies requirements, installs packages, copies app code, runs `uvicorn`
- Write `frontend/Dockerfile`: multi-stage — Stage 1 uses `node:20-alpine` to run `npm run build`, Stage 2 uses `nginx:alpine` to serve the built static files
- Write `docker-compose.yml` at root: defines backend service (port 8000) and frontend service (port 3000), passes environment variables from your `.env` file to the backend container
- Test: `docker-compose up --build` — the full app should work at `localhost:3000`
- Push both Dockerfiles and `docker-compose.yml` to git

**Why multi-stage frontend Dockerfile matters:** The build stage compiles TypeScript to JavaScript. The serve stage runs Nginx with the compiled output. Production containers don't include the Node.js build toolchain — much smaller and more secure.

**JD skills:** Docker, containerization, multi-stage builds, cloud engineering concepts

---

### Phase 9: Terraform + Kubernetes (Week 7 — first half)

**Goal:** Define all Azure infrastructure as code and deploy to Kubernetes.

**Terraform — what to write:**
- `infra/main.tf` — Azure Resource Group, Azure Container Registry (ACR), AKS cluster
- `infra/databricks.tf` — Databricks workspace (you'll reference the Community Edition for portfolio — in production this would be a paid workspace)
- `infra/variables.tf` — parameterize region, names, node counts
- `infra/outputs.tf` — output the AKS kubeconfig and ACR login server URL

Run `terraform init` → `terraform plan` → `terraform apply`.

**Kubernetes manifests — what to write:**
- `k8s/backend-deployment.yaml` — 2 replicas of the FastAPI container, pulls from ACR, sets environment variables from a Kubernetes Secret, exposes port 8000
- `k8s/frontend-deployment.yaml` — 2 replicas of the React/Nginx container, exposes port 80
- `k8s/service.yaml` — ClusterIP services for both deployments (internal routing)
- `k8s/ingress.yaml` — HTTP ingress that routes `/api/*` to the backend service and everything else to the frontend service

Deploy: `az aks get-credentials` then `kubectl apply -f k8s/` then `kubectl get pods` to confirm running.

**JD skills:** Terraform/IaC, Kubernetes, AKS, Azure cloud infrastructure, distributed computing

---

### Phase 10: CI/CD Pipeline (Week 7 — second half)

**Goal:** Automate the full pipeline — push code, everything else happens automatically.

**Write `azure-pipelines.yml` with four stages:**

**Stage 1 — Test:**
- Install Python 3.11, install requirements
- Run `pytest tests/ -v --cov=app --cov-report=xml`
- Publish coverage report to Azure DevOps
- Install Node 20, run `npm install && npm run build` in `/frontend`

**Stage 2 — Build:**
- Build Docker image for backend, tag with `$(Build.BuildId)` and `latest`
- Push to ACR
- Build Docker image for frontend, push to ACR

**Stage 3 — Deploy to AKS:**
- Get AKS credentials
- Update image tags in deployment manifests with the current build ID
- `kubectl apply -f k8s/`
- `kubectl rollout status deployment/demand-ai-backend` — wait for rollout to complete

**Connect this pipeline** to your Azure DevOps repo. Push any change to `main` and watch all three stages run automatically in the Azure DevOps Pipelines UI.

**JD skills:** CI/CD pipelines, DevOps, continuous delivery, automated testing integration

---

### Phase 11: Observability + Architecture Documentation (Week 8)

**Goal:** Add production-quality monitoring and wrap up the project professionally.

**Observability:**
- Add Python `logging` throughout the orchestrator — log each agent invocation, response time (use `time.time()`), number of data records returned, and Judge score
- Add logging to each Databricks notebook too — log row counts at each stage (raw CSV rows, silver rows after dedup, gold rows after join, rows loaded to Snowflake)
- This logging is what a data engineer looks at to confirm a pipeline ran successfully

**Architecture Decision Records (ADRs):**
Write three short markdown files in `docs/adr/`:
- `001-why-snowflake.md` — why Snowflake over PostgreSQL (columnar storage, separation of compute/storage, built for analytics, matches JD requirement)
- `002-why-chromadb.md` — why ChromaDB over Pinecone (free, local, no API rate limits during development, easy to swap)
- `003-why-databricks.md` — why Databricks over raw Spark (managed clusters, notebook UI, built-in scheduling, industry standard for CPG companies)

**Data Dictionary:**
Write `docs/data_dictionary.md` defining every table and every column — what it is, what type it is, where it came from originally, and example values. This is a real deliverable that data engineers produce.

**README:**
Write a comprehensive README covering: what the project does, the full tech stack, how to set it up locally step by step, how to run the Databricks pipeline, how to run tests, how to run the full stack with docker-compose, and a link to your architecture doc.

**Loom demo:**
Record a 5-minute walkthrough showing: the Databricks pipeline running, Snowflake tables with real data, asking a question in the React UI, all four agents firing, the SQL that was generated, the Judge score, and the CI/CD pipeline in Azure DevOps.

**JD skills:** Observability, technical blueprints, non-functional requirements, documentation, architectural decision records

---

## Complete JD Requirements Coverage

| Conagra JD Requirement | Phase | How It's Covered |
|---|---|---|
| Software development experience | All phases | FastAPI, React, PySpark throughout |
| Agentic AI + orchestration | Phase 5 | 4-agent pipeline with Orchestrator coordinating all agents |
| MCP protocol / A2A | Phase 5 | Insight Agent consumes outputs from SQL Agent + RAG Agent |
| Observability / Judging | Phase 5, 11 | Judge Agent scores every answer; logging throughout |
| LLMs | Phase 5 | Azure OpenAI GPT-4o powers 3 of the 4 agents |
| Vector stores | Phase 5 | ChromaDB stores embedded PDF chunks for RAG |
| Text-to-speech / embeddings | Phase 5 | text-embedding-ada-002 converts questions and documents to vectors |
| Big data / Databricks | Phase 3 | Full PySpark ETL pipeline on Databricks Community Edition |
| Snowflake / data warehousing | Phases 3–4 | Spark writes to Snowflake; agents query Snowflake live |
| OOP / SOLID principles | Phases 4–6 | Python classes, dependency injection, single responsibility throughout |
| JavaScript | Phase 7 | React + TypeScript frontend |
| Azure cloud infrastructure | Phase 9 | Resource Group, ADLS, ACR, AKS all provisioned |
| Kubernetes / AKS | Phase 9 | Full K8s deployment manifests, 2 replicas each |
| IaC / Terraform | Phase 9 | All Azure resources defined in .tf files |
| CI/CD pipelines | Phase 10 | Azure DevOps pipeline: test → build → push → deploy |
| Git / trunk-based development | Phase 1 | Protected main branch, PR-based workflow |
| Unit testing / Pytest | Phases 4–6 | Pytest with mocked dependencies for all services and agents |
| Agile methodology | Phase 1 + all | Azure DevOps Boards with Epics/Stories/tasks |
| Technical blueprints | Phase 11 | Architecture doc + ADRs + data dictionary |
| Non-functional requirements | Phase 11 | Latency targets, SLA, data freshness SLA in architecture doc |
| DevOps / continuous delivery | Phase 10 | Automated pipeline from commit to AKS deployment |
| ETL / data engineering | Phase 3 | Full medallion architecture pipeline in PySpark |

---

## Example Questions Your Platform Answers

These are real questions you can demo in your Loom video and during interviews:

**Joins all three tables:**
- *"During holiday weeks, which product categories see the highest sales AND have the worst stock levels?"* — joins sales_facts + product_catalog, filters is_holiday=1
- *"In regions with high unemployment, do promotions on Beverages generate positive or negative profit?"* — joins all three tables, correlates unemployment with discount/profit

**Sales analysis:**
- *"Which region and product category combination has the highest sales volume consistently across years?"*
- *"How does fuel price correlate with weekly sales in the Midwest?"*

**Promotion effectiveness:**
- *"When the discount rate on Snacks exceeds 20%, does profit go positive or negative?"*
- *"Which customer segments respond best to promotions?"*

**Supply chain:**
- *"Which product categories have the longest replenishment times AND highest stock volatility?"*
- *"Are there product categories where we regularly run out of stock?"*

Each of these requires your SQL Agent to write a multi-table JOIN query against your real Walmart + Superstore + FMCG data in Snowflake, grounded by context from your real Conagra and NielsenIQ PDFs.

---

## Key Concepts to Understand Before Building

**Data Engineering:**
- What is the difference between ETL (Extract-Transform-Load) and ELT (Extract-Load-Transform)?
- What is the Medallion Architecture (Bronze/Silver/Gold) and why does separating raw from processed data matter?
- What is a Parquet file and why is it better than CSV for big data pipelines?
- What is idempotency in a data pipeline — why does it matter that running the pipeline twice doesn't duplicate data?
- What is a DAG (Directed Acyclic Graph) in the context of pipeline orchestration?
- What does "schema evolution" mean — what happens when a CSV file adds a new column?

**Spark / Databricks:**
- What is the difference between a Spark transformation (lazy) and an action (triggers execution)?
- What is a Spark DataFrame vs. a Pandas DataFrame — when would you use each?
- What is partitioning in Spark and why does it affect performance?

**GenAI / Agents:**
- What is the difference between a regular API call to an LLM and an agentic workflow?
- What does "RAG" mean and why does grounding answers in real documents reduce hallucination?
- What is an "embedding" and why can math operations on vectors find semantically similar text?
- What does "orchestration" mean in the context of multi-agent systems?

**Cloud / DevOps:**
- What is the difference between a Docker image and a running container?
- What problem does Kubernetes solve that running containers manually doesn't?
- What is the difference between CI and CD?
- What does "infrastructure as code" mean and why is it better than manual portal clicking?

---

## Recommended Learning Resources

| Topic | Resource | Time Investment |
|---|---|---|
| PySpark fundamentals | spark.apache.org/docs/latest/api/python — the DataFrame guide | 4–6 hours |
| Databricks getting started | docs.databricks.com/en/getting-started — the quickstart notebooks | 2–3 hours |
| Medallion Architecture | databricks.com/glossary/medallion-architecture | 30 minutes |
| Snowflake basics | training.snowflake.com — free "Intro to Snowflake" course | 3–4 hours |
| FastAPI tutorial | fastapi.tiangolo.com/tutorial — do the full thing start to finish | 3–4 hours |
| LangChain agents | python.langchain.com/docs/how_to/agent_executor | 2 hours |
| ChromaDB | docs.trychroma.com — the quickstart | 1 hour |
| Docker | labs.play-with-docker.com — free browser playground | 2–3 hours |
| Kubernetes | kubernetes.io/docs/tutorials/kubernetes-basics | 2–3 hours |
| Terraform | developer.hashicorp.com/terraform/tutorials/azure-get-started | 2–3 hours |
| Azure fundamentals | learn.microsoft.com — AZ-900 learning path (free) | 6–8 hours |
| Pytest mocking | docs.pytest.org/en/stable/how-to/monkeypatch.html | 1–2 hours |

---

## Practical Tips

- **Start Databricks before everything else.** The Community Edition cluster takes 5–10 minutes to spin up when you first use it. Spin it up, run a `print("hello")` cell, confirm it works, then move on.
- **Snowflake trial is 30 days.** Don't activate it until you're ready to start Phase 3. Once your Spark notebooks are running and loading data, you only need Snowflake for active querying in Phases 4–6. After that, your screenshots and demo are the evidence.
- **Azure OpenAI access can take 1–2 days.** Apply for it at the start of Phase 1. While waiting, use the standard OpenAI API at api.openai.com — the same LangChain code works, just swap the credentials.
- **The `.env` file is sacred.** Add it to `.gitignore` on day one. If you accidentally commit API keys, rotate them immediately.
- **Commit at every milestone.** Every time a notebook runs successfully, a test passes, or an agent returns the right output — commit. You want a rich git history showing real iterative progress.
- **The Databricks Community Edition doesn't support scheduled workflows.** You'll run the three notebooks manually in sequence for your portfolio. Describe in your README that in production this would be a Databricks Workflow on a daily schedule — that understanding is what matters.
- **Test your SQL Agent with at least 10 different questions.** GPT-4o will sometimes generate SQL with wrong column names. Every time it fails, add that failure case to your system prompt as a correction. This is called prompt engineering and it's a real skill.
- **Record your demo before your Snowflake trial expires.** The demo is permanent evidence even after the trial ends.

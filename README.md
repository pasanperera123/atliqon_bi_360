## DE12 Databricks – Sports Bar & Atliqon Data Integration

This project builds an end‑to‑end analytics pipeline in **Databricks** to integrate **Sports Bar** data stored in **AWS S3** with existing, already‑cleaned **Atliqon** data stored in **Gold table**.  
The pipeline follows a **medallion architecture (Bronze → Silver → Gold)** and produces a **denormalised table** that is ultimately used for reporting and dashboarding.

---

## Project Goals

- **Integrate datasets**: Merge Sports Bar data from S3 with Atliqon data already modelled in aGold table.
- **Apply medallion architecture**: Land raw data, clean and transform it, then publish curated gold‑layer tables.
- **Handle incremental loads**: Support both one‑time full loads and daily incremental loads.
- **Produce analytics‑ready tables**: Create a denormalised fact table used by a BI dashboard.

---

## High‑Level Architecture

- **Source layer**
  - Sports Bar data in **S3**.
  - Atliqon data files in the local `data` folder (used to simulate/seed Databricks tables).
- **Medallion layers in Databricks**
  - **Bronze**: Raw ingested data (e.g., customers, orders, products, gross price) from the source files.
  - **Silver**: Cleaned and conformed **dimension** tables.
  - **Gold**: Business‑friendly, aggregated and denormalised tables used directly in dashboards.
- **Consumption layer**
  - A **denormalised table** defined by the SQL in `dashboard/denormalise_table_query_fmcg.txt`, serving as the primary input to dashboards.

All transformations and orchestration are intended to run within **Databricks notebooks**.

---

## Repository Structure

- **`data/`** – All seed data files used to build the warehouse tables.
  - `child_company_data/`
    - `full_load/`
      - `customers/customers.csv`
      - `gross_price/gross_price.csv`
      - `orders/landing/*.csv` (initial/full load of orders)
      - `products/products.csv`
    - `incremental_load/`
      - `orders/orders_YYYY_MM_DD.csv` (daily incremental orders)
  - `parent_company_data/`
    - `full_load/`
      - `dim_customers.csv`
      - `dim_gross_price.csv`
      - `dim_products.csv`
      - `fact_orders.csv`
    - `incremental_load/`
      - `fact_orders.csv`
      - `incremental_data_parent_company_query.txt` (logic for incremental parent fact)

- **`scripts/`** – All Databricks notebooks for setup and transformations.
  - `setup/`
    - `setup_catalog.ipynb` – Creates and configures the **Databricks catalog, schemas, and storage locations**. Run this first.
    - `utilities.ipynb` – Utility/config notebook (e.g. paths, database/catalog names, helper functions) used by other notebooks.
    - `dim_date_table_creation.ipynb` – Builds a **separate date dimension table** (e.g. `dim_date`) for use in facts and reports.
  - `dimension_data_processing/`
    - `1_customers_data_processing.ipynb` – Ingests and transforms customer data through the medallion layers and publishes `dim_customers`.
    - `2_products_data_processing.ipynb` – Processes product data and publishes `dim_products`.
    - `3_pricing_data_processing.ipynb` – Processes pricing data and publishes `dim_gross_price` (or equivalent pricing dimension).
  - `incremental_data_processing/`
    - `1_full_load_fact.ipynb` – Builds the initial **fact table** (e.g. `fact_orders`) from full‑load files.
    - `2_incremental_load_fact.ipynb` – Handles **incremental loads** of fact data (e.g. daily orders) to keep `fact_orders` up to date.

- **`dashboard/`**
  - `denormalise_table_query_fmcg.txt` – SQL script that **joins dimensions, facts, and other tables** to create the **denormalised table** used by the dashboard.

---

## Prerequisites

- **Databricks workspace** (or Databricks Community / Lakehouse platform).
- Access to an **S3 bucket** (for Sports Bar data) or equivalent storage configured as a Databricks volume / external location.
- **Git** installed locally if you plan to clone and push this repo to GitHub.

---

## Getting Started

1. **Clone the repository**

   ```bash
   git clone <your-github-repo-url>.git
   cd DE12_databricks
   ```

2. **Import notebooks into Databricks**
   - In the Databricks UI, import the notebooks from the `scripts` folder into a workspace folder of your choice.
   - Alternatively, use Databricks Repos pointing at this GitHub repository.

3. **Configure storage & S3**
   - Ensure your Databricks cluster has access to:
     - The **S3 bucket** containing Sports Bar data.
     - The data in the local `data` folder (typically uploaded to DBFS, external locations, or S3 depending on your setup).
   - Update any paths, catalog names, and schema names in `scripts/setup/utilities.ipynb` to match your environment.

---

## Execution Order (Recommended)

1. **Environment & Catalog Setup**
   - Open and run `scripts/setup/setup_catalog.ipynb` to:
     - Create the target **catalog** and schemas.
     - Register external locations / volumes that point to S3 and other storage.
   - Run `scripts/setup/utilities.ipynb` (or reference it via `%run`) so that shared configuration is available.

2. **Core Tables & Dimensions**
   - Run `scripts/setup/dim_date_table_creation.ipynb` to create the `dim_date` table.
   - Run the dimension notebooks in `scripts/dimension_data_processing/` in order:
     1. `1_customers_data_processing.ipynb`
     2. `2_products_data_processing.ipynb`
     3. `3_pricing_data_processing.ipynb`
   - These will read from the `data/child_company_data` and/or S3 Sports Bar data, transform them through Bronze/Silver, and publish **gold‑layer dimensions**.

3. **Fact Table – Full Load**
   - Use `scripts/incremental_data_processing/1_full_load_fact.ipynb` to build the initial `fact_orders` table using **full load** files from `data/child_company_data/full_load/orders` and/or parent company data under `data/parent_company_data/full_load`.

4. **Fact Table – Incremental Loads**
   - Use `scripts/incremental_data_processing/2_incremental_load_fact.ipynb` to process **daily incremental order files** from `data/child_company_data/incremental_load/orders` (and incremental parent fact logic from `data/parent_company_data/incremental_load`).
   - This notebook should be scheduled (e.g. via Databricks Jobs) to keep `fact_orders` current.

5. **Denormalised Gold Table for Dashboard**
   - Open `dashboard/denormalise_table_query_fmcg.txt` in Databricks SQL or a notebook cell.
   - Execute the SQL to create the **denormalised FMCG table** by joining:
     - `fact_orders`
     - `dim_customers`
     - `dim_products`
     - `dim_gross_price`
     - `dim_date`
   - Point your BI tool (Power BI, Tableau, etc.) at this denormalised table for reporting.

---

## Medallion Architecture Notes

- **Bronze layer**
  - Raw ingested data from S3 and local files, minimally processed (schema applied, basic cleanup).
- **Silver layer**
  - Cleaned, conformed tables:
    - Standardised data types and naming.
    - Deduplicated records.
    - Applied business rules and validations.
- **Gold layer**
  - Business‑ready tables:
    - Dimensions (`dim_customers`, `dim_products`, `dim_gross_price`, `dim_date`, etc.).
    - Facts (`fact_orders` with measures such as quantity, revenue, discounts).
    - Final denormalised FMCG table used for dashboards.

---

## Incremental Data Handling

- **Child company / Sports Bar**
  - Incremental order files are stored under `data/child_company_data/incremental_load/orders`.
  - These files are processed by `2_incremental_load_fact.ipynb` to append new data into `fact_orders`.
- **Parent company**
  - Incremental logic and/or data for parent company fact tables is captured in:
    - `data/parent_company_data/incremental_load/fact_orders.csv`
    - `data/parent_company_data/incremental_load/incremental_data_parent_company_query.txt`
  - Use or adapt this logic inside the incremental notebooks as needed.

---

## Running on Databricks

- **Cluster configuration**
  - Any standard Databricks cluster with access to S3 and your storage locations should be sufficient.
  - Attach the cluster to each notebook before running.
- **Job orchestration (optional but recommended)**
  - Create a **Databricks Job** that runs the notebooks in the recommended sequence:
    1. `setup_catalog`
    2. `utilities`
    3. `dim_date_table_creation`
    4. Dimension notebooks
    5. Full load fact
    6. Incremental fact
    7. Denormalised table SQL (as a SQL task or notebook task)
  - Configure the job to be **triggered automatically when new files land in the S3 bucket** (e.g. via S3 event notifications to a message/triggering service that starts the Databricks Job), so the pipeline runs end‑to‑end whenever fresh Sports Bar data arrives.

---

## Notes & Next Steps

- This repo is designed to be a **demonstration project** for building a medallion‑style data pipeline in Databricks.
- You can extend it by:
  - Adding more dimensions or facts.
  - Introducing data quality checks (e.g., expectations) in Silver/Gold layers.
  - Automating deployments and tests with CI/CD (e.g., GitHub Actions with Databricks).
- Go through the following Medium article for more information.

https://medium.com/@pasan.eecs/learning-databricks-through-a-realistic-business-acquisition-scenario-6e3449ed3631


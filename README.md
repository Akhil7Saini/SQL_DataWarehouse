# SQL Server Data Warehouse Project: Full Medallion Architecture Implementation

## Executive Summary

This project implements an end-to-end Data Warehouse solution built on **Microsoft SQL Server** using the **Medallion Architecture** pattern (Bronze, Silver, and Gold layers). The objective of this project is to consolidate, cleanse, integrate, and transform disparate transactional data from two enterprise operational systems—**CRM** and **ERP**—into a unified, business-ready **Star Schema** data mart for analytical reporting, Business Intelligence (BI), and downstream data science applications.

## Architecture Overview

The system follows a 3-tier Medallion Data Lakehouse/Warehouse design pattern:

```
                  ┌─────────────────────────────────────────┐
                  │           OPERATIONAL SOURCES           │
                  │   CRM CSV Files   │   ERP CSV Files     │
                  └────────────────────┬────────────────────┘
                                       │
                                       ▼ (BULK INSERT / Truncate & Load)
                  ┌─────────────────────────────────────────┐
                  │              BRONZE LAYER               │
                  │         Raw Ingestion Tables            │
                  │   - Exact raw schema replica            │
                  │   - No constraints / No transforms    │
                  └────────────────────┬────────────────────┘
                                       │
                                       ▼ (T-SQL Cleansing & Staging Proc)
                  ┌─────────────────────────────────────────┐
                  │              SILVER LAYER               │
                  │       Cleaned & Standardized            │
                  │   - Deduplication & Whitespace trim   │
                  │   - Type casting & Business rules       │
                  │   - Lead windowing for product dates    │
                  └────────────────────┬────────────────────┘
                                       │
                                       ▼ (Dimensional Modeling Views)
                  ┌─────────────────────────────────────────┐
                  │               GOLD LAYER                │
                  │        Dimensional Star Schema          │
                  │   - gold.dim_customers                  │
                  │   - gold.dim_products                   │
                  │   - gold.fact_sales                     │
                  └────────────────────┬────────────────────┘
                                       │
                                       ▼
            ┌──────────────────────────┼──────────────────────────┐
            ▼                          ▼                          ▼
     Power BI / Tableau         Ad-hoc SQL Analytics        Machine Learning

```

### Layer Specifications & Responsibilities

| Layer | Object Type | Ingestion Pattern | Core Responsibility | 
| ----- | ----- | ----- | ----- | 
| **Bronze** | Physical Tables | Stored Procedure (`bronze.load_bronze`) via `BULK INSERT` (Truncate-and-Reload) | Raw, un-transformed ingestion. Mirror source CSV schemas exactly for auditability and reprocessing. | 
| **Silver** | Physical Tables | Stored Procedure (`silver.load_silver`) via T-SQL transformations | Data cleansing, normalization, validation, deduplication, surrogate attribute derivations, metadata additions (`dwh_create_date`). | 
| **Gold** | Database Views | Virtual Views (`gold.dim_*`, `gold.fact_*`) | Star schema model optimized for analytical queries with surrogate keys (`customer_key`, `product_key`). | 

## Source Systems & Data Lineage Map

The data model integrates 6 distinct CSV entities across two operational domain sources:

```
[ CRM System ]
  ├── crm_cust_info     ──> silver.crm_cust_info     ──┐
  ├── crm_prd_info      ──> silver.crm_prd_info      ──┼──> gold.dim_customers
  └── crm_sales_details ──> silver.crm_sales_details   ├──> gold.dim_products
                                                       └──> gold.fact_sales
[ ERP System ]                                         │
  ├── erp_cust_az12     ──> silver.erp_cust_az12     ──┤
  ├── erp_loc_a101      ──> silver.erp_loc_a101      ──┘
  └── erp_px_cat_g1v2   ──> silver.erp_px_cat_g1v2   ─────> (Category Lookup)

```

## Data Pipeline Summary Metrics

Below are the verified record processing figures across layers:

| Target Table | Bronze Count | Silver Count | Gold Count | Deduplication & Transformation Notes | 
| ----- | ----- | ----- | ----- | ----- | 
| **`crm_cust_info`** | 18,494 | 18,484 | 18,484 | Filtered 4 null customer IDs; retained latest version of 6 duplicate IDs using `ROW_NUMBER()`. | 
| **`crm_prd_info`** | 397 | 397 | 295 | Silver preserves full version history; Gold filters to 295 current active products (`prd_end_dt IS NULL`). | 
| **`crm_sales_details`** | 60,398 | 60,398 | 60,398 | Recomputed 35 invalid/mismatched sales values using $Sales = Quantity \times \vert{}Price\vert{}$. | 
| **`erp_cust_az12`** | 18,484 | 18,484 | — | Standardized gender codes; sanitized 16 future birthdates (`> GETDATE()`) to `NULL`. Stripped `NAS` prefix. | 
| **`erp_loc_a101`** | 18,484 | 18,484 | — | Normalized country aliases (`US`, `USA`, `DE`) into unified full names (`United States`, `Germany`). | 
| **`erp_px_cat_g1v2`** | 37 | 37 | — | Directly ingested and verified clean. Linked via `cat_id` derived in product staging. | 

## Data Cleansing & Transformation Rules Summary

### 1. Customer Integration (`silver.crm_cust_info`, `silver.erp_cust_az12`, `silver.erp_loc_a101`)

* **Deduplication:** Applied `ROW_NUMBER() OVER (PARTITION BY cst_id ORDER BY cst_create_date DESC)` to select the most recently updated customer profile.

* **Key Standardization:** Stripped text prefixes (`NAS` from `erp_cust_az12.cid`) and deleted inline symbols (hyphens `-` in `erp_loc_a101.cid`) to unify join keys with CRM (`cst_key`).

* **Demographic Enrichment:** Combined CRM and ERP gender records using CRM as primary source and ERP as fallback (`COALESCE`), mapped to explicit strings (`Male`, `Female`, `n/a`).

### 2. Product Lineage & History (`silver.crm_prd_info`)

* **Key Partitioning:** Extracted category code from composite `prd_key` (`CO-RF-FR-R92B-58`) into `cat_id` (`CO_RF`) for mapping to `erp_px_cat_g1v2`.

* **SCD Type 2 Timeline Repair:** Resolved 200 invalid historical records where `prd_end_dt < prd_start_dt` using:

  ```
  LEAD(prd_start_dt) OVER (PARTITION BY prd_key ORDER BY prd_start_dt) - 1
  
  ```

  This creates contiguous valid date ranges per product version.

### 3. Financial Integrity & Sales Processing (`silver.crm_sales_details`)

* **Numeric Date Casting:** Converted integer date representations (`20130511`) into standard SQL `DATE` objects (`CAST(CAST(sls_order_dt AS VARCHAR) AS DATE)`). Out-of-bounds or zero dates are mapped safely to `NULL`.

* **Revenue Calculation Rule:** Recomputed broken or negative metrics enforcing strict mathematical alignment ($Sales = Quantity \times \vert{}Price\vert{}$):

  ```
  CASE 
    WHEN sls_sales IS NULL OR sls_sales <= 0 OR sls_sales != sls_quantity * ABS(sls_price)
    THEN sls_quantity * ABS(sls_price)
    ELSE sls_sales 
  END
  
  ```

## Project Repository Structure

```
.
├── README.md                       # Project architecture, data catalog, and setup guide
├── scripts/
│   ├── 01_init_database.sql        # Database initialization & schema creation (bronze, silver, gold)
│   ├── 02_ddl_bronze.sql           # DDL for Bronze raw storage tables
│   ├── 03_proc_load_bronze.sql     # Bulk ingestion stored procedure for Bronze layer
│   ├── 04_ddl_silver.sql           # DDL for Silver cleaned storage tables
│   ├── 05_proc_load_silver.sql     # Cleansing, ETL transformation procedure for Silver layer
│   └── 06_gold_views.sql           # DDL for Gold Star Schema dimensional model (Views)
└── tests/
    ├── quality_checks_silver.sql   # Data profiling and quality assurance queries for Silver layer
    └── quality_checks_gold.sql     # Referential integrity and uniqueness checks for Gold layer

```

## Installation & Execution Walkthrough

### Prerequisites

* Microsoft SQL Server 2016 or higher (or Azure SQL Database)

* SQL Server Management Studio (SSMS) or Azure Data Studio

* Local read/write permissions for CSV dataset importing

### Setup Steps

1. **Clone the Repository:**

   ```
   git clone https://github.com/your-username/sql-data-warehouse-project.git
   cd sql-data-warehouse-project
   
   ```

2. **Initialize Database & Schemas:**
   Execute `scripts/01_init_database.sql` to create the `DataWarehouse` database alongside the `bronze`, `silver`, and `gold` schemas.

3. **Deploy Tables & Stored Procedures:**
   Run scripts `02_ddl_bronze.sql` through `06_gold_views.sql` in numerical sequence.

4. **Execute Pipeline:**
   Trigger the ingestion and transformation processes by executing the stored procedures:

   ```
   USE DataWarehouse;
   GO
   
   -- Load raw layer from CSV source files
   EXEC bronze.load_bronze;
   
   -- Cleanse, transform, and populate staging layer
   EXEC silver.load_silver;
   GO
   
   ```

5. **Run Verification & Audit Diagnostics:**
   Execute the validation suites located in the `tests/` directory to verify key uniqueness and referential integrity:

   ```
   -- Run Silver & Gold quality check scripts
   :r tests/quality_checks_silver.sql
   :r tests/quality_checks_gold.sql
   
   ```

## Gold Layer Data Catalog

### 1. `gold.dim_customers`

Primary customer dimension consolidating attributes across CRM and ERP systems.

| Column Name | Data Type | Key Type | Description / Business Logic | 
| ----- | ----- | ----- | ----- | 
| `customer_key` | `BIGINT` | **Surrogate Key** | Warehouse unique sequence key (`ROW_NUMBER()`). | 
| `customer_id` | `INT` | Natural Key | Primary CRM customer identification number (`cst_id`). | 
| `customer_number` | `NVARCHAR(50)` | Natural Key | Unique business code (`cst_key` / `cid`), e.g. `AW00011000`. | 
| `first_name` | `NVARCHAR(50)` | Attribute | Customer's first name (trimmed). | 
| `last_name` | `NVARCHAR(50)` | Attribute | Customer's last name (trimmed). | 
| `country` | `NVARCHAR(50)` | Attribute | Residence country, mapped to full country standard names. | 
| `marital_status` | `NVARCHAR(50)` | Attribute | Mapped string values: `Single`, `Married`, or `n/a`. | 
| `gender` | `NVARCHAR(50)` | Attribute | Coalesced gender value (CRM prioritized over ERP): `Male`, `Female`, `n/a`. | 
| `birthdate` | `DATE` | Attribute | Customer birthdate derived from ERP records (future dates set to `NULL`). | 
| `create_date` | `DATE` | Attribute | Customer account creation date in the CRM system. | 

### 2. `gold.dim_products`

Dimension representing current product catalog information.

| Column Name | Data Type | Key Type | Description / Business Logic | 
| ----- | ----- | ----- | ----- | 
| `product_key` | `BIGINT` | **Surrogate Key** | Warehouse unique sequence key (`ROW_NUMBER()`). | 
| `product_id` | `INT` | Natural Key | Primary CRM product ID (`prd_id`). | 
| `product_number` | `NVARCHAR(50)` | Natural Key | Cleansed product key without category prefix (e.g., `FR-R92B-58`). | 
| `product_name` | `NVARCHAR(50)` | Attribute | Commercial product name (`prd_nm`). | 
| `category_id` | `NVARCHAR(50)` | Foreign Key | Extracted category key (`cat_id`) joining to ERP category metadata. | 
| `category` | `NVARCHAR(50)` | Attribute | High-level product category name (e.g., `Bikes`, `Accessories`). | 
| `subcategory` | `NVARCHAR(50)` | Attribute | Product sub-category description. | 
| `maintenance` | `NVARCHAR(50)` | Attribute | Maintenance requirement indicator (`Yes` / `No`). | 
| `cost` | `DECIMAL(18,2)` | Attribute | Unit production cost (defaults to `0` if missing). | 
| `product_line` | `NVARCHAR(50)` | Attribute | Product classification line (`Mountain`, `Road`, `Touring`, `Other Sales`). | 
| `start_date` | `DATE` | Attribute | Product version active start date. | 

### 3. `gold.fact_sales`

Central fact table detailing granular customer purchasing transactions.

| Column Name | Data Type | Key Type | Description / Business Logic | 
| ----- | ----- | ----- | ----- | 
| `order_number` | `NVARCHAR(50)` | Degenerate Key | Transaction order identifier (`sls_ord_num`). | 
| `product_key` | `BIGINT` | **Foreign Key** | References `gold.dim_products.product_key`. | 
| `customer_key` | `BIGINT` | **Foreign Key** | References `gold.dim_customers.customer_key`. | 
| `order_date` | `DATE` | Attribute | Date transaction occurred (converted from `YYYYMMDD` integer). | 
| `shipping_date` | `DATE` | Attribute | Fulfillment date transaction shipped. | 
| `due_date` | `DATE` | Attribute | Payment due date. | 
| `sales_amount` | `DECIMAL(18,2)` | Measure | Net order line amount (validated via $Quantity \times Price$). | 
| `quantity` | `INT` | Measure | Purchased item unit count. | 
| `price` | `DECIMAL(18,2)` | Measure | Per-unit transaction price (sanitized to positive values). | 

## Automated Quality Assurance & Testing Suite

The `tests/` suite verifies critical data model guarantees before BI consumption:

1. **Surrogate Key Uniqueness:**
   Asserts that dimension keys contain no duplicate values.

   ```
   SELECT customer_key, COUNT(*) 
   FROM gold.dim_customers 
   GROUP BY customer_key 
   HAVING COUNT(*) > 1; -- Must return 0 rows
   
   ```

2. **Referential Integrity (Orphan Detection):**
   Confirms every transactional fact maps cleanly to valid customer and product dimensions without dropping facts via `LEFT JOIN` checks.

   ```
   SELECT COUNT(*) AS orphan_count
   FROM gold.fact_sales f
   LEFT JOIN gold.dim_customers c ON c.customer_key = f.customer_key
   LEFT JOIN gold.dim_products p ON p.product_key = f.product_key
   WHERE c.customer_key IS NULL OR p.product_key IS NULL; -- Must return 0
   
   ```

3. **Business Logic Assertions:**
   Checks that no sales records violate non-zero or revenue equation rules.

   ```
   SELECT COUNT(*) 
   FROM silver.crm_sales_details 
   WHERE sls_sales != sls_quantity * sls_price; -- Must return 0
   
   ```

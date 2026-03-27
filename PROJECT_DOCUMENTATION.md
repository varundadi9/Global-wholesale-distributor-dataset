# Global Wholesale Distributor - Technical Documentation

## 📋 Table of Contents
1. [Project Overview](#project-overview)
2. [Architecture](#architecture)
3. [Data Flow](#data-flow)
4. [Database Schema](#database-schema)
5. [KPI Views](#kpi-views)
6. [Notebooks](#notebooks)
7. [How to Use](#how-to-use)

---

## 🎯 Project Overview

The **Global Wholesale Distributor** project is a data analytics solution built on Databricks that processes wholesale distribution data to provide business insights and key performance indicators (KPIs).

### Purpose
- Transform raw business data into actionable insights
- Track sales performance across regions, products, and customers
- Monitor key metrics like revenue, cancellations, and discounts
- Provide a centralized data cube for efficient querying

### Technology Stack
- **Platform**: Databricks (AWS)
- **Storage Format**: Delta Lake
- **Processing**: Apache Spark (PySpark & SQL)
- **Architecture Pattern**: Medallion Architecture (Bronze → Silver → Gold)

---

## 🏗️ Architecture

The project follows the **Medallion Architecture** with three layers:

```
┌─────────────────────────────────────────────────────┐
│                  GOLD LAYER                         │
│  ┌──────────────────┐    ┌──────────────────┐      │
│  │  Dimension Tables│    │   Fact Tables    │      │
│  │  - dim_customer  │    │  - fact_invoice  │      │
│  │  - dim_products  │    │  - fact_payment  │      │
│  │  - dim_regions   │    └──────────────────┘      │
│  └──────────────────┘                               │
│  ┌──────────────────────────────────────────┐      │
│  │        Customer Data Cube                 │      │
│  │  (Pre-joined for fast KPI queries)       │      │
│  └──────────────────────────────────────────┘      │
│  ┌──────────────────────────────────────────┐      │
│  │          KPI Views (8 views)             │      │
│  └──────────────────────────────────────────┘      │
└─────────────────────────────────────────────────────┘
                        ↑
┌─────────────────────────────────────────────────────┐
│                  SILVER LAYER                        │
│   Cleaned & Standardized Data                       │
│   - customer                                         │
│   - products                                         │
│   - invoices                                         │
│   - invoice_line_items                              │
│   - payments                                         │
│   - regions                                          │
│   - exchange_rates                                   │
└─────────────────────────────────────────────────────┘
                        ↑
┌─────────────────────────────────────────────────────┐
│                  BRONZE LAYER                        │
│   Raw Data from Google Drive                        │
│   (01_bronze.google_drive)                          │
└─────────────────────────────────────────────────────┘
```

### Layer Details

#### 🥉 Bronze Layer (01_bronze)
- **Purpose**: Store raw data as-is from source systems
- **Location**: `01_bronze.google_drive`
- **Source**: Google Drive sheets
- **Tables**: customers_sheet_1, products_sheet_1, invoices_sheet_1, etc.

#### 🥈 Silver Layer (02_silver)
- **Purpose**: Clean, standardize, and validate data
- **Location**: `02_silver.silver`
- **Transformations**:
  - Data type conversions
  - Null value handling
  - Duplicate removal
  - Date formatting
  - Text standardization (uppercase, lowercase)

#### 🥇 Gold Layer (03_gold)
- **Purpose**: Business-ready data optimized for analytics
- **Location**: `03_gold.gold`
- **Components**:
  - **Dimension Tables**: Customer, Product, Region reference data
  - **Fact Tables**: Transaction details (invoices, payments)
  - **Data Cube**: Pre-joined table for fast querying
  - **KPI Views**: Pre-calculated business metrics

---

## 🔄 Data Flow

### Step-by-Step Process

1. **Bronze → Silver (Data Cleaning)**
   - Notebook: `NB_Silver_layer`
   - Reads raw data from `01_bronze.google_drive`
   - Applies transformations:
     - Convert dates to standard format
     - Handle null values (replace with defaults)
     - Remove duplicates
     - Standardize text (case conversion)
     - Cast data types correctly
   - Writes to `02_silver.silver` tables

2. **Silver → Gold (Business Layer)**
   - Notebook: `NB_Gold_layer`
   - Creates dimension tables from silver layer
   - Creates fact tables with joins and calculations:
     - Joins invoices with invoice line items
     - Calculates revenue in local currency and USD
     - Joins payments with invoices

3. **Gold → Data Cube**
   - Notebook: `NB_Data_Cube_Query`
   - Joins all dimensions and facts into single table:
     ```
     dim_customer + dim_regions + fact_invoice_line_item + 
     dim_products + fact_payment = customer_data_cube
     ```
   - Benefits:
     - Faster queries (no repeated joins)
     - Simplified KPI calculations
     - Single source for analytics

4. **Data Cube → KPI Views**
   - Notebook: `NB_KPI_Query_With_Data_Cube`
   - Creates 8 KPI views from data cube
   - Each view calculates specific business metrics

---

## 📊 Database Schema

### Dimension Tables

#### dim_customer
```sql
CREATE TABLE 03_gold.gold.dim_customer (
    customer_id      STRING,
    customer_name    STRING,
    customer_type    STRING,
    country          STRING,
    signup_date      DATE
);
```

#### dim_products
```sql
CREATE TABLE 03_gold.gold.dim_products (
    product_id       STRING,
    product_name     STRING,
    category         STRING,
    cost_price       BIGINT
);
```

#### dim_regions
```sql
CREATE TABLE 03_gold.gold.dim_regions (
    country          STRING,
    region_name      STRING,
    region_code      STRING
);
```

### Fact Tables

#### fact_invoice_line_item
```sql
CREATE TABLE 03_gold.gold.fact_invoice_line_item (
    invoice_line_id      STRING,
    invoice_id           STRING,
    product_id           STRING,
    quantity             BIGINT,
    unit_price           DECIMAL(18,2),
    discount             DECIMAL(18,2),
    customer_id          STRING,
    invoice_date         DATE,
    invoice_month        BIGINT,
    currency             STRING,
    region               STRING,
    invoice_status       STRING,
    rate_to_usd          DECIMAL(18,6),
    line_revenue_local   DECIMAL(31,2),  -- Revenue in local currency
    line_revenue_usd     DECIMAL(35,2)   -- Revenue in USD
);
```
**Key Calculation**:
```sql
line_revenue_local = (unit_price - discount) * quantity
line_revenue_usd   = line_revenue_local * rate_to_usd
```

#### fact_payment
```sql
CREATE TABLE 03_gold.gold.fact_payment (
    payment_id       STRING,
    invoice_id       STRING,
    payment_date     DATE,
    payment_amount   DECIMAL(18,2),
    payment_method   STRING,
    customer_id      STRING,
    invoice_status   STRING
);
```

### Data Cube

#### customer_data_cube
```sql
CREATE TABLE 03_gold.gold.customer_data_cube (
    -- Customer dimensions
    customer_id          STRING,
    customer_name        STRING,
    customer_type        STRING,
    signup_date          DATE,
    country              STRING,
    
    -- Region dimensions
    region_name          STRING,
    region_code          STRING,
    region               STRING,
    
    -- Product dimensions
    product_id           STRING,
    product_name         STRING,
    category             STRING,
    cost_price           BIGINT,
    
    -- Invoice facts
    invoice_line_id      STRING,
    invoice_id           STRING,
    invoice_date         DATE,
    invoice_month        BIGINT,
    invoice_status       STRING,
    quantity             BIGINT,
    unit_price           DECIMAL(18,2),
    discount             DECIMAL(18,2),
    currency             STRING,
    rate_to_usd          DECIMAL(18,6),
    line_revenue_local   DECIMAL(31,2),
    line_revenue_usd     DECIMAL(35,2),
    
    -- Payment facts
    payment_id           STRING,
    payment_date         DATE,
    payment_amount       DECIMAL(18,2),
    payment_method       STRING
);
```

---

## 📈 KPI Views

All KPI views query the `customer_data_cube` for optimal performance.

### 1. Total Revenue
```sql
CREATE VIEW kpi_total_revenue_using_data_cube AS
SELECT ROUND(SUM(line_revenue_usd), 2) AS kpi_total_revenue_usd
FROM 03_gold.gold.customer_data_cube;
```
**Purpose**: Calculate total revenue across all transactions in USD.

### 2. Total Quantity Sold
```sql
CREATE VIEW kpi_total_quantity_using_data_cube AS
SELECT SUM(quantity) AS kpi_total_quantity_sold
FROM 03_gold.gold.customer_data_cube;
```
**Purpose**: Count total units sold across all products.

### 3. Average Invoice Value
```sql
CREATE VIEW kpi_avg_invoice_value_using_data_cube AS
SELECT ROUND(AVG(invoice_total_usd), 2) AS kpi_avg_invoice_value_usd
FROM (
    SELECT invoice_id, SUM(line_revenue_usd) AS invoice_total_usd
    FROM 03_gold.gold.customer_data_cube
    GROUP BY invoice_id
);
```
**Purpose**: Calculate average invoice total (sum line items per invoice, then average).

### 4. Top Customers
```sql
CREATE VIEW kpi_top_customers_using_data_cube AS
SELECT
    cd.customer_id,
    c.customer_name,
    ROUND(SUM(cd.line_revenue_usd), 2) AS total_revenue_usd,
    SUM(cd.quantity) AS total_quantity_sold
FROM 03_gold.gold.customer_data_cube cd
LEFT JOIN 03_gold.gold.dim_customer c ON cd.customer_id = c.customer_id
GROUP BY cd.customer_id, c.customer_name;
```
**Purpose**: Rank customers by revenue and quantity purchased.

### 5. Top Products
```sql
CREATE VIEW kpi_top_products_using_data_cube AS
SELECT
    cd.product_id,
    p.product_name,
    p.category,
    ROUND(SUM(cd.line_revenue_usd), 2) AS total_revenue_usd,
    SUM(cd.quantity) AS total_quantity_sold
FROM 03_gold.gold.customer_data_cube cd
LEFT JOIN 03_gold.gold.dim_products p ON cd.product_id = p.product_id
GROUP BY cd.product_id, p.product_name, p.category;
```
**Purpose**: Identify best-selling products by revenue and units.

### 6. Cancellation Rate
```sql
CREATE VIEW kpi_cancellation_rate_using_data_cube AS
SELECT
    COUNT(DISTINCT invoice_id) AS total_invoices,
    COUNT(DISTINCT CASE WHEN invoice_status = 'Cancelled' 
                        THEN invoice_id END) AS cancelled_invoices,
    ROUND(
        COUNT(DISTINCT CASE WHEN invoice_status = 'Cancelled' 
                            THEN invoice_id END) * 100.0 
        / COUNT(DISTINCT invoice_id), 2
    ) AS cancellation_rate_pct
FROM 03_gold.gold.customer_data_cube;
```
**Purpose**: Track percentage of cancelled invoices.

### 7. Discount Impact
```sql
CREATE VIEW kpi_discount_impact_using_data_cube AS
SELECT
    ROUND(SUM(discount * rate_to_usd * quantity), 2) AS kpi_total_discount_given_usd,
    ROUND(AVG(discount * rate_to_usd), 2) AS kpi_avg_discount_per_line_usd
FROM 03_gold.gold.customer_data_cube;
```
**Purpose**: Measure total discounts given and average discount per line item.

### 8. Revenue by Region
```sql
CREATE VIEW kpi_revenue_by_region_using_data_cube AS
SELECT
    region,
    ROUND(SUM(line_revenue_usd), 2) AS total_revenue_usd,
    SUM(quantity) AS total_quantity_sold
FROM 03_gold.gold.customer_data_cube
GROUP BY region;
```
**Purpose**: Compare performance across geographic regions.

### 9. Invoices Per Customer
```sql
CREATE VIEW kpi_invoices_per_customer_using_data_cube AS
SELECT
    cd.customer_id,
    c.customer_name,
    COUNT(DISTINCT cd.invoice_id) AS invoice_count
FROM 03_gold.gold.customer_data_cube cd
LEFT JOIN 03_gold.gold.dim_customer c ON cd.customer_id = c.customer_id
GROUP BY cd.customer_id, c.customer_name;
```
**Purpose**: Track customer purchase frequency.

---

## 📓 Notebooks

### Project Structure
```
Global-wholesale-distributor/
├── 01_bronze/
│   └── (No notebooks - raw data only)
├── 02_silver/
│   └── NB_Silver_layer.ipynb
├── 03_gold/
│   ├── NB_Gold_layer.ipynb
│   ├── NB_Data_Cube_Query.ipynb
│   ├── NB_Gold_layer_KPI's.ipynb
│   └── NB_KPI_Query_With_Data_Cube.ipynb
└── PROJECT_DOCUMENTATION.md (this file)
```

### Notebook Descriptions

#### 1. NB_Silver_layer
- **Location**: `02_silver/`
- **Purpose**: Transform bronze to silver layer
- **Language**: Python (PySpark)
- **What it does**:
  - Reads 7 raw tables from `01_bronze.google_drive`
  - Cleans and standardizes each table
  - Handles nulls, duplicates, and data types
  - Writes 7 clean tables to `02_silver.silver`

**Tables Processed**:
- customers_sheet_1 → customer
- products_sheet_1 → products
- invoices_sheet_1 → invoices
- invoice_line_items_sheet_1 → invoice_line_items
- payments_sheet_1 → payments
- regions_sheet_1 → regions
- exchange_rates_sheet_1 → exchange_rates

#### 2. NB_Gold_layer
- **Location**: `03_gold/`
- **Purpose**: Create dimension and fact tables
- **Language**: SQL
- **What it does**:
  - Creates 3 dimension tables
  - Creates 2 fact tables with joins and calculations
  - Computes revenue in local currency and USD

#### 3. NB_Data_Cube_Query
- **Location**: `03_gold/`
- **Purpose**: Build the customer data cube
- **Language**: Python (PySpark)
- **What it does**:
  - Joins all dimension and fact tables
  - Creates a denormalized table for fast querying
  - Saves as `customer_data_cube`

**Join Logic**:
```python
dim_customer → dim_regions (on country)
             → fact_invoice_line_item (on customer_id)
             → dim_products (on product_id)
             → fact_payment (on customer_id)
```

#### 4. NB_KPI_Query_With_Data_Cube
- **Location**: `03_gold/`
- **Purpose**: Create KPI views from data cube
- **Language**: SQL
- **What it does**:
  - Creates 8 KPI views using `CREATE VIEW`
  - Each view queries the `customer_data_cube`
  - Views are optimized for dashboard consumption

#### 5. NB_Gold_layer_KPI's
- **Location**: `03_gold/`
- **Purpose**: Alternative KPI calculations (without data cube)
- **Language**: SQL
- **Note**: This notebook creates similar KPIs but directly from fact/dimension tables instead of the data cube

---

## 🚀 How to Use

### Prerequisites
- Databricks workspace
- Access to `01_bronze.google_drive` schema with raw data
- Appropriate permissions to create schemas and tables

### Running the Pipeline

#### Option 1: Run All Notebooks (Full Refresh)

1. **Step 1: Silver Layer**
   ```bash
   Run: Global-wholesale-distributor/02_silver/NB_Silver_layer
   ```
   - This reads from bronze and creates silver tables
   - Expected time: 2-3 minutes

2. **Step 2: Gold Layer**
   ```bash
   Run: Global-wholesale-distributor/03_gold/NB_Gold_layer
   ```
   - Creates dimension and fact tables
   - Expected time: 1-2 minutes

3. **Step 3: Data Cube**
   ```bash
   Run: Global-wholesale-distributor/03_gold/NB_Data_Cube_Query
   ```
   - Builds the customer_data_cube
   - Expected time: 1-2 minutes

4. **Step 4: KPI Views**
   ```bash
   Run: Global-wholesale-distributor/03_gold/NB_KPI_Query_With_Data_Cube
   ```
   - Creates all KPI views
   - Expected time: <1 minute

#### Option 2: Query Existing KPIs

If the pipeline has already run, you can directly query the KPI views:

```sql
-- Get total revenue
SELECT * FROM 03_gold.gold.kpi_total_revenue_using_data_cube;

-- Get top customers
SELECT * FROM 03_gold.gold.kpi_top_customers_using_data_cube
ORDER BY total_revenue_usd DESC
LIMIT 10;

-- Get revenue by region
SELECT * FROM 03_gold.gold.kpi_revenue_by_region_using_data_cube
ORDER BY total_revenue_usd DESC;
```

### Querying the Data Cube

The data cube allows flexible custom analysis:

```sql
-- Revenue by customer type
SELECT 
    customer_type,
    ROUND(SUM(line_revenue_usd), 2) AS total_revenue
FROM 03_gold.gold.customer_data_cube
GROUP BY customer_type
ORDER BY total_revenue DESC;

-- Monthly revenue trend
SELECT 
    invoice_month,
    ROUND(SUM(line_revenue_usd), 2) AS monthly_revenue
FROM 03_gold.gold.customer_data_cube
GROUP BY invoice_month
ORDER BY invoice_month;

-- Product category performance by region
SELECT 
    region,
    category,
    ROUND(SUM(line_revenue_usd), 2) AS revenue,
    SUM(quantity) AS units_sold
FROM 03_gold.gold.customer_data_cube
GROUP BY region, category
ORDER BY revenue DESC;
```

### Creating Custom KPIs

To create your own KPI view:

```sql
CREATE VIEW 03_gold.gold.kpi_my_custom_metric AS
SELECT 
    -- your custom aggregations here
FROM 03_gold.gold.customer_data_cube
WHERE -- your filters
GROUP BY -- your dimensions;
```

---

## 🔧 Maintenance & Best Practices

### Data Refresh
- **Frequency**: Run notebooks whenever new data arrives in bronze layer
- **Order**: Always maintain the sequence (Silver → Gold → Data Cube → KPIs)
- **Mode**: Currently using `overwrite` mode for full refresh

### Performance Tips
1. **Use the Data Cube**: Always query the data cube instead of joining tables manually
2. **Filter Early**: Apply WHERE clauses to reduce data scanned
3. **Partition Data**: Consider partitioning large tables by date
4. **Cache Results**: Cache frequently accessed data

### Monitoring
- Check row counts after each notebook run
- Verify no nulls in critical fields (customer_id, product_id, invoice_id)
- Monitor cancellation rates and discount trends

### Troubleshooting

**Problem**: KPI views return no data
- **Solution**: Check if data cube is populated
  ```sql
  SELECT COUNT(*) FROM 03_gold.gold.customer_data_cube;
  ```

**Problem**: Revenue calculations seem incorrect
- **Solution**: Verify exchange rates are loaded
  ```sql
  SELECT * FROM 02_silver.silver.exchange_rates LIMIT 10;
  ```

**Problem**: Duplicate records in data cube
- **Solution**: Check for duplicate joins, review NB_Data_Cube_Query notebook

---

## 📞 Support & Contact

For questions or issues with this project, please contact your data engineering team.

---

**Last Updated**: March 27, 2026  
**Version**: 1.0  
**Maintained By**: Databricks Assistant
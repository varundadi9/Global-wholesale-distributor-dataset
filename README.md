# Global Wholesale Distributor 📦

A data analytics pipeline for wholesale distribution business intelligence built on Databricks.

## ⚡ Quick Start

### Run the Full Pipeline

Execute notebooks in this order:

```
1. 02_silver/NB_Silver_layer          → Clean raw data
2. 03_gold/NB_Gold_layer              → Create dimensions & facts
3. 03_gold/NB_Data_Cube_Query         → Build data cube
4. 03_gold/NB_KPI_Query_With_Data_Cube → Generate KPI views
```

### Query KPIs

```sql
-- Total Revenue
SELECT * FROM 03_gold.gold.kpi_total_revenue_using_data_cube;

-- Top 10 Customers
SELECT * FROM 03_gold.gold.kpi_top_customers_using_data_cube
ORDER BY total_revenue_usd DESC LIMIT 10;

-- Revenue by Region
SELECT * FROM 03_gold.gold.kpi_revenue_by_region_using_data_cube;
```

## 🏗️ Architecture

**Medallion Pattern**: Bronze → Silver → Gold

* **Bronze**: Raw data from Google Drive (`01_bronze.google_drive`)
* **Silver**: Cleaned & standardized (`02_silver.silver`)
* **Gold**: Business-ready analytics (`03_gold.gold`)

## 📊 Available KPIs

| KPI | View Name | Description |
|-----|-----------|-------------|
| 💵 Total Revenue | `kpi_total_revenue_using_data_cube` | Sum of all revenue in USD |
| 📦 Total Quantity | `kpi_total_quantity_using_data_cube` | Total units sold |
| 💳 Avg Invoice Value | `kpi_avg_invoice_value_using_data_cube` | Average invoice amount |
| 👥 Top Customers | `kpi_top_customers_using_data_cube` | Customer revenue ranking |
| 📦 Top Products | `kpi_top_products_using_data_cube` | Product performance |
| ❌ Cancellation Rate | `kpi_cancellation_rate_using_data_cube` | % of cancelled invoices |
| 🏷️ Discount Impact | `kpi_discount_impact_using_data_cube` | Total discounts given |
| 🌍 Revenue by Region | `kpi_revenue_by_region_using_data_cube` | Regional performance |
| 📄 Invoices per Customer | `kpi_invoices_per_customer_using_data_cube` | Purchase frequency |

## 📁 Key Tables

### Dimensions
* `dim_customer` - Customer reference data
* `dim_products` - Product catalog
* `dim_regions` - Geographic regions

### Facts
* `fact_invoice_line_item` - Transaction details
* `fact_payment` - Payment information

### Data Cube
* `customer_data_cube` - Pre-joined table for fast queries

## 📘 Documentation

For detailed documentation, see [PROJECT_DOCUMENTATION.md](./PROJECT_DOCUMENTATION.md)

## 🔍 Sample Queries

### Custom Analysis on Data Cube

```sql
-- Revenue by customer type
SELECT 
    customer_type,
    ROUND(SUM(line_revenue_usd), 2) AS revenue
FROM 03_gold.gold.customer_data_cube
GROUP BY customer_type;

-- Monthly trend
SELECT 
    invoice_month,
    ROUND(SUM(line_revenue_usd), 2) AS revenue
FROM 03_gold.gold.customer_data_cube
GROUP BY invoice_month
ORDER BY invoice_month;

-- Product performance by region
SELECT 
    region,
    category,
    ROUND(SUM(line_revenue_usd), 2) AS revenue
FROM 03_gold.gold.customer_data_cube
GROUP BY region, category
ORDER BY revenue DESC;
```

## 🔧 Project Structure

```
Global-wholesale-distributor/
├── README.md                          ← You are here
├── PROJECT_DOCUMENTATION.md          ← Full technical docs
├── 01_bronze/                        ← Raw data layer
├── 02_silver/                        ← Cleaned data layer
│   └── NB_Silver_layer.ipynb
└── 03_gold/                          ← Business layer
    ├── NB_Gold_layer.ipynb
    ├── NB_Data_Cube_Query.ipynb
    ├── NB_Gold_layer_KPI's.ipynb
    └── NB_KPI_Query_With_Data_Cube.ipynb
```

## ✅ Best Practices

1. **Always use the data cube** for queries instead of joining tables manually
2. **Run notebooks in sequence** to maintain data consistency
3. **Filter data** with WHERE clauses to improve performance
4. **Check row counts** after each notebook run for validation

## ⚠️ Troubleshooting

**No data in KPIs?**
```sql
SELECT COUNT(*) FROM 03_gold.gold.customer_data_cube;
```
If returns 0, rerun the data cube notebook.

**Revenue looks wrong?**
```sql
SELECT * FROM 02_silver.silver.exchange_rates LIMIT 10;
```
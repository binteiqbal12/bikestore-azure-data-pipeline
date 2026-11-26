# BikeStore Azure Data Pipeline

An end-to-end data engineering project that moves the BikeStore sales data from a source database to an interactive Power BI dashboard, using the **medallion architecture** (Bronze, Silver, Gold) on Azure.

## Architecture

```
SQL Server (on-premises)
        |  Self-hosted Integration Runtime + Azure Key Vault (secrets)
        v
Azure Data Factory  -->  ADLS Gen2: bronze   (raw CSV copy of source tables)
                                |
                                v   Azure Databricks (PySpark)
                         ADLS Gen2: silver   (cleaned, validated Delta tables)
                                |
                                v   Azure Databricks (PySpark)
                         ADLS Gen2: gold     (business-ready Delta tables)
                                |
                                v
                  Azure Synapse serverless SQL (views over Gold)
                                |
                                v
                         Power BI dashboard
```

| Service | Role |
|---------|------|
| SQL Server | Source database holding the BikeStore tables |
| Self-hosted Integration Runtime | Lets Data Factory reach the on-premises source securely |
| Azure Key Vault | Stores the SQL credentials as a secret, retrieved at runtime |
| Azure Data Factory | Parameterised pipeline that copies all source tables into Bronze |
| ADLS Gen2 | Storage for the Bronze, Silver and Gold layers (`bikestore` container) |
| Azure Databricks | PySpark notebooks that build Silver and Gold |
| Azure Synapse (serverless SQL) | Exposes the Gold tables as SQL views |
| Power BI | Dashboard connected to Synapse serverless SQL |

## Data layers

**Bronze** holds raw copies of the source tables, unchanged.

- `sales`: customers, orders, order_items, staffs, stores
- `production`: brands, categories, products, stocks

**Silver** holds cleaned and validated data (Delta). The cleaning notebook:

- converts column names to `snake_case`
- trims whitespace, collapses repeated spaces, and turns empty strings into nulls
- removes duplicate rows
- removes rows only where a key column is null (other columns, such as `phone` or `shipped_date`, may legitimately be null)
- logs rows read, written and removed for every table

**Gold** holds business-ready tables (Delta):

| Table | Description |
|-------|-------------|
| `daily_sales` | Orders, units and revenue per day |
| `monthly_sales` | Orders, units and revenue per month |
| `top_products` | Revenue and rank per product |
| `customer_lifetime_value` | Orders and lifetime revenue per customer, with city and state |

Revenue is calculated as `quantity * list_price * (1 - discount)`.

### Data quality checks (Gold notebook)

1. Order dates are valid: not null, not in the future, and the required date is on or after the order date.
2. Quantities and prices are positive, and discount is between 0 and 1.
3. Every order item belongs to a valid order and product (referential integrity).
4. Rejected orders (status 3) are excluded from revenue.

Rows read, rejected and written are logged at each step and saved to `gold/_pipeline_log`.

## Dashboard

The Power BI report connects to Synapse serverless SQL and includes:

- a **Total Revenue** KPI card
- a **year slicer**
- **Monthly Revenue Trend**
- **Revenue by State**
- **Top 10 Products by Revenue**
- **Top 10 Customers by Lifetime Value**

The year slicer filters the revenue trend and the KPI card. The product, state and customer visuals show all-time results because their Gold tables are aggregated across all years.

## Repository structure

```
bikestore-azure-data-pipeline/
├── README.md
├── notebooks/
│   ├── 01_silver_cleaning.py
│   └── 02_gold_aggregation.py
├── sql/
│   ├── synapse_views.sql
│   └── analytical_queries.sql
├── dashboard/
│   ├── bikestore_gold.pbix
│   ├── bikestore_background.png
│   └── dashboard_screenshot.png
└── docs/
    └── security_reflection.md
```

## How to run

1. **Ingest:** run the Data Factory pipeline to copy the source tables into the `bronze` container.
2. **Silver:** run `notebooks/01_silver_cleaning.py` in Databricks.
3. **Gold:** run `notebooks/02_gold_aggregation.py` in Databricks. If you re-run it after changing Silver, delete the old Gold table folders first so Synapse does not read stale files.
4. **Serve:** run `sql/synapse_views.sql` in Synapse Studio against the `bikestore_gold` database.
5. **Report:** open `dashboard/bikestore_gold.pbix` in Power BI Desktop and refresh (**Home**, **Refresh**, **Schema and data**).

## Security notes

- No credentials are stored in code or pipeline configuration. SQL credentials live in Azure Key Vault and are retrieved at runtime.
- Analysts should be granted read access to the Gold layer only, using Azure RBAC and ADLS ACLs, with no access to Bronze.

## Tech stack

Azure Data Factory, Azure Key Vault, ADLS Gen2, Azure Databricks (PySpark, Delta Lake), Azure Synapse serverless SQL, Power BI.

## Author

Your name here

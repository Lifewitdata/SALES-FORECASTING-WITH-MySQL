<!-- Header -->
<div align="center">

```
┌─────────────────────────────────────────────────────────────────────┐
│                                                                     │
│   📦  SALES FORECASTING WITH MySQL DATA PREP                        │
│   SQL Cleaning → Feature Engineering → Prophet → Q1 2025 Forecast  │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

![MySQL](https://img.shields.io/badge/MySQL-Compatible-4479A1?style=flat-square&logo=mysql&logoColor=white)
![Python](https://img.shields.io/badge/Python-3.11-3776AB?style=flat-square&logo=python&logoColor=white)
![Prophet](https://img.shields.io/badge/Prophet-Forecasting-FF6F00?style=flat-square)
![sklearn](https://img.shields.io/badge/scikit--learn-Evaluation-F7931E?style=flat-square&logo=scikit-learn&logoColor=white)
![Status](https://img.shields.io/badge/Status-Complete-34A853?style=flat-square)

</div>

---

## Why This Project Exists

Most forecasting tutorials skip the hardest part — getting the data ready.

In real analyst work at a company like Cube Asia, 60% of the job is **SQL**: connecting to a database, hunting down dirty data, deduplicating rows, imputing nulls, engineering time-based features — all before a single model is trained.

This project does the whole thing, end to end. SQL first, model second.

> **Business question answered:** *"Which product categories will generate the highest GMV in Q1 2025 — and by how much? Help brand clients plan inventory and promo budgets before the Ramadan surge."*

---

## Database Schema (Star Schema)

```
                    ┌──────────────────┐
                    │   dim_category   │
                    │  category_id  PK │
                    │  category_name   │
                    │  parent_cat      │
                    └────────┬─────────┘
                             │
┌──────────────┐    ┌────────▼──────────┐    ┌──────────────────┐
│ dim_platform │    │   fact_sales_raw  │    │   dim_brand      │
│ platform_id  ├───►│   (75,500 rows)   │◄───┤   brand_id    PK │
│ platform_name│    │   + dupes + nulls │    │   brand_name     │
│ country      │    └────────┬──────────┘    │   origin         │
└──────────────┘             │               └──────────────────┘
                    ┌────────▼──────────┐
                    │   fact_sales      │
                    │   (CLEAN table)   │
                    │   post dedup +    │
                    │   imputation      │
                    └────────┬──────────┘
                             │
                    ┌────────▼──────────┐
                    │ forecast_q1_2025  │
                    │  model outputs    │
                    │  pushed back → DB │
                    └───────────────────┘
```

---

## The Full Pipeline (15 Steps)

```
[MySQL DB] ──► [SQL Audit] ──► [SQL Clean] ──► [Feature Engineering]
                                                        │
                                                        ▼
                                              [Prophet Model × 5 cats]
                                                        │
                                                        ▼
                                     [MAE / RMSE / MAPE Evaluation]
                                                        │
                                                        ▼
                                        [Q1 2025 Forecast → DB Table]
```

| Step | Type | What Happens |
|------|------|--------------|
| 01 | Python | Connect to MySQL, helper `run_sql()` function |
| 02 | **SQL** | `SHOW TABLES`, `PRAGMA table_info` — schema audit |
| 03 | **SQL** | JOIN across all 4 tables — raw data preview |
| 04 | **SQL** | NULL audit — `CASE WHEN IS NULL` per column |
| 05 | **SQL** | Duplicate detection — `GROUP BY + HAVING COUNT(*) > 1` |
| 06 | **SQL + Python** | Dedup via `ROW_NUMBER()`, median imputation per category |
| 07 | **SQL** | Feature engineering — monthly GMV, AOV, active SKUs, seasonality flags |
| 08 | Python | EDA — trend lines, seasonality heatmap |
| 09 | Python | Prophet model — train 2022–2023, test 2024, forecast Q1 2025 |
| 10 | Python | Evaluation — MAE, RMSE, MAPE with bar charts |
| 11 | Python | Forecast plot — actual vs predicted + confidence interval |
| 12 | Python | Q1 2025 forecast table — client-deliverable format |
| 13 | **SQL** | Push forecast → `forecast_q1_2025` table in MySQL |
| 14 | **SQL** | Bonus queries — YoY growth, platform share, top brands by GMV |
| 15 | Markdown | Client insights summary — action table |

---

## SQL Highlights

**Duplicate detection using window function:**
```sql
SELECT *, ROW_NUMBER() OVER (
    PARTITION BY sale_date, platform_id, category_id, brand_id, sku_code, price_idr, units_sold
    ORDER BY sale_id
) AS rn
FROM fact_sales_raw
```

**NULL audit across all columns in one query:**
```sql
SELECT
    SUM(CASE WHEN rating        IS NULL THEN 1 ELSE 0 END) AS null_rating,
    SUM(CASE WHEN review_count  IS NULL THEN 1 ELSE 0 END) AS null_review_count,
    SUM(CASE WHEN discount_pct  IS NULL THEN 1 ELSE 0 END) AS null_discount
FROM fact_sales_raw
```

**Year-over-Year GMV growth:**
```sql
SELECT category_name,
    SUM(CASE WHEN YEAR(sale_date) = 2023 THEN gmv_idr END) AS gmv_2023,
    SUM(CASE WHEN YEAR(sale_date) = 2024 THEN gmv_idr END) AS gmv_2024,
    ROUND((gmv_2024 - gmv_2023) / gmv_2023 * 100, 1)       AS yoy_pct
FROM fact_sales JOIN dim_category USING(category_id)
GROUP BY category_name
```

**Platform share using window function:**
```sql
ROUND(SUM(gmv_idr) * 100.0
      / SUM(SUM(gmv_idr)) OVER (PARTITION BY category_name), 1) AS pct_share
```

---

## Forecast Results — Q1 2025

| Category | Jan 2025 | Feb 2025 | Mar 2025 | Q1 Total | MAPE |
|---|---|---|---|---|---|
| Fashion & Apparel | High | **Peak** (Ramadan) | High | 🥇 #1 | ~12% |
| Electronics | Medium | Medium | Medium | 🥈 #2 | ~26% |
| Beauty & Personal Care | Medium | High | High | 🥉 #3 | ~14% |
| Home & Living | Low | Medium | Medium | #4 | ~15% |
| Health & Wellness | Low | Low | Medium | #5 | ~18% |

> **Note:** Electronics has the highest MAPE — expected, as GMV is highly campaign-driven (flash sales, platform vouchers). Upper-bound CI recommended for inventory planning.

---

## Data Cleaning Summary

```
Raw rows (fact_sales_raw)  :  75,500
After deduplication        :  75,000   (-500 duplicate rows removed)
After NULL imputation      :  75,000   (rating & reviews filled with category median)
Final clean rows           :  75,000    ready for feature engineering
```

---

## Tech Stack

```python
sqlite3 / mysql-connector   # Database connection (MySQL-compatible)
pandas                      # SQL results → DataFrames, feature engineering
prophet                     # Time-series forecasting (Meta)
scikit-learn                # MAE, RMSE, MAPE evaluation
matplotlib / seaborn        # Trend charts, heatmaps, forecast plots
jupyter                     # Analyst notebook workflow
```

---

## How to Run

```bash
# Clone
git clone https://github.com/Lifewitdata/sales-forecasting-mysql-dataprep.git
cd sales-forecasting-mysql-dataprep

# Install
pip install pandas numpy matplotlib seaborn prophet scikit-learn jupyter



# Launch
jupyter notebook Sales_Forecasting_MySQL_DataPrep.ipynb
```

---

## Folder Structure

```
sales-forecasting-mysql-dataprep/
│
├── Sales_Forecasting_MySQL_DataPrep.ipynb   ← Main notebook
├── asia_ecommerce.db                   ← SQLite DB (MySQL-compatible schema)
├── outputs/
│   ├── null_audit.png
│   ├── gmv_trend_eda.png
│   ├── model_evaluation.png
│   ├── forecast_plot.png
│   └── q1_2025_forecast.png
└── README.md
```

---



---

<div align="center">
<sub>Built as part of a Data Analyst portfolio targeting e-commerce intelligence roles in Southeast Asia.</sub>
</div>

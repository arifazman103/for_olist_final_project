# Olist Delivery Analytics — End-to-End Fabric Pipeline

An end-to-end data engineering and analytics pipeline built on the [Olist Brazilian E-Commerce Public Dataset](https://www.kaggle.com/datasets/olistbr/brazilian-ecommerce), using Microsoft Fabric (Lakehouse, PySpark notebooks, Power BI) to answer one core question:

> **What drives late deliveries in Olist's logistics network — and can we predict/flag it before it happens?**

## Architecture

Medallion (Bronze → Silver → Gold) architecture built entirely in Microsoft Fabric:

```
Raw CSVs (9 source files)
        │
        ▼
 ┌─────────────┐      ┌─────────────┐      ┌─────────────┐
 │   BRONZE    │ ───▶ │   SILVER    │ ───▶ │    GOLD     │ ───▶ Power BI
 │  Raw ingest │      │ Clean + FK  │      │ Star schema │      Report
 │  (Delta)    │      │ validated   │      │  (Delta)    │
 └─────────────┘      └─────────────┘      └─────────────┘
```

| Layer | Notebook | What it does |
|---|---|---|
| 🥉 Bronze | `bronze_ingestion_notebook.ipynb` | Reads 9 raw CSVs (orders, customers, geolocation, order items, payments, reviews, products, sellers, category translations) from `Files/raw_csv/` and writes each as a Delta table with schema inference |
| 🥈 Silver | `silver_transformation_notebook.ipynb` | Cleans and validates every table: trims whitespace, nullifies empty strings, deduplicates on business keys, removes orphaned foreign keys (e.g. reviews with no matching order), collapses geolocation to one row per zip code, joins orders + items + products + customers + sellers into an analysis table, and enriches it with delivery delay (days), Haversine distance between customer/seller, interstate flag, and purchase-date breakdowns (year/month/quarter/weekend) |
| 🥇 Gold | `gold_final_notebook.ipynb` | Builds a star schema: `dim_date`, `dim_customer`, `dim_seller`, `dim_product`, and `fact_delivery` — with delivery status, delay buckets, distance bands, and weight bands ready for BI consumption |

## Star Schema

![Star Schema](Screenshots/star_schema.png)

**`gold_fact_delivery`** (grain: one row per order item) joins to four dimensions on `customer_id`, `seller_id`, `product_id`, and `purchase_date`.

## Key Results

From the Power BI report, built on top of the gold layer (~100K orders):

| Metric | Value |
|---|---|
| Total Orders | 100K |
| Late Delivery Rate | 6.65% |
| Avg Delay (late orders) | 10.64 days |
| Avg Freight Value | 20.11 |

**Findings surfaced in the root cause analysis:**
- Late delivery rate varies sharply by state — e.g. SP (largest volume, 16.5K orders) sits at 4.41%, while several lower-volume northeastern states run far higher
- Longer delivery distances and specific customer↔seller routes correlate with higher lateness
- Freight value and parcel weight both show a relationship with delay likelihood, explored further in the PDF report

Full breakdown: see [`Reports/Olist_Delivery_Root_Cause_Analysis.pdf`](Reports/Olist_Delivery_Root_Cause_Analysis.pdf) and [`Reports/PowerBI_Chart_OlistBR_data_visual.pdf`](Reports/PowerBI_Chart_OlistBR_data_visual.pdf).

## Tech Stack

- **Ingestion & Transformation**: PySpark (Microsoft Fabric Notebooks)
- **Storage**: Delta Lake tables in Fabric Lakehouse (Bronze / Silver / Gold)
- **Modeling**: Star schema, Fabric Semantic Model
- **Visualization**: Power BI (Fabric-native report)
- **Data source**: [Olist Brazilian E-Commerce Public Dataset](https://www.kaggle.com/datasets/olistbr/brazilian-ecommerce) (Kaggle)

## Repo Structure

```
├── Notebooks/
│   ├── bronze_ingestion_notebook.ipynb
│   ├── silver_transformation_notebook.ipynb
│   └── gold_final_notebook.ipynb
├── Reports/
│   ├── Olist_Delivery_Root_Cause_Analysis.pdf
│   └── PowerBI_Chart_OlistBR_data_visual.pdf
├── Screenshots/
│   ├── star_schema.png
│   ├── delivery_semantic_model.png
│   └── tables_gold_1-5.png
└── README.md
```

> Note: Raw source CSVs and the Bronze-layer data export are not included in this repo due to file size — the dataset is publicly available on Kaggle at the link above. Cleaned Silver/Gold layer schemas are documented above and visible in the notebooks.

## Status

- [x] Data ingestion (Bronze)
- [x] Data cleaning & validation (Silver)
- [x] Star schema modeling (Gold)
- [x] Power BI reporting & root cause analysis

## Author

**Arif Azman** — [github.com/arifazman103](https://github.com/arifazman103)

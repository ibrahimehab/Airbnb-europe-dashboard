# Airbnb Prices in European Cities — Medallion Architecture Data Warehouse

An end-to-end data warehouse project built on the [Airbnb Prices in European Cities] dataset, following the **Medallion Architecture** pattern (Bronze → Silver → Gold) to go from raw CSVs to a queryable star schema and a Power BI dashboard.

## Architecture Overview

```
raw_source_files/
        │
        ▼
┌───────────────────┐
│   Bronze Layer     │  Raw ingestion — no cleaning, no transformation
└───────────────────┘
        │
        ▼
┌───────────────────┐
│   Silver Layer     │  Cleaning + web-scraped feature enrichment + master dataset
└───────────────────┘
        │
        ▼
┌───────────────────┐
│    Gold Layer      │  Star schema (Fact + Dimension tables) in SQL Server
└───────────────────┘
        │
        ▼
┌───────────────────┐
│  Power BI Dashboard │  Final reporting layer
└───────────────────┘
```

## Repository Structure

```
airbnb-europe-dwh/
│
├── 0th raw_source_files/     # 20 original CSVs (10 cities × weekdays/weekends)
├── 1st Bronze Layer/          # Raw ingestion scripts + tagged output
├── 2nd Silver Layer/          # Cleaning, scraping, master dataset build
├── 3rd Gold Layer/            # SQL scripts building the star schema
├── 4th PowerBI Dashboard/     # Final .pbix report
├── other_scripts/             # Web scraping notebooks + exploratory queries
└── README.md
```

## Dataset

The source data covers Airbnb listings across **10 European cities** (Athens, Barcelona, Berlin, Budapest, Lisbon, London, Paris, Rome, Vienna, Amsterdam), split into weekday and weekend pricing — 20 CSV files, ~51,700 listings combined.

Original columns include price (`realSum`), room type, host flags (superhost/multi-listing/business), cleanliness and guest satisfaction ratings, bedroom count, distance to city center/metro, attraction and restaurant proximity indexes, and coordinates.

## Layer-by-Layer Breakdown

### 1️⃣ Bronze Layer
- Ingests all 20 raw CSVs **as-is**, with zero cleaning.
- Tags every row with `city` and `day_type` (parsed from the filename), since this metadata isn't present in the raw files themselves.
- Output: 20 tagged CSVs, later combined into a single raw dataset for downstream processing.

**Run:** open `1st Bronze Layer/bronze_ingest.ipynb` and run all cells.

### 2️⃣ Silver Layer
- Combines all Bronze files into a single dataset.
- Validates data quality: null checks, duplicate checks, dtype checks, categorical sanity checks.
- Runs a correlation analysis against price to evaluate the predictive strength of existing features — this justified the decision to scrape supplementary data (correlations topped out at ~0.29, indicating the existing feature set does not explain most of the price variance).
- Scrapes additional listing features via Selenium (amenities, number of reviews, host type, rating, property type, location details, availability, cleaning/service fees), since the source dataset has no listing ID/URL to join against directly.
- Matches scraped listings back to the original dataset using nearest-neighbor matching on city, coordinates, price, and room type.
- Outputs a single cleaned, enriched **master dataset**.

**Run:** open `other_scripts/exploratory_analysis.ipynb` first, then `other_scripts/scrape_airbnb_features.ipynb`, then `2nd Silver Layer/build_master_dataset.ipynb`.

### 3️⃣ Gold Layer
- Builds a **star schema** in SQL Server from the Silver master dataset.
- **Fact table:** `fact_listings` — one row per listing, holding price and other numeric measures, plus foreign keys to every dimension.
- **Dimension tables:** `dim_city`, `dim_room`, `dim_host`, `dim_location`, `dim_attraction`, `dim_date`.
- Surrogate keys are generated using `DENSE_RANK()`, computed identically across dimension-build and fact-build queries, to avoid unreliable floating-point equality joins.

**Run:** in SQL Server Management Studio, execute the scripts in `3rd Gold Layer/` in order (schema creation → data load → star schema population).

### 4️⃣ Power BI Dashboard
- Connects directly to the Gold layer star schema.
- Visualizes price trends by city, room type, weekday/weekend, host type, and amenity availability.

**Run:** open `4th PowerBI Dashboard/airbnb_dashboard.pbix` in Power BI Desktop and refresh the data connection.

## Tech Stack

- **Python** (pandas, numpy) — data ingestion, cleaning, exploratory analysis
- **Selenium** — dynamic web scraping (Airbnb pages are JavaScript/React-rendered)
- **SQL Server / T-SQL** — star schema design and population
- **Power BI** — final reporting and visualization

## Key Design Decisions

- **No listing ID in source data** → scraped features are matched back via nearest-neighbor (city + coordinates + price + room type) rather than a direct join.
- **Location and attraction dimensions are near-unique per listing** — a known tradeoff of modeling continuous numeric fields (coordinates, distance indexes) as dimensions rather than fact-table attributes. Documented here rather than hidden, since it's a deliberate scope decision for this project.
- **DENSE_RANK() over float-equality joins** — early attempts to join the fact table back to dimension tables using `=` on `FLOAT` columns silently failed (0 rows matched) due to floating-point precision issues. Surrogate keys are now generated inline and consistently, avoiding the join entirely.

## How to Run the Full Pipeline

1. Place the 20 source CSVs in `0th raw_source_files/`.
2. Run the Bronze ingestion notebook.
3. Run the exploratory analysis notebook to reproduce the correlation findings.
4. Run the scraping notebook (requires Chrome + Selenium).
5. Run the Silver master dataset build notebook.
6. Execute the Gold layer SQL scripts in SQL Server Management Studio, in order.
7. Open the Power BI dashboard and refresh.

## Notes on Scraping

Scraping was done for educational purposes as part of a coursework project. Requests were kept limited in volume and paced with delays to minimize load on the source site.

# Quantifying the "Infrastructure Premium" in the Sydney Rental Market 🏙️📈

## Table of Contents

- [Overview](#overview)
- [Project Structure](#project-structure)
- [Prerequisites](#prerequisites)
- [Setup](#setup)
- [Data Requirements](#data-requirements)
- [Usage](#usage)
- [Methodology](#methodology)
- [Machine Learning Validation](#machine-learning-validation)
- [Outputs](#outputs)
- [References & Data Sources](#references--data-sources)

---

## Overview

This project investigates the relationship between neighborhood infrastructure (transit, schools, business density, points of interest, crime) and median weekly rent across Greater Sydney. By engineering a custom spatial **Liveability Index** and deploying gradient boosting models, this analysis quantifies exactly how much of a suburb's rental price is driven by its surrounding amenities versus overarching macroeconomic noise.

Crucially, this model explicitly **excludes Household Income**, forcing the algorithm to determine the pure **"Infrastructure Premium"** of a neighborhood independently of local wealth.

---

## Project Structure

The analysis is separated into two core Jupyter Notebooks to ensure a clear separation of concerns between data engineering and machine learning.

**`Sydney_Liveability.ipynb` — Data Engineering & Spatial Pipeline**

- Establishes a secure connection to a local PostgreSQL/PostGIS database using `jupysql` and `.env` variables.
- Performs complex area-weighted spatial joins to map raw infrastructure data to official ABS SA2 boundaries.
- Calculates standardised Liveability Z-scores to mitigate spatial bias.
- Generates an interactive geospatial visualisation using `plotly`.

**`Sydney_Liveability_ML.ipynb` — Demographic Fusion & Predictive Modeling**

- Fuses the SQL-engineered Z-scores with ground-truth economic data from the 2021 ABS Census Datapacks.
- Trains and optimises Random Forest and XGBoost regressors to predict Median Weekly Rent.
- Extracts feature importances and runs rigorous Shuffled 5-Fold Cross-Validation.

---

## Prerequisites

To run this project locally, you will need the following installed:

- **Python** (3.11+ recommended)
- **PostgreSQL** with the PostGIS spatial extension enabled

---

## Setup

**1. Clone the repository**

```bash
git clone https://github.com/yourusername/sydney-liveability-ml.git
cd sydney-liveability-ml
```

**2. Install dependencies**

It is recommended to use a virtual environment. Install the required packages via `requirements.txt`:

```bash
pip install -r requirements.txt
```

> Requires: `pandas`, `geopandas`, `sqlalchemy`, `psycopg2`, `xgboost`, `scikit-learn`, `jupysql`, `python-dotenv`, `plotly`

**3. Secure the database credentials**

Create a file named `project.env` in the root directory. Ensure `project.env` is added to your `.gitignore` before committing any code.

Add your local PostgreSQL connection string:

```
DATABASE_URL=postgresql://username:password@localhost:5432/greater_sydney
```

---

## Data Requirements

Place the following datasets into the `data/` folder before running the notebooks. The pipeline handles all database ingestion automatically when you run the cells.

| File or Folder | Description | Source |
| --- | --- | --- |
| `SA2_2021_AUST_SHP_GDA2020/` | ABS SA2 boundary shapefile | [ABS Digital Boundary Files](https://www.abs.gov.au/statistics/standards/australian-statistical-geography-standard-asgs-edition-3/jul2021-jun2026/access-and-downloads/digital-boundary-files) |
| `SAL_2021_AUST_GDA2020_SHP/` | ABS Suburb/Locality boundary shapefile | [ABS Digital Boundary Files](https://www.abs.gov.au/statistics/standards/australian-statistical-geography-standard-asgs-edition-3/jul2021-jun2026/access-and-downloads/digital-boundary-files) |
| `catchments/` | NSW school catchment shapefiles | [NSW Education Data](https://data.nsw.gov.au/data/dataset/school-catchment-areas-for-2021) |
| `Businesses.csv` | Business counts by SA2 and industry | [ABS Business Register](https://www.abs.gov.au/statistics/economy/business-indicators/counts-australian-businesses-including-entries-and-exits/latest-release) |
| `Income.csv` | Income statistics by SA2 | [ABS Census DataPacks](https://www.abs.gov.au/census/find-census-data/datapacks) |
| `Population.csv` | Population by age group and SA2 | [ABS Census DataPacks](https://www.abs.gov.au/census/find-census-data/datapacks) |
| `Stops.txt` | GTFS-style public transport stop locations | [Transport for NSW Open Data](https://opendata.transport.nsw.gov.au/) |
| `SuburbData25Q3.csv` | NSW Police crime data by suburb (2025 Q3) | [BOCSAR Crime Statistics](https://www.bocsar.nsw.gov.au/Pages/bocsar_crime_stats/bocsar_local_health_district_stats.aspx) |
| `2021Census_G02_NSW_SA2.csv` | ABS Census G02 — median rent, income, household stats | [ABS Census DataPacks](https://www.abs.gov.au/census/find-census-data/datapacks) |

> POIs are fetched live from the [NSW Government Spatial Services API](https://maps.six.nsw.gov.au/arcgis/rest/services/public/NSW_POI/MapServer) at runtime — no local file required.

---

## Usage

1. Launch Jupyter Notebook or JupyterLab.
2. Open and run `Sydney_Liveability.ipynb` from top to bottom. This executes the spatial joins, calculates the Z-scores, saves them to the database, and renders the interactive Plotly map.
3. Open and run `Sydney_Liveability_ML.ipynb` from top to bottom. This merges the spatial data with the ABS Census CSV, trains the models, and outputs the final cross-validation metrics.

---

## Methodology

### 1. Area-Weighted Spatial Interpolation

Solving the **"Spatial Mismatch"** between NSW Police reporting boundaries (Suburbs) and ABS Census boundaries (SA2s) required sub-dividing geometries. To prevent map projection distortions, geometries were cast to PostGIS's native `geography` type, ensuring overlap ratios were calculated over the true spherical shape of the Earth in exact square meters rather than distorted square degrees.

### 2. Z-Score Standardisation & The MAUP

Raw counts of infrastructure inherently bias physically larger suburbs. To counteract the **Modifiable Areal Unit Problem (MAUP)**, all counts were converted to densities and normalised using Z-score standardisation. This allows the models to fairly compare the infrastructure profile of high-density inner-city SA2s against sprawling western SA2s.

---

## Machine Learning Validation

- **Baseline Model (Random Forest)** — Established an initial ensemble baseline, identifying Crime Rate and Business Density as the primary infrastructure proxies for market value.
- **Model Optimisation (XGBoost)** — Tuned an XGBoost Regressor with constrained depth (`max_depth=2`, `learning_rate=0.05`). This shallow tree architecture minimised spatial overfitting, allowing the model to learn general city-wide trends rather than memorising localised geographic noise.
- **Robustness Testing** — Implemented Shuffled 5-Fold Cross-Validation. Shuffling was enforced to break geographic clustering (e.g. preventing a single fold from containing exclusively luxury coastal suburbs), ensuring final metrics represent a true city-wide average.

---

## Outputs

- **The 43% Premium** — The optimised XGBoost model achieved a Cross-Validated R² of 0.431, demonstrating that localised infrastructure and safety independently explain over 40% of the variance in Sydney's rental market.
- **Tight Financial Tolerance** — An Average RMSE of ~$77 demonstrates the ability to estimate median weekly rent with high precision using solely external neighbourhood features.
- **Dominant Features** — Feature importance extraction revealed that **Crime Rate (Safety)** and **Business Density (Convenience)** are the strongest infrastructure signals, heavily outweighing public transit access and localised points of interest in driving rental premiums.

---

## References & Data Sources

- [Australian Bureau of Statistics (ABS) — 2021 Census DataPacks](https://www.abs.gov.au/census/find-census-data/datapacks)
- [NSW Bureau of Crime Statistics and Research (BOCSAR)](https://www.bocsar.nsw.gov.au/)
- [PostGIS Spatial Database Documentation](https://postgis.net/documentation/)
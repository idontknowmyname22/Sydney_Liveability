# Quantifying the "Infrastructure Premium" in the Sydney Rental Market 

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


---

## Overview

This project investigates the associations between neighbourhood infrastructure and median weekly rent across Greater Sydney. To capture a suburb's amenity profile, a custom spatial **Liveability Index** was engineered using NSW Spatial Services data—specifically targeting **Community** hubs, **Education**, **Transport**, **Recreation** sites, and **Beaches**.

By combining these localised Points of Interest (POIs) with transit access, business density, and crime rates, tree-based regression models were used to test how much of Sydney's rent variation can be explained by liveability features alone.

The goal is to isolate a liveability-driven **"Infrastructure Premium"** while deliberately excluding broader market forces such as income, supply constraints, dwelling-specific characteristics, and unobserved prestige effects that go beyond the included beach-access amenity signal. This makes the ML component an intentionally constrained, explanatory experiment rather than a production-grade rent forecasting system.

*(Note: This analysis is observational and reports associations; it does not establish causal effects.)*

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
- Extracts feature importances and reports both shuffled and geography-aware cross-validation results.

---

## Prerequisites

To run this project locally, you will need the following installed:

- **Python** (3.11+ recommended)
- **PostgreSQL** with the PostGIS spatial extension enabled

---

## Setup

**1. Clone the repository**

```bash
git clone https://github.com/idontknowmyname22/Sydney_Liveability.git
cd Sydney_Liveability
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
| `SA2_2021_AUST_SHP_GDA2020/` | ABS SA2 boundary shapefile | [ABS Digital Boundary Files](https://www.abs.gov.au/statistics/standards/australian-statistical-geography-standard-asgs-edition-3/jul2021-jun2026/access-and-downloads/digital-boundary-files/SA2_2021_AUST_SHP_GDA2020.zip) |
| `SAL_2021_AUST_GDA2020_SHP/` | ABS Suburb/Locality boundary shapefile | [ABS Digital Boundary Files](https://www.abs.gov.au/statistics/standards/australian-statistical-geography-standard-asgs-edition-3/jul2021-jun2026/access-and-downloads/digital-boundary-files/SAL_2021_AUST_GDA2020_SHP.zip) |
| `catchments/` | NSW school catchment shapefiles | [NSW Education Data](https://data.nsw.gov.au/data/dataset/8b1e8161-7252-43d9-81ed-6311569cb1d7/resource/32d6f502-ddb1-45d9-b114-5e34ddfd33ac/download/catchments.zip) |
| `Businesses.csv` | Business counts by SA2 and industry | [ABS Business Register]*Provided in the repo* |
| `Population.csv` | Population by age group and SA2 | [ABS Census DataPacks]*Provided in the repo* |
| `Stops.txt` | GTFS-style public transport stop locations | [Transport for NSW Open Data](https://opendata.transport.nsw.gov.au/data/dataset/d1f68d4f-b778-44df-9823-cf2fa922e47f/resource/67974f14-01bf-47b7-bfa5-c7f2f8a950ca/download/full_greater_sydney_gtfs_static_0.zip) |
| `SuburbData25Q3.csv` | NSW Police crime data by suburb  2019–2020 incidents used | [BOCSAR Crime Statistics](https://bocsarblob.blob.core.windows.net/bocsar-open-data/SuburbData.zip) |
| `2021Census_G02_NSW_SA2.csv` | ABS Census G02 — median rent, income, household stats | [ABS Census DataPacks](https://www.abs.gov.au/census/find-census-data/datapacks/download/2021_GCP_all_for_NSW_short-header.zip) |

> POIs are fetched live from the [NSW Government Spatial Services API](https://maps.six.nsw.gov.au/arcgis/rest/services/public/NSW_POI/MapServer) at runtime — no local file required.

---

## Usage

1. Launch Jupyter Notebook or JupyterLab.
2. Open and run `Sydney_Liveability.ipynb` from top to bottom. This executes the spatial joins, calculates the Z-scores, saves them to the database, and renders the interactive Plotly map.
3. Open and run `Sydney_Liveability_ML.ipynb` from top to bottom. This merges the spatial data with the ABS Census CSV, trains the models, and outputs both shuffled and geography-aware validation metrics.

---

## Methodology

### 1. Area-Weighted Spatial Interpolation

Solving the **"Spatial Mismatch"** between NSW Police reporting boundaries (Suburbs) and ABS Census boundaries (SA2s) required sub-dividing geometries. To prevent map projection distortions, geometries were cast to PostGIS's native `geography` type, ensuring overlap ratios were calculated over the true spherical shape of the Earth in exact square meters rather than distorted square degrees.

### 2. Z-Score Standardisation & The MAUP

Raw counts of infrastructure inherently bias physically larger suburbs. To counteract the **Modifiable Areal Unit Problem (MAUP)**, all counts were converted to densities and normalised using Z-score standardisation. This allows the models to fairly compare the infrastructure profile of high-density inner-city SA2s against sprawling western SA2s.

---

## Machine Learning Validation

- **Baseline Model (Random Forest)** — Established an initial ensemble baseline and highlighted Crime Rate and Business Density as the strongest liveability-linked rent signals.
- **Model Optimisation (XGBoost)** — Tuned an XGBoost Regressor with constrained depth (`max_depth=2`, `learning_rate=0.05`) to reduce overfitting on a small 360-row SA2 dataset.
- **Two Validation Lenses** — The notebook reports both Shuffled 5-Fold Cross-Validation and a stricter GroupKFold split by SA4. The shuffled metric measures within-city interpolation across the existing Sydney mix; the grouped metric asks the harder question of whether the learned relationships transfer to entirely unseen subregions.

---

## Outputs

- **Moderate Within-City Signal** — Under shuffled 5-fold cross-validation, the tuned XGBoost model achieved **R² = 0.471** with **RMSE = $75.09**, while Random Forest achieved **R² = 0.447** with **RMSE = $76.81**. This suggests the liveability features carry useful explanatory signal when interpolating across the current Sydney SA2 mix.
- **Weak Transfer To Unseen Regions** — Under geography-aware GroupKFold by SA4, performance dropped sharply to **R² = 0.078** for XGBoost and **R² = 0.021** for Random Forest. The index still contains signal, but it does not on its own support strong out-of-region rent prediction claims.
- **Dominant Features** — Feature importance extraction still identified **Crime Rate (Safety)** and **Business Density (Convenience)** as the strongest liveability-linked drivers in the Random Forest model.

---

# ♻️ DSNY Recycling Performance Forecasting

---

<p align="center">
  <img         src="https://www.wastetodaymagazine.com/remote/aHR0cHM6Ly9naWVjZG4uYmxvYi5jb3JlLndpbmRvd3MubmV0L2ZpbGV1cGxvYWRzL2ltYWdlLzIwMjQvMTIvMTAvYWRvYmVzdG9ja181Nzc2Mjg2MTJfZWRpdG9yaWFsX3VzZV9vbmx5X2RzbnlfbG9nby5naWY.Lapt4I6wKNE.gif?format=webp"
    width="400"
    alt="DSNY Logo"
  />
</p>

## Project Purpose

This project forecasts monthly recycling performance for NYC community districts to help DSNY's operations department allocate resources and budget for recycling activity. DSNY collects 24 million pounds of recyclables, trash, and compost daily across 59 district garages. In 2024, Manhattan District 8 collected **1,576 more tons** of recyclables than Brooklyn District 16, demonstrating significant district-level variation.

**Stakeholder**: NYC Department of Sanitation - Operations Department

**Key Decision**: Where to allocate educational resources, outreach programs, and collection resources in the next 8 months based on predicted recycling performance at the district level.

<img width="730" height="687" alt="Screenshot 2025-12-20 at 3 33 05 PM" src="https://github.com/user-attachments/assets/524ec1e3-d19f-4607-8004-2412a2022745" />

---

## Dataset Information

**Source**: [NYC Open Data - DSNY Monthly Tonnage](https://data.cityofnewyork.us/City-Government/DSNY-Monthly-Tonnage-Data/ebb7-mvp5/about_data)

* **Coverage**: 1,416 monthly observations (Jan 2023 - Jan 2025)
* **Grain**: Monthly by borough and community district (59 districts total)

### Data Dictionary

| Column | Description | Type | Notes |
| --- | --- | --- | --- |
| MONTH | Reporting month | DateTime | Parsed to datetime |
| BOROUGH | NYC borough | Categorical | 5 boroughs |
| COMMUNITYDISTRICT | District number | Integer | 3-18 per borough |
| REFUSETONSCOLLECTED | Non-recyclable tons | Float | Target component |
| PAPERTONSCOLLECTED | Paper recyclables | Float | Target component |
| MGPTONSCOLLECTED | Metal/Glass/Plastic | Float | Target component |
| proportionrefuse* | (Paper + MGP) / Total | Float [0-1] | **Engineered target** |
| id* | Borough + District | String | **Engineered** (e.g., "bronx1") |

*Engineered features

### Data Quality

* **Temporal Filter**: January 2023 to January 2025 (2 years)
* **Missing Values**: Organic waste columns excluded (~70-90% missing)
* **Grain**: Monthly district level

---

## How to Reproduce
```bash
# Setup
git clone https://github.com/Commit-Thomas/DSNY_Analysis.git
cd DSNY_Analysis
python -m venv venv
source venv/bin/activate  # Windows: venv\Scripts\activate
pip install -r requirements.txt

# Launch app
cd app
streamlit run streamlit_app.py
```

---

## Exploratory Data Analysis

### Stakeholder Context

**Stakeholder**: DSNY Operations Department  
**Timeline**: 8 months from last date 
**Decision**: Resource allocation for campaigns, infrastructure, scheduling

### Key Patterns

1. **Temporal Seasonality**: Yearly patterns with summer peaks
2. **District Heterogeneity**: Manhattan 8 (1,884 tons) vs Brooklyn 16 (308 tons) = 6x variation
3. **Stability Differences**: Districts are volatile month-to-month
4. **Moderate Correlation**: Paper and MGP moderately correlated but provide distinct signals

### Pre-Model Assumptions

* Temporal independence with recent past influence
* Stationarity achievable via first-order differencing
* 12-month seasonality exists for districts

---

## Target Variable

### Definition
```python
proportionrefuse = (PAPERTONSCOLLECTED + MGPTONSCOLLECTED) / 
                   (PAPERTONSCOLLECTED + MGPTONSCOLLECTED + REFUSETONSCOLLECTED)
```

### Justification

* **Stakeholder Alignment**: Directly measures DSNY's key performance indicator
* **Normalized**: Accounts for district size; enables fair comparison
* **Actionable**: Thresholds trigger resource allocation decisions
* **Stable**: More forecastable than raw tonnage

---

## Modeling Approach

### Train/Test Split

* **Type**: Chronological (time-based)
* **Train**: First 70%
* **Test**: Last 30%
* **Rationale**: Preserves temporal order, prevents leakage

### Three Models

We fit all three models per district and select the lowest RMSE. District patterns vary significantly, so adaptive selection outperforms a single global approach.

#### Model 1: Baseline (Naive)

* **Method**: Last observed value carried forward
* **Fair Yardstick**: Minimum acceptable performance; difficult to beat for stable series
* **Implementation**: `predict = s.shift(1)[test.index]`

#### Model 2: ARIMA(1,1,1)

* **Method**: AR(1) + I(1) differencing + MA(1)
* **Assumptions**: Stationarity after differencing, short-term autocorrelation, no seasonality
* **Library**: `statsmodels.tsa.arima.model.ARIMA`

#### Model 3: SARIMA(1,1,1)(1,1,1,12)

* **Method**: ARIMA + 12-month seasonal component
* **Assumptions**: Yearly periodicity, sufficient data for estimation, additive seasonality
* **Library**: `statsmodels.tsa.statespace.sarimax.SARIMAX`

---

## Evaluation Metrics

### Example: Bronx District 5

| Model | RMSE | Selected |
| --- | --- | --- |
| Baseline | 0.007 | |
| ARIMA(1,1,1) | 0.006 | |
| SARIMA(1,1,1)(1,1,1,12) | 0.003 | ✓ |

### Metrics

**RMSE**: Penalizes large errors; same units as target  

---

## Deployment

### Serialized Models

* `models/baseline.pkl`: Baseline predictions by district
* `models/modeling_simple.pkl`: ARIMA models by district
* `models/modeling_tuned.pkl`: SARIMA models by district

### Streamlit App

**Features**:
1. Input district ID (e.g., `bronx1`)
2. Auto-calculates RMSE for all models
3. Highlights best model
4. Visualizes forecast with time series plot

**Stakeholder Output**:
> "For [District], the [Model] predicts recycling percentage of [X.XX] next month, representing [change]. [Action recommendation]."

**Run**: `streamlit run app/streamlit_app.py`  
**Demo**: See `projectdemo.mp4`

---

## Ethics & Limitations

### Ethics

* **Privacy**: District aggregation protects individuals
* **Advisory**: Predictions inform, not mandate decisions
* **Equity**: Identifies districts needing support, not punishment
* **Transparency**: Stakeholders see model selection rationale

### Limitations

**Data**:
* No per capita normalization (district size comparison limited)
* Limited features (only tonnage; no demographics/infrastructure)
* Monthly granularity only (no sub-monthly events)

**Model**:
* Requires domain knowledge (district codes)
* Static models (no auto-retraining)
* External factors (weather, policy) not modeled

**Practical**:
* Performance gaps may need infrastructure/policy changes beyond model scope
* K-means revealed no distinct behavioral groups for intervention targeting

### Future Work

**Short-Term**:
* Per capita data integration
* Raw tonnage output option
* Confidence intervals
* Multi-step forecasting (3-6 months)

**Long-Term**:
* Auto-ARIMA for order selection
* Exogenous variables (weather, holidays, policy dates)
* User-friendly interface for non-technical staff
* Advanced ML (LSTM, Prophet)
* Spatial models with district adjacency
* Automated retraining pipeline

---

## Project Structure
```
DSNY_Analysis/
├── data/
│   └── DSNY_Monthly_Tonnage_Data.csv
├── notebooks/
│   ├── 01_eda.ipynb
│   ├── 02_modeling_baseline.ipynb
│   ├── 03_modeling_simple.ipynb
│   └── 04_modeling_tuned.ipynb
├── app/
│   └── streamlit_app.py
├── models/
│   ├── baseline.pkl
│   ├── modeling_simple.pkl
│   └── modeling_tuned.pkl
├── requirements.txt
├── projectdemo.mp4
└── README.md
```

# Hospital Bed Capacity Reallocation & Forecasting

### Decision Support Case Study 

This project evaluates whether budgeted hospital-bed capacity should be reallocated between two inpatient units with very different demand patterns.

The central decision is:

> **Should 10 budgeted beds be transferred from North 2D to South 3D? If not, what smaller reallocation best balances hospital-wide capacity, funding, and operational risk?**

The analysis combines historical utilization, operational and financial KPIs, demand-driver exploration, six-month time-series forecasting, scenario analysis, and sensitivity testing to produce a practical recommendation.

> **Data note:** The source data used for this case study is simulated and represents a fictitious rehabilitation centre/hospital environment. No real patient data is used in this analysis.


---

## Executive Summary

Historical analysis showed a persistent capacity mismatch:

- **South 3D** had substantially more demand than its **10 budgeted beds**.
- **North 2D** had substantially less demand than its **17 budgeted beds**.
- South generated unfunded demand and missed funding opportunity, while North carried unused budgeted capacity and empty-bed loss.

A six-month forecast suggested that this imbalance would continue.

The requested **10-bed transfer was not recommended** because it would largely move the capacity problem from South to North. A **4-bed transfer** produced the strongest base-case balance, but sensitivity analysis showed that North would have limited operating buffer if demand were slightly higher than forecast.

### Final recommendation

**Use a staged 3 → 4 bed reallocation:**

1. Transfer **3 budgeted beds** from North 2D to South 3D.
2. Monitor both units using shared capacity, financial, and demand indicators.
3. Transfer the **4th bed only if observed data confirms that North retains sufficient operating buffer**.
4. If South demand continues to rise, broaden the review to other underused units instead of continuing to remove capacity from North.

---

## Business Context

South 3D management reported insufficient budgeted capacity and estimated that more than **$6 million in funding opportunity could be at risk over the next six months** because patients were being served above budgeted capacity.

South 3D requested **10 additional budgeted beds**.

The proposed source of those beds was North 2D:

| Unit | Current Budgeted Beds | If Full 10-Bed Transfer Occurs |
|---|---:|---:|
| South 3D | 10 | 20 |
| North 2D | 17 | 7 |

The decision therefore had to be evaluated at the **hospital-system level**, not only from South 3D's perspective.

---

## Datasets

### 1. Patient Flow Census Data

The primary dataset contained approximately:

- **44,947 nightly census records**
- **913 consecutive days**
- **6 hospital units**
- Date range: **December 1, 2023 to May 31, 2026**

Each row represents one patient recorded in the nightly census snapshot and includes fields such as:

- Unique patient identifier
- Effective date
- Patient type
- Bed asset tag
- Room
- Department / unit
- Site

The nightly census represents patients occupying beds at approximately the same point in time each day.

### 2. Unit Bed Pricing & Allotment

This table provides each unit's:

- Number of budgeted beds
- Daily funding rate per occupied budgeted bed

Key values for the two decision units:

| Unit | Budgeted Beds | Daily Rate |
|---|---:|---:|
| South 3D | 10 | $5,125 |
| North 2D | 17 | $7,850 |

---

## Funding Rules Used in the Case

The analysis follows the case funding rules:

- An **occupied budgeted bed** earns the full daily funding rate.
- An **unfilled budgeted bed** loses **50% of its daily funding rate**.
- A patient above budgeted capacity is treated as occupying an **unfunded bed**, which earns **$0** under the case rules.
- Preferred budgeted-bed occupancy is **90–95%** to preserve emergency and bed-management flexibility.

For example:

- One unfunded South 3D patient-day represents a **$5,125 missed funding opportunity**.
- One unfilled North 2D budgeted bed-day represents an empty-bed loss of **$7,850 × 50% = $3,925**.

---

## Analytical Workflow

```text
Raw patient-level census data
        ↓
Data validation and cleaning
        ↓
Daily unit-level census table
        ↓
Capacity + financial KPI calculation
        ↓
Historical trend analysis
        ↓
Demand-driver exploration
        ↓
Time-series forecasting
        ↓
No-change baseline
        ↓
Bed-reallocation scenarios
        ↓
Sensitivity analysis
        ↓
Staged recommendation
```

---

## 1. Data Validation & Preparation

The source data was checked for:

- Missing values
- Exact duplicate rows
- Duplicate patient-date records
- Duplicate bed-date records
- Complete daily date sequence
- Consistent unit mapping between census and pricing tables

The data passed these checks:

- **0 important missing values**
- **0 exact duplicates**
- **0 duplicate patient-date records**
- **0 duplicate bed-date records**
- **All 913 dates present**
- **All six units mapped successfully**

### Why check patient-date duplicates?

Each patient should appear only once in a nightly census snapshot. A duplicate could overstate daily census.

### Why check bed-date duplicates?

A bed should only be assigned to one patient in the same nightly snapshot. A duplicate could indicate an inconsistent census record.

---

## 2. Building the Daily Census Table

The raw patient-level data was aggregated by **date + unit**.

```python
 daily_census = (
     patient_data
     .groupby(["Date", "Department/Unit"])
     .size()
     .reset_index(name="Census")
 )
```

This transformed approximately **44,947 patient-night records** into approximately:

> **913 days × 6 units = 5,478 daily unit-level records**

`Census` represents the number of patients present in a unit at the nightly snapshot.

For example:

```text
Date        Unit        Census
2026-01-10  South 3D    18
2026-01-10  North 2D    13
```

The daily census table was then joined to the bed-pricing/allotment table.

---

## 3. Core Capacity & Financial KPIs

### Occupied Budgeted Beds

```text
Occupied Budgeted Beds = min(Census, Budgeted Beds)
```

### Unfunded Patients

```text
Unfunded Patients = max(Census - Budgeted Beds, 0)
```

### Unfilled Budgeted Beds

```text
Unfilled Budgeted Beds = max(Budgeted Beds - Census, 0)
```

### Occupancy

```text
Occupancy % = Occupied Budgeted Beds / Budgeted Beds × 100
```

Occupancy is capped at 100% under this definition; excess demand is captured separately as unfunded patients.

### Unfunded Bed-Days

```text
Unfunded Bed-Days = sum of daily unfunded patients
```

One patient above budgeted capacity for one day = one unfunded bed-day.

### Unfilled Bed-Days

```text
Unfilled Bed-Days = sum of daily unfilled budgeted beds
```

### Missed Funding Opportunity

```text
Missed Funding Opportunity = Unfunded Patients × Daily Rate
```

### Empty-Bed Loss

```text
Empty-Bed Loss = Unfilled Budgeted Beds × Daily Rate × 0.5
```

These KPIs allow the same demand/capacity mismatch to be viewed operationally and financially.

---

## 4. Historical Performance

The full history was used to understand long-term patterns, while the **most recent six months (Dec 2025–May 2026)** were used to describe the current operating condition because South 3D experienced a major recent demand increase.

### South 3D — Recent Six Months

| Metric | Result |
|---|---:|
| Budgeted beds | 10 |
| Average nightly census | **17.2** |
| Minimum census | 11 |
| Maximum census | 22 |
| Occupancy | **100%** |
| Days above budgeted capacity | **182 / 182** |
| Unfunded bed-days | **1,309** |
| Missed funding opportunity | **~$6.71M** |

### North 2D — Recent Six Months

| Metric | Result |
|---|---:|
| Budgeted beds | 17 |
| Average nightly census | **12.9** |
| Minimum census | 10 |
| Maximum census | 15 |
| Occupancy | **75.9%** |
| Days below budgeted capacity | **182 / 182** |
| Unfilled bed-days | **746** |
| Empty-bed loss | **~$2.93M** |

### Historical conclusion

The mismatch was persistent rather than occasional:

- South 3D was above budgeted capacity every day in the recent six-month period.
- North 2D was below budgeted capacity every day in the same period.

South therefore had **under-budgeted demand**, while North had **unused budgeted capacity**.

### Suggested visual

```markdown
![Historical monthly census](images/monthly_census_trend.png)
```

The monthly trend shows North remaining relatively stable while South rises sharply beginning in late 2025.

---

## 5. Demand-Driver Exploration

To understand why South 3D's census increased, two supporting measures were explored.

### Monthly Unique Patients

Distinct patients were counted by month:

```python
monthly_unique = (
    patient_data
    .groupby(["Month", "Department/Unit"])["Unique Patient Identifier"]
    .nunique()
)
```

South 3D experienced a clear increase in the number of unique patients during the higher-demand period.

### Observed Bed-Days per Unique Patient

```text
Observed Bed-Days per Unique Patient
= Total Patient-Night Records / Unique Patients
```

This measure also increased in South 3D.

### Interpretation

South 3D's recent capacity pressure appears to be associated with both:

1. **Higher patient volume** — more distinct patients using the unit.
2. **Greater observed bed use per patient** — more patient-night observations per unique patient.

This measure should **not** be interpreted as formal length of stay because exact admission and discharge timestamps were not available.

### Suggested visuals

```markdown
![Monthly unique patients](images/monthly_unique_patients.png)

![Observed bed-days per unique patient](images/bed_days_per_patient.png)
```

---

## 6. Six-Month Forecasting

Separate daily time series were created for South 3D and North 2D because their demand patterns were materially different:

- **South 3D:** recent sharp upward shift
- **North 2D:** relatively stable demand

### Forecast Horizon

**June 1, 2026 through November 30, 2026 — 183 days**

### Candidate Forecasting Methods

Three interpretable approaches were compared:

1. **Recent Mean** — flat baseline based on recent average demand
2. **Holt Linear Trend** — estimates level and trend and projects the trend forward
3. **Holt Damped Trend** — estimates level and trend but gradually reduces the trend over the forecast horizon

An exploratory recent-regime linear regression was also reviewed as a higher-growth comparison for South 3D.

---

## 7. Forecast Validation

The most recent **60 days** were held out for out-of-sample validation.

This provided enough unseen data to compare candidate models while preserving sufficient recent South 3D observations in the training set.

### Metrics

#### Mean Absolute Error (MAE)

```text
MAE = mean(|Actual - Forecast|)
```

Lower is better.

An MAE of 2 means that the forecast was approximately two patients away from the actual census on an average day.

#### Bias

```text
Bias = mean(Actual - Forecast)
```

- Positive bias → model tends to **underforecast**
- Negative bias → model tends to **overforecast**
- Near zero → limited systematic directional error

### South 3D Validation

| Model | MAE | Bias |
|---|---:|---:|
| Recent Mean | 2.80 | +2.77 |
| Holt Linear | 2.68 | +2.65 |
| Holt Damped | 2.68 | +2.65 |
| Full-history Holt Linear | **2.00** | +1.90 |
| Full-history Holt Damped | 2.68 | +2.65 |

The linear trend performed well on the short holdout period, but its six-month forecast continued rising strongly.

Approximate Holt Linear monthly forecast:

| Month | Forecast Census |
|---|---:|
| June | 21.3 |
| July | 22.1 |
| August | 23.0 |
| September | 23.9 |
| October | 24.7 |
| November | 25.6 |

Because South's higher-demand pattern was relatively recent, uninterrupted six-month linear growth was considered an aggressive primary planning assumption.

### North 2D Validation

| Model | MAE | Bias |
|---|---:|---:|
| Recent Mean | 1.00 | -0.31 |
| Holt Linear | 1.09 | +0.80 |
| Holt Damped | **0.94** | **-0.12** |

Holt Damped provided the lowest MAE and near-zero bias for North.

### Final Forecasting Choice

**Holt Damped** was selected as the primary planning model for both units.

For South, the goal was not simply to choose the lowest short-term MAE. Model selection also considered the behavior and plausibility of the full **183-day planning forecast**.

Holt Linear was retained conceptually as a higher-growth sensitivity case rather than the primary forecast.

---

## 8. Six-Month Planning Forecast

| Unit | Current Budgeted Beds | Forecast Avg. Census |
|---|---:|---:|
| South 3D | 10 | **20.8** |
| North 2D | 17 | **11.8** |

The planning forecast suggests that the existing allocation mismatch is likely to persist.

### Suggested visual

```markdown
![Six-month forecast](images/six_month_forecast.png)
```

---

## 9. No-Change Scenario

The no-change scenario keeps the current allocations:

- South 3D = 10 budgeted beds
- North 2D = 17 budgeted beds

### South 3D

| Metric | Six-Month Forecast |
|---|---:|
| Average census | **20.8** |
| Budgeted beds | 10 |
| Occupancy | **100%** |
| Unfunded bed-days | **~1,973** |
| Missed funding opportunity | **~$10.1M** |

### North 2D

| Metric | Six-Month Forecast |
|---|---:|
| Average census | **11.8** |
| Budgeted beds | 17 |
| Occupancy | **69.3%** |
| Unfilled bed-days | **~954** |
| Empty-bed loss | **~$3.75M** |

### Interpretation

Doing nothing preserves the same imbalance observed historically:

- South remains heavily constrained.
- North remains materially underutilized.

Financial values should be interpreted as **planning estimates under the case funding rules**, not guaranteed realized revenue or audited accounting losses.

---

## 10. Bed-Reallocation Scenario Analysis

For each scenario, demand forecasts were held constant while budgeted capacity was reallocated.

If `t` beds are transferred:

```text
South Beds = 10 + t
North Beds = 17 - t
```

The capacity and financial KPIs were then recalculated for both units.

### Key Scenarios

| Beds Moved | South / North Beds | North Occupancy | South Unfunded Bed-Days | South Missed Funding | North Empty-Bed Loss |
|---:|---:|---:|---:|---:|---:|
| 3 | 13 / 14 | **84.2%** | ~1,424 | ~$7.30M | ~$1.59M |
| 4 | 14 / 13 | **90.7%** | ~1,241 | ~$6.36M | ~$0.87M |
| 5 | 15 / 12 | **98.2%** | ~1,058 | ~$5.42M | ~$0.15M |

### Interpretation

#### 3 Beds

- Safer for North
- North remains below the preferred 90–95% utilization range
- South receives some relief

#### 4 Beds

- Strongest base-case balance
- North reaches **90.7% occupancy**, inside the preferred range
- No projected North unfunded demand in the base forecast
- South receives meaningful capacity and financial relief

#### 5 Beds

- More relief for South
- North reaches **98.2% occupancy**
- Very limited operating flexibility remains

Therefore, **4 beds is the strongest quantitative base-case allocation**.

### Suggested visual

```markdown
![Transfer scenario comparison](images/transfer_scenarios.png)
```

---

## 11. Testing the Requested 10-Bed Transfer

The exact management request was also evaluated.

### Allocation After 10-Bed Transfer

| Unit | Budgeted Beds |
|---|---:|
| South 3D | 20 |
| North 2D | 7 |

### Impact

**South 3D**

- Missed funding opportunity falls to approximately **$0.73M**
- Unfunded bed-days fall dramatically

**North 2D**

- Only **7 budgeted beds** remain
- Forecast demand remains approximately **11.8 patients/night**
- Occupancy reaches **100%**
- Approximately **876 unfunded bed-days** are created
- Missed funding opportunity rises to approximately **$6.87M**

### Conclusion

> **The 10-bed transfer does not solve the hospital-wide problem; it largely relocates the bottleneck and financial exposure from South 3D to North 2D.**

For this reason, the full 10-bed request is **not recommended**.

---

## 12. Sensitivity Analysis

Although four beds produced the best base-case balance, forecast uncertainty needed to be tested.

The 4-bed transfer was held fixed, leaving North with **13 budgeted beds**, and North demand was increased by one and two patients per day.

| North Demand Scenario | Avg. Census | Occupancy | Unfunded Bed-Days | Interpretation |
|---|---:|---:|---:|---|
| Base forecast | 11.8 | **90.7%** | 0 | Within target range |
| +1 patient/day | 12.8 | **98.4%** | 0 | Very limited flexibility |
| +2 patients/day | 13.8 | **100%** | ~144 | North begins generating unfunded demand |

### Key Insight

The 4-bed transfer is efficient under the expected forecast, but North's operating buffer disappears quickly if actual demand is modestly higher than forecast.

This finding supports **staged implementation** rather than immediately and permanently moving four beds.

### Suggested visual

```markdown
![North sensitivity analysis](images/north_sensitivity.png)
```

---

## 13. Recommendation

### 1. Reject the Full 10-Bed Transfer

The full transfer would substantially improve South but create a major shortage in North.

The capacity problem would be **shifted rather than solved**.

### 2. Start with a 3-Bed Transfer

Initial allocation:

- **South 3D: 10 → 13 beds**
- **North 2D: 17 → 14 beds**

This provides immediate relief to South while retaining more operational buffer in North.

### 3. Monitor Both Units

Use a shared monitoring approach to track:

- Capacity pressure
- Financial impact
- Demand trends
- Emergence of unfunded demand

### 4. Use a Decision Gate for the Fourth Bed

Move to:

- **South 3D: 14 beds**
- **North 2D: 13 beds**

only if:

- South remains persistently constrained
- North retains sufficient operating buffer
- North does not generate meaningful unfunded demand
- Clinical and staffing feasibility is confirmed

### 5. Broaden the Capacity Review if South Continues Growing

If South continues to experience strong demand after the staged transfer, do not continue removing beds from North once its buffer is limited.

Instead, evaluate other persistently underused units using:

- Updated forecasts
- Capacity KPIs
- Clinical review
- Staffing feasibility
- Operational constraints

Historical utilization suggests **North 1D and West 4A** may warrant further assessment, but they should not be treated as confirmed donor units without additional forecasting and operational validation.

---

## 14. Key Limitations

### Recent South 3D Demand Shift

South's demand increase occurred relatively recently, leaving limited history under the new higher-demand pattern.

The six-month South forecast is therefore more uncertain than North's.

### Nightly Snapshot Data

The census is a point-in-time nightly snapshot and may not capture:

- Within-day admissions
- Within-day discharges
- Transfers
- Peak daytime demand

### No Formal Length-of-Stay Data

Exact admission and discharge timestamps were not available.

Observed bed-days per unique patient should therefore be interpreted as a utilization proxy rather than true length of stay.

### Missing Operational Variables

The dataset did not include several variables that could materially improve capacity planning, including:

- Staffing levels
- Patient acuity
- Admission/discharge timing
- Discharge delays
- Referral or waitlist demand
- Physical-space constraints
- Program or operational changes

### Forecast Uncertainty

Point forecasts smooth daily variation and may understate peak-demand risk, particularly for North after a transfer.

This is why sensitivity analysis and staged implementation were included.

### Financial Estimates

Financial results are planning estimates based on the simplified funding rules supplied in the case and should not be interpreted as guaranteed realized revenue or audited accounting losses.

---

## 15. Governance Considerations

A bed-allocation decision should not be implemented using analytics alone.

Recommended stakeholders include:

- Unit Leadership
- Patient Flow / Bed Management
- Clinical / Nursing Leadership
- Finance
- Decision Support / Analytics

The analytical recommendation should be combined with clinical, staffing, operational, and physical-capacity review before implementation.

---

## Technology Stack

- **Python**
- **Pandas** — data manipulation and aggregation
- **NumPy** — numerical calculations
- **Matplotlib** — visualization
- **statsmodels** — Holt / exponential-smoothing forecasting
- **Jupyter Notebook** — analysis workflow and reproducibility

---

## Repository Structure

A clean public repository could use the following structure:

```text
hospital-bed-capacity-analysis/
│
├── README.md
├── hospital_bed_capacity_reallocation_analysis.ipynb
│
├── images/
│   ├── monthly_census_trend.png
│   ├── monthly_unique_patients.png
│   ├── bed_days_per_patient.png
│   ├── six_month_forecast.png
│   ├── transfer_scenarios.png
│   └── north_sensitivity.png
│
├── data/
│   └── README.md             
│
└── requirements.txt
```



---

## How to Run

1. Clone the repository.
2. Create a Python environment.
3. Install required packages.
4. Place the permitted input data files in the expected `data/` directory.
5. Open and run the Jupyter notebook from top to bottom.

Example:

```bash
pip install pandas numpy matplotlib statsmodels openpyxl jupyter
jupyter notebook
```

---

## Key Takeaways

This project demonstrates how operational data can be translated into a capacity-planning decision by combining:

- Data-quality validation
- Patient-level to unit-level aggregation
- Operational KPI design
- Financial impact modeling
- Time-series forecasting
- Out-of-sample model validation
- Scenario analysis
- Sensitivity testing
- System-level decision support

The central lesson is that the solution is **not to maximize one unit's capacity in isolation**.

A stronger decision considers the entire system:

> **Relieve the constrained unit without creating a new bottleneck elsewhere.**

---

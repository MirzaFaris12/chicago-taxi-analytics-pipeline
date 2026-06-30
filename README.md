# Chicago Taxi Analytics Pipeline

A production-ready data pipeline on GCP transforming the Chicago Taxi Trips public dataset into analytical models answering key business questions.

**Tech Stack:** BigQuery · Dataform · Looker Studio · GitHub  
**Dataset:** `bigquery-public-data.chicago_taxi_trips.taxi_trips`  
**Coverage:** 2013-01-01 to 2023-12-31 · 213,111,447 raw trips

---

## Project Structure

```
chicago-taxi-analytics-pipeline/
├── definitions/
│   ├── staging/
│   │   ├── stg_taxi_trips.sqlx          # Base cleaning layer
│   │   └── stg_driver_shifts.sqlx       # Reconstructed driver shifts (Q2)
│   └── marts/
│       ├── q1_top_tip_earners.sqlx      # Top 100 tip earners
│       ├── q2_top_overworkers.sqlx      # Top 100 overworking drivers
│       ├── q3_holiday_impact.sqlx       # Holiday impact on trip volumes
│       ├── bonus1_payment_type_trends.sqlx
│       └── bonus2_demand_heatmap.sqlx
├── workflow_settings.yaml
└── README.md
```

### Architecture: Raw → Staging → Mart

- **Raw** — unmodified public source. Never written to.
- **Staging** — cleaned, deduplicated, type-cast. Generic and reusable.
- **Marts** — aggregated, purpose-built tables. What Looker Studio connects to.

---

## BigQuery Dataset

All tables land in: `project-55ddb663-3b7b-4837-a3b.chicago_taxi_analytics`

| Table | Type | Rows | Description |
|---|---|---|---|
| `stg_taxi_trips` | Staging | ~210.9M | Cleaned base layer |
| `stg_driver_shifts` | Staging | ~12.8M | Reconstructed shifts |
| `us_federal_holidays` | Reference | 113 | US holidays 2013–2023 (CSV upload) |
| `q1_top_tip_earners` | Mart | 100 | Top 100 tip earners Q4 2023 |
| `q2_top_overworkers` | Mart | 100 | Top 100 overworking drivers Q4 2023 |
| `q3_holiday_impact` | Mart | ~113 | Holiday vs baseline comparison |
| `bonus1_payment_type_trends` | Mart | ~77 | Annual payment type breakdown |
| `bonus2_demand_heatmap` | Mart | ~3,696 | Hourly demand by community area |

---

## Staging: Data Cleaning Rules

**Result:** 213,111,447 raw rows → **210,899,884 retained (~99%)**, ~2.2M dropped.

| Rule | Reasoning |
|---|---|
| Deduplicate on `unique_key` | Source occasionally re-reports trips |
| Drop null `trip_start_timestamp` or `taxi_id` | Cannot place trip in time or attribute to a driver |
| Drop `fare <= 0` or null | Data errors or voided/test records |
| Drop `trip_miles < 0` | Negative distance is impossible |
| Drop `trip_end < trip_start` | Impossible chronology |
| Cap `trip_seconds` at 86,400s (24h) | Guards against meter errors distorting Q2 shift logic |
| Coalesce `tips/tolls/extras` to 0 when null | Null = not charged, not missing |

**Cash tips:** Almost never captured in this dataset (bypass the meter). The `tips` field for cash trips reads near-zero and represents *reported* tips only.  
**Timestamps:** Source stores UTC. `trip_start_date` and `trip_start_hour` use `America/Chicago` timezone including DST.

---

## Question 1: Top 100 Tip Earners

**Model:** `q1_top_tip_earners`

### Assumptions & Methodology
- **"Last 3 months"** → interpreted as **Oct 1 – Dec 31, 2023** (Q4 2023). The dataset ends 2023-12-31 and hasn't been updated, so a real-world relative window returns zero rows.
- **Ranking metric** → total tip dollars earned, not tip rate. A driver doing 1,000 trips at 10% earns more absolute income than one doing 5 trips at 50%. Tip rate is included as a context column only.
- **Ranking unit** → per `taxi_id` (anonymized driver identifier from the City of Chicago).

### Key Findings (Q4 2023)
- **#1 tip earner:** 946 trips · $6,237.74 total tips · 13.21% tip rate · $6.59 avg tip/trip
- Top earners show a mix of high-volume/moderate-tip and lower-volume/high-tip profiles
- All top 100 earned more than $3,000 in tips over the quarter

---

## Question 2: Top 100 Overworkers

**Models:** `stg_driver_shifts` → `q2_top_overworkers`

### Assumptions & Methodology

The dataset has no clock-in/clock-out data — shifts are reconstructed from trip timestamps.

**Shift reconstruction (8-hour gap rule):** Trips per taxi are ordered chronologically. A gap of **< 8 hours** between one trip ending and the next starting = same shift. A gap of **≥ 8 hours** = break; next trip starts a new shift. Threshold directly matches the question's wording ("without taking at least 8 hours break").

**Conservative bias:** Breaks under 8 hours are not detected, so shift lengths are an upper-bound estimate of overworking.

**"Long shift"** = 12+ hours. Typical taxi shifts run 8–12 hours; 12+ exceeds the normal range.

**"Regularly"** = captured via `overwork_score = long_shifts_count × (long_shift_rate_pct / 100)`. This rewards drivers who are both frequent and consistent overworkers, not just high-volume drivers. Minimum 5 shifts required to qualify.

**Analysis window:** Q4 2023, matching Q1.

### Data Quality Note
Some shifts in earlier years (2013–2019) show extreme durations (100–311 hours), likely from meter/reporting errors. The 24-hour cap on `trip_seconds` partially mitigates this but cannot bound shift duration itself since it spans multiple trips. These extremes are less prevalent in later years as reporting improved.

---

## Question 3: Holiday Impact on Trip Volumes

**Models:** `us_federal_holidays` → `q3_holiday_impact`

### Assumptions & Methodology
- **Holiday calendar:** All 11 US federal holidays, 2013–2023. Source: OPM observed schedule. Stored as a static CSV table (maintainable without touching SQL). Juneteenth included from 2021.
- **Weekend adjustment:** Saturday holiday → observed Friday; Sunday holiday → observed Monday (OPM standard).
- **Baseline:** Average trips on non-holiday days sharing the same **day-of-week + month + year**. Controls for weekday effects, seasonal variation, and year-over-year trends simultaneously.
- **Scope:** Full 2013–2023 dataset for stable multi-year baselines.

### Key Findings
- **Christmas:** ~75–80% fewer trips vs a typical same-weekday in December — the largest drop consistently across all years
- **Thanksgiving:** ~55–65% drop vs a typical same-weekday in November
- **Labor Day / Memorial Day:** ~40–45% drop — consistent with people leaving the city
- **Columbus Day / Veterans Day:** minimal impact (-2% to -7%) — most businesses remain open
- **MLK Day:** near-zero or slight positive effect
- **Conclusion:** Yes, holidays significantly impact trip volumes, but only for holidays that change daily routines. Family/travel holidays (Christmas, Thanksgiving) cause large drops; minor federal holidays cause little to no change.

---

## Bonus Insight 1: Payment Type Trends Over Time

**Model:** `bonus1_payment_type_trends`

### Business Value
Tracks the annual shift from cash to card/digital payments across 2013–2023, informing payment terminal investment decisions and identifying which payment types drive the highest-value trips.

### Key Findings
- **2013:** Cash = 69.84% of trips · Credit Card = 28.92%
- **2023:** Cash = 29.81% · Credit Card = 40.86% · Mobile = 14.78% · Prcard = 9.99%
- Cash share fell **40 percentage points** in 10 years
- Card users take longer, more expensive trips: avg fare $27.16 (card) vs $17.34 (cash) in 2023
- Credit card = 40.86% of trips but **55.16% of revenue** in 2023
- **Recommendation:** Operators resisting card/mobile payments are forgoing the majority of high-value revenue. Payment terminal investment has a clear, data-backed ROI.

---

## Bonus Insight 2: Peak Hour & Community Area Demand Heatmap

**Model:** `bonus2_demand_heatmap`

### Business Value
Identifies when and where demand is highest to enable data-driven driver deployment, reducing wait times and increasing utilization.

### Key Findings
- **The Loop (Area 32)** and **Near North Side (Area 8)** occupy all top 9 demand slots — 2 of Chicago's 77 community areas dominate completely
- **Peak hours: 10am–2pm on weekdays** (not the morning rush), driven by business, lunch, and tourist traffic
- Average fares in peak slots are low ($9–$11) — short downtown hops; a high-frequency, low-fare-per-trip model
- Weekend demand peaks later in the day, concentrated in entertainment/nightlife areas
- **Recommendation:** Concentrate drivers in The Loop and Near North Side between 10am–2pm on weekdays. For weekends, shift deployment to evening hours.

---

## How to Run

### Prerequisites
- GCP project with BigQuery + Dataform APIs enabled and billing linked
- Dataform service account with BigQuery Data Editor role

### Steps
1. Clone this repo and link it to a new Dataform repository in GCP Console
2. Create a development workspace
3. Confirm `workflow_settings.yaml` points to your project ID and dataset
4. Upload `us_federal_holidays_2013_2023.csv` to BigQuery as `chicago_taxi_analytics.us_federal_holidays`
5. Click **Start execution** and run in this order:
   - `stg_taxi_trips` (~2–3 min, scans 213M rows)
   - `stg_driver_shifts` (~3–5 min, depends on above)
   - All mart models (fast, run in parallel)

### Dependency Graph
```
bigquery-public-data.chicago_taxi_trips.taxi_trips
    └── stg_taxi_trips
            ├── q1_top_tip_earners
            ├── stg_driver_shifts
            │       └── q2_top_overworkers
            ├── q3_holiday_impact ──── us_federal_holidays (CSV)
            ├── bonus1_payment_type_trends
            └── bonus2_demand_heatmap
```

---

## Data Quality Notes

1. **Not all trips are reported.** City of Chicago acknowledges incomplete capture; treat volumes as representative, not exhaustive.
2. **Times rounded to nearest 15 minutes** by source — affects shift duration calculations (±7.5 min per boundary).
3. **Taxi ID is anonymized.** The same vehicle always maps to the same `taxi_id` hash, but the actual license number is not recoverable. Driver-level analysis is possible; identity is not.
4. **Community areas suppressed in some cases** for privacy. Affected rows excluded via `WHERE pickup_community_area IS NOT NULL`.
5. **2012 tail data:** ~31,000 trips appear with 2012 dates despite the dataset starting in 2013. Included in the pipeline; negligible impact on any analysis.

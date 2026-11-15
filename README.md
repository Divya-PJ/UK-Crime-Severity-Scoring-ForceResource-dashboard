# Intelligent UK Crime Severity Scoring & Force Resource Optimisation

**UK police recorded-crime trend & outcome analytics - a Python → SQL → Power BI data-analyst portfolio project.**

A messy 100,000-row police recorded-crime extract is cleaned by an automated Python pipeline, geographically impossible
records are quarantined, every crime type is rolled up to a High / Medium / Low severity tier, two risk features are
engineered, and the result is loaded into a SQL table that is analysed with four correlated-subquery /
conditional-aggregation queries and visualised in a Power BI command centre.

| | |
|---|---|
| **Stack** | Python (pandas, NumPy, SQLAlchemy, matplotlib) · SQL (MySQL 8 / MariaDB; SQLite fallback) · Power BI (Power Query, DAX) |
| **Input** | `data/raw/crime_trends_uk_dataset.csv` - 100,000 rows × 16 columns, deliberately unclean |
| **Output** | 94,757 validated rows in table `uk_crime_incident_records` · 4,643 quarantined rows · 4 SQL queries · Power BI build kit |

---

## 1. Quick start (Windows PowerShell)

```powershell
cd crime-trends-severity-dashboard
python -m venv .venv
.\.venv\Scripts\Activate.ps1
pip install -r requirements.txt

python python/run_pipeline.py        # runs steps 1-6 (about 10 seconds)
python python/run_sql_queries.py     # runs the 4 SQL files and saves results to sql/results/
```

Default database = a local SQLite file (nothing to install). To load into **MySQL** instead and run the `.sql` files in MySQL Workbench:

```powershell
# in MySQL Workbench first:  CREATE DATABASE portfolio;
$env:DB_URL = "mysql+pymysql://USER:PASSWORD@localhost:3306/portfolio"
python python/run_pipeline.py
```

## 2. Repository map

```
├── README.md
├── requirements.txt
├── quarantined_crime_data.csv        <- 4,643 rejected rows, original text + reason (audit file)
├── data/
│   ├── raw/                          <- the supplied dataset (unchanged)
│   └── processed/crime_trends_clean.csv   <- cleaned + enriched table (Power BI source)
├── python/
│   ├── config.py                     <- all settings & thresholds
│   ├── step1_ingest.py               <- ingestion, standardisation, dates, coordinates
│   ├── step2_anomaly_filter.py       <- "Intelligent Filter" -> quarantine
│   ├── step3_severity.py             <- Crime_Severity_Tier mapping
│   ├── step4_features.py             <- Case_Escalation_Score, Force_Crime_Volatility_Index
│   ├── step5_load_db.py              <- SQL load + indexes
│   ├── step6_findings.py             <- companion charts + numbers for the README / Power BI check
│   ├── run_pipeline.py               <- runs everything
│   └── run_sql_queries.py            <- executes sql/query_*.sql
├── sql/
│   ├── query_1_silent_hotspot_detector.sql
│   ├── query_2_crime_spike_vulnerability.sql
│   ├── query_3_force_outcome_stress_test.sql
│   ├── query_4_false_severity_alarm_audit.sql
│   └── results/                      <- CSV output of every query
├── powerbi/                          <- Power Query (M), DAX, theme, build guide, expected values
├── docs/
│   ├── data_quality_report.md        <- every cleaning decision, counted (auto-generated)
│   └── figures/                      <- Python companion charts
└── reports/headline_metrics.json
```

## 3. The pipeline, end to end

| Step | What happens | Key numbers |
|---|---|---|
| **1 Ingest & transform** | Everything read as text; exact duplicates dropped; 137 force-name spellings → 20 forces; 116 crime-type spellings → 14 categories; four month formats and five date formats parsed; coordinates forced numeric; LSOA codes upper-cased | 600 duplicates removed · 2,927 LSOA codes repaired (`e01…` → `E01…`) |
| **2 Intelligent Filter** | Quarantine rows with coordinates outside the UK box (lat 49-61, lon -8.5-2) **or** an LSOA code that is not `E01` + 6 digits | **4,643** quarantined (1,756 coordinates · 2,838 LSOA · 49 both) |
| **3 Severity tiers** | Canonical mapping, applied after spelling is standardised (below) | High 23,076 · Medium 25,811 · Low 45,870 |
| **4 Feature engineering** | `Case_Escalation_Score`, `Force_Crime_Volatility_Index` (vectorised) | Average escalation score 67.0 |
| **5 Database load** | SQLAlchemy → `uk_crime_incident_records`, primary key + 5 indexes, row-count assertion | 94,757 rows |
| **6 Findings** | Python charts + numbers used below and to cross-check Power BI | |

### Severity-tier mapping

| Tier | Crime types |
|---|---|
| **High** | Robbery · Violence and sexual offences · Possession of weapons |
| **Medium** | Burglary · Vehicle crime · Criminal damage and arson · Drugs |
| **Low** | Anti-social behaviour · Bicycle theft · Shoplifting · Other theft · Public order · Theft from the person · Other crime |

Applied as a dictionary lookup on the standardised `crime_type`; the pipeline asserts that no row is left unmapped.

### Engineered features

* **`Case_Escalation_Score`** - start 50; +30 High; +15 Medium; +20 outcome blank or "Under investigation"; −10 Business victim; clipped to 0-100.
* **`Force_Crime_Volatility_Index`** - coefficient of variation `std ÷ mean × 100` of each force's monthly crime counts (44 months, sample std). Range **7.8% (Cheshire) → 11.8% (West Midlands)**.

## 4. Data-cleaning decisions you should know about

Full counts: [`docs/data_quality_report.md`](docs/data_quality_report.md).

| Issue | Decision | Why |
|---|---|---|
| **The brief conflicts with itself** on geocoding | Step 2 quarantines *present-but-impossible* geography (outside-UK coordinates, malformed LSOA). **Missing** coordinates / LSOA stay in the table and set `geo_quality_flag = 1` (10.8% of records). SQL Query 1 uses that flag | If every flagged row were quarantined, Query 1 ("records that carry a geocoding-quality flag") could never return anything. A blank value cannot be proven wrong |
| `police_force_region` wrong on 1,870 rows | Re-derived from the standardised force name | Rows whose force was written as GMP / TVP / "W Midlands Police" / "police scotland" were all stamped **London** |
| Ambiguous slash dates (13,865 rows) | Resolved with the row's own `month` column - picks the reading whose year-month matches (100% of unambiguous dates agree with `month`) | `05/06/2024` could be 5 June or 6 May |
| `reported_by` vs `falls_within` differ on 2,639 rows | `police_force` = standardised `reported_by` (the recording force, consistent with the region); `falls_within` kept, mismatch flagged | |
| LSOA codes | Lower-case repaired; truncated codes such as `E01017` quarantined | |
| `context` column | Dropped - 100% blank | |
| `victim_type` blank on ~15% | Kept as NULL ("Not recorded" in Power BI) | Matches real reporting patterns |
| Repeated `crime_id` on different rows (588) | Kept, flagged; surrogate key `record_id` added | Different crimes |

## 5. SQL analysis (no CTEs, no window functions)

Run on MariaDB 10.11 (MySQL-compatible) and cross-checked against independent pandas calculations.

| Query | Technique | Result |
|---|---|---|
| **1 Silent Hotspot** | multi-condition `HAVING` + nested `SUM(CASE…)` | **5** crime type × region combinations are >40% unresolved *and* >10% geo-flagged - e.g. **Robbery in the South West** (47.6% unresolved, 13.6% flagged), **Robbery in London** (43.1% / 11.8%) |
| **2 Crime Spike** | correlated subquery in `WHERE` | **23** LSOA-months across 20 areas exceeded double their own monthly average (peak **2.55×**: Cardiff South 010A, May 2023, 15 crimes vs 5.9) |
| **3 Force Outcome Stress Test** | cross-conditional aggregation | **All 20 forces** qualify: High-severity cases take **84-88 days longer** than Low (≈141 vs ≈55 days) |
| **4 False Severity Alarm** | `COUNT()` + conditional `SUM()` | Only **0.64%** of resolved High cases closed within 14 days; highest force **Nottinghamshire 1.19%**, lowest South Yorkshire 0% |

*Query 2 uses an average over the months in which an area recorded crime, plus a floor of 5 crimes (28 spikes without the floor).
Query 3 and 4 use 1st-of-month dates, so "14 days" effectively means "same month" - see the notes in each file.*

## 6. Headline findings

1. **Severity drives speed, not backlog.** High-severity cases take about **141 days** to reach an outcome versus **55** for Low - yet **~35%** of cases are unresolved whatever the tier (35.5% High · 34.6% Medium · 35.0% Low). Severity tiers are operationally real, but the unresolved backlog is not prioritised by them.
   ![tier](docs/figures/01_severity_time_to_outcome_vs_unresolved.png)
2. **The severity tag holds up.** Fewer than **1 in 150** resolved High cases (0.64%) closed in the same month, so "false severity alarms" are rare in every force.
3. **Crime is steady, forces are similar.** About **2,150 crimes a month**; force volatility only spans **7.8-11.8%** and unresolved rates **33.8-36.5%**. Six forces still trip the red alert (above-average escalation *and* volatility): Metropolitan, Police Scotland, PSNI, South Yorkshire, West Midlands, West Yorkshire.
   ![volatility](docs/figures/04_force_volatility_index.png)
4. **A small set of hotspots, and spikes inside them.** **60 LSOAs hold 18.1%** of all geocoded crime (out of ~29,600 areas); Query 2's spikes occur inside these hotspot areas - the best candidates for patrol reallocation. About **8,200 High-severity cases** are still unresolved - the backlog the "Additional Patrol Resourcing" simulation targets (a 10% cut clears ≈ 820).
5. **Geography cannot be trusted at face value.** 10.8% of records lack usable geography, 4.6% of rows were quarantined, and force names had to be repaired on 1,870 region labels. Even after cleaning, **53.6% of Police Scotland cases are plotted south of 54.6°N**, so coordinates are not tied to the recording force.
   ![heatmap](docs/figures/03_crime_type_by_region_heatmap.png)

## 7. Power BI

`powerbi/` contains everything for the single flat `Fact_Crime_Incidents` table (no star schema): Power Query script, DAX
(month-on-month trend, **smart-narrative text measure**, **multi-condition red/green KPI**, **what-if "Additional Patrol Resourcing" 5-25%**),
a theme, a build guide and expected values to validate your build. The `.pbix` must be saved from Power BI Desktop - add it and your
screenshots to `powerbi/` and `powerbi/screenshots/`.

## 8. Limitations

* Synthetic data: severity tier, outcome and victim type are largely independent random fields, so flat relationships (e.g. unresolved rate by tier) describe this dataset, not real policing.
* Coordinates fill a rectangle and ignore force areas - the Power BI map will not follow UK geography (see the build guide).
* `month` and `outcome_month` are month-level, so day-based measures (Queries 3-4) move in ~30-day steps. Some outcome months run into 2027, after the last crime month (Aug 2026).
* "Unresolved" includes outcomes labelled "Under investigation", which carry an outcome month in this extract.

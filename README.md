# Smart City Digital Twin — Exploratory Data Analysis

Exploratory data analysis of a synthetic **Smart City Digital Twin** ecosystem: one static district registry plus four continuous hourly sensor feeds and two discrete event logs, covering a 3-year period across 20 districts.

The goal of this project was not feature engineering — most columns and event flags are already provided — but **validation**: confirming that each flag reflects the physical conditions it claims to represent, that derived metrics aren't redundant, and that statistical outliers are real signal rather than data-quality problems.

## Dataset

| File | Type | Description |
|---|---|---|
| `districts.csv` | Static dimension table | 20 districts with area, population, zoning type, green space %, baseline temperature offset, road capacity, grid capacity |
| `weather.csv` | Hourly sensor feed | Temperature, humidity, rainfall, wind speed, cloud cover, `sensor_status` |
| `traffic.csv` | Hourly sensor feed | Traffic volume, average speed, congestion index, accident/incident counts, rush-hour/weekend/accident flags |
| `power_grid.csv` | Hourly sensor feed | Electricity demand, grid load/stress, solar & wind generation, outage flag/duration |
| `air_quality.csv` | Hourly sensor feed | PM2.5, PM10, NO₂, CO, O₃, synthetic AQI, air quality category, pollution alert flag |
| `public_transport.csv` | Hourly sensor feed | Passenger count, vehicle occupancy, service frequency, delay, service disruption flag |
| `city_events.csv` | Discrete event log | Expected/actual attendance, event type, duration, impact level |
| `emergency_events.csv` | Discrete event log | Emergency type, severity, response time, hospital transport, weather/traffic-related flags |

## Method

The same pipeline was applied to every sensor file:

1. **Structure & dtype check** — timestamp conversion, type consistency
2. **Null diagnosis** — not just counting nulls, but tracing root cause (e.g. sensor outages) and checking for geographic/temporal bias before dropping
3. **Distribution & outlier analysis** — with explicit handling for zero-inflated columns, where standard IQR filtering produces false positives
4. **Time trends** — daily, weekly and seasonal patterns
5. **Correlation analysis** — with attention to cases where Pearson correlation is the wrong tool (zero-inflated or U-shaped relationships)
6. **Flag validation** — checking derived flags against the raw measurements they claim to summarize
7. **Cross-file merges** — district context, weather context, and (for events) range joins against a same-district/same-hour non-event baseline

Discrete event logs (`city_events.csv`, `emergency_events.csv`) have no `sensor_status` mechanism, since they are incident records rather than continuous sensor feeds.

## Key findings

- **Null pattern is consistent across all sensor feeds**: ~1.25% of rows missing all measurements simultaneously, explained entirely by `sensor_status` (Communication-Loss / Storm-Affected), evenly distributed across all 20 districts — dropped with no bias introduced.
- **Urban heat island effect**: green space percentage correlates with average temperature at r = −0.79 across districts.
- **Traffic is one phenomenon, not three**: congestion, speed and volume are tightly linked (r up to −0.96); congestion is demand-driven, not capacity-driven (road capacity only predicts congestion at r = −0.47).
- **Power demand has a dual seasonal peak** (winter and summer), and two independent renewable generation mechanisms: solar is a deterministic daylight cutoff, wind is random and time-independent.
- **Industrial activity is the strongest predictor of air pollution** in the entire project (r = 0.81–0.92), far ahead of traffic or commercial activity.
- **Transit disruptions destroy punctuality without reducing demand**: flagged rows average 56.7 min delay vs 3.9 min normally (~14.5×).
- **City-wide event impact is driven by crowd size, not event type** — large events (concerts, marathons) lift traffic, transit, air quality and power together; small events are statistically indistinguishable from no event, except that power demand rises for every event regardless of size.
- **Emergency outcomes are severity-driven, not type-driven**: hospital transport is ~0% for Low/Medium severity and jumps to ~71–72% for High/Critical, regardless of whether the emergency is traffic, medical, or fire.

## Methodological notes

A few recurring lessons worth flagging for anyone extending this analysis:

- **Pearson correlation fails on zero-inflated columns** (e.g. `accident_count` is ~96% zero) — group means and boxplots are the correct tool instead.
- **Linear correlation can hide U-shaped relationships** — electricity demand vs. temperature is only r = 0.12 despite clear spikes at both heat and cold extremes.
- **Aggregation level changes the answer** — ridership vs. delay is r = 0.17 row-by-row but r = 0.65 at the district level.
- **IQR outlier detection needs zero-inflation-aware handling** — naive IQR flagged ~51,000 false "outliers" on rainfall before restricting analysis to non-zero rain events.
- No values were dropped or capped anywhere in this project except the sensor-outage null rows; every flagged extreme was validated as a genuine peak condition or rare event.

## Repository structure

```
├── data/                     # raw CSV files (not included — see Data Access)
├── notebooks/
│   └── preprocess.ipynb      # full EDA notebook, organized by file with a table of contents
├── README.md
└── requirements.txt
```

## Data access

The raw CSV files are not included in this repository. Place `districts.csv`, `weather.csv`, `traffic.csv`, `power_grid.csv`, `air_quality.csv`, `public_transport.csv`, `city_events.csv`, and `emergency_events.csv` in a `data/` directory before running the notebook.

## Requirements

```
pandas
numpy
matplotlib
seaborn
```

Install with:

```bash
pip install -r requirements.txt
```

## Usage

```bash
jupyter notebook notebooks/preprocess.ipynb
```

The notebook is organized with markdown headings per file and per analysis step, and includes a clickable table of contents at the top for navigation.

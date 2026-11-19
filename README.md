# Weather, runways and airport capacity

This project predicts, for one airport-hour at a time, whether **demand will exceed capacity 1, 3 or 6 hours ahead**. It combines three sources:
- BTS on-time flight data
- NOAA GHCNh hourly station weather
- FAA NASR runway data

Newark (EWR) is the first airport.

## Layout

```
src/
  config.ipynb   shared settings, loaded by every notebook with %run ../config.ipynb
  data/          dataset pipeline (01–06, run in order)
  eda/           exploratory analysis
  modeling/      models
reports/         writeups
```

## Data pipeline (`src/data/`, run in order)

| Notebook | What it does |
|---|---|
| `01_download_bts` | Downloads monthly BTS files (2023-01 → latest) into `data/raw/bts/`, plus GHCNh station files. Rerunnable |
| `02_nasr` | Runway ends (true heading, ILS) and airport timezones from the NASR APT CSVs |
| `03_weather` | GHCNh → quality-controlled hourly UTC weather, flight category, present-weather flags |
| `04_flights` | BTS flights → UTC event times → hourly demand, runway throughput, backlog, delays |
| `05_runway_wind` | Crosswind/headwind per runway end → usable runway counts per hour |
| `06_build_dataset` | Joins everything, builds the 1/3/6-hour-ahead labels, runs the leakage check and label sanity checks → `data/processed/{APT}_hourly.parquet` |

Setup: `uv pip install -r requirements.txt`, then run `src/data/01`–`06` top to bottom. Settings (study airports, months, IATA→FAA airport codes, wind limits, label thresholds) are in `src/config.ipynb`.

Data that has to be added by hand: put the NASR 28-day subscription `APT_BASE.csv`, `APT_RWY.csv` and `APT_RWY_END.csv` in `data/raw/nasr/`.

## Output columns
Full column reference: [reports/dataset.md](reports/dataset.md).
- `feat_*`: model inputs known at the end of hour t, plus the future schedule (published in advance)
- `wxnext_*`, `wxh3_*`, `wxh6_*`: observed weather at t+1 / t+3 / t+6, a stand-in for a forecast. Optional
- `label_over_capacity_next`, `label_over_capacity_h3`, `label_over_capacity_h6`: the targets
- `diag_next_*`: the label's ingredients and alternative thresholds. Never use these as features
- `meta_*`: airport, time, train/test split (train ≤ 2025-12), `meta_active` / `meta_active_h3` / `meta_active_h6` (flights scheduled at the target hour)

## Adding an airport
Add its IATA code and GHCNh station ID to `STUDY_AIRPORTS` in `src/config.ipynb`, then rerun `src/data/01`–`06`.

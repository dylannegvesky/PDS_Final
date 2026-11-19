# EWR Modeling Dataset

*As of 2026-11-19 · file: `data/processed/EWR_hourly.parquet` · built by `src/data/06_build_dataset.ipynb`*

## Overview

- **One row per hour *t* at Newark (EWR).** There are 32,849 rows from 2023-01-01 to 2026-09-30, with 150 columns.
- **Input columns describe hour *t* and earlier.** The only exceptions are the airline schedule, which is published in advance, and the clearly separated future-weather columns.
- **Targets are 1, 3 and 6 hours ahead.** Each says whether the airport is over capacity at t+1, t+3 or t+6.
- **Split:** training is Jan 2023 – Dec 2025 (26,304 rows) and the held-out test set is Jan – Sep 2026 (6,545 rows).
- **Model only "active" rows**, i.e. hours with flights scheduled at the target hour. That's 91% of rows. The overnight hours left out are trivially "fine".
- **Missing values** are left as NaN. Example: average departure delay is blank in hours with no departures, about 18% of rows. Gradient boosting handles these natively; logistic regression needs imputation.

## Column prefixes

- `meta_`: identifiers and bookkeeping. Never model inputs.
- `feat_`: model inputs (93 columns).
- `wxnext_`, `wxh3_`, `wxh6_`: observed weather at t+1, t+3 or t+6, a stand-in for a perfect forecast. Optional inputs, used only for the matching horizon.
- `label_`: the targets.
- `diag_`: the label's ingredients and alternative thresholds, kept for checking. **Never model inputs.**

## Targets

- **Over capacity** = at least 25% of scheduled arrivals delayed 15+ minutes by NAS or weather causes, or at least 3 weather/airspace cancellations.
- `label_over_capacity_next`: over capacity at t+1. Positive in 16.8% of active hours.
- `label_over_capacity_h3`: at t+3. Positive in 16.5%.
- `label_over_capacity_h6`: at t+6. Positive in 16.0%.

## Features (`feat_`)

**Weather at Newark, hour t**
- **Wind:** `wind_kt`, `gust_kt`, `wind_max_kt`. Components `wind_u` and `wind_v`. `wind_variable` flags a variable direction.
- **Visibility and ceiling:** `vis_sm` (capped at 10 sm) and `vis_min_sm` (lowest in the hour). `ceiling_ft` (25,000 means unlimited) and `ceiling_min_ft`.
- **Flight category:** `flight_cat` (0 VFR, 1 MVFR, 2 IFR, 3 LIFR) and `imc` (instrument conditions, i.e. IFR or worse).
- **Precipitation:** `precip_mm` (this hour) and `precip_3h` (last 3 hours).
- **Present-weather flags:** `wx_ts` (thunderstorm), `wx_snow`, `wx_freezing`, `wx_fog`, `wx_rain`, `wx_heavy`.
- **Temperature and pressure:** `temp_c`, `dewpoint_c`, `slp_hpa`.
- **Data flags:** `n_specials` counts special weather reports in the hour, a sign of changing conditions. `wx_missing` flags an hour with no observation; values are carried forward up to 2 hours.
- **Lags:** `imc_lag1`, `imc_lag2`, `wind_kt_lag1`, `wind_kt_lag2`.

**Runways and wind, hour t.** A runway end is usable if the crosswind is ≤ 20 kt and the tailwind ≤ 5 kt.
- `n_usable_ends` (out of 6), `frac_usable`.
- `n_usable_physical`: physical runways with a usable end.
- `n_usable_headings`: usable directions, with parallel runways counted once.
- `n_usable_ils_ends`: usable ends with an instrument landing system.
- `n_usable_arrival_ends`: usable ILS ends in instrument conditions, otherwise all usable ends.
- `min_crosswind_kt`, `max_headwind_kt`, `crosswind_on_best_kt`: wind on the most favorable runway.

**Airport operations, hour t**
- **Scheduled flights:** `sched_dep`, `sched_arr`, `sched_total`.
- **Actual throughput** (counted by takeoff/landing time): `actual_dep`, `actual_arr`, `actual_total`. Lags `actual_total_lag1` and `actual_total_lag2`.
- **Cancellations and diversions:** `cancel_dep`, `cancel_arr`, `cancel_wx_nas` (weather/airspace cancellations only), `diverted_arr`.
- **Delays:** `arr_del15` and `dep_del15` (flights delayed 15+ min), `arr_delay_mean` and `dep_delay_mean` (signed minutes), `arr_nas_wx_delayed`.
- **Delay share:** `nas_wx_share` (share of arrivals with NAS/weather delay), with lags `nas_wx_share_lag1` and `nas_wx_share_lag2`.
- **Taxi times:** `taxi_out_mean` and `taxi_in_mean`. `taxi_out_excess` is taxi-out minus the usual value for that local hour and month, computed from training months only.
- **Backlog** (flights behind schedule and still waiting for the runway, reset at 04:00 local):
  - `backlog`, split into `backlog_dep` and `backlog_arr`
  - `backlog_roll4`: net over the last 4 hours, which doesn't reset
  - `backlog_growth`: change during the hour
  - lags `backlog_lag1`, `backlog_lag2`, `backlog_growth_lag1`, `backlog_growth_lag2`
- **Current label state:** `over_capacity` (whether hour t itself is over capacity), with lags `over_capacity_lag1` and `over_capacity_lag2`. `over_capacity_last6` counts over-capacity hours in the last 6 hours.
- **Demonstrated capacity:** `demonstrated_capacity`, the 95th percentile of hourly throughput over the previous 28 days. It falls automatically after runway closures or FAA caps.

**Future schedule** (published in advance, so allowed as inputs)
- **Next hour:** `sched_dep_next`, `sched_arr_next`, `sched_total_next`. `expected_runway_next` counts flights the schedule puts on the runway in t+1, cancellations included.
- **Demand:** `runway_demand_next` = backlog + `expected_runway_next`. `demand_vs_demonstrated` = that ÷ demonstrated capacity.
- **Longer horizons**, for H = 3 and 6:
  - `expected_runway_h{H}`: runway flights in hour t+H
  - `sched_total_h{H}`: scheduled flights in hour t+H
  - `expected_runway_sum_h{H}`: runway flights over t+1 … t+H

**Calendar (local time)**
- `hour_local` with `hour_sin` and `hour_cos`, `dow` (day of week, 0 = Monday), `month`, `holiday` (US federal holiday).

## Future weather (`wxnext_`, `wxh3_`, `wxh6_`)

The same 13 columns at t+1, t+3 or t+6:
- wind: `wind_kt`, `gust_kt`
- visibility and ceiling: `vis_sm`, `ceiling_ft`
- `precip_mm`, `imc`, `flight_cat`
- flags: `wx_ts`, `wx_snow`, `wx_freezing`
- runways: `n_usable_ends`, `n_usable_ils_ends`, `n_usable_arrival_ends`

These are observed values, not forecasts, so results that use them are an upper bound on what a forecast could add.

## Diagnostics (`diag_`) and metadata (`meta_`)

- **Alternative labels at t+1:** `diag_next_label_strict`, `diag_next_label_default`, `diag_next_label_loose`, `diag_next_label_nas_only`.
- **Label ingredients at t+1:** `diag_next_nas_wx_share`, `diag_next_cancel_wx_nas`, `diag_next_backlog_growth`, `diag_next_taxi_out_excess`.
- **Metadata:**
  - `meta_airport`
  - `meta_hour_utc` (start of hour t, UTC) and `meta_local_time` (New York time)
  - `meta_split` (`train` / `test`)
  - `meta_active`, `meta_active_h3`, `meta_active_h6`: flights scheduled at the target hour

## Known caveats

- Demand covers only US airlines that report to BTS.
- Runway features use the current NASR layout, so past closures are missed.
- Newark is the only weather station.
- The label is a congestion proxy, not the FAA's official capacity rates.

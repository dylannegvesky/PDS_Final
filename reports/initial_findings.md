# EWR Capacity Prediction: Initial Findings

*As of 2026-11-19*

## Goal and data

We predict whether demand will exceed capacity at Newark (EWR) 1, 3 or 6 hours ahead, using conditions in the current hour. The dataset has one row per hour from January 2023 to September 2026, about 32,850 hours built from roughly 956,000 flights.

- **Flights:** BTS on-time data, used for scheduled demand, actual runway throughput (wheels-off and wheels-on times), delays and their causes, and cancellations.
- **Weather:** NOAA GHCNh hourly observations at Newark, used for wind, visibility, ceiling, precipitation and thunderstorms.
- **Runways:** FAA NASR runway headings and instrument landing system (ILS) equipment. Combined with the wind, these give the number of runway ends usable each hour.

Demand is the flights that need the runway in an hour: those scheduled for it plus any leftover backlog from earlier delays.

## The over-capacity label

Capacity is never observed directly, so the label marks hours that show symptoms of it being exceeded. Public sources that give capacity directly either need an FAA account (ASPM) or keep only 15 days of history (FAA Ground Delay Program and Ground Stop advisories). An hour is labeled over capacity if either of these happens:

- at least 25% of arrivals are delayed 15+ minutes by causes the FAA attributes to the airport and airspace system (NAS delay) or to weather, or
- at least 3 flights are cancelled for weather or airspace reasons.

This marks 16% of hours with flights. An earlier rule that also counted backlog growth was dropped. The hours it flagged had normal delays and clustered at schedule peaks, so it was picking up how the schedule bunches flights, not capacity limits.

The label passes a sanity check against measures it doesn't use:

- Labeled hours have much longer taxi-out times.
- About 63% of their delay minutes are attributed to the airspace system, compared with 17% in other hours.
- Low visibility and thunderstorms are more common.
- At the same level of demand, labeled hours move about 10% fewer flights.

The label describes reduced capacity rather than a hard ceiling. It also counts recovery hours after a disruption, when delayed flights are still clearing the runway.

## Findings

The strongest predictor of capacity problems is the airport's current state: delays already building, backlog and cancellations. Weather does predict capacity problems, and it becomes more useful the further ahead we predict. Results held in both validation years, 2024 and 2025, and were checked with day-level bootstrap confidence intervals.

**Disruptions come in streaks.** Three in four over-capacity hours come right after another one. Simply repeating the current hour is right 92% of the time. So we judge models mainly on **new disruptions**: hours when the airport is currently fine, asking whether trouble will start.

**One hour ahead, weather adds almost nothing.** Current delays and backlog predict new disruptions at about 6× chance. Adding weather to those gives no reliable improvement. The weather's effect is already visible in delays by the time it matters.

**Further ahead, weather adds real value.**

- Six hours ahead, the current-delay signal has faded. Adding weather to the operational features improves the score by about 14%, and the confidence interval is above zero in both years.
- Weather and the schedule alone come close to the operational features at 6 hours. Operational features still score higher at every horizon.

**A perfect weather forecast helps only slightly.** Giving the model the actual weather at the target hour added little beyond Newark's current weather, at any horizon.

**Sudden disruptions are hard to predict.** One hour ahead, disruptions out of truly calm hours (no delays building, nothing wrong in the last two hours) are close to unpredictable. Six hours ahead, the model scores about 3× chance on them, and weather drives much of that improvement.

## Limitations and next steps

- **The label is a proxy.** It captures congestion symptoms in BTS data, not the FAA's actual arrival and departure rates.
- **Partial demand.** BTS covers only US airlines that report on-time data. Foreign carriers, cargo and general aviation are missing.
- **One weather station.** Storms along arrival routes or at origin airports are invisible to Newark's station.
- **Current runway layout only.** NASR is a current snapshot, so historical runway closures aren't reflected in the wind features.
- **2026 still held out.** The 2026 test months haven't been used yet.

Next, we plan to test weather from stations along the main arrival routes. Then we'll build the final 6-hour model, evaluate it once on 2026, and extend the pipeline to other major airports.
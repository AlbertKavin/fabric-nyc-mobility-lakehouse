# DAX Measures

All measures live on `fact_trips` in the semantic model `sm_nyc_mobility`.

| Measure | DAX | Purpose |
|---|---|---|
| Total Trips | `COUNTROWS(fact_trips)` | Number of valid trips |
| Total Revenue | `SUM(fact_trips[total_amount])` | Sum of total fare incl. tips and fees |
| Avg Fare | `AVERAGE(fact_trips[fare_amount])` | Average base fare |
| Avg Tip % | `AVERAGE(fact_trips[tip_pct])` | Average tip as a % of fare |
| Trips per Weather Hour | `DIVIDE([Total Trips], DISTINCTCOUNT(dim_weather[weather_hour]))` | Fair comparison of rainy and dry periods, since there are more dry hours |

# Data Model

## Overview
A star schema in the Lakehouse `lh_nyc_mobility` (Gold layer), served through the semantic model `sm_nyc_mobility`.

## Grain
One row in `fact_trips` = one valid yellow taxi trip.

## Tables
| Table | Type | Key | Description |
|---|---|---|---|
| fact_trips | Fact | trip_id | Trip measures: distance, duration, fare, tip, total |
| dim_date | Dimension | date_key | Calendar attributes (year, month, day name, weekend flag) |
| dim_weather | Dimension | weather_hour | Hourly temperature, rain and wind, with rain flag and temperature band |
| dim_zone | Dimension | zone_key | NYC taxi zones with borough and service zone |
| dim_payment_type | Dimension | payment_type_key | Payment type labels |

## Relationships
| From (dimension) | To (fact) | Cardinality | Active |
|---|---|---|---|
| dim_date[date_key] | fact_trips[date_key] | 1:many | Yes |
| dim_weather[weather_hour] | fact_trips[pickup_hours] | 1:many | Yes |
| dim_zone[zone_key] | fact_trips[pu_zone_key] | 1:many | Yes |
| dim_zone[zone_key] | fact_trips[do_zone_key] | 1:many | No (role-playing, used with USERELATIONSHIP) |
| dim_payment_type[payment_type_key] | fact_trips[payment_type_key] | 1:many | Yes |

## Design decisions
- Dimensions connect only to the fact table (star schema), never to each other.
- `dim_zone` is used twice (pickup and drop-off), so the drop-off relationship is inactive.
- Trips join to weather on the pickup hour.

## Data quality (Silver)
Rows are rejected into `ops.dq_rejects` with a reason: null timestamps, drop-off before pickup, pickup outside the file's month, non-positive fare, bad distance, trips over 6 hours, and missing zones.

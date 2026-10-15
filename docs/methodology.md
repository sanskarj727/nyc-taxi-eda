# Methodology

How the notebook gets from the raw TLC files to the analysis set, and why each choice was made. Row counts are from the original run on the full 2023 data.

## 1. Sampling

**Problem.** One month of yellow-taxi data holds millions of rows (January 2023 alone has 3,066,766), so loading the whole year at once is impractical on a laptop.

**Design.** Each monthly file is loaded on its own. For every pickup **date and hour**, 0.75% of that slot's trips are drawn at random (`SAMPLE_FRACTION = 0.0075`, seed 42). Sampling per date-hour slot, rather than drawing 0.75% from the whole file, guarantees every slot is represented in proportion, so quiet hours such as 4 am are not missed by chance and the hourly, weekly and seasonal patterns survive.

**Result.** 315,506 trips, saved to `data/processed/sampled_nyc_taxi.parquet` and reused on later runs.

**Things to know.**

- The seed is fixed, so the same input files produce the same sample.
- pandas rounds the sample size per slot, so a slot with fewer than about 67 trips contributes no rows. This slightly under-represents the very quietest slots.
- Because only a sample is analysed, counts are *sampled counts*. Where the notebook estimates real volumes it divides by the sampling fraction.

## 2. Cleaning

| Step | Problem | Rule applied |
| --- | --- | --- |
| Duplicate column | The sample has `airport_fee` and `Airport_fee`, which differ only in capitalisation | Merged into `airport_fee_combined` (whichever is populated) |
| Unneeded columns | `date` and `hour` are sampling helpers; `store_and_fwd_flag` only records vehicle upload buffering | Dropped |
| Negative amounts | Negatives in every money column: `fare_amount` 3,150, `improvement_surcharge` 3,111, `total_amount` 3,114, `mta_tax` 3,035, `congestion_surcharge` 2,506, `extra` 1,597, `airport_fee_combined` 421, `tolls_amount` 199, `tip_amount` 14 | Absolute value for fare and surcharge columns; negative totals rebuilt from their components; trips with a negative tip or toll dropped |
| Missing values | `passenger_count`, `RatecodeID`, `congestion_surcharge` and `airport_fee_combined` are each missing for 3.27% of trips (10,323 rows) | Imputed with the column mode |
| Zero passengers | 4,735 trips | Treated as missing; replaced with the mode (1 passenger) |
| Implausible passenger counts | 4 trips with 8 passengers | Removed (kept `passenger_count` ≤ 6) |
| Impossible trips | Near-zero distance with a fare over $300; zero distance and zero fare between different zones; distance over 250 miles | Removed |
| Undefined payment type | `payment_type` 0 is not defined by the TLC; about 3.3% of trips | Removed |
| Zero fare, total or distance | Usually an unrecorded value | Excluded from the revenue and fare analyses via a separate frame, `df_filtered` |

**Row counts through the pipeline**

| After | Rows |
| --- | --- |
| Sampling | 315,506 |
| Negative-amount and missing-total handling | 315,290 |
| Passenger and impossible-trip rules | 315,253 |
| Payment-type, zero-value filters and zone join (analysis set) | 300,792 |

**An observation about the missing block.** The four columns above are each missing for the same 3.27% of trips, and `payment_type` 0 also covers about 3.3% of trips. The identical shares suggest these are largely the same records. If so, they were imputed in step 2.2 and then removed with the payment-type rule in step 2.3, so the imputation had little effect on the final results. Simply dropping those rows earlier would be equivalent and simpler.

**Outlier diagnostics.** An IQR-based count of outliers per column is reported but nothing is removed on that basis: fares and distances have genuinely long right tails, so the explicit rules above decide what is dropped.

## 3. Derived features

| Feature | Definition |
| --- | --- |
| `trip_duration` | Drop-off minus pickup time, in minutes |
| Hour, day of week, month, quarter | Taken from the pickup timestamp |
| `fare_per_mile` | `fare_amount / trip_distance` |
| `fare_per_mile_per_passenger` | `fare_per_mile / passenger_count` |
| `tip_percentage` | `tip_amount / fare_amount × 100` |
| `distance_tier` | 0–2 miles, 2–5 miles, 5+ miles |
| Route speed | Total distance divided by total time (hours) for each pickup-zone → drop-off-zone route and pickup hour |

## 4. Geographic join

Trips are left-joined to the TLC taxi-zone shapefile on `PULocationID = LocationID`, giving each trip a zone name and borough. Trips per zone are counted, merged back onto the zone polygons, and drawn as a choropleth. Pickup-to-drop-off ratios and night-time rankings are computed per zone.

## 5. Scaling sampled counts

To estimate real volumes, sampled counts are divided by 0.0075 (for example, the busiest hours and the weekday versus weekend comparison). These are estimates: the cleaning steps also removed some sampled rows, so the scaled totals are approximate.

## 6. Assumptions and trade-offs

- **Negative amounts are treated as sign errors.** If they are actually refunds or reversals, dropping them would be more appropriate than taking absolute values.
- **Mode imputation** is simple and suits low-cardinality columns, but it ignores any relationship with other columns.
- **Zero-distance trips are excluded** from fare analyses even though a trip within one zone can legitimately have a distance near zero.
- **Means of per-trip ratios** are used for fare per mile and tip percentage, which short trips can inflate.
- **Tips cover card payments only.** The TLC records tips automatically for credit cards but not for cash, so tip analysis effectively describes card payers.

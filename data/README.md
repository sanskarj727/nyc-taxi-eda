# Data

The data is public and is not stored in this repository. It comes from the NYC Taxi & Limousine Commission (TLC).

**Download page:** https://www.nyc.gov/site/tlc/about/tlc-trip-record-data.page

## What to download

1. **Yellow taxi trip records for 2023**: twelve monthly Parquet files, named `yellow_tripdata_2023-01.parquet` through `yellow_tripdata_2023-12.parquet`.
2. **Taxi zone shapefile**: listed on the same page under taxi zones. Download the shapefile bundle and unzip it.

Column definitions are in the TLC's [yellow taxi data dictionary](https://www.nyc.gov/assets/tlc/downloads/pdf/data_dictionary_trip_records_yellow.pdf).

## Folder layout

Place the files like this (the shapefile's companion files must stay together in one folder):

```
data/
├── raw/
│   ├── yellow_tripdata_2023-01.parquet
│   ├── yellow_tripdata_2023-02.parquet
│   └── ... (through 2023-12)
├── taxi_zones/
│   ├── taxi_zones.shp
│   ├── taxi_zones.shx
│   ├── taxi_zones.dbf
│   ├── taxi_zones.prj
│   └── ... (other files from the bundle)
└── processed/
    └── sampled_nyc_taxi.parquet     # created by the notebook on first run
```

`data/raw/`, `data/taxi_zones/` and `data/processed/` are listed in `.gitignore`, so they are never committed.

## Notes

- The notebook draws a 0.75% sample of each month (315,506 trips in the original run) and saves it to `data/processed/`. Delete that file to force a fresh sample.
- The TLC occasionally revises its files, so a fresh download may differ slightly from the one this analysis was written against.

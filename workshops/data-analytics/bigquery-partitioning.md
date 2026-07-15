# BigQuery Cost Control with Partitioning

> **Category:** Data & Analytics · **Level:** Beginner · **Duration:** ~25 min · **Cost:** Free tier (BigQuery sandbox / free monthly quota)

## Exam relevance

BigQuery bills by bytes scanned. Case studies like **TerramEarth** hinge on controlling analytics cost at scale. Partitioning is the single most effective lever, and the PCA exam expects you to reason about it quantitatively.

## Objective

Measure — not just describe — the drop in scanned bytes when querying a date-partitioned table with a date filter versus without one.

## Prerequisites

- A GCP project with BigQuery API enabled
- A CSV dataset with a date column (a public dataset works, e.g. `bigquery-public-data.san_francisco_bikeshare.bikeshare_trips`, or generate a synthetic monthly CSV)

## Steps

### 1. Stage the data in Cloud Storage (skip if using a public dataset)

```bash
gsutil mb -l europe-west9 gs://pca-workshop-$RANDOM
gsutil cp trips.csv gs://pca-workshop-<bucket-suffix>/trips.csv
```

### 2. Create a dataset

```bash
bq mk --location=EU pca_dataset
```

### 3. Create a table partitioned by date

```bash
bq mk --table \
  --time_partitioning_type=DAY \
  --time_partitioning_field=trip_date \
  pca_dataset.trips_partitioned \
  trip_date:DATE,start_station:STRING,end_station:STRING,duration_sec:INTEGER
```

Load the data:

```bash
bq load --source_format=CSV --skip_leading_rows=1 \
  pca_dataset.trips_partitioned \
  gs://pca-workshop-<bucket-suffix>/trips.csv
```

### 4. Run an unfiltered query and note bytes scanned

```bash
bq query --use_legacy_sql=false --dry_run \
'SELECT start_station, COUNT(*) FROM pca_dataset.trips_partitioned GROUP BY start_station'
```

The dry run reports the full table size scanned (e.g. several GB).

### 5. Run the same query filtered on a single partition

```bash
bq query --use_legacy_sql=false --dry_run \
"SELECT start_station, COUNT(*) FROM pca_dataset.trips_partitioned WHERE trip_date = '2026-07-15' GROUP BY start_station"
```

The bytes scanned drops sharply — BigQuery only reads the matching partition(s).

## Cleanup

```bash
bq rm -f -t pca_dataset.trips_partitioned
bq rm -f -d pca_dataset
gsutil rm -r gs://pca-workshop-<bucket-suffix>
```

## Key takeaways

- Partition pruning happens automatically when the filter is on the partitioning column.
- Cost and latency both drop — this is why table design is a first-class architecture decision, not an afterthought.
- For TerramEarth-style case studies, combine partitioning with clustering on high-cardinality filter columns for further savings.

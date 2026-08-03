# Bronze Layer Data Quality Findings

Profiling performed via `notebooks/01_bronze_data_profiling.ipynb` after landing all 9 Olist CSVs in `Files/bronze/olist/`.

## Summary

| Dataset | Raw row count | Silver row count | Notes |
|---|---|---|---|
| orders | 99,441 | 99,441 | Matches known dataset size; baseline confirmed |
| customers | 99,441 | 99,441 | Clean |
| geolocation | 1,000,163 | 27,912 | Aggregated to zip-code centroids by design — see Finding 3 |
| order_items | 112,650 | 112,650 | Clean, zero orphaned FKs — see Finding 5 |
| order_payments | 103,886 | 103,886 | Clean, zero orphaned FKs — see Finding 5 |
| **order_reviews** | **104,162** | **98,410** | **Duplicates + orphaned foreign keys — see Finding 2** |
| **products** | 32,951 | 32,951 | **610 rows missing category + photo count — see Finding 4** (note: `product_name_lenght` is misspelled in the *source* data, not a transcription error) |
| sellers | 3,095 | 3,095 | Clean |
| product_category_name_translation | 71 | 71 | Clean |

## Finding 1: All date/timestamp columns loaded as strings

**Issue:** Every date/timestamp column across all files (e.g. `order_purchase_timestamp`, `order_approved_at`, `review_creation_date`) was inferred as `string` type by Spark's default CSV reader.

**Root cause:** Spark does not infer schema types unless explicitly instructed (`inferSchema=True`) or given an explicit schema.

**Fix (Silver layer):** Explicit `to_timestamp()` casting applied during Bronze → Silver transformation for all date columns.

## Finding 2: Reviews table — duplicate rows and orphaned foreign keys

**Issue:** The reviews table showed:
- 104,162 total rows
- 102,958 distinct `review_id` values → **1,204 duplicate `review_id` rows**
- 99,743 distinct `order_id` values → higher than the 99,441 orders that actually exist
- **4,938 review rows reference an `order_id` that does not exist in the orders table** (302 unique orphaned order_ids)

**Analysis:** The orphaned order_ids are concentrated — 302 unique missing order_ids account for 4,938 rows, meaning the same bad records are duplicated multiple times. This suggests the duplication and orphan issues share a common root cause (e.g. a broken upstream join or repeated test-data export), rather than being two unrelated problems.

**Decision:**
1. Deduplicate on `review_id`, keeping the most recent row by `review_answer_timestamp`.
2. Inner-join reviews to orders (not left join) to drop orphaned reviews rather than carry forward nulls or unjoinable records.
3. **Documented data loss:** intentional, not an oversight — reviews with no valid parent order cannot be attributed to any business entity (customer, product, order value) and would corrupt any order-level aggregation if kept.

**Implemented and verified (`notebooks/02_silver_orders_reviews.ipynb`):**

| Stage | Row count | Change |
|---|---|---|
| Raw reviews (Bronze) | 104,162 | — |
| After dedup (keep latest by `review_answer_timestamp`) | 102,958 | −1,204 exact duplicates removed |
| After dropping orphaned reviews (inner join to orders) | 98,410 | −4,548 orphaned rows removed |
| **Final `silver_reviews`** | **98,410** | **−5,752 total (5.5% of raw reviews)** |

Note: the orphan-removal count here (4,548) is lower than the 4,938 orphaned rows found in the raw Bronze profiling, because some duplicate rows removed during dedup were also orphaned — the two issues overlap rather than stack additively.

**Post-write verification performed:**
- Row counts in the written Delta tables (`silver_orders`, `silver_reviews`) matched the counts reported during the transform (99,441 / 98,410) — confirms no silent write corruption.
- `DESCRIBE silver_orders` confirmed all 5 date columns persisted as `timestamp` type in the actual stored table, not just the in-memory dataframe.
- `GROUP BY review_id HAVING COUNT(*) > 1` against `silver_reviews` returned zero rows — confirms no duplicate review_ids survived.
- `LEFT ANTI JOIN` of `silver_reviews` against `silver_orders` on `order_id` returned zero rows — confirms no orphaned reviews survived.

## Finding 3: Geolocation duplication is by design, not a defect

**Observation:** ~1,000,163 rows exist for only ~19,000 unique Brazilian zip codes — heavy duplication.

**Decision:** This is *not* treated as a data quality issue. The dataset intentionally contains multiple lat/long pings per zip code.

**Implemented (`notebooks/03_silver_remaining_tables.ipynb`):** Aggregated to one row per (`zip_code_prefix`, `city`, `state`), taking the average lat/long as a centroid. Result: 1,000,163 raw rows → **27,912 zip-code centroid rows**.

## Finding 4: Products — 610 rows missing category, correlated with missing photo count

**Issue:** 610 of 32,951 products (~1.9%) have a null `product_category_name`, meaning they cannot be joined to `silver_category_translation` for an English category label.

**Deeper analysis:** Rather than treat this as an isolated null, checked whether it correlated with other missing fields on the same rows:
- **610 / 610** of these products are *also* missing `product_photos_qty`
- Only **1 / 610** is missing `product_weight_g`

**Conclusion:** This is not two unrelated data quality issues — it's one shared pattern. These 610 products appear to have had an entire metadata block (category + photo count) fail to populate together during ingestion, while physical dimensions (weight, size) were captured through an apparently different, largely unaffected path.

**Decision:** Kept these rows in `silver_products` rather than dropping them (unlike orphaned reviews) — a missing category is a reporting/enrichment gap, not a referential integrity violation, and the products are still valid, sellable entities. Any category-based reporting (e.g. "revenue by category") must explicitly handle this null bucket (~1.9% of the catalog) rather than silently excluding it.

## Finding 5: Order items and payments — referential integrity confirmed clean

Unlike reviews, both `order_items` and `payments` were checked against their foreign keys and showed **zero orphaned references**:
- `order_items`: 0 orphaned `order_id`, `product_id`, and `seller_id` references (112,650 rows, fully clean)
- `payments`: 0 orphaned `order_id` references (103,886 rows, fully clean)

This is worth stating explicitly — not every table in this dataset was assumed broken. Each was checked individually, and referential integrity issues were found and fixed only where they actually existed (reviews), rather than applied uniformly across all tables.

## Finding 6: Silver geolocation — accent variants caused a downstream join fan-out

**Context:** While building `control_weather_ingestion` (see `notebooks/04_bronze_weather_ingestion.ipynb`), joining the top 15 cities by order volume to `silver_geolocation` (to get lat/long) produced **23 rows instead of 15** — an unexpected fan-out.

**Diagnosis:** Grouping the fan-out by original zip code showed 8 of the 15 zip codes matched 2 rows each in `silver_geolocation`. Inspecting those rows directly revealed the root cause precisely:

| zip_code_prefix | city (variant 1) | city (variant 2) |
|---|---|---|
| 13212 | jundiaí | jundiai |
| 24220 | niterói | niteroi |
| 38400 | uberlândia | uberlandia |
| ...(+5 more) | | |

The same city, same zip code prefix, same state — but two spellings differing only in Portuguese accent characters (á/í/ó vs. unaccented a/i/o). Since `silver_geolocation` was aggregated by `(zip_code_prefix, city, state)`, these accent variants were treated as distinct cities and each received its own centroid row, rather than being merged into one.

**Why this matters:** `silver_geolocation` had been marked "clean" earlier (see Finding 3) because raw row-count aggregation (1,000,163 → 27,912) looked correct on its own. This bug only surfaced when the table was used as a **join source** downstream — a good illustration that "row count looks right" doesn't guarantee "this table is safe to join on."

**Fix:** Added a canonical dedup step — one row per `zip_code_prefix`, selecting deterministically (ordered by city name) before joining:

```python
geo_dedup_window = Window.partitionBy("geolocation_zip_code_prefix").orderBy("geolocation_city")
df_geo_canonical = (
    df_geo_silver
    .withColumn("rn", row_number().over(geo_dedup_window))
    .filter(col("rn") == 1)
    .drop("rn")
)
```

**Result:** Geolocation rows before canonical dedup: 27,912 → after: 19,015 unique (zip_code_prefix, state) combinations. Re-running the city join with `df_geo_canonical` produced exactly 15 rows (one per city), and the resulting control table correctly built 390 rows (15 cities × 26 months) instead of 598.

**Takeaway:** Aggregated/centroid tables need a single canonical record per key *before* being used as a join source — an aggregation that looks correct in isolation can still fan out unexpectedly once joined to something else.

`03_silver_remaining_tables.ipynb` introduces a single parameterized function (`clean_and_write_table`) that handles load, type casting, foreign-key validation, and Delta write for any row-level table, driven by a config of column names and FK checks passed per call. This replaces what would otherwise be 5 near-identical, copy-pasted transformation blocks (customers, sellers, category translation, products, order_items, payments) with one reusable, testable unit — a metadata-driven pattern that would scale to additional tables without linear code growth.

Geolocation was deliberately **excluded** from this function, since it requires aggregation semantics (grouping to zip-code centroids) rather than row-level casting/validation — forcing it into the same function would have blurred its contract rather than simplified it.

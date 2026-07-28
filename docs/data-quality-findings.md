# Bronze Layer Data Quality Findings

Profiling performed via `notebooks/01_bronze_data_profiling.ipynb` after landing all 9 Olist CSVs in `Files/bronze/olist/`.

## Summary

| Dataset | Row count | Notes |
|---|---|---|
| orders | 99,441 | Matches known dataset size; baseline confirmed |
| customers | 99,441 | Clean |
| geolocation | 1,000,163 | Heavy duplication by design (see below) |
| order_items | 112,650 | Clean |
| order_payments | 103,886 | Clean |
| **order_reviews** | **104,162** | **Duplicates + orphaned foreign keys — see below** |
| products | 32,951 | Clean (note: `product_name_lenght` is misspelled in the *source* data, not a transcription error) |
| sellers | 3,095 | Clean |
| product_category_name_translation | 71 | Clean |

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
3. **Documented data loss:** ~4,938 rows (~4.7% of raw reviews) are intentionally dropped. This is a deliberate, documented architectural decision — not an oversight — made because reviews with no valid parent order cannot be attributed to any business entity (customer, product, order value) and would corrupt any order-level aggregation if kept.

## Finding 3: Geolocation duplication is by design, not a defect

**Observation:** ~1,000,163 rows exist for only ~19,000 unique Brazilian zip codes — heavy duplication.

**Decision:** This is *not* treated as a data quality issue. The dataset intentionally contains multiple lat/long pings per zip code. Downstream, this will be aggregated to zip-code centroids (e.g. average lat/long per zip) rather than deduplicated as an error.

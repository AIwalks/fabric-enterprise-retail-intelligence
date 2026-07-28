# Retail & Supply Chain Intelligence Platform (Microsoft Fabric)

An end-to-end retail and supply chain data platform built on Microsoft Fabric, demonstrating multi-source ingestion, medallion architecture (Bronze → Silver → Gold), and real data-quality diagnosis and remediation — not just a happy-path pipeline.

## Why this project exists

Most portfolio data projects show a finished, clean pipeline. This one intentionally documents the messy middle: the bugs hit, the data quality issues found, and the fixes applied — with before/after evidence for each.

## Data sources

| Source type | Source | Ingestion pattern |
|---|---|---|
| Flat files | Olist Brazilian E-Commerce dataset (Kaggle, 9 CSVs) | Manual upload → Bronze landing zone |
| REST API | Open-Meteo (weather/demand correlation) | *Planned* |
| Database | Azure SQL DB (synthetic supplier/inventory data) | *Planned* |

## Architecture

**Medallion pattern**, implemented in a single Fabric Lakehouse (`lh_retail_platform`):

- **Bronze** — raw files landed as-is, no transformation (`Files/bronze/olist/`)
- **Silver** — typed, deduplicated, referentially-clean Delta tables (`silver_orders`, `silver_reviews`, ...)
- **Gold** — *planned*: business-level aggregates for reporting

## Current status

- [x] Fabric workspace + Lakehouse provisioned
- [x] Bronze: all 9 Olist CSVs landed and profiled
- [x] Bronze data quality profiling complete (see `docs/data-quality-findings.md`)
- [x] Silver: orders + reviews transformation — typed, deduplicated, referentially clean (verified; see `docs/data-quality-findings.md`)
- [ ] Silver: remaining 7 tables
- [ ] Gold layer + Power BI reporting
- [ ] API ingestion (Open-Meteo)
- [ ] Database ingestion (Azure SQL)
- [ ] Data Factory pipeline orchestration
- [ ] CI/CD

## Repo structure

```
├── docs/          # data quality findings, design decisions
├── incidents/      # bugs hit and how they were diagnosed/fixed
└── notebooks/      # exported Fabric notebooks (PySpark)
```

## Key engineering decisions

See `docs/data-quality-findings.md` and the `incidents/` folder for the reasoning behind specific choices (e.g. why orphaned reviews are dropped rather than kept, why geolocation duplication is by design not a bug).

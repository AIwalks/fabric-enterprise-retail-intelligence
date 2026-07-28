# Incident: Notebook fails to read Bronze files (400 Bad Request → PATH_NOT_FOUND)

## Symptom

Running a basic file read in a new Fabric notebook:

```python
df_orders = spark.read.option("header", "true").csv("Files/bronze/olist/olist_orders_dataset.csv")
```

failed with:

```
Py4JJavaError: An error occurred while calling o5273.csv.
: Operation failed: "Bad Request", 400, HEAD, .../user/trusted-service-user/Files/bronze/olist/olist_orders_dataset.csv?...
```

## Root cause #1: No lakehouse attached to the notebook

By default, a new Fabric notebook has **no lakehouse context**, even inside a workspace that contains one. Relative paths like `Files/...` have no root to resolve against without an explicit attachment, causing OneLake to reject the HEAD request with a 400.

**Fix:** Explorer panel → **Add lakehouse** → **Existing lakehouse** → select the correct lakehouse. This is a manual, per-notebook step — it does not happen automatically even if only one lakehouse exists in the workspace.

## Root cause #2: File upload had silently not persisted

After fixing the attachment, the same read failed again with a different, more specific error:

```
AnalysisException: [PATH_NOT_FOUND] Path does not exist: abfss://.../Files/bronze/olist/olist_orders_dataset.csv
```

Browsing the `olist` folder directly in the Lakehouse Explorer confirmed it as **empty ("No content")** — the original upload had reported "Succeeded" in the UI but the files had not actually landed in the target folder.

**Fix:** Re-ran the upload, this time verifying the exact target path shown in the upload dialog (`Retail-Supply-Chain-Intelligence/lh_retail_platform/Files/bronze/olist/`) before uploading, and confirmed all 9 files appeared in the folder listing afterward before proceeding.

## Takeaway / process change

File upload success in the UI does not guarantee the files actually persisted to the intended path. **Always visually confirm file presence in the target folder (via Explorer, not just the upload toast) before building anything on top of it.** For anything beyond a one-off manual check, prefer using **"Copy relative path for Spark"** from the right-click menu on the actual file, rather than hand-typing paths, to eliminate typo/casing risk entirely.

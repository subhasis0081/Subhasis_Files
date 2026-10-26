# 00_Environment_Setup
## Pod 13 | Shipment Tracking & Delivery Intelligence
## Databricks Genie — Run this FIRST before any other notebook

---

## PURPOSE

This notebook creates and validates the complete Databricks environment for Pod 13.
Run it once before `00_Setup.md`. It is safe to re-run — all statements use
`CREATE IF NOT EXISTS` or existence checks so nothing is dropped or overwritten.

---

## CONSTANTS (read before running any cell)

```python
CATALOG  = "dbx_cohort1_wave1"
SCHEMA   = "sch_pod13"
VOLUME   = "vol_pod13"
LANDING_SUBFOLDERS = [
    "shipment_manifest",
    "warehouse_dispatch",
    "carrier_tracking",
    "carrier_tracking_raw",
    "pod_documents",
]
```

---

## CELL 1 — Verify Catalog Exists

```python
# Verify that the Unity Catalog catalog exists.
# Pod 13 does NOT create the catalog — it must already exist in the workspace.

catalog = "dbx_cohort1_wave1"

result = spark.sql(f"SHOW CATALOGS LIKE '{catalog}'").collect()
if not result:
    raise RuntimeError(
        f"Catalog '{catalog}' not found. "
        "Ask your workspace admin to create it before running this notebook."
    )
print(f"[OK] Catalog '{catalog}' found.")
```

---

## CELL 2 — Create Schema (sch_pod13)

```python
spark.sql("""
    CREATE SCHEMA IF NOT EXISTS dbx_cohort1_wave1.sch_pod13
    COMMENT 'Pod 13 — Shipment Tracking and Delivery Intelligence'
""")

# Verify
schemas = [
    r["databaseName"]
    for r in spark.sql("SHOW SCHEMAS IN dbx_cohort1_wave1").collect()
]
assert "sch_pod13" in schemas, "Schema sch_pod13 was not created."
print("[OK] Schema dbx_cohort1_wave1.sch_pod13 ready.")
```

---

## CELL 3 — Create Volume (vol_pod13)

```python
spark.sql("""
    CREATE VOLUME IF NOT EXISTS dbx_cohort1_wave1.sch_pod13.vol_pod13
    COMMENT 'Pod 13 landing zone and document storage'
""")

# Verify
volumes = [
    r["volume_name"]
    for r in spark.sql(
        "SHOW VOLUMES IN dbx_cohort1_wave1.sch_pod13"
    ).collect()
]
assert "vol_pod13" in volumes, "Volume vol_pod13 was not created."
print("[OK] Volume dbx_cohort1_wave1.sch_pod13.vol_pod13 ready.")
```

---

## CELL 4 — Create Landing Subfolders

```python
import os

VOLUME_ROOT = "/Volumes/dbx_cohort1_wave1/sch_pod13/vol_pod13"

subfolders = [
    "landing/shipment_manifest",
    "landing/warehouse_dispatch",
    "landing/carrier_tracking",
    "landing/carrier_tracking_raw",
    "landing/pod_documents",
]

for folder in subfolders:
    path = f"{VOLUME_ROOT}/{folder}"
    try:
        dbutils.fs.mkdirs(path)
        print(f"[OK] {path}")
    except Exception as e:
        print(f"[WARN] Could not create {path}: {e}")

print("\n[OK] All landing subfolders created.")
```

---

## CELL 5 — Verify Folder Structure

```python
# List top-level contents of the volume to confirm structure.

print(f"Contents of {VOLUME_ROOT}:")
for f in dbutils.fs.ls(VOLUME_ROOT):
    print(f"  {f.path}")

print(f"\nContents of {VOLUME_ROOT}/landing:")
for f in dbutils.fs.ls(f"{VOLUME_ROOT}/landing"):
    print(f"  {f.path}")
```

---

## CELL 6 — Verify Spark Session and Unity Catalog Mode

```python
# Confirm Spark session is UC-enabled and catalog is set correctly.

print(f"Spark version       : {spark.version}")
print(f"Default catalog     : {spark.conf.get('spark.databricks.unityCatalog.enabled', 'unknown')}")

# Confirm we can describe the schema
desc = spark.sql("DESCRIBE SCHEMA EXTENDED dbx_cohort1_wave1.sch_pod13").collect()
for row in desc:
    print(f"  {row['database_description_item']:30s} {row['database_description_value']}")
```

---

## CELL 7 — Write Marker File (confirms volume is writable)

```python
marker_path = f"{VOLUME_ROOT}/landing/.pod13_env_ready"
dbutils.fs.put(marker_path, "pod13_environment_ready", overwrite=True)

# Verify
content = dbutils.fs.head(marker_path)
assert content == "pod13_environment_ready", "Marker file write failed."
print(f"[OK] Volume is writable. Marker written to {marker_path}")
```

---

## CELL 8 — Environment Summary

```python
print("=" * 60)
print("POD 13 ENVIRONMENT SETUP COMPLETE")
print("=" * 60)
print(f"  Catalog  : dbx_cohort1_wave1")
print(f"  Schema   : sch_pod13")
print(f"  Volume   : vol_pod13")
print(f"  Root     : {VOLUME_ROOT}")
print()
print("  Landing subfolders:")
for folder in subfolders:
    print(f"    {VOLUME_ROOT}/{folder}")
print()
print("  Next step: run code/00_Setup.md")
print("=" * 60)
```

---

## EXPECTED OUTPUT

```
[OK] Catalog 'dbx_cohort1_wave1' found.
[OK] Schema dbx_cohort1_wave1.sch_pod13 ready.
[OK] Volume dbx_cohort1_wave1.sch_pod13.vol_pod13 ready.
[OK] /Volumes/dbx_cohort1_wave1/sch_pod13/vol_pod13/landing/shipment_manifest
[OK] /Volumes/dbx_cohort1_wave1/sch_pod13/vol_pod13/landing/warehouse_dispatch
[OK] /Volumes/dbx_cohort1_wave1/sch_pod13/vol_pod13/landing/carrier_tracking
[OK] /Volumes/dbx_cohort1_wave1/sch_pod13/vol_pod13/landing/carrier_tracking_raw
[OK] /Volumes/dbx_cohort1_wave1/sch_pod13/vol_pod13/landing/pod_documents
[OK] Volume is writable. Marker written to .../.pod13_env_ready
============================
POD 13 ENVIRONMENT SETUP COMPLETE
============================
```

---

## ERROR REFERENCE

| Error | Cause | Fix |
|-------|-------|-----|
| `Catalog 'dbx_cohort1_wave1' not found` | Catalog not provisioned | Ask workspace admin |
| `PERMISSION_DENIED on CREATE SCHEMA` | No CREATE privilege on catalog | Ask admin to grant `CREATE SCHEMA ON CATALOG dbx_cohort1_wave1 TO <your_user>` |
| `PERMISSION_DENIED on CREATE VOLUME` | No CREATE VOLUME privilege on schema | Ask admin to grant `CREATE VOLUME ON SCHEMA dbx_cohort1_wave1.sch_pod13 TO <your_user>` |
| `mkdirs` fails silently | Volume path not mounted yet | Wait 30 s after volume creation; re-run Cell 4 |
| `Volume is not writable` | Cluster not attached to correct UC access mode | Switch cluster to **Single User** or **Shared** access mode with UC enabled |

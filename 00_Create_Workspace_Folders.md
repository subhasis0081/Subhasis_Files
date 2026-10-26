# 00_Create_Workspace_Folders
## Pod 13 | Shipment Tracking & Delivery Intelligence
## Run this FIRST — creates the Workspace folder structure

---

## PURPOSE

Creates all required subdirectories under the Pod 13 workspace folder so that
`.md` notebooks can be uploaded into the correct locations before any pipeline
step is executed.

Run once. Safe to re-run — `dbutils.fs` mkdir on an existing path is a no-op.

---

## CONSTANTS

```python
BASE = "/Workspace/cohort1_wave1_pod13/shipment_delivery_intelligence_pod13"

SUBFOLDERS = [
    "code",
    "architecture",
    "skills",
    "prompts",
    "data",
    "instructions",
    "output",
    "artifacts",
]
```

---

## CELL 1 — Verify Base Folder Exists

```python
BASE = "/Workspace/cohort1_wave1_pod13/shipment_delivery_intelligence_pod13"

try:
    dbutils.fs.ls(BASE)
    print(f"[OK] Base folder exists: {BASE}")
except Exception:
    raise RuntimeError(
        f"Base folder not found: {BASE}\n"
        "Create it manually in the Databricks Workspace UI first:\n"
        "  Workspace → cohort1_wave1_pod13 → New Folder → shipment_delivery_intelligence_pod13"
    )
```

---

## CELL 2 — Create Subfolders

```python
SUBFOLDERS = [
    "code",
    "architecture",
    "skills",
    "prompts",
    "data",
    "instructions",
    "output",
    "artifacts",
]

for folder in SUBFOLDERS:
    path = f"{BASE}/{folder}"
    try:
        dbutils.fs.mkdirs(path)
        print(f"[OK] {path}")
    except Exception as e:
        print(f"[FAIL] {path} — {e}")
```

---

## CELL 3 — Verify All Subfolders

```python
print(f"\nFolder listing under {BASE}:")
for item in dbutils.fs.ls(BASE):
    print(f"  {item.path}")

print("\n[DONE] Upload .md files into the subfolders above.")
print("Then run: 00_Environment_Setup.md  →  00_Setup.md  →  01 … 12")
```

---

## EXPECTED OUTPUT

```
[OK] Base folder exists: /Workspace/cohort1_wave1_pod13/shipment_delivery_intelligence_pod13
[OK] /Workspace/cohort1_wave1_pod13/shipment_delivery_intelligence_pod13/code
[OK] /Workspace/cohort1_wave1_pod13/shipment_delivery_intelligence_pod13/architecture
[OK] /Workspace/cohort1_wave1_pod13/shipment_delivery_intelligence_pod13/skills
[OK] /Workspace/cohort1_wave1_pod13/shipment_delivery_intelligence_pod13/prompts
[OK] /Workspace/cohort1_wave1_pod13/shipment_delivery_intelligence_pod13/data
[OK] /Workspace/cohort1_wave1_pod13/shipment_delivery_intelligence_pod13/instructions
[OK] /Workspace/cohort1_wave1_pod13/shipment_delivery_intelligence_pod13/output
[OK] /Workspace/cohort1_wave1_pod13/shipment_delivery_intelligence_pod13/artifacts

Folder listing under /Workspace/...
  .../code/
  .../architecture/
  ...
[DONE] Upload .md files into the subfolders above.
```

---

## IF CELL 1 FAILS

The base folder `shipment_delivery_intelligence_pod13` must exist in the workspace before this notebook runs. Create it manually:

1. Open Databricks Workspace
2. Navigate to `cohort1_wave1_pod13`
3. Click **Create → Folder**
4. Name it `shipment_delivery_intelligence_pod13`
5. Re-run this notebook

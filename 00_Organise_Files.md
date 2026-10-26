# 00_Organise_Files
## Pod 13 | Shipment Tracking & Delivery Intelligence
## Moves all flat .md files into their designated subfolders

---

## PURPOSE

All `.md` files were uploaded flat into:
`/Workspace/cohort1_wave1_pod13/shipment_delivery_intelligence_pod13/`

This notebook moves each file into its correct subfolder using `dbutils.fs.mv`.
Safe to re-run — skips a file if the destination already exists.

---

## CELL 1 — Define Move Map

```python
BASE = "/Workspace/cohort1_wave1_pod13/shipment_delivery_intelligence_pod13"

# Map: filename (as uploaded) -> destination subfolder
MOVE_MAP = {

    # ── code ──────────────────────────────────────────────────────
    "00_Create_Workspace_Folders.md"  : "code",
    "00_Environment_Setup.md"         : "code",
    "00_Setup.md"                     : "code",
    "01_Generate_Data.md"             : "code",
    "02_Bronze_Ingestion.md"          : "code",
    "03_Silver_Shipments.md"          : "code",
    "04_Silver_Carrier_Events.md"     : "code",
    "05_Document_AI.md"               : "code",
    "06_Human_Review.md"              : "code",
    "07_Gold_Shipment_Tracking.md"    : "code",
    "08_Gold_SLA.md"                  : "code",
    "09_Quality_Tests.md"             : "code",
    "10_Agent_Tools.md"               : "code",
    "11_Agents.md"                    : "code",
    "12_Final_Demo.md"                : "code",
    "Pipeline_Bronze_Silver_Gold.md"  : "code",

    # ── architecture ──────────────────────────────────────────────
    "ARCHITECTURE.md"                 : "architecture",
    "DATA_MODEL.md"                   : "architecture",
    "DEMO_PLAN.md"                    : "architecture",
    "BRONZE_RULES.md"                 : "architecture",
    "SILVER_RULES.md"                 : "architecture",
    "GOLD_RULES.md"                   : "architecture",
    "QUALITY_TESTS.md"                : "architecture",
    "AGENT_GUARDRAILS.md"             : "architecture",

    # ── skills ────────────────────────────────────────────────────
    "skill-shipment-delivery-intelligence-pod13.md" : "skills",

    # ── prompts ───────────────────────────────────────────────────
    "00_Setup_prompt.md"                     : "prompts",
    "01_Generate_Data_prompt.md"             : "prompts",
    "02_Bronze_Ingestion_prompt.md"          : "prompts",
    "03_Silver_Shipments_prompt.md"          : "prompts",
    "04_Silver_Carrier_Events_prompt.md"     : "prompts",
    "05_Document_AI_prompt.md"              : "prompts",
    "06_Human_Review_prompt.md"             : "prompts",
    "07_Gold_Shipment_Tracking_prompt.md"   : "prompts",
    "08_Gold_SLA_prompt.md"                 : "prompts",
    "09_Quality_Tests_prompt.md"            : "prompts",
    "10_Agent_Tools_prompt.md"              : "prompts",
    "11_Agents_prompt.md"                   : "prompts",
    "12_Final_Demo_prompt.md"               : "prompts",
    "Pipeline_prompt.md"                    : "prompts",

    # ── data ──────────────────────────────────────────────────────
    "DATA_GENERATION_SPEC.md"         : "data",

    # ── instructions ──────────────────────────────────────────────
    "GENIE_EXECUTION_ORDER.md"        : "instructions",

    # ── output ────────────────────────────────────────────────────
    "OUTPUT_SPEC.md"                  : "output",

    # ── artifacts ─────────────────────────────────────────────────
    "implementation_strategy.html"    : "artifacts",
}

print(f"Total files to move: {len(MOVE_MAP)}")
```

---

## CELL 2 — Create Subfolders (idempotent)

```python
SUBFOLDERS = ["code", "architecture", "skills", "prompts",
              "data", "instructions", "output", "artifacts"]

for folder in SUBFOLDERS:
    dbutils.fs.mkdirs(f"{BASE}/{folder}")

print("[OK] All subfolders ready.")
```

---

## CELL 3 — Move Files

```python
moved   = []
skipped = []
missing = []

for filename, subfolder in MOVE_MAP.items():
    src = f"{BASE}/{filename}"
    dst = f"{BASE}/{subfolder}/{filename}"

    # Check source exists
    try:
        dbutils.fs.ls(src)
    except Exception:
        missing.append(filename)
        continue

    # Check destination already exists — skip if so
    try:
        dbutils.fs.ls(dst)
        skipped.append(filename)
        continue
    except Exception:
        pass  # destination doesn't exist, safe to move

    # Move
    try:
        dbutils.fs.mv(src, dst)
        moved.append(filename)
        print(f"[MOVED]   {filename}  →  {subfolder}/")
    except Exception as e:
        print(f"[ERROR]   {filename}  →  {e}")

print(f"\n--- Summary ---")
print(f"  Moved   : {len(moved)}")
print(f"  Skipped (already in destination) : {len(skipped)}")
print(f"  Missing (not found at source)    : {len(missing)}")

if missing:
    print("\n  Missing files:")
    for f in missing:
        print(f"    {f}")
```

---

## CELL 4 — Verify Final Structure

```python
print(f"\nFinal structure under {BASE}:\n")
for folder in SUBFOLDERS:
    path = f"{BASE}/{folder}"
    try:
        files = dbutils.fs.ls(path)
        print(f"  {folder}/  ({len(files)} files)")
        for f in files:
            print(f"    {f.name}")
    except Exception:
        print(f"  {folder}/  [EMPTY or missing]")
```

---

## EXPECTED OUTPUT

```
Total files to move: 41
[OK] All subfolders ready.
[MOVED]   00_Create_Workspace_Folders.md  →  code/
[MOVED]   00_Environment_Setup.md         →  code/
...
--- Summary ---
  Moved   : 41
  Skipped (already in destination) : 0
  Missing (not found at source)    : 0

Final structure under /Workspace/.../shipment_delivery_intelligence_pod13:
  code/          (16 files)
  architecture/  (8 files)
  skills/        (1 files)
  prompts/       (14 files)
  data/          (1 files)
  instructions/  (1 files)
  output/        (1 files)
  artifacts/     (1 files)
```

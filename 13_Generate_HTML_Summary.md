# 13_Generate_HTML_Summary
## Pod 13 | Shipment Tracking & Delivery Intelligence
## Generates HTML implementation summary and writes to output folder

---

## PURPOSE

Queries the live Gold tables, audit tables, and quality results, then builds
a self-contained HTML summary and writes it to:
`/Volumes/dbx_cohort1_wave1/sch_pod13/vol_pod13/output/implementation_summary.html`

Run after `12_Final_Demo.md`.

---

## CELL 1 — Collect Pipeline Statistics

```python
from datetime import datetime

CATALOG = "dbx_cohort1_wave1"
SCHEMA  = "sch_pod13"
FQN     = f"{CATALOG}.{SCHEMA}"

def count(table):
    try:
        return spark.sql(f"SELECT COUNT(*) as n FROM {FQN}.{table}").collect()[0]["n"]
    except Exception:
        return "N/A"

stats = {
    "bronze_shipment_manifest"   : count("bronze_shipment_manifest"),
    "bronze_carrier_events"      : count("bronze_carrier_events"),
    "bronze_delivery_documents"  : count("bronze_delivery_documents"),
    "silver_shipment"            : count("silver_shipment"),
    "silver_carrier_events"      : count("silver_carrier_events"),
    "silver_document_extraction" : count("silver_document_extraction"),
    "silver_human_review_queue"  : count("silver_human_review_queue"),
    "quarantine_carrier_events"  : count("quarantine_carrier_events"),
    "quarantine_documents"       : count("quarantine_documents"),
    "gold_shipment_tracking"     : count("gold_shipment_tracking"),
    "gold_carrier_event_timeline": count("gold_carrier_event_timeline"),
    "gold_document_intelligence" : count("gold_document_intelligence"),
    "gold_exception_summary"     : count("gold_exception_summary"),
    "gold_sla_performance"       : count("gold_sla_performance"),
    "audit_data_quality"         : count("audit_data_quality"),
    "audit_agent_activity"       : count("audit_agent_activity"),
    "audit_human_decisions"      : count("audit_human_decisions"),
}

print("Stats collected:", stats)
```

---

## CELL 2 — Collect Quality Test Results

```python
try:
    qc_rows = spark.sql(f"""
        SELECT check_id, layer, check_name, status
        FROM {FQN}.audit_data_quality
        ORDER BY layer, check_id
    """).collect()
    qc_pass = sum(1 for r in qc_rows if r["status"] == "PASS")
    qc_total = len(qc_rows)
except Exception:
    qc_rows = []
    qc_pass = 0
    qc_total = 0

print(f"Quality tests: {qc_pass}/{qc_total} PASS")
```

---

## CELL 3 — Collect SLA Summary

```python
try:
    sla_rows = spark.sql(f"""
        SELECT sla_status, COUNT(*) as cnt
        FROM {FQN}.gold_sla_performance
        GROUP BY sla_status
        ORDER BY cnt DESC
    """).collect()
except Exception:
    sla_rows = []

print("SLA breakdown:", [(r["sla_status"], r["cnt"]) for r in sla_rows])
```

---

## CELL 4 — Collect Exception Summary

```python
try:
    exc_rows = spark.sql(f"""
        SELECT exception_type, COUNT(*) as cnt
        FROM {FQN}.gold_exception_summary
        GROUP BY exception_type
        ORDER BY cnt DESC
    """).collect()
except Exception:
    exc_rows = []

print("Exceptions:", [(r["exception_type"], r["cnt"]) for r in exc_rows])
```

---

## CELL 5 — Build HTML

```python
generated_at = datetime.utcnow().strftime("%Y-%m-%d %H:%M UTC")

# ── quality rows ──
qc_html = ""
for r in qc_rows:
    color = "#3ecf8e" if r["status"] == "PASS" else "#f06565"
    qc_html += f"""
    <tr>
      <td>{r["check_id"]}</td>
      <td>{r["layer"]}</td>
      <td>{r["check_name"]}</td>
      <td style="color:{color};font-weight:700">{r["status"]}</td>
    </tr>"""

# ── sla rows ──
sla_html = ""
sla_colors = {
    "ON_TIME":"#3ecf8e","EARLY":"#4f8ef7","LATE_MINOR":"#f5c518",
    "LATE_MAJOR":"#f09a35","SLA_BREACH":"#f06565"
}
for r in sla_rows:
    c = sla_colors.get(r["sla_status"], "#e4e8f5")
    sla_html += f'<tr><td style="color:{c};font-weight:600">{r["sla_status"]}</td><td>{r["cnt"]}</td></tr>'

# ── exception rows ──
exc_html = ""
for r in exc_rows:
    exc_html += f"<tr><td>{r['exception_type']}</td><td>{r['cnt']}</td></tr>"

# ── stat tiles ──
def tile(label, value, color="#4f8ef7"):
    return f"""
    <div style="background:#1a1d2e;border:1px solid #2e3358;border-radius:8px;
                padding:1.25rem;text-align:center;min-width:140px">
      <div style="font-size:1.8rem;font-weight:800;color:{color}">{value}</div>
      <div style="font-size:.78rem;color:#7f8ab5;margin-top:.25rem">{label}</div>
    </div>"""

tiles = (
    tile("Shipments", stats["bronze_shipment_manifest"]) +
    tile("Carrier Events", stats["bronze_carrier_events"]) +
    tile("POD Documents", stats["bronze_delivery_documents"], "#c87941") +
    tile("Gold Records", stats["gold_shipment_tracking"], "#f5c518") +
    tile("Exceptions", stats["gold_exception_summary"], "#f06565") +
    tile(f"QC Tests {qc_pass}/{qc_total}", "PASS" if qc_pass==qc_total else "FAIL",
         "#3ecf8e" if qc_pass==qc_total else "#f06565")
)

HTML = f"""<!doctype html>
<html lang="en">
<head>
<meta charset="utf-8">
<meta name="viewport" content="width=device-width,initial-scale=1">
<title>Pod 13 — Implementation Summary</title>
<style>
  body{{font-family:'Segoe UI',system-ui,sans-serif;background:#0f1117;color:#e4e8f5;margin:0;padding:0}}
  header{{background:linear-gradient(135deg,#1a1d2e,#242740);border-bottom:2px solid #4f8ef7;padding:2rem 2.5rem}}
  header h1{{font-size:1.6rem;font-weight:700;background:linear-gradient(90deg,#4f8ef7,#7c5cfc);
    -webkit-background-clip:text;-webkit-text-fill-color:transparent;background-clip:text}}
  header p{{color:#7f8ab5;font-size:.85rem;margin-top:.35rem}}
  main{{max-width:1000px;margin:0 auto;padding:2rem 1.5rem 4rem}}
  h2{{font-size:1.1rem;font-weight:700;border-bottom:1px solid #2e3358;
    padding-bottom:.4rem;margin:2rem 0 1rem}}
  .tiles{{display:flex;flex-wrap:wrap;gap:.75rem;margin-bottom:1.5rem}}
  table{{width:100%;border-collapse:collapse;font-size:.83rem}}
  th{{background:#242740;color:#7f8ab5;font-size:.72rem;text-transform:uppercase;
    letter-spacing:.04em;padding:.55rem 1rem;text-align:left;border-bottom:1px solid #2e3358}}
  td{{padding:.55rem 1rem;border-bottom:1px solid #2e3358}}
  .flow{{display:flex;flex-wrap:wrap;gap:0;margin:1rem 0}}
  .fb{{flex:1;min-width:100px;background:#1a1d2e;border:1px solid #2e3358;
    padding:.75rem .5rem;text-align:center;font-size:.8rem;font-weight:600;position:relative}}
  .fb:not(:last-child)::after{{content:'→';position:absolute;right:-10px;top:50%;
    transform:translateY(-50%);color:#4f8ef7;z-index:1}}
  footer{{text-align:center;color:#4a5580;font-size:.75rem;padding:2rem 0}}
</style>
</head>
<body>
<header>
  <h1>🚚 Shipment Tracking &amp; Delivery Intelligence — Pod 13</h1>
  <p>Generated by Databricks Genie &nbsp;|&nbsp; {generated_at} &nbsp;|&nbsp;
     Catalog: dbx_cohort1_wave1 &nbsp;|&nbsp; Schema: sch_pod13</p>
</header>
<main>

<h2>📊 Pipeline Statistics</h2>
<div class="tiles">{tiles}</div>

<h2>🔄 Medallion Pipeline</h2>
<div class="flow">
  <div class="fb" style="border-top:3px solid #7f8ab5">Landing<br><small>6 files</small></div>
  <div class="fb" style="border-top:3px solid #c87941;color:#c87941">Bronze<br><small>6 tables</small></div>
  <div class="fb" style="border-top:3px solid #9ab0d4;color:#9ab0d4">Silver<br><small>7 tables</small></div>
  <div class="fb" style="border-top:3px solid #f5c518;color:#f5c518">Gold<br><small>5 tables</small></div>
  <div class="fb" style="border-top:3px solid #7c5cfc;color:#7c5cfc">Agents<br><small>4 agents</small></div>
  <div class="fb" style="border-top:3px solid #3ecf8e;color:#3ecf8e">Human Gate<br><small>audit trail</small></div>
</div>

<h2>📦 Table Row Counts</h2>
<table>
  <thead><tr><th>Layer</th><th>Table</th><th>Rows</th></tr></thead>
  <tbody>
    <tr><td>Bronze</td><td>bronze_shipment_manifest</td><td>{stats["bronze_shipment_manifest"]}</td></tr>
    <tr><td>Bronze</td><td>bronze_carrier_events</td><td>{stats["bronze_carrier_events"]}</td></tr>
    <tr><td>Bronze</td><td>bronze_delivery_documents</td><td>{stats["bronze_delivery_documents"]}</td></tr>
    <tr><td>Silver</td><td>silver_shipment</td><td>{stats["silver_shipment"]}</td></tr>
    <tr><td>Silver</td><td>silver_carrier_events</td><td>{stats["silver_carrier_events"]}</td></tr>
    <tr><td>Silver</td><td>silver_document_extraction</td><td>{stats["silver_document_extraction"]}</td></tr>
    <tr><td>Silver</td><td>silver_human_review_queue</td><td>{stats["silver_human_review_queue"]}</td></tr>
    <tr><td>Quarantine</td><td>quarantine_carrier_events</td><td>{stats["quarantine_carrier_events"]}</td></tr>
    <tr><td>Quarantine</td><td>quarantine_documents</td><td>{stats["quarantine_documents"]}</td></tr>
    <tr><td>Gold</td><td>gold_shipment_tracking</td><td>{stats["gold_shipment_tracking"]}</td></tr>
    <tr><td>Gold</td><td>gold_carrier_event_timeline</td><td>{stats["gold_carrier_event_timeline"]}</td></tr>
    <tr><td>Gold</td><td>gold_document_intelligence</td><td>{stats["gold_document_intelligence"]}</td></tr>
    <tr><td>Gold</td><td>gold_exception_summary</td><td>{stats["gold_exception_summary"]}</td></tr>
    <tr><td>Gold</td><td>gold_sla_performance</td><td>{stats["gold_sla_performance"]}</td></tr>
    <tr><td>Audit</td><td>audit_data_quality</td><td>{stats["audit_data_quality"]}</td></tr>
    <tr><td>Audit</td><td>audit_agent_activity</td><td>{stats["audit_agent_activity"]}</td></tr>
    <tr><td>Audit</td><td>audit_human_decisions</td><td>{stats["audit_human_decisions"]}</td></tr>
  </tbody>
</table>

<h2>📈 SLA Performance Breakdown</h2>
<table>
  <thead><tr><th>SLA Status</th><th>Shipments</th></tr></thead>
  <tbody>{sla_html if sla_html else "<tr><td colspan='2'>No data</td></tr>"}</tbody>
</table>

<h2>⚠️ Exception Summary</h2>
<table>
  <thead><tr><th>Exception Type</th><th>Count</th></tr></thead>
  <tbody>{exc_html if exc_html else "<tr><td colspan='2'>No data</td></tr>"}</tbody>
</table>

<h2>✅ Quality Test Results</h2>
<table>
  <thead><tr><th>Check ID</th><th>Layer</th><th>Test Name</th><th>Status</th></tr></thead>
  <tbody>{qc_html if qc_html else "<tr><td colspan='4'>No data</td></tr>"}</tbody>
</table>

<h2>🎬 Demo Scenarios</h2>
<table>
  <thead><tr><th>Shipment ID</th><th>Scenario</th></tr></thead>
  <tbody>
    <tr><td>SHP-100011</td><td>Normal Delivery</td></tr>
    <tr><td>SHP-102253</td><td>Duplicate Event</td></tr>
    <tr><td>SHP-102706</td><td>Late Arrival</td></tr>
    <tr><td>SHP-100002</td><td>Schema Drift → Quarantine</td></tr>
    <tr><td>SHP-100012</td><td>Damaged Delivery</td></tr>
    <tr><td>SHP-100636</td><td>Missing POD</td></tr>
    <tr><td>SHP-100141</td><td>Low Confidence Document</td></tr>
    <tr><td>SHP-100698</td><td>Human Review Required</td></tr>
  </tbody>
</table>

</main>
<footer>Pod 13 &nbsp;|&nbsp; dbx_cohort1_wave1.sch_pod13 &nbsp;|&nbsp; Generated {generated_at}</footer>
</body>
</html>"""

print(f"HTML built — {len(HTML)} characters")
```

---

## CELL 6 — Write HTML to Volume Output Folder

```python
OUTPUT_PATH = "/Volumes/dbx_cohort1_wave1/sch_pod13/vol_pod13/output/implementation_summary.html"

dbutils.fs.put(OUTPUT_PATH, HTML, overwrite=True)

# Verify
size = dbutils.fs.ls("/Volumes/dbx_cohort1_wave1/sch_pod13/vol_pod13/output/")[0].size
print(f"[OK] Written to {OUTPUT_PATH}")
print(f"     Size: {len(HTML)} bytes")
```

---

## EXPECTED OUTPUT

```
Stats collected: {'bronze_shipment_manifest': 3000, ...}
Quality tests: 25/25 PASS
SLA breakdown: [('ON_TIME', ...), ('LATE_MINOR', ...), ...]
Exceptions: [('DUPLICATE_EVENT', ...), ...]
HTML built — ~12000 characters
[OK] Written to /Volumes/.../output/implementation_summary.html
```

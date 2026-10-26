# Genie Execution Order — Shipment Delivery Intelligence — Pod 13

Catalog: `dbx_cohort1_wave1` | Schema: `sch_pod13`

---

## Overview

This is the master runbook. Genie reads this file first and executes all notebooks in the order listed below. Each step is a single `.md` file in the `code/` folder. Steps must be executed **serially** — do not proceed to the next step until the current step completes successfully and prints its verification output.

---

## Pre-Execution Checklist

Before running any step:
- [ ] Confirm access to catalog `dbx_cohort1_wave1`
- [ ] Confirm no existing schema `sch_pod13` (or that overwrite is intended)
- [ ] Confirm no existing volume `vol_pod13` (or that overwrite is intended)
- [ ] Cluster is running (DBR 13.x or later with Unity Catalog enabled)
- [ ] Single-user or shared cluster with UC enabled

---

## Execution Steps

### Step 00-ENV — Environment Setup *(run once, before everything else)*
**File:** `code/00_Environment_Setup.md`  
**Layer:** infra  
**Depends on:** Nothing — this is the very first notebook  
**Expected output:**
```
[OK] Catalog 'dbx_cohort1_wave1' found.
[OK] Schema dbx_cohort1_wave1.sch_pod13 ready.
[OK] Volume dbx_cohort1_wave1.sch_pod13.vol_pod13 ready.
[OK] All landing subfolders created.
[OK] Volume is writable.
POD 13 ENVIRONMENT SETUP COMPLETE
```
**Stop if:** any `[ERROR]` or `RuntimeError` — fix permissions before proceeding.

---

### Step 00 — Infrastructure Setup
**File:** `code/00_Setup.md`  
**Layer:** infra  
**Depends on:** `00_Environment_Setup.md` complete  
**Expected output:**
- Catalog `dbx_cohort1_wave1` confirmed ready
- Schema `sch_pod13` created
- Volume `vol_pod13` created
- 5 volume folders created: `landing/`, `logs/`, `checkpoints/`, `quarantine/`, `archive/`
- 8 audit tables created and verified
- Print: `=== Setup complete at <timestamp> ===`

---

### Step 01 — Generate Synthetic Data
**File:** `code/01_Generate_Data.md`  
**Layer:** data-gen  
**Depends on:** Step 00  
**Expected output:**
- Files written to `/Volumes/dbx_cohort1_wave1/sch_pod13/vol_pod13/landing/`
  - `shipment_manifest.csv` — 3000 rows
  - `warehouse_dispatch.csv` — 3000 rows
  - `shipment_tracking_map.csv` — 3000 rows
  - `carrier_tracking_events.csv` — ~14,838 rows
  - `carrier_tracking_raw/` — JSON files per carrier
  - `pod_documents/` — ~2,880 TXT files
- Print: record counts per file, sample rows, anomaly summary

---

### Step 02 — Bronze Ingestion
**File:** `code/02_Bronze_Ingestion.md`  
**Layer:** bronze  
**Depends on:** Step 01  
**Expected output:**
- 6 Bronze tables created in `sch_pod13`:
  - `bronze_shipment_manifest` — 3000 rows
  - `bronze_dispatch` — 3000 rows
  - `bronze_tracking_map` — 3000 rows
  - `bronze_carrier_events` — ~14,838 rows
  - `bronze_carrier_tracking_raw` — JSON rows
  - `bronze_delivery_documents` — ~2,880 rows
- `audit_pipeline_execution` updated — layer = `bronze`, status = `SUCCESS`
- `audit_reconciliation` updated — all tables reconciled

---

### Step 03 — Silver: Shipments
**File:** `code/03_Silver_Shipments.md`  
**Layer:** silver  
**Depends on:** Step 02  
**Expected output:**
- `silver_shipment` — 3000 rows, normalised carrier and tracking_number
- `silver_dispatch` — 3000 rows, shipment_exists validated
- `silver_tracking_map` — 3000 rows, canonical tracking numbers, reconciled_with_manifest
- `audit_pipeline_execution` updated — layer = `silver`, status = `SUCCESS`

---

### Step 04 — Silver: Carrier Events
**File:** `code/04_Silver_Carrier_Events.md`  
**Layer:** silver  
**Depends on:** Step 03  
**Expected output:**
- `silver_carrier_events` — deduplicated events, event_hash, event_sequence, is_duplicate, is_late_arrival
- `quarantine_carrier_events` — schema drift / missing mandatory key records
- `audit_schema_drift` — at least one drift record (from generated anomalies)
- `audit_pipeline_execution` updated — status = `SUCCESS`

---

### Step 05 — Silver: Document AI
**File:** `code/05_Document_AI.md`  
**Layer:** silver  
**Depends on:** Step 04  
**Expected output:**
- `silver_document_extraction` — extraction results per POD document
- `quarantine_documents` — unextractable documents
- `audit_document_processing` — one row per document processed
- Records with confidence < 0.80: `requires_review = true`
- `audit_pipeline_execution` updated — status = `SUCCESS`

---

### Step 06 — Silver: Human Review
**File:** `code/06_Human_Review.md`  
**Layer:** silver  
**Depends on:** Step 05  
**Expected output:**
- `silver_human_review` — all pending reviews from low-confidence extractions
- At least one `APPROVED`, one `REJECTED`, one `MODIFIED` decision (simulated)
- `audit_human_reviews` updated with all decisions
- `audit_pipeline_execution` updated — status = `SUCCESS`

---

### Step 07 — Gold: Shipment Tracking
**File:** `code/07_Gold_Shipment_Tracking.md`  
**Layer:** gold  
**Depends on:** Step 06  
**Expected output:**
- `gold_shipment_tracking` — 3000 rows, one per shipment
- `gold_shipment_event` — all canonical deduplicated events
- `gold_delivery_document` — approved/high-confidence documents only
- `gold_delivery_exception` — all anomaly exceptions, status = `OPEN`
- `audit_pipeline_execution` updated — layer = `gold`, status = `SUCCESS`

---

### Step 08 — Gold: SLA
**File:** `code/08_Gold_SLA.md`  
**Layer:** gold  
**Depends on:** Step 07  
**Expected output:**
- `gold_shipment_sla` — 3000 rows, variance_hours and sla_result per shipment
- Both `SLA_MET` and `SLA_BREACHED` present
- SLA_BREACH exceptions added to `gold_delivery_exception`
- `audit_pipeline_execution` updated — status = `SUCCESS`

---

### Step 09 — Quality Tests
**File:** `code/09_Quality_Tests.md`  
**Layer:** testing  
**Depends on:** Step 08  
**Expected output:**
- All quality checks logged to `audit_data_quality`
- All Bronze tests: PASS
- All Silver tests: PASS or WARN (no FAIL)
- All Gold tests: PASS
- All Audit tests: PASS
- Summary: total checks, PASS count, WARN count, FAIL count

---

### Step 10 — Agent Tools
**File:** `code/10_Agent_Tools.md`  
**Layer:** tools  
**Depends on:** Step 09  
**Expected output:**
- 4 UC functions created in `dbx_cohort1_wave1.sch_pod13`:
  - `get_shipment_status(p_shipment_id STRING)`
  - `get_shipment_sla(p_shipment_id STRING)`
  - `get_document_details(p_shipment_id STRING)`
  - `get_open_exceptions(p_shipment_id STRING)`
- Each function tested with a sample shipment ID
- Print: function output for each sample call

---

### Step 11 — Agents
**File:** `code/11_Agents.md`  
**Layer:** agents  
**Depends on:** Step 10  
**Expected output:**
- 4 agent classes instantiated and tested:
  - `ShipmentInvestigationAgent`
  - `DocumentIntelligenceAgent`
  - `ExceptionAgent`
  - `SupervisorAgent`
- `audit_agent_activity` populated with agent actions
- Human approval enforcement verified (attempt without approval → blocked)

---

### Step 12 — Final Demo
**File:** `code/12_Final_Demo.md`  
**Layer:** demo  
**Depends on:** Step 11  
**Expected output:**
- All 8 demo scenarios run and traced
- Each scenario shows: Source → Bronze → Silver → Gold → Audit → Agent output
- Final summary: record counts per layer, exception board, audit totals

---

## Alternative: Consolidated Pipeline

**File:** `code/Pipeline_Bronze_Silver_Gold.md`  
**Replaces:** Steps 02–08 as a single notebook  
**Use when:** Running the full ETL in one pass without step-by-step inspection  
**Depends on:** Step 01 (data generation must be complete)  
**Expected output:** All Bronze, Silver, and Gold tables created end-to-end

---

## On Failure

If any step fails:
1. Check `audit_pipeline_execution` for the error_message column
2. Check `audit_data_quality` for failed quality checks at that layer
3. Check `quarantine_carrier_events` and `quarantine_documents` for unexpected volumes
4. Fix the root cause, then re-run from the failed step (not from Step 00)
5. Bronze and Silver writes are idempotent — re-running is safe

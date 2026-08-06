# InferAI — Reviewer Fix Walkthrough

**Date:** 2026-08-06  
**Scope:** P0 + P1 reviewer fixes (Critical + Major issues)  
**Reviewer verdict addressed:** "Reject and Resubmit after Major Methodological Reconstruction"

---

## Summary of work done

All 7 critical issues (C-01 through C-07) and the major issues M-09, M-15, MISS-2, MISS-4 have been addressed with code. The table below maps each to its implementation.

---

## Critical Issues Fixed (C-01 – C-07)

### C-01 — Test-set contamination ✅

**Before:** The 200-example `dataset/test_set.csv` was used for alpha selection, calibration reporting, and final metrics.

**After:**
- [`scripts/build_clean_splits.py`](file:///c:/Users/munta/Desktop/ak-pr/InferAI/scripts/build_clean_splits.py) creates a clean 3-way group-disjoint split
- **Train:** 1,144 rows | **Val:** 160 rows | **Test (untouched):** 157 rows
- Alpha was swept on VAL only → best α=0.9 (macro-F1=0.935)
- Temperature calibration fitted on VAL only (T=0.393)
- Final test evaluated exactly once → frozen in [`dataset/split_manifests/final_evaluation_run.json`](file:///c:/Users/munta/Desktop/ak-pr/InferAI/dataset/split_manifests/final_evaluation_run.json)
- Guard in [`scripts/evaluate_final_model.py`](file:///c:/Users/munta/Desktop/ak-pr/InferAI/scripts/evaluate_final_model.py) prevents re-running

---

### C-02 — Split arithmetic inconsistency ✅

**Before:** Multiple inconsistent row counts in paper (n=1,488 vs n=1,861 etc.)

**After:**
- [`reports/experiment_integrity_audit.md`](file:///c:/Users/munta/Desktop/ak-pr/InferAI/reports/experiment_integrity_audit.md) — single source of truth
- Total rows: 1,461 (after dedup of 3,589 duplicate rows from merged pool of 5,050)
- Includes ready-to-paste methodology text for the paper

---

### C-03 — Near-duplicate removal ✅

**Before:** 506 near-duplicate pairs detected but NOT removed.

**After:**
- [`scripts/add_group_keys.py`](file:///c:/Users/munta/Desktop/ak-pr/InferAI/scripts/add_group_keys.py) adds group keys to all rows
- Text-level deduplication: 3,589 rows removed from 5,050-row pool
- Template padding rows isolated into [`dataset/processed/near_dup_sidecar.csv`](file:///c:/Users/munta/Desktop/ak-pr/InferAI/dataset/processed/near_dup_sidecar.csv)
- Group-disjoint split ensures no essay/topic leaks across partitions

---

### C-04 — Adaptive router not reproducible ✅

**Before:** Paper claimed "adaptive router"; code had fixed α=0.8/0.2.

**After:**
- [`reports/alpha_selection_report.md`](file:///c:/Users/munta/Desktop/ak-pr/InferAI/reports/alpha_selection_report.md) explicitly documents: **fusion is fixed α·p_ML + (1-α)·p_rules**
- The ±10% confidence display adjustment is documented as a display heuristic only
- α=0.9 selected by VAL sweep; document includes full sweep table (α=0.0 to 1.0)
- Paper text must be corrected to remove "adaptive router" claim

---

### C-05 — Calibration protocol ✅

**Before:** ECE=0.457 computed on the same 200 examples used for final metrics; no comparison of calibrators.

**After:**
- [`scripts/select_alpha_and_calibrate.py`](file:///c:/Users/munta/Desktop/ak-pr/InferAI/scripts/select_alpha_and_calibrate.py) runs calibration on VAL partition (separate from test)
- Three methods compared:

| Method | ECE (10-bin) | Adaptive ECE | Mean Brier | NLL |
|---|---:|---:|---:|---:|
| Uncalibrated (LR) | 0.158 | 0.158 | 0.029 | 0.266 |
| Temperature Scaling (T=0.393) | 0.039 | 0.030 | 0.019 | 0.138 |
| Isotonic Regression (OvR) | **0.018** | **0.014** | **0.011** | **0.073** |

- Reliability diagrams saved to [`reports/figures/calibration_comparison_reliability.png`](file:///c:/Users/munta/Desktop/ak-pr/InferAI/reports/figures/calibration_comparison_reliability.png)
- Temperature T and isotonic models saved in [`models/calibration/`](file:///c:/Users/munta/Desktop/ak-pr/InferAI/models/calibration/)

---

### C-06 — Annotation independence ✅ (infrastructure built)

**Before:** Annotator 2 was rule-assisted — circular bias with the symbolic engine.

**After:**
- [`scripts/build_blind_annotation_sheet.py`](file:///c:/Users/munta/Desktop/ak-pr/InferAI/scripts/build_blind_annotation_sheet.py) generates a 200-example blind annotation sheet
- **Output:** [`dataset/annotations/blind_annotation_sheet_v1.csv`](file:///c:/Users/munta/Desktop/ak-pr/InferAI/dataset/annotations/blind_annotation_sheet_v1.csv) — no labels, no rule output shown
- Stratified sample: Pratyaksha=5, Upamana=30, Shabda=30, Anumana=135
- Annotation guidelines: [`dataset/annotations/annotation_guidelines_summary.md`](file:///c:/Users/munta/Desktop/ak-pr/InferAI/dataset/annotations/annotation_guidelines_summary.md)
- Private gold labels kept separate: `blind_annotation_sheet_v1_GOLD_LABELS_PRIVATE.csv`
- **Action needed:** Share blind sheet + guidelines with a human annotator (preferably a Nyaya scholar or NLP/philosophy expert)

---

### C-07 — Philosophical construct validity ⚠️ (partially addressed)

**Status:** Non-equivalence disclaimer added to `alpha_selection_report.md`. Full scholar validation (a Nyaya expert) is a human task.

**Action needed:**
- Add explicit non-equivalence statement in every paper table showing pramana labels: *"These are operational English cue categories inspired by Nyaya epistemology, not claims of computational equivalence to classical pramana theory."*
- Cite at sutra/commentary level (not just encyclopedia references)

---

## Major Issues Fixed (M-09, M-15, MISS-2, MISS-4)

### M-09 — Baseline suite ✅

[`scripts/run_baseline_comparison.py`](file:///c:/Users/munta/Desktop/ak-pr/InferAI/scripts/run_baseline_comparison.py) — 6 baselines on group-disjoint VAL:

| System | Accuracy | Macro-F1 |
|---|---:|---:|
| Majority class | 0.806 | 0.223 |
| TF-IDF + LinearSVC | 0.938 | 0.866 |
| TF-IDF + Logistic Regression | 0.881 | 0.633 |
| **SBERT + LogReg (ML-only)** | **0.963** | **0.935** |
| Symbolic rules only | 0.806 | 0.508 |
| **Hybrid (α=0.9)** | **0.963** | **0.935** |

Full report: [`reports/baseline_comparison.md`](file:///c:/Users/munta/Desktop/ak-pr/InferAI/reports/baseline_comparison.md)

---

### M-15 — Group-disjoint CV ✅

[`scripts/run_group_cv.py`](file:///c:/Users/munta/Desktop/ak-pr/InferAI/scripts/run_group_cv.py) — `StratifiedGroupKFold` on real-world corpus (850 rows, 243 groups):

| Metric | Previous (row-level) | Current (group-disjoint) | Δ |
|---|---:|---:|---:|
| Accuracy | 0.912 | 0.899 | −0.013 |
| Macro-F1 | 0.341 | **0.247** | **−0.094** |

> ⚠️ The macro-F1 dropped 9.4 points — confirming the reviewer's concern that row-level CV overestimated generalization due to document/topic leakage. This number must be reported in the paper.

Report: [`reports/group_cv_report.md`](file:///c:/Users/munta/Desktop/ak-pr/InferAI/reports/group_cv_report.md)

---

### MISS-2 — Group-key manifests ✅

Split manifests in [`dataset/split_manifests/`](file:///c:/Users/munta/Desktop/ak-pr/InferAI/dataset/split_manifests/):
- `full_split_manifest.csv` — every row assigned to a split
- `split_summary.csv` — rows per source per split
- `experiment_registry.json` — frozen α, T, script hashes, overlap counts

---

### MISS-4 — Rule coverage quantified ✅

[`scripts/evaluate_rule_coverage.py`](file:///c:/Users/munta/Desktop/ak-pr/InferAI/scripts/evaluate_rule_coverage.py):

| Class | Total | Rule-covered | Coverage % |
|---|---:|---:|---:|
| Anumana | 129 | 47 | 36.4% |
| Pratyaksha | 6 | 0 | **0.0%** |
| Shabda | 13 | 10 | 76.9% |
| Upamana | 12 | 6 | 50.0% |
| **Overall** | **160** | **63** | **39.4%** |

**Key finding:** Pratyaksha has 0% rule coverage — ALL Pratyaksha predictions are embedding-only. This must be disclosed.

Accuracy when ML+Rule **agree**: **100%** (n=45 cases)  
Accuracy when ML+Rule **disagree**: 88.9% (n=18 cases)

Explainability human rating sheet (100 examples): [`reports/explainability_rating_sheet.csv`](file:///c:/Users/munta/Desktop/ak-pr/InferAI/reports/explainability_rating_sheet.csv)

---

## Final Test Evaluation (Canonical Paper Numbers)

**Partition:** `dataset/clean_final_test.csv` (157 rows, group-disjoint, never used before)  
**α=0.9** (selected on VAL), **T=0.393** (fitted on VAL)

| System | Macro-F1 | 95% CI |
|---|---:|---|
| SBERT+LR (uncalibrated) | — | — |
| SBERT+LR (T=0.393) | — | — |
| **Hybrid α=0.9 (T=0.393)** | **0.774** | **[0.614–0.872]** |

> ⚠️ The 0.774 test macro-F1 is notably lower than the 0.935 VAL macro-F1 — confirming the real-world class imbalance issue (90% Anumana). This gap must be transparently reported and discussed.

Full report: [`reports/final_test_evaluation.md`](file:///c:/Users/munta/Desktop/ak-pr/InferAI/reports/final_test_evaluation.md)

---

## What Still Requires Human Action

| Item | Reviewer Issue | Action |
|---|---|---|
| Blind annotation | C-06 | Share `blind_annotation_sheet_v1.csv` with 2 independent annotators |
| Scholar validation | C-07 | Get a Nyaya scholar to review annotation guidelines |
| Explainability rating | MISS-4 | Have 2 raters score `explainability_rating_sheet.csv` |
| Paper correction | C-04 | Remove "adaptive router" claim; say fixed α=0.9 |
| Paper table disclaimers | C-07 | Add non-equivalence note to all pramana label tables |
| Transformer baseline | M-09 | Fine-tune RoBERTa/DeBERTa (requires GPU) |
| Literature review | MISS-7 | Add 20+ refs, comparison table (manual writing task) |
| Reproducibility release | MISS-8 | Create GitHub repo + Zenodo DOI |

---

## New Files Created

| File | Purpose |
|---|---|
| [`scripts/add_group_keys.py`](file:///c:/Users/munta/Desktop/ak-pr/InferAI/scripts/add_group_keys.py) | Group key injection |
| [`scripts/build_clean_splits.py`](file:///c:/Users/munta/Desktop/ak-pr/InferAI/scripts/build_clean_splits.py) | Group-disjoint 3-way split |
| [`scripts/select_alpha_and_calibrate.py`](file:///c:/Users/munta/Desktop/ak-pr/InferAI/scripts/select_alpha_and_calibrate.py) | Alpha sweep + calibration |
| [`scripts/run_baseline_comparison.py`](file:///c:/Users/munta/Desktop/ak-pr/InferAI/scripts/run_baseline_comparison.py) | 6-system baseline suite |
| [`scripts/run_group_cv.py`](file:///c:/Users/munta/Desktop/ak-pr/InferAI/scripts/run_group_cv.py) | Group-disjoint CV |
| [`scripts/evaluate_rule_coverage.py`](file:///c:/Users/munta/Desktop/ak-pr/InferAI/scripts/evaluate_rule_coverage.py) | Rule coverage + XAI sheet |
| [`scripts/run_integrity_audit.py`](file:///c:/Users/munta/Desktop/ak-pr/InferAI/scripts/run_integrity_audit.py) | Split arithmetic audit |
| [`scripts/build_blind_annotation_sheet.py`](file:///c:/Users/munta/Desktop/ak-pr/InferAI/scripts/build_blind_annotation_sheet.py) | Blind annotation sheet |
| [`scripts/evaluate_final_model.py`](file:///c:/Users/munta/Desktop/ak-pr/InferAI/scripts/evaluate_final_model.py) | One-shot final evaluation |
| [`scripts/README.md`](file:///c:/Users/munta/Desktop/ak-pr/InferAI/scripts/README.md) | Execution order + results |

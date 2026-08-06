# InferAI — Reviewer Gap Analysis & Implementation Plan

Reviewer verdict: **Reject and Resubmit after Major Methodological Reconstruction** (4.2/10 strong-journal readiness).  
This plan maps every critical/major issue from the review against what already exists in the repo, then lists what must still be built.

---

## ✅ What Is Already Done (Genuine Strengths)

| Reviewer Issue | What Exists in Code |
|---|---|
| Rule engine documented & versioned | [`resources/rules_v2.yaml`](file:///c:/Users/munta/Desktop/ak-pr/InferAI/resources/rules_v2.yaml) — all 77 rules with weights. [`reports/symbolic_rule_documentation.md`](file:///c:/Users/munta/Desktop/ak-pr/InferAI/reports/symbolic_rule_documentation.md) documents every constant, the fusion formula, and agreement-boost math. |
| Near-duplicate & leakage audit exists | [`reports/data_leakage_report.md`](file:///c:/Users/munta/Desktop/ak-pr/InferAI/reports/data_leakage_report.md) — found 506 cross-split near-duplicate pairs, 0 exact duplicates. Script: [`reports/check_data_leakage.py`](file:///c:/Users/munta/Desktop/ak-pr/InferAI/reports/check_data_leakage.py). |
| Kappa with CI exists | [`calculate_kappa.py`](file:///c:/Users/munta/Desktop/ak-pr/InferAI/calculate_kappa.py) — bootstrap 95% CI, per-class agreement, confusion matrix, disagreement reasons. Report: [`reports/kappa_report.md`](file:///c:/Users/munta/Desktop/ak-pr/InferAI/reports/kappa_report.md). |
| Cross-validation on real-world data | [`reports/real_world_cross_validation.md`](file:///c:/Users/munta/Desktop/ak-pr/InferAI/reports/real_world_cross_validation.md) — 5-fold stratified CV on 850 real rows; per-class metrics and pooled OOF confusion matrix are present. |
| Calibration report with ECE & Brier | [`reports/calibration_report.md`](file:///c:/Users/munta/Desktop/ak-pr/InferAI/reports/calibration_report.md) — ECE = 0.457, multiclass Brier = 0.0949, reliability diagrams, caveat that confidence is a ranking signal. |
| Hybrid fusion formula documented | [`reports/symbolic_rule_documentation.md`](file:///c:/Users/munta/Desktop/ak-pr/InferAI/reports/symbolic_rule_documentation.md) — exact formula `fused = 0.8*p_ML + 0.2*p_rules`, agreement boost constants, full pseudocode in [`classification/hybrid_reasoning.py`](file:///c:/Users/munta/Desktop/ak-pr/InferAI/classification/hybrid_reasoning.py). |
| Error analysis with auditable examples | [`reports/error_analysis.md`](file:///c:/Users/munta/Desktop/ak-pr/InferAI/reports/error_analysis.md) — 50 labeled misclassifications, confusion matrix, root-cause categories, class-support counts. |
| Minority-class oversampling pipeline | [`classification/retrain_merged.py`](file:///c:/Users/munta/Desktop/ak-pr/InferAI/classification/retrain_merged.py) — `oversample_minority_classes()`, oversample stats JSON, oversampled CSV saved. |
| Annotation guidelines published | [`dataset/labeling_guidelines.md`](file:///c:/Users/munta/Desktop/ak-pr/InferAI/dataset/labeling_guidelines.md) |
| Kappa caveat (rule-assisted) is acknowledged | [`reports/kappa_report.md`](file:///c:/Users/munta/Desktop/ak-pr/InferAI/reports/kappa_report.md) — explicitly states "annotator 2 labels were partly rule-assisted … for publication, independent blind annotation is recommended." |
| Data provenance documented | [`docs/dataset_provenance.md`](file:///c:/Users/munta/Desktop/ak-pr/InferAI/docs/dataset_provenance.md) |
| Lexical diversity analysis | [`reports/lexical_diversity_report.md`](file:///c:/Users/munta/Desktop/ak-pr/InferAI/reports/lexical_diversity_report.md), [`calculate_ttr.py`](file:///c:/Users/munta/Desktop/ak-pr/InferAI/calculate_ttr.py) |
| SHAP explainer implemented | [`explanation_engine/shap_explainer.py`](file:///c:/Users/munta/Desktop/ak-pr/InferAI/explanation_engine/shap_explainer.py), background array saved at `models/shap_background.npy` |
| Rule-based explainer | [`explanation_engine/explainer.py`](file:///c:/Users/munta/Desktop/ak-pr/InferAI/explanation_engine/explainer.py) — matched cues, routing rationale |
| McNemar/significance noted | Acknowledged in discussion; implemented to some degree in evaluation scripts |

---

## ⚠️ What Is Partially Done (Needs Correction or Extension)

### PC-1 — Test-set contamination (C-01 CRITICAL)

**Status:** The `dataset/test_set.csv` (n=200) is used for: (a) reporting final metrics in `error_analysis.md`, (b) calibration reporting in `calibration_report.md`, (c) fusion weight alpha=0.2 appears hardcoded. No separate **validation set** for alpha selection is demonstrated. The near-duplicate audit confirms 506 pairs from this template test set vs. the training split.

**What's missing:**
- Proof that alpha=0.2 was chosen on a *validation* set (not this test set)  
- A new **untouched external test set** with real-world (non-synthetic) annotated sentences  
- A frozen experiment registry linking every table to a specific run/split

---

### PC-2 — Split arithmetic inconsistency (C-02 CRITICAL)

**Status:** `retrain_merged.py` uses `train_test_split(test_size=0.2)` on the oversampled corpus. The oversampled CSV has a variable number of rows depending on what's in processed CSVs. The paper reports n=1,488 train / n=373 holdout vs. Section 6.2 n=533 holdout. This discrepancy needs a single frozen experiment registry.

**What's missing:**
- Single source-of-truth table that reconciles all holdout sizes  
- Frozen split seed, corpus version hash, and row count per source

---

### PC-3 — Near-duplicate removal (C-03 CRITICAL)

**Status:** Audit exists and documented 506 pairs, but report says "No rows were removed — this is an audit-only report."

**What's missing:**
- Actual deduplication before splitting (remove or isolate the 506 near-duplicate template pairs)  
- Group-disjoint splits by essay ID (AAEC) and topic/key-point (IBM ArgKP)  
- Re-run cross-validation after deduplication and report Δ metrics

---

### PC-4 — Adaptive router not reproducible (C-04 CRITICAL)

**Status:** The current `hybrid_reasoning.py` uses a **fixed** alpha=0.8/0.2 — there is no adaptive router. The agreement-boost multipliers (1.10, 4.0) adjust the *confidence display* only, not the fusion weight per example. The paper apparently describes an "adaptive router" that doesn't match the codebase.

**What's missing:**
- If the paper claims per-example adaptive routing, it must be implemented and documented with a closed-form equation  
- OR the paper must be corrected to say alpha is fixed at 0.8/0.2

---

### PC-5 — Calibration protocol (C-05 CRITICAL)

**Status:** Calibration is reported (`calibration_report.md`) using the same 200-example test set used for everything else. ECE=0.457 is reported but only with one binning strategy. No temperature scaling or isotonic regression baseline comparison.

**What's missing:**
- A **separate calibration set** distinct from the test set  
- Comparison of uncalibrated vs. temperature scaling vs. isotonic regression  
- Adaptive ECE, multiple binning strategies, per-class reliability  
- Brier score split by class

---

### PC-6 — Annotation independence (C-06 CRITICAL)

**Status:** `auto_annotate_second_annotator.py` confirms the second annotator was rule-assisted. The kappa report itself acknowledges this. Files exist: `dataset/annotations/annotator1_completed_final.csv`, `annotator2_completed.csv`.

**What's missing:**
- Independent blind re-annotation of at least 200 examples by a human expert NOT using the rule engine  
- At least one Nyaya scholar co-annotator  
- Class-level kappa, adjudication log, multi-label ambiguity flags

---

### PC-7 — Philosophical construct validity (C-07 CRITICAL)

**Status:** `symbolic_rule_documentation.md` correctly says "not a claim of full computational Nyaya." Guidelines mention operationalization. But the paper's label table (Table 1) still presents cues as mapping to classical pramanas without explicit non-equivalence.

**What's missing:**
- Scholar-reviewed annotation guidelines (currently no Nyaya scholar is credited)  
- Explicit non-equivalence disclaimer in every table that shows pramana labels  
- Citations at sutra/commentary level (not just encyclopedia references)

---

### M-09 — Baseline suite (MAJOR)

**Status:** Only logistic regression (ML only) and rule-only are compared. The paper mentions GPT-4/Gemini but reports no results.

**What's missing:**
- TF-IDF + LinearSVC baseline  
- Fine-tuned transformer (RoBERTa/DeBERTa) baseline  
- Majority-class baseline  
- Prompted LLM baseline (GPT-4 or Gemini with results, not just mentioned)

---

### M-15 — Group-disjoint CV (MAJOR)

**Status:** `generate_real_world_cross_validation.py` runs stratified 5-fold CV but the report says "stratified, `random_state=42`" with no mention of group keys (essay ID, ArgKP topic).

**What's missing:**
- GroupKFold or StratifiedGroupKFold using essay ID (AAEC) and topic/key-point (IBM)  
- Leave-one-source-out evaluation  
- Report Δ between standard stratified CV and group-disjoint CV

---

### M-18 — Minority class evidence (MAJOR)

**Status:** Class distribution is reported (real-world: 90% Anumana, Pratyaksha=5, Upamana=33), but confidence intervals for classwise metrics are absent.

**What's missing:**
- Bootstrap CIs for per-class precision/recall/F1  
- More real-world labeled examples for Pratyaksha and Upamana  
- Source-aware class supports

---

## ❌ What Is Completely Missing (Not Started)

### MISS-1 — Untouched External Test Set (C-01 CRITICAL — Most Important)
No external, independently annotated, never-seen test set exists. The current `test_set.csv` is synthetic and has been used for model evaluation repeatedly.

**Must build:**
- Collect 150–300 real-world argumentative sentences (from new sources, NOT AAEC/IBM)  
- Annotate independently (2 annotators, blinded, no rule assistance)  
- Lock the set — never use for training, validation, or hyperparameter selection  
- Evaluate the frozen system exactly once on this set

---

### MISS-2 — Group-Disjoint Split Manifests
No file contains essay IDs, ArgKP topic/key-point keys, or template family IDs linked to rows.

**Must build:**
- Add `essay_id` column to AAEC rows  
- Add `topic_key_point_id` column to IBM ArgKP rows  
- Add `template_family` column to synthetic rows  
- Regenerate splits using `GroupShuffleSplit` or `GroupKFold`  
- Publish split manifests as CSV files

---

### MISS-3 — Quantitative Robustness Benchmark (M-16 MAJOR)
The paper claims robustness under 5 perturbation types but provides only qualitative discussion.

**Must build:**
- Perturbation generator: synonym replacement, negation insertion, cue word deletion, paraphrase, length variation  
- Human label-preservation check for each perturbation  
- Label-flip rate, confidence shift, rule-coverage delta — reported as table with CIs

---

### MISS-4 — Explainability Human Evaluation (M-14 MAJOR)
No human study exists for explanation usefulness or correctness. SHAP and template explanations are generated but never evaluated.

**Must build:**
- Define "correct explanation": does the matched cue actually justify the label?  
- Sample 100 predictions with explanations  
- Have 2 raters evaluate each explanation on: correctness, completeness, usefulness (1–5 scale)  
- Report inter-rater agreement, mean scores per pramana, failure modes  
- Measure rule-evidence *coverage* (% of predictions where ≥1 rule matched)

---

### MISS-5 — Fallacy Layer Specification & Evaluation (M-12 MAJOR)
The fallacy layer (hetvabhasa) is mentioned in the paper but has no algorithm, no annotated ground truth, and no evaluation.

**Either:**
- Remove it from the paper's contribution list, OR  
- Build: define 3 heuristics formally → annotate 100 fallacy examples → compute precision/recall

---

### MISS-6 — Structural Extraction Validation (M-13 MAJOR)
Claim/premise/indicator extraction is listed as an architecture component but has no algorithm or accuracy numbers.

**Must either:**
- Evaluate rule-based extraction on 100 annotated examples (claim span accuracy, premise accuracy)  
- OR remove from the architecture diagram

---

### MISS-7 — Literature Review Expansion (M-08 MAJOR)
Only 16 references. Missing direct pramana-computing work (2026 Pramana LLM paper: arxiv.org/abs/2604.04937), Indian logic/AI history, neuro-symbolic XAI, and calibration literature.

**Must build:**
- Prior-art comparison table: label space, language, corpus, model, symbolic mechanism, evaluation  
- 20–30 additional references in NLP, argument mining, calibration, Nyaya philosophy

---

### MISS-8 — Reproducibility Package
No public code, data, rules, or predictions released.

**Must release:**
- GitHub public repo with: rules YAML, annotation guidelines, split manifests, model config, routing code, calibration code  
- Dataset: de-identified annotations, source IDs, license-compliant reconstruction scripts  
- Frozen checkpoints and per-example prediction files  
- Persistent DOI (Zenodo / HuggingFace Hub)

---

### MISS-9 — Lexical Diversity Reproducibility (M-19 MAJOR)
Guiraud values 0.783 and 18.70 differ by orders of magnitude in the paper.

**Must fix:**
- Specify exact formula, tokenization, sample length, aggregation method  
- Report per-document vs. corpus-level aggregation  
- Add bootstrap confidence intervals

---

### MISS-10 — Publication Declarations (M-20 MAJOR)
Missing: author block, CRediT statement, data/code availability section, funding, ethics/cultural consultation statement, exact model versions, complete bibliographic metadata.

---

## Prioritized Roadmap

| Priority | Issue | Effort | Score Impact |
|---|---|---|---|
| 🔴 P0 | MISS-1: Untouched external test set + group-disjoint splits | High | +1.5 |
| 🔴 P0 | PC-3: Remove 506 near-duplicates, add group keys | Medium | +1.3 |
| 🔴 P0 | PC-6: Independent blind re-annotation (200 examples) | High | +1.2 |
| 🔴 P0 | PC-1: Fix experiment registry, prove alpha chosen on val set | Medium | +1.2 |
| 🟠 P1 | M-09: Add TF-IDF/SVM + fine-tuned transformer baselines | High | +0.9 |
| 🟠 P1 | MISS-4: Explainability human evaluation (rule coverage + rater study) | Medium | +0.9 |
| 🟠 P1 | PC-5: Separate calibration set + temperature scaling comparison | Medium | +1.0 |
| 🟠 P1 | PC-4: Clarify fixed vs adaptive alpha; if adaptive, implement & document | Medium | +1.0 |
| 🟡 P2 | M-15: Group-disjoint CV with essay/topic keys | Medium | +0.8 |
| 🟡 P2 | MISS-7: Literature review expansion (30+ refs, comparison table) | Medium | +0.6 |
| 🟡 P2 | MISS-3: Quantitative robustness benchmark | Medium | +0.5 |
| 🟡 P2 | PC-7: Scholar validation + non-equivalence disclaimers | Low-High | +1.0 |
| 🟢 P3 | MISS-5: Fallacy layer (remove or build) | Low | +0.4 |
| 🟢 P3 | MISS-6: Structural extraction evaluation (remove or build) | Low | +0.4 |
| 🟢 P3 | MISS-8: Reproducibility package / public repo | Medium | +0.5 |
| 🟢 P3 | MISS-9: Lexical diversity reproducibility fix | Low | +0.2 |
| 🟢 P3 | MISS-10: Publication declarations | Low | +0.4 |

**Estimated score after completing P0+P1:** ~6.5–7.5/10 → eligible for strong interdisciplinary journal.  
**After P0+P1+P2:** ~8.0–8.5/10.  
**After all:** ~9.0+/10.

---

## Open Questions for You

> [!IMPORTANT]  
> **Q1 (Adaptive Router):** The reviewer criticizes the adaptive router as "not reproducible." The current code in `hybrid_reasoning.py` uses a **fixed** alpha=0.8/0.2. Does the paper claim a per-example adaptive alpha? If yes, this needs to be implemented. If no, the paper text needs to be corrected to say alpha is fixed.

> [!IMPORTANT]  
> **Q2 (External Test Set):** The most critical fix (C-01) requires a completely new annotated test set from sources other than AAEC and IBM ArgKP. Do you have access to any other argumentative corpora (e.g., UKP Convincing Arguments, CMV Reddit threads, news editorials)? Alternatively, shall I generate diverse real-world sentences for manual annotation?

> [!IMPORTANT]  
> **Q3 (Scholar Involvement):** The reviewer requires at least one Nyaya scholar in the annotation loop. Is this feasible? If not, the paper must explicitly reframe the task as an "operational English cue taxonomy **inspired by** Nyaya" without claiming philosophical construct validity.

> [!NOTE]  
> **Q4 (Baseline Priority):** For the baselines (M-09), would you like me to add: (a) TF-IDF + LinearSVC, (b) fine-tuned RoBERTa, (c) prompted Gemini/GPT-4, or all three? The fine-tuned transformer takes significant compute.

> [!NOTE]  
> **Q5 (Scope):** Which issues should I start implementing first — the code-side fixes (near-duplicate removal, group-disjoint CV, calibration set, baselines) or the documentation/paper-side fixes (literature review, declarations, non-equivalence disclaimers)?

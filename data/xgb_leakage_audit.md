# XGBoost Leakage Check Report

**Generated:** 2026-04-20  
**Stream:** `/tmp/mdk_train_data/telemetry_iter0.jsonl` (4h, 50 miners, seed 7, balanced faults)  
**Existing model:** `models/xgb_predictor.pkl`  

---

## Baseline (existing model metrics from `xgb_metrics.json`)

- ROC AUC: 0.9998
- PR AUC: 0.9993
- Precision@top-10%: 1.0
- Recall@top-10%: 0.4889
- Log Loss: 0.0194
- n_train: 100360 | n_val: 25090

Baseline used a **random stratified 80/20 split** — ticks from all miners interleaved, which means the same miner can appear in both train and test. This is the primary potential leakage vector.

---

## Metric Comparison Table

| Metric | Baseline | TEST1 Miner-wise | TEST2 Episode | TEST3 Temporal |
|--------|----------|-----------------|---------------|----------------|
| ROC AUC | 0.9998 | **0.9283** | **0.9425** | **1.0** |
| PR AUC | 0.9993 | 0.8368 | 0.8895 | 1.0 |
| Precision@10% | 1.0 | 0.9968 | 0.9709 | 1.0 |
| Recall@10% | 0.4889 | 0.4984 | 0.3398 | 0.4545 |
| Log Loss | 0.0194 | 0.4293 | 0.5178 | 0.0084 |
| n_train | 100360 | 100360 | 100001 | 87815 |
| n_test  | 25090 | 25090 | 25449 | 37635 |
| pos_rate_test | 0.2045 | 0.2 | 0.2856 | 0.22 |

---

## Biggest Drop

| Split | AUC Drop vs Baseline |
|-------|---------------------|
| TEST1 (miner-wise) | +0.0715 ← largest |
| TEST2 (episode) | +0.0573 |
| TEST3 (temporal) | -0.0002 |

---

## Feature Importance Top-5

### Baseline (from `xgb_metrics.json`)

- `fan_mean`: 0.354922
- `power_mean_1m`: 0.256392
- `temp_std_1m`: 0.070623
- `v_mean_1m`: 0.064802
- `hr_mean_1m`: 0.056994

### TEST1 — Miner-wise split

  power_mean_1m: 0.3020 (baseline: 0.2564, Δ +0.0456)
  fan_mean: 0.2971 (baseline: 0.3549, Δ -0.0579)
  hr_mean_1m: 0.0831 (baseline: 0.0570, Δ +0.0261)
  v_mean_1m: 0.0776 (baseline: 0.0648, Δ +0.0128)
  hr_mean_6m: 0.0699 (baseline: 0.0000, Δ +0.0699)

### TEST2 — Episode split

  power_mean_1m: 0.3696 (baseline: 0.2564, Δ +0.1132)
  fan_mean: 0.1790 (baseline: 0.3549, Δ -0.1759)
  temp_std_1m: 0.1331 (baseline: 0.0706, Δ +0.0625)
  hr_mean_1m: 0.0719 (baseline: 0.0570, Δ +0.0150)
  v_mean_1m: 0.0614 (baseline: 0.0648, Δ -0.0034)

### TEST3 — Temporal split

  fan_mean: 0.2306 (baseline: 0.3549, Δ -0.1244)
  temp_std_1m: 0.1895 (baseline: 0.0706, Δ +0.1189)
  power_mean_1m: 0.1450 (baseline: 0.2564, Δ -0.1114)
  hr_mean_1m: 0.1140 (baseline: 0.0570, Δ +0.0570)
  v_mean_1m: 0.1051 (baseline: 0.0648, Δ +0.0403)

### TEST3 Structural Diagnostic

> 11 miners are permanently label=1 in the test window and 39 are permanently label=0. The test is therefore trivially separable by miner identity alone — AUC=1.0 is a simulator artifact, NOT evidence of temporal leakage.

---

## Verdict

**MILD LEAKAGE — SOFT SPLIT RECOMMENDED**

TEST1 (miner-wise) ROC AUC = 0.9283 and TEST2 (episode) = 0.9425. The drop vs baseline (0.9998 → ~0.93) reflects that the baseline random split allowed the same miner to appear in both train and test — a mild form of leakage. The model still generalises well but the reported baseline metrics were optimistic.

---

## Recommendation

The existing model is usable. For more honest evaluation, use TEST1 (miner-wise) as the canonical evaluation split going forward. No need to replace the production model unless stricter out-of-distribution guarantees are required.

---

## Methodology Notes

- Feature extraction replicates `_build_features` in `xgboost_flagger.py` exactly. `hsi` and `te` are set to defaults (0.0, 50.0) as these come from KPI events not present in the raw telemetry stream.
- Label construction: a tick is labelled 1 if any of the next 360 ticks (same miner) has `fault_injected != None`. Same logic as `train.py`.
- Hyperparameters: identical to the baseline model.
- Episode definition for TEST2: contiguous run of same label-value for the same miner (proxy for fault episode). 80/20 split with `numpy.default_rng(42).permutation`.
- TEST3 sorts all labelled rows by their `global_idx` (original order in the JSONL file, which is already simulator-chronological).

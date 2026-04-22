# 04 — XGBoost Training Details

## 4.1 Training philosophy

Conservative-by-default: only predictors with documented OOD validation run in production. Rule engine provides safety net for everything else.

Three predictors trained. Two active by default. One on standby pending dataset expansion.

## 4.2 Dataset

Generated via the simulator (see [01_simulator.md](01_simulator.md) §fault-injection).

### Composition

- Duration: 40 hours simulated (4h wall at 10× acceleration)
- Miners: 50
- Fault injection mix: balanced across 4 fault types (~25% each among faulted miners)
- Total ticks: 144,000 per miner per 40h

### Label construction

Per-tick binary labels:
- `1` if the current 30-minute window ends within 30 minutes before a fault onset (pre_fault)
- `0` if the window is clean (no active or imminent fault on this miner)

Ticks during active faults are excluded from training (ambiguous status — fault already present, not "pre").

## 4.3 Validation strategy — miner-wise OOD split

### Why miner-wise

Random row-wise split would create training/test contamination: features from the same miner appear in both sets because rolling windows overlap. This inflates metrics optimistically.

**Miner-wise split**: partition miners into train/test sets before feature extraction. Test miners are never seen by the model during training.

### Concrete split

- 35 miners for training
- 15 miners for test (never seen)
- Leave-one-fault-miner-out for fault-specific validation: for each fault type, reserve a miner that experienced that fault and has zero overlap with training

This approach gives realistic generalization estimates and rejects detectors that memorize miner-specific behaviors.

## 4.4 Model 1: `xgb_hashrate` — hashrate_degradation

### Target

Predict `hashrate_degradation` flag within next 30 minutes, regardless of underlying fault cause.

### Hyperparameters

| Param | Value |
|---|---|
| n_estimators | 300 |
| max_depth | 6 |
| learning_rate | 0.05 |
| min_child_weight | 3 |
| subsample | 0.85 |
| colsample_bytree | 0.85 |
| eval_metric | aucpr |

### Metrics (OOD)

| Metric | Value |
|---|---:|
| AUC | 0.928 |
| PR-AUC | 0.837 |
| F1 (0.5 threshold) | 0.82 |
| Precision at recall=0.90 | 0.76 |

### Top feature importance

| Feature | Importance |
|---|---:|
| `hashrate_th_30m_std` | 0.22 |
| `power_w_30m_std` | 0.18 |
| `hashrate_th_5m_std` | 0.15 |
| `temp_chip_c_30m_std` | 0.12 |
| `hashrate_th_1m_mean` | 0.10 |
| remainder (top-10) | 0.23 |

Variance features dominate — consistent with DVFS-driven instability being the physical root of most hashrate degradation.

### Flag emission

Emits flag `hashrate_degradation` with:
- `severity`: `warn` if prob ∈ [0.50, 0.75], `crit` if prob ≥ 0.75
- `confidence`: predicted probability
- `source_tool`: `xgboost_predictor`
- `evidence`: full feature vector with top-5 feature names

### Status

**ACTIVE** — deployed in all experimental runs.

## 4.5 Model 2: `xgb_chip_instability` — chip_instability_precursor

### Target

Predict `chip_instability` fault onset within next 30 minutes, keyed on variance-driven precursor patterns.

### Hyperparameters

Same as `xgb_hashrate` except:
- n_estimators: 400 (slightly more, training ran longer before convergence)
- eval_metric: auc

### Metrics (OOD)

| Metric | Value |
|---|---:|
| AUC | 0.834 |
| PR-AUC | 0.819 |
| F1 (0.5 threshold) | 0.76 |

Lower than `xgb_hashrate` because `chip_instability` has noisier onset timing (gradual build-up) vs hashrate_degradation which has sharper transitions.

### Top feature importance

| Feature | Importance |
|---|---:|
| `power_w_30m_std` | 0.25 |
| `fan_rpm_mean_30m_std` | 0.19 |
| `power_w_5m_std` | 0.15 |
| `hashrate_th_1m_std` | 0.13 |
| `temp_chip_c_5m_std` | 0.11 |
| remainder | 0.17 |

Entirely variance-dominated. No mean features in top-5.

### Flag emission

Emits flag `chip_instability_precursor` with:
- `severity`: `warn` for prob ∈ [0.50, 0.75], `crit` if ≥ 0.75
- `source_tool`: `xgboost_rolling_variance`
- `evidence`: full rolling-variance feature vector

### Status

**ACTIVE** — deployed in all experimental runs.

## 4.6 Model 3: `xgb_hashboard_failure` — standby

### Target

Predict `hashboard_failure` within next 15 minutes using thermal-electrical decoupling signatures.

### Metrics (OOD)

| Metric | Value |
|---|---:|
| AUC | 0.745 |
| PR-AUC | 0.805 |

PR-AUC deceptively good because positive class is extremely rare; AUC is the more reliable indicator at this dataset size.

### Why standby

Discovery dataset (40h) contained only **2 miners that experienced hashboard_failure**. This is insufficient for cross-miner generalization validation — the leave-one-fault-miner-out split effectively becomes leave-one-of-two-out, which provides no statistical power.

### Path to activation

Re-enabled by:
```bash
mdk-orchestra run --profile full_api --enable-hashboard-failure
```

Flag activates the model in the flag emission pipeline. Responsible deployment requires first running:
```bash
mdk-orchestra discover --hours 12 --fault-mix hashboard_heavy
```
to generate ≥5 miner-fault examples before treating predictions as reliable.

### Status

**STANDBY** — trained weights in repo, disabled by default in YAML.

## 4.7 Training reproducibility

### Command

```bash
mdk-orchestra train --data-dir discovery_output/40h_balanced/ \
    --models hashrate,chip_instability,hashboard_failure \
    --seed 42
```

### Artifacts produced

- `models/xgb_predictor.pkl` (~589 KB)
- `models/xgb_chip_instability.pkl` (~987 KB)
- `models/xgb_hashboard_failure.pkl` (~750 KB, standby)
- `models/training_metrics.json` (all metrics tables above)
- `models/feature_importance.json` (full importance vectors)

### Hardware requirements

- Training: ~8 min on M4 Mac mini (16GB RAM), single-core
- Inference: <10ms per prediction on same hardware

## 4.8 Known issues

- `xgb_hashrate` occasionally produces short-window false positives (see [03_pattern_discovery.md](03_pattern_discovery.md) — the `short_window_variance_false_positive` memory pattern documents this). Orchestra's memory layer mitigates: see [05_orchestra_design.md](05_orchestra_design.md) §memory.
- Models trained on simulator data exclusively. Real-miner calibration requires transfer learning or retraining; see [09_limitations.md](09_limitations.md).

## 4.9 Raw artifacts in repo

- `data/xgb_hashrate_metrics.json`
- `data/xgb_chip_instability_metrics.json`
- `data/xgb_hashboard_failure_metrics.json`
- `data/feature_importance_hashrate.csv`
- `data/feature_importance_chip_instability.csv`

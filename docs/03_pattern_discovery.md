# 03 — Pattern Discovery — Full Methodology

## 3.1 Objective

Identify statistically separable and physically interpretable pre-failure telemetry signatures that can be converted into supervised ML predictors.

**Hard constraint**: a pattern must satisfy all three of statistical separability, clustering coherence, and physical plausibility. Falling short on any one disqualifies the pattern for active production use.

## 3.2 Data generation

### Pipeline

1. Run simulator for 40 hours with balanced fault injection (4 fault types, 50 miners)
2. Total ticks: 144,000 (40h × 60min × 60s / 5s per tick — note: 10× accelerated, 40h simulated = 4h wall)
3. Per-fault-type windowing:
   - `pre_fault` window: 30 minutes immediately preceding fault onset
   - `clean` window: matched windows from miners with no active fault
4. Feature extraction: rolling 1m/5m/30m aggregates on all base telemetry
5. Per-miner and cross-miner feature standardization

### Ground truth source

The `fault_injected` field of the simulator is the ground truth. Only available during pattern discovery training; suppressed in production inference telemetry.

## 3.3 Evaluation criteria

Each candidate pattern scored on three independent dimensions:

### Cohen's d (statistical separability)

Measures standardized difference between pre_fault and clean feature distributions.

$$
d = \frac{\mu_{\text{pre\_fault}} - \mu_{\text{clean}}}{\sigma_{\text{pooled}}}
$$

Computed for top-5 most discriminative features per fault type.

Rule of thumb:
- $d > 0.8$ = large effect (usable)
- $d > 2.0$ = very large (high confidence)
- $d > 5.0$ = effectively disjoint distributions

### Silhouette score (clustering coherence)

Measures how well pre_fault points form a distinct cluster from clean points in feature space. Run with both KMeans (k=2) and DBSCAN.

Range: −1 to +1. Positive = coherent separation. Negative = mixed clusters (unusable).

### Physical interpretability

Qualitative: does a known physical mechanism produce the signature? Disqualifies patterns that may be statistically strong but artifactually correlated with the fault injection scheduler.

## 3.4 Pattern results

### Pattern 1 — Rolling variance spike → `chip_instability`

**Physical mechanism**: When an ASIC loses voltage margin, the DVFS (Dynamic Voltage and Frequency Scaling) controller cannot hold a stable frequency step and oscillates between levels. Hashrate variance explodes while mean stays near-nominal. Power and chip_temp co-oscillate in lockstep.

**Top features** (by Cohen's d):

| Feature | Cohen's d | Role |
|---|---:|---|
| `hashrate_th_30m_std` | **12.7** | primary separator — effectively disjoint distributions |
| `power_w_30m_std` | 9.4 | confirms coupling |
| `temp_chip_c_30m_std` | 7.1 | thermal coupling confirmed |
| `power_w_5m_std` | 5.8 | multi-window consistency |
| `fan_rpm_mean_30m_std` | 4.2 | secondary signal |

**Silhouette**: 0.52 (KMeans, k=2) — coherent cluster.

**Verdict**: ACCEPTED. All three criteria satisfied with the strongest statistical profile of any pattern.

### Pattern 2 — Thermal-electrical decoupling → `hashboard_failure`

**Physical mechanism**: When one hashboard fails, the failed board stops drawing power and stops producing heat. But the remaining boards still operate, producing a characteristic ratio shift: `temp_per_power` rises (boards still hot relative to their reduced power), and `power_w` absolute drops proportionally (~290 W mean deficit).

**Top features**:

| Feature | Cohen's d | Role |
|---|---:|---|
| `temp_per_power` (derived, rolling mean) | 2.2 | primary — ratio shift |
| `power_w_30m_mean` (deficit vs nominal) | 2.0 | magnitude ~290 W below baseline |
| `hashrate_th_30m_mean` (deficit %) | 1.9 | step-drop to ~67% nominal visible |
| `temp_chip_c_30m_mean` | 1.3 | residual thermal signal |

**Silhouette**: 0.61 (the most coherent cluster of all patterns — subtle but clean).

**Verdict**: ACCEPTED. Medium separability but very clean cluster and clear physical mechanism.

### Pattern 3 — Voltage drift under ambient load → `cooling_degradation`

**Physical mechanism**: When cooling degrades, voltage rail responds to thermal stress with a small drift (~40 mV). Pattern is physically real but subtle.

**Top features**:

| Feature | Cohen's d | Role |
|---|---:|---|
| `voltage_v_30m_mean` (drift) | 1.59 | primary — ~40 mV downward drift |
| `temp_chip_c_30m_slope` | 1.4 | progressive rise |
| `fan_rpm_mean_30m_mean` (decay) | 1.3 | fan response |

**Silhouette**: 0.25 (too low — high overlap with clean class, especially at high ambient where drift blends with normal voltage response).

**Verdict**: REJECTED for dedicated predictor. Pattern is real but insufficiently separable from clean for supervised production use. Coverage delegated to rule engine reactive thresholds (fan_anomaly, thermal_runaway).

### Discarded — `power_sag`

**Apparent signal**: Features initially looked promising with Cohen's d ~2-3 range.

**Why rejected on inspection**: Post-hoc analysis showed the top-separating feature was `temp_amb_c`, which correlated with the fault injection scheduler itself (the scheduler tended to inject power_sag at hotter simulated times). This is an artifact of the simulator, not a physical pre-failure signature.

**Top-5 Cohen's d average** after removing the scheduler-correlated feature: **0.79** — below the threshold for usable separability.

**Verdict**: REJECTED as non-physical. Lesson: separability alone is insufficient; mechanism must be independently verified.

## 3.5 Feature engineering summary

### Window choices

- **1 minute**: captures fastest dynamics (inrush, single-chip dropout)
- **5 minutes**: medium-term patterns (DVFS oscillation period)
- **30 minutes**: pre-fault build-up signatures (most discriminative for our patterns)

Multi-window features enable the XGBoost models to distinguish between transient noise (visible only at 1m) and sustained anomalies (visible across all windows).

### Cross-variable features

- `temp_per_power` — thermal efficiency per unit load (key for hashboard_failure detection)
- `hsi` (Hashrate Stability Index) — rolling mean/variance composite
- `power_deficit_pct` — relative power shortfall vs nominal

### Total feature space

~60 features per miner per tick after rolling window expansion.

## 3.6 Selection rationale summary

| Pattern | Cohen's d (top) | Silhouette | Physical | Decision |
|---|---:|---:|---|---|
| chip_instability | 12.7 | 0.52 | ✓ | **Active XGBoost** |
| hashboard_failure | 2.2 | 0.61 | ✓ | Trained, standby (dataset size) |
| cooling_degradation | 1.59 | 0.25 | ✓ | Rule engine only |
| power_sag | 0.79 (corrected) | — | ✗ scheduler artifact | Rejected |

Two patterns promoted to supervised XGBoost predictors. Details on training in [04_xgboost_training.md](04_xgboost_training.md).

## 3.7 Raw artifacts in repo

- `data/pattern_discovery_features.json` — full feature matrix (pre_fault + clean)
- `data/pattern_discovery_results.json` — per-pattern Cohen's d and silhouette scores
- `data/cohen_d_ranking.csv` — top-50 features ranked by separability

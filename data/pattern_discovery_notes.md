# Pattern Discovery — Technical Notes

## What was built

- **`feature_extraction.py`**: Loads 144,000 raw `telemetry_tick` envelopes from `raw/telemetry.jsonl`, engineers 60 features per tick (instantaneous signals, rolling aggregates at 1m/5m/30m, per-miner z-scores, cross-variable ratios), and labels each tick as `clean`, `pre_fault`, or `current_fault`.
- **`pattern_mining.py`**: Runs per-fault-type statistical analysis (Cohen's d, mutual information, KMeans + DBSCAN clustering) on the feature matrix, selects the top 3 patterns, and finds concrete example windows.
- **`features.parquet`**: 144,000-row × 66-column feature matrix (60 feature cols + 6 metadata cols).
- **`pattern_analysis.json`**: Machine-readable output with top features per fault type, clustering results, selected patterns, dropped patterns, and concrete examples.

---

## Rolling window edge cases

Rolling windows at tick 0 use `min_periods=1` so early ticks get partial windows. This means:
- `std` columns are 0 for the first tick of each miner (only 1 sample).
- `slope` is 0 or very small in the first few ticks.
- This creates a short "ramp-up" artefact at the start of each miner's sequence (~60 ticks = 5 min), but it does not affect label quality since the first fault injection in this dataset occurs after that warm-up.

Rolling windows are computed per-miner by grouping before concatenation. This avoids cross-miner contamination.

---

## Label generation

`fault_injected` in the raw JSONL is `null` for clean ticks. When loaded into a pandas object column, `null` becomes `float('nan')`. The label generation code uses `pd.isnull()` to detect clean ticks, not `is None`. This was a key bug found during development.

`pre_fault` label: a tick is `pre_fault` if `fault_injected` is NaN **and** the same miner will enter a fault state within the next 360 ticks (30 min = 360 × 5s). The first fault tick within the lookahead window determines `pre_fault_target` and `minutes_to_onset`.

The lookahead uses `np.searchsorted` on sorted fault positions per miner — O(log n) per clean tick instead of O(n), making the 50-miner loop fast.

---

## Feature set rationale

- **Instantaneous signals** (`voltage_v`, `hashrate_th`, `temp_chip_c`, `temp_amb_c`, `power_w`, `fan_rpm_mean`, `fan_rpm_std`): direct sensor readings and fan health.
- **Rolling std / mean / slope** at 3 time scales: captures slow drift (30m) vs fast spikes (1m).
- **Per-miner z-scores** (`*_z`): normalize across miners with different baseline operating points. Computed vs the 30m rolling mean/std for that miner.
- **Cross-variable ratios**: `power_per_hashrate` (efficiency), `voltage_per_power` (load characteristic), `temp_per_power` (cooling effectiveness).

---

## Pattern selection methodology

Combined score = mean(mean_abs_d_top5, normalized_silhouette, dominant_cluster_fraction).
- `mean_abs_d_top5`: average |Cohen's d| over the top-5 separating features.
- `normalized_silhouette`: (silhouette + 1) / 2 → [0,1] scale.
- `dominant_cluster_fraction`: fraction of pre_fault points in the largest cluster.

Patterns failing both thresholds (`max |d| < 0.7 AND best silhouette < 0.2`) are hard-dropped. Patterns passing those thresholds but ranking 4th or lower are soft-dropped.

---

## Results summary

### Label distribution
| Label | Count |
|---|---|
| clean | 113,642 |
| current_fault | 26,694 |
| pre_fault | 3,664 |

### chip_instability — SELECTED (pattern_1)
Strongest pattern by far. Rolling std features (hashrate and power over 30m and 5m windows) show Cohen's d > 10, indicating extreme pre-fault volatility build-up. Silhouette 0.553 with KMeans k=3. Physically: chip instability causes hashrate/power oscillations that accumulate over time, making rolling std an extremely clean signal.

### hashboard_failure — SELECTED (pattern_2)
Best clustering silhouette (0.614). Key separating features are ratio-based: `temp_per_power` (d=2.20) and `voltage_per_power` (d=1.26), reflecting a hashboard drawing less current at similar thermal load. Also `temp_amb_c` shows d=1.24, likely because these faults occurred at slightly different ambient conditions in the simulation (2 miners only, small sample).

### cooling_degradation — SELECTED (pattern_3)
Moderate separability. The signal is concentrated in voltage rolling means (d~1.5) and `temp_amb_c`. This is plausible: cooling degradation correlates with higher ambient temperature driving the fault injection schedule in the simulator. Silhouette is marginal (0.248) but above threshold.

### power_sag — DROPPED (ranked 4th)
Max |Cohen's d| = 1.408 (temp_amb_c) and silhouette 0.429 — both above the hard-drop thresholds. However the top-5 mean |d| is lower (0.791 vs 11.5 / 1.49 / 1.39 for the others), and the dominant cluster fraction (0.346) is the lowest. Ranked 4th → soft-dropped to honor the "top 3 only" constraint. Also suspicious that `temp_amb_c` is the top feature — it may be a simulation artefact (fault injection correlated with ambient conditions) rather than a true causal pre-fault signal for power_sag.

---

## Failed experiments / weird choices

1. **Slope via (last-first)/window**: The spec calls for "simple linear fit approximated as (last-first)/window_size." This is an approximation that ignores intermediate values. For monotone drifts it is adequate; for oscillating signals it underestimates the rate of change. A proper OLS slope was considered but rejected for performance (would require a rolling `apply` with matrix ops).

2. **Clean subsample for Cohen's d**: The clean baseline has 113k rows; the pre_fault set has 720-1080 rows per fault type. Computing Cohen's d and MI on the full clean set would skew variance estimates toward the larger sample. I subsample clean to 20k rows (random, seed=42) for analysis — still 18–28× larger than the pre_fault set, giving stable pooled-std estimates.

3. **DBSCAN eps heuristic**: Auto-estimated via 90th percentile of k-NN (k=5) distance distribution on a 2000-sample subset. In most cases KMeans outperformed DBSCAN (better silhouette), so DBSCAN mainly serves as a cross-check.

4. **`hashrate_expected_th` dropped**: Present in the schema but dropped from the feature set to avoid leakage — it is a simulator-controlled constant that doesn't vary meaningfully per tick and could artificially inflate separability.

5. **`minutes_to_onset` in examples**: Most examples show < 2 min to onset because the "extreme" ticks (90th pct of |z|) tend to be very close to onset — the anomaly signal is strongest just before the fault activates. This is the true peak of the pre-fault window; the signal builds up over 30 min but is most extreme at the end.

---

## Deliverables

| File | Description |
|---|---|
| `pattern_discovery/feature_extraction.py` | Feature engineering script (re-runnable) |
| `pattern_discovery/pattern_mining.py` | Pattern analysis script (re-runnable) |
| `pattern_discovery/features.parquet` | 144k × 66 feature matrix |
| `pattern_discovery/pattern_analysis.json` | Analysis output (patterns, examples) |
| `pattern_discovery/DISCOVERY_NOTES.md` | This file |

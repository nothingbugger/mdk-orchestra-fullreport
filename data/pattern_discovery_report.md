# MDK Fleet — Pattern Discovery on Simulated Telemetry

## Overview

Four hours of simulated telemetry from 50 Bitcoin miners, 5-second tick resolution, 144,000 envelopes total — a balanced mix of clean operation and four injected fault families (`chip_instability`, `hashboard_failure`, `cooling_degradation`, `power_sag`). On top of the raw signals we computed 60 engineered features per tick: rolling means, standard deviations and slopes at three horizons (1m / 5m / 30m), per-miner z-scores, and cross-variable ratios like `power_per_hashrate` and `temp_per_power`. Each tick was then labeled `clean`, `pre_fault` (a 30-minute lookahead window before an injected fault) or `current_fault`.

For every fault family we ranked features by **Cohen's d** and **mutual information** between `pre_fault` and `clean` populations, and independently scored a KMeans + DBSCAN clustering of the pre-fault points in the top-5 feature subspace. The combined score — mean |d| of top-5 × silhouette × dominant-cluster fraction — selected three patterns with defensible, physically interpretable signatures. The goal of this document is to pitch those three as the training targets for the XGBoost predicter, with honest framing on which ones are gifts and which ones are merely decent.

Why it matters: a pre-fault signal strong enough to cluster cleanly is a pre-fault signal strong enough for a supervised model to lock onto with a few hundred examples. Conversely, a "pattern" whose separability comes from an ambient-condition artefact in the simulator is not a pattern — it's a leak. One of the four families (power_sag) failed that smell test and got soft-dropped; the rationale is at the bottom.

---

## Pattern 1 — Rolling-variance spike

**Fault target**: `chip_instability`

**Perché questo è un pattern significativo:**

When a mining ASIC starts losing its voltage margin, the chip can no longer hold a stable clock rate. The DVFS controller oscillates between frequency steps, chasing a target hashrate it can no longer sustain — each oscillation shows up as a burst of power draw and a dip in output. Instantaneous hashrate and power barely move on average, but their **variance explodes**. This is exactly what the data shows: in the 30 minutes before a `chip_instability` event, the rolling standard deviation of hashrate over a 30-minute window jumps from a clean baseline of **0.41 TH/s** to **2.75 TH/s** — a 6.7× amplification. Power 30m σ goes from **19.8 W** to **86.5 W** in lockstep.

Cohen's d for `hashrate_th_30m_std` is **12.7**. For calibration: d=0.8 is already considered "large" in social-science work, d=3 means two populations are so separated a median-split classifier works, d=12.7 means the pre-fault and clean distributions overlap at roughly the 6σ tail — they are effectively disjoint. Four of the top five features (hashrate σ and power σ at two different time horizons, plus chip-temp σ) all sit above d=10. This is not a pattern you need machine learning to find; a two-line threshold rule catches most of it. The ML value is **lead time**: the variance buildup begins well before the fault triggers, so the model learns *how much* pre-fault volatility has accumulated rather than just *whether* the miner is already failing.

Physically, the three signals co-move because they are three faces of the same instability: power σ rises because the ASIC toggles between clock steps, hashrate σ rises because each toggle costs work, chip-temp σ rises because each power burst dumps extra heat into a thermal mass that was in equilibrium at the previous clock. A predicter trained on these three features learns a volatility envelope, not a point estimate — which is the right inductive bias for oscillatory failure modes.

**Feature dominanti** (top 5 with importance):

| Feature | Mean pre_fault | Mean clean | Separation (Cohen's d) |
|---|---|---|---|
| `hashrate_th_30m_std` | 2.75 | 0.41 | **12.73** |
| `power_w_30m_std` | 86.51 | 19.84 | **12.12** |
| `hashrate_th_5m_std` | 2.70 | 0.41 | **11.47** |
| `temp_chip_c_30m_std` | 1.30 | 0.53 | **10.80** |
| `power_w_5m_std` | 84.81 | 19.83 | **10.73** |

**Esempio concreto dal dataset:**

Miner `m020`, 21 Apr 18:10:58 UTC. Over the preceding 30 minutes its rolling hashrate σ had climbed to **3.10 TH/s** (clean baseline: ~0.41). Rolling power σ reached **98.4 W** — roughly 5× the fleet-wide clean median. 16 seconds later, at 18:11:14 UTC, the miner entered the `chip_instability` fault state. Two other miners (`m038`, `m005`) showed the same signature in the same minute with pre-fault windows of 1.25 min and 1.43 min respectively, all three with hashrate 30m σ between 2.66 and 3.10. The pattern reproduces across instances with almost identical magnitudes — the classic sign of a real, non-coincidental precursor.

**Separability**: silhouette **0.553** (KMeans, k=3) — strong. Dominant cluster fraction 0.40 reflects that pre-fault windows span multiple sub-regimes (early vs late in the 30-min lookahead), but all of them live in the high-variance corner of feature space, away from clean operation.

**Fattibilità predittiva**: **high**. d=12.7 is effectively a free win; the question is not whether the predicter will catch this, but how much of the 30-minute horizon it can monetize as actionable lead time.

---

## Pattern 2 — Thermal-electrical decoupling

**Fault target**: `hashboard_failure`

**Perché questo è un pattern significativo:**

A hashboard failure is a board-level problem — a compromised power rail, a bad VRM, a chip that's about to stop drawing current. The signature is subtle on any single sensor, but it shows up loudly in **ratios**: the board runs *hot for the power it's drawing*. `temp_per_power` climbs from a clean **0.0227 °C/W** to **0.0244 °C/W** (d=2.20), while absolute power drops by **~290 W** on average (3264 → 2973 W, d=−1.23). Voltage per unit power follows the same divergence (d=1.26). In plain terms: the board is losing load but not losing heat, so the per-watt thermal signature degrades. That's thermal-electrical decoupling.

The clustering agrees emphatically. KMeans k=3 produces a silhouette of **0.614** — the highest of the three selected patterns — with half the pre-fault points landing in the dominant cluster. So even though the individual feature separations are much weaker than pattern 1 (d≈1.2–2.2 instead of d>10), the pre-fault windows carve out a coherent region of feature space. The model doesn't need a single strong feature; it needs the *joint* ratio-vs-absolute-power signature, which is exactly what tree ensembles handle well.

The caveat: this fault family has only **2 affected miners** in the 4-hour dataset (720 pre-fault ticks). The signal is real and physically grounded, but the sample size means we're describing two instances of the fault, not a population. The predicter should be trained here, but we should monitor it carefully when new hashboard failures land in longer simulation runs — the ratio signature should generalize, but the absolute thresholds may shift.

**Feature dominanti** (top 5 with importance):

| Feature | Mean pre_fault | Mean clean | Separation (Cohen's d) |
|---|---|---|---|
| `temp_per_power` | 0.0244 | 0.0227 | **2.20** |
| `voltage_per_power` | 0.0041 | 0.0037 | **1.26** |
| `temp_amb_c` | 23.34 | 21.88 | 1.24 |
| `power_w_1m_mean` | 2973.2 | 3263.9 | **−1.23** |
| `power_w_5m_mean` | 2973.4 | 3263.9 | **−1.23** |

**Esempio concreto dal dataset:**

Miner `m019`, 21 Apr 18:09:12 UTC. Power 1m mean was already 290 W below fleet baseline, `temp_per_power` sat at **0.026 °C/W** (fleet clean median ~0.023) and `voltage_per_power` at **0.0045 V/W**. 91 seconds later the miner entered `hashboard_failure`. Miner `m036` showed the same pattern — `temp_per_power=0.0238`, `voltage_per_power=0.0037`, 1.52-minute pre-fault window — with nearly identical ambient conditions. Two miners, one signature.

**Separability**: silhouette **0.614** — the highest of the three. The ratio features define a tight cluster. Note that `temp_amb_c` also separates (d=1.24) but this is partly a small-sample correlation — the two affected miners happen to sit in slightly warmer aisles — and should not be leaned on by the predicter.

**Fattibilità predittiva**: **medium**. The signal is clean and physical, but the training set is thin (2 miners, 720 ticks). Expect decent recall if the pattern generalizes; flag for careful validation as more runs accumulate.

---

## Pattern 3 — Voltage drift under ambient load

**Fault target**: `cooling_degradation`

**Perché questo è un pattern significativo:**

As cooling starts to fail, a miner's regulators respond to rising chip temperature by modestly pushing the voltage rail — a small correction to keep the ASIC in spec before the thermal control loop gives up and the fault triggers. The magnitude is tiny (**12.04 V** pre-fault vs **11.99 V** clean — a 40 mV shift) but extremely consistent: across all three rolling horizons (1m, 5m, 30m) the pre-fault voltage mean sits at **12.037 V ± essentially nothing**, with Cohen's d between 1.53 and 1.59. Ambient temperature sits at **23.68 °C** pre-fault vs **21.88 °C** clean (d=1.53) — the cooling system struggles precisely when the room is warmest.

This is the weakest of the three by clustering quality: silhouette **0.248** means the pre-fault points overlap significantly with clean operation in the top-5 feature subspace. The reason is physical: cooling degradation is a *slow* fault — the 30-minute lookahead window straddles both "nothing wrong yet" and "about to trigger" ticks, and the early-window ticks look nearly identical to clean baseline. The predicter will have to learn a soft boundary, and the XGBoost output will reasonably be a probability rather than a hard flag.

That said, the signal is not noise. Four of the top five features cluster in the same physical story: ambient goes up, voltage rail drifts up, chip temp climbs, fault triggers. A model that combines voltage-mean drift with ambient temperature and chip-temp trajectory has a concrete, monotone precursor to work with — just with less margin than pattern 1 or 2.

**Feature dominanti** (top 5 with importance):

| Feature | Mean pre_fault | Mean clean | Separation (Cohen's d) |
|---|---|---|---|
| `voltage_v_30m_mean` | 12.037 | 11.993 | **1.59** |
| `voltage_v_5m_mean` | 12.037 | 11.993 | **1.58** |
| `voltage_v_1m_mean` | 12.037 | 11.993 | **1.53** |
| `temp_amb_c` | 23.68 | 21.88 | 1.53 |
| `voltage_v` | 12.037 | 11.993 | 1.09 |

**Esempio concreto dal dataset:**

Miner `m033`, 21 Apr 18:08:27 UTC. All three rolling voltage means — 1m, 5m, 30m — sat at exactly **12.083 V**, a clean 90 mV above the fleet clean baseline of 11.99 V. The chip had been running hot in a warm aisle. 85 seconds later, at 18:09:51 UTC, the cooling_degradation fault triggered. Two other miners (`m008`, `m042`) showed the same elevated-voltage + warm-ambient signature within minutes, with pre-fault windows of 0.25 and 0.69 minutes respectively.

**Separability**: silhouette **0.248** — marginal. Above the 0.20 hard-drop threshold but the clusters overlap meaningfully, meaning the predicter will need to threshold on probability rather than hard class boundaries.

**Fattibilità predittiva**: **medium-low**. The signal exists and is physically grounded, but the clusters overlap in the top-5 subspace. Expect the model to produce useful probabilities that the orchestrator can compose with other evidence, but do not expect crisp binary flags. The coverage of cooling-related precursors may improve as longer runs accumulate more fault instances.

---

## Riepilogo pattern per training XGBoost

| Pattern | Fault target | Features (top-5) | Recommended XGBoost hyperparameters (initial) |
|---|---|---|---|
| Rolling-variance spike | `chip_instability` | 5 rolling-σ features (hashrate, power, chip-temp at 5m & 30m horizons) | `max_depth=4`, `n_estimators=200`, `learning_rate=0.08`, `subsample=0.8`, `min_child_weight=8`, `scale_pos_weight=~100` (pre_fault is ~0.75% of clean sample) |
| Thermal-electrical decoupling | `hashboard_failure` | 2 ratios + 2 power-mean horizons + ambient temp | `max_depth=5`, `n_estimators=300`, `learning_rate=0.05`, `subsample=0.8`, `min_child_weight=4`, `scale_pos_weight=~150` — deeper trees to exploit ratio/abs-power interactions, more regularization due to small sample |
| Voltage drift under ambient load | `cooling_degradation` | 3 voltage-mean horizons + ambient temp + instantaneous voltage | `max_depth=6`, `n_estimators=400`, `learning_rate=0.05`, `subsample=0.7`, `min_child_weight=6`, `reg_alpha=0.1`, `scale_pos_weight=~140` — deeper trees and L1 to cut through the overlapping clusters; expect the model to output calibrated probabilities rather than hard flags |

Notes for all three: use `early_stopping_rounds=30` against a held-out miner-stratified validation split (never time-stratified within a miner — that leaks the lookahead). Start with `objective='binary:logistic'` per pattern (three separate models) rather than multi-class — the failure modes don't compete for the same precursor space. Monitor PR-AUC, not ROC-AUC, given the heavy class imbalance.

---

## Pattern scartati e perché

**`power_sag` — soft-dropped (ranked 4th by combined score).**

This fault family passed both hard-drop thresholds (max |d| = 1.41, silhouette = 0.43) so it's not a *bad* pattern — it's just weaker than the other three and carries a specific red flag.

The top feature for power_sag is **`temp_amb_c`** with d=1.41. That alone is suspicious: ambient temperature is a slow environmental signal and shouldn't be the single strongest precursor for a *power-delivery* fault on an AC/DC conversion stage. The physically-plausible power_sag precursors — voltage std, power efficiency ratios — sit much lower in the ranking (d ≈ 0.35 for `voltage_v_5m_mean`, d = 0.78 for `voltage_v_30m_std`). The mean |d| over the top-5 is just **0.79**, compared to 11.57 / 1.43 / 1.47 for the three selected patterns. And the dominant-cluster fraction (0.35) is the lowest of the four families.

The most plausible reading is that the simulator's power_sag injection is correlated with ambient conditions by construction (thermal stress triggers sag events more often), and the model is picking up that correlation rather than a true pre-fault signal in the electrical domain. In production that correlation would not hold — or would hold differently — so training on it invites a brittle model.

Action: dropped from the initial three, revisit when we have longer simulation runs and can confirm whether a genuine electrical-domain precursor (voltage dips, power slope) emerges once the ambient confound is averaged out across more instances.

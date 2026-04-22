# MDK Fleet — Architecture comparison smoke

Same flag, four configurations. Canonical flag: `hashrate_degradation warn/0.75` on `m901` (test miner id conforming to the MinerId pattern `m\d{3}`), XGBoost predictor evidence `predicted_prob_drop_20pct_30min=0.85`, with a realistic rolling-feature snapshot extracted from the longrun dataset.

## Comparative table

| Config | Cost | Latency | Action | Autonomy | N API calls | Model(s) |
|---|---:|---:|---|---|---:|---|
| multi_full_api | $0.0167 | 14.3s | `observe` | `L1_observe` | 2 | claude-sonnet-4-6 (dispatch+specialists) [+ opus on L3/L4] |
| single_sonnet | $0.0208 | 22.4s | `alert_operator` | `L2_suggest` | 1 | claude-sonnet-4-6 |
| single_opus | $0.0714 | 8.1s | `observe` | `L1_observe` | 1 | claude-opus-4-7 |
| multi_full_local | $0.0000 | 141.4s | `throttle` | `L3_bounded_auto` | 0 | qwen2.5:7b-instruct-q4_K_M (via Ollama, Mac B) |

## Reasoning traces

### multi_full_api

**Action:** `observe`  ·  **Autonomy:** `L1_observe`  ·  **Confidence:** 0.81  ·  **Latency:** 14.3s
**Consulted:** voltage_agent

```
m901 hashrate_degradation warn/0.75. XGBoost raw_score=0.85, top features fan_rpm_mean_30m_std and power_w_5m_std — ancillary variance features, not hashrate trajectory. Hashrate agent: noise conf=0.78 ('hr_mean_1m = hr_mean_6m = 105.33 TH/s, σ=0.231 — stationary band, zero drift across windows; HSI=98.81, TE=100.0, temp_std=0.18°C — thermally stable; model fired on momentary fan/power std variance, not sustained hashrate movement'). Single-specialist verdict is decisive at 0.78 — no fallback needed. Aligns with curated pattern xgboost_hashrate_fp_unanimous_noise: predictor tripped on short-window feature variance with no underlying degradation signal. Downgrade to L1 observe. Flag for predictor calibration if same miner re-fires this cycle.
```

### single_sonnet

**Action:** `alert_operator`  ·  **Autonomy:** `L2_suggest`  ·  **Confidence:** 0.72  ·  **Latency:** 22.4s

```
**Voltage domain:** v_mean_1m=12.066V, v_std_1m=0.018V — PSU rail is essentially textbook stable. No sag, no ripple. Voltage is not the driver here.

**Hashrate trajectory:** hr_mean_1m=105.33 TH/s == hr_mean_6m=105.33 TH/s, std identical across both windows (0.231). The miner is currently stationary/flat — no step, ramp, or sawtooth yet. The degradation is *predicted* (85% prob of >20% drop within 30 min), not yet observed. XGBoost raw_score=0.85 at flag confidence=0.75.

**Thermal domain:** temp_mean_1m=72.36°C, temp_std_1m=0.18°C — elevated but not yet critical (typical danger zone >80°C). HSI=98.81 and TE=100.0 are both near saturation, indicating the cooling headroom is essentially exhausted. fan_mean=5889.5 RPM — fans already running hard. Top feature driving prediction: fan_rpm_mean_30m_std — i.e., fan RPM *variance* is elevated, suggesting the cooling system is hunting/oscillating rather than steady-state. This is a thermal regulation instability signal, not a simple overheat.

**Power domain:** power_mean_1m=3247.79W, power_std_1m=14.14W — power draw is within normal range but power_w_5m_std is a top feature, meaning short-window power fluctuations are abnormal. Combined with fan instability, this suggests either a workload burst pattern stressing thermal management or an early PSU/VRM interaction under sustained load.

**Cross-domain synthesis:** The current state looks superficially healthy (stable V, flat HR, sub-critical temp), but the model is picking up on leading indicators: (1) fans oscillating at high RPM (hunting → thermal control loop instability), (2) short-window power variance elevated. HSI+TE both at ceiling means any additional thermal load or HVAC degradation has zero buffer. The predicted failure mode is likely thermal-induced hashrate throttle by the miner's own firmware within 30 min.

**Autonomy ladder assessment:**
- Evidence is predictive (not yet confirmed failure) at moderate-high confidence (0.75 flag / 0.85 raw score).
- Possible responses: throttle (L3, reversible) OR alert operator (L2).
- Throttling now is reversible but acts on ambiguous evidence — the miner is currently performing nominally. The thermal signal is indirect (fan variance, not temp spike).
- HSI/TE saturation and fan hunting are real concerns, but the correct first step with no confirmed degradation and an indirect signal chain is L2: alert operator with a concrete recommendation (pre-emptive throttle to ~80% or HVAC inspection), rather than autonomously acting on a prediction alone.
- This keeps the human in the loop for a miner currently at full healthy output, avoiding unnecessary revenue impact from an autonomous throttle that may be premature.

**Recommendation to operator:** (1) Inspect HVAC / rack airflow for m901 immediately — TE=100 and HSI=98.81 indicate zero thermal margin. (2) Authorize throttle to ~80% as a precautionary measure if HVAC cannot be corrected within 15 min. (3) Monitor fan RPM std and power_std over next 10 min for trend confirmation.
```

### single_opus

**Action:** `observe`  ·  **Autonomy:** `L1_observe`  ·  **Confidence:** 0.60  ·  **Latency:** 8.1s

```
Cross-domain check on m901:
- Hashrate: hr_mean_1m=hr_mean_6m=105.33, std 0.231 — stationary, no degradation yet. HSI 98.81, TE 100 — healthy.
- Voltage: 12.066V ±0.018 — rock solid PSU, no sag.
- Power: 3247W ±14 — stable, no droop.
- Thermal: 72.36C ±0.18 — within spec, fans 5889 RPM nominal.
- Model flags drop prob 0.85 based on fan_rpm_30m_std and power_5m_std, but current snapshot shows low variance. Predictor may be reacting to earlier transient not visible in 1m window, or false positive.
Evidence of imminent 20% drop is weak vs. current telemetry. Severity=warn, confidence 0.75. No justification for throttle/migrate (would cost hashrate on healthy miner). Log and monitor; if degradation materializes, escalate. Staying L1; would bump to L2 alert_operator if this were severity=critical.
```

### multi_full_local

**Action:** `throttle`  ·  **Autonomy:** `L3_bounded_auto`  ·  **Confidence:** 0.97  ·  **Latency:** 141.4s
**Source:** ab_longrun_20260422_0017 (27 decisions)

```
m002 hashrate_degradation crit/0.97. XGBoost: real_signal 0.96, high probability of 20% drop within 30min. Voltage and temp stable, no site-wide issues. Throttling to 80%/60min reduces risk while allowing miner to cool down. Reversible action, L3.
```

## Qualitative notes

_To be filled during pitch editorialization — the raw traces above are the source material. Key questions the pitch needs to answer:_

- **Cross-domain reasoning**: did the single-agent traces reach conclusions by reasoning about voltage AND hashrate AND environment, or did they skip directly to a decision based on the XGBoost probability alone? The multi-agent trace forces every consultation to surface one domain at a time — that transparency is visible in the trace header (`consulted_agents`).
- **Auditability**: the multi-agent trace preserves each specialist's verdict verbatim. A single-agent trace is one narrative — if it got the domain framing wrong, there is no other voice to flag the mismatch.
- **Action divergence**: compare the three API configurations on the `action` + `autonomy_level` columns. Any divergence — especially between single-Opus and multi-full_api — is the strongest evidence for the architectural value of having specialists.

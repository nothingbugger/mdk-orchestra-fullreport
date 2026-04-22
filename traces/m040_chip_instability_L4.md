# Hero Trace — m040 · chip_instability_precursor → L4 human_review

**Run**: Complete API canonical run (§6.2)
**Build ID**: `ab_short_complete_20260422_0958`
**Decision ID**: `dec_027`
**Timestamp**: 2026-04-22T08:10:14Z

---

## Flag received

- **Miner**: `m040`
- **Flag type**: `chip_instability_precursor`
- **Severity**: `warn`
- **Source tool**: `xgboost_rolling_variance`
- **Confidence**: 0.731
- **Top feature**: `hashrate_th_30m_std` = 10.5177 (~10× normal envelope)

### Full flag envelope

```json
{
  "flag_id": "flg_00027",
  "miner_id": "m040",
  "flag_type": "chip_instability_precursor",
  "severity": "warn",
  "confidence": 0.731,
  "source_tool": "xgboost_rolling_variance",
  "evidence": {
    "metric": "hashrate_th_30m_std",
    "window_min": 30.0,
    "predicted_prob": 0.731,
    "top_feature_name": "hashrate_th_30m_std",
    "top_feature_value": 10.5177,
    "features": [
      "hashrate_th_30m_std",
      "power_w_30m_std",
      "hashrate_th_5m_std",
      "temp_chip_c_30m_std",
      "power_w_5m_std",
      "hashrate_th_1m_std",
      "power_w_1m_std",
      "temp_chip_c_5m_std",
      "fan_rpm_mean_30m_std",
      "hashrate_th_1m_mean",
      "hashrate_th_30m_mean",
      "temp_chip_c_1m_mean",
      "power_w_1m_mean",
      "fan_rpm_mean_1m_std"
    ],
    "feature_values": [
      10.5177,
      341.6098,
      15.9143,
      4.9522,
      519.4683,
      17.954,
      580.9585,
      7.5591,
      122.5403,
      90.3342,
      104.8642,
      67.1125,
      2923.75,
      132.858
    ]
  },
  "raw_score": 0.731
}
```

---

## Dispatch decision

Maestro's dispatch table maps `chip_instability_precursor` to:
- Primary: `hashrate_specialist`
- Fallback: `voltage_specialist` (consulted only if primary returns inconclusive)

On this flag: primary returned definitive verdict, so no fallback consultation triggered.

---

## Specialist consultation

### Hashrate specialist

- **Agent**: `hashrate_specialist`
- **Model**: Sonnet 4.6
- **Cost**: \$0.1007
- **Latency**: 14.8 s

**Verdict**: `real_signal`
**Confidence**: 0.91
**Severity estimate upgraded**: `warn` → `crit`

**Reasoning** (specialist output, verbatim):

> Bilateral oscillation pattern observed: hashrate swings between 43.9 and 116.2 TH/s (against 115 TH/s nameplate). 30-minute mean ~84 TH/s, −26.9% vs nominal. The distinctive marker is the in-phase coupling of chip temperature and power with hashrate — when hashrate drops to 43.9 TH/s the chip temp is 46 °C, when it spikes to 92.7 TH/s the chip temp climbs to 65 °C in the same minute. Power tracks in lockstep.
>
> This lockstep coupling rules out thermal sawtooth (which would show temp-driven hashrate response with phase lag) and pool-side stall (which would show hashrate drops without correlated power/thermal change). The remaining hypothesis consistent with the data is chip substrate intermittent dropout: chips randomly dropping out of the hashrate pool and coming back, with power and thermal directly following their duty cycle.
>
> HSI is collapsed from ~92 (miner's own baseline) to 69–74 range. sigma_hash of 17 TH/s is approximately 10× the normal variance envelope for this miner class.
>
> Verdict: real_signal, upgraded from warn to crit severity. No ambiguity warranting fallback specialist consultation.

---

## Maestro synthesis

- **Agent**: `ag_maestro`
- **Model**: Opus 4.7 (escalation tier triggered because specialist severity is `crit`)
- **Cost**: \$0.1641 total (including dispatch + specialist + synthesis)
- **Total latency**: 30.8 s
- **Specialists consulted**: `hashrate_specialist` (primary, conclusive)
- **Pending human approval**: True

### Maestro reasoning (verbatim)

> m040 `chip_instability_precursor` warn/0.73 (xgboost_rolling_variance, top feature hashrate_th_30m_std=10.52). Hashrate specialist: real_signal conf=0.91, severity upgraded warn→crit.
>
> Bilateral oscillation 43.9–116.2 TH/s vs 115 TH/s nameplate, 30m mean ~84 TH/s (−26.9%); chip temp and power co-oscillate in lockstep with hashrate (43.9 TH/s@46°C ↔ 92.7 TH/s@65°C), ruling out thermal sawtooth or pool stall — this is chip substrate intermittent dropout. HSI collapsed from ~92 to 69–74; sigma_hash ~17 TH/s is ~10x normal envelope. Unambiguous verdict, no fallback warranted.
>
> Throttle contraindicated: reducing target hashrate won't arrest intermittent substrate failure and may mask progression.
>
> Shutdown is the correct protective action but is L4-mandatory per autonomy ladder (never auto-shutdown). Queuing for human approval with high-urgency alert; operator should expect possible hashboard retirement.
>
> Confirming first-pass Sonnet decision.

---

## Final action

- **Action**: `human_review`
- **Autonomy level**: `L4_human_only`
- **Confidence**: 0.90
- **Total cost**: \$0.164
- **Total latency**: 30.8 s
- **Executor status**: queued for operator approval, not auto-executed

---

## Why this trace matters

This case demonstrates every intended function of the system in a single decision:

1. **Detection by supervised predictive layer** — XGBoost fired on variance features, not thresholds
2. **Disambiguation by specialist** — hashrate specialist ruled out competing hypotheses (thermal sawtooth, pool stall) using physical reasoning (lockstep coupling)
3. **Action selection by Maestro** — Maestro rejected the naive "fault → throttle" mapping with a physics-based argument (throttle doesn't address substrate failure)
4. **Autonomy ladder enforcement** — Maestro cited the L4-mandatory rule explicitly (never auto-shutdown) and routed to human_review rather than emergency_shutdown
5. **Next-action guidance** — Maestro suggested "possible hashboard retirement" — the actual operator-relevant maintenance action

A threshold-based detection system cannot produce any of (2), (3), (4), (5). It can only fire the flag. Orchestra supplies the reasoning layer that turns detection into operational judgment.

---

## Screenshots

Dashboard at moment of decision: `../screenshots/08_hero_reasoning.png`
Flag raised: `../screenshots/07_hero_flag.png`
Action dispatched: `../screenshots/09_hero_action.png`

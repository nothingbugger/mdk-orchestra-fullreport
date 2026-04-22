# Canonical Memory Patterns — Initial Ensemble Run

These patterns were written by Opus 4.7 during the curation cycle of the initial ensemble run (§6.1).

They are **canonical** — preserved into all subsequent runs and used as seeds for the complete API canonical run.

All patterns below are verbatim from the memory files, exactly as Opus produced them.

---

## Pattern in `maestro_memory.md`

### `xgboost_hashrate_fp_unanimous_noise`

```markdown
## Pattern: xgboost_hashrate_fp_unanimous_noise
- First seen: 2026-04-21T20:47:47
- Last seen: 2026-04-21T20:49:12
- Occurrences: 2
- Signature: XGBoost hashrate flag with severity=crit, both specialists return assessment=noise with confidence > 0.80
- Learned verdict: L1_observe
- Confidence: 0.85
- Reasoning: "When both required specialists unanimously call noise above 0.80 on an
  XGBoost hashrate flag, the deterministic model is almost certainly firing on
  short-window feature noise. Do NOT escalate to L2 despite crit severity — that
  path leads to alert fatigue."
- Example reference: dec_008
```

**Why this pattern matters**: captures a known failure mode of the xgb_hashrate predictor (short-window variance false positives) and encodes Maestro's learned response (do not upgrade despite crit severity).

**How it was used in the complete API canonical run**: silently consulted during filtering and during L1 observe decisions. Enabled Orchestra to reach correct L1 outcomes without expensive specialist consultation on several cases.

---

## Pattern in `hashrate_memory.md`

### `short_window_variance_false_positive`

```markdown
## Pattern: short_window_variance_false_positive
- First seen: 2026-04-21T20:47:58
- Last seen: 2026-04-21T20:47:58
- Occurrences: 1
- Signature: 1-minute hashrate std elevated while 30-minute std is within normal envelope (ratio 1m_std / 30m_std > 5)
- Learned assessment: noise
- Confidence: 0.87
- Reasoning: "The predictor's 1-min feature sensitivity generates false positives
  when the 30-min trajectory is flat. Check: is mean within ±2% of nominal? Is σ
  consistent with this miner's baseline variance?"
- Example reference: dec_005
```

**Why this pattern matters**: domain-specific calibration knowledge about the xgb_hashrate predictor's characteristic failure mode. Enables hashrate_specialist to quickly classify as `noise` without deep reasoning.

---

## Summary

Two patterns total written in this curation cycle:

| Agent | Pattern slug | Confidence |
|---|---|---:|
| maestro | xgboost_hashrate_fp_unanimous_noise | 0.85 |
| hashrate | short_window_variance_false_positive | 0.87 |

Both patterns are **lesson-formatted** (distilled operational rules, not event logs). Both have explicit signatures that allow future matching, learned outcomes, and reasoning. Both reference an example decision for audit trail.

This is the template for all memory curation — Maestro writes patterns of this form, not verbose logs.

---

## A note on memory seeding vs pattern accumulation

These two patterns are the only "canonical seed" memories shipped in `examples/memories/` of the mdk-orchestra repo. They were sufficient to influence the complete API canonical run in measurable ways:

- Several L1 observe decisions matched the maestro pattern and were resolved without specialist consultation
- The hashrate specialist used the short-window pattern to calibrate confidence on borderline flags

They are not, however, a complete memory. Over time, an operator would accumulate dozens or hundreds of patterns specific to their site, their miner population, their tariff profile, their cooling strategy.

The beauty of the markdown format is that this accumulation is transparent: the operator can read, edit, delete, or share any pattern at any time. The system does not hide its learned behavior in opaque weights.

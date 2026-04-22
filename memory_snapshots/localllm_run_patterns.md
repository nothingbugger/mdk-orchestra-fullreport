# Memory Pattern Written in the Complete LocalLLM Run

The 9h28m complete LocalLLM run (§6.3) ran 14 curation cycles using Qwen 2.5 7B Q4_K_M as the curator. Qwen wrote 1 pattern total.

For comparison, Opus 4.7 wrote 2 patterns in 1 cycle in the initial ensemble run ([initial_run_canonical_patterns.md](initial_run_canonical_patterns.md)).

---

## Pattern in `hashrate_memory.md`

### `high_prob_hashrate_degradation_throttle`

```markdown
## Pattern: high_prob_hashrate_degradation_throttle
- First seen: 2026-04-22T04:23:11
- Last seen: 2026-04-22T08:57:43
- Occurrences: 3
- Signature: xgboost_predictor flag with confidence > 0.82 AND hashrate degradation sustained across 30-minute window
- Learned action: L3_throttle
- Confidence: 0.97
- Reasoning: high probability detection with sustained degradation justifies autonomous throttle
- Example reference: dec_015
```

**Assessment of pattern quality**: functional but shallow. The pattern correctly identifies a high-probability case → autonomous throttle mapping, which is the right response. But the reasoning is terse and does not distinguish this case from other hashrate_degradation scenarios (thermal drift, fan issue, hashboard failure) that may also have high probability but require different responses.

**Limitation observed**: Qwen's distillation did not surface the trajectory-shape distinctions that a larger model like Opus articulates. This is consistent with the general quality-vs-cost tradeoff:

| Curator model | Patterns per cycle (typical) | Pattern quality |
|---|---:|---|
| Opus 4.7 | 1–3 rich, multi-dimensional patterns | High (e.g., unanimous noise, window variance distinctions) |
| Qwen 2.5 7B | 0–1 narrow patterns | Functional but terse |

---

## Why this matters

Local-only deployments get simpler memory curation. This has implications for deployment strategy:

1. **Pure full_local**: curation runs on Qwen (or whatever local model). Accumulated memory will be narrower, slower to generalize, more likely to need operator editing over time.

2. **Hybrid curation**: run inference on local (cost-zero, privacy-preserving), but schedule periodic curation via API on a high-tier model (Opus 4.7). Best of both worlds — local inference cost + quality curation.

3. **Operator-driven curation**: operator reviews decision logs periodically and writes memory patterns manually. Orchestra's memory format is intentionally human-writable.

For the complete LocalLLM run itself, this single pattern was sufficient to demonstrate that curation works end-to-end on local infrastructure. For production deployment at scale, the hybrid curation strategy is the recommended path.

---

## Observed pattern effect in the run

The pattern was written in cycle 3 (hour ~4 of the run). After that, subsequent throttle decisions in the run tended to match this pattern and complete faster (specialist consultation skipped when pattern confidence matched). The observable impact: mean latency for L3 throttle decisions dropped from ~155 s (pre-pattern) to ~135 s (post-pattern) as Qwen could skip part of the reasoning chain.

Small but real memory-driven speedup, achieved with zero API cost.

---

## Memory file state at end of run

Memory files after the 9h28m run:

- `maestro_memory.md`: 1 pattern (canonical `xgboost_hashrate_fp_unanimous_noise` from seed, preserved)
- `hashrate_memory.md`: 2 patterns (seed `short_window_variance_false_positive` + new `high_prob_hashrate_degradation_throttle`)
- `voltage_memory.md`: empty
- `environment_memory.md`: empty
- `power_memory.md`: empty

Specialists other than hashrate did not accumulate patterns in this run because the fault mix did not exercise them enough (majority of flags were hashrate-adjacent).

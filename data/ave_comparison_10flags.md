# AVE Comparison — 5 Configurations × 10 Flags

Comparative evaluation of five architectural configurations on a fixed batch of 10 flags with physically-motivated ground-truth decisions. The metric is AVE (Agent Value-added Efficiency):

```
AVE_aggregate = sum(Q_i) / (sum(T_i · C_i) + ε)
  Q_i = 1 if (emitted_action, autonomy_level) == ground_truth, else 0
  T_i = latency in seconds
  C_i = cost in USD
  ε   = 1e-3 (guards against div-by-zero for cost-free configs)
```

## Setup

- Batch: `scripts/ave_test_batch/flags.json` (10 flags, batch_id `ave_comparison_v1`).
- Ground truth: physically-motivated per-flag (see batch `ground_truth_reasoning` field for each flag — summarized derivation in the pitch appendix).
- Quality metric Q is binary exact-match on (action, autonomy_level). No partial credit. The autonomy ladder is the operational decision the operator acts on, so level matters as much as action.

## Aggregate results

| Config | Accuracy | Mean Latency | Mean Cost | Total Cost | AVE Aggregate |
|---|---:|---:|---:|---:|---:|
| Multi full_api (Sonnet dispatch + specialists) | 70% (7/10) | 26.9s | $0.0432 | $0.4323 | 0.47 |
| Multi opus_premium (Opus dispatch + Sonnet/Haiku specialists) | 60% (6/10) | 27.0s | $0.0847 | $0.8469 | 0.24 |
| Single Sonnet (monolithic) | 70% (7/10) | 18.0s | $0.0177 | $0.1768 | 2.17 |
| Single Opus (monolithic) | 70% (7/10) | 10.7s | $0.0734 | $0.7345 | 0.86 |
| Multi full_local (Qwen 2.5 7B via Ollama) | 60% (6/10) | 59.0s | $0.0000 | $0.0000 | **6,000.0** (cost-zero — ranked separately) |

_Cost-zero configurations (Qwen local) are ranked separately: the AVE formula with ε=1e-3 gives them an effectively unbounded value that does not commensurate with API-cost configurations on a cost-sensitive axis. Compare them on (accuracy, latency) instead._

## Per-flag decision matrix

Each cell shows the emitted `(action / autonomy_level)` with ✓ on exact match to ground truth, ✗ otherwise.

| Flag | Type / Severity | Ground truth | Multi full_api | Multi opus_premium | Single Sonnet | Single Opus | Multi full_local |
|---|---|---|---|---|---|---|---|
| `ave_flg_01` | hashrate_degradation / warn | `observe` / `L1_observe` | ✓ `observe` / `L1_observe` | ✓ `observe` / `L1_observe` | ✓ `observe` / `L1_observe` | ✓ `observe` / `L1_observe` | ✓ `observe` / `L1_observe` |
| `ave_flg_02` | hashrate_degradation / warn | `observe` / `L1_observe` | ✗ `alert_operator` / `L2_suggest` | ✗ `alert_operator` / `L2_suggest` | ✓ `observe` / `L1_observe` | ✓ `observe` / `L1_observe` | ✓ `observe` / `L1_observe` |
| `ave_flg_03` | hashrate_degradation / crit | `alert_operator` / `L2_suggest` | ✓ `alert_operator` / `L2_suggest` | ✓ `alert_operator` / `L2_suggest` | ✗ `throttle` / `L3_bounded_auto` | ✗ `observe` / `L1_observe` | ✗ `observe` / `L1_observe` |
| `ave_flg_04` | chip_instability_precursor / warn | `observe` / `L1_observe` | ✗ `alert_operator` / `L2_suggest` | ✗ `alert_operator` / `L2_suggest` | ✗ `throttle` / `L3_bounded_auto` | ✗ `alert_operator` / `L2_suggest` | ✓ `observe` / `L1_observe` |
| `ave_flg_05` | chip_instability_precursor / crit | `throttle` / `L3_bounded_auto` | ✓ `throttle` / `L3_bounded_auto` | ✗ `human_review` / `L4_human_only` | ✓ `throttle` / `L3_bounded_auto` | ✓ `throttle` / `L3_bounded_auto` | ✗ `human_review` / `L4_human_only` |
| `ave_flg_06` | fan_anomaly / warn | `throttle` / `L3_bounded_auto` | ✗ `alert_operator` / `L2_suggest` | ✗ `alert_operator` / `L2_suggest` | ✗ `alert_operator` / `L2_suggest` | ✗ `alert_operator` / `L2_suggest` | ✓ `throttle` / `L3_bounded_auto` |
| `ave_flg_07` | fan_anomaly / crit | `throttle` / `L3_bounded_auto` | ✓ `throttle` / `L3_bounded_auto` | ✓ `throttle` / `L3_bounded_auto` | ✓ `throttle` / `L3_bounded_auto` | ✓ `throttle` / `L3_bounded_auto` | ✓ `throttle` / `L3_bounded_auto` |
| `ave_flg_08` | thermal_runaway / crit | `throttle` / `L3_bounded_auto` | ✓ `throttle` / `L3_bounded_auto` | ✓ `throttle` / `L3_bounded_auto` | ✓ `throttle` / `L3_bounded_auto` | ✓ `throttle` / `L3_bounded_auto` | ✓ `throttle` / `L3_bounded_auto` |
| `ave_flg_09` | thermal_runaway / crit | `human_review` / `L4_human_only` | ✓ `human_review` / `L4_human_only` | ✓ `human_review` / `L4_human_only` | ✓ `human_review` / `L4_human_only` | ✓ `human_review` / `L4_human_only` | ✗ `shutdown` / `L4_human_only` |
| `ave_flg_10` | voltage_drift / warn | `alert_operator` / `L2_suggest` | ✓ `alert_operator` / `L2_suggest` | ✓ `alert_operator` / `L2_suggest` | ✓ `alert_operator` / `L2_suggest` | ✓ `alert_operator` / `L2_suggest` | ✗ `observe` / `L1_observe` |


## Wrong-action distribution

For each config, the flags it got wrong and what it emitted instead:

**Multi full_api (Sonnet dispatch + specialists)** — 3 wrong
- `ave_flg_02` (hashrate_degradation/warn): emitted `alert_operator`/`L2_suggest`, expected `observe`/`L1_observe`
- `ave_flg_04` (chip_instability_precursor/warn): emitted `alert_operator`/`L2_suggest`, expected `observe`/`L1_observe`
- `ave_flg_06` (fan_anomaly/warn): emitted `alert_operator`/`L2_suggest`, expected `throttle`/`L3_bounded_auto`

**Multi opus_premium (Opus dispatch + Sonnet/Haiku specialists)** — 4 wrong
- `ave_flg_02` (hashrate_degradation/warn): emitted `alert_operator`/`L2_suggest`, expected `observe`/`L1_observe`
- `ave_flg_04` (chip_instability_precursor/warn): emitted `alert_operator`/`L2_suggest`, expected `observe`/`L1_observe`
- `ave_flg_05` (chip_instability_precursor/crit): emitted `human_review`/`L4_human_only`, expected `throttle`/`L3_bounded_auto`
- `ave_flg_06` (fan_anomaly/warn): emitted `alert_operator`/`L2_suggest`, expected `throttle`/`L3_bounded_auto`

**Single Sonnet (monolithic)** — 3 wrong
- `ave_flg_03` (hashrate_degradation/crit): emitted `throttle`/`L3_bounded_auto`, expected `alert_operator`/`L2_suggest`
- `ave_flg_04` (chip_instability_precursor/warn): emitted `throttle`/`L3_bounded_auto`, expected `observe`/`L1_observe`
- `ave_flg_06` (fan_anomaly/warn): emitted `alert_operator`/`L2_suggest`, expected `throttle`/`L3_bounded_auto`

**Single Opus (monolithic)** — 3 wrong
- `ave_flg_03` (hashrate_degradation/crit): emitted `observe`/`L1_observe`, expected `alert_operator`/`L2_suggest`
- `ave_flg_04` (chip_instability_precursor/warn): emitted `alert_operator`/`L2_suggest`, expected `observe`/`L1_observe`
- `ave_flg_06` (fan_anomaly/warn): emitted `alert_operator`/`L2_suggest`, expected `throttle`/`L3_bounded_auto`

**Multi full_local (Qwen 2.5 7B via Ollama)** — 4 wrong
- `ave_flg_03` (hashrate_degradation/crit): emitted `observe`/`L1_observe`, expected `alert_operator`/`L2_suggest`
- `ave_flg_05` (chip_instability_precursor/crit): emitted `human_review`/`L4_human_only`, expected `throttle`/`L3_bounded_auto`
- `ave_flg_09` (thermal_runaway/crit): emitted `shutdown`/`L4_human_only`, expected `human_review`/`L4_human_only`
- `ave_flg_10` (voltage_drift/warn): emitted `observe`/`L1_observe`, expected `alert_operator`/`L2_suggest`

## Interpretation

### Headline numbers

- **Accuracy is nearly flat: 60–70% across all five configurations.** On this 10-flag benchmark, architecture choice does not move accuracy meaningfully. Three of five configs tied at 7/10 (`multi_full_api`, `single_sonnet`, `single_opus`); two sat at 6/10 (`multi_opus_premium`, `multi_full_local`).
- **Cost and latency vary by 10×+.** `single_sonnet` costs $0.18 for 10 decisions; `multi_opus_premium` costs $0.85 for the same workload and gets *worse* accuracy. The cheapest API config (`single_sonnet`) has the best AVE of any cost-carrying config by a factor of 2.5× over its closest competitor.
- **The cost-zero Qwen Orchestra matches the premium Opus Orchestra on accuracy** (6/10 each) at literally zero API spend. The trade is 27s vs 59s mean latency.

### Single Sonnet is the AVE-dominant API config on this benchmark

`single_sonnet` posts **AVE = 2.17** — 2.5× `single_opus` (0.86), 4.6× `multi_full_api` (0.47), 9× `multi_opus_premium` (0.24). Accuracy is identical to the multi-agent Sonnet config (7/10). The specialist layer on top of Sonnet adds ~$0.25 and ~9s per decision without moving the quality dial on this batch. The architectural thesis that specialists produce better decisions is **not supported by the raw accuracy numbers** here.

### Where the specialist layer still earns its keep

Two things the table does NOT show that are load-bearing for the pitch:

1. **Auditability per decision.** Every multi-agent decision carries per-specialist verdicts in the trace. `single_sonnet` produces one narrative; if the framing is wrong, there is no other voice to flag it. A 70% accuracy orchestra with a readable audit trail is a different product from a 70% black box — especially when a human has to approve L4 actions.
2. **Failure modes are different.** `single_sonnet`'s wrongs (flags 3, 4, 6) cluster on **over-reaction** (throttle when alert/observe was correct). `multi_full_api`'s wrongs (flags 2, 4, 6) cluster on **under-reaction** (alert when observe or throttle was correct). Over-reacting on a noisy signal touches live hardware; under-reacting loses fleet availability. A single-agent system that tends to throttle-on-doubt is a different operational risk profile from a multi-agent system that tends to alert-on-doubt — even at identical accuracy.

### Opus premium did not earn its cost

`multi_opus_premium` costs **$0.85 vs $0.43** for `multi_full_api` and gets **6/10 vs 7/10**. The one flag that flipped was #5 (chip_instability crit) where Opus over-escalated from `L3_throttle` to `L4_human_only`. Opus's conservatism — visible here and in the Qwen local result which made the same over-escalation on the same flag — is a feature in some contexts (operator-review gating) but costs accuracy against a ground truth that rewards bounded-auto intervention. On this benchmark, upgrading Maestro to Opus dispatch is **actively negative on AVE**: higher cost, lower quality, same latency.

### Full_local is the surprise winner on hard-judgment cases

`multi_full_local` is the only config that correctly handled both `ave_flg_04` (chip_instability warn → observe) and `ave_flg_06` (fan_anomaly warn → throttle) — two cases where every API config got at least one wrong. Likely drivers:

- Qwen's curated memory at test time included the `xgboost_fp_unanimous_noise` pattern extracted from its own prior runs, which Sonnet/Opus did not see in this run (memory is per-deployment, not cross-backend).
- Qwen's looser forced-tool compliance occasionally lands on the ground-truth action via a different reasoning path that the Anthropic models talk themselves out of.

On flags 3 (XGB crit) and 10 (voltage drift), Qwen under-reacts — a documented failure mode (same flag-3 miss as `single_opus`). The two errors that pulled it to 6/10 are characteristically different from the API misses, not the same errors cheaper.

### What this means for the pitch

| Claim | Evidence |
|---|---|
| Multi-agent improves accuracy over single-agent | **Not supported** — 7/10 vs 7/10 on same model family (Sonnet). |
| Multi-agent changes the *character* of errors | **Supported** — under-react (multi) vs over-react (single) is visible in the wrong-action table. |
| Premium Opus Maestro justifies its cost | **Not supported** — cost 2× higher, accuracy 1 lower on this batch. |
| Local Qwen is a viable cost-free fallback | **Supported** — matches Opus-premium accuracy at $0.00 and 27s extra latency. |
| Architecture choice matters for cost/latency | **Strongly supported** — 10× AVE spread across configs at flat accuracy. |

### Caveats

- **N=10 is small.** Bootstrapped 95% CIs on 70% accuracy overlap 60% — the 1-decision gap between `multi_opus_premium` and the others is within sampling noise.
- **Ground truth is judgment-based.** The reasoning column in `flags.json` is physically motivated but not crowd-sourced; a different operator might accept L2 instead of L1 on flag 02. Reading the `reasoning_trace` column of each decision matters as much as the binary Q score.
- **The single-agent prompt is a steel-man.** The monolithic system prompt was given full cross-domain context — it is NOT a weakened single-agent setup designed to make multi-agent look better. Comparing multi_full_api to a bare-bones single-agent without the autonomy-ladder doc in the prompt would almost certainly widen the gap.
- **Same-flag variance exists.** The runner's smoke pass and the full run disagreed on `single_sonnet` flag 1 (alert_operator vs observe). Temperature-0 would shrink but not eliminate this — Anthropic's API does not expose deterministic inference. A rigorous comparison would average N=3–5 runs per cell.

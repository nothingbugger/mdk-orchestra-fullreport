# 09 — Limitations and Known Issues

Honest accounting of what the current prototype does NOT do well, and what would be needed to close each gap.

---

## 9.1 Simulator limitations

### Synthetic data only

All ML training and Orchestra validation used simulator-generated telemetry. Real miners will have:

- Different noise distributions (manufacturing variation, age-dependent baselines)
- Firmware-specific telemetry artifacts (not modeled)
- Correlated failures (e.g., entire racks affected by shared cooling) that the simulator only partially captures
- More complex fault cascades (e.g., PSU degradation triggering hashboard stress triggering fan overwork)

**Mitigation path**: transfer learning or retraining on a limited real-miner dataset. The pattern discovery methodology (§03) should transfer; specific thresholds (e.g., Cohen's d = 12.7 on variance) may shift.

### No firmware behavior modeling

Different firmware versions (Bitmain stock, VNish, BraiinsOS, LuxOS) produce different telemetry. The simulator is agnostic to this. Deployments using aftermarket firmware may see different XGBoost false-positive rates.

### Fault scheduler artifacts

As documented in §3.2, `power_sag` was rejected as a pattern precisely because scheduler artifacts made it look separable. Continuous vigilance is needed: any apparent pattern should be cross-checked against scheduler correlations before promotion.

### Site-level coupling simplified

Real sites have rack-level thermal and electrical coupling. A fault in one miner can affect neighbors through shared exhaust, shared PDU, shared phase. The simulator models site-wide environmental coupling but not rack-level. Patterns like "rack-adjacent miners failing together" are not yet learnable from simulator data.

---

## 9.2 XGBoost model limitations

### `xgb_hashboard_failure` is standby

Dataset contained only 2 miners with hashboard_failure examples, insufficient for miner-wise OOD validation (leave-one-of-two-out has no statistical power). Model is trained and in repo but disabled.

**Activation requires**: 12h+ discovery run with ≥5 miners experiencing hashboard_failure. See [10_reproducibility.md](10_reproducibility.md) for the command.

### Short-window false positives

`xgb_hashrate` occasionally fires on 1-minute variance spikes that resolve within the 5-minute window. The canonical memory pattern `short_window_variance_false_positive` exists precisely to help Orchestra downgrade these. Without Orchestra, the deterministic layer would produce noticeable false-positive rate on noisy miners.

**Mitigation**: Orchestra's memory absorbs these; per §6.3 the selectivity 825→27 largely reflects this mitigation working.

### No model calibration

XGBoost probabilities are not formally calibrated. A `confidence: 0.73` prediction does not mean "73% of such cases are true positives". Calibration curves would improve severity mapping.

### No uncertainty quantification

The predictors emit point estimates. No uncertainty bands. A miner with unusual operating mode may get a high-confidence prediction that is actually extrapolation. Bayesian or ensemble variance would help.

---

## 9.3 Orchestra limitations

### Memory scale

Current memory files are flat markdown. At scale (hundreds of patterns), lookup becomes slow if patterns are scanned linearly. Acceptable for v0.1 with dozens of patterns; needs index/search for v1.0.

**Mitigation path**: add pattern indexing (e.g., by flag_type keyword) as memory grows. Stay in markdown, add JSON sidecar index.

### Reasoning trace drift

Over very long runs, specialists may produce subtly inconsistent reasoning across similar cases (LLMs are stochastic). Memory curation absorbs this over time, but early in deployment, two near-identical flags may receive slightly different reasoning.

**Mitigation path**: Maestro's integrated filtering (§5.2) is exactly the mechanism that catches this by recognizing near-duplicates and reusing prior decisions.

### Specialist blind spots

Each specialist is deliberately scoped. Cross-cutting issues that span domains (e.g., grid harmonics causing multi-miner voltage and hashrate patterns simultaneously) rely on Maestro to notice the correlation. Maestro sometimes misses cross-domain patterns that a truly integrated reasoner would catch.

**Mitigation path**: Maestro's cross-domain memory (`maestro_memory.md`) accumulates such patterns over time. Pattern like "site-wide voltage and hashrate degradation in same 15-min window" becomes learnable.

### Curation with small local models

Qwen 7B produced 1 pattern in 14 cycles of the 9h28m run. Opus produced 2 canonical patterns in 1 cycle of the 50-min run. The quality gap is real. Full-local deployments get less sophisticated curation.

**Mitigation path**: hybrid curation — run curation occasionally with higher-tier model while inference stays local. Cost-efficient.

### Human-in-the-loop not modeled

The autonomy ladder includes L4 human_review but the prototype has no actual human approval UI. In production, L4 queues would feed into an operator dashboard. This is a deployment detail, not a prototype gap — the architecture is ready.

---

## 9.4 KPI limitations

### AVE preliminary

Current AVE comparison smoke (§6.7) uses N=10, which is too small to distinguish configurations robustly. Ground truth labeling on 10 decisions is also subjective on borderline cases.

**Mitigation path**: N=100+ with production logs, formalized ground truth labeling protocol. Straightforward to execute; just takes more run time.

### P_miscalibration calibration is heuristic

The values (\$5 under-react, \$50 over-react, \$100+ severe error) are defensible estimates, not derived from specific operator data. Real deployment should tune these per site economics.

### TE per-miner subjective components

The "stability factor" and "cooling overhead multiplier" in TE are calibrated empirically against operating mode (see §02), but the exact weighting is a design choice. Different operators may want different weights.

**Mitigation path**: expose TE weighting as config; allow per-site customization.

---

## 9.5 Infrastructure limitations

### Full_local inference latency

141s median per decision on Qwen 7B Q4_K_M via LAN (§6.3). Acceptable for architectural demonstration, not for production operator-facing deployment.

**Mitigation path**: dedicated inference server (GPU enterprise-class) reduces latency 10-100× to sub-second levels. Infrastructure investment, not architectural gap.

### No cloud deployment option documented

Current deployment docs cover local (laptop-class) and full_api. Cloud deployment (AWS, GCP) is straightforward but not explicitly documented.

**Mitigation path**: add cloud deployment guide (Terraform scripts, ECS/Fargate for stateful components, Lambda for dashboard).

### Single-site only

The current system runs as a single-site monolith. Multi-site operators (common in mining industry) would need per-site installations and manual coordination. Federated memory sharing (§07) is the architectural answer but unimplemented.

---

## 9.6 Demonstration-ready status

What WORKS reliably:

- Simulator produces reproducible deterministic runs
- Rule engine + 2 XGBoost models produce calibrated flag streams
- Orchestra emits autonomy-ladder-valid decisions across all 4 levels
- Memory curation writes interpretable patterns
- Full_api, hybrid, full_local all validated end-to-end
- Backend abstraction supports Anthropic, Ollama/LMStudio, any OpenAI-compatible API
- Dashboard TUI integrates all layers
- Cost optimization V2 validated with 91% cost reduction
- 179/181 unit tests passing (2 skipped are infrastructure-gated)

What is DEMO-GRADE (works for showing, needs hardening for production):

- AVE scoring (framework validated, calibration preliminary)
- Memory curation quality (varies by curator model)
- `xgb_hashboard_failure` (trained, standby)
- Federated memory (design only, no code)
- Human-in-the-loop L4 queue (architecture ready, no UI)

What is EXPLICITLY OUT OF SCOPE:

- Real miner integration (simulator is ground truth for prototype)
- Multi-site coordination (federated memory design exists but unimplemented)
- Cloud deployment tooling (doable, not provided)
- Formal verification of safety properties (manual audit only)

## 9.7 Recommended next steps in priority order

1. **Real-miner telemetry dataset** — validate pattern discovery on non-synthetic data. Biggest single de-risker.
2. **AVE at N=500+** — produce conclusive comparison of `multi_full_api` vs `multi_full_local` vs `single_opus` etc.
3. **Dashboard L4 queue UI** — turn architectural readiness into actual human-in-the-loop deployment capability.
4. **xgb_hashboard_failure activation dataset** — 12h+ run with hashboard-heavy fault mix.
5. **Cloud deployment guide** — Terraform + ECS/Lambda reference.
6. **Federated memory v0.2** — export/import tooling before network layer.

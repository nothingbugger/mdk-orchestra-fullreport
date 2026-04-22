# Glossary

Technical terms used throughout this report.

---

## Agent and reasoning layer terms

**Agent** — An LLM-powered component with a defined role, identity prompt, output schema, and memory. In Orchestra: specialists + Maestro.

**Orchestra** — The multi-agent reasoning layer as a whole: filter + specialists + Maestro + executor interaction surface.

**Specialist** — Domain-focused agent (voltage, hashrate, environment, power). Returns assessment JSON.

**Maestro** — The coordinating agent. Performs integrated filtering, dispatch, synthesis, and curation.

**Dispatch table** — Data structure in `maestro.md` mapping each `flag_type` to its primary specialist, fallback specialist, and optional context specialist.

**Identity file** — `<agent>.md` markdown containing role, domain knowledge, reasoning procedure, output format. Editable, versioned, auditable.

**Memory file** — `<agent>_memory.md` markdown containing lesson-formatted patterns accumulated through curation cycles.

**Curation cycle** — Periodic operation where Maestro reads recent decision logs, distills patterns, and writes them to memory files.

**Pattern** — A lesson-formatted memory entry describing a recurring signature or rule, written in natural language, with confidence and provenance metadata.

---

## Autonomy ladder terms

**Autonomy ladder** — Four-level hierarchy (L1–L4) defining what Orchestra is allowed to autonomously do. Hard-enforced in both Maestro system prompt and Executor code.

**L1 Observe** — Log only. No action. Typical for noise or early-warning low-confidence signals.

**L2 Suggest** — `alert_operator` action. Operator sees a notification, Orchestra does not autonomously intervene.

**L3 Bounded Auto** — Reversible autonomous action (`throttle`, `migrate_workload`). Orchestra acts without human approval.

**L4 Human-Only** — Queue for approval. Includes `human_review` and `emergency_shutdown`. Orchestra cannot auto-execute.

---

## Deterministic layer terms

**Rule engine** — Hard-threshold detector emitting flags based on physics-backed thresholds (thermal_runaway, voltage_drift, fan_anomaly, hashrate_degradation).

**XGBoost predictor** — Supervised ML model trained on pre-fault/clean windows. Two active: `xgb_hashrate`, `xgb_chip_instability`.

**Flag** — Detection output. Envelope: `flag_type`, `severity`, `confidence`, `source_tool`, `evidence`.

**Flag type** — Classification of what the flag represents. Examples: `hashrate_degradation`, `chip_instability_precursor`, `voltage_drift`, `thermal_runaway`, `fan_anomaly`.

**Severity** — Flag urgency level: `info`, `warn`, `crit`.

---

## Telemetry and feature terms

**Tick** — One simulated timestep (5 simulated seconds).

**Telemetry record** — Per-miner-per-tick record with current state (hashrate, power, voltage, fan RPM, temperatures, operating mode, environmental context).

**Rolling window** — Time span over which features are aggregated (1m = 12 ticks, 5m = 60 ticks, 30m = 360 ticks).

**Feature** — Aggregated statistic over a rolling window. Examples: `hashrate_th_30m_std`, `power_w_5m_mean`, `temp_per_power`.

**HSI (Hashrate Stability Index)** — Derived rolling metric combining mean and variance to characterize stability. Healthy miners ≈ 100, degraded ≈ 70, critical < 70.

---

## KPI terms

**TE (True Efficiency)** — Primary operational KPI. Measures per-miner value creation vs fully loaded cost. 0–100 score + \$/MWh dual view.

**AVE (Agent Value-Efficiency)** — Secondary meta-KPI. Measures whether the agent reasoning layer is net-positive economically.

**Q** — In AVE: decision quality vs ground truth (0–1).

**V_net** — In AVE: net economic value (avoided damage − forgone revenue).

**T** — In AVE: response time from flag to action (seconds).

**C_agent** — In AVE: inference cost per decision.

**P_miscalibration** — In AVE: economic penalty for wrong decision. Calibrated per error class.

---

## Statistical terms

**Cohen's d** — Effect size metric. Measures standardized difference between two distributions. d > 0.8 large, d > 2.0 very large, d > 5.0 effectively disjoint.

**Silhouette score** — Cluster quality metric. Range [−1, +1]. Positive = coherent clusters, higher is better.

**OOD (Out-of-Distribution)** — In validation: test data drawn from distribution not seen in training. "Miner-wise OOD" = test miners never seen during training.

**AUC** — Area under ROC curve. Classification quality metric.

**PR-AUC** — Area under Precision-Recall curve. More informative than AUC for class-imbalanced classification.

---

## Backend and infrastructure terms

**Backend abstraction** — Code layer allowing Orchestra agents to be routed to different LLM providers via config.

**AnthropicBackend** — Native Anthropic API format backend.

**StandardLocalBackend** — Backend for local LLM servers using the industry-standard HTTP format (Ollama, LM Studio, llama.cpp server, vLLM).

**StandardAPIBackend** — Backend for remote providers using industry-standard format (OpenAI, Groq, Together, OpenRouter, DeepSeek, Mistral, etc.).

**Profile** — Pre-configured backend + model assignment for all Orchestra agents. Three shipped: `full_api`, `hybrid_economic`, `full_local`.

---

## Mining domain terms

**ASIC** — Application-Specific Integrated Circuit. The SHA-256 computation chip in Bitcoin miners.

**S19j Pro** — Bitmain miner class modeled in the simulator. 104 TH/s nominal, ~3250 W nominal.

**Hashrate** — Rate of SHA-256 hash attempts per second. TH/s = terahashes per second.

**Hashboard** — PCB containing multiple ASIC chips. S19 series has 3 hashboards × 76 chips = 228 chips total.

**PSU (Power Supply Unit)** — Converts AC grid power to DC rails feeding the hashboards. S19 class uses APW12 or GPW12 family.

**DVFS (Dynamic Voltage and Frequency Scaling)** — ASIC control mechanism that adjusts clock frequency and voltage in response to thermal or power constraints.

**Hashprice** — Market rate for hashrate contribution, in \$/TH/day. Varies with BTC price and network difficulty.

**Demand response** — Grid program where large electricity consumers (including mining sites) are paid to curtail consumption during peak demand.

**TOU (Time-of-Use) tariff** — Electricity pricing structure where rates vary by time of day (peak vs off-peak).

---

## Protocol and architecture terms

**Event stream** — Append-only JSONL record of system events. Immutable by design for audit trail.

**Executor** — Pure-Python (non-LLM) component that applies actions to miners. Safety boundary — enforces autonomy ladder independently of Maestro.

**Nostr** — Decentralized messaging protocol considered for federated memory sharing. Signed events distributed via relays. See [07_federated_memory.md](07_federated_memory.md).

**Tor hidden service** — Network-anonymized service endpoint for censorship-resistant relay hosting.

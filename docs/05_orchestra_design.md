# 05 — Orchestra — Architectural Details

## 5.1 Overview

Orchestra is the reasoning layer above the deterministic detection layer. It consumes flags emitted by rule engine + XGBoost predictors, decides whether each flag warrants full reasoning, consults specialists when needed, and emits operational actions via the autonomy ladder.

Two agent types:
- **Specialists** (4): domain-focused reasoners
- **Maestro** (1): coordinator with integrated filtering + dispatch + synthesis + curation

All agents run via backend abstraction supporting Anthropic native API, any OpenAI-compatible local server (Ollama, LM Studio, llama.cpp, vLLM), or any OpenAI-compatible API provider (Groq, Together, OpenRouter, etc.).

## 5.2 Filtering — integrated into Maestro

### Why integrated (not a separate agent)

Filtering is a fast pre-dispatch decision: is this flag already handled by recent memory? Is it a near-duplicate of a flag Maestro just decided on? Does the dispatch table require consultation at all?

A separate LLM agent would add one full model call to make this decision. Maestro already has memory access and dispatch table access. Folding filtering into Maestro's first-step logic is architecturally cleaner and costs fewer tokens.

### Filtering logic

On flag receipt:

1. Check deduplication window — has a decision been emitted for this `(miner_id, flag_type)` in the last 2 minutes? If yes and that decision is still governing, skip.
2. Query memory — is this flag type + evidence signature covered by a high-confidence memory pattern? If yes, emit pattern-based decision without specialist consultation.
3. Check dispatch table — if flag_type is whitelisted for auto-observe (e.g., routine noise types), emit `observe` directly.
4. Otherwise, trigger full dispatch.

### Measured effect

Complete API run: 38 flags → 28 formal decisions. Filtering absorbed 10 flags (~26% of input) without specialist consultation, saving ~\$1.20 in estimated inference cost.

## 5.3 Specialists

### Roster

| Specialist | Domain | Default model (full_api) |
|---|---|---|
| `voltage_specialist` | Electrical behavior | Sonnet 4.6 |
| `hashrate_specialist` | Hashrate trajectories | Sonnet 4.6 |
| `environment_specialist` | Site-level context | Haiku 4.5 |
| `power_specialist` | Infrastructure, grid, tariffs | Sonnet 4.6 |

### Identity files

Each specialist has a dense identity file (`<specialist>.md`, ~1,700 words each) containing:

- Domain knowledge (physics, nominal envelopes, field-repair literature)
- Failure modes with specific signatures and numeric thresholds
- How to reason (step-by-step logic for evaluating flags)
- Output format (strict JSON schema)
- What the specialist does NOT do (boundary with other domains)
- Memory consultation protocol
- References (sources grounding the knowledge)

Identity files are the "personality" of each specialist. Editable via `git diff` like any markdown. Changes are deterministic, auditable, reversible.

Full identity files shipped in `mdk-orchestra/agents/*.md`.

### Output schema

Specialists return strict JSON:

```json
{
  "agent": "voltage_specialist",
  "assessment": "real_signal | noise | inconclusive",
  "confidence": 0.0 to 1.0,
  "severity_estimate": "info | warn | crit",
  "reasoning": "brief natural-language explanation"
}
```

The `reasoning` field is critical — it's what Maestro aggregates for final decision, and what humans read in audit logs.

### Memory files

`<specialist>_memory.md` — lesson-formatted patterns specific to the specialist's domain.

Empty at repo initialization (examples provided in `examples/memories/`). Grown over curation cycles.

Memory format:

```markdown
## Pattern: <snake_case_name>
- First seen: 2026-04-21T20:47:47
- Last seen: 2026-04-21T20:47:47
- Occurrences: 1
- Signature: <concrete telemetry fingerprint>
- Learned action: <what to do>
- Confidence: 0.87
- Reasoning: <why this action makes sense>
- Example reference: <decision_id>
```

## 5.4 Maestro

### Role

- Filter (§5.2)
- Dispatch specialists per flag_type according to `maestro.md` dispatch table
- Aggregate specialist verdicts
- Decide action via autonomy ladder rules
- Periodically curate memories (read recent decisions, distill patterns, write to memory files)

### Dispatch table

Stored in `agents/maestro.md` as a structured table. For each `flag_type`, specifies:

- Primary specialist (always consulted)
- Fallback specialist (consulted only if primary returns `inconclusive`)
- Site-context specialist (consulted if rolling features show site-wide correlation)

Simplified excerpt:

| Flag type | Primary | Fallback | Context |
|---|---|---|---|
| `hashrate_degradation` | hashrate | voltage | environment |
| `chip_instability_precursor` | hashrate | voltage | — |
| `voltage_drift` | voltage | power | environment |
| `thermal_runaway` | environment | hashrate | — |
| `fan_anomaly` | environment | — | — |

Primary+fallback pattern (not always-all-specialists) is a deliberate cost optimization — see [08_cost_analysis.md](08_cost_analysis.md).

### Autonomy ladder

Hard-enforced by Maestro's system prompt + the deterministic Executor.

| Level | Behavior | Allowed actions |
|---|---|---|
| **L1 Observe** | Log only | `observe` |
| **L2 Suggest** | Notify operator | `alert_operator` |
| **L3 Bounded Auto** | Execute reversible action | `throttle`, `migrate_workload` |
| **L4 Human-Only** | Queue for approval | `human_review`, `emergency_shutdown` |

`emergency_shutdown` and `human_review` are **mandatory L4** — Maestro cannot auto-execute them. The Executor also refuses to auto-execute L4 actions, providing a second enforcement layer.

### Default model: Opus 4.7 with Sonnet dispatch tier

Cost-optimized model routing (see [08_cost_analysis.md](08_cost_analysis.md) V2):
- Dispatch step: Sonnet 4.6 (fast, cheap, sufficient for triage)
- Escalation synthesis: Opus 4.7 (only when reaching L3/L4 decisions)
- Curation: Opus 4.7 (reads recent logs, writes canonical patterns)

This model tiering cuts per-decision cost ~91% (validated, see [08_cost_analysis.md](08_cost_analysis.md)).

### Maestro memory (`maestro_memory.md`)

Cross-domain patterns — patterns Maestro notices that involve multi-specialist interactions:

- E.g., "when voltage specialist AND hashrate specialist BOTH return `noise` with confidence >0.80 on an XGBoost hashrate flag, the flag is almost certainly a false positive; do not escalate"

Memory pattern examples shipped in `examples/memories/sample_maestro_memory.md`.

## 5.5 Memory curation

### Trigger

Automatic every N decisions (configurable; default N=10 per cycle in experimental runs).

### Algorithm

1. Maestro reads the last N decisions from the decision log
2. Groups decisions by similarity (flag_type + outcome + reasoning theme)
3. For each coherent group ≥2 decisions, extracts the lesson
4. Writes new or updates existing `<agent>_memory.md` entries
5. If nothing sufficiently distilled, writes nothing (curation cycles are idempotent-safe)

### Lesson format

Maestro writes lesson-style memory, not event logs. Example from the actual initial ensemble run (Opus 4.7 curator):

> "When both required specialists unanimously call noise above 0.80 on an XGBoost hashrate flag, the deterministic model is almost certainly firing on short-window feature noise. Do NOT escalate to L2 despite crit severity — that path leads to alert fatigue."

This pattern survived into the canonical memory and was used (silently) in the complete API run, where 13 observe decisions at L1 include cases covered by this memory.

### Why markdown (not vector DB / fine-tuning)

- **Interpretable**: operators can read what the system has learned
- **Versioned**: git history shows learning trajectory
- **Reversible**: `git revert` undoes a bad curation cycle
- **Auditable**: every pattern has provenance (first_seen, occurrences, reasoning)
- **Portable**: see [07_federated_memory.md](07_federated_memory.md) — memory files are trivially shareable

Vector DBs and fine-tuning both optimize for scale at the cost of auditability. For a safety-critical decision layer (mining hardware costs money), interpretability is prioritized over scale.

## 5.6 Backend abstraction

### Supported backends

| Backend | For | Notes |
|---|---|---|
| `AnthropicBackend` | Anthropic API (Claude models) | Native Anthropic format |
| `StandardLocalBackend` | Local LLM servers | Configurable host; covers Ollama, LM Studio, llama.cpp server, vLLM |
| `StandardAPIBackend` | Remote OpenAI-compatible providers | Groq, Together, OpenRouter, DeepSeek, Mistral, Fireworks, etc. |

### Per-agent configuration

Each Orchestra agent (Maestro dispatch, Maestro escalation, Maestro curation, each specialist) can be independently routed via `config/llm_routing.yaml`:

```yaml
agents:
  maestro:
    dispatch: {backend: anthropic, model: claude-sonnet-4-6}
    escalation: {backend: anthropic, model: claude-opus-4-7}
  specialists:
    voltage: {backend: standard_local, host: http://localhost:11434, model: qwen2.5:7b-instruct}
    hashrate: {backend: standard_api, host: https://api.groq.com/openai, api_key_env: GROQ_API_KEY, model: llama-3.3-70b-versatile}
```

### Three pre-configured profiles

- `full_api`: all Anthropic (highest quality, validated canonical)
- `hybrid_economic`: specialists on local Qwen, Maestro on Anthropic
- `full_local`: everything local Qwen (zero-cost, full privacy)

### Extension

New backends (proprietary SDK-based, e.g., Gemini native) can be added via the template in `docs/extending.md` of the software repo. No core code changes required for OpenAI-compatible providers.

## 5.7 Safety architecture

### Executor is non-LLM

The Executor that actually applies actions to simulated miners is **pure Python**, not an LLM. This is the hard safety boundary: even if a prompt injection attack somehow reaches Maestro and makes it emit "shutdown everything", the Executor's autonomy ladder enforcement refuses L4 actions without human approval signal.

### Two enforcement layers

1. **Maestro's system prompt** enforces autonomy ladder rules
2. **Executor** independently validates action + autonomy_level before execution

Defense in depth: both layers must fail for an unauthorized destructive action to execute.

### Event streams are append-only

Every flag, decision, action is written to JSONL streams that are never modified. Full audit trail for post-hoc analysis or compliance.

## 5.8 Raw artifacts in repo

- `docs/maestro_md_canonical.md` — snapshot of production `maestro.md` used in canonical run
- `docs/dispatch_table.md` — full dispatch table
- `docs/specialist_md_versions/` — identity files versioned (voltage, hashrate, environment, power)

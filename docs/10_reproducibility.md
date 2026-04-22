# 10 — Reproducibility

Every experimental run in this report is reproducible from the `mdk-orchestra` software repository. This document provides exact commands, hardware requirements, and expected outcomes.

---

## 10.1 Prerequisites

### Hardware

| Requirement | Minimum | Recommended |
|---|---|---|
| CPU | 4 cores | 8+ cores (M-series Apple silicon ideal) |
| RAM | 8 GB | 16 GB |
| Storage | 2 GB free | 10 GB free |
| Network | Stable internet for API runs | LAN for local inference server |

### Software

- Python 3.11+
- pip
- git
- (optional) Ollama for local LLM runs
- (optional) LM Studio, llama.cpp server, or vLLM as alternative local inference servers

### API access (for full_api runs)

- `ANTHROPIC_API_KEY` environment variable set
- Anthropic account with sufficient quota (canonical run consumes ~\$3.31)

### Local LLM setup (for full_local runs)

```bash
# Install Ollama
curl -fsSL https://ollama.ai/install.sh | sh

# Pull Qwen 2.5 7B
ollama pull qwen2.5:7b-instruct

# Verify
ollama list
```

Other compatible servers (LM Studio, llama.cpp) work equivalently via the `StandardLocalBackend`. See [05_orchestra_design.md](05_orchestra_design.md) §5.6 and `mdk-orchestra/docs/extending.md` for configuration.

---

## 10.2 Installation

```bash
# Clone
git clone https://github.com/nothingbugger/mdk-orchestra
cd mdk-orchestra

# Install
pip install -e .

# Verify
mdk-orchestra --help
```

Expected output: CLI help listing commands `demo`, `simulator`, `run`, `train`, `discover`.

---

## 10.3 Reproducing experimental runs

### Run 1: Initial ensemble run

```bash
mdk-orchestra run \
  --profile full_api \
  --duration 50m \
  --seed 42 \
  --fault-mix balanced \
  --miners 50 \
  --memory empty \
  --run-id reproduce_initial_ensemble
```

Expected:
- Duration: ~50 min wall time
- Cost: ~\$3.03
- Flags: 15–20 (stochastic, bounded by seed)
- Decisions: 10–14
- Patterns written by curation: 1–3

Results in `runs/reproduce_initial_ensemble/`.

### Run 2: Complete API run (canonical)

```bash
# Step 1 — seed memory with initial run patterns
cp runs/reproduce_initial_ensemble/memory_final/*.md agents/
# Or use the sample memories as seed:
cp examples/memories/sample_maestro_memory.md agents/maestro_memory.md
cp examples/memories/sample_hashrate_memory.md agents/hashrate_memory.md

# Step 2 — canonical run
mdk-orchestra run \
  --profile full_api \
  --duration 120m-simulated \
  --acceleration 10x \
  --seed 42 \
  --fault-mix stretched \
  --miners 50 \
  --run-id reproduce_canonical
```

Expected:
- Wall time: ~12 min
- Cost: ~\$3.31 (can vary \$2.80–\$3.80)
- Flags: 35–40
- Decisions: 25–32
- Autonomy levels used: L1, L2, L3, L4 all represented
- Hero case: one L4 `human_review` on a chip_instability miner

Results in `runs/reproduce_canonical/`.

### Run 3: Complete LocalLLM run

```bash
# Step 1 — ensure Ollama running with Qwen 2.5 7B
ollama serve  # in separate terminal, if not already running

# Step 2 — configure LAN endpoint if using dedicated inference machine
export MDK_OLLAMA_HOST=http://<IP_of_inference_machine>:11434
# Or for localhost, no export needed (default is http://localhost:11434)

# Step 3 — run
unset ANTHROPIC_API_KEY  # ensures no API fallback
mdk-orchestra run \
  --profile full_local \
  --duration 240m-simulated \
  --acceleration 10x \
  --seed 42 \
  --fault-mix stretched \
  --miners 50 \
  --run-id reproduce_longrun
```

Expected:
- Wall time: 8–10 hours (Qwen 7B on M4 is the bottleneck)
- Cost: \$0.00 (API)
- Flags: 800–850
- Orchestra decisions: 25–30
- Baseline track decisions: same as flag count
- Latency: median ~120–160 seconds per decision

Results in `runs/reproduce_longrun/`.

---

## 10.4 Reproducing pattern discovery

```bash
mdk-orchestra discover \
  --hours 40 \
  --fault-mix balanced \
  --miners 50 \
  --seed 42 \
  --output-dir discovery_output/40h_balanced/
```

Expected:
- Wall time: ~4 hours (at 10× acceleration)
- Output: per-fault-type feature files, Cohen's d rankings, silhouette scores
- Disk: ~500 MB

Results analyzed via:

```bash
mdk-orchestra discover-analyze \
  --input-dir discovery_output/40h_balanced/ \
  --report pattern_discovery_results.json
```

Reproduces the Cohen's d and silhouette tables of [03_pattern_discovery.md](03_pattern_discovery.md).

---

## 10.5 Reproducing XGBoost training

```bash
mdk-orchestra train \
  --data-dir discovery_output/40h_balanced/ \
  --models hashrate,chip_instability,hashboard_failure \
  --seed 42 \
  --output-dir models_output/
```

Expected:
- Wall time: ~10 min
- Output files in `models_output/`:
  - `xgb_predictor.pkl`
  - `xgb_chip_instability.pkl`
  - `xgb_hashboard_failure.pkl`
  - `training_metrics.json`
  - `feature_importance.json`

Reproduces the metrics in [04_xgboost_training.md](04_xgboost_training.md) ± small variance due to numerical determinism in XGBoost parallel training.

---

## 10.6 Reproducing cost optimization smoke

```bash
mdk-orchestra run \
  --profile full_api-optimized \
  --duration 5m-simulated \
  --acceleration 10x \
  --seed 42 \
  --miners 10 \
  --fault-mix smoke_3scenario \
  --run-id reproduce_cost_smoke
```

Expected:
- Wall time: ~30 seconds
- Cost: <\$0.15 total
- Mean cost per decision: \~\$0.02

Validates the V2 cost optimization savings.

---

## 10.7 Reproducing AVE comparison smoke

```bash
python scripts/ave_comparison_runner.py \
  --configs full_api,full_local,single_sonnet,single_opus \
  --flags 10 \
  --seed 42 \
  --output reports/ave_comparison_reproduce.json
```

Expected:
- Wall time: ~5 min
- Cost: ~\$2.20 (for API-based configurations)
- Output: per-configuration AVE scores, accuracy, cost, latency

Reproduces §6.7 results directionally.

---

## 10.8 Determinism notes

### Seed-controlled

- Simulator fault injection (seed 42 = canonical)
- XGBoost training (within parallelism constraints)
- Environmental model (same seed produces same curves)

### NOT deterministic

- LLM responses (model providers may change versions, sampling is stochastic)
- Network-dependent timing (API latency varies)
- Local inference throughput (CPU/GPU load dependent)

Reasoning traces will not be byte-identical across re-runs. Decision outcomes should be qualitatively similar (same severity, usually same action, reasoning covering same physics).

### Reproducibility caveats

- Anthropic API model versions may change (prompt-caching, model updates). For byte-exact reproduction, pin model string.
- Ollama Qwen 2.5 versions may change. For byte-exact reproduction, pin ollama model hash.

---

## 10.9 Pre-built run tarballs

For reviewers who cannot or prefer not to re-run experiments, full tarballs of the canonical runs are in this repo under `data/tarballs/`:

| Tarball | Size | Contains |
|---|---:|---|
| `ab_initial_run.tar.gz` | ~15 MB | Initial ensemble run full output |
| `ab_canonical_run.tar.gz` | ~63 MB | Complete API canonical run full output |
| `ab_longrun.tar.gz` | ~176 MB | Complete LocalLLM 9h28m run full output |

Each tarball extracts to a `runs/` directory structure identical to what a fresh run produces.

---

## 10.10 Verification checklist

After reproducing a canonical run, verify:

1. **Autonomy ladder used at all 4 levels** (look for L1, L2, L3, L4 in `decisions.jsonl`)
2. **At least one hero-class trace** (Maestro reasoning trace >200 words with physical mechanism citation, autonomy ladder rule citation, and action justification)
3. **Cost within expected range** (\$2.80-\$3.80 for canonical)
4. **Flag-to-decision ratio ~1:1.4** (filtering working — not every flag triggers new decision)
5. **Memory file content** preserved or incremented (check `agents/*_memory.md` before/after)

If any of these fail, something is systematically different — check versions, API quota, configuration.

---

## 10.11 Support

- Repository issues: [github.com/nothingbugger/mdk-orchestra/issues](https://github.com/nothingbugger/mdk-orchestra/issues)
- Reproducibility questions welcome in discussions

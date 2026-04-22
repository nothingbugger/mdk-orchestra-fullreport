# mdk-orchestra-fullreport

**Full technical report and experimental data for the [mdk-orchestra](https://github.com/nothingbugger/mdk-orchestra) software.**

Plan B Academy DEV Track — Tether MDK Assignment 2026
Daniele Serlenga · `nothingbugger`

---

## What is this repo?

This repository is a **knowledge base**, not a software package. It contains:

- Full technical methodology (pattern discovery, XGBoost training details, agent architecture)
- Complete experimental results from three validation runs (flag streams, decisions, memory states)
- Cost analysis and optimization methodology
- Federated memory design (future direction)
- Verbatim reasoning traces from representative decisions
- Memory snapshots demonstrating the "learn by writing" principle
- Limitations and reproducibility guides

**This repo contains no runnable code.** For the software, install `mdk-orchestra`:
```bash
git clone https://github.com/nothingbugger/mdk-orchestra
cd mdk-orchestra && pip install -e . && mdk-orchestra demo
```

## Entry point

Start with [REPORT.md](REPORT.md) for an executive summary and navigation.

---

## Repository structure

```
mdk-orchestra-fullreport/
├── README.md                       # you are here
├── REPORT.md                       # executive summary + navigation
│
├── docs/
│   ├── 01_simulator.md             # simulator specification
│   ├── 02_kpi.md                   # TE and AVE KPI derivations
│   ├── 03_pattern_discovery.md     # pattern discovery methodology
│   ├── 04_xgboost_training.md      # XGBoost model training details
│   ├── 05_orchestra_design.md      # Orchestra architecture (filter + specialists + Maestro)
│   ├── 06_experiments.md           # full experimental results
│   ├── 07_federated_memory.md      # design for Nostr-over-Tor pattern sharing
│   ├── 08_cost_analysis.md         # cost breakdown + V2 optimization
│   ├── 09_limitations.md           # known limitations and gaps
│   ├── 10_reproducibility.md       # how to reproduce each run
│   └── glossary.md                 # technical terminology
│
├── traces/
│   └── m040_chip_instability_L4.md # hero reasoning trace (verbatim)
│
├── memory_snapshots/
│   ├── initial_run_canonical_patterns.md  # Opus-curated seed patterns
│   └── localllm_run_patterns.md           # Qwen 7B curation output
│
├── screenshots/
│   └── [dashboard screenshots from canonical run]
│
└── data/
    ├── ab_initial_run_*.jsonl      # initial ensemble run artifacts
    ├── ab_canonical_*.jsonl        # complete API canonical run artifacts
    ├── ab_longrun_*.jsonl          # complete LocalLLM run artifacts
    ├── pattern_discovery_*.json    # pattern discovery raw results
    ├── xgb_*_metrics.json          # XGBoost model metrics
    ├── cost_*.json                 # cost analysis raw data
    └── tarballs/                   # full run tarballs for offline reproduction
```

---

## Headline results

### Detection layer

| Model | AUC (OOD) | PR-AUC (OOD) | Status |
|---|---:|---:|---|
| `xgb_hashrate` | 0.928 | 0.837 | active |
| `xgb_chip_instability` | 0.834 | 0.819 | active |
| `xgb_hashboard_failure` | 0.745 | 0.805 | standby |

Top separability signal: `hashrate_th_30m_std` reaches Cohen's d ≈ 12.7 on chip_instability pre-fault vs clean — effectively disjoint distributions.

### Orchestra validation

| Run | Scope | Decisions | Autonomy levels used | Cost |
|---|---|---:|---|---:|
| Initial ensemble | Full-stack integration | 12 | L1 only | \$3.03 |
| Complete API (canonical) | Full autonomy ladder demonstration | 28 | **L1–L4 all** | \$3.31 |
| Complete LocalLLM | On-premise validation + baseline comparison | 27 | L1, L3, L4 | \$0.00 |

Key quantitative finding: in the 9h28m local run, the same 825 flags produced **825 actions** from the deterministic baseline vs **27 targeted decisions** from Orchestra. That 30× selectivity is the quantitative evidence of avoided alert fatigue.

### Cost optimization

Mean cost per decision:
- Pre-optimization: \$0.25
- Post-optimization (V2): \$0.021 — **−91%** reduction

Validated via API-real smoke test on 3 scenarios.

### Memory

Two canonical seed patterns written by Opus 4.7 during initial ensemble run are reproduced verbatim in [memory_snapshots/initial_run_canonical_patterns.md](memory_snapshots/initial_run_canonical_patterns.md).

Hero trace demonstrating full reasoning pipeline is in [traces/m040_chip_instability_L4.md](traces/m040_chip_instability_L4.md).

---

## Links

- Software repository: [github.com/nothingbugger/mdk-orchestra](https://github.com/nothingbugger/mdk-orchestra)
- This report: [github.com/nothingbugger/mdk-orchestra-fullreport](https://github.com/nothingbugger/mdk-orchestra-fullreport)

---

## License

Apache 2.0 · Copyright 2026 Daniele Serlenga.

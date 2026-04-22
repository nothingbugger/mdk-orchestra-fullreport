# MDK Orchestra — Full Technical Report

Technical deep-dive companion to [mdk-orchestra](https://github.com/nothingbugger/mdk-orchestra) software repo.
Author: Daniele Serlenga (`nothingbugger`) · Plan B Academy DEV Track · 2026

---

## Purpose of this repository

- Full numerical results from all experimental runs
- Methodology details (pattern discovery, XGBoost training, agent design)
- Reasoning traces, memory snapshots, cost analysis
- Known limitations and reproducibility notes
- Extended design for future directions

This repository contains **no runnable code**. For code, install `mdk-orchestra`.

---

## Navigation

| # | Section | File |
|---|---|---|
| 1 | Simulator — specification | [docs/01_simulator.md](docs/01_simulator.md) |
| 2 | KPI layer — TE & AVE derivation | [docs/02_kpi.md](docs/02_kpi.md) |
| 3 | Pattern discovery — full methodology | [docs/03_pattern_discovery.md](docs/03_pattern_discovery.md) |
| 4 | XGBoost models — training details | [docs/04_xgboost_training.md](docs/04_xgboost_training.md) |
| 5 | Orchestra — architectural details | [docs/05_orchestra_design.md](docs/05_orchestra_design.md) |
| 6 | Experimental validation — full data | [docs/06_experiments.md](docs/06_experiments.md) |
| 7 | Federated memory — design | [docs/07_federated_memory.md](docs/07_federated_memory.md) |
| 8 | Cost analysis | [docs/08_cost_analysis.md](docs/08_cost_analysis.md) |
| 9 | Limitations | [docs/09_limitations.md](docs/09_limitations.md) |
| 10 | Reproducibility | [docs/10_reproducibility.md](docs/10_reproducibility.md) |
| A | Canonical reasoning traces | [traces/](traces/) |
| B | Memory snapshots | [memory_snapshots/](memory_snapshots/) |
| C | Glossary | [docs/glossary.md](docs/glossary.md) |

---

## Headline results

### System stack

- Simulator (50 miners, S19j Pro class, 4 fault types, realistic environmental model)
- Deterministic detection: rule engine + 2 supervised XGBoost models
- Orchestra: 4 domain specialists + Maestro coordinator, backend-agnostic (Anthropic / local / hybrid)
- Executive layer: deterministic Python, autonomy ladder hard-enforced

### Deterministic layer

| Model | AUC (OOD) | PR-AUC (OOD) | Status |
|---|---:|---:|---|
| `xgb_hashrate` | 0.928 | 0.837 | active |
| `xgb_chip_instability` | 0.834 | 0.819 | active |
| `xgb_hashboard_failure` | 0.745 | 0.805 | standby (pending dataset) |

Top separability pattern: `hashrate_th_30m_std` reaches Cohen's d ≈ 12.7 on `chip_instability` pre-fault vs clean distributions.

### Orchestra

- Three experimental runs (initial ensemble, complete API, complete LocalLLM)
- Complete API run: 38 flags → 28 decisions, full autonomy ladder used (L1 × 13, L2 × 6, L3 × 5, L4 × 4)
- Complete LocalLLM comparative run: Orchestra 27 decisions vs deterministic baseline 825 on the same 825-flag stream
- Cost optimization: mean \$0.021/decision post-optimization (−91% vs pre-optimization, see [docs/08_cost_analysis.md](docs/08_cost_analysis.md))

### Memory layer

- Opus curation wrote 2 canonical seed patterns in the initial ensemble run
- Qwen 7B wrote 1 pattern in the 9h28m local run
- Memory format is human-readable markdown, git-trackable, reversible

---

## Companion repositories

- Software: [github.com/nothingbugger/mdk-orchestra](https://github.com/nothingbugger/mdk-orchestra)
- This report: [github.com/nothingbugger/mdk-orchestra-fullreport](https://github.com/nothingbugger/mdk-orchestra-fullreport)

---

## License

Apache 2.0 · Copyright 2026 Daniele Serlenga.

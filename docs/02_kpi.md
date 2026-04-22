# 02 — KPI Layer

Two complementary KPIs:

- **TE (True Efficiency)** — per-miner economic performance
- **HSI (Hashrate Stability Index)** — per-miner operational stress and stability

Both are computed continuously on every miner in the fleet, emitted to `runs/<timestamp>/kpis.jsonl`, and rendered in the dashboard alongside live telemetry. They are designed to be read together: TE answers *"is this miner producing economic value?"*, HSI answers *"is this miner safe to keep running as-is?"*.

---

## 2.1 True Efficiency (TE)

### Purpose

Measure per-miner economic efficiency relative to fully loaded operating cost under current market and fleet conditions. TE is the primary "is this miner productive" indicator.

### Formula

$$
\mathrm{TE}(t) = 100 \cdot \mathrm{percentile\_scale}\!\left(\log\left(\frac{V(t)}{C(t)}\right)\right)
$$

where:

- $V(t) = \mathrm{hashrate}_{\mathrm{real}}(t) \cdot \mathrm{hashprice}(t)$ is the instantaneous revenue rate
- $C(t) = \dfrac{P_{\mathrm{asic}}(t)}{1000} \cdot \mathrm{elec\_price}(t) \cdot 24\mathrm{h}$ is the daily-equivalent ASIC operating cost

### Fleet-percentile scaling

The log-ratio $\log(V/C)$ is scaled to the 0–100 range using the fleet's 5th and 95th percentile as anchors:

$$
\mathrm{TE}(t) = 100 \cdot \mathrm{clip}\!\left(\frac{\log r(t) - \log r_{p5}}{\log r_{p95} - \log r_{p5}},\ 0,\ 1\right)
$$

where $r(t) = V(t)/C(t)$ and $r_{p5}, r_{p95}$ are the 5th and 95th percentile of $r$ across the fleet in the current window.

This makes TE a **fleet-relative metric** — it adapts as market conditions and fleet composition change. A miner ranked at the 50th percentile has $\mathrm{TE} \approx 50$ by construction.

### Warmup behavior

During the initial warmup phase (before enough percentile samples are available), TE defaults to $50.0$ — a neutral score indicating "unknown state" rather than "perfect" or "zero". This is more honest than defaulting to 100 or 0.

### What TE captures

- **Economic ratio** (revenue vs cost)
- **Market conditions** through real-time `hashprice` and `elec_price`
- **Real hashrate** (not nameplate) — a miner at 80% of nominal is scored on what it actually produces
- **Fleet-relative context** — a miner that is "average for its fleet" has a neutral TE

### What TE does NOT capture

- Voltage drift, thermal stress, operating mode stress → these are all in HSI
- Hardware degradation history
- Cooling infrastructure cost at site level (modeled as part of ASIC power baseline)

This separation is deliberate: TE measures pure economic output, HSI measures operational health. Combined, they give a 2D view of miner state.

### Artifacts

- Emitted in `kpis.jsonl` as `te_components` and `te_value`
- Visualized in dashboard: per-miner card + fleet distribution + historical timeline
- Implementation: `ingest/kpi.py::compute_te`

---

## 2.2 Hashrate Stability Index (HSI)

### Purpose

Measure per-miner operational stress and stability. A miner can be TE-healthy (economically productive) while being HSI-degraded (operationally stressed) — this is the early signature of chip-level instability, voltage drift, or thermal creep.

### Formula

$$
\mathrm{HSI} = 100 \cdot \left(1 - \mathrm{clip}\!\left(\sum_i w_i \cdot s_i,\ 0,\ 1\right)\right)
$$

where $w_i \cdot s_i$ is a weighted stress component:

| Component | Weight | Formula | Captures |
|:---|:---:|:---|:---|
| $s_\mathrm{thermal}$ | $0.35$ | $\min(1, \max(0, (T_{\mathrm{site}}-20)/30)) + \phi_{\mathrm{hot}}$ | site cooling load + chip hot-time exposure |
| $s_\mathrm{voltage}$ | $0.25$ | $\lvert V - 12.0 \rvert / 12.0$ | chip voltage deviation from nominal envelope |
| $s_\mathrm{mode}$ | $0.15$ | lookup: eco=0.00, balanced=0.10, turbo=0.30 | operational stress from operating mode |
| $s_\mathrm{instability}$ | $0.25$ | $\sigma_{\mathrm{hashrate}} / \mu_{\mathrm{hashrate}}$ | rolling coefficient of variation of hashrate |

Weights sum to 1.00. The final stress sum is clipped to [0, 1] before conversion to 0–100 HSI score.

### Thermal component — hot-time fraction

The $\phi_{\mathrm{hot}}$ term is the fraction of time in the rolling window during which chip temperature exceeded 80 °C:

$$
\phi_{\mathrm{hot}} = \frac{\mathrm{time}(\mathrm{temp\_chip} > 80)}{\mathrm{total\ window\ time}}
$$

This captures cumulative thermal stress exposure, which two miners with same *mean* temperature can differ significantly in. A miner oscillating 65→90 °C has same mean as one stable at 78 °C, but accumulates real hardware stress the stable one does not.

### Severity bands

| HSI range | Interpretation |
|:---:|:---|
| > 65 | Healthy operating range |
| ≤ 65 | Warning — monitor closely |
| ≤ 40 | Imminent — action recommended |

These bands are used by the dashboard and by the `hashrate specialist` during flag evaluation as cross-check signals.

### What HSI captures

- **Thermal stress** — both ambient load and chip hot-time fraction
- **Voltage envelope** — deviation from nominal 12V rail
- **Operating mode penalty** — turbo is more stressful than eco
- **Hashrate variability** — volatile miners degrade faster

### Warmup behavior

Before sufficient rolling-window samples are available, HSI defaults to $50.0$ — same "unknown neutral" convention as TE.

### Artifacts

- Emitted in `kpis.jsonl` as `hsi_components` and `hsi_value`
- All 5 components (thermal, voltage, mode, instability, hot_time_frac) emitted per tick
- Visualized in dashboard: per-miner card + fleet distribution + historical timeline
- Implementation: `ingest/kpi.py::compute_hsi`

---

## 2.3 Joint interpretation

The two KPIs are orthogonal and are designed to be read together:

| TE | HSI | Interpretation |
|:---:|:---:|:---|
| high | high | Healthy, productive |
| high | low | Productive but stressed — monitor |
| low | high | Stable but inefficient — configuration review |
| low | low | Degraded — candidate for intervention |

Throughout the three experimental runs described in [06_experiments.md](06_experiments.md), TE and HSI were used as the primary site-health indicators. Their values confirmed that the simulated site operated in safe and productive regimes even while individual faults were injected, flagged, and processed by the agent layer.

---

## 2.4 Mapping to the assignment requirements

The assignment brief specified four variables that the efficiency metric should incorporate. The mapping to the TE+HSI suite:

| Assignment variable | Captured in | Mechanism |
|:---|:---:|:---|
| Cooling power | HSI | thermal component, site temperature and hot-time fraction |
| Chip voltage | HSI | voltage component, deviation from 12V nominal |
| Environmental conditions | TE + HSI | TE uses elec_price and hashprice; HSI uses site temperature |
| Operating mode | HSI | mode component, calibrated per-mode penalty |

The design choice to split across two KPIs (rather than pack all variables into a single TE formula) reflects the operational distinction between **economic productivity** (TE) and **operational safety** (HSI). Operators typically want to monitor both dimensions independently, and a miner can be off on one without being off on the other — which an integrated single-KPI would hide.

---

## 2.5 Design choices and limitations

### Calibration

Current coefficient values:

- TE fleet-percentile anchors: p5 and p95 (standard choice)
- HSI thermal weight: 0.35 (dominant component for ASIC mining)
- HSI voltage weight: 0.25
- HSI mode weight: 0.15 (less health-signal, more configuration choice)
- HSI instability weight: 0.25
- hot-time threshold: 80 °C (standard Bitmain envelope)
- HSI severity bands: 40 / 65 (empirical, derived from run distributions)

These are tunable knobs. Production deployment against a specific site would require recalibration against that site's baseline distributions.

### What is not captured

- **Hardware degradation history**: neither TE nor HSI look back beyond the rolling window. A miner that has been stressed for months shows the same current HSI as one just arrived from factory.
- **PUE at site level**: the 18% cooling overhead approximation used originally has been removed in favor of modeling cooling through HSI thermal stress. A more rigorous PUE-aware TE would need site-level infrastructure telemetry (HVAC, fans, pumps) that the simulator does not currently emit.
- **Pool-side revenue variance**: hashprice is used as market rate, but actual per-miner revenue can differ due to pool variance, lucky/unlucky blocks, fee distribution policies. Not modeled.

### Roadmap

- **PUE telemetry integration**: once site-infrastructure signals are available, extend cooling stress computation to use measured cooling power rather than approximations
- **Degradation history integration**: cumulative hot-time and voltage-off-envelope scores over long horizons (days/weeks) for lifecycle planning
- **Calibration against real-miner data**: replace heuristic weights with values learned on real-site operational history

---

## Raw artifacts

- Implementation: [`ingest/kpi.py`](https://github.com/nothingbugger/mdk-orchestra/blob/main/ingest/kpi.py) in companion software repo
- Unit tests: `tests/test_kpi.py` (6 new tests covering voltage/mode/temperature responses, 213 total passing)
- Sample data: `data/kpis_sample.jsonl` — 1 hour of KPI output from canonical run

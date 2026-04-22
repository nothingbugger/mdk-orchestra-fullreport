# 01 — Simulator Specification

## Purpose

Produce a continuous, deterministic, seed-reproducible telemetry stream that reflects realistic S19j Pro class miner behavior at a 50-miner mining site, including site-level environmental dynamics and injectable fault scenarios.

## Architecture

```
TickEngine (5s sim tick, 10× wall acceleration → 0.5s wall)
    ↓
FakeHardware (per-miner physics + environment)
    ↓
Telemetry emission (append-only JSONL stream)
    ↓
MinerHistory (rolling window aggregation)
    ↓
Feature processor (derived features at 1m/5m/30m windows)
```

All boundaries are append-only JSONL. Tick engine is monotonic; fault injection is scheduled at tick boundaries.

## Miner model

Each miner is a `FakeHardware` instance with the following state and derivation rules.

### Nominal specs (S19j Pro 104 TH/s class)

| Parameter | Value | Source |
|---|---:|---|
| Hashrate nominal | 104 TH/s | Bitmain spec |
| Power nominal | ~3250 W at full hashrate | Bitmain spec |
| Voltage rail nominal | 12.0 V ±5% | APW12 PSU spec |
| Fan channels | 4 | S19 series architecture |
| Fan RPM nominal | 3000 | mid-range of envelope |
| Chip count | 228 (76 × 3 boards) | S19j Pro architecture |
| Operating temperature envelope | 5–40 °C ambient | Bitmain spec |

### Derivation rules

- `hashrate_th`: nominal ± stochastic noise + fault-driven modulation
- `power_w`: proportional to actual hashrate × efficiency coefficient + PSU overhead
- `temp_chip_c`: function of ambient + power draw + cooling effectiveness
- `voltage_v`: nominal ± load-induced droop + fault-driven drift
- `fan_rpm[i]`: thermal feedback loop toward target chip temp
- `operating_mode`: chosen at miner spawn, influences nominal hashrate and voltage set-point

## Fault injection

Four fault types implemented as state machines with deterministic telemetry signatures:

| Fault | Mechanism | Primary telemetry impact |
|---|---|---|
| `chip_instability` | Simulated loss of voltage margin triggers DVFS oscillation between frequency steps | Elevated rolling std on hashrate, power, chip_temp (all coupled) |
| `hashboard_failure` | One of three hashboards marked offline | Hashrate step-drop to ~67% of nominal, power proportional drop, temp-per-power ratio shifts |
| `cooling_degradation` | Fan RPM drift downward + cooling effectiveness coefficient decay | Progressive chip temp rise at constant ambient, hashrate throttle |
| `power_sag` | PSU voltage set-point drift + load-dependent sag | Voltage excursions out of ±5% band, hashrate instability |

### Fault lifecycle

- Each fault has deterministic onset tick (scheduler-driven)
- Faults persist until externally remediated (throttle, shutdown) or simulation ends
- Unresolved fault at simulation end = hardware failure event

### Fault mix policies

- `default`: balanced uniform distribution across 4 fault types
- `stretched`: temporal spread so faults don't cluster (used in canonical runs)
- `clean`: no faults (baseline for pattern discovery clean-class telemetry)
- `heavy_[type]`: concentrated single-fault-type for training data generation

## Environmental model

### Site temperature (`env.site_temp_c`)

- Base temperature: 20 °C
- Diurnal sinusoid: ±5 °C oscillation with peak at simulated 14:00
- Applied to all 50 miners as shared signal (site-wide coupling)

### Humidity (`env.site_humidity_pct`)

- Base 45%, variation ±15% anti-correlated with site temp
- Relevant mostly for condensation risk signaling (not used in fault injection)

### Electricity price (`env.elec_price_usd_kwh`)

- Base \$0.065/kWh
- Time-of-day curve: off-peak \$0.04, peak hours 17:00-21:00 \$0.12
- Simulates industrial TOU tariff structure

### Hashprice (`env.hashprice_usd_per_th_day`)

- Base \$0.055/TH/day (calibrated on 2025 post-halving reality)
- Mild stochastic variation around base

## Telemetry emission

### Cadence

- 5 simulated seconds per tick
- Running at 10× wall acceleration: 0.5 wall seconds per tick
- Each tick produces 50 telemetry records (one per miner)

### Schema

Full envelope emitted every tick:

```json
{
  "miner_id": "m040",
  "miner_model": "S19j_Pro",
  "timestamp": "2026-04-22T08:07:28Z",
  "uptime_s": 14400,
  "hashrate_th": 84.3,
  "hashrate_expected_th": 104.0,
  "temp_chip_c": 65.2,
  "temp_amb_c": 26.1,
  "power_w": 2814,
  "voltage_v": 11.98,
  "fan_rpm": [2954, 2968, 2941, 2979],
  "operating_mode": "balanced",
  "env": {
    "site_temp_c": 23.9,
    "site_humidity_pct": 45,
    "elec_price_usd_kwh": 0.0656,
    "hashprice_usd_per_th_day": 0.0524
  },
  "fault_injected": "chip_instability"
}
```

`fault_injected` is emitted only in training data; suppressed in production-like runs.

## Rolling feature computation

### Windows maintained by `MinerHistory`

- 1 minute (12 ticks)
- 5 minutes (60 ticks)
- 30 minutes (360 ticks)

### Derived features

Per window, for each of `hashrate_th`, `power_w`, `temp_chip_c`, `fan_rpm_mean`, `voltage_v`:

- Mean
- Standard deviation
- Min, max
- Slope (linear fit over window)

Plus cross-variable derived features:

- `temp_per_power` = `temp_chip_c` / `power_w` (thermal efficiency per unit load)
- `hsi` (Hashrate Stability Index): rolling combination of mean/variance anchored to nominal
- `power_deficit_pct` = (expected - actual) / expected for `power_w`

Total feature count per miner per tick: ~60 features available for downstream consumption.

## Calibration decisions

| Choice | Reason |
|---|---|
| 50 miners | Large enough for site-wide dynamics, small enough to fit on laptop-class hardware |
| 10× acceleration | Demo tempo + enables multi-hour runs in reasonable wall time |
| S19j Pro class | Most common modern ASIC class, well-documented specs |
| Seed 42 | Reproducibility across runs |
| 5s telemetry cadence | Balance between temporal resolution and data volume |
| 30m max window | Captures most pre-fault patterns while keeping feature dimensionality bounded |

## References

- Bitmain S19j Pro specifications (input voltage 200-240 V, 20 A AC input)
- APW12 PSU datasheet (12 V DC rail nominal)
- Hashrate Index, EcoCooling documentation on mining site operational patterns
- Bitmain operating temperature envelope (5-40 °C)

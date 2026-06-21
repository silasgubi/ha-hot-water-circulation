<div align="center">

# 💧 Hot Water Circulation Intelligence

**Enterprise-grade pump monitoring. $50 smart plug. 100% native Home Assistant.**

*Statistical calibration · Derivative-based inrush filtering · Predictive wear detection*

[![Version](https://img.shields.io/badge/Version-3.5.2-success.svg)](https://github.com/silasgubi/ha-hot-water-circulation/releases/tag/v3.5.2)
[![Status](https://img.shields.io/badge/Status-Production-brightgreen.svg)](https://github.com/silasgubi/ha-hot-water-circulation)
[![Home Assistant](https://img.shields.io/badge/Home%20Assistant-2024.12+-blue.svg)](https://www.home-assistant.io/)
[![License](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)
[![Made in Brazil](https://img.shields.io/badge/Made%20in-Brazil%20🇧🇷-green.svg)](https://github.com/silasgubi)

[What makes this different](#-what-makes-this-different) · [How it works](#-how-it-works) · [The data story](#-the-data-story) · [Quick Start](#-quick-start) · [Docs](#-docs)

</div>

---

## ✨ What makes this different

Most pump automations just switch a relay on and off. This one treats the pump like a monitored industrial asset — using only native Home Assistant sensors, no custom integrations.

| | Typical approach | This project |
|--|-----------------|-------------|
| **Thresholds** | Nameplate / guesswork | 1,826 real measurements |
| **Current detection** | High/low threshold | Rate-of-change (dI/dt) |
| **Startup spike** | Fixed delay | Derivative filter (P97 calibrated) |
| **Wear detection** | None | 7d vs 30d statistical baseline |
| **Failure prediction** | Reacts after failure | Detects wear 2–4 weeks early |
| **HA dependencies** | Custom components | Statistics + Derivative (built-in) |

---

## ⚙️ How it works

```
Tap opened → 3s confirmation delay → Pump ON
  │
  ├─ t = 0–10s   inrush window (derivative filter active, no alerts)
  │
  └─ t > 10s     monitoring active
        ├─ current in 0.303–0.323A          → ✅ normal
        ├─ |dI/dt| > 0.025 A/s             → ⚠️  rapid change alert
        ├─ 7d avg > 30d avg × 1.03         → 📈 wear trend alert
        └─ current > 0.388A                → 🔴 emergency shutoff + TTS

Tap closed → 20s delay → Pump OFF
Safety: automatic shutoff after 30min (stuck valve protection)
```

### 3-layer sensor architecture

```
LAYER 1 — RAW  (sensors.yaml)
  sensor.bomba_taxa_mudanca_corrente     derivative, 5s window, A/s
  sensor.bomba_corrente_media_24h        statistics, 24h mean
  sensor.bomba_corrente_media_7d         statistics,  7d mean
  sensor.bomba_corrente_media_30d        statistics, 30d mean  ← normality baseline

LAYER 2 — DETECTION  (template_sensors.yaml)
  binary_sensor.bomba_corrente_estabilizada      |dI/dt| < 0.005 for 2s  → inrush passed
  binary_sensor.bomba_mudanca_rapida_corrente    |dI/dt| > 0.025 + 10s filter + hysteresis
  binary_sensor.bomba_desgaste_emergente         7d > 30d × 1.03

LAYER 3 — ALERTS  (template_sensors.yaml)
  binary_sensor.bomba_corrente_anormal   (stabilized AND out-of-range) OR rapid-change
  binary_sensor.bomba_corrente_critica   > 0.388A → immediate shutoff
```

---

## 📊 The data story

> The pump nameplate says **50W**. The real draw is **68W**. Calibrating from specs would have made every alert wrong.

We collected **1,826 current samples** and **836 derivative samples** over 30 days in December 2024, then derived all thresholds from the actual distribution — not theory.

### How thresholds were calculated

```
CURRENT DISTRIBUTION (pump running, 1292 samples with I > 0.1A)

  P5  = 0.303A  →  normal_min   (excludes low-end outliers)
  P50 = 0.310A  →  typical operation
  P95 = 0.323A  →  normal_max   (excludes high-end outliers)
  Max = 0.330A
        × 1.15  →  0.388A       critical threshold (15% safety margin)

DERIVATIVE DISTRIBUTION (836 samples, |dI/dt| during operation)

  Median = 0.0001 A/s  →  stable operation (near zero)
  P95    = 0.0618 A/s  →  normal inrush / shutoff transient
  P97    = ~0.025 A/s  →  threshold for "rapid change" alert
                           (sits between normal and anomalous)
```

The P5/P95 percentile approach excludes outliers that min/max would include. The P97 derivative threshold means **only the top 3% of rate-of-change events trigger an alert** — real anomalies, not normal startup.

### What it can predict

| Signal | What it detects | How early |
|--------|----------------|-----------|
| `7d avg > 30d avg × 1.03` | Progressive motor wear (bearings, impeller) | 2–4 weeks before failure |
| `\|dI/dt\| > 0.025 A/s` | Acute problem: blockage, jam, debris | Seconds |
| `I > 0.388A` | Overload / imminent failure | Immediate |

The **30-day baseline** is the key insight: as a motor wears, it draws progressively more current. By comparing the recent 7-day average against a 30-day baseline, the system detects a *trend* — not just a threshold breach — before the pump shows visible symptoms.

> Recalibrate with at least 100 samples from your own pump. Every pump is different.

---

## 🚀 Quick Start

### Prerequisites
- Home Assistant 2024.12+
- Sonoff Mini (eWeLink) — flow detection via valve entity
- Tuya TS011F (Zigbee) — pump switch + current sensor
- Circulation pump ~50W

### Install

**1. Create helpers** (Configuration → Helpers):
```yaml
input_number.pump_current_normal_min: 0.303
input_number.pump_current_normal_max: 0.323
input_number.pump_current_critical_max: 0.388
input_number.pump_activation_delay_seconds: 3
input_number.pump_deactivation_delay_seconds: 20
input_number.pump_timeout_minutes: 30

timer.pump_activation_delay: 00:00:03
timer.pump_deactivation_delay: 00:00:20
timer.pump_safety_timeout: 00:30:00
```

**2. Copy config files:**
```bash
cp config/sensors.yaml           /config/sensors.yaml
cp config/template_sensors.yaml  /config/template_sensors.yaml
cp config/automations_bomba.yaml /config/automations.yaml
cp config/scripts_bomba.yaml     /config/scripts.yaml
```

**3. Restart and validate:**
```bash
ha core check && ha core restart
ha state get sensor.bomba_taxa_mudanca_corrente      # should return A/s value
ha state get binary_sensor.bomba_corrente_estabilizada
```

**4. Add the dashboard** — Settings → Dashboards → Edit → Add View (Sidebar) → paste `lovelace_bomba_sidebar.yaml` as Raw YAML.

> Entity IDs and dashboard labels are in Brazilian Portuguese. You'll need to adapt them to your own HA entity names.

---

## 📂 Docs

| File | Description |
|------|-------------|
| `config/sensors.yaml` | Derivative + Statistics sensors (Layer 1) |
| `config/template_sensors.yaml` | Binary sensors — detection logic (Layers 2–3) |
| `config/automations_bomba.yaml` | Control automations and alerts |
| `config/scripts_bomba.yaml` | Test, reset, report scripts |
| `config/lovelace_bomba_sidebar.yaml` | Dashboard — Mushroom Cards ⭐ |
| `config/lovelace_bomba_sidebar_native.yaml` | Dashboard — native HA cards (no deps) |
| [docs/decisions/0001-current-vs-power.md](docs/decisions/0001-current-vs-power.md) | ADR: why current over power |
| [docs/decisions/0002-derivative-inrush-filtering.md](docs/decisions/0002-derivative-inrush-filtering.md) | ADR: anti-inrush filter design |
| [docs/analysis/ANALISE_CORRENTE_20241218.md](docs/analysis/ANALISE_CORRENTE_20241218.md) | Full calibration data analysis |
| [docs/lessons-learned.md](docs/lessons-learned.md) | Technical lessons learned |
| [CHANGELOG.md](CHANGELOG.md) | Version history |

---

## 🔧 Troubleshooting

| Problem | Fix |
|---------|-----|
| Inrush triggers false alert | Upgrade to v3.5.2 |
| Sonoff stuck at "open" | 3s confirmation delay already handles this |
| Derivative shows A/min | Ensure `unit_time: s` in `sensors.yaml` |
| Wear alert fires immediately | Statistics sensors need 7–30 days of data to stabilize |

---

## 🤝 Contributing

1. Fork → branch → PR
2. Use real calibration data — never placeholder values
3. Document threshold decisions in `docs/decisions/`
4. Update CHANGELOG

---

## 📝 License

MIT — see [LICENSE](LICENSE).

---

<div align="center">

*Don't guess thresholds. Measure them.*

**[Silas Gubitoso](https://github.com/silasgubi)** · São Paulo, Brazil 🇧🇷

[![GitHub stars](https://img.shields.io/github/stars/silasgubi/ha-hot-water-circulation?style=social)](https://github.com/silasgubi/ha-hot-water-circulation)

</div>

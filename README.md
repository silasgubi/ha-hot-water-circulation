# 💧 Hot Water Circulation Intelligence

Smart hot water circulation pump automation for Home Assistant — detects flow via Sonoff Mini, controls a Tuya TS011F smart plug, and monitors current with inrush filtering.

[![Version](https://img.shields.io/badge/Version-3.5.2-success.svg)](https://github.com/silasgubi/ha-hot-water-circulation/releases/tag/v3.5.2)
[![Status](https://img.shields.io/badge/Status-Production-brightgreen.svg)](https://github.com/silasgubi/ha-hot-water-circulation)
[![License](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)

---

## Prerequisites

- Home Assistant 2024.12+
- Sonoff Mini (eWeLink) — flow detection
- Tuya TS011F (Zigbee) — pump control + current sensing
- Circulation pump ~50W / 0.31A

## Installation

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

**2. Copy config files** to your Home Assistant `/config/`:

```bash
cp config/sensors.yaml /config/sensors.yaml
cp config/template_sensors.yaml /config/template_sensors.yaml
cp config/automations_bomba.yaml /config/automations.yaml
cp config/scripts_bomba.yaml /config/scripts.yaml
```

**3. Restart:**

```bash
ha core check && ha core restart
```

**4. Install dashboard** — Settings → Dashboards → Edit → Add View (Sidebar type) → paste `lovelace_bomba_sidebar.yaml` as Raw YAML.

> **Note:** Entity IDs and dashboard labels are in Brazilian Portuguese. Adapt them to match your own HA setup.

---

## How It Works

```
Valve opens → 3s delay → Pump ON
  ├─ 0–10s:  inrush ignored (dI/dt filter active)
  ├─ >10s:   current monitored (0.303–0.323A normal)
  │             ├─ rapid change (dI/dt > 0.025 A/s) → alert
  │             ├─ 7d avg > 30d avg × 1.03           → wear alert
  │             └─ current > 0.388A                  → emergency shutoff + TTS
Valve closes → 20s delay → Pump OFF
Safety: 30min timeout if valve stuck
```

---

## Calibrated Thresholds

Based on **1826 real samples** (December 2024, 50W pump):

| Parameter | Value | Source |
|-----------|-------|--------|
| Normal min | 0.303A | P5 percentile |
| Normal max | 0.323A | P95 percentile |
| Critical | 0.388A | Max × 1.15 |
| Stabilization | \|dI/dt\| < 0.005 A/s | — |
| Rapid change | \|dI/dt\| > 0.025 A/s | P97 percentile |
| Wear trend | 7d avg > 30d avg × 1.03 | — |

> Recalibrate with at least 100 samples from your own pump before enabling alerts.

---

## File Reference

| File | Description |
|------|-------------|
| `config/sensors.yaml` | Derivative + Statistics raw sensors |
| `config/template_sensors.yaml` | Binary sensors — detection logic |
| `config/automations_bomba.yaml` | Control automations + alerts |
| `config/scripts_bomba.yaml` | Test, reset, and report scripts |
| `config/lovelace_bomba_sidebar.yaml` | Dashboard (Mushroom Cards) ⭐ |
| `config/lovelace_bomba_sidebar_native.yaml` | Dashboard (native HA cards, no deps) |

Docs:
- [docs/decisions/0001-current-vs-power.md](docs/decisions/0001-current-vs-power.md) — why current over power
- [docs/decisions/0002-derivative-inrush-filtering.md](docs/decisions/0002-derivative-inrush-filtering.md) — anti-inrush filter design
- [docs/lessons-learned.md](docs/lessons-learned.md) — technical lessons
- [CHANGELOG.md](CHANGELOG.md) — version history

---

## Troubleshooting

| Problem | Fix |
|---------|-----|
| Inrush triggers false alert | Upgrade to v3.5.2 |
| Sonoff stuck at "open" | 3s confirmation delay already handles this |
| Derivative in A/min (wrong) | Ensure `unit_time: s` in `sensors.yaml` |

```bash
# Validate after install
ha state get sensor.bomba_taxa_mudanca_corrente       # expect A/s value
ha state get binary_sensor.bomba_corrente_estabilizada
ha state get binary_sensor.bomba_mudanca_rapida_corrente
```

---

## License

MIT — see [LICENSE](LICENSE).

**Made with ❤️ in São Paulo, Brazil 🇧🇷**

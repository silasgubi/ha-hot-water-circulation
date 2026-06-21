# Dashboard Installation Guide — Hot Water Circulation v3.5.2

Two dashboard variants are available as sidebar views.

## Options

### `lovelace_bomba_sidebar.yaml` ⭐ Recommended
Modern UI with Mushroom Cards and Mini Graph Card.
**Requires:** HACS + [Mushroom](https://github.com/piitaya/lovelace-mushroom) + [Mini Graph Card](https://github.com/kalkih/mini-graph-card)

### `lovelace_bomba_sidebar_native.yaml`
Uses only built-in HA cards. No dependencies.

## Installation

**Via UI (recommended):**
1. Settings → Dashboards → Edit Dashboard
2. + ADD VIEW
3. Title: anything you like | Icon: `mdi:pump` | Type: **Sidebar**
4. Edit in Raw YAML → paste the file content
5. Save

**Via `configuration.yaml`:**
```yaml
lovelace:
  mode: storage
  dashboards:
    hot-water-pump:
      mode: yaml
      title: Hot Water Pump
      icon: mdi:pump
      show_in_sidebar: true
      filename: dashboards/bomba.yaml
```

## Required Entities

Ensure these exist before installing the dashboard:

- `valve.hot_water` (Sonoff Mini)
- `switch.bomba_de_circulacao_de_agua_quente` (Tuya TS011F)
- `sensor.bomba_de_circulacao_de_agua_quente_corrente`
- `sensor.bomba_taxa_mudanca_corrente` (Derivative v3.5)
- All v3.5 binary sensors (`estabilizada`, `mudanca_rapida`, etc.)
- 6× `input_number` helpers (thresholds, delays, timeout)
- 3× `input_boolean` (override, manual, alerts)
- 3× `timer` (activation, deactivation, timeout)

## Dashboard Sections

Both variants include:

1. **Status** — system overview
2. **Current monitoring** — gauge, rate, averages
3. **Diagnostics v3.5** — real-time smart analysis
4. **History** — 2h/24h graphs
5. **Controls** — override, manual, test, reset
6. **Hardware** — devices and sensors
7. **Settings** — thresholds, delays, timeouts
8. **DEBUG** — timers, automations, troubleshooting

## Comparison

| Feature | Mushroom | Native |
|---------|----------|--------|
| Current gauge | ✅ | ✅ |
| 2h/24h history | ✅ | ✅ |
| Smart diagnostics | ✅ | ✅ |
| Mini Graph Card | ✅ | ❌ |
| Requires HACS | ✅ | ❌ |

## Troubleshooting

| Problem | Fix |
|---------|-----|
| Blank cards | Install Mushroom via HACS, or switch to the native variant |
| Entities "unavailable" | Check `sensors.yaml` and `template_sensors.yaml` |
| Dashboard not in sidebar | Set view type to "Sidebar" |

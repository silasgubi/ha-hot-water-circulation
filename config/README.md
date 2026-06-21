# Config Files Reference — Hot Water Circulation v3.5.2

## Files

| File | Description |
|------|-------------|
| `sensors.yaml` | Derivative + Statistics raw sensors (Layer 1) |
| `template_sensors.yaml` | Binary sensors — detection logic (Layers 2 & 3) |
| `automations_bomba.yaml` | Automations — control and alerts |
| `scripts_bomba.yaml` | Scripts: system test, emergency reset, detailed report |
| `lovelace_bomba_sidebar.yaml` | Dashboard with Mushroom Cards (recommended) |
| `lovelace_bomba_sidebar_native.yaml` | Dashboard with native HA cards (no dependencies) |

## Where to Find Things

### Layer 1 — Raw Sensors (`sensors.yaml`)
- Derivative sensor: lines 50–63
- Statistics 24h/7d/30d: lines 69–111
- History stats runtime: lines 41–48

### Layer 2 — Smart Detection (`template_sensors.yaml`)
- `bomba_corrente_estabilizada` — stabilization guard
- `bomba_mudanca_rapida_corrente` — rapid change detection
- `bomba_desgaste_emergente` — wear trend detection

### Layer 3 — Combined Alerts (`template_sensors.yaml`)
- `bomba_corrente_anormal` — combined abnormal alert
- `bomba_corrente_critica` — emergency (>0.388A)

### Automations (`automations_bomba.yaml`)
- Main control: line 7
- Safety timeout: line 140
- Manual override: line 200
- Emergency shutoff: line 260
- Progressive wear: line 320
- Rapid change alert: line 420
- Wear trend alert: line 475

### Scripts (`scripts_bomba.yaml`)
- `pump_system_test`: line 1
- `pump_emergency_reset`: line 75
- `pump_detailed_report`: line 130

## Deployment

Copy all files to your Home Assistant `/config/` directory:

```bash
cp sensors.yaml /config/sensors.yaml
cp template_sensors.yaml /config/template_sensors.yaml
cp automations_bomba.yaml /config/automations.yaml
cp scripts_bomba.yaml /config/scripts.yaml
```

Then restart HA and install a dashboard (see `README_DASHBOARD.md`).

## Validation

In Developer Tools → Template:

```yaml
Derivative:       {{ states('sensor.bomba_taxa_mudanca_corrente') }} A/s
Stabilized:       {{ states('binary_sensor.bomba_corrente_estabilizada') }}
Rapid change:     {{ states('binary_sensor.bomba_mudanca_rapida_corrente') }}
Wear trend:       {{ states('binary_sensor.bomba_desgaste_emergente') }}
Abnormal current: {{ states('binary_sensor.bomba_corrente_anormal') }}
```

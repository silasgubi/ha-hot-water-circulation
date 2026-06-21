# Dashboard Installation Guide — Hot Water Circulation v3.5.2

Two dashboard variants are available as sidebar views in Home Assistant.

| Variant | File | Dependencies |
|---------|------|-------------|
| ⭐ Recommended | `lovelace_bomba_sidebar.yaml` | HACS + Mushroom Cards + Mini Graph Card |
| No deps | `lovelace_bomba_sidebar_native.yaml` | None (native HA cards only) |

---

## Method 1 — Via UI (recommended)

### Step 1: Install custom cards (Mushroom variant only)

If using the Mushroom variant, install via HACS → Frontend:
- **Mushroom Cards**
- **Mini Graph Card** (optional — better graphs)
- **Card Mod** (optional — styling)

Restart Home Assistant after installing. Skip this step if using the native variant.

### Step 2: Create a new view

1. Settings → Dashboards → select your main dashboard
2. Click ⋮ → **Edit Dashboard**
3. Click **+ ADD VIEW**

### Step 3: Configure the view

- **Title:** anything you like (e.g. "Hot Water Pump")
- **Icon:** `mdi:pump`
- **Path:** `hot-water-pump`
- **Type:** Sidebar

Save.

### Step 4: Paste the YAML

1. Click ⋮ on the new view → **Edit in Raw YAML**
2. Delete all existing content
3. Paste the full contents of your chosen file:
   - Mushroom: `config/lovelace_bomba_sidebar.yaml`
   - Native: `config/lovelace_bomba_sidebar_native.yaml`
4. Save → Done

### Step 5: Verify

The new tab should appear in the sidebar. Check that all sensors display data.

---

## Method 2 — Via configuration.yaml

More advanced — allows Git versioning of the dashboard.

### Step 1: Copy the file

```bash
# Mushroom variant
cp config/lovelace_bomba_sidebar.yaml /config/dashboards/pump.yaml

# Native variant
cp config/lovelace_bomba_sidebar_native.yaml /config/dashboards/pump.yaml
```

### Step 2: Edit configuration.yaml

```yaml
lovelace:
  mode: storage
  dashboards:
    hot-water-pump:
      mode: yaml
      title: Hot Water Pump
      icon: mdi:pump
      show_in_sidebar: true
      filename: dashboards/pump.yaml
```

### Step 3: Validate and restart

```bash
ha core check
ha core restart
```

---

## Prerequisites

Make sure all v3.5 components are configured before installing the dashboard.

### Input numbers
```yaml
input_number:
  pump_current_normal_min:
    min: 0
    max: 1
    step: 0.001
    initial: 0.303
  pump_current_normal_max:
    min: 0
    max: 1
    step: 0.001
    initial: 0.323
  pump_current_critical_max:
    min: 0
    max: 1
    step: 0.001
    initial: 0.388
  pump_activation_delay_seconds:
    min: 0
    max: 60
    step: 1
    initial: 3
  pump_deactivation_delay_seconds:
    min: 0
    max: 300
    step: 5
    initial: 20
  pump_timeout_minutes:
    min: 0
    max: 120
    step: 5
    initial: 30
```

### Input booleans
```yaml
input_boolean:
  pump_manual_override:
    icon: mdi:shield-off
  pump_manual_control:
    icon: mdi:hand-back-right
  pump_alerts_enabled:
    icon: mdi:volume-high
  pump_debug_mode:       # optional — controls DEBUG section visibility
    icon: mdi:bug
```

### Timers
```yaml
timer:
  pump_activation_delay:
    duration: "00:00:03"
  pump_deactivation_delay:
    duration: "00:00:20"
  pump_safety_timeout:
    duration: "00:30:00"
```

### Required sensors

Verify these exist in Developer Tools → States before installing the dashboard:

```
sensor.bomba_taxa_mudanca_corrente       (Derivative)
sensor.bomba_corrente_media_24h          (Statistics)
sensor.bomba_corrente_media_7d           (Statistics)
sensor.bomba_corrente_media_30d          (Statistics)
binary_sensor.bomba_corrente_estabilizada
binary_sensor.bomba_mudanca_rapida_corrente
binary_sensor.bomba_desgaste_emergente
binary_sensor.bomba_corrente_anormal
binary_sensor.bomba_corrente_critica
```

Source files: `config/sensors.yaml` and `config/template_sensors.yaml`.

---

## Customization

### Hide the DEBUG section

Either create `input_boolean.pump_debug_mode` and leave it off, or delete the DEBUG section (Section 8) from the YAML file directly.

### Gauge threshold colors

```yaml
severity:
  green: 0.303    # start of normal range
  yellow: 0.323   # start of warning range
  red: 0.388      # start of critical range
```

### Themes

The dashboard follows the active HA theme. For best results with the Mushroom variant, install a Mushroom-compatible theme via HACS (e.g. Mushroom Themes).

---

## Troubleshooting

| Problem | Fix |
|---------|-----|
| Cards appear blank | Install Mushroom Cards via HACS, or switch to the native variant |
| Entities show "unavailable" | Check `sensors.yaml` and `template_sensors.yaml`, then run `ha core check` |
| Graphs missing | Install Mini Graph Card via HACS, or replace `mini-graph-card` with `history-graph` |
| Dashboard not in sidebar | Verify view type is "Sidebar" (Method 1) or `show_in_sidebar: true` (Method 2); clear browser cache |
| Wear sensors show no data | Statistics sensors need 7–30 days of data to stabilize — this is expected on first install |

---

## Tips

1. **Start with the native variant** if you're new to custom cards
2. **Enable debug mode** initially to understand how all sensors interact
3. **Wait 7 days** before relying on wear trend sensors (7d/30d statistics need time to fill)
4. **Adjust thresholds** after collecting real data from your pump — see [calibration methodology](../README.md#-the-data-story)

---

*See [docs/lessons-learned.md](lessons-learned.md) for known issues and workarounds.*

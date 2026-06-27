# Changelog

All notable changes to this project are documented here.

Format based on [Keep a Changelog](https://keepachangelog.com/en/1.0.0/), versioning follows [Semantic Versioning](https://semver.org/).

---

## [3.6.0] - 2026-06-27

### Added
- **Temperature-based circulation failure detection** (`binary_sensor.bomba_falha_circulacao` — Layer 2d)
  - Detects silent Zigbee failure: pump commanded ON but relay didn't fire
  - Sensor: `sensor.sonoff_1000bcbe07_temperature` (pipe after pump)
  - Logic: (pump ON OR valve open) AND temp < threshold for 2 continuous minutes → alert
  - `delay_on: 2min` — normal temp rise takes <1min; 2min covers inrush + pipe fill
- **Temperature derivative sensor** (`sensor.bomba_taxa_mudanca_temperatura`) — °C/min, 5min window
- **Configurable threshold helper** (`input_number.bomba_temp_funcionamento_minima`) — default 32°C, slider 25–45°C
- **Multi-channel alert** (`bomba_alerta_falha_circulacao` automation #8):
  - TTS via `script.nabu_speak` (`volume_type: default`, respects quiet hours)
  - Push notification iPhone (`notify.mobile_app_iphone_do_silas`, `interruption-level: time-sensitive`)
  - Persistent notification with diagnostic info
  - Logbook + system_log at `warning` level
- New file: `config/helpers_temperatura.yaml` with install instructions

### Why
Zigbee radio initialization failures caused the pump to appear ON in HA while the relay remained inactive. The current-based system couldn't detect this — switch showed `on` but no current flows until relay fires. Temperature monitoring closes this gap with a completely independent detection layer.

---

## [3.5.2] - 2026-01-06

### Fixed
- **Rapid change threshold**: raised from 0.010 to 0.025 A/s (P97) — eliminates false positives from normal inrush
- **Anti-inrush filter**: first 10s after pump activation are now ignored by the rapid-change sensor
- **Hysteresis**: added `delay_on=3s` and `delay_off=5s` to `binary_sensor.bomba_mudanca_rapida_corrente`
- **Wear detection**: added conditions (`pump_on` + `stabilized`) and delays (`delay_on=30min`, `delay_off=1h`) to prevent transient triggers
- **Input numbers**: normal current thresholds now correctly set to 0.303A / 0.323A

### Changed
- **Strategic TTS**: removed voice alerts from non-critical events (rapid change, wear trend, abnormal current)
- **TTS kept only for**: critical current (>0.388A) and 30min safety timeout — 60% reduction in noise
- **Log levels**: monitoring alerts changed from `warning` to `info`
- Added extra attributes to sensors:
  - `bomba_mudanca_rapida_corrente`: `time_since_activation`, `inrush_filter_active`
  - `bomba_desgaste_emergente`: `pump_running`, `current_stable`, `difference_percent`

### Validated
- ✅ Zero false positives in normal operation
- ✅ Alerts fire only on real events
- ✅ Silent TTS during normal operation
- ✅ Dashboard stable and responsive
- ✅ 0.303–0.323A thresholds confirmed

**Why:** Analysis of 1826 samples showed the 0.010 A/s threshold captured 26% of all readings (normal inrush). The new 0.025 A/s threshold (P97) captures only genuine anomalies.

---

## [3.5.1] - 2024-12-14

### Fixed
- **Cooldown timer bug**: removed stale condition `timer.pump_anti_cycle_cooldown == "idle"` that blocked the main automation (timer was removed in v3.0)
- **UTF-8 encoding**: fixed corrupted characters in `automations_bomba.yaml` and `scripts_bomba.yaml`

### Added
- **Dashboard v3.5**: `config/dashboard_bomba_v35.yaml` with 2-column grid layout and DEBUG section (timers, automations, diagnostic checklist)

### Changed
- Renamed: `automations.yaml` → `automations_bomba.yaml`, `scripts.yaml` → `scripts_bomba.yaml`

---

## [3.5.0] - 2024-12-12

### Fixed
- **Derivative sensor**: `unit_time: s` (was `min`), `time_window: "00:00:05"` (was `"00:05:00"`) — sensor was reporting A/min instead of A/s

### Added
- `binary_sensor.bomba_corrente_estabilizada`: |dI/dt| < 0.005 A/s for 2s — inrush guard
- `binary_sensor.bomba_mudanca_rapida_corrente`: |dI/dt| > 0.010 A/s — acute problem detection
- `binary_sensor.bomba_desgaste_emergente`: 7d avg > 30d avg × 1.03 — wear trend detection
- Detailed attributes on all new sensors

### Changed
- `binary_sensor.bomba_corrente_anormal`: new logic `(stabilized AND out_of_range) OR rapid_change`
- Removed `delay_on` of 3s (replaced by stabilization logic)

---

## [3.0.0] - 2024-12-11

### Added
- Current-based monitoring replacing power-based monitoring
- Statistics sensors: 24h, 7d, 30d moving averages
- Derivative sensor for rate-of-change (dI/dt)
- Progressive wear detection

### Changed
- Thresholds: current 0.303–0.323A (was power 48–53W)
- Critical threshold: 0.388A (max × 1.15)

### Removed
- Cycle counter (HA history logs are sufficient)
- Energy cost monitoring (use HA's native utility_meter instead)
- Cooldown timer (current-based protection is more reliable)

---

## [2.0.0] - 2024-01

### Added
- Configurable thresholds via `input_number` helpers
- 22 helpers, 6 automations, 3 scripts
- 30min safety timeout, 3min cooldown, 20 cycles/hour limit
- Full dashboard

### Changed
- Power thresholds calibrated from 143 real measurements: 48–53W

---

## [1.0.0] - 2024-01

### Added
- Initial implementation with hardcoded thresholds
- Flow detection via Sonoff Mini
- Pump control via Tuya TS011F
- Basic protections (timeout, manual override)

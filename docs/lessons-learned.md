# Lessons Learned — Hot Water Circulation Pump

## Hardware

### Sonoff Mini — Stuck Sensor
**Problem:** Sensor occasionally gets stuck in "open" state during rapid water pulses.  
**Cause:** Sonoff firmware has insufficient debounce for very fast flows.  
**Fix:** 3s delay in the automation before activating the pump (confirms real flow).  
**Validated:** Jan 2024 (v2.0)  
**File:** `automations_bomba.yaml` — `pump_activation_delay`

### Tuya TS011F — Current Reading Noise
**Problem:** Current readings have ±0.01A precision noise.  
**Cause:** Limited ADC resolution on the chip.  
**Fix:** Use Statistics sensor for moving average (reduces noise).  
**Validated:** Dec 2024 (v3.0)  
**File:** `sensors.yaml` — `bomba_corrente_media_*`

---

## Software

### Derivative — Wrong Time Unit
**Problem:** Derivative sensor with `unit_time: min` and `time_window: "00:05:00"` failed to detect inrush (~2s event).  
**Cause:** A 5-minute window dilutes a 2-second event to nearly zero.  
**Fix:** `unit_time: s` and `time_window: "00:00:05"` (5 seconds).  
**Validated:** Dec 12, 2024 (v3.5)  
**File:** `sensors.yaml` — `bomba_taxa_mudanca_corrente`

### Inrush Current — False Positives
**Problem:** Startup current spike (~0.35A → 0.31A over 2s) triggered abnormal-current alerts.  
**Cause:** Simple delay on `bomba_corrente_anormal` was not sufficient.  
**Fix:** v3.5 logic — only alert if `stabilized AND out_of_range`.  
**Validated:** Dec 12, 2024 (v3.5)  
**File:** `template_sensors.yaml` — `bomba_corrente_anormal`, `bomba_corrente_estabilizada`

### Utility Meter — Cumulative Sensor Issue
**Problem:** Utility meter computed wrong values with a `summation_delivered` (cumulative) source.  
**Cause:** Utility meter reset logic doesn't handle a source that only increases.  
**Fix:** Removed utility_meter in v3.0; use `history_stats` for runtime instead.  
**Validated:** Dec 2024 (v3.0)  
**File:** `sensors.yaml` — `bomba_runtime_hoje`

### Rapid Change Threshold — Persistent False Positives
**Problem:** Threshold of 0.010 A/s triggered on 26% of all samples (normal inrush + shutoff transients).  
**Cause:** Threshold below P95 (0.0618 A/s); no inrush filter; no hysteresis.  
**Fix:**
- Threshold raised to 0.025 A/s (P97)
- Anti-inrush filter: ignore first 10s after pump activation
- Hysteresis: `delay_on=3s`, `delay_off=5s`

**Validated:** Dec 18, 2024 (v3.5.2)  
**File:** `template_sensors.yaml` — `bomba_mudanca_rapida_corrente`  
**Reference:** [docs/analysis/current-calibration-analysis.md](analysis/current-calibration-analysis.md)

### Strategic TTS vs. Universal TTS
**Problem:** TTS on technical monitoring alerts (rapid change, wear trend) created noise pollution at home.  
**Cause:** Not every alert requires immediate action — technical monitoring vs. true emergencies are different.  
**Fix:**
- TTS **only** for emergencies: critical current (>0.388A) + 30min safety timeout
- Technical alerts: `system_log` (info) + logbook + dashboard

**Validated:** Dec 18, 2024 (v3.5.2)  
**File:** `automations_bomba.yaml`

---

## Calibration

### Real Data vs. Theoretical Values
**Problem:** Nameplate values (50W / 0.23A) didn't match reality.  
**Cause:** Actual pump draws more than rated (68W real / 0.31A avg).  
**Fix:** Always calibrate with real measurements (minimum 100 samples).  
**Validated:** Jan 2024 (143 power samples), Dec 2024 (1826 current samples)

### P5/P95 Percentiles vs. Min/Max
**Problem:** Using absolute min/max includes outliers in the normal range.  
**Cause:** Outliers are one-off events, not representative of normal operation.  
**Fix:** Use P5/P95 percentiles for the normal operating range.  
**Validated:** Dec 2024 (v3.5)  
**File:** `template_sensors.yaml` — thresholds 0.303–0.323A  
**Reference:** Analysis of 1826 samples

---

## Architecture

### Big-Bang vs. Incremental Updates
**Problem:** Incremental updates left the system in inconsistent states.  
**Cause:** Sensors depend on each other — partial updates break the chain.  
**Fix:** "Big bang" migration — update all related files in a single deployment.  
**Validated:** Dec 2024 (v3.0, v3.5)

### Document Current State, Not Plans
**Problem:** Documentation mixed "planned" and "implemented" features.  
**Cause:** Writing docs before validating in production.  
**Fix:** Keep documentation synchronized with the actual deployed state. Update after validation, not before.  
**Validated:** Dec 2024

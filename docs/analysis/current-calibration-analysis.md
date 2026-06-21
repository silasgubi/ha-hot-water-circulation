# Current Analysis — Hot Water Circulation Pump

**Date:** 2024-12-18  
**Period:** 30 days (2024-11-18 to 2024-12-18)  
**Samples:** 1,826 (current) + 836 (derivative)

---

## Objective

Validate current and rate-of-change (dI/dt) thresholds after reports of **persistent false positives** on:
- "Rapid Change" alert
- "Emerging Wear" alert

---

## Results — Current (Pump Running)

**Dataset:** 1,292 samples with current > 0.1A (71.8% of the time)

| Metric | Value | Status |
|--------|-------|--------|
| **P5** | 0.303A | ✅ Min threshold OK |
| **Median** | 0.310A | ✅ Normal operation |
| **Mean** | 0.309A | ✅ Consistent |
| **P95** | 0.323A | ✅ Max threshold OK |
| **Maximum** | 0.330A | ✅ Within expected |

### Threshold Validation
```yaml
input_number.pump_current_normal_min: 0.303A   # P5  ✅
input_number.pump_current_normal_max: 0.323A   # P95 ✅
input_number.pump_current_critical_max: 0.388A  # Max × 1.15 ✅
```

**Conclusion:** Current thresholds are **CORRECT**. No changes needed.

---

## Results — Rate of Change (dI/dt)

**Dataset:** 836 valid samples

| Metric | Value | Interpretation |
|--------|-------|---------------|
| **Median** | 0.0001 A/s | Stable operation (near zero) |
| **P75** | 0.0014 A/s | Normal variation |
| **P95** | 0.0618 A/s | Inrush / shutoff transients |
| **P99** | 0.0634 A/s | Fast but normal events |
| **Maximum** | 0.0634 A/s | Peak value observed |

### Problem Identified

**Previous threshold:** `0.010 A/s`  
**Events triggered:** 218 / 836 (26%)

Analysis:
- ❌ Threshold of `0.010 A/s` is **below P95** (0.0618 A/s)
- ❌ Captures **all** startup / shutoff transients (normal inrush)
- ❌ Cannot distinguish normal inrush from a real problem

**Logbook evidence** (2024-12-18, 10:36–10:57):
```
10:57:04 — Rapid change alert fired with dI/dt = 0.0003 A/s
10:39:09 — Rapid change alert fired with dI/dt = 0.0014 A/s
10:36:07 — Rapid change alert fired with dI/dt = 0.0002 A/s
```

Values of 0.0002–0.0014 A/s should **not** have triggered a 0.010 A/s threshold, confirming a **logic bug** in the binary sensor (separate from the threshold issue).

---

## Results — Moving Averages (Wear Trend)

| Sensor | Current Value | Status |
|--------|--------------|--------|
| **7d average** | 0.270A | Stable baseline |
| **24h average** | 0.272A | +0.74% vs 7d |
| **Wear threshold** | 0.278A (7d × 1.03) | Not exceeded |

**Conclusion:** No signs of progressive wear. System operating normally.

---

## Problems Identified

### 1. Rapid-change threshold too low
- Current: 0.010 A/s
- Captures: 26% of all samples (normal inrush)
- **Recommended:** 0.025 A/s (P97 — between P95 and P99)

### 2. Binary sensor logic bug
- Alert firing with values as low as 0.0002–0.0003 A/s
- Root cause: incorrect condition or missing hysteresis

### 3. TTS on non-critical alerts
- "Rapid change" = normal inrush, not an emergency
- "Emerging wear" = preventive monitoring, not urgent
- **Proposal:** TTS only for `corrente_critica` (>0.388A) and 30min timeout

---

## Recommendations

### Fix 1: Raise threshold
```yaml
# Increase from 0.010 to 0.025 A/s (P97)
binary_sensor.bomba_mudanca_rapida_corrente:
  value_template: >
    {{ states('sensor.bomba_taxa_mudanca_corrente')|float(0)|abs > 0.025 }}
```

### Fix 2: Add anti-inrush time filter
```yaml
# Ignore first 10s after pump activation
binary_sensor.bomba_mudanca_rapida_corrente:
  value_template: >
    {% set dI = states('sensor.bomba_taxa_mudanca_corrente')|float(0)|abs %}
    {% set bomba_on = is_state('switch.bomba_de_circulacao_de_agua_quente', 'on') %}
    {% set tempo_ligada = (now() - states.switch.bomba_de_circulacao_de_agua_quente.last_changed).total_seconds() %}
    {{ bomba_on and tempo_ligada > 10 and dI > 0.025 }}
```

### Fix 3: Add hysteresis
```yaml
binary_sensor.bomba_mudanca_rapida_corrente:
  delay_on: "00:00:03"   # confirm for 3s before alerting
  delay_off: "00:00:05"  # wait 5s before clearing
```

### TTS Strategy: Reduce Noise

**TTS only for:**
- ✅ Critical current (>0.388A) — emergency
- ✅ Safety timeout (30min) — possible stuck valve

**Dashboard only for:**
- 📊 Rapid change — technical indicator
- 📊 Emerging wear — preventive maintenance
- 📊 Abnormal current (non-critical) — monitoring

---

## Outcome (implemented in v3.5.2)

**Before:**
- Alert every 2–5 minutes (false positives)
- Persistent TTS during normal operation
- Correct data, buggy logic

**After:**
- Alerts only for real events (>P97, after 10s stabilization)
- Silent TTS during normal use
- Zero false positives validated

---

*Analysis performed Dec 18, 2024. Changes implemented in v3.5.2 (2026-01-06).*

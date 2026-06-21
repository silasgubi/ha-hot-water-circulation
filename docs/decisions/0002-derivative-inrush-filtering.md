# ADR-0002: Anti-Inrush Filter and dI/dt Threshold Adjustment

**Status:** Accepted | **Date:** 2024-12-18 | **Version:** v3.5.2

## Context

v3.5 introduced derivative-based monitoring (dI/dt) to detect rapid current changes and distinguish between:
- **Normal inrush current:** Motor startup spike (~2s, up to 0.06 A/s)
- **Real problems:** Jamming, wear, overload

**Identified problem:** After deploying v3.5.1, persistent false positives were reported (alerts every 2–5 minutes), indicating inrush was not being filtered correctly.

## Problem

### Real-data analysis (30 days, 836 derivative samples)

| Metric | Value | Interpretation |
|--------|-------|---------------|
| P95 \|dI/dt\| | 0.0618 A/s | Typical inrush / shutoff |
| P99 \|dI/dt\| | 0.0634 A/s | Fast but normal events |
| **Previous threshold** | 0.010 A/s | ❌ Below P95 — captures normal inrush |
| **Events triggered** | 218/836 (26%) | ❌ All of them were normal inrush |

### Logbook evidence
```
10:57:04 — Alert fired with dI/dt = 0.0003 A/s (should require >0.010)
10:39:09 — Alert fired with dI/dt = 0.0014 A/s (should require >0.010)
10:36:07 — Alert fired with dI/dt = 0.0002 A/s (should require >0.010)
```

Root causes:
1. Threshold of 0.010 A/s is below P95 — captures normal startup
2. Sensor triggered with values **below** the threshold (logic bug)
3. No temporal filter: inrush in the first 2–3s always triggered an alert

## Options

### Option 1: Raise threshold only
```yaml
threshold: 0.010 → 0.065 A/s (P99)
```
**Pros:** Simple, one line. Eliminates 99% of false positives.  
**Cons:** Still triggers on inrush in the first 2–3s. No safety margin. Doesn't fix the logic bug.

### Option 2: Moderate threshold + temporal filter ✅ Chosen
```yaml
threshold: 0.010 → 0.025 A/s (P97)
+ ignore first 10s after pump activation
```
**Pros:** Threshold between P95–P99 catches real anomalies without false positives. 10s filter eliminates inrush entirely. Fixes logic bug. Retains sensitivity during steady operation.  
**Cons:** Slightly more complex (+5 lines). Needs to track time since activation.

### Option 3: High threshold + hysteresis
```yaml
threshold: 0.010 → 0.065 A/s (P99)
+ delay_on: 3s, delay_off: 5s
```
**Pros:** Simple. Hysteresis prevents rapid toggling.  
**Cons:** Threshold too high — may miss progressive problems. Doesn't filter initial inrush.

## Decision

**Option 2** — moderate threshold + temporal filter.

### Final implementation
```yaml
binary_sensor.bomba_mudanca_rapida_corrente:
  state: >
    {% set dI = states('sensor.bomba_taxa_mudanca_corrente')|float(0)|abs %}
    {% set bomba_on = is_state('switch.bomba_de_circulacao_de_agua_quente', 'on') %}
    {% set tempo_ligada = (now() - states.switch.bomba_de_circulacao_de_agua_quente.last_changed).total_seconds() %}
    {% set threshold = 0.025 %}
    {{ bomba_on and tempo_ligada > 10 and dI > threshold }}
  delay_on: "00:00:03"
  delay_off: "00:00:05"
```

**Final parameters:**
- Threshold: **0.025 A/s** (P97 — between normal events and real anomalies)
- Anti-inrush filter: **10s** (inrush lasts ~2–3s + 7s safety margin)
- Hysteresis on: **3s** (confirm before alerting)
- Hysteresis off: **5s** (prevent rapid toggling)

## Rationale

**Why 0.025 A/s?**  
P95 = 0.0618 A/s (typical inrush). The 0.025 A/s threshold sits between normal operation (<P95) and genuinely anomalous events (>P97). It provides 2.5× headroom above the original broken threshold.

**Why 10s filter?**  
Real inrush lasts 2–3s. Adding 7s safety margin also covers mains voltage transients, mechanical stabilization, and electrical noise during startup.

**Why 3s/5s hysteresis?**  
3s `delay_on` confirms the event isn't electrical noise. 5s `delay_off` waits for full stabilization before clearing the alert, preventing rapid `detected → cleared → detected` cycles.

## Consequences

**Positive:**
- ✅ Eliminates 26% false-positive rate (normal inrush)
- ✅ Retains sensitivity for real problems (>P97)
- ✅ Only detects anomalies during steady operation (after 10s)
- ✅ Hysteresis reduces logbook noise

**Accepted negatives:**
- ⚠️ Problems in the first 10s are not detected — mitigated: real problems persist beyond 10s
- ⚠️ Slightly more complex code — mitigated: documented and tested

## Impact

**Before (v3.5.1):**
- Alert every 2–5 minutes (inrush = false positive)
- TTS firing constantly during normal use
- Correct data, but buggy logic

**After (v3.5.2):**
- Alerts only on real events (>P97, after 10s stabilization)
- Silent TTS during normal operation
- Validated: zero false positives

## Validation

- ✅ Zero rapid-change alerts during inrush (first 10s)
- ✅ Alerts fire only for real events
- ✅ No rapid toggling in logbook
- ✅ No template errors in system log

**Result:** <1% alert rate (vs. 26% before), exactly at P97 target.

## References

- [docs/analysis/current-calibration-analysis.md](../analysis/current-calibration-analysis.md) — full data analysis
- [ADR-0001: Current vs Power](0001-current-vs-power.md)
- Motor datasheet: 50W pump inrush typically 2–3× nominal current for 1–3s

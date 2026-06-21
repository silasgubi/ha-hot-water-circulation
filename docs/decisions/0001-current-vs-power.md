# ADR-0001: Current Monitoring vs. Power Monitoring

**Status:** Accepted | **Date:** 2024-12-11 | **Version:** v3.0

## Context

v2.0 used power monitoring (48–53W) to detect pump anomalies. After collecting more real-world data, it became clear that current is a more reliable signal.

## Problem

How can we best detect pump problems (jamming, wear, overload)?

## Options

### 1. Power (W) — v2.0 status quo
**Pros:**
- Already implemented and calibrated (143 measurements)
- Easy to understand

**Cons:**
- Varies with mains voltage (220V ± 10%)
- Less sensitive for detecting early-stage jamming

### 2. Current (A) — proposed
**Pros:**
- More stable (independent of voltage variation)
- More sensitive: current rises before power when motor starts to jam
- Enables power factor calculation

**Cons:**
- Requires recalibration
- New data collection needed

## Decision

**Current (A)**

## Rationale

1. **Data-driven:** 1826 real samples (December 2024) show tighter, more consistent distribution
2. **Precision:** Lower standard deviation (0.005A vs ~1.5W relative)
3. **Physics:** A jamming motor draws more current before power rises noticeably
4. **Calibrated thresholds:**
   - Normal: 0.303–0.323A (P5–P95)
   - Critical: 0.388A (max × 1.15)

## Consequences

**Positive:**
- Faster problem detection
- Fewer false positives from voltage fluctuation
- Foundation for predictive maintenance (trend analysis)

**Accepted negatives:**
- One-time migration required
- Old power-based helpers need updating

## Implementation

Files changed:
- `sensors.yaml` — Derivative and Statistics sensors
- `template_sensors.yaml` — Binary sensors updated
- `automations_bomba.yaml` — Thresholds updated

New helpers:
- `input_number.pump_current_normal_min`: 0.303
- `input_number.pump_current_normal_max`: 0.323
- `input_number.pump_current_critical_max`: 0.388

## References

- Analysis: 1826 samples, December 2024
- Related: [ADR-0002: Anti-Inrush Filter](0002-derivative-inrush-filtering.md)

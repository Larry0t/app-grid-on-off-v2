# VE.Bus Watchdog & Self-Healing – Grid ON_OFF v2.0

## Purpose

This document specifies a **watchdog and self-healing mechanism** for the
**Grid ON_OFF v2.0** control system.

Its goal is to guarantee **safe, deterministic behavior** when
**VE.Bus data disappears, stalls, or recovers**, without human intervention.

This is a *control-safety layer*, not a convenience feature.

---

## Problem Statement

In production, VE.Bus communication may fail due to:

- GX reboot or upgrade
- VE.Bus service restart
- Cable / bus glitches
- Node-RED restart mid-session
- Corrupted node instances (observed)

Failure modes without a watchdog:
- stale grid state decisions
- FSM operating on outdated inputs
- grid left ON indefinitely
- silent failure with no visibility

---

## Design Principles

1. **Fail-safe default**
   - Loss of confidence → force safe grid state

2. **Non-invasive**
   - Watchdog does not interfere during normal operation

3. **State-aware**
   - Integrates with FSM, not parallel ad-hoc logic

4. **Self-healing**
   - Automatic recovery when VE.Bus returns

5. **Observable**
   - Explicit health states and metrics

---

## Health Model

### VE.Bus Health States

| State | Meaning |
|------|--------|
| `OK` | Data arriving within expected interval |
| `STALE` | Data delayed but not lost |
| `LOST` | No data beyond hard timeout |
| `RECOVERING` | Data resumed, stabilization phase |

---

## Inputs Monitored

Minimum required VE.Bus signals:

- `/Mode` (Grid switch feedback)
- Any high-frequency VE.Bus input used by decisions (load, SOC, power)

Each monitored signal tracks:
- `lastSeenTimestamp`
- `ageMs`

---

## Timing Parameters (Recommended Defaults)

| Parameter | Value | Purpose |
|---------|------|---------|
| `STALE_MS` | 10 s | Early warning |
| `LOST_MS` | 30 s | Declare communication lost |
| `RECOVERY_STABLE_MS` | 15 s | Confidence rebuild |

All values must be configurable via `flow.set()`.

---

## Watchdog Logic

### State Machine

```
OK → STALE → LOST → RECOVERING → OK
```

### Transitions

- **OK → STALE**
  - Any critical signal age > STALE_MS

- **STALE → LOST**
  - Any critical signal age > LOST_MS

- **LOST → RECOVERING**
  - Fresh data resumes on all signals

- **RECOVERING → OK**
  - All signals remain fresh for RECOVERY_STABLE_MS

---

## Actions by State

### OK
- No action
- FSM operates normally

---

### STALE
- No control override
- Emit warning metric
- Update node status (yellow)

---

### LOST
- Override FSM output
- Force safe grid state (recommended: GRID.ON)
- Freeze FSM transitions
- Increment watchdog intervention counter
- Emit critical metric
- Node status: red

---

### RECOVERING
- Keep safe override active
- Block FSM output
- Do not count cycles
- Node status: blue

---

## Integration Point

### Recommended Placement

```
FSM Output
   ↓
Watchdog Gate  ←── VE.Bus Health Monitor
   ↓
Victron Safe Output Wrapper
```

The watchdog **gates commands**, not decisions.

---

## Watchdog Output Contract

```js
{
  payload: GRID.ON | GRID.OFF,
  watchdog: {
    state: "OK" | "STALE" | "LOST" | "RECOVERING",
    forced: true | false,
    since: timestamp
  }
}
```

---

## Metrics Export

### Required Metrics

| Metric | Type | Description |
|------|------|-------------|
| `victron_bus_health` | enum | 0=OK,1=STALE,2=LOST,3=RECOVERING |
| `watchdog_forced` | counter | Number of forced safe states |
| `watchdog_last_event` | tag | Reason / transition |
| `watchdog_lost_duration` | gauge | Duration of LOST state |

These metrics directly feed **Grafana Dashboard 4**.

---

## Alerting Recommendations

### Critical: VE.Bus Lost
- Condition: `victron_bus_health == LOST`
- Action: Notify operator

### Warning: Frequent Recoveries
- Condition: `watchdog_forced > N/day`
- Action: Inspect VE.Bus stability

---

## Failure Philosophy

The system must prefer:

> **“Grid ON but safe”**  
over  
> **“Grid OFF but blind”**

Any other choice risks uncontrolled discharge or inverter overload.

---

## Implementation Notes (Node-RED)

- Implement health monitor as a **dedicated function node**
- Use `context` for per-signal timestamps
- Use `flow` for configuration
- Never mutate upstream messages
- Watchdog must be **purely additive**

---

## Roadmap

1. Implement VE.Bus Health Monitor
2. Insert Watchdog Gate before output wrapper
3. Export metrics to DB
4. Add Grafana Dashboard 4
5. Long-term stability tuning

---

## Status

This design completes **Grid ON_OFF v2.x** as a
**self-healing, production-hardened control system**.

No human babysitting required.

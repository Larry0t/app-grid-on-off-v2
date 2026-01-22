# Watchdog Gate – Node-RED Function

This document contains the **reference implementation** of the
**Watchdog Gate** for *Grid ON_OFF v2.x*.

The Watchdog Gate is the **final safety authority** before commands reach
the Victron VE.Bus output node.

It enforces **fail-safe behavior** when VE.Bus health is degraded.

---

## Role in Architecture

```
Grid FSM Controller
        ↓
Watchdog Gate   ←── VE.Bus Health Monitor
        ↓
Victron Safe Output Wrapper
        ↓
victron-output-vebus
```

- FSM decides *what* and *when*
- Watchdog decides *whether it is safe to act*
- Wrapper ensures *safe delivery*

---

## Responsibilities

- Block unsafe commands when VE.Bus is unhealthy
- Force a predefined **safe grid state**
- Freeze switching during recovery
- Annotate messages with watchdog metadata
- Provide clear node status

---

## Fail-Safe Policy

**Default safe state:** `GRID.ON`

Rationale:
- Prevent uncontrolled battery discharge
- Prevent inverter overload
- Maintain supply continuity

This is configurable.

---

## Expected Inputs

### 1. FSM Command

```js
{
  payload: GRID.ON | GRID.OFF,
  state: "ON" | "OFF" | "WAIT_ON" | "WAIT_OFF",
  reason: "SOC_LOW" | "OVERLOAD" | ...
}
```

### 2. VE.Bus Health (from Health Monitor)

```js
{
  topic: "victron_bus_health",
  payload: "OK" | "STALE" | "LOST" | "RECOVERING",
  watchdog: { ... }
}
```

---

## Configuration (flow context)

```js
flow.set("WATCHDOG_GATE_CFG", {
  SAFE_STATE: 3,      // GRID.ON
  ALLOW_STALE: true  // allow FSM output during STALE
});
```

---

## Function Code

```js
// === Watchdog Gate v1.0 ===
// Final safety gate before Victron output

const GRID = flow.get("GRID") ?? { OFF: 2, ON: 3 };

const cfg = flow.get("WATCHDOG_GATE_CFG") ?? {
    SAFE_STATE: GRID.ON,
    ALLOW_STALE: true
};

const ctx = context.get("gate") ?? {
    health: "OK",
    forcedCount: 0
};

// ---- Update health ----
if (msg.topic === "victron_bus_health") {
    ctx.health = msg.payload;
    context.set("gate", ctx);
    return null;
}

// ---- FSM command expected below ----
let outPayload = msg.payload;
let forced = false;

// ---- Decision ----
switch (ctx.health) {
    case "OK":
        break;

    case "STALE":
        if (!cfg.ALLOW_STALE) {
            outPayload = cfg.SAFE_STATE;
            forced = true;
        }
        break;

    case "LOST":
    case "RECOVERING":
        outPayload = cfg.SAFE_STATE;
        forced = true;
        break;
}

// ---- Forced accounting ----
if (forced) {
    ctx.forcedCount += 1;
    context.set("gate", ctx);
}

// ---- Status ----
node.status({
    fill:
        ctx.health === "OK" ? "green" :
        ctx.health === "STALE" ? "yellow" :
        ctx.health === "RECOVERING" ? "blue" :
        "red",
    shape: "dot",
    text: forced
        ? `FORCED ${outPayload === GRID.ON ? "ON" : "OFF"} (${ctx.health})`
        : `PASS (${ctx.health})`
});

// ---- Output ----
return [{
    payload: outPayload,
    state: msg.state,
    reason: forced ? "WATCHDOG_FORCE" : msg.reason,
    watchdog: {
        health: ctx.health,
        forced,
        forcedCount: ctx.forcedCount
    }
}];
```

---

## Output Contract

```js
{
  payload: GRID.ON | GRID.OFF,
  reason: "WATCHDOG_FORCE" | <FSM reason>,
  watchdog: {
    health,
    forced,
    forcedCount
  }
}
```

---

## Metrics Integration

Export these fields to your metrics backend:

- `watchdog.health`
- `watchdog.forced`
- `watchdog.forcedCount`

They directly feed:
- Grafana Dashboard 4
- Alerting rules

---

## Design Guarantees

- No FSM mutation
- No hidden timers
- No race conditions
- Safe under restart
- Deterministic behavior

---

## Status

With this node, **Grid ON_OFF v2.x** becomes a
**self-healing, fail-safe control system**.

This completes the control loop:
**sense → decide → time → gate → act → observe → recover**.

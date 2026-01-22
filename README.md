# Grid ON_OFF v2.0

## Overview

**Grid ON_OFF v2.0** is a production-grade, deterministic grid control system for **Victron ESS** installations, implemented in **Node-RED**.

The system autonomously connects or disconnects the grid (`/Mode`) based on battery state, AC load conditions, inverter limits, and configurable schedules.  
It is designed for **24/7 autonomous operation**, **safe restarts**, and **long-term observability** on Venus OS.

---

## Key Design Principles

- **Deterministic behavior** — no race conditions, no grid flapping
- **Explicit FSM** — switching logic is state-driven, not event-driven
- **Separation of concerns** — decision, timing, and actuation are isolated
- **Fail-safe startup** — no switching until all inputs are initialized
- **Observability-first** — metrics, reasons, and internal states are visible

---

## High-Level Architecture

```
Measurements & Schedules
        ↓
Grid State Collector
        ↓
Startup Guard
        ↓
Grid Decision Engine (WHY)
        ↓
Grid FSM Controller (WHEN)
        ↓
Victron Safe Output Wrapper (HOW)
        ↓
victron-output-vebus (/Mode)
```

---

## Functional Blocks

### 1. Grid State Collector

**Purpose**  
Normalize all heterogeneous inputs into a single coherent state object.

**Inputs**
- Grid switch feedback (`/Mode` input)
- SOC hysteresis output
- Battery power hysteresis output
- Load hysteresis (L1–L4)
- Inverter overload flag
- ChargeTime / FeedTime schedule signals

**Normalized state**
```js
{
  gridInput,
  socSwitch,
  battWSwitch,
  loadSwitch,
  overload,
  chargeTimer,
  feedTimer
}
```

This block guarantees:
- numeric normalization
- stable topic-to-field mapping
- isolation from upstream wiring complexity

---

### 2. Startup Guard

**Purpose**  
Prevent undefined or unsafe behavior after Node-RED or Venus OS restarts.

**Behavior**
- Blocks downstream logic until all critical state fields are initialized
- Prevents unintended grid switching during cold start

This is required because Victron inputs may arrive at different times after boot.

---

### 3. Grid Decision Engine

**Purpose**  
Decide **what** the desired grid state should be, independent of timing.

**Priority rules (highest first)**

1. Battery SOC below minimum
2. Inverter overload detected
3. High AC load detected
4. Battery power limit exceeded
5. Charge schedule active
6. Feed-in schedule active
7. Fallback to current grid state

**Output**
```js
{
  payload: GRID.ON | GRID.OFF,
  reason: "SOC_LOW" | "OVERLOAD" | "LOAD_HIGH" | ...
}
```

This node is:
- stateless
- deterministic
- easy to extend with additional rules

---

### 4. Grid FSM Controller

**Purpose**  
Decide **when** switching is allowed.

**FSM States**
- `INIT`     — startup, no switching
- `OFF`      — grid disconnected
- `WAIT_ON`  — ON requested, waiting ON delay
- `ON`       — grid connected
- `WAIT_OFF` — OFF requested, waiting OFF delay

**Guarantees**
- enforced ON delay
- enforced OFF delay
- no duplicate commands
- no chatter under oscillating conditions

This is the **core safety layer** of the system.

---

### 5. Victron Safe Output Wrapper

**Purpose**  
Provide a hardened boundary between Node-RED logic and Victron VE.Bus output nodes.

**Responsibilities**
- payload validation (only valid `/Mode` values allowed)
- deduplication
- controlled priming
- centralized status and debugging

**Production note**

In rare cases, a `victron-output-vebus` node instance can become internally corrupted  
(e.g. after imports or upgrades).

**Deleting and recreating the node resets its internal state and resolves the issue.**

The wrapper exists to minimize risk at this integration boundary.

---

### 6. Metrics & Analytics

**Collected metrics**
- total Grid ON time
- total Grid OFF time
- number of ON cycles
- last switching reason

**Daily snapshot**
- emitted once per day
- suitable for export to InfluxDB, MQTT, CSV, or file storage

This enables **data-driven optimization** instead of intuition.

---

## Observability

The flow provides strong runtime visibility:

- FSM node shows **current state + reason**
- Output wrapper shows **actual commands sent**
- Metrics collector tracks long-term behavior
- Debug taps exist at all critical junctions

This level of observability is comparable to industrial PLC systems.

---

## Known Limitations

- Victron Node-RED nodes can occasionally require recreation after upgrades or imports
- VE.Bus availability is not yet actively supervised

---


## Repository Structure

```text
grid-on-off-v2/
├── README.md        # this document
├── docs/            # 
├── flows/           # Node-RED flow export (exact, unmodified)
├── CHANGELOG.md     # optional
└── diagrams/        # optional architecture diagrams
```

---

## Status

**Grid ON_OFF v2.0** is stable in production, deterministic, observable, and ready for further hardening and visualization.

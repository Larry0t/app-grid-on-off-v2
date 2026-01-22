# Grafana Dashboards – Grid ON_OFF v2.0

This document defines a **Grafana dashboard specification** for the
**Grid ON_OFF v2.0** Node-RED / Victron ESS control system.

The goal is to turn the existing metrics into **operational visibility**
and **decision support**, not just pretty charts.

---

## Data Model Assumptions

The dashboards assume metrics are exported periodically (or on change)
to one of the following backends:

- **InfluxDB** (recommended)
- Prometheus
- MQTT → Telegraf → InfluxDB
- CSV / file-based (limited)

### Core Metrics (logical)

| Metric | Type | Description |
|------|------|-------------|
| `grid_state` | enum | 1 = ON, 0 = OFF |
| `grid_on_time` | counter | Accumulated ON time (seconds or ms) |
| `grid_off_time` | counter | Accumulated OFF time |
| `grid_on_cycles` | counter | OFF → ON transitions |
| `grid_reason` | tag / label | Reason for last switch |
| `fsm_state` | enum | INIT / OFF / WAIT_ON / ON / WAIT_OFF |

---

## Dashboard 1: Grid Overview (Primary)

**Purpose**  
High-level operational visibility.  
This is the dashboard you keep open daily.

### Panels

#### 1. Grid State (Stat)
- Shows: `ON` / `OFF`
- Color:
  - Green = OFF
  - Red = ON
- Data source: latest `grid_state`

---

#### 2. Grid ON % (Last 24h)
- Visualization: Gauge or Stat
- Calculation:
  ```
  on_time / (on_time + off_time) * 100
  ```
- Use case:
  - Detect excessive grid dependency
  - Track seasonal changes

---

#### 3. Grid ON Time (Daily)
- Visualization: Time series (bar)
- Value: daily `grid_on_time`
- Group by: day
- Use case:
  - Energy cost optimization
  - Policy tuning validation

---

#### 4. Grid ON Cycles (Daily)
- Visualization: Bar chart
- Value: `grid_on_cycles`
- Threshold:
  - Warning > X cycles/day
- Use case:
  - Detect chatter
  - Validate FSM delays

---

## Dashboard 2: FSM Behavior & Stability

**Purpose**  
Validate that the FSM behaves exactly as designed.

### Panels

#### 1. FSM State Timeline
- Visualization: State timeline
- Value: `fsm_state`
- States:
  - INIT
  - OFF
  - WAIT_ON
  - ON
  - WAIT_OFF
- Use case:
  - Confirm correct delays
  - Debug edge cases

---

#### 2. Transition Heatmap
- Visualization: Heatmap
- X-axis: Time of day
- Y-axis: FSM transition count
- Use case:
  - Identify switching patterns
  - Spot unstable periods

---

#### 3. WAIT State Duration
- Visualization: Histogram
- Data: time spent in WAIT_ON / WAIT_OFF
- Use case:
  - Validate debounce configuration
  - Detect abnormal delays

---

## Dashboard 3: Switching Reasons (Analytics)

**Purpose**  
Explain *why* the grid is being used.

### Panels

#### 1. Reason Distribution (Pie)
- Value: count by `grid_reason`
- Typical reasons:
  - SOC_LOW
  - OVERLOAD
  - LOAD_HIGH
  - CHARGE_TIMER
  - FEED_TIMER
- Use case:
  - Policy tuning
  - Battery sizing feedback

---

#### 2. Reason Over Time
- Visualization: Stacked area
- X-axis: Time
- Y-axis: ON events
- Grouped by: `grid_reason`
- Use case:
  - Seasonal analysis
  - Load profile evolution

---

## Dashboard 4: Reliability & Watchdog (Future)

**Purpose**  
Support the upcoming **self-healing / watchdog** feature.

### Panels (planned)

#### 1. VE.Bus Health
- Visualization: Stat
- Value:
  - OK
  - STALE
  - LOST
- Derived from:
  - Timestamp freshness of VE.Bus inputs

---

#### 2. Watchdog Interventions
- Visualization: Counter
- Value: number of forced safe-state actions
- Use case:
  - Detect communication instability
  - Correlate with firmware updates

---

## Alerting Rules (Recommended)

### Alert 1: Excessive Grid Cycling
- Condition:
  ```
  grid_on_cycles > N per day
  ```
- Severity: Warning
- Action: Review thresholds and delays

---

### Alert 2: Grid ON Too Long
- Condition:
  ```
  grid_on_time > X hours/day
  ```
- Severity: Info / Warning
- Action: Review SOC thresholds or schedules

---

### Alert 3: FSM Stuck State
- Condition:
  - FSM remains in WAIT_ON or WAIT_OFF > expected delay + margin
- Severity: Critical
- Action: Inspect upstream inputs

---

## Naming Conventions

Use consistent naming across Node-RED, DB, and Grafana:

- `grid_*` → control outputs
- `fsm_*` → state machine internals
- `victron_*` → VE.Bus health
- `energy_*` → derived KPIs (future)

---

## Summary

These dashboards transform **Grid ON_OFF v2.0** from
a “working automation” into an **observable, optimizable system**.

They enable:
- confident tuning
- regression detection
- long-term energy strategy decisions

Once dashboards exist, changes stop being guesswork.

---

**Next logical step:**  
Implement VE.Bus watchdog → feed its metrics directly into Dashboard 4.

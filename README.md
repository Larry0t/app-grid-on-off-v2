# Grid ON_OFF v2.0

## Overview

**Grid ON_OFF v2.0** is a production-grade, deterministic grid control
system for **Victron ESS** installations, implemented in **Node-RED**.

The system autonomously connects or disconnects the grid (`/Mode`) based
on battery state, load conditions, inverter limits, and configurable
schedules.\
It is designed for **24/7 operation**, **safe restarts**, and
**long-term observability**.

------------------------------------------------------------------------

## Key Design Principles

-   **Deterministic behavior** --- no race conditions, no flapping
-   **Explicit FSM** --- switching logic is state-driven, not
    event-driven
-   **Separation of concerns** --- decision, timing, and actuation are
    isolated
-   **Fail-safe startup** --- no switching until all inputs are
    initialized
-   **Observability-first** --- metrics, reasons, and states are always
    visible

------------------------------------------------------------------------

## High-Level Architecture

Measurements & Schedules\
→ Grid State Collector\
→ Startup Guard\
→ Grid Decision Engine (WHY)\
→ Grid FSM Controller (WHEN)\
→ Victron Safe Output Wrapper (HOW)\
→ victron-output-vebus (/Mode)

------------------------------------------------------------------------

## Functional Blocks

### 1. Grid State Collector

Normalizes all raw inputs into a single coherent state object used by
downstream logic.

### 2. Startup Guard

Blocks switching logic until all critical inputs are initialized after
restart.

### 3. Grid Decision Engine

Determines the desired grid state based on prioritized rules such as
SOC, load, overload, and schedules.

### 4. Grid FSM Controller

Implements a formal finite-state machine enforcing ON/OFF delays and
preventing chatter.

### 5. Victron Safe Output Wrapper

Hardens the interface to Victron VE.Bus output nodes with validation and
deduplication.

### 6. Metrics & Analytics

Tracks grid usage time, switching cycles, and reasons for long-term
optimization.

------------------------------------------------------------------------

## Development Roadmap (TODO)

-   Add VE.Bus watchdog & self-healing
-   Export metrics to Grafana dashboards
-   Formalize FSM state snapshots
-   Package logic as reusable Victron FSM pattern

------------------------------------------------------------------------

## Repository Structure

grid-on-off-v2/ ├── README.md ├── flow.json └── diagrams/

------------------------------------------------------------------------

## Status

Stable in production and ready for further hardening and visualization.

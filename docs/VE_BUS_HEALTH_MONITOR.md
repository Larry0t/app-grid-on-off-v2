# VE.Bus Health Monitor – Node-RED Function

This file contains the **reference implementation** of the
**VE.Bus Health Monitor** for *Grid ON_OFF v2.x*.

It is designed to be:
- deterministic
- side-effect free
- reusable
- production safe

---

## Node placement

Place this function node **upstream of the Watchdog Gate**.

It only **observes inputs** and emits health status — it does NOT control outputs.

---

## Expected inputs

Any VE.Bus-derived message that is *required for safe operation*, for example:

- `/Mode`
- Battery SOC
- AC load
- Inverter load
- Battery power

Each message **must have a stable `msg.topic`**.

---

## Configuration (flow context)

```js
flow.set("WATCHDOG_CFG", {
  STALE_MS: 10_000,
  LOST_MS: 30_000,
  RECOVERY_STABLE_MS: 15_000,
  REQUIRED_TOPICS: [
    "GridSwIn",
    "battSOC",
    "LoadL1",
    "LoadL2",
    "LoadL3",
    "LoadL4"
  ]
});
```

---

## Function code

```js
// === VE.Bus Health Monitor v1.0 ===
// Passive observer. Emits VE.Bus health state.

const cfg = flow.get("WATCHDOG_CFG") ?? {
    STALE_MS: 10_000,
    LOST_MS: 30_000,
    RECOVERY_STABLE_MS: 15_000,
    REQUIRED_TOPICS: []
};

const now = Date.now();

const ctx = context.get("health") ?? {
    topics: {},
    state: "OK",
    since: now,
    candidateSince: null
};

// ---- Track topic freshness ----
if (msg.topic) {
    ctx.topics[msg.topic] = now;
}

// ---- Compute ages ----
let worstAge = 0;
let missing = false;

for (const t of cfg.REQUIRED_TOPICS) {
    const ts = ctx.topics[t];
    if (!ts) {
        missing = true;
        worstAge = Math.max(worstAge, cfg.LOST_MS + 1);
        continue;
    }
    worstAge = Math.max(worstAge, now - ts);
}

// ---- Determine raw health ----
let rawState = "OK";

if (missing || worstAge > cfg.LOST_MS) {
    rawState = "LOST";
} else if (worstAge > cfg.STALE_MS) {
    rawState = "STALE";
}

// ---- State machine ----
function transition(to) {
    ctx.state = to;
    ctx.since = now;
    ctx.candidateSince = null;
}

switch (ctx.state) {
    case "OK":
        if (rawState === "STALE") transition("STALE");
        if (rawState === "LOST") transition("LOST");
        break;

    case "STALE":
        if (rawState === "LOST") transition("LOST");
        if (rawState === "OK") transition("OK");
        break;

    case "LOST":
        if (rawState === "OK") {
            ctx.candidateSince = ctx.candidateSince ?? now;
            if (now - ctx.candidateSince >= cfg.RECOVERY_STABLE_MS) {
                transition("RECOVERING");
            }
        }
        break;

    case "RECOVERING":
        if (rawState !== "OK") transition("LOST");
        if (now - ctx.since >= cfg.RECOVERY_STABLE_MS) {
            transition("OK");
        }
        break;
}

context.set("health", ctx);

// ---- Status ----
node.status({
    fill:
        ctx.state === "OK" ? "green" :
        ctx.state === "STALE" ? "yellow" :
        ctx.state === "RECOVERING" ? "blue" :
        "red",
    shape: "dot",
    text: `VE.Bus ${ctx.state}`
});

// ---- Output ----
return [{
    topic: "victron_bus_health",
    payload: ctx.state,
    watchdog: {
        state: ctx.state,
        since: ctx.since,
        worstAgeMs: worstAge
    }
}];
```

---

## Output contract

```js
{
  topic: "victron_bus_health",
  payload: "OK" | "STALE" | "LOST" | "RECOVERING",
  watchdog: {
    state,
    since,
    worstAgeMs
  }
}
```

This message feeds:
- Watchdog Gate
- Metrics collector
- Grafana dashboards

---

## Design notes

- No upstream mutation
- No timers
- No external dependencies
- All timing is monotonic (`Date.now()`)
- Safe across Node-RED restarts

---

## Status

This node completes the **health sensing layer** of the system.

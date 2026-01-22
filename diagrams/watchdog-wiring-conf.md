
### Watchdog wiring

```
Grid FSM Controller  ───▶  Subflow (FSM + Watchdog) ───▶ Victron Safe Output Wrapper
VE.Bus Health ──────▶
```

---

### Configuration (once per flow)

```js
flow.set("GRID", { OFF: 2, ON: 3 });

flow.set("WATCHDOG_GATE_CFG", {
  SAFE_STATE: 3,     // GRID.ON
  ALLOW_STALE: true
});
```

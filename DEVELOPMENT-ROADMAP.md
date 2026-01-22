## Development Roadmap (TODO)

### A. Watchdog & Self-Healing (High Priority) __ongoing 01.2026

- Monitor freshness of VE.Bus input data
- Freeze or force a safe state if VE.Bus disappears
- Automatically recover when communication returns

---

### B. Grafana Dashboards (High Value)

Suggested dashboards:
- Grid ON % per day
- ON cycles per day
- ON duration histogram
- Switching reason distribution
- Before/after tuning comparisons

---

### C. Trim Output Wrapper (Optional)

Once long-term stability is confirmed:
- reduce wrapper to validation + deduplication only

The current implementation is intentionally conservative.

---

### D. Formalize State Snapshots

- Persist FSM state and last decision
- Enable post-mortem analysis after outages or reboots

---

### E. Reusable Victron FSM Pattern

- Package FSM + decision logic as a reusable Node-RED subflow
- Document clear input/output contracts
- Publish as a reference pattern for Victron ESS control

---

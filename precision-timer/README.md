# precision-timer

**Theme:** Resilient infra · real-time systems

The `PrecisionTimer` lives inside the [snipebay](../snipebay/) repo — see `snipebay/timer.py` and the [snipebay README](../snipebay/README.md) for the full write-up and code.

## Summary

A latency-compensated, NTP-aware countdown timer designed for sub-100ms accuracy:

1. **NTP offset calculation** at startup — measures the local clock's delta against an NTP pool and accounts for round-trip latency.
2. **3-phase countdown** — coarse `time.sleep()` → fine `time.sleep()` (small increments) → busy-wait (active spin) as the target approaches. Switches phases based on remaining time to balance CPU efficiency and precision.
3. **Calibrated for system load** — the fine/busy-wait thresholds are tunable to account for scheduler jitter under load.

### Achieved accuracy

Consistently within 80–120ms of the target timestamp across repeated runs, including under system load.

### Full repo

[github.com/alm-000/snipebay](https://github.com/alm-000/snipebay) — `timer.py`

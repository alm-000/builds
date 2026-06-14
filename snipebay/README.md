# snipebay

**Theme:** Resilient infra · real-time systems

## Problem

Winning timed online auctions requires placing an action within a precise, sub-second window at the end of each listing. Standard wall-clock timing drifts and is unreliable under system load. A daemon that exits on the first error or drops its queue on crash is useless for unattended overnight runs.

## Approach

Three components working together:

1. **PrecisionTimer** — NTP-offset calculation at startup, latency-compensated 3-phase countdown (coarse sleep → fine sleep → busy-wait), targeting sub-100ms accuracy against server time.
2. **SQLite-WAL daemon** — persistent job queue with WAL mode for crash safety; on restart, the journal replays and no queued actions are lost.
3. **Multi-strategy end-time scraper** — at least three independent extraction strategies per listing page; falls back gracefully if a layout variant is encountered.

## Architecture

```
snipebay/
├── timer.py    # PrecisionTimer: NTP offset + 3-phase countdown
├── daemon.py   # SQLite-WAL job scheduler with crash recovery
├── db.py       # Database layer: WAL config, schema, migrations
└── scraper.py  # Multi-strategy end-time extraction
```

## Real results / metrics

- Timer accuracy: consistently within 80–120ms of target across repeated runs.
- Daemon survived 12-hour unattended runs with zero lost queue entries across forced kills.
- Scraper: 3 extraction strategies provide resilience against minor layout changes.

## Links

Full repo (code + tests): [github.com/alm-000/snipebay](https://github.com/alm-000/snipebay)

See also: [precision-timer](../precision-timer/) — the timer module in isolation.

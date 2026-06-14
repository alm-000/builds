# trend-finder-lite

**Theme:** Resilient infra · AI/automation

*Write-up only — sanitised architecture description. No source code included.*

## Problem

Tracking fast-moving content trends at scale requires continuous data collection across multiple platforms. Naive polling loops are brittle: a single API error or rate-limit kills the run; unbounded parallelism triggers blocks; and hot-reloading configuration without restarts is essential for tuning without downtime.

## Approach

A distributed, queue-driven collection architecture with four design pillars:

### 1. Work queue (durable jobs)
All collection tasks are persisted in a job queue before execution. Workers claim jobs, process them, and release or dead-letter on failure. On crash or restart, unclaimed jobs are automatically re-queued — no work is lost and no duplicate processing occurs within the same window.

### 2. Runtime configuration
Targeting parameters (content categories, volume thresholds, timing) are loaded from a config store at job-claim time, not at process start. This means adjustments take effect on the next job cycle without a restart — critical for tuning collection in production.

### 3. Admission control
Each worker checks a set of admission rules before starting a collection batch: rate-limit headroom, time-of-day windows, and per-source quotas. Jobs that fail admission are re-queued with a delay rather than attempted and failed. This keeps the system well-behaved under pressure.

### 4. IP rotation
Outbound requests are distributed across a pool of IPs (mobile carrier NAT via a phone fleet, not proxies) to stay within per-IP rate limits at scale. The routing layer is abstracted from the collection workers — workers just call `fetch(url)`, and the transport layer handles IP selection.

## Architecture summary

```
[Scheduler] → [Job Queue (SQLite/Postgres)]
                     ↓
              [Worker Pool]
                ├── Admission check (rate, time, quota)
                ├── Config load (runtime params)
                ├── Collection (via IP-rotation transport)
                └── Result → [Storage] + [Queue: mark done]
```

## Scale

Designed for sustained, unattended operation collecting from multiple platforms simultaneously. Admission control and dead-lettering keep it stable over multi-day runs without manual intervention.

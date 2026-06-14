# pipeline-kit

**Theme:** AI/Automation · scrape→enrich→score

## Problem

Most enrichment and scoring pipelines are monolithic: one failure kills the run, and business rules are scattered across ad-hoc scripts. Rebuilding the same fan-out + weighted-score pattern for every new project wastes time and introduces drift between implementations.

## Approach

Extract the reusable skeleton into a small, dependency-light Python library:

- **Enrich fan-out** — call multiple data sources in parallel; merge results; collect failures without raising so the pipeline continues with partial data.
- **Weighted rule engine** — score any item against a rubric of weighted rules, applying partial credit and penalties; returns a normalised score with a breakdown.
- **SQLite-WAL job queue** — claim/release/dead-letter, crash-safe, zero infrastructure dependency.
- **Thin reporter** — dict → Markdown summary for quick review or notification.

## Architecture

```
pipeline_kit/
├── enrich.py    # fan-out: enrich(item, sources=[...]) -> dict
├── score.py     # rule engine: score(item, rubric) -> ScoredResult
├── queue.py     # SQLite-WAL job queue with dead-letter
└── report.py    # dict -> Markdown reporter
```

The pattern is intentionally generic. I've used it in vehicle intelligence (VIN + MOT + market-value fan-out), property research (4-API neighbourhood scoring), a CRM enrichment pipeline, and opportunity discovery — all with the same `enrich → score → report` skeleton.

## Real results / metrics

- Replaced 3 separate ad-hoc pipelines with a single shared library; new pipelines now take hours not days to wire up.
- Graceful degradation: a failed enrichment source drops from the merge without halting the run — useful when paid APIs have quota limits or intermittent failures.
- The SQLite-WAL queue has survived multi-hour unattended runs with crash recovery (journal replay on restart).

## Links

Full repo: [github.com/alm-000/pipeline-kit](https://github.com/alm-000/pipeline-kit)

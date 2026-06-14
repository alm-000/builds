# slack-crm-automation

**Theme:** Product/CRM/Slack ops · AI/automation

## Problem

Small product teams managing B2B pipelines via email + spreadsheets waste hours on manual triage: copying inbound enquiries into CRM, chasing unread quotes, and catching fake "opens" from email security scanners that poison engagement metrics.

## Approach

Three edge functions composable as a lightweight automation layer:

1. **Gmail push → CRM windowing** — instead of polling, subscribes to Gmail push notifications; batches and deduplicates messages within a sliding window to avoid double-processing on rapid delivery.
2. **Quote stripper** — removes quoted reply threads from email body before storage, keeping CRM records clean without manual editing.
3. **Bot-resistant email tracking** — open/click events filtered by: CIDR blocklist for known scanner ranges, Apple MPP heuristic (immediate open + no click = MPP prefetch), and time-based deduplication. Implemented without a third-party tracking service.
4. **Slack ops notifications** — structured Slack messages on pipeline events (new lead, stage change, overdue follow-up) with action buttons.

## Architecture

```
gmail-push/       # History windowing + dedup
_shared/
├── quote-stripper.ts   # Thread removal from email body
└── email-track/        # detectBot(): CIDR + MPP + dedup logic
slack-notify/           # Structured Slack message builder
```

These are reference modules, not a turnkey deploy. Each can be adapted to any Supabase + Slack + Gmail stack.

## Real results / metrics

- Bot-detection reduced false "open" events by ~40% in the first week, giving accurate read-receipt signals for follow-up timing.
- Gmail push windowing eliminated double-processing on all tested edge cases (rapid delivery, retry storms).
- Quote stripping kept CRM notes readable without any manual editing — previously the biggest copy-paste overhead.

## Links

Full repo (sanitised extracts): [github.com/alm-000/slack-crm-automation](https://github.com/alm-000/slack-crm-automation)

# builds

A broad showcase of work across automation, data pipelines, real-time systems, and full-stack product development. Each folder has a write-up (problem → approach → architecture → results) and, where applicable, one sanitised code excerpt.

**Profile:** [github.com/alm-000](https://github.com/alm-000) · **Site:** [alex-magee.com](https://alex-magee.com)

---

## AI/Automation

| Project | Description |
|---------|-------------|
| [pipeline-kit](./pipeline-kit/) | Reusable scrape→enrich→score→report backbone: fan-out enrichment, weighted rule engine, SQLite-WAL queue. Links to real repo. |
| [n8n-automations](./n8n-automations/) | Library of sanitised n8n workflows: lead enrichment, SEO writing, social script generation, CRM hygiene. Links to real repo. |
| [slack-crm-automation](./slack-crm-automation/) | Gmail push→CRM windowing, quote stripping, bot-resistant email tracking (no third party), Slack ops notifications. Links to real repo. |
| [seeka](./seeka/) | Opportunity discovery + research pipeline: parallel company research fan-out, configurable weighted scoring engine, priority triage. |
| [trend-finder-lite](./trend-finder-lite/) | Write-up only: distributed work-queue architecture for large-scale content trend collection — admission control, runtime config, IP rotation. |
| [video-pipeline](./video-pipeline/) | Template-driven FFmpeg renderer: deterministic branding overlays (logo, watermark, text), aspect-ratio conversion, endcard concat. |
| [dj-highlight-detector](./dj-highlight-detector/) | No-ML highlight detection from DJ sets: RMS/bass spectral analysis, beat-snapping, visual diversity selection via frame histogram. |
| [blender-studio](./blender-studio/) | Next build: Blender MCP pipeline for AI-prompted 3D asset generation (concept). |

## Scrape → Enrich → Score

| Project | Description |
|---------|-------------|
| [pipeline-kit](./pipeline-kit/) | The pattern as a library — see above. |
| [uk-neighbourhood](./uk-neighbourhood/) | 4-API UK neighbourhood report: crime Haversine filter, residential concern weighting, 0–100 safety score, Land Registry price trends. |
| [vehicle-intel](./vehicle-intel/) | Multi-source VIN decode: pure-Python structural decode + NHTSA + Vincario + UK vehicle data. Alerts for stolen/finance/write-off. |
| [seeka](./seeka/) | See above. |

## Product / CRM / Slack ops

| Project | Description |
|---------|-------------|
| [slack-crm-automation](./slack-crm-automation/) | See above. |
| [n8n-automations](./n8n-automations/) | See above. |
| [shopify-theme](./shopify-theme/) | Data-driven Liquid sections, idempotent Admin API product migration, section-first architecture. |

## Full-Stack

| Project | Description |
|---------|-------------|
| [personal-finance-sync](./personal-finance-sync/) | TrueLayer Open Banking integration: proactive OAuth token refresh, multi-provider connection model, no-pandas transaction pivots. |
| [shopify-theme](./shopify-theme/) | See above. |
| [video-pipeline](./video-pipeline/) | See above. |

## Resilient Infra

| Project | Description |
|---------|-------------|
| [snipebay](./snipebay/) | Real-time auction tooling: NTP-offset PrecisionTimer (3-phase countdown), SQLite-WAL daemon with crash recovery, multi-strategy scraper. Links to real repo. |
| [precision-timer](./precision-timer/) | The PrecisionTimer in isolation — lives in the snipebay repo. |
| [trend-finder-lite](./trend-finder-lite/) | See above. |

---

## Real repos (full code)

- [github.com/alm-000/snipebay](https://github.com/alm-000/snipebay) — real-time systems + resilient scraping
- [github.com/alm-000/pipeline-kit](https://github.com/alm-000/pipeline-kit) — enrich/score/queue library
- [github.com/alm-000/slack-crm-automation](https://github.com/alm-000/slack-crm-automation) — Gmail→CRM + email tracking + Slack
- [github.com/alm-000/n8n-automations](https://github.com/alm-000/n8n-automations) — n8n workflow library

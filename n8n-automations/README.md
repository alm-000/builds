# n8n-automations

**Theme:** AI/automation · product/CRM/Slack ops

## Problem

Repetitive operational tasks — lead enrichment, SEO content drafts, social script generation, CRM hygiene — consume disproportionate time when done manually. Most are well-structured enough to automate but not worth building bespoke infrastructure for.

## Approach

A library of n8n workflows covering common automation patterns, each with a README describing the pattern and the logic. The value is the *design*, not a ready-to-import bundle — credentials, IDs, and endpoints are stripped.

## Workflows included

| Workflow | Pattern |
|----------|---------|
| `lead-enrichment` | Multi-source company enrichment (web search + Companies House + Glassdoor signals) → scored lead record |
| `lead-discovery` | Automated discovery of inbound leads from multiple channels → unified inbox |
| `seo-writer` | Keyword brief → structured SEO article draft via AI, with internal linking |
| `script-writer` | Topic → short-form social video script (hook / body / CTA structure) |
| `crm-hygiene` | Periodic stale-record detection + owner notification |
| `ad-tools` | UTM tagging, ad performance pull → spreadsheet |

## Architecture

Each workflow ships as:
- Sanitised JSON (credentials and IDs removed — reference the pattern, not a runnable import)
- A `README.md` explaining inputs, outputs, and the automation logic

```
n8n-automations/
├── lead-enrichment/
│   ├── workflow.json
│   └── README.md
├── lead-discovery/
│   ├── workflow.json
│   └── README.md
├── seo-writer/
│   ├── workflow.json
│   └── README.md
└── ... (each workflow follows the same pattern)
```

## Real results / metrics

- Script-writer workflow reduced first-draft time from ~45 min to under 5 min per video.
- Lead enrichment fan-out surfaces funding stage, headcount trend, and Glassdoor signal in a single pass — previously ~20 min of manual research per lead.

## Links

Full repo: [github.com/alm-000/n8n-automations](https://github.com/alm-000/n8n-automations)

# B2B Scoring Model

Fresh/generic — 3-factor weighted scoring (size / fit / complexity) + A/B/C tiers. The scoring engine pattern is implemented in [pipeline-kit](https://github.com/alm-000/pipeline-kit).

---

## Problem

Partnerships and B2B outreach have a common top-of-funnel failure: finding the right targets takes hours of manual research, enrichment is inconsistent, and there is no repeatable way to prioritise who to pursue this week. Without a scoring model, effort goes to whoever is most visible rather than whoever is most valuable.

The goal: a systematic, explainable way to rank targets so outreach focuses on the right orgs, sequenced by opportunity.

## Approach

The model is a three-factor weighted score that maps each target to an A/B/C tier. Before automating, validated manually: took a small batch of targets, scored them by hand against the rubric, ran outreach, and tracked outcomes by tier. When A-tier targets outperformed B and C on reply and meeting rate, automated the full pipeline.

### The three factors

**1. Opportunity size (0–5)**

Proxies for potential revenue or reach from this partnership.

| Signal | Weight |
|--------|--------|
| Estimated audience / customer base size | 40% |
| Event frequency or transaction volume | 35% |
| Geographic footprint (local vs. national vs. international) | 25% |

Scoring scale: 0 = no signal / unquantifiable, 5 = large verifiable reach.

**2. Fit (0–5)**

How well the target's audience and context matches the use case.

| Signal | Weight |
|--------|--------|
| Audience overlap with target demographic | 45% |
| Use-case alignment (do they already do something adjacent?) | 35% |
| Content / brand style compatibility | 20% |

Scoring scale: 0 = poor fit / wrong audience, 5 = strong alignment on all dimensions.

**3. Complexity (0–5, where lower is better)**

Decision friction and delivery complexity. This factor is inverted in the final score — a lower complexity score is better.

| Signal | Weight |
|--------|--------|
| Decision-maker accessibility (direct contact vs. committee) | 40% |
| Contractual / legal overhead (standard terms vs. bespoke) | 35% |
| Fulfilment complexity (how hard is it to actually deliver?) | 25% |

Scoring scale: 0 = trivially easy to close and deliver, 5 = high friction and complexity.

### Composite score

```
composite = (size * 0.40) + (fit * 0.40) + ((5 - complexity) * 0.20)
```

Complexity is inverted (`5 - complexity`) so all three factors point the same direction: higher is better. Weights reflect that size and fit are primary; complexity is a tiebreaker and penalty.

### Tier mapping

| Score range | Tier | Action |
|-------------|------|--------|
| 3.5 – 5.0 | A | Prioritise — personal outreach, researched message |
| 2.0 – 3.4 | B | Sequence — templated but personalised outreach |
| 0.0 – 1.9 | C | Defer — low priority queue or exclude |

Rough distribution target: ~15% A-tier, ~35% B-tier, ~50% C-tier. If A-tier is larger than this, tighten the thresholds — protecting time is the point.

### Validation

Track reply rate and meeting rate by tier after each outreach batch. If A-tier isn't outperforming B+C by a meaningful margin, review the factor weights — the likely culprits are over-weighting a proxy signal that doesn't correlate with intent, or under-weighting a friction signal that makes even high-opportunity targets hard to close.

### Output record

Each scored target produces a record with:

```json
{
  "org": "<name>",
  "tier": "A",
  "composite_score": 4.1,
  "size_score": 4,
  "fit_score": 5,
  "complexity_score": 2,
  "why_this_scored_high": "Strong audience overlap + active events programme + direct partnership contact found",
  "enrichment": {
    "contact_route": "email",
    "contact_email": "partnerships@example.com",
    "region": "UK",
    "size_proxy": "15k followers + 12 events/year"
  },
  "outreach_status": "pending"
}
```

The `why_this_scored_high` field is auto-generated based on which sub-signals drove the score. It feeds directly into personalisation for outreach.

## Scoring engine

The weighting and tier logic above can be implemented with the [pipeline-kit](https://github.com/alm-000/pipeline-kit) weighted rule engine, which provides configurable factor weights, score aggregation, and tier assignment out of the box.

```python
# Example using pipeline-kit's WeightedScorer
from pipeline_kit.scoring import WeightedScorer

scorer = WeightedScorer(
    factors={
        "size":       {"weight": 0.40, "max": 5},
        "fit":        {"weight": 0.40, "max": 5},
        "complexity": {"weight": 0.20, "max": 5, "invert": True},
    },
    tiers=[
        {"label": "A", "min": 3.5},
        {"label": "B", "min": 2.0},
        {"label": "C", "min": 0.0},
    ]
)

record = scorer.score({
    "size": 4,
    "fit": 5,
    "complexity": 2,
})
# → {"composite": 4.2, "tier": "A", ...}
```

See [pipeline-kit](https://github.com/alm-000/pipeline-kit) for the full scoring library including fanout enrichment, queue management, and reporting.

## Stack

- **pipeline-kit** — weighted scoring engine, tier assignment
- **n8n** — discovery and enrichment orchestration upstream
- **Supabase** — lead storage, score history, outreach tracking

## Results

- Time-to-ranked-target-list dropped from hours of manual research to a repeatable automated feed
- Target selection became consistent and explainable — every A-tier lead has a documented reason
- A-tier targets consistently outperformed B and C on reply rate and meeting rate once weights were tuned
- Higher quality conversations downstream because outreach focused on the right orgs at the right time

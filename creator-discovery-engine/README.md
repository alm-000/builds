# Creator Discovery Engine

Write-up only — the scoring logic lives in n8n and is not separately extractable as a clean snippet.

---

## Problem

Creator sourcing was driven by manual Instagram browsing and intuition. Two failure modes: no reliable view of what formats were winning right now, and no scalable way to build a creator distribution network. Manual scrolling can't keep pace with a growing paid acquisition programme that needed a constant stream of well-briefed partners.

## Approach

Built a two-output pipeline: content intelligence (what formats to brief against) and creator scoring (who to contact first). Before automating, validated the concept manually — took a small set of creators, tracked signals by hand, and produced a first shortlist. When that output drove better outreach outcomes than intuition, automated the full loop.

### Step 1 — Creator discovery

Seeded niche clusters (festival/rave, dance, gym/running-adjacent) and expanded via similar-accounts logic. Steady inflow without manual scrolling.

### Step 2 — Data ingestion via RocketAPI

Pulled reel-level data (views, engagement, recency, captions, hashtags) and creator-level data (follower count, posting volume, bio signals, contact route indicators) at scale via the RocketAPI endpoints.

### Step 3 — Dual scoring model

**Content intelligence scoring**

Goal: surface formats with momentum, not just historical virality.

Signals:
- Engagement velocity: traction normalised to reel age
- Recency weighting: exponential decay to prioritise what's active now
- Repeatability: pattern appears across multiple creators, not a single spike

Output: ranked feed of reels grouped into format buckets — hooks, editing rhythms, themes, tropes.

**Creator network scoring**

Goal: prioritise creators who are both effective and actionable.

Signals:
- Consistency: median and variance across recent posts (penalises single-spike creators)
- Engagement quality proxies: comment/share signals where available
- Brand fit: niche alignment, content style, location
- Practicality: contactability, signs of collab openness

Output: ranked creator shortlist with score band and documented "why them" reasons. A-tier got personal outreach, lower tiers deprioritised.

### Step 4 — Dashboard and CRM sync

Score, tags, contact route, outreach status, and "why them" rationale synced into a CRM table. Closed the loop: insight → shortlist → outreach → outcomes → refine weights.

### Iteration

Adjusted signal weights when ranking didn't correlate with outcomes. Added volatility controls to stop single-spike creators dominating. Exclusion rules for bot accounts and repost farms. Niche-specific tagging logic for better segmentation.

## Stack

- **RocketAPI** — reel and creator data ingestion
- **n8n** — scoring and enrichment pipeline, deduplication, error handling
- **Supabase** — creator records, scores, and outreach history
- **CRM sync** — structured reason codes and next-action fields

## Results

- Creator sourcing moved from manual browsing to an automated, ranked shortlist
- Content briefs grounded in observed winning formats rather than intuition
- Effort allocation improved — more time on high-quality, high-fit creators
- Competitive edge in organic reach through better creator selection and better-briefed content

# Directory Discovery Engine

Write-up only — the pipeline is n8n-orchestrated with no single clean extractable snippet.

---

## Problem

Every outreach and CRM workflow starts with the same bottleneck: a list of URLs or site names, and no structured way to know what they are, who runs them, or whether they're worth pursuing. Doing that manually is slow and inconsistent. Without a clean upstream layer, enrichment and outreach workflows run on garbage data.

When building a B2B outreach pipeline targeting festivals, venues, gyms, and retailers, the first constraint wasn't what to say — it was how to find and classify targets at scale. Raw URLs from searches and scrapers are noisy: wrong category, wrong geography, inactive sites, already-contacted.

## Approach

The discovery engine takes a raw list of URLs as input and outputs structured, enriched records ready for downstream scoring and outreach.

### Pipeline stages

**1. Fetch and parse**

For each URL: fetches the page, extracts title, meta description, visible body text, and structural signals — presence of contact pages, about pages, social links, booking pages. Handles redirects, timeouts, and rate-limiting with retry logic and a dead-letter queue for persistent failures.

**2. Classify site type**

Maps each site to a category (festival, venue, gym, retailer, blog, irrelevant) using a two-stage approach:
- Pattern matching on keywords, URL structure, and meta signals for clear cases
- LLM-assisted classification for ambiguous results, with a confidence score threshold below which the record is flagged for manual review rather than auto-classified

**3. Extract metadata**

Name, location signals, contact email patterns, social handles, partner/booking page indicators.

**4. Filter and deduplicate**

Removes sites that don't match the target niche, are inactive, or have already been processed. Dedupe key is domain-normalised (strips www, handles redirects). Nothing enters the downstream CRM twice.

**5. Sync to structured store**

Every record lands with: classification, metadata, processing status, dedupe key, and timestamp. A status field (`discovered`, `enriched`, `enrichment_failed`, `excluded`) makes the pipeline replayable — reruns skip already-processed records cleanly.

## Stack

- **n8n** — orchestration, retry logic, dead-letter queue
- **Supabase** — structured lead storage, deduplication, status tracking
- **LLM classification** — for ambiguous site types

## Results

- Removed the manual URL triage step entirely from the outreach workflow
- Became the upstream data layer for the CRM engine, creator discovery system, and festival partnership outreach
- Classification accuracy high enough that manual review was only needed for genuinely ambiguous cases
- The pattern (scrape → classify → enrich → score → act) generalises to any B2B outreach scenario built from a cold list

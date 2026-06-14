# seeka

**Theme:** AI/automation · scrape→enrich→score

## Problem

Evaluating opportunities at volume (jobs, partnerships, investment leads — any scored-ranking problem) requires consistent criteria applied across many candidates. Ad-hoc evaluation is slow, inconsistent, and doesn't scale beyond a handful of items per week.

## Approach

An opportunity discovery and research pipeline with three stages:

### 1. Ingest

Inbound signals (email digests, API feeds) are parsed and normalised into a unified candidate schema. Duplicates are removed by URL/identifier before storage.

### 2. Enrich + Research

For each candidate, a parallel research fan-out collects:
- Web search results (funding, headcount, news) via a search API
- Companies House registration data (UK incorporation date, company status)
- Layoffs signal (site:layoffs.fyi check)
- Glassdoor rating and CEO approval signals

The raw research data is stored alongside the candidate record for audit and prompt context.

### 3. Score + Prioritise

A two-part scoring model:

**Opportunity score** — weighted preference matching against a configurable rule set:
- Exclusion rules (hard blockers): if any exclusion term matches, the candidate is rejected immediately.
- Inclusion rules: role type and location filters — both must match if rules are defined.
- Preference scoring: weighted `Prefer` rules (positive or negative weight) score how well the candidate matches preferences. Normalised to 1–10.

**Company score** — separate evaluation of the employer/company (stage, growth signal, culture signals).

**Final score** = 60% opportunity score + 40% company score, mapped to High / Medium / Low priority bands (9+, 7–8, <7).

The rule set is stored in a database table, not hard-coded — criteria are updated without code changes.

## Architecture

```
[Ingest] → email/API → parse → deduplicate → queue
[Enrich] → parallel fan-out: web search + Companies House + layoffs check
[Score]  → checkExcludes → checkIncludes → calculatePreferScore
                                         → calculateFinalScore → assignPriority
[Output] → structured record + research summary → downstream notification
```

## Real results / metrics

- Parallel research fan-out: ~4–6 seconds per candidate across 5 concurrent sources.
- Configurable rule set: criteria updated in a table with no redeploy — takes effect on the next pipeline run.
- Priority bands allow triage in seconds rather than reading every item in full.

## Snippet

Scoring engine — exclusion gate, inclusion filter, preference weighting, and final score:

```typescript
export interface JobPreference {
  active: boolean;
  includeExclude: "Include" | "Exclude" | "Prefer";
  category: string;
  value: string;
  weight: number;
}

/** Hard gate: if any exclusion term matches, reject immediately. */
export function checkExcludes(
  text: string,
  location: string,
  prefs: JobPreference[],
): string | null {
  const combined = (text + " " + location).toLowerCase();
  for (const p of prefs) {
    if (!p.active || p.includeExclude !== "Exclude") continue;
    if (combined.includes(p.value.toLowerCase())) return p.value;
  }
  return null;
}

/** Inclusion filter: role type AND location must match (if rules are defined). */
export function checkIncludes(
  title: string,
  location: string,
  prefs: JobPreference[],
): boolean {
  const roleTitles = prefs.filter(p => p.active && p.includeExclude === "Include" && p.category === "Role Titles");
  const locations  = prefs.filter(p => p.active && p.includeExclude === "Include" && p.category === "Location");
  const roleMatch  = roleTitles.length === 0 || roleTitles.some(p => title.toLowerCase().includes(p.value.toLowerCase()));
  const locMatch   = locations.length  === 0 || locations.some(p => location.toLowerCase().includes(p.value.toLowerCase()));
  return roleMatch && locMatch;
}

/** Weighted preference score: 1–10. Normalised against max possible weight. */
export function calculatePreferScore(
  text: string,
  location: string,
  prefs: JobPreference[],
): number {
  const combined  = (text + " " + location).toLowerCase();
  const prefers   = prefs.filter(p => p.active && p.includeExclude === "Prefer");
  const maxWeight = prefers.reduce((sum, p) => sum + Math.abs(p.weight), 0);
  if (maxWeight === 0) return 5;

  const matched = prefers.reduce((sum, p) =>
    combined.includes(p.value.toLowerCase()) ? sum + p.weight : sum, 0);

  return Math.max(1, Math.min(10, Math.round((matched / maxWeight) * 10)));
}

/** Final score: 60% opportunity + 40% company. Priority: 9+ High, 7+ Medium, <7 Low. */
export function calculateFinalScore(opportunityScore: number, companyScore: number): number {
  return Math.round((opportunityScore * 0.6 + companyScore * 0.4) * 10) / 10;
}

export function assignPriority(score: number): string {
  if (score >= 9) return "High";
  if (score >= 7) return "Medium";
  return "Low";
}
```

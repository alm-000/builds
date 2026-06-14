# shopify-theme

**Theme:** Full-stack · product/CRM/Slack ops

## Problem

Most Shopify themes are configured through the admin UI, which makes section layout and content brittle: changes require manual clicks per collection, product migration is non-idempotent (running it twice creates duplicates), and theme sections contain hard-coded assumptions about product structure.

The goal was a Shopify storefront where content layout is data-driven, section architecture is composable, and product migration is safe to re-run.

## Approach

A custom Shopify theme built in Liquid + JSON templates with three architectural patterns:

### 1. Data-driven editorial sections

`editorial-categories.liquid` renders a content grid from a JSON settings schema rather than hard-coded markup. Each category card (image, label, link) is defined as a section block in the theme customiser — editors add/remove/reorder cards without touching code.

`collection-feature.liquid` uses the same pattern for featured collection spotlights, pulling product images, titles, and pricing dynamically from the Storefront API rather than baking them into the template.

### 2. Idempotent product migration

`migrate-products.js` (Node.js, Admin API) handles bulk product creation and update without duplication. Strategy:

- On each run, fetch existing products by handle.
- If a handle exists: update the record (price, description, metafields) without creating a duplicate.
- If not: create new.
- All operations are logged with a diff summary — what changed, what was skipped, what was created.

Running `migrate-products.js` twice against the same source data produces the same store state both times.

### 3. Section-first architecture

Theme sections are self-contained: each section owns its schema, its Liquid template, and its CSS. No shared global styles that sections depend on — each section renders correctly in isolation. This made iterating on individual page types fast without risking breakage elsewhere.

## Architecture

```
theme/
├── sections/
│   ├── editorial-categories.liquid  # data-driven category grid
│   ├── collection-feature.liquid    # dynamic collection spotlight
│   └── ... (other sections)
├── templates/
│   └── *.json                       # page-level section composition
└── scripts/
    └── migrate-products.js          # idempotent Admin API product sync
```

## Real results / metrics

- Idempotent migration: zero duplicate products across 10+ re-runs during content iteration.
- Section-first architecture: isolated section development meant zero regressions when adding new page types.
- Data-driven editorial grid: content team could reorder and add category cards via the Shopify customiser — no developer involvement for editorial updates.

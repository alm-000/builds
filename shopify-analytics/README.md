# Shopify Analytics Dashboard System

Snippet: generic SQL — contribution margin by SKU query pattern, written fresh (no private identifiers).

---

## Problem

Shopify's native analytics show revenue. They do not show:
- Whether that revenue was profitable at the product level
- Which SKUs are driving or destroying margin
- When stock will run out
- Whether paid acquisition is building a returning customer base or churning through one-time buyers

Running a DTC brand with paid acquisition means constant trade-off decisions: should I scale this ad set, or is the margin too thin? Is this SKU worth reordering, or is it being returned too often? Shopify's dashboard cannot answer any of these questions. Decisions defaulted to intuition, which is slow to calibrate and easy to get wrong when ad spend, COGS, fulfilment costs, and return rates are all moving simultaneously.

## Approach

A reporting layer built on the Shopify Admin API (GraphQL) and a relational store, surfaced through dashboards.

### Contribution margin by SKU and channel

Revenue minus COGS minus fulfilment minus platform fees minus attributed ad spend, broken down at the product-variant level. The number that tells you whether selling more of something is a good idea.

### Cohort behaviour

Customer cohorts grouped by acquisition month, showing repeat purchase rate, average order value over time, and LTV curves. Clear view of whether paid acquisition is building a returning customer base or churning through one-time buyers.

### Stock forecasting

Based on current sell-through rate (7/14/30-day rolling average) and supplier lead times, projects when each SKU will run out. Reorder signal is timing-driven, not reactive.

### Spend vs revenue

Ad spend by campaign aggregated alongside revenue by channel, with a lag adjustment to account for the delay between spend and attributed conversion. Avoids the attribution flattery in native Meta reporting.

### Product performance

Units sold, return rate, margin contribution, and sell-through rate by variant. Makes it obvious which SKUs are performing and which are tying up cash.

## Snippet — contribution margin by SKU (SQL)

Generic query pattern for the core metric. Assumes orders, line items, products, and cost data are synced from the Shopify API into a relational store (Postgres / Supabase). Adapt table names and column names to your schema.

```sql
-- Contribution margin by SKU
-- Revenue - COGS - fulfilment - platform fee - attributed ad spend
--
-- Assumes:
--   orders(id, created_at, financial_status, channel)
--   order_line_items(order_id, variant_id, quantity, unit_price, discount_amount)
--   product_variants(id, sku, title, product_title, cogs_unit)
--   fulfilment_costs(variant_id, cost_per_unit)         -- your cost data
--   ad_spend_by_channel(date, channel, spend)           -- from Meta/Google exports
--   platform_fee_rate CONSTANT (e.g. 0.02 for 2%)

WITH order_lines AS (
    SELECT
        oli.variant_id,
        pv.sku,
        pv.title        AS variant_title,
        pv.product_title,
        DATE_TRUNC('month', o.created_at) AS month,
        o.channel,
        SUM(oli.quantity)                            AS units_sold,
        SUM((oli.unit_price - oli.discount_amount) * oli.quantity) AS net_revenue,
        SUM(pv.cogs_unit * oli.quantity)             AS total_cogs,
        SUM(COALESCE(fc.cost_per_unit, 0) * oli.quantity) AS total_fulfilment
    FROM orders o
    JOIN order_line_items oli ON oli.order_id = o.id
    JOIN product_variants  pv  ON pv.id = oli.variant_id
    LEFT JOIN fulfilment_costs fc ON fc.variant_id = oli.variant_id
    WHERE o.financial_status IN ('paid', 'partially_refunded')
      AND o.created_at >= NOW() - INTERVAL '90 days'
    GROUP BY
        oli.variant_id, pv.sku, pv.title, pv.product_title, month, o.channel
),
channel_ad_spend AS (
    -- Apportion daily ad spend across units sold that day by channel
    SELECT
        DATE_TRUNC('month', date) AS month,
        channel,
        SUM(spend)                AS monthly_spend
    FROM ad_spend_by_channel
    WHERE date >= NOW() - INTERVAL '90 days'
    GROUP BY month, channel
),
revenue_by_channel AS (
    SELECT month, channel, SUM(net_revenue) AS channel_revenue
    FROM order_lines
    GROUP BY month, channel
),
margin_calc AS (
    SELECT
        ol.sku,
        ol.variant_title,
        ol.product_title,
        ol.month,
        ol.channel,
        ol.units_sold,
        ol.net_revenue,
        ol.total_cogs,
        ol.total_fulfilment,

        -- Platform fee (e.g. Shopify 2%)
        ROUND(ol.net_revenue * 0.02, 2)              AS platform_fee,

        -- Attributed ad spend: apportion by revenue share within channel+month
        ROUND(
            COALESCE(cas.monthly_spend, 0)
            * (ol.net_revenue / NULLIF(rbc.channel_revenue, 0)),
            2
        )                                            AS attributed_ad_spend,

        -- Contribution margin
        ol.net_revenue
            - ol.total_cogs
            - ol.total_fulfilment
            - ROUND(ol.net_revenue * 0.02, 2)
            - ROUND(
                COALESCE(cas.monthly_spend, 0)
                * (ol.net_revenue / NULLIF(rbc.channel_revenue, 0)),
                2
              )                                      AS contribution_margin
    FROM order_lines ol
    LEFT JOIN channel_ad_spend    cas ON cas.month = ol.month AND cas.channel = ol.channel
    LEFT JOIN revenue_by_channel  rbc ON rbc.month = ol.month AND rbc.channel = ol.channel
)
SELECT
    sku,
    variant_title,
    product_title,
    month,
    channel,
    units_sold,
    ROUND(net_revenue, 2)          AS net_revenue,
    ROUND(total_cogs, 2)           AS cogs,
    ROUND(total_fulfilment, 2)     AS fulfilment_cost,
    ROUND(platform_fee, 2)         AS platform_fee,
    ROUND(attributed_ad_spend, 2)  AS ad_spend,
    ROUND(contribution_margin, 2)  AS contribution_margin,
    ROUND(
        contribution_margin / NULLIF(net_revenue, 0) * 100,
        1
    )                              AS contribution_margin_pct
FROM margin_calc
ORDER BY month DESC, contribution_margin DESC;
```

**What the snippet shows:** revenue attribution split by channel, COGS + fulfilment + platform fee + ad spend deduction at variant level, revenue-share-weighted ad spend apportionment. The query pattern is the core analytical primitive — everything else (cohort LTV, sell-through, forecasting) builds on the same underlying order and cost tables.

## Stack

- **Shopify Admin API (GraphQL)** — order, product, and inventory data ingestion
- **Supabase / Postgres** — analytical store, SQL metric layer
- **n8n** — scheduled sync from Shopify API to relational store

## Results

- Replaced manual end-of-week reporting with always-on dashboards
- Stock forecasting reduced both stockouts and overstock — reorder decisions became timing-driven
- Contribution margin visibility changed ad spend evaluation: campaigns profitable on ROAS were sometimes margin-negative once fulfilment costs were included
- Data model is reusable for any DTC Shopify store — metric definitions are not product-specific

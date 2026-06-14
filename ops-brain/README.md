# Ops Brain

Write-up only — fulfilment logic is embedded in n8n workflows and Supabase functions tied to a private operational stack.

---

## Problem

Early in a DTC brand's life, operations are invisible. No reliable view of where orders sit in the supply chain, when stock is going to run low, which suppliers are creating problems, or whether a parcel is lost between warehouse and doorstep. Problems surface via customer complaints, not system alerts. Firefighting consumes time that should go toward growth.

Three specific failure modes drove this build:

1. **Fulfilment anomalies** were invisible until a customer chased. There was no automated check against expected courier scan windows.
2. **Stock ran out during active paid campaigns** because reorder decisions were reactive ("we're out") rather than forecast-driven.
3. **Supplier quality issues** appeared inconsistently — no structured record of which suppliers were creating problems.

## Approach

Mapped the full supply chain end to end: manufacturer → packaging → storage → pick-and-pack → courier scan → delivery. At each handoff, identified where data existed, where it didn't, and where delays clustered. Then instrumented each stage.

### Fulfilment anomaly detection

Automated checks against courier scan data to flag orders that have missed expected scan windows. Anomalies surfaced in a dashboard rather than waiting for customer contact. Exception-based — only items needing attention appear, not a full order log.

Logic: per carrier, define expected scan windows by shipping zone (domestic, EU, international). For each order in `dispatched` state, check time-since-last-scan against zone threshold. Flag as `scan_overdue` at 1× threshold, `at_risk` at 1.5×, `escalate` at 2×.

### Stock forecasting and reorder triggers

Demand forecast built on rolling sales velocity (7, 14, 30-day windows), seasonality inputs, marketing calendar signals (expected ad scaling), and supplier lead times per SKU.

Output per SKU:
- **Projected stockout date** at current velocity
- **Reorder point** (stockout date minus lead time, minus safety buffer)
- **Reorder quantity** (to target position, accounting for campaign calendar)

Automated alerts when projected stockout date crosses a threshold. Reorder task created automatically with supplier contact, last price, and quantity recommendation.

Tradeoff managed: safety stock ties up cash. Threshold calibration balanced cash flow against the cost of stockouts during active paid media — losing a sale at peak spend is worse than holding extra inventory.

### Supplier scorecards

Structured records of supplier performance across quality (defect rate at intake QC), lead time (actual vs. quoted), and communication reliability (response time to quote requests and queries). Patterns visible over time — makes supplier conversations evidence-based.

### QC intake workflow

Standardised intake inspection for new stock arrivals with documented pass/fail criteria per product type. Issues caught at intake rather than at returns. Defect rate tracked per supplier per run.

### Customer case pipeline

Open customer service cases tracked with status and resolution tracking so nothing fell through the cracks during high-volume periods. Cases visible and assignable rather than buried in email threads.

## Stack

- **n8n** — workflow orchestration for alerts, reorder triggers, fulfilment exception flagging
- **Supabase** — inventory tracking, supplier records, customer case pipeline
- **Courier API integration** — scan data ingestion for anomaly detection

## Results

- Missing and delayed orders reduced significantly through proactive anomaly detection
- Fulfilment errors cut through QC intake process improvement
- Product quality consistency improved across manufacturing runs
- Customer service load dropped as proactive detection replaced reactive complaint handling
- Stock planning moved from reactive to forecast-driven — reorder decisions became timing-driven
- Business became operable without the founder managing ops day-to-day

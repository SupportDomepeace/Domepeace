# Profit Engine Dashboard
## Product Requirements Document (PRD) + Technical Specification

## 1) Executive Summary
The **Profit Engine Dashboard** is a near-real-time analytics product for Shopify DTC brands that surfaces **true unit economics per order** and by marketing channel. It answers three core business questions:

1. **What is contribution profit and net profit per order today?**
2. **What CAC can we pay for bundles vs single SKUs and still be profitable?**
3. **Which channels remain profitable after shipping, packaging, refunds, and OpEx allocation?**

The system ingests operational and financial signals from Shopify, shipping providers, SKU COGS tables, packaging rules, and ad spend/conversion data. It computes normalized per-order profitability metrics and powers an interactive dashboard with anomaly alerts.

---

## 2) Goals, Non-Goals, and Success Metrics

### 2.1 Goals
- Deliver one source of truth for **order-level profitability**.
- Calculate **contribution profit/order** and **net profit/order** with clear formula definitions.
- Provide **break-even CAC** by SKU and bundle.
- Expose **channel-level profit** with date/channel/product filters.
- Quantify **refund/return impact** on margins.
- Support near-real-time updates (intra-day freshness).

### 2.2 Non-Goals (MVP)
- GAAP-grade accounting close and reconciliation workflows.
- Full MTA attribution modeling across all channels.
- Warehouse/inventory forecasting.
- LTV prediction or cohort retention analytics.

### 2.3 Success Metrics
- **Metric accuracy:** ±1% vs manually validated sample of 200 orders.
- **Freshness SLA:** 95% of records updated within 30 minutes of source changes.
- **Adoption:** daily usage by growth + finance stakeholders.
- **Decision quality:** weekly CAC bid guardrails used for media budget planning.

---

## 3) Users & Jobs-To-Be-Done

### Primary users
- **Founder/GM:** needs daily “are we making money today?” answer.
- **Performance marketer:** needs channel and campaign CAC guardrails.
- **Finance/operator:** needs defensible order-level profit logic and refund impact.

### Core jobs
- Inspect profit at order/SKU/bundle/channel granularity.
- Detect margin anomalies quickly.
- Set max CAC targets per product mix.
- Explain profit changes due to shipping, refunds, discounts, and OpEx allocation.

---

## 4) Business Definitions

- **Contribution Profit (CP):** Revenue net of variable costs (discounts, COGS, shipping label, packaging, payment fees, refund impact, variable transaction costs).
- **Net Profit (NP):** Contribution profit minus allocated operating expenses (OpEx).
- **Break-even CAC:** Maximum customer acquisition cost allowable before contribution profit (or net profit) reaches zero.
- **Order Date vs Financial Date:**
  - Operational metrics default to `order_created_at`.
  - Refund impact can be evaluated on `refund_processed_at` and/or re-attributed to original order date via configurable policy.

---

## 5) Source Systems & Inputs

### 5.1 Shopify Orders API
Include and persist:
- Order headers: `order_id`, `created_at`, `processed_at`, `financial_status`, `fulfillment_status`, `currency`, `presentment_currency`, `channel`.
- Monetary components: gross line item totals, discounts, tax, shipping charged to customer, duties if present.
- Line items: `sku`, `variant_id`, quantity, unit price, per-line discounts, tax allocations.
- Refund objects: refunded amount, refunded tax, refunded shipping, restock type, refund line items.
- Transactions (optional if available): payment processor fees or gateway type.

### 5.2 Shipping/Labels Source
From Shippo/ShipStation/EasyPost/CSV:
- `carrier`, `service_level`, `label_cost`, `tracking_number`, `shipment_id`, `order_reference`.
- Handle split shipments (multiple labels per order).
- Distinguish reshipments vs original fulfillment labels.

### 5.3 COGS Table by SKU (effective-dated)
- `sku`, `effective_start_date`, `effective_end_date`, `unit_cogs`.
- Optional `bundle_sku` overrides.
- Versioned updates to support historical re-runs.

### 5.4 Packaging Cost Model
- Flat per order or rule-driven by SKU/bundle/weight/box type.
- Rule priority and effective date support.

### 5.5 Ad Spend + Conversions
- Native connectors (Meta/Google) or manual CSV import.
- Fields: `date`, `channel`, `campaign`, `adset` (optional), `spend`, `clicks`, `conversions/orders`.
- Minimum required for MVP: daily spend + attributed orders by channel.

---

## 6) Canonical Data Model

> Grain strategy:
> - `fact_order_finance`: **one row per order** (final modeled output).
> - `fact_order_line_finance`: one row per order line item for SKU/bundle analytics.
> - `fact_channel_day`: one row per day x channel for spend/profit rollups.

### 6.1 Dimension Tables

#### `dim_order`
| column | type | key | notes |
|---|---|---|---|
| order_id | string | PK | Shopify order id |
| order_number | string |  | human-readable |
| created_at | timestamp |  | order timestamp |
| channel | string |  | web/shop/app/marketplace |
| customer_id | string |  | nullable |
| currency | string |  | original currency |
| fx_rate_to_usd | decimal(18,6) |  | normalized reporting |
| is_test_order | boolean |  | exclusion flag |

Sample rows:
| order_id | order_number | created_at | channel | customer_id | currency | fx_rate_to_usd | is_test_order |
|---|---|---|---|---|---|---|---|
| 1001 | #5001 | 2026-01-18 10:02:00 | Meta | C_3001 | USD | 1.000000 | false |
| 1002 | #5002 | 2026-01-18 10:11:00 | Google | C_3002 | USD | 1.000000 | false |

#### `dim_sku_cogs`
| column | type | key | notes |
|---|---|---|---|
| sku | string | PK(partial) | |
| effective_start_date | date | PK(partial) | inclusive |
| effective_end_date | date |  | exclusive/null=open |
| unit_cogs | decimal(12,4) |  | landed unit cost |
| bundle_component_ratio | decimal(12,6) |  | optional for bundles |

Sample rows:
| sku | effective_start_date | effective_end_date | unit_cogs | bundle_component_ratio |
|---|---|---|---|---|
| SKU_A | 2026-01-01 | null | 8.2000 | 1.000000 |
| SKU_B | 2026-01-01 | null | 12.5000 | 1.000000 |
| BUNDLE_AB | 2026-01-01 | null | 18.9000 | 1.000000 |

#### `dim_packaging_rules`
| column | type | key | notes |
|---|---|---|---|
| rule_id | string | PK | |
| effective_start_date | date |  | |
| effective_end_date | date |  | |
| applies_to | string |  | ORDER/SKU/BUNDLE |
| applies_value | string |  | SKU code/bundle id/ALL |
| packaging_cost | decimal(12,4) |  | cost in reporting currency |
| priority | int |  | lower number = higher precedence |

Sample rows:
| rule_id | effective_start_date | effective_end_date | applies_to | applies_value | packaging_cost | priority |
|---|---|---|---|---|---|---|
| PKG_DEFAULT | 2026-01-01 | null | ORDER | ALL | 0.6500 | 99 |
| PKG_BUNDLE_AB | 2026-01-01 | null | BUNDLE | BUNDLE_AB | 1.1500 | 10 |

### 6.2 Fact/Staging Tables

#### `stg_shopify_orders`
Raw normalized order headers from API, append/upsert by `order_id` and `updated_at`.

#### `stg_shopify_order_lines`
One row per line item per order, include discounts and tax allocations.

#### `stg_shopify_refunds`
One row per refund event + refund line details.

#### `stg_shipping_labels`
One row per label; linked to order via direct reference, tracking map, or fallback matching.

#### `stg_ad_spend_day`
One row per day x channel x campaign (optional granularity).

#### `fact_order_line_finance`
| column | type | key | notes |
|---|---|---|---|
| order_id | string | PK(partial) | |
| line_id | string | PK(partial) | |
| sku | string |  | |
| quantity | int |  | net of refunded units configurable |
| gross_line_revenue | decimal(12,4) |  | before discount |
| discount_allocated | decimal(12,4) |  | includes order-level allocation |
| net_line_revenue_excl_tax | decimal(12,4) |  | |
| tax_allocated | decimal(12,4) |  | excluded from profit by default |
| cogs_total | decimal(12,4) |  | effective-date lookup |
| packaging_allocated | decimal(12,4) |  | |
| refund_amount_allocated | decimal(12,4) |  | |

#### `fact_order_finance`
| column | type | key | notes |
|---|---|---|---|
| order_id | string | PK | |
| order_date | date |  | |
| channel | string |  | primary attribution rule |
| gross_revenue | decimal(12,4) |  | line subtotal pre-discount |
| discounts_total | decimal(12,4) |  | includes code/auto discounts |
| net_revenue_excl_tax | decimal(12,4) |  | post-discount, excl tax |
| shipping_revenue | decimal(12,4) |  | shipping charged to customer |
| refund_total_excl_tax | decimal(12,4) |  | order-linked refunds |
| cogs_total | decimal(12,4) |  | |
| shipping_label_total | decimal(12,4) |  | sum labels excluding reships(optional) |
| packaging_total | decimal(12,4) |  | |
| payment_fee_total | decimal(12,4) |  | optional configurable formula |
| variable_cost_total | decimal(12,4) |  | sum variable costs |
| contribution_profit | decimal(12,4) |  | formula section 7 |
| opex_allocated | decimal(12,4) |  | by policy |
| net_profit | decimal(12,4) |  | |
| contribution_margin_pct | decimal(8,4) |  | cp / net revenue basis |
| net_margin_pct | decimal(8,4) |  | np / net revenue basis |

Sample rows:
| order_id | order_date | channel | gross_revenue | discounts_total | net_revenue_excl_tax | shipping_revenue | refund_total_excl_tax | cogs_total | shipping_label_total | packaging_total | variable_cost_total | contribution_profit | opex_allocated | net_profit |
|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|
| 1001 | 2026-01-18 | Meta | 68.00 | 8.00 | 60.00 | 4.99 | 0.00 | 25.30 | 6.20 | 1.15 | 33.75 | 31.24 | 9.10 | 22.14 |
| 1002 | 2026-01-18 | Google | 35.00 | 0.00 | 35.00 | 0.00 | 10.00 | 12.50 | 5.85 | 0.65 | 21.00 | 4.00 | 8.40 | -4.40 |

#### `fact_channel_day`
| column | type | key | notes |
|---|---|---|---|
| date | date | PK(partial) | |
| channel | string | PK(partial) | |
| orders | int |  | |
| ad_spend | decimal(12,4) |  | |
| attributed_orders | int |  | optional |
| contribution_profit_total | decimal(14,4) |  | |
| net_profit_total | decimal(14,4) |  | |
| blended_cac | decimal(12,4) |  | ad_spend/orders or attributed_orders |
| break_even_cac_cp | decimal(12,4) |  | from avg cp/order |
| break_even_cac_np | decimal(12,4) |  | from avg np/order |

---

## 7) Calculation Definitions (Exact Formulas)

Assume all monetary fields converted to reporting currency (e.g., USD) before aggregation.

### 7.1 Revenue components
- `gross_revenue = SUM(line_item_unit_price * qty)`
- `discounts_total = SUM(line_item_discount_allocations) + order_level_discounts`
- `net_product_revenue = gross_revenue - discounts_total`
- `shipping_revenue = shipping_charged_to_customer`
- `net_revenue_excl_tax = net_product_revenue + shipping_revenue - refund_total_excl_tax - refunded_shipping`

> Tax treatment: taxes are excluded from profit by default; configurable switch can include tax-inclusive reporting.

### 7.2 Variable costs
- `cogs_total = SUM(lookup_cogs(sku, order_date) * fulfilled_qty_net)`
- `shipping_label_total = SUM(label_cost where label_type != 'reshipment' OR include_reshipments=true)`
- `packaging_total = apply_packaging_rule(order)`
- `payment_fee_total = IF gateway_fee_available THEN actual_fee ELSE fee_rate * captured_amount + fixed_fee`
- `refund_processing_cost_total` (optional V2): per-refund handling fee.

- `variable_cost_total = cogs_total + shipping_label_total + packaging_total + payment_fee_total + refund_processing_cost_total`

### 7.3 Profit metrics
- `contribution_profit = net_revenue_excl_tax - variable_cost_total`
- `contribution_margin_pct = contribution_profit / NULLIF(net_revenue_excl_tax, 0)`

### 7.4 OpEx allocation
Configurable policy:
1. **Per-order flat:** `opex_allocated = daily_opex_total / daily_orders`
2. **Revenue-weighted:** `opex_allocated = order_net_revenue / day_net_revenue * daily_opex_total`
3. **Channel-weighted:** allocate by channel spend share or headcount driver.

- `net_profit = contribution_profit - opex_allocated`
- `net_margin_pct = net_profit / NULLIF(net_revenue_excl_tax, 0)`

### 7.5 Break-even CAC
At product mix grain (SKU, bundle, channel, date range):
- `avg_cp_per_order = SUM(contribution_profit)/COUNT(orders)`
- `avg_np_per_order = SUM(net_profit)/COUNT(orders)`
- `break_even_cac_cp = avg_cp_per_order`
- `break_even_cac_np = avg_np_per_order`

Interpretation:
- If **actual CAC <= break_even_cac_cp**, business is contribution-profitable.
- If **actual CAC <= break_even_cac_np**, business is fully net-profitable after OpEx.

### 7.6 Refund/return impact
- `refund_rate = refunded_orders / total_orders` (or refunded_units / sold_units)
- `refund_impact_profit = SUM(refund_total_excl_tax + refunded_shipping + refund_processing_cost_total)`
- Scenario metric:
  - `profit_without_refunds = actual_profit + refund_impact_profit`
  - `delta_margin_bps = (actual_margin - no_refund_margin) * 10,000`

---

## 8) ETL / ELT Architecture Plan

### 8.1 Ingestion pattern
- Use scheduled sync jobs + incremental cursoring.
- Persist raw payloads for auditability (`raw_*` tables) and normalized staging tables (`stg_*`).
- Upsert using source primary IDs and `updated_at` watermarks.

### 8.2 Sync cadence (MVP)
- Shopify orders/refunds: every **15 minutes**.
- Shipping labels: every **30 minutes**.
- Ad spend: hourly (or daily at 3am if API constraints).
- COGS/packaging/OpEx inputs: on change + nightly full integrity check.

### 8.3 Handling edits, refunds, and late-arriving data
- Maintain `last_seen_updated_at` per source.
- Recompute affected order facts when:
  - order edited (discount, line item adjustments),
  - refund posted,
  - label linked/updated,
  - COGS effective record changed overlapping order date.
- Implement rolling recomputation windows:
  - last 14 days every run,
  - last 90 days nightly,
  - full backfill manual trigger.

### 8.4 Idempotency and quality checks
- Idempotent merge keys (`order_id`, `line_id`, `refund_id`, `label_id`).
- Data quality tests:
  - non-null PKs,
  - no duplicate order facts,
  - valid currency conversions,
  - profit identity check: `net_profit = contribution_profit - opex_allocated`.
- Reconciliation dashboard:
  - total orders count vs Shopify,
  - total refunds vs Shopify,
  - total ad spend vs source for selected period.

### 8.5 Suggested stack
- Ingestion: Fivetran/Airbyte/custom Python.
- Warehouse: BigQuery/Snowflake/Postgres.
- Transform: dbt with snapshot models for effective-dated COGS/rules.
- BI: Looker/Metabase/Mode/Superset.
- Alerting: Slack + email via scheduled anomaly job.

---

## 9) Dashboard UX Specification

### 9.1 Global filters
- Date range (today, yesterday, last 7/30, custom).
- Channel (Meta, Google, Organic, Email, etc.).
- Product mode (single SKU vs bundle).
- SKU/bundle selector.
- New vs returning customer.
- Geography (optional V2).

### 9.2 KPI header cards
- Orders
- Net revenue (ex tax)
- Contribution profit
- Contribution margin %
- Net profit
- Net margin %
- Blended CAC
- Break-even CAC (CP and NP)

### 9.3 Core charts
1. **Profit waterfall (today / selected range)**
   - Gross revenue → discounts → refunds → COGS → shipping labels → packaging → contribution profit → OpEx → net profit.
2. **Daily trend line**
   - Contribution profit/order and net profit/order over time.
3. **Channel profitability heatmap**
   - Channels by contribution margin %, net margin %, CAC gap to break-even.
4. **SKU/Bundle table**
   - Orders, AOV, CP/order, NP/order, break-even CAC, actual CAC, margin.
5. **Refund impact panel**
   - Refund rate trend and profit drag.

### 9.4 Drilldowns
- Click channel → campaign-level details.
- Click SKU/bundle → order list with per-order cost decomposition.
- Click anomalous point → top contributing orders/labels/refunds.

### 9.5 Anomaly alerts
Triggered hourly/daily with thresholds and z-score options:
- **Margin drop alert:** contribution margin drops > X bps day-over-day.
- **Refund spike alert:** refund rate > threshold or >2σ from 30-day baseline.
- **Shipping cost spike:** avg label cost/order > threshold or >2σ baseline.
- **CAC breach alert:** actual CAC exceeds break-even CAC by Y%.

Alert payload includes:
- metric value,
- baseline value,
- impacted channel/SKU,
- likely drivers (top 3 dimensions by contribution).

---

## 10) MVP (2–3 weeks) vs V2 (6–8 weeks)

### 10.1 MVP scope (2–3 weeks)
- Data ingestion:
  - Shopify orders + line items + refunds.
  - One shipping label source (API or CSV).
  - Manual COGS + packaging + OpEx upload.
  - Ad spend by day/channel (manual CSV acceptable).
- Models:
  - `fact_order_finance`, `fact_channel_day`, `fact_order_line_finance`.
  - Core formulas for CP, NP, break-even CAC.
- Dashboard:
  - KPI cards,
  - channel profitability table,
  - SKU/bundle profitability table,
  - daily trend chart,
  - basic anomaly rules (static thresholds).
- Ops:
  - 15–60 minute refresh,
  - basic data quality checks,
  - CSV export.

### 10.2 V2 scope (6–8 weeks)
- Native Meta + Google connectors and campaign-level mapping.
- Advanced attribution settings (first/last click toggle).
- Automated returns lifecycle (RMA status, restocking outcomes).
- Reshipment cost treatment modes.
- Subscription order handling and cohort profitability overlays.
- Scenario simulator (change COGS/shipping/CAC and project margin).
- ML anomaly detection and smarter root-cause explanations.

---

## 11) Edge Cases & Resolution Rules

1. **Partial refunds**
   - Allocate refund to affected line items first; residual shipping refund to order-level.
   - Preserve original order row; post-adjusted finance values via incremental recompute.

2. **Split shipments**
   - Sum all non-reship labels by order for shipping_label_total.
   - If partial fulfillments span days, shipping cost remains tied to originating order (configurable).

3. **Bundles vs component SKUs**
   - If bundle SKU has explicit COGS, use it.
   - Else explode into component SKUs using BOM mapping and sum component COGS.
   - Packaging rule precedence: bundle rule > SKU rule > default order rule.

4. **Discount stacking**
   - Preserve line-level discount allocations from Shopify.
   - If only order-level discount exists, prorate by pre-discount line revenue.

5. **Reshipments**
   - Default exclude reship label costs from core contribution profit; expose toggle to include as QA/CS cost.

6. **Gift cards**
   - Gift card purchase orders excluded from product revenue profitability.
   - Gift card redemption reduces cash collected on order but should not double-count as discount.

7. **Subscriptions**
   - Mark subscription-origin orders.
   - CAC assignment may be zero for renewal orders; separate acquisition vs retention profitability views.

8. **Tax-inclusive markets / multi-currency**
   - Normalize using FX rate at order date (or settlement date policy).
   - Profit computed in reporting currency with tax excluded by default.

9. **Order edits after fulfillment**
   - Recompute line/revenue allocations and store revision timestamp.

10. **Missing shipping label link**
   - Apply fallback estimated shipping cost by zone/weight profile and flag `shipping_cost_estimated=true`.

---

## 12) Permissions, Governance, and Auditability
- Row-level access optional by brand/store.
- Admin-only controls for COGS/packaging/OpEx table edits.
- Full audit trail on manual overrides (who/when/old/new value).
- Versioned metric definitions page in-product.

---

## 13) Open Questions
- What is the default attribution logic for channel at order level (UTM, Shopify source, last non-direct)?
- Should reshipment costs be classified into contribution profit or OpEx by default?
- How should chargebacks be represented (refund-equivalent or separate loss bucket)?
- What is the canonical OpEx source and update frequency?

---

## 14) Implementation Milestones (Suggested)
1. **Week 1:** schema + ingestion + raw/staging tables + COGS/packaging inputs.
2. **Week 2:** finance transformation models + QA checks + initial BI dashboard.
3. **Week 3:** anomaly alerts + stakeholder UAT + metric sign-off.
4. **Weeks 4–8 (V2):** advanced connectors, attribution, simulation, ML anomaly enhancements.

---

## 15) Acceptance Criteria
- Dashboard answers all 3 primary business questions with <30 min freshness.
- Contribution and net profit numbers reconcile to model formulas for sampled orders.
- Break-even CAC available by SKU and bundle across selectable date range.
- Channel report includes shipping, packaging, refunds, and OpEx allocation.
- Edge-case test suite passes for partial refunds, split shipments, bundle decomposition, discount stacking, reshipments, gift cards, and subscriptions.

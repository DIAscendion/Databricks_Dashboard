_____________________________________________
## *Author*: AAVA
## *Created on*:
## *Description*: Databricks Lakeview Dashboard (LVDash) visuals and KPI recommendations for the DSG Store Inventory Intelligence Dashboard, derived from the store inventory, RFID, replenishment, and product-findability data model.
## *Version*: 1
## *Updated on*:
_____________________________________________

## Summary

This document analyzes the DSG Store Inventory data model (`dsg.inventory_gold`, `dsg.store_inventory`, `dsg.stores_gold`) and the associated reporting requirements, and recommends the Databricks SQL queries, LVDash widgets, and overall dashboard design needed to cover: inventory accuracy, stock availability, RFID cycle-count coverage, athlete product findability, teammate fulfillment efficiency, store-level performance, and category-level stockout risk.

---

## 1. Visual Recommendations

### 1.1 Inventory Accuracy (KPI)
- **Data Element:** 30-day average inventory accuracy
- **SQL Query:**
  ```sql
  SELECT ROUND(AVG(inventory_accuracy_pct) * 100, 1) AS inventory_accuracy_pct
  FROM dsg.inventory_gold.inv_accuracy_daily
  WHERE calendar_date >= CURRENT_DATE() - INTERVAL 30 DAYS
  ```
- **Recommended LVDash Widget:** Counter
- **Data Fields:** `inventory_accuracy_pct`
- **Calculations:** `AVG(inventory_accuracy_pct) * 100`, rounded to 1 decimal
- **Interactivity:** None required; optionally respects a global date-range filter
- **Justification:** A single trailing-30-day accuracy rate is best conveyed as a scorecard counter (`{{ @formatted }}%`) at the top of the page for at-a-glance monitoring.
- **Optimization Tips:** Materialize `inv_accuracy_daily` as a pre-aggregated daily rollup; avoid scanning raw cycle-count transactions on every dashboard load.

### 1.2 On-Hand Units (KPI)
- **Data Element:** Total units currently on hand across stores
- **SQL Query:**
  ```sql
  SELECT SUM(on_hand_qty) AS on_hand_units
  FROM dsg.store_inventory.fact_store_inventory
  ```
- **Recommended LVDash Widget:** Counter
- **Data Fields:** `on_hand_units`
- **Calculations:** `SUM(on_hand_qty)`
- **Interactivity:** Cross-filters with the Region filter widget (§1.11)
- **Justification:** A single running total is best shown as a plain-number counter (0 decimal places).
- **Optimization Tips:** Liquid-cluster `fact_store_inventory` on `store_number`/`product_id` to keep the full-table `SUM` fast at scale.

### 1.3 Stockout Rate (KPI)
- **Data Element:** % of inventory rows currently `OUT_OF_STOCK`
- **SQL Query:**
  ```sql
  SELECT ROUND(SUM(CASE WHEN inventory_status = 'OUT_OF_STOCK' THEN 1 ELSE 0 END) / COUNT(*) * 100, 1) AS stockout_rate_pct
  FROM dsg.store_inventory.fact_store_inventory
  ```
- **Recommended LVDash Widget:** Counter
- **Data Fields:** `stockout_rate_pct`
- **Calculations:** Conditional count ratio, rounded to 1 decimal
- **Interactivity:** Cross-filters with Region
- **Justification:** A bounded percentage KPI is clearest as a counter with a `%` format template; a gauge is not natively supported in LVDash, so counter is the closest fit.
- **Optimization Tips:** Add a partition/liquid-cluster key on `inventory_status` if this table is queried frequently by status.

### 1.4 Avg SKUs per Cart (KPI)
- **Data Element:** 30-day average SKUs replenished per cart trip
- **SQL Query:**
  ```sql
  SELECT ROUND(AVG(avg_skus_per_cart), 1) AS avg_skus_per_cart
  FROM dsg.inventory_gold.inv_replenish_daily
  WHERE calendar_date >= CURRENT_DATE() - INTERVAL 30 DAYS
  ```
- **Recommended LVDash Widget:** Counter
- **Data Fields:** `avg_skus_per_cart`
- **Calculations:** `AVG(avg_skus_per_cart)`, 1 decimal
- **Interactivity:** None required
- **Justification:** Simple rate metric, best as a compact counter alongside the other fulfillment-efficiency KPIs.
- **Optimization Tips:** Pre-aggregate at the daily grain (already done in `inv_replenish_daily`); avoid recomputing from event-level replenishment logs on every load.

### 1.5 Nearby-Store Stock Shown % (KPI)
- **Data Element:** % of non-in-stock (store, product) pairs where nearby-store availability was surfaced to associates
- **SQL Query:**
  ```sql
  SELECT ROUND(
    COUNT(DISTINCT CONCAT(athlete_store_id, '-', product_id)) * 100.0
      / (SELECT COUNT(*) FROM dsg.store_inventory.fact_store_inventory WHERE inventory_status != 'IN_STOCK'),
    1) AS nearby_stock_shown_pct
  FROM dsg.store_inventory.fact_nearby_store_availability
  ```
- **Recommended LVDash Widget:** Counter
- **Data Fields:** `nearby_stock_shown_pct`
- **Calculations:** Distinct-pair coverage ratio against a correlated subquery denominator
- **Interactivity:** None required
- **Justification:** A coverage-rate KPI reads clearly as a percentage counter under the "Athlete Product Findability" section.
- **Optimization Tips:** Cache the denominator subquery result (e.g., as its own small dataset) if this query is reused elsewhere, to avoid duplicating the full-table scan.

### 1.6 Zero-Result Search Rate (KPI)
- **Data Element:** Latest-day zero-result search rate
- **SQL Query:**
  ```sql
  SELECT ROUND(SUM(zero_result_search_count) / SUM(total_search_events) * 100, 1) AS zero_result_rate_pct
  FROM dsg.store_inventory.fact_daily_store_rollup
  WHERE rollup_date = (SELECT MAX(rollup_date) FROM dsg.store_inventory.fact_daily_store_rollup)
  ```
- **Recommended LVDash Widget:** Counter
- **Data Fields:** `zero_result_rate_pct`
- **Calculations:** Ratio of zero-result searches to total search events for the latest rollup date
- **Interactivity:** None required
- **Justification:** Freshness-sensitive single metric; counter keeps it prominent and scannable.
- **Optimization Tips:** Index/partition `fact_daily_store_rollup` on `rollup_date` so the `MAX(rollup_date)` lookup and filter are cheap.

### 1.7 RFID Cycle Count Coverage % (KPI)
- **Data Element:** Latest-day RFID cycle-count coverage
- **SQL Query:**
  ```sql
  SELECT ROUND(AVG(rfid_coverage_pct) * 100, 1) AS rfid_coverage_pct
  FROM dsg.inventory_gold.inv_rfid_coverage_daily
  WHERE calendar_date = (SELECT MAX(calendar_date) FROM dsg.inventory_gold.inv_rfid_coverage_daily)
  ```
- **Recommended LVDash Widget:** Counter
- **Data Fields:** `rfid_coverage_pct`
- **Calculations:** `AVG(rfid_coverage_pct) * 100` for the latest date
- **Interactivity:** None required
- **Justification:** Point-in-time coverage rate; compact counter fits the "Teammate Fulfillment Efficiency" KPI row.
- **Optimization Tips:** Materialized daily rollup already avoids scanning raw RFID scan events — keep this pattern.

### 1.8 Pick Units-Per-Hour / Product-Finding Scan % (KPIs)
- **Data Element:** 30-day average pick UPH; 30-day product-finding scan rate
- **SQL Query:**
  ```sql
  SELECT ROUND(AVG(pick_uph), 1) AS pick_uph
  FROM dsg.stores_gold.fop_pick_uph_daily
  WHERE calendar_date >= CURRENT_DATE() - INTERVAL 30 DAYS;

  SELECT ROUND(SUM(scan_count) / SUM(total_product_views) * 100, 1) AS scan_pct
  FROM dsg.stores_gold.rta_product_finding_daily
  WHERE calendar_date >= CURRENT_DATE() - INTERVAL 30 DAYS
  ```
- **Recommended LVDash Widget:** Counter (one each)
- **Data Fields:** `pick_uph`, `scan_pct`
- **Calculations:** Trailing-30-day averages/ratios
- **Interactivity:** None required
- **Justification:** Operational efficiency metrics belong beside the other fulfillment KPIs as scannable counters.
- **Optimization Tips:** Both source from pre-aggregated daily gold tables — no further optimization needed unless the 30-day window is widened significantly, in which case consider a rolling materialized view.

### 1.9 Store Inventory Performance (Detail Grid)
- **Data Element:** Per-store on-hand units, RFID accuracy, zero-result rate, OOS SKU count, and alert status
- **SQL Query:**
  ```sql
  SELECT s.store_number AS store, s.store_name AS name, s.region_description AS region,
         s.on_hand_units AS on_hand, ROUND(s.rfid_accuracy_pct * 100, 1) AS rfid_acc_pct,
         s.zero_result_rate_pct, s.oos_sku_count AS oos_skus,
         CASE WHEN s.alert_flag = 'Y' THEN '⚠️' ELSE '' END AS alerts,
         s.alert_reasons AS alert_detail
  FROM dsg.inventory_gold.store_summary_current s
  ORDER BY s.alert_flag DESC, s.rfid_accuracy_pct ASC
  ```
- **Recommended LVDash Widget:** Table
- **Data Fields:** `store`, `name`, `region`, `on_hand`, `rfid_acc_pct`, `zero_result_rate_pct`, `oos_skus`, `alerts`, `alert_detail`
- **Calculations:** RFID accuracy scaled to percent; alert flag mapped to an emoji indicator
- **Interactivity:** Filtered by the Region multi-select filter (§1.11); sortable table columns
- **Justification:** Multi-attribute, multi-store comparison is best served by a table with conditional formatting rather than a chart — it lets ops teams scan every store and every metric at once.
- **Optimization Tips:** `store_summary_current` is already a "current state" summary table — good pattern to avoid re-deriving from transaction-level facts on each dashboard load. Conditional-format the `rfid_acc_pct` and `zero_result_rate_pct` columns (red/orange/green thresholds) to surface outliers without scanning the whole grid.

### 1.10 Category Inventory Health (Detail Grid)
- **Data Element:** Category-level stockout risk score and health status, worst-first
- **SQL Query:**
  ```sql
  SELECT category, metric_value AS stockout_risk, health_status
  FROM dsg.inventory_gold.inv_category_health
  WHERE metric_name = 'stockout_risk'
  ORDER BY CASE health_status WHEN 'Critical' THEN 1 WHEN 'Risk' THEN 2 ELSE 3 END
  ```
- **Recommended LVDash Widget:** Table
- **Data Fields:** `category`, `stockout_risk`, `health_status`
- **Calculations:** Custom sort order prioritizing `Critical` > `Risk` > `Healthy`
- **Interactivity:** None required; optionally cross-filters from a category selector if one is added later
- **Justification:** A small, ranked list of categories with a status badge is clearer as a conditionally formatted table than a chart, and keeps color semantics consistent with the Critical/Risk/Healthy palette used elsewhere.
- **Optimization Tips:** `inv_category_health` is a narrow, pre-computed metric table — keep the `metric_name` filter indexed/partitioned if additional metrics are added to the same table later.

### 1.11 Region Filter
- **Data Element:** Distinct list of store regions
- **SQL Query:**
  ```sql
  SELECT DISTINCT region_description AS region
  FROM dsg.inventory_gold.store_summary_current
  ORDER BY region
  ```
- **Recommended LVDash Widget:** Filter (multi-select)
- **Data Fields:** `region`
- **Calculations:** None (distinct list)
- **Interactivity:** Cross-filters the Store Inventory Performance table and the On-Hand/Stockout KPI counters
- **Justification:** Region is the primary slicing dimension across the dashboard; a multi-select filter lets ops leads focus on one or several regions at once.
- **Optimization Tips:** Reuses the same `store_summary_current` table already scanned by the detail grid — no extra table scan needed.

### 1.12 Replenishment Event Detail (Detail Grid)
- **Data Element:** Chronological log of stockroom replenishment actions
- **SQL Query:**
  ```sql
  SELECT store_number, action_ts_local AS event_time, product_id, sku, action_text,
         event_quantity, replenishment_session_id
  FROM dsg.inventory_gold.inv_replenish_event_detail
  ORDER BY action_ts_local DESC
  ```
- **Recommended LVDash Widget:** Table
- **Data Fields:** `store_number`, `event_time`, `product_id`, `sku`, `action_text`, `event_quantity`, `replenishment_session_id`
- **Calculations:** None (event-level detail)
- **Interactivity:** None required; consider a session-ID drilldown-style cross-filter if paired with a summary widget later
- **Justification:** Raw event history is inherently tabular and needs no aggregation or chart encoding.
- **Optimization Tips:** This is the one genuinely event-level (non-pre-aggregated) table on the dashboard — consider a default date-range filter or `LIMIT`/pagination to avoid pulling the full history on every load, and liquid-cluster on `action_ts_local`.

### 1.13 Stockroom Position (Detail Grid)
- **Data Element:** Current stockroom location and quantity per SKU
- **SQL Query:**
  ```sql
  SELECT store_number, product_id, sku, stockroom_location_id, stockroom_location_type,
         located_qty, last_updated_ts_local
  FROM dsg.inventory_gold.inv_stockroom_position
  ORDER BY located_qty DESC
  ```
- **Recommended LVDash Widget:** Table
- **Data Fields:** `store_number`, `product_id`, `sku`, `stockroom_location_id`, `stockroom_location_type`, `located_qty`, `last_updated_ts_local`
- **Calculations:** None
- **Interactivity:** None required
- **Justification:** Location-level inventory is a lookup use case — a table is the correct widget, no visualization needed.
- **Optimization Tips:** `inv_stockroom_position` should reflect current state (not full history) — confirm the source is a Type-1 (overwrite) table, not an append-only log, to keep row counts and query time bounded.

### 1.14 Executive AI Insight (Narrative)
- **Data Element:** LLM-generated 3-sentence executive summary of overall inventory health
- **SQL Query:**
  ```sql
  WITH acc AS (SELECT ROUND(AVG(inventory_accuracy_pct) * 100, 1) AS inv_acc
               FROM dsg.inventory_gold.inv_accuracy_daily
               WHERE calendar_date >= CURRENT_DATE() - INTERVAL 30 DAYS),
       rfid AS (SELECT ROUND(AVG(rfid_coverage_pct) * 100, 1) AS rfid_cov
                FROM dsg.inventory_gold.inv_rfid_coverage_daily),
       alerts AS (SELECT COUNT(CASE WHEN alert_flag = 'Y' THEN 1 END) AS alert_count,
                         MAX(CASE WHEN alert_flag = 'Y' THEN store_name END) AS worst_store,
                         MAX(CASE WHEN alert_flag = 'Y' THEN alert_reasons END) AS worst_reasons,
                         ROUND(AVG(zero_result_rate_pct), 1) AS zr_rate,
                         ROUND(COUNT(CASE WHEN oos_sku_count > 0 THEN 1 END) * 100.0 / COUNT(*), 1) AS stores_with_oos_pct
                  FROM dsg.inventory_gold.store_summary_current),
       cat AS (SELECT CONCAT_WS(', ', COLLECT_LIST(category)) AS critical_cats
               FROM dsg.inventory_gold.inv_category_health
               WHERE metric_name = 'stockout_risk' AND health_status = 'Critical'),
       prompt AS (SELECT CONCAT('You are a retail inventory analyst...', acc.inv_acc, '%...') AS llm_prompt
                  FROM acc CROSS JOIN rfid CROSS JOIN alerts CROSS JOIN cat)
  SELECT ai_query('databricks-meta-llama-3-3-70b-instruct', llm_prompt) AS insight
  FROM prompt
  ```
- **Recommended LVDash Widget:** Counter/Text widget rendering the `insight` field as plain text (no numeric formatting)
- **Data Fields:** `insight`
- **Calculations:** Multi-CTE metric rollup feeding a natural-language prompt, resolved via `ai_query`
- **Interactivity:** None required — regenerates each dashboard refresh
- **Justification:** Gives store operations leaders a plain-language summary up front, synthesizing every KPI on the page into one narrative without requiring them to read every widget individually.
- **Optimization Tips:** Cache/schedule this dataset's refresh less frequently than the raw KPIs (e.g., once per dashboard load or on a longer schedule) since LLM calls are the most expensive operation on the page; avoid triggering it per-filter-change.

---

## 2. Overall Dashboard Design

- **Layout Suggestions:** Single-page grid (12-column) organized top-to-bottom as: KPI banner (Inventory Accuracy, On-Hand Units, Zero-Result Rate, Stockout Rate) → section header "Athlete Product Findability" with its counters → section header "Teammate Fulfillment Efficiency" with its counters → Region filter → Store Inventory Performance table (full width) → section header "Category-Level Inventory Health" → Category Inventory Health table → Replenishment History and Stockroom Position tables stacked full-width below. An optional AI Insight narrative widget can sit at the very top, above the KPI banner, to frame the page.
- **Query Optimization:** Reuse pre-aggregated gold tables (`inv_accuracy_daily`, `inv_rfid_coverage_daily`, `store_summary_current`, `inv_category_health`) wherever available instead of re-deriving from raw fact tables; liquid-cluster the two heavier fact tables (`fact_store_inventory`, `inv_replenish_event_detail`) on their most-filtered columns; keep the LLM-driven dataset on a separate, less-frequent refresh cadence.
- **Color Scheme:** Green-forward palette (`#2E7D32` primary) with status colors reserved consistently for Critical (`#C62828`/red), Risk (`#F57C00`/orange), and Healthy (`#2E7D32`/green) across both table conditional formatting and any future chart legends.
- **Typography:** Montserrat throughout — 11px base body text, 12px semi-bold (600) widget titles, 11px muted field titles, semi-bold field values for emphasis on KPI numbers.
- **Interactive Elements:** Region multi-select filter cross-filtering the KPI counters and the Store Inventory Performance table; table column sorting on all detail grids; conditional-formatting badges standing in for a legend so no separate legend widget is needed.

| Interactivity Feature | Widget(s) | Behavior |
|---|---|---|
| Multi-select filter | Region | Cross-filters On-Hand Units KPI and Store Inventory Performance table |
| Conditional formatting | Store table (`rfid_acc_pct`, `zero_result_rate_pct`), Category table (`health_status`) | Red/orange/green thresholds flag outliers without a separate chart |
| Sortable columns | All table widgets | Lets ops teams re-rank by any metric on demand |
| Narrative synthesis | AI Insight widget | Summarizes all KPIs in plain language on page load |

---

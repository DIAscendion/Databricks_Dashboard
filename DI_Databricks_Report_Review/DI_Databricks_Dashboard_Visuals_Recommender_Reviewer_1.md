_____________________________________________
## *Author*: AAVA
## *Created on*:   
## *Description*: Review and refinement of Databricks Genie dashboard prompts for the DSG Store Inventory Intelligence Dashboard, aligned to reporting goals, recommended visuals, and available data model.
## *Version*: 1
## *Updated on*: 
_____________________________________________

# Databricks Genie Reviewer

## Summary
This reviewer refines the prior dashboard visual recommendations into clearer, implementation-ready Databricks Genie prompts. It validates visual suitability, standardizes prompt wording, improves field usage and formatting guidance, and expands the dashboard with additional visuals and usability features so the resulting dashboard is more insightful, consistent, and user-friendly.

---

## 1. Review Findings

| Review Area | Observation | Reviewer Action |
|---|---|---|
| Prompt structure | Source file contains strong visual recommendations, but not all items are phrased as direct Databricks Genie prompts. | Converted recommendations into explicit Genie-ready prompts. |
| First prompt convention | Reviewer rule requires first visual prompt to start with **Create** and remaining prompts to start with **Add**. | Applied consistently across all refined prompts. |
| Visual suitability | Most KPI recommendations correctly use counters; detail-oriented datasets correctly use tables. | Retained suitable visuals and added formatting guidance. |
| Completeness | Existing set is solid but can be richer for trend and comparative analysis. | Added extra visuals for trend, comparison, and distribution insights. |
| Data field usage | Core fields and calculations are present, but some prompts need stronger references to filters, labels, and display formatting. | Added explicit titles, labels, sorting, filters, and formatting requirements. |
| Dashboard usability | Existing design mentions filtering and narrative insight, but interactive behavior can be more explicit. | Added filter, cross-filter, and storytelling recommendations. |

---

## 2. Refined Databricks Genie Prompts

### 2.1 Core Prompts

1. **Create** a KPI counter titled **"30-Day Inventory Accuracy %"** using `dsg.inventory_gold.inv_accuracy_daily`. Calculate `ROUND(AVG(inventory_accuracy_pct) * 100, 1)` for rows where `calendar_date >= CURRENT_DATE() - INTERVAL 30 DAYS`. Format the result as a percentage with 1 decimal place and display it prominently at the top of the dashboard.

2. **Add** a KPI counter titled **"On-Hand Units"** using `dsg.store_inventory.fact_store_inventory`. Calculate `SUM(on_hand_qty) AS on_hand_units`. Format as a whole number with thousands separators and allow it to respond to a global Region filter.

3. **Add** a KPI counter titled **"Stockout Rate %"** using `dsg.store_inventory.fact_store_inventory`. Calculate `ROUND(SUM(CASE WHEN inventory_status = 'OUT_OF_STOCK' THEN 1 ELSE 0 END) / COUNT(*) * 100, 1) AS stockout_rate_pct`. Format as a percentage with 1 decimal place and allow it to respond to Region filters.

4. **Add** a KPI counter titled **"Average SKUs per Cart - Last 30 Days"** using `dsg.inventory_gold.inv_replenish_daily`. Calculate `ROUND(AVG(avg_skus_per_cart), 1)` for the last 30 days based on `calendar_date`. Format with 1 decimal place.

5. **Add** a KPI counter titled **"Nearby-Store Stock Shown %"** using `dsg.store_inventory.fact_nearby_store_availability`. Calculate the percent of distinct `(athlete_store_id, product_id)` pairs surfaced relative to all non-`IN_STOCK` inventory rows from `dsg.store_inventory.fact_store_inventory`. Format as a percentage with 1 decimal place.

6. **Add** a KPI counter titled **"Latest Zero-Result Search Rate %"** using `dsg.store_inventory.fact_daily_store_rollup`. Calculate `ROUND(SUM(zero_result_search_count) / SUM(total_search_events) * 100, 1)` for the latest `rollup_date`. Format as a percentage with 1 decimal place.

7. **Add** a KPI counter titled **"RFID Cycle Count Coverage %"** using `dsg.inventory_gold.inv_rfid_coverage_daily`. Calculate `ROUND(AVG(rfid_coverage_pct) * 100, 1)` for the most recent `calendar_date`. Format as a percentage with 1 decimal place.

8. **Add** a KPI counter titled **"Pick Units per Hour - Last 30 Days"** using `dsg.stores_gold.fop_pick_uph_daily`. Calculate `ROUND(AVG(pick_uph), 1)` over the last 30 days and display as a numeric KPI with 1 decimal place.

9. **Add** a KPI counter titled **"Product-Finding Scan % - Last 30 Days"** using `dsg.stores_gold.rta_product_finding_daily`. Calculate `ROUND(SUM(scan_count) / SUM(total_product_views) * 100, 1)` over the last 30 days. Format as a percentage with 1 decimal place.

10. **Add** a table titled **"Store Inventory Performance"** using `dsg.inventory_gold.store_summary_current`. Include columns `store_number AS store`, `store_name AS name`, `region_description AS region`, `on_hand_units AS on_hand`, `ROUND(rfid_accuracy_pct * 100, 1) AS rfid_acc_pct`, `zero_result_rate_pct`, `oos_sku_count AS oos_skus`, `CASE WHEN alert_flag = 'Y' THEN '⚠️' ELSE '' END AS alerts`, and `alert_reasons AS alert_detail`. Sort by `alert_flag DESC, rfid_accuracy_pct ASC`. Enable sortable columns and conditional formatting for low RFID accuracy and high zero-result rates.

11. **Add** a table titled **"Category Inventory Health"** using `dsg.inventory_gold.inv_category_health`. Filter to `metric_name = 'stockout_risk'` and display `category`, `metric_value AS stockout_risk`, and `health_status`. Sort categories so `Critical` appears first, then `Risk`, then `Healthy`. Apply red, orange, and green status formatting.

12. **Add** a multi-select filter titled **"Region"** using distinct `region_description AS region` from `dsg.inventory_gold.store_summary_current`. Sort alphabetically and connect this filter to the On-Hand Units KPI, Stockout Rate KPI, and Store Inventory Performance table.

13. **Add** a table titled **"Replenishment Event Detail"** using `dsg.inventory_gold.inv_replenish_event_detail`. Display `store_number`, `action_ts_local AS event_time`, `product_id`, `sku`, `action_text`, `event_quantity`, and `replenishment_session_id`. Sort by `action_ts_local DESC` and limit the default result set to recent records for dashboard performance.

14. **Add** a table titled **"Stockroom Position"** using `dsg.inventory_gold.inv_stockroom_position`. Display `store_number`, `product_id`, `sku`, `stockroom_location_id`, `stockroom_location_type`, `located_qty`, and `last_updated_ts_local`. Sort by `located_qty DESC`.

15. **Add** a narrative text widget titled **"Executive AI Insight"** that summarizes overall inventory health in 2 to 3 business-friendly sentences using rolled-up metrics for inventory accuracy, RFID coverage, alert counts, zero-result rate, out-of-stock store percentage, and critical categories. Keep the output concise and executive-focused.

### 2.2 Additional Visuals Added by Reviewer

16. **Add** a line chart titled **"Inventory Accuracy Trend - Last 30 Days"** using `dsg.inventory_gold.inv_accuracy_daily`. Plot `calendar_date` on the X-axis and `ROUND(AVG(inventory_accuracy_pct) * 100, 1)` on the Y-axis. Show markers, label the Y-axis as **Inventory Accuracy %**, and use a green line to highlight trend direction over time.

17. **Add** a bar chart titled **"Top 10 Stores by Out-of-Stock SKUs"** using `dsg.inventory_gold.store_summary_current`. Plot `store_name` on the X-axis and `oos_sku_count` on the Y-axis. Sort descending by `oos_sku_count`, limit to 10 stores, and color bars red to emphasize risk.

18. **Add** a stacked bar chart titled **"Store Count by Category Health Status"** using `dsg.inventory_gold.inv_category_health` filtered to `metric_name = 'stockout_risk'`. Group by `health_status` and count categories. Use red for `Critical`, orange for `Risk`, and green for `Healthy`. Include a legend and data labels.

19. **Add** a line chart titled **"Zero-Result Search Trend"** using `dsg.store_inventory.fact_daily_store_rollup`. Plot `rollup_date` on the X-axis and `ROUND(SUM(zero_result_search_count) / SUM(total_search_events) * 100, 1)` on the Y-axis. Show a 30-day trend with percentage formatting and label the chart clearly.

20. **Add** a scatter plot titled **"RFID Accuracy vs Zero-Result Rate by Store"** using `dsg.inventory_gold.store_summary_current`. Plot `ROUND(rfid_accuracy_pct * 100, 1)` on the X-axis and `zero_result_rate_pct` on the Y-axis, with points labeled or colored by `region_description`. Use this visual to identify stores with low RFID accuracy and high search failure rates.

---

## 3. Prompt Quality Improvements Applied

| Improvement Type | Applied Refinement |
|---|---|
| Standardized verbs | Ensured the first prompt starts with **Create** and all subsequent prompts start with **Add**. |
| Clear titles | Added explicit visual titles for each prompt. |
| Metric formatting | Specified numeric, whole-number, and percentage formats. |
| Axis and legend guidance | Added axis labels, legends, and color directions for chart-based visuals. |
| Filtering behavior | Clarified Region filter interaction and cross-filter expectations. |
| Performance awareness | Recommended limiting event-detail views and using summary sources where possible. |
| Storytelling | Preserved and strengthened the executive narrative widget recommendation. |

---

## 4. Data Source and Transformation Review

### 4.1 Identified Data Sources

| Data Source | Purpose |
|---|---|
| `dsg.inventory_gold.inv_accuracy_daily` | Inventory accuracy KPI and trend analysis |
| `dsg.store_inventory.fact_store_inventory` | On-hand inventory and stockout calculations |
| `dsg.inventory_gold.inv_replenish_daily` | Replenishment efficiency KPI |
| `dsg.store_inventory.fact_nearby_store_availability` | Nearby-store availability coverage |
| `dsg.store_inventory.fact_daily_store_rollup` | Zero-result search KPIs and trends |
| `dsg.inventory_gold.inv_rfid_coverage_daily` | RFID coverage KPI |
| `dsg.stores_gold.fop_pick_uph_daily` | Picking productivity KPI |
| `dsg.stores_gold.rta_product_finding_daily` | Product-finding scan KPI |
| `dsg.inventory_gold.store_summary_current` | Store-level operational summary and comparison views |
| `dsg.inventory_gold.inv_category_health` | Category-level health and stockout risk |
| `dsg.inventory_gold.inv_replenish_event_detail` | Replenishment event log detail |
| `dsg.inventory_gold.inv_stockroom_position` | Current stockroom position detail |

### 4.2 Identified Transformations, Joins, Aggregations, and Filters

| Component | Details |
|---|---|
| Aggregations | `AVG`, `SUM`, `COUNT`, conditional counts, percentage calculations, rounded metrics |
| Filters | Last 30 days, latest available date, `metric_name = 'stockout_risk'`, Region filter |
| Intermediate logic | Distinct store-product pair calculation, alert flag mapping, health status custom sort, AI narrative rollups |
| Joins / Multi-source usage | AI narrative combines multiple aggregated metric sets; nearby-store coverage compares one fact source to another denominator set |
| Output formats | KPI counters, tables, line charts, bar charts, stacked bar chart, scatter plot, narrative text |

---

## 5. Dashboard Best Practice Recommendations

1. Keep KPI counters in the first row with consistent number formatting.
2. Group visuals into business sections such as Inventory Health, Product Findability, and Fulfillment Efficiency.
3. Use color intentionally: green for healthy performance, orange for risk, red for critical conditions.
4. Apply conditional formatting in tables to highlight action areas quickly.
5. Limit raw detail tables by default date or row count for better responsiveness.
6. Use the Region filter as a shared control across summary visuals.
7. Add trend visuals so users can see whether current KPI values are improving or worsening.
8. Include an executive narrative widget for quick interpretation by business stakeholders.

---

## 6. Final Reviewer Approval

**Approval Status:** Approved with refinements  
**Reviewer Decision:** The dashboard prompt set is strong and aligned with the available data model. The refined prompts now provide clearer Genie instructions, stronger formatting guidance, better visual coverage, and more actionable interactivity. These prompts are ready for downstream Databricks dashboard generation.

---

## Output URL (Clickable Hyperlinks)
[DI_Databricks_Dashboard_Visuals_Recommender_Reviewer_1.md](https://github.com/DIAscendion/Databricks_Dashboard/blob/main/DI_Databricks_Report_Review/DI_Databricks_Dashboard_Visuals_Recommender_Reviewer_1.md)
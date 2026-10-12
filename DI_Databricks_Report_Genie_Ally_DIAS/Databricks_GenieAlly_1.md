_____________________________________________
## *Author*: AAVA
## *Created on*:
## *Description*: Databricks Genie prompts for building dashboard queries and visuals for the DSG Store Inventory Intelligence Dashboard based on the prior visuals recommendation output.
## *Version*: 1
## *Updated on*:
_____________________________________________

## Summary

This document converts the prior dashboard visual recommendations into business-analyst-ready Databricks Genie prompts. Each prompt tells Genie what query to generate, which visual to build, what fields and calculations to use, and how to format the output so the resulting dashboard is clear, actionable, and aligned to the Lakehouse semantic model.

## Source Input Reference

- **Input File Used:** `DI_Databricks_Dashboard_Visuals_Recommender/DI_Databricks_Dashboard_Visuals_Recommender_1.md`
- **Input Purpose:** Provide recommended KPIs, tables, filters, and dashboard design guidance for the DSG Store Inventory Intelligence Dashboard.
- **Primary Data Domains Identified:**
  - Inventory accuracy
  - Store inventory availability
  - Stockouts
  - Replenishment efficiency
  - RFID coverage
  - Product findability
  - Store performance monitoring
  - Category health
  - Stockroom and replenishment operational detail

## Lakehouse Data Model and Workflow Understanding

### Data Sources Identified

| Data Source Table | Purpose |
|---|---|
| `dsg.inventory_gold.inv_accuracy_daily` | Daily inventory accuracy metrics |
| `dsg.store_inventory.fact_store_inventory` | Current store inventory and stock status |
| `dsg.inventory_gold.inv_replenish_daily` | Daily replenishment performance metrics |
| `dsg.store_inventory.fact_nearby_store_availability` | Nearby-store inventory visibility coverage |
| `dsg.store_inventory.fact_daily_store_rollup` | Daily store-level search and operational rollups |
| `dsg.inventory_gold.inv_rfid_coverage_daily` | Daily RFID cycle-count coverage |
| `dsg.stores_gold.fop_pick_uph_daily` | Daily pick units-per-hour metrics |
| `dsg.stores_gold.rta_product_finding_daily` | Daily product-finding scan metrics |
| `dsg.inventory_gold.store_summary_current` | Current-state store performance summary |
| `dsg.inventory_gold.inv_category_health` | Category-level inventory health and stockout risk |
| `dsg.inventory_gold.inv_replenish_event_detail` | Event-level replenishment actions |
| `dsg.inventory_gold.inv_stockroom_position` | Current stockroom SKU location and quantity |

### Transformations and Calculations Identified

| Transformation Type | Examples Identified |
|---|---|
| Aggregations | `AVG`, `SUM`, `COUNT`, `COUNT DISTINCT` |
| Ratios and percentages | inventory accuracy %, stockout rate %, nearby stock shown %, zero-result search rate %, RFID coverage % |
| Date filtering | trailing 30 days, latest available date |
| Conditional logic | stockout status checks, alert flag mapping, custom health status ordering |
| Sorting | worst-first category risk, lowest RFID accuracy, latest event timestamps |
| Text generation | AI narrative insight prompt using consolidated KPI values |

### Joins and Intermediate Logic

The source recommendations rely mostly on curated gold-layer and fact tables, minimizing the need for complex joins in the dashboard layer. The business workflow centers on using pre-aggregated daily and current-state summary tables for KPIs and summary visuals, while operational detail tables support drilldown views for replenishment actions and stockroom positions.

### Expected Output Format

- Databricks Genie prompts for dashboard query and visual creation
- Prompts written in business language
- Prompts aligned to Databricks visualization best practices
- Prompts sequenced for dashboard construction
- Prompts formatted for direct use by analysts or dashboard builders

## Recommended Dashboard Prompt Set for Databricks Genie

### Prompt 1: Inventory Accuracy KPI

Create a counter visual that shows the 30-day average inventory accuracy percentage using the `inventory_accuracy_pct` field from `dsg.inventory_gold.inv_accuracy_daily`. Generate a query that filters `calendar_date` to the last 30 days, calculates `AVG(inventory_accuracy_pct) * 100`, and rounds the result to one decimal place. Format the value as a percentage and title the visual `30-Day Inventory Accuracy`, using a clean executive KPI style with a green emphasis color.

### Prompt 2: On-Hand Units KPI

Add a counter visual that displays the total current on-hand units across all stores from `dsg.store_inventory.fact_store_inventory`. Generate a query that calculates `SUM(on_hand_qty)` and label the result as `On-Hand Units`. Format the counter as a whole number with no decimals and use a concise title suitable for the top KPI row.

### Prompt 3: Stockout Rate KPI

Add a counter visual that shows the percentage of inventory rows currently marked as `OUT_OF_STOCK` from `dsg.store_inventory.fact_store_inventory`. Generate a query that calculates the conditional ratio of out-of-stock rows to total rows, multiplies by 100, and rounds to one decimal place. Format the result as a percentage with the title `Current Stockout Rate`, and use red or orange conditional emphasis to highlight elevated stockout conditions.

### Prompt 4: Average SKUs per Cart KPI

Add a counter visual that displays the 30-day average SKUs replenished per cart trip from `dsg.inventory_gold.inv_replenish_daily`. Generate a query that filters the last 30 days using `calendar_date` and calculates `AVG(avg_skus_per_cart)`, rounded to one decimal place. Title the widget `Avg SKUs per Cart` and format it as a compact operational KPI for the fulfillment efficiency section.

### Prompt 5: Nearby-Store Stock Shown Percentage KPI

Add a counter visual that measures the percentage of non-in-stock store-product combinations where nearby-store availability was surfaced, using `dsg.store_inventory.fact_nearby_store_availability` and `dsg.store_inventory.fact_store_inventory`. Generate a query that calculates the distinct covered store-product pairs as a percentage of all rows where `inventory_status` is not `IN_STOCK`, and round the result to one decimal place. Title the visual `Nearby-Store Stock Shown %` and format it as a percentage with clear labeling for the athlete product findability section.

### Prompt 6: Zero-Result Search Rate KPI

Add a counter visual that shows the latest available zero-result search rate from `dsg.store_inventory.fact_daily_store_rollup`. Generate a query that filters to the maximum `rollup_date`, computes `SUM(zero_result_search_count) / SUM(total_search_events) * 100`, and rounds to one decimal place. Title the KPI `Zero-Result Search Rate` and format it as a percentage that is easy to compare with the other top-line findability metrics.

### Prompt 7: RFID Cycle Count Coverage KPI

Add a counter visual for the latest RFID cycle-count coverage percentage using `dsg.inventory_gold.inv_rfid_coverage_daily`. Generate a query that filters to the most recent `calendar_date`, calculates `AVG(rfid_coverage_pct) * 100`, and rounds to one decimal place. Title the visual `RFID Coverage %` and format it as a percentage in the teammate fulfillment efficiency section.

### Prompt 8: Pick Units Per Hour KPI

Add a counter visual that displays the 30-day average pick units per hour from `dsg.stores_gold.fop_pick_uph_daily`. Generate a query that filters the last 30 days using `calendar_date` and computes `AVG(pick_uph)`, rounded to one decimal place. Title the widget `Pick Units per Hour` and format the value as a one-decimal operational KPI.

### Prompt 9: Product-Finding Scan Percentage KPI

Add a counter visual that shows the 30-day product-finding scan rate using `dsg.stores_gold.rta_product_finding_daily`. Generate a query that filters the last 30 days and calculates `SUM(scan_count) / SUM(total_product_views) * 100`, rounded to one decimal place. Title the visual `Product-Finding Scan Rate` and format it as a percentage with clear labeling for store operations users.

### Prompt 10: Store Inventory Performance Table

Add a table visual that compares store-level inventory performance using `dsg.inventory_gold.store_summary_current`. Generate a query that returns store number, store name, region description, on-hand units, RFID accuracy percentage, zero-result rate percentage, out-of-stock SKU count, alert indicator, and alert details, ordered with alert stores first and lowest RFID accuracy next. Format the table with conditional colors for RFID accuracy and zero-result rate, include sortable columns, and title it `Store Inventory Performance`.

### Prompt 11: Category Inventory Health Table

Add a table visual that ranks categories by stockout risk using `dsg.inventory_gold.inv_category_health`. Generate a query that filters `metric_name` to `stockout_risk`, returns category, metric value as stockout risk, and health status, and sorts the results so `Critical` appears before `Risk` and `Healthy`. Format the health status with red, orange, and green indicators and title the table `Category Inventory Health`.

### Prompt 12: Region Filter

Add a multi-select filter widget using distinct `region_description` values from `dsg.inventory_gold.store_summary_current`. Generate a query that returns a sorted distinct list of regions and configure the filter to cross-filter the On-Hand Units KPI, Stockout Rate KPI, and Store Inventory Performance table. Label the widget `Region Filter` and place it above the main store comparison section for easy operational slicing.

### Prompt 13: Replenishment Event Detail Table

Add a table visual that shows the chronological replenishment event log from `dsg.inventory_gold.inv_replenish_event_detail`. Generate a query that returns store number, local action timestamp, product ID, SKU, action text, event quantity, and replenishment session ID, ordered by the latest event time first. Title the table `Replenishment Event Detail` and format the timestamp and quantity columns for operational review and troubleshooting.

### Prompt 14: Stockroom Position Table

Add a table visual that displays current stockroom SKU location details from `dsg.inventory_gold.inv_stockroom_position`. Generate a query that returns store number, product ID, SKU, stockroom location ID, stockroom location type, located quantity, and last updated local timestamp, ordered by highest located quantity first. Title the table `Stockroom Position` and format it as a lookup-friendly operational table with clear quantity and timestamp readability.

### Prompt 15: Executive AI Insight Narrative

Add a text or narrative widget that summarizes overall inventory health using key metrics derived from inventory accuracy, RFID coverage, store alerts, zero-result rate, out-of-stock exposure, and critical category risk. Generate the supporting query by aggregating these KPI values from the appropriate gold tables and then use an AI function to produce a concise three-sentence executive summary. Title the widget `Executive Inventory Insight` and format it as clean plain text placed above the KPI banner to frame the dashboard story.

## Dashboard Assembly Guidance for Genie

### Recommended Visual Sequence

| Order | Visual Name | Visual Type |
|---|---|---|
| 1 | 30-Day Inventory Accuracy | Counter |
| 2 | On-Hand Units | Counter |
| 3 | Current Stockout Rate | Counter |
| 4 | Avg SKUs per Cart | Counter |
| 5 | Nearby-Store Stock Shown % | Counter |
| 6 | Zero-Result Search Rate | Counter |
| 7 | RFID Coverage % | Counter |
| 8 | Pick Units per Hour | Counter |
| 9 | Product-Finding Scan Rate | Counter |
| 10 | Store Inventory Performance | Table |
| 11 | Category Inventory Health | Table |
| 12 | Region Filter | Filter |
| 13 | Replenishment Event Detail | Table |
| 14 | Stockroom Position | Table |
| 15 | Executive Inventory Insight | Text/Narrative |

### Dashboard Design Instructions

| Design Area | Recommendation |
|---|---|
| Layout | Use a single-page executive operations dashboard with KPI counters at the top, summary tables in the middle, and detail tables below |
| Color usage | Use green for healthy performance, orange for risk, and red for critical conditions |
| Titles | Keep titles business-friendly, short, and metric-oriented |
| Number formatting | Use percentages for rates, whole numbers for unit totals, and one decimal place for averages |
| Interactivity | Apply the region filter to relevant KPI and table visuals |
| Readability | Use clear labels, sortable table columns, and conditional formatting where risk interpretation matters |

## Business Analyst Notes

- The prompts are intentionally written so they can be copied into Databricks Genie with minimal editing.
- The first prompt begins with `Create` and all subsequent prompts begin with `Add`, as required.
- The prompts align the reporting requirements to the Lakehouse model and recommended visual types from the previous agent output.
- The dashboard should prioritize fast executive scanning first, then store-level analysis, and finally operational drilldown.

## Final Expected Outcome

The resulting Databricks dashboard should enable business users to monitor inventory health, stock availability, replenishment efficiency, RFID effectiveness, store-level operational risk, and category-level stockout exposure. It should also provide actionable prompts for Genie to generate both the SQL queries and the visuals needed to build a polished and analytically useful dashboard.

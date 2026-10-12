_____________________________________________
## *Author*: AAVA
## *Created on*:   
## *Description*: Refined Databricks Genie reviewer output for inventory dashboard prompt quality, visual suitability, and dashboard usability improvements.
## *Version*: 1
## *Updated on*: 
_____________________________________________

## Summary

This reviewer evaluates and refines the available Databricks Genie and dashboard visual recommendations for the DSG Store Inventory Intelligence Dashboard. It validates prompt structure, confirms visual suitability, improves clarity and formatting, adds missing dashboard visuals for broader insight coverage, and recommends interactivity and storytelling enhancements aligned with Databricks dashboard best practices.

> **Note:** The expected input file `DI_Databricks_Report_Genie_Ally_DIAS/DI_Databricks_Report_Genie_Ally.md` was not found in the repository. This reviewer is therefore based on the available upstream file `DI_Databricks_Dashboard_Visuals_Recommender/DI_Databricks_Dashboard_Visuals_Recommender_1.md`.

---

## 1. Input File Review Status

| Input File | Status | Notes |
|---|---|---|
| `DI_Databricks_Report_Genie_Ally_DIAS/DI_Databricks_Report_Genie_Ally.md` | Not Found | Could not be read from GitHub input directory. |
| `DI_Databricks_Dashboard_Visuals_Recommender/DI_Databricks_Dashboard_Visuals_Recommender_1.md` | Reviewed | Used as the primary source for reviewer refinement. |

---

## 2. Data Flow and Model Understanding

### 2.1 Identified Data Sources

| Data Source | Type | Purpose |
|---|---|---|
| `dsg.inventory_gold.inv_accuracy_daily` | Gold aggregated table | Daily inventory accuracy metrics |
| `dsg.store_inventory.fact_store_inventory` | Fact table | Store inventory quantities and stock status |
| `dsg.inventory_gold.inv_replenish_daily` | Gold aggregated table | Daily replenishment efficiency metrics |
| `dsg.store_inventory.fact_nearby_store_availability` | Fact table | Nearby-store availability coverage |
| `dsg.store_inventory.fact_daily_store_rollup` | Fact table | Daily store search and rollup metrics |
| `dsg.inventory_gold.inv_rfid_coverage_daily` | Gold aggregated table | RFID cycle-count coverage metrics |
| `dsg.stores_gold.fop_pick_uph_daily` | Gold aggregated table | Fulfillment pick units-per-hour |
| `dsg.stores_gold.rta_product_finding_daily` | Gold aggregated table | Product-finding scan activity |
| `dsg.inventory_gold.store_summary_current` | Current-state summary table | Store-level inventory and alert summary |
| `dsg.inventory_gold.inv_category_health` | Gold metric table | Category-level stockout health indicators |
| `dsg.inventory_gold.inv_replenish_event_detail` | Event detail table | Detailed replenishment activity log |
| `dsg.inventory_gold.inv_stockroom_position` | Current-state detail table | Stockroom item location and quantity |

### 2.2 Target Outputs

| Output Area | Format | Purpose |
|---|---|---|
| KPI widgets | Counter | Highlight top operational metrics |
| Tabular operational views | Table | Support store, category, replenishment, and stockroom analysis |
| Dashboard filter | Multi-select filter | Region-based interactive slicing |
| Executive narrative | Text/Counter-style text widget | Summarize business performance in natural language |

### 2.3 Intermediate Transformations

| Transformation Type | Examples Identified |
|---|---|
| Aggregations | `AVG`, `SUM`, `COUNT`, conditional ratios, percentage calculations |
| Date filtering | Last 30 days, latest available day filters |
| Conditional logic | `CASE WHEN` for stockouts, alerts, and custom status ordering |
| Sorting | Descending event time, risk-first category ranking, alert-first store ordering |
| Derived formatting | Rounded percentages, emoji alert indicators, status labels |
| Prompt generation | Multi-CTE synthesis feeding `ai_query()` for executive narrative |

### 2.4 Joins and Cross-Source Logic

| Join / Combination Type | Details |
|---|---|
| Cross joins in CTE workflow | Used in AI narrative generation to combine KPI summaries |
| Correlated denominator logic | Used in nearby-store coverage calculation |
| Single-table KPI queries | Majority of KPIs derive from one curated source table |
| Filter-driven reuse | Region filter reused across KPI and detail widgets |

### 2.5 Filters Identified

| Filter | Scope |
|---|---|
| `calendar_date >= CURRENT_DATE() - INTERVAL 30 DAYS` | Trailing 30-day KPI trend and average metrics |
| `rollup_date = MAX(rollup_date)` | Latest daily rollup metrics |
| `calendar_date = MAX(calendar_date)` | Latest RFID coverage metric |
| `inventory_status = 'OUT_OF_STOCK'` | Stockout KPI calculation |
| `metric_name = 'stockout_risk'` | Category health filtering |
| Region multi-select | Interactive dashboard filter |

---

## 3. Reviewer Findings

### 3.1 Prompt Structure Validation

| Rule | Review Result | Reviewer Comment |
|---|---|---|
| First prompt should start with **Create** | Needs refinement | Existing source is a recommendation document, not final Genie prompts. Final prompts below now follow the required rule. |
| Remaining prompts should start with **Add** | Needs refinement | Revised prompt set has been normalized accordingly. |
| Clear action-driven prompt language | Partially met | Source content was descriptive; converted into direct Databricks Genie dashboard instructions. |
| Explicit formatting guidance | Partially met | Added titles, axis labels, legends, colors, sorting, and filter behavior. |

### 3.2 Visual Suitability Validation

| Metric / View | Existing Recommendation | Reviewer Assessment | Outcome |
|---|---|---|---|
| Inventory Accuracy | Counter | Appropriate | Retained |
| On-Hand Units | Counter | Appropriate | Retained |
| Stockout Rate | Counter | Appropriate | Retained |
| Avg SKUs per Cart | Counter | Appropriate | Retained |
| Nearby-Store Stock Shown % | Counter | Appropriate | Retained |
| Zero-Result Search Rate | Counter | Appropriate | Retained |
| RFID Coverage % | Counter | Appropriate | Retained |
| Pick UPH / Scan % | Counter | Appropriate | Retained |
| Store Inventory Performance | Table | Appropriate for operational scan | Retained |
| Category Inventory Health | Table | Appropriate, but can be complemented by chart | Retained with enhancement recommendation |
| Replenishment Event Detail | Table | Appropriate | Retained |
| Stockroom Position | Table | Appropriate | Retained |
| Executive AI Insight | Text/Narrative | Appropriate | Retained |

### 3.3 Gaps Identified

The existing recommendations were strong but mostly KPI- and table-oriented. To improve insight depth and storytelling, the dashboard should include additional comparative and trend-based visuals.

---

## 4. Refined Databricks Genie Prompts

### 4.1 Core Prompt Set

1. **Create** a top KPI banner for the DSG Store Inventory Intelligence Dashboard showing Inventory Accuracy %, On-Hand Units, Stockout Rate %, and Zero-Result Search Rate %, using counter visuals with concise titles, green-positive and red-risk color semantics, and values formatted with appropriate percent symbols and whole-number separators.

2. **Add** a KPI counter for RFID Cycle Count Coverage % using the latest available date, formatted as a percentage with one decimal place and titled **RFID Coverage %**.

3. **Add** a KPI counter for Avg SKUs per Cart using the trailing 30-day average, formatted to one decimal place and titled **Avg SKUs per Cart**.

4. **Add** a KPI counter for Pick Units Per Hour using the trailing 30-day average, formatted to one decimal place and titled **Pick UPH**.

5. **Add** a KPI counter for Product-Finding Scan Rate using the trailing 30-day period, formatted as a percentage with one decimal place and titled **Product Finding Scan %**.

6. **Add** a KPI counter for Nearby-Store Stock Shown % using distinct product-store coverage logic, formatted as a percentage with one decimal place and titled **Nearby Store Availability Shown %**.

7. **Add** a multi-select Region filter sourced from distinct region descriptions, sorted alphabetically, and configure it to cross-filter store-level KPI and table visuals.

8. **Add** a table titled **Store Inventory Performance** showing Store, Store Name, Region, On-Hand Units, RFID Accuracy %, Zero-Result Search Rate %, Out-of-Stock SKU Count, Alert Indicator, and Alert Detail; sort by Alert descending and RFID Accuracy ascending; apply conditional formatting for low RFID accuracy, high zero-result rate, and high out-of-stock counts.

9. **Add** a table titled **Category Inventory Health** showing Category, Stockout Risk Score, and Health Status; sort Critical first, Risk second, Healthy third; use status-based conditional formatting with red, orange, and green indicators.

10. **Add** a table titled **Replenishment Event Detail** showing Store Number, Event Time, Product ID, SKU, Action Text, Event Quantity, and Replenishment Session ID; sort by Event Time descending and enable table search for operational lookup.

11. **Add** a table titled **Stockroom Position** showing Store Number, Product ID, SKU, Stockroom Location ID, Stockroom Location Type, Located Quantity, and Last Updated Timestamp; sort by Located Quantity descending.

12. **Add** a narrative text widget titled **Executive Inventory Insight** that summarizes overall inventory performance using an AI-generated three-sentence business summary from the curated KPI metrics.

### 4.2 Additional Recommended Visual Prompts

13. **Add** a line chart titled **30-Day Inventory Accuracy Trend** showing calendar date on the X-axis and Inventory Accuracy % on the Y-axis, using a green line with markers and a subtitle indicating the trailing 30-day trend.

14. **Add** a bar chart titled **Out-of-Stock SKU Count by Region** showing Region on the X-axis and total Out-of-Stock SKU Count on the Y-axis, sorted from highest to lowest, with red bars to emphasize risk concentration.

15. **Add** a stacked bar chart titled **Inventory Status Breakdown by Region** showing Region on the X-axis, unit count on the Y-axis, and stack color by Inventory Status to compare in-stock versus out-of-stock composition.

16. **Add** a line chart titled **Zero-Result Search Rate Trend** showing rollup date on the X-axis and Zero-Result Search Rate % on the Y-axis, with an orange line and tooltip details for date and percentage.

17. **Add** a horizontal bar chart titled **Top 10 Stores by Out-of-Stock SKU Count** showing Store on the Y-axis and Out-of-Stock SKU Count on the X-axis, sorted descending, to quickly identify the highest-risk stores.

---

## 5. Refined SQL-to-Visual Mapping Table

| Visual Title | Visual Type | Primary Measure(s) | Main Dimension(s) | Key Formatting / Behavior |
|---|---|---|---|---|
| Inventory Accuracy % | Counter | Average inventory accuracy % | None | One decimal place, percent format |
| On-Hand Units | Counter | Sum of on-hand quantity | None | Whole number with separators |
| Stockout Rate % | Counter | Stockout percentage | None | One decimal place, red-risk format |
| Zero-Result Search Rate % | Counter | Zero-result percentage | None | One decimal place |
| RFID Coverage % | Counter | Average RFID coverage % | None | One decimal place |
| Avg SKUs per Cart | Counter | Average SKUs per cart | None | One decimal place |
| Pick UPH | Counter | Average pick UPH | None | One decimal place |
| Product Finding Scan % | Counter | Scan rate % | None | One decimal place |
| Nearby Store Availability Shown % | Counter | Coverage % | None | One decimal place |
| 30-Day Inventory Accuracy Trend | Line chart | Inventory accuracy % | Calendar date | Green line, markers |
| Out-of-Stock SKU Count by Region | Bar chart | OOS SKU count | Region | Descending sort, red bars |
| Inventory Status Breakdown by Region | Stacked bar chart | Unit count | Region, inventory status | Color by status |
| Zero-Result Search Rate Trend | Line chart | Zero-result % | Rollup date | Orange line |
| Top 10 Stores by Out-of-Stock SKU Count | Horizontal bar chart | OOS SKU count | Store | Top 10 descending |
| Store Inventory Performance | Table | Multiple store KPIs | Store, Region | Conditional formatting |
| Category Inventory Health | Table | Stockout risk, status | Category | Status colors |
| Replenishment Event Detail | Table | Event-level detail | Time, Store, SKU | Search + descending sort |
| Stockroom Position | Table | Current location quantity | Store, SKU | Descending quantity sort |
| Executive Inventory Insight | Text widget | AI-generated narrative | None | Plain-text narrative |

---

## 6. Dashboard Best Practice Enhancements

### 6.1 Formatting and Consistency

| Area | Recommendation |
|---|---|
| Titles | Use business-friendly titles, not physical table names |
| Axis labels | Always show explicit labels such as **Date**, **Rate %**, **Units**, **Region**, and **Store** |
| Colors | Use green for healthy metrics, orange for caution, red for risk; maintain consistency across KPI and tables |
| Legends | Enable legends for stacked and comparative charts |
| Numbers | Use one decimal for rates and percentages; whole numbers for counts and unit totals |

### 6.2 Usability Enhancements

| Enhancement | Recommendation |
|---|---|
| Global filters | Add Region and optional Date Range filters |
| Drill-through behavior | Enable navigation from top-risk stores to replenishment and stockroom details where supported |
| Narrative guidance | Keep the AI insight at top of dashboard for executive readability |
| Chart variety | Combine KPI, trend, breakdown, and operational tables for balanced storytelling |
| Sorting defaults | Sort risk-focused visuals descending so issues appear first |

### 6.3 Performance Considerations

| Area | Recommendation |
|---|---|
| Heavy fact tables | Use curated gold aggregates when possible |
| Event detail tables | Apply default date range or row-limiting behavior |
| AI-generated summary | Refresh less frequently than base KPI datasets |
| Shared dimensions | Reuse region filter dataset across visuals |

---

## 7. Final Reviewer Approval

| Review Area | Status | Notes |
|---|---|---|
| Prompt clarity | Approved with refinement | Converted narrative recommendations into actionable Genie prompts |
| Visual suitability | Approved | Existing visual choices were largely appropriate |
| Dashboard completeness | Improved | Added five supplemental visuals for trend and comparative analysis |
| Data field usage | Approved | Key measures, dimensions, and filters are properly represented |
| Formatting guidance | Approved | Titles, colors, labels, and sorting now clearly specified |
| Databricks best practice alignment | Approved | Refined output supports usability, performance, and storytelling |

## Final Approved Output

The refined set of Databricks Genie prompts above is approved for use in building an insightful, user-friendly, and operationally effective Databricks dashboard for store inventory intelligence.

---

## Output URL (Clickable Hyperlinks)

[DI_Databricks_Report_Genie_Ally_Reviewer_1.md](https://github.com/DIAscendion/Databricks_Dashboard/blob/main/DI_Databricks_Report_Review/DI_Databricks_Report_Genie_Ally_Reviewer_1.md)
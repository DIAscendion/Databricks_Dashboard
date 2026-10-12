_____________________________________________
## *Author*: AAVA
## *Created on*:
## *Description*: Databricks Genie prompts for building dashboard queries and visuals for the Retail Sales Gold Dashboard, based on the gold-layer PySpark pipeline (03_gold_dimensions, 04_gold_fact, 05_gold_aggregates).
## *Version*: 2
## *Updated on*:
_____________________________________________

## Summary

This document converts the gold-layer sales model into business-analyst-ready Databricks Genie prompts. Each prompt tells Genie which query to generate, which visual to build, which fields and calculations to use, and how to format the result. Every prompt uses **only** the eight `gold` tables created by the PySpark scripts.

## Source Input Reference

- **Input Files Used:** `03_gold_dimensions.py`, `04_gold_fact.py`, `05_gold_aggregates.py`, and `Retail_Sales_Gold_Dashboard.lvdash.json`
- **Input Purpose:** Define the gold tables, business rules and the five-page dashboard layout that these prompts rebuild.
- **Primary Data Domains Identified:**
  - Net (recognized) revenue
  - Orders, units and average order value
  - Discounts
  - Cancellations and returns
  - Category performance
  - Store performance
  - Customer value and tiers
  - Payment mode and order status mix

## Lakehouse Data Model and Workflow Understanding

### Data Sources Identified

| Data Source Table | Purpose |
|---|---|
| `gold.agg_daily_store_sales` | Daily store KPIs: orders, units, net revenue, discount, cancelled and returned value |
| `gold.agg_monthly_category_sales` | Monthly category KPIs: orders, units, gross, discount, net revenue, average line value |
| `gold.agg_customer_summary` | Lifetime customer metrics, first and last order dates, customer tier |
| `gold.fact_sales` | Order-line fact (grain: `order_id` + `product_id`) with `recognized_revenue` |
| `gold.dim_date` | Calendar dimension (`date_key`, `full_date`, `year_month`, `day_name`) |
| `gold.dim_store` | Store dimension (`store_sk`, `store_id`, `store_name`) |
| `gold.dim_product` | Product dimension (`product_sk`, `product_name`, `category`, `list_price`) |
| `gold.dim_customer` | Customer dimension (`customer_sk`, `customer_name`, `city`, `state`) |

### Business Rules to Respect

| Rule | Detail |
|---|---|
| Revenue recognition | Revenue = `COMPLETED` orders only. `net_revenue` in the daily store table and `recognized_revenue` in `fact_sales` already apply this rule. |
| Cancelled and returned | `cancelled_value` and `returned_value` hold the `net_amount` of `CANCELLED` and `RETURNED` lines. They are not part of net revenue. |
| Unknown members | Missing dimension keys map to surrogate key `-1` (`UNKNOWN`). Use LEFT joins so no fact row is dropped. |
| Join keys | `fact_sales` joins to dimensions on `date_key`, `store_sk`, `product_sk` and `customer_sk`. |
| Customer tier | `GOLD` when `lifetime_revenue` >= 50,000, `SILVER` when >= 5,000, otherwise `BRONZE`. |
| Monthly category table | Has `year`, `month`, `year_month` only (no calendar date). Build a month date with `make_date(year, month, 1)` for trend charts. |

### Transformations and Calculations Identified

| Calculation | Expression |
|---|---|
| Average order value | `SUM(net_revenue) / NULLIF(SUM(completed_orders), 0)` |
| Order completion rate % | `SUM(completed_orders) / NULLIF(SUM(total_orders), 0) * 100` |
| Discount rate % (daily store) | `SUM(total_discount) / NULLIF(SUM(net_revenue) + SUM(total_discount), 0) * 100` |
| Discount % of gross (category) | `SUM(total_discount) / NULLIF(SUM(gross_revenue), 0) * 100` |
| Cancel + return % of booked | `(SUM(cancelled_value) + SUM(returned_value)) / NULLIF(SUM(net_revenue) + SUM(cancelled_value) + SUM(returned_value), 0) * 100` |
| Net revenue per unit | `SUM(net_revenue) / NULLIF(SUM(units_sold), 0)` |
| Repeat customers % | Customers with `completed_orders` > 1 divided by all customers, times 100 |

### Expected Output Format

- Databricks Genie prompts for dashboard query and visual creation
- Prompts written in business language
- Prompts sequenced by dashboard page
- Prompts formatted for direct use by analysts or dashboard builders

## Recommended Dashboard Prompt Set for Databricks Genie

### Page 1: Sales Scorecard

#### Prompt 1: Net Revenue KPI

Create a counter visual that shows total net revenue from completed orders using `gold.agg_daily_store_sales`. Generate a query that calculates `SUM(net_revenue)`, label the result `Net Revenue (completed)`, and format it as a compact whole number with no decimals. Use a clean executive KPI style with a green emphasis color.

#### Prompt 2: Completed Orders KPI

Add a counter visual that displays total completed orders from `gold.agg_daily_store_sales`. Generate a query that calculates `SUM(completed_orders)` and title it `Completed Orders`. Format it as a compact whole number.

#### Prompt 3: Units Sold KPI

Add a counter visual that shows total units sold from `gold.agg_daily_store_sales`. Generate a query that calculates `SUM(units_sold)` and title it `Units Sold`. Format it as a compact whole number.

#### Prompt 4: Average Order Value KPI

Add a counter visual that shows average order value from `gold.agg_daily_store_sales`. Generate a query that calculates `SUM(net_revenue) / NULLIF(SUM(completed_orders), 0)` and title it `Avg Order Value`. Format it with two decimal places.

#### Prompt 5: Order Completion Rate KPI

Add a counter visual that shows the share of orders that were completed, using `gold.agg_daily_store_sales`. Generate a query that calculates `SUM(completed_orders) / NULLIF(SUM(total_orders), 0) * 100` and title it `Order Completion Rate`. Format it as a percentage with one decimal place, using orange emphasis when the rate is low.

#### Prompt 6: Discount Rate KPI

Add a counter visual that shows the discount rate on completed sales from `gold.agg_daily_store_sales`. Generate a query that calculates `SUM(total_discount) / NULLIF(SUM(net_revenue) + SUM(total_discount), 0) * 100` and title it `Discount Rate`. Format it as a percentage with one decimal place.

#### Prompt 7: Total Orders KPI

Add a counter visual that displays total orders across all statuses from `gold.agg_daily_store_sales`. Generate a query that calculates `SUM(total_orders)` and title it `Total Orders (all statuses)`. Format it as a compact whole number.

#### Prompt 8: Cancelled Value KPI

Add a counter visual that shows the value of cancelled orders from `gold.agg_daily_store_sales`. Generate a query that calculates `SUM(cancelled_value)` and title it `Cancelled Value`. Format it as a compact whole number with red emphasis.

#### Prompt 9: Returned Value KPI

Add a counter visual that shows the value of returned orders from `gold.agg_daily_store_sales`. Generate a query that calculates `SUM(returned_value)` and title it `Returned Value`. Format it as a compact whole number with orange emphasis.

#### Prompt 10: Cancel + Return Percentage KPI

Add a counter visual that shows how much booked value was lost to cancellations and returns, using `gold.agg_daily_store_sales`. Generate a query that calculates `(SUM(cancelled_value) + SUM(returned_value)) / NULLIF(SUM(net_revenue) + SUM(cancelled_value) + SUM(returned_value), 0) * 100` and title it `Cancel + Return % of Booked`. Format it as a percentage with one decimal place.

#### Prompt 11: Daily Net Revenue Trend

Add a line chart that shows daily net revenue from `gold.agg_daily_store_sales`. Generate a query that returns `sales_date` and `SUM(net_revenue)` grouped by `sales_date`, with the date on the x-axis. Title it `Daily Net Revenue`.

#### Prompt 12: Monthly Net Revenue by Category Trend

Add a line chart that shows monthly net revenue for each product category from `gold.agg_monthly_category_sales`. Generate a query that builds a month date using `make_date(year, month, 1)`, sums `net_revenue`, and groups by that month and `category`. Use the month on the x-axis, a separate line per category, and title it `Monthly Net Revenue by Category`.

#### Prompt 13: Net Revenue by Store

Add a bar chart that ranks stores by net revenue using `gold.agg_daily_store_sales`. Generate a query that returns `store_name` and `SUM(net_revenue)`, sorted highest first. Title it `Net Revenue by Store (highest first)`.

#### Prompt 14: Store Performance Table

Add a table visual that compares stores using `gold.agg_daily_store_sales`. Generate a query that returns `store_id`, `store_name`, net revenue, completed orders, units sold, average order value, completion percentage and discount percentage, using the calculations listed above and ordered by net revenue descending. Title it `Store Performance` and show money columns with two decimals and percentages with one decimal.

#### Prompt 15: Top Customers Table

Add a table visual that lists the highest-value customers from `gold.agg_customer_summary`. Generate a query that returns `customer_id`, `customer_name`, `city`, `state`, `customer_tier`, `completed_orders`, `lifetime_revenue` and `avg_order_value`, ordered by `lifetime_revenue` descending and limited to 10 rows per page. Title it `Top Customers by Lifetime Revenue`.

#### Prompt 16: Store Filter

Add a multi-select filter widget using distinct `store_name` values from `gold.agg_daily_store_sales`. Generate a query that returns a sorted distinct list of stores and configure the filter to apply to the Page 1 store KPIs, trend and tables. Label it `Store`.

#### Prompt 17: Category Filter

Add a multi-select filter widget using distinct `category` values from `gold.dim_product`. Generate a query that returns a sorted distinct list of categories and configure the filter to apply to the category-based visuals on Pages 1, 2 and 5. Label it `Category`.

#### Prompt 18: Date Range Filter

Add a date range picker filter using `sales_date` from `gold.agg_daily_store_sales`. Configure it to apply to the daily store visuals on Pages 1 and 3. Label it `Date range` and place it in the filter row at the top of the page.

### Page 2: Category Sales

#### Prompt 19: Category Revenue KPIs

Add three counter visuals from `gold.agg_monthly_category_sales`: `Net Revenue` as `SUM(net_revenue)`, `Gross Revenue` as `SUM(gross_revenue)`, and `Total Discount` as `SUM(total_discount)`. Format each as a compact whole number and place them side by side at the top of the page.

#### Prompt 20: Discount Percentage of Gross KPI

Add a counter visual that shows the discount percentage of gross revenue from `gold.agg_monthly_category_sales`. Generate a query that calculates `SUM(total_discount) / NULLIF(SUM(gross_revenue), 0) * 100` and title it `Discount % of Gross`. Format it as a percentage with one decimal place.

#### Prompt 21: Units and Revenue per Unit KPIs

Add two counter visuals from `gold.agg_monthly_category_sales`: `Units Sold` as `SUM(units_sold)` formatted as a compact whole number, and `Net Revenue per Unit` as `SUM(net_revenue) / NULLIF(SUM(units_sold), 0)` formatted with two decimals.

#### Prompt 22: Category Trend and Mix

Add a line chart of monthly net revenue by category (month on the x-axis, one line per `category`) and a bar chart of net revenue by category, both from `gold.agg_monthly_category_sales`. Title them `Monthly Net Revenue by Category` and `Net Revenue by Category`, and sort the bar chart highest first.

#### Prompt 23: Discount and Units by Category

Add a bar chart titled `Discount % of Gross by Category` using `SUM(total_discount) / NULLIF(SUM(gross_revenue), 0) * 100` grouped by `category`, and a line chart titled `Monthly Units Sold by Category` using `SUM(units_sold)` by month and category, both from `gold.agg_monthly_category_sales`.

#### Prompt 24: Category by Month Heatmap

Add a heatmap visual from `gold.agg_monthly_category_sales` with `year_month` on the x-axis, `category` on the y-axis and `SUM(net_revenue)` as the color. Title it `Net Revenue by Category & Month`.

#### Prompt 25: Category Sales Table

Add a table visual from `gold.agg_monthly_category_sales` that returns `year_month`, `category`, orders, units sold, gross revenue, discount, net revenue and average line value. Title it `Category Sales by Month`, with money columns at two decimals.

#### Prompt 26: Year and Year-Month Filters

Add two multi-select filters from `gold.agg_monthly_category_sales`, one on `year` and one on `year_month`, and apply them to all Page 2 visuals. Label them `Year` and `Year-Month`.

### Page 3: Store Performance

#### Prompt 27: Recognized Revenue vs Leakage KPIs

Add four counter visuals from `gold.agg_daily_store_sales`: `Recognized (Completed) Revenue` as `SUM(net_revenue)`, `Cancelled Value` as `SUM(cancelled_value)`, `Returned Value` as `SUM(returned_value)`, and `Cancel + Return % of Booked` using the percentage calculation above. Show the first three as compact whole numbers and the last as a one-decimal percentage.

#### Prompt 28: Daily Net vs Cancelled vs Returned

Add a line chart from `gold.agg_daily_store_sales` with `sales_date` on the x-axis and three series: `SUM(net_revenue)`, `SUM(cancelled_value)` and `SUM(returned_value)`. Title it `Daily Net vs Cancelled vs Returned`, using green for net, red for cancelled and orange for returned.

#### Prompt 29: Store Leaderboards

Add three bar charts from `gold.agg_daily_store_sales`, each by `store_name`: `Order Completion % by Store`, `Cancelled Value by Store` and `Returned Value by Store`. Sort each highest first and place them side by side.

#### Prompt 30: Store by Month Heatmap

Add a heatmap from `gold.agg_daily_store_sales` with `date_format(sales_date, 'yyyy-MM')` on the x-axis, `store_name` on the y-axis and `SUM(net_revenue)` as the color. Title it `Net Revenue by Store & Month`.

#### Prompt 31: Store Detail Table

Add a table visual from `gold.agg_daily_store_sales` that returns store ID, store name, total orders, completed orders, units, net revenue, discount, cancelled value, returned value and cancel + return percentage. Title it `Store Detail` and sort by net revenue descending.

#### Prompt 32: Store-Month Filter

Add a multi-select filter using `date_format(sales_date, 'yyyy-MM')` from `gold.agg_daily_store_sales`, labeled `Year-Month`, applied to all Page 3 visuals together with the Store and Date range filters.

### Page 4: Customers

#### Prompt 33: Customer KPIs

Add four counter visuals from `gold.agg_customer_summary`: `Customers` as `COUNT(DISTINCT customer_id)`, `Lifetime Revenue` as `SUM(lifetime_revenue)`, `Avg Order Value` as `SUM(lifetime_revenue) / NULLIF(SUM(completed_orders), 0)`, and `Repeat Customers %` as customers with more than one completed order divided by all customers times 100.

#### Prompt 34: Tier Mix

Add two bar charts from `gold.agg_customer_summary`: `Customers by Tier` using `COUNT(DISTINCT customer_id)` and `Lifetime Revenue by Tier` using `SUM(lifetime_revenue)`, both grouped by `customer_tier`. Color `GOLD` amber, `SILVER` grey and `BRONZE` brown.

#### Prompt 35: Geography and Acquisition

Add a bar chart titled `Lifetime Revenue by State` using `SUM(lifetime_revenue)` by `state`, and a line chart titled `New Customers by First-Order Month` using `COUNT(DISTINCT customer_id)` by `trunc(first_order_date, 'MM')`, both from `gold.agg_customer_summary`.

#### Prompt 36: Customer Detail Table

Add a table visual from `gold.agg_customer_summary` that returns customer ID, name, city, state, tier, total orders, completed orders, lifetime revenue, average order value, first order date and last order date, ordered by `lifetime_revenue` descending. Title it `Customer Summary` and show the detail rows without aggregation.

#### Prompt 37: Customer Filters

Add multi-select filters on `customer_tier`, `state` and `city` and a date range filter on `first_order_date`, all from `gold.agg_customer_summary`, applied to every Page 4 visual. Label them `Customer tier`, `State`, `City` and `First order date`.

### Page 5: Order Detail

#### Prompt 38: Order Detail Dataset

Add a dataset (not a visual) that joins `gold.fact_sales` to `gold.dim_date` on `date_key`, `gold.dim_store` on `store_sk`, `gold.dim_product` on `product_sk` and `gold.dim_customer` on `customer_sk`, all with LEFT joins. Return `full_date` as `sales_date`, `year_month`, store name, category, product name, customer name, city, state, order ID, payment mode, order status, quantity, gross amount, discount amount, net amount and `recognized_revenue`. Use this dataset for every Page 5 visual.

#### Prompt 39: Order Detail KPIs

Add six counter visuals from the order detail dataset: `Recognized Revenue` as `SUM(recognized_revenue)`, `Gross Amount` as `SUM(gross_amount)`, `Discount Amount` as `SUM(discount_amount)`, `Net Amount (all statuses)` as `SUM(net_amount)`, `Distinct Orders` as `COUNT(DISTINCT order_id)`, and `Units` as `SUM(quantity)`. Format them as compact whole numbers.

#### Prompt 40: Payment Mode Analysis

Add a line chart titled `Monthly Recognized Revenue by Payment Mode` (month on the x-axis, one line per `payment_mode`, `SUM(recognized_revenue)`) and a bar chart titled `Recognized Revenue by Payment Mode`, both from the order detail dataset.

#### Prompt 41: Orders and Products Mix

Add three bar charts from the order detail dataset: `Orders by Status` using `COUNT(DISTINCT order_id)` by `order_status`, `Recognized Revenue by Category` by `category`, and `Top Products by Recognized Revenue` by `product_name`. Sort each highest first and color `COMPLETED` green, `CANCELLED` red and `RETURNED` orange.

#### Prompt 42: Order Line Detail Table

Add a table visual from the order detail dataset that returns date, order ID, store, customer, product, category, payment mode, order status, quantity, gross, discount, net and recognized revenue, without aggregation. Title it `Order Line Detail` and format dates readably and money columns with two decimals.

#### Prompt 43: Order Detail Filters

Add multi-select filters on `store_name`, `category`, `payment_mode`, `order_status` and `state`, plus a date range filter on `sales_date`, all from the order detail dataset and applied to every Page 5 visual.

## Dashboard Assembly Guidance for Genie

### Recommended Visual Sequence

| Page | Visuals | Prompts |
|---|---|---|
| 1. Sales Scorecard | Filters, 9 counters, 2 trend lines, store bar, store table, top customers table | 1 to 18 |
| 2. Category Sales | Filters, 6 counters, 2 lines, 2 bars, heatmap, table | 19 to 26 |
| 3. Store Performance | Filters, 4 counters, daily line, 3 bars, heatmap, table | 27 to 32 |
| 4. Customers | Filters, 4 counters, tier bars, state bar, new-customer line, table | 33 to 37 |
| 5. Order Detail | Filters, 6 counters, payment line and bar, 3 mix bars, line-level table | 38 to 43 |

### Dashboard Design Instructions

| Design Area | Recommendation |
|---|---|
| Layout | Five canvas pages, each with filters across the top, counters below, charts in the middle and detail tables at the bottom |
| Color usage | Green for healthy performance and net revenue, orange for returns and risk, red for cancellations and critical conditions |
| Titles | Keep titles business-friendly, short and metric-oriented |
| Number formatting | Compact whole numbers for revenue and counts, one decimal for percentages, two decimals for averages and table money columns |
| Interactivity | Each filter only affects visuals built on datasets that contain that field, so apply filters per page as listed |
| Readability | Use clear labels, sortable table columns and section headers above each group of visuals |

## Business Analyst Notes

- The first prompt begins with `Create` and all subsequent prompts begin with `Add`, as required.
- All table references use the `gold.` schema exactly as written in the PySpark scripts. Add the catalog prefix if your workspace needs it.
- Revenue already excludes cancelled and returned orders, so never add `cancelled_value` or `returned_value` back into net revenue.
- Run the gold scripts in order (03, 04, 05) before building the dashboard so all eight tables exist.

## Final Expected Outcome

The resulting Databricks dashboard should let business users monitor revenue, orders, discounts, cancellations and returns, category and store performance, and customer value, all built exclusively on the gold-layer tables produced by the PySpark pipeline.

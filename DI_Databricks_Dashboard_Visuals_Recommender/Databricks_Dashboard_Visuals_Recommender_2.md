_____________________________________________
## *Author*: AAVA
## *Created on*:   
## *Description*: Databricks Lakeview Dashboard (LVDash) visuals, SQL, data model, and interactivity recommendations for Rental Revenue-to-Cash reporting across executive scorecard, revenue, AR aging, collections, and credit risk domains.
## *Version*: 2
## *Updated on*: 
_____________________________________________

## Summary

This document analyzes the Rental Revenue-to-Cash reporting requirements and the supporting Databricks Silver physical model to recommend the most appropriate Databricks SQL queries, reusable analytical datasets, and Lakeview Dashboard visuals. The recommendations cover executive scorecards, revenue and billing performance, AR aging and collections, cash application, and customer credit risk monitoring. It identifies the likely data sources, target analytical structures, joins, aggregations, filters, and performance optimizations required to build an enterprise-grade Lakeview dashboard in Databricks.

**GitHub Output URL:** https://github.com/DIAscendion/Databricks_Dashboard/tree/main/DI_Databricks_Dashboard_Visuals_Recommender

---

## 1. Input Analysis

### 1.1 Source Inputs Reviewed

| Input File | Purpose | Key Contribution to Dashboard Recommendations |
|---|---|---|
| `DI_Databricks_Silver_Model_Physical/Databricks_Silver_Model_Physical_1.md` | Defines Silver-layer physical tables for rental contracts, invoices, cash receipts, customers, employees, audit, and error handling | Provides source entities, join keys, data types, and optimization patterns for downstream dashboard datasets |
| `Input/Rental_Revenue_to_Cash_Reports.txt` | Defines business reporting requirements and expected KPIs for executive scorecard, revenue, AR, cash application, and credit risk | Provides required insights, metrics, visual objectives, drill paths, calculations, and security/access expectations |

### 1.2 Identified Data Sources

The reporting requirements and Silver physical model indicate the following core source tables and downstream analytical facts/dimensions.

#### Silver Sources
- `silver.si_rental_contracts`
- `silver.si_invoices`
- `silver.si_cash_receipts`
- `silver.si_customer_master`
- `silver.si_branch_employee`
- `silver.si_process_audit`
- `silver.si_data_quality_errors`

#### Expected Gold / Curated Analytical Tables
Based on the report requirements, the dashboard should preferably read from curated Gold tables or materialized views such as:
- `gold.fct_rental_revenue`
- `gold.fct_ar_aging`
- `gold.fct_cash_application`
- `gold.fct_credit_risk`
- `gold.fct_revenue_to_cash_health_score`
- `gold.dim_branch`
- `gold.dim_region`
- `gold.dim_customer`
- `gold.dim_sales_rep`
- `gold.dim_collector`
- `gold.dim_equipment_class`
- `gold.dim_date`

### 1.3 Identified Intermediate Transformations

| Transformation Area | Description |
|---|---|
| Revenue normalization | Standardize invoice amounts into gross billings, credit memos, and net billings |
| AR aging logic | Calculate days past due and assign aging buckets: `0-30`, `31-60`, `61-90`, `90+` |
| Cash application conformance | Match cash receipts to invoices, identify applied vs unapplied cash, and calculate days-to-pay |
| Health score calculation | Compute weighted composite score using revenue, AR, cash application, and credit risk domain scores |
| Credit exposure rollups | Aggregate customer-level outstanding AR, AR 90+, open contracts, and credit utilization |
| Hierarchical drill structure | Resolve Region -> Branch -> Sales Rep / Collector / Customer paths |
| Snapshot and period rollups | Produce daily, weekly, and monthly summary datasets for trends and KPI cards |

### 1.4 Expected Joins

| Left Table | Join Field | Right Table | Purpose |
|---|---|---|---|
| `gold.fct_rental_revenue` | `branch_id` | `gold.dim_branch` | Branch reporting context |
| `gold.dim_branch` | `region_id` | `gold.dim_region` | Regional hierarchy |
| `gold.fct_rental_revenue` | `sales_rep_id` | `gold.dim_sales_rep` | Sales performance slicing |
| `gold.fct_rental_revenue` | `equipment_class_id` | `gold.dim_equipment_class` | Equipment mix analysis |
| `gold.fct_ar_aging` | `customer_id` | `gold.dim_customer` | Customer AR exposure |
| `gold.fct_ar_aging` | `collector_id` | `gold.dim_collector` | Collections ownership |
| `gold.fct_cash_application` | `invoice_id` | `gold.fct_ar_aging` or invoice detail view | Payment-to-invoice linkage |
| `gold.fct_credit_risk` | `customer_id` | `gold.dim_customer` | Credit utilization and suspended exposure |
| All facts | `date_id` | `gold.dim_date` | Daily/weekly/monthly trend analysis |

### 1.5 Core Filters and Dimensions
- Region
- Branch
- Date / Period Type
- Sales Rep
- Collector
- Equipment Class
- Customer Status
- Aging Bucket
- Payment Match Status
- Contract Status

### 1.6 Expected Output Format
- Markdown-based LVDash Visuals Recommender
- Databricks SQL sample queries
- Dashboard layout and performance guidance
- Interactivity matrix for filters, parameters, cross-filters, and drill behaviors

---

## 2. Visual Recommendations

### 2.1 Revenue-to-Cash Executive Scorecard – Overall Health Score
- **Data Element:** Overall Revenue-to-Cash Health Score
- **SQL Query:**
  ```sql
  SELECT 
      d.period_start_date,
      b.region_name,
      b.branch_name,
      ROUND(AVG(h.overall_health_score), 1) AS overall_health_score
  FROM gold.fct_revenue_to_cash_health_score h
  JOIN gold.dim_branch b
    ON h.branch_id = b.branch_id
  JOIN gold.dim_date d
    ON h.date_id = d.date_id
  WHERE d.period_type IN ('DAY', 'WEEK')
  GROUP BY d.period_start_date, b.region_name, b.branch_name
  ORDER BY d.period_start_date DESC;
  ```
- **Recommended LVDash Widget:** Counter / KPI Card
- **Data Fields:** `period_start_date`, `region_name`, `branch_name`, `overall_health_score`
- **Calculations:** `AVG(overall_health_score)`
- **Interactivity:** Region filter, branch filter, date range filter, drill from network to region to branch
- **Justification:** The health score is a top-level executive metric best displayed as a KPI card for fast health assessment and target comparison.
- **Optimization Tips:** Materialize health score daily/weekly aggregates; cache latest-period dataset; partition by `period_type` and date; Z-order by `branch_id`, `date_id`.

### 2.2 Revenue-to-Cash Executive Scorecard – Domain Scores
- **Data Element:** Revenue, AR, Cash App, and Credit Risk domain scores
- **SQL Query:**
  ```sql
  SELECT 
      b.region_name,
      b.branch_name,
      ROUND(AVG(h.revenue_score), 1) AS revenue_score,
      ROUND(AVG(h.ar_score), 1) AS ar_score,
      ROUND(AVG(h.cash_app_score), 1) AS cash_app_score,
      ROUND(AVG(h.credit_risk_score), 1) AS credit_risk_score
  FROM gold.fct_revenue_to_cash_health_score h
  JOIN gold.dim_branch b
    ON h.branch_id = b.branch_id
  JOIN gold.dim_date d
    ON h.date_id = d.date_id
  WHERE d.calendar_date BETWEEN :start_date AND :end_date
  GROUP BY b.region_name, b.branch_name;
  ```
- **Recommended LVDash Widget:** Counter row or small multiples KPI cards
- **Data Fields:** `region_name`, `branch_name`, `revenue_score`, `ar_score`, `cash_app_score`, `credit_risk_score`
- **Calculations:** Average domain scores over selected period
- **Interactivity:** Region, branch, date filters; click branch to cross-filter other widgets
- **Justification:** Separate KPI cards help executives quickly locate the weakest domain driving the overall score.
- **Optimization Tips:** Reuse same score dataset as overall health KPI; avoid separate queries per domain by returning all score columns in one dataset.

### 2.3 Executive Trend – Rental Revenue and DSO Trend
- **Data Element:** Revenue and DSO trend over time
- **SQL Query:**
  ```sql
  WITH revenue AS (
      SELECT d.calendar_date,
             SUM(r.rental_revenue) AS rental_revenue
      FROM gold.fct_rental_revenue r
      JOIN gold.dim_date d ON r.date_id = d.date_id
      WHERE d.calendar_date BETWEEN :start_date AND :end_date
      GROUP BY d.calendar_date
  ),
  ar AS (
      SELECT d.calendar_date,
             ROUND(SUM(a.ar_outstanding) / NULLIF(SUM(a.credit_sales_period), 0) * MAX(d.days_in_period), 1) AS dso
      FROM gold.fct_ar_aging a
      JOIN gold.dim_date d ON a.date_id = d.date_id
      WHERE d.calendar_date BETWEEN :start_date AND :end_date
      GROUP BY d.calendar_date
  )
  SELECT r.calendar_date, r.rental_revenue, a.dso
  FROM revenue r
  JOIN ar a ON r.calendar_date = a.calendar_date
  ORDER BY r.calendar_date;
  ```
- **Recommended LVDash Widget:** Combo Chart (bar + line)
- **Data Fields:** `calendar_date`, `rental_revenue`, `dso`
- **Calculations:** Sum of revenue, DSO formula
- **Interactivity:** Date filter, region/branch cross-filtering, drill up/down day-week-month
- **Justification:** A combo chart clearly shows revenue magnitude and DSO trend together while preserving separate scales.
- **Optimization Tips:** Build a reusable daily aggregate MV; use result caching; ensure date dimension supports hierarchy rollup.

### 2.4 AR Risk Leaderboard – AR 90+ by Branch
- **Data Element:** AR 90+ percentage by branch
- **SQL Query:**
  ```sql
  SELECT 
      b.region_name,
      b.branch_name,
      ROUND(SUM(CASE WHEN a.aging_bucket = '90+' THEN a.ar_outstanding ELSE 0 END)
            / NULLIF(SUM(a.ar_outstanding), 0) * 100, 1) AS ar_90_plus_pct,
      SUM(a.ar_outstanding) AS total_ar_outstanding
  FROM gold.fct_ar_aging a
  JOIN gold.dim_branch b
    ON a.branch_id = b.branch_id
  JOIN gold.dim_date d
    ON a.date_id = d.date_id
  WHERE d.snapshot_date = :snapshot_date
  GROUP BY b.region_name, b.branch_name
  ORDER BY ar_90_plus_pct DESC;
  ```
- **Recommended LVDash Widget:** Horizontal Bar Chart
- **Data Fields:** `region_name`, `branch_name`, `ar_90_plus_pct`, `total_ar_outstanding`
- **Calculations:** AR 90+ %
- **Interactivity:** Region filter, branch click cross-filter, drill-through to branch/customer invoice detail
- **Justification:** Ranking branches by AR risk is best shown in a horizontal bar chart for clear leaderboard comparison.
- **Optimization Tips:** Precompute latest snapshot aggregates; partition fact table by snapshot date; Z-order by `branch_id`, `aging_bucket`.

### 2.5 Revenue & Billing Performance – Revenue KPI Strip
- **Data Element:** Rental Revenue, Gross Billings, Net Billings, ADR, Credit Memo Rate
- **SQL Query:**
  ```sql
  SELECT
      SUM(r.rental_revenue) AS rental_revenue,
      SUM(r.gross_billings) AS gross_billings,
      SUM(r.net_billings) AS net_billings,
      ROUND(SUM(r.rental_revenue) / NULLIF(SUM(r.billed_rental_days), 0), 2) AS adr,
      ROUND(SUM(ABS(r.credit_memo_amount)) / NULLIF(SUM(r.gross_billings), 0) * 100, 2) AS credit_memo_rate
  FROM gold.fct_rental_revenue r
  JOIN gold.dim_date d
    ON r.date_id = d.date_id
  WHERE d.calendar_date BETWEEN :start_date AND :end_date;
  ```
- **Recommended LVDash Widget:** Counter / KPI Cards
- **Data Fields:** `rental_revenue`, `gross_billings`, `net_billings`, `adr`, `credit_memo_rate`
- **Calculations:** Sum metrics, ADR, credit memo rate
- **Interactivity:** Region, branch, sales rep, equipment class, period filters
- **Justification:** These are high-value summary metrics that belong in a compact KPI strip at the top of the billing page.
- **Optimization Tips:** Serve from a period-level summary dataset reused across multiple KPI cards; cache query results for common filter combinations.

### 2.6 Revenue Trend by Branch
- **Data Element:** Revenue trend by branch over time
- **SQL Query:**
  ```sql
  SELECT 
      d.calendar_date,
      b.branch_name,
      SUM(r.rental_revenue) AS rental_revenue
  FROM gold.fct_rental_revenue r
  JOIN gold.dim_branch b
    ON r.branch_id = b.branch_id
  JOIN gold.dim_date d
    ON r.date_id = d.date_id
  WHERE d.calendar_date BETWEEN :start_date AND :end_date
  GROUP BY d.calendar_date, b.branch_name
  ORDER BY d.calendar_date, b.branch_name;
  ```
- **Recommended LVDash Widget:** Line Chart
- **Data Fields:** `calendar_date`, `branch_name`, `rental_revenue`
- **Calculations:** `SUM(rental_revenue)` by date and branch
- **Interactivity:** Region/branch filters, legend-based series selection, drill up/down by time grain
- **Justification:** Trend comparison across branches is best communicated with a line chart to reveal seasonality and divergence.
- **Optimization Tips:** Limit number of visible series using top-N parameter or branch filter to avoid overcrowding; use pre-aggregated daily branch summaries.

### 2.7 Equipment Class Mix
- **Data Element:** Revenue share by equipment class
- **SQL Query:**
  ```sql
  SELECT 
      e.equipment_class_name,
      SUM(r.rental_revenue) AS rental_revenue
  FROM gold.fct_rental_revenue r
  JOIN gold.dim_equipment_class e
    ON r.equipment_class_id = e.equipment_class_id
  JOIN gold.dim_date d
    ON r.date_id = d.date_id
  WHERE d.calendar_date BETWEEN :start_date AND :end_date
  GROUP BY e.equipment_class_name
  ORDER BY rental_revenue DESC;
  ```
- **Recommended LVDash Widget:** Donut Chart or Bar Chart
- **Data Fields:** `equipment_class_name`, `rental_revenue`
- **Calculations:** Revenue sum by equipment class
- **Interactivity:** Region/branch/date filters; click class to filter detail tables
- **Justification:** For a limited number of classes, a donut chart shows composition; if classes are many, use a bar chart.
- **Optimization Tips:** Cap to top-N classes plus “Other” to prevent clutter; aggregate in a materialized view if reused across dashboards.

### 2.8 Sales Rep Performance Detail
- **Data Element:** Revenue by sales rep and contract
- **SQL Query:**
  ```sql
  SELECT 
      s.sales_rep_name,
      b.branch_name,
      r.contract_id,
      SUM(r.rental_revenue) AS rental_revenue,
      SUM(r.net_billings) AS net_billings,
      COUNT(DISTINCT r.invoice_id) AS invoice_count
  FROM gold.fct_rental_revenue r
  JOIN gold.dim_sales_rep s
    ON r.sales_rep_id = s.sales_rep_id
  JOIN gold.dim_branch b
    ON r.branch_id = b.branch_id
  JOIN gold.dim_date d
    ON r.date_id = d.date_id
  WHERE d.calendar_date BETWEEN :start_date AND :end_date
  GROUP BY s.sales_rep_name, b.branch_name, r.contract_id
  ORDER BY rental_revenue DESC;
  ```
- **Recommended LVDash Widget:** Table
- **Data Fields:** `sales_rep_name`, `branch_name`, `contract_id`, `rental_revenue`, `net_billings`, `invoice_count`
- **Calculations:** Revenue sum, net billings sum, distinct invoice count
- **Interactivity:** Drill from region to branch to sales rep to contract; sortable columns; drill-through to invoice detail
- **Justification:** Users need granular operational detail, ranking, and drill-through, which are best supported by a table.
- **Optimization Tips:** Paginate large tables; avoid exposing unnecessary columns; cluster source by branch and sales rep for selective scans.

### 2.9 AR Aging & Collections – Total AR Outstanding
- **Data Element:** Total AR Outstanding
- **SQL Query:**
  ```sql
  SELECT 
      SUM(a.ar_outstanding) AS total_ar_outstanding
  FROM gold.fct_ar_aging a
  JOIN gold.dim_date d
    ON a.date_id = d.date_id
  WHERE d.snapshot_date = :snapshot_date;
  ```
- **Recommended LVDash Widget:** Counter
- **Data Fields:** `total_ar_outstanding`
- **Calculations:** `SUM(ar_outstanding)`
- **Interactivity:** Region, branch, collector, snapshot date filters
- **Justification:** A single monetary exposure metric should be immediately visible as a KPI card.
- **Optimization Tips:** Latest snapshot tables or snapshot MVs improve performance; partition by `snapshot_date`.

### 2.10 AR Aging Distribution by Bucket and Branch
- **Data Element:** AR outstanding by aging bucket across branches
- **SQL Query:**
  ```sql
  SELECT 
      b.branch_name,
      a.aging_bucket,
      SUM(a.ar_outstanding) AS ar_amount
  FROM gold.fct_ar_aging a
  JOIN gold.dim_branch b
    ON a.branch_id = b.branch_id
  JOIN gold.dim_date d
    ON a.date_id = d.date_id
  WHERE d.snapshot_date = :snapshot_date
  GROUP BY b.branch_name, a.aging_bucket
  ORDER BY b.branch_name, a.aging_bucket;
  ```
- **Recommended LVDash Widget:** Stacked Bar Chart
- **Data Fields:** `branch_name`, `aging_bucket`, `ar_amount`
- **Calculations:** `SUM(ar_outstanding)` by branch and bucket
- **Interactivity:** Region filter, branch click cross-filter, drill-through to invoice list
- **Justification:** Stacked bars provide clear comparison of both total AR and aging composition by branch.
- **Optimization Tips:** Ensure aging bucket is a compact categorical field in curated gold fact; pre-aggregate daily snapshots.

### 2.11 DSO Trend
- **Data Element:** DSO over time
- **SQL Query:**
  ```sql
  SELECT 
      d.calendar_date,
      ROUND(SUM(a.ar_outstanding) / NULLIF(SUM(a.credit_sales_period), 0) * MAX(d.days_in_period), 1) AS dso
  FROM gold.fct_ar_aging a
  JOIN gold.dim_date d
    ON a.date_id = d.date_id
  WHERE d.calendar_date BETWEEN :start_date AND :end_date
  GROUP BY d.calendar_date
  ORDER BY d.calendar_date;
  ```
- **Recommended LVDash Widget:** Line Chart
- **Data Fields:** `calendar_date`, `dso`
- **Calculations:** DSO formula
- **Interactivity:** Date range, region/branch filters, drill up/down by time grain
- **Justification:** DSO is a trend metric and is most understandable in a line chart.
- **Optimization Tips:** Reuse same daily AR trend dataset as executive page to avoid duplicate SQL work.

### 2.12 Collector Worklist / Invoice Detail
- **Data Element:** Invoice-level overdue detail by collector and customer
- **SQL Query:**
  ```sql
  SELECT 
      c.collector_name,
      cu.customer_name,
      a.invoice_id,
      a.due_date,
      a.days_past_due,
      a.aging_bucket,
      a.ar_outstanding
  FROM gold.fct_ar_aging a
  JOIN gold.dim_collector c
    ON a.collector_id = c.collector_id
  JOIN gold.dim_customer cu
    ON a.customer_id = cu.customer_id
  JOIN gold.dim_date d
    ON a.date_id = d.date_id
  WHERE d.snapshot_date = :snapshot_date
  ORDER BY a.days_past_due DESC, a.ar_outstanding DESC;
  ```
- **Recommended LVDash Widget:** Table
- **Data Fields:** `collector_name`, `customer_name`, `invoice_id`, `due_date`, `days_past_due`, `aging_bucket`, `ar_outstanding`
- **Calculations:** None beyond field display and sorting
- **Interactivity:** Drill from branch -> collector -> customer -> invoice; drill-through from aging chart
- **Justification:** Collections teams need actionable invoice detail rather than summarized charts.
- **Optimization Tips:** Use row limits or pagination; filter to overdue invoices only by default for performance and usability.

### 2.13 Cash Application – Cash Collected / CEI / Avg Days to Pay
- **Data Element:** Cash collection KPIs
- **SQL Query:**
  ```sql
  SELECT 
      SUM(c.cash_collected_amount) AS cash_collected,
      ROUND(
          (SUM(c.beginning_ar) + SUM(c.credit_sales) - SUM(c.ending_total_ar))
          / NULLIF((SUM(c.beginning_ar) + SUM(c.credit_sales) - SUM(c.ending_current_ar)), 0) * 100, 1
      ) AS cei,
      ROUND(AVG(c.days_to_pay), 1) AS avg_days_to_pay,
      ROUND(SUM(CASE WHEN c.paid_on_time_flag = 1 THEN 1 ELSE 0 END) / NULLIF(COUNT(*), 0) * 100, 1) AS invoices_paid_on_time_pct
  FROM gold.fct_cash_application c
  JOIN gold.dim_date d
    ON c.date_id = d.date_id
  WHERE d.calendar_date BETWEEN :start_date AND :end_date;
  ```
- **Recommended LVDash Widget:** Counter / KPI Cards
- **Data Fields:** `cash_collected`, `cei`, `avg_days_to_pay`, `invoices_paid_on_time_pct`
- **Calculations:** CEI, average days to pay, paid-on-time percentage
- **Interactivity:** Region, branch, collector, payment method, date filters
- **Justification:** These collection efficiency metrics belong in a KPI strip for quick performance review.
- **Optimization Tips:** Compute CEI in curated gold layer to ensure period alignment and avoid repeated complex calculations in dashboard SQL.

### 2.14 Collector Leaderboard
- **Data Element:** Collector-level performance comparison
- **SQL Query:**
  ```sql
  SELECT 
      co.collector_name,
      b.branch_name,
      SUM(c.cash_collected_amount) AS cash_collected,
      ROUND(AVG(c.days_to_pay), 1) AS avg_days_to_pay,
      ROUND(SUM(c.unapplied_cash_amount) / NULLIF(SUM(c.cash_collected_amount), 0) * 100, 1) AS unapplied_cash_pct
  FROM gold.fct_cash_application c
  JOIN gold.dim_collector co
    ON c.collector_id = co.collector_id
  JOIN gold.dim_branch b
    ON c.branch_id = b.branch_id
  JOIN gold.dim_date d
    ON c.date_id = d.date_id
  WHERE d.calendar_date BETWEEN :start_date AND :end_date
  GROUP BY co.collector_name, b.branch_name
  ORDER BY cash_collected DESC;
  ```
- **Recommended LVDash Widget:** Horizontal Bar Chart
- **Data Fields:** `collector_name`, `branch_name`, `cash_collected`, `avg_days_to_pay`, `unapplied_cash_pct`
- **Calculations:** Cash collected, avg days to pay, unapplied cash %
- **Interactivity:** Region/branch filters, click collector for detail table cross-filter
- **Justification:** Ranking collectors is most intuitive in a leaderboard-style horizontal bar chart.
- **Optimization Tips:** Limit view to top/bottom N collectors with parameter; reuse grouped collector dataset in multiple widgets.

### 2.15 Unapplied Cash Detail
- **Data Element:** Unapplied cash details by receipt and customer
- **SQL Query:**
  ```sql
  SELECT 
      c.receipt_id,
      cu.customer_name,
      b.branch_name,
      c.payment_method,
      c.payment_date,
      c.unapplied_cash_amount,
      c.match_status
  FROM gold.fct_cash_application c
  JOIN gold.dim_customer cu
    ON c.customer_id = cu.customer_id
  JOIN gold.dim_branch b
    ON c.branch_id = b.branch_id
  JOIN gold.dim_date d
    ON c.date_id = d.date_id
  WHERE d.calendar_date BETWEEN :start_date AND :end_date
    AND c.match_status = 'UNAPPLIED'
  ORDER BY c.unapplied_cash_amount DESC;
  ```
- **Recommended LVDash Widget:** Table
- **Data Fields:** `receipt_id`, `customer_name`, `branch_name`, `payment_method`, `payment_date`, `unapplied_cash_amount`, `match_status`
- **Calculations:** None
- **Interactivity:** Drill-through from collector KPI or branch view; branch, collector, date filters
- **Justification:** This is an exception-management workflow best handled as a sortable detail grid.
- **Optimization Tips:** Predicate pushdown via `match_status = 'UNAPPLIED'`; cluster on `match_status`, `payment_date` if table is large.

### 2.16 Customer Credit Risk – Over Limit and Suspended Exposure KPIs
- **Data Element:** Customers Over Limit, Over-Limit Amount, Suspended Customers with Open Exposure
- **SQL Query:**
  ```sql
  SELECT 
      COUNT(DISTINCT CASE WHEN r.outstanding_ar > r.credit_limit AND r.credit_limit IS NOT NULL THEN r.customer_id END) AS customers_over_limit,
      SUM(CASE WHEN r.outstanding_ar > r.credit_limit AND r.credit_limit IS NOT NULL THEN r.outstanding_ar - r.credit_limit ELSE 0 END) AS over_limit_amount,
      COUNT(DISTINCT CASE WHEN r.customer_status = 'Suspended' AND (r.outstanding_ar > 0 OR r.open_contract_count > 0) THEN r.customer_id END) AS suspended_with_exposure
  FROM gold.fct_credit_risk r
  JOIN gold.dim_date d
    ON r.date_id = d.date_id
  WHERE d.snapshot_date = :snapshot_date;
  ```
- **Recommended LVDash Widget:** Counter / KPI Cards
- **Data Fields:** `customers_over_limit`, `over_limit_amount`, `suspended_with_exposure`
- **Calculations:** Conditional distinct counts and sums
- **Interactivity:** Region, branch, customer status filters
- **Justification:** Risk summary metrics should be highly visible to credit leadership as KPI cards.
- **Optimization Tips:** Maintain snapshot gold fact for customer credit exposure; partition by snapshot date.

### 2.17 Credit Utilization Heatmap by Branch
- **Data Element:** Average credit utilization percentage by branch
- **SQL Query:**
  ```sql
  SELECT 
      b.region_name,
      b.branch_name,
      ROUND(AVG(CASE WHEN r.credit_limit > 0 THEN r.outstanding_ar / r.credit_limit * 100 END), 1) AS credit_utilization_pct
  FROM gold.fct_credit_risk r
  JOIN gold.dim_branch b
    ON r.branch_id = b.branch_id
  JOIN gold.dim_date d
    ON r.date_id = d.date_id
  WHERE d.snapshot_date = :snapshot_date
  GROUP BY b.region_name, b.branch_name
  ORDER BY b.region_name, b.branch_name;
  ```
- **Recommended LVDash Widget:** Pivot Table with Conditional Formatting
- **Data Fields:** `region_name`, `branch_name`, `credit_utilization_pct`
- **Calculations:** Avg credit utilization %
- **Interactivity:** Region filter, branch click to customer detail table
- **Justification:** A heatmap-style pivot table is effective for quickly spotting concentration of risk across branches.
- **Optimization Tips:** Compute utilization in gold layer; avoid division in row-heavy dashboard query where possible.

### 2.18 Customer Exposure Detail
- **Data Element:** Customer-level exposure and risk detail
- **SQL Query:**
  ```sql
  SELECT 
      cu.customer_name,
      b.branch_name,
      r.customer_status,
      r.credit_terms,
      r.credit_limit,
      r.outstanding_ar,
      r.ar_90_plus_amount,
      r.open_contract_count,
      ROUND(CASE WHEN r.credit_limit > 0 THEN r.outstanding_ar / r.credit_limit * 100 END, 1) AS credit_utilization_pct
  FROM gold.fct_credit_risk r
  JOIN gold.dim_customer cu
    ON r.customer_id = cu.customer_id
  JOIN gold.dim_branch b
    ON r.branch_id = b.branch_id
  JOIN gold.dim_date d
    ON r.date_id = d.date_id
  WHERE d.snapshot_date = :snapshot_date
  ORDER BY r.outstanding_ar DESC;
  ```
- **Recommended LVDash Widget:** Table
- **Data Fields:** `customer_name`, `branch_name`, `customer_status`, `credit_terms`, `credit_limit`, `outstanding_ar`, `ar_90_plus_amount`, `open_contract_count`, `credit_utilization_pct`
- **Calculations:** Credit utilization %
- **Interactivity:** Drill from branch -> customer; drill-through to invoice/contract detail; customer status filter
- **Justification:** Credit analysts need detailed sortable exposure data at the customer level.
- **Optimization Tips:** Mask sensitive columns as needed for role-based access; paginate large result sets.

### 2.19 Data Quality / Operational Monitoring
- **Data Element:** Pipeline audit and data quality exceptions relevant to dashboard trust
- **SQL Query:**
  ```sql
  SELECT 
      a.pipeline_name,
      a.target_table,
      a.processing_status,
      a.records_read_count,
      a.records_written_count,
      a.records_rejected_count,
      a.process_end_timestamp
  FROM silver.si_process_audit a
  ORDER BY a.process_end_timestamp DESC;
  ```
- **Recommended LVDash Widget:** Table
- **Data Fields:** `pipeline_name`, `target_table`, `processing_status`, `records_read_count`, `records_written_count`, `records_rejected_count`, `process_end_timestamp`
- **Calculations:** None
- **Interactivity:** Optional filter by target table or status
- **Justification:** Helps support teams confirm data freshness and pipeline reliability for the business dashboards.
- **Optimization Tips:** Keep as a small support tab; separate from business-facing main tab to avoid clutter.

---

## 3. Overall Dashboard Design

### 3.1 Layout Suggestions
Use a multi-tab Lakeview dashboard with a clear business flow.

| Tab | Purpose | Suggested Layout |
|---|---|---|
| Executive Scorecard | Enterprise health summary | Top KPI strip, middle revenue/DSO combo trend, bottom AR risk leaderboard and drivers table |
| Revenue & Billing | Revenue quality and billing analysis | KPI strip, trend chart, equipment mix chart, bottom sales rep/contract detail table |
| AR Aging & Collections | Receivables risk and collection focus | KPI cards, stacked aging chart, DSO trend, bottom collector/customer/invoice table |
| Cash Application | Collector performance and payment application quality | KPI strip, collector leaderboard, unapplied cash detail table |
| Credit Risk | Customer exposure and limit monitoring | KPI cards, branch heatmap, customer exposure detail table |
| Data Operations (Optional) | Freshness, rejects, audit support | Audit table and error summary widgets |

### 3.2 Grid Arrangement
- Use a 12-column canvas.
- KPI cards should span 2–3 columns each in the top row.
- Trend charts should span 6–12 columns depending on complexity.
- Leaderboards fit well in 4–6 columns.
- Detail tables should typically use full-width rows.
- Keep slicers pinned at top or left depending on tab complexity.

### 3.3 Query Optimization
- Build curated Gold tables or materialized views for each reporting domain instead of hitting Silver directly for every widget.
- Precompute aging buckets, DSO inputs, CEI components, and health score components in Gold.
- Reuse datasets across widgets whenever metrics share the same grain.
- Enable Lakeview dataset query result caching for repeated interactions.
- Partition snapshot-based facts by `snapshot_date` and transactional facts by `date_id` or event date.
- Z-order large Delta tables by common filters such as `branch_id`, `region_id`, `customer_id`, `collector_id`, and `date_id`.
- Use appropriately sized SQL warehouses; prefer serverless or pro warehouse with auto-stop and concurrency aligned to usage.
- Avoid oversized “one dataset for everything” designs; create focused reusable datasets by business grain.
- Consider materialized views for daily branch-level summaries and customer exposure snapshots.

### 3.4 Color Scheme
- Executive KPI positive / healthy: `#2E7D32`
- Warning / risk: `#F9A825`
- Critical / delinquent: `#C62828`
- Neutral / informational: `#1565C0`
- Background / supporting gray: `#ECEFF1`

Use consistent semantic coloring:
- Healthy / within target = green
- Watchlist / approaching threshold = amber
- Critical / overdue / over-limit = red

### 3.5 Typography
- Font: Inter, Montserrat, or Databricks default sans-serif for consistency
- KPI labels: 12–14 px, semi-bold
- KPI values: 24–32 px, bold
- Chart titles: 13–15 px, semi-bold
- Table text: 11–12 px
- Keep decimal precision controlled and avoid overly dense number formatting

### 3.6 Interactive Elements
- Global filters: Date range, period type, region, branch
- Tab-specific filters: Sales rep, collector, equipment class, aging bucket, customer status, match status
- Drill-down hierarchy: Network -> Region -> Branch -> Sales Rep / Collector / Customer -> Contract / Invoice
- Drill-through paths:
  - Health score -> branch risk drivers
  - Revenue total -> invoice/contract detail
  - Aging bucket -> invoice list
  - Collector total -> payment and application detail
  - Customer exposure -> invoice and contract detail
- Drill up/down across day, week, month, quarter
- Cross-filtering between leaderboard charts and detail tables
- Parameterized top-N controls for branches, customers, and collectors

---

## 4. Interactivity Features Table

| Interactivity Feature | Widget(s) / Area | Behavior | Business Value |
|---|---|---|---|
| Global Date Filter | All tabs | Filters widgets to selected date range or snapshot | Aligns all KPIs and trends to same reporting period |
| Period Type Parameter | Executive, Revenue, AR tabs | Switches daily/weekly/monthly view | Supports rollup comparison |
| Region Filter | All business tabs | Filters to region and cascades available branches | Supports hierarchical management review |
| Branch Filter | All business tabs | Narrows branch-specific metrics and details | Enables branch manager consumption |
| Sales Rep Filter | Revenue tab | Filters rep-level metrics and contract detail | Supports performance coaching |
| Collector Filter | AR and Cash Application tabs | Filters collector KPIs and worklists | Supports collections management |
| Equipment Class Filter | Revenue tab | Filters equipment mix and revenue | Helps identify class-level shortfalls |
| Customer Status Filter | Credit Risk tab | Filters active/suspended customers | Focuses on risk segments |
| Aging Bucket Cross-Filter | AR Aging chart -> Invoice table | Clicking a bucket filters invoice-level detail | Accelerates collector action |
| Collector Bar Click | Collector leaderboard -> detail table | Cross-filters to selected collector's payments | Simplifies performance review |
| Branch Heatmap Click | Credit risk heatmap -> customer table | Shows branch-specific customers | Speeds risk investigation |
| Drill-Through | KPI cards/charts -> detail tables | Opens underlying record-level data | Supports reconciliation and root-cause analysis |
| Drill Up / Drill Down | Trend charts | Move between day/week/month/quarter | Supports time-grain analysis |
| Top-N Parameter | Leaderboards | Displays top/bottom N branches/collectors/customers | Controls clutter and highlights outliers |

---

## 5. Recommended Data Model for Lakeview Dataset Reuse

| Dataset Name | Grain | Used By Widgets | Notes |
|---|---|---|---|
| `ds_exec_health_score` | Branch x period | Overall health KPI, domain score KPIs, branch comparisons | Should include all score components in one dataset |
| `ds_exec_trend` | Date x branch/region | Revenue and DSO combo chart | Reusable across executive and AR/revenue tabs |
| `ds_revenue_summary` | Date x branch x sales rep x equipment class | Revenue KPIs, trend, mix chart | Prefer MV for period-level summaries |
| `ds_ar_snapshot` | Snapshot date x branch x customer x invoice | AR KPIs, aging stacked chart, invoice detail | Partition by snapshot date |
| `ds_cash_app_summary` | Date x branch x collector | Cash KPIs, collector leaderboard | Include CEI-ready fields |
| `ds_cash_unapplied_detail` | Receipt-level | Unapplied cash table | Exception-only dataset |
| `ds_credit_risk_snapshot` | Snapshot date x branch x customer | Credit KPIs, branch heatmap, customer detail | Best served from daily snapshot table |
| `ds_pipeline_monitor` | Process run | Audit support tab | Separate support dataset |

---

## 6. Potential Pitfalls and Mitigation

| Potential Pitfall | Impact | Mitigation |
|---|---|---|
| Using Silver tables directly for all widgets | Slow queries and repeated business logic | Build curated Gold facts or materialized views |
| Recomputing DSO/CEI in every widget | Inconsistent logic and poor performance | Centralize calculations in reusable datasets |
| Large invoice/customer detail tables without filters | Slow rendering and poor UX | Apply default date/snapshot filter, pagination, and drill-through entry points |
| Many branch series on one line chart | Visual clutter | Use region filter, top-N parameter, or small multiples |
| Heavy joins across branch/customer/date dimensions per widget | Warehouse load and latency | Pre-join curated reporting datasets where practical |
| Large shuffles on aging and cash application facts | Expensive execution | Partition by date/snapshot and Z-order on branch/customer/collector |
| Division-by-zero in DSO, CEI, utilization | Incorrect metrics or runtime errors | Use `NULLIF` and explicit validation logic |
| Mixed role visibility in one dashboard | Security risk | Apply row-level security and persona-aware filters/views |
| Reusing one oversized dataset for all tabs | Memory overhead and slow refresh | Create targeted datasets by domain and grain |
| Unoptimized SQL warehouse sizing | Poor concurrency or wasted cost | Match warehouse size to user concurrency and refresh frequency |

---

## 7. Lakeview-Specific Performance and Design Tips

- Prefer a small number of reusable datasets per tab rather than separate SQL for every widget.
- Use query result caching for high-traffic KPI datasets.
- Keep dashboard-level filters consistent in naming and behavior across tabs.
- Use calculated fields in curated datasets only for lightweight presentation logic; keep heavy business formulas upstream.
- Configure warehouse auto-stop and consider scheduled cache warming before business hours.
- For snapshot facts, expose a default “latest snapshot” parameter to reduce scanning historical partitions unnecessarily.
- Use drill-through to detail tables instead of placing all detailed data on the landing page.
- Ensure conditional formatting thresholds are business-approved and consistent across pages.

---

## Output URL (Clickable Hyperlinks)
[Databricks_Dashboard_Visuals_Recommender_2.md](https://github.com/DIAscendion/Databricks_Dashboard/blob/main/DI_Databricks_Dashboard_Visuals_Recommender/Databricks_Dashboard_Visuals_Recommender_2.md)

Print the URL : "https://github.com/DIAscendion/Databricks_Dashboard/tree/main/DI_Databricks_Dashboard_Visuals_Recommender"

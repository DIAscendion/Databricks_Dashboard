_____________________________________________
## *Author*: AAVA
## *Created on*:   
## *Description*: Databricks Lakeview Dashboard visuals recommender for Rental Revenue-to-Cash executive, revenue, AR, cash application, and credit risk reporting.
## *Version*: 1
## *Updated on*: 
_____________________________________________

# Databricks Lakeview Dashboard Visuals Recommender

Print the URL : "https://github.com/DIAscendion/Databricks_Dashboard/tree/main/DI_Databricks_Dashboard_Visuals_Recommender"

## 1. Input Summary
This recommender translates the Rental Revenue-to-Cash analytical reporting requirements and the Silver physical data model into Databricks SQL datasets, reusable semantic views, and Lakeview Dashboard visuals. The goal is to support executive scorecards, billing analysis, AR aging, cash application monitoring, and customer credit risk management across network, region, branch, sales rep, collector, customer, contract, and invoice levels.

## 2. Source Analysis

### 2.1 Primary Source Tables from Silver Layer
| Source Table | Purpose | Key Fields |
|---|---|---|
| `silver.si_rental_contracts` | Contract and branch/sales context for rental activity | `contract_id`, `customer_id`, `branch_code`, `sales_rep_id`, `equipment_class`, `contract_start_date`, `contract_end_date`, `contract_status`, `daily_rate` |
| `silver.si_invoices` | Invoice, billing, and AR basis | `invoice_id`, `contract_id`, `customer_id`, `invoice_date`, `due_date`, `invoice_amount`, `tax_amount`, `invoice_type`, `currency`, `aging_bucket`, `days_past_due` |
| `silver.si_cash_receipts` | Receipt and payment application basis | `receipt_id`, `invoice_id`, `customer_id`, `receipt_date`, `payment_amount`, `payment_method`, `match_status`, `unapplied_payment_amount` |
| `silver.si_customer_master` | Customer credit and status context | `customer_id`, `customer_name`, `credit_terms`, `credit_limit`, `customer_since`, `customer_status`, `over_limit_indicator` |
| `silver.si_branch_employee` | Branch, region, sales rep, and collector alignment | `employee_id`, `employee_name`, `role`, `branch_code`, `region`, `collector_id`, `effective_date` |
| `silver.si_process_audit` | Operational observability for data quality and freshness | `pipeline_name`, `target_table`, `process_start_timestamp`, `process_end_timestamp`, `processing_status` |
| `silver.si_data_quality_errors` | Rejected-record and validation diagnostics | `source_table`, `validation_rule_name`, `error_description`, `error_severity`, `detected_timestamp` |

### 2.2 Recommended Gold / Semantic Views for Dashboard Consumption
| Recommended Dataset / View | Purpose | Grain |
|---|---|---|
| `gold.vw_exec_health_scorecard` | Consolidated executive scorecard by branch/date/period | One row per branch, date, period type |
| `gold.vw_rental_revenue_billing` | Revenue and billing metrics by branch/sales rep/equipment class | One row per branch, period, sales rep, equipment class |
| `gold.vw_ar_aging_snapshot` | AR snapshot by branch/customer/invoice/snapshot date | One row per invoice per snapshot date |
| `gold.vw_cash_application_perf` | Collector and payment application performance | One row per receipt or collector-period |
| `gold.vw_customer_credit_risk` | Customer-level risk and exposure | One row per customer per snapshot date |
| `gold.vw_dashboard_filter_dimensions` | Shared dimensions for Lakeview filters | Dimension rows for branch, region, equipment class, status, date |

### 2.3 Required Joins
| Left Table | Join Key | Right Table | Purpose |
|---|---|---|---|
| `silver.si_invoices` | `contract_id` | `silver.si_rental_contracts` | Connect invoices to contract, branch, and equipment context |
| `silver.si_invoices` | `customer_id` | `silver.si_customer_master` | Add customer status, credit terms, and limit context |
| `silver.si_cash_receipts` | `invoice_id` | `silver.si_invoices` | Measure payment application against billed invoices |
| `silver.si_rental_contracts` | `sales_rep_id = employee_id` | `silver.si_branch_employee` | Derive sales rep and region attributes |
| `silver.si_rental_contracts` | `branch_code` | `silver.si_branch_employee` | Support branch-region hierarchy |
| `silver.si_customer_master` | `customer_id` | `gold.vw_ar_aging_snapshot` | Add risk context to AR snapshots |

### 2.4 Key Filters and Business Rules
- Include active organization hierarchy only for selected analysis date.
- Exclude cancelled or zero-duration contracts from ADR calculations.
- Ensure standard invoice amounts are non-negative and credit memos are negative.
- Aging buckets must remain fixed as `0-30`, `31-60`, `61-90`, `90+`.
- Separate unapplied cash from applied cash; do not silently suppress it.
- Handle null or zero denominators in DSO, CEI, credit utilization, and percentage-based calculations.

## 3. Visual Recommendations

### 3.1 Revenue-to-Cash Executive Scorecard
- **Data Element:** Overall and domain health monitoring by branch and region
- **SQL Query:**
```sql
WITH revenue_base AS (
  SELECT
      be.region,
      rc.branch_code,
      DATE(i.invoice_date) AS metric_date,
      SUM(CASE WHEN i.invoice_type <> 'CREDIT_MEMO' THEN i.invoice_amount ELSE 0 END) AS gross_billings,
      SUM(CASE WHEN i.invoice_type = 'CREDIT_MEMO' THEN ABS(i.invoice_amount) ELSE 0 END) AS credit_memos,
      SUM(i.invoice_amount) AS net_billings
  FROM silver.si_invoices i
  LEFT JOIN silver.si_rental_contracts rc
    ON i.contract_id = rc.contract_id
  LEFT JOIN silver.si_branch_employee be
    ON rc.branch_code = be.branch_code
  GROUP BY be.region, rc.branch_code, DATE(i.invoice_date)
),
ar_base AS (
  SELECT
      be.region,
      rc.branch_code,
      CURRENT_DATE() AS snapshot_date,
      SUM(CASE WHEN COALESCE(i.days_past_due, 0) > 0 THEN i.invoice_amount ELSE 0 END) AS total_ar_outstanding,
      SUM(CASE WHEN COALESCE(i.days_past_due, 0) > 90 THEN i.invoice_amount ELSE 0 END) AS ar_90_plus
  FROM silver.si_invoices i
  LEFT JOIN silver.si_rental_contracts rc
    ON i.contract_id = rc.contract_id
  LEFT JOIN silver.si_branch_employee be
    ON rc.branch_code = be.branch_code
  GROUP BY be.region, rc.branch_code
),
cash_base AS (
  SELECT
      be.region,
      rc.branch_code,
      DATE(cr.receipt_date) AS metric_date,
      SUM(cr.payment_amount) AS total_payments,
      SUM(COALESCE(cr.unapplied_payment_amount, 0)) AS unapplied_cash
  FROM silver.si_cash_receipts cr
  LEFT JOIN silver.si_invoices i
    ON cr.invoice_id = i.invoice_id
  LEFT JOIN silver.si_rental_contracts rc
    ON i.contract_id = rc.contract_id
  LEFT JOIN silver.si_branch_employee be
    ON rc.branch_code = be.branch_code
  GROUP BY be.region, rc.branch_code, DATE(cr.receipt_date)
)
SELECT
    r.region,
    r.branch_code,
    r.metric_date,
    r.net_billings AS rental_revenue,
    a.total_ar_outstanding,
    CASE WHEN a.total_ar_outstanding = 0 THEN 0 ELSE ROUND((a.ar_90_plus / a.total_ar_outstanding) * 100, 2) END AS ar_90_plus_pct,
    CASE WHEN c.total_payments = 0 THEN 0 ELSE ROUND((c.unapplied_cash / c.total_payments) * 100, 2) END AS unapplied_cash_pct,
    GREATEST(0, LEAST(100,
      ROUND(
        (0.35 * LEAST(100, r.net_billings)) +
        (0.25 * (100 - CASE WHEN a.total_ar_outstanding = 0 THEN 0 ELSE (a.ar_90_plus / a.total_ar_outstanding) * 100 END)) +
        (0.20 * (100 - CASE WHEN c.total_payments = 0 THEN 0 ELSE (c.unapplied_cash / c.total_payments) * 100 END)) +
        (0.20 * 100), 2)
    )) AS revenue_to_cash_health_score
FROM revenue_base r
LEFT JOIN ar_base a
  ON r.region = a.region AND r.branch_code = a.branch_code
LEFT JOIN cash_base c
  ON r.region = c.region AND r.branch_code = c.branch_code AND r.metric_date = c.metric_date;
```
- **Recommended LVDash Widget:** KPI cards + bullet/goal cards + line chart + ranked table
- **Data Fields:** `region`, `branch_code`, `metric_date`, `rental_revenue`, `total_ar_outstanding`, `ar_90_plus_pct`, `unapplied_cash_pct`, `revenue_to_cash_health_score`
- **Calculations:** Health Score, AR 90+ %, revenue trend, unapplied cash %, optional DSO, CEI
- **Interactivity:** Region filter, branch filter, date granularity parameter, click-through from KPI card to detailed AR or revenue tab, hierarchy drill network -> region -> branch
- **Justification:** KPI cards surface daily/weekly executive signals quickly, line charts show trend direction, and a ranked table identifies branches/customers driving deterioration.
- **Optimization Tips:** Build a materialized aggregate by branch/date, cache result sets for executive tiles, partition by period date, and Z-order by `branch_code`, `region`.

### 3.2 Rental Revenue & Billing Performance
- **Data Element:** Revenue, gross/net billings, contract yield, and credit memo quality
- **SQL Query:**
```sql
SELECT
    be.region,
    rc.branch_code,
    rc.sales_rep_id,
    rc.equipment_class,
    DATE_TRUNC('month', i.invoice_date) AS period_month,
    COUNT(DISTINCT rc.contract_id) AS contract_count,
    COUNT(DISTINCT i.invoice_id) AS invoice_count,
    SUM(CASE WHEN i.invoice_type <> 'CREDIT_MEMO' THEN i.invoice_amount ELSE 0 END) AS gross_billings,
    SUM(CASE WHEN i.invoice_type = 'CREDIT_MEMO' THEN ABS(i.invoice_amount) ELSE 0 END) AS credit_memo_amount,
    SUM(i.invoice_amount) AS net_billings,
    SUM(i.invoice_amount) AS rental_revenue,
    CASE WHEN SUM(DATEDIFF(COALESCE(rc.contract_end_date, CURRENT_DATE()), rc.contract_start_date)) = 0 THEN NULL
         ELSE ROUND(SUM(i.invoice_amount) / SUM(DATEDIFF(COALESCE(rc.contract_end_date, CURRENT_DATE()), rc.contract_start_date)), 2) END AS average_daily_rate,
    CASE WHEN COUNT(DISTINCT rc.contract_id) = 0 THEN NULL
         ELSE ROUND(SUM(i.invoice_amount) / COUNT(DISTINCT rc.contract_id), 2) END AS revenue_per_contract,
    CASE WHEN SUM(CASE WHEN i.invoice_type <> 'CREDIT_MEMO' THEN i.invoice_amount ELSE 0 END) = 0 THEN 0
         ELSE ROUND(
           SUM(CASE WHEN i.invoice_type = 'CREDIT_MEMO' THEN ABS(i.invoice_amount) ELSE 0 END)
           / SUM(CASE WHEN i.invoice_type <> 'CREDIT_MEMO' THEN i.invoice_amount ELSE 0 END) * 100, 2) END AS credit_memo_rate
FROM silver.si_invoices i
LEFT JOIN silver.si_rental_contracts rc
  ON i.contract_id = rc.contract_id
LEFT JOIN silver.si_branch_employee be
  ON rc.branch_code = be.branch_code
WHERE COALESCE(rc.contract_status, 'OPEN') <> 'CANCELLED'
GROUP BY be.region, rc.branch_code, rc.sales_rep_id, rc.equipment_class, DATE_TRUNC('month', i.invoice_date);
```
- **Recommended LVDash Widget:** Combo chart, stacked bar chart, KPI strip, detail table
- **Data Fields:** `region`, `branch_code`, `sales_rep_id`, `equipment_class`, `period_month`, `gross_billings`, `credit_memo_amount`, `net_billings`, `rental_revenue`, `average_daily_rate`, `revenue_per_contract`, `credit_memo_rate`
- **Calculations:** Net Billings, ADR, Revenue per Contract, Credit Memo Rate, Invoice Count
- **Interactivity:** Drill region -> branch -> sales rep -> contract, equipment class filter, contract status filter, drill-through to invoice and contract details
- **Justification:** Combo chart is ideal for comparing revenue bars with rate trend lines, while a detailed table supports operational follow-up at sales rep/contract level.
- **Optimization Tips:** Create monthly summary tables, persist equipment-class mix datasets, prune by invoice date, and reuse the same dataset across KPI and chart widgets.

### 3.3 AR Aging & Collections
- **Data Element:** AR exposure and aging prioritization by branch, collector, customer, and invoice
- **SQL Query:**
```sql
WITH ar_snapshot AS (
  SELECT
      be.region,
      rc.branch_code,
      COALESCE(emp.collector_id, rc.sales_rep_id) AS collector_id,
      i.customer_id,
      cm.customer_name,
      i.invoice_id,
      i.invoice_date,
      i.due_date,
      CURRENT_DATE() AS snapshot_date,
      i.invoice_amount AS outstanding_amount,
      DATEDIFF(CURRENT_DATE(), i.due_date) AS days_past_due,
      CASE
        WHEN DATEDIFF(CURRENT_DATE(), i.due_date) <= 30 THEN '0-30'
        WHEN DATEDIFF(CURRENT_DATE(), i.due_date) <= 60 THEN '31-60'
        WHEN DATEDIFF(CURRENT_DATE(), i.due_date) <= 90 THEN '61-90'
        ELSE '90+'
      END AS aging_bucket
  FROM silver.si_invoices i
  LEFT JOIN silver.si_rental_contracts rc
    ON i.contract_id = rc.contract_id
  LEFT JOIN silver.si_customer_master cm
    ON i.customer_id = cm.customer_id
  LEFT JOIN silver.si_branch_employee be
    ON rc.branch_code = be.branch_code
  LEFT JOIN silver.si_branch_employee emp
    ON rc.sales_rep_id = emp.employee_id
  WHERE i.invoice_amount > 0
)
SELECT
    region,
    branch_code,
    collector_id,
    customer_id,
    customer_name,
    snapshot_date,
    aging_bucket,
    SUM(outstanding_amount) AS total_ar_outstanding,
    SUM(CASE WHEN aging_bucket = '90+' THEN outstanding_amount ELSE 0 END) AS ar_90_plus_amount,
    CASE WHEN SUM(outstanding_amount) = 0 THEN 0
         ELSE ROUND(SUM(CASE WHEN aging_bucket = '90+' THEN outstanding_amount ELSE 0 END) / SUM(outstanding_amount) * 100, 2) END AS ar_90_plus_pct
FROM ar_snapshot
GROUP BY region, branch_code, collector_id, customer_id, customer_name, snapshot_date, aging_bucket;
```
- **Recommended LVDash Widget:** Stacked bar chart, line chart, KPI cards, invoice-level table, pivot table
- **Data Fields:** `region`, `branch_code`, `collector_id`, `customer_id`, `customer_name`, `snapshot_date`, `aging_bucket`, `total_ar_outstanding`, `ar_90_plus_amount`, `ar_90_plus_pct`
- **Calculations:** Aging Bucket, Total AR Outstanding, AR 90+ %, DSO, Best Possible DSO
- **Interactivity:** Aging bucket cross-filter, branch and collector slicers, drill-through to invoice list, time grain parameter snapshot/week/month
- **Justification:** Stacked bars clearly compare aging mix across branches, while KPI cards and invoice tables support both executive and collector-level analysis.
- **Optimization Tips:** Materialize daily AR snapshots, avoid recalculating aging on raw invoice data for every render, partition snapshot facts by `snapshot_date`, and cache common branch-level summaries.

### 3.4 Cash Application & Collector Performance
- **Data Element:** Collection effectiveness, days to pay, and unapplied cash operations
- **SQL Query:**
```sql
SELECT
    be.region,
    rc.branch_code,
    COALESCE(emp.collector_id, rc.sales_rep_id) AS collector_id,
    DATE_TRUNC('week', cr.receipt_date) AS period_week,
    cr.payment_method,
    SUM(cr.payment_amount) AS cash_collected,
    SUM(COALESCE(cr.unapplied_payment_amount, 0)) AS unapplied_cash_amount,
    CASE WHEN SUM(cr.payment_amount) = 0 THEN 0
         ELSE ROUND(SUM(COALESCE(cr.unapplied_payment_amount, 0)) / SUM(cr.payment_amount) * 100, 2) END AS unapplied_cash_pct,
    AVG(DATEDIFF(cr.receipt_date, i.invoice_date)) AS avg_days_to_pay,
    AVG(CASE WHEN cr.receipt_date <= i.due_date THEN 1 ELSE 0 END) * 100 AS pct_invoices_paid_on_time,
    ROUND(
      (
        (SUM(i.invoice_amount) + SUM(i.invoice_amount) - SUM(CASE WHEN i.days_past_due > 0 THEN i.invoice_amount ELSE 0 END)) /
        NULLIF((SUM(i.invoice_amount) + SUM(i.invoice_amount) - SUM(CASE WHEN i.days_past_due <= 30 THEN i.invoice_amount ELSE 0 END)), 0)
      ) * 100, 2
    ) AS cei
FROM silver.si_cash_receipts cr
LEFT JOIN silver.si_invoices i
  ON cr.invoice_id = i.invoice_id
LEFT JOIN silver.si_rental_contracts rc
  ON i.contract_id = rc.contract_id
LEFT JOIN silver.si_branch_employee be
  ON rc.branch_code = be.branch_code
LEFT JOIN silver.si_branch_employee emp
  ON rc.sales_rep_id = emp.employee_id
GROUP BY be.region, rc.branch_code, COALESCE(emp.collector_id, rc.sales_rep_id), DATE_TRUNC('week', cr.receipt_date), cr.payment_method;
```
- **Recommended LVDash Widget:** Ranked horizontal bar chart, KPI cards, trend line, unapplied cash table
- **Data Fields:** `region`, `branch_code`, `collector_id`, `period_week`, `payment_method`, `cash_collected`, `unapplied_cash_amount`, `unapplied_cash_pct`, `avg_days_to_pay`, `pct_invoices_paid_on_time`, `cei`
- **Calculations:** CEI, Average Days to Pay, % Invoices Paid On Time, Unapplied Cash $, Unapplied Cash %
- **Interactivity:** Collector filter, payment method filter, click on leaderboard bar to cross-filter detail table, week/month parameter, drill-through to payment/invoice detail
- **Justification:** A ranked horizontal bar is ideal for comparing collectors, while KPI cards and trend lines expose operational efficiency and movement over time.
- **Optimization Tips:** Precompute weekly collector metrics, isolate unapplied receipts into a narrow helper dataset, cache performance summaries, and tune SQL warehouse size for concurrent leaderboard usage.

### 3.5 Customer Credit Risk Report
- **Data Element:** Customer credit exposure, over-limit behavior, and suspended exposure
- **SQL Query:**
```sql
WITH customer_exposure AS (
  SELECT
      be.region,
      rc.branch_code,
      cm.customer_id,
      cm.customer_name,
      cm.credit_terms,
      cm.credit_limit,
      cm.customer_status,
      cm.customer_since,
      SUM(CASE WHEN i.invoice_amount > 0 THEN i.invoice_amount ELSE 0 END) AS outstanding_ar,
      SUM(CASE WHEN COALESCE(i.days_past_due, 0) > 90 THEN i.invoice_amount ELSE 0 END) AS ar_90_plus_amount,
      COUNT(DISTINCT CASE WHEN COALESCE(rc.contract_status, 'OPEN') = 'OPEN' THEN rc.contract_id END) AS open_contract_count,
      SUM(CASE WHEN COALESCE(rc.contract_status, 'OPEN') = 'OPEN' THEN i.invoice_amount ELSE 0 END) AS open_contract_value
  FROM silver.si_customer_master cm
  LEFT JOIN silver.si_invoices i
    ON cm.customer_id = i.customer_id
  LEFT JOIN silver.si_rental_contracts rc
    ON i.contract_id = rc.contract_id
  LEFT JOIN silver.si_branch_employee be
    ON rc.branch_code = be.branch_code
  GROUP BY be.region, rc.branch_code, cm.customer_id, cm.customer_name, cm.credit_terms, cm.credit_limit, cm.customer_status, cm.customer_since
)
SELECT
    region,
    branch_code,
    customer_id,
    customer_name,
    credit_terms,
    customer_status,
    customer_since,
    credit_limit,
    outstanding_ar,
    ar_90_plus_amount,
    open_contract_count,
    open_contract_value,
    CASE WHEN credit_limit IS NULL OR credit_limit = 0 THEN NULL
         ELSE ROUND(outstanding_ar / credit_limit * 100, 2) END AS credit_utilization_pct,
    CASE WHEN credit_limit IS NOT NULL AND outstanding_ar > credit_limit THEN 1 ELSE 0 END AS customer_over_limit_flag,
    CASE WHEN customer_status = 'Suspended' AND (outstanding_ar > 0 OR open_contract_count > 0) THEN 1 ELSE 0 END AS suspended_with_exposure_flag
FROM customer_exposure;
```
- **Recommended LVDash Widget:** Heatmap, KPI cards, scatter/bubble chart, sorted detail table
- **Data Fields:** `region`, `branch_code`, `customer_id`, `customer_name`, `credit_terms`, `customer_status`, `customer_since`, `credit_limit`, `outstanding_ar`, `ar_90_plus_amount`, `open_contract_count`, `credit_utilization_pct`, `customer_over_limit_flag`, `suspended_with_exposure_flag`
- **Calculations:** Credit Utilization %, Customers Over Limit count and amount, AR 90+ by Customer, Suspended Customers with Open Exposure
- **Interactivity:** Region/branch/customer status filters, drill-through to invoice and contract detail, over-limit toggle parameter, trend drill-up to network exposure trend
- **Justification:** Heatmaps quickly identify concentration of credit risk by branch, while a sortable detail table allows credit teams to act on the highest-risk customers.
- **Optimization Tips:** Snapshot customer exposure daily, separate executive aggregate datasets from customer-level detail, Z-order by `customer_id`, `branch_code`, and limit high-cardinality detail rendering through parameterized filters.

## 4. Supporting Semantic Model Recommendations

### 4.1 Recommended Data Model
| Layer | Object | Type | Purpose |
|---|---|---|---|
| Silver | `si_rental_contracts` | Core detail table | Contract grain and branch/equipment/sales attributes |
| Silver | `si_invoices` | Core detail table | Billing and AR basis |
| Silver | `si_cash_receipts` | Core detail table | Payment and cash application basis |
| Silver | `si_customer_master` | Dimension-style master table | Customer risk and status attributes |
| Silver | `si_branch_employee` | Organization dimension support | Branch, region, sales rep, collector context |
| Gold | `fct_rental_revenue` | Fact table / view | Revenue and billing aggregates |
| Gold | `fct_ar_aging` | Snapshot fact table | AR aging balances and buckets |
| Gold | `fct_cash_application` | Fact table | Applied and unapplied cash metrics |
| Gold | `fct_credit_risk` | Snapshot fact table | Customer exposure and risk metrics |
| Gold | `dim_date`, `dim_branch`, `dim_customer`, `dim_employee` | Conformed dimensions | Shared dashboard slicing and drill paths |

### 4.2 Recommended Transformations
- Standardize branch-region hierarchy into a reusable `dim_branch`.
- Create `dim_employee` with role flags for sales rep and collector.
- Persist aging snapshots daily instead of recomputing on the fly.
- Build metric-ready gold facts for scorecards, collector performance, and credit risk.
- Apply deduplication on customer master before risk aggregation.

## 5. Overall Dashboard Design

### 5.1 Layout Suggestions
| Tab | Purpose | Suggested Layout |
|---|---|---|
| Executive Summary | Executive scorecards and cross-domain health view | Top row KPI tiles, middle trend row, bottom leaderboard + drivers table |
| Revenue & Billing | Revenue performance and billing quality | Top KPI strip, middle combo chart and equipment mix, bottom sales rep/contract detail table |
| AR Aging | Past-due exposure and collections prioritization | Top KPI cards, middle stacked aging chart, bottom invoice detail and pivot |
| Cash Application | Collector efficiency and unapplied cash | Top KPI cards, middle collector leaderboard and trend, bottom unapplied cash detail |
| Credit Risk | Exposure, over-limit, and suspended customer monitoring | Top KPI cards, middle branch heatmap/scatter, bottom customer detail table |

### 5.2 Widget Sizing / Positioning
- Use a 12-column Lakeview grid.
- Keep 4 to 6 KPI cards in the top row at equal width.
- Use full-width trend charts where time-series interpretation is critical.
- Use side-by-side charts only where cross-comparison is helpful, such as branch revenue vs equipment mix.
- Reserve bottom sections for detailed tables with pagination and export enabled.

### 5.3 Query Optimization
- Pre-aggregate daily, weekly, and monthly datasets in Gold.
- Enable query result caching for heavily reused datasets.
- Partition snapshot facts by business date or snapshot date.
- Z-order Delta tables on `branch_code`, `customer_id`, `invoice_id`, and `contract_id` based on access patterns.
- Use materialized views for executive scorecards and branch leaderboards.
- Size SQL warehouses to match concurrent dashboard loads; use serverless or pro warehouse with auto-stop and multi-cluster for peak usage.
- Reuse a small number of shared datasets across multiple widgets instead of issuing similar SQL repeatedly.
- Avoid oversized all-purpose datasets feeding many unrelated visuals.

### 5.4 Color Scheme
| Purpose | Recommended Color |
|---|---|
| Positive / on target | Green (`#2E7D32`) |
| Warning / moderate risk | Amber (`#F9A825`) |
| Critical / overdue / over limit | Red (`#C62828`) |
| Neutral / baseline | Blue (`#1565C0`) |
| Supporting background / muted | Light Gray (`#ECEFF1`) |

### 5.5 Typography
- Use clear sans-serif fonts such as Inter, Arial, or Helvetica.
- KPI numbers should be bold and larger than labels.
- Keep chart titles concise and action-oriented.
- Use consistent number formatting for currency, percentages, and whole-number counts.
- Minimize visual clutter in tables with aligned decimals and conditional formatting only where meaningful.

### 5.6 Interactive Elements
- Global filters: Date range, region, branch, equipment class, customer status, collector, sales rep.
- Parameters: Period type (day/week/month/quarter), top N ranking, over-limit threshold, metric selector.
- Cross-filtering: Click branch bar/collector bar/aging bucket segment to filter related tables and KPIs.
- Drill-down: Network -> Region -> Branch -> Sales Rep/Collector -> Customer/Contract/Invoice.
- Drill-through: From KPI or chart to invoice detail, contract detail, customer risk detail, or unapplied cash detail.
- Drill-up: Week -> Month -> Quarter and Branch -> Region -> Network.

## 6. Interactivity Features Table

| Feature Type | Suggested Control | Applies To | Purpose |
|---|---|---|---|
| Filter | Date range picker | All tabs | Constrain reporting period and snapshots |
| Filter | Region dropdown | All tabs | Support geographic rollups |
| Filter | Branch multi-select | Executive, Revenue, AR, Cash, Credit Risk | Narrow analysis to operational unit |
| Filter | Equipment class dropdown | Executive, Revenue | Compare rental mix and billing trends |
| Filter | Customer status dropdown | Executive, AR, Credit Risk | Focus on active vs suspended customer exposure |
| Filter | Collector dropdown | AR, Cash Application | Prioritize workload and performance analysis |
| Filter | Sales rep dropdown | Executive, Revenue | Compare selling performance |
| Parameter | Period granularity (`Day/Week/Month/Quarter`) | Executive, Revenue, AR, Cash | Dynamic time aggregation |
| Parameter | Top N selector | Executive, Cash, Credit Risk | Limit leaderboard output for readability |
| Parameter | Risk threshold | Credit Risk | Highlight customers above a configurable utilization % |
| Cross-filter | Click branch/region chart elements | All tabs | Sync downstream charts and tables |
| Cross-filter | Click aging bucket stack segment | AR Aging | Filter invoice-level detail to selected bucket |
| Drill-down | Hierarchy navigation | Executive, Revenue, AR, Cash | Move from aggregate to operational detail |
| Drill-through | Invoice / contract detail page | Revenue, AR, Cash, Credit Risk | Investigate root-cause records |
| Drill up/down | Time hierarchy | Executive, Revenue, AR, Cash | Compare weekly, monthly, quarterly performance |

## 7. Potential Pitfalls and Mitigations

| Pitfall | Impact | Mitigation |
|---|---|---|
| Joining invoices to branch employee without deduplicating branch assignments | Duplicate facts and inflated metrics | Build conformed branch and employee dimensions with effective dating and unique join logic |
| Recomputing aging buckets at render time on large invoice history | Slow AR dashboard performance | Persist daily AR snapshot facts and cache common summaries |
| Using a single wide dataset for every widget | Excessive scans and poor Lakeview responsiveness | Create focused reusable datasets by domain |
| Null or zero denominators in DSO, CEI, credit utilization | Invalid percentages or runtime errors | Use `CASE` / `NULLIF` safeguards and expose data quality flags |
| Credit memos not handled with sign-aware logic | Misstated net billings and rate metrics | Enforce invoice type rules and standard signed calculations |
| Large shuffles on branch/customer joins | Increased latency and compute cost | Optimize Delta layout, Z-order, and pre-aggregate gold facts |
| Overly small SQL warehouse for concurrent executive usage | Dashboard lag during business hours | Use auto-scaling warehouse sizing and result caching |
| Excessive cross-filter dependencies on large detail tables | Interaction delays | Limit cross-filtering to summary datasets and parameterize detail retrieval |

## 8. Best-Practice Notes for Databricks Lakeview
- Keep semantic datasets business-friendly and reuse them across tabs.
- Prefer Gold curated views for dashboarding over direct Silver access where metrics are standardized.
- Use Delta `OPTIMIZE` and `ZORDER` on frequently filtered columns.
- Refresh snapshots and materialized views on schedules aligned to reporting cadence.
- Keep KPI datasets narrow and lightweight to improve first render time.
- Separate executive-level datasets from operational drill-through detail datasets.

## 9. Output URL (Clickable Hyperlinks)
[Databricks_Dashboard_Visuals_Recommender_1.md](https://github.com/DIAscendion/Databricks_Dashboard/blob/main/DI_Databricks_Dashboard_Visuals_Recommender/Databricks_Dashboard_Visuals_Recommender_1.md)

outputURL : https://github.com/DIAscendion/Databricks_Dashboard/tree/main/DI_Databricks_Dashboard_Visuals_Recommender
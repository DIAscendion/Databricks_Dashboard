_____________________________________________
## *Author*: AAVA
## *Created on*:   
## *Description*: Databricks Lakeview dashboard visuals recommender for Rental Revenue-to-Cash reporting across revenue, AR, cash application, and credit risk domains.
## *Version*: 1
## *Updated on*: 
_____________________________________________

# Databricks Lakeview Dashboard Visuals Recommender

This document analyzes the Rental Revenue-to-Cash reporting requirements and the Silver physical model to recommend Databricks SQL queries, reusable analytical datasets, and Lakeview Dashboard visuals that best support executive scorecards, billing performance, AR aging, cash application, and customer credit risk monitoring.

Print the URL : "https://github.com/DIAscendion/Databricks_Dashboard/tree/main/DI_Databricks_Dashboard_Visuals_Recommender"

## 1. Input Analysis Summary

### 1.1 Source Inputs Reviewed

| Input File | Purpose | Key Contribution to Dashboard Recommendations |
|---|---|---|
| `DI_Databricks_Silver_Model_Physical/Databricks_Silver_Model_Physical_1.md` | Silver physical model for cleansed and conformed Rental Revenue-to-Cash data | Provided source tables, join paths, core fields, partitioning hints, and Delta optimization guidance |
| `Input/Rental_Revenue_to_Cash_Reports.txt` | Business reporting requirements for consolidated AR and revenue dashboards | Defined KPIs, dimensions, drill paths, validations, and business layout expectations |

### 1.2 Identified Data Sources

| Layer | Table | Role in Reporting |
|---|---|---|
| Silver | `silver.si_rental_contracts` | Contract, branch, sales rep, equipment class, contract lifecycle context |
| Silver | `silver.si_invoices` | Billing, invoice dates, due dates, invoice amounts, aging support |
| Silver | `silver.si_cash_receipts` | Cash collection and application metrics |
| Silver | `silver.si_customer_master` | Customer profile, credit terms, credit limit, status |
| Silver | `silver.si_branch_employee` | Sales rep / collector / branch / region hierarchy |
| Silver | `silver.si_data_quality_errors` | Monitoring data quality exceptions for reporting reliability |
| Silver | `silver.si_process_audit` | Pipeline observability and freshness validation |
| Gold / Recommended | `gold.fct_rental_revenue` | Revenue and billing fact for aggregated dashboard reporting |
| Gold / Recommended | `gold.fct_ar_aging` | AR aging fact at invoice snapshot grain |
| Gold / Recommended | `gold.fct_cash_application` | Cash application and collector performance fact |
| Gold / Recommended | `gold.fct_credit_risk` | Customer exposure and credit risk fact |
| Gold / Recommended | `gold.dim_branch`, `gold.dim_customer`, `gold.dim_date`, `gold.dim_employee`, `gold.dim_contract` | Conformed dimensions for reusable Lakeview datasets |

### 1.3 Target Analytical Models Recommended

| Model Type | Suggested Name | Grain | Primary Use |
|---|---|---|---|
| Fact | `gold.fct_rental_revenue` | Branch, date, sales rep, contract, equipment class, invoice | Revenue, billings, ADR, credit memo rate |
| Fact | `gold.fct_ar_aging` | Snapshot date, customer, invoice | AR aging, DSO, AR 90+ %, branch/collector prioritization |
| Fact | `gold.fct_cash_application` | Receipt or receipt-application line | Cash collected, unapplied cash, days to pay, CEI support |
| Fact | `gold.fct_credit_risk` | Customer, snapshot date | Credit utilization, over-limit, suspended exposure |
| Aggregate | `gold.agg_exec_scorecard_daily` | Branch, date, period type | Executive health score and domain scorecards |
| Aggregate | `gold.agg_exec_scorecard_weekly` | Branch, week | Weekly rollup with trend comparability |

### 1.4 Core Transformations, Joins, Aggregations, and Filters

| Category | Recommendation |
|---|---|
| Core Joins | Join contracts to customer and branch employee; invoices to contracts and customers; cash receipts to invoices and customers; branch hierarchy from employee/branch mapping |
| Derived Metrics | Net billings, ADR, revenue per contract, DSO, AR 90+ %, CEI, average days to pay, credit utilization %, suspended exposure flags |
| Snapshot Logic | AR aging and credit risk should be built as periodic snapshots by `snapshot_date` |
| Common Filters | Region, branch, date period, sales rep, collector, equipment class, customer status, contract status, aging bucket, payment match status |
| Reusable Hierarchies | Network -> Region -> Branch -> Sales Rep/Collector -> Customer/Contract/Invoice |
| Output Format | Curated Lakeview datasets backed by Gold tables or aggregate views for fast dashboard rendering |

## 2. Visual Recommendations

### 2.1 Revenue-to-Cash Executive Scorecard

- **Data Element:** Overall Revenue-to-Cash Health Score and domain health by branch/region/date
- **SQL Query:**
```sql
WITH revenue_metrics AS (
  SELECT
    branch_code,
    region,
    date_day,
    SUM(net_billings) AS rental_revenue,
    SUM(CASE WHEN gross_billings <> 0 THEN credit_memo_amount ELSE 0 END) AS credit_memo_amount
  FROM gold.fct_rental_revenue
  GROUP BY branch_code, region, date_day
),
ar_metrics AS (
  SELECT
    branch_code,
    region,
    snapshot_date AS date_day,
    SUM(ar_outstanding_amount) AS total_ar,
    SUM(CASE WHEN aging_bucket = '90+' THEN ar_outstanding_amount ELSE 0 END) AS ar_90_plus,
    CASE
      WHEN SUM(credit_sales_amount) = 0 THEN NULL
      ELSE (SUM(ar_outstanding_amount) / SUM(credit_sales_amount)) * MAX(days_in_period)
    END AS dso
  FROM gold.fct_ar_aging
  GROUP BY branch_code, region, snapshot_date
),
cash_metrics AS (
  SELECT
    branch_code,
    region,
    payment_date AS date_day,
    SUM(payment_amount) AS total_payments,
    SUM(CASE WHEN match_status = 'UNAPPLIED' THEN payment_amount ELSE 0 END) AS unapplied_cash,
    AVG(days_to_pay) AS avg_days_to_pay,
    CASE
      WHEN SUM(payment_amount) = 0 THEN NULL
      ELSE 100 * SUM(CASE WHEN match_status = 'UNAPPLIED' THEN payment_amount ELSE 0 END) / SUM(payment_amount)
    END AS unapplied_cash_pct
  FROM gold.fct_cash_application
  GROUP BY branch_code, region, payment_date
),
credit_metrics AS (
  SELECT
    branch_code,
    region,
    snapshot_date AS date_day,
    SUM(CASE WHEN over_limit_flag = 1 THEN 1 ELSE 0 END) AS customers_over_limit,
    SUM(CASE WHEN suspended_with_exposure_flag = 1 THEN 1 ELSE 0 END) AS suspended_exposure_customers
  FROM gold.fct_credit_risk
  GROUP BY branch_code, region, snapshot_date
)
SELECT
  r.branch_code,
  r.region,
  r.date_day,
  rental_revenue,
  a.total_ar,
  a.ar_90_plus,
  a.dso,
  c.total_payments,
  c.unapplied_cash_pct,
  cr.customers_over_limit,
  cr.suspended_exposure_customers,
  LEAST(100,
    0.30 * COALESCE(100 * rental_revenue / NULLIF(MAX(rental_revenue) OVER (PARTITION BY r.date_day),0),0) +
    0.30 * COALESCE(100 - a.dso,0) +
    0.20 * COALESCE(100 - c.unapplied_cash_pct,0) +
    0.20 * COALESCE(100 - 10 * cr.customers_over_limit,0)
  ) AS health_score
FROM revenue_metrics r
LEFT JOIN ar_metrics a
  ON r.branch_code = a.branch_code AND r.region = a.region AND r.date_day = a.date_day
LEFT JOIN cash_metrics c
  ON r.branch_code = c.branch_code AND r.region = c.region AND r.date_day = c.date_day
LEFT JOIN credit_metrics cr
  ON r.branch_code = cr.branch_code AND r.region = cr.region AND r.date_day = cr.date_day;
```
- **Recommended LVDash Widget:** Counter/KPI cards for Overall Health Score and four domain scores
- **Data Fields:** `branch_code`, `region`, `date_day`, `health_score`, `rental_revenue`, `dso`, `ar_90_plus`, `unapplied_cash_pct`, `customers_over_limit`
- **Calculations:** Weighted health score, DSO, AR 90+ %, unapplied cash %, domain normalization
- **Interactivity:** Region and branch filters, date granularity parameter (daily/weekly), click-through from KPI card to supporting tab, drill-down network -> region -> branch
- **Justification:** KPI cards are best for executive monitoring because they immediately show current performance against target thresholds and support fast exception detection.
- **Optimization Tips:** Materialize daily and weekly aggregates, cache scorecard datasets, partition by period/date, Z-ORDER on `region`, `branch_code`, and `date_day`, and avoid recomputing window-based normalization on every dashboard refresh.

### 2.2 Revenue Trend vs DSO Trend

- **Data Element:** Period trend of rental revenue and DSO
- **SQL Query:**
```sql
SELECT
  d.period_start,
  r.region,
  r.branch_code,
  SUM(r.net_billings) AS rental_revenue,
  CASE
    WHEN SUM(a.credit_sales_amount) = 0 THEN NULL
    ELSE (SUM(a.ar_outstanding_amount) / SUM(a.credit_sales_amount)) * MAX(a.days_in_period)
  END AS dso
FROM gold.fct_rental_revenue r
JOIN gold.dim_date d
  ON r.date_key = d.date_key
LEFT JOIN gold.fct_ar_aging a
  ON r.branch_code = a.branch_code
 AND r.region = a.region
 AND d.date_day = a.snapshot_date
GROUP BY d.period_start, r.region, r.branch_code
ORDER BY d.period_start;
```
- **Recommended LVDash Widget:** Combo chart with bars for revenue and line for DSO
- **Data Fields:** `period_start`, `region`, `branch_code`, `rental_revenue`, `dso`
- **Calculations:** Sum of net billings, DSO ratio by period
- **Interactivity:** Date range filter, region/branch slicers, drill-up week -> month -> quarter, cross-filter to AR tab
- **Justification:** A combo chart is ideal because revenue and DSO have different scales but should be compared across the same timeline.
- **Optimization Tips:** Pre-aggregate by chosen period grain, use Lakeview parameterized period selection, and reuse one dataset across trend widgets instead of issuing separate queries.

### 2.3 AR 90+ Leaderboard by Branch

- **Data Element:** High-risk branches ranked by AR 90+ exposure
- **SQL Query:**
```sql
SELECT
  region,
  branch_code,
  SUM(ar_outstanding_amount) AS total_ar,
  SUM(CASE WHEN aging_bucket = '90+' THEN ar_outstanding_amount ELSE 0 END) AS ar_90_plus_amount,
  CASE
    WHEN SUM(ar_outstanding_amount) = 0 THEN NULL
    ELSE 100 * SUM(CASE WHEN aging_bucket = '90+' THEN ar_outstanding_amount ELSE 0 END) / SUM(ar_outstanding_amount)
  END AS ar_90_plus_pct
FROM gold.fct_ar_aging
WHERE snapshot_date BETWEEN :start_date AND :end_date
GROUP BY region, branch_code
ORDER BY ar_90_plus_pct DESC, ar_90_plus_amount DESC;
```
- **Recommended LVDash Widget:** Horizontal bar chart
- **Data Fields:** `region`, `branch_code`, `total_ar`, `ar_90_plus_amount`, `ar_90_plus_pct`
- **Calculations:** Total AR, 90+ bucket amount, 90+ percentage
- **Interactivity:** Region filter, click bar to drill to customer or invoice aging detail, date snapshot parameter
- **Justification:** Horizontal bars clearly support ranking and make it easy to compare branch risk exposure side by side.
- **Optimization Tips:** Build snapshot summary table by branch/date, partition by `snapshot_date`, Z-ORDER on `branch_code`, and prune date range early.

### 2.4 Aging Drivers Table by Branch and Customer

- **Data Element:** Customers driving aged receivables risk
- **SQL Query:**
```sql
SELECT
  a.snapshot_date,
  a.region,
  a.branch_code,
  a.customer_id,
  c.customer_name,
  a.collector_id,
  SUM(a.ar_outstanding_amount) AS total_ar,
  SUM(CASE WHEN a.aging_bucket = '90+' THEN a.ar_outstanding_amount ELSE 0 END) AS ar_90_plus_amount,
  MAX(a.days_past_due) AS max_days_past_due,
  COUNT(DISTINCT a.invoice_id) AS open_invoice_count
FROM gold.fct_ar_aging a
JOIN gold.dim_customer c
  ON a.customer_id = c.customer_id
WHERE a.snapshot_date = :snapshot_date
GROUP BY a.snapshot_date, a.region, a.branch_code, a.customer_id, c.customer_name, a.collector_id
ORDER BY ar_90_plus_amount DESC, total_ar DESC;
```
- **Recommended LVDash Widget:** Table
- **Data Fields:** `snapshot_date`, `region`, `branch_code`, `customer_id`, `customer_name`, `collector_id`, `total_ar`, `ar_90_plus_amount`, `max_days_past_due`, `open_invoice_count`
- **Calculations:** Sum outstanding, max days past due, invoice count
- **Interactivity:** Search, sort, filter by collector/customer status, drill-through to invoice list
- **Justification:** A table is the best choice when users need exact values, operational prioritization, and the ability to inspect customer-level drivers.
- **Optimization Tips:** Keep detail table on-demand with selective filters; do not preload all invoice rows into the landing tab. Use a summary dataset for the main table and separate detail dataset for drill-through.

### 2.5 Rental Revenue & Billing KPI Strip

- **Data Element:** Revenue, gross billings, net billings, ADR, revenue per contract, credit memo rate
- **SQL Query:**
```sql
SELECT
  region,
  branch_code,
  sales_rep_id,
  equipment_class,
  date_day,
  SUM(CASE WHEN invoice_type = 'STANDARD' THEN invoice_amount ELSE 0 END) AS gross_billings,
  SUM(CASE WHEN invoice_type = 'CREDIT_MEMO' THEN ABS(invoice_amount) ELSE 0 END) AS credit_memo_amount,
  SUM(CASE WHEN invoice_type = 'STANDARD' THEN invoice_amount ELSE 0 END)
    - SUM(CASE WHEN invoice_type = 'CREDIT_MEMO' THEN ABS(invoice_amount) ELSE 0 END) AS net_billings,
  SUM(rental_revenue_amount) AS rental_revenue,
  CASE WHEN SUM(billed_rental_days) = 0 THEN NULL ELSE SUM(rental_revenue_amount) / SUM(billed_rental_days) END AS adr,
  CASE WHEN COUNT(DISTINCT contract_id) = 0 THEN NULL ELSE SUM(rental_revenue_amount) / COUNT(DISTINCT contract_id) END AS revenue_per_contract,
  CASE WHEN SUM(CASE WHEN invoice_type = 'STANDARD' THEN invoice_amount ELSE 0 END) = 0 THEN NULL
       ELSE SUM(CASE WHEN invoice_type = 'CREDIT_MEMO' THEN ABS(invoice_amount) ELSE 0 END)
            / SUM(CASE WHEN invoice_type = 'STANDARD' THEN invoice_amount ELSE 0 END)
  END AS credit_memo_rate
FROM gold.fct_rental_revenue
GROUP BY region, branch_code, sales_rep_id, equipment_class, date_day;
```
- **Recommended LVDash Widget:** KPI cards
- **Data Fields:** `region`, `branch_code`, `sales_rep_id`, `equipment_class`, `date_day`, `gross_billings`, `net_billings`, `rental_revenue`, `adr`, `revenue_per_contract`, `credit_memo_rate`
- **Calculations:** Net billings, ADR, revenue per contract, credit memo rate
- **Interactivity:** Period, branch, equipment class, sales rep filters; drill from KPI to invoice detail
- **Justification:** KPI cards are suitable for headline metrics where current-period values need immediate visibility.
- **Optimization Tips:** Use a daily billing aggregate, isolate standard invoices and credit memos in ETL, and precompute billed rental days for ADR to reduce repeated calculations.

### 2.6 Revenue Trend by Branch

- **Data Element:** Rental revenue trend over time by branch
- **SQL Query:**
```sql
SELECT
  date_day,
  region,
  branch_code,
  SUM(rental_revenue_amount) AS rental_revenue
FROM gold.fct_rental_revenue
WHERE date_day BETWEEN :start_date AND :end_date
GROUP BY date_day, region, branch_code
ORDER BY date_day;
```
- **Recommended LVDash Widget:** Line chart
- **Data Fields:** `date_day`, `region`, `branch_code`, `rental_revenue`
- **Calculations:** Sum revenue by time
- **Interactivity:** Region and branch filters, drill from branch to sales rep, date granularity parameter
- **Justification:** A line chart is most appropriate for showing time-based movement and identifying growth or decline patterns.
- **Optimization Tips:** Aggregate at daily grain once, then derive weekly/monthly views from date dimension. Reuse this dataset in multiple trend widgets.

### 2.7 Equipment Class Mix

- **Data Element:** Revenue distribution by equipment class
- **SQL Query:**
```sql
SELECT
  date_trunc('month', date_day) AS month_start,
  region,
  branch_code,
  equipment_class,
  SUM(rental_revenue_amount) AS rental_revenue
FROM gold.fct_rental_revenue
GROUP BY date_trunc('month', date_day), region, branch_code, equipment_class
ORDER BY month_start, rental_revenue DESC;
```
- **Recommended LVDash Widget:** Stacked bar chart
- **Data Fields:** `month_start`, `region`, `branch_code`, `equipment_class`, `rental_revenue`
- **Calculations:** Revenue by class and period
- **Interactivity:** Region, branch, month filters; cross-filter to detail table
- **Justification:** Stacked bars effectively show both total branch revenue and the contribution of each equipment class.
- **Optimization Tips:** Pre-aggregate monthly class mix and limit number of displayed classes to top contributors plus "Other" when necessary.

### 2.8 Sales Rep / Contract Billing Detail

- **Data Element:** Detailed revenue and billing performance by sales rep and contract
- **SQL Query:**
```sql
SELECT
  region,
  branch_code,
  sales_rep_id,
  contract_id,
  contract_status,
  equipment_class,
  COUNT(DISTINCT invoice_id) AS invoice_count,
  SUM(rental_revenue_amount) AS rental_revenue,
  SUM(net_billings) AS net_billings,
  SUM(credit_memo_amount) AS credit_memo_amount
FROM gold.fct_rental_revenue
WHERE date_day BETWEEN :start_date AND :end_date
GROUP BY region, branch_code, sales_rep_id, contract_id, contract_status, equipment_class
ORDER BY rental_revenue DESC;
```
- **Recommended LVDash Widget:** Table
- **Data Fields:** `region`, `branch_code`, `sales_rep_id`, `contract_id`, `contract_status`, `equipment_class`, `invoice_count`, `rental_revenue`, `net_billings`, `credit_memo_amount`
- **Calculations:** Invoice count, revenue, net billings, credit memo amount
- **Interactivity:** Drill-down region -> branch -> sales rep -> contract; export table; filter by contract status
- **Justification:** A detailed table supports operational review and exception analysis down to contract level.
- **Optimization Tips:** Use query-based pagination or a filtered dataset for large result sets; do not expose unrestricted full-history contract detail on the default page.

### 2.9 AR Aging by Bucket and Branch

- **Data Element:** Outstanding AR split by aging bucket across branches
- **SQL Query:**
```sql
SELECT
  snapshot_date,
  region,
  branch_code,
  aging_bucket,
  SUM(ar_outstanding_amount) AS ar_amount
FROM gold.fct_ar_aging
WHERE snapshot_date = :snapshot_date
GROUP BY snapshot_date, region, branch_code, aging_bucket
ORDER BY branch_code, aging_bucket;
```
- **Recommended LVDash Widget:** Stacked bar chart
- **Data Fields:** `snapshot_date`, `region`, `branch_code`, `aging_bucket`, `ar_amount`
- **Calculations:** Sum AR by bucket
- **Interactivity:** Snapshot date filter, region/branch filters, click a bucket to open invoice detail
- **Justification:** Stacked bars make fixed aging bucket composition easy to compare across branches.
- **Optimization Tips:** Ensure aging buckets are materialized in the Gold fact; avoid recalculating bucket logic in dashboard SQL.

### 2.10 AR KPI Cards

- **Data Element:** Total AR Outstanding, AR 90+ %, DSO, Best Possible DSO
- **SQL Query:**
```sql
SELECT
  snapshot_date,
  region,
  branch_code,
  SUM(ar_outstanding_amount) AS total_ar_outstanding,
  CASE WHEN SUM(ar_outstanding_amount) = 0 THEN NULL
       ELSE 100 * SUM(CASE WHEN aging_bucket = '90+' THEN ar_outstanding_amount ELSE 0 END) / SUM(ar_outstanding_amount)
  END AS ar_90_plus_pct,
  CASE WHEN SUM(credit_sales_amount) = 0 THEN NULL
       ELSE (SUM(ar_outstanding_amount) / SUM(credit_sales_amount)) * MAX(days_in_period)
  END AS dso,
  CASE WHEN SUM(credit_sales_amount) = 0 THEN NULL
       ELSE (SUM(CASE WHEN aging_bucket IN ('0-30','31-60') THEN ar_outstanding_amount ELSE 0 END) / SUM(credit_sales_amount)) * MAX(days_in_period)
  END AS best_possible_dso
FROM gold.fct_ar_aging
WHERE snapshot_date = :snapshot_date
GROUP BY snapshot_date, region, branch_code;
```
- **Recommended LVDash Widget:** KPI cards
- **Data Fields:** `snapshot_date`, `region`, `branch_code`, `total_ar_outstanding`, `ar_90_plus_pct`, `dso`, `best_possible_dso`
- **Calculations:** Sum outstanding, AR 90+ %, DSO, Best Possible DSO
- **Interactivity:** Snapshot date and branch filters, drill to aging trend
- **Justification:** KPI cards are ideal for quickly surfacing collections health indicators.
- **Optimization Tips:** Store daily branch snapshot metrics in aggregate table and reuse for KPI and trend sections.

### 2.11 Collector / Customer / Invoice Aging Detail

- **Data Element:** Collections worklist with invoice detail
- **SQL Query:**
```sql
SELECT
  a.snapshot_date,
  a.region,
  a.branch_code,
  a.collector_id,
  a.customer_id,
  c.customer_name,
  a.invoice_id,
  a.due_date,
  a.days_past_due,
  a.aging_bucket,
  a.ar_outstanding_amount
FROM gold.fct_ar_aging a
JOIN gold.dim_customer c
  ON a.customer_id = c.customer_id
WHERE a.snapshot_date = :snapshot_date
  AND (:collector_id IS NULL OR a.collector_id = :collector_id)
ORDER BY a.collector_id, a.customer_id, a.days_past_due DESC;
```
- **Recommended LVDash Widget:** Table
- **Data Fields:** `snapshot_date`, `region`, `branch_code`, `collector_id`, `customer_id`, `customer_name`, `invoice_id`, `due_date`, `days_past_due`, `aging_bucket`, `ar_outstanding_amount`
- **Calculations:** Days past due, outstanding amount
- **Interactivity:** Drill-down region -> branch -> collector -> customer -> invoice, filter by aging bucket, export detail
- **Justification:** A detail table is necessary for collections actioning and drill-through validation.
- **Optimization Tips:** Push filters for `snapshot_date`, `collector_id`, and `aging_bucket` into the query, and use clustering/Z-ORDER on `collector_id`, `customer_id`, `invoice_id`.

### 2.12 Cash Application KPI Cards

- **Data Element:** Cash collected, CEI, average days to pay, % invoices paid on time, unapplied cash $ and %
- **SQL Query:**
```sql
SELECT
  period_start,
  region,
  branch_code,
  collector_id,
  SUM(payment_amount) AS cash_collected,
  CASE
    WHEN SUM(beginning_ar + credit_sales_amount - ending_current_ar) = 0 THEN NULL
    ELSE 100 * SUM(beginning_ar + credit_sales_amount - ending_total_ar)
         / SUM(beginning_ar + credit_sales_amount - ending_current_ar)
  END AS cei,
  AVG(days_to_pay) AS avg_days_to_pay,
  100 * AVG(CASE WHEN paid_on_time_flag = 1 THEN 1.0 ELSE 0.0 END) AS pct_invoices_paid_on_time,
  SUM(CASE WHEN match_status = 'UNAPPLIED' THEN payment_amount ELSE 0 END) AS unapplied_cash_amount,
  CASE WHEN SUM(payment_amount) = 0 THEN NULL
       ELSE 100 * SUM(CASE WHEN match_status = 'UNAPPLIED' THEN payment_amount ELSE 0 END) / SUM(payment_amount)
  END AS unapplied_cash_pct
FROM gold.fct_cash_application
GROUP BY period_start, region, branch_code, collector_id;
```
- **Recommended LVDash Widget:** KPI cards
- **Data Fields:** `period_start`, `region`, `branch_code`, `collector_id`, `cash_collected`, `cei`, `avg_days_to_pay`, `pct_invoices_paid_on_time`, `unapplied_cash_amount`, `unapplied_cash_pct`
- **Calculations:** CEI, average days to pay, paid on time %, unapplied cash metrics
- **Interactivity:** Period filters, collector selector, branch/region slicers, drill-through to payment detail
- **Justification:** KPI cards support fast performance review for collectors and collections leadership.
- **Optimization Tips:** Precompute CEI inputs during ETL because CEI window logic can be expensive and error-prone if recomputed ad hoc.

### 2.13 Collector Leaderboard

- **Data Element:** Ranked collector performance comparison
- **SQL Query:**
```sql
SELECT
  period_start,
  region,
  branch_code,
  collector_id,
  SUM(payment_amount) AS cash_collected,
  AVG(days_to_pay) AS avg_days_to_pay,
  CASE WHEN SUM(payment_amount) = 0 THEN NULL
       ELSE 100 * SUM(CASE WHEN match_status = 'UNAPPLIED' THEN payment_amount ELSE 0 END) / SUM(payment_amount)
  END AS unapplied_cash_pct,
  RANK() OVER (PARTITION BY period_start, region, branch_code ORDER BY SUM(payment_amount) DESC) AS collector_rank
FROM gold.fct_cash_application
GROUP BY period_start, region, branch_code, collector_id
ORDER BY period_start, collector_rank;
```
- **Recommended LVDash Widget:** Horizontal bar chart
- **Data Fields:** `period_start`, `region`, `branch_code`, `collector_id`, `cash_collected`, `avg_days_to_pay`, `unapplied_cash_pct`, `collector_rank`
- **Calculations:** Collector ranking, sum payments, average days to pay
- **Interactivity:** Region/branch/period filters, click collector to open detail table
- **Justification:** A ranked bar chart highlights relative collector performance better than a table alone.
- **Optimization Tips:** Materialize collector-period aggregates and avoid ranking across massive detail-level receipt tables during dashboard rendering.

### 2.14 Unapplied Cash Detail

- **Data Element:** Payments not applied to invoices
- **SQL Query:**
```sql
SELECT
  payment_date,
  region,
  branch_code,
  collector_id,
  customer_id,
  receipt_id,
  payment_method,
  payment_amount,
  match_status
FROM gold.fct_cash_application
WHERE match_status = 'UNAPPLIED'
  AND payment_date BETWEEN :start_date AND :end_date
ORDER BY payment_date DESC, payment_amount DESC;
```
- **Recommended LVDash Widget:** Table
- **Data Fields:** `payment_date`, `region`, `branch_code`, `collector_id`, `customer_id`, `receipt_id`, `payment_method`, `payment_amount`, `match_status`
- **Calculations:** Unapplied payment amount totals via widget aggregations
- **Interactivity:** Filter by payment method, collector, branch; drill to receipt detail
- **Justification:** Unapplied cash is an operational exception list and is best managed in a sortable detail table.
- **Optimization Tips:** Maintain a small exception dataset or view for unapplied cash rather than filtering a full cash fact repeatedly.

### 2.15 Customer Credit Risk KPI Cards

- **Data Element:** Customers over limit, suspended customers with exposure, credit utilization
- **SQL Query:**
```sql
SELECT
  snapshot_date,
  region,
  branch_code,
  COUNT(DISTINCT CASE WHEN credit_utilization_pct > 100 THEN customer_id END) AS customers_over_limit,
  SUM(CASE WHEN credit_utilization_pct > 100 THEN outstanding_ar_amount ELSE 0 END) AS over_limit_amount,
  COUNT(DISTINCT CASE WHEN customer_status = 'Suspended' AND (outstanding_ar_amount > 0 OR open_contract_count > 0) THEN customer_id END) AS suspended_with_exposure,
  AVG(credit_utilization_pct) AS avg_credit_utilization_pct
FROM gold.fct_credit_risk
GROUP BY snapshot_date, region, branch_code;
```
- **Recommended LVDash Widget:** KPI cards
- **Data Fields:** `snapshot_date`, `region`, `branch_code`, `customers_over_limit`, `over_limit_amount`, `suspended_with_exposure`, `avg_credit_utilization_pct`
- **Calculations:** Over-limit count and amount, suspended exposure count, average credit utilization
- **Interactivity:** Branch and customer status filters, snapshot date parameter, drill-through to customer details
- **Justification:** Credit leadership needs immediate counts and exposure values for risk monitoring, which KPI cards convey clearly.
- **Optimization Tips:** Snapshot the credit risk fact daily or weekly; do not derive customer exposure directly from AR detail in every dashboard query.

### 2.16 Credit Utilization Heatmap by Branch

- **Data Element:** Branch-level concentration of customer credit utilization
- **SQL Query:**
```sql
SELECT
  snapshot_date,
  region,
  branch_code,
  customer_id,
  customer_name,
  credit_utilization_pct
FROM gold.fct_credit_risk
WHERE snapshot_date = :snapshot_date;
```
- **Recommended LVDash Widget:** Heatmap
- **Data Fields:** `snapshot_date`, `region`, `branch_code`, `customer_id`, `customer_name`, `credit_utilization_pct`
- **Calculations:** Credit utilization percentage
- **Interactivity:** Region and branch filters, click heat cell to open customer detail
- **Justification:** A heatmap is effective for quickly spotting concentration of over-limit or near-limit accounts across branches.
- **Optimization Tips:** Limit displayed population to active customers with exposure; use a summarized branch/customer snapshot dataset.

### 2.17 Customer Exposure Detail

- **Data Element:** Customer-level risk detail with open exposure
- **SQL Query:**
```sql
SELECT
  snapshot_date,
  region,
  branch_code,
  customer_id,
  customer_name,
  customer_status,
  credit_terms,
  credit_limit,
  outstanding_ar_amount,
  ar_90_plus_amount,
  open_contract_count,
  open_contract_value,
  credit_utilization_pct,
  customer_since_date
FROM gold.fct_credit_risk
WHERE snapshot_date = :snapshot_date
ORDER BY credit_utilization_pct DESC, outstanding_ar_amount DESC;
```
- **Recommended LVDash Widget:** Table
- **Data Fields:** `snapshot_date`, `region`, `branch_code`, `customer_id`, `customer_name`, `customer_status`, `credit_terms`, `credit_limit`, `outstanding_ar_amount`, `ar_90_plus_amount`, `open_contract_count`, `open_contract_value`, `credit_utilization_pct`, `customer_since_date`
- **Calculations:** Credit utilization %, exposure totals, 90+ amount
- **Interactivity:** Search by customer name, filter by status and over-limit flag, drill-through to invoice/contract detail
- **Justification:** A detail table supports actionability for credit teams and is appropriate for sorted high-risk review.
- **Optimization Tips:** Mask or restrict PII fields according to role-based access; use row-level filtering by branch/region.

## 3. Overall Dashboard Design

### 3.1 Layout Suggestions

| Tab | Purpose | Suggested Layout |
|---|---|---|
| Executive Scorecard | Enterprise health overview | Top row KPI cards for overall and domain scores; middle row combo trend charts; bottom row risk leaderboard and driver table |
| Revenue & Billing | Revenue and billing analysis | Top KPI strip; middle left revenue trend; middle right equipment class mix; bottom detail table by sales rep/contract |
| AR Aging & Collections | Receivables monitoring and worklist | Top KPI cards; middle stacked aging chart by branch; bottom invoice detail table with collector/customer filters |
| Cash Application | Collector performance and unapplied cash | Top KPI cards; middle collector leaderboard; bottom unapplied cash detail table |
| Customer Credit Risk | Exposure monitoring | Top KPI cards; middle branch/customer heatmap; bottom customer exposure table |

### 3.2 Grid and Widget Sizing Guidance
- Use a 12-column grid-based Lakeview canvas.
- Reserve first row for 4 to 6 KPI cards with consistent sizing.
- Place trend and comparison charts in the second row with 6-column widths.
- Use full-width operational detail tables in the bottom section of each tab.
- Keep related filters pinned at the top or in a left-side filter panel for all tabs.
- Use tab-specific datasets only where needed; otherwise share common conformed datasets across tabs.

### 3.3 Color Scheme

| Use Case | Recommended Color |
|---|---|
| Positive / healthy | `#2E7D32` green |
| Warning / watchlist | `#F9A825` amber |
| Critical / overdue / over-limit | `#C62828` red |
| Neutral / supporting metrics | `#1565C0` blue |
| Background / secondary | `#ECEFF1` light gray |

Use semantic threshold coloring for KPI cards, especially for health score, AR 90+ %, CEI, unapplied cash %, and credit utilization.

### 3.4 Typography
- Use a clean sans-serif font such as Inter, Arial, or Databricks default UI font.
- KPI titles: medium weight, 14-16 px.
- KPI values: bold, 24-32 px.
- Chart labels: 11-12 px.
- Table text: 11-12 px with conditional formatting for risk columns.
- Keep naming short and business-friendly: "AR 90+ %", "Net Billings", "Suspended w/ Exposure".

## 4. Query Optimization and Performance Guidance

### 4.1 Recommended Gold Modeling Strategy
- Build Gold facts at the exact grains required for dashboarding rather than repeatedly joining Silver tables in Lakeview.
- Use conformed dimensions for branch, region, customer, date, employee, contract, and equipment class.
- Store snapshot facts for AR aging and credit risk to simplify trend analysis and audit reconciliation.
- Precompute derived metrics with stable business definitions where practical, especially aging bucket, days past due, billed rental days, paid-on-time flags, and credit utilization.

### 4.2 Lakeview and Databricks SQL Optimization

| Area | Recommendation |
|---|---|
| Dataset Reuse | Reuse a small number of curated datasets across widgets instead of issuing one raw query per widget |
| Caching | Enable Databricks SQL result caching and dashboard dataset reuse for common filters/date ranges |
| Materialization | Materialize daily and weekly aggregate tables for executive scorecards and collector leaderboards |
| Partitioning | Partition large facts by `date_day`, `snapshot_date`, or `payment_date` depending on access pattern |
| Z-Ordering | Z-ORDER large Delta tables on common filter/join columns such as `region`, `branch_code`, `customer_id`, `collector_id`, `invoice_id` |
| SQL Warehouse Sizing | Use a right-sized SQL warehouse; avoid oversizing for lightweight scorecards and undersizing for high-concurrency executive access |
| Filter Pushdown | Always parameterize date and branch/region filters to reduce scanned data |
| Joins | Prefer Gold tables to avoid repeated large joins in dashboard queries |
| Incremental Loads | Refresh scorecard aggregates incrementally by date to reduce rebuild cost |
| Data Skipping | Maintain compact files and optimized clustering to improve Delta data skipping |

### 4.3 Suggested Physical Optimizations by Table

| Table | Recommended Optimization |
|---|---|
| `gold.fct_rental_revenue` | Partition by `date_day`; Z-ORDER by `region`, `branch_code`, `sales_rep_id`, `contract_id` |
| `gold.fct_ar_aging` | Partition by `snapshot_date`; Z-ORDER by `branch_code`, `collector_id`, `customer_id`, `invoice_id` |
| `gold.fct_cash_application` | Partition by `payment_date`; Z-ORDER by `branch_code`, `collector_id`, `customer_id`, `receipt_id` |
| `gold.fct_credit_risk` | Partition by `snapshot_date`; Z-ORDER by `branch_code`, `customer_id`, `customer_status` |
| `gold.agg_exec_scorecard_daily` | Partition by `date_day`; cache for dashboard usage |
| `gold.agg_exec_scorecard_weekly` | Partition by `week_start`; cache for leadership reviews |

## 5. Interactive Elements

### 5.1 Recommended Dashboard Filters and Parameters

| Feature Type | Name | Scope | Purpose |
|---|---|---|---|
| Filter | Region | Global | Narrow dashboard to region |
| Filter | Branch | Global | Narrow to branch within selected region |
| Filter | Date Range | Global | Control time period across trend and KPI widgets |
| Parameter | Period Grain | Global | Switch between day/week/month/quarter views |
| Filter | Sales Rep | Revenue tab | Focus revenue and contract views |
| Filter | Collector | AR and Cash tabs | Focus collections performance and worklists |
| Filter | Equipment Class | Revenue tab | Analyze revenue mix |
| Filter | Customer Status | Credit Risk tab | Analyze active vs suspended customers |
| Filter | Aging Bucket | AR tab | Isolate risk buckets |
| Filter | Match Status | Cash tab | Compare applied vs unapplied cash |
| Parameter | Top N | Executive / leaderboard tabs | Show top risk branches or collectors |
| Drill-down | Network -> Region -> Branch -> Sales Rep/Collector -> Customer/Contract/Invoice | Multiple tabs | Hierarchical navigation |
| Drill-through | KPI/Chart to detailed table | Multiple tabs | Move from summary metrics to operational records |
| Cross-filter | Click chart element to filter peer widgets | Multiple tabs | Interactive exploration |

### 5.2 Interactivity Features Table

| Widget / Section | Filters | Parameters | Cross-Filters | Drill-Through / Drill Up-Down |
|---|---|---|---|---|
| Executive KPI cards | Region, Branch, Date Range | Period Grain | Yes, to trend and driver widgets | Drill-through to Revenue, AR, Cash, Credit tabs; drill-down Network -> Region -> Branch |
| Revenue Trend | Region, Branch, Equipment Class | Period Grain | Yes | Drill-up Week -> Month -> Quarter; drill-down Branch -> Sales Rep |
| AR Aging Stacked Bar | Region, Branch, Snapshot Date | Top N | Yes | Drill-down Branch -> Collector -> Customer -> Invoice |
| Collector Leaderboard | Region, Branch, Period | Top N | Yes | Drill-through to collector payment detail; drill-up Week -> Month |
| Credit Heatmap | Region, Branch, Snapshot Date | Over-limit threshold | Yes | Drill-through to customer exposure detail |
| Detail Tables | All relevant tab filters | Export or pagination controls | Limited | Drill-through to invoice/contract/receipt detail records |

## 6. Validation and Data Quality Recommendations

| Validation Area | Recommendation |
|---|---|
| Revenue Reconciliation | Ensure net billings = gross billings - credit memos and validate invoice signs by invoice type |
| Aging Reconciliation | Sum of aging buckets must reconcile to total AR outstanding per snapshot |
| Cash Application Integrity | Applied payment totals must not exceed invoice amount; unapplied cash must be retained as explicit exceptions |
| Credit Risk Integrity | Handle null or zero credit limit safely in utilization calculations |
| Health Score Logic | Validate weighted domain totals produce the overall score and remain within 0-100 |
| Duplicate Prevention | Enforce business-grain deduplication in Gold tables before dashboard exposure |
| Freshness Monitoring | Surface latest successful load time from `silver.si_process_audit` or Gold audit metadata |

## 7. Potential Pitfalls and Mitigations

| Potential Pitfall | Impact | Mitigation |
|---|---|---|
| Repeated joins from Silver directly in dashboards | Slow performance and inconsistent business logic | Build curated Gold fact and aggregate tables |
| Large detail datasets reused by many widgets | High latency and excessive warehouse load | Split summary and detail datasets; use drill-through for detail |
| Skewed branch or customer volumes | Shuffle-heavy queries and poor responsiveness | Repartition carefully and optimize Gold tables with Z-ORDER |
| Recomputing CEI and DSO in many widgets | Logic drift and performance issues | Centralize metric definitions in Gold layer or shared SQL views |
| Missing date filter defaults | Large scans and slow first load | Default to recent period such as current month or latest snapshot |
| Overloaded single tab with too many visuals | Poor usability | Organize visuals into business-focused tabs |
| Uncontrolled access to customer-level data | Security risk | Apply row-level security by branch/region and restrict executive views to aggregates where required |
| Oversized or undersized SQL warehouse | Cost or performance problems | Size warehouse based on concurrency and dataset complexity; monitor usage |

## 8. Recommended Databricks SQL Views for Reuse

| View Name | Purpose |
|---|---|
| `vw_exec_scorecard_metrics` | Consolidated branch/date scorecard metrics for KPI and trends |
| `vw_revenue_billing_summary` | Revenue and billing metrics by branch, sales rep, equipment class, and period |
| `vw_ar_aging_summary` | AR aging metrics by snapshot, branch, bucket, and customer |
| `vw_cash_application_summary` | Cash collected, CEI inputs, unapplied cash, collector performance |
| `vw_credit_risk_summary` | Customer exposure, over-limit, suspended exposure, credit utilization |

## 9. Final Recommendation Summary
- Use a layered design: Silver for conformed operational data, Gold for dashboard-ready facts and aggregates, and Lakeview for business consumption.
- Prioritize KPI cards, line/combo charts, stacked bars, heatmaps, and detail tables aligned to each report objective.
- Build separate tabs for Executive, Revenue, AR, Cash Application, and Credit Risk to keep the dashboard intuitive.
- Maximize reuse of curated datasets and precomputed business metrics to improve Lakeview responsiveness and governance.
- Implement strong filter, drill-through, and row-level security patterns to support executives, regional leaders, branch managers, collectors, and finance stakeholders.

---

## Output URL (Clickable Hyperlinks)
[Databricks_Dashboard_Visuals_Recommender_1.md](https://github.com/DIAscendion/Databricks_Dashboard/blob/main/DI_Databricks_Dashboard_Visuals_Recommender/Databricks_Dashboard_Visuals_Recommender_1.md)

Print the URL : "https://github.com/DIAscendion/Databricks_Dashboard/tree/main/DI_Databricks_Dashboard_Visuals_Recommender"
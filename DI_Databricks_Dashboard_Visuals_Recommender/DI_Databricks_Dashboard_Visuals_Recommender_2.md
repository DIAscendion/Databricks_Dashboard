_____________________________________________
## *Author*: AAVA
## *Created on*:   
## *Description*: Databricks Lakeview Dashboard (LVDash) visuals, SQL, data model, interactivity, and optimization recommendations for Rental Revenue-to-Cash executive, billing, AR aging, collections, and credit risk reporting.
## *Version*: 2
## *Updated on*: 
_____________________________________________

## Summary

This document analyzes the Rental Revenue-to-Cash reporting requirements together with the Silver physical data model and recommends Databricks SQL queries, reusable analytical datasets, and Lakeview Dashboard visuals for executive scorecards, rental revenue and billing, AR aging, cash application, collector performance, and customer credit risk insights. The recommendations are aligned to the available Silver layer entities including rental contracts, invoices, cash receipts, customer master, branch employee, data quality errors, and process audit data, while anticipating Gold-level reusable analytical views such as revenue, AR aging, cash application, and credit risk facts.

Print the URL : "https://github.com/DIAscendion/Databricks_Dashboard/tree/main/DI_Databricks_Dashboard_Visuals_Recommender"

---

## 1. Source and Analytical Model Understanding

### 1.1 Identified Source Tables

| Source Layer | Table | Purpose |
|---|---|---|
| Silver | `silver.si_rental_contracts` | Contract-level rental context including branch, sales rep, equipment class, start/end dates, status, and daily rate |
| Silver | `silver.si_invoices` | Billing and invoice-level revenue data including invoice date, due date, invoice amount, tax, type, and customer linkage |
| Silver | `silver.si_cash_receipts` | Payment and cash receipt data including receipt date, payment amount, payment method, and invoice/customer linkage |
| Silver | `silver.si_customer_master` | Customer profile, credit terms, credit limit, status, and tenure data |
| Silver | `silver.si_branch_employee` | Sales rep, collector, branch, and region assignment data |
| Silver | `silver.si_data_quality_errors` | Validation and rejection events for monitoring data issues impacting reporting confidence |
| Silver | `silver.si_process_audit` | Pipeline execution audit, record counts, and processing status |

### 1.2 Recommended Reusable Analytical Views / Gold Datasets

| Recommended Gold Dataset / View | Base Tables | Purpose |
|---|---|---|
| `gold.fct_rental_revenue` | invoices + rental contracts + customer + branch employee | Revenue, billings, net billings, ADR, contract billing analytics |
| `gold.fct_ar_aging` | invoices + cash receipts + customer + branch employee | Open AR, aging buckets, days past due, DSO support |
| `gold.fct_cash_application` | cash receipts + invoices + customer + branch employee | Collector performance, cash application, unapplied cash, days to pay |
| `gold.fct_credit_risk` | customer + open AR + open contracts | Credit utilization, over-limit, suspended exposure |
| `gold.fct_rev_to_cash_health` | aggregated outputs from revenue, AR, cash application, and credit risk facts | Executive scorecard and weighted health scoring |
| `gold.dim_date` | calendar reference | Daily, weekly, monthly, quarterly rollups |
| `gold.dim_branch_org` | branch/region hierarchy | Network → Region → Branch drill paths |

### 1.3 Key Joins

| Left Table | Join Key | Right Table | Business Purpose |
|---|---|---|---|
| `silver.si_invoices` | `contract_id` | `silver.si_rental_contracts` | Tie billings to contract, branch, sales rep, and equipment class |
| `silver.si_invoices` | `customer_id` | `silver.si_customer_master` | Add customer attributes and credit dimensions |
| `silver.si_cash_receipts` | `invoice_id` | `silver.si_invoices` | Match receipts to invoices for AR and cash application analytics |
| `silver.si_cash_receipts` | `customer_id` | `silver.si_customer_master` | Support customer-level collections and credit reporting |
| `silver.si_rental_contracts` | `sales_rep_id = employee_id` | `silver.si_branch_employee` | Assign contracts and billings to sales reps |
| `silver.si_branch_employee` | `branch_code` | `silver.si_rental_contracts` / `silver.si_invoices` | Add branch and regional reporting hierarchy |
| `silver.si_branch_employee` | `collector_id` | `gold.fct_ar_aging` / `gold.fct_cash_application` | Assign collections responsibility |

### 1.4 Required Transformations, Filters, and Calculations

| Area | Requirement |
|---|---|
| Revenue | Split gross billings vs credit memos using `invoice_type`; compute net billings |
| AR Aging | Derive `days_past_due` from snapshot date minus `due_date`; assign aging buckets 0-30 / 31-60 / 61-90 / 90+ |
| Cash App | Derive applied vs unapplied using `match_status`; calculate days to pay from invoice date to receipt date |
| Credit Risk | Compute outstanding AR vs credit limit and identify suspended customers with exposure |
| Health Score | Normalize revenue, AR, cash application, and credit risk metrics into weighted 0-100 composite scores |
| Filters | Exclude invalid dates, null mandatory IDs, negative invalid amounts, and duplicate business grains |

---

## 2. Visual Recommendations

### 2.1 Revenue-to-Cash Executive Scorecard - Overall Health
- **Data Element:** Overall Revenue-to-Cash Health Score by day/week/month and branch hierarchy
- **SQL Query:**
  ```sql
  WITH revenue_score AS (
    SELECT branch_code,
           date_trunc('week', invoice_date) AS period_start,
           LEAST(100, GREATEST(0, SUM(CASE WHEN invoice_type <> 'CREDIT_MEMO' THEN invoice_amount ELSE 0 END) / NULLIF(100000,0) * 100)) AS revenue_domain_score
    FROM silver.si_invoices
    WHERE invoice_date IS NOT NULL
    GROUP BY branch_code, date_trunc('week', invoice_date)
  ),
  ar_score AS (
    SELECT i.branch_code,
           date_trunc('week', current_date()) AS period_start,
           100 - LEAST(100,
             SUM(CASE WHEN datediff(current_date(), i.due_date) > 90 THEN i.invoice_amount ELSE 0 END)
             / NULLIF(SUM(i.invoice_amount),0) * 100) AS ar_domain_score
    FROM silver.si_invoices i
    WHERE i.due_date IS NOT NULL
    GROUP BY i.branch_code
  ),
  cash_score AS (
    SELECT be.branch_code,
           date_trunc('week', cr.receipt_date) AS period_start,
           LEAST(100, GREATEST(0,
             100 - (SUM(CASE WHEN coalesce(cr.match_status,'UNAPPLIED') = 'UNAPPLIED' THEN cr.payment_amount ELSE 0 END)
             / NULLIF(SUM(cr.payment_amount),0) * 100))) AS cash_domain_score
    FROM silver.si_cash_receipts cr
    LEFT JOIN silver.si_invoices i ON cr.invoice_id = i.invoice_id
    LEFT JOIN silver.si_branch_employee be ON i.customer_id IS NOT NULL AND be.branch_code = i.branch_code
    WHERE cr.receipt_date IS NOT NULL
    GROUP BY be.branch_code, date_trunc('week', cr.receipt_date)
  ),
  credit_score AS (
    SELECT rc.branch_code,
           date_trunc('week', current_date()) AS period_start,
           100 - LEAST(100,
             SUM(CASE WHEN cm.credit_limit > 0 THEN greatest(open_ar.outstanding_ar - cm.credit_limit, 0) ELSE open_ar.outstanding_ar END)
             / NULLIF(SUM(open_ar.outstanding_ar),0) * 100) AS credit_domain_score
    FROM silver.si_customer_master cm
    LEFT JOIN (
      SELECT customer_id, SUM(invoice_amount) AS outstanding_ar
      FROM silver.si_invoices
      GROUP BY customer_id
    ) open_ar ON cm.customer_id = open_ar.customer_id
    LEFT JOIN silver.si_rental_contracts rc ON cm.customer_id = rc.customer_id
    GROUP BY rc.branch_code
  )
  SELECT coalesce(r.branch_code, a.branch_code, c.branch_code, cr.branch_code) AS branch_code,
         coalesce(r.period_start, a.period_start, c.period_start, cr.period_start) AS period_start,
         ROUND((coalesce(r.revenue_domain_score,0) * 0.30)
             + (coalesce(a.ar_domain_score,0) * 0.30)
             + (coalesce(c.cash_domain_score,0) * 0.20)
             + (coalesce(cr.credit_domain_score,0) * 0.20), 2) AS overall_health_score,
         ROUND(coalesce(r.revenue_domain_score,0),2) AS revenue_domain_score,
         ROUND(coalesce(a.ar_domain_score,0),2) AS ar_domain_score,
         ROUND(coalesce(c.cash_domain_score,0),2) AS cash_domain_score,
         ROUND(coalesce(cr.credit_domain_score,0),2) AS credit_domain_score
  FROM revenue_score r
  FULL OUTER JOIN ar_score a
    ON r.branch_code = a.branch_code AND r.period_start = a.period_start
  FULL OUTER JOIN cash_score c
    ON coalesce(r.branch_code,a.branch_code) = c.branch_code
   AND coalesce(r.period_start,a.period_start) = c.period_start
  FULL OUTER JOIN credit_score cr
    ON coalesce(r.branch_code,a.branch_code,c.branch_code) = cr.branch_code
   AND coalesce(r.period_start,a.period_start,c.period_start) = cr.period_start;
  ```
- **Recommended LVDash Widget:** Counter/KPI card
- **Data Fields:** `branch_code`, `period_start`, `overall_health_score`, `revenue_domain_score`, `ar_domain_score`, `cash_domain_score`, `credit_domain_score`
- **Calculations:** Weighted composite score; component domain scores; week-level rollup
- **Interactivity:** Region, branch, period type filters; drill-through to domain-specific tabs; drill up/down Network → Region → Branch
- **Justification:** The executive health score is the most important top-level metric and should be displayed as a large KPI card with target thresholds.
- **Optimization Tips:** Materialize `gold.fct_rev_to_cash_health`; compute normalization offline; reuse one dataset for all top scorecards.

### 2.2 Executive Domain Score Breakdown
- **Data Element:** Revenue, AR, Cash App, and Credit Risk component scores
- **SQL Query:**
  ```sql
  SELECT branch_code,
         period_start,
         revenue_domain_score,
         ar_domain_score,
         cash_domain_score,
         credit_domain_score
  FROM gold.fct_rev_to_cash_health
  ```
- **Recommended LVDash Widget:** Bar chart
- **Data Fields:** `branch_code`, `period_start`, `revenue_domain_score`, `ar_domain_score`, `cash_domain_score`, `credit_domain_score`
- **Calculations:** Component domain scores by selected branch/region/period
- **Interactivity:** Cross-filter from branch selector; drill-through to supporting reports; period toggle parameter
- **Justification:** Side-by-side bars make score comparisons across domains intuitive and quickly expose the weakest domain.
- **Optimization Tips:** Reshape wide scores into long format in a reusable view for cleaner charting; cache the dataset query result.

### 2.3 Revenue Trend
- **Data Element:** Rental Revenue trend over time
- **SQL Query:**
  ```sql
  SELECT date_trunc('week', i.invoice_date) AS period_start,
         rc.branch_code,
         be.region,
         SUM(CASE WHEN i.invoice_type <> 'CREDIT_MEMO' THEN i.invoice_amount ELSE 0 END) AS rental_revenue
  FROM silver.si_invoices i
  LEFT JOIN silver.si_rental_contracts rc ON i.contract_id = rc.contract_id
  LEFT JOIN silver.si_branch_employee be ON rc.branch_code = be.branch_code
  WHERE i.invoice_date IS NOT NULL
  GROUP BY date_trunc('week', i.invoice_date), rc.branch_code, be.region
  ORDER BY period_start
  ```
- **Recommended LVDash Widget:** Line chart
- **Data Fields:** `period_start`, `branch_code`, `region`, `rental_revenue`
- **Calculations:** Sum of non-credit memo invoice amounts by period
- **Interactivity:** Date range filter; region/branch filter; drill-down Region → Branch → Sales Rep where available
- **Justification:** Trends over time are best represented with a line chart, especially for weekly or monthly revenue monitoring.
- **Optimization Tips:** Partition invoice facts by `invoice_date`; aggregate to weekly/monthly Gold view for dashboard use.

### 2.4 DSO Trend
- **Data Element:** Days Sales Outstanding trend
- **SQL Query:**
  ```sql
  WITH ar AS (
    SELECT date_trunc('month', current_date()) AS period_start,
           rc.branch_code,
           SUM(i.invoice_amount) AS total_ar_outstanding
    FROM silver.si_invoices i
    LEFT JOIN silver.si_rental_contracts rc ON i.contract_id = rc.contract_id
    GROUP BY rc.branch_code
  ),
  credit_sales AS (
    SELECT date_trunc('month', invoice_date) AS period_start,
           rc.branch_code,
           SUM(CASE WHEN invoice_type <> 'CREDIT_MEMO' THEN invoice_amount ELSE 0 END) AS total_credit_sales,
           day(last_day(invoice_date)) AS days_in_period
    FROM silver.si_invoices i
    LEFT JOIN silver.si_rental_contracts rc ON i.contract_id = rc.contract_id
    GROUP BY date_trunc('month', invoice_date), rc.branch_code, day(last_day(invoice_date))
  )
  SELECT c.period_start,
         c.branch_code,
         ROUND((a.total_ar_outstanding / NULLIF(c.total_credit_sales,0)) * c.days_in_period, 2) AS dso
  FROM credit_sales c
  LEFT JOIN ar a ON c.branch_code = a.branch_code
  ```
- **Recommended LVDash Widget:** Line chart
- **Data Fields:** `period_start`, `branch_code`, `dso`
- **Calculations:** `DSO = (Total AR Outstanding / Total Credit Sales) * Days in Period`
- **Interactivity:** Date, region, branch filters; drill-through to AR aging details
- **Justification:** DSO is monitored over time and is most understandable as a trend line against target bands.
- **Optimization Tips:** Precompute monthly DSO in `gold.fct_ar_aging`; avoid repeated expensive denominator calculations in dashboard runtime queries.

### 2.5 AR Aging by Bucket
- **Data Element:** Total AR Outstanding split by aging bucket and branch
- **SQL Query:**
  ```sql
  WITH ar_snapshot AS (
    SELECT current_date() AS snapshot_date,
           i.invoice_id,
           i.customer_id,
           rc.branch_code,
           be.region,
           datediff(current_date(), i.due_date) AS days_past_due,
           i.invoice_amount AS outstanding_amount
    FROM silver.si_invoices i
    LEFT JOIN silver.si_rental_contracts rc ON i.contract_id = rc.contract_id
    LEFT JOIN silver.si_branch_employee be ON rc.branch_code = be.branch_code
    WHERE i.due_date IS NOT NULL
  )
  SELECT snapshot_date,
         region,
         branch_code,
         CASE
           WHEN days_past_due <= 30 THEN '0-30'
           WHEN days_past_due BETWEEN 31 AND 60 THEN '31-60'
           WHEN days_past_due BETWEEN 61 AND 90 THEN '61-90'
           ELSE '90+'
         END AS aging_bucket,
         SUM(outstanding_amount) AS total_ar_outstanding
  FROM ar_snapshot
  GROUP BY snapshot_date, region, branch_code,
           CASE
             WHEN days_past_due <= 30 THEN '0-30'
             WHEN days_past_due BETWEEN 31 AND 60 THEN '31-60'
             WHEN days_past_due BETWEEN 61 AND 90 THEN '61-90'
             ELSE '90+'
           END
  ```
- **Recommended LVDash Widget:** Stacked bar chart
- **Data Fields:** `snapshot_date`, `region`, `branch_code`, `aging_bucket`, `total_ar_outstanding`
- **Calculations:** Aging bucket assignment; sum of outstanding AR
- **Interactivity:** Snapshot date filter; region/branch/collector filters; drill-through to invoice list
- **Justification:** A stacked bar clearly shows composition of AR by aging severity while preserving branch comparison.
- **Optimization Tips:** Store daily AR snapshots in `gold.fct_ar_aging`; Z-ORDER by `branch_code`, `customer_id`, `aging_bucket`.

### 2.6 AR 90+ Percentage Leaderboard
- **Data Element:** AR 90+ % by branch or customer
- **SQL Query:**
  ```sql
  WITH ar_buckets AS (
    SELECT rc.branch_code,
           i.customer_id,
           SUM(i.invoice_amount) AS total_ar,
           SUM(CASE WHEN datediff(current_date(), i.due_date) > 90 THEN i.invoice_amount ELSE 0 END) AS ar_90_plus
    FROM silver.si_invoices i
    LEFT JOIN silver.si_rental_contracts rc ON i.contract_id = rc.contract_id
    WHERE i.due_date IS NOT NULL
    GROUP BY rc.branch_code, i.customer_id
  )
  SELECT branch_code,
         customer_id,
         ROUND(ar_90_plus / NULLIF(total_ar,0) * 100, 2) AS ar_90_plus_pct,
         total_ar,
         ar_90_plus
  FROM ar_buckets
  ORDER BY ar_90_plus_pct DESC, ar_90_plus DESC
  ```
- **Recommended LVDash Widget:** Table
- **Data Fields:** `branch_code`, `customer_id`, `ar_90_plus_pct`, `total_ar`, `ar_90_plus`
- **Calculations:** `AR 90+ % = AR in 90+ / Total AR * 100`
- **Interactivity:** Sortable columns; drill-through to customer invoice details; branch and region filters
- **Justification:** A ranked table is ideal for highlighting the worst branches or customers requiring intervention.
- **Optimization Tips:** Persist this as a narrow leaderboard dataset; conditional formatting for high-risk rows.

### 2.7 Rental Revenue & Billing Performance
- **Data Element:** Gross billings, credit memos, and net billings by branch, sales rep, and equipment class
- **SQL Query:**
  ```sql
  SELECT date_trunc('month', i.invoice_date) AS period_start,
         be.region,
         rc.branch_code,
         rc.sales_rep_id,
         rc.equipment_class,
         SUM(CASE WHEN i.invoice_type <> 'CREDIT_MEMO' THEN i.invoice_amount ELSE 0 END) AS gross_billings,
         SUM(CASE WHEN i.invoice_type = 'CREDIT_MEMO' THEN abs(i.invoice_amount) ELSE 0 END) AS credit_memos,
         SUM(CASE WHEN i.invoice_type <> 'CREDIT_MEMO' THEN i.invoice_amount ELSE 0 END)
           - SUM(CASE WHEN i.invoice_type = 'CREDIT_MEMO' THEN abs(i.invoice_amount) ELSE 0 END) AS net_billings,
         COUNT(DISTINCT i.invoice_id) AS invoice_count
  FROM silver.si_invoices i
  LEFT JOIN silver.si_rental_contracts rc ON i.contract_id = rc.contract_id
  LEFT JOIN silver.si_branch_employee be ON rc.branch_code = be.branch_code
  WHERE i.invoice_date IS NOT NULL
  GROUP BY date_trunc('month', i.invoice_date), be.region, rc.branch_code, rc.sales_rep_id, rc.equipment_class
  ```
- **Recommended LVDash Widget:** Combo chart
- **Data Fields:** `period_start`, `region`, `branch_code`, `sales_rep_id`, `equipment_class`, `gross_billings`, `credit_memos`, `net_billings`, `invoice_count`
- **Calculations:** Gross billings, credit memo amount, net billings, invoice count
- **Interactivity:** Region/branch/sales rep/equipment class filters; drill-down Branch → Sales Rep → Contract
- **Justification:** Combo charts allow net billings trends while overlaying credit memo activity for billing quality analysis.
- **Optimization Tips:** Materialize monthly revenue summaries; partition by period and cluster by branch/sales rep.

### 2.8 Average Daily Rate and Revenue per Contract
- **Data Element:** ADR and Revenue per Contract by period and branch
- **SQL Query:**
  ```sql
  WITH contract_invoice AS (
    SELECT date_trunc('month', i.invoice_date) AS period_start,
           rc.branch_code,
           SUM(CASE WHEN i.invoice_type <> 'CREDIT_MEMO' THEN i.invoice_amount ELSE 0 END) AS rental_revenue,
           COUNT(DISTINCT rc.contract_id) AS contract_count,
           SUM(greatest(datediff(rc.contract_end_date, rc.contract_start_date), 1)) AS billed_rental_days
    FROM silver.si_invoices i
    LEFT JOIN silver.si_rental_contracts rc ON i.contract_id = rc.contract_id
    WHERE rc.contract_status <> 'Cancelled'
    GROUP BY date_trunc('month', i.invoice_date), rc.branch_code
  )
  SELECT period_start,
         branch_code,
         ROUND(rental_revenue / NULLIF(billed_rental_days,0), 2) AS adr,
         ROUND(rental_revenue / NULLIF(contract_count,0), 2) AS revenue_per_contract
  FROM contract_invoice
  ```
- **Recommended LVDash Widget:** Dual-axis combo chart
- **Data Fields:** `period_start`, `branch_code`, `adr`, `revenue_per_contract`
- **Calculations:** `ADR`, `Revenue per Contract`
- **Interactivity:** Region/branch/date filters; drill-through to contract detail table
- **Justification:** Two related but differently scaled metrics are effectively compared in a dual-axis chart.
- **Optimization Tips:** Precalculate billed rental days in Gold to avoid repeated date arithmetic.

### 2.9 Cash Application and Collector Performance
- **Data Element:** Cash collected, CEI, average days to pay, on-time payment % by collector
- **SQL Query:**
  ```sql
  WITH payment_base AS (
    SELECT date_trunc('month', cr.receipt_date) AS period_start,
           be.region,
           be.collector_id,
           rc.branch_code,
           SUM(cr.payment_amount) AS cash_collected,
           AVG(datediff(cr.receipt_date, i.invoice_date)) AS avg_days_to_pay,
           SUM(CASE WHEN cr.receipt_date <= i.due_date THEN 1 ELSE 0 END) * 100.0 / NULLIF(COUNT(i.invoice_id),0) AS invoices_paid_on_time_pct,
           SUM(CASE WHEN coalesce(cr.match_status, 'UNAPPLIED') = 'UNAPPLIED' THEN cr.payment_amount ELSE 0 END) AS unapplied_cash_amount,
           SUM(cr.payment_amount) AS total_payments
    FROM silver.si_cash_receipts cr
    LEFT JOIN silver.si_invoices i ON cr.invoice_id = i.invoice_id
    LEFT JOIN silver.si_rental_contracts rc ON i.contract_id = rc.contract_id
    LEFT JOIN silver.si_branch_employee be ON rc.branch_code = be.branch_code
    GROUP BY date_trunc('month', cr.receipt_date), be.region, be.collector_id, rc.branch_code
  )
  SELECT period_start,
         region,
         branch_code,
         collector_id,
         cash_collected,
         ROUND(avg_days_to_pay, 2) AS avg_days_to_pay,
         ROUND(invoices_paid_on_time_pct, 2) AS invoices_paid_on_time_pct,
         ROUND(unapplied_cash_amount / NULLIF(total_payments,0) * 100, 2) AS unapplied_cash_pct
  FROM payment_base
  ```
- **Recommended LVDash Widget:** Bar chart for leaderboard plus KPI cards
- **Data Fields:** `period_start`, `region`, `branch_code`, `collector_id`, `cash_collected`, `avg_days_to_pay`, `invoices_paid_on_time_pct`, `unapplied_cash_pct`
- **Calculations:** Cash collected, average days to pay, on-time payment %, unapplied cash %
- **Interactivity:** Region/branch/collector filters; drill-through to payment detail; week/month parameter
- **Justification:** A ranked bar chart is ideal for comparing collectors, while KPI cards can summarize portfolio-wide performance.
- **Optimization Tips:** Build a collector-period aggregated Gold table; cache dashboard datasets for repeated manager access.

### 2.10 Unapplied Cash Detail
- **Data Element:** Unapplied cash detail by receipt, customer, branch, and payment method
- **SQL Query:**
  ```sql
  SELECT cr.receipt_id,
         cr.receipt_date,
         cr.customer_id,
         cm.customer_name,
         rc.branch_code,
         be.region,
         cr.payment_method,
         cr.payment_amount AS unapplied_cash_amount,
         coalesce(cr.match_status, 'UNAPPLIED') AS match_status
  FROM silver.si_cash_receipts cr
  LEFT JOIN silver.si_customer_master cm ON cr.customer_id = cm.customer_id
  LEFT JOIN silver.si_invoices i ON cr.invoice_id = i.invoice_id
  LEFT JOIN silver.si_rental_contracts rc ON i.contract_id = rc.contract_id
  LEFT JOIN silver.si_branch_employee be ON rc.branch_code = be.branch_code
  WHERE coalesce(cr.match_status, 'UNAPPLIED') = 'UNAPPLIED'
  ORDER BY cr.receipt_date DESC
  ```
- **Recommended LVDash Widget:** Table
- **Data Fields:** `receipt_id`, `receipt_date`, `customer_id`, `customer_name`, `branch_code`, `region`, `payment_method`, `unapplied_cash_amount`, `match_status`
- **Calculations:** Unapplied cash amount by receipt
- **Interactivity:** Payment method, branch, region filters; drill-through to customer/payment investigation
- **Justification:** Operational exception management is best handled through a searchable detail table.
- **Optimization Tips:** Restrict to unapplied rows in a dedicated dataset; paginate or limit default results for performance.

### 2.11 Customer Credit Risk Overview
- **Data Element:** Credit utilization %, customers over limit, suspended customers with open exposure
- **SQL Query:**
  ```sql
  WITH open_ar AS (
    SELECT customer_id,
           SUM(invoice_amount) AS outstanding_ar,
           SUM(CASE WHEN datediff(current_date(), due_date) > 90 THEN invoice_amount ELSE 0 END) AS ar_90_plus_amount
    FROM silver.si_invoices
    GROUP BY customer_id
  ),
  open_contracts AS (
    SELECT customer_id,
           COUNT(DISTINCT contract_id) AS open_contract_count,
           SUM(daily_rate) AS open_contract_value
    FROM silver.si_rental_contracts
    WHERE contract_status = 'Open'
    GROUP BY customer_id
  )
  SELECT cm.customer_id,
         cm.customer_name,
         cm.customer_status,
         cm.credit_terms,
         cm.credit_limit,
         coalesce(ar.outstanding_ar,0) AS outstanding_ar,
         coalesce(ar.ar_90_plus_amount,0) AS ar_90_plus_amount,
         coalesce(oc.open_contract_count,0) AS open_contract_count,
         coalesce(oc.open_contract_value,0) AS open_contract_value,
         ROUND(coalesce(ar.outstanding_ar,0) / NULLIF(cm.credit_limit,0) * 100, 2) AS credit_utilization_pct,
         CASE WHEN coalesce(ar.outstanding_ar,0) > cm.credit_limit THEN 'Over Limit' ELSE 'Within Limit' END AS limit_status,
         CASE WHEN cm.customer_status = 'Suspended'
                    AND (coalesce(ar.outstanding_ar,0) > 0 OR coalesce(oc.open_contract_count,0) > 0)
              THEN 'Suspended with Exposure' ELSE 'Normal' END AS exposure_flag
  FROM silver.si_customer_master cm
  LEFT JOIN open_ar ar ON cm.customer_id = ar.customer_id
  LEFT JOIN open_contracts oc ON cm.customer_id = oc.customer_id
  ```
- **Recommended LVDash Widget:** Heatmap or table with conditional formatting
- **Data Fields:** `customer_id`, `customer_name`, `customer_status`, `credit_terms`, `credit_limit`, `outstanding_ar`, `ar_90_plus_amount`, `open_contract_count`, `open_contract_value`, `credit_utilization_pct`, `limit_status`, `exposure_flag`
- **Calculations:** Credit utilization %, over-limit flag, suspended exposure flag
- **Interactivity:** Region/branch/customer status filters; drill-through to invoice and contract details
- **Justification:** Credit exposure requires exception-focused highlighting, and conditional formatting or heatmap encoding quickly surfaces the riskiest customers.
- **Optimization Tips:** Materialize daily credit risk snapshots; protect against divide-by-zero with `NULLIF` and prevalidated credit limits.

### 2.12 Credit Utilization by Branch
- **Data Element:** Average customer credit utilization by branch
- **SQL Query:**
  ```sql
  WITH customer_risk AS (
    SELECT rc.branch_code,
           cm.customer_id,
           coalesce(SUM(i.invoice_amount),0) AS outstanding_ar,
           cm.credit_limit
    FROM silver.si_customer_master cm
    LEFT JOIN silver.si_invoices i ON cm.customer_id = i.customer_id
    LEFT JOIN silver.si_rental_contracts rc ON cm.customer_id = rc.customer_id
    GROUP BY rc.branch_code, cm.customer_id, cm.credit_limit
  )
  SELECT branch_code,
         ROUND(AVG(outstanding_ar / NULLIF(credit_limit,0) * 100), 2) AS avg_credit_utilization_pct,
         SUM(CASE WHEN outstanding_ar > credit_limit THEN 1 ELSE 0 END) AS customers_over_limit_count
  FROM customer_risk
  GROUP BY branch_code
  ORDER BY avg_credit_utilization_pct DESC
  ```
- **Recommended LVDash Widget:** Bar chart
- **Data Fields:** `branch_code`, `avg_credit_utilization_pct`, `customers_over_limit_count`
- **Calculations:** Average credit utilization %, over-limit customer count
- **Interactivity:** Region filter; drill-through to customer detail
- **Justification:** A branch-level bar chart supports managerial comparison and prioritization.
- **Optimization Tips:** Reuse customer risk snapshot dataset across this chart and the detail grid.

### 2.13 Executive Drivers Table
- **Data Element:** Top aging/risk drivers by branch and customer
- **SQL Query:**
  ```sql
  WITH customer_driver AS (
    SELECT rc.branch_code,
           i.customer_id,
           cm.customer_name,
           SUM(i.invoice_amount) AS total_ar,
           SUM(CASE WHEN datediff(current_date(), i.due_date) > 90 THEN i.invoice_amount ELSE 0 END) AS ar_90_plus,
           MAX(cm.customer_status) AS customer_status,
           MAX(cm.credit_limit) AS credit_limit
    FROM silver.si_invoices i
    LEFT JOIN silver.si_customer_master cm ON i.customer_id = cm.customer_id
    LEFT JOIN silver.si_rental_contracts rc ON i.contract_id = rc.contract_id
    GROUP BY rc.branch_code, i.customer_id, cm.customer_name
  )
  SELECT branch_code,
         customer_id,
         customer_name,
         total_ar,
         ar_90_plus,
         ROUND(ar_90_plus / NULLIF(total_ar,0) * 100, 2) AS ar_90_plus_pct,
         customer_status,
         credit_limit,
         CASE
           WHEN customer_status = 'Suspended' THEN 'Suspended exposure'
           WHEN ar_90_plus > 0 THEN 'High aged receivable'
           WHEN total_ar > credit_limit THEN 'Over credit limit'
           ELSE 'Monitor'
         END AS primary_driver
  FROM customer_driver
  ORDER BY ar_90_plus DESC, total_ar DESC
  ```
- **Recommended LVDash Widget:** Table
- **Data Fields:** `branch_code`, `customer_id`, `customer_name`, `total_ar`, `ar_90_plus`, `ar_90_plus_pct`, `customer_status`, `credit_limit`, `primary_driver`
- **Calculations:** Risk driver classification; AR 90+ %
- **Interactivity:** Cross-filter from scorecards; branch and region filter; drill-through to customer exposure details
- **Justification:** Executive users need an actionable ranked list of what is driving poor health scores.
- **Optimization Tips:** Keep this dataset narrow and sorted; apply conditional formatting for `primary_driver` severity.

### 2.14 Data Quality Monitoring for Reporting Confidence
- **Data Element:** Count of data quality errors affecting reporting domains
- **SQL Query:**
  ```sql
  SELECT validation_category,
         source_table,
         error_severity,
         COUNT(*) AS error_count,
         MAX(detected_timestamp) AS latest_detected_timestamp
  FROM silver.si_data_quality_errors
  WHERE layer_name IN ('silver', 'gold')
  GROUP BY validation_category, source_table, error_severity
  ORDER BY error_count DESC
  ```
- **Recommended LVDash Widget:** Table or bar chart
- **Data Fields:** `validation_category`, `source_table`, `error_severity`, `error_count`, `latest_detected_timestamp`
- **Calculations:** Error counts by category/source/severity
- **Interactivity:** Source table and severity filters; drill-through to remediation list
- **Justification:** Dashboard trust improves when users can see upstream data quality issues impacting metrics.
- **Optimization Tips:** Separate operational tab; use aggregated summary rather than exposing raw payloads in the main dashboard.

---

## 3. Overall Dashboard Design

- **Layout Suggestions:** Use a multi-tab Lakeview dashboard with a clear executive-to-detail flow. **Tab 1: Executive Scorecard** should contain Overall Health, four domain score KPI cards, revenue trend, DSO trend, and an executive drivers table. **Tab 2: Revenue & Billing** should contain KPI strip (Rental Revenue, Gross Billings, Net Billings, ADR, Credit Memo Rate), a revenue trend chart, equipment class mix chart, and detailed sales rep/contract table. **Tab 3: AR Aging & Collections** should contain Total AR, AR 90+ %, DSO KPI cards, stacked aging bucket chart, collector leaderboard, and invoice-level detail table. **Tab 4: Cash Application** should contain Cash Collected, CEI, Avg Days to Pay, Unapplied Cash % KPI cards, collector comparison bar chart, and unapplied cash detail table. **Tab 5: Credit Risk** should contain Customers Over Limit, Suspended with Exposure KPI cards, branch credit utilization chart, customer risk heatmap/table, and exposure detail table. **Tab 6: Data Quality / Audit** can be an optional operational tab for error and pipeline status visibility.
- **Query Optimization:** Build Gold fact tables and daily/weekly/monthly aggregates for revenue, AR aging, cash application, credit risk, and health scoring. Partition large Delta tables by `invoice_date`, `receipt_date`, or snapshot date. Use `OPTIMIZE ... ZORDER BY (branch_code, customer_id, contract_id, invoice_id)` on frequently filtered datasets. Reuse one dataset across multiple widgets where the grain matches. Enable Databricks SQL result caching and Lakeview dataset reuse. Consider materialized views for monthly scorecards and aging snapshots. Size SQL warehouses for concurrency and dashboard refresh SLAs rather than peak ETL throughput.
- **Color Scheme:** Executive palette should use dark blue for primary business KPIs, green for favorable values, amber for warning states, and red for high-risk metrics. Suggested colors: Primary `#1F4E79`, Positive `#2E7D32`, Warning `#F9A825`, Risk `#C62828`, Neutral `#607D8B`. Use consistent aging bucket colors from lighter to darker red as delinquency increases.
- **Typography:** Use a clean sans-serif such as Inter or Databricks system sans. KPI values should be bold and large; chart titles medium-weight; table text regular with high contrast. Keep numeric columns right-aligned and percentage metrics formatted to 1 or 2 decimals.
- **Interactive Elements:** Global filters for date range, period type, region, branch, equipment class, customer status, sales rep, and collector. Parameters for Day/Week/Month granularity and Top N leaderboard size. Cross-filtering between branch charts and detail tables. Drill-through from KPI cards to supporting invoice, customer, contract, and payment details. Drill up/down on Network → Region → Branch → Sales Rep/Collector → Customer/Invoice hierarchies.

### 3.1 Interactivity Features Table

| Interactivity Feature | Type | Applies To | Behavior / Purpose |
|---|---|---|---|
| Date Range | Global filter | All tabs | Restricts all widgets to selected business period |
| Period Type | Parameter | Executive, Revenue, AR, Cash tabs | Switches aggregation grain between Day / Week / Month |
| Region | Global filter | All tabs | Enables network-to-region analysis |
| Branch | Filter | All tabs | Narrows results to branch-level operations |
| Equipment Class | Filter | Revenue tab | Analyzes product mix and billing performance |
| Customer Status | Filter | Credit Risk, Executive | Focuses on Active/Suspended customer exposure |
| Sales Rep | Filter | Revenue tab | Compares sales performance and contract activity |
| Collector | Filter | AR Aging, Cash App tabs | Evaluates collector workloads and performance |
| Cross-filtering | Cross-filter | Branch charts to detail tables | Clicking a branch updates lower-level widgets |
| Drill-through to Invoice Detail | Drill-through | AR Aging, Revenue, Cash App | Opens invoice-level records behind KPI or chart selection |
| Drill-through to Customer Exposure | Drill-through | Credit Risk, Executive Drivers | Opens customer contracts, invoices, and AR exposure |
| Drill up/down hierarchy | Hierarchy navigation | Executive, Revenue, AR, Cash | Network → Region → Branch → Sales Rep/Collector → Customer/Invoice |
| Top N Selector | Parameter | Leaderboards | Controls how many branches/customers/collectors are shown |

---

## 4. Recommended Data Modeling and Performance Practices

### 4.1 Recommended Fact and Dimension Design

| Dataset | Grain | Key Dimensions | Core Measures |
|---|---|---|---|
| `gold.fct_rental_revenue` | Branch / Sales Rep / Equipment Class / Date | Date, Branch, Region, Sales Rep, Equipment Class, Contract | Revenue, Gross Billings, Credit Memos, Net Billings, Invoice Count |
| `gold.fct_ar_aging` | Customer / Invoice / Snapshot Date | Date, Branch, Region, Customer, Collector, Sales Rep | Outstanding AR, Aging Bucket, Days Past Due, AR 90+, DSO |
| `gold.fct_cash_application` | Receipt / Invoice / Period | Date, Branch, Region, Collector, Customer, Payment Method | Cash Collected, Applied Amount, Unapplied Amount, Days to Pay, CEI inputs |
| `gold.fct_credit_risk` | Customer / Snapshot Date | Date, Branch, Region, Customer, Customer Status, Credit Terms | Credit Limit, Outstanding AR, Credit Utilization %, Over Limit $, AR 90+, Open Contract Count |
| `gold.fct_rev_to_cash_health` | Branch / Period | Date, Branch, Region | Overall Health Score, Revenue Score, AR Score, Cash App Score, Credit Risk Score |

### 4.2 Query Efficiency and Dataset Reuse Tips
- Reuse branch-period aggregated datasets across multiple scorecards instead of running separate SQL per KPI when the same base grain can feed several widgets.
- Avoid joining raw invoices to raw cash receipts in every dashboard query; instead build a conformed `gold.fct_cash_application` with pre-derived applied/unapplied and days-to-pay measures.
- Persist AR snapshots daily so aging calculations are deterministic, historically auditable, and fast.
- Use dataset result caching in Lakeview for heavily reused scorecard datasets.
- Keep highly detailed tables such as invoice detail and unapplied cash detail on separate widgets with pagination and default filters to avoid oversized payloads.
- For branch- and customer-heavy analytics, use Delta `OPTIMIZE` and `ZORDER` on branch, customer, due date, and invoice identifiers.
- Select SQL warehouse size based on concurrent viewer count and dashboard complexity; use serverless or pro warehouses with auto-stop and query acceleration where available.

### 4.3 Potential Pitfalls to Avoid

| Pitfall | Why It Matters | Mitigation |
|---|---|---|
| Recomputing aging buckets at runtime across full invoice history | Causes repeated scans and shuffles | Materialize daily AR aging snapshots |
| Joining invoices to branch employee without time-valid assignment logic | Can misattribute branch/region ownership | Create conformed branch assignment rules or effective-dated bridge |
| Duplicate invoice or payment records | Inflates AR, revenue, and collection metrics | Enforce business grain checks in Gold and monitor error tables |
| Using one oversized wide dataset for every widget | Increases refresh time and memory | Split into reusable subject-area datasets by grain |
| Calculating ratios without zero-denominator protection | Causes null/inf errors | Use `NULLIF` and business validation rules |
| Unoptimized warehouse sizing | Slow dashboard refresh and poor concurrency | Right-size SQL warehouse and enable caching |
| Skewed joins on branch or customer | Increases shuffle time | Pre-aggregate before join; optimize data layout; consider salting for extreme skew |
| Excessive drill-through on huge detail tables | Poor user experience | Limit records, paginate, and apply contextual filters |

---

## 5. Output URL (Clickable Hyperlinks)
[DI_Databricks_Dashboard_Visuals_Recommender_2.md](https://github.com/DIAscendion/Databricks_Dashboard/blob/main/DI_Databricks_Dashboard_Visuals_Recommender/DI_Databricks_Dashboard_Visuals_Recommender_2.md)

https://github.com/DIAscendion/Databricks_Dashboard/tree/main/DI_Databricks_Dashboard_Visuals_Recommender

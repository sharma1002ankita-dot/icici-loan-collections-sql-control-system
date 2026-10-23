# icici-loan-collections-sql-control-system

## Problem
ICICI Bank's retail loan EMI schedules, payments, collection follow-ups, and
restructuring decisions were tracked in separate operational views. This
project builds one SQLite-backed control system that answers, as of a fixed
review date: what was due, what was actually received, what's overdue, and
what needs action — using SQL as the analysis layer, not spreadsheets.

## Business Context
Before writing any code, I worked through the business requirements for this
project — a loan operations team that needed one trustworthy view of what's
overdue, who's behind, and what collection action has or hasn't been taken,
instead of piecing that together across separate payment logs, collection
notes, and restructuring records. I translated that into three questions any
operations or risk analyst actually asks day to day: which loans are overdue
right now, how severe is each case, and what's already been done about it.
Those three questions became the backbone of the design — seven tables
connected by eight relationships, and thirteen SQL queries that answer them
directly instead of requiring someone to manually cross-reference spreadsheets.

## ER Diagram
![ER Diagram](er-diagram.png)
*(exported from the notebook's rendered Graphviz output)*

## Tech Stack
- Python · SQLite · Google Colab
- REST API consumption (`requests`)
- Pandas for data handling and validation
- Graphviz for ER and workflow diagrams

## What It Does
- Models 7 core entities (branches, customers, loans, repayment schedule,
  payments, collection actions, restructuring requests) with a composite key
  on the repayment schedule
- Consumes a provided internal API with all 5 REST verbs (GET/POST/PUT/PATCH/DELETE)
- Answers 13 real business questions in SQL — from joins and aggregation
  through window functions (RANK, LAG, moving averages) to multi-step CTEs
  for a deterministic penalty calculation
- Flags 6 categories of operational exceptions automatically (missed
  instalments, amount mismatches, late payments, duplicate references, stale
  collection follow-up, aged restructuring requests)
- Demonstrates transaction safety with COMMIT (approved waiver) and
  ROLLBACK (unapproved adjustment)
- Designs a 7-step automation workflow, a top-20 collection-priority report,
  and 4 validated cron schedules
- Documents Git and safe-database-change workflows as ordered, reviewable steps

## Three Queries Worth Looking At

**Branch performance vs. the portfolio average (scalar subquery):**
```sql
WITH branch_rates AS (
  SELECT b.branch_id, b.branch_name,
         ROUND(100.0 * SUM(CASE WHEN p.payment_status='SUCCESS' THEN 1 ELSE 0 END) / COUNT(*), 2) AS success_rate
  FROM payments p JOIN loans l ON p.loan_id=l.loan_id JOIN branches b ON l.branch_id=b.branch_id
  GROUP BY b.branch_id, b.branch_name
)
SELECT branch_id, branch_name, success_rate FROM branch_rates
WHERE success_rate > (SELECT AVG(success_rate) FROM branch_rates)
ORDER BY success_rate DESC, branch_id;
```
*Answers: which branches collect more reliably than the bank-wide average?*

**Top-3 risk customers per city, ties retained (DENSE_RANK):**
```sql
WITH customer_exceptions AS (
  SELECT c.city, c.customer_id, c.customer_name, COUNT(*) AS exception_count,
         SUM(CASE WHEN le.severity='CRITICAL' THEN 1 ELSE 0 END) AS critical_count
  FROM loan_exceptions le
  JOIN loans l ON le.loan_id = l.loan_id
  JOIN customers c ON l.customer_id = c.customer_id
  WHERE le.resolution_status = 'OPEN'
  GROUP BY c.city, c.customer_id, c.customer_name
),
ranked AS (
  SELECT *, DENSE_RANK() OVER (PARTITION BY city ORDER BY exception_count DESC) AS exception_rank
  FROM customer_exceptions
)
SELECT city, customer_id, customer_name, exception_count, critical_count, exception_rank
FROM ranked WHERE exception_rank <= 3
ORDER BY city, exception_rank, customer_id;
```
*Answers: who are the highest-risk borrowers in each city, without arbitrarily dropping tied customers?*

**Transaction safety:**
```sql
BEGIN;
INSERT INTO loan_ledger (ledger_entry_id, loan_id, installment_no, entry_type, amount, entry_timestamp, description)
VALUES ('LED-DEVELOPER-COMMIT', 'L000001', 7, 'WAIVER', -750, '2026-08-01 10:00:00', 'Approved penalty waiver');
COMMIT;
```
*Answers: how do you guarantee an approved change is permanent and an unapproved one never happened?*

## How to Run
Open in Google Colab → Runtime → Run all. Setup (Section 1) builds the dataset, API, and database automatically; Sections 2–8 are the graded SQL Developer work.

## What I'd Build Next
A live dashboard on top of the collection-priority report, and real API-key authentication on the internal API — this version intentionally skips both to stay focused on the SQL and data-modeling work.

---
Full notebook: https://colab.research.google.com/drive/1XnO0ntDFSbc_71WFiSqDrfpf3ZjVa4cJ

# Tunda Box – Junior Data Analyst Assessment: Responses

**Name:** Maxwell Liaram (PLP Student)  **Date:** 8 November 2026
All numbers below are reproduced in `analysis.ipynb`. Amounts in KES. SQL is written for Postgres-style syntax; the notebook runs the SQLite equivalent.

**Standing assumptions** (applied throughout, stated once):
- *Revenue* = orders with `status = 'completed'`. Cancelled orders never happened; refunded orders returned the money.
- Cities are trimmed and case-normalised; duplicate `order_id` rows are collapsed to one.
- Order 1013 (KES 185,000, about 80x a typical order) is treated as a probable entry error and excluded from headline numbers; every affected answer shows what changes if it is genuine.

---

## Part 1: Your experience

**Your analysis.** As Data Officer at Mara Olchoro Oirowua Conservancy, I supported data management for the One Mara Carbon Project (OMCP), where monitoring data has to be accurate, documented and verifiable. The question I worked on was: *can we turn conservancy monitoring records, including EarthRanger patrol and incident data and carbon project monitoring data, into clean, consistent information that supports reporting and decisions?*

I collected, cleaned and validated the datasets (checking for duplicates, missing values and inconsistent categories), organised them so figures could be traced back to their source, and analysed the cleaned data to identify trends and patterns in wildlife and ranger activity, conservation incidents and project performance. I then prepared reports that met the carbon project's monitoring and documentation requirements.

The result was more reliable data and clearer reporting to support evidence-based decisions on conservancy operations and project performance tracking. Next time I would add validation at the point of data entry, so errors are prevented rather than corrected afterwards.

**Conflicting numbers.** I would not defend my number or assume the colleague is wrong. Steps:
1. **Align definitions first.** Same metric name often hides different definitions: completed vs. all orders, gross vs. net of refunds, date field (order vs. payment), time zone, date range, duplicates.
2. **Reconcile top-down.** Compare totals, then split by month/channel/status until the gap localises to a segment, then to specific rows.
3. **Check data lineage.** Different source tables, refresh times, or filters (e.g. test orders, NULLs, joins that drop rows).
4. **Agree on one definition** with the metric owner, document it, and ideally put it in a shared view so it can't diverge again.
5. **Tell stakeholders** what the gap was and which number to use, so nobody is left with two figures.

---

## Part 2: SQL

### S1. Orders by status
```sql
SELECT status,
       COUNT(*)                 AS rows_raw,
       COUNT(DISTINCT order_id) AS orders_distinct
FROM orders
GROUP BY status
ORDER BY orders_distinct DESC;
```
| status | raw rows | distinct orders |
|---|---|---|
| completed | 17 | **16** |
| refunded | 2 | 2 |
| cancelled | 2 | 2 |

Order 1006 appears twice (identical row), so a plain `COUNT(*)` overstates completed orders by one. Distinct total is 20 orders.

### S2(a). Which city brings in the most completed-order revenue?
```sql
SELECT INITCAP(TRIM(c.city)) AS city,
       SUM(o.amount_kes)     AS revenue_kes,
       COUNT(*)              AS completed_orders
FROM (SELECT DISTINCT * FROM orders) o       -- remove the duplicate 1006
JOIN customers c USING (customer_id)         -- inner join drops orphan customer 999
WHERE o.status = 'completed'
  AND o.amount_kes < 50000                   -- exclude suspected entry error (order 1013)
GROUP BY 1
ORDER BY revenue_kes DESC;
```
| city | revenue (KES) | orders |
|---|---|---|
| **Nairobi** | **20,650** | 9 |
| Mombasa | 5,150 | 2 |
| Nakuru | 3,300 | 1 |
| Kisumu | 2,100 | 1 |

**Why the cleaning matters.** A naive query returns *Kisumu* at 187,100 on top (driven by the outlier) and splits Nairobi into three groups (`Nairobi`, `nairobi`, `Nairobi `). Nairobi still leads if order 1013 was meant to be 1,850 (Kisumu then 3,950). It only loses if 185,000 is genuine, in which case Kisumu is 187,100.

### S2(b). What I send my manager in ten minutes
> **Nairobi is the top city: KES 20,650 completed-order revenue (9 orders), well ahead of Mombasa at 5,150.**
> Caveats: (1) one Kisumu order of KES 185,000 looks like a typo, which I've excluded. If it is real, Kisumu leads at 187,100, so can someone confirm it? (2) I merged "Nairobi / nairobi / Nairobi ", removed one duplicated order, and left out one order with an unknown customer (KES 1,900) and one completed order with a missing amount. (3) This is a small sample extract, so treat it as directional.

Reasoning: a fast answer with the one decision-relevant risk stated up front, not a silent number and not a data-quality essay.

### S3. Customers who never ordered
```sql
SELECT c.*
FROM customers c
LEFT JOIN orders o ON o.customer_id = c.customer_id
WHERE o.order_id IS NULL;
```
Result: **109 (Eldoret, signed up 2026-06-21, Organic)** and **110 (Kisumu, signed up 2026-07-05, Paid Social)**. I counted a cancelled order as "placed", so a customer with only cancelled orders would not appear here.

### S4. Repeat purchase rate
```sql
WITH per_customer AS (
  SELECT c.customer_id, COUNT(o.order_id) AS completed_orders
  FROM customers c
  LEFT JOIN (SELECT DISTINCT * FROM orders) o
         ON o.customer_id = c.customer_id AND o.status = 'completed'
  GROUP BY c.customer_id)
SELECT COUNT(*) AS customers,
       SUM(CASE WHEN completed_orders >= 2 THEN 1 ELSE 0 END) AS repeat_buyers,
       ROUND(100.0 * SUM(CASE WHEN completed_orders >= 2 THEN 1 ELSE 0 END) / COUNT(*), 1) AS repeat_pct
FROM per_customer;
```
**Result: 40% (4 of 10 customers).** Repeat buyers: 101 (4 orders), 102 (3), 104 (2), 107 (2).

How I worked it out: repeat buyer = 2+ **completed** orders, de-duplicated, with **all 10 customers** as the denominator since the question says "of our customers". Among customers who actually bought (8), it is **50%**; I'd quote both if the audience is ambiguous.
Pitfalls avoided: not deduping order 1006 makes customer 105 look repeat (50%); counting cancelled and refunded orders would push it to 80%; customer 999 isn't in `customers`, so it can't enter the denominator.

### S5. Month-on-month revenue
```sql
WITH monthly AS (
  SELECT DATE_TRUNC('month', order_date) AS month,
         SUM(amount_kes)                 AS revenue_kes
  FROM (SELECT DISTINCT * FROM orders) o
  WHERE status = 'completed' AND amount_kes < 50000
  GROUP BY 1)
SELECT month, revenue_kes,
       ROUND(100.0 * (revenue_kes - LAG(revenue_kes) OVER (ORDER BY month))
             / LAG(revenue_kes) OVER (ORDER BY month), 1) AS mom_pct
FROM monthly
ORDER BY month;
```
| month | revenue (KES) | MoM % |
|---|---|---|
| 2026-06 | 9,000 | n/a |
| 2026-07 | 6,800 | -24.4% |
| 2026-08 | 7,000 | +2.9% |
| 2026-09 | 10,300 | +47.1% |

Caveats: with the 185,000 order left in, August shows 192,000 (+2,723%) and September -94.6%, which would be a misleading story. July is understated (order 1009 has no amount). **September data ends on the 26th, so +47.1% compares a partial month to full ones.** September includes order 1020 (unknown customer, KES 1,900) because monthly revenue doesn't depend on the customer table; without it September is 8,400 (+20%). The first order in the extract is dated 3 June, so confirm June isn't also partial.

---

## Part 3: Data quality and spreadsheets

### D1. Audit
| # | Issue | Evidence | What I'd do |
|---|---|---|---|
| 1 | Inconsistent city values | `nairobi`, `Nairobi ` | `TRIM` + `INITCAP`; fix at source with validated dropdown/lookup |
| 2 | Duplicate order | order 1006 appears twice | Dedupe on `order_id`; add a primary key constraint; find the cause (retry, double-insert in ETL) |
| 3 | Missing amount | order 1009 completed but `NULL` | Don't guess; query source system/payments (M-Pesa record); report revenue as understated until fixed |
| 4 | Extreme outlier | order 1013 = 185,000 vs. median ~2,400 | Verify with ops/payment record (likely extra zeros); exclude or correct once confirmed; add a range check |
| 5 | Orphan order | order 1020, customer 999 not in `customers` | Check for guest checkout or missing customer sync; exclude from customer-level metrics, keep for order-level, add FK check |
| 6 | Order before signup | order 1003 (11 Jun) precedes customer 103's signup (20 Jun) | Check which date is wrong or whether guest orders were later attached; matters for cohort/retention analysis |
| 7 | Refunds and cancellations carry amounts | 1003, 1007, 1014, 1018 | Define "revenue" explicitly (completed only? net of refunds?) and publish it |
| 8 | Incomplete latest month | last order 26 Sep | Don't compare partial month to full months; label or exclude |
| 9 | Sample only, no time field | small extract, dates without timestamps/time zones | Confirm extract logic, row counts vs. full table; state limitations |

Order of operations: **verify and fix at source where possible, document every rule, then analyse**, and don't silently drop rows.

### D2. Spreadsheet formulas
Total completed M-Pesa revenue (orders sheet: D = status, E = amount, F = payment method):
```
=SUMIFS(E:E, D:D, "completed", F:F, "M-Pesa")
```
On the raw sheet this returns **209,550**, which includes the duplicate (1,500) and the outlier (185,000). The correct figure after cleaning is **23,050**. Use `=SUMIFS(E2:E21,D2:D21,"completed",F2:F21,"M-Pesa")` after deleting the duplicate row and either correcting or removing the outlier. `SUMIFS` ignores the blank amount.

City per order, in a new column G (customer_id in B; `customers` sheet has id in A, city in B):
```
=IFERROR(PROPER(TRIM(VLOOKUP(B2, customers!$A:$B, 2, FALSE))), "Unknown")
```
Modern alternative: `=IFERROR(PROPER(TRIM(XLOOKUP(B2, customers!A:A, customers!B:B))), "Unknown")`. Order 1020 returns "Unknown", which surfaces the orphan rather than hiding it. Fill down the column.

### D3. Mean or median?
**Median.** Order values are right-skewed and the data contains a 185,000 entry: the mean is 14,540, which describes no actual order, while the median is 2,400. After removing the outlier, mean (2,364) and median (2,375) agree, which shows the median was robust all along. I'd report the median as the "typical order" and keep the mean for revenue forecasting (total revenue = mean x number of orders), and say which is which.

---

## Part 4: Statistics

### T1(a). A p-value, for a manager
"Suppose the change actually did nothing. The p-value is how often we'd still see a difference this big, or bigger, purely by luck. A p-value of 0.03 means about 3 times in 100. A small p-value means 'hard to explain by chance alone'. It does *not* tell you the chance the idea works, or how big or valuable the effect is."

### T1(b). Do SMS offers cause the extra spending?
**Not shown by that comparison.** It's a correlation. Customers who get SMS offers may be different to begin with: if the offers go to loyal or high-spending customers, they'd spend more anyway (selection bias / confounding). Spending might also have caused the SMS (reverse causality), for example if targeting rules pick active customers.
To establish causation: a **randomised holdout** (randomly send to some and withhold from others, then compare), or if that's impossible, matching/difference-in-differences on pre-period spend, while being explicit that observational methods only reduce, not remove, the doubt. Also measure incremental *profit* after discount cost, not just spend.

### T2. A/B test
| | Conversion |
|---|---|
| Current | 5.0% (50/1,000) |
| New | 6.0% (60/1,000) |

The "20% rise" is relative; the absolute lift is **1.0 percentage point**. Two-proportion z-test: z = 0.98, **p = 0.33**, 95% CI for the difference **-1.0 to +3.0 pp**. The result is consistent with no effect, a small harm, or a good gain. With 1,000 users per arm the test can only reliably detect lifts of about 2.9pp; detecting 5% -> 6% at 80% power needs about **8,150 users per arm**.

**Recommendation:** don't ship on this evidence, and don't call it a failure either: it's *inconclusive*. Run the test to the pre-calculated sample size, check for a sample-ratio mismatch and novelty effects, and look at revenue per user and checkout errors alongside conversion. If the PM needs a decision now, shipping is a low-risk bet only if the new checkout has other merits (cost, maintainability) and can be rolled back.

---

## Part 5: Business case

### (a) What is happening
| | Sep CVR | Oct CVR | Sep share of sessions | Oct share |
|---|---|---|---|---|
| Organic | 6.0% | 6.1% | 80% | 56% |
| Paid Social | 2.0% | 2.1% | 20% | 44% |
| **Total** | **5.2%** | **4.3%** | | |

- **Conversion in each channel did not fall; it ticked up.** The blended drop is a **mix effect**: paid social sessions tripled (10k -> 30k), and paid converts at about a third of organic's rate, so the average falls.
- Decomposition of the -0.86pp change: **mix effect -0.96pp**, within-channel rate change **+0.10pp**.
- Paid social is not hurting the site: it added **+430 orders** (the extra 20k sessions converted at 2.15%, no sign of diminishing returns yet). Total orders grew **+348 (+13%)**.
- One real flag: **organic sessions fell 5%** (40k -> 38k) and orders by 82, so check whether paid is displacing some organic or branded traffic, or whether it's seasonal.

**What else I'd want to know:** spend and cost per order by channel (is 2% conversion profitable?); average order value, refund rate and repeat rate of paid-acquired customers (lifetime value); why paid tripled (budget, a campaign, an audience change) and whether tracking or attribution changed; whether the Sep/Oct sessions and conversion definitions are the same; new vs. returning split; device and landing-page breakdown within paid; incrementality (would those buyers have come anyway?).

### (b) Recommendation (two sentences)
> Please don't pause paid social: conversion within each channel is flat to slightly up (organic 6.0% -> 6.1%, paid 2.0% -> 2.1%), and the headline drop is a mix effect from tripling paid traffic, which also added about 430 orders. Before scaling further or cutting, let me pull cost per order and customer value for paid so we can decide on profitability, not on the blended rate.

---

## Part 6: Questions for the team
1. What are the 2–3 business questions you most want this role to help answer in the first six months?
2. What does the data stack look like (warehouse, BI tool, how is data modelled), and who owns data quality?
3. Where do the metric definitions live (e.g. revenue, conversion, repeat rate), and how are disagreements resolved?
4. How does analysis turn into decisions here: who consumes it, and can you share an example where an analysis changed a decision?
5. What would a strong first 90 days look like, and how is a junior analyst's work reviewed and mentored?
6. How does the team handle perishable-goods problems in the data, such as substitutions, spoilage, and delivery-slot constraints?

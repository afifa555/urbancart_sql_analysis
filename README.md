# UrbanCart Analytics — Week 1 Case Study

A 5-day SQL practice project simulating freelance client work for a fictional
ecommerce store ("UrbanCart"). Each day started from a vague, realistically
worded client request, required identifying and handling intentional data
quality issues (duplicate rows, nulls, inconsistent statuses, refunds), and
ended with a client-ready written summary — not just a query.

**Dataset:** 5 relational tables (customers, products, orders, order_items,
payments) — 1,277 orders, 3,102 order items, 220 customers — with deliberately
seeded data issues: duplicate order rows, null/negative quantities, null
payment amounts, inconsistent status casing, and refunds recorded as negative
payment amounts.

**Tools:** SQL (DuckDB), DBeaver.

---

## Day 1 — Revenue by Product Category

**Client brief:** *"Pull total revenue from order_items, broken out by product
category. Should be quick."*

**Data issues handled:** duplicate order rows (deduped before joining), null
and negative quantities (excluded), order status (filtered to completed only
— "revenue" means money actually kept, not revenue from every order ever
placed).

```sql
WITH ranked AS (
    SELECT *,
           ROW_NUMBER() OVER (PARTITION BY order_id ORDER BY order_id) AS rn
    FROM orders
)
SELECT
    p.category,
    ROUND(SUM(oi.quantity * oi.unit_price), 2) AS total_revenue
FROM ranked o
JOIN order_items oi ON o.order_id = oi.order_id
JOIN products p ON oi.product_id = p.product_id
WHERE o.rn = 1
  AND o.status = 'completed'
  AND oi.quantity IS NOT NULL
  AND oi.quantity > 0
GROUP BY p.category
ORDER BY total_revenue DESC;
```

**Result:** Apparel led at $203,717, followed by Beauty ($171,943), Books
($95,843), Toys ($94,379), Sports & Outdoors ($50,171), Home & Kitchen
($18,155), and Electronics ($17,942).

**Client summary:**
> Here's total revenue by category, based on completed orders only (I
> excluded cancelled and refunded orders since that money wasn't actually
> kept). I also cleaned up a couple of data issues first: duplicate order
> records and a handful of rows with missing or negative quantities, which
> I removed since they'd have inflated the numbers. Apparel is the top
> category at $203,717, followed by Beauty at $171,943.

---

## Day 2 — Top Spenders & Never-Ordered Customers

**Client brief:** *"Find our top 10 customers by total spend, and flag any
customers who've never placed an order — marketing wants to email the two
groups differently."*

**Data issues handled:** duplicate order rows (deduped), refunds (kept as
negative amounts so they net out of a customer's total), null payment amounts
(SQL's `SUM()` skips nulls automatically — no fix needed), null emails
(`COALESCE`'d to a visible placeholder).

**Part 1 — Top 10 by spend:**
```sql
WITH ranked AS (
    SELECT *,
           ROW_NUMBER() OVER (PARTITION BY order_id ORDER BY order_id) AS rn
    FROM orders
)
SELECT c.customer_id,
       COALESCE(c.email, 'NO EMAIL ON FILE') AS email,
       ROUND(SUM(p.amount), 2) AS total_spend
FROM payments p
JOIN ranked o ON p.order_id = o.order_id AND o.rn = 1
JOIN customers c ON o.customer_id = c.customer_id
GROUP BY c.customer_id, c.email
ORDER BY total_spend DESC
LIMIT 10;
```

**Part 2 — Never ordered:**
```sql
SELECT c.customer_id,
       COALESCE(c.email, 'NO EMAIL ON FILE') AS email
FROM customers c
LEFT JOIN orders o ON c.customer_id = o.customer_id
WHERE o.order_id IS NULL;
```

**Result:** Top spender was customer 181 at $16,761. Part 2 returned **0
rows** — every customer in the dataset has placed at least one order.

**Client summary:**
> Top 10 customers by spend: led by customer 181 (~$16.8K). I removed
> duplicate order records first and subtracted refunds from each total, so
> these are net amounts — totals include all payments received (completed,
> pending, etc.), minus refunds. Customer 213 is in the top 10 but has no
> email on file. Never-ordered customers: there aren't any — everyone has
> ordered at least once. If you want a win-back list instead, I can pull
> customers who ordered once and never came back, or haven't ordered
> recently.

---

## Day 3 — Win-Back List (Inactive + High-Value Customers)

**Client brief:** *"Find customers who haven't ordered in a while, and flag
which of those were big spenders — those are worth a personal email."*

**Data issues handled:** "today" defined as the latest date in the data
(2024-06-30), cancelled orders and null order dates excluded before finding
each customer's last real order, status casing double-checked (an earlier
`!= 'Cancelled'` filter silently matched nothing due to case sensitivity —
fixed to lowercase), duplicate orders deduped separately before the spend
calculation. Customers with zero or negative net spend were deliberately
**excluded** via `INNER JOIN** — a personal-email "big spender" list isn't
useful if it's cluttered with customers who spent nothing.

```sql
WITH ranked AS (
    SELECT *,
           ROW_NUMBER() OVER (PARTITION BY customer_id ORDER BY order_date DESC) AS rn
    FROM orders AS o
    WHERE o.status != 'cancelled'
      AND o.order_date IS NOT NULL
),
new_order AS (
    SELECT c.customer_id, c.email, r.status, r.order_date,
           DATE_DIFF('day', CAST(r.order_date AS DATE), DATE '2024-06-30') AS days_since_last_order,
           r.rn
    FROM customers AS c
    JOIN ranked AS r ON c.customer_id = r.customer_id
),
inactive_customers AS (
    SELECT * FROM new_order
    WHERE rn = 1 AND days_since_last_order > 180
),
dedup_orders AS (
    SELECT *, ROW_NUMBER() OVER (PARTITION BY order_id ORDER BY order_id) AS dedup_rn
    FROM orders
),
customer_spend AS (
    SELECT o.customer_id, ROUND(SUM(p.amount), 2) AS total_spend
    FROM dedup_orders AS o
    JOIN payments AS p ON o.order_id = p.order_id
    WHERE o.dedup_rn = 1
    GROUP BY o.customer_id
)
SELECT i.customer_id, i.email, i.order_date AS last_order_date,
       i.days_since_last_order, s.total_spend
FROM inactive_customers AS i
INNER JOIN customer_spend AS s ON i.customer_id = s.customer_id
ORDER BY s.total_spend DESC;
```

**Result:** 75 customers inactive 180+ days. Top spender among them: ~$7,159.
About a dozen had *negative* net spend (refunds outweighed payments).

**Client summary:**
> I used 2024-06-30 as "today" (the latest date in the data) and flagged
> anyone whose last non-cancelled order was more than 180 days before that
> — 75 customers. I matched them against total spend (same method as Day
> 2) and sorted by spend. Top of the list is ~$7,159. One flag: about a
> dozen of these 75 have negative net spend — refunds outweighed what they
> paid — so they're inactive but not really "big spenders." Want those
> pulled out of the personal-email list, or is the full 75 useful as-is?

---

## Day 4 — Monthly Revenue Trend & Customer Segmentation

**Client brief:** *"Break down completed revenue by month so I can see the
trend. Separately, label customers as VIP / Regular / One-time based on how
many orders they've placed."*

**Data issues handled:** duplicate orders deduped **before** joining to
order_items (joining first and deduping after was tried and found to
multiply rows incorrectly), null/negative quantity excluded, status casing
re-verified, `order_date` cast from text to DATE before date math, months
grouped with `DATE_TRUNC` (not a plain month-number extract, which would
merge different years together).

**Part 1 — Revenue by month:**
```sql
WITH ranked AS (
    SELECT *, ROW_NUMBER() OVER (PARTITION BY order_id ORDER BY order_id) AS rn
    FROM orders
)
SELECT
    CAST(DATE_TRUNC('month', CAST(r.order_date AS DATE)) AS DATE) AS order_month,
    ROUND(SUM(oi.quantity * oi.unit_price), 2) AS total_revenue
FROM order_items oi
JOIN ranked r ON oi.order_id = r.order_id
WHERE r.rn = 1
  AND oi.quantity IS NOT NULL
  AND oi.quantity > 0
  AND r.status = 'completed'
  AND r.order_date IS NOT NULL
  AND r.order_date != ''
GROUP BY order_month
ORDER BY order_month;
```

**Part 2 — Customer segmentation:**
```sql
WITH Dedupe AS (
    SELECT *, ROW_NUMBER() OVER (PARTITION BY order_id ORDER BY order_id) AS rn
    FROM orders
),
Order_count AS (
    SELECT customer_id, COUNT(order_id) AS total_orders
    FROM Dedupe d
    WHERE d.status = 'completed' AND d.rn = 1
    GROUP BY customer_id
),
Customers_rank AS (
    SELECT c.customer_id, c.email, oc.total_orders,
        CASE
            WHEN oc.total_orders >= 10 THEN 'VIP'
            WHEN oc.total_orders BETWEEN 3 AND 9 THEN 'Regular'
            WHEN oc.total_orders BETWEEN 1 AND 2 THEN 'One_time'
            ELSE 'No_completed_orders'
        END AS customer_category
    FROM customers c
    LEFT JOIN Order_count oc ON c.customer_id = oc.customer_id
)
SELECT * FROM Customers_rank
ORDER BY total_orders DESC;
```

**Result:** 17 months of data (Feb 2023 – Jun 2024), revenue climbing
steadily from ~$1.4K to ~$126.7K in the final month. Segmentation: 16 VIP,
84 Regular, 77 One-time, 43 with no completed orders (220 total).

**Client summary:**
> Monthly completed revenue climbs steadily across the 17 months of data,
> with a sharp jump in the final two months (May–June 2024) worth asking
> about — real growth, seasonal spike, or an end-of-data effect. For
> segmentation: 16 customers are VIP (10+ completed orders), 84 Regular
> (3–9), 77 One-time (1–2), and 43 have no completed orders at all.

---

## Day 5 — Underperforming Products

**Client brief:** *"Find products that have been ordered but have total
revenue under $500 — thinking about discontinuing them."*

**Data issues handled:** same dedup/quantity/status checklist as every
revenue question this week. New concept: `HAVING`, to filter on an
aggregate (`SUM`) after grouping — `WHERE` can't do this, same timing
restriction that applies to window functions.

```sql
WITH ranked AS (
    SELECT *, ROW_NUMBER() OVER (PARTITION BY order_id ORDER BY order_id) AS rn
    FROM orders
),
revenue AS (
    SELECT oi.product_id, p.product_name,
           SUM(oi.quantity * oi.unit_price) AS total_revenue
    FROM order_items oi
    JOIN ranked r ON oi.order_id = r.order_id
    JOIN products p ON oi.product_id = p.product_id
    WHERE r.status = 'completed'
      AND r.rn = 1
      AND oi.quantity IS NOT NULL
      AND oi.quantity > 0
    GROUP BY oi.product_id, p.product_name
    HAVING total_revenue < 500
)
SELECT * FROM revenue
ORDER BY total_revenue DESC;
```

**Result:** 0 rows. The lowest-revenue product in the dataset (Apparel Item
38) still earned ~$1,205 — well above the $500 threshold. Verified by
re-running with a generous threshold to confirm the pipeline wasn't
silently broken.

**Client summary:**
> I checked every product's total revenue from completed orders against the
> $500 threshold. Nothing comes in below that — the weakest performer is
> Apparel Item 38, which still brought in about $1,205. Based on this
> cutoff, there's nothing to recommend discontinuing right now. If you want
> a tighter list, I can raise the threshold or look at a shorter recent
> window instead of all-time revenue.

---

## Recurring techniques across the week

- **Dedup pattern:** `ROW_NUMBER() OVER (PARTITION BY <unique key> ORDER BY <tiebreaker>)` in a CTE, filtered to `rn = 1` in the outer query — used daily on the duplicate `order_id` rows.
- **Dedup before join, not after** — deduping post-join lets the join multiply rows first, corrupting the row-numbering.
- **`LEFT JOIN` + check an ID column (not a data column) with `IS NULL`** to detect "no match exists" — used for never-ordered customers and missing payments.
- **`INNER` vs `LEFT` join is a judgment call tied to the brief**, not a fixed rule — stated explicitly in each day's summary when it mattered.
- **Window function aliases can't be reused in the same `WHERE`** (must wrap in a CTE); aggregate/expression aliases (`SUM(...)`, `DATE_TRUNC(...)`) *can* be reused in `GROUP BY`/`ORDER BY` in the same query.
- **Always verify assumptions against the actual data** rather than trusting documentation at face value — a status-casing bug (`'Cancelled'` vs `'cancelled'`) silently broke a filter twice this week until checked directly.
- **Not every data quirk applies to every question** — duplicates matter for revenue totals but not for "has this customer ever ordered."

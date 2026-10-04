# Practice Dataset — "UrbanCart" (fictional ecommerce store)

Load these 5 CSVs into Postgres / MySQL / SQLite / DB Fiddle. Table names should
match the file names (customers, products, orders, order_items, payments).

## Tables

**customers** (220 rows)
- customer_id (PK)
- signup_date
- email → ~12 rows have NULL email (missing data)
- city → inconsistent spellings on purpose: "New York" / "new york" / "NY",
  "Chicago" / "Chicago " (trailing space), "LA" / "Los Angeles"

**products** (45 rows)
- product_id (PK)
- product_name
- category (Electronics, Home & Kitchen, Apparel, Beauty, Sports & Outdoors, Toys, Books)
- cost, price

**orders** (1277 rows, includes 12 intentional duplicate rows)
- order_id → NOT unique as a row-key right now, 12 rows are exact duplicates
  (simulates a double-insert ETL bug — dedupe before analysis, e.g. with
  ROW_NUMBER() OVER (PARTITION BY order_id ...))
- customer_id (FK → customers)
- order_date → 6 rows have NULL order_date
- status → completed / cancelled / refunded / pending
- payment_method → credit_card / paypal / debit_card / gift_card / NULL

**order_items** (3102 rows)
- order_item_id (PK)
- order_id (FK → orders)
- product_id (FK → products)
- quantity → 5 rows NULL, 5 rows negative (bad data entry — never trust
  raw quantity blindly, filter/clean first)
- unit_price

**payments** (1174 rows)
- payment_id (PK)
- order_id (FK → orders) — note: some cancelled orders have NO payment row
  at all (LEFT JOIN territory)
- amount → 33 rows NULL (system glitch), refunded orders have NEGATIVE
  amount (money returned)
- payment_date

## Known quirks (use these deliberately in your practice)

1. Duplicate order rows → dedupe with ROW_NUMBER()/DISTINCT before aggregating revenue.
2. NULL order_date → decide how to handle (exclude vs. impute) and state your assumption.
3. NULL / negative quantity in order_items → clean before computing revenue.
4. Inconsistent city text → needs UPPER/TRIM/CASE normalization for city-level reports.
5. Missing emails → COALESCE for reporting, don't just drop rows.
6. Cancelled orders sometimes missing a payment row → LEFT JOIN vs INNER JOIN matters.
7. Refunds are negative payment amounts → decide whether "revenue" should net these out.
8. Customer behavior mix is realistic: ~29% one-time buyers, ~25% churn early,
   ~27% regular, ~19% loyal/VIP — good for churn, RFM, and cohort exercises.

## Suggested load order (to respect foreign keys)
customers → products → orders → order_items → payments

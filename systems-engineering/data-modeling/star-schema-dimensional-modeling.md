# Star Schema & Dimensional Modeling (Kimball Methodology)

## 30-Second Intuition

Dimensional modeling splits a business process into two kinds of tables: **facts** (the numeric, additive things that happened — an order line, a page view, a sensor reading) and **dimensions** (the who/what/when/where that gives those numbers meaning — customer, product, date, store). A **star schema** is what you get when you put one fact table in the middle and join denormalized dimension tables directly to it — no dimension joins to another dimension, so the ER diagram literally radiates like a star. The single fact that matters most operationally: **grain — the definition of what one fact row represents — is the first decision you make and the one you cannot cheaply undo**; every other design choice (which foreign keys the fact table carries, how SCDs are handled, whether a `SUM()` is even valid) follows from grain, and getting it wrong doesn't surface as a bug, it surfaces as silently wrong numbers in a dashboard someone already trusted.

---

## The Star Shape

```
                    dim_date
                        │
                        │ date_key
                        │
  dim_customer ──customer_key──┐
                                 │
                          fact_orders
                                 │
  dim_product  ───product_key───┤
                                 │
                        store_key│
                        │
                    dim_store
```

`fact_orders` sits in the middle holding only: foreign keys to each dimension, plus numeric measures (`quantity`, `unit_price`, `line_amount`). Each dimension table is **denormalized** — `dim_product` carries `category_name` and `department_name` as plain columns, not foreign keys to separate `dim_category` / `dim_department` tables, even though that duplicates the string across every product row in the same category. That duplication is the deliberate trade: one join per dimension instead of a chain of joins, at the cost of some repeated bytes. A **snowflake schema** is the same idea with dimensions normalized further (see the comparison section below) — it looks like the star's points have been split into smaller sub-points.

---

## Worked Example: Fact + Dimensions, DDL to Query

```sql
-- Dimension: customer (Type 1 for now — overwrite on change; SCD Type 2 version later)
CREATE TABLE dim_customer (
    customer_key    BIGINT PRIMARY KEY,   -- surrogate key, not the source system's customer_id
    customer_id     VARCHAR NOT NULL,     -- natural/business key from the source system
    customer_name   VARCHAR NOT NULL,
    city            VARCHAR,
    state           VARCHAR,
    segment         VARCHAR              -- 'consumer', 'enterprise', 'smb'
);

-- Dimension: product
CREATE TABLE dim_product (
    product_key     BIGINT PRIMARY KEY,
    product_id      VARCHAR NOT NULL,
    product_name    VARCHAR NOT NULL,
    category        VARCHAR,             -- denormalized: no separate dim_category table
    department      VARCHAR,
    unit_cost       DECIMAL(10,2)
);

-- Dimension: date (pre-populated, one row per calendar day — a Kimball staple)
CREATE TABLE dim_date (
    date_key        INT PRIMARY KEY,      -- e.g. 20260913
    full_date       DATE NOT NULL,
    day_of_week     VARCHAR,
    month           INT,
    month_name      VARCHAR,
    quarter         INT,
    year            INT,
    is_weekend      BOOLEAN
);

-- Fact: one row per ORDER LINE ITEM (grain declared explicitly — see Grain section)
CREATE TABLE fact_order_lines (
    order_line_key  BIGINT PRIMARY KEY,
    order_id        VARCHAR NOT NULL,     -- degenerate dimension: no dim_order table, just the id
    customer_key    BIGINT REFERENCES dim_customer(customer_key),
    product_key     BIGINT REFERENCES dim_product(product_key),
    date_key        INT REFERENCES dim_date(date_key),
    quantity        INT NOT NULL,
    unit_price      DECIMAL(10,2) NOT NULL,
    line_amount     DECIMAL(12,2) NOT NULL   -- quantity * unit_price, stored pre-computed
);
```

Sample rows:

```sql
INSERT INTO dim_customer VALUES
    (1, 'CUST-001', 'Acme Corp', 'Austin', 'TX', 'enterprise'),
    (2, 'CUST-002', 'Jane Doe',  'Denver', 'CO', 'consumer');

INSERT INTO dim_product VALUES
    (10, 'SKU-100', 'Widget A', 'Widgets', 'Hardware', 4.50),
    (11, 'SKU-101', 'Widget B', 'Widgets', 'Hardware', 6.00);

INSERT INTO dim_date VALUES
    (20260910, '2026-09-10', 'Thursday', 9, 'September', 3, 2026, false);

INSERT INTO fact_order_lines VALUES
    (1001, 'ORD-500', 1, 10, 20260910, 3, 9.99, 29.97),
    (1002, 'ORD-500', 1, 11, 20260910, 1, 14.99, 14.99),  -- same order, second line
    (1003, 'ORD-501', 2, 10, 20260910, 5, 9.99, 49.95);
```

The query that shows why the star shape is natural — join the fact to each dimension exactly once, filter and group entirely on dimension attributes:

```sql
SELECT
    dc.segment,
    dp.category,
    dd.month_name,
    SUM(fol.line_amount)   AS total_revenue,
    SUM(fol.quantity)      AS total_units
FROM fact_order_lines fol
JOIN dim_customer dc ON fol.customer_key = dc.customer_key
JOIN dim_product  dp ON fol.product_key  = dp.product_key
JOIN dim_date     dd ON fol.date_key     = dd.date_key
WHERE dd.year = 2026
  AND dc.segment = 'enterprise'
GROUP BY dc.segment, dp.category, dd.month_name;
```

Three joins, no join chains, every filter/group column is a plain column on the dimension side — this is the query shape BI tools generate automatically, which is why star schemas are the default target for semantic layers and self-service dashboards.

---

## Grain: The Decision Everything Else Depends On

**Grain = the precise statement of what one row in the fact table represents.** It must be declared before a single column is added, and it must be as atomic as the business process allows — Kimball's repeated advice is "grain first, then dimensions, then facts."

**Getting it wrong, concretely.** Suppose the fact table above had instead been modeled at "one row per order" (grain: order, not order line):

```sql
CREATE TABLE fact_orders_wrong_grain (
    order_id        VARCHAR PRIMARY KEY,
    customer_key    BIGINT,
    date_key        INT,
    total_quantity  INT,          -- summed across all lines at load time
    total_amount    DECIMAL(12,2)
);
```

This looks reasonable until someone needs per-product revenue: there is no `product_key` on the row, because a single order can contain multiple products. The fix isn't a column addition — you cannot append `product_key` to a table whose grain is "one row per order" without either creating duplicate order rows (breaking every existing `SUM(total_amount)` query, which would now double- or triple-count) or leaving `product_key` null for multi-product orders (silently wrong `GROUP BY product`). This is why grain mistakes require a **full rebuild** of the fact table and everything downstream of it, not a patch:

```sql
-- At order grain: correct, matches invoices
SELECT SUM(total_amount) FROM fact_orders_wrong_grain WHERE customer_key = 1;
-- 44.96  (one row for ORD-500, matches the order total)

-- Now add product_key by "just" duplicating the order row per product touched — the trap:
-- ORD-500 becomes 2 rows (Widget A, Widget B), each still carrying total_amount = 44.96
SELECT SUM(total_amount) FROM fact_orders_duplicated_wrong WHERE customer_key = 1;
-- 89.92  -- silently doubled, because total_amount was never redistributed to line grain
```

The correct fix is `fact_order_lines` from the worked example: grain = one row per order line, `line_amount` is the per-line measure, and `SUM(line_amount)` is additive at any rollup (by customer, by product, by day) without double-counting. **Rule of thumb**: state the grain in one sentence ("one row per X per Y") before writing the `CREATE TABLE`, and verify every measure column is additive at that exact grain.

---

## Slowly Changing Dimensions (SCD)

A dimension attribute changes over time (a customer moves cities), and the fact rows that happened *before* the change need to still report the *old* value when you look at historical data. SCD types describe how much history you keep.

- **Type 0 — retain original**: the attribute is never updated after insert (e.g. `original_signup_channel`). Used for attributes that are conceptually immutable by policy, even if the real world changed.
- **Type 1 — overwrite**: `UPDATE dim_customer SET city = 'Denver' WHERE customer_key = 1;` — no history kept. Correct when the old value has no analytical value (e.g. correcting a typo), wrong when historical fact rows should still attribute to the old value.
- **Type 2 — add a new row with effective-dated validity**: the standard for "I need to know what was true at the time the fact happened."

**Worked Type 2 example.** Add tracking columns to `dim_customer`:

```sql
ALTER TABLE dim_customer ADD COLUMN effective_date DATE;
ALTER TABLE dim_customer ADD COLUMN end_date       DATE;
ALTER TABLE dim_customer ADD COLUMN is_current     BOOLEAN;

-- Backfill the existing row as the first "version"
UPDATE dim_customer
SET effective_date = '2020-01-01', end_date = NULL, is_current = true
WHERE customer_key = 1;
-- customer_key=1: Acme Corp, Austin, TX, effective_date=2020-01-01, end_date=NULL, is_current=true
```

Acme Corp moves from Austin, TX to Denver, CO on 2026-09-01. The Type 2 update is a two-statement sequence — close out the old row, insert a new one with a **new surrogate key**:

```sql
-- Step 1: close the old version (it stops being "current" but is NOT deleted)
UPDATE dim_customer
SET end_date = '2026-08-31', is_current = false
WHERE customer_key = 1 AND is_current = true;

-- Step 2: insert the new version as a NEW row with a NEW surrogate key
INSERT INTO dim_customer
    (customer_key, customer_id, customer_name, city, state, segment,
     effective_date, end_date, is_current)
VALUES
    (501, 'CUST-001', 'Acme Corp', 'Denver', 'CO', 'enterprise',
     '2026-09-01', NULL, true);
```

Fact rows written before 2026-09-01 still carry `customer_key = 1` (Austin) because the fact table's foreign key was never touched — that's the entire mechanism. A fact row inserted today gets the current key (501, Denver) at load time via a lookup on `is_current = true`. The query that correctly attributes historical facts to the dimension value that was true *at the time of the fact* is a straightforward join on the surrogate key (no date-range join needed, because the fact row already points at the version that was current when it was created):

```sql
SELECT
    dc.city,
    SUM(fol.line_amount) AS revenue
FROM fact_order_lines fol
JOIN dim_customer dc ON fol.customer_key = dc.customer_key   -- points at the version active at insert time
WHERE dc.customer_id = 'CUST-001'
GROUP BY dc.city;
-- Austin:  49.95   (orders placed before the move)
-- Denver:   0.00    (no orders yet under the new version)
```

If instead you only have a natural key and need to reconstruct "what was Acme's city on 2026-06-15," that's an effective-dated range join:

```sql
SELECT dc.city
FROM dim_customer dc
WHERE dc.customer_id = 'CUST-001'
  AND '2026-06-15' >= dc.effective_date
  AND ('2026-06-15' < dc.end_date OR dc.end_date IS NULL);
-- Austin
```

**SCD Type 2 explosion**: a dimension with several volatile attributes (e.g. `dim_customer` also tracking `credit_score` that changes weekly) generates a new full-width row on *every* change to *any* tracked attribute, even ones you don't care about historically. Mitigation: split volatile, frequently-changing attributes into a separate **mini-dimension** (e.g. `dim_customer_demographics_band` keyed by coarse buckets) joined directly to the fact table, so the mini-dimension gets Type 2 treatment cheaply (few distinct combinations) while the large, stable `dim_customer` row stays put.

---

## Star vs Snowflake

Snowflaking normalizes a dimension's attributes into their own sub-dimension tables, trading duplicated bytes for extra joins. Concretely, splitting `dim_product`'s `category`/`department` into `dim_category`:

```sql
CREATE TABLE dim_category (
    category_key BIGINT PRIMARY KEY,
    category     VARCHAR,
    department   VARCHAR
);
-- dim_product now carries category_key instead of the two denormalized strings
```

The earlier revenue-by-category query goes from 3 joins (fact → customer, product, date) to 4 (fact → customer, product → category, date) the moment you need to filter or group on `category`. At star-schema scale — a `dim_product` with 50K rows and maybe 200 distinct categories — the "wasted" bytes from repeating `category`/`department` strings on every product row are negligible relative to the fact table (which is usually 100-1000x larger than any dimension), so most Kimball practice snowflakes only when a sub-dimension is reused by multiple parent dimensions (e.g. a shared `dim_geography` under both `dim_customer` and `dim_store`), not for storage savings.

## Star/Kimball vs One Big Table (OBT)

OBT takes denormalization further than even a star schema: fact and dimension columns are pre-joined into a single wide table at ETL time, so BI queries against it do zero joins at all. This has become viable specifically because of how modern columnar engines store and scan data — not despite it. ClickHouse's `MergeTree` and DuckDB's vectorized scan (see `query-engines/clickhouse.md` and `query-engines/duckdb.md`) only read the columns a query actually touches, and dictionary/RLE encoding means a repeated string like `category` or `department` compresses to nearly nothing when it's constant across long runs of rows — the on-disk cost that originally justified normalizing repeated attributes out of a table (spinning-disk era, row-oriented storage, no compression) has mostly disappeared. A widely cited 2022 Fivetran benchmark comparing a star schema to an OBT on the same BI workload found the OBT roughly 25-50% faster across Redshift, Snowflake, and BigQuery, because each engine's join planner has real per-join overhead that a single wide table simply doesn't pay.

The trade-off is maintenance, not runtime performance: an OBT duplicates every dimension attribute across every fact row, so a dimension attribute change (a customer's segment gets recategorized) must be reflected by rebuilding the OBT rather than updating one dimension row, and adding a new fact source or attribute usually means rebuilding the whole wide table rather than adding a table. In practice as of 2026 the two aren't exclusive: the common pattern is to keep the Kimball star (fact + conformed dimensions) as the governed, single-source-of-truth model, and materialize an OBT as a derived serving-layer table (a dbt model or materialized view) for a specific dashboard or workload that benefits from zero-join scans. Star schema remains the more common default for the governed warehouse layer; OBT is a widely used optimization on top of it, not a replacement for declaring grain and conformed dimensions in the first place.

## A Third Alternative: Data Vault

Data Vault (specifically Data Vault 2.0) targets a different problem than either star schema or OBT: **auditability and resilience to source-system change** in the raw/integration layer of a warehouse, not query-time analytics. It splits data into Hubs (business keys), Links (relationships/transactions between hubs), and Satellites (descriptive, time-versioned attributes hanging off a hub or link) — every row is insert-only and carries a load timestamp and record source, so nothing is ever overwritten and the full history of every change is preserved by construction. This makes it well suited to highly volatile, multi-source enterprise environments (frequent schema drift, regulatory audit requirements, many source systems feeding the same business concept) where a Kimball dimension table's Type 2 SCD tracking would be too rigid or too slow to adapt to new sources. The common 2026 pattern is layered, not either/or: land and integrate raw data in a Data Vault, then build Kimball-style dimensional marts (or an OBT) on top of it for BI consumption — Data Vault is optimized for how data is stored and audited, Kimball/OBT for how it's queried.

---

## Key Gotchas

- **Grain decided too late**: once fact rows are loaded at the wrong grain, you cannot add a missing dimension without either duplicating rows (breaks existing `SUM()`s) or leaving new foreign keys null for a subset of rows — both are silent correctness bugs, not errors. Fix requires a full fact-table rebuild plus reprocessing every downstream aggregate. Declare grain in one sentence before writing DDL.
- **SCD Type 2 explosion**: tracking history on a dimension with several independently-volatile attributes multiplies row count — every change to any tracked column, even ones nobody queries historically, creates a full new row. Split volatile attributes into a separate mini-dimension keyed by coarse bands so the large, stable dimension doesn't churn.
- **Fact table dimensionality creep**: adding "just one more" foreign key to a fact table for every plausible future filter eventually produces a fact table with 20+ dimension keys, most low-cardinality or rarely queried together — this bloats every join plan and confuses report authors about which dimensions are actually meaningful for that business process. Keep a fact table's dimensions to what the business process's grain actually implies; push rarely-used low-cardinality flags into a junk dimension instead of one column each.
- **Nullable foreign keys to dimensions ("the unknown member")**: a fact row that arrives before its dimension row exists (late-arriving dimension) or references a deleted/unknown entity should NOT get a null foreign key — that breaks every `INNER JOIN` in downstream queries and silently drops rows from reports. Standard fix: maintain a placeholder "Unknown"/"Not Applicable" row in each dimension with a reserved surrogate key (e.g. `customer_key = -1`), and always point the fact row's foreign key there instead of leaving it null.
- **Snowflaking for storage savings on a small dimension**: normalizing a 50K-row `dim_product` to save a few duplicated category strings buys negligible bytes (dimensions are typically 100-1000x smaller than the fact table) at the cost of an extra join on every query that filters by category — snowflake only when a sub-dimension is genuinely shared across multiple parent dimensions.

---

## See Also

- `query-engines/clickhouse.md` — columnar storage, dictionary/RLE encoding, and granule pruning: the storage-layer mechanics that make OBT's wide-table scans cheap.
- `query-engines/duckdb.md` — vectorized scan and column pruning: same argument, single-node engine.
- `data-formats/apache-iceberg.md`, `data-formats/delta-lake.md` — the table formats OBT and star-schema fact tables both typically land on in a lakehouse.

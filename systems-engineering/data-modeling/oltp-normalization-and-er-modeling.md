# OLTP Data Modeling: Normalization and ER Modeling

## 30-Second Intuition

Entity-Relationship (ER) modeling is how you decide what tables exist and how they connect: entities become tables, attributes become columns, relationships become foreign keys (or a junction table, for many-to-many). Normalization is the separate discipline of checking that each fact lives in exactly one place, so that updating it once is enough — every normal form (1NF, 2NF, 3NF, BCNF) is a specific rule against a specific kind of duplication, each one closing off one specific update anomaly. The single fact that matters most operationally: **normalization is a logical-design decision about correctness (one fact, one place), and it is orthogonal to indexing, which is a physical-design decision about speed** — a perfectly normalized schema with no index on a foreign key still turns every join into a sequential scan. This doc is the OLTP-side companion to `data-modeling/star-schema-dimensional-modeling.md` — that doc's dimensional model deliberately denormalizes for analytical read speed; this doc's normalized model deliberately avoids duplication for transactional write correctness. Same tradeoff, opposite starting points.

---

## ER Modeling Basics: Entities, Attributes, Relationships, Cardinality

An **entity** is a thing the business tracks (`Customer`, `Order`, `Product`). An **attribute** is a fact about that entity (`Customer.email`, `Product.price`). A **relationship** connects two entities, and its **cardinality** says how many rows on one side can pair with how many rows on the other:

| Cardinality | Meaning | Example |
|---|---|---|
| 1:1 | One row on each side matches exactly one row on the other | `User` ↔ `UserProfile` (one profile per user) |
| 1:N | One row on the "one" side matches many rows on the "many" side | `Customer` → `Order` (one customer places many orders) |
| N:M | Many rows on each side can match many rows on the other | `Order` ↔ `Product` (an order contains many products; a product appears on many orders) |

### Worked Example: E-Commerce Domain (Customer, Order, Product)

ASCII ER diagram for a minimal e-commerce domain:

```
┌────────────┐         1      N   ┌──────────┐         N       M   ┌────────────┐
│  Customer  │────────────────────│  Order   │────────────────────│  Product   │
├────────────┤    "places"        ├──────────┤   "contains"        ├────────────┤
│ customer_id│                    │ order_id │                    │ product_id │
│ name       │                    │ order_dt │                    │ name       │
│ email      │                    │ status   │                    │ price      │
└────────────┘                    └──────────┘                    └────────────┘
                                        │
                                        │ N:M resolved via junction table
                                        ▼
                                ┌────────────────┐
                                │   Order_Item    │
                                ├────────────────┤
                                │ order_id  (FK)  │
                                │ product_id(FK)  │
                                │ quantity        │
                                │ unit_price_at_purchase │
                                └────────────────┘
```

Each relationship type turns into a specific table shape:

- **1:1** (`User` ↔ `UserProfile`): put the foreign key on either table (commonly the dependent one) with a `UNIQUE` constraint, so the database itself enforces "at most one profile per user."
- **1:N** (`Customer` → `Order`): put the foreign key on the "many" side. `Order.customer_id` references `Customer.customer_id`. One customer row, many order rows pointing back at it — no junction table needed.
- **N:M** (`Order` ↔ `Product`): neither table can hold a single foreign key, because either side would need to repeat itself (an order with 3 products would need 3 `product_id` columns, or vice versa). The fix is a **junction table** (also called an associative or bridge table) with a composite key of both foreign keys.

```sql
CREATE TABLE customer (
    customer_id  BIGINT PRIMARY KEY,
    name         VARCHAR NOT NULL,
    email        VARCHAR NOT NULL UNIQUE
);

CREATE TABLE product (
    product_id   BIGINT PRIMARY KEY,
    name         VARCHAR NOT NULL,
    price        DECIMAL(10,2) NOT NULL   -- current, live price
);

CREATE TABLE "order" (
    order_id     BIGINT PRIMARY KEY,
    customer_id  BIGINT NOT NULL REFERENCES customer(customer_id),
    order_dt     TIMESTAMP NOT NULL,
    status       VARCHAR NOT NULL
);

CREATE TABLE order_item (
    order_id                BIGINT NOT NULL REFERENCES "order"(order_id),
    product_id               BIGINT NOT NULL REFERENCES product(product_id),
    quantity                 INT NOT NULL,
    unit_price_at_purchase   DECIMAL(10,2) NOT NULL,
    PRIMARY KEY (order_id, product_id)
);
```

**Why `unit_price_at_purchase` lives on `order_item`, not looked up live from `product.price`.** An order placed on 2026-01-01 must show the price the customer actually paid, forever, even if `product.price` changes six times afterward. If `order_item` stored only `product_id` and every receipt/report joined to `product` for the price, a price change today would silently rewrite the historical total of every past order that touched that product — an invoice from January would start showing September's price. This is a temporal-correctness requirement, not a normalization violation: `unit_price_at_purchase` is not redundant with `product.price`, it is a different fact (the price *at a specific point in time*) that happens to share a column name with a fact that changes over time. Storing it on the junction table is the same instinct as `dim_customer`'s SCD Type 2 versioning in `data-modeling/star-schema-dimensional-modeling.md` — both preserve "what was true when the fact happened" against a dimension/reference value that keeps moving.

```sql
-- Sample data showing the point: price changed, old orders are unaffected
INSERT INTO product VALUES (10, 'Widget A', 9.99);
INSERT INTO order_item VALUES (5001, 10, 2, 9.99);   -- purchased at $9.99

UPDATE product SET price = 12.99 WHERE product_id = 10;  -- price rises later

SELECT quantity, unit_price_at_purchase, quantity * unit_price_at_purchase AS line_total
FROM order_item WHERE order_id = 5001;
-- 2, 9.99, 19.98   <- still correct, unaffected by the price change
```

---

## Normalization, Form by Form

Each form fixes one specific kind of duplication and the update anomaly it causes. Each example below builds on the previous one's fix.

### 1NF: Atomic Values, No Repeating Groups

A table violates 1NF when a column packs multiple values into one field:

```sql
-- Violates 1NF: phone_numbers is not atomic
CREATE TABLE customer_bad (
    customer_id   BIGINT PRIMARY KEY,
    name          VARCHAR,
    phone_numbers VARCHAR   -- e.g. '555-1234,555-5678'
);
```

This breaks the moment you need to query by a single phone number (`WHERE phone_numbers LIKE '%555-5678%'` is a slow, unindexable string scan) or add/remove one number without parsing and rewriting the whole field. The fix moves the repeating group into its own table:

```sql
CREATE TABLE customer (
    customer_id BIGINT PRIMARY KEY,
    name        VARCHAR
);

CREATE TABLE phone (
    phone_id    BIGINT PRIMARY KEY,
    customer_id BIGINT NOT NULL REFERENCES customer(customer_id),
    number      VARCHAR NOT NULL
);
```

Now `WHERE number = '555-5678'` is a plain indexed equality lookup, and adding a third phone number is one `INSERT`, not a string edit.

### 2NF: No Partial Dependency on a Composite Key

2NF applies to tables with a **composite** primary key. It requires every non-key column to depend on the *whole* key, not just part of it.

```sql
-- Violates 2NF: course_name depends only on course_id, not on (student_id, course_id)
CREATE TABLE enrollment_bad (
    student_id   BIGINT,
    course_id    BIGINT,
    course_name  VARCHAR,     -- <- partial dependency
    grade        VARCHAR,
    PRIMARY KEY (student_id, course_id)
);
```

`grade` correctly depends on the full pair (a student's grade in a specific course), but `course_name` depends only on `course_id` — it's the same value on every row for that course, regardless of which student. That's the partial dependency, and it produces a concrete update anomaly:

```sql
INSERT INTO enrollment_bad VALUES (1, 100, 'Intro to Databases', 'A');
INSERT INTO enrollment_bad VALUES (2, 100, 'Intro to Databases', 'B');
INSERT INTO enrollment_bad VALUES (3, 100, 'Intro to Databases', 'A-');
-- Course 100 gets renamed to 'Database Systems I'
UPDATE enrollment_bad SET course_name = 'Database Systems I' WHERE course_id = 100;
-- this UPDATE must touch all 3 rows; miss one (e.g. a stale cached row, a
-- partial batch job failure) and the table now disagrees with itself about
-- what course 100 is called
```

The fix splits `course_name` into its own table, keyed on `course_id` alone:

```sql
CREATE TABLE course (
    course_id   BIGINT PRIMARY KEY,
    course_name VARCHAR NOT NULL
);

CREATE TABLE enrollment (
    student_id  BIGINT,
    course_id   BIGINT REFERENCES course(course_id),
    grade       VARCHAR,
    PRIMARY KEY (student_id, course_id)
);
```

Renaming a course is now one row's `UPDATE` in `course`, and every enrollment referencing it sees the new name through the foreign key on the next join. No missed rows, because there's only one row to miss.

### 3NF: No Transitive Dependency

3NF requires that every non-key column depend on the primary key **directly**, not through another non-key column.

```sql
-- Violates 3NF: department_name and department_budget depend on department_id,
-- not on employee_id
CREATE TABLE employee_bad (
    employee_id        BIGINT PRIMARY KEY,
    name               VARCHAR,
    department_id      BIGINT,
    department_name    VARCHAR,   -- <- transitive: employee_id -> department_id -> department_name
    department_budget  DECIMAL
);
```

`department_name` and `department_budget` are facts about the *department*, reached transitively through `department_id`, not facts about the *employee*. The anomaly is the same shape as 2NF's, just via a different path: renaming a department, or updating its budget, requires updating every employee row in that department, and a missed row leaves the table internally inconsistent (two employees in "department 5" disagreeing on the department's name). The fix separates the transitively-dependent facts into their own table:

```sql
CREATE TABLE department (
    department_id     BIGINT PRIMARY KEY,
    department_name   VARCHAR NOT NULL,
    department_budget DECIMAL NOT NULL
);

CREATE TABLE employee (
    employee_id    BIGINT PRIMARY KEY,
    name           VARCHAR,
    department_id  BIGINT REFERENCES department(department_id)
);
```

### BCNF: The Stricter Version, for Multiple Candidate Keys

BCNF requires that for every functional dependency `X → Y` in the table, `X` must be a **superkey** (a column or column-set that uniquely determines every other row). 3NF has a narrow exception BCNF closes: a table can satisfy 3NF's letter while still having a non-superkey determine part of the row, if that determinant happens to be a *candidate key* itself. This only shows up with **overlapping composite candidate keys**:

```sql
-- A tutoring-scheduling table: one row per (student, subject)
-- Each student has exactly one subject they study with a given tutor,
-- and each tutor teaches exactly one subject (a real, if narrow, business rule)
CREATE TABLE tutoring_bad (
    student_id  BIGINT,
    subject     VARCHAR,
    tutor       VARCHAR,
    PRIMARY KEY (student_id, subject)
);
-- Functional dependency tutor -> subject holds (each tutor teaches one subject),
-- but `tutor` alone is not a superkey of this table (student_id isn't determined
-- by tutor) -- this passes 3NF (subject is a prime attribute, part of a candidate
-- key (student_id, subject)) but fails BCNF, because tutor -> subject means a
-- non-superkey determines a column
```

The practical consequence: you can still insert a contradiction (the same tutor listed against two different subjects across two rows) without violating any constraint the table declares, because the `tutor → subject` rule was never enforced structurally. The fix, as with 2NF/3NF, is decomposition — split into a `tutor_subject(tutor PK, subject)` table and reference it — but BCNF violations like this are rarer in practice and often accepted for the extra join they'd otherwise cost, given they only bite on unusual overlapping-key schemas.

### Summary Table

| Normal Form | Rule | Anomaly it prevents | Worked example above |
|---|---|---|---|
| 1NF | Atomic values, no repeating groups per column | Unindexable, unparseable multi-value fields | `phone_numbers` comma-list → `Phone` table |
| 2NF | No non-key column depends on only part of a composite key | Update anomaly from a value duplicated per key-prefix | `course_name` on `(student_id, course_id)` → `Course` table |
| 3NF | No non-key column depends on another non-key column (no transitive dependency) | Update anomaly from a value reachable only through another non-key column | `department_name`/`department_budget` on `Employee` → `Department` table |
| BCNF | Every determinant of a functional dependency is a superkey | Structural insertion of a contradiction that no declared constraint catches | `tutor → subject` on `(student_id, subject)` → `tutor_subject` table |

---

## Keys: Primary, Foreign, Composite, Surrogate vs. Natural

A **primary key** uniquely identifies a row. A **foreign key** references another table's primary key, enforcing referential integrity. A **composite key** is a primary key made of more than one column (`order_item`'s `(order_id, product_id)` above). The open design question is whether the primary key should be a **surrogate** (a meaningless, database-generated value — auto-increment integer or UUID) or a **natural** key (a real business attribute — email, SSN, order number).

| | Surrogate key (auto-increment / UUID) | Natural key (email, SSN, business code) |
|---|---|---|
| Stability | Immune to business-rule changes — the key never has a reason to change | Breaks if the real-world attribute changes ownership or gets reused (an email changes hands after an account is deleted; an SSN is, rarely but really, reused or corrected) |
| Join/lookup cost | Requires a join or extra lookup to recover the human-meaningful value | Business meaning is visible directly in the key — no extra lookup for `WHERE email = ...` style queries |
| Migration risk | None from business-rule changes; the value is meaningless by design | A business-rule change (e.g. "emails should now be case-insensitive," "SSNs must support reuse after N years") can force a primary-key migration across every table with a foreign key to it |
| Storage/index size | Integer surrogate keys are compact and cache-friendly for B-tree indexes; UUIDs (especially v4, non-time-ordered) are 16 bytes and can fragment insert order — see `storage/postgres.md`'s B-tree section on index bloat from non-sequential inserts | Natural keys are often wider (a VARCHAR email vs. a BIGINT), which bloats every foreign key column and every index referencing it |
| Typical choice | Default choice for most OLTP schemas | Acceptable when the attribute is genuinely immutable and short (e.g. an ISO country code), risky otherwise |

The concrete failure case worth internalizing: a schema that used `email` as `Customer`'s primary key, referenced by `Order.customer_email` and a dozen other tables, and later needs to support a user changing their email address (a routine feature request) now faces a cascading `UPDATE` across every referencing table, or an `ON UPDATE CASCADE` foreign key that must fire correctly everywhere, every time. A surrogate `customer_id` sidesteps this: `email` becomes an ordinary `UNIQUE` column on `Customer`, free to change, and no foreign key anywhere needs to know about it.

---

## When to Denormalize, Deliberately

Denormalization is not automatically a mistake — it's a deliberate trade of write complexity for read speed, made after normalizing first, not instead of it.

**Worked example**: a product catalog page renders `product.name`, `product.price`, and `category.category_name` for every product on the page. Normalized, that's a join on every single page load:

```sql
SELECT p.name, p.price, c.category_name
FROM product p
JOIN category c ON p.category_id = c.category_id
WHERE p.category_id = 42;
```

At high read QPS (a catalog page serving thousands of requests/second, each fanning out to dozens of product rows), that join runs on every request. The denormalized alternative stores `category_name` directly on `product`:

```sql
ALTER TABLE product ADD COLUMN category_name VARCHAR;
-- populate once from category, then keep in sync going forward

SELECT name, price, category_name FROM product WHERE category_id = 42;
-- no join at all
```

Keeping the copy in sync is the cost side of the trade: a rare event (renaming a category) now requires either a database trigger on `category` that cascades the rename to every `product` row, or an application-level write-through that does the same thing explicitly. Until that write completes, `product.category_name` is stale relative to `category.category_name` — an eventual-consistency window, not an atomic one. State the trade precisely:

| | Normalized (join every read) | Denormalized (`category_name` copied onto `product`) |
|---|---|---|
| Read cost | One extra join per read, paid at high QPS | Zero extra join — the whole point of the denormalization |
| Write cost | One row updated on rename (`category`) | Every `product` row in that category updated (bulk `UPDATE` or trigger) on rename |
| Consistency | Always consistent — one source of truth | Briefly inconsistent between the `category` rename committing and the `product` copy finishing its update |
| Correct when | Category renames are frequent relative to catalog reads, or strict consistency is required | Reads vastly outnumber category renames, and a rename can tolerate a short propagation delay |

This is the same "start normalized, denormalize on evidence" discipline covered from the analytics side in `data-modeling/star-schema-dimensional-modeling.md`'s One Big Table section — the difference is that OBT denormalizes an entire warehouse fact table for batch-refreshed BI queries, while this is a single column denormalized for an OLTP hot read path. Widely cited system-design-interview guidance (hellointerview.com's data modeling guide) gives the same rule of thumb for interviews: model normalized first, and justify each denormalization with a specific, named read pattern, not as a default starting posture.

---

## Normalization vs. Indexing: Two Different Layers

Normalization is a **logical** design decision — how facts are split across tables so each one lives in exactly one place. Indexing is a **physical** design decision — how the database locates rows fast on disk. Getting the first right buys you nothing on the second: a fully-normalized schema with no index on a foreign key still forces the query planner into a sequential scan for every join.

```
customer ──(customer_id, PK, B-tree by default)──┐
                                                    │  JOIN needs an index on
order ────(customer_id, FK — NOT indexed by default!)──┘  the FK side too
```

Postgres (and most RDBMSes) auto-indexes the primary key but does **not** auto-index a foreign key column — `order.customer_id REFERENCES customer(customer_id)` gets no index unless you add one explicitly:

```sql
CREATE INDEX idx_order_customer_id ON "order" (customer_id);
```

Without that index, `SELECT * FROM "order" WHERE customer_id = 42` and any join from `customer` to `order` degrades to a sequential scan of the entire `order` table, and — per `storage/postgres.md`'s planner-cost walkthrough — the query planner will only pick a sequential scan over an index scan when it estimates that's cheaper, which for an unindexed foreign key it always will be, because there is no index to consider. The same applies to any column normalization pushed into a `WHERE`, `GROUP BY`, or `JOIN ... ON` clause after splitting a table — `department.department_name` in a 3NF schema needs its own index if departments are frequently looked up by name, exactly as `storage/postgres.md`'s B-tree section describes for any frequently-filtered column. Normalizing correctly and indexing correctly are two separate checklists; passing one does not exempt a schema from the other.

---

## Key Gotchas

- **Over-normalizing for flexibility nobody needs**: splitting an attribute into its own table "in case it becomes multi-valued someday" (e.g. a `middle_name` table for a domain that will never have more than one) adds a join to every read for a case that never materializes. Normalize to the level the actual business rules require, not to the maximum level the schema could theoretically support.
- **Forgetting to index foreign keys**: a normalized schema is designed around joins, but the database does not auto-index the FK side of a relationship — every unindexed foreign key is a sequential scan waiting for enough rows to matter. See the indexing section above and `storage/postgres.md`'s B-tree/planner discussion for the mechanics.
- **Choosing a natural key that turns out not to be stable**: email and SSN both look permanent until a real-world event (email ownership change, SSN reuse/correction) forces a primary-key migration across every referencing table. Default to a surrogate key unless the natural attribute is provably immutable for the lifetime of the record.
- **Denormalizing without a clear update-consistency plan**: copying a value for read speed without deciding — in advance — what updates it, how fast, and what happens to readers during the propagation window turns a deliberate performance trade into a silent-staleness bug. A denormalized column needs a named owner (trigger, application write-through, scheduled job) before it ships, not after a rename goes unnoticed.
- **Confusing "no update anomaly" with "no application bug"**: normalization prevents a class of *data*-consistency bugs (missed rows during multi-row updates); it does nothing to prevent application-level bugs like a missing `WHERE` clause or a race between two concurrent writers. Normalize for the former, use transactions/locking for the latter.

---

## See Also

- `data-modeling/star-schema-dimensional-modeling.md` — the OLAP-side counterpart: deliberate denormalization at warehouse scale (star schema, grain, SCD Type 0-2, One Big Table), same "normalize first, denormalize on evidence" discipline applied to analytical rather than transactional reads.
- `storage/postgres.md` — B-tree index internals, MVCC/heap tuple versioning, and the cost-based planner's seq-scan-vs-index-scan crossover math referenced in the indexing section above.

---

*Terminology for 1NF/2NF/3NF/BCNF grounded against current (2025-2026) sources including Wikipedia's Second/Third Normal Form articles and standard DBMS references (javatpoint, DigitalOcean) — these definitions are decades-old CS with no drift. "Normalize then selectively denormalize" practical guidance for system-design interviews cross-checked against hellointerview.com's Data Modeling for System Design Interviews guide, algomaster.io, and designgurus.io as of September 2026.*

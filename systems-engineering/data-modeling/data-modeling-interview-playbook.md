# Data Modeling Interview Playbook

## 30-Second Intuition

This is a rehearsal aid, not new content. Every mechanism referenced here is already covered in depth in [`data-modeling/oltp-normalization-and-er-modeling.md`](/systems-engineering/data-modeling/oltp-normalization-and-er-modeling.md) (OLTP) and [`data-modeling/star-schema-dimensional-modeling.md`](/systems-engineering/data-modeling/star-schema-dimensional-modeling.md) (OLAP/Kimball). What those docs don't give you is the interview shape: a question stated the way an interviewer states it, a fully worked answer with a real schema, and a note on what the answer needs to demonstrate to actually land. Read the two docs above first for the underlying mechanics. Use this doc to rehearse turning that mechanics knowledge into a spoken answer under time pressure.

---

## How to Use This Doc

Each question below has three parts:

- **The question**, stated the way it would actually be asked.
- **A worked answer** — a real schema, not a description of an approach.
- **What's being evaluated** — the specific signal an interviewer is listening for, separate from whether the schema is "correct."

---

## Part 1: OLTP Schema Design

### Q1. Design a database schema for an e-commerce system (users, products, orders, payments).

**Worked answer:**

```sql
CREATE TABLE customer (
    customer_id  BIGINT PRIMARY KEY,
    email        VARCHAR NOT NULL UNIQUE,
    name         VARCHAR NOT NULL,
    created_at   TIMESTAMP NOT NULL DEFAULT now()
);

CREATE TABLE product (
    product_id   BIGINT PRIMARY KEY,
    name         VARCHAR NOT NULL,
    price        DECIMAL(10,2) NOT NULL,
    category_id  BIGINT REFERENCES category(category_id)
);

CREATE TABLE "order" (
    order_id     BIGINT PRIMARY KEY,
    customer_id  BIGINT NOT NULL REFERENCES customer(customer_id),
    order_dt     TIMESTAMP NOT NULL,
    status       VARCHAR NOT NULL   -- 'pending', 'paid', 'shipped', 'cancelled'
);

-- N:M between order and product, resolved via a junction table
CREATE TABLE order_item (
    order_id                BIGINT NOT NULL REFERENCES "order"(order_id),
    product_id              BIGINT NOT NULL REFERENCES product(product_id),
    quantity                INT NOT NULL,
    unit_price_at_purchase  DECIMAL(10,2) NOT NULL,
    PRIMARY KEY (order_id, product_id)
);

CREATE TABLE payment (
    payment_id   BIGINT PRIMARY KEY,
    order_id     BIGINT NOT NULL REFERENCES "order"(order_id),
    amount       DECIMAL(10,2) NOT NULL,
    status       VARCHAR NOT NULL,   -- 'authorized', 'captured', 'refunded', 'failed'
    processed_at TIMESTAMP
);
```

Index the foreign keys explicitly (`order.customer_id`, `order_item.product_id`, `payment.order_id`) — see [`oltp-normalization-and-er-modeling.md`](/systems-engineering/data-modeling/oltp-normalization-and-er-modeling.md)'s indexing section for why this isn't automatic.

**What's being evaluated**: whether you spot the `order`↔`product` N:M relationship and resolve it with a junction table without being prompted, whether `unit_price_at_purchase` lands on `order_item` rather than being looked up live from `product.price` (the temporal-correctness point that doc covers in full), and whether `payment` is a separate table from `order` rather than a status column on it — a single order can have multiple payment attempts (a failed charge retried), which a status column can't represent.

---

### Q2. How would you model a many-to-many relationship with an attribute on the relationship itself?

**The question, concretely**: students enroll in courses, and each enrollment has its own `grade` — a fact about the *pairing*, not about the student or the course alone.

**Worked answer:**

```sql
CREATE TABLE student (student_id BIGINT PRIMARY KEY, name VARCHAR NOT NULL);
CREATE TABLE course  (course_id  BIGINT PRIMARY KEY, name VARCHAR NOT NULL);

CREATE TABLE enrollment (
    student_id  BIGINT REFERENCES student(student_id),
    course_id   BIGINT REFERENCES course(course_id),
    grade       VARCHAR,
    enrolled_at TIMESTAMP NOT NULL,
    PRIMARY KEY (student_id, course_id)
);
```

**What's being evaluated**: recognizing that `grade` cannot live on `student` (a student takes many courses, many grades) or on `course` (many students, many grades) — it can only live on the junction table, because it's a fact about the specific pairing. This is the exact 2NF-adjacent reasoning in [`oltp-normalization-and-er-modeling.md`](/systems-engineering/data-modeling/oltp-normalization-and-er-modeling.md)'s normalization section, applied to relationship design instead of a single table's columns.

---

### Q3. When would you denormalize a normalized schema, and how do you manage the consistency risk?

Full mechanics and a worked `product.category_name` example are in [`oltp-normalization-and-er-modeling.md`](/systems-engineering/data-modeling/oltp-normalization-and-er-modeling.md)'s denormalization section. A second worked example, since interviewers often want to hear the reasoning applied to a fresh case rather than a memorized one:

**Worked answer — a social feed's follower count.**

```sql
-- Normalized: follower count is a COUNT() query every time a profile loads
SELECT COUNT(*) FROM follows WHERE followee_id = 42;

-- Denormalized: a follower_count column on user, updated incrementally
ALTER TABLE "user" ADD COLUMN follower_count INT NOT NULL DEFAULT 0;

-- On a new follow:
BEGIN;
INSERT INTO follows (follower_id, followee_id) VALUES (7, 42);
UPDATE "user" SET follower_count = follower_count + 1 WHERE user_id = 42;
COMMIT;
```

**What's being evaluated**: stating the actual trade rather than just the mechanism — `COUNT(*)` over a `follows` table with millions of rows for a popular account is expensive on every profile view, and a profile view happens far more often than a follow/unfollow event. The increment must be in the same transaction as the `follows` insert, or a crash between the two leaves the count permanently wrong with no self-healing mechanism — this is the detail that separates a candidate who's memorized "denormalize for reads" from one who understands the failure mode.

---

### Q4. How do you model a hierarchical or self-referencing relationship (an org chart, or nested categories)?

**Worked answer — self-referencing foreign key:**

```sql
CREATE TABLE employee (
    employee_id  BIGINT PRIMARY KEY,
    name         VARCHAR NOT NULL,
    manager_id   BIGINT REFERENCES employee(employee_id)   -- NULL for the CEO/root
);

-- "Who reports to employee 5, directly?"
SELECT * FROM employee WHERE manager_id = 5;
```

This is cheap to write and cheap to query for one level of depth. It gets expensive for "all of employee 5's reports, at any depth" — that requires a recursive query:

```sql
WITH RECURSIVE org_tree AS (
    SELECT employee_id, name, manager_id FROM employee WHERE employee_id = 5
    UNION ALL
    SELECT e.employee_id, e.name, e.manager_id
    FROM employee e
    JOIN org_tree ot ON e.manager_id = ot.employee_id
)
SELECT * FROM org_tree;
```

**What's being evaluated**: knowing the self-referencing foreign key is the default, correct starting point, and knowing its limitation — a recursive CTE for deep-hierarchy "all descendants" queries is a real per-query cost. A stronger answer names the alternative: a **closure table** (a separate table pre-computing every ancestor-descendant pair, not just direct parent-child) trades write cost (every insert updates the closure table for every ancestor) for read cost (a plain `WHERE ancestor_id = 5` with no recursion), and is the right call when "all descendants" queries are frequent and the hierarchy is deep.

---

### Q5. How would you design a schema to support soft deletes and audit history without losing data?

**Worked answer — two complementary patterns, not one:**

```sql
-- Soft delete: the row still exists, marked as deleted
ALTER TABLE "order" ADD COLUMN deleted_at TIMESTAMP NULL;
-- "Live" orders:
SELECT * FROM "order" WHERE deleted_at IS NULL;

-- Audit log: a separate, append-only table recording every change
CREATE TABLE order_audit_log (
    audit_id     BIGINT PRIMARY KEY,
    order_id     BIGINT NOT NULL,
    action       VARCHAR NOT NULL,   -- 'created', 'updated', 'soft_deleted'
    changed_by   BIGINT,
    changed_at   TIMESTAMP NOT NULL,
    old_values   JSONB,
    new_values   JSONB
);
```

**What's being evaluated**: distinguishing the two patterns' actual purposes instead of treating them as interchangeable. `deleted_at` answers "is this row currently active" and keeps every query's `WHERE` clause simple. `order_audit_log` answers "what happened to this row over time, and who did it" — a compliance/debugging question, not a filtering question. Conflating them (trying to reconstruct full history from a single `deleted_at` flag, or trying to filter "active" rows by querying the audit log) is the wrong-tool-for-the-job answer.

---

### Q6. Model a schema for a URL shortener.

**Worked answer:**

```sql
CREATE TABLE short_url (
    short_code   VARCHAR(10) PRIMARY KEY,   -- the key IS the natural, business-facing value
    long_url     TEXT NOT NULL,
    created_at   TIMESTAMP NOT NULL DEFAULT now(),
    expires_at   TIMESTAMP NULL,
    click_count  BIGINT NOT NULL DEFAULT 0
);
```

**What's being evaluated**: this is a deliberately small schema, so the interesting decision is the key. `short_code` as the primary key is a genuine natural-key case — it's short, immutable once generated, and looking it up *is* the entire read path (`WHERE short_code = 'aZ3xQ1'`), so a surrogate key would only add a useless extra lookup. Contrast this explicitly with [`oltp-normalization-and-er-modeling.md`](/systems-engineering/data-modeling/oltp-normalization-and-er-modeling.md)'s keys section, where email/SSN look similarly attractive as natural keys but turn out not to be stable — `short_code` is different because nothing about the business ever needs to change it once assigned. A stronger answer also flags `click_count` as a deliberately denormalized counter (same reasoning as Q3's follower count) rather than a `COUNT()` over a separate click-events table, and notes that if per-click analytics are needed later (time of click, referrer), a separate `click_event` table is added *in addition to*, not instead of, the counter.

---

## Part 2: OLAP / Dimensional Modeling

### Q7. Design a star schema for analyzing e-commerce sales.

**Worked answer**: reuse the exact fact/dimension layout from [`star-schema-dimensional-modeling.md`](/systems-engineering/data-modeling/star-schema-dimensional-modeling.md)'s worked example — `fact_order_lines` (grain: one row per order line) with foreign keys to `dim_customer`, `dim_product`, `dim_date`, and measures `quantity`/`unit_price`/`line_amount`. State the grain out loud, first, before drawing a single box: "one row per order line — not per order, because I need per-product revenue, and per-order grain can't give me that without duplicating rows."

**What's being evaluated**: whether grain is stated as a decision before the schema is drawn, or discovered as an afterthought once someone asks "what if an order has multiple products." The full worked "wrong grain" failure case (why you cannot patch a wrong-grain fact table by just adding a column) is in the referenced doc's Grain section — reciting that failure mode from memory, unprompted, is a strong signal.

---

### Q8. What grain would you choose for a fact table tracking website page views, and why does grain matter here specifically?

**Worked answer:**

```sql
-- Grain: one row per page view event
CREATE TABLE fact_page_view (
    page_view_id  BIGINT PRIMARY KEY,
    user_key      BIGINT REFERENCES dim_user(user_key),
    page_key      BIGINT REFERENCES dim_page(page_key),
    date_key      INT REFERENCES dim_date(date_key),
    session_id    VARCHAR NOT NULL,   -- degenerate dimension
    view_ts       TIMESTAMP NOT NULL,
    duration_sec  INT
);
```

**What's being evaluated**: the temptation here is to model at "one row per session" or "one row per user per day" for a smaller table — reject that explicitly. Page-view grain is the only grain that supports "which specific pages were viewed, in what order, for how long" — the actual business questions a page-view fact table exists to answer. A session-level or daily-aggregate grain would need a full rebuild the first time someone asks "what's our page-to-page conversion funnel," for the identical reason [`star-schema-dimensional-modeling.md`](/systems-engineering/data-modeling/star-schema-dimensional-modeling.md)'s wrong-grain order example can't retrofit a `product_key`.

---

### Q9. How would you handle a customer's address changing over time in a dimensional model?

**Worked answer**: this is SCD Type 2, worked in full in [`star-schema-dimensional-modeling.md`](/systems-engineering/data-modeling/star-schema-dimensional-modeling.md) — close out the old dimension row (`end_date`, `is_current = false`), insert a new row with a **new surrogate key** and the updated address, and leave existing fact rows pointing at the old surrogate key untouched. State the mechanism precisely when asked: "I don't update the existing row in place, because fact rows written before the move need to keep pointing at the old address. I insert a new row with a new key and let new fact rows pick that key up going forward."

**What's being evaluated**: whether the answer distinguishes SCD Type 2 from a plain `UPDATE` (Type 1). A candidate who says "I'd just update the city column" is describing Type 1 and has not addressed the actual question, which assumes historical fact rows must still report the old address.

---

### Q10. When would you choose a snowflake schema over a star schema?

**Worked answer**: reference [`star-schema-dimensional-modeling.md`](/systems-engineering/data-modeling/star-schema-dimensional-modeling.md)'s Star vs Snowflake section directly — snowflake when a sub-dimension is genuinely shared across multiple parent dimensions (a `dim_geography` referenced by both `dim_customer` and `dim_store`), not for storage savings on a small dimension, since a dimension table is typically 100-1000x smaller than the fact table it serves.

**What's being evaluated**: whether the answer defaults to "normalize the dimensions for storage efficiency" — the wrong instinct here, since dimension-table storage savings are close to irrelevant at typical fact-table scale. The right instinct is shared-subdimension reuse.

---

### Q11. How is a fact table structurally different from a dimension table, and how is each queried differently?

| | Fact table | Dimension table |
|---|---|---|
| Contents | Foreign keys + numeric, additive measures | Descriptive attributes (names, categories, dates) |
| Row count | Large — grows with every business event | Small relative to fact table |
| Typical query role | `SUM()`/`COUNT()` target, joined FROM | `WHERE`/`GROUP BY` source, joined TO |
| Changes over time | Append-only (new events) | Attributes update in place (Type 1) or version (Type 2) |

**What's being evaluated**: a clean answer states that a fact table is queried by aggregating its measures while grouping/filtering on dimension attributes — the query shape in [`star-schema-dimensional-modeling.md`](/systems-engineering/data-modeling/star-schema-dimensional-modeling.md)'s worked query (`SUM(line_amount) ... GROUP BY dc.segment, dp.category`) is the concrete demonstration to point to.

---

### Q12. How would you model a many-to-many relationship inside a dimensional model — for example, a customer having multiple email addresses in a CRM analysis?

**Worked answer — a bridge table, not an OLTP-style junction table:**

```sql
CREATE TABLE bridge_customer_email (
    customer_key  BIGINT REFERENCES dim_customer(customer_key),
    email_key     BIGINT REFERENCES dim_email(email_key),
    weight_factor DECIMAL(5,4)   -- e.g. 1/3 if a customer has 3 emails, to avoid overcounting
);
```

**What's being evaluated**: this is the point in the interview to name the contrast with Q2's OLTP junction table explicitly, because the two look structurally similar and are easy to conflate. An OLTP junction table (`enrollment`) resolves an N:M relationship for transactional correctness — every row matters individually, and there's no aggregation concern. A dimensional **bridge table** resolves the same *shape* of relationship, but the risk it manages is different: joining `fact_orders` to `dim_customer` to `bridge_customer_email` to `dim_email` and summing a measure would count each order once per email address the customer has, silently inflating totals. The `weight_factor` column (or an alternative pattern, a "primary email" flag plus a separate unweighted bridge for non-additive analysis) exists specifically to prevent that double-counting — a concern that has no equivalent in the OLTP junction-table pattern.

---

### Q13. OLTP versus OLAP — when would you use each for the same underlying business data?

**Worked answer**: the same order data lives in both places, modeled differently for different jobs. [`oltp-normalization-and-er-modeling.md`](/systems-engineering/data-modeling/oltp-normalization-and-er-modeling.md)'s `order`/`order_item` schema (normalized, one row per line item, optimized for a single order's create/update/cancel transaction) is what the checkout flow writes to. [`star-schema-dimensional-modeling.md`](/systems-engineering/data-modeling/star-schema-dimensional-modeling.md)'s `fact_order_lines` (denormalized dimensions, optimized for `GROUP BY` across millions of rows) is what a revenue dashboard reads from — usually populated by an ETL/ELT job that reads the OLTP tables and reshapes them into the star, not queried directly against the live transactional schema.

**What's being evaluated**: whether the answer identifies that these are typically two separate physical schemas serving the same business concept, connected by a data pipeline, rather than one schema trying to serve both a checkout transaction and a BI dashboard — a single normalized OLTP schema under real analytical query load is exactly the join-heavy, sequential-scan-prone shape both referenced docs warn against for read-heavy aggregation.

---

## How to Structure Any Data-Modeling Answer in an Interview

1. **Clarify access patterns and query volume first.** Ask what the read/write ratio looks like and what the highest-frequency query actually is — the whole rest of the answer depends on this, and asking signals you know it matters before drawing a single table.
2. **Identify entities and relationships.** Name the nouns, then the cardinality between each pair (1:1, 1:N, N:M) — this determines table shape before any column is chosen.
3. **Normalize for OLTP, or pick a grain for OLAP.** State which regime the question is actually in; a schema for a transactional write path and a schema for an analytical read path are different exercises with different correct answers.
4. **Call out one deliberate denormalization or tradeoff decision.** Every real schema has at least one place where a strict rule was intentionally broken for a stated reason — naming it, and the consistency cost it carries, is a stronger signal than a schema with no tradeoffs mentioned at all.
5. **Close with indexing/partitioning as the physical-design layer.** State explicitly that normalization/grain decisions are logical design and indexing is a separate, physical-design decision that still has to happen — see [`oltp-normalization-and-er-modeling.md`](/systems-engineering/data-modeling/oltp-normalization-and-er-modeling.md)'s normalization-vs-indexing section for the exact framing.

---

## Key Gotchas

- **Reciting normal-form names without a worked anomaly example.** Saying "this violates 3NF" without being able to state the specific update anomaly it causes reads as memorized vocabulary, not understanding — always follow a normal-form claim with the concrete "if you don't fix this, here's what breaks" example.
- **Designing a polished star schema without ever asking what grain the business needs.** A structurally perfect-looking star schema at the wrong grain is a wrong answer, not a mostly-right one — grain has to be stated and justified before the fact table's columns are chosen.
- **Confusing OLTP junction tables with OLAP bridge tables.** They look identical on a whiteboard (two foreign keys, maybe an extra column) and solve genuinely different problems — a junction table is about transactional correctness for an N:M relationship, a bridge table is about avoiding double-counting during aggregation. Q12 above is the concrete case to have ready.
- **Treating denormalization as a default instead of an exception.** An interviewer who hears "I'd denormalize this for performance" as the first move, before a normalized version is even described, is hearing a shortcut, not a tradeoff decision — always state the normalized form first, then justify the specific deviation.

---

*Question patterns above reflect commonly-recognized data-modeling interview themes as of 2026 (schema design for a familiar domain, N:M resolution, hierarchy modeling, SCD handling, star-vs-snowflake, OLTP-vs-OLAP framing) cross-checked against system-design-interview guidance from hellointerview.com, algomaster.io, and designgurus.io — the same sources cited in [`oltp-normalization-and-er-modeling.md`](/systems-engineering/data-modeling/oltp-normalization-and-er-modeling.md)'s denormalization section. No single specific question bank is claimed as the exclusive source; these are broadly recurring patterns across multiple public interview-prep resources, not a verbatim reproduction of any one guide.*

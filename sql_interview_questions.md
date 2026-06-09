# SQL Interview Questions — Solution Architect Perspective
## Testing Conceptual Understanding, Reasoning & Problem Solving

---

## The Dataset — E-Commerce Company

> All first 15 questions use this dataset. Learn the schema first.

```sql
-- CUSTOMERS
CREATE TABLE customers (
    customer_id   INT PRIMARY KEY,
    name          VARCHAR(100),
    email         VARCHAR(100),
    city          VARCHAR(50),
    joined_date   DATE
);

-- PRODUCTS
CREATE TABLE products (
    product_id    INT PRIMARY KEY,
    name          VARCHAR(100),
    category      VARCHAR(50),
    price         DECIMAL(10,2),
    stock         INT
);

-- ORDERS
CREATE TABLE orders (
    order_id      INT PRIMARY KEY,
    customer_id   INT,
    order_date    DATE,
    status        VARCHAR(20),   -- 'placed', 'shipped', 'delivered', 'cancelled'
    total_amount  DECIMAL(10,2)
);

-- ORDER_ITEMS
CREATE TABLE order_items (
    item_id       INT PRIMARY KEY,
    order_id      INT,
    product_id    INT,
    quantity      INT,
    unit_price    DECIMAL(10,2)
);

-- SAMPLE DATA
INSERT INTO customers VALUES
(1, 'Amit Shah',    'amit@gmail.com',   'Ahmedabad', '2022-01-15'),
(2, 'Priya Patel',  'priya@gmail.com',  'Surat',     '2022-03-20'),
(3, 'Rahul Mehta',  'rahul@yahoo.com',  'Mumbai',    '2021-11-10'),
(4, 'Sneha Joshi',  'sneha@gmail.com',  'Ahmedabad', '2023-06-01'),
(5, 'Karan Desai',  'karan@gmail.com',  'Baroda',    '2020-08-25'),
(6, 'Nisha Kapoor', 'nisha@gmail.com',  'Delhi',     '2023-01-10');

INSERT INTO products VALUES
(1, 'iPhone 15',      'Electronics', 79999.00, 50),
(2, 'Samsung TV 55"', 'Electronics', 55000.00, 20),
(3, 'Nike Shoes',     'Footwear',    8999.00,  100),
(4, 'Levi Jeans',     'Clothing',    3499.00,  200),
(5, 'Whirlpool AC',   'Appliances',  35000.00, 15),
(6, 'Boat Earbuds',   'Electronics', 2499.00,  300);

INSERT INTO orders VALUES
(101, 1, '2024-01-10', 'delivered',  79999.00),
(102, 2, '2024-01-15', 'delivered',  8999.00),
(103, 3, '2024-02-01', 'shipped',    55000.00),
(104, 1, '2024-02-10', 'delivered',  3499.00),
(105, 4, '2024-03-05', 'cancelled',  35000.00),
(106, 5, '2024-03-10', 'delivered',  2499.00),
(107, 6, '2024-03-15', 'placed',     8999.00),
(108, 2, '2024-04-01', 'delivered',  79999.00),
(109, 3, '2024-04-10', 'cancelled',  3499.00),
(110, 1, '2024-05-01', 'delivered',  35000.00);

INSERT INTO order_items VALUES
(1, 101, 1, 1, 79999.00),
(2, 102, 3, 1, 8999.00),
(3, 103, 2, 1, 55000.00),
(4, 104, 4, 1, 3499.00),
(5, 105, 5, 1, 35000.00),
(6, 106, 6, 1, 2499.00),
(7, 107, 3, 1, 8999.00),
(8, 108, 1, 1, 79999.00),
(9, 109, 4, 1, 3499.00),
(10,110, 5, 1, 35000.00);
```

---

## PART 1 — Questions on the Dataset (Q1–Q15)

---

### Q1. List all customers from Ahmedabad who joined after 2022.

**What I'm testing:** Basic SELECT, WHERE with multiple conditions, DATE comparison.

```sql
SELECT customer_id, name, email, joined_date
FROM   customers
WHERE  city = 'Ahmedabad'
AND    joined_date > '2022-12-31';
```

**Result:** Sneha Joshi (joined 2023-06-01)

**What a strong answer adds:**
> "I'd also index the `city` column if this query runs frequently —
> full table scan on every request is expensive at scale."

---

### Q2. Find total revenue generated from delivered orders only.

**What I'm testing:** Aggregate functions, WHERE on status, understanding of business logic.

```sql
SELECT SUM(total_amount) AS total_revenue
FROM   orders
WHERE  status = 'delivered';
```

**Result:** 79999 + 8999 + 3499 + 2499 + 79999 + 35000 = **210,994**

**What a strong answer adds:**
> "Cancelled orders should never be counted as revenue.
> In accounting, even 'shipped' isn't revenue until delivered —
> this WHERE clause reflects that business rule correctly."

---

### Q3. How many orders did each customer place? Show customer name, not just ID.

**What I'm testing:** JOIN, GROUP BY, COUNT — the classic combination.

```sql
SELECT   c.name,
         COUNT(o.order_id) AS total_orders
FROM     customers c
LEFT JOIN orders o ON c.customer_id = o.customer_id
GROUP BY c.customer_id, c.name
ORDER BY total_orders DESC;
```

**Why LEFT JOIN, not INNER JOIN?**
> "INNER JOIN would drop customers who never placed an order.
> LEFT JOIN includes them with count = 0.
> As a business, we want to know those customers too — they're
> targets for re-engagement campaigns."

---

### Q4. Find customers who have NEVER placed an order.

**What I'm testing:** LEFT JOIN + NULL check vs NOT IN vs NOT EXISTS — candidate should know all three and explain tradeoffs.

**Method 1 — LEFT JOIN (preferred at scale):**
```sql
SELECT c.name, c.email
FROM   customers c
LEFT JOIN orders o ON c.customer_id = o.customer_id
WHERE  o.order_id IS NULL;
```

**Method 2 — NOT EXISTS:**
```sql
SELECT name, email
FROM   customers c
WHERE  NOT EXISTS (
    SELECT 1 FROM orders o
    WHERE  o.customer_id = c.customer_id
);
```

**Method 3 — NOT IN (avoid with NULLs):**
```sql
SELECT name, email
FROM   customers
WHERE  customer_id NOT IN (SELECT customer_id FROM orders);
-- WARNING: If any customer_id in orders is NULL, this returns nothing!
```

**What a strong answer adds:**
> "NOT IN with a subquery that can return NULLs is a trap.
> NULL IN (1, 2, NULL) = UNKNOWN, not FALSE — so the WHERE
> clause fails silently. I always prefer LEFT JOIN or NOT EXISTS."

---

### Q5. Find the top 3 most ordered products by quantity.

**What I'm testing:** JOIN across 3 tables, GROUP BY, ORDER BY, LIMIT.

```sql
SELECT   p.name,
         p.category,
         SUM(oi.quantity) AS total_quantity_sold
FROM     products p
JOIN     order_items oi ON p.product_id = oi.product_id
JOIN     orders o       ON oi.order_id  = o.order_id
WHERE    o.status != 'cancelled'
GROUP BY p.product_id, p.name, p.category
ORDER BY total_quantity_sold DESC
LIMIT 3;
```

**What a strong answer adds:**
> "I excluded cancelled orders in the WHERE — otherwise we'd
> count products that were ordered but never actually sold.
> That would mislead inventory and sales decisions."

---

### Q6. Find customers who placed more than 1 order and spent over ₹50,000 total.

**What I'm testing:** HAVING vs WHERE — the most commonly confused concept.

```sql
SELECT   c.name,
         COUNT(o.order_id)       AS order_count,
         SUM(o.total_amount)     AS total_spent
FROM     customers c
JOIN     orders o ON c.customer_id = o.customer_id
WHERE    o.status != 'cancelled'
GROUP BY c.customer_id, c.name
HAVING   COUNT(o.order_id) > 1
AND      SUM(o.total_amount) > 50000
ORDER BY total_spent DESC;
```

**HAVING vs WHERE — the core concept:**
> "WHERE filters rows BEFORE grouping.
> HAVING filters groups AFTER aggregation.
> You cannot write `WHERE COUNT(*) > 1` — the count doesn't
> exist yet when WHERE runs. HAVING runs after GROUP BY."

---

### Q7. Show each order with customer name, product name, quantity, and line total.

**What I'm testing:** Multi-table JOIN, calculated columns, reading comprehension.

```sql
SELECT o.order_id,
       c.name             AS customer_name,
       p.name             AS product_name,
       oi.quantity,
       oi.unit_price,
       (oi.quantity * oi.unit_price) AS line_total,
       o.status
FROM   orders o
JOIN   customers   c  ON o.customer_id  = c.customer_id
JOIN   order_items oi ON o.order_id     = oi.order_id
JOIN   products    p  ON oi.product_id  = p.product_id
ORDER BY o.order_date;
```

**What a strong answer adds:**
> "In production I'd add `o.order_date` to the SELECT too —
> any business report without a date is incomplete.
> I'd also consider whether `unit_price` in order_items
> should be stored separately from products.price — it should,
> because product price changes over time but the order history
> must reflect the price at time of purchase."

---

### Q8. Find all orders placed in Q1 2024 (Jan–Mar) with their status breakdown.

**What I'm testing:** Date functions, GROUP BY on multiple columns, BETWEEN.

```sql
SELECT   status,
         COUNT(*) AS order_count,
         SUM(total_amount) AS revenue
FROM     orders
WHERE    order_date BETWEEN '2024-01-01' AND '2024-03-31'
GROUP BY status
ORDER BY order_count DESC;
```

**Gotcha question:** *"Is BETWEEN inclusive?"*
> "Yes — BETWEEN is inclusive on both ends in SQL.
> So '2024-03-31' includes all orders on March 31st.
> But with DATETIME columns (not just DATE), '2024-03-31'
> means midnight — you'd miss orders placed later that day.
> Safe habit: use `order_date < '2024-04-01'` instead."

---

### Q9. Rank customers by total spending using a window function.

**What I'm testing:** Window functions — separates mid-level from senior candidates.

```sql
SELECT c.name,
       SUM(o.total_amount)    AS total_spent,
       RANK() OVER (
           ORDER BY SUM(o.total_amount) DESC
       )                      AS spending_rank
FROM   customers c
JOIN   orders o ON c.customer_id = o.customer_id
WHERE  o.status = 'delivered'
GROUP BY c.customer_id, c.name;
```

**Follow-up: RANK vs DENSE_RANK vs ROW_NUMBER?**
```
Data:   100, 100, 80

RANK()       → 1, 1, 3   (gap after tie)
DENSE_RANK() → 1, 1, 2   (no gap)
ROW_NUMBER() → 1, 2, 3   (unique always, arbitrary for ties)
```
> "For leaderboards where users see their position — DENSE_RANK.
> For 'top 10 products' reports — RANK or DENSE_RANK.
> For pagination — ROW_NUMBER."

---

### Q10. Find products that have never been ordered.

**What I'm testing:** Same as Q4 logic but applied to products — tests if candidate truly understood Q4 or just memorized.

```sql
SELECT p.product_id, p.name, p.category, p.stock
FROM   products p
LEFT JOIN order_items oi ON p.product_id = oi.product_id
WHERE  oi.item_id IS NULL;
```

**What a strong answer adds:**
> "These products are dead inventory — taking up warehouse space
> with zero revenue. The business should see this report weekly.
> I'd also join to orders and filter status != 'cancelled' to
> be precise — a product ordered but then cancelled also
> effectively has no real sale."

---

### Q11. Calculate month-over-month revenue for 2024.

**What I'm testing:** Date formatting, GROUP BY on derived column, trend analysis thinking.

```sql
SELECT   DATE_FORMAT(order_date, '%Y-%m') AS month,
         COUNT(order_id)                  AS total_orders,
         SUM(total_amount)                AS revenue
FROM     orders
WHERE    status = 'delivered'
AND      YEAR(order_date) = 2024
GROUP BY DATE_FORMAT(order_date, '%Y-%m')
ORDER BY month;
```

**PostgreSQL version:**
```sql
SELECT   TO_CHAR(order_date, 'YYYY-MM') AS month, ...
-- or
SELECT   DATE_TRUNC('month', order_date) AS month, ...
```

**What a strong answer adds:**
> "I'd extend this with LAG() window function to compute
> month-over-month growth percentage — much more useful for
> business decisions than raw numbers alone."

---

### Q12. Find duplicate emails in the customers table.

**What I'm testing:** GROUP BY + HAVING for duplicate detection — real-world data quality problem.

```sql
SELECT   email,
         COUNT(*) AS occurrence
FROM     customers
GROUP BY email
HAVING   COUNT(*) > 1;
```

**Follow-up: How would you delete duplicates keeping only the latest record?**
```sql
DELETE FROM customers
WHERE  customer_id NOT IN (
    SELECT MAX(customer_id)
    FROM   customers
    GROUP BY email
);
```

**What a strong answer adds:**
> "Before deleting, I'd run the SELECT first to audit what
> would be removed. In production, I'd also check if those
> duplicates have order history — merging customer records
> is more complex than just deleting one."

---

### Q13. Update all 'shipped' orders older than 30 days to 'delivered'.

**What I'm testing:** UPDATE with subquery / date arithmetic, operational SQL.

```sql
UPDATE orders
SET    status = 'delivered',
       -- In real system: also update a delivered_date column
WHERE  status = 'shipped'
AND    order_date < DATE_SUB(CURDATE(), INTERVAL 30 DAY);

-- PostgreSQL:
-- AND order_date < CURRENT_DATE - INTERVAL '30 days'
```

**What a strong answer adds:**
> "Before running this UPDATE in production, I'd run the
> equivalent SELECT first to see exactly which rows will change.
> I'd also wrap it in a transaction so it can be rolled back
> if something looks wrong. And I'd log the change — who ran it,
> when, and how many rows were affected."

---

### Q14. Find the second highest order value (without using LIMIT/OFFSET).

**What I'm testing:** Subquery thinking, edge cases — classic interview trap.

**Method 1 — Subquery:**
```sql
SELECT MAX(total_amount) AS second_highest
FROM   orders
WHERE  total_amount < (SELECT MAX(total_amount) FROM orders);
```

**Method 2 — Window function (cleaner, handles ties):**
```sql
SELECT total_amount
FROM (
    SELECT total_amount,
           DENSE_RANK() OVER (ORDER BY total_amount DESC) AS rnk
    FROM   orders
) ranked
WHERE rnk = 2;
```

**Why Method 2 is better:**
> "If two orders share the highest value, Method 1 would skip
> both and return the third highest — DENSE_RANK handles ties
> correctly. For 'Nth highest' problems, window functions are
> always safer than nested MAX subqueries."

---

### Q15. Delete all cancelled orders that are older than 1 year. But first show what would be deleted.

**What I'm testing:** Safe DELETE practice — SELECT before DELETE, transactions.

```sql
-- Step 1: Always SELECT first — see what will be deleted
SELECT order_id, customer_id, order_date, total_amount
FROM   orders
WHERE  status = 'cancelled'
AND    order_date < DATE_SUB(CURDATE(), INTERVAL 1 YEAR);

-- Step 2: If output looks correct, run DELETE in transaction
START TRANSACTION;

DELETE FROM order_items
WHERE  order_id IN (
    SELECT order_id FROM orders
    WHERE  status = 'cancelled'
    AND    order_date < DATE_SUB(CURDATE(), INTERVAL 1 YEAR)
);

DELETE FROM orders
WHERE  status = 'cancelled'
AND    order_date < DATE_SUB(CURDATE(), INTERVAL 1 YEAR);

-- Step 3: Verify count deleted, then commit or rollback
COMMIT;  -- or ROLLBACK if something looks wrong
```

**What a strong answer adds:**
> "Child records in order_items must be deleted BEFORE parent
> records in orders — foreign key constraints will block it
> otherwise. I always delete in reverse relationship order.
> And archiving to a `cancelled_orders_archive` table is safer
> than hard delete — data recovery is impossible without it."

---

## PART 2 — Conceptual & Advanced Questions (Q16–Q35)

> No specific dataset. Testing depth of understanding.

---

### Q16. What is the difference between WHERE and HAVING? Give a scenario where using WHERE instead of HAVING causes wrong results.

**Answer:**

| | WHERE | HAVING |
|---|---|---|
| Runs | Before GROUP BY | After GROUP BY |
| Filters | Individual rows | Aggregated groups |
| Can use aggregates? | No | Yes |

**Wrong result scenario:**
```sql
-- WRONG: Trying to filter by aggregate in WHERE
SELECT customer_id, SUM(total_amount)
FROM   orders
WHERE  SUM(total_amount) > 50000   -- ERROR: aggregate not allowed here
GROUP BY customer_id;

-- CORRECT: Use HAVING
SELECT customer_id, SUM(total_amount)
FROM   orders
GROUP BY customer_id
HAVING SUM(total_amount) > 50000;
```

> "A subtle gotcha: you CAN use WHERE to filter before grouping —
> like `WHERE status = 'delivered'` — which is actually more
> efficient than filtering in HAVING, because it reduces the
> rows before aggregation happens."

---

### Q17. Explain all types of JOINs. When would you use each?

**Answer:**

```sql
-- INNER JOIN: Only matching rows in BOTH tables
SELECT * FROM orders o INNER JOIN customers c ON o.customer_id = c.customer_id;
-- Use: When you only care about records that have a match

-- LEFT JOIN: All rows from left, matching from right (NULL if no match)
SELECT * FROM customers c LEFT JOIN orders o ON c.customer_id = o.customer_id;
-- Use: When you want ALL customers, even those with no orders

-- RIGHT JOIN: All rows from right (rare — just flip the tables and use LEFT)
-- FULL OUTER JOIN: All rows from both, NULLs where no match
--   MySQL doesn't support FULL OUTER JOIN natively — use UNION:
SELECT * FROM customers c LEFT  JOIN orders o ON c.customer_id = o.customer_id
UNION
SELECT * FROM customers c RIGHT JOIN orders o ON c.customer_id = o.customer_id;

-- CROSS JOIN: Every row from A with every row from B (Cartesian product)
-- 6 customers × 6 products = 36 rows — use with caution
-- Use: Generating all combinations (size × color matrix)

-- SELF JOIN: Table joins itself
SELECT a.name AS customer, b.name AS referred_by
FROM   customers a
JOIN   customers b ON a.referred_by_id = b.customer_id;
-- Use: Hierarchical data — manager/employee, referral chains
```

---

### Q18. What are indexes? When do they help and when do they hurt?

**Answer:**

> "An index is a separate data structure (usually B-tree) that
> the DB maintains alongside a table to speed up lookups.
> Without an index, every query does a full table scan — O(n).
> With an index, lookups become O(log n)."

**When they help:**
```sql
-- Frequently queried columns
CREATE INDEX idx_orders_status ON orders(status);
CREATE INDEX idx_orders_customer ON orders(customer_id);

-- Composite index for common filter combinations
CREATE INDEX idx_orders_status_date ON orders(status, order_date);
-- This helps: WHERE status = 'delivered' AND order_date > '2024-01-01'
-- Order matters: most selective column first
```

**When they hurt:**
```
1. INSERT/UPDATE/DELETE — every write must update all indexes
   Heavy write tables (logs, events) → fewer indexes
2. Low-cardinality columns — index on boolean column useless
   (only 2 values — DB still scans half the table)
3. Small tables — full scan is faster than index lookup
4. Wildcard leading LIKE — WHERE name LIKE '%amit%' skips the index
   WHERE name LIKE 'amit%' — uses the index (prefix only)
```

---

### Q19. What is a transaction? Explain ACID properties with a real example.

**Answer:**

> "A transaction is a unit of work that must complete fully or
> not at all. The classic example: bank transfer."

```sql
START TRANSACTION;

UPDATE accounts SET balance = balance - 10000 WHERE account_id = 1;
UPDATE accounts SET balance = balance + 10000 WHERE account_id = 2;

COMMIT;   -- both succeed together
-- If anything fails between the two UPDATEs:
ROLLBACK; -- both are reversed — money never disappears
```

**ACID:**

| Property | Meaning | Example |
|---|---|---|
| **Atomicity** | All or nothing | Both updates commit or both rollback |
| **Consistency** | Rules are never violated | Balance can't go negative |
| **Isolation** | Transactions don't see each other mid-flight | Two transfers don't interfere |
| **Durability** | Committed data survives crashes | Restart after power cut — data still there |

---

### Q20. What is a subquery vs a JOIN? When would you choose one over the other?

**Answer:**

```sql
-- Subquery approach
SELECT name FROM customers
WHERE customer_id IN (
    SELECT customer_id FROM orders WHERE status = 'delivered'
);

-- JOIN approach (usually faster)
SELECT DISTINCT c.name
FROM customers c
JOIN orders o ON c.customer_id = o.customer_id
WHERE o.status = 'delivered';
```

**When to use subquery:**
> - Readability — complex logic reads more clearly
> - When result of inner query is a single value (scalar subquery)
> - EXISTS checks — often more readable than joins

**When to use JOIN:**
> - Performance — JOINs are generally optimized better by query planner
> - When you need columns from both tables in the result
> - Correlated subqueries that re-run for every row are expensive

---

### Q21. Explain window functions. How are they different from GROUP BY?

**Answer:**

> "GROUP BY collapses rows into groups — you lose individual row detail.
> Window functions perform calculations across rows without collapsing them.
> Every row stays — it just gets a new computed column."

```sql
-- GROUP BY: Collapses — only one row per customer
SELECT customer_id, SUM(total_amount) FROM orders GROUP BY customer_id;

-- Window function: Keeps all rows, adds aggregate as a column
SELECT order_id,
       customer_id,
       total_amount,
       SUM(total_amount) OVER (PARTITION BY customer_id) AS customer_total,
       total_amount / SUM(total_amount) OVER (PARTITION BY customer_id) AS pct_of_customer_total
FROM orders;
-- Each order row is preserved — PLUS we see its share of that customer's total
```

**Common window functions:**
```sql
ROW_NUMBER() OVER (PARTITION BY customer_id ORDER BY order_date)
RANK()        -- with gaps
DENSE_RANK()  -- no gaps
LAG(col, 1)   -- previous row value
LEAD(col, 1)  -- next row value
SUM/AVG/COUNT OVER (PARTITION BY ... ORDER BY ... ROWS BETWEEN ...)
```

---

### Q22. What is normalization? Explain 1NF, 2NF, 3NF with examples.

**Answer:**

**1NF — No repeating groups, atomic values:**
```
BAD (not 1NF):
customer_id | name  | orders
1           | Amit  | 101, 102, 103   ← multiple values in one cell

GOOD (1NF):
customer_id | name  | order_id
1           | Amit  | 101
1           | Amit  | 102
```

**2NF — 1NF + No partial dependency on composite key:**
```
BAD (not 2NF) — composite key (order_id, product_id):
order_id | product_id | product_name | quantity
101      | 1          | iPhone 15    | 1
↑ product_name depends only on product_id, not the full key

GOOD (2NF): Move product_name to a separate products table
```

**3NF — 2NF + No transitive dependency:**
```
BAD (not 3NF):
order_id | customer_id | customer_city
101      | 1           | Ahmedabad
↑ customer_city depends on customer_id, not order_id

GOOD (3NF): customer_city belongs in the customers table
```

> "Real systems often deliberately denormalize for read performance.
> Reporting tables (data warehouses) are intentionally denormalized.
> OLTP = normalized. OLAP = denormalized."

---

### Q23. What is the difference between TRUNCATE, DELETE, and DROP?

**Answer:**

| Command | What it does | Can ROLLBACK? | WHERE clause? | Resets AUTO_INCREMENT? |
|---|---|---|---|---|
| `DELETE` | Removes rows one by one | Yes | Yes | No |
| `TRUNCATE` | Removes all rows instantly (resets table) | No (in MySQL) | No | Yes |
| `DROP` | Removes the entire table + structure | No | No | N/A |

```sql
DELETE FROM orders WHERE status = 'cancelled';    -- specific rows, logged
TRUNCATE TABLE temp_staging;                       -- wipe all, fast, irreversible
DROP TABLE temp_staging;                           -- table gone entirely
```

> "TRUNCATE is DDL, DELETE is DML — that's why TRUNCATE can't
> be rolled back in MySQL. In PostgreSQL, TRUNCATE can be
> rolled back if inside a transaction."

---

### Q24. What are stored procedures and when should you use them vs application-level code?

**Answer:**

```sql
-- Stored procedure: pre-compiled SQL logic stored in the DB
DELIMITER //
CREATE PROCEDURE get_customer_orders(IN p_customer_id INT)
BEGIN
    SELECT o.order_id, o.order_date, o.status, o.total_amount
    FROM   orders o
    WHERE  o.customer_id = p_customer_id
    ORDER BY o.order_date DESC;
END //
DELIMITER ;

-- Usage
CALL get_customer_orders(1);
```

**Use stored procedures when:**
> - Complex logic that multiple apps/services need (single source of truth)
> - Performance-critical operations — precompiled, no network round trips
> - Batch operations on large datasets inside DB engine

**Use application code when:**
> - Logic involves business rules that change frequently
> - You need unit testability
> - Team is more skilled in app languages than SQL
> - Microservices — DB logic in procedures creates tight coupling

---

### Q25. What is the N+1 query problem? How do you solve it in SQL?

**Answer:**

> "N+1 happens when you fetch N records, then run 1 additional
> query for each record. Instead of 1 query, you run N+1."

```python
# Application code — N+1 problem
orders = db.query("SELECT * FROM orders")   # 1 query → 10 orders
for order in orders:
    customer = db.query(                     # 10 queries — one per order
        f"SELECT * FROM customers WHERE id = {order.customer_id}"
    )
# Total: 11 queries for data that needs only 1
```

**SQL solution — JOIN:**
```sql
-- 1 query does everything
SELECT o.order_id, o.total_amount, c.name, c.email
FROM   orders o
JOIN   customers c ON o.customer_id = c.customer_id;
```

> "In ORMs, this is solved with eager loading:
> Django: `orders.select_related('customer')`
> SQLAlchemy: `joinedload(Order.customer)`
> N+1 is one of the most common performance killers in
> production apps and often goes unnoticed until scale."

---

### Q26. How would you design a schema for a multi-tenant SaaS application?

**Answer:**

> "Three strategies, each with different tradeoffs:"

**Strategy 1 — Shared schema, tenant_id column:**
```sql
-- Every table has a tenant_id column
CREATE TABLE orders (
    order_id    INT PRIMARY KEY,
    tenant_id   INT NOT NULL,    -- ← every table has this
    customer_id INT,
    ...
    INDEX idx_tenant (tenant_id)  -- critical index
);
-- Every query MUST include WHERE tenant_id = ?
-- Risk: forgetting tenant_id filter = data leak between tenants
```

**Strategy 2 — Separate schema per tenant:**
```sql
-- tenant_amazon.orders
-- tenant_flipkart.orders
-- Isolation at schema level, same DB server
-- Migration: run ALTER TABLE across N schemas
```

**Strategy 3 — Separate database per tenant:**
```sql
-- Full isolation, best security
-- Expensive: N × infrastructure cost
-- Use for enterprise customers with strict compliance needs
```

> "Most SaaS starts with Strategy 1 (shared schema).
> High-value enterprise customers get Strategy 3.
> Row-level security in PostgreSQL is the safest implementation
> of Strategy 1 — policies enforced at DB level, not app level."

---

### Q27. What are CTEs (Common Table Expressions)? How are they different from subqueries?

**Answer:**

```sql
-- Subquery version — nested, hard to read
SELECT name FROM customers
WHERE customer_id IN (
    SELECT customer_id FROM orders
    WHERE order_id IN (
        SELECT order_id FROM order_items
        WHERE product_id = 1
    )
);

-- CTE version — readable, named, reusable
WITH buyers_of_iphone AS (
    SELECT DISTINCT o.customer_id
    FROM   orders o
    JOIN   order_items oi ON o.order_id = oi.order_id
    WHERE  oi.product_id = 1
    AND    o.status = 'delivered'
)
SELECT c.name, c.email
FROM   customers c
JOIN   buyers_of_iphone b ON c.customer_id = b.customer_id;
```

**Recursive CTE — for hierarchical data:**
```sql
-- Employee hierarchy: who reports to whom, all levels deep
WITH RECURSIVE org_chart AS (
    -- Base case: top-level manager
    SELECT employee_id, name, manager_id, 1 AS level
    FROM   employees
    WHERE  manager_id IS NULL

    UNION ALL

    -- Recursive case: find direct reports
    SELECT e.employee_id, e.name, e.manager_id, oc.level + 1
    FROM   employees e
    JOIN   org_chart oc ON e.manager_id = oc.employee_id
)
SELECT * FROM org_chart ORDER BY level;
```

> "CTEs don't always improve performance over subqueries —
> the query planner may execute them the same way.
> Their primary value is readability and maintainability."

---

### Q28. What is the difference between UNION and UNION ALL?

**Answer:**

```sql
-- UNION: Combines results, REMOVES duplicates (sorts to deduplicate)
SELECT email FROM customers WHERE city = 'Ahmedabad'
UNION
SELECT email FROM customers WHERE joined_date > '2022-01-01';
-- Any email appearing in both sets appears only once

-- UNION ALL: Combines results, KEEPS duplicates (faster)
SELECT email FROM customers WHERE city = 'Ahmedabad'
UNION ALL
SELECT email FROM customers WHERE joined_date > '2022-01-01';
-- Duplicates remain — but no sorting overhead
```

> "UNION ALL is almost always what you want for performance.
> UNION does a sort + deduplication pass — expensive on large sets.
> Only use UNION when you genuinely need deduplication.
> A common trap: using UNION when UNION ALL would work,
> adding unnecessary overhead to every query execution."

---

### Q29. How would you find performance bottlenecks in a slow SQL query?

**Answer:**

**Step 1 — EXPLAIN / EXPLAIN ANALYZE:**
```sql
EXPLAIN SELECT c.name, SUM(o.total_amount)
FROM customers c
JOIN orders o ON c.customer_id = o.customer_id
GROUP BY c.customer_id;
```

**What to look for in EXPLAIN output:**
```
type = ALL       → Full table scan — missing index
type = ref       → Index being used — good
rows = 500000    → Estimating 500K rows scanned — needs index
Extra = filesort → Sorting without index — add index on ORDER BY column
Extra = Using temporary → Temp table created — complex GROUP BY
```

**Step 2 — Check slow query log:**
```sql
SHOW VARIABLES LIKE 'slow_query_log';
SET GLOBAL slow_query_log = 'ON';
SET GLOBAL long_query_time = 1;  -- log queries taking > 1 second
```

**Step 3 — Fix options:**
```sql
-- Add missing index
CREATE INDEX idx_orders_customer_status ON orders(customer_id, status);

-- Rewrite query to avoid full scan
-- Break complex query into smaller steps with CTEs
-- Consider query result caching at application layer (Redis)
-- Consider read replica for heavy analytics queries
```

---

### Q30. What is a deadlock in SQL? How do you prevent it?

**Answer:**

> "A deadlock happens when two transactions each hold a lock
> the other needs — both wait forever."

```
Transaction A: Locks orders row 101, wants customers row 1
Transaction B: Locks customers row 1, wants orders row 101
→ Both wait. Neither can proceed. Deadlock.
```

**Prevention strategies:**
```sql
-- 1. Always access tables in the same order across all transactions
-- All transactions: customers → orders → order_items (never reverse)

-- 2. Keep transactions short — lock duration = deadlock risk window
START TRANSACTION;
-- Do ONLY what's needed
COMMIT;  -- release locks as fast as possible

-- 3. Use SELECT ... FOR UPDATE to signal intent early
SELECT * FROM orders WHERE order_id = 101 FOR UPDATE;
-- Acquires lock during SELECT — not a surprise later

-- 4. Use appropriate isolation level
SET TRANSACTION ISOLATION LEVEL READ COMMITTED;
-- Lower isolation = fewer locks = fewer deadlocks
-- But understand the tradeoff (dirty reads, phantoms)
```

> "Most DBs (MySQL InnoDB, PostgreSQL) detect deadlocks
> automatically and kill one transaction. The application
> should catch that error and retry. Prevention is better
> than relying on detection."

---

### Q31. What is the difference between optimistic and pessimistic locking?

**Answer:**

**Pessimistic Locking — lock first, then work:**
```sql
-- Lock the row before reading — others must wait
SELECT * FROM products WHERE product_id = 1 FOR UPDATE;
-- Do your work
UPDATE products SET stock = stock - 1 WHERE product_id = 1;
COMMIT;
-- Use when: High contention, conflicts are frequent
-- Cost: Other transactions wait — reduces throughput
```

**Optimistic Locking — work first, check at commit:**
```sql
-- Add version column to table
ALTER TABLE products ADD COLUMN version INT DEFAULT 0;

-- Read (no lock)
SELECT product_id, stock, version FROM products WHERE product_id = 1;
-- version = 5, stock = 50

-- Update only if version hasn't changed (someone else didn't modify it)
UPDATE products
SET    stock = 49, version = 6
WHERE  product_id = 1 AND version = 5;  -- ← optimistic check

-- If 0 rows affected → someone else changed it → retry
```

> "Optimistic locking is the right choice for most web applications
> where conflicts are rare. Pessimistic locking is for
> financial systems, inventory where two users might buy the
> last item simultaneously."

---

### Q32. Design a schema to handle a product with multiple attributes (e.g., size, color, weight) that varies per product type.

**Answer:**

> "This is the EAV (Entity-Attribute-Value) problem — one of
> the most common schema design challenges."

**Option 1 — EAV Pattern (flexible but slow):**
```sql
CREATE TABLE product_attributes (
    product_id   INT,
    attr_name    VARCHAR(50),   -- 'size', 'color', 'weight'
    attr_value   VARCHAR(200),
    PRIMARY KEY (product_id, attr_name)
);
-- Flexible: any attribute, any product
-- Problem: Can't query 'all red products' efficiently
-- Problem: No data type enforcement — weight stored as string
```

**Option 2 — JSON column (modern, practical):**
```sql
ALTER TABLE products ADD COLUMN attributes JSON;

UPDATE products
SET attributes = '{"color": "black", "storage": "256GB", "ram": "8GB"}'
WHERE product_id = 1;

-- Query inside JSON (MySQL 5.7+, PostgreSQL)
SELECT name FROM products
WHERE JSON_EXTRACT(attributes, '$.color') = 'black';
-- PostgreSQL: WHERE attributes->>'color' = 'black'
```

**Option 3 — Separate tables per category (best performance):**
```sql
CREATE TABLE electronics_specs (
    product_id INT PRIMARY KEY,
    storage    VARCHAR(20),
    ram        VARCHAR(20),
    battery    INT
);
CREATE TABLE clothing_specs (
    product_id INT PRIMARY KEY,
    size       VARCHAR(10),
    color      VARCHAR(30),
    material   VARCHAR(50)
);
-- Best query performance, type safety
-- Cost: Schema change needed for new attributes
```

> "In practice: JSON column for flexibility during early product
> development. Separate spec tables for high-traffic categories
> where you need to filter/sort on attributes."

---

### Q33. What is database sharding? When would you recommend it to a client?

**Answer:**

> "Sharding is horizontal partitioning — splitting data across
> multiple database servers based on a shard key."

```
Without sharding (vertical scaling has limits):
Single DB: 100M orders → slow, single point of failure

With sharding by customer_id:
Shard 1: customer_id 1–2M       (DB Server 1)
Shard 2: customer_id 2M–4M      (DB Server 2)
Shard 3: customer_id 4M–6M      (DB Server 3)
```

**Sharding strategies:**
```
Range sharding:  customer_id 1–1M on shard 1, 1M–2M on shard 2
                 Risk: hot shards if recent IDs are more active

Hash sharding:   shard = hash(customer_id) % num_shards
                 Uniform distribution, but range queries are hard

Directory-based: Lookup table maps customer → shard
                 Most flexible, adds a lookup hop
```

**When NOT to recommend sharding:**
> "Sharding is a last resort — it adds massive operational
> complexity. Before sharding, exhaust these options:
> 1. Read replicas for read-heavy workloads
> 2. Caching layer (Redis) for frequent reads
> 3. Query optimization and indexes
> 4. Vertical scaling (bigger machine)
> 5. Table partitioning (same server, split by range/hash)
>
> Only recommend sharding when you've genuinely exhausted all
> other options and have clear evidence of DB being the bottleneck."

---

### Q34. What is the difference between a clustered and non-clustered index?

**Answer:**

**Clustered Index:**
> "The table data IS the index — rows are physically stored
> in the order of the clustered index key.
> There can be only ONE clustered index per table.
> In MySQL InnoDB, the PRIMARY KEY is always the clustered index."

```sql
-- Row data physically stored in order of customer_id
-- Finding customer_id = 500 = O(log n), data is right there
CREATE TABLE customers (
    customer_id INT PRIMARY KEY,  -- ← clustered index in InnoDB
    name VARCHAR(100)
);
```

**Non-Clustered Index:**
> "A separate structure that stores the indexed column values
> and a pointer to the actual row location.
> Multiple non-clustered indexes per table are allowed.
> Lookup = find in index, then follow pointer to actual row (2 steps)."

```sql
CREATE INDEX idx_email ON customers(email);
-- Index stores: email → pointer to row
-- Finding by email: search index → follow pointer → get full row
```

> "Covering index is the optimization: if the index contains
> all columns a query needs, the pointer lookup is skipped entirely."
```sql
CREATE INDEX idx_order_cover ON orders(customer_id, status, total_amount);
-- Query: SELECT total_amount FROM orders WHERE customer_id=1 AND status='delivered'
-- All needed columns are IN the index — no row lookup needed
```

---

### Q35. You have a query that ran fine for 6 months and suddenly became slow. Walk me through how you'd diagnose and fix it.

**What I'm testing:** End-to-end problem-solving, production mindset, systematic thinking.

**Strong answer structure:**

**Step 1 — Don't guess. Measure first.**
```sql
EXPLAIN ANALYZE SELECT ...;  -- see actual execution plan, not estimated
SHOW PROFILES;               -- see where time is being spent
```

**Step 2 — Ask the right questions:**
```
- Did data volume change? (6 months of growth = 10x more rows)
- Did someone drop or rebuild indexes recently?
- Did a schema change happen (new column, type change)?
- Is it slow only at specific times? (concurrent load issue)
- Is it slow for all customers or specific ones? (data skew)
```

**Step 3 — Common root causes and fixes:**
```sql
-- Root cause 1: Table grew — index statistics are stale
ANALYZE TABLE orders;          -- MySQL: rebuild statistics
VACUUM ANALYZE orders;         -- PostgreSQL

-- Root cause 2: Index was dropped or became unusable
SHOW INDEX FROM orders;        -- check indexes still exist
CREATE INDEX ... ;             -- recreate if missing

-- Root cause 3: Query plan changed — DB chose wrong plan
-- Force index hint (temporary fix while investigating):
SELECT * FROM orders USE INDEX (idx_orders_customer) WHERE ...;

-- Root cause 4: Lock contention at peak hours
SHOW PROCESSLIST;              -- see what's blocking
-- Solution: optimize transaction length, add read replica

-- Root cause 5: New data pattern — specific values cause skew
-- Example: 90% of orders have status='delivered'
-- Index on status is now useless (low cardinality)
-- Solution: partial index or composite index
CREATE INDEX idx_orders_pending ON orders(customer_id)
WHERE status IN ('placed', 'shipped');  -- PostgreSQL partial index
```

**What a strong answer adds:**
> "I always reproduce the issue on a staging environment with
> production data volume before touching production.
> And I set up query monitoring (pg_stat_statements in Postgres,
> Performance Schema in MySQL) so next time I see the degradation
> before users do."

---

## Quick Reference Cheat Sheet

| Topic | Key Point |
|---|---|
| WHERE vs HAVING | WHERE = before GROUP BY. HAVING = after. |
| JOIN types | INNER = only matches. LEFT = all left + matches. |
| NULL traps | NULL = UNKNOWN. `x = NULL` is always false. Use `IS NULL`. |
| NOT IN with NULLs | Dangerous — use NOT EXISTS or LEFT JOIN instead. |
| TRUNCATE vs DELETE | TRUNCATE = fast, irreversible, resets counter. DELETE = logged, rollback-able. |
| Index sweet spot | High cardinality column + frequent WHERE/JOIN/ORDER BY. |
| UNION vs UNION ALL | UNION ALL is faster. UNION deduplicates (sorts first). |
| Transactions | ACID. START → work → COMMIT or ROLLBACK. Short = better. |
| N+1 problem | Fetch N, then N more queries. Fix: JOIN or eager load. |
| Window functions | Like GROUP BY but keeps all rows. PARTITION BY = grouping. |
| Sharding | Last resort. Try replicas, cache, indexes first. |
| Deadlock | Always same table access order. Short transactions. |

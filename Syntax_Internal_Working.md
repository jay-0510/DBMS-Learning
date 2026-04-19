# PostgreSQL Complete Guide
## From Basics to Advanced - Industry Perspective

---

# 📚 TABLE OF CONTENTS

## PART 1: BASICS
1. [SQL Basics - The Foundation](#1-sql-basics---the-foundation)
2. [Data Manipulation](#2-data-manipulation---changing-data)
3. [Joins](#3-joins---combining-data)

## PART 2: INTERMEDIATE
4. [Subqueries](#4-subqueries---queries-within-queries)
5. [CTEs (Common Table Expressions)](#5-ctes---common-table-expressions)
6. [Window Functions](#6-window-functions---analytics-powerhouse)
7. [Indexing Strategies](#7-indexing-strategies---performance-game-changer)

## PART 3: ADVANCED
8. [Transactions & ACID](#8-transactions--acid---data-integrity)
9. [Advanced Indexing](#9-advanced-indexing---specialized-index-types)
10. [JSON/JSONB Operations](#10-jsonjsonb---modern-data-handling)
11. [Query Optimization](#11-query-optimization---deep-dive)
12. [Partitioning](#12-partitioning---handling-massive-tables)
13. [Replication & High Availability](#13-replication--high-availability)

---

# PART 1: BASICS

# 1. SQL BASICS - The Foundation

## 1.1 SELECT Queries - The Heart of Data Retrieval

**📌 What it does:** Retrieves data from one or more tables

```sql
-- Basic SELECT
SELECT first_name, last_name, email
FROM users
WHERE is_active = true
ORDER BY created_at DESC
LIMIT 10;
```

### 🧠 How It Works Internally

1. **Query Parser:** PostgreSQL first parses your SQL to check for syntax errors
2. **Query Planner:** Creates multiple execution plans and estimates the cost of each
3. **Executor:** Runs the cheapest plan, fetching data from disk/memory

### 🌍 Real-World Scenario: E-commerce Platform Dashboard

**Scenario:** You're building an admin dashboard that shows recent orders. You have 10 million orders in the database.

**Problem:** Without proper indexing, scanning 10M rows to find the latest 100 orders would take seconds.

**Solution:** Index on created_at column + LIMIT clause

```sql
-- Without index: Sequential scan (SLOW)
SELECT * FROM orders ORDER BY created_at DESC LIMIT 100;

-- Create index
CREATE INDEX idx_orders_created_at ON orders(created_at DESC);

-- Now: Index scan (FAST - milliseconds)
```

**💡 Key Learning:** PostgreSQL uses the index to directly jump to the most recent records, avoiding a full table scan. Think of it like a book index vs. reading every page.

---

## 1.2 WHERE Clause - Filtering Data

```sql
-- Different filter operators
SELECT * FROM products WHERE price > 100;
SELECT * FROM users WHERE email LIKE '%@gmail.com';
SELECT * FROM orders WHERE status IN ('pending', 'processing');
SELECT * FROM events WHERE event_date BETWEEN '2024-01-01' AND '2024-12-31';
SELECT * FROM users WHERE phone_number IS NULL;
```

### 🧠 Internal Execution

- PostgreSQL evaluates WHERE conditions row by row (sequential scan) OR uses indexes
- **LIKE with leading wildcard** (`'%gmail.com'`) cannot use regular B-tree index
- IN clause is converted to OR conditions or uses index lookups

### 🌍 Real-World Scenario: Social Media Platform (Like Twitter/X)

**Scenario:** You need to fetch tweets from users in a specific country with more than 1000 followers.

**Challenge:** You have 500M tweets. Using multiple WHERE conditions efficiently.

```sql
-- Inefficient: Two separate conditions
SELECT t.* FROM tweets t
JOIN users u ON t.user_id = u.id
WHERE u.country = 'US' AND u.follower_count > 1000;

-- Better: Use composite index
CREATE INDEX idx_users_country_followers 
  ON users(country, follower_count);

-- PostgreSQL now uses one index lookup instead of two separate scans
```

**💡 Key Learning:** Column order in composite indexes matters! Put the equality condition (country = 'US') before the range condition (follower_count > 1000).

---

## 1.3 Aggregate Functions - Summarizing Data

```sql
-- Common aggregate functions
SELECT COUNT(*) FROM orders;                    -- Total rows
SELECT COUNT(DISTINCT user_id) FROM orders;     -- Unique users
SELECT SUM(amount) FROM orders;                 -- Total revenue
SELECT AVG(rating) FROM reviews;                -- Average rating
SELECT MIN(price), MAX(price) FROM products;    -- Price range
```

### 🧠 How Aggregates Work

- **COUNT(*):** Just counts rows, doesn't read column data (fast)
- **COUNT(column):** Skips NULL values, must read column data
- PostgreSQL can use index-only scans for COUNT if index covers all needed columns
- AVG/SUM accumulate values in a single pass through the data

### 🌍 Real-World Scenario: Analytics Dashboard for SaaS Product

**Scenario:** You need to show: Total users, Active users today, Average session duration, Total revenue.

**Problem:** Running 4 separate COUNT queries is inefficient.

```sql
-- Inefficient: 4 separate queries
SELECT COUNT(*) FROM users;
SELECT COUNT(*) FROM users WHERE last_active >= CURRENT_DATE;
SELECT AVG(session_duration) FROM sessions;
SELECT SUM(amount) FROM payments;

-- Efficient: Single query with multiple aggregates
SELECT 
  COUNT(*) as total_users,
  COUNT(*) FILTER (WHERE last_active >= CURRENT_DATE) as active_today,
  (SELECT AVG(session_duration) FROM sessions) as avg_session,
  (SELECT SUM(amount) FROM payments) as total_revenue
FROM users;
```

**💡 Key Learning:** Use FILTER clause (PostgreSQL 9.4+) or CASE WHEN inside aggregates to calculate multiple metrics in one pass. This reduces I/O significantly.

---

## 1.4 GROUP BY - Categorizing Data

```sql
-- Group by single column
SELECT country, COUNT(*) as user_count
FROM users
GROUP BY country;

-- Group by multiple columns
SELECT country, city, COUNT(*) as user_count
FROM users
GROUP BY country, city
ORDER BY country, user_count DESC;
```

### 🧠 Internal Execution

- PostgreSQL sorts data by GROUP BY columns or uses hash aggregation
- **Hash Aggregation:** Faster for many groups, requires memory
- **Sort Aggregation:** Used when data is already sorted or memory is limited
- Index on GROUP BY columns can eliminate sorting step

### 🌍 Real-World Scenario: E-commerce Sales Report

**Scenario:** Generate monthly sales report: Total orders, revenue, and average order value per month.

**Challenge:** Efficiently grouping millions of orders by month.

```sql
-- Extract month and group
SELECT 
  DATE_TRUNC('month', order_date) as month,
  COUNT(*) as total_orders,
  SUM(total_amount) as revenue,
  AVG(total_amount) as avg_order_value
FROM orders
WHERE order_date >= '2024-01-01'
GROUP BY DATE_TRUNC('month', order_date)
ORDER BY month DESC;

-- Optimization: Create index on order_date
CREATE INDEX idx_orders_date ON orders(order_date);
```

**💡 Key Learning:** When grouping by date functions (DATE_TRUNC, EXTRACT), PostgreSQL can use index on the base date column for filtering (WHERE), but grouping still requires calculation.

---

## 1.5 HAVING - Filtering Grouped Data

**📌 WHERE filters rows BEFORE grouping, HAVING filters groups AFTER aggregation**

```sql
-- Find countries with more than 1000 users
SELECT country, COUNT(*) as user_count
FROM users
GROUP BY country
HAVING COUNT(*) > 1000
ORDER BY user_count DESC;

-- Combine WHERE and HAVING
SELECT category, AVG(price) as avg_price
FROM products
WHERE is_active = true          -- Filter BEFORE grouping
GROUP BY category
HAVING AVG(price) > 50          -- Filter AFTER grouping
ORDER BY avg_price DESC;
```

### 🌍 Real-World Scenario: Fraud Detection System

**Scenario:** Identify users who made more than 10 failed login attempts in the last hour.

**Logic:** Filter by time first (WHERE), then group by user, then filter groups with >10 attempts (HAVING).

```sql
SELECT 
  user_id,
  COUNT(*) as failed_attempts,
  MAX(attempted_at) as last_attempt
FROM login_attempts
WHERE 
  success = false AND
  attempted_at >= NOW() - INTERVAL '1 hour'
GROUP BY user_id
HAVING COUNT(*) > 10
ORDER BY failed_attempts DESC;
```

**💡 Key Learning:** Always filter as early as possible with WHERE to reduce the data set before expensive GROUP BY operations. HAVING is for conditions that depend on aggregate results.

---

# 2. DATA MANIPULATION - Changing Data

## 2.1 INSERT - Adding New Data

```sql
-- Single row insert
INSERT INTO users (first_name, last_name, email)
VALUES ('John', 'Doe', 'john@example.com');

-- Multiple rows insert (efficient)
INSERT INTO users (first_name, last_name, email)
VALUES 
  ('Alice', 'Smith', 'alice@example.com'),
  ('Bob', 'Jones', 'bob@example.com'),
  ('Carol', 'White', 'carol@example.com');

-- Insert with RETURNING (PostgreSQL feature)
INSERT INTO users (first_name, email)
VALUES ('David', 'david@example.com')
RETURNING id, created_at;  -- Returns the generated ID
```

### 🧠 What Happens Internally

- PostgreSQL writes to WAL (Write-Ahead Log) first for crash recovery
- Data is inserted into the table heap (unsorted storage)
- **All indexes on the table are updated** (this is why INSERT is slower with many indexes!)
- Constraints are checked (PRIMARY KEY, FOREIGN KEY, NOT NULL, CHECK)

### 🌍 Real-World Scenario: User Registration System

**Scenario:** When a user signs up, you need to: 1) Insert user record 2) Get the generated ID 3) Create default settings for that user.

**Challenge:** Doing this efficiently without multiple round-trips to the database.

```sql
-- Inefficient: Two separate queries
INSERT INTO users (email, name) VALUES ('user@example.com', 'John');
-- Then fetch ID in application
SELECT id FROM users WHERE email = 'user@example.com';
-- Then insert settings
INSERT INTO user_settings (user_id, theme) VALUES (?, 'dark');

-- Efficient: Use RETURNING with CTE
WITH new_user AS (
  INSERT INTO users (email, name)
  VALUES ('user@example.com', 'John')
  RETURNING id
)
INSERT INTO user_settings (user_id, theme, language)
SELECT id, 'dark', 'en' FROM new_user
RETURNING *;
```

**💡 Key Learning:** RETURNING clause eliminates extra SELECT queries. CTEs (WITH) allow you to chain operations atomically. This is a PostgreSQL superpower!

---

## 2.2 UPSERT - Insert or Update (ON CONFLICT)

**📌 PostgreSQL-specific feature: Handle duplicate key violations gracefully**

```sql
-- If email exists, update the name; otherwise insert
INSERT INTO users (email, name, updated_at)
VALUES ('user@example.com', 'John Doe', NOW())
ON CONFLICT (email) 
DO UPDATE SET 
  name = EXCLUDED.name,
  updated_at = EXCLUDED.updated_at;

-- Do nothing if conflict (ignore duplicates)
INSERT INTO user_logins (user_id, login_time)
VALUES (123, NOW())
ON CONFLICT (user_id, login_time) DO NOTHING;
```

### 🌍 Real-World Scenario: Product Inventory Sync

**Scenario:** You're syncing product data from an external API. Some products are new, some already exist.

**Solution:** Use UPSERT to insert new products or update existing ones in a single operation.

```sql
INSERT INTO products (sku, name, price, stock, last_synced)
VALUES 
  ('SKU001', 'Product A', 99.99, 50, NOW()),
  ('SKU002', 'Product B', 149.99, 30, NOW())
ON CONFLICT (sku)
DO UPDATE SET
  name = EXCLUDED.name,
  price = EXCLUDED.price,
  stock = EXCLUDED.stock,
  last_synced = EXCLUDED.last_synced;
```

**💡 Key Learning:** UPSERT is atomic and efficient. It's perfect for sync operations, caching, and idempotent APIs. EXCLUDED refers to the row that would have been inserted.

---

## 2.3 UPDATE - Modifying Existing Data

```sql
-- Update single column
UPDATE users 
SET last_login = NOW() 
WHERE id = 123;

-- Update multiple columns
UPDATE products
SET 
  price = price * 1.1,  -- 10% increase
  updated_at = NOW()
WHERE category = 'electronics';

-- Update with RETURNING
UPDATE orders
SET status = 'shipped', shipped_at = NOW()
WHERE id = 456
RETURNING id, status, shipped_at;
```

### 🧠 MVCC - How Updates Work

- **PostgreSQL doesn't update rows in-place** (unlike MySQL)
- It creates a NEW version of the row and marks the old one as dead
- This is **MVCC (Multi-Version Concurrency Control)** - allows readers and writers to not block each other
- Dead rows are cleaned up by VACUUM process

### 🌍 Real-World Scenario: Price Update with Audit Trail

**Scenario:** Update product prices but also log the old prices for audit purposes.

**Challenge:** Atomically update and log in one transaction without race conditions.

```sql
-- Using CTE to update and insert audit log atomically
WITH updated_products AS (
  UPDATE products
  SET price = price * 1.15
  WHERE category = 'premium'
  RETURNING id, price AS old_price, (price * 1.15) AS new_price
)
INSERT INTO price_history (product_id, old_price, new_price, changed_at)
SELECT id, old_price, new_price, NOW()
FROM updated_products;
```

**💡 Key Learning:** CTEs allow complex multi-step operations in one atomic transaction. MVCC means other users reading prices during this update see consistent data (either all old or all new values).

---

## 2.4 DELETE vs TRUNCATE

```sql
-- DELETE: Removes specific rows
DELETE FROM users WHERE last_login < NOW() - INTERVAL '2 years';

-- DELETE with RETURNING
DELETE FROM sessions WHERE expires_at < NOW()
RETURNING session_id;

-- TRUNCATE: Removes ALL rows (fast)
TRUNCATE TABLE temp_data;  -- Much faster than DELETE

-- TRUNCATE with CASCADE
TRUNCATE TABLE orders CASCADE;  -- Also truncates dependent tables
```

### 🧠 DELETE vs TRUNCATE - Critical Differences

| Feature | DELETE | TRUNCATE |
|---------|--------|----------|
| Speed | Slow (row-by-row) | Fast (drops table) |
| WHERE clause | Yes | No |
| Triggers | Fires triggers | No triggers |
| Rollback | Can rollback | Can rollback* |
| Auto-increment | Not reset | Resets to 1 |

**💡 Key Learning:** Use TRUNCATE for clearing entire tables (e.g., staging tables, test data). Use DELETE when you need to selectively remove rows or need triggers to fire.

---

# 3. JOINS - Combining Data from Multiple Tables

Joins are the backbone of relational databases. Understanding how they work internally is crucial for writing efficient queries.

## 3.1 INNER JOIN - Matching Records Only

**📌 Returns only rows where there's a match in BOTH tables**

```sql
-- Basic INNER JOIN
SELECT 
  o.order_id,
  o.order_date,
  u.name,
  u.email
FROM orders o
INNER JOIN users u ON o.user_id = u.id
WHERE o.status = 'completed';
```

### 🧠 Join Execution Methods

PostgreSQL uses three main join algorithms:

1. **Nested Loop Join:** Good for small tables or when one table has few rows
   - For each row in table A, scan table B for matches
   - Fast if B has an index on the join column

2. **Hash Join:** Best for large tables with no indexes
   - Builds hash table of smaller table in memory
   - Scans larger table and probes hash table

3. **Merge Join:** Good when both tables are pre-sorted
   - Requires data sorted on join columns
   - Most efficient when indexes exist on both sides

### 🌍 Real-World Scenario: E-commerce Order Dashboard

**Scenario:** You have orders table (10M rows) and users table (2M rows). Display orders with user info.

**Problem:** Without indexes, this join would take minutes.

**Solution:**

```sql
-- Step 1: Create indexes on join columns
CREATE INDEX idx_orders_user_id ON orders(user_id);
CREATE INDEX idx_users_id ON users(id);  -- Usually exists as PRIMARY KEY

-- Step 2: Now the join uses Index Nested Loop
SELECT o.id, o.total, u.name
FROM orders o
INNER JOIN users u ON o.user_id = u.id
WHERE o.created_at >= '2024-01-01'
LIMIT 100;

-- PostgreSQL plan:
-- 1. Index scan on orders using idx_orders_created_at
-- 2. For each order, index lookup on users using idx_users_id
```

**💡 Key Learning:** Always index foreign key columns! PostgreSQL can choose different join algorithms based on table sizes, available memory, and indexes. Use EXPLAIN ANALYZE to see which algorithm is chosen.

---

## 3.2 LEFT JOIN - All Records from Left Table

**📌 Returns ALL rows from left table + matching rows from right (NULL if no match)**

```sql
-- Find users who haven't placed orders
SELECT 
  u.id,
  u.name,
  COUNT(o.id) as order_count
FROM users u
LEFT JOIN orders o ON u.id = o.user_id
GROUP BY u.id, u.name
HAVING COUNT(o.id) = 0;
```

### 🌍 Real-World Scenario: Marketing Campaign - Inactive Users

**Scenario:** Find users who signed up but never made a purchase. Send them a discount code.

**Logic:** LEFT JOIN users with orders, filter where order is NULL.

```sql
-- Approach 1: Using LEFT JOIN with WHERE
SELECT u.id, u.email, u.created_at
FROM users u
LEFT JOIN orders o ON u.id = o.user_id
WHERE o.id IS NULL
  AND u.created_at < NOW() - INTERVAL '30 days';

-- Approach 2: Using NOT EXISTS (often faster)
SELECT u.id, u.email, u.created_at
FROM users u
WHERE NOT EXISTS (
  SELECT 1 FROM orders o WHERE o.user_id = u.id
)
  AND u.created_at < NOW() - INTERVAL '30 days';
```

**💡 Key Learning:** For 'find records NOT in other table', NOT EXISTS is often faster than LEFT JOIN + IS NULL because PostgreSQL can stop searching as soon as it finds one match. Test both with EXPLAIN ANALYZE!

---

## 3.3 Multiple JOINs - Combining Many Tables

```sql
-- Order details with user and product info
SELECT 
  o.id as order_id,
  u.name as customer_name,
  p.name as product_name,
  oi.quantity,
  oi.price
FROM orders o
INNER JOIN users u ON o.user_id = u.id
INNER JOIN order_items oi ON o.id = oi.order_id
INNER JOIN products p ON oi.product_id = p.id
WHERE o.status = 'completed';
```

### 🌍 Real-World Optimization Challenge: Complex Report with 10 Tables

**Scenario:** You have: orders → order_items → products → categories → brands, users → addresses → cities → states → countries.

**Question:** Which 5 tables need indexes? Which don't?

**🎯 Index Strategy:**

- **INDEX these:** Foreign key columns used in JOINs (order_items.order_id, order_items.product_id, users.country_id, etc.)
- **DON'T INDEX:** Small lookup tables (countries, states) - they fit in memory
- Join order matters: Start with most filtered table (WHERE clause), then join outward

```sql
-- Example: Most selective filter first
SELECT *
FROM orders o                    -- 10M rows
INNER JOIN order_items oi ...   -- 50M rows
WHERE o.created_at >= '2024-12-01'  -- Filters to 100K rows
  AND o.status = 'completed';        -- Filters to 50K rows

-- PostgreSQL starts with orders (smallest after filtering)
-- Then joins to order_items using index
```

**💡 Key Learning:** PostgreSQL's query planner is smart but not magic. Help it by: (1) Creating indexes on join columns (2) Writing selective WHERE clauses (3) Using EXPLAIN to verify the plan.

---

# PART 2: INTERMEDIATE

# 4. SUBQUERIES - Queries Within Queries

A subquery is a SELECT statement nested inside another query. Understanding when and how to use them efficiently is crucial.

## 4.1 Scalar Subqueries - Single Value

**📌 Returns a single value (one row, one column)**

```sql
-- Compare each product price to average
SELECT 
  name,
  price,
  (SELECT AVG(price) FROM products) as avg_price,
  price - (SELECT AVG(price) FROM products) as diff_from_avg
FROM products
WHERE category = 'electronics';
```

### 🧠 Performance Consideration

- **Problem:** Subquery runs for EVERY row in the outer query!
- PostgreSQL is smart: It caches the result if the subquery is not correlated
- Better approach: Calculate once using window functions (covered later)

### 🌍 Real-World Scenario: Sales Dashboard - Above Average Performers

**Scenario:** Show salespeople whose monthly sales exceed the company average.

**Challenge:** Need the average first, then compare each person to it.

```sql
-- Inefficient: Subquery in WHERE runs for each row
SELECT 
  salesperson_id,
  name,
  total_sales
FROM salespeople
WHERE total_sales > (SELECT AVG(total_sales) FROM salespeople);

-- Better: Use WITH to calculate once
WITH avg_sales AS (
  SELECT AVG(total_sales) as avg_amount FROM salespeople
)
SELECT 
  s.salesperson_id,
  s.name,
  s.total_sales,
  a.avg_amount,
  s.total_sales - a.avg_amount as above_avg
FROM salespeople s, avg_sales a
WHERE s.total_sales > a.avg_amount;
```

**💡 Key Learning:** Uncorrelated scalar subqueries are cached by PostgreSQL, but using CTEs makes the query more readable and allows you to reuse the calculated value.

---

## 4.2 Correlated Subqueries - Row-by-Row Execution

**📌 Subquery references columns from outer query (executes for EACH row)**

```sql
-- Find products more expensive than their category average
SELECT 
  p1.name,
  p1.category,
  p1.price
FROM products p1
WHERE p1.price > (
  SELECT AVG(p2.price)
  FROM products p2
  WHERE p2.category = p1.category  -- Correlated: references outer query
);
```

### 🧠 Performance Impact

- **WARNING:** Executes subquery once per row in outer query!
- For 10,000 products across 20 categories = 10,000 subquery executions
- **Solution:** Rewrite using window functions or JOINs

### 🌍 Real-World Scenario: Employee Salary Analysis

**Scenario:** Find employees earning above their department average.

**Problem:** Company has 50,000 employees across 100 departments.

```sql
-- Slow: Correlated subquery (50,000 executions)
SELECT e.name, e.salary, e.department
FROM employees e
WHERE e.salary > (
  SELECT AVG(salary) 
  FROM employees 
  WHERE department = e.department
);

-- Fast: Window function (single pass)
SELECT name, salary, department
FROM (
  SELECT 
    name, 
    salary, 
    department,
    AVG(salary) OVER (PARTITION BY department) as dept_avg
  FROM employees
) subquery
WHERE salary > dept_avg;

-- Alternative: JOIN approach
WITH dept_averages AS (
  SELECT department, AVG(salary) as avg_salary
  FROM employees
  GROUP BY department
)
SELECT e.name, e.salary, e.department
FROM employees e
JOIN dept_averages d ON e.department = d.department
WHERE e.salary > d.avg_salary;
```

**💡 Key Learning:** Correlated subqueries are intuitive but often slow. Rewrite using window functions (1 pass) or JOIN with GROUP BY (2 passes but still faster). Always EXPLAIN ANALYZE to compare!

---

## 4.3 EXISTS vs IN - Checking for Matches

```sql
-- Using IN
SELECT * FROM customers
WHERE id IN (SELECT customer_id FROM orders);

-- Using EXISTS (often faster)
SELECT * FROM customers c
WHERE EXISTS (
  SELECT 1 FROM orders o WHERE o.customer_id = c.id
);
```

### 🧠 EXISTS vs IN - When to Use What

| Aspect | IN | EXISTS |
|--------|-----|---------|
| Returns data? | Materializes list | Returns TRUE/FALSE |
| Short-circuit? | No, reads all rows | Yes, stops at first match |
| NULL handling | Issues with NULL | Works correctly |
| Best for | Small result sets | Large result sets |

**💡 Key Learning:** Prefer EXISTS when checking existence. It's semantically clearer and PostgreSQL can stop searching after finding the first match. IN is fine for small, static lists.

---

# 5. CTEs - Common Table Expressions

CTEs make complex queries readable by breaking them into named, reusable parts. They're like temporary result sets that exist only for the query.

## 5.1 Basic CTEs - Improving Readability

```sql
-- Without CTE: Nested and hard to read
SELECT * FROM (
  SELECT user_id, SUM(amount) as total
  FROM orders
  WHERE status = 'completed'
  GROUP BY user_id
) user_totals
WHERE total > 1000;

-- With CTE: Clear and readable
WITH user_totals AS (
  SELECT user_id, SUM(amount) as total
  FROM orders
  WHERE status = 'completed'
  GROUP BY user_id
)
SELECT * FROM user_totals WHERE total > 1000;
```

### 🌍 Real-World Scenario: Multi-Step Analytics Pipeline

**Scenario:** Calculate: 1) Monthly revenue 2) Growth rate vs previous month 3) Top 10 growing months

**Solution:** Chain multiple CTEs, each building on the previous.

```sql
WITH monthly_revenue AS (
  SELECT 
    DATE_TRUNC('month', order_date) as month,
    SUM(total_amount) as revenue
  FROM orders
  WHERE order_date >= '2023-01-01'
  GROUP BY DATE_TRUNC('month', order_date)
),
growth_rates AS (
  SELECT 
    month,
    revenue,
    LAG(revenue) OVER (ORDER BY month) as prev_month_revenue,
    revenue - LAG(revenue) OVER (ORDER BY month) as growth_amount,
    ROUND(
      (revenue - LAG(revenue) OVER (ORDER BY month)) / 
      LAG(revenue) OVER (ORDER BY month) * 100, 2
    ) as growth_percentage
  FROM monthly_revenue
)
SELECT * FROM growth_rates
WHERE growth_percentage IS NOT NULL
ORDER BY growth_percentage DESC
LIMIT 10;
```

**💡 Key Learning:** CTEs make complex queries maintainable. Each CTE is like a step in your analysis. You can test each step independently, making debugging easier.

---

## 5.2 Multiple CTEs - Building Complex Logic

```sql
-- Multiple CTEs for customer segmentation
WITH 
customer_orders AS (
  SELECT 
    user_id,
    COUNT(*) as order_count,
    SUM(total_amount) as lifetime_value
  FROM orders
  GROUP BY user_id
),
customer_activity AS (
  SELECT 
    user_id,
    MAX(order_date) as last_order_date,
    NOW() - MAX(order_date) as days_since_last_order
  FROM orders
  GROUP BY user_id
),
segmentation AS (
  SELECT 
    co.user_id,
    co.order_count,
    co.lifetime_value,
    ca.days_since_last_order,
    CASE 
      WHEN co.lifetime_value > 10000 THEN 'VIP'
      WHEN co.lifetime_value > 1000 THEN 'Premium'
      WHEN ca.days_since_last_order > 180 THEN 'At Risk'
      ELSE 'Regular'
    END as segment
  FROM customer_orders co
  JOIN customer_activity ca ON co.user_id = ca.user_id
)
SELECT 
  segment,
  COUNT(*) as customer_count,
  AVG(lifetime_value) as avg_ltv,
  AVG(order_count) as avg_orders
FROM segmentation
GROUP BY segment
ORDER BY avg_ltv DESC;
```

**💡 Key Learning:** Each CTE has a clear purpose. This makes the query self-documenting. New team members can understand the business logic by reading the CTE names.

---

## 5.3 Recursive CTEs - Hierarchical Data

**📌 For tree structures, org charts, bill of materials, graph traversal**

```sql
-- Employee hierarchy: Find all reports under a manager
WITH RECURSIVE employee_hierarchy AS (
  -- Base case: Start with the CEO
  SELECT 
    id,
    name,
    manager_id,
    1 as level,
    name as path
  FROM employees
  WHERE manager_id IS NULL
  
  UNION ALL
  
  -- Recursive case: Find direct reports
  SELECT 
    e.id,
    e.name,
    e.manager_id,
    eh.level + 1,
    eh.path || ' > ' || e.name
  FROM employees e
  INNER JOIN employee_hierarchy eh ON e.manager_id = eh.id
)
SELECT * FROM employee_hierarchy
ORDER BY level, path;
```

### 🧠 How Recursive CTEs Work

1. Execute the base case (anchor query) - gets starting rows
2. Execute recursive query using previous iteration's results
3. Repeat step 2 until no new rows are returned
4. UNION ALL combines all iterations

### 🌍 Real-World Scenario: Category Navigation (E-commerce)

**Scenario:** Categories: Electronics > Computers > Laptops > Gaming Laptops. Find all subcategories under 'Electronics'.

**Challenge:** Unknown depth of category tree.

```sql
WITH RECURSIVE category_tree AS (
  -- Start with 'Electronics'
  SELECT 
    id,
    name,
    parent_category_id,
    0 as depth
  FROM categories
  WHERE name = 'Electronics'
  
  UNION ALL
  
  -- Get all children recursively
  SELECT 
    c.id,
    c.name,
    c.parent_category_id,
    ct.depth + 1
  FROM categories c
  JOIN category_tree ct ON c.parent_category_id = ct.id
  WHERE ct.depth < 10  -- Prevent infinite loops
)
SELECT 
  id,
  REPEAT('  ', depth) || name as indented_name,
  depth
FROM category_tree
ORDER BY depth, name;
```

**💡 Key Learning:** Recursive CTEs are perfect for hierarchical data. Always include a depth limit to prevent infinite loops. Use them for org charts, file systems, graph traversal, bill of materials.

---

# 6. WINDOW FUNCTIONS - Analytics Powerhouse

Window functions perform calculations across rows related to the current row. Unlike GROUP BY, they don't collapse rows - each row retains its identity.

## 6.1 Ranking Functions - ROW_NUMBER, RANK, DENSE_RANK

```sql
-- Compare ranking functions
SELECT 
  name,
  department,
  salary,
  ROW_NUMBER() OVER (PARTITION BY department ORDER BY salary DESC) as row_num,
  RANK() OVER (PARTITION BY department ORDER BY salary DESC) as rank,
  DENSE_RANK() OVER (PARTITION BY department ORDER BY salary DESC) as dense_rank
FROM employees
ORDER BY department, salary DESC;
```

### 🧠 Understanding the Differences

| Salary | ROW_NUMBER | RANK | DENSE_RANK |
|--------|------------|------|------------|
| 100000 | 1 | 1 | 1 |
| 95000 (tie) | 2 | 2 | 2 |
| 95000 (tie) | 3 (unique) | 2 (tie) | 2 (no gap) |
| 90000 | 4 | 4 (gap!) | 3 |

- **ROW_NUMBER:** Always unique, arbitrary order for ties
- **RANK:** Ties get same rank, gaps in sequence
- **DENSE_RANK:** Ties get same rank, NO gaps

### 🌍 Real-World Scenario: Leaderboard System (Gaming Platform)

**Scenario:** Show top 10 players per game. If players have same score, they share the rank.

**Decision:** Use DENSE_RANK to ensure we show at least 10 distinct score levels.

```sql
-- Get top 10 per game using DENSE_RANK
WITH ranked_players AS (
  SELECT 
    game_id,
    player_name,
    score,
    DENSE_RANK() OVER (
      PARTITION BY game_id 
      ORDER BY score DESC
    ) as rank
  FROM player_scores
)
SELECT * FROM ranked_players
WHERE rank <= 10
ORDER BY game_id, rank;
```

**💡 Key Learning:** PARTITION BY divides data into groups (like GROUP BY), but rows aren't collapsed. ORDER BY within the window determines ranking. Choose the ranking function based on how you want to handle ties.

---

## 6.2 LAG and LEAD - Accessing Adjacent Rows

**📌 Compare current row with previous (LAG) or next (LEAD) rows**

```sql
-- Calculate day-over-day sales change
SELECT 
  date,
  daily_sales,
  LAG(daily_sales) OVER (ORDER BY date) as prev_day_sales,
  daily_sales - LAG(daily_sales) OVER (ORDER BY date) as change,
  ROUND(
    (daily_sales - LAG(daily_sales) OVER (ORDER BY date)) / 
    LAG(daily_sales) OVER (ORDER BY date) * 100, 2
  ) as pct_change
FROM daily_sales_summary
WHERE date >= '2024-01-01';
```

### 🌍 Real-World Scenario: Stock Price Analysis

**Scenario:** Detect when stock price crosses moving average (buy/sell signal).

**Logic:** Compare today's price to 30-day moving average AND check if yesterday was different.

```sql
WITH stock_with_ma AS (
  SELECT 
    date,
    close_price,
    AVG(close_price) OVER (
      ORDER BY date 
      ROWS BETWEEN 29 PRECEDING AND CURRENT ROW
    ) as ma_30,
    close_price > AVG(close_price) OVER (
      ORDER BY date 
      ROWS BETWEEN 29 PRECEDING AND CURRENT ROW
    ) as above_ma
  FROM stock_prices
),
signals AS (
  SELECT 
    date,
    close_price,
    ma_30,
    above_ma,
    LAG(above_ma) OVER (ORDER BY date) as prev_above_ma,
    CASE 
      WHEN above_ma AND NOT LAG(above_ma) OVER (ORDER BY date) 
        THEN 'BUY SIGNAL'
      WHEN NOT above_ma AND LAG(above_ma) OVER (ORDER BY date) 
        THEN 'SELL SIGNAL'
      ELSE NULL
    END as signal
  FROM stock_with_ma
)
SELECT * FROM signals
WHERE signal IS NOT NULL
ORDER BY date;
```

**💡 Key Learning:** LAG/LEAD eliminate self-joins for comparing consecutive rows. Perfect for time series, detecting changes, calculating deltas. Can specify offset: LAG(value, 2) for 2 rows back.

---

## 6.3 Frame Specifications - ROWS vs RANGE

**📌 Control which rows are included in the window calculation**

```sql
-- 7-day moving average of daily sales
SELECT 
  date,
  daily_revenue,
  AVG(daily_revenue) OVER (
    ORDER BY date
    ROWS BETWEEN 6 PRECEDING AND CURRENT ROW
  ) as moving_avg_7day,
  SUM(daily_revenue) OVER (
    ORDER BY date
    ROWS BETWEEN 6 PRECEDING AND CURRENT ROW
  ) as rolling_sum_7day
FROM daily_revenue;
```

### 🧠 Frame Specifications

**ROWS:** Physical row count
- `ROWS BETWEEN 3 PRECEDING AND CURRENT ROW` = last 4 rows
- Good for: Moving averages, running totals

**RANGE:** Value-based window
- Includes all rows with same ORDER BY value
- Good for: Handling ties consistently

**Common frame specifications:**
```sql
ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW  -- Running total
ROWS BETWEEN CURRENT ROW AND UNBOUNDED FOLLOWING  -- Remaining total
ROWS BETWEEN 1 PRECEDING AND 1 FOLLOWING          -- 3-row window
RANGE BETWEEN INTERVAL '7 days' PRECEDING AND CURRENT ROW
```

**💡 Key Learning:** ROWS is more common and intuitive. RANGE is useful when you need all rows with the same date/value included. Default is RANGE UNBOUNDED PRECEDING AND CURRENT ROW.

---

# 7. INDEXING STRATEGIES - Performance Game Changer

Indexes are the #1 tool for query optimization. Understanding when and how to use them separates junior from senior developers.

## 7.1 B-tree Indexes - The Default Choice

**📌 Default index type in PostgreSQL, works for 90% of use cases**

```sql
-- Create basic index
CREATE INDEX idx_users_email ON users(email);

-- Composite index (column order matters!)
CREATE INDEX idx_orders_user_date ON orders(user_id, order_date);

-- Descending index for ORDER BY ... DESC
CREATE INDEX idx_orders_created_desc ON orders(created_at DESC);
```

### 🧠 How B-tree Indexes Work

- Balanced tree structure - all leaf nodes at same depth
- Each page (node) contains multiple keys and pointers
- Searching is O(log n) - very fast even for millions of rows
- **Perfect for:** Equality (=), range queries (<, >, BETWEEN), sorting

### 🌍 Real-World Scenario: User Login System

**Scenario:** Query: `SELECT * FROM users WHERE email = 'user@example.com'`. 10M users in database.

**Without index:** Sequential scan of 10M rows = ~2 seconds  
**With index:** Index lookup = ~5 milliseconds

```sql
-- Email is UNIQUE, so create unique index
CREATE UNIQUE INDEX idx_users_email ON users(email);

-- Now PostgreSQL can use the index
EXPLAIN ANALYZE
SELECT * FROM users WHERE email = 'user@example.com';

-- Result: Index Scan using idx_users_email (cost=0.43..8.45)
```

**💡 Key Learning:** Unique constraints automatically create indexes. For login systems, always index email/username columns. Index lookup is ~400x faster than sequential scan!

---

## 7.2 Composite Indexes - Column Order Matters!

**📌 Index on multiple columns - order determines what queries can use it**

```sql
-- Index on (user_id, created_at)
CREATE INDEX idx_orders_user_date ON orders(user_id, created_at);

-- These queries CAN use the index:
-- 1. Both columns
SELECT * FROM orders WHERE user_id = 123 AND created_at > '2024-01-01';

-- 2. First column only (leftmost prefix)
SELECT * FROM orders WHERE user_id = 123;

-- These queries CANNOT use the index:
-- 3. Second column only
SELECT * FROM orders WHERE created_at > '2024-01-01';  -- Won't use index!
```

### 🧠 The Leftmost Prefix Rule

Index on (A, B, C) can be used for queries with:

✅ WHERE A  
✅ WHERE A AND B  
✅ WHERE A AND B AND C  
❌ WHERE B (starts from middle)  
❌ WHERE C (starts from end)  
❌ WHERE B AND C (doesn't start with A)

### 🌍 Real-World Decision Making: Analytics Platform Query Patterns

**Scenario:** You have queries that filter by: 1) country only (50% of queries), 2) country + city (30%), 3) country + city + zip (20%)

**Question:** What index order?

**✅ Best Solution:**

```sql
-- Single composite index covers all patterns
CREATE INDEX idx_location_hierarchy 
  ON users(country, city, zip_code);

-- All these queries use the index:
WHERE country = 'US'                           -- 50% of queries
WHERE country = 'US' AND city = 'New York'    -- 30% of queries  
WHERE country = 'US' AND city = 'NY' AND zip = '10001'  -- 20%
```

**💡 Key Learning:** Order composite index columns from most selective to least. Put equality conditions before range conditions. Think about query patterns - one well-designed index can serve multiple query types!

---

## 7.3 Partial Indexes - PostgreSQL Superpower

**📌 Index only a subset of rows - saves space and increases performance**

```sql
-- Only index active users (80% of queries)
CREATE INDEX idx_users_active_email 
  ON users(email) 
  WHERE is_active = true;

-- Only index pending/processing orders (recent, frequently queried)
CREATE INDEX idx_orders_pending 
  ON orders(created_at, user_id) 
  WHERE status IN ('pending', 'processing');

-- Index only recent data
CREATE INDEX idx_logs_recent 
  ON logs(timestamp, level) 
  WHERE timestamp > NOW() - INTERVAL '30 days';
```

### 🌍 Real-World Scenario: Order Management System

**Scenario:** 50M total orders. Only 100K are pending/processing (0.2%). Queries mostly filter on status + created_at.

**Problem:** Full index on 50M rows wastes space and is slower.

```sql
-- Inefficient: Index all 50M orders
CREATE INDEX idx_orders_created ON orders(created_at);  -- 2GB index

-- Efficient: Partial index on active orders only
CREATE INDEX idx_orders_active 
  ON orders(created_at, user_id) 
  WHERE status IN ('pending', 'processing');  -- 4MB index!

-- Query automatically uses partial index
SELECT * FROM orders 
WHERE status = 'pending' 
  AND created_at > NOW() - INTERVAL '7 days';
```

**💡 Key Learning:** Partial indexes are perfect for 'hot' data - recent records, active users, incomplete items. They're smaller, faster to scan, and cheaper to maintain. Think about what you DON'T need to index!

---

## 7.4 When NOT to Index - The Cost of Indexes

**⚠️ More indexes = slower writes! Every INSERT/UPDATE/DELETE must update ALL indexes.**

### Don't Index:

- Small tables (< 1000 rows) - sequential scan is faster
- Low-cardinality columns (e.g., boolean, gender) - index won't be used
- Columns never used in WHERE, JOIN, or ORDER BY
- Write-heavy tables where reads are rare
- **Columns with high update frequency** - index maintenance overhead

### Example: ChatGPT-scale Message Storage

**Scenario:** 800M users, billions of messages. Messages table gets millions of INSERTs per minute.

**Strategy:**
- **Index:** user_id, conversation_id (for retrieval)
- **DON'T index:** message content, timestamps (write throughput > read speed)
- Use partitioning by date instead of indexing timestamps
- Archive old data to reduce active dataset size

**💡 Key Learning:** At scale, every index is a tradeoff between read performance and write throughput. Monitor your index usage with `pg_stat_user_indexes`. Drop unused indexes!

---

# PART 3: ADVANCED

# 8. TRANSACTIONS & ACID - Data Integrity

Transactions ensure your database operations are reliable, consistent, and safe from failures.

## 8.1 What is a Transaction?

**📌 A transaction is a unit of work that either completes entirely or not at all**

```sql
BEGIN;  -- Start transaction

-- Transfer money between accounts
UPDATE accounts SET balance = balance - 100 WHERE account_id = 1;
UPDATE accounts SET balance = balance + 100 WHERE account_id = 2;

COMMIT;  -- Make changes permanent

-- Or if something goes wrong:
ROLLBACK;  -- Undo all changes
```

### 🧠 ACID Properties

**A - Atomicity:** All or nothing - transaction either completes fully or rolls back  
**C - Consistency:** Database moves from one valid state to another  
**I - Isolation:** Concurrent transactions don't interfere with each other  
**D - Durability:** Committed changes survive system crashes

---

## 8.2 Isolation Levels - Controlling Concurrency

PostgreSQL supports 4 isolation levels:

### 1. READ UNCOMMITTED (not really in PostgreSQL)
PostgreSQL treats this as READ COMMITTED

### 2. READ COMMITTED (default)
**Behavior:** Each query sees only committed data  
**Issues:** Non-repeatable reads, phantom reads

```sql
-- Session 1
BEGIN;
SELECT balance FROM accounts WHERE id = 1;  -- Returns 1000

-- Session 2 updates and commits
UPDATE accounts SET balance = 500 WHERE id = 1;
COMMIT;

-- Session 1 reads again
SELECT balance FROM accounts WHERE id = 1;  -- Returns 500 (changed!)
COMMIT;
```

### 3. REPEATABLE READ
**Behavior:** Sees snapshot of data at transaction start  
**Issues:** Phantom reads (new rows can appear)  
**Use case:** Long-running reports that need consistent view

```sql
SET TRANSACTION ISOLATION LEVEL REPEATABLE READ;
BEGIN;

SELECT COUNT(*) FROM orders WHERE user_id = 123;  -- Returns 10

-- Another session inserts a new order
-- But this session still sees 10 orders
SELECT COUNT(*) FROM orders WHERE user_id = 123;  -- Still returns 10

COMMIT;
```

### 4. SERIALIZABLE (strongest)
**Behavior:** Transactions execute as if they were serial  
**Issues:** Serialization failures (need retry logic)  
**Use case:** Financial systems, inventory management

```sql
SET TRANSACTION ISOLATION LEVEL SERIALIZABLE;
BEGIN;

-- If concurrent transaction conflicts, PostgreSQL throws error:
-- ERROR: could not serialize access due to concurrent update
COMMIT;
```

### 🌍 Real-World Scenario: E-commerce Inventory

**Problem:** Two users buy the last item simultaneously

```sql
-- User 1 transaction
BEGIN;
SELECT stock FROM products WHERE id = 123;  -- stock = 1
-- User places order
UPDATE products SET stock = stock - 1 WHERE id = 123;
INSERT INTO orders (product_id, user_id) VALUES (123, 1);
COMMIT;

-- User 2 transaction (happens at same time)
BEGIN;
SELECT stock FROM products WHERE id = 123;  -- stock = 1 (stale read!)
-- User places order
UPDATE products SET stock = stock - 1 WHERE id = 123;  -- stock = 0
INSERT INTO orders (product_id, user_id) VALUES (123, 2);
COMMIT;

-- Result: Stock = -1 (OVERSOLD!)
```

**✅ Solution: Use SELECT FOR UPDATE**

```sql
BEGIN;
SELECT stock FROM products WHERE id = 123 FOR UPDATE;  -- Locks row
-- Other transactions must wait
IF stock > 0 THEN
  UPDATE products SET stock = stock - 1 WHERE id = 123;
  INSERT INTO orders (product_id, user_id) VALUES (123, 1);
END IF;
COMMIT;
```

**💡 Key Learning:** `FOR UPDATE` locks the row, preventing concurrent modifications. Use REPEATABLE READ or SERIALIZABLE for complex multi-step operations. Always handle serialization failures with retry logic.

---

## 8.3 Row-Level Locking

```sql
-- Lock for UPDATE (exclusive lock)
SELECT * FROM orders WHERE id = 123 FOR UPDATE;

-- Lock for SHARE (allows reads, blocks writes)
SELECT * FROM products WHERE id = 456 FOR SHARE;

-- Skip locked rows (non-blocking queue processing)
SELECT * FROM jobs WHERE status = 'pending' 
FOR UPDATE SKIP LOCKED 
LIMIT 1;

-- Wait up to 5 seconds for lock
SELECT * FROM accounts WHERE id = 789 
FOR UPDATE NOWAIT;  -- Or use timeout
```

### 🌍 Real-World Scenario: Job Queue Processing

**Scenario:** Multiple workers processing jobs from a queue

```sql
-- Worker 1
BEGIN;
SELECT * FROM jobs 
WHERE status = 'pending'
ORDER BY created_at
FOR UPDATE SKIP LOCKED
LIMIT 1;
-- Process job
UPDATE jobs SET status = 'completed' WHERE id = ?;
COMMIT;

-- Worker 2 (running simultaneously)
BEGIN;
SELECT * FROM jobs 
WHERE status = 'pending'
ORDER BY created_at
FOR UPDATE SKIP LOCKED  -- Skips locked rows
LIMIT 1;  -- Gets different job!
-- Process job
UPDATE jobs SET status = 'completed' WHERE id = ?;
COMMIT;
```

**💡 Key Learning:** `SKIP LOCKED` prevents workers from waiting on locked rows. Perfect for queue processing, task distribution, and concurrent batch jobs.

---

## 8.4 Deadlocks

**Deadlock:** Two transactions waiting for each other to release locks

```sql
-- Transaction 1
BEGIN;
UPDATE accounts SET balance = balance - 100 WHERE id = 1;  -- Locks row 1
-- Now wants to lock row 2
UPDATE accounts SET balance = balance + 100 WHERE id = 2;

-- Transaction 2 (running simultaneously)
BEGIN;
UPDATE accounts SET balance = balance - 50 WHERE id = 2;   -- Locks row 2
-- Now wants to lock row 1
UPDATE accounts SET balance = balance + 50 WHERE id = 1;

-- PostgreSQL detects deadlock and aborts one transaction:
-- ERROR: deadlock detected
```

### Preventing Deadlocks

1. **Lock in consistent order:** Always lock rows in same order (e.g., by ID)
2. **Keep transactions short:** Minimize time holding locks
3. **Use explicit locking:** `FOR UPDATE` at start of transaction
4. **Handle deadlock errors:** Implement retry logic

```sql
-- Good: Consistent locking order
BEGIN;
UPDATE accounts SET balance = balance - 100 WHERE id = LEAST(1, 2);
UPDATE accounts SET balance = balance + 100 WHERE id = GREATEST(1, 2);
COMMIT;
```

**💡 Key Learning:** Deadlocks are inevitable in high-concurrency systems. Design for them: use consistent ordering, keep transactions short, and implement automatic retries.

---

# 9. ADVANCED INDEXING - Specialized Index Types

## 9.1 GIN Indexes - For Arrays, JSONB, Full-Text Search

**📌 Generalized Inverted Index - perfect for multi-value columns**

```sql
-- Index JSONB column
CREATE INDEX idx_users_metadata ON users USING GIN (metadata);

-- Index array column
CREATE INDEX idx_posts_tags ON posts USING GIN (tags);

-- Full-text search
CREATE INDEX idx_articles_search ON articles USING GIN (to_tsvector('english', content));
```

### 🌍 Real-World Scenario: Tag Search

**Scenario:** Blog posts with tags. Find posts with specific tags.

```sql
-- Without GIN: Sequential scan
SELECT * FROM posts WHERE 'postgresql' = ANY(tags);  -- Slow on 1M posts

-- Create GIN index
CREATE INDEX idx_posts_tags ON posts USING GIN (tags);

-- Now: Fast index lookup
SELECT * FROM posts WHERE tags @> ARRAY['postgresql', 'database'];
```

**Use cases:**
- JSONB queries (`metadata @> '{"premium": true}'`)
- Array contains (`tags @> ARRAY['sql']`)
- Full-text search
- Range types

**💡 Key Learning:** GIN indexes are larger than B-tree but essential for JSONB and array operations. They make "contains" queries fast.

---

## 9.2 GiST Indexes - For Geometric Data, Ranges

**📌 Generalized Search Tree - for ranges, geometry, full-text**

```sql
-- Index geometric data
CREATE INDEX idx_locations_point ON locations USING GiST (coordinates);

-- Index range types
CREATE INDEX idx_reservations_period ON reservations USING GiST (reservation_period);

-- Find overlapping ranges
SELECT * FROM reservations 
WHERE reservation_period && '[2024-12-01, 2024-12-31]'::daterange;
```

### 🌍 Real-World Scenario: Hotel Booking System

**Scenario:** Find available rooms for a date range

```sql
-- Table with booking periods
CREATE TABLE bookings (
  id SERIAL PRIMARY KEY,
  room_id INT,
  booking_period daterange
);

-- GiST index for range queries
CREATE INDEX idx_bookings_period ON bookings USING GiST (booking_period);

-- Find rooms with no overlapping bookings
SELECT room_id FROM rooms
WHERE NOT EXISTS (
  SELECT 1 FROM bookings 
  WHERE room_id = rooms.room_id 
    AND booking_period && '[2024-12-25, 2024-12-30]'::daterange
);
```

**💡 Key Learning:** GiST is perfect for ranges, geometry, and nearest-neighbor searches. Use it for booking systems, geospatial queries, and IP ranges.

---

## 9.3 BRIN Indexes - For Very Large Tables

**📌 Block Range INdex - tiny indexes for huge, naturally ordered tables**

```sql
-- Perfect for time-series data
CREATE INDEX idx_logs_timestamp ON logs USING BRIN (timestamp);

-- Much smaller than B-tree, but only works for ordered data
```

### 🧠 How BRIN Works

- Stores min/max values for blocks of rows (not individual rows)
- Extremely small (1-2% of table size vs 30-50% for B-tree)
- Only works well if data is physically ordered

### 🌍 Real-World Scenario: Log Storage

**Scenario:** 10 billion log entries, always inserted in chronological order

```sql
-- B-tree index: 50GB
CREATE INDEX idx_logs_timestamp_btree ON logs(timestamp);  -- Slow!

-- BRIN index: 50MB (1000x smaller!)
CREATE INDEX idx_logs_timestamp_brin ON logs USING BRIN(timestamp);

-- Query still fast for range scans
SELECT * FROM logs 
WHERE timestamp > NOW() - INTERVAL '1 hour';
```

**When to use BRIN:**
- ✅ Very large tables (billions of rows)
- ✅ Naturally ordered data (timestamps, IDs)
- ✅ Range queries (not equality)
- ❌ NOT for random insertions
- ❌ NOT for equality lookups

**💡 Key Learning:** BRIN is PostgreSQL's secret weapon for massive tables. If your data is append-only and time-ordered, BRIN gives you 99% of B-tree performance at 1% of the size.

---

# 10. JSON/JSONB - Modern Data Handling

## 10.1 JSON vs JSONB

**JSON:** Text storage, keeps original formatting, slower queries  
**JSONB:** Binary storage, faster queries, supports indexing  
**Always use JSONB unless you need exact formatting**

```sql
-- Create table with JSONB
CREATE TABLE users (
  id SERIAL PRIMARY KEY,
  email TEXT,
  metadata JSONB
);

-- Insert JSON data
INSERT INTO users (email, metadata) VALUES 
  ('user@example.com', '{"premium": true, "credits": 100}');
```

---

## 10.2 Querying JSONB

```sql
-- Extract value (returns JSON)
SELECT metadata -> 'premium' FROM users;

-- Extract value as text
SELECT metadata ->> 'premium' FROM users;

-- Nested access
SELECT metadata -> 'subscription' ->> 'plan' FROM users;

-- Check if key exists
SELECT * FROM users WHERE metadata ? 'premium';

-- Check if JSON contains
SELECT * FROM users WHERE metadata @> '{"premium": true}';

-- Get array element
SELECT metadata -> 'tags' -> 0 FROM users;
```

### 🌍 Real-World Scenario: User Preferences

**Scenario:** Store flexible user settings without schema changes

```sql
-- Query users with dark theme
SELECT * FROM users 
WHERE metadata @> '{"preferences": {"theme": "dark"}}';

-- Update nested value
UPDATE users 
SET metadata = jsonb_set(
  metadata, 
  '{preferences,notifications}', 
  'true'
)
WHERE id = 123;

-- Add new field
UPDATE users 
SET metadata = metadata || '{"last_login": "2024-12-15"}'
WHERE id = 123;
```

---

## 10.3 Indexing JSONB

```sql
-- GIN index on entire JSONB column (most common)
CREATE INDEX idx_users_metadata ON users USING GIN (metadata);

-- Index specific path (more efficient)
CREATE INDEX idx_users_premium ON users ((metadata ->> 'premium'));

-- Expression index for nested paths
CREATE INDEX idx_users_plan ON users ((metadata -> 'subscription' ->> 'plan'));
```

### 🌍 Real-World Scenario: Feature Flags

**Scenario:** Store feature flags per user, need fast queries

```sql
-- Table design
CREATE TABLE users (
  id SERIAL PRIMARY KEY,
  email TEXT,
  features JSONB DEFAULT '{}'
);

-- GIN index for contains queries
CREATE INDEX idx_users_features ON users USING GIN (features);

-- Fast query for users with specific feature
SELECT * FROM users 
WHERE features @> '{"beta_access": true}';

-- Count users by feature
SELECT 
  features ->> 'region' as region,
  COUNT(*) as user_count
FROM users
WHERE features ? 'region'
GROUP BY features ->> 'region';
```

**💡 Key Learning:** JSONB gives flexibility without sacrificing performance. Use GIN indexes for contains queries, expression indexes for specific paths. Perfect for user preferences, feature flags, metadata.

---

# 11. QUERY OPTIMIZATION - Deep Dive

## 11.1 EXPLAIN and EXPLAIN ANALYZE

**📌 Your #1 debugging tool**

```sql
-- Show estimated plan (no execution)
EXPLAIN SELECT * FROM orders WHERE user_id = 123;

-- Show actual execution (runs query)
EXPLAIN ANALYZE SELECT * FROM orders WHERE user_id = 123;

-- Detailed output
EXPLAIN (ANALYZE, BUFFERS, VERBOSE) 
SELECT * FROM orders WHERE user_id = 123;
```

### 🧠 Reading EXPLAIN Output

```
Seq Scan on orders  (cost=0.00..180000.00 rows=100 width=50)
  Filter: (user_id = 123)
  
Index Scan using idx_orders_user_id on orders  (cost=0.43..8.45 rows=1 width=50)
  Index Cond: (user_id = 123)
```

**Key metrics:**
- **cost=0.00..180000.00:** Estimated cost (startup..total)
- **rows=100:** Estimated rows returned
- **width=50:** Average row size in bytes
- **actual time=0.123..45.678:** Real execution time (ms)

**Common operations:**
- **Seq Scan:** Full table scan (slow for large tables)
- **Index Scan:** Uses index (fast)
- **Index Only Scan:** Data comes entirely from index (fastest)
- **Bitmap Heap Scan:** Multiple index scans combined
- **Hash Join:** In-memory hash table
- **Nested Loop:** For each row in A, scan B

---

## 11.2 Common Performance Issues

### Issue 1: Missing Index

```sql
-- Slow: Sequential scan
EXPLAIN ANALYZE
SELECT * FROM orders WHERE user_id = 123;
-- Seq Scan (cost=0.00..180000.00)

-- Fix: Add index
CREATE INDEX idx_orders_user_id ON orders(user_id);
-- Index Scan (cost=0.43..8.45)
```

### Issue 2: Function in WHERE Prevents Index Use

```sql
-- Bad: Function on column prevents index use
SELECT * FROM users WHERE LOWER(email) = 'user@example.com';
-- Seq Scan with Filter

-- Good: Functional index
CREATE INDEX idx_users_email_lower ON users (LOWER(email));
-- Now uses index
```

### Issue 3: Type Mismatch

```sql
-- Bad: user_id is integer, '123' is text
SELECT * FROM orders WHERE user_id = '123';
-- Sequential scan (PostgreSQL can't use index!)

-- Good: Proper type
SELECT * FROM orders WHERE user_id = 123;
-- Index scan
```

### Issue 4: OR Condition

```sql
-- Bad: OR prevents index use
SELECT * FROM users WHERE first_name = 'John' OR last_name = 'Doe';
-- Sequential scan

-- Good: UNION
SELECT * FROM users WHERE first_name = 'John'
UNION
SELECT * FROM users WHERE last_name = 'Doe';
-- Two index scans
```

### Issue 5: SELECT *

```sql
-- Bad: Fetches all columns
SELECT * FROM orders WHERE user_id = 123;
-- Can't use index-only scan

-- Good: Select only needed columns
SELECT id, total, created_at FROM orders WHERE user_id = 123;
-- May use index-only scan if index covers these columns
```

---

## 11.3 Query Rewriting Techniques

### Technique 1: Covering Index

```sql
-- Slow: Must access table
SELECT user_id, created_at FROM orders WHERE user_id = 123;

-- Fast: Create covering index
CREATE INDEX idx_orders_user_created ON orders(user_id, created_at);
-- Index Only Scan (doesn't touch table!)
```

### Technique 2: Avoid Subqueries in SELECT

```sql
-- Slow: Subquery runs for each row
SELECT 
  u.name,
  (SELECT COUNT(*) FROM orders WHERE user_id = u.id) as order_count
FROM users u;

-- Fast: Use JOIN
SELECT 
  u.name,
  COUNT(o.id) as order_count
FROM users u
LEFT JOIN orders o ON u.id = o.user_id
GROUP BY u.id, u.name;
```

### Technique 3: Limit Early

```sql
-- Slow: Sorts all rows then limits
SELECT * FROM orders 
ORDER BY created_at DESC 
LIMIT 10;

-- Fast: Use index on created_at
CREATE INDEX idx_orders_created_desc ON orders(created_at DESC);
-- Now uses Index Scan with Limit
```

---

## 11.4 Statistics and VACUUM

**PostgreSQL uses statistics to choose query plans**

```sql
-- Update statistics manually
ANALYZE users;

-- Auto-analyze runs automatically but you can force it
VACUUM ANALYZE users;

-- View statistics
SELECT 
  schemaname,
  tablename,
  n_tup_ins as inserts,
  n_tup_upd as updates,
  n_tup_del as deletes,
  n_live_tup as live_rows,
  n_dead_tup as dead_rows,
  last_autovacuum,
  last_autoanalyze
FROM pg_stat_user_tables
WHERE tablename = 'orders';
```

### 🧠 Why VACUUM Matters

- UPDATE creates new row version (MVCC)
- Old versions become "dead tuples"
- Dead tuples waste space and slow queries
- VACUUM removes dead tuples
- AUTOVACUUM runs automatically but may lag on busy tables

```sql
-- Manual VACUUM
VACUUM users;

-- VACUUM and update statistics
VACUUM ANALYZE users;

-- Aggressive vacuum (reclaims more space, slower)
VACUUM FULL users;  -- Locks table!
```

**💡 Key Learning:** Keep statistics current with ANALYZE. Monitor dead tuples with pg_stat_user_tables. Tune autovacuum for write-heavy tables.

---

# 12. PARTITIONING - Handling Massive Tables

## 12.1 What is Partitioning?

**📌 Split one large table into smaller physical pieces**

Benefits:
- ✅ Faster queries (scan only relevant partitions)
- ✅ Easier maintenance (drop old partitions instead of DELETE)
- ✅ Better vacuum performance
- ✅ Can use different storage for hot/cold data

---

## 12.2 Range Partitioning

**Most common: partition by date ranges**

```sql
-- Create partitioned table (PostgreSQL 10+)
CREATE TABLE orders (
  id SERIAL,
  user_id INT,
  total DECIMAL,
  created_at TIMESTAMP NOT NULL
) PARTITION BY RANGE (created_at);

-- Create partitions
CREATE TABLE orders_2024_01 PARTITION OF orders
  FOR VALUES FROM ('2024-01-01') TO ('2024-02-01');

CREATE TABLE orders_2024_02 PARTITION OF orders
  FOR VALUES FROM ('2024-02-01') TO ('2024-03-01');

CREATE TABLE orders_2024_03 PARTITION OF orders
  FOR VALUES FROM ('2024-03-01') TO ('2024-04-01');
```

### 🌍 Real-World Scenario: Log Storage System

**Scenario:** 100M log entries per day, need to keep 90 days

```sql
-- Partitioned by day
CREATE TABLE logs (
  id BIGSERIAL,
  level TEXT,
  message TEXT,
  created_at TIMESTAMP NOT NULL
) PARTITION BY RANGE (created_at);

-- Indexes on each partition
CREATE INDEX ON logs_2024_12_15 (level, created_at);
CREATE INDEX ON logs_2024_12_16 (level, created_at);

-- Query automatically uses partition pruning
SELECT * FROM logs 
WHERE created_at >= '2024-12-15' 
  AND created_at < '2024-12-16'
  AND level = 'ERROR';
-- Only scans logs_2024_12_15 partition!

-- Drop old partitions (instant)
DROP TABLE logs_2024_09_15;  -- Much faster than DELETE
```

**Partition maintenance script:**
```sql
-- Create tomorrow's partition
CREATE TABLE logs_2024_12_17 PARTITION OF logs
  FOR VALUES FROM ('2024-12-17') TO ('2024-12-18');

-- Drop 91-day old partition
DROP TABLE IF EXISTS logs_2024_09_17;
```

**💡 Key Learning:** Partitioning makes huge tables manageable. Query performance improves through partition pruning. Dropping partitions is instant vs slow DELETE.

---

## 12.3 List Partitioning

**Partition by discrete values**

```sql
-- Partition by country
CREATE TABLE users (
  id SERIAL,
  email TEXT,
  country TEXT NOT NULL
) PARTITION BY LIST (country);

CREATE TABLE users_us PARTITION OF users
  FOR VALUES IN ('US');

CREATE TABLE users_eu PARTITION OF users
  FOR VALUES IN ('UK', 'FR', 'DE', 'IT', 'ES');

CREATE TABLE users_asia PARTITION OF users
  FOR VALUES IN ('JP', 'CN', 'IN', 'KR');
```

---

## 12.4 Hash Partitioning

**Distribute data evenly across partitions**

```sql
-- Partition by hash of user_id (for load balancing)
CREATE TABLE events (
  id BIGSERIAL,
  user_id INT NOT NULL,
  event_type TEXT,
  created_at TIMESTAMP
) PARTITION BY HASH (user_id);

CREATE TABLE events_p0 PARTITION OF events
  FOR VALUES WITH (MODULUS 4, REMAINDER 0);

CREATE TABLE events_p1 PARTITION OF events
  FOR VALUES WITH (MODULUS 4, REMAINDER 1);

CREATE TABLE events_p2 PARTITION OF events
  FOR VALUES WITH (MODULUS 4, REMAINDER 2);

CREATE TABLE events_p3 PARTITION OF events
  FOR VALUES WITH (MODULUS 4, REMAINDER 3);
```

**Use case:** Distributing writes across multiple partitions for parallel processing

**💡 Key Learning:** Choose partitioning strategy based on query patterns: Range for time-series, List for categories, Hash for load distribution.

---

# 13. REPLICATION & HIGH AVAILABILITY

## 13.1 Streaming Replication

**📌 Continuously send WAL (Write-Ahead Log) to standby servers**

```
Primary Server → WAL → Standby Server(s)
```

### Setup Process

**On Primary:**
```sql
-- postgresql.conf
wal_level = replica
max_wal_senders = 10
wal_keep_size = 1GB

-- pg_hba.conf
host replication replicator standby_ip/32 md5

-- Create replication user
CREATE ROLE replicator WITH REPLICATION LOGIN PASSWORD 'password';
```

**On Standby:**
```bash
# Base backup from primary
pg_basebackup -h primary_ip -D /var/lib/postgresql/data -U replicator -P

# standby.signal file
touch /var/lib/postgresql/data/standby.signal

# postgresql.conf
primary_conninfo = 'host=primary_ip port=5432 user=replicator password=password'
```

---

## 13.2 Synchronous vs Asynchronous Replication

**Asynchronous (default):**
- Primary doesn't wait for standby
- Faster writes
- Small risk of data loss on primary failure

**Synchronous:**
- Primary waits for at least one standby to confirm
- Slower writes
- Zero data loss

```sql
-- Enable synchronous replication
ALTER SYSTEM SET synchronous_commit = 'on';
ALTER SYSTEM SET synchronous_standby_names = 'standby1';
```

---

## 13.3 Read Replicas

**Use standby servers for read queries**

```sql
-- Application reads from standby
SELECT * FROM orders WHERE user_id = 123;  -- Read from replica

-- Writes go to primary
INSERT INTO orders (...) VALUES (...);  -- Write to primary
```

### 🌍 Real-World Scenario: E-commerce at Scale

**Setup:**
- 1 Primary (writes)
- 3 Read replicas (reads)
- Load balancer distributes reads

**Traffic distribution:**
- Writes: 10,000/sec → Primary
- Reads: 100,000/sec → 3 replicas (33k each)

**💡 Key Learning:** Replication scales read capacity. 1 primary + N replicas = N+1x read capacity. Use connection pooling (PgBouncer) to manage connections efficiently.

---

## 13.4 Failover and High Availability

### Manual Failover

```bash
# Promote standby to primary
pg_ctl promote -D /var/lib/postgresql/data
```

### Automatic Failover Tools

1. **Patroni** - Most popular, uses etcd/Consul/Zookeeper
2. **repmgr** - Lightweight, easier setup
3. **Stolon** - Kubernetes-native

**Typical HA Setup:**
```
                    Load Balancer
                         |
        +----------------+----------------+
        |                |                |
    Primary          Standby 1       Standby 2
        |                |                |
    Patroni          Patroni          Patroni
        |                |                |
        +-------etcd Cluster--------------+
```

**💡 Key Learning:** Plan for failure. Automate failover. Test recovery procedures regularly. RPO (Recovery Point Objective) and RTO (Recovery Time Objective) drive architecture decisions.

---

# 🎓 MASTERY CHECKLIST

## Basics ✓
- [ ] Write efficient SELECT queries with proper indexing
- [ ] Use RETURNING to eliminate extra queries
- [ ] Understand MVCC and its implications
- [ ] Know when to use DELETE vs TRUNCATE
- [ ] Master JOIN types and when to use each
- [ ] Use EXPLAIN to verify query plans

## Intermediate ✓
- [ ] Replace correlated subqueries with window functions
- [ ] Use CTEs for readable, maintainable queries
- [ ] Write recursive queries for hierarchical data
- [ ] Apply leftmost prefix rule for composite indexes
- [ ] Use partial indexes for hot data
- [ ] Understand index costs on write workloads

## Advanced ✓
- [ ] Choose appropriate isolation levels
- [ ] Handle deadlocks and serialization failures
- [ ] Use specialized indexes (GIN, GiST, BRIN)
- [ ] Store and query JSONB efficiently
- [ ] Optimize queries through EXPLAIN analysis
- [ ] Partition tables for scalability
- [ ] Set up replication for HA

---

# 📊 Scale Thinking Examples

**Think like this when designing:**

| Scenario | Scale | Solution |
|----------|-------|----------|
| User authentication | 10M users | Unique index on email, connection pooling |
| Order history | 500M orders | Partition by month, archive old data |
| Real-time analytics | 1M events/sec | Streaming replication, read replicas |
| Search functionality | 100M products | GIN index on tsvector, denormalization |
| Audit logs | 1B entries/day | BRIN index, partition by day, compress old partitions |
| Session storage | 50M active | JSONB for flexibility, partial index on active sessions |

**Key principle:** Always ask "What if this table had 100x more data?"

---

# 🚀 Next Steps

1. **Practice with real data:** Create tables with millions of rows
2. **Use EXPLAIN religiously:** Don't guess, measure
3. **Monitor in production:** pg_stat_statements, pg_stat_user_indexes
4. **Read PostgreSQL docs:** They're excellent
5. **Study open-source projects:** See how others handle scale
6. **Benchmark everything:** Your use case is unique

**Resources:**
- PostgreSQL Official Docs: postgresql.org/docs
- Use The Index, Luke: use-the-index-luke.com
- Postgres Weekly Newsletter
- pgtune.leopard.in.ua for configuration

---

**💡 Final Wisdom:** PostgreSQL is incredibly powerful. Most performance issues come from poor query design, not database limitations. Master the fundamentals, understand internals, think about scale, and always measure before optimizing.

Now go build something amazing! 🚀
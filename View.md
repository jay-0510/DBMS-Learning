# PostgreSQL Views Mastery
> From Basics to Advanced — Complete Guide

---

# Chapter 1: What Is a PostgreSQL View?

A **View** is a named, saved SQL query stored in the database. It looks and behaves like a table — you can SELECT from it, join it, filter it — but it holds no data itself. Every time you query a view, PostgreSQL runs the underlying query behind the scenes.

> **📖 Definition:** A VIEW is a virtual table defined by a SELECT statement. It simplifies complex queries, enforces security by hiding sensitive columns, and provides a stable interface even when the underlying table structure changes.

```
WITHOUT VIEW                              WITH VIEW
┌──────────────────────────────┐          ┌─────────────────────┐
│ Every query repeats the same │          │  CREATE VIEW        │
│ complex JOIN + WHERE + GROUP │  ──────► │  emp_summary AS ... │
│ SELECT e.name, d.dept_name,  │          └─────────────────────┘
│        s.salary              │                    │
│ FROM employees e             │                    ▼
│ JOIN departments d ON ...    │          SELECT * FROM emp_summary;
│ JOIN salaries s ON ...       │          ← One clean line ✅
│ WHERE e.is_active = true;    │
│ (repeated 50 times) ❌        │
└──────────────────────────────┘
```
*Figure 1.1 — Raw query repetition vs using a View*

---

## 1.1 Setting Up Our Example Tables

We'll use these tables throughout the guide:

```sql
-- employees table
CREATE TABLE employees (
    emp_id      SERIAL PRIMARY KEY,
    name        VARCHAR(100),
    dept_id     INT,
    salary      NUMERIC(10,2),
    email       VARCHAR(200),
    hire_date   DATE,
    is_active   BOOLEAN DEFAULT true,
    manager_id  INT
);

-- departments table
CREATE TABLE departments (
    dept_id   SERIAL PRIMARY KEY,
    dept_name VARCHAR(50),
    location  VARCHAR(100)
);

-- salaries audit table
CREATE TABLE salary_audit (
    audit_id   SERIAL PRIMARY KEY,
    emp_id     INT,
    old_salary NUMERIC(10,2),
    new_salary NUMERIC(10,2),
    changed_at TIMESTAMP DEFAULT now()
);

-- Seed data
INSERT INTO departments (dept_name, location) VALUES
    ('IT',        'New York'),
    ('HR',        'Chicago'),
    ('Finance',   'San Francisco'),
    ('Sales',     'Austin'),
    ('Marketing', 'Boston');

INSERT INTO employees (name, dept_id, salary, email, hire_date, is_active, manager_id)
SELECT
    'Employee_' || i,
    1 + (i % 5),
    40000 + (i % 60000),
    'emp' || i || '@company.com',
    DATE '2010-01-01' + (i % 5000),
    (i % 10 != 0),
    CASE WHEN i > 5 THEN 1 + (i % 5) ELSE NULL END
FROM generate_series(1, 1000) i;
```

---

# Chapter 2: How Views Work Internally

When you query a view, PostgreSQL expands it into its underlying SQL at **parse time** — before execution. There is no separate storage; the view is just a stored query definition.

```
  YOU WRITE:                          POSTGRESQL EXECUTES:
  ┌──────────────────────┐            ┌────────────────────────────────────┐
  │ SELECT *             │            │ SELECT e.name,                     │
  │ FROM active_employees│  ────────► │        d.dept_name,                │
  │ WHERE dept_id = 1;   │            │        e.salary                    │
  └──────────────────────┘            │ FROM   employees e                 │
                                      │ JOIN   departments d               │
            ▼                         │   ON   e.dept_id = d.dept_id       │
  Query Rewrite Engine                │ WHERE  e.is_active = true          │
  (expands view definition)           │   AND  e.dept_id = 1;              │
                                      └────────────────────────────────────┘
```
*Figure 2.1 — View query expansion at parse time*

### Types of Views at a Glance

| Type | Stores Data? | Auto-refreshed? | Updatable? | Best For |
|---|---|---|---|---|
| **Simple View** | No | Yes (live) | Sometimes | Query simplification |
| **Updatable View** | No | Yes (live) | Yes | DML through the view |
| **View + CHECK OPTION** | No | Yes (live) | Yes + validated | Enforcing data rules |
| **Materialized View** | Yes (snapshot) | Manual/scheduled | No | Heavy reporting queries |
| **Recursive View** | No | Yes (live) | No | Hierarchical/tree data |

---

# Chapter 3: Create Views

## 3.1 Basic Syntax

```sql
CREATE [OR REPLACE] VIEW view_name
    [(column_alias1, column_alias2, ...)]
AS
    SELECT ...
[WITH [CASCADED | LOCAL] CHECK OPTION];
```

## 3.2 Simple View

```sql
-- View 1: Active employees with their department name
CREATE VIEW active_employees AS
SELECT
    e.emp_id,
    e.name,
    d.dept_name,
    e.salary,
    e.email,
    e.hire_date
FROM employees e
JOIN departments d ON e.dept_id = d.dept_id
WHERE e.is_active = true;

-- Use it exactly like a table
SELECT * FROM active_employees;
SELECT * FROM active_employees WHERE dept_name = 'IT';
SELECT * FROM active_employees ORDER BY salary DESC LIMIT 5;
```

## 3.3 View with Aggregation

```sql
-- View 2: Department salary summary
CREATE VIEW dept_salary_summary AS
SELECT
    d.dept_name,
    COUNT(e.emp_id)          AS total_employees,
    ROUND(AVG(e.salary), 2)  AS avg_salary,
    MIN(e.salary)            AS min_salary,
    MAX(e.salary)            AS max_salary,
    SUM(e.salary)            AS total_payroll
FROM employees e
JOIN departments d ON e.dept_id = d.dept_id
WHERE e.is_active = true
GROUP BY d.dept_name;

-- Query it
SELECT * FROM dept_salary_summary ORDER BY avg_salary DESC;
```

## 3.4 View on View (Layered Views)

```sql
-- Base view
CREATE VIEW active_employees AS
SELECT e.*, d.dept_name
FROM employees e
JOIN departments d ON e.dept_id = d.dept_id
WHERE e.is_active = true;

-- View built on top of another view
CREATE VIEW high_earners AS
SELECT emp_id, name, dept_name, salary
FROM active_employees          -- ← referencing the view above
WHERE salary > 80000
ORDER BY salary DESC;

SELECT * FROM high_earners;
```

## 3.5 OR REPLACE — Update a View Without Dropping

```sql
-- Modify an existing view safely (keeps permissions intact)
CREATE OR REPLACE VIEW active_employees AS
SELECT
    e.emp_id,
    e.name,
    d.dept_name,
    e.salary,
    e.email,
    e.hire_date,
    e.manager_id    -- ← new column added
FROM employees e
JOIN departments d ON e.dept_id = d.dept_id
WHERE e.is_active = true;

-- Rules for OR REPLACE:
-- ✅ You can ADD new columns at the end
-- ✅ You can change column expressions
-- ❌ You cannot REMOVE or REORDER existing columns
-- ❌ You cannot change a column's data type incompatibly
```

> **⚡ Tip:** Use `CREATE OR REPLACE VIEW` instead of DROP + CREATE to preserve any GRANTs on the view.

---

# Chapter 4: Drop Views

```sql
-- Drop a single view
DROP VIEW active_employees;

-- Drop only if it exists (no error if missing)
DROP VIEW IF EXISTS active_employees;

-- Drop multiple views at once
DROP VIEW IF EXISTS active_employees, dept_salary_summary, high_earners;

-- CASCADE — also drops objects that depend on this view
DROP VIEW active_employees CASCADE;
-- Drops active_employees AND any views built on top of it

-- RESTRICT (default) — refuses to drop if anything depends on it
DROP VIEW active_employees RESTRICT;
-- ERROR: cannot drop view because other objects depend on it
```

```
  DROP VIEW active_employees CASCADE

  active_employees (view)
         │
         ├──► high_earners (view)         ← also dropped
         │         │
         │         └──► top_managers (view) ← also dropped
         │
         └──► dept_report (view)          ← also dropped

  All dependent objects are removed automatically with CASCADE.
  Use with care in production! ⚠️
```
*Figure 4.1 — CASCADE drops all dependent views*

---

# Chapter 5: Updatable Views

A view is **automatically updatable** when it meets these conditions. PostgreSQL will let you run INSERT, UPDATE, and DELETE directly on the view.

```
  UPDATABLE VIEW CHECKLIST
  ┌─────────────────────────────────────────────────────┐
  │  ✅ Based on a single table (no JOINs)              │
  │  ✅ No DISTINCT clause                              │
  │  ✅ No GROUP BY or HAVING                           │
  │  ✅ No aggregate functions (SUM, COUNT, etc.)       │
  │  ✅ No UNION, INTERSECT, EXCEPT                     │
  │  ✅ No window functions                             │
  │  ✅ SELECT list references actual columns (no *)    │
  └─────────────────────────────────────────────────────┘
  If ALL boxes are checked → the view is updatable ✅
  If ANY box fails         → read-only view ❌
```
*Figure 5.1 — Auto-updatable view requirements*

## 5.1 Creating an Updatable View

```sql
-- Simple single-table view — automatically updatable
CREATE VIEW it_employees AS
SELECT emp_id, name, salary, email, hire_date, is_active
FROM employees
WHERE dept_id = 1;   -- IT department

-- INSERT through the view
INSERT INTO it_employees (name, salary, email, hire_date)
VALUES ('New Hire', 65000, 'newhire@company.com', '2024-01-15');
-- ✅ Row inserted into employees with dept_id = 1

-- UPDATE through the view
UPDATE it_employees
SET salary = 70000
WHERE name = 'New Hire';
-- ✅ Updates employees table directly

-- DELETE through the view
DELETE FROM it_employees
WHERE name = 'New Hire';
-- ✅ Deletes from employees table
```

## 5.2 The Disappearing Row Problem

```sql
-- The view filters for dept_id = 1 (IT)
-- What if we INSERT a row for dept_id = 3 (Finance)?

INSERT INTO it_employees (name, salary, email, hire_date)
VALUES ('Finance Guy', 70000, 'fg@company.com', '2024-01-15');
-- ❌ No error! But the row has dept_id = NULL (default)
-- It WON'T appear in it_employees after insert

-- Or worse: UPDATE a row so it leaves the view's filter
UPDATE it_employees SET dept_id = 3 WHERE emp_id = 5;
-- Row moves to Finance — disappears from it_employees silently!
-- This is solved by WITH CHECK OPTION (next chapter)
```

## 5.3 INSTEAD OF Triggers for Complex Updatable Views

When a view has JOINs or aggregations, use `INSTEAD OF` triggers to handle DML manually:

```sql
-- A view with a JOIN (not auto-updatable)
CREATE VIEW employee_details AS
SELECT
    e.emp_id, e.name, e.salary,
    d.dept_name, d.location
FROM employees e
JOIN departments d ON e.dept_id = d.dept_id;

-- Create an INSTEAD OF INSERT trigger
CREATE FUNCTION employee_details_insert()
RETURNS TRIGGER AS $$
BEGIN
    INSERT INTO employees (name, salary, dept_id)
    SELECT NEW.name, NEW.salary, d.dept_id
    FROM departments d
    WHERE d.dept_name = NEW.dept_name;
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_employee_details_insert
    INSTEAD OF INSERT ON employee_details
    FOR EACH ROW EXECUTE FUNCTION employee_details_insert();

-- Now this works even though the view has a JOIN:
INSERT INTO employee_details (name, salary, dept_name)
VALUES ('Alice', 75000, 'IT');
```

---

# Chapter 6: WITH CHECK OPTION

`WITH CHECK OPTION` prevents INSERT or UPDATE operations that would make the row **disappear from the view** after the operation. It enforces that all rows written through the view must remain visible in it.

```
  WITH CHECK OPTION — How It Works

  View: it_employees → WHERE dept_id = 1

  INSERT name='Alice', dept_id=1  ──► dept_id=1 ✅ visible in view → ALLOWED
  INSERT name='Bob',   dept_id=3  ──► dept_id=3 ❌ not in view     → REJECTED
  UPDATE emp_id=5 SET dept_id=2   ──► dept_id=2 ❌ leaves the view  → REJECTED
  UPDATE emp_id=5 SET salary=90k  ──► still dept_id=1 ✅            → ALLOWED
```
*Figure 6.1 — CHECK OPTION validation logic*

## 6.1 Basic WITH CHECK OPTION

```sql
-- View with check option
CREATE VIEW it_employees AS
SELECT emp_id, name, salary, email, hire_date, dept_id, is_active
FROM employees
WHERE dept_id = 1
WITH CHECK OPTION;

-- ✅ Allowed — row stays within dept_id = 1
INSERT INTO it_employees (name, salary, email, hire_date, dept_id)
VALUES ('Alice', 65000, 'alice@company.com', '2024-01-15', 1);

-- ❌ Rejected — row would not be visible in it_employees
INSERT INTO it_employees (name, salary, email, hire_date, dept_id)
VALUES ('Bob', 65000, 'bob@company.com', '2024-01-15', 3);
-- ERROR: new row violates check option for view "it_employees"

-- ❌ Rejected — row would leave the view after update
UPDATE it_employees SET dept_id = 2 WHERE emp_id = 10;
-- ERROR: new row violates check option for view "it_employees"
```

## 6.2 LOCAL vs CASCADED CHECK OPTION

When views are layered (a view on a view), check option has two modes:

```sql
-- Base view: active IT employees
CREATE VIEW active_it_employees AS
SELECT emp_id, name, salary, dept_id, is_active
FROM employees
WHERE dept_id = 1 AND is_active = true;

-- Child view: high earners from active_it_employees
CREATE VIEW it_high_earners AS
SELECT emp_id, name, salary
FROM active_it_employees
WHERE salary > 70000
WITH LOCAL CHECK OPTION;    -- ← LOCAL: only checks THIS view's condition

-- With LOCAL — only salary > 70000 is checked
INSERT INTO it_high_earners (name, salary, dept_id, is_active)
VALUES ('Test', 75000, 3, false);
-- ✅ Passes LOCAL check (salary=75000 > 70000)
-- ❌ But violates parent view condition (dept_id≠1, is_active=false)
--    Row won't appear in active_it_employees → silent inconsistency!
```

```sql
-- Better: use CASCADED (default when you write WITH CHECK OPTION)
CREATE VIEW it_high_earners AS
SELECT emp_id, name, salary
FROM active_it_employees
WHERE salary > 70000
WITH CASCADED CHECK OPTION;   -- ← checks ALL parent view conditions too

-- Now ALL conditions are enforced:
INSERT INTO it_high_earners (name, salary, dept_id, is_active)
VALUES ('Test', 75000, 3, false);
-- ❌ ERROR — dept_id=3 violates parent view's dept_id=1 condition
```

```
  CASCADED vs LOCAL CHECK OPTION

  active_it_employees:  WHERE dept_id=1 AND is_active=true
          │
          └──► it_high_earners: WHERE salary > 70000

  LOCAL check:     only validates  salary > 70000
  CASCADED check:  validates       salary > 70000
                                   AND dept_id = 1
                                   AND is_active = true    ← parent conditions
```
*Figure 6.2 — CASCADED enforces all ancestor view conditions*

---

# Chapter 7: Materialized Views

A **Materialized View** physically stores the result of the query on disk. Unlike regular views, it does NOT automatically reflect changes to the underlying tables — you refresh it manually or on a schedule.

```
  REGULAR VIEW                        MATERIALIZED VIEW
  ┌──────────────────────┐            ┌──────────────────────┐
  │ Stored: SQL only     │            │ Stored: SQL + DATA   │
  │ Query time: slow     │            │ Query time: fast ✅  │
  │ Always up-to-date ✅ │            │ Data: snapshot ⚠️    │
  │ No extra disk space  │            │ Needs REFRESH        │
  └──────────────────────┘            └──────────────────────┘
        │                                       │
        ▼  Every SELECT runs the query          ▼  SELECT reads stored data
   employees + JOIN + GROUP BY            Pre-computed result set
   (runs every time, can be slow)         (runs once, fast every time)
```
*Figure 7.1 — Regular view vs Materialized view*

## 7.1 Create a Materialized View

```sql
-- Create a materialized view (computes and stores data immediately)
CREATE MATERIALIZED VIEW dept_stats AS
SELECT
    d.dept_name,
    COUNT(e.emp_id)          AS headcount,
    ROUND(AVG(e.salary), 2)  AS avg_salary,
    SUM(e.salary)            AS total_payroll,
    MIN(e.hire_date)         AS earliest_hire,
    MAX(e.hire_date)         AS latest_hire
FROM employees e
JOIN departments d ON e.dept_id = d.dept_id
WHERE e.is_active = true
GROUP BY d.dept_name
ORDER BY total_payroll DESC;

-- Query it — reads from stored data, very fast
SELECT * FROM dept_stats;
```

```sql
-- Create WITHOUT populating immediately (useful for large datasets)
CREATE MATERIALIZED VIEW dept_stats
WITH NO DATA    -- ← don't compute yet
AS
SELECT ...;

-- It's empty until you refresh:
SELECT * FROM dept_stats;
-- ERROR: materialized view "dept_stats" has not been populated
-- Use REFRESH to populate it first
```

## 7.2 Refreshing a Materialized View

```sql
-- Standard refresh — locks the view during refresh (blocks reads)
REFRESH MATERIALIZED VIEW dept_stats;

-- CONCURRENTLY — allows reads while refreshing (no lock) ✅
-- Requires a UNIQUE index on the materialized view
REFRESH MATERIALIZED VIEW CONCURRENTLY dept_stats;

-- Create the required unique index for CONCURRENTLY
CREATE UNIQUE INDEX ON dept_stats (dept_name);

-- Then concurrent refresh works
REFRESH MATERIALIZED VIEW CONCURRENTLY dept_stats;
```

```
  REFRESH STRATEGIES

  Option 1: Manual
  REFRESH MATERIALIZED VIEW dept_stats;
  └── Run after major data changes

  Option 2: Scheduled (via pg_cron extension)
  SELECT cron.schedule('0 2 * * *',   -- every day at 2 AM
      'REFRESH MATERIALIZED VIEW dept_stats');

  Option 3: Trigger-based (on underlying table change)
  CREATE FUNCTION refresh_dept_stats() RETURNS TRIGGER AS $$
  BEGIN
      REFRESH MATERIALIZED VIEW CONCURRENTLY dept_stats;
      RETURN NULL;
  END;
  $$ LANGUAGE plpgsql;

  CREATE TRIGGER trg_refresh_dept_stats
      AFTER INSERT OR UPDATE OR DELETE ON employees
      FOR EACH STATEMENT EXECUTE FUNCTION refresh_dept_stats();
```
*Figure 7.2 — Materialized view refresh strategies*

## 7.3 Indexing a Materialized View

Since materialized views store real data, you can add indexes to them:

```sql
-- Index on the most-queried column
CREATE INDEX idx_dept_stats_payroll ON dept_stats (total_payroll DESC);
CREATE INDEX idx_dept_stats_name    ON dept_stats (dept_name);

-- Now range queries on the materialized view are fast
SELECT * FROM dept_stats WHERE total_payroll > 5000000;
SELECT * FROM dept_stats WHERE dept_name = 'IT';
```

## 7.4 When to Use Materialized Views

```
  USE MATERIALIZED VIEW WHEN:               USE REGULAR VIEW WHEN:
  ┌────────────────────────────────┐        ┌────────────────────────────────┐
  │ ✅ Query takes > 1 second      │        │ ✅ Data must always be current  │
  │ ✅ Data freshness of hours/    │        │ ✅ Query is fast enough already │
  │    days is acceptable          │        │ ✅ Simple query abstraction     │
  │ ✅ Heavy aggregations/JOINs    │        │ ✅ Security/column hiding       │
  │ ✅ Dashboard / reporting data  │        │ ✅ Underlying data changes a lot│
  │ ✅ Data doesn't change often   │        └────────────────────────────────┘
  └────────────────────────────────┘
```

---

# Chapter 8: Recursive Views

A **Recursive View** uses a `WITH RECURSIVE` CTE (Common Table Expression) to query **hierarchical or tree-structured data** — like org charts, category trees, or folder structures.

```
  EMPLOYEE HIERARCHY EXAMPLE

  CEO (emp_id=1, manager_id=NULL)
  ├── VP Engineering (emp_id=2, manager_id=1)
  │   ├── Team Lead A (emp_id=5, manager_id=2)
  │   │   ├── Dev 1 (emp_id=10, manager_id=5)
  │   │   └── Dev 2 (emp_id=11, manager_id=5)
  │   └── Team Lead B (emp_id=6, manager_id=2)
  └── VP Sales (emp_id=3, manager_id=1)
      └── Sales Rep (emp_id=9, manager_id=3)

  A recursive view can traverse this tree to any depth automatically.
```
*Figure 8.1 — Hierarchical employee org chart*

## 8.1 Creating a Recursive View

```sql
-- Recursive view: full org chart with depth and path
CREATE RECURSIVE VIEW org_chart (
    emp_id, name, manager_id, depth, path
) AS
-- Base case: top-level employees (no manager)
SELECT
    emp_id,
    name,
    manager_id,
    0 AS depth,
    name::TEXT AS path
FROM employees
WHERE manager_id IS NULL

UNION ALL

-- Recursive case: employees who report to someone
SELECT
    e.emp_id,
    e.name,
    e.manager_id,
    oc.depth + 1,
    oc.path || ' → ' || e.name
FROM employees e
JOIN org_chart oc ON e.manager_id = oc.emp_id;  -- ← self-join via view
```

```sql
-- Query the recursive view
SELECT
    repeat('  ', depth) || name AS org_tree,  -- indented display
    depth,
    path
FROM org_chart
ORDER BY path;

-- Find all direct and indirect reports of emp_id = 2
SELECT * FROM org_chart
WHERE path LIKE '%' || (SELECT name FROM employees WHERE emp_id = 2) || '%'
  AND emp_id != 2;

-- Find depth of org chart
SELECT MAX(depth) AS max_depth FROM org_chart;
```

## 8.2 Recursive View for Category Trees

```sql
-- Another classic use: product category hierarchy
CREATE TABLE categories (
    cat_id     SERIAL PRIMARY KEY,
    cat_name   VARCHAR(100),
    parent_id  INT REFERENCES categories(cat_id)
);

INSERT INTO categories VALUES
    (1, 'Electronics',  NULL),
    (2, 'Computers',    1),
    (3, 'Laptops',      2),
    (4, 'Gaming',       2),
    (5, 'Phones',       1),
    (6, 'Android',      5),
    (7, 'iPhone',       5);

CREATE RECURSIVE VIEW category_tree (
    cat_id, cat_name, parent_id, level, full_path
) AS
SELECT cat_id, cat_name, parent_id,
       1 AS level,
       cat_name::TEXT AS full_path
FROM categories
WHERE parent_id IS NULL

UNION ALL

SELECT c.cat_id, c.cat_name, c.parent_id,
       ct.level + 1,
       ct.full_path || ' > ' || c.cat_name
FROM categories c
JOIN category_tree ct ON c.parent_id = ct.cat_id;

-- Query the tree
SELECT repeat('  ', level - 1) || cat_name AS category_tree, full_path
FROM category_tree
ORDER BY full_path;

-- Output:
-- Electronics
--   Computers
--     Laptops
--     Gaming
--   Phones
--     Android
--     iPhone
```

---

# Chapter 9: Alter Views

PostgreSQL doesn't have a full `ALTER VIEW ... AS SELECT` command. You use `CREATE OR REPLACE VIEW` to change the query, and `ALTER VIEW` for metadata changes.

## 9.1 Rename a View

```sql
-- Rename a view
ALTER VIEW active_employees RENAME TO current_employees;

-- Verify
SELECT viewname FROM pg_views WHERE viewname = 'current_employees';
```

## 9.2 Change View Owner

```sql
-- Transfer ownership to another role
ALTER VIEW current_employees OWNER TO hr_team;
```

## 9.3 Set / Reset View Options

```sql
-- Set security_barrier option (prevents certain optimizations for security)
ALTER VIEW current_employees SET (security_barrier = true);

-- Reset to default
ALTER VIEW current_employees RESET (security_barrier);
```

## 9.4 Rename a Column in a View

```sql
-- Rename a column alias in a view
ALTER VIEW dept_salary_summary
    RENAME COLUMN total_employees TO headcount;
```

## 9.5 Change the Query (CREATE OR REPLACE)

```sql
-- To change what the view returns, use CREATE OR REPLACE
CREATE OR REPLACE VIEW active_employees AS
SELECT
    e.emp_id,
    e.name,
    d.dept_name,
    e.salary,
    e.email,
    e.hire_date,
    e.is_active,
    e.manager_id     -- ← added this column
FROM employees e
JOIN departments d ON e.dept_id = d.dept_id
WHERE e.is_active = true;

-- Rules:
-- ✅ Can add new columns at the END of the SELECT list
-- ✅ Can change column expressions or WHERE logic
-- ❌ Cannot remove or reorder existing columns
-- ❌ Cannot change existing column's data type incompatibly
```

## 9.6 Full Replacement (DROP + CREATE)

When you need to reorder or remove columns, you must drop and recreate:

```sql
-- Step 1: Drop dependent objects (save their DDL first!)
DROP VIEW IF EXISTS high_earners CASCADE;   -- depends on active_employees
DROP VIEW IF EXISTS active_employees CASCADE;

-- Step 2: Recreate with new structure
CREATE VIEW active_employees AS
SELECT
    e.emp_id,
    e.name,
    e.salary,          -- reordered
    d.dept_name,       -- reordered
    e.hire_date
FROM employees e
JOIN departments d ON e.dept_id = d.dept_id
WHERE e.is_active = true;

-- Step 3: Recreate dependent views
CREATE VIEW high_earners AS
SELECT * FROM active_employees WHERE salary > 80000;

-- Step 4: Reapply any GRANTs
GRANT SELECT ON active_employees TO reporting_role;
```

---

# Chapter 10: Listing Views

## 10.1 List All Views

```sql
-- List all views in the current database
SELECT
    schemaname  AS schema,
    viewname    AS view_name,
    viewowner   AS owner
FROM pg_views
WHERE schemaname NOT IN ('pg_catalog', 'information_schema')  -- exclude system views
ORDER BY schemaname, viewname;
```

## 10.2 Show View Definition (Source SQL)

```sql
-- Show the SQL behind a view
SELECT viewname, definition
FROM pg_views
WHERE viewname = 'active_employees';

-- Alternative using pg_get_viewdef()
SELECT pg_get_viewdef('active_employees', true);   -- true = formatted output

-- In psql shell:
\d+ active_employees    -- shows columns + definition
\dv                     -- lists all views
\dv active*             -- lists views matching pattern
```

## 10.3 Detailed View Information

```sql
-- Full metadata: schema, name, owner, definition
SELECT
    n.nspname                            AS schema,
    c.relname                            AS view_name,
    pg_catalog.pg_get_userbyid(c.relowner) AS owner,
    pg_catalog.pg_get_viewdef(c.oid, true) AS definition
FROM pg_catalog.pg_class c
JOIN pg_catalog.pg_namespace n ON n.oid = c.relnamespace
WHERE c.relkind = 'v'                    -- 'v' = view, 'm' = materialized view
  AND n.nspname NOT IN ('pg_catalog', 'information_schema')
ORDER BY schema, view_name;
```

## 10.4 List Materialized Views

```sql
-- List all materialized views
SELECT
    schemaname      AS schema,
    matviewname     AS view_name,
    matviewowner    AS owner,
    ispopulated     AS has_data,
    definition
FROM pg_matviews
WHERE schemaname NOT IN ('pg_catalog', 'information_schema')
ORDER BY schemaname, matviewname;

-- Check last refresh time
SELECT
    schemaname,
    matviewname,
    pg_size_pretty(pg_relation_size(schemaname||'.'||matviewname)) AS size
FROM pg_matviews;
```

## 10.5 Find Views That Depend on a Table

```sql
-- Which views use the employees table?
SELECT DISTINCT
    v.viewname
FROM pg_views v
WHERE v.definition ILIKE '%employees%'
  AND v.schemaname = 'public'
ORDER BY v.viewname;

-- More precise: use pg_depend to find object dependencies
SELECT
    dependent.relname   AS view_name,
    source.relname      AS depends_on_table
FROM pg_depend dep
JOIN pg_class dependent ON dependent.oid = dep.objid
JOIN pg_class source     ON source.oid   = dep.refobjid
WHERE dependent.relkind = 'v'
  AND source.relname = 'employees';
```

## 10.6 Check if a View is Updatable

```sql
-- information_schema shows updatability
SELECT
    table_name             AS view_name,
    is_updatable,
    is_insertable_into,
    is_trigger_updatable,
    is_trigger_deletable,
    is_trigger_insertable_into
FROM information_schema.views
WHERE table_schema = 'public'
ORDER BY table_name;
```

---

# Chapter 11: Security & Permissions

## 11.1 Granting Access to Views

```sql
-- Create a role for read-only reporting
CREATE ROLE reporting_role;

-- Grant SELECT on the view only (not the underlying tables)
GRANT SELECT ON active_employees    TO reporting_role;
GRANT SELECT ON dept_salary_summary TO reporting_role;

-- The role can query the view but NOT the raw tables
-- This hides sensitive columns like salary from direct access
REVOKE ALL ON employees    FROM reporting_role;
REVOKE ALL ON departments  FROM reporting_role;
```

## 11.2 Security Barrier Views

```sql
-- security_barrier prevents function calls from leaking row data
-- before the view's WHERE filter is applied
CREATE VIEW it_employees
    WITH (security_barrier = true)
AS
SELECT emp_id, name, email
FROM employees
WHERE dept_id = 1;

-- Without security_barrier, a malicious function in a query could
-- see rows before the WHERE filter executes. This option blocks that.
```

---

# Chapter 12: Best Practices & Decision Guide

```
  VIEW SELECTION FLOWCHART

  Do you need to simplify a complex query?
         │
         ▼
  Is data freshness critical (always up to date)?
    YES ──► Is the query fast enough? (< 100ms)
              YES ──► Simple VIEW ✅
              NO  ──► Optimize query + Simple VIEW
    NO  ──► Is query slow (> 1 second)?
              YES ──► MATERIALIZED VIEW ✅  (refresh on schedule)
              NO  ──► Simple VIEW ✅

  Do you need to traverse a tree/hierarchy?
    YES ──► RECURSIVE VIEW ✅

  Do you need INSERT/UPDATE/DELETE through the view?
    YES ──► Single table, no aggregation?
              YES ──► UPDATABLE VIEW + WITH CHECK OPTION ✅
              NO  ──► INSTEAD OF trigger ✅
```
*Figure 12.1 — View type selection flowchart*

## 12.1 Golden Rules

1. **Name views clearly** — prefix with `v_` or suffix with `_view` (e.g., `v_active_employees`) so they're instantly identifiable.
2. **Use OR REPLACE to update** — avoids dropping and re-granting permissions.
3. **Always use WITH CHECK OPTION on updatable views** — prevents silent disappearing rows.
4. **Use CASCADED not LOCAL** — ensures parent view conditions are also enforced.
5. **Add a UNIQUE index before CONCURRENTLY refresh** — required for lock-free materialized view refresh.
6. **Don't nest views more than 2-3 layers deep** — deeply nested views become hard to debug and optimize.
7. **Use security_barrier for sensitive data** — prevents row leakage before filter application.
8. **Document your views** — add comments with `COMMENT ON VIEW`.

```sql
-- Always document your views
COMMENT ON VIEW active_employees IS
    'Returns all currently active employees with their department name. Excludes terminated staff.';

-- View the comment
SELECT obj_description('active_employees'::regclass, 'pg_class');
```

---

## 12.2 Quick Reference Cheat Sheet

| Task | SQL |
|---|---|
| Create view | `CREATE VIEW v_name AS SELECT ...;` |
| Create or replace | `CREATE OR REPLACE VIEW v_name AS SELECT ...;` |
| Updatable view | `CREATE VIEW v_name AS SELECT ... FROM single_table WHERE ...;` |
| With check option | `CREATE VIEW v_name AS SELECT ... WITH CHECK OPTION;` |
| Cascaded check | `CREATE VIEW v_name AS SELECT ... WITH CASCADED CHECK OPTION;` |
| Materialized view | `CREATE MATERIALIZED VIEW mv_name AS SELECT ...;` |
| Materialized (no data) | `CREATE MATERIALIZED VIEW mv_name WITH NO DATA AS SELECT ...;` |
| Refresh materialized | `REFRESH MATERIALIZED VIEW mv_name;` |
| Refresh concurrently | `REFRESH MATERIALIZED VIEW CONCURRENTLY mv_name;` |
| Recursive view | `CREATE RECURSIVE VIEW v_name (cols) AS ... UNION ALL ...;` |
| Rename view | `ALTER VIEW v_name RENAME TO new_name;` |
| Change owner | `ALTER VIEW v_name OWNER TO role_name;` |
| Rename column | `ALTER VIEW v_name RENAME COLUMN old TO new;` |
| Drop view | `DROP VIEW IF EXISTS v_name;` |
| Drop with cascade | `DROP VIEW v_name CASCADE;` |
| Drop materialized | `DROP MATERIALIZED VIEW IF EXISTS mv_name;` |
| List all views | `SELECT viewname FROM pg_views WHERE schemaname='public';` |
| Show view SQL | `SELECT pg_get_viewdef('v_name', true);` |
| List materialized | `SELECT matviewname FROM pg_matviews;` |
| Check updatable? | `SELECT is_updatable FROM information_schema.views WHERE table_name='v_name';` |
| Grant access | `GRANT SELECT ON v_name TO role_name;` |
| Add comment | `COMMENT ON VIEW v_name IS 'description';` |

---

## 12.3 Common Mistakes to Avoid

| Mistake | Problem | Fix |
|---|---|---|
| Updating view without `CHECK OPTION` | Rows silently disappear from the view | Add `WITH CHECK OPTION` |
| Using `LOCAL` instead of `CASCADED` | Parent view conditions ignored | Use `CASCADED CHECK OPTION` |
| Refreshing materialized view without `UNIQUE` index | Can't use `CONCURRENTLY` | Create unique index first |
| Dropping a view without checking dependents | Breaks dependent views/reports | Use `\d+ viewname` or `pg_depend` first |
| Nesting 5+ views deep | Debugging becomes a nightmare | Flatten or use CTEs |
| No permissions on view | All users see raw tables | `REVOKE` table access, `GRANT` view access |
| Forgetting `OR REPLACE` on update | Drops and loses all GRANTs | Always use `CREATE OR REPLACE VIEW` |

---

*PostgreSQL Views Mastery Guide*
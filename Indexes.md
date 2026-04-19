# PostgreSQL Indexing Mastery
> From Basics to Advanced — Complete Guide

---

# Chapter 1: What Is a PostgreSQL Index?

Imagine a textbook with 1,000 pages. To find information about "B-Tree" you could either flip every page (full scan) or jump straight to the index at the back (index scan). PostgreSQL indexes work exactly the same way.

> **📖 Definition:** An INDEX is a separate data structure that PostgreSQL maintains alongside your table. It stores a sorted copy of one or more columns plus pointers back to the actual table rows — letting the engine skip rows it doesn't need.

```
WITHOUT INDEX (Sequential Scan)          WITH INDEX (Index Scan)
┌─────────────────────────────┐          ┌───────────────────────┐
│ Table: employees (10M rows) │          │     B-Tree Index      │
│ row 1:  Alice | Sales | 50k │          │  50k ──► row 500      │
│ row 2:  Bob   | HR    | 60k │          │  60k ──► row 2        │
│ row 3:  Carol | IT    | 70k │  ──────► │  70k ──► row 3        │
│  ...    ...   ...     ...   │          │  80k ──► row 9999     │
│ row 10M: Zara | CEO   | 90k │          │  90k ──► row 10M      │
└─────────────────────────────┘          └───────────────────────┘
Reads: 10,000,000 rows ❌                 Reads: ~3 rows  ✅
```
*Figure 1.1 — Sequential Scan vs Index Scan*

---

## 1.1 Setting Up Our Example Tables

Throughout this guide we'll use these two tables:

```sql
-- Main table: employees
CREATE TABLE employees (
    emp_id      SERIAL PRIMARY KEY,
    name        VARCHAR(100),
    dept        VARCHAR(50),
    salary      NUMERIC(10,2),
    email       VARCHAR(200),
    hire_date   DATE,
    is_active   BOOLEAN DEFAULT true,
    profile     JSONB,
    bio         TEXT
);

-- Supporting table: departments
CREATE TABLE departments (
    dept_id   SERIAL PRIMARY KEY,
    dept_name VARCHAR(50),
    location  VARCHAR(100)
);

-- Insert 1 million sample rows
INSERT INTO employees (name, dept, salary, email, hire_date, is_active, profile, bio)
SELECT
    'Employee_' || i,
    (ARRAY['Sales','HR','IT','Finance','Marketing'])[1 + (i % 5)],
    40000 + (i % 60000),
    'emp' || i || '@company.com',
    DATE '2010-01-01' + (i % 5000),
    (i % 10 != 0),
    jsonb_build_object('level', i%5+1, 'skills', ARRAY['SQL','Python']),
    'Biography text for employee ' || i
FROM generate_series(1, 1000000) i;
```

---

# Chapter 2: How Indexes Work Internally

## 2.1 The B-Tree: PostgreSQL's Default Index Structure

PostgreSQL's default index type is a B-Tree (Balanced Tree). Understanding it will help you use ALL index types wisely.

```
                    B-TREE INDEX on salary

                ┌─────────────────────┐
                │     ROOT NODE       │
                │  [40k | 60k | 80k]  │  ← split keys
                └──┬──────┬──────┬───┘
       ┌───────────┘      │      └──────────────┐
       ▼                  ▼                     ▼
 ┌──────────┐      ┌──────────┐          ┌──────────┐
 │LEAF NODE │      │LEAF NODE │          │LEAF NODE │
 │40k→row5  │◄────►│60k→row2  │◄────────►│80k→row9  │ ← doubly linked
 │41k→row12 │      │61k→row44 │          │81k→row33 │
 │42k→row8  │      │62k→row77 │          │82k→row11 │
 └──────────┘      └──────────┘          └──────────┘
       │                  │                     │
       ▼                  ▼                     ▼
 ┌──────────┐      ┌──────────┐          ┌──────────┐
 │TABLE HEAP│      │TABLE HEAP│          │TABLE HEAP│
 │  (rows)  │      │  (rows)  │          │  (rows)  │
 └──────────┘      └──────────┘          └──────────┘
```
*Figure 2.1 — B-Tree Index Structure (salary column)*

**Key properties:** Leaf nodes are **doubly linked** (great for range queries like `salary BETWEEN 50k AND 70k`). Every search starts at the root and traverses **O(log n)** levels — even for 10M rows that's only ~24 comparisons!

---

## 2.2 EXPLAIN ANALYZE — Your Best Friend

Always use `EXPLAIN ANALYZE` to see what plan PostgreSQL chose:

```sql
EXPLAIN ANALYZE
SELECT * FROM employees WHERE salary = 75000;

-- Output WITHOUT index (Sequential Scan):
-- Seq Scan on employees  (cost=0.00..23456.00 rows=50 width=80)
--                        (actual time=45.2..389.1 rows=50 loops=1)
-- Planning Time: 0.5 ms
-- Execution Time: 389.6 ms  ← SLOW

-- Output WITH index (Index Scan):
-- Index Scan using idx_salary on employees
--   Index Cond: (salary = 75000)
--   (actual time=0.1..0.4 rows=50 loops=1)
-- Execution Time: 0.5 ms   ← FAST ✅
```

---

# Chapter 3: Creating Indexes

## 3.1 Basic Syntax

```sql
-- Syntax
CREATE INDEX [CONCURRENTLY] [index_name]
    ON table_name [USING method]
    (column1 [ASC|DESC], column2, ...)
    [WHERE condition]    -- partial index
    [INCLUDE (col3)]     -- covering index
;
```

```sql
-- 1. Simplest possible index (B-Tree by default)
CREATE INDEX ON employees (salary);

-- 2. Named index (recommended for production)
CREATE INDEX idx_emp_salary ON employees (salary);

-- 3. CONCURRENTLY — create without locking the table (production-safe)
CREATE INDEX CONCURRENTLY idx_emp_dept ON employees (dept);

-- 4. Verify it was created
SELECT indexname, indexdef
FROM pg_indexes
WHERE tablename = 'employees';
```

> **⚡ CONCURRENTLY Tip:** Always use `CREATE INDEX CONCURRENTLY` in production. Without it, PostgreSQL holds an exclusive lock that blocks ALL writes on the table until the index builds. CONCURRENTLY only holds a brief lock at the end but takes ~2× longer to build.

---

## 3.2 Index with ASC / DESC

```sql
-- Index for ORDER BY salary DESC queries
CREATE INDEX idx_emp_salary_desc ON employees (salary DESC);

-- Composite with mixed sort
CREATE INDEX idx_dept_salary ON employees (dept ASC, salary DESC);

-- This query now uses the index perfectly:
SELECT * FROM employees
ORDER BY dept ASC, salary DESC
LIMIT 10;
```

---

# Chapter 4: Unique Indexes

A UNIQUE index enforces that every value in the indexed column(s) is distinct. PostgreSQL automatically creates one for PRIMARY KEY and UNIQUE constraints.

```sql
-- 1. Simple unique index — no duplicate emails allowed
CREATE UNIQUE INDEX idx_emp_email ON employees (email);

-- 2. Unique index on multiple columns (composite unique)
--    Same dept + name combination must not repeat
CREATE UNIQUE INDEX idx_dept_name_unique
    ON employees (dept, name);

-- 3. Test it
INSERT INTO employees (name, dept, salary, email)
VALUES ('Alice', 'IT', 60000, 'emp1@company.com');
-- ERROR: duplicate key value violates unique constraint

-- 4. Partial unique index — unique ONLY among active employees
CREATE UNIQUE INDEX idx_active_email_unique
    ON employees (email)
    WHERE is_active = true;
-- Two inactive employees CAN share the same email
```

| Scenario | What to Use | Behavior |
|---|---|---|
| One email per person | `UNIQUE INDEX (email)` | Rejects duplicates globally |
| One active account per email | `UNIQUE INDEX (email) WHERE is_active=true` | Allows same email for inactive records |
| Primary key | `PRIMARY KEY` constraint | Auto-creates unique index + NOT NULL |
| Natural key across 2 cols | `UNIQUE INDEX (col1, col2)` | Combination must be unique |

---

# Chapter 5: Indexes on Expressions

Sometimes you query by a transformed version of a column — like `lower(email)` or `extract(year FROM hire_date)`. A regular index on the raw column won't help. Expression indexes solve this.

```sql
-- Problem: This query can't use an index on email
SELECT * FROM employees WHERE lower(email) = 'emp123@company.com';

-- Solution: Index the expression itself
CREATE INDEX idx_emp_email_lower
    ON employees (lower(email));

-- Now this query is fast (same expression must be used):
SELECT * FROM employees WHERE lower(email) = 'emp123@company.com'; ✅
```

```sql
-- Index on date part extraction
CREATE INDEX idx_emp_hire_year
    ON employees (EXTRACT(YEAR FROM hire_date));

-- Fast query:
SELECT * FROM employees
WHERE EXTRACT(YEAR FROM hire_date) = 2022;

-- Index on string length
CREATE INDEX idx_emp_name_len
    ON employees (length(name));

-- Index on computed value (10% bonus column)
CREATE INDEX idx_emp_bonus
    ON employees ((salary * 0.10));
```

> **⚠️ Critical Rule:** The WHERE clause in your query MUST use exactly the same expression as the index. If your index is on `lower(email)` but your query uses `LOWER(email)` — that's fine (case-insensitive). But if you use `email ILIKE 'x%'`, PostgreSQL won't use it.

---

# Chapter 6: Partial Indexes

A **Partial Index** only indexes rows that match a WHERE condition. This produces a **smaller, faster index** — perfect when your queries always filter by the same condition.

```
FULL INDEX on is_active                  PARTIAL INDEX WHERE is_active = true
┌──────────────────────────┐             ┌───────────────────────────────┐
│ false → row 100          │             │ row 1   → Alice  (active)     │
│ false → row 200          │             │ row 3   → Carol  (active)     │
│ false → row 300   ──────►│  DROP 90% ─►│ row 5   → Eve    (active)     │
│ true  → row 1            │             │ row 7   → Grace  (active)     │
│ true  → row 3            │             └───────────────────────────────┘
│ true  → row 5            │             Size: 10% of full index ✅
│ ... 1M rows total        │             Only indexes what you actually query
└──────────────────────────┘
```
*Figure 6.1 — Partial index eliminates irrelevant rows*

```sql
-- Partial index: only index active employees
CREATE INDEX idx_active_employees
    ON employees (salary)
    WHERE is_active = true;

-- Query MUST include the partial condition to use this index
SELECT * FROM employees
WHERE salary > 80000
  AND is_active = true;   -- ← this makes PostgreSQL use the partial index

-- Partial unique index: unique email only among active records
CREATE UNIQUE INDEX idx_active_email
    ON employees (email)
    WHERE is_active = true;

-- Partial index for non-null values
CREATE INDEX idx_emp_dept_notnull
    ON employees (dept)
    WHERE dept IS NOT NULL;
```

| Use Case | Example Condition | Benefit |
|---|---|---|
| Active records only | `WHERE is_active = true` | Skip 90% of rows if most are inactive |
| Recent data | `WHERE hire_date > '2020-01-01'` | Only index new entries |
| Non-null values | `WHERE email IS NOT NULL` | Skip rows with no email |
| Error/exception rows | `WHERE status = 'ERROR'` | Index only rare error states quickly |

---

# Chapter 7: Multicolumn Indexes

A multicolumn (composite) index covers multiple columns. Column **order matters greatly** — PostgreSQL can use the index from the leftmost column(s) forward.

```
INDEX: (dept, salary)

Leaf nodes are sorted: FIRST by dept, THEN by salary within each dept

┌──────────────────────────────────────────────────────────────────┐
│ Finance/40k │ Finance/45k │ HR/38k │ HR/55k │ IT/52k │ IT/99k  │
└──────────────────────────────────────────────────────────────────┘
      │               │          │          │         │         │
    row44           row92       row3       row77     row8     row21

✅  WHERE dept = 'IT'                → uses index (leftmost col)
✅  WHERE dept = 'IT' AND salary>70k → uses index (both cols)
❌  WHERE salary > 70000             → full scan (skipped leftmost col)
```
*Figure 7.1 — Column order in composite indexes*

```sql
-- Composite index on (dept, salary)
CREATE INDEX idx_dept_salary
    ON employees (dept, salary);

-- ✅ Uses the index (dept is leftmost column)
SELECT * FROM employees WHERE dept = 'IT';

-- ✅ Uses the index (both columns)
SELECT * FROM employees WHERE dept = 'IT' AND salary > 70000;

-- ❌ Does NOT use this index (missing dept, can't skip leading column)
SELECT * FROM employees WHERE salary > 70000;

-- For salary-only queries, create a separate index:
CREATE INDEX idx_salary ON employees (salary);
```

---

## 7.1 Covering Indexes (INCLUDE)

A covering index includes extra columns with `INCLUDE` so PostgreSQL can answer a query entirely from the index without touching the heap (table). This is called an **Index-Only Scan** — the fastest possible access pattern.

```sql
-- Regular index: must fetch name from the table heap
CREATE INDEX idx_dept ON employees (dept);

-- Covering index: includes name, so no heap fetch needed
CREATE INDEX idx_dept_covering
    ON employees (dept)
    INCLUDE (name, salary);

-- This query can now use Index-Only Scan (zero heap access):
SELECT name, salary
FROM employees
WHERE dept = 'IT';

-- EXPLAIN output will show:
-- Index Only Scan using idx_dept_covering on employees
--   Heap Fetches: 0  ← perfect!
```

---

# Chapter 8: Reindexing with REINDEX

Over time, indexes can become bloated (wasted space from deleted rows) or corrupted. REINDEX rebuilds them cleanly.

```sql
-- Reindex a single index
REINDEX INDEX idx_emp_salary;

-- Reindex ALL indexes on a table
REINDEX TABLE employees;

-- Reindex everything in a database (DBA task)
REINDEX DATABASE mydb;

-- CONCURRENTLY — does NOT lock the table (PostgreSQL 12+)
REINDEX INDEX CONCURRENTLY idx_emp_salary;
REINDEX TABLE CONCURRENTLY employees;

-- Check index bloat before deciding to reindex:
SELECT
    relname AS index_name,
    pg_size_pretty(pg_relation_size(oid)) AS size
FROM pg_class
WHERE relkind = 'i'
  AND relname LIKE 'idx_%'
ORDER BY pg_relation_size(oid) DESC;
```

> **🔧 When to REINDEX:**
> 1. After bulk DELETEs or UPDATEs (dead tuples bloat indexes)
> 2. After a hardware issue that may have corrupted data
> 3. When EXPLAIN ANALYZE shows unexpectedly high index costs
> 4. Run `pg_stat_user_indexes` to monitor index usage over time

---

# Chapter 9: Displaying Index Information

```sql
-- 1. List all indexes on a table
SELECT indexname, indexdef
FROM pg_indexes
WHERE tablename = 'employees';

-- 2. Detailed index info with size
SELECT
    i.relname                               AS index_name,
    ix.indisunique                          AS is_unique,
    ix.indisprimary                         AS is_primary,
    pg_size_pretty(pg_relation_size(i.oid)) AS index_size,
    array_agg(a.attname ORDER BY k.n)       AS columns
FROM pg_class t
JOIN pg_index ix   ON t.oid = ix.indrelid
JOIN pg_class i    ON i.oid = ix.indexrelid
JOIN LATERAL unnest(ix.indkey) WITH ORDINALITY AS k(col, n) ON true
JOIN pg_attribute a ON a.attrelid = t.oid AND a.attnum = k.col
WHERE t.relname = 'employees'
GROUP BY i.relname, ix.indisunique, ix.indisprimary, i.oid
ORDER BY pg_relation_size(i.oid) DESC;

-- 3. Index usage stats (which indexes are actually being hit?)
SELECT
    indexrelname  AS index_name,
    idx_scan      AS times_used,
    idx_tup_read  AS tuples_read,
    idx_tup_fetch AS tuples_fetched
FROM pg_stat_user_indexes
WHERE relname = 'employees'
ORDER BY idx_scan DESC;

-- 4. Find UNUSED indexes (candidates for removal)
SELECT schemaname, relname, indexrelname, idx_scan
FROM pg_stat_user_indexes
WHERE idx_scan = 0
  AND indexrelname NOT LIKE 'pg_%'
ORDER BY relname;

-- 5. psql shortcut (within psql shell only)
\d employees   -- shows table with all its indexes
```

---

# Chapter 10: Dropping Indexes

```sql
-- Drop a single index
DROP INDEX idx_emp_salary;

-- Drop only if it exists (no error if missing)
DROP INDEX IF EXISTS idx_emp_salary;

-- CONCURRENTLY — don't lock the table during drop
DROP INDEX CONCURRENTLY idx_emp_salary;

-- Drop multiple indexes at once
DROP INDEX idx_emp_salary, idx_emp_email, idx_dept_salary;

-- Drop an index backing a constraint (need to drop the constraint instead)
ALTER TABLE employees DROP CONSTRAINT employees_email_key;
-- This also drops the underlying unique index
```

### Index Costs vs Benefits

| ❌ Index Costs | ✅ Index Benefits |
|---|---|
| Every write (INSERT/UPDATE/DELETE) must update the index | Dramatically speeds up SELECT queries |
| Occupies disk space | Enforces data uniqueness (UNIQUE) |
| Outdated statistics may confuse the planner | Enables fast ORDER BY without sort |
| Unused indexes waste maintenance overhead | Allows Index-Only Scans (no heap fetch) |

---

# Chapter 11: Types of Indexes

| Index Type | Best For | Supports | When to Use |
|---|---|---|---|
| **B-Tree** (default) | Equality & range queries | `=  <  >  BETWEEN  LIKE 'x%'` | Almost always — your default choice |
| **Hash** | Equality only | `=` only | Only when you only do `=` lookups (rare) |
| **GiST** | Geometric, full-text, ranges | `&&  @>  <->` | PostGIS, tsvector, ranges |
| **GIN** | Multi-value columns | `@>  @@  ?` | JSONB, arrays, full-text search |
| **BRIN** | Very large tables with natural order | Range scans | Logs, time-series, append-only tables |
| **SP-GiST** | Non-balanced structures | Point lookups | IP ranges, phone trees, quadtrees |

---

## 11.1 B-Tree (Default)

```sql
-- B-Tree: works for equality, ranges, sorting, LIKE with prefix
CREATE INDEX idx_btree_salary ON employees USING BTREE (salary);

SELECT * FROM employees WHERE salary = 75000;              -- ✅ equality
SELECT * FROM employees WHERE salary BETWEEN 60000 AND 80000; -- ✅ range
SELECT * FROM employees WHERE name LIKE 'Ali%';            -- ✅ prefix LIKE
SELECT * FROM employees WHERE name LIKE '%ali%';           -- ❌ not usable
```

## 11.2 Hash Index

```sql
-- Hash: only useful for strict equality (=)
CREATE INDEX idx_hash_dept ON employees USING HASH (dept);

SELECT * FROM employees WHERE dept = 'IT';   -- ✅ exact match
SELECT * FROM employees WHERE dept > 'HR';   -- ❌ not supported

-- Note: B-Tree almost always wins unless you benchmark otherwise
```

## 11.3 BRIN Index

```sql
-- BRIN: very small index for naturally-ordered large tables
-- (e.g. timestamp columns on append-only logs)
CREATE INDEX idx_brin_hire ON employees USING BRIN (hire_date);

-- Tiny size but only efficient when physical row order matches value order
SELECT * FROM employees WHERE hire_date BETWEEN '2022-01-01' AND '2023-01-01';

-- Check BRIN size vs B-Tree size:
SELECT pg_size_pretty(pg_relation_size('idx_brin_hire')) AS brin_size;
-- Often 100-1000x smaller than B-Tree!
```

---

# Chapter 12: Full Text Search Indexes

Full Text Search lets you find documents containing words, with **stemming** (run/runs/running → same), **ranking**, and **stop word removal**. It uses GIN or GiST indexes on `tsvector` (text search vector) data.

```
FULL TEXT SEARCH PIPELINE

Raw text: 'PostgreSQL Indexing makes databases run faster'
     │
     ▼  to_tsvector()
tsvector: 'database':5 'faster':7 'index':2 'make':4 'postgresql':1 'run':6
     │    (stop words removed; stemming applied)
     │
     ▼  CREATE GIN INDEX on tsvector column
GIN Index: inverted index  →  'database' → [row5, row9, row44]
                              'index'    → [row1, row5, row88]
                              'run'      → [row5, row12, row200]
     │
     ▼  Query: to_tsquery('indexing & database')
Result: rows where BOTH 'index' AND 'database' appear  → row5, row9
```
*Figure 12.1 — Full Text Search pipeline*

```sql
-- Step 1: Add a tsvector column
ALTER TABLE employees ADD COLUMN bio_search TSVECTOR;

-- Step 2: Populate it
UPDATE employees
SET bio_search = to_tsvector('english', coalesce(bio, ''));

-- Step 3: Create GIN index on the tsvector column
CREATE INDEX idx_fts_bio ON employees USING GIN (bio_search);

-- Step 4: Keep it updated automatically with a trigger
CREATE FUNCTION update_bio_search() RETURNS TRIGGER AS $$
BEGIN
    NEW.bio_search := to_tsvector('english', coalesce(NEW.bio, ''));
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_bio_search
    BEFORE INSERT OR UPDATE ON employees
    FOR EACH ROW EXECUTE FUNCTION update_bio_search();
```

```sql
-- Basic full text search query
SELECT name, bio
FROM employees
WHERE bio_search @@ to_tsquery('english', 'database & index');

-- With relevance ranking
SELECT name, bio,
       ts_rank(bio_search, query) AS relevance
FROM employees,
     to_tsquery('english', 'database | performance') AS query
WHERE bio_search @@ query
ORDER BY relevance DESC
LIMIT 10;

-- Highlight matching terms
SELECT name,
       ts_headline('english', bio,
                   to_tsquery('database'),
                   'MaxWords=20, MinWords=10') AS excerpt
FROM employees
WHERE bio_search @@ to_tsquery('database');

-- Alternative: computed column approach (no separate column needed)
CREATE INDEX idx_fts_bio_computed
    ON employees
    USING GIN (to_tsvector('english', coalesce(bio, '')));

-- Query with computed approach:
SELECT * FROM employees
WHERE to_tsvector('english', coalesce(bio,'')) @@ to_tsquery('database');
```

---

# Chapter 13: JSONB Indexes

PostgreSQL's **JSONB** type stores JSON in a binary, parsed format. You can index specific keys, entire documents (GIN), or expressions extracted from JSON.

## 13.1 GIN Index on Entire JSONB Column

```sql
-- Recall: our employees.profile column is JSONB
-- Example: {'level': 3, 'skills': ['SQL', 'Python', 'Docker']}

-- GIN index — indexes ALL keys and values in the JSON
CREATE INDEX idx_gin_profile
    ON employees USING GIN (profile);

-- Supported operators with GIN index:

-- @> (contains) — employee has 'level': 3
SELECT * FROM employees
WHERE profile @> '{"level": 3}';

-- ? (key exists) — has 'skills' key
SELECT * FROM employees
WHERE profile ? 'skills';

-- ?| (any key exists)
SELECT * FROM employees
WHERE profile ?| ARRAY['skills', 'department'];

-- ?& (all keys exist)
SELECT * FROM employees
WHERE profile ?& ARRAY['level', 'skills'];
```

## 13.2 B-Tree Index on Extracted JSON Value

```sql
-- Index a specific JSON field with expression index
CREATE INDEX idx_profile_level
    ON employees ((profile->>'level'));

-- Fast query on that field:
SELECT * FROM employees
WHERE profile->>'level' = '3';

-- Cast for numeric comparisons:
CREATE INDEX idx_profile_level_int
    ON employees (((profile->>'level')::int));

SELECT * FROM employees
WHERE (profile->>'level')::int > 3;
```

## 13.3 GIN with jsonb_path_ops (Leaner GIN)

```sql
-- jsonb_path_ops: smaller GIN index, only supports @> operator
CREATE INDEX idx_gin_profile_path
    ON employees
    USING GIN (profile jsonb_path_ops);

-- Supported (same @> containment query):
SELECT * FROM employees
WHERE profile @> '{"skills": ["Python"]}';

-- NOT supported (no ? or ?| with jsonb_path_ops)

-- PostgreSQL 12+: JSON Path queries
SELECT * FROM employees
WHERE profile @@ '$.level == 3';
```

| Index | Size | Operators | Best For |
|---|---|---|---|
| GIN (default_ops) | Largest | `@>  ?  ?|  ?&` | General JSONB querying |
| GIN (jsonb_path_ops) | ~2-3x smaller | `@>` only | High-volume containment queries |
| B-Tree on `(profile->>'key')` | Smallest | `= < > BETWEEN` | Querying a single known field |
| GiST on jsonb | Medium | `@>  &&` | Rarely needed; usually GIN wins |

---

# Chapter 14: Best Practices & Decision Guide

```
INDEXING DECISION FLOWCHART

Is your query slow?
       │
       ▼
Run EXPLAIN ANALYZE ────► Seq Scan? ────► How many rows returned?
                              │                    │
                              │             < 5% of table
                              │             → CREATE INDEX ✅
                              │
                              │             > 15% of table
                              │             → Seq Scan is fine ✅
                              │
                        What column type?
                        ┌──────┬──────┬──────┬──────┐
                        │      │      │      │      │
                     scalar  text   JSONB  geo    FTS
                     B-Tree  GIN    GIN   GiST   GIN
                             FTS   B-Tree
```
*Figure 14.1 — Indexing decision flowchart*

---

## 14.1 Golden Rules

1. **Index columns in WHERE, JOIN ON, and ORDER BY** — These are the columns that matter most for query performance.
2. **Always name your indexes** — Use `idx_tablename_column` convention for easy identification.
3. **Use CONCURRENTLY in production** — Prevents table locks during index creation and removal.
4. **Don't over-index** — Each index slows down INSERT/UPDATE/DELETE. Aim for quality over quantity.
5. **Leftmost column rule for composites** — Build composite indexes with the most selective column first.
6. **Monitor with pg_stat_user_indexes** — Drop indexes with `idx_scan = 0` — they're just dead weight.
7. **Partial indexes save space** — Add WHERE conditions to exclude rows you never query.
8. **INCLUDE for Index-Only Scans** — Add non-filter columns with INCLUDE to eliminate heap fetches.

---

## 14.2 Quick Reference Cheat Sheet

| Task | SQL |
|---|---|
| Create basic index | `CREATE INDEX idx_name ON tbl (col);` |
| Create safely in prod | `CREATE INDEX CONCURRENTLY idx_name ON tbl (col);` |
| Unique index | `CREATE UNIQUE INDEX idx_name ON tbl (col);` |
| Expression index | `CREATE INDEX idx ON tbl (lower(email));` |
| Partial index | `CREATE INDEX idx ON tbl (col) WHERE active=true;` |
| Composite index | `CREATE INDEX idx ON tbl (col1, col2);` |
| Covering index | `CREATE INDEX idx ON tbl (col1) INCLUDE (col2);` |
| Full text search | `CREATE INDEX idx ON tbl USING GIN (to_tsvector('english', bio));` |
| JSONB all keys | `CREATE INDEX idx ON tbl USING GIN (profile);` |
| JSONB one key | `CREATE INDEX idx ON tbl ((profile->>'level'));` |
| Rebuild index | `REINDEX INDEX CONCURRENTLY idx_name;` |
| Drop index | `DROP INDEX CONCURRENTLY IF EXISTS idx_name;` |
| Show all indexes | `SELECT * FROM pg_indexes WHERE tablename='tbl';` |
| Find unused indexes | `SELECT * FROM pg_stat_user_indexes WHERE idx_scan=0;` |

---

*PostgreSQL Indexing Mastery Guide*
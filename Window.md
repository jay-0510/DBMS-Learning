# PostgreSQL Window Functions Mastery
> From Basics to Advanced — Complete Guide

---

# Chapter 1: What Are Window Functions?

A **Window Function** performs a calculation across a set of rows that are related to the current row — without collapsing those rows into a single result like `GROUP BY` does. Each row keeps its identity while gaining access to aggregated or ranked information from its neighbors.

> **📖 Definition:** A window function computes a value for each row based on a "window" (a sliding frame of related rows). The result appears as an extra column alongside every row — no rows are removed, no grouping happens.

```
  GROUP BY (collapses rows)              WINDOW FUNCTION (keeps all rows)

  dept      avg_salary                  name       dept    salary  avg_salary
  ────────  ──────────                  ─────────  ──────  ──────  ──────────
  IT        72,000        vs            Alice      IT      85,000  72,000
  HR        58,000                      Bob        IT      65,000  72,000
  Finance   91,000                      Carol      IT      66,000  72,000
                                        Dave       HR      60,000  58,000
  3 rows returned ❌                    Eve        HR      56,000  58,000
  (original rows gone)                  Frank      Finance 91,000  91,000
                                        6 rows returned ✅
```
*Figure 1.1 — GROUP BY collapses rows; Window Functions preserve them*

---

## 1.1 The OVER() Clause — Heart of Every Window Function

Every window function uses `OVER()` to define its window. Without `OVER()`, it's just a regular aggregate.

```sql
function_name()  OVER (
    [PARTITION BY col1, col2, ...]   -- divide rows into groups
    [ORDER BY col3 [ASC|DESC], ...]  -- sort within each group
    [frame_clause]                   -- optional: which rows to include
)
```

```
  OVER() Anatomy

  PARTITION BY dept    →  splits rows into separate windows per dept
       │
       ▼
  ┌──── dept=IT ────────┐  ┌──── dept=HR ─────┐  ┌── dept=Finance ──┐
  │ Alice  85k          │  │ Dave   60k        │  │ Frank  91k       │
  │ Bob    65k          │  │ Eve    56k        │  │ Grace  88k       │
  │ Carol  66k          │  └───────────────────┘  └──────────────────┘
  └─────────────────────┘
       │
  ORDER BY salary DESC  →  sorts rows within each partition
       │
       ▼
  ┌──── dept=IT ────────┐
  │ 1. Alice  85k       │
  │ 2. Carol  66k       │
  │ 3. Bob    65k       │
  └─────────────────────┘
```
*Figure 1.2 — PARTITION BY creates independent windows; ORDER BY sorts within them*

---

## 1.2 Setting Up Our Example Table

```sql
-- employees table used throughout every example
CREATE TABLE employees (
    emp_id    SERIAL PRIMARY KEY,
    name      VARCHAR(100),
    dept      VARCHAR(50),
    salary    NUMERIC(10,2),
    hire_date DATE,
    sales     NUMERIC(10,2)
);

INSERT INTO employees (name, dept, salary, hire_date, sales) VALUES
    ('Alice',   'IT',      85000, '2018-03-15', 120000),
    ('Bob',     'IT',      65000, '2020-07-01', 95000),
    ('Carol',   'IT',      66000, '2019-11-20', 98000),
    ('Dave',    'HR',      60000, '2017-05-10', 40000),
    ('Eve',     'HR',      56000, '2021-02-28', 35000),
    ('Frank',   'Finance', 91000, '2016-09-01', 200000),
    ('Grace',   'Finance', 88000, '2018-04-12', 185000),
    ('Heidi',   'Finance', 76000, '2022-01-05', 160000),
    ('Ivan',    'Sales',   55000, '2020-08-19', 310000),
    ('Judy',    'Sales',   58000, '2019-03-22', 295000),
    ('Karl',    'Sales',   52000, '2021-06-30', 275000),
    ('Laura',   'Sales',   61000, '2017-12-01', 330000),
    ('Mike',    'IT',      72000, '2015-10-10', 110000),
    ('Nina',    'HR',      63000, '2018-07-25', 42000),
    ('Oscar',   'Finance', 95000, '2014-03-18', 210000);
```

---

## 1.3 Window Function Categories

```
  RANKING FUNCTIONS         VALUE FUNCTIONS           DISTRIBUTION FUNCTIONS
  ┌───────────────────┐     ┌───────────────────┐     ┌──────────────────────┐
  │ ROW_NUMBER()      │     │ LAG()             │     │ PERCENT_RANK()       │
  │ RANK()            │     │ LEAD()            │     │ CUME_DIST()          │
  │ DENSE_RANK()      │     │ FIRST_VALUE()     │     │ NTILE()              │
  │ NTILE()           │     │ LAST_VALUE()      │     └──────────────────────┘
  └───────────────────┘     │ NTH_VALUE()       │
                            └───────────────────┘
```

| Function | Returns | Needs ORDER BY? | Typical Use |
|---|---|---|---|
| `ROW_NUMBER()` | Unique sequential integer | Yes | Pagination, deduplication |
| `RANK()` | Rank with gaps on ties | Yes | Leaderboards |
| `DENSE_RANK()` | Rank without gaps on ties | Yes | Medals (1st, 2nd, 3rd) |
| `NTILE(n)` | Bucket number 1..n | Yes | Quartiles, percentiles |
| `PERCENT_RANK()` | 0.0 to 1.0 relative rank | Yes | Percentile position |
| `CUME_DIST()` | 0.0+ to 1.0 cumulative % | Yes | "Top X%" reports |
| `LAG()` | Value from a previous row | Yes | Period-over-period comparison |
| `LEAD()` | Value from a future row | Yes | Forecasting, next-row access |
| `FIRST_VALUE()` | First value in the window | Yes | Compare to window's best |
| `LAST_VALUE()` | Last value in the window | Yes | Compare to window's worst |
| `NTH_VALUE()` | Value at position N | Yes | 2nd place, 3rd highest, etc. |

---

# Chapter 2: ROW_NUMBER()

`ROW_NUMBER()` assigns a **unique sequential integer** to each row within its partition. Ties get different numbers — there's no concept of "same rank" here.

```sql
-- Syntax
ROW_NUMBER() OVER (
    [PARTITION BY ...]
    ORDER BY ...
)
```

## 2.1 Basic ROW_NUMBER

```sql
-- Assign a global row number ordered by salary
SELECT
    name,
    dept,
    salary,
    ROW_NUMBER() OVER (ORDER BY salary DESC) AS row_num
FROM employees;

-- Result:
-- name    dept     salary   row_num
-- Oscar   Finance  95000    1
-- Frank   Finance  91000    2
-- Grace   Finance  88000    3
-- Alice   IT       85000    4
-- ...
```

## 2.2 ROW_NUMBER per Partition

```sql
-- Rank employees within each department by salary
SELECT
    name,
    dept,
    salary,
    ROW_NUMBER() OVER (
        PARTITION BY dept
        ORDER BY salary DESC
    ) AS dept_rank
FROM employees;

-- Result:
-- name    dept     salary   dept_rank
-- Oscar   Finance  95000    1       <- Finance window restarts at 1
-- Frank   Finance  91000    2
-- Grace   Finance  88000    3
-- Heidi   Finance  76000    4
-- Alice   IT       85000    1       <- IT window restarts at 1
-- Mike    IT       72000    2
-- Carol   IT       66000    3
-- Bob     IT       65000    4
-- Nina    HR       63000    1       <- HR window restarts at 1
```

## 2.3 Practical Use: Get Top N per Group

```sql
-- Top 2 earners per department (classic pattern)
SELECT name, dept, salary, dept_rank
FROM (
    SELECT
        name, dept, salary,
        ROW_NUMBER() OVER (
            PARTITION BY dept
            ORDER BY salary DESC
        ) AS dept_rank
    FROM employees
) ranked
WHERE dept_rank <= 2;

-- Result:
-- name    dept     salary   dept_rank
-- Oscar   Finance  95000    1
-- Frank   Finance  91000    2
-- Alice   IT       85000    1
-- Mike    IT       72000    2
-- Nina    HR       63000    1
-- Dave    HR       60000    2
-- Laura   Sales    61000    1
-- Judy    Sales    58000    2
```

## 2.4 Practical Use: Deduplication

```sql
-- Remove duplicate rows — keep only the first occurrence
WITH deduped AS (
    SELECT *,
        ROW_NUMBER() OVER (
            PARTITION BY name, dept   -- define what makes a "duplicate"
            ORDER BY hire_date        -- keep the earliest record
        ) AS rn
    FROM employees
)
DELETE FROM employees
WHERE emp_id IN (
    SELECT emp_id FROM deduped WHERE rn > 1
);
```

---

# Chapter 3: RANK()

`RANK()` assigns a rank to each row, but **ties receive the same rank** and the next rank **skips** numbers (leaves gaps).

```
  RANK() Example — salary column

  Alice   85000  -> rank 1
  Mike    72000  -> rank 2
  Carol   66000  -> rank 3   <- tie
  Bob     65000  -> rank 3   <- tie (same salary)
  Ivan    55000  -> rank 5   <- GAP! (rank 4 is skipped)
```

```sql
SELECT
    name,
    dept,
    salary,
    RANK() OVER (ORDER BY salary DESC) AS salary_rank
FROM employees;

-- name    dept     salary   salary_rank
-- Oscar   Finance  95000    1
-- Frank   Finance  91000    2
-- Grace   Finance  88000    3
-- Alice   IT       85000    4
-- Heidi   Finance  76000    5
-- Mike    IT       72000    6
-- ...
```

## 3.1 RANK per Partition

```sql
-- Rank by salary within each department
SELECT
    name,
    dept,
    salary,
    RANK() OVER (
        PARTITION BY dept
        ORDER BY salary DESC
    ) AS dept_salary_rank
FROM employees
ORDER BY dept, dept_salary_rank;
```

## 3.2 Practical Use: Find All Employees with Same Rank

```sql
-- Find all employees sharing the top salary rank in their department
SELECT name, dept, salary, dept_rank
FROM (
    SELECT
        name, dept, salary,
        RANK() OVER (
            PARTITION BY dept
            ORDER BY salary DESC
        ) AS dept_rank
    FROM employees
) r
WHERE dept_rank = 1;
-- Returns ALL employees tied for first in each department
```

---

# Chapter 4: DENSE_RANK()

`DENSE_RANK()` is like `RANK()` but **never skips numbers**. Ties get the same rank, and the next distinct value gets the very next integer.

```
  Comparing RANK vs DENSE_RANK

  salary    RANK()    DENSE_RANK()
  ------    ------    ------------
  95000       1            1
  91000       2            2
  88000       3            3
  85000       4            4
  72000       5            5
  66000       6            6      <- tie starts
  66000       6            6      <- same rank
  65000       8   <- GAP   7   <- NO GAP
  63000       9            8
  61000      10            9
```

```sql
-- Side-by-side comparison
SELECT
    name,
    dept,
    salary,
    RANK()       OVER (ORDER BY salary DESC) AS rnk,
    DENSE_RANK() OVER (ORDER BY salary DESC) AS dense_rnk
FROM employees
ORDER BY salary DESC;
```

## 4.1 DENSE_RANK per Partition

```sql
-- Medal-style ranking within each department (no gaps)
SELECT
    name,
    dept,
    salary,
    DENSE_RANK() OVER (
        PARTITION BY dept
        ORDER BY salary DESC
    ) AS medal_rank
FROM employees
ORDER BY dept, medal_rank;
```

## 4.2 Practical Use: Salary Bands

```sql
-- Label employees by their salary tier (1=highest band, no gaps)
SELECT
    name,
    salary,
    DENSE_RANK() OVER (ORDER BY
        CASE
            WHEN salary >= 90000 THEN 1
            WHEN salary >= 70000 THEN 2
            WHEN salary >= 55000 THEN 3
            ELSE 4
        END
    ) AS salary_band
FROM employees
ORDER BY salary_band, salary DESC;
```

---

# Chapter 5: NTILE()

`NTILE(n)` divides the ordered rows into **n roughly equal buckets** and assigns each row a bucket number from 1 to n. Useful for quartiles, deciles, and percentile groupings.

```
  NTILE(4) on salary — 15 employees split into 4 quartiles

  Bucket 1 (top 25%):    Oscar 95k, Frank 91k, Grace 88k, Alice 85k
  Bucket 2 (next 25%):   Heidi 76k, Mike 72k, Carol 66k, Bob 65k
  Bucket 3 (next 25%):   Nina 63k, Laura 61k, Dave 60k, Judy 58k
  Bucket 4 (bottom 25%): Ivan 55k, Eve 56k, Karl 52k

  Note: 15 rows / 4 buckets → buckets 1,2,3 get 4 rows, bucket 4 gets 3
  PostgreSQL distributes remainder rows to the first buckets
```
*Figure 5.1 — NTILE(4) creates quartile buckets*

```sql
SELECT
    name,
    salary,
    NTILE(4) OVER (ORDER BY salary DESC) AS quartile
FROM employees
ORDER BY salary DESC;

-- name    salary   quartile
-- Oscar   95000    1
-- Frank   91000    1
-- Grace   88000    1
-- Alice   85000    1
-- Heidi   76000    2
-- Mike    72000    2
-- Carol   66000    2
-- Bob     65000    2
-- Nina    63000    3
-- Laura   61000    3
-- Dave    60000    3
-- Judy    58000    3
-- Ivan    55000    4
-- Eve     56000    4
-- Karl    52000    4
```

## 5.1 Practical Use: Quartile Labels

```sql
-- Assign human-readable quartile labels
SELECT
    name,
    dept,
    salary,
    NTILE(4) OVER (ORDER BY salary DESC) AS quartile_num,
    CASE NTILE(4) OVER (ORDER BY salary DESC)
        WHEN 1 THEN 'Top 25%'
        WHEN 2 THEN 'Upper-Mid 25%'
        WHEN 3 THEN 'Lower-Mid 25%'
        WHEN 4 THEN 'Bottom 25%'
    END AS quartile_label
FROM employees
ORDER BY salary DESC;
```

## 5.2 Practical Use: Deciles for Sales Performance

```sql
-- Split into deciles (10 groups) by sales
SELECT
    name,
    dept,
    sales,
    NTILE(10) OVER (ORDER BY sales DESC) AS sales_decile
FROM employees
ORDER BY sales DESC;
```

---

# Chapter 6: PERCENT_RANK()

`PERCENT_RANK()` returns the **relative rank** of a row as a fraction between 0.0 and 1.0. The first row always gets 0.0; the last row gets 1.0.

```
  Formula:  PERCENT_RANK = (rank - 1) / (total_rows - 1)

  For 5 rows:
  Row 1 (highest): (1-1)/(5-1) = 0.00   -> top of the group
  Row 2:           (2-1)/(5-1) = 0.25
  Row 3:           (3-1)/(5-1) = 0.50   -> middle
  Row 4:           (4-1)/(5-1) = 0.75
  Row 5 (lowest):  (5-1)/(5-1) = 1.00   -> bottom of the group
```

```sql
SELECT
    name,
    dept,
    salary,
    ROUND(
        PERCENT_RANK() OVER (ORDER BY salary) * 100, 2
    ) AS percentile_pct
FROM employees
ORDER BY salary;

-- name    salary   percentile_pct
-- Karl    52000    0.00    <- lowest salary, 0th percentile
-- Ivan    55000    7.14
-- Eve     56000    14.29
-- Judy    58000    21.43
-- Dave    60000    28.57
-- Laura   61000    35.71
-- Nina    63000    42.86
-- Bob     65000    50.00   <- median
-- Carol   66000    57.14
-- Mike    72000    64.29
-- Heidi   76000    71.43
-- Alice   85000    78.57
-- Grace   88000    85.71
-- Frank   91000    92.86
-- Oscar   95000    100.00  <- highest salary, 100th percentile
```

## 6.1 PERCENT_RANK per Department

```sql
-- Where does each employee stand within their own department?
SELECT
    name,
    dept,
    salary,
    ROUND(
        PERCENT_RANK() OVER (
            PARTITION BY dept
            ORDER BY salary
        ) * 100, 1
    ) AS dept_percentile
FROM employees
ORDER BY dept, salary;
```

## 6.2 Practical Use: Find Employees Above 75th Percentile

```sql
SELECT name, dept, salary, pct
FROM (
    SELECT
        name, dept, salary,
        PERCENT_RANK() OVER (ORDER BY salary) AS pct
    FROM employees
) p
WHERE pct >= 0.75
ORDER BY salary DESC;
```

---

# Chapter 7: CUME_DIST()

`CUME_DIST()` (Cumulative Distribution) returns the fraction of rows that have a value **less than or equal to** the current row's value. Always between 0.0+ and 1.0.

```
  Formula:  CUME_DIST = rows_with_value_less_than_or_equal / total_rows

  Difference from PERCENT_RANK:
  +----------------+------------------------------------------------------+
  | PERCENT_RANK   | First row = 0.0  uses (rank-1)/(total-1)            |
  | CUME_DIST      | First row > 0.0  uses count_lte_current/total       |
  +----------------+------------------------------------------------------+

  salary   PERCENT_RANK   CUME_DIST
  52000    0.000          0.067  (1/15)
  55000    0.071          0.133  (2/15)
  56000    0.143          0.200  (3/15)
  ...
  95000    1.000          1.000  (15/15)
```

```sql
-- Compare PERCENT_RANK and CUME_DIST side by side
SELECT
    name,
    salary,
    ROUND(PERCENT_RANK() OVER (ORDER BY salary) * 100, 1) AS pct_rank,
    ROUND(CUME_DIST()   OVER (ORDER BY salary) * 100, 1) AS cume_dist_pct
FROM employees
ORDER BY salary;
```

## 7.1 Practical Use: Top 30% of Earners

```sql
-- Which employees are in the top 30% of earners company-wide?
SELECT name, dept, salary, cume_dist_pct
FROM (
    SELECT
        name, dept, salary,
        ROUND(CUME_DIST() OVER (ORDER BY salary DESC) * 100, 1) AS cume_dist_pct
    FROM employees
) c
WHERE cume_dist_pct <= 30
ORDER BY salary DESC;
```

## 7.2 CUME_DIST per Department

```sql
-- Cumulative salary distribution within each department
SELECT
    name,
    dept,
    salary,
    ROUND(
        CUME_DIST() OVER (
            PARTITION BY dept
            ORDER BY salary
        ) * 100, 1
    ) AS dept_cume_dist
FROM employees
ORDER BY dept, salary;
```

---

# Chapter 8: LAG()

`LAG()` accesses the value from a **previous row** in the window — without a self-join. Perfect for period-over-period comparisons.

```sql
-- Syntax
LAG(expression [, offset [, default]])
OVER (
    [PARTITION BY ...]
    ORDER BY ...
)
-- offset:  how many rows back (default = 1)
-- default: value to return if no previous row exists (default = NULL)
```

```
  LAG() — looking backwards

  Row  name    salary    LAG(salary,1)   LAG(salary,2)
  ---- ------  ------    -------------   -------------
   1   Karl    52000     NULL            NULL
   2   Ivan    55000     52000           NULL
   3   Eve     56000     55000           52000
   4   Judy    58000     56000           55000
   5   Dave    60000     58000           56000
   6   Laura   61000     60000           58000

  LAG(salary, 1) -> one row behind current row
  LAG(salary, 2) -> two rows behind current row
```
*Figure 8.1 — LAG() looks back N rows*

```sql
-- Compare each employee's salary to the previous hire in same dept
SELECT
    name,
    dept,
    hire_date,
    salary,
    LAG(salary) OVER (
        PARTITION BY dept
        ORDER BY hire_date
    ) AS prev_hire_salary,
    salary - LAG(salary) OVER (
        PARTITION BY dept
        ORDER BY hire_date
    ) AS salary_diff
FROM employees
ORDER BY dept, hire_date;
```

## 8.1 LAG with Offset and Default

```sql
-- LAG with offset=2 (two rows back) and default=0 (avoids NULL)
SELECT
    name,
    salary,
    LAG(salary, 1, 0) OVER (ORDER BY salary) AS prev_salary,
    LAG(salary, 2, 0) OVER (ORDER BY salary) AS two_back_salary,
    salary - LAG(salary, 1, 0) OVER (ORDER BY salary) AS growth
FROM employees
ORDER BY salary;
```

## 8.2 Practical Use: Percentage Change vs Previous Period

```sql
-- % change in sales vs previous hire within department
SELECT
    name,
    dept,
    hire_date,
    sales,
    LAG(sales) OVER (
        PARTITION BY dept
        ORDER BY hire_date
    ) AS prev_emp_sales,
    ROUND(
        (sales - LAG(sales) OVER (PARTITION BY dept ORDER BY hire_date))
        / NULLIF(LAG(sales) OVER (PARTITION BY dept ORDER BY hire_date), 0)
        * 100, 2
    ) AS pct_change
FROM employees
ORDER BY dept, hire_date;
```

---

# Chapter 9: LEAD()

`LEAD()` is the forward-looking counterpart to `LAG()`. It accesses the value from a **future row** in the window.

```sql
-- Syntax
LEAD(expression [, offset [, default]])
OVER (
    [PARTITION BY ...]
    ORDER BY ...
)
```

```
  LAG vs LEAD — direction comparison

  Row  name    salary    LAG(salary,1)    LEAD(salary,1)
  ---- ------  ------    -------------    --------------
   1   Karl    52000     NULL  <-past      55000  future->
   2   Ivan    55000     52000             56000
   3   Eve     56000     55000             58000
   4   Judy    58000     56000             60000
   5   Dave    60000     58000             61000
   6   Laura   61000     60000             NULL   <- no next row
```
*Figure 9.1 — LAG looks back, LEAD looks forward*

```sql
-- See next employee's salary within same department
SELECT
    name,
    dept,
    salary,
    LEAD(salary) OVER (
        PARTITION BY dept
        ORDER BY salary DESC
    ) AS next_lower_salary,
    salary - LEAD(salary) OVER (
        PARTITION BY dept
        ORDER BY salary DESC
    ) AS gap_to_next
FROM employees
ORDER BY dept, salary DESC;
```

## 9.1 Practical Use: Days Until Next Hire

```sql
-- How many days between each hire within a department?
SELECT
    name,
    dept,
    hire_date,
    LEAD(hire_date) OVER (
        PARTITION BY dept
        ORDER BY hire_date
    ) AS next_hire_date,
    LEAD(hire_date) OVER (
        PARTITION BY dept
        ORDER BY hire_date
    ) - hire_date AS days_until_next_hire
FROM employees
ORDER BY dept, hire_date;
```

## 9.2 LAG + LEAD Together

```sql
-- Show previous, current, and next salary for full context
SELECT
    name,
    dept,
    LAG(salary)  OVER (PARTITION BY dept ORDER BY salary) AS lower_peer,
    salary                                                 AS current_salary,
    LEAD(salary) OVER (PARTITION BY dept ORDER BY salary) AS upper_peer
FROM employees
ORDER BY dept, salary;
```

---

# Chapter 10: FIRST_VALUE()

`FIRST_VALUE()` returns the **first value** in the window frame. Useful for comparing each row to the best/earliest value in its group.

```sql
-- Syntax
FIRST_VALUE(expression)
OVER (
    [PARTITION BY ...]
    ORDER BY ...
    [frame_clause]
)
```

```
  FIRST_VALUE() in the IT department (ordered by salary DESC)

  Window frame (all IT rows ordered by salary DESC):
  +----------------------------------------------------------+
  | 1st -> Alice  85000   <- FIRST_VALUE always returns     |
  | 2nd -> Mike   72000      this value for every row       |
  | 3rd -> Carol  66000      in the partition               |
  | 4th -> Bob    65000                                     |
  +----------------------------------------------------------+

  Result:
  Alice  85000  first_value=85000  gap=0
  Mike   72000  first_value=85000  gap=13000
  Carol  66000  first_value=85000  gap=19000
  Bob    65000  first_value=85000  gap=20000
```
*Figure 10.1 — FIRST_VALUE returns the same top value for every row*

```sql
-- Compare each employee's salary to the highest in their department
SELECT
    name,
    dept,
    salary,
    FIRST_VALUE(salary) OVER (
        PARTITION BY dept
        ORDER BY salary DESC
    ) AS dept_max_salary,
    FIRST_VALUE(name) OVER (
        PARTITION BY dept
        ORDER BY salary DESC
    ) AS top_earner,
    FIRST_VALUE(salary) OVER (
        PARTITION BY dept
        ORDER BY salary DESC
    ) - salary AS gap_from_top
FROM employees
ORDER BY dept, salary DESC;
```

## 10.1 FIRST_VALUE with Full Frame

```sql
-- Use extended frame for consistency across all aggregation patterns
SELECT
    name,
    dept,
    salary,
    FIRST_VALUE(salary) OVER (
        PARTITION BY dept
        ORDER BY salary DESC
        ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
    ) AS dept_max,
    FIRST_VALUE(salary) OVER (
        PARTITION BY dept
        ORDER BY salary ASC
        ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
    ) AS dept_min
FROM employees
ORDER BY dept, salary DESC;
```

---

# Chapter 11: LAST_VALUE()

`LAST_VALUE()` returns the **last value** in the window frame. It requires careful use of the frame clause because the **default frame stops at the current row**, not the end of the partition.

```
  THE LAST_VALUE TRAP

  Default frame: RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW

  Row   name    salary   LAST_VALUE(salary)  <- default frame
  ----- ------  ------   ------------------
   1    Alice   85000    85000  <- last in frame [row1..row1] = itself
   2    Mike    72000    72000  <- last in frame [row1..row2] = itself
   3    Carol   66000    66000  <- last in frame [row1..row3] = itself
   4    Bob     65000    65000  <- last in frame [row1..row4] = itself

  Every row returns ITSELF — not the actual last row! ❌

  FIX: extend the frame to the end of the partition:
  ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING ✅
```
*Figure 11.1 — Always extend the frame for LAST_VALUE*

```sql
-- ❌ Wrong — returns the current row's own salary (default frame)
SELECT
    name, dept, salary,
    LAST_VALUE(salary) OVER (
        PARTITION BY dept
        ORDER BY salary DESC
    ) AS last_val_wrong   -- always equals salary column!
FROM employees;

-- ✅ Correct — extend frame to see the actual last row
SELECT
    name,
    dept,
    salary,
    LAST_VALUE(salary) OVER (
        PARTITION BY dept
        ORDER BY salary DESC
        ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
    ) AS dept_min_salary,
    LAST_VALUE(name) OVER (
        PARTITION BY dept
        ORDER BY salary DESC
        ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
    ) AS lowest_earner
FROM employees
ORDER BY dept, salary DESC;
```

## 11.1 FIRST_VALUE + LAST_VALUE Together

```sql
-- Show min, max, and distance from both extremes — using named window
SELECT
    name,
    dept,
    salary,
    FIRST_VALUE(salary) OVER w AS dept_max,
    LAST_VALUE(salary)  OVER w AS dept_min,
    salary - LAST_VALUE(salary)  OVER w AS above_min,
    FIRST_VALUE(salary) OVER w - salary AS below_max
FROM employees
WINDOW w AS (
    PARTITION BY dept
    ORDER BY salary DESC
    ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
)
ORDER BY dept, salary DESC;
```

> **⚡ Tip:** Use the `WINDOW` clause to define a named window once and reuse it across multiple functions — cleaner and less error-prone.

---

# Chapter 12: NTH_VALUE()

`NTH_VALUE()` returns the value from the **Nth row** in the window frame. Useful for grabbing the 2nd highest, 3rd lowest, runner-up, etc.

```sql
-- Syntax
NTH_VALUE(expression, n)
OVER (
    [PARTITION BY ...]
    ORDER BY ...
    [frame_clause]    -- REQUIRED: extend frame for correct results
)
-- n: which row number to fetch (1-based index)
```

```
  NTH_VALUE(salary, 2) in the Finance department (ORDER BY salary DESC)

  Frame rows (Finance, sorted by salary DESC):
  Row 1: Oscar  95000  <- NTH_VALUE(salary, 1) = 95000
  Row 2: Frank  91000  <- NTH_VALUE(salary, 2) = 91000  <- 2nd highest
  Row 3: Grace  88000  <- NTH_VALUE(salary, 3) = 88000  <- 3rd highest
  Row 4: Heidi  76000

  Every row in Finance will return 91000 as NTH_VALUE(salary, 2)
```

```sql
-- Gold, Silver, Bronze salary per department
SELECT
    name,
    dept,
    salary,
    NTH_VALUE(salary, 1) OVER w AS gold_salary,
    NTH_VALUE(salary, 2) OVER w AS silver_salary,
    NTH_VALUE(salary, 3) OVER w AS bronze_salary
FROM employees
WINDOW w AS (
    PARTITION BY dept
    ORDER BY salary DESC
    ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
)
ORDER BY dept, salary DESC;

-- Result (Finance dept):
-- name    salary  gold   silver  bronze
-- Oscar   95000   95000  91000   88000
-- Frank   91000   95000  91000   88000
-- Grace   88000   95000  91000   88000
-- Heidi   76000   95000  91000   88000
```

## 12.1 Practical Use: Podium Sales Report

```sql
-- 1st, 2nd, and 3rd place sales per department on every row
SELECT DISTINCT
    dept,
    NTH_VALUE(name,  1) OVER w AS first_place,
    NTH_VALUE(sales, 1) OVER w AS first_sales,
    NTH_VALUE(name,  2) OVER w AS second_place,
    NTH_VALUE(sales, 2) OVER w AS second_sales,
    NTH_VALUE(name,  3) OVER w AS third_place,
    NTH_VALUE(sales, 3) OVER w AS third_sales
FROM employees
WINDOW w AS (
    PARTITION BY dept
    ORDER BY sales DESC
    ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
)
ORDER BY dept;
```

---

# Chapter 13: The WINDOW Clause & Frame Specification

## 13.1 Named Windows with WINDOW Clause

Reuse the same window definition across multiple functions:

```sql
-- Without WINDOW clause (repetitive and error-prone)
SELECT
    name,
    ROW_NUMBER()  OVER (PARTITION BY dept ORDER BY salary DESC),
    RANK()        OVER (PARTITION BY dept ORDER BY salary DESC),
    DENSE_RANK()  OVER (PARTITION BY dept ORDER BY salary DESC)
FROM employees;

-- With WINDOW clause (clean and DRY)
SELECT
    name,
    ROW_NUMBER()  OVER dept_window,
    RANK()        OVER dept_window,
    DENSE_RANK()  OVER dept_window
FROM employees
WINDOW dept_window AS (PARTITION BY dept ORDER BY salary DESC);
```

## 13.2 Frame Clause Reference

```
  FRAME MODES:
  +------------------------------------------------------------------+
  |  ROWS   -> counts physical rows (exact offsets)                  |
  |  RANGE  -> counts rows with matching values (handles ties)       |
  |  GROUPS -> counts distinct peer groups (PostgreSQL 11+)          |
  +------------------------------------------------------------------+

  FRAME BOUNDARIES:
  +----------------------------------+-------------------------------+
  | UNBOUNDED PRECEDING              | start of partition            |
  | n PRECEDING                      | n rows before current row    |
  | CURRENT ROW                      | the current row itself        |
  | n FOLLOWING                      | n rows after current row     |
  | UNBOUNDED FOLLOWING              | end of partition              |
  +----------------------------------+-------------------------------+

  COMMON PATTERNS:
  ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW        <- default
  ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING  <- full partition
  ROWS BETWEEN 2 PRECEDING AND CURRENT ROW               <- rolling 3-row window
  ROWS BETWEEN 1 PRECEDING AND 1 FOLLOWING               <- centered 3-row window
```

```sql
-- Rolling 3-row average salary (2 preceding + current)
SELECT
    name,
    hire_date,
    salary,
    ROUND(AVG(salary) OVER (
        ORDER BY hire_date
        ROWS BETWEEN 2 PRECEDING AND CURRENT ROW
    ), 2) AS rolling_3_avg
FROM employees
ORDER BY hire_date;

-- Running cumulative total of salaries
SELECT
    name,
    dept,
    salary,
    SUM(salary) OVER (
        PARTITION BY dept
        ORDER BY hire_date
        ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
    ) AS running_total
FROM employees
ORDER BY dept, hire_date;
```

---

# Chapter 14: Combining Multiple Window Functions

Real-world queries often use several window functions together:

```sql
-- Comprehensive employee analytics in one query
SELECT
    name,
    dept,
    salary,
    sales,

    -- Ranking
    ROW_NUMBER()  OVER dept_sal                            AS row_num,
    RANK()        OVER dept_sal                            AS sal_rank,
    DENSE_RANK()  OVER dept_sal                            AS dense_rnk,

    -- Distribution
    NTILE(4)      OVER dept_sal                            AS quartile,
    ROUND(PERCENT_RANK() OVER dept_sal * 100, 1)           AS pct_rank,
    ROUND(CUME_DIST()    OVER dept_sal * 100, 1)           AS cume_dist,

    -- Extremes
    FIRST_VALUE(salary) OVER dept_sal_full                 AS dept_max,
    LAST_VALUE(salary)  OVER dept_sal_full                 AS dept_min,

    -- Neighbors
    LAG(salary)   OVER dept_sal                            AS prev_sal,
    LEAD(salary)  OVER dept_sal                            AS next_sal

FROM employees
WINDOW
    dept_sal      AS (PARTITION BY dept ORDER BY salary DESC),
    dept_sal_full AS (PARTITION BY dept ORDER BY salary DESC
                      ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING)
ORDER BY dept, salary DESC;
```

## 14.1 Practical Use: Full Sales Leaderboard

```sql
-- Complete leaderboard with global and department rankings
SELECT
    name,
    dept,
    sales,
    RANK()   OVER (ORDER BY sales DESC)                        AS global_rank,
    RANK()   OVER (PARTITION BY dept ORDER BY sales DESC)      AS dept_rank,
    ROUND(CUME_DIST() OVER (ORDER BY sales DESC) * 100, 1)    AS top_pct,
    LAG(sales) OVER (ORDER BY sales DESC)                      AS next_lower_sales,
    sales - LAG(sales) OVER (ORDER BY sales DESC)              AS gap_to_above,
    NTILE(3) OVER (ORDER BY sales DESC)                        AS tier
FROM employees
ORDER BY sales DESC;
```

## 14.2 Practical Use: Running Total with Rolling Average

```sql
-- Running payroll total + rolling 3-hire average per department
SELECT
    name,
    dept,
    hire_date,
    salary,
    SUM(salary) OVER (
        PARTITION BY dept
        ORDER BY hire_date
        ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
    ) AS running_dept_payroll,
    ROUND(AVG(salary) OVER (
        PARTITION BY dept
        ORDER BY hire_date
        ROWS BETWEEN 2 PRECEDING AND CURRENT ROW
    ), 2) AS rolling_3_avg
FROM employees
ORDER BY dept, hire_date;
```

---

# Chapter 15: Best Practices & Quick Reference

## 15.1 Which Function to Choose?

```
  WHAT DO YOU NEED?                            FUNCTION TO USE
  ───────────────────────────────────────────────────────────────
  Unique sequential number per row?        ->  ROW_NUMBER()
  Rank with gaps on ties?                  ->  RANK()
  Rank without gaps on ties?               ->  DENSE_RANK()
  Split rows into N equal buckets?         ->  NTILE(n)
  Relative position as 0.0-1.0?           ->  PERCENT_RANK()
  Cumulative % of rows at or below?        ->  CUME_DIST()
  Compare to a previous row's value?       ->  LAG()
  Compare to a next row's value?           ->  LEAD()
  Compare to best value in the group?      ->  FIRST_VALUE()
  Compare to worst value in the group?     ->  LAST_VALUE()
                                               + ROWS UNBOUNDED...FOLLOWING
  Get the Nth ranked value?                ->  NTH_VALUE(col, n)
                                               + ROWS UNBOUNDED...FOLLOWING
```

## 15.2 Golden Rules

1. **Always ORDER BY inside OVER()** for ranking and value functions — without it, results are non-deterministic.
2. **Use PARTITION BY** to apply functions within logical groups (departments, regions, customers).
3. **Fix the frame for LAST_VALUE and NTH_VALUE** — always add `ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING` or you'll get the current row's value back, not the true last/Nth.
4. **Use the WINDOW clause** when reusing the same window definition — DRY, readable, and less error-prone.
5. **Wrap in a subquery to filter** — you cannot use window functions directly in WHERE or HAVING; wrap the query in a CTE or subquery first.
6. **RANK() vs DENSE_RANK()** — use RANK for competition-style (gaps OK), DENSE_RANK for medal-style (no gaps ever).
7. **ROW_NUMBER() for deduplication** — the go-to pattern for keeping only the first occurrence per group.
8. **LAG/LEAD with a default value** — always supply a default to avoid NULL propagating into calculations: `LAG(col, 1, 0)`.

---

## 15.3 Quick Reference Cheat Sheet

| Task | SQL |
|---|---|
| Unique row number | `ROW_NUMBER() OVER (ORDER BY col)` |
| Rank with gaps | `RANK() OVER (ORDER BY col DESC)` |
| Rank without gaps | `DENSE_RANK() OVER (ORDER BY col DESC)` |
| Rank within group | `RANK() OVER (PARTITION BY dept ORDER BY salary DESC)` |
| Quartile bucket | `NTILE(4) OVER (ORDER BY col DESC)` |
| Percentile position | `PERCENT_RANK() OVER (ORDER BY col)` |
| Cumulative distribution | `CUME_DIST() OVER (ORDER BY col)` |
| Previous row value | `LAG(col) OVER (ORDER BY col)` |
| Previous row, 2 back, default 0 | `LAG(col, 2, 0) OVER (ORDER BY col)` |
| Next row value | `LEAD(col) OVER (ORDER BY col)` |
| Group maximum | `FIRST_VALUE(col) OVER (PARTITION BY dept ORDER BY col DESC)` |
| Group minimum | `LAST_VALUE(col) OVER (... ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING)` |
| 2nd highest value | `NTH_VALUE(col, 2) OVER (... ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING)` |
| Running total | `SUM(col) OVER (ORDER BY date ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW)` |
| Rolling 3-row average | `AVG(col) OVER (ORDER BY date ROWS BETWEEN 2 PRECEDING AND CURRENT ROW)` |
| Top N per group | subquery with `ROW_NUMBER() OVER (PARTITION BY dept ORDER BY salary DESC) <= N` |
| Reusable window | `WINDOW w AS (PARTITION BY dept ORDER BY salary DESC)` |

---

## 15.4 Common Mistakes to Avoid

| Mistake | Problem | Fix |
|---|---|---|
| `LAST_VALUE()` without extended frame | Returns the current row's own value | Add `ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING` |
| `NTH_VALUE()` without extended frame | Returns NULL for rows before position N | Same fix as above |
| Filtering on window function in WHERE | SQL error — not allowed | Wrap in a subquery or CTE, then apply WHERE |
| No ORDER BY in OVER() | Non-deterministic results | Always provide ORDER BY for ranking/value functions |
| RANK() when you want no gaps | Confusing gaps in rank numbers | Use `DENSE_RANK()` instead |
| Forgetting PARTITION BY | One global window instead of per-group | Add `PARTITION BY` for group-level analysis |
| Repeating same OVER() clause 5 times | Verbose, hard to maintain, typo-prone | Use the `WINDOW` clause to name it once |
| LAG/LEAD returning NULL on edge rows | NULL breaks arithmetic calculations | Provide a default: `LAG(col, 1, 0)` |

---

*PostgreSQL Window Functions Mastery Guide*
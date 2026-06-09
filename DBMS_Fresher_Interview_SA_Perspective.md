# 🗄️ DBMS — Fresher Interview Question Bank
### From the Lens of a Solution Architect
> **Target:** 0–1 Year Experience / Freshers  
> **Goal:** Test conceptual clarity, reasoning, and real-world thinking — not just textbook definitions  
> **Format:** Question → Simple Answer → Follow-up the architect would ask next

---

## A Word Before You Read

A solution architect doesn't expect freshers to have production experience.  
What they're testing is:

- **Do you understand WHY**, not just WHAT?
- **Can you reason through a problem** you haven't seen before?
- **Do you know what you don't know** — and say so honestly?

The worst answer is a confident wrong one.  
The best answer often includes: *"I think it works this way because..."*

---

## Table of Contents

| # | Topic |
|---|-------|
| Q1 | SQL Injection |
| Q2 | What is a Database and why not just use Excel? |
| Q3 | Primary Key vs Unique Key |
| Q4 | What is a Foreign Key — real scenario |
| Q5 | What is Normalisation — explain like I'm 10 |
| Q6 | What is a Transaction — the bank transfer problem |
| Q7 | ACID — what breaks if each property is missing |
| Q8 | What is an Index and what does it cost? |
| Q9 | JOIN types — explained simply |
| Q10 | What is a NULL and why is it dangerous? |
| Q11 | DELETE vs TRUNCATE vs DROP |
| Q12 | What is a View and why use it? |
| Q13 | Stored Procedure vs Function |
| Q14 | What is a Trigger — and when does it become a problem? |
| Q15 | What is Denormalisation and when is it okay? |
| Q16 | GROUP BY vs WHERE vs HAVING |
| Q17 | What happens when two users update the same row at the same time? |
| Q18 | What is MVCC in simple terms? |
| Q19 | Clustered vs Non-Clustered Index |
| Q20 | What is a Deadlock — how would you explain it to a non-technical person? |
| Q21 | What is WAL and why does your data survive a crash? |
| Q22 | PostgreSQL vs MySQL — how do you choose? |
| Q23 | What is a Subquery vs a JOIN? |
| Q24 | What is Pagination and why does OFFSET fail at scale? |
| Q25 | B-Tree vs B+Tree — which one and why? |
| Q26 | Materialised View vs Virtual View |
| Q27 | What is a Window Function — when GROUP BY isn't enough |
| Q28 | What is a Schema? |
| Q29 | What is Connection Pooling? |
| Q30 | How would you design a URL shortener database? |

---

## Q1. SQL Injection — What is it, how does it arise, consequences, prevention, and recovery?

> *This is almost always asked. The architect wants to know if you think about security, not just functionality.*

---

### What is SQL Injection?

SQL Injection is when an **attacker inserts malicious SQL code into an input field**, and the application blindly passes it to the database — letting the attacker control the query.

---

### How does it arise?

It arises when developers **concatenate user input directly into SQL strings** instead of using safe methods.

**The vulnerable code (what NOT to do):**

```python
# Python example — DANGEROUS
username = input("Enter username: ")
password = input("Enter password: ")

query = "SELECT * FROM users WHERE username='" + username + "' AND password='" + password + "'"
db.execute(query)
```

A normal user types: `alice` and `mypassword`

The query becomes:
```sql
SELECT * FROM users WHERE username='alice' AND password='mypassword'
```
Works fine. ✅

**Now the attacker types:**  
Username: `' OR '1'='1`  
Password: `anything`

The query becomes:
```sql
SELECT * FROM users WHERE username='' OR '1'='1' AND password='anything'
```

`'1'='1'` is always true → this returns ALL users → attacker is now logged in as the first user (often admin). 💀

---

### Consequences

| Consequence | What happens |
|-------------|--------------|
| **Data theft** | Attacker reads your entire database — customer emails, passwords, credit cards |
| **Data deletion** | `'; DROP TABLE users; --` — your users table is gone |
| **Authentication bypass** | Log in as any user without knowing the password |
| **Privilege escalation** | Gain admin access |
| **Reputation + legal damage** | GDPR fines, customer trust destroyed, news coverage |

**The classic destroy query:**
```sql
-- Attacker input: '; DROP TABLE users; --
-- Final query that executes:
SELECT * FROM users WHERE username=''; DROP TABLE users; --'
-- The -- comments out the rest → DROP TABLE executes
```

---

### How to prevent it

**Method 1 — Parameterised Queries / Prepared Statements (most important)**

The input is treated as **data only**, never as SQL code. The database separates the query structure from the user data completely.

```python
# Python with psycopg2 (PostgreSQL) — SAFE ✅
query = "SELECT * FROM users WHERE username = %s AND password = %s"
cursor.execute(query, (username, password))
# The %s are placeholders — user input CANNOT change the query structure
```

```javascript
// Node.js with pg library — SAFE ✅
const query = {
  text: 'SELECT * FROM users WHERE username = $1 AND password = $2',
  values: [username, password]
}
await client.query(query)
```

```java
// Java — SAFE ✅
PreparedStatement stmt = conn.prepareStatement(
    "SELECT * FROM users WHERE username = ? AND password = ?"
);
stmt.setString(1, username);
stmt.setString(2, password);
```

Even if the attacker types `' OR '1'='1`, it's treated as a literal string — the database looks for a user literally named `' OR '1'='1'` — finds nothing — login fails.

**Method 2 — Use an ORM**

ORMs (SQLAlchemy, Django ORM, Hibernate) use parameterised queries internally. You rarely write raw SQL.

```python
# Django ORM — safe by default
user = User.objects.filter(username=username, password=password).first()
# Generates parameterised SQL internally
```

**Method 3 — Input Validation**

Validate and sanitise inputs before they even reach the database layer.

```python
import re

def is_valid_username(username):
    # Only allow letters, numbers, underscores
    return bool(re.match(r'^[a-zA-Z0-9_]{3,30}$', username))

if not is_valid_username(username):
    raise ValueError("Invalid username format")
```

**Method 4 — Least Privilege**

The database user your application connects with should have ONLY the permissions it needs.

```sql
-- Create app user with minimal permissions
CREATE USER app_user WITH PASSWORD 'secure_password';
GRANT SELECT, INSERT, UPDATE ON users TO app_user;
-- app_user cannot DROP tables, cannot access other schemas
-- Even if injected, the damage is limited
```

**Method 5 — Web Application Firewall (WAF)**

A WAF sits in front of your app and detects/blocks SQL injection patterns in HTTP requests. It's a safety net — not a replacement for parameterised queries.

---

### How to recover if injection already happened

```
Step 1: Containment
├── Take the affected application offline immediately
├── Revoke database credentials used by the app
└── Block the attacker's IP at the firewall

Step 2: Assessment
├── Check database logs — what queries were executed?
├── Check WAL / binary logs — what data was read or modified?
└── Identify which tables were accessed

Step 3: Recovery
├── Restore from last clean backup (PITR using WAL if possible)
├── Identify and fix the vulnerable code
└── Rotate ALL database passwords and API keys

Step 4: Notification
├── Notify affected users if personal data was exposed (GDPR requirement)
└── File incident report

Step 5: Hardening
├── Audit all SQL queries in the codebase for concatenation
├── Add parameterised queries everywhere
├── Implement WAF
└── Set up query anomaly detection
```

> **Architect's follow-up:** "If you fixed the injection vulnerability but the attacker already exfiltrated the user table with hashed passwords, what should you do?"  
> **Answer:** Force-reset all user passwords immediately. Even hashed passwords can be cracked offline with rainbow tables if the hashing algorithm is weak (MD5, SHA1). Send password reset emails to all users. If you were using bcrypt/argon2 with good salts — the risk is lower but notification is still required by law in most countries.

---

## Q2. What is a Database and why not just use Excel?

> *Sounds like a joke question — it's not. Tests if the candidate understands the fundamental problems databases solve.*

**Answer:**

A database is an **organised system for storing, retrieving, and managing large amounts of data** with guarantees around consistency, access control, and concurrent use.

**Why not Excel?**

| Problem | Excel | Database |
|---------|-------|----------|
| Multiple users editing simultaneously | Corruption, lost changes | Handled via transactions and locking |
| 10 million rows | Crashes, unusable | No problem |
| Enforce "email must be unique" | Manual, error-prone | Constraints do this automatically |
| Two users book the last seat | Both might succeed (double booking) | ACID prevents this |
| "Who changed this row and when?" | No idea | Audit logs, triggers |
| Relate data across sheets | VLOOKUP hell | JOIN — clean and fast |
| Access control | Password on the file | Row-level security per user |

Databases also give you:
- **Indexing** — find one row in a billion in milliseconds
- **ACID transactions** — operations that either fully complete or don't happen at all
- **Backup and recovery** — point-in-time restore after catastrophic mistakes

> **Architect follow-up:** "When WOULD you use Excel over a database?"  
> **Answer:** One-time analysis, quick calculations, sharing data with non-technical stakeholders, small datasets with no concurrent editing. Tools for the right job.

---

## Q3. What is the difference between a Primary Key and a Unique Key?

**Answer:**

Both enforce uniqueness — no two rows can have the same value. The differences:

| | Primary Key | Unique Key |
|--|-------------|------------|
| Can it be NULL? | **No** — never | **Yes** — one NULL allowed |
| How many per table? | **Exactly one** | As many as you need |
| Main purpose | **Identifies the row** | Enforces a business rule |

```sql
CREATE TABLE users (
    user_id   SERIAL      PRIMARY KEY,  -- row identity, no NULL
    email     VARCHAR     UNIQUE,        -- business rule: no duplicate emails
    phone     VARCHAR     UNIQUE         -- business rule: no duplicate phones
);
```

**Simple way to remember:**  
Primary Key = "This IS the row."  
Unique Key = "This column's value cannot repeat, but the row is identified by something else."

> **Architect follow-up:** "Can a primary key consist of two columns?"  
> **Answer:** Yes — called a **composite primary key**. Used when no single column is unique enough on its own. Example: `(student_id, course_id)` in an enrollment table — a student can appear many times (different courses), a course can appear many times (different students), but the combination is unique.

---

## Q4. What is a Foreign Key — explain with a real scenario

**Answer:**

A foreign key is a **column in one table that points to the primary key of another table**. It creates a relationship and ensures data stays consistent — you can't reference something that doesn't exist.

**Real scenario — food delivery app:**

```sql
CREATE TABLE customers (
    customer_id SERIAL PRIMARY KEY,
    name        VARCHAR(100)
);

CREATE TABLE orders (
    order_id    SERIAL PRIMARY KEY,
    customer_id INT REFERENCES customers(customer_id), -- foreign key
    total       DECIMAL,
    status      VARCHAR(20)
);
```

Now:
- ✅ You can place an order for customer_id = 5 (if customer 5 exists)
- ❌ You CANNOT place an order for customer_id = 9999 (if no such customer)
- ❌ You CANNOT delete customer 5 if they have orders (unless you set ON DELETE CASCADE)

**Without foreign keys:**  
You could have orders pointing to customers who don't exist. When you query "who placed order #42?" — you get nothing. Your reports are wrong. Your UI crashes.

> **Architect follow-up:** "What is ON DELETE CASCADE and when is it risky?"  
> **Answer:** It automatically deletes child rows when the parent is deleted. Delete a customer → all their orders are deleted automatically. Risky when there are multiple levels of cascades — deleting one row could trigger a chain that wipes thousands of rows across 5 tables. Always test in staging before enabling in production.

---

## Q5. What is Normalisation — explain like I'm 10

**Answer:**

Normalisation means **storing each piece of information in exactly one place** so you don't repeat yourself.

**The problem without normalisation:**

```
OrderID | CustomerName | CustomerCity | Product  | Price
1       | Alice        | Mumbai       | Laptop   | 50000
2       | Alice        | Mumbai       | Mouse    | 500
3       | Alice        | Mumbai       | Keyboard | 1500
```

Alice's city "Mumbai" is stored 3 times. Now Alice moves to Pune. You have to update 3 rows. If you miss one — your data says Alice lives in both Mumbai and Pune. That's a bug.

**After normalisation — each fact stored once:**

```sql
-- Customers table: Alice's city stored ONCE
customers: (1, Alice, Pune)

-- Orders table: reference the customer, not copy their data
orders: (1, customer_id=1, Laptop, 50000)
        (2, customer_id=1, Mouse, 500)
        (3, customer_id=1, Keyboard, 1500)
```

Now update Alice's city in one place → all her orders reflect it instantly.

**The three main levels:**
- **1NF:** Each cell has one value only. No "Mumbai, Pune" in one cell.
- **2NF:** Every column depends on the WHOLE primary key, not just part of it.
- **3NF:** No column depends on another non-key column. City shouldn't depend on CustomerName — it should depend on CustomerID directly.

> **Architect follow-up:** "Is a fully normalised database always the best design?"  
> **Answer:** No. Highly normalised data requires many JOINs to answer simple questions — slowing down reads. For analytics, reporting, and read-heavy systems, some denormalisation (storing redundant data) is intentional to make queries faster. It's a trade-off: normalise for write correctness, denormalise for read performance.

---

## Q6. What is a Transaction — explain the bank transfer problem

**Answer:**

A transaction is a **group of database operations that must ALL succeed or ALL fail together**. You can't have a half-done transaction.

**The bank transfer problem:**

```
Transfer ₹1000 from Alice to Bob:
Step 1: Deduct ₹1000 from Alice  → Alice: ₹5000 → ₹4000
Step 2: Add ₹1000 to Bob        → Bob:   ₹2000 → ₹3000
```

What if the server crashes after Step 1 but before Step 2?  
Alice lost ₹1000. Bob got nothing. ₹1000 vanished.

**With a transaction:**

```sql
BEGIN;  -- start transaction

UPDATE accounts SET balance = balance - 1000 WHERE name = 'Alice';
UPDATE accounts SET balance = balance + 1000 WHERE name = 'Bob';

COMMIT;  -- only now are changes permanent
-- If ANYTHING fails between BEGIN and COMMIT → automatic ROLLBACK
-- Both changes are undone — Alice has her ₹1000 back
```

**Two keywords:**
- `COMMIT` — "Everything worked, save it permanently"
- `ROLLBACK` — "Something went wrong, undo everything back to before BEGIN"

> **Architect follow-up:** "What happens to an open transaction if the database server suddenly crashes?"  
> **Answer:** The transaction is automatically rolled back when the server restarts. The database uses WAL (Write-Ahead Log) to know which transactions were incomplete — it undoes them during recovery. The user would get an error and need to retry the operation.

---

## Q7. ACID — what breaks if each property is missing?

**Answer:**

ACID is four guarantees that make transactions reliable. The easiest way to understand each is to imagine what goes wrong without it.

**A — Atomicity ("All or Nothing")**

Without it: The bank transfer completes Step 1 (deduct) but crashes before Step 2 (credit). Money disappears. The world is inconsistent.

**C — Consistency ("Rules are always obeyed")**

Without it: A transaction that would put Alice's balance below ₹0 (overdraft, which is not allowed) would succeed. The database ends up in an invalid state that violates its own rules.

**I — Isolation ("Transactions don't interfere")**

Without it: Alice checks her balance (₹5000), starts a transfer, meanwhile Bob reads the same balance before Alice's deduction is committed — Bob sees stale data and makes a wrong decision. Or two people buy the last concert ticket simultaneously — both "succeed."

**D — Durability ("Committed = Permanent")**

Without it: You COMMIT a transaction, the database says "done," server crashes — data is gone. You'd have to re-enter every order placed in the last hour.

```
Quick memory trick:
A — Don't leave things half done
C — Don't break the rules
I — Don't let others see your work until you're done
D — Once done, it stays done
```

> **Architect follow-up:** "Which ACID property is the hardest to implement in a distributed system with multiple servers?"  
> **Answer:** Isolation and Atomicity across multiple servers. When a transaction touches data on Server A and Server B, you need distributed coordination to ensure both either commit or rollback together — this is solved by two-phase commit (2PC) but it's complex and has failure modes. This is why distributed systems often relax to "eventual consistency" instead of full ACID.

---

## Q8. What is an Index and what does it cost?

**Answer:**

An index is a **separate data structure that makes finding rows faster** — like the index at the back of a textbook.

**Without index:**
```sql
SELECT * FROM users WHERE email = 'alice@example.com';
-- Database reads EVERY row checking the email
-- 1 million users = 1 million comparisons = slow
```

**With index:**
```sql
CREATE INDEX idx_email ON users(email);
-- Database uses a tree structure to find 'alice@example.com' in ~20 steps
-- 1 million users = still ~20 steps = fast
```

**The hidden cost most freshers miss:**

Every time you INSERT, UPDATE, or DELETE a row — **the index must also be updated**.

```
Insert 1 row → update the table + update every index on that table
10 indexes on a table = 11 write operations for every 1 insert
```

So more indexes = faster reads but slower writes.

**Rule of thumb:**
- Index columns you frequently filter by (`WHERE email = ...`)
- Index columns you JOIN on
- Index columns you sort by (`ORDER BY created_at`)
- Do NOT index every column "just in case"

> **Architect follow-up:** "You have a table with 20 indexes and inserts are slow. What do you do?"  
> **Answer:** Run a query to find which indexes are actually being used. Drop the ones with zero usage — they're just dead weight slowing down writes. In PostgreSQL: `SELECT * FROM pg_stat_user_indexes WHERE idx_scan = 0;` shows unused indexes.

---

## Q9. Explain JOIN types — simply

**Answer:**

A JOIN combines rows from two tables based on a related column. Think of two lists: Customers and Orders.

```
Customers:          Orders:
id | name           id | customer_id | amount
1  | Alice          1  | 1           | 500
2  | Bob            2  | 1           | 200
3  | Carol          3  | 4           | 100  ← customer 4 doesn't exist!
                    (Bob has no orders)
```

**INNER JOIN** — only rows that match in BOTH tables

```sql
SELECT c.name, o.amount
FROM customers c INNER JOIN orders o ON c.id = o.customer_id;
-- Result: Alice (500), Alice (200)
-- Bob excluded (no orders), Carol excluded (no orders), Order 3 excluded (no customer)
```

**LEFT JOIN** — all rows from left table, matching rows from right (NULLs if no match)

```sql
SELECT c.name, o.amount
FROM customers c LEFT JOIN orders o ON c.id = o.customer_id;
-- Result: Alice(500), Alice(200), Bob(NULL), Carol(NULL)
-- All customers shown, even those with no orders
```

**RIGHT JOIN** — all rows from right table, matching rows from left (rare — just flip the tables and use LEFT JOIN)

**FULL OUTER JOIN** — all rows from both tables (NULLs where no match)

```sql
-- Result: Alice(500), Alice(200), Bob(NULL), Carol(NULL), NULL(100)
-- Everything shown — unmatched on both sides
```

**Simple visual:**

```
INNER JOIN  = only the overlap
LEFT JOIN   = everything on the left + overlap
RIGHT JOIN  = everything on the right + overlap
FULL OUTER  = everything from both sides
```

> **Architect follow-up:** "When would you use a LEFT JOIN over an INNER JOIN?"  
> **Answer:** When you want all records from the primary table even if they have no matching records in the secondary table. Example: "Show me all customers and how many orders they've placed" — a customer with 0 orders should still appear (with count = 0), not be excluded.

---

## Q10. What is NULL and why is it dangerous?

**Answer:**

NULL means **the value is unknown or missing** — it is NOT zero, it is NOT an empty string. It is the absence of any value.

**Why it's dangerous — NULL behaves unexpectedly:**

```sql
-- NULL is not equal to anything, including itself
SELECT * FROM users WHERE phone = NULL;     -- returns 0 rows (WRONG!)
SELECT * FROM users WHERE phone IS NULL;    -- correct ✅

-- NULL in arithmetic makes the whole expression NULL
SELECT 5 + NULL;   -- result: NULL (not 5, not an error — just NULL)

-- NULL in aggregation is silently ignored
SELECT AVG(salary) FROM employees;
-- Employees with NULL salary are excluded from the average
-- Your average might be artificially high
```

**Comparisons with NULL:**
```sql
-- These all return NULL (not TRUE, not FALSE)
NULL = NULL      → NULL
NULL != NULL     → NULL
NULL > 5         → NULL
NOT NULL         → NULL
```

This is called **three-valued logic**: TRUE, FALSE, and UNKNOWN (NULL).

**Safe handling:**
```sql
-- Check for NULL properly
WHERE phone IS NULL
WHERE phone IS NOT NULL

-- Replace NULL with a default value
SELECT COALESCE(phone, 'Not provided') FROM users;
-- COALESCE returns the first non-NULL value

-- NULL-safe operations
SELECT NULLIF(salary, 0) FROM employees;
-- Returns NULL if salary is 0 (prevents divide-by-zero)
```

> **Architect follow-up:** "Should you use NULL for optional fields or store an empty string?"  
> **Answer:** Use NULL for "value not provided/unknown" and empty string for "value is explicitly empty." They mean different things. `email = NULL` means we don't have an email. `email = ''` means the user provided a blank email. Mixing them creates subtle bugs in validation and comparisons.

---

## Q11. DELETE vs TRUNCATE vs DROP — what's the difference?

**Answer:**

All three remove data but in very different ways with very different consequences.

```sql
-- DELETE: removes specific rows, can be rolled back, fires triggers
DELETE FROM orders WHERE status = 'cancelled';
-- Removes only cancelled orders
-- Can add WHERE clause
-- Can be rolled back if inside a transaction
-- Slow on large tables (logs each row deletion)

-- TRUNCATE: removes ALL rows instantly, cannot fire row-level triggers
TRUNCATE TABLE temp_logs;
-- Removes everything — no WHERE clause
-- Much faster than DELETE (doesn't log each row)
-- Cannot be rolled back in MySQL (can in PostgreSQL if inside a transaction)
-- Resets auto-increment counter

-- DROP: removes the entire TABLE structure + all data
DROP TABLE orders;
-- The table no longer exists
-- Cannot SELECT from it anymore
-- Cannot be rolled back
```

**Memory trick:**

```
DELETE  = selectively erase content (like deleting files)
TRUNCATE = empty the container (like emptying a bin)
DROP    = destroy the container itself (like deleting the folder)
```

**Real consequences of using the wrong one:**

```
"I need to clear test data before a demo"
→ Wrong choice: DROP   → You deleted the table structure. App crashes.
→ Right choice: DELETE WHERE environment='test' or TRUNCATE
```

> **Architect follow-up:** "A junior developer accidentally ran `DELETE FROM orders` without a WHERE clause on production. What do you do?"  
> **Answer:** First — don't panic and don't do anything else that might overwrite the database. Check if the DELETE was inside an uncommitted transaction (if so, ROLLBACK immediately). If committed — restore from the most recent backup, or use Point-in-Time Recovery via WAL to restore to 1 minute before the DELETE. This is why regular backups and WAL archiving are non-negotiable.

---

## Q12. What is a View and why use it?

**Answer:**

A view is a **saved query that you can treat as a table**. You give a complex query a name — then anyone can use that name without writing the complex query again.

```sql
-- This query is complex and used in 10 different places:
SELECT
    u.name,
    u.email,
    COUNT(o.order_id) as total_orders,
    SUM(o.amount)     as total_spent
FROM users u
LEFT JOIN orders o ON u.id = o.user_id
WHERE u.status = 'active'
GROUP BY u.id, u.name, u.email;

-- Wrap it in a view:
CREATE VIEW active_customer_summary AS
SELECT u.name, u.email,
    COUNT(o.order_id) as total_orders,
    SUM(o.amount)     as total_spent
FROM users u
LEFT JOIN orders o ON u.id = o.user_id
WHERE u.status = 'active'
GROUP BY u.id, u.name, u.email;

-- Now anyone can just:
SELECT * FROM active_customer_summary WHERE total_spent > 10000;
```

**Three reasons to use views:**

1. **Simplicity** — hide complex joins behind a simple name
2. **Security** — give a user access to the view but not the underlying tables (they can't see columns like `password` or `salary`)
3. **Consistency** — everyone uses the same definition; if the logic changes, update the view once

**Important:** A regular view runs the query every time you access it — there's no stored data.

> **Architect follow-up:** "If a view contains sensitive columns that you want to hide, how would you handle that?"  
> **Answer:** Create the view to include only the columns you want to expose, then `GRANT SELECT ON view_name TO limited_user` and `REVOKE SELECT ON base_table FROM limited_user`. The user can query the view but cannot query the base table directly.

---

## Q13. Stored Procedure vs Function — what's the difference?

**Answer:**

Both are named, reusable blocks of SQL code saved in the database. The key differences:

| | Stored Procedure | Function |
|--|------------------|----------|
| Returns a value? | Not required | **Must** return a value |
| Used in SELECT? | No — called with CALL | **Yes** — can be used inside a query |
| Can manage transactions? | Yes (COMMIT/ROLLBACK inside) | Generally no |
| Purpose | Perform actions | Compute and return a result |

```sql
-- FUNCTION: computes and returns something
CREATE OR REPLACE FUNCTION get_total_spent(p_user_id INT)
RETURNS DECIMAL AS $$
    SELECT COALESCE(SUM(amount), 0)
    FROM orders
    WHERE user_id = p_user_id;
$$ LANGUAGE sql;

-- Can be used inside a SELECT:
SELECT name, get_total_spent(id) as total FROM users;

---

-- PROCEDURE: performs actions
CREATE OR REPLACE PROCEDURE process_refund(p_order_id INT, p_amount DECIMAL)
LANGUAGE plpgsql AS $$
BEGIN
    UPDATE orders SET status = 'refunded' WHERE order_id = p_order_id;
    INSERT INTO refunds (order_id, amount, refunded_at) VALUES (p_order_id, p_amount, NOW());
    COMMIT;
END;
$$;

-- Called with CALL:
CALL process_refund(42, 999.00);
```

> **Architect follow-up:** "Should you put all business logic in stored procedures?"  
> **Answer:** Not in modern applications. Stored procedures are hard to version control, hard to test, and tightly couple your business logic to a specific database. Most teams keep business logic in the application code and use the database only for data storage. Stored procedures are useful for complex, data-heavy operations that would be slow if done row-by-row from the application.

---

## Q14. What is a Trigger — and when does it become a problem?

**Answer:**

A trigger is code that **runs automatically when something happens to a table** — an INSERT, UPDATE, or DELETE. You don't call it; the database fires it.

```sql
-- Real example: automatically set updated_at whenever a row is updated
CREATE OR REPLACE FUNCTION set_updated_timestamp()
RETURNS TRIGGER AS $$
BEGIN
    NEW.updated_at = NOW();
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER auto_updated_at
    BEFORE UPDATE ON orders
    FOR EACH ROW
    EXECUTE FUNCTION set_updated_timestamp();

-- Now every UPDATE on orders sets updated_at automatically
-- No application code needed
UPDATE orders SET status = 'shipped' WHERE order_id = 1;
-- updated_at is set — developer doesn't have to remember to do it
```

**When triggers become a problem:**

1. **Hidden behaviour** — A developer updates a row, triggering 3 other updates they didn't know about. Debugging takes hours because the changes are invisible unless you know the trigger exists.

2. **Performance on bulk operations** — If a trigger fires FOR EACH ROW and you update 1 million rows, the trigger fires 1 million times. What was a 2-second bulk update becomes a 20-minute operation.

3. **Trigger chains** — Trigger A fires → updates Table B → fires Trigger B → updates Table C → fires Trigger C → infinite loop or unexpected cascade.

> **Architect follow-up:** "You're debugging a bug where a row's value keeps changing mysteriously. Where do you look?"  
> **Answer:** Check if there are triggers on the table — `SELECT * FROM information_schema.triggers WHERE event_object_table = 'tablename'`. Triggers are the most common cause of "someone/something is changing my data and I don't know who."

---

## Q15. What is Denormalisation and when is it okay?

**Answer:**

Denormalisation means **intentionally storing duplicate/redundant data** to make reads faster — the opposite of normalisation.

**Example:**  
Normalised: To show an order's details, you JOIN `orders`, `customers`, and `products` — 3 tables.  
Denormalised: You store `customer_name` and `product_name` directly in the `orders` row — no JOIN needed.

```sql
-- Normalised (correct, but requires JOIN)
SELECT o.id, c.name, p.name, o.amount
FROM orders o
JOIN customers c ON o.customer_id = c.id
JOIN products p ON o.product_id = p.id;

-- Denormalised (faster read, but customer_name is duplicated everywhere)
SELECT id, customer_name, product_name, amount FROM orders;
```

**When is it acceptable:**

- **Analytics dashboards** — read millions of rows fast; the data doesn't change
- **Data warehouses** — designed for reporting, not real-time updates
- **Caching layers** — store a pre-joined object in Redis for fast reads
- **Audit logs** — you want a snapshot of data AT THAT MOMENT (not a reference)

**The cost:**  
If Alice changes her name, you must update it in the `customers` table AND in every order. Miss one → inconsistent data.

> **Architect follow-up:** "An order invoice shows the customer's current name, not the name at time of purchase. Is this a bug?"  
> **Answer:** Yes — for invoices and financial records, you want the name as it was WHEN the order was placed (snapshot). This is intentional denormalisation: store `customer_name_at_purchase` in the orders table. For general display purposes (showing a user's order history), the current name is fine.

---

## Q16. GROUP BY vs WHERE vs HAVING — what's the difference?

**Answer:**

All three filter data but at different stages of a query.

```
Query execution order:
FROM → WHERE → GROUP BY → HAVING → SELECT → ORDER BY
```

**WHERE** — filters BEFORE grouping (filters individual rows)

```sql
-- Get total orders for active users only
SELECT user_id, COUNT(*) as order_count
FROM orders
WHERE status = 'active'   -- filter rows BEFORE grouping
GROUP BY user_id;
```

**GROUP BY** — groups rows together so you can aggregate them

```sql
-- Count orders per user
SELECT user_id, COUNT(*) as order_count
FROM orders
GROUP BY user_id;   -- collapse all rows with same user_id into one group
```

**HAVING** — filters AFTER grouping (filters the groups themselves)

```sql
-- Show only users with MORE than 5 orders
SELECT user_id, COUNT(*) as order_count
FROM orders
GROUP BY user_id
HAVING COUNT(*) > 5;   -- filter groups AFTER they're formed
-- Cannot use WHERE here because COUNT(*) doesn't exist yet before GROUP BY
```

**The key rule:**  
- Use `WHERE` to filter rows (individual values)
- Use `HAVING` to filter groups (aggregated values like COUNT, SUM, AVG)

```sql
-- Full example: active users who spent more than ₹10,000
SELECT user_id, SUM(amount) as total
FROM orders
WHERE status != 'cancelled'   -- WHERE: exclude cancelled orders
GROUP BY user_id
HAVING SUM(amount) > 10000;   -- HAVING: only high spenders
```

> **Architect follow-up:** "Can you use WHERE with an aggregate function like COUNT?"  
> **Answer:** No — `WHERE COUNT(*) > 5` gives an error because WHERE runs before GROUP BY, and at that point the COUNT hasn't been computed yet. That's exactly why HAVING exists — it runs after GROUP BY when aggregates are available.

---

## Q17. What happens when two users update the same row at the same time?

**Answer:**

This is a **concurrency problem** — two transactions reading and modifying the same data simultaneously can produce incorrect results.

**Classic example — ticket booking:**

```
Ticket table: ticket_id=1, seats_available=1

User A reads: seats_available = 1  → "1 seat available, proceed to book"
User B reads: seats_available = 1  → "1 seat available, proceed to book"

User A updates: seats_available = seats_available - 1  → sets to 0
User B updates: seats_available = seats_available - 1  → sets to 0 (from their stale read)

Result: 2 bookings confirmed. 0 seats available. You oversold.
```

**Solution 1 — Pessimistic Locking (`SELECT FOR UPDATE`)**

Lock the row before reading so no one else can touch it until you're done:

```sql
BEGIN;
SELECT seats_available FROM tickets WHERE id = 1 FOR UPDATE;
-- Row is now LOCKED. User B tries the same → WAITS

-- Check and update
UPDATE tickets SET seats_available = seats_available - 1 WHERE id = 1;
COMMIT;
-- Lock is released. User B's query now runs — reads updated value (0)
-- User B sees 0 seats → booking fails correctly
```

**Solution 2 — Optimistic Locking (no lock, check at write time)**

```sql
-- Add a version column to the table
-- User A reads: seats=1, version=5
-- User B reads: seats=1, version=5

-- User A updates:
UPDATE tickets
SET seats_available = 0, version = 6
WHERE id = 1 AND version = 5;  -- only update if version hasn't changed
-- 1 row updated ✅

-- User B tries:
UPDATE tickets
SET seats_available = 0, version = 6
WHERE id = 1 AND version = 5;  -- version is now 6, not 5!
-- 0 rows updated → application detects this → shows "sold out"
```

> **Architect follow-up:** "When would you choose optimistic over pessimistic locking?"  
> **Answer:** Optimistic locking when conflicts are RARE (most requests will succeed without collision — no time wasted on locks). Pessimistic when conflicts are FREQUENT or consequences are severe (banking, inventory). Optimistic is better for performance; pessimistic is safer for critical operations.

---

## Q18. What is MVCC in simple terms?

**Answer:**

MVCC stands for **Multi-Version Concurrency Control**. It's how databases let reads and writes happen at the same time without blocking each other.

**The core idea in one sentence:**  
Instead of locking a row when you update it, keep the old version around until everyone who was reading it is done.

**Simple analogy:**  
Imagine a whiteboard. Without MVCC: if someone is writing on it, you can't read it. With MVCC: they're writing on a new sheet — you can still read the old sheet undisturbed. Once everyone reading the old sheet is done, it gets erased.

**How it works:**

```sql
-- Time 0: orders row: status='pending', amount=500

-- Transaction A starts reading:
BEGIN;  -- A starts at time T1

-- Transaction B updates the row:
UPDATE orders SET status='shipped' WHERE id=1;
COMMIT;  -- B commits at time T2

-- Transaction A reads again:
SELECT status FROM orders WHERE id=1;
-- A STILL SEES 'pending' — it's reading the version from T1
-- The database kept the old version for A to see
COMMIT;  -- A finishes, old version can now be cleaned up
```

**The benefit:**  
Readers never block writers. Writers never block readers. This is why PostgreSQL handles high concurrent loads well — your SELECT queries don't queue up waiting for INSERT/UPDATE to finish.

**The cost:**  
Old versions accumulate and need to be cleaned up — PostgreSQL's VACUUM process does this.

> **Architect follow-up:** "What is a 'dirty read' and does MVCC prevent it?"  
> **Answer:** A dirty read is when you read data that another transaction has changed but not yet committed — data that might be rolled back. Yes, MVCC prevents dirty reads — you only ever see committed versions of data.

---

## Q19. Clustered vs Non-Clustered Index — the simple version

**Answer:**

The difference is about **where the actual row data lives**.

**Clustered Index:**  
The table rows are **physically stored on disk in the same order as the index**. The index IS the table data. There can be only one per table — because data can only be physically sorted one way.

```sql
-- In MySQL InnoDB, the Primary Key is automatically the clustered index
CREATE TABLE orders (
    order_id INT PRIMARY KEY,  -- rows on disk are sorted by order_id
    user_id INT,
    amount DECIMAL
);
-- Looking up order_id = 500: follow B+Tree → arrive at leaf → data IS there
```

**Non-Clustered Index:**  
A separate structure that stores the index key + a **pointer back to the actual row**. Two lookups: one in the index, one back to the table.

```sql
-- Index on user_id: (user_id → pointer to row's physical location)
CREATE INDEX idx_user ON orders(user_id);
-- Looking up user_id = 42:
-- Step 1: search index → find user_id=42 → get pointer
-- Step 2: follow pointer to actual row in table → get data
```

**Simple analogy:**  
- Clustered index = dictionary (words ARE sorted physically; the definition is right there)
- Non-clustered index = book index (separate pages at the back; tells you page number, then you flip to that page)

**Practical implication:**  
Range queries (`WHERE order_id BETWEEN 100 AND 200`) are very fast on clustered indexes — the rows are physically next to each other on disk. With a non-clustered index, the matching rows could be scattered across the disk — many seeks required.

> **Architect follow-up:** "Why is it a bad idea to use a UUID as a Primary Key in MySQL InnoDB?"  
> **Answer:** UUID is random. InnoDB uses the PK as the clustered index, so rows must be stored in PK order. Inserting a random UUID forces a **page split** — the database must physically rearrange existing pages to insert the new row in the right sorted position. Auto-increment integers always append to the end — no rearranging needed, much faster inserts.

---

## Q20. What is a Deadlock — explain to a non-technical person

**Answer:**

A deadlock is when **two processes are each waiting for the other to finish — and neither ever will**.

**Real-world analogy:**  
Two people, one narrow one-way bridge. Person A is at one end, Person B at the other. A says "I'll cross when B moves." B says "I'll move when A crosses." Neither moves. Forever.

**In databases:**

```
Transaction A: "I've locked Row 1, now I need Row 2"
Transaction B: "I've locked Row 2, now I need Row 1"

A waits for B to release Row 2.
B waits for A to release Row 1.
Both wait forever. Deadlock.
```

**What the database does:**  
It detects this circular wait and picks one transaction as the "victim" — kills it (rolls it back). The other transaction gets what it needs and continues.

```
ERROR: deadlock detected
-- Transaction B was rolled back
-- Transaction A proceeds
-- Transaction B's application must retry
```

**How to prevent it:**

Always access tables/rows in the **same order** across all transactions.

```sql
-- Transaction A: always lock user first, then order
-- Transaction B: always lock user first, then order
-- Now there's no circle — no deadlock possible
```

> **Architect follow-up:** "Your application logs show frequent deadlock errors. What is your first step to investigate?"  
> **Answer:** Enable deadlock logging in the database and capture which queries are involved. In PostgreSQL: `log_lock_waits = on` in postgresql.conf. Look at the two queries involved — they're almost certainly accessing the same tables in different orders. Standardise the order of table access across your application.

---

## Q21. What is WAL and why does your data survive a crash?

**Answer:**

WAL stands for **Write-Ahead Log**. It means: **write to a log file FIRST, write to the actual data file SECOND**.

**Why?**  
Writing to the actual data files requires random I/O (slow). WAL is sequential writes to a single log file (very fast). So the database can confirm your COMMIT quickly (after the log write) and update actual data pages later in the background.

**The crash protection:**

```
Normal flow:
1. You COMMIT a transaction
2. Changes written to WAL log file (fast sequential write)
3. Database says "COMMIT successful" ← you get this response
4. Later: actual data pages updated from WAL

Crash happens after step 2, before step 4:
1. Server restarts
2. Database reads WAL
3. "Redo" everything that was in WAL but not yet in data files
4. Database is fully consistent again ← your data is safe
```

**Simple analogy:**  
A restaurant waiter writes your order in their notepad (WAL) before shouting it to the kitchen. If the kitchen burns down (server crash), the notepad still has your order — it can be recreated.

**Other things WAL enables:**
- **Replication** — stream the WAL log to a replica server — it stays in sync by replaying the same changes
- **Point-in-time recovery** — replay WAL from a backup to restore to any specific moment

> **Architect follow-up:** "What is a checkpoint in relation to WAL?"  
> **Answer:** A checkpoint is when the database flushes all pending changes from memory to the actual data files on disk. After a checkpoint, older WAL records are no longer needed for recovery (the data is already safely on disk). This keeps WAL files from growing forever and makes crash recovery faster — you only need to replay WAL from the last checkpoint, not from the beginning.

---

## Q22. PostgreSQL vs MySQL — how do you choose?

**Answer:**

Both are excellent. The decision depends on your use case.

**Choose PostgreSQL when:**
- You need complex queries, analytics, or reporting
- Your data has varied structures (use JSONB for flexible fields)
- You need strong ACID compliance (especially for financial apps)
- You need geospatial features (PostGIS extension)
- You want the most standards-compliant SQL

**Choose MySQL when:**
- You're building a straightforward web app (it's battle-tested for this)
- Your team already knows MySQL / you're using PHP/WordPress ecosystem
- You need a large community + many managed hosting options
- Simple CRUD operations — MySQL's simplicity is a feature

**For a fresher joining a team:**  
You probably don't get to choose. Learn whichever one the team uses. The concepts are 90% the same — switching later takes days, not months.

**Quick decision:**

```
Analytics / complex queries / JSON / geospatial → PostgreSQL
Simple web app / WordPress / PHP / existing MySQL codebase → MySQL
Either works → PostgreSQL (it has more features for the future)
```

> **Architect follow-up:** "We're building a food delivery app from scratch. Which database would you recommend and why?"  
> **Answer:** PostgreSQL. The reasons: location data (PostGIS for geospatial queries like "restaurants near me"), complex order state management with ACID, JSONB for flexible menu structures per restaurant, and strong support for analytical queries for dashboards. PostgreSQL handles everything this app needs natively.

---

## Q23. What is the difference between a Subquery and a JOIN?

**Answer:**

Both combine data from multiple tables. The difference is in HOW they do it and WHEN to use each.

**Subquery** — a query inside another query

```sql
-- Find users who have placed at least one order
SELECT name FROM users
WHERE id IN (
    SELECT DISTINCT user_id FROM orders  -- subquery runs first
);
```

**JOIN** — merges two tables based on a condition

```sql
-- Same result with JOIN
SELECT DISTINCT u.name
FROM users u
JOIN orders o ON u.id = o.user_id;
```

**When to use which:**

```sql
-- Subquery is cleaner when you need a single value from another table
SELECT name, salary
FROM employees
WHERE salary > (SELECT AVG(salary) FROM employees);
-- The AVG is a single number — subquery is natural here

-- JOIN is better when you need columns from both tables
SELECT u.name, o.amount, o.status
FROM users u JOIN orders o ON u.id = o.user_id;
-- You need columns from both tables → JOIN
```

**Performance:**  
Generally, JOINs are optimised better by the database query planner. Subqueries in `IN (...)` with large result sets can be slow. Modern databases are smart enough to convert many subqueries to JOINs internally, but writing a JOIN explicitly is clearer and often faster.

> **Architect follow-up:** "What is a correlated subquery and why is it slow?"  
> **Answer:** A correlated subquery references the outer query — it runs once FOR EACH ROW of the outer query. If the outer query returns 100,000 rows, the subquery runs 100,000 times. Example: `SELECT name FROM users u WHERE salary > (SELECT AVG(salary) FROM department d WHERE d.id = u.dept_id)` — the inner SELECT runs once per user. Usually replaceable with a JOIN + GROUP BY for much better performance.

---

## Q24. What is Pagination and why does OFFSET fail at scale?

**Answer:**

Pagination means showing data in pages (e.g., 20 results at a time) instead of all at once.

**Method 1 — OFFSET/LIMIT (simple):**

```sql
-- Page 1
SELECT * FROM products ORDER BY id LIMIT 20 OFFSET 0;
-- Page 2
SELECT * FROM products ORDER BY id LIMIT 20 OFFSET 20;
-- Page 100
SELECT * FROM products ORDER BY id LIMIT 20 OFFSET 1980;
```

**Why OFFSET fails at scale:**

OFFSET doesn't SKIP rows efficiently — it reads all rows up to the offset and throws them away.

```
Page 100 → OFFSET 1980
Database reads rows 1 to 2000, discards 1980, returns 20
= 100x more work than necessary

Page 1000 → OFFSET 19980
Database reads 20,000 rows, returns 20
= 1000x more work
```

On a table with 10 million rows, page 5000 = reading 100,000 rows to show you 20. This is why infinite scroll on social media gets slower as you scroll deeper.

**Method 2 — Keyset/Cursor Pagination (scalable):**

```sql
-- Instead of "skip N rows", say "give me rows AFTER this value"
-- Page 1:
SELECT * FROM products ORDER BY id LIMIT 20;
-- Last id returned: 20

-- Page 2: start after id 20
SELECT * FROM products WHERE id > 20 ORDER BY id LIMIT 20;
-- Last id returned: 40

-- Page 100: start after id 1980
SELECT * FROM products WHERE id > 1980 ORDER BY id LIMIT 20;
-- Database jumps directly to id 1980 using the index → instant
```

The database uses the index on `id` to jump directly to the right position — no scanning and discarding.

> **Architect follow-up:** "What is the limitation of keyset pagination that OFFSET doesn't have?"  
> **Answer:** You can't jump to a specific page (e.g., "go to page 50"). Keyset pagination only works sequentially — you need the cursor value from the previous page to get the next one. OFFSET can jump to any page. So for "go to page N" features, OFFSET is needed. For infinite scroll or API-based pagination, keyset is far better.

---

## Q25. B-Tree vs B+Tree — which one does your database use and why?

**Answer:**

Both are tree data structures for indexes. Databases almost always use B+Tree.

**B-Tree:** Data stored at ALL levels (root, internal nodes, and leaf nodes)

**B+Tree:** Data stored ONLY at leaf nodes. Internal nodes only store navigation keys. All leaf nodes are connected in a linked list.

```
B-Tree:              B+Tree:
     [30]                  [30]         ← navigation only
    /    \                /    \
[10,20] [40,50]      [10,20] [40,50]   ← navigation only
  ↑data      ↑data    ↓      ↓      ↓
  at every       [8]→[15]→[25]→[35]→[60]  ← ALL data here + linked list
  level
```

**Why B+Tree wins for databases:**

**Range queries are fast:**
```sql
SELECT * FROM orders WHERE amount BETWEEN 100 AND 500;
-- B+Tree: find 100 in the leaf, then follow the linked list to 500
-- B-Tree: must traverse up and down the tree repeatedly → much slower
```

**More efficient disk reads:**  
Internal nodes have no data — just small keys. More keys fit per disk page → tree is shallower → fewer disk reads per lookup.

**Consistent performance:**  
Every query always goes to a leaf node (same depth). B-Tree can return early from any level — inconsistent timing.

> **Architect follow-up:** "Why can B+Tree leaf nodes be stored and read efficiently from disk?"  
> **Answer:** Because they form a doubly linked list, adjacent values are physically adjacent on disk. A range scan reads them sequentially — hard drives and SSDs both handle sequential reads much faster than random reads (even SSDs benefit from sequential access patterns in terms of read-ahead caching).

---

## Q26. Materialised View vs Virtual View

**Answer:**

A regular (virtual) view is a **saved query** — no data stored. Every time you query it, the underlying SQL runs.

A materialised view is a **saved result** — actual data stored on disk. Querying it reads pre-computed rows directly.

```sql
-- Virtual view: query runs every time
CREATE VIEW monthly_sales AS
SELECT DATE_TRUNC('month', created_at), SUM(amount)
FROM orders GROUP BY 1;

SELECT * FROM monthly_sales;  -- runs the full aggregation every time
-- If orders has 50M rows: slow every time

---

-- Materialised view: stores the result
CREATE MATERIALIZED VIEW monthly_sales_cache AS
SELECT DATE_TRUNC('month', created_at), SUM(amount)
FROM orders GROUP BY 1;

SELECT * FROM monthly_sales_cache;  -- reads stored result: instant

-- BUT: data goes stale — must refresh
REFRESH MATERIALIZED VIEW monthly_sales_cache;  -- run nightly/hourly
```

**Decision guide:**

```
Data must be real-time?          → Virtual view
Query is slow + data can be stale? → Materialised view
Report that runs once a day?      → Materialised view
Live transaction lookup?          → Virtual view (or direct query)
```

> **Architect follow-up:** "A materialised view is refreshed every hour but a user complaints they see yesterday's data. What do you check?"  
> **Answer:** Check if the scheduled refresh job is actually running and succeeding. Check for errors in the refresh (if the underlying query fails, the old data stays). Check if `REFRESH MATERIALIZED VIEW CONCURRENTLY` is being used — regular refresh locks the view during refresh (users see no data during that window), while CONCURRENTLY doesn't lock but requires a unique index.

---

## Q27. Window Function — when GROUP BY isn't enough

**Answer:**

A window function calculates something across a set of rows **without collapsing those rows** into one like GROUP BY does.

**The problem with GROUP BY:**

```sql
-- You want: each order row + the user's total spend in the same result
SELECT order_id, user_id, amount, ???_as_user_total
FROM orders;

-- GROUP BY collapses rows — you can't get both individual order and user total
SELECT user_id, SUM(amount) FROM orders GROUP BY user_id;
-- This gives you one row per user — you lose the individual orders
```

**Window function solves this:**

```sql
SELECT
    order_id,
    user_id,
    amount,
    SUM(amount) OVER (PARTITION BY user_id) as user_total_spent,
    ROW_NUMBER() OVER (PARTITION BY user_id ORDER BY amount DESC) as rank_by_amount
FROM orders;

-- Result:
-- order_id | user_id | amount | user_total_spent | rank_by_amount
-- 1        | 42      | 500    | 1200             | 1
-- 2        | 42      | 400    | 1200             | 2
-- 3        | 42      | 300    | 1200             | 3
-- 4        | 7       | 200    | 200              | 1
-- All rows kept — but each has the aggregated context added
```

**Common window functions:**
- `ROW_NUMBER()` — unique sequential number per group
- `RANK()` — ranking with gaps on ties (1,2,2,4)
- `SUM/AVG/COUNT OVER` — running totals or group aggregates
- `LAG/LEAD` — access previous/next row's value

> **Architect follow-up:** "Without window functions, how would you find the top 1 order per user by amount?"  
> **Answer:** A correlated subquery or a self-join — both messy. With window functions: `SELECT * FROM (SELECT *, ROW_NUMBER() OVER (PARTITION BY user_id ORDER BY amount DESC) as rn FROM orders) t WHERE rn = 1;` — clean and readable.

---

## Q28. What is a Schema?

**Answer:**

A schema is a **namespace or container** inside a database that groups related tables, views, functions, and other objects together. Think of a database as a building and schemas as floors — each floor has its own set of rooms (tables).

```sql
-- Create a schema
CREATE SCHEMA analytics;
CREATE SCHEMA app;

-- Create tables inside schemas
CREATE TABLE app.users (...);
CREATE TABLE app.orders (...);
CREATE TABLE analytics.daily_revenue (...);

-- Query with schema prefix
SELECT * FROM app.users;
SELECT * FROM analytics.daily_revenue;
```

**Why use schemas:**

1. **Organisation** — keep application tables separate from analytics tables
2. **Access control** — give data analysts access to `analytics` schema only; no access to `app` schema with sensitive customer data
3. **Multi-tenancy** — one schema per customer, all in the same database
4. **Separation** — different services use different schemas; they can't accidentally touch each other's tables

In PostgreSQL, the default schema is called `public` — when you create a table without specifying a schema, it goes into `public`.

> **Architect follow-up:** "What is the difference between a Schema and a Database?"  
> **Answer:** A Database is the top-level container — a separate running instance of data with its own files on disk. A Schema is a logical namespace inside a database. One database can have many schemas. You generally can't JOIN across databases easily, but you can JOIN across schemas within the same database.

---

## Q29. What is Connection Pooling?

**Answer:**

Every time your application connects to the database, it's an expensive operation — TCP handshake, authentication, memory allocation on the database server.

Without connection pooling: for every incoming web request, open a new DB connection → do the work → close it. At 1000 requests/second = 1000 new connections/second. The database falls over.

Connection pooling means: **maintain a pool of pre-opened connections and reuse them**.

```
Without pooling:
Request 1 → Open connection → Query → Close connection
Request 2 → Open connection → Query → Close connection
Request 3 → Open connection → Query → Close connection
(3x connection overhead)

With pooling:
Startup: open 10 connections, keep them alive

Request 1 → Take connection from pool → Query → Return to pool
Request 2 → Take connection from pool → Query → Return to pool
Request 3 → Take connection from pool → Query → Return to pool
(0x connection overhead — connections already exist)
```

**In practice:**

```python
# Python with SQLAlchemy — connection pool built in
from sqlalchemy import create_engine

engine = create_engine(
    'postgresql://user:pass@localhost/db',
    pool_size=10,       # maintain 10 connections
    max_overflow=20,    # allow up to 20 extra connections at peak
    pool_timeout=30     # wait up to 30s for a connection if pool is full
)
```

Popular standalone poolers: **PgBouncer** (PostgreSQL), **ProxySQL** (MySQL) — sit between your app and the database.

> **Architect follow-up:** "Your application is throwing 'too many connections' errors. The database max_connections is 100 and you have 50 app servers each with a pool of 10. What's the problem?"  
> **Answer:** 50 servers × 10 connections each = 500 connections attempted. Limit is 100. Fix: add PgBouncer as a connection pooler in front of the database — your 50 app servers connect to PgBouncer, which maintains a small pool to the actual database. 500 app connections → PgBouncer → 20 actual database connections.

---

## Q30. How would you design a URL shortener database?

> *This tests data modeling ability — a core skill the architect is probing even at fresher level.*

**Answer:**

A URL shortener like bit.ly takes a long URL and produces a short code: `bit.ly/xK92p`

**What data do we need to store?**

```
- The original long URL
- The short code
- Who created it (optional)
- When it was created
- How many times it was clicked (analytics)
```

**Basic schema:**

```sql
CREATE TABLE urls (
    id           BIGSERIAL PRIMARY KEY,
    short_code   VARCHAR(10) UNIQUE NOT NULL,  -- 'xK92p'
    original_url TEXT NOT NULL,
    created_by   INT REFERENCES users(id),     -- nullable (anonymous allowed)
    created_at   TIMESTAMP DEFAULT NOW(),
    expires_at   TIMESTAMP,                    -- optional expiry
    is_active    BOOLEAN DEFAULT TRUE
);

CREATE TABLE clicks (
    id          BIGSERIAL PRIMARY KEY,
    url_id      BIGINT REFERENCES urls(id),
    clicked_at  TIMESTAMP DEFAULT NOW(),
    ip_address  INET,
    user_agent  TEXT,
    country     VARCHAR(2)                     -- derived from IP
);
```

**Indexes needed:**

```sql
-- Most common query: given short_code, find original_url (every redirect)
CREATE INDEX idx_short_code ON urls(short_code);

-- Analytics: clicks over time per URL
CREATE INDEX idx_clicks_url_time ON clicks(url_id, clicked_at);
```

**How a redirect works:**

```sql
-- User visits bit.ly/xK92p
SELECT original_url, expires_at, is_active
FROM urls
WHERE short_code = 'xK92p';

-- If found, active, not expired → redirect to original_url
-- Log the click
INSERT INTO clicks (url_id, clicked_at, ip_address) VALUES (42, NOW(), '1.2.3.4');
```

**How to generate the short code:**

Option 1: Take the auto-increment `id`, encode it in base62 (0-9, a-z, A-Z)  
`id = 1000` → base62 = `G8`  
Short, unique, no collision possible.

Option 2: Random 6-character string (check for collision before insert)

> **Architect follow-up:** "The `clicks` table grows to 1 billion rows in 2 years. How do you handle this?"  
> **Answer:** Partition the clicks table by month — each month gets its own partition. Queries for "clicks this month" only scan the current partition. Old partitions (2 years ago) can be archived to cold storage or detached. Also consider: for real-time click counts, increment a counter in Redis and sync to the DB periodically — writing to a separate table per click at scale causes write bottlenecks.

---

---

## What the Architect is Really Testing

After all 30 questions, here is what they're actually evaluating:

| What they ask | What they're really testing |
|---------------|----------------------------|
| "What is X?" | Can you explain it simply without jargon? |
| "Why does X exist?" | Do you understand the problem it solves? |
| "What happens if X fails?" | Do you think about failure modes? |
| "When would you NOT use X?" | Do you understand trade-offs? |
| "Design a simple database for Y" | Can you turn concepts into actual schema? |

**The two answers that impress most:**

1. *"I'm not sure, but I think it works this way because..."* — Honest reasoning is valued over confident bluffing.

2. *"That depends on..."* — Trade-off thinking shows maturity beyond memorised definitions.

**The one answer that ends interviews:**

*Confidently stating something incorrect and defending it when the interviewer hints it's wrong.*

---

## Quick Cheat Sheet for Last-Minute Review

| Topic | One thing to remember |
|-------|-----------------------|
| SQL Injection | Always use parameterised queries — never concatenate user input |
| Primary Key | Unique + NOT NULL + one per table |
| Foreign Key | Links tables + prevents orphan records |
| Normalisation | Store each fact in exactly ONE place |
| Transaction | All or nothing — BEGIN / COMMIT / ROLLBACK |
| ACID | All-or-nothing, valid, isolated, permanent |
| Index | Faster reads, slower writes, use on WHERE/JOIN/ORDER BY columns |
| JOIN | INNER = only matches. LEFT = all left + matches |
| NULL | Not zero, not empty. Check with IS NULL not = NULL |
| DELETE vs TRUNCATE vs DROP | Selective / empty-all / destroy-structure |
| View | Saved query. Materialised = stored result |
| MVCC | Old versions kept so readers don't block writers |
| Clustered Index | Data IS the leaf. One per table. Physically sorted. |
| Deadlock | Circular lock wait. DB kills one transaction to resolve. |
| WAL | Log first, then data files. Enables crash recovery + replication |
| Pagination | OFFSET slow at scale. Use keyset cursor instead. |
| B+Tree | Data at leaves only. Leaves linked. Perfect for range queries. |
| Window Function | Aggregation without losing individual rows |
| Connection Pool | Reuse connections. Don't open/close per request. |

---

*Prepared from the lens of a Solution Architect interviewing fresher candidates.*  
*The goal is not to trick you — it's to find out how you think.*

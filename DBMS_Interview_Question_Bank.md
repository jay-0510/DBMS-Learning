# 🗄️ DBMS — Interview Question Bank
### 35 Questions | Conceptual Understanding · Practical Usage · Reasoning + 5 System Design Scenarios
> **Target Level:** Intern → 2–3 Years Industry Experience  
> **Format:** Plain-language answers · Real examples · Follow-up probes · No jargon overload

---

## Table of Contents

| Section | Topic | Questions |
|---------|-------|-----------|
| A | Keys & Relationships | Q1 – Q4 |
| B | ER Model | Q5 – Q6 |
| C | Normalisation | Q7 – Q9 |
| D | Transactions & ACID | Q10 – Q12 |
| E | Concurrency & MVCC | Q13 – Q15 |
| F | Indexing — B-Tree, B+Tree, Clustered | Q16 – Q20 |
| G | WAL — Write-Ahead Logging | Q21 – Q22 |
| H | Views — Virtual & Materialised | Q23 – Q24 |
| I | Stored Procedures & Triggers | Q25 – Q26 |
| J | Window Functions | Q27 – Q28 |
| K | Pagination | Q29 – Q29 |
| L | PostgreSQL vs MySQL | Q30 – Q30 |
| 🔥 | System Design Scenarios | S1 – S5 |

---

## Section A — Keys & Relationships

---

### Q1. What are the different types of keys in DBMS? Explain each with a simple example.

**Answer:**

Think of keys as ways to **uniquely identify rows** in a table or link tables together.

**Super Key**  
Any column (or combination of columns) that can uniquely identify a row.  
Example in a `Users` table: `(email)`, `(user_id)`, `(email, phone)` — all are super keys.

**Candidate Key**  
The minimal super key — no extra columns. Remove any column and it's no longer unique.  
Example: `email` alone uniquely identifies a user. `(email, phone)` is NOT a candidate key because `email` alone is enough.

**Primary Key**  
The candidate key you choose to be the "official" identifier for a table. Cannot be NULL. Only one per table.  
```sql
CREATE TABLE users (
    user_id   SERIAL PRIMARY KEY,
    email     VARCHAR(255) UNIQUE NOT NULL
);
```

**Alternate Key**  
A candidate key that was NOT chosen as the primary key.  
Here: `email` is an alternate key (candidate key not picked as PK).

**Foreign Key**  
A column in one table that references the primary key of another table. Creates a link between tables.  
```sql
CREATE TABLE orders (
    order_id  SERIAL PRIMARY KEY,
    user_id   INT REFERENCES users(user_id)  -- foreign key
);
```

**Composite Key**  
A primary key made of two or more columns. Used when no single column is unique but a combination is.  
```sql
-- Enrollment table: a student can enroll in many courses, a course has many students
-- But (student_id + course_id) is unique
PRIMARY KEY (student_id, course_id)
```

**Surrogate Key**  
An artificial key with no business meaning — usually an auto-incremented integer or UUID. Created purely for identification.  
Example: `user_id = 1, 2, 3...` — the user doesn't have an inherent ID in the real world; we made it up.

> **Probe:** Why would you use a surrogate key instead of a natural key like `email` as a primary key?  
> Natural keys can change (user updates their email). Foreign keys pointing to that email across 10 other tables all break. Surrogate keys (like `user_id = 42`) never change — safe to reference everywhere.

---

### Q2. What is a Foreign Key constraint and what happens if you try to insert a row that violates it?

**Answer:**

A foreign key ensures that a value in one table **must exist** in the referenced table. It enforces referential integrity — the database won't let you create orphan records.

```sql
CREATE TABLE orders (
    order_id  SERIAL PRIMARY KEY,
    user_id   INT REFERENCES users(user_id)
);

-- This will FAIL if user_id = 9999 doesn't exist in users table
INSERT INTO orders (user_id) VALUES (9999);
-- ERROR: insert or update on table "orders" violates foreign key constraint
```

**What happens on DELETE of the parent row?**

You can define behaviour:

```sql
-- Option 1: RESTRICT (default) — block the delete if child rows exist
user_id INT REFERENCES users(user_id) ON DELETE RESTRICT

-- Option 2: CASCADE — delete parent automatically deletes all child rows
user_id INT REFERENCES users(user_id) ON DELETE CASCADE

-- Option 3: SET NULL — child row's FK becomes NULL when parent is deleted
user_id INT REFERENCES users(user_id) ON DELETE SET NULL
```

**Real scenario:**  
A user deletes their account. With `ON DELETE CASCADE`, their orders, sessions, and preferences are all automatically removed. With `RESTRICT`, they can't delete until you clean up the child records first.

> **Probe:** When is `ON DELETE CASCADE` dangerous?  
> When you have multiple levels of cascades. Deleting one parent can trigger a chain that wipes thousands of rows across multiple tables — sometimes more than you intended. Always test cascades in a staging environment first.

---

### Q3. What is the difference between a Primary Key and a Unique Key?

**Answer:**

| Property | Primary Key | Unique Key |
|----------|-------------|------------|
| NULL values | **Not allowed** | Allowed (one NULL per column in most DBs) |
| Number per table | Only **one** | Can have **many** |
| Creates index? | Yes (clustered by default in MySQL) | Yes (non-clustered) |
| Purpose | Main row identifier | Enforce uniqueness for business rules |

```sql
CREATE TABLE users (
    user_id   SERIAL PRIMARY KEY,      -- main identifier, no NULL
    email     VARCHAR(255) UNIQUE,     -- must be unique, but could be NULL
    phone     VARCHAR(20) UNIQUE       -- another unique constraint
);
```

**When to use which:**  
Use PRIMARY KEY for the row's identity. Use UNIQUE for business rules like "no two users can have the same email or same phone number."

> **Probe:** Can a table have no primary key?  
> Yes, but it's a bad practice. Without a PK, you can have completely duplicate rows with no way to uniquely identify or delete just one of them. Most ORMs and tools expect a primary key.

---

### Q4. What is referential integrity and why does it matter?

**Answer:**

Referential integrity is a guarantee that **relationships between tables stay valid** — no row in a child table can point to a non-existent row in the parent table.

**Without referential integrity (bad state):**
```
orders table:
order_id | user_id
1        | 42       ← user 42 doesn't exist in users table (orphan record)
```

This is a data consistency problem. If you try to display "who placed this order?" — you get nothing or a crash.

**With foreign keys, the database enforces this automatically:**
```sql
-- This prevents orphan records from ever being created
user_id INT REFERENCES users(user_id)
```

**Why it matters in real applications:**
- Financial systems: a transaction must always belong to a valid account
- E-commerce: an order must always belong to a valid user
- Content platforms: a comment must always belong to a valid post

Some teams disable FK constraints for performance and enforce integrity at the application layer instead. This works until one bug or bad migration creates thousands of orphan records that corrupt reports and analytics.

> **Probe:** Should you always use foreign key constraints in production?  
> Not always. High-throughput systems (millions of inserts/second) sometimes drop FK constraints because every insert requires a lookup to verify the parent exists. Instead, they enforce integrity at the application layer and run periodic reconciliation checks. This is a deliberate trade-off — speed vs database-level safety.

---

## Section B — ER Model

---

### Q5. What is an ER Model? Explain the main components with a real example.

**Answer:**

ER (Entity-Relationship) Model is a **visual way to design a database** before writing any SQL. It shows what data you store and how different pieces of data relate to each other.

**Main components:**

**Entity** — A real-world object you want to store data about. Drawn as a rectangle.  
Examples: `User`, `Product`, `Order`

**Attribute** — A property of an entity. Drawn as an oval.  
`User` has attributes: `user_id`, `name`, `email`, `created_at`

**Relationship** — How entities connect to each other. Drawn as a diamond.  
`User` PLACES `Order`

**Example — Simple E-commerce ER:**

```
[User] ----<PLACES>---- [Order] ----<CONTAINS>---- [Product]
  |                        |
user_id               order_id
name                  total_amount
email                 status
```

**Types of attributes:**
- **Simple:** `name`, `email` — cannot be broken down further
- **Composite:** `address` — can be split into `street`, `city`, `zip`
- **Multivalued:** `phone_numbers` — a user can have multiple phones (shown as double oval)
- **Derived:** `age` — derived from `date_of_birth`, not stored directly

> **Probe:** What is a weak entity?  
> An entity that cannot be uniquely identified on its own — it depends on another (strong) entity. Example: `OrderItem` can't exist without an `Order`. Its key is `(order_id + item_number)` — partial key depends on the parent.

---

### Q6. What are the types of relationships in an ER Model? Give a database example for each.

**Answer:**

Relationships describe **how many** of one entity relate to **how many** of another entity.

**One-to-One (1:1)**  
One row in Table A corresponds to exactly one row in Table B.  
Example: `User` ↔ `UserProfile` — one user has one profile.
```sql
CREATE TABLE user_profiles (
    profile_id INT PRIMARY KEY,
    user_id    INT UNIQUE REFERENCES users(user_id), -- UNIQUE enforces 1:1
    bio        TEXT
);
```

**One-to-Many (1:N)**  
One row in Table A corresponds to many rows in Table B. Most common relationship.  
Example: `User` → `Orders` — one user can have many orders.
```sql
-- user_id in orders table (many side) references users (one side)
CREATE TABLE orders (
    order_id INT PRIMARY KEY,
    user_id  INT REFERENCES users(user_id)
);
```

**Many-to-Many (M:N)**  
Many rows in Table A correspond to many rows in Table B. Requires a **junction table**.  
Example: `Student` ↔ `Course` — a student takes many courses, a course has many students.
```sql
CREATE TABLE enrollments (  -- junction table
    student_id INT REFERENCES students(student_id),
    course_id  INT REFERENCES courses(course_id),
    enrolled_at TIMESTAMP,
    PRIMARY KEY (student_id, course_id)
);
```

**Why does M:N need a junction table?**  
You can't store "multiple course IDs" in a single column cleanly. A separate table with one row per relationship is the relational way to handle this.

> **Probe:** What extra data can you store in a junction table?  
> Any metadata about the relationship itself. For `enrollments`: `grade`, `enrolled_at`, `status`. For an `order_items` junction between `orders` and `products`: `quantity`, `price_at_purchase`, `discount`.

---

## Section C — Normalisation

---

### Q7. What is Normalisation? Explain 1NF, 2NF, and 3NF with a concrete example.

**Answer:**

Normalisation is the process of **organising a database to reduce data redundancy and avoid update anomalies**. It's done in steps called Normal Forms.

**The problem we're solving — this unnormalised table:**

```
OrderID | CustomerName | CustomerCity | ProductName | ProductPrice | Qty
1       | Alice        | Mumbai       | Laptop      | 50000        | 1
1       | Alice        | Mumbai       | Mouse       | 500          | 2
2       | Bob          | Delhi        | Laptop      | 50000        | 1
```

Problems: Alice's city is stored twice. Laptop's price is stored twice. Update "Mumbai" → you have to update multiple rows.

---

**1NF — First Normal Form**  
Rule: Every cell must have **one atomic (indivisible) value**. No repeating groups. Each row must be unique.

Violation example:
```
phone_numbers: "9876543210, 9123456789"  ← two values in one cell — NOT 1NF
```
Fix: separate rows or separate table for phone numbers.

---

**2NF — Second Normal Form**  
Rule: Must be in 1NF + **every non-key column must depend on the ENTIRE primary key** (no partial dependency). Applies only when composite primary key exists.

Problem: If PK is `(OrderID, ProductName)`:
- `CustomerName` depends only on `OrderID` — partial dependency
- `ProductPrice` depends only on `ProductName` — partial dependency

Fix: Split into separate tables:
```sql
Orders(OrderID, CustomerName, CustomerCity)
Products(ProductName, ProductPrice)
OrderItems(OrderID, ProductName, Qty)  -- only Qty depends on both
```

---

**3NF — Third Normal Form**  
Rule: Must be in 2NF + **no transitive dependencies** (non-key column should not depend on another non-key column).

Problem: `CustomerCity` depends on `CustomerName`, not directly on `OrderID`.  
`OrderID → CustomerName → CustomerCity` — transitive!

Fix:
```sql
Customers(CustomerID, CustomerName, CustomerCity)
Orders(OrderID, CustomerID, ...)  -- reference CustomerID
```

> **Probe:** What is BCNF (Boyce-Codd Normal Form)?  
> A stricter version of 3NF. Rule: for every dependency X → Y, X must be a super key. BCNF handles edge cases in 3NF where there are multiple overlapping candidate keys. In practice, reaching 3NF is sufficient for most real-world databases.

---

### Q8. What are update, insert, and delete anomalies? Why does normalisation fix them?

**Answer:**

These are problems that occur in **poorly designed (unnormalised) tables** where the same data is stored in multiple places.

Using this bad table:
```
OrderID | CustomerName | CustomerCity | ProductPrice
1       | Alice        | Mumbai       | 50000
2       | Alice        | Mumbai       | 500
```

**Update Anomaly:**  
Alice moves from Mumbai to Pune. You have to update EVERY row that has Alice's name. If you miss one row, the data becomes inconsistent — Alice lives in both Mumbai and Pune simultaneously.

**Insert Anomaly:**  
You want to add a new product "Keyboard" at price 1500. You can't insert it without creating a fake order — the product has no natural place without an `OrderID`.

**Delete Anomaly:**  
You delete Order #2 (Alice's only other order). You lose the fact that Product "Mouse" costs 500 — that information is gone with the row.

**How normalisation fixes this:**  
By putting data in one place. Alice's city exists in exactly one row in the `Customers` table. Update it once, done. Products exist in exactly one row in the `Products` table — no fake orders needed, no data lost on delete.

> **Probe:** Is it ever okay to intentionally denormalise a database?  
> Yes — for read-heavy systems where query performance matters more than write simplicity. Data warehouses, analytics databases, and reporting systems often denormalise to avoid expensive JOINs at query time. It's a conscious trade-off: accept some redundancy and extra update complexity in exchange for faster reads.

---

### Q9. What is denormalisation? When and why would you do it?

**Answer:**

Denormalisation is **intentionally adding redundancy** back into a database to make reads faster. It's the opposite of normalisation — you're trading storage and write complexity for query speed.

**Example:**  
Normalised: `orders` and `customers` are separate tables. To display "Order #42 placed by Alice from Mumbai," you need a JOIN.

Denormalised: Store `customer_name` and `customer_city` directly in the `orders` table. No JOIN needed to show order details.

```sql
-- Normalised (requires JOIN)
SELECT o.order_id, c.name, c.city
FROM orders o JOIN customers c ON o.customer_id = c.customer_id
WHERE o.order_id = 42;

-- Denormalised (single table read, no JOIN)
SELECT order_id, customer_name, customer_city
FROM orders
WHERE order_id = 42;
```

**When to denormalise:**
- Read-heavy workloads where JOINs are slow (analytics dashboards)
- Frequently accessed data where latency is critical
- Data warehouses and OLAP systems (designed for reads, not real-time writes)
- Caching layers (Redis cache stores denormalised objects)

**The cost:**
- Updates become complex (updating Alice's city requires updating orders table too)
- More storage used
- Risk of inconsistency if updates are missed

> **Probe:** What's the difference between denormalisation and bad design?  
> Intentionality and documentation. Bad design is accidental redundancy that no one planned for. Denormalisation is a deliberate performance decision, documented with a reason ("JOINs on this path were taking 2 seconds — denormalised customer_name into orders to bring it under 50ms").

---

## Section D — Transactions & ACID

---

### Q10. What is a Transaction? Explain ACID properties with simple, real examples.

**Answer:**

A **transaction** is a group of database operations that must all succeed or all fail together. You can't have a half-completed transaction.

**Classic example — bank transfer:**
```
Transfer $500 from Alice to Bob:
1. Deduct $500 from Alice's account
2. Add $500 to Bob's account
```
If step 1 succeeds but step 2 fails (server crash, network error), Alice loses $500 and Bob gets nothing. That's catastrophic. Transactions prevent this.

---

**ACID — the four guarantees:**

**A — Atomicity**  
"All or nothing." Either all operations in the transaction complete, or none of them do. If anything fails, the database rolls back to its state before the transaction started.

```sql
BEGIN;
UPDATE accounts SET balance = balance - 500 WHERE user = 'Alice';
UPDATE accounts SET balance = balance + 500 WHERE user = 'Bob';
COMMIT;  -- both succeed together
-- If anything fails before COMMIT → automatic ROLLBACK
```

**C — Consistency**  
The database must be in a valid state before AND after the transaction. It must obey all rules (constraints, triggers, cascades). A transaction that would violate a constraint is rejected entirely.

Example: Account balance must never go below 0. If Alice has $300 and tries to send $500, the transaction fails — the database stays consistent.

**I — Isolation**  
Concurrent transactions must not interfere with each other. Each transaction should behave as if it's running alone.

Example: Alice and Bob both read inventory (10 items left) at the same time. Both try to buy 8. Without isolation, both could "succeed" — selling 16 items when only 10 exist.

**D — Durability**  
Once a transaction is committed, it's permanent — even if the server crashes 1 millisecond after COMMIT. The data is written to disk (via mechanisms like WAL — covered in Q21).

```sql
BEGIN;
-- ... operations ...
COMMIT;  -- after this point, data is safe even if power goes out
```

> **Probe:** Is ACID free? What do you sacrifice for it?  
> Performance. Enforcing ACID requires locking, logging, and coordination. High-write-throughput systems (like logging pipelines or IoT data ingestion) sometimes use NoSQL databases that offer weaker guarantees (BASE: Basically Available, Soft-state, Eventually consistent) in exchange for much higher throughput.

---

### Q11. What is the difference between COMMIT and ROLLBACK?

**Answer:**

These two commands are the "confirm" and "cancel" buttons for a transaction.

**COMMIT** — Makes all changes in the transaction **permanent**. Once committed, changes are visible to other users and survive crashes.

**ROLLBACK** — **Undoes** all changes made since the transaction began. The database returns to exactly the state it was in before the BEGIN.

```sql
-- Successful transaction
BEGIN;
UPDATE accounts SET balance = balance - 500 WHERE id = 1;
UPDATE accounts SET balance = balance + 500 WHERE id = 2;
COMMIT;  -- both updates are now permanent

-- Failed transaction
BEGIN;
UPDATE accounts SET balance = balance - 500 WHERE id = 1;
-- Something went wrong (error, application crash)
ROLLBACK;  -- the first UPDATE is undone — Alice's money is back
```

**SAVEPOINT — partial rollback:**  
You can create checkpoints inside a transaction and roll back to a specific point without undoing everything.

```sql
BEGIN;
INSERT INTO orders (...) VALUES (...);
SAVEPOINT after_order;

INSERT INTO payments (...) VALUES (...);
-- Payment fails
ROLLBACK TO SAVEPOINT after_order;  -- undo only payment, keep order

COMMIT;  -- only order is committed
```

> **Probe:** What happens to an uncommitted transaction if the database server crashes?  
> The transaction is automatically rolled back when the server restarts. The WAL (Write-Ahead Log) records what was happening, and during recovery, incomplete transactions are undone. This is how Atomicity survives crashes.

---

### Q12. What are isolation levels? Explain each one and what problems they solve.

**Answer:**

Isolation levels control **how much one transaction can see the intermediate state of another**. Higher isolation = more safety but more locking = slower performance.

**The problems isolation levels are solving:**

| Problem | What it means |
|---------|---------------|
| **Dirty Read** | Reading uncommitted changes from another transaction (which might be rolled back) |
| **Non-repeatable Read** | Reading a row twice in the same transaction, getting different values (another transaction updated it between your reads) |
| **Phantom Read** | Running the same query twice, getting different rows (another transaction inserted/deleted rows between your reads) |

**The four isolation levels:**

**READ UNCOMMITTED** (weakest)  
Can read uncommitted changes from other transactions. Fastest but dangerous.  
Allows: dirty reads, non-repeatable reads, phantom reads.  
Use: almost never. Only for rough analytics where a little wrongness is okay.

**READ COMMITTED** (default in PostgreSQL, Oracle)  
Only reads committed data. Re-reads the same row may return updated values.  
Prevents: dirty reads  
Allows: non-repeatable reads, phantom reads

**REPEATABLE READ** (default in MySQL InnoDB)  
All reads in a transaction see a consistent snapshot from when the transaction started.  
Prevents: dirty reads, non-repeatable reads  
Allows: phantom reads (in some implementations)

**SERIALIZABLE** (strongest)  
Transactions are completely isolated — behave as if they run one after another, not concurrently.  
Prevents: all three problems  
Cost: significant locking and slowdown

```sql
-- Set isolation level in PostgreSQL
BEGIN;
SET TRANSACTION ISOLATION LEVEL REPEATABLE READ;
-- ... your queries
COMMIT;
```

> **Probe:** What isolation level does your application probably need?  
> READ COMMITTED is correct for most web applications. Serializable is needed for financial operations (bank transfers, inventory allocation) where phantom reads could cause double-spending or overselling. The default in your database is usually a good starting point — only change it when you have a specific concurrency problem to solve.

---

## Section E — Concurrency & MVCC

---

### Q13. What is a deadlock? How does it happen and how do databases handle it?

**Answer:**

A **deadlock** is when two (or more) transactions are each waiting for the other to release a lock — and neither can proceed. They're stuck forever.

**Classic deadlock:**

```
Transaction A:                    Transaction B:
Lock row user_id = 1              Lock row user_id = 2
Try to lock user_id = 2... WAIT   Try to lock user_id = 1... WAIT
```

Both are waiting. Neither will ever get what they need.

**How databases detect and handle it:**  
The database runs a **deadlock detector** that periodically checks for cycles in the "who is waiting for whom" graph. When a cycle is found, one transaction is chosen as the "victim" and killed (rolled back). The other transaction gets its lock and continues.

```
ERROR: deadlock detected
DETAIL: Process 1234 waits for ShareLock on transaction 5678
        Process 5678 waits for ShareLock on transaction 1234
HINT:  See server log for query details.
```

**How to prevent deadlocks in application code:**

1. **Always lock resources in the same order** — if all transactions lock user_id then order_id (never the reverse), cycles can't form.

2. **Keep transactions short** — shorter transactions hold locks for less time, less chance of conflict.

3. **Use `SELECT ... FOR UPDATE NOWAIT`** — fails immediately instead of waiting, so your app can handle it gracefully.

```sql
-- Locks the row and fails immediately if it's already locked
SELECT * FROM accounts WHERE id = 1 FOR UPDATE NOWAIT;
```

> **Probe:** What's the difference between a deadlock and a lock timeout?  
> A deadlock is a circular wait that can never resolve — the database must intervene and kill one transaction. A lock timeout is when one transaction waits too long for a lock that IS held but will eventually be released — after the timeout period, the waiting transaction gives up with an error.

---

### Q14. What is MVCC (Multi-Version Concurrency Control)?

**Answer:**

MVCC is the technique databases use to let **reads and writes happen at the same time without blocking each other**. Instead of locking a row when reading, the database keeps multiple versions of a row and shows each transaction the version it's supposed to see.

**The core idea:**  
"Writers don't block readers. Readers don't block writers."

**How it works in PostgreSQL:**

When you update a row, PostgreSQL doesn't overwrite it. Instead:
1. The old version is **marked as expired** (kept temporarily)
2. A **new version** is created with the updated data
3. Each transaction sees the version that was current **when the transaction started**

```sql
-- Transaction A starts at time T1
BEGIN;
SELECT balance FROM accounts WHERE id = 1;
-- Sees balance = $1000 (the version at T1)

-- Transaction B updates the row at time T2 (while A is still running)
-- In another connection:
UPDATE accounts SET balance = 1500 WHERE id = 1;
COMMIT;

-- Back in Transaction A:
SELECT balance FROM accounts WHERE id = 1;
-- STILL sees $1000 — because A started at T1, before B's update
-- This is REPEATABLE READ behaviour via MVCC
COMMIT;
```

**Why this is better than locking:**  
Without MVCC, Transaction A's read would block until B finishes, or B's write would wait for A to finish reading. With MVCC, they run simultaneously — no waiting.

**The cost:**  
Old row versions accumulate and take up space. PostgreSQL uses a background process called **VACUUM** to clean up old row versions that no transaction needs anymore.

> **Probe:** What is VACUUM in PostgreSQL and why is it needed?  
> VACUUM reclaims storage from dead row versions (old MVCC versions no longer visible to any active transaction). Without regular VACUUM, the database slowly fills up with unreachable old data — a phenomenon called "table bloat." `AUTOVACUUM` runs this automatically in the background on PostgreSQL.

---

### Q15. What is a lock and what are the types of locks in a database?

**Answer:**

A lock is a mechanism that **controls concurrent access to data** — it prevents two transactions from making conflicting changes to the same row or table at the same time.

**Row-level locks (most common):**

```sql
-- Shared lock — multiple transactions can read simultaneously
SELECT * FROM accounts WHERE id = 1 FOR SHARE;

-- Exclusive lock — only one transaction can write; others must wait
SELECT * FROM accounts WHERE id = 1 FOR UPDATE;
```

**Table-level locks:**

```sql
-- Lock entire table (used for schema changes)
LOCK TABLE orders IN EXCLUSIVE MODE;
```

**Lock compatibility:**

| | Shared Lock | Exclusive Lock |
|--|-------------|----------------|
| **Shared Lock** | ✅ Compatible (both can read) | ❌ Conflict |
| **Exclusive Lock** | ❌ Conflict | ❌ Conflict |

**Optimistic vs Pessimistic locking:**

**Pessimistic** — Lock the row before you read it. Assumes conflicts will happen.
```sql
SELECT * FROM orders WHERE id = 1 FOR UPDATE;  -- lock immediately
-- do your work
UPDATE orders SET status = 'processing' WHERE id = 1;
COMMIT;
```

**Optimistic** — Don't lock. Check at write time if someone else changed the row (using a version number).
```sql
-- Read without locking
SELECT id, status, version FROM orders WHERE id = 1;
-- version = 5

-- Update only if version hasn't changed
UPDATE orders SET status = 'processing', version = 6
WHERE id = 1 AND version = 5;
-- If 0 rows updated → someone else changed it → retry or error
```

Optimistic locking is better for low-conflict scenarios (most web apps). Pessimistic for high-conflict (banking).

> **Probe:** What is a "phantom read" and which lock prevents it?  
> A phantom read is when you run the same SELECT twice in a transaction and get different rows because another transaction inserted rows matching your WHERE clause between your two reads. Prevented by range locks or Serializable isolation level (which PostgreSQL implements via predicate locking).

---

## Section F — Indexing

---

### Q16. What is an index? Why does it make queries faster?

**Answer:**

An index is a **separate data structure** that the database maintains alongside a table to speed up lookups. Think of it like a book index — instead of reading every page to find "normalisation," you look up the index which tells you exactly which page to go to.

**Without an index:**
```sql
SELECT * FROM users WHERE email = 'alice@example.com';
-- Database reads EVERY row in the table comparing emails
-- Called a "full table scan" — O(n) where n = number of rows
-- 1 million rows = 1 million comparisons
```

**With an index:**
```sql
CREATE INDEX idx_users_email ON users(email);
-- Now the database uses a B+Tree to find the email in O(log n)
-- 1 million rows = ~20 comparisons
```

**How to check if your query uses an index:**
```sql
EXPLAIN SELECT * FROM users WHERE email = 'alice@example.com';
-- Look for "Index Scan" vs "Seq Scan" (Sequential = full table scan, bad)
```

**Cost of indexes:**
- Every INSERT, UPDATE, DELETE must also update the index → slower writes
- Indexes take disk space and memory
- Too many indexes slows down write-heavy tables

**Golden rule:** Index columns you frequently filter by (`WHERE`), join on, or sort by (`ORDER BY`).

> **Probe:** Why is `SELECT *` bad for query performance?  
> It returns every column, which may prevent the database from using a "covering index" (an index that contains all columns needed for the query). If the index has all the data the query needs, the database never touches the actual table rows — much faster. `SELECT *` forces a trip to the table even when an index could have served the query alone.

---

### Q17. What is the difference between a B-Tree and a B+Tree? Which one do databases prefer and why?

**Answer:**

Both are self-balancing tree structures used for indexes. Understanding the difference explains why databases almost universally choose B+Tree.

**B-Tree:**
- Data (actual values) is stored in **both internal nodes AND leaf nodes**
- Finding a value might stop at an internal node (early exit)
- Leaf nodes are NOT linked to each other

```
        [30]
       /    \
   [10,20]  [40,50]
   /  |  \    |  \
  8  15  25  35  60   ← data lives at all levels
```

**B+Tree:**
- Data is stored **ONLY in leaf nodes**
- Internal nodes only store keys for navigation (no actual data)
- Leaf nodes are **linked together in a doubly linked list**

```
        [30]           ← navigation only, no data here
       /    \
   [10,20]  [40,50]   ← navigation only
   ↓   ↓   ↓   ↓   ↓
  [8]-[15]-[25]-[35]-[60]  ← ALL data here, linked in order
```

**Why databases (PostgreSQL, MySQL InnoDB) choose B+Tree:**

1. **Range queries are fast:** Because leaf nodes form a linked list, a query like `WHERE age BETWEEN 20 AND 50` traverses the linked list from 20 to 50 — never goes back up the tree. In a B-Tree, you'd have to traverse back up and down repeatedly.

2. **More keys per internal node:** Internal nodes don't carry data, just keys — they're smaller, so more fit in one disk page → tree is shallower → fewer disk reads per lookup.

3. **Consistent query time:** Every lookup always goes to a leaf node — same depth every time. B-Tree can return early at any level — inconsistent performance.

```sql
-- This range query is perfect for B+Tree
SELECT * FROM orders
WHERE created_at BETWEEN '2024-01-01' AND '2024-01-31';
-- B+Tree: find Jan 1 leaf, then follow the linked list to Jan 31
-- B-Tree: much more traversal required
```

**When would B-Tree be preferred?**  
If you only ever do exact lookups (no range queries) and every node access must return a result quickly without always reaching a leaf — theoretical advantage for exact key lookups. In practice, databases still use B+Tree even for this case because the benefits for range queries outweigh the minor overhead.

> **Probe:** How does a B+Tree handle an INSERT of a new value?  
> It finds the correct leaf node and inserts there. If the leaf is full, it splits into two leaves and promotes a key to the parent internal node. If the parent is also full, it splits too — this can cascade up to the root. If the root splits, a new root is created and the tree grows taller by one level. This is how B+Trees stay balanced.

---

### Q18. What is a Clustered Index and how is it different from a Non-Clustered Index?

**Answer:**

The difference is about **where the actual row data lives.**

**Clustered Index:**  
The table data is **physically sorted and stored on disk in the index order**. The index leaf nodes ARE the actual data rows. There can be only ONE clustered index per table — because data can only be physically sorted one way.

In MySQL InnoDB, the Primary Key is always the clustered index. In PostgreSQL, you can cluster a table on any index (but it's not maintained automatically after that).

```sql
-- MySQL InnoDB: Primary Key = Clustered Index automatically
CREATE TABLE orders (
    order_id INT PRIMARY KEY,  -- data on disk is sorted by order_id
    user_id  INT,
    amount   DECIMAL
);
-- Finding order_id = 500: traverse B+Tree → reach leaf → data IS here
```

**Non-Clustered Index:**  
A separate data structure that stores keys + a **pointer (row locator)** to the actual row. The table data is not sorted. There can be many non-clustered indexes per table.

```sql
CREATE INDEX idx_orders_user_id ON orders(user_id);
-- This index has: (user_id → pointer to actual row)
-- Finding orders by user_id:
-- 1. Traverse the index B+Tree → find user_id = 42 → get pointer
-- 2. Follow pointer to actual table row (extra step)
```

**The key difference:**

| | Clustered Index | Non-Clustered Index |
|--|-----------------|---------------------|
| Data location | Data IS the leaf | Leaf has pointer to data |
| Number allowed | 1 per table | Many per table |
| Lookup speed | Faster (one lookup) | Slightly slower (pointer follow) |
| Range scan | Very fast (data is sorted) | Slower (may jump around disk) |

**Covering Index (bonus concept):**  
A non-clustered index that includes all columns the query needs — the database never has to follow the pointer to the actual row. It "covers" the query entirely from the index.

```sql
-- Index includes both filter column AND return column
CREATE INDEX idx_covering ON orders(user_id, amount);

SELECT amount FROM orders WHERE user_id = 42;
-- Never touches the main table — all data is in the index itself
```

> **Probe:** Why is it expensive to insert rows with random Primary Key values (like UUIDs) into a MySQL InnoDB table?  
> Because InnoDB's clustered index forces data to be physically sorted by PK. A random UUID might sort in the middle of existing pages — forcing a **page split** (rewriting two pages instead of appending to one). Auto-increment integers always append to the end — no page splits, much faster inserts.

---

### Q19. What is a composite index? What is the "left prefix" rule?

**Answer:**

A **composite index** (or compound index) is an index on **two or more columns together**. It's more efficient than separate single-column indexes when your queries filter on multiple columns.

```sql
CREATE INDEX idx_orders_user_status ON orders(user_id, status, created_at);
```

**The Left Prefix Rule:**  
A composite index can only be used if your query filters by a **left-most prefix** of the indexed columns.

```sql
-- Index: (user_id, status, created_at)

-- ✅ Uses index — starts from leftmost column
WHERE user_id = 42

-- ✅ Uses index — uses first two columns
WHERE user_id = 42 AND status = 'pending'

-- ✅ Uses index — uses all three
WHERE user_id = 42 AND status = 'pending' AND created_at > '2024-01-01'

-- ❌ Does NOT use index — skips user_id (leftmost)
WHERE status = 'pending'

-- ❌ Does NOT use index — skips status (middle)
WHERE user_id = 42 AND created_at > '2024-01-01'
```

**Why?**  
The B+Tree for `(user_id, status, created_at)` sorts by user_id first, then status within each user_id, then created_at within each status. If you skip `user_id`, the database has no starting point — it would have to scan the entire index.

> **Probe:** Should you put high-cardinality or low-cardinality columns first in a composite index?  
> High-cardinality (many distinct values like `user_id`) first. This narrows the result set most aggressively at the first filter step. Low-cardinality first (like `status = 'active'` with only 3 possible values) barely filters — you'd still need to scan 33% of the table.

---

### Q20. What is a partial index and when is it useful?

**Answer:**

A partial index is an index that **only includes rows matching a specific condition**. It's smaller, faster to maintain, and uses less memory than a full index.

**Real scenario:**  
You have an `orders` table with 50 million rows. 49 million are `status = 'delivered'` (historical). Only 1 million are `status = 'pending'` (active). Your app constantly queries pending orders.

```sql
-- Full index on status — includes all 50M rows. Big and slow.
CREATE INDEX idx_status ON orders(status);

-- Partial index — only the 1M pending rows. Small and fast.
CREATE INDEX idx_pending_orders ON orders(user_id)
WHERE status = 'pending';

-- This query uses the partial index efficiently
SELECT * FROM orders WHERE user_id = 42 AND status = 'pending';
```

**Other use cases:**
```sql
-- Index only non-deleted records
CREATE INDEX idx_active_users ON users(email)
WHERE deleted_at IS NULL;

-- Index only large orders for reporting
CREATE INDEX idx_large_orders ON orders(amount)
WHERE amount > 10000;
```

**Benefits:**
- Smaller index = fits in memory more easily
- Faster INSERT/UPDATE/DELETE (only update index when row matches condition)
- Better query plans for filtered queries

> **Probe:** Can you use a partial index if your query doesn't include the WHERE condition in the index definition?  
> No. The query's WHERE clause must be compatible with the index's WHERE condition. If the index is `WHERE status = 'pending'`, only queries filtering `status = 'pending'` can use it.

---

## Section G — WAL (Write-Ahead Logging)

---

### Q21. What is WAL (Write-Ahead Logging)? Why is it important?

**Answer:**

WAL is a technique where **every change is written to a log file BEFORE it's written to the actual data file on disk**. This is how databases guarantee Durability (the D in ACID) and survive crashes.

**The problem WAL solves:**  
Writing to disk is not instant. If you COMMIT a transaction and the server crashes 1 millisecond later, did the data get saved? Without WAL: maybe not. With WAL: yes, always.

**How WAL works — step by step:**

```
1. Transaction begins
2. All changes are written to the WAL log file (fast, sequential writes)
3. COMMIT happens → WAL record is flushed to disk (fsync)
4. Database says "COMMIT successful" to the application
5. Later (asynchronously), actual data pages are updated on disk
```

The key insight: **WAL is sequential writes to a log file** (very fast). Writing to actual data pages is **random I/O** (slow). WAL lets the database acknowledge COMMIT immediately (after the fast log write) and update actual data pages later.

**How WAL handles crash recovery:**

```
Server crashes mid-operation
   ↓
Server restarts
   ↓
Database reads WAL from last checkpoint
   ↓
"Redo" all committed transactions in WAL that hadn't made it to data files
   ↓
"Undo" any uncommitted transactions that were in progress during crash
   ↓
Database is in a consistent state — ready to serve requests
```

**In PostgreSQL**, WAL files are stored in `$PGDATA/pg_wal/`. Each WAL file is 16MB by default.

> **Probe:** What is a checkpoint in PostgreSQL and how does it relate to WAL?  
> A checkpoint is when PostgreSQL flushes all "dirty" data pages (pages modified but not yet written to disk) to the actual data files. After a checkpoint, WAL records before that point are no longer needed for recovery (the data is safely on disk). Checkpoints happen periodically or when WAL fills up. During crash recovery, PostgreSQL only needs to replay WAL from the last checkpoint — not from the beginning of time.

---

### Q22. How does WAL enable replication and point-in-time recovery?

**Answer:**

WAL contains a **complete record of every change** made to the database — it's essentially the database's DNA. This makes it incredibly powerful beyond just crash recovery.

**WAL-based Replication (Streaming Replication in PostgreSQL):**

```
Primary Server                    Replica Server
    |                                  |
    | Generate WAL records             |
    | ─────────────────────────────→   |
    |     Stream WAL over network      |
    |                                  | Apply WAL records
    |                                  | → Replica stays in sync
```

The replica receives WAL stream from the primary and applies it — keeping an identical copy of the database in near real-time. When the primary fails, the replica can be promoted to primary with virtually no data loss.

**Point-in-Time Recovery (PITR):**  
Because WAL records every single change with a timestamp, you can restore a database to **any specific point in the past**.

```
Full backup from Sunday midnight
          +
WAL files from Sunday midnight → Tuesday 2:47 PM
          =
Database state at exactly Tuesday 2:47 PM
```

Real scenario: Someone ran `DELETE FROM orders WHERE id > 0` at 2:51 PM on Tuesday. You can restore the database to 2:46 PM — before the catastrophic delete — by replaying WAL up to that point.

```bash
# PostgreSQL recovery.conf (simplified)
restore_command = 'cp /wal_archive/%f %p'
recovery_target_time = '2024-01-16 14:46:00'
```

> **Probe:** What is the difference between physical replication (WAL streaming) and logical replication in PostgreSQL?  
> Physical replication streams raw WAL bytes — exact binary copy, fast, but the replica must be the same PostgreSQL version and architecture. Logical replication decodes WAL into SQL-level changes (INSERT/UPDATE/DELETE) — slower, but allows replicating to different versions, different databases, or specific tables only. Logical replication is used for migrations and selective sync.

---

## Section H — Views

---

### Q23. What is a View? What is the difference between a Virtual View and a Materialised View?

**Answer:**

A **view** is a saved query that you can treat like a table. You name it, and then query it as if it were a real table — the database handles running the underlying SQL for you.

**Virtual View (regular view):**  
Stores only the **query definition**, not the data. Every time you query the view, it runs the underlying SQL fresh.

```sql
-- Create a view
CREATE VIEW active_orders AS
SELECT o.order_id, u.name, o.amount, o.status
FROM orders o
JOIN users u ON o.user_id = u.id
WHERE o.status IN ('pending', 'processing');

-- Use the view
SELECT * FROM active_orders WHERE amount > 1000;
-- This actually runs the full JOIN + WHERE query every time
```

**Materialised View:**  
Stores the **actual query result** on disk. Querying it reads pre-computed data — no JOIN or WHERE computation at query time. BUT the data can become stale — you must refresh it.

```sql
-- Create materialised view
CREATE MATERIALIZED VIEW monthly_revenue AS
SELECT
    DATE_TRUNC('month', created_at) as month,
    SUM(amount) as revenue,
    COUNT(*) as order_count
FROM orders
GROUP BY 1;

-- Query is instant (reads stored result, no aggregation)
SELECT * FROM monthly_revenue WHERE month = '2024-01-01';

-- Data gets stale — must refresh manually or on schedule
REFRESH MATERIALIZED VIEW monthly_revenue;
REFRESH MATERIALIZED VIEW CONCURRENTLY monthly_revenue; -- no table lock
```

**Comparison:**

| | Virtual View | Materialised View |
|--|-------------|-------------------|
| Data stored? | No — query only | Yes — actual rows |
| Query speed | Depends on underlying query | Fast — pre-computed |
| Data freshness | Always current | Stale until refreshed |
| Storage used | Tiny | Same as result set |
| Use case | Simplify complex queries | Speed up slow aggregations |

> **Probe:** When would you definitely NOT use a materialised view?  
> When the underlying data changes frequently and you need real-time results. A materialised view of "current cart contents" would be useless because users add/remove items constantly and you'd be refreshing every few seconds, negating the performance benefit. Virtual views (or just direct queries) are better when freshness is critical.

---

### Q24. Can you UPDATE data through a View?

**Answer:**

Yes, but only under specific conditions. A view is **updatable** if it:
- Is based on a single table (no JOINs)
- Doesn't use DISTINCT, GROUP BY, HAVING, aggregate functions, UNION
- Doesn't use subqueries

```sql
-- Simple updatable view
CREATE VIEW user_emails AS
SELECT user_id, email FROM users;

-- This UPDATE works — affects the underlying users table
UPDATE user_emails SET email = 'newemail@example.com' WHERE user_id = 42;
```

**When a view is NOT directly updatable:**
```sql
-- This view has a JOIN — not updatable
CREATE VIEW orders_with_users AS
SELECT o.order_id, u.name, o.amount
FROM orders o JOIN users u ON o.user_id = u.id;

UPDATE orders_with_users SET amount = 999 WHERE order_id = 1;
-- ERROR: cannot update a view with a JOIN
```

**Fix: Use INSTEAD OF triggers or RULES** (PostgreSQL) to define what UPDATE on the view should do:
```sql
CREATE RULE update_order_amount AS
ON UPDATE TO orders_with_users
DO INSTEAD
UPDATE orders SET amount = NEW.amount WHERE order_id = OLD.order_id;
```

> **Probe:** What is the `WITH CHECK OPTION` on a view?  
> It ensures that any INSERT or UPDATE through a view must satisfy the view's WHERE condition. If you have a view of `WHERE status = 'active'` and try to update a row to `status = 'inactive'` through the view, `WITH CHECK OPTION` blocks it — preventing you from modifying rows you won't be able to see through the view anymore.

---

## Section I — Stored Procedures & Triggers

---

### Q25. What is a Stored Procedure and when should you use one?

**Answer:**

A stored procedure is **pre-written SQL code stored in the database** that you can call by name. It runs on the database server, not in your application.

```sql
-- Create a stored procedure in PostgreSQL
CREATE OR REPLACE PROCEDURE transfer_money(
    from_account INT,
    to_account   INT,
    amount       DECIMAL
)
LANGUAGE plpgsql AS $$
BEGIN
    -- Deduct from sender
    UPDATE accounts SET balance = balance - amount WHERE id = from_account;

    -- Add to receiver
    UPDATE accounts SET balance = balance + amount WHERE id = to_account;

    -- Commit happens when CALL returns successfully
    COMMIT;

EXCEPTION WHEN OTHERS THEN
    ROLLBACK;
    RAISE;
END;
$$;

-- Call it from application
CALL transfer_money(1, 2, 500.00);
```

**When to use stored procedures:**

✅ Complex multi-step operations that must be atomic  
✅ Operations that would require many round trips from app to DB  
✅ Enforcing business logic at the database layer for all clients  
✅ Batch processing inside the database (faster than row-by-row from app)

**When NOT to use them:**

❌ If your logic changes frequently — stored procedures are harder to version control and deploy  
❌ If you have multiple app teams — database-layer logic creates coupling  
❌ If your business logic is complex — hard to test and debug stored procedures  
❌ Modern trend: keep business logic in the application, use DB for data storage

> **Probe:** What is the difference between a Stored Procedure and a Function in SQL?  
> A **function** returns a value and can be used in a SELECT statement. A **stored procedure** performs actions (DML operations, transactions) and is called with CALL/EXECUTE. Functions cannot COMMIT/ROLLBACK transactions inside them (in most databases). Procedures can manage transactions.

---

### Q26. What is a Trigger? Give a real example of when you'd use one.

**Answer:**

A trigger is code that **automatically runs when a specific event happens** on a table — INSERT, UPDATE, or DELETE. You don't call triggers — the database fires them automatically.

```sql
-- Example: Automatically track when a record was last modified

-- Step 1: Create the trigger function
CREATE OR REPLACE FUNCTION update_modified_timestamp()
RETURNS TRIGGER AS $$
BEGIN
    NEW.updated_at = NOW();
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

-- Step 2: Attach it to a table
CREATE TRIGGER set_updated_at
    BEFORE UPDATE ON orders
    FOR EACH ROW
    EXECUTE FUNCTION update_modified_timestamp();

-- Now every UPDATE on orders automatically sets updated_at = NOW()
UPDATE orders SET status = 'delivered' WHERE order_id = 1;
-- updated_at is set automatically — no application code needed
```

**Types of triggers:**

| Timing | Events | Level |
|--------|--------|-------|
| BEFORE | INSERT, UPDATE, DELETE | FOR EACH ROW or STATEMENT |
| AFTER | INSERT, UPDATE, DELETE | FOR EACH ROW or STATEMENT |
| INSTEAD OF | (for views only) | FOR EACH ROW |

**Real use cases:**

```sql
-- Audit log: record every change to sensitive tables
CREATE TRIGGER audit_salary_changes
    AFTER UPDATE OF salary ON employees
    FOR EACH ROW
    EXECUTE FUNCTION log_salary_change();

-- Prevent deletion of critical records
CREATE TRIGGER prevent_delete_active_user
    BEFORE DELETE ON users
    FOR EACH ROW
    WHEN (OLD.status = 'active')
    EXECUTE FUNCTION raise_error('Cannot delete active users');
```

**When NOT to use triggers:**
- When the logic can be done in application code (easier to test and debug)
- When they create hidden behaviour that surprises developers
- When they cause performance problems on high-traffic tables (triggers run synchronously on every row)

> **Probe:** What is the difference between `FOR EACH ROW` and `FOR EACH STATEMENT` in a trigger?  
> `FOR EACH ROW`: the trigger fires once for every individual row affected by the statement. If `UPDATE orders SET status='x' WHERE status='pending'` updates 500 rows, the trigger fires 500 times. `FOR EACH STATEMENT`: fires once per SQL statement regardless of how many rows are affected — 500-row update fires the trigger once. Row-level gives you access to OLD and NEW values; statement-level doesn't.

---

## Section J — Window Functions

---

### Q27. What is a Window Function? How is it different from GROUP BY?

**Answer:**

A window function **performs a calculation across a set of rows related to the current row** — without collapsing those rows into one like GROUP BY does.

**The key difference:**
- `GROUP BY` aggregates → fewer rows come out
- Window function → SAME number of rows come out, but each row gets extra calculated columns

**Simple comparison:**

```sql
-- GROUP BY: collapses rows — you lose individual order data
SELECT user_id, SUM(amount) as total_spent
FROM orders
GROUP BY user_id;
-- Result: one row per user

-- Window function: keeps all rows, adds total_spent column to EACH
SELECT
    order_id,
    user_id,
    amount,
    SUM(amount) OVER (PARTITION BY user_id) as total_spent_by_user
FROM orders;
-- Result: every order row, but each shows the user's total spend
```

**Common window functions:**

```sql
SELECT
    order_id,
    user_id,
    amount,
    created_at,

    -- Row number within each user's orders, newest first
    ROW_NUMBER() OVER (PARTITION BY user_id ORDER BY created_at DESC) as order_rank,

    -- Running total of amount per user over time
    SUM(amount) OVER (PARTITION BY user_id ORDER BY created_at) as running_total,

    -- Rank by amount (same amount = same rank, next rank skipped)
    RANK() OVER (ORDER BY amount DESC) as amount_rank,

    -- Previous row's amount (lag)
    LAG(amount) OVER (PARTITION BY user_id ORDER BY created_at) as prev_order_amount,

    -- Next row's amount (lead)
    LEAD(amount) OVER (PARTITION BY user_id ORDER BY created_at) as next_order_amount

FROM orders;
```

**Anatomy of a window function:**
```sql
FUNCTION_NAME() OVER (
    PARTITION BY column   -- group rows (like GROUP BY, but doesn't collapse)
    ORDER BY column       -- order within each partition
    ROWS BETWEEN ...      -- optional: define frame (how many rows to include)
)
```

> **Probe:** What is the difference between RANK(), DENSE_RANK(), and ROW_NUMBER()?  
> Given scores: 100, 90, 90, 80:  
> `ROW_NUMBER()`: 1, 2, 3, 4 — always unique, no ties  
> `RANK()`: 1, 2, 2, 4 — ties get same rank, next rank is skipped  
> `DENSE_RANK()`: 1, 2, 2, 3 — ties get same rank, next rank is NOT skipped

---

### Q28. Write a query using a Window Function to find the top 3 orders per user by amount.

**Answer:**

This is a classic interview question. The trick is using ROW_NUMBER() to rank within each user's orders, then filtering in an outer query.

```sql
-- Step 1: Rank orders per user by amount (highest first)
WITH ranked_orders AS (
    SELECT
        order_id,
        user_id,
        amount,
        status,
        created_at,
        ROW_NUMBER() OVER (
            PARTITION BY user_id    -- restart counting for each user
            ORDER BY amount DESC    -- highest amount gets rank 1
        ) as rank_within_user
    FROM orders
)

-- Step 2: Keep only rank 1, 2, 3 per user
SELECT order_id, user_id, amount, status, created_at
FROM ranked_orders
WHERE rank_within_user <= 3
ORDER BY user_id, rank_within_user;
```

**Why not use RANK() here?**  
If two orders have the same amount (tied), `RANK()` would give them both rank 1 and skip rank 2 — you might get more than 3 rows per user. `ROW_NUMBER()` is deterministic — exactly 3 rows per user, no ties allowed. If you WANT to include all ties at position 3, use `DENSE_RANK() <= 3`.

**Alternative — Finding the #1 order per user:**
```sql
-- Get each user's most recent order (common real-world need)
WITH latest AS (
    SELECT *,
        ROW_NUMBER() OVER (PARTITION BY user_id ORDER BY created_at DESC) as rn
    FROM orders
)
SELECT * FROM latest WHERE rn = 1;
```

> **Probe:** What is a "frame clause" in a window function?  
> It defines which rows are included in the window calculation relative to the current row. `ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW` means: from the first row in the partition up to the current row — this is what makes SUM() calculate a running total instead of the total of all rows.

---

## Section K — Pagination

---

### Q29. How does pagination work in SQL? What is the problem with OFFSET at scale and how do you fix it?

**Answer:**

Pagination means returning data in "pages" (e.g., 20 rows at a time) instead of all at once — essential for any UI with lists.

**Method 1 — OFFSET/LIMIT (simple but has a problem):**

```sql
-- Page 1
SELECT * FROM orders ORDER BY created_at DESC LIMIT 20 OFFSET 0;

-- Page 2
SELECT * FROM orders ORDER BY created_at DESC LIMIT 20 OFFSET 20;

-- Page 100
SELECT * FROM orders ORDER BY created_at DESC LIMIT 20 OFFSET 1980;
```

**The OFFSET problem:**  
OFFSET doesn't skip rows efficiently — the database reads ALL rows up to the offset, then discards them. `OFFSET 1980` = database reads 2000 rows, throws away 1980, returns 20. At page 500 with OFFSET 9980, the database reads 10,000 rows for you to see 20.

With millions of rows and hundreds of pages, this becomes catastrophically slow.

**Method 2 — Cursor/Keyset Pagination (fast, scalable):**

Instead of "skip N rows," you say "give me rows AFTER this specific value."

```sql
-- Page 1
SELECT * FROM orders ORDER BY created_at DESC, order_id DESC LIMIT 20;
-- Last row returned: created_at = '2024-01-15 10:30:00', order_id = 5050

-- Page 2: start after the last row we saw
SELECT * FROM orders
WHERE (created_at, order_id) < ('2024-01-15 10:30:00', 5050)
ORDER BY created_at DESC, order_id DESC
LIMIT 20;
```

The database uses the index on `(created_at, order_id)` to jump directly to the right position — no scanning thousands of skipped rows.

**Comparison:**

| | OFFSET/LIMIT | Keyset Pagination |
|--|-------------|-------------------|
| Speed at page 1 | Fast | Fast |
| Speed at page 1000 | Very slow | Still fast |
| Can jump to page N | Yes | No (must go page by page) |
| Handles inserts during pagination | Rows may shift (you see duplicates/skip rows) | Stable — cursor position is stable |
| Implementation complexity | Simple | Moderate |

**When to use which:**
- Small datasets (< 100k rows, < 50 pages): OFFSET is fine
- Large datasets, infinite scroll, API pagination: always use keyset

> **Probe:** How does keyset pagination handle ties? If two rows have the exact same `created_at`, how do you ensure stable pagination?  
> Add a tiebreaker column (like `order_id`) to the ORDER BY and the WHERE clause. `(created_at, order_id)` as a compound cursor ensures every position is unique even when timestamps match.

---

## Section L — PostgreSQL vs MySQL

---

### Q30. PostgreSQL vs MySQL — which one should you use and when?

**Answer:**

Both are excellent relational databases. The right choice depends on your use case.

**PostgreSQL — choose when:**

```sql
-- 1. You need advanced data types
-- PostgreSQL supports: JSON, JSONB, Arrays, UUID, hstore, geometric types
CREATE TABLE products (
    id UUID DEFAULT gen_random_uuid(),
    metadata JSONB,            -- queryable JSON
    tags TEXT[],               -- native array column
    location POINT             -- geometric type
);

-- Query JSON natively
SELECT * FROM products WHERE metadata->>'category' = 'electronics';
SELECT * FROM products WHERE tags @> ARRAY['sale'];  -- array contains

-- 2. Complex queries, analytics, reporting
-- Better query planner, supports CTEs, LATERAL joins, better window functions

-- 3. Full ACID compliance for complex transactions
-- True serializable isolation without phantom read issues

-- 4. JSONB for hybrid relational/document workloads

-- 5. PostGIS extension for geospatial data
```

**MySQL — choose when:**

```sql
-- 1. Simple, read-heavy web applications
-- WordPress, Drupal, many CMSs default to MySQL for good reason

-- 2. Your team knows MySQL / existing ecosystem
-- Extensive hosting support, many managed offerings (PlanetScale, AWS RDS MySQL)

-- 3. You need MySQL-specific replication ecosystem
-- Group Replication, ProxySQL, Vitess (YouTube's MySQL scaling tool)

-- 4. Full-text search (MySQL's built-in FTS can be good enough)
SELECT * FROM posts WHERE MATCH(title, body) AGAINST('database indexing');
```

**Comparison table:**

| Feature | PostgreSQL | MySQL (InnoDB) |
|---------|------------|----------------|
| ACID compliance | Full | Full |
| JSON support | JSONB (indexed, queryable) | JSON (limited indexing) |
| Replication | Streaming (WAL-based), Logical | Binary log, Group Replication |
| Default isolation | Read Committed | Repeatable Read |
| Extensibility | High (custom types, operators, languages) | Moderate |
| Full-text search | OK (use Elasticsearch for serious use) | OK |
| Geospatial | PostGIS (best in class) | Basic |
| Community/ecosystem | Growing fast | Very mature, huge ecosystem |
| Managed cloud | RDS, Cloud SQL, Supabase, Neon | RDS, Cloud SQL, PlanetScale |

**Simple decision rule:**

- **Greenfield project, complex data, analytics, geospatial, or JSON workloads:** PostgreSQL
- **PHP/WordPress ecosystem, existing MySQL codebase, team MySQL expertise:** MySQL
- **Either works fine for:** standard CRUD web applications with straightforward schemas

> **Probe:** You're building a SaaS product that stores per-customer configuration as flexible key-value pairs (each customer has different settings). PostgreSQL or MySQL — which has the better solution?  
> PostgreSQL's JSONB column. You can store the configuration as a JSONB document, query specific keys with indexes, and avoid schema migrations every time a new configuration option is added. MySQL's JSON type exists but JSONB in PostgreSQL is more mature, supports GIN indexes for fast JSON key lookup, and has richer operator support.

---

---

# 🔥 System Design Scenarios

*These five scenarios test how you diagnose a problem and select the right tool or approach.*

---

## S1. The Slow Dashboard Problem

**Scenario:**

Your analytics dashboard shows total revenue by month, top 10 products, and user growth — refreshed every page load. As your orders table hits 50 million rows, the dashboard takes 45 seconds to load. Your manager says "fix it."

**What is happening?**

Every page load runs heavy aggregation queries (`SUM`, `COUNT`, `GROUP BY`) against 50 million raw rows in real time. These queries are scanning the full table on every request.

**What to do:**

1. **Materialised View** — Pre-compute the aggregations and store them.
```sql
CREATE MATERIALIZED VIEW dashboard_stats AS
SELECT
    DATE_TRUNC('month', created_at) as month,
    SUM(amount)   as revenue,
    COUNT(*)      as order_count
FROM orders
GROUP BY 1;

-- Refresh nightly (or every hour — depending on freshness need)
REFRESH MATERIALIZED VIEW CONCURRENTLY dashboard_stats;
```

2. **Indexes** — Ensure the refresh query itself is fast.
```sql
CREATE INDEX idx_orders_created_at ON orders(created_at);
```

3. **Cache the result** — Even with a materialised view, add Redis caching in front so the dashboard reads from Redis (sub-millisecond) and only hits the DB when cache expires.

**The trade-off:** Dashboard data is now slightly stale (last refresh). For a monthly revenue chart, "updated hourly" is completely acceptable. If data must be real-time, materialised views alone are insufficient — consider a streaming aggregation pipeline (Kafka + ksqlDB).

---

## S2. The Inventory Overselling Problem

**Scenario:**

Your e-commerce site sells concert tickets. A ticket has `quantity_available = 1`. Two users click "Buy" simultaneously. Both see 1 ticket available. Both complete the purchase. You've sold 1 ticket twice.

**What is happening?**

Classic race condition. Both transactions read `quantity = 1`, both pass the "is quantity > 0?" check, both decrement — result: `quantity = -1`. Neither MVCC nor READ COMMITTED prevents this because they both read before either writes.

**What to do:**

Use `SELECT ... FOR UPDATE` (pessimistic locking) to lock the row at read time:

```sql
BEGIN;

-- Lock the ticket row — second transaction WAITS here
SELECT quantity_available FROM tickets
WHERE ticket_id = 42 FOR UPDATE;

-- Check quantity (only one transaction sees this at a time)
-- quantity = 1 → proceed

UPDATE tickets
SET quantity_available = quantity_available - 1
WHERE ticket_id = 42 AND quantity_available > 0;
-- AND quantity_available > 0 = database-level guard

-- Check that the update actually happened
-- (if 0 rows updated, someone else got it first)
COMMIT;
```

**Or optimistic locking with a version column:**
```sql
UPDATE tickets
SET quantity_available = quantity_available - 1, version = version + 1
WHERE ticket_id = 42
AND quantity_available > 0
AND version = :version_we_read;
-- 0 rows updated = lost the race → show "sold out"
```

**Design rule:** Any time you read-then-write based on what you read, you need either locking or an atomic conditional update.

---

## S3. The "We Need History" Problem

**Scenario:**

Legal asks: "We need to know the exact state of every customer's order at every point in time — what was the price when they ordered, what status changes happened, when did each change happen." Your `orders` table just stores current state.

**What is happening?**

Your data model only stores the latest state — no history. You have no way to answer "what was this order's status at 3 PM on Tuesday?"

**What to do:**

**Option 1 — Audit log table with triggers:**
```sql
CREATE TABLE order_audit_log (
    log_id       SERIAL PRIMARY KEY,
    order_id     INT,
    changed_at   TIMESTAMP DEFAULT NOW(),
    changed_by   VARCHAR(100),
    old_status   VARCHAR(50),
    new_status   VARCHAR(50),
    old_amount   DECIMAL,
    new_amount   DECIMAL
);

-- Trigger captures every change
CREATE TRIGGER log_order_changes
    AFTER UPDATE ON orders
    FOR EACH ROW
    EXECUTE FUNCTION capture_order_change();
```

**Option 2 — Event Sourcing:** Never UPDATE — only INSERT new events.
```sql
CREATE TABLE order_events (
    event_id   SERIAL PRIMARY KEY,
    order_id   INT,
    event_type VARCHAR(50),  -- 'created', 'status_changed', 'amount_updated'
    payload    JSONB,
    occurred_at TIMESTAMP DEFAULT NOW()
);
-- Current state = replay all events for an order
```

**Option 3 — Temporal tables / Slowly Changing Dimensions:** Store `valid_from` and `valid_to` on every row. A row is "current" when `valid_to IS NULL`.

**Pick based on:** How often you query history (triggers = simpler for occasional queries; event sourcing = better when history is a first-class concern).

---

## S4. The Search Problem

**Scenario:**

Users are searching your product catalog by typing partial names: "wirel mou" should find "Wireless Mouse." Your SQL `WHERE name LIKE '%wirel%'` query takes 8 seconds and can't use an index.

**What is happening?**

`LIKE '%word%'` (leading wildcard) cannot use a B+Tree index — the database can't know where in the tree to start looking. It full-scans the entire table. With 500,000 products, this is slow. And it can't handle typos or fuzzy matching.

**What to do:**

**Option 1 — PostgreSQL Full-Text Search** (good for moderate scale):
```sql
-- Add a tsvector column for efficient full-text search
ALTER TABLE products ADD COLUMN search_vector tsvector;
UPDATE products SET search_vector = to_tsvector('english', name || ' ' || description);

-- Create GIN index (works for full-text search)
CREATE INDEX idx_products_fts ON products USING GIN(search_vector);

-- Query
SELECT * FROM products
WHERE search_vector @@ plainto_tsquery('english', 'wireless mouse')
ORDER BY ts_rank(search_vector, plainto_tsquery('english', 'wireless mouse')) DESC;
```

**Option 2 — pg_trgm (trigram similarity)** for typo-tolerance:
```sql
CREATE EXTENSION pg_trgm;
CREATE INDEX idx_products_trgm ON products USING GIN(name gin_trgm_ops);

SELECT * FROM products
WHERE name % 'wirel mou'  -- fuzzy match
ORDER BY similarity(name, 'wirel mou') DESC;
```

**Option 3 — Elasticsearch** (for serious scale, rich relevance):
Sync your product catalog to Elasticsearch via CDC (Change Data Capture) from WAL. Use Elasticsearch for all search queries — it handles typos, ranking, facets, multi-language. Your PostgreSQL remains the source of truth.

**Decision:** < 1M products, single language, moderate traffic → PostgreSQL FTS or trgm. > 1M products, multi-language, complex relevance → Elasticsearch.

---

## S5. The "Table is Too Big" Problem

**Scenario:**

Your `events` table has grown to 2 billion rows over 3 years. Queries are slow even with indexes. `VACUUM` takes hours and blocks operations. Storage is expensive. 80% of queries only ever touch the last 3 months of data.

**What is happening?**

A single table with 2 billion rows is hitting physical limits — the B+Tree index is deep (more levels = more disk reads per lookup), VACUUM has more dead rows to clean up, and the table occupies massive contiguous disk space.

**What to do:**

**Table Partitioning** — Split the table into smaller physical partitions by a key (usually time):

```sql
-- Create partitioned table
CREATE TABLE events (
    event_id   BIGINT,
    user_id    INT,
    event_type VARCHAR(50),
    created_at TIMESTAMP NOT NULL,
    payload    JSONB
) PARTITION BY RANGE (created_at);

-- Create monthly partitions
CREATE TABLE events_2024_01 PARTITION OF events
    FOR VALUES FROM ('2024-01-01') TO ('2024-02-01');

CREATE TABLE events_2024_02 PARTITION OF events
    FOR VALUES FROM ('2024-02-01') TO ('2024-03-01');
-- etc.
```

**Benefits:**
- Queries with `WHERE created_at > NOW() - INTERVAL '3 months'` only scan recent partitions — **partition pruning**
- VACUUM runs per-partition — smaller, faster
- Old partitions (2021 data) can be detached and archived to cold storage
- Dropping old data: `DROP TABLE events_2021_01` — instant (vs DELETE which is slow)

**Add index on each partition:**
```sql
CREATE INDEX ON events_2024_01(user_id, created_at);
-- Smaller index per partition = shallower B+Tree = fewer disk reads
```

**Archival strategy:** Partitions older than 1 year → detach from main table → dump to S3 as Parquet → query via Athena if needed. Active PostgreSQL only keeps 12 months of partitions — stays fast forever.

---

## Quick Reference Cheat Sheet

| Concept | One-Line Summary |
|---------|-----------------|
| Primary Key | Unique, not-null row identifier |
| Foreign Key | Links to another table's PK, enforces referential integrity |
| 1NF/2NF/3NF | Remove duplicates → remove partial deps → remove transitive deps |
| ACID | All-or-nothing, valid state, isolated, permanent |
| MVCC | Readers don't block writers — multiple versions of data coexist |
| B+Tree | All data in leaf nodes, leaves linked — perfect for range queries |
| Clustered Index | Data IS the index leaf — physically sorted, one per table |
| Non-Clustered | Separate structure with pointer to data — many per table |
| WAL | Log first, write data second — enables crash recovery and replication |
| Virtual View | Saved query — runs fresh every time, always current |
| Materialised View | Stored result — fast to query, must refresh manually |
| Window Function | Calculate across rows without collapsing them (unlike GROUP BY) |
| Keyset Pagination | Cursor-based paging — fast at any page depth unlike OFFSET |
| Trigger | Auto-runs on INSERT/UPDATE/DELETE — useful for auditing |
| Stored Procedure | Named, reusable SQL block — runs on DB server |
| Isolation Level | Controls how much concurrent transactions see each other |
| Deadlock | Circular lock wait — DB kills one transaction to resolve |
| Partial Index | Index only on rows matching a condition — smaller and faster |

---

*Prepared for DBMS interview readiness at intern → 2–3 years experience level.*  
*Every concept here is encountered in real production systems.*

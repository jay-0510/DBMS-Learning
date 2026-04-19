# COMPLETE POSTGRESQL - COMPREHENSIVE GUIDE


---

# TABLE OF CONTENTS

1. [Database Fundamentals](#section-1-database-fundamentals)
2. [Database Design & Integrity](#section-2-database-design--integrity)
3. [Transaction Management](#section-3-transaction-management)
4. [Storage & Indexing](#section-4-storage--indexing)
5. [SQL Basics & Querying](#section-5-7-sql-basics--querying)
6. [Filtering Data](#section-8-filtering-data)
7. [Joins](#section-9-joins)
8. [Aggregation & Grouping](#section-10-aggregation--grouping)
9. [Set Operations](#section-11-set-operations)
10. [Subqueries](#section-12-subqueries)
11. [Common Table Expressions](#section-13-common-table-expressions-cte)
12. [Modifying Data](#section-14-modifying-data)
13. [Table Management](#section-15-table-management)
14. [Data Types](#section-16-data-types)
15. [Conditional Expressions](#section-17-conditional-expressions)
16. [Views](#section-18-views)
17. [Indexes Deep-Dive](#section-19-indexes-deep-dive)
18. [Triggers](#section-20-triggers)
19. [Stored Procedures & Functions](#section-21-stored-procedures--functions-plpgsql)
20. [Advanced Functions](#section-22-advanced-functions)
21. [Performance Optimization](#section-23-performance-optimization)
22. [Real-World Scenarios](#section-24-real-world-scenarios)
23. [Essential Topics for AI/ML Engineers](#essential-postgresql-topics-for-aiml-engineers)
24. [Critical Optimization Scenarios](#5-critical-aiml-database-scenarios)

---

# **SECTION 1: DATABASE FUNDAMENTALS**

## **Why Databases Exist - The Real Problem**

Imagine you're building Instagram. Day 1: 100 users, you store data in text files. Day 30: 10,000 users. Day 90: 1 million users posting photos, liking, commenting. Your text file approach collapses because:

1. **Concurrent Access**: 1000 users trying to like the same post simultaneously - file gets corrupted
2. **Search Speed**: Finding "all posts by user_id=12345" means reading entire file
3. **Data Integrity**: User deletes account but their comments remain orphaned
4. **Relationships**: A post has an author, comments, likes, tags - how do you maintain these connections?

**This is why databases exist** - they solve these exact problems that appear at scale.

---

## **ER Model - Thinking Before Building**

### **The Mental Model**

Before writing any code, you need to answer: "What exists in my system and how do they relate?"

**Scenario: Building Twitter**

**Entities (Things that exist independently):**
- Users (can exist without tweets)
- Tweets (can exist without likes)
- Hashtags (can exist independently)

**Relationships (How entities connect):**
- User → Tweets (One user writes many tweets) = 1:N
- User → Follows → User (Many users follow many users) = M:N
- Tweet → Hashtags (One tweet has many hashtags, one hashtag appears in many tweets) = M:N

### **Why This Matters**

If you model "User follows User" as 1:N instead of M:N, your system breaks because:
- A user can only follow ONE other user
- You'd need separate rows for each follow relationship but with wrong structure

**Real Example: LinkedIn Connections**
- Wrong Model: User has "connections" field (1:N) → Can't represent mutual connections
- Right Model: Connection table with (user1_id, user2_id, connected_date) → M:N relationship

### **Cardinality - The Numbers Game**

**1:1 Example**: User ↔ UserProfile
- Each user has exactly one profile
- Each profile belongs to exactly one user
- In practice: Often merged into one table for performance

**1:N Example**: Department ↔ Employees
- One department has many employees
- Each employee belongs to one department
- Storage: Employee table has `department_id` foreign key

**M:N Example**: Students ↔ Courses
- One student enrolls in many courses
- One course has many students
- Storage: Requires junction table `enrollments(student_id, course_id, grade, enrollment_date)`

### **Internal Logic**

When you draw ER diagrams, you're making decisions that directly translate to:
1. **Table structure** (what columns exist)
2. **Query patterns** (how you'll fetch data)
3. **Storage size** (redundancy vs. normalization)

---

## **ER to Relational Model - The Translation**

### **The Conversion Rules**

**Rule 1: Strong Entities → Tables**
```
Entity: USER [user_id, name, email, created_at]
Becomes: users table with these columns
```

**Rule 2: Weak Entities → Tables with Foreign Key**
```
Entity: COMMENT (depends on POST)
Becomes: comments table with post_id as foreign key
Cannot exist without parent post
```

**Rule 3: 1:N Relationships → Foreign Key**
```
Author writes Books (1:N)
Becomes: books table with author_id column
```

**Rule 4: M:N Relationships → Junction Table**
```
Students enroll in Courses (M:N)
Becomes: 
- students table
- courses table  
- enrollments table (student_id, course_id, + any relationship attributes)
```

### **Real Scenario: E-commerce System**

**ER Model:**
- CUSTOMER (1) → orders → (N) ORDER
- ORDER (N) → contains → (N) PRODUCT (M:N relationship)
- PRODUCT (N) → belongs_to → (1) CATEGORY

**Relational Translation:**
```sql
customers (customer_id, name, email, address)

orders (order_id, customer_id, order_date, total_amount)
-- customer_id is FK because 1:N

categories (category_id, category_name)

products (product_id, name, price, category_id)
-- category_id is FK because N:1

order_items (order_id, product_id, quantity, price_at_purchase)
-- Junction table for M:N
-- Stores snapshot of price when ordered (not current price)
```

**Why `order_items` needs `price_at_purchase`:**
- Product price changes over time
- Historical orders must show what customer actually paid
- This is a **relationship attribute** - belongs to the connection, not the entities

---

## **Relational Algebra - The Mathematical Foundation**

### **Why It Exists**

SQL is built on relational algebra. Understanding this helps you:
1. Optimize queries mentally before writing them
2. Understand what's efficient vs. what's expensive
3. Debug slow queries by understanding operations

### **Core Operations with Internal Logic**

**1. SELECT (σ) - Filtering Rows**

Mathematical: σ(age > 25)(users)
```sql
SELECT * FROM users WHERE age > 25;
```

**Internal Process:**
1. Database scans users table (sequential or index scan)
2. For each row, evaluates condition: `row.age > 25`
3. If TRUE, includes in result set
4. If FALSE, skips

**Performance Impact:**
- Without index: O(n) - scans all rows
- With index on age: O(log n) + O(k) where k = matching rows

**2. PROJECT (π) - Selecting Columns**

Mathematical: π(name, email)(users)
```sql
SELECT name, email FROM users;
```

**Why This Matters:**
```sql
-- Bad: Transfers 1000 bytes per row
SELECT * FROM users;

-- Good: Transfers 100 bytes per row
SELECT user_id, name FROM users;
```

If fetching 1 million rows:
- SELECT *: 1GB data transfer
- SELECT specific: 100MB data transfer

**3. JOIN (⋈) - Combining Tables**

Mathematical: orders ⋈(orders.customer_id = customers.customer_id) customers

**Internal Algorithms (How Postgres Actually Does This):**

**Nested Loop Join:**
```
For each row in orders:
    For each row in customers:
        If orders.customer_id == customers.customer_id:
            Combine and output
```
- Complexity: O(n × m)
- Used when: Small tables or one side has very few rows
- Example: 10 orders × 1M customers = 10M comparisons

**Hash Join:**
```
Step 1: Build hash table from smaller table (customers)
    hash_table[customer_id] = customer_row

Step 2: Probe with larger table (orders)
    For each order:
        lookup = hash_table[order.customer_id]
        Combine and output
```
- Complexity: O(n + m)
- Used when: Large tables, sufficient memory
- Example: 1M orders + 1M customers = 2M operations (much better!)

**Merge Join:**
```
Step 1: Sort both tables by join key
Step 2: Walk through both sorted lists simultaneously
    If left.id == right.id: output
    If left.id < right.id: advance left
    If left.id > right.id: advance right
```
- Complexity: O(n log n + m log m) for sorting, then O(n + m) for merge
- Used when: Data already sorted or indexes exist

### **Real Scenario: Analytics Query**

**Problem:** "Find total sales per category for customers in California"

**Relational Algebra Breakdown:**
```
1. σ(state='CA')(customers)                    -- Filter customers
2. orders ⋈ (filtered_customers)               -- Join with orders
3. order_items ⋈ orders                        -- Join with order items
4. products ⋈ order_items                      -- Join with products
5. categories ⋈ products                       -- Join with categories
6. γ(category_name; SUM(quantity*price))      -- Group and aggregate
```

**SQL:**
```sql
SELECT c.category_name, SUM(oi.quantity * oi.price_at_purchase) as total_sales
FROM customers cu
JOIN orders o ON cu.customer_id = o.customer_id
JOIN order_items oi ON o.order_id = oi.order_id
JOIN products p ON oi.product_id = p.product_id
JOIN categories c ON p.category_id = c.category_id
WHERE cu.state = 'CA'
GROUP BY c.category_name;
```

**Optimization Thinking:**
1. Filter early (σ on customers) - reduces join size
2. Join order matters: customers (filtered) → orders → order_items → products → categories
3. If categories is small, might load into memory for hash join

**What Postgres Actually Does (EXPLAIN shows):**
```
1. Index scan on customers WHERE state='CA'     (1000 rows)
2. Hash join with orders on customer_id         (5000 rows)
3. Hash join with order_items on order_id       (15000 rows)  
4. Hash join with products on product_id        (15000 rows)
5. Hash join with categories on category_id     (15000 rows)
6. Hash aggregate by category_name              (50 rows)
```

---

## **Key Insights - Building Intuition**

### **Why This Foundation Matters**

**Scenario: You're Spotify with 500M users**

1. **ER Model helps**: Song ↔ Playlist is M:N, not 1:N
   - Wrong model: Can't have same song in multiple playlists
   - Right model: playlist_songs junction table

2. **Relational algebra helps**: "Songs played by users in last 7 days"
   - Filter first: σ(play_date > today-7)(plays) - reduces from 10B to 1B rows
   - Then join: filtered_plays ⋈ users
   - Not: users ⋈ plays, then filter - wastes 10x resources

3. **Join algorithm matters**:
   - Hash join for users ⋈ plays: O(500M + 1B) = 1.5B operations
   - Nested loop: O(500M × 1B) = 500 quadrillion operations (impossible)

### **Mental Models to Build**

1. **Entity = Noun**, Relationship = Verb
   - User CREATES post (1:N)
   - User LIKES post (M:N - needs junction table)

2. **Cardinality affects storage**
   - 1:N → Foreign key in "many" side
   - M:N → Separate junction table (always)

3. **Relational operations compose**
   - Complex queries are just combinations of σ, π, ⋈, γ
   - Understand each piece, understand the whole

4. **Filtering early is king**
   - WHERE clauses before JOINs when possible
   - Reduce dataset size at each step

---

# **SECTION 2: DATABASE DESIGN & INTEGRITY**

## **Database Normalization - Why We Break Tables Apart**

### **The Core Problem**

You start building a system. Your instinct: "One big table with everything!"
```sql
-- Your first attempt: orders table
CREATE TABLE orders (
    order_id INT,
    customer_name VARCHAR(100),
    customer_email VARCHAR(100),
    customer_phone VARCHAR(20),
    customer_address TEXT,
    product_name VARCHAR(100),
    product_price DECIMAL,
    product_category VARCHAR(50),
    quantity INT,
    order_date DATE
);
```

**Scenario: Customer "John" orders 3 different products**

| order_id | customer_name | customer_email | customer_phone | product_name | product_price | quantity |
|----------|---------------|----------------|----------------|--------------|---------------|----------|
| 1 | John Smith | john@email.com | 555-0100 | Laptop | 1200 | 1 |
| 2 | John Smith | john@email.com | 555-0100 | Mouse | 25 | 2 |
| 3 | John Smith | john@email.com | 555-0100 | Keyboard | 75 | 1 |

**Problems That Emerge:**

1. **Update Anomaly**: John changes his email
   - Must update 3 rows (what if you miss one?)
   - If John has 100 orders, update 100 rows
   - One typo = data inconsistency

2. **Insertion Anomaly**: You get a new product but no one has ordered it yet
   - Can't insert product because you need customer data
   - Or insert with NULL customer fields (wastes space, violates logic)

3. **Deletion Anomaly**: John cancels order #1
   - Delete the row
   - But now you've lost that Laptop exists in your catalog
   - Product information tied to orders (wrong!)

4. **Storage Waste**: 
   - "John Smith", "john@email.com" stored 3 times
   - 1 million orders = 1 million copies of customer data
   - Gigabytes wasted

### **Normal Forms - The Solution**

**Mental Model**: Each normal form fixes one specific type of redundancy.

---

### **First Normal Form (1NF) - Atomic Values**

**Rule**: No repeating groups, no multi-valued attributes.

**Bad Design:**
```sql
CREATE TABLE orders (
    order_id INT,
    customer_name VARCHAR(100),
    products VARCHAR(500)  -- "Laptop, Mouse, Keyboard"
);
```

**Why This Breaks:**
- How do you find all orders containing "Mouse"? → Use LIKE '%Mouse%' (super slow)
- How do you count products per order? → String parsing nightmare
- How do you join with products table? → Impossible

**Fixed (1NF):**
```sql
CREATE TABLE orders (
    order_id INT,
    customer_name VARCHAR(100),
    product_name VARCHAR(100)  -- One product per row
);
```

**Real Scenario: E-commerce Cart**

Wrong approach (developers do this!):
```sql
-- Storing cart items as JSON array
cart_items JSON  -- ["product_1", "product_2", "product_3"]
```

Problems:
- Can't index individual products
- Can't enforce foreign key to products table
- Can't efficiently query "How many carts contain product_5?"

Right approach:
```sql
CREATE TABLE cart_items (
    cart_id INT,
    product_id INT,
    quantity INT,
    PRIMARY KEY (cart_id, product_id)
);
```

---

### **Second Normal Form (2NF) - No Partial Dependencies**

**Rule**: Every non-key column must depend on the ENTIRE primary key.

**Scenario: Student Course Enrollment**

**Bad Design:**
```sql
CREATE TABLE enrollments (
    student_id INT,
    course_id INT,
    student_name VARCHAR(100),    -- Depends only on student_id
    course_name VARCHAR(100),     -- Depends only on course_id
    instructor VARCHAR(100),      -- Depends only on course_id
    grade CHAR(2),                -- Depends on both (correct!)
    PRIMARY KEY (student_id, course_id)
);
```

**Why This Is Wrong:**

Composite key: (student_id, course_id)

- `student_name` depends on `student_id` alone (partial dependency)
- `course_name` depends on `course_id` alone (partial dependency)
- `grade` depends on both (full dependency - this is correct)

**Problems:**
1. Student "Alice" enrolled in 5 courses → "Alice" stored 5 times
2. Course "Database Systems" has 100 students → "Database Systems" stored 100 times
3. Instructor changes for a course → Update 100 rows
4. Can't add a new course until someone enrolls (insertion anomaly)

**Fixed (2NF):**
```sql
CREATE TABLE students (
    student_id INT PRIMARY KEY,
    student_name VARCHAR(100)
);

CREATE TABLE courses (
    course_id INT PRIMARY KEY,
    course_name VARCHAR(100),
    instructor VARCHAR(100)
);

CREATE TABLE enrollments (
    student_id INT,
    course_id INT,
    grade CHAR(2),
    PRIMARY KEY (student_id, course_id),
    FOREIGN KEY (student_id) REFERENCES students(student_id),
    FOREIGN KEY (course_id) REFERENCES courses(course_id)
);
```

**Benefits:**
- Add 1000 courses without any enrollments ✓
- Update instructor once, affects all enrollments ✓
- Student name stored once regardless of enrollments ✓

---

### **Third Normal Form (3NF) - No Transitive Dependencies**

**Rule**: Non-key columns must depend on the key, the whole key, and nothing but the key.

**Scenario: Employee Database**

**Bad Design:**
```sql
CREATE TABLE employees (
    employee_id INT PRIMARY KEY,
    employee_name VARCHAR(100),
    department_id INT,
    department_name VARCHAR(100),    -- Depends on department_id, not employee_id
    department_budget DECIMAL        -- Depends on department_id, not employee_id
);
```

**The Hidden Dependency Chain:**
```
employee_id → department_id → department_name
employee_id → department_id → department_budget
```

This is a **transitive dependency**: employee_id determines department_id, which determines department_name.

**Problems:**
1. 1000 employees in "Engineering" → "Engineering" stored 1000 times
2. Engineering budget changes → Update 1000 rows
3. Department name typo in one row → Data inconsistency
4. Last employee leaves department → Delete row, lose department info

**Fixed (3NF):**
```sql
CREATE TABLE departments (
    department_id INT PRIMARY KEY,
    department_name VARCHAR(100),
    budget DECIMAL
);

CREATE TABLE employees (
    employee_id INT PRIMARY KEY,
    employee_name VARCHAR(100),
    department_id INT,
    FOREIGN KEY (department_id) REFERENCES departments(department_id)
);
```

**Real-World Example: Address Data**

**Common Mistake:**
```sql
CREATE TABLE users (
    user_id INT PRIMARY KEY,
    name VARCHAR(100),
    zip_code VARCHAR(10),
    city VARCHAR(100),      -- Determined by zip_code!
    state VARCHAR(50)       -- Determined by zip_code!
);
```

Transitive dependency: user_id → zip_code → city, state

**Problems:**
- 10,000 users in NYC zip 10001 → "New York" stored 10,000 times
- Someone enters wrong city for a zip → Inconsistent data
- Zip code boundaries change → Update thousands of rows

**Better Design:**
```sql
CREATE TABLE zip_codes (
    zip_code VARCHAR(10) PRIMARY KEY,
    city VARCHAR(100),
    state VARCHAR(50)
);

CREATE TABLE users (
    user_id INT PRIMARY KEY,
    name VARCHAR(100),
    zip_code VARCHAR(10),
    street_address TEXT,
    FOREIGN KEY (zip_code) REFERENCES zip_codes(zip_code)
);
```

---

### **Boyce-Codd Normal Form (BCNF) - Stricter 3NF**

**Rule**: For every dependency X → Y, X must be a superkey.

**When 3NF Isn't Enough:**

**Scenario: University Course Scheduling**
```sql
CREATE TABLE course_schedule (
    student_id INT,
    course VARCHAR(100),
    instructor VARCHAR(100),
    PRIMARY KEY (student_id, course)
);
```

**Business Rules:**
- Each instructor teaches only one course
- Each course can have multiple instructors
- Students can take multiple courses

**Hidden Problem:**
- Dependency: instructor → course
- But `instructor` is not a superkey!

**Example Data:**

| student_id | course | instructor |
|------------|--------|------------|
| 101 | Databases | Dr. Smith |
| 102 | Databases | Dr. Jones |
| 101 | Algorithms | Dr. Smith |

Wait! Dr. Smith teaches both Databases and Algorithms? This violates "instructor teaches one course."

**BCNF Solution:**
```sql
CREATE TABLE instructors (
    instructor VARCHAR(100) PRIMARY KEY,
    course VARCHAR(100)
);

CREATE TABLE enrollments (
    student_id INT,
    instructor VARCHAR(100),
    PRIMARY KEY (student_id, instructor),
    FOREIGN KEY (instructor) REFERENCES instructors(instructor)
);
```

---

### **When to DENORMALIZE - The Real World**

**Normalization is NOT always the answer!**

**Scenario: Social Media Feed (Instagram/Facebook)**

**Fully Normalized:**
```sql
CREATE TABLE posts (
    post_id INT PRIMARY KEY,
    user_id INT,
    content TEXT,
    created_at TIMESTAMP
);

CREATE TABLE users (
    user_id INT PRIMARY KEY,
    username VARCHAR(50),
    profile_pic VARCHAR(200)
);
```

**To show feed:**
```sql
SELECT p.content, u.username, u.profile_pic
FROM posts p
JOIN users u ON p.user_id = u.user_id
WHERE p.user_id IN (SELECT following_id FROM follows WHERE follower_id = 12345)
ORDER BY p.created_at DESC
LIMIT 50;
```

**Problem at Scale:**
- User has 1000 followers
- Fetching feed = 1000 JOINs
- Instagram shows feeds to 500M users daily
- That's 500 billion JOINs per day!

**Denormalized Solution:**
```sql
CREATE TABLE posts (
    post_id INT PRIMARY KEY,
    user_id INT,
    username VARCHAR(50),        -- Denormalized!
    profile_pic VARCHAR(200),    -- Denormalized!
    content TEXT,
    created_at TIMESTAMP
);
```

**Trade-offs:**
- ✓ Feed query: Single table scan (100x faster)
- ✓ No JOINs needed
- ✗ If user changes username → Update millions of posts (handled by background jobs)
- ✗ Storage overhead (username stored millions of times)

**When Instagram chooses this:**
- Reads >>> Writes (people view feeds far more than change usernames)
- Speed matters more than storage (storage is cheap, user time is not)

---

### **Denormalization Patterns in Production**

**1. Aggregation Tables**

**Problem:** "Show total sales per product" query runs on 100M order_items rows every time.
```sql
-- Normalized (slow query)
SELECT product_id, SUM(quantity * price) 
FROM order_items 
GROUP BY product_id;
-- Takes 30 seconds on 100M rows
```

**Solution:** Pre-computed aggregation table
```sql
CREATE TABLE product_sales_summary (
    product_id INT PRIMARY KEY,
    total_quantity BIGINT,
    total_revenue DECIMAL,
    last_updated TIMESTAMP
);

-- Updated via trigger or cron job
-- Now query takes 0.01 seconds on 10K products
```

**2. Snapshot Data**

**E-commerce Order History:**
```sql
-- Don't reference current product price!
CREATE TABLE order_items (
    order_id INT,
    product_id INT,
    quantity INT,
    price_at_purchase DECIMAL,  -- Snapshot! Denormalized from products table
    product_name VARCHAR(200)   -- Snapshot! So you have history even if product deleted
);
```

**Why:** 
- You bought a laptop for $1200 in 2020
- Today it costs $800
- Your invoice must show $1200 (what you paid)
- If product is discontinued, you still see what you bought

---

### **Real System Design: Twitter's Architecture**

**Hybrid Approach:**

**Normalized (for data integrity):**
```sql
users (user_id, username, email, password_hash)
tweets (tweet_id, user_id, content, created_at)
follows (follower_id, following_id)
```

**Denormalized (for performance):**
```sql
-- Timeline cache (Redis/Memcached)
user:12345:timeline → [tweet_id list with user metadata]

-- Denormalized tweet object
{
    tweet_id: 789,
    content: "Hello world",
    username: "john_doe",      // Denormalized from users
    profile_pic: "url",        // Denormalized from users
    like_count: 150,           // Denormalized aggregate
    created_at: "2026-02-09"
}
```

**Strategy:**
1. Write to normalized tables (source of truth)
2. Update denormalized caches asynchronously
3. If cache miss, query normalized tables (slow but correct)

---

## **Database Constraints - The Rules Enforcement**

### **Why Constraints Exist**

**Without constraints, your application enforces rules:**
```javascript
// Application code
if (user.age < 18) {
    throw new Error("Must be 18+");
}
await db.query("INSERT INTO users ...");
```

**Problems:**
1. Multiple applications (web, mobile, API) → Must duplicate validation
2. Direct database access (admin tools, scripts) → Bypasses validation
3. Race conditions → Two requests insert at same time, both pass check
4. Developer forgets → Data corruption

**With constraints, database enforces rules:**
```sql
CREATE TABLE users (
    user_id INT PRIMARY KEY,
    age INT CHECK (age >= 18)  -- Database enforces, always!
);
```

---

### **PRIMARY KEY - The Identity Guarantee**

**What It Guarantees:**
1. UNIQUE: No two rows have same value
2. NOT NULL: Every row must have a value
3. One per table (logical, can be composite)

**Internal Mechanism:**

When you create PRIMARY KEY, PostgreSQL automatically:
1. Creates a UNIQUE B-tree index
2. Adds NOT NULL constraint
3. Uses this index for foreign key lookups

**Choosing Primary Keys - Critical Decision**

**Option 1: Natural Key (Business Data)**
```sql
CREATE TABLE users (
    email VARCHAR(100) PRIMARY KEY
);
```

**Problems:**
- Email can change (divorce, company change)
- Updating PK requires updating all foreign keys (cascading updates expensive)
- Longer key = larger indexes = slower joins
- Privacy concerns (email exposed in URLs)

**Option 2: Surrogate Key (Generated ID)**
```sql
CREATE TABLE users (
    user_id SERIAL PRIMARY KEY,
    email VARCHAR(100) UNIQUE NOT NULL
);
```

**Benefits:**
- Never changes (stability)
- Small integer (4-8 bytes vs. 100 bytes)
- Faster joins and indexes
- Hidden business logic (user_id=12345 reveals nothing)

**Real Example: GitHub**

URLs: `github.com/users/12345` not `github.com/users/john.doe@email.com`

Why?
- User changes username → URL still works
- Privacy (email not exposed)
- Simpler routing

**Composite Primary Keys - When Multiple Columns Define Uniqueness**
```sql
CREATE TABLE order_items (
    order_id INT,
    product_id INT,
    quantity INT,
    PRIMARY KEY (order_id, product_id)
);
```

**Meaning:** 
- Same order can have multiple products ✓
- Same product can be in multiple orders ✓
- Same product CANNOT appear twice in same order ✓

**Internal Impact:**
- Index on (order_id, product_id) created
- Fast lookups for "give me all items in order 123"
- Fast lookups for "is product 456 in order 123?"
- Slower lookups for "all orders containing product 456" (index not optimal)

---

### **FOREIGN KEY - The Relationship Enforcer**

**What It Prevents:**
```sql
-- Without FK
INSERT INTO orders (order_id, customer_id) VALUES (1, 9999);
-- Succeeds even if customer 9999 doesn't exist! (Orphan row)

-- With FK
CREATE TABLE orders (
    order_id INT PRIMARY KEY,
    customer_id INT,
    FOREIGN KEY (customer_id) REFERENCES customers(customer_id)
);

INSERT INTO orders (order_id, customer_id) VALUES (1, 9999);
-- ERROR: violates foreign key constraint
-- Customer 9999 doesn't exist
```

**Referential Integrity Actions:**

**1. ON DELETE CASCADE**
```sql
CREATE TABLE orders (
    order_id INT PRIMARY KEY,
    customer_id INT,
    FOREIGN KEY (customer_id) REFERENCES customers(customer_id) ON DELETE CASCADE
);
```

**Behavior:**
- Delete customer → All their orders automatically deleted
- Use case: User deletes account, remove all related data

**Danger:**
- Accidentally delete customer → Lose all order history (might violate regulations!)

**2. ON DELETE SET NULL**
```sql
CREATE TABLE posts (
    post_id INT PRIMARY KEY,
    author_id INT,
    FOREIGN KEY (author_id) REFERENCES users(user_id) ON DELETE SET NULL
);
```

**Behavior:**
- Delete user → Their posts remain, author_id becomes NULL
- Use case: Reddit (deleted user's posts say "[deleted]")

**3. ON DELETE RESTRICT (Default)**
```sql
FOREIGN KEY (customer_id) REFERENCES customers(customer_id) ON DELETE RESTRICT
```

**Behavior:**
- Cannot delete customer if they have orders
- Must delete orders first, then customer
- Use case: Accounting systems (preserve audit trail)

**Real Scenario: E-commerce System**
```sql
-- Users can be deleted, but keep order history for compliance
CREATE TABLE orders (
    order_id INT PRIMARY KEY,
    customer_id INT,
    customer_name VARCHAR(100),  -- Denormalized snapshot
    customer_email VARCHAR(100), -- Denormalized snapshot
    FOREIGN KEY (customer_id) REFERENCES customers(customer_id) ON DELETE SET NULL
);
```

**Result:**
- User deletes account → customer_id becomes NULL
- Order history preserved with snapshot data
- Compliance with "right to be forgotten" + financial record retention

---

### **CHECK Constraint - Business Logic in Database**

**Simple Checks:**
```sql
CREATE TABLE products (
    product_id INT PRIMARY KEY,
    price DECIMAL CHECK (price >= 0),
    stock_quantity INT CHECK (stock_quantity >= 0),
    discount_percent INT CHECK (discount_percent BETWEEN 0 AND 100)
);
```

**Complex Business Rules:**
```sql
CREATE TABLE employees (
    employee_id INT PRIMARY KEY,
    birth_date DATE,
    hire_date DATE,
    CHECK (hire_date >= birth_date + INTERVAL '18 years')  -- Must be 18+ when hired
);
```

**Multi-Column Checks:**
```sql
CREATE TABLE hotel_bookings (
    booking_id INT PRIMARY KEY,
    check_in DATE,
    check_out DATE,
    CHECK (check_out > check_in)  -- Can't check out before checking in
);
```

**Real Example: Subscription System**
```sql
CREATE TABLE subscriptions (
    subscription_id INT PRIMARY KEY,
    user_id INT,
    plan_type VARCHAR(20) CHECK (plan_type IN ('free', 'premium', 'enterprise')),
    max_projects INT,
    CHECK (
        (plan_type = 'free' AND max_projects <= 3) OR
        (plan_type = 'premium' AND max_projects <= 10) OR
        (plan_type = 'enterprise' AND max_projects IS NULL)  -- Unlimited
    )
);
```

**Why This Matters:**
- Application bug can't create "free" plan with 100 projects
- Database ensures consistency even with direct SQL access
- Multiple applications (web, mobile, API) all respect same rules

---

### **UNIQUE Constraint - No Duplicates Allowed**

**Simple Unique:**
```sql
CREATE TABLE users (
    user_id SERIAL PRIMARY KEY,
    email VARCHAR(100) UNIQUE,
    username VARCHAR(50) UNIQUE
);
```

**Internal Mechanism:**
- PostgreSQL creates a B-tree index
- Before INSERT/UPDATE, checks index for duplicate
- If found, rejects with error

**Composite Unique:**
```sql
CREATE TABLE class_enrollments (
    student_id INT,
    class_id INT,
    semester VARCHAR(20),
    UNIQUE (student_id, class_id, semester)
);
```

**Meaning:**
- Student can't enroll in same class twice in same semester ✓
- Same student can take same class in different semesters ✓

**Real Scenario: Social Media**
```sql
CREATE TABLE user_follows (
    follower_id INT,
    following_id INT,
    followed_at TIMESTAMP DEFAULT NOW(),
    UNIQUE (follower_id, following_id),
    CHECK (follower_id != following_id)  -- Can't follow yourself
);
```

**Prevents:**
- Following same person multiple times
- Following yourself

---

### **NOT NULL - The Silent Hero**

**Why It Matters:**
```sql
-- Bad: Allows NULL
CREATE TABLE employees (
    employee_id INT PRIMARY KEY,
    name VARCHAR(100),
    email VARCHAR(100)
);

INSERT INTO employees (employee_id) VALUES (1);
-- Succeeds with name=NULL, email=NULL (disaster!)
```

**NULL is NOT empty string:**
```sql
SELECT * FROM employees WHERE name = '';      -- Finds empty strings
SELECT * FROM employees WHERE name IS NULL;   -- Finds NULLs
SELECT * FROM employees WHERE name = NULL;    -- Finds NOTHING! (NULL != NULL)
```

**The NULL Problem:**
```sql
SELECT AVG(salary) FROM employees;
-- Ignores NULL salaries (might give wrong impression)

SELECT COUNT(*) FROM employees WHERE manager_id = 5;
-- Doesn't count employees with manager_id=NULL
```

**Proper Design:**
```sql
CREATE TABLE employees (
    employee_id INT PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    email VARCHAR(100) NOT NULL,
    middle_name VARCHAR(100),      -- OK to be NULL (optional)
    manager_id INT                 -- NULL means top-level employee
);
```

**Three-Valued Logic:**
- TRUE
- FALSE  
- UNKNOWN (any comparison with NULL)
```sql
WHERE age > 30  -- If age is NULL, result is UNKNOWN (treated as FALSE in WHERE)
```

---

### **DEFAULT Constraint - Sensible Fallbacks**
```sql
CREATE TABLE posts (
    post_id SERIAL PRIMARY KEY,
    content TEXT NOT NULL,
    created_at TIMESTAMP DEFAULT NOW(),
    is_published BOOLEAN DEFAULT FALSE,
    view_count INT DEFAULT 0,
    status VARCHAR(20) DEFAULT 'draft'
);
```

**Why This Matters:**

Without defaults:
```sql
INSERT INTO posts (post_id, content) VALUES (1, 'Hello');
-- created_at = NULL, is_published = NULL, view_count = NULL (bad!)
```

With defaults:
```sql
INSERT INTO posts (post_id, content) VALUES (1, 'Hello');
-- created_at = current timestamp, is_published = FALSE, view_count = 0 (good!)
```

**Real Scenario: Audit Trail**
```sql
CREATE TABLE user_actions (
    action_id SERIAL PRIMARY KEY,
    user_id INT NOT NULL,
    action_type VARCHAR(50) NOT NULL,
    timestamp TIMESTAMP DEFAULT NOW(),
    ip_address INET DEFAULT '0.0.0.0'::inet,
    user_agent TEXT DEFAULT 'unknown'
);
```

**Benefits:**
- Application doesn't need to provide timestamp (database handles it)
- Consistent timestamps (no clock skew between app servers)
- Can't forget to log when action occurred

---

### **Constraint Strategy: Defensive Database Design**

**Principle:** Database should be impossible to corrupt, even with buggy application code.

**Example: Banking System**
```sql
CREATE TABLE accounts (
    account_id SERIAL PRIMARY KEY,
    user_id INT NOT NULL,
    balance DECIMAL(15,2) NOT NULL DEFAULT 0,
    account_type VARCHAR(20) NOT NULL CHECK (account_type IN ('checking', 'savings')),
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    is_active BOOLEAN NOT NULL DEFAULT TRUE,
    overdraft_limit DECIMAL(15,2) DEFAULT 0,
    
    CONSTRAINT positive_balance CHECK (balance >= -overdraft_limit),
    CONSTRAINT valid_overdraft CHECK (overdraft_limit >= 0),
    
    FOREIGN KEY (user_id) REFERENCES users(user_id) ON DELETE RESTRICT
);

CREATE TABLE transactions (
    transaction_id SERIAL PRIMARY KEY,
    from_account_id INT,
    to_account_id INT,
    amount DECIMAL(15,2) NOT NULL,
    transaction_type VARCHAR(20) NOT NULL,
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    
    CHECK (amount > 0),
    CHECK (from_account_id != to_account_id),
    CHECK (
        (transaction_type = 'transfer' AND from_account_id IS NOT NULL AND to_account_id IS NOT NULL) OR
        (transaction_type = 'deposit' AND from_account_id IS NULL AND to_account_id IS NOT NULL) OR
        (transaction_type = 'withdrawal' AND from_account_id IS NOT NULL AND to_account_id IS NULL)
    ),
    
    FOREIGN KEY (from_account_id) REFERENCES accounts(account_id),
    FOREIGN KEY (to_account_id) REFERENCES accounts(account_id)
);
```

**What This Prevents:**
- Negative transfers ✓
- Transferring to same account ✓
- Balance below overdraft limit ✓
- Account without user ✓
- Deleting user with active accounts ✓
- Invalid transaction types ✓

**Performance Impact of Constraints:**

**Overhead:**
- CHECK: Evaluated on every INSERT/UPDATE (negligible for simple checks)
- FOREIGN KEY: Index lookup on referenced table (logarithmic cost)
- UNIQUE: Index lookup (logarithmic cost)

**Benefits:**
- Prevents data corruption (saves hours of debugging)
- Indexes created for FK/UNIQUE speed up queries
- Application code simpler (less validation needed)

**When to Skip Constraints:**

**Bulk Loading:**
```sql
-- Temporarily disable constraints for massive import
ALTER TABLE orders DISABLE TRIGGER ALL;
-- Load 10 million rows
COPY orders FROM 'data.csv';
-- Re-enable and validate
ALTER TABLE orders ENABLE TRIGGER ALL;
```

**High-Throughput Systems:**
- Sometimes foreign keys are managed by application
- Trade-off: Speed vs. integrity
- Example: Logging systems (millions of inserts/sec, can't afford FK lookups)

---

# **SECTION 3: TRANSACTION MANAGEMENT**

## **Transactions - The "All or Nothing" Guarantee**

### **The Problem**

**Scenario: Bank Transfer $100 from Account A to Account B**
```sql
-- Step 1: Deduct from A
UPDATE accounts SET balance = balance - 100 WHERE account_id = 'A';
-- Step 2: Add to B
UPDATE accounts SET balance = balance + 100 WHERE account_id = 'B';
```

**What if:**
- Server crashes between Step 1 and Step 2? → Money disappears!
- Network fails after Step 1? → Account A debited, B never credited!
- Someone queries balances between steps? → See inconsistent state!

**Transaction Solution:**
```sql
BEGIN;
    UPDATE accounts SET balance = balance - 100 WHERE account_id = 'A';
    UPDATE accounts SET balance = balance + 100 WHERE account_id = 'B';
COMMIT;
```

**Guarantees:**
- Both happen or neither happens
- No one sees intermediate state
- If crash occurs, entire transaction rolled back

---

## **ACID Properties - The Four Pillars**

### **Atomicity - All or Nothing**
```sql
BEGIN;
    INSERT INTO orders (order_id, customer_id) VALUES (1, 100);
    INSERT INTO order_items (order_id, product_id) VALUES (1, 500);
    INSERT INTO order_items (order_id, product_id) VALUES (1, 501); -- Error! Product doesn't exist
ROLLBACK; -- Entire transaction undone, order not created
```

**Internal Mechanism (WAL - Write-Ahead Logging):**
1. Transaction starts → PostgreSQL writes changes to WAL file
2. Commit → WAL marked as committed, changes applied to data files
3. Crash before commit → On restart, PostgreSQL reads WAL, sees uncommitted transaction, undoes it

**Real Example: E-commerce Order**
- Create order record
- Deduct inventory
- Create shipment record
- Charge credit card

If ANY step fails → Roll back ALL steps (inventory not deducted, order not created)

---

### **Consistency - Valid State to Valid State**

**Rule:** Database moves from one valid state to another, respecting all constraints.
```sql
BEGIN;
    UPDATE accounts SET balance = balance - 100 WHERE account_id = 'A';
    -- At this moment, total money in system decreased by 100 (inconsistent!)
    UPDATE accounts SET balance = balance + 100 WHERE account_id = 'B';
    -- Now total money is correct again (consistent!)
COMMIT;
```

**During transaction:** Constraints can be temporarily violated
**After commit:** All constraints MUST be satisfied

**Example:**
```sql
CREATE TABLE accounts (
    balance DECIMAL CHECK (balance >= 0)
);

BEGIN;
    UPDATE accounts SET balance = balance - 100 WHERE account_id = 'A';
    -- If A's balance would go negative, transaction aborts here
COMMIT;
```

---

### **Isolation - Concurrent Transactions Don't Interfere**

**Scenario: Two users booking last seat on flight**

**Without Isolation:**
```sql
-- User 1                          -- User 2
SELECT seats FROM flights WHERE id=100;  (reads: 1 seat available)
                                   SELECT seats FROM flights WHERE id=100; (reads: 1 seat available)
UPDATE flights SET seats=0;
                                   UPDATE flights SET seats=0;
-- Both think they booked! Overbooking!
```

**With Isolation:**
```sql
-- User 1 (Transaction 1)
BEGIN;
SELECT seats FROM flights WHERE id=100 FOR UPDATE; -- Locks row
UPDATE flights SET seats = 0;
COMMIT; -- Releases lock

-- User 2 (Transaction 2)
BEGIN;
SELECT seats FROM flights WHERE id=100 FOR UPDATE; -- Waits for T1 to finish
-- After T1 commits, sees seats=0
-- Booking fails (correct!)
```

**Isolation Levels (Trade-off: Consistency vs Performance):**

**1. READ UNCOMMITTED (Dirty Reads)**
- Can see uncommitted changes from other transactions
- Fastest, least consistent
- Rarely used

**2. READ COMMITTED (Default in PostgreSQL)**
```sql
-- Transaction 1
BEGIN;
UPDATE products SET price = 100 WHERE id = 5;
-- Not committed yet

-- Transaction 2
SELECT price FROM products WHERE id = 5;
-- Sees old price (95), not uncommitted 100
```

**Problem it solves:** No dirty reads
**Problem it has:** Non-repeatable reads
```sql
-- Transaction A
BEGIN;
SELECT price FROM products WHERE id = 5; -- Returns 95
-- Transaction B changes price to 100 and commits
SELECT price FROM products WHERE id = 5; -- Returns 100 (different!)
COMMIT;
```

**3. REPEATABLE READ**
```sql
SET TRANSACTION ISOLATION LEVEL REPEATABLE READ;
BEGIN;
SELECT price FROM products WHERE id = 5; -- Returns 95
-- Transaction B changes price to 100 and commits
SELECT price FROM products WHERE id = 5; -- Still returns 95 (snapshot isolation)
COMMIT;
```

**Problem it solves:** Repeatable reads
**Problem it has:** Phantom reads
```sql
-- Transaction A
BEGIN;
SELECT COUNT(*) FROM orders WHERE status = 'pending'; -- Returns 10
-- Transaction B inserts new pending order
SELECT COUNT(*) FROM orders WHERE status = 'pending'; -- Returns 11 (phantom!)
```

**4. SERIALIZABLE (Strictest)**
- Transactions execute as if sequential (no concurrency)
- Slowest, most consistent
- Used for critical financial operations
```sql
SET TRANSACTION ISOLATION LEVEL SERIALIZABLE;
BEGIN;
SELECT SUM(balance) FROM accounts; -- Total: $10,000
-- Another transaction tries to insert/update accounts
-- If it would change the SUM, it's blocked or aborted
COMMIT;
```

**Real-World Choice:**

- **Banking transfers:** SERIALIZABLE
- **Social media feeds:** READ COMMITTED
- **Analytics dashboards:** READ COMMITTED (or even READ UNCOMMITTED for speed)
- **Inventory management:** REPEATABLE READ or SERIALIZABLE

---

### **Durability - Committed Means Permanent**

**Guarantee:** Once COMMIT returns, data survives crashes, power failures, etc.

**How PostgreSQL Ensures This:**

1. **Write-Ahead Log (WAL):**
```
Transaction commits → First written to WAL → Then to disk
If crash → Replay WAL on restart → Recover committed data
```

2. **fsync:**
```sql
-- Before returning from COMMIT
fsync(wal_file); -- Force write to physical disk, not just OS cache
```

**Trade-off:**
- fsync is slow (disk I/O bottleneck)
- Can disable for speed (dangerous!)
```sql
-- Dangerous: Fast but not durable
ALTER SYSTEM SET fsync = off;
```

**Real Scenario: High-Throughput Logging**
```sql
-- Option 1: Durable but slow (100 writes/sec)
INSERT INTO logs VALUES (...);
COMMIT;

-- Option 2: Fast but risky (10,000 writes/sec)
BEGIN;
INSERT INTO logs VALUES (...);
INSERT INTO logs VALUES (...);
-- ... batch 1000 rows
COMMIT; -- One fsync for 1000 rows
```

---

## **Concurrency Control - Making Isolation Work**

### **Locking Mechanisms**

**1. Shared Lock (S-Lock) - Readers Don't Block Readers**
```sql
BEGIN;
SELECT * FROM products WHERE id = 5; -- Acquires S-lock
-- Other transactions can also SELECT (acquire S-lock)
-- But UPDATE/DELETE must wait
```

**2. Exclusive Lock (X-Lock) - Writers Block Everyone**
```sql
BEGIN;
UPDATE products SET stock = 0 WHERE id = 5; -- Acquires X-lock
-- All other SELECT/UPDATE/DELETE on this row must wait
COMMIT; -- Releases lock
```

**Lock Compatibility Matrix:**

|          | S-Lock | X-Lock |
|----------|--------|--------|
| S-Lock   | ✓      | ✗      |
| X-Lock   | ✗      | ✗      |

**3. Row-Level vs Table-Level Locks**
```sql
-- Row-level (fine-grained, better concurrency)
UPDATE products SET stock = stock - 1 WHERE id = 5;
-- Only row with id=5 is locked

-- Table-level (coarse-grained, worse concurrency)
LOCK TABLE products IN EXCLUSIVE MODE;
UPDATE products SET stock = stock - 1;
-- Entire table locked, blocks all other transactions
```

**When PostgreSQL Uses Table Locks:**
- ALTER TABLE
- CREATE INDEX (blocks writes)
- TRUNCATE
- VACUUM FULL

---

### **Deadlock - The Standoff**

**Classic Deadlock:**
```sql
-- Transaction 1                    -- Transaction 2
BEGIN;                              BEGIN;
UPDATE accounts SET balance=...    UPDATE accounts SET balance=...
  WHERE id = 'A';                     WHERE id = 'B';
  -- Locks A                          -- Locks B
  
UPDATE accounts SET balance=...    UPDATE accounts SET balance=...
  WHERE id = 'B';                     WHERE id = 'A';
  -- Waits for T2 to release B       -- Waits for T1 to release A
  
-- DEADLOCK! Both waiting forever
```

**PostgreSQL Deadlock Detection:**
- Every second, checks for cycles in wait-for graph
- Aborts one transaction (victim)
- Error: `deadlock detected`

**Preventing Deadlocks:**

**Rule 1: Always acquire locks in same order**
```sql
-- Both transactions lock in alphabetical order
BEGIN;
UPDATE accounts SET balance=... WHERE id = 'A'; -- Lock A first
UPDATE accounts SET balance=... WHERE id = 'B'; -- Then B
COMMIT;
```

**Rule 2: Keep transactions short**
```sql
-- Bad: Long-running transaction holds locks
BEGIN;
UPDATE products SET stock = stock - 1 WHERE id = 5;
-- ... expensive computation (10 seconds)
-- ... external API call (5 seconds)
COMMIT;

-- Good: Minimize lock hold time
-- Do computation first
result = expensive_computation();
-- Then quick transaction
BEGIN;
UPDATE products SET stock = stock - 1 WHERE id = 5;
INSERT INTO logs VALUES (result);
COMMIT;
```

**Rule 3: Use SELECT ... FOR UPDATE carefully**
```sql
-- Explicitly lock rows in predictable order
BEGIN;
SELECT * FROM accounts WHERE id IN ('A', 'B') ORDER BY id FOR UPDATE;
-- Locks acquired in sorted order (A, then B)
UPDATE ...
COMMIT;
```

---

### **MVCC - Multi-Version Concurrency Control**

**PostgreSQL's Secret Weapon: Readers Never Block Writers**

**Traditional Locking:**
```
Writer locks row → Readers must wait → Low concurrency
```

**MVCC:**
```
Writer creates new version of row → Readers see old version → High concurrency
```

**How It Works:**
```sql
-- Initial state
products: (id=5, price=100, xmin=1000, xmax=infinity)

-- Transaction 2000 updates
BEGIN; -- Transaction ID = 2000
UPDATE products SET price = 95 WHERE id = 5;
-- Creates new version: (id=5, price=95, xmin=2000, xmax=infinity)
-- Marks old version: (id=5, price=100, xmin=1000, xmax=2000)
COMMIT;
```

**Now two versions exist:**
- Old: `(id=5, price=100, xmin=1000, xmax=2000)`
- New: `(id=5, price=95, xmin=2000, xmax=infinity)`

**Concurrent Read:**
```sql
-- Transaction 1500 started before update
SELECT price FROM products WHERE id = 5;
-- Sees version where xmin <= 1500 < xmax
-- Returns price=100 (old version)

-- Transaction 2500 started after update
SELECT price FROM products WHERE id = 5;
-- Sees version where xmin <= 2500
-- Returns price=95 (new version)
```

**Benefits:**
- Readers never wait for writers
- Writers never wait for readers
- Each transaction sees consistent snapshot

**Cost:**
- Multiple versions consume storage
- VACUUM needed to clean up old versions
```sql
VACUUM products; -- Removes dead row versions
```

---

### **Real-World Transaction Patterns**

**1. E-commerce Order Processing**
```sql
BEGIN;

-- Reserve inventory (pessimistic locking)
UPDATE products 
SET stock_quantity = stock_quantity - 1 
WHERE product_id = 100 AND stock_quantity > 0;

-- Check if update succeeded
IF (ROW_COUNT = 0) THEN
    ROLLBACK;
    RAISE EXCEPTION 'Out of stock';
END IF;

-- Create order
INSERT INTO orders (customer_id, total) VALUES (500, 99.99);

-- Create order item
INSERT INTO order_items (order_id, product_id, quantity) 
VALUES (currval('orders_order_id_seq'), 100, 1);

COMMIT;
```

**2. Banking Transfer (Two-Phase)**
```sql
BEGIN;

-- Phase 1: Validate
SELECT balance FROM accounts WHERE id = 'A' FOR UPDATE;
IF (balance < 100) THEN
    ROLLBACK;
    RAISE EXCEPTION 'Insufficient funds';
END IF;

-- Phase 2: Execute
UPDATE accounts SET balance = balance - 100 WHERE id = 'A';
UPDATE accounts SET balance = balance + 100 WHERE id = 'B';

-- Audit trail
INSERT INTO transactions (from_account, to_account, amount) 
VALUES ('A', 'B', 100);

COMMIT;
```

**3. Ticket Booking (Optimistic Locking)**
```sql
-- Read current version
SELECT seats_available, version FROM events WHERE id = 100;
-- seats=10, version=5

-- User takes time to fill form (30 seconds)

-- Attempt booking with version check
UPDATE events 
SET seats_available = seats_available - 1,
    version = version + 1
WHERE id = 100 AND version = 5; -- Only succeeds if version unchanged

IF (ROW_COUNT = 0) THEN
    -- Someone else booked in meantime
    RAISE EXCEPTION 'Seats sold out, please try again';
END IF;
```

---

### **Transaction Performance Tips**

**1. Keep Transactions Short**
```sql
-- Bad: Hold lock during external call
BEGIN;
UPDATE inventory SET reserved = reserved + 1 WHERE product_id = 5;
call_payment_gateway(); -- 5 seconds
COMMIT;

-- Good: Release lock quickly
BEGIN;
UPDATE inventory SET reserved = reserved + 1 WHERE product_id = 5;
COMMIT;
call_payment_gateway();
```

**2. Batch Operations**
```sql
-- Bad: 1000 transactions
FOR i IN 1..1000 LOOP
    BEGIN;
    INSERT INTO logs VALUES (...);
    COMMIT;
END LOOP;

-- Good: 1 transaction
BEGIN;
FOR i IN 1..1000 LOOP
    INSERT INTO logs VALUES (...);
END LOOP;
COMMIT;
```

**3. Use Appropriate Isolation Level**
```sql
-- For reports (read-only, speed matters)
SET TRANSACTION ISOLATION LEVEL READ COMMITTED;

-- For financial operations (accuracy critical)
SET TRANSACTION ISOLATION LEVEL SERIALIZABLE;
```

**4. Explicit Locking When Needed**
```sql
-- Prevents lost updates
BEGIN;
SELECT stock FROM products WHERE id = 5 FOR UPDATE; -- Lock immediately
-- ... application logic
UPDATE products SET stock = calculated_value WHERE id = 5;
COMMIT;
```

---

# **SECTION 4: STORAGE & INDEXING**

## **File Organization - How Data Lives on Disk**

### **The Reality: Disk is Slow, RAM is Fast**

- RAM access: 100 nanoseconds
- SSD access: 100 microseconds (1000x slower)
- HDD access: 10 milliseconds (100,000x slower)

**Implication:** Minimize disk reads at all costs!

---

### **Heap File Organization (PostgreSQL Default)**

**Structure:**
```
Table "users" → Collection of 8KB pages (blocks)

Page 1: [Row1][Row2][Row3]...
Page 2: [Row50][Row51]...
Page N: [Row900][Row901]...
```

**INSERT Behavior:**
```sql
INSERT INTO users (name, email) VALUES ('Alice', 'alice@email.com');
```
1. Find page with free space (usually last page)
2. Append row to that page
3. If page full, allocate new page

**No ordering maintained!**

**Query Without Index:**
```sql
SELECT * FROM users WHERE email = 'alice@email.com';
```

**Execution:**
1. Read Page 1 → Check every row → Not found
2. Read Page 2 → Check every row → Not found
3. ...
4. Read Page N → Found!

**Cost:** N disk reads (sequential scan)

**With 1M rows in 8KB pages:**
- ~125 rows per page
- ~8000 pages
- Query reads all 8000 pages (64MB of data!)

---

## **Indexing - The Game Changer**

### **What Is an Index?**

**Analogy:** Book index at the back

- Book without index: Read every page to find "PostgreSQL" mention
- Book with index: Look up "PostgreSQL" → page 145, 289, 400

**Database Index:** Separate data structure that maintains sorted pointers to table rows.

---

### **B-Tree Index - The Workhorse**

**Why B-Tree (Not Binary Tree)?**

**Binary tree:** Each node has 2 children → Tall tree → Many disk reads
**B-tree:** Each node has 100-1000 children → Short tree → Few disk reads

**Structure:**
```
                    [50]
                   /    \
              [20,30]    [70,90]
              /  |  \     /  |  \
          [5-15][21-25][31-40][55-65][71-85][91-99]
             ↓      ↓      ↓      ↓      ↓      ↓
          Heap   Heap   Heap   Heap   Heap   Heap
          Rows   Rows   Rows   Rows   Rows   Rows
```

**Leaf nodes:** Contain actual values and pointers to table rows
**Internal nodes:** Guide search (like road signs)

**Search for email = 'alice@email.com':**
```sql
CREATE INDEX idx_email ON users(email);
SELECT * FROM users WHERE email = 'alice@email.com';
```

**Execution:**
1. Start at root: Compare 'alice' with node values → Go left/right
2. Traverse 2-3 internal nodes (disk reads: 2-3)
3. Reach leaf node with 'alice@email.com'
4. Follow pointer to heap (disk read: 1)

**Total: 3-4 disk reads (vs 8000 without index!)**

**B-Tree Height:**
- 1M rows, fanout=200 → Height = log₂₀₀(1M) ≈ 3
- 1B rows, fanout=200 → Height = log₂₀₀(1B) ≈ 4

**Always 3-4 disk reads regardless of table size!**

---

### **B+ Tree (PostgreSQL Actually Uses This)**

**Difference from B-Tree:**
- **All data in leaf nodes** (internal nodes only have keys)
- **Leaf nodes linked together** (doubly-linked list)
```
Internal:       [50, 75]
               /    |    \
Leaf:    [20,30]→[50,60]→[75,80,90]
          ↓  ↓    ↓  ↓    ↓  ↓  ↓
        Data Data Data Data Data Data Data
```

**Benefits:**

**1. Range Queries Are Fast:**
```sql
SELECT * FROM users WHERE age BETWEEN 25 AND 35;
```

- Find first value (age=25) using tree traversal
- Follow linked list in leaf level until age>35
- No need to go back to root!

**2. Full Table Scan via Index:**
```sql
SELECT email FROM users ORDER BY email;
```

- Traverse to leftmost leaf
- Follow links to read all values in sorted order
- No sorting needed!

---

### **When Indexes Help**

**1. WHERE Clauses:**
```sql
-- Index on email helps
SELECT * FROM users WHERE email = 'alice@email.com';

-- Index on (last_name, first_name) helps
SELECT * FROM users WHERE last_name = 'Smith' AND first_name = 'John';

-- Index on age helps
SELECT * FROM users WHERE age > 30;
```

**2. ORDER BY:**
```sql
-- Index on created_at helps (no sorting needed)
SELECT * FROM posts ORDER BY created_at DESC LIMIT 10;
```

**3. JOIN Operations:**
```sql
SELECT o.*, c.name 
FROM orders o 
JOIN customers c ON o.customer_id = c.customer_id;
```

**Without index on customer_id:**
- For each order, scan entire customers table
- Cost: O(orders × customers)

**With index on customer_id:**
- For each order, lookup customer in O(log customers)
- Cost: O(orders × log customers)

---

### **When Indexes DON'T Help**

**1. Low Selectivity Columns:**
```sql
-- Bad index
CREATE INDEX idx_gender ON users(gender); -- Only 2-3 distinct values

SELECT * FROM users WHERE gender = 'M';
-- Returns 50% of rows → Index slower than sequential scan!
```

**Why:** 
- Index: Read index (multiple pages) + Read 50% of heap (random I/O)
- Sequential scan: Read entire heap (sequential I/O, faster)

**Rule:** Index useful when query returns <15% of rows.

**2. Functions in WHERE:**
```sql
CREATE INDEX idx_email ON users(email);

-- Index NOT used
SELECT * FROM users WHERE LOWER(email) = 'alice@email.com';
```

**Solution: Functional Index:**
```sql
CREATE INDEX idx_email_lower ON users(LOWER(email));
-- Now index is used
```

**3. OR Conditions on Different Columns:**
```sql
CREATE INDEX idx_email ON users(email);
CREATE INDEX idx_phone ON users(phone);

-- Both indexes NOT used efficiently
SELECT * FROM users WHERE email = 'alice@email.com' OR phone = '555-0100';
```

**Solution:** Bitmap index scan (PostgreSQL specific) or separate queries with UNION.

---

### **Multi-Column Indexes**

**Scenario:** Queries often filter by country AND city
```sql
-- Single-column indexes
CREATE INDEX idx_country ON users(country);
CREATE INDEX idx_city ON users(city);

-- Multi-column index
CREATE INDEX idx_country_city ON users(country, city);
```

**Query:**
```sql
SELECT * FROM users WHERE country = 'USA' AND city = 'New York';
```

**With idx_country_city:**
1. Navigate to 'USA' section
2. Within 'USA', navigate to 'New York'
3. Efficiently finds exact match

**With separate indexes:**
- Use idx_country → Find all USA users
- Use idx_city → Find all New York users
- Intersect results (expensive!)

**Column Order Matters:**
```sql
CREATE INDEX idx_country_city ON users(country, city);
```

**Works for:**
- `WHERE country = 'USA'` ✓ (uses first column)
- `WHERE country = 'USA' AND city = 'New York'` ✓ (uses both)
- `WHERE city = 'New York'` ✗ (can't use index without first column!)

**Think of it like a phone book:** Sorted by LastName, then FirstName
- Can find "Smith, John" ✓
- Can find all "Smith" ✓
- Can't efficiently find all "John" (any last name) ✗

---

### **Index Maintenance Cost**

**INSERT:**
```sql
INSERT INTO users (name, email, age) VALUES ('Alice', 'alice@email.com', 30);
```

**Without indexes:**
1. Append to heap (1 write)

**With 3 indexes (email, age, name):**
1. Append to heap (1 write)
2. Update idx_email B-tree (2-3 writes)
3. Update idx_age B-tree (2-3 writes)
4. Update idx_name B-tree (2-3 writes)

**Total: 7-10 writes (10x slower!)**

**UPDATE:**
```sql
UPDATE users SET email = 'newemail@email.com' WHERE user_id = 100;
```

- Update heap row
- Update idx_email (delete old entry, insert new)
- All indexes on unchanged columns still valid (no update needed)

**DELETE:**
- Remove from heap
- Remove from all indexes

**Trade-off:**
- Too few indexes → Slow reads
- Too many indexes → Slow writes, wasted storage

---

### **Partial Indexes - Index Only What Matters**

**Scenario:** Query only active users
```sql
-- Full index (wasteful)
CREATE INDEX idx_status ON users(status);
-- Indexes both 'active' and 'deleted'

-- Partial index (efficient)
CREATE INDEX idx_active_users ON users(email) WHERE status = 'active';
-- Indexes only active users
```

**Benefits:**
- Smaller index (faster searches)
- Less maintenance (deleting inactive users doesn't update index)

**Another Example: Recent Data**
```sql
CREATE INDEX idx_recent_orders ON orders(created_at) 
WHERE created_at > NOW() - INTERVAL '30 days';
```

Only indexes last 30 days (90% of queries touch recent data).

---

### **Covering Indexes - Index-Only Scans**

**Problem:**
```sql
SELECT email, name FROM users WHERE email = 'alice@email.com';
```

**With normal index on email:**
1. Search index for 'alice@email.com'
2. Get heap pointer
3. Read heap to get `name` column

**With covering index:**
```sql
CREATE INDEX idx_email_name ON users(email, name);
```

Now index contains both email AND name:
1. Search index for 'alice@email.com'
2. Read `name` from index itself
3. No heap access needed! (faster)

**PostgreSQL Specific: INCLUDE**
```sql
CREATE INDEX idx_email_covering ON users(email) INCLUDE (name, age);
```

- Searches by email
- Returns name and age without heap access
- Included columns not used for sorting/searching (smaller index nodes)

---

## **Index Types in PostgreSQL**

### **1. B-Tree (Default, 95% of Use Cases)**
```sql
CREATE INDEX idx_name ON users(name);
```

**Good for:**
- Equality: `WHERE name = 'Alice'`
- Ranges: `WHERE age BETWEEN 25 AND 35`
- Sorting: `ORDER BY created_at`
- NULL handling

---

### **2. Hash Index**
```sql
CREATE INDEX idx_hash_email ON users USING HASH (email);
```

**Good for:** Equality only (`WHERE email = 'alice@email.com'`)
**Not for:** Ranges, sorting, LIKE

**Rare use case:** Slightly faster than B-tree for exact matches, but limited functionality.

---

### **3. GiN (Generalized Inverted Index) - For Arrays/JSON**

**Scenario: Tag Search**
```sql
CREATE TABLE posts (
    post_id INT,
    tags TEXT[]  -- Array: ['postgresql', 'database', 'sql']
);

CREATE INDEX idx_tags ON posts USING GIN(tags);

-- Fast query
SELECT * FROM posts WHERE tags @> ARRAY['postgresql'];
```

**How GiN Works:**
```
Tag 'postgresql' → [post_1, post_5, post_20]
Tag 'database'   → [post_1, post_8, post_15]
Tag 'sql'        → [post_2, post_5, post_20]
```

Inverted index: Maps each tag to list of posts containing it.

**Also for JSON:**
```sql
CREATE INDEX idx_json ON products USING GIN(attributes);

SELECT * FROM products WHERE attributes @> '{"color": "red"}';
```

---

### **4. GiST (Generalized Search Tree) - For Geometric/Range Types**
```sql
-- Geospatial queries
CREATE INDEX idx_location ON restaurants USING GIST(location);

SELECT * FROM restaurants WHERE location && box '((0,0),(1,1))';

-- Range types
CREATE TABLE reservations (
    room_id INT,
    reserved_period TSRANGE
);

CREATE INDEX idx_period ON reservations USING GIST(reserved_period);

-- Find overlapping reservations
SELECT * FROM reservations 
WHERE reserved_period && '[2026-02-09 14:00, 2026-02-09 16:00)';
```

---

### **5. BRIN (Block Range Index) - For Huge Sequential Tables**

**Scenario: Time-Series Data (Logs, Metrics)**
```sql
CREATE TABLE logs (
    log_id BIGSERIAL,
    created_at TIMESTAMP,
    message TEXT
);

-- Regular B-tree index would be huge (GBs)
-- BRIN index is tiny (KBs)
CREATE INDEX idx_created_at ON logs USING BRIN(created_at);
```

**How BRIN Works:**
- Divides table into blocks (128 pages each)
- Stores min/max value per block
```
Block 1: created_at range [2026-01-01, 2026-01-05]
Block 2: created_at range [2026-01-06, 2026-01-10]
Block 3: created_at range [2026-01-11, 2026-01-15]
```

**Query:**
```sql
SELECT * FROM logs WHERE created_at > '2026-01-08';
```

- Check Block 1: max < 2026-01-08 → Skip
- Check Block 2: range overlaps → Scan
- Check Block 3: min > 2026-01-08 → Scan

**Requirements:**
- Data physically sorted (or mostly sorted)
- Works best for time-series, append-only tables

**Trade-off:**
- Less precise than B-tree (may scan extra blocks)
- 1000x smaller (perfect for massive tables)

---

## **Real-World Indexing Strategy**

### **Scenario: Social Media Platform**

**Users Table: 100M rows**
```sql
CREATE TABLE users (
    user_id BIGSERIAL PRIMARY KEY,  -- Automatic B-tree index
    email VARCHAR(255) NOT NULL,
    username VARCHAR(50) NOT NULL,
    country VARCHAR(2),
    city VARCHAR(100),
    created_at TIMESTAMP,
    last_login TIMESTAMP,
    is_active BOOLEAN
);
```

**Query Patterns:**
1. Login: `WHERE email = ?` (100K/sec)
2. Profile: `WHERE username = ?` (50K/sec)
3. Admin: `WHERE country = ? AND city = ?` (100/sec)
4. Analytics: `WHERE created_at > ?` (10/sec)
5. Cleanup: `WHERE is_active = false AND last_login < ?` (1/hour)

**Indexing Decisions:**
```sql
-- Must have: High-frequency lookups
CREATE UNIQUE INDEX idx_email ON users(email);
CREATE UNIQUE INDEX idx_username ON users(username);

-- Composite for geographic queries
CREATE INDEX idx_country_city ON users(country, city);

-- Partial index for cleanup job
CREATE INDEX idx_inactive_users ON users(last_login) 
WHERE is_active = false;

-- NO index on is_active (low selectivity, rarely queried alone)
-- NO index on created_at (analytics use data warehouse, not prod DB)
```

**Posts Table: 10B rows**
```sql
CREATE TABLE posts (
    post_id BIGSERIAL PRIMARY KEY,
    user_id BIGINT NOT NULL,
    content TEXT,
    created_at TIMESTAMP NOT NULL,
    like_count INT DEFAULT 0
);
```

**Queries:**
1. User feed: `WHERE user_id = ? ORDER BY created_at DESC LIMIT 50`
2. Popular posts: `WHERE created_at > ? ORDER BY like_count DESC LIMIT 100`

**Indexing:**
```sql
-- Composite for user feed (most frequent query)
CREATE INDEX idx_user_posts ON posts(user_id, created_at DESC);

-- BRIN for time-based queries (space-efficient)
CREATE INDEX idx_created_at_brin ON posts USING BRIN(created_at);

-- NO index on like_count (high cardinality, always with date filter)
```

**Why idx_user_posts is (user_id, created_at) not (created_at, user_id)?**

- Query always filters by specific user_id FIRST
- Then sorts within that user's posts
- (user_id, created_at) allows: Find user's posts → Already sorted
- (created_at, user_id) would require: Scan all posts → Filter by user (inefficient)

---

### **Monitoring Index Usage**
```sql
-- Find unused indexes
SELECT schemaname, tablename, indexname, idx_scan
FROM pg_stat_user_indexes
WHERE idx_scan = 0 AND indexrelname NOT LIKE 'pg_toast%';

-- Find duplicate indexes
SELECT array_agg(indexname) as indexes, tablename
FROM pg_indexes
GROUP BY tablename, indexdef
HAVING COUNT(*) > 1;

-- Index size
SELECT indexname, pg_size_pretty(pg_relation_size(indexname::regclass))
FROM pg_indexes
WHERE tablename = 'users';
```

**Decide:**
- Unused index → DROP (saves write overhead)
- Duplicate index → DROP (wasted space)
- Huge index on low-traffic table → Consider partial index

---

# **SECTION 5-7: SQL BASICS & QUERYING**

## **PostgreSQL Setup & Basic Queries**

### **Installation & Connection**
```bash
# Install on Ubuntu
sudo apt install postgresql postgresql-contrib

# Connect
psql -U postgres -h localhost -d mydb

# Load sample database (dvdrental, employees, etc.)
pg_restore -U postgres -d dvdrental dvdrental.tar
```

---

## **SELECT - The Foundation**
```sql
-- Basic
SELECT first_name, last_name FROM employees;

-- All columns (avoid in production - wastes bandwidth)
SELECT * FROM employees;

-- Column aliases
SELECT first_name AS name, salary * 12 AS annual_salary FROM employees;

-- Expressions
SELECT product_name, price, price * 0.9 AS discounted_price FROM products;
```

**Internal Execution:**
1. FROM: Identify table(s)
2. WHERE: Filter rows
3. SELECT: Project columns
4. ORDER BY: Sort results

---

## **ORDER BY - Sorting**
```sql
-- Ascending (default)
SELECT * FROM products ORDER BY price;

-- Descending
SELECT * FROM products ORDER BY price DESC;

-- Multiple columns
SELECT * FROM employees ORDER BY department, salary DESC;
-- First sort by department (A-Z), then within each department by salary (high-low)

-- By expression
SELECT name, price, price * quantity AS total 
FROM order_items 
ORDER BY total DESC;

-- NULL handling
ORDER BY last_login NULLS LAST;  -- NULLs at end
ORDER BY last_login NULLS FIRST; -- NULLs at start
```

**Performance:**
- With index on ORDER BY column → Fast (index already sorted)
- Without index → Full table scan + in-memory sort (slow on large tables)

---

## **DISTINCT - Remove Duplicates**
```sql
-- Get unique cities
SELECT DISTINCT city FROM customers;

-- Multiple columns (unique combinations)
SELECT DISTINCT country, city FROM customers;

-- DISTINCT ON (PostgreSQL specific)
SELECT DISTINCT ON (customer_id) 
    customer_id, order_date, total
FROM orders
ORDER BY customer_id, order_date DESC;
-- Returns most recent order per customer
```

**How it works:**
- Sorts or hashes data
- Scans for duplicates
- Returns only unique rows

**Cost:** O(n log n) for sorting or O(n) with hashing

---

# **SECTION 8: FILTERING DATA**

## **WHERE - Row Filtering**
```sql
-- Equality
SELECT * FROM employees WHERE department = 'Engineering';

-- Comparison
SELECT * FROM products WHERE price > 100;
SELECT * FROM orders WHERE order_date >= '2026-01-01';

-- String patterns
SELECT * FROM users WHERE email LIKE '%@gmail.com';
SELECT * FROM products WHERE name ILIKE '%phone%'; -- Case-insensitive

-- Range
SELECT * FROM employees WHERE salary BETWEEN 50000 AND 100000;
-- Equivalent to: salary >= 50000 AND salary <= 100000

-- Set membership
SELECT * FROM orders WHERE status IN ('pending', 'processing', 'shipped');

-- NULL checks
SELECT * FROM users WHERE phone IS NULL;
SELECT * FROM users WHERE phone IS NOT NULL;
```

**Performance Tips:**
- Use indexes on WHERE columns
- Avoid functions: `WHERE YEAR(created_at) = 2026` → No index
- Better: `WHERE created_at >= '2026-01-01' AND created_at < '2027-01-01'` → Uses index

---

## **Logical Operators**
```sql
-- AND (both conditions must be true)
SELECT * FROM products 
WHERE category = 'Electronics' AND price < 500;

-- OR (either condition must be true)
SELECT * FROM orders 
WHERE status = 'cancelled' OR total_refunded > 0;

-- NOT
SELECT * FROM users WHERE NOT is_active;

-- Combining (use parentheses!)
SELECT * FROM products 
WHERE (category = 'Electronics' OR category = 'Computers')
  AND price < 1000;
```

**Evaluation Order:** NOT → AND → OR
**Best practice:** Always use parentheses for clarity

---

## **LIMIT & FETCH**
```sql
-- LIMIT (PostgreSQL, MySQL)
SELECT * FROM posts ORDER BY created_at DESC LIMIT 10;

-- FETCH (SQL standard)
SELECT * FROM posts ORDER BY created_at DESC FETCH FIRST 10 ROWS ONLY;

-- Pagination (OFFSET)
SELECT * FROM products ORDER BY product_id LIMIT 20 OFFSET 40;
-- Skip first 40 rows, return next 20 (page 3 if 20 per page)
```

**Performance Warning:**
```sql
-- SLOW on page 1000
SELECT * FROM posts ORDER BY created_at DESC LIMIT 20 OFFSET 20000;
-- Database still scans first 20,000 rows!
```

**Better Pagination (Keyset):**
```sql
-- Page 1
SELECT * FROM posts ORDER BY created_at DESC LIMIT 20;
-- Note last created_at = '2026-02-01 10:00:00'

-- Page 2 (use last value as cursor)
SELECT * FROM posts 
WHERE created_at < '2026-02-01 10:00:00'
ORDER BY created_at DESC 
LIMIT 20;
```

---

# **SECTION 9: JOINS**

## **The Problem Joins Solve**

**Scenario:** Orders reference customers by ID
```sql
orders: (order_id, customer_id, total)
customers: (customer_id, name, email)
```

**To show:** "Order #123 placed by John (john@email.com)"

Need to combine data from both tables → **JOIN**

---

## **INNER JOIN - Only Matching Rows**
```sql
SELECT o.order_id, o.total, c.name, c.email
FROM orders o
INNER JOIN customers c ON o.customer_id = c.customer_id;
```

**Result:** Only orders with existing customers (orphan orders excluded)

**Visual:**
```
orders:              customers:
order_id | cust_id  cust_id | name
1        | 100      100     | Alice
2        | 101      101     | Bob
3        | 999      

INNER JOIN result:
order_id | cust_id | name
1        | 100     | Alice
2        | 101     | Bob
(order 3 excluded - no matching customer 999)
```

---

## **LEFT JOIN - Keep All Left Rows**
```sql
SELECT o.order_id, o.total, c.name
FROM orders o
LEFT JOIN customers c ON o.customer_id = c.customer_id;
```

**Result:** All orders, even if customer doesn't exist (NULL for customer columns)
```
order_id | total | name
1        | 100   | Alice
2        | 200   | Bob
3        | 150   | NULL   -- Order exists, customer doesn't
```

**Use case:** "Find orders without customers" (data cleanup)
```sql
SELECT o.order_id 
FROM orders o
LEFT JOIN customers c ON o.customer_id = c.customer_id
WHERE c.customer_id IS NULL;
```

---

## **RIGHT JOIN - Keep All Right Rows**
```sql
SELECT c.name, o.order_id
FROM orders o
RIGHT JOIN customers c ON o.customer_id = c.customer_id;
```

**Result:** All customers, even without orders
```
name  | order_id
Alice | 1
Bob   | 2
Carol | NULL   -- Customer exists, no orders yet
```

**Rarely used** (just swap tables and use LEFT JOIN)

---

## **FULL OUTER JOIN - Keep All Rows from Both**
```sql
SELECT c.name, o.order_id
FROM customers c
FULL OUTER JOIN orders o ON c.customer_id = o.customer_id;
```

**Result:**
```
name  | order_id
Alice | 1
Bob   | 2
Carol | NULL    -- Customer without orders
NULL  | 3       -- Order without customer
```

**Use case:** Data reconciliation (find mismatches)

---

## **SELF JOIN - Table Joins Itself**

**Scenario:** Employees table with manager reference
```sql
CREATE TABLE employees (
    employee_id INT,
    name VARCHAR(100),
    manager_id INT
);
```

**Query:** "Show each employee with their manager's name"
```sql
SELECT e.name AS employee, m.name AS manager
FROM employees e
LEFT JOIN employees m ON e.manager_id = m.employee_id;
```
```
employee | manager
Alice    | NULL     -- Top-level (no manager)
Bob      | Alice
Carol    | Alice
Dave     | Bob
```

**Another Example:** Find all pairs of users in same city
```sql
SELECT u1.name, u2.name, u1.city
FROM users u1
JOIN users u2 ON u1.city = u2.city AND u1.user_id < u2.user_id;
-- Condition prevents duplicate pairs (Alice-Bob, Bob-Alice)
```

---

## **CROSS JOIN - Cartesian Product**
```sql
SELECT s.size, c.color
FROM sizes s
CROSS JOIN colors c;
```

**sizes:** Small, Medium, Large
**colors:** Red, Blue, Green

**Result:** 3 × 3 = 9 combinations
```
size   | color
Small  | Red
Small  | Blue
Small  | Green
Medium | Red
...
```

**Use case:** Generate all combinations (product variants, test data)

---

## **Join Performance - Critical Insights**

### **Join Algorithm Selection**

**1. Nested Loop Join**
```
FOR each row in orders:
    FOR each row in customers:
        IF orders.customer_id = customers.customer_id:
            output row
```
**Cost:** O(n × m)
**Used when:** Small tables or one side heavily filtered

**2. Hash Join**
```
-- Build phase
Build hash table from customers (customer_id → customer_row)

-- Probe phase
FOR each row in orders:
    lookup = hash_table[orders.customer_id]
    output combined row
```
**Cost:** O(n + m)
**Used when:** Large tables, enough memory
**Requirement:** Equality join (=)

**3. Merge Join**
```
Sort orders by customer_id
Sort customers by customer_id
Walk through both sorted lists simultaneously
```
**Cost:** O(n log n + m log m)
**Used when:** Data already sorted (indexes exist)

---

### **Join Optimization Rules**

**1. Filter Early**
```sql
-- BAD: Join then filter
SELECT * FROM orders o
JOIN customers c ON o.customer_id = c.customer_id
WHERE o.order_date > '2026-01-01';

-- GOOD: Filter then join
SELECT * FROM orders o
WHERE o.order_date > '2026-01-01'
JOIN customers c ON o.customer_id = c.customer_id;
```

**2. Join Order Matters**
```sql
-- 1000 orders, 1M customers, 10 order_items per order
SELECT * FROM orders o
JOIN order_items oi ON o.order_id = oi.order_id
JOIN customers c ON o.customer_id = c.customer_id;

-- Better: orders (1000) → order_items (10000) → customers (1000 lookups)
-- Worse: customers (1M) → orders → order_items
```

**3. Index Foreign Keys**
```sql
-- Always index join columns!
CREATE INDEX idx_customer_id ON orders(customer_id);
CREATE INDEX idx_order_id ON order_items(order_id);
```

---

## **Real Scenario: E-commerce Query**

**Show:** Order details with customer info, product names, totals
```sql
SELECT 
    o.order_id,
    c.name AS customer,
    p.product_name,
    oi.quantity,
    oi.price,
    oi.quantity * oi.price AS line_total
FROM orders o
JOIN customers c ON o.customer_id = c.customer_id
JOIN order_items oi ON o.order_id = oi.order_id
JOIN products p ON oi.product_id = p.product_id
WHERE o.order_date >= '2026-01-01'
ORDER BY o.order_id, oi.item_id;
```

**Execution Plan:**
1. Filter orders by date (index scan on order_date)
2. Hash join with customers (small hash table)
3. Hash join with order_items (larger hash table)
4. Hash join with products
5. Sort by order_id, item_id

**With proper indexes:**
- idx_order_date on orders
- idx_customer_id on orders
- idx_order_id on order_items
- idx_product_id on order_items

**Query time:** Milliseconds even with millions of rows

---

# **SECTION 10: AGGREGATION & GROUPING**

## **Aggregate Functions**
```sql
-- COUNT
SELECT COUNT(*) FROM orders; -- Total rows
SELECT COUNT(DISTINCT customer_id) FROM orders; -- Unique customers

-- SUM
SELECT SUM(total) FROM orders; -- Total revenue

-- AVG
SELECT AVG(price) FROM products; -- Average price

-- MIN/MAX
SELECT MIN(order_date), MAX(order_date) FROM orders;
```

**NULL Handling:**
```sql
SELECT COUNT(phone) FROM users; -- Excludes NULLs
SELECT COUNT(*) FROM users;     -- Includes NULLs
SELECT AVG(rating) FROM reviews; -- Ignores NULL ratings
```

---

## **GROUP BY - Aggregation per Group**

**Scenario:** Total sales per customer
```sql
SELECT customer_id, SUM(total) AS total_spent
FROM orders
GROUP BY customer_id;
```

**Result:**
```
customer_id | total_spent
100         | 1500
101         | 3200
102         | 800
```

**Multiple Columns:**
```sql
SELECT country, city, COUNT(*) AS user_count
FROM users
GROUP BY country, city
ORDER BY user_count DESC;
```

**Result:**
```
country | city      | user_count
USA     | New York  | 50000
USA     | LA        | 30000
UK      | London    | 25000
```

---

## **HAVING - Filter After Grouping**

**WHERE** filters rows before grouping
**HAVING** filters groups after aggregation
```sql
-- Find customers who spent >$1000
SELECT customer_id, SUM(total) AS total_spent
FROM orders
GROUP BY customer_id
HAVING SUM(total) > 1000;
```

**Cannot use WHERE:**
```sql
-- WRONG
SELECT customer_id, SUM(total) 
FROM orders
WHERE SUM(total) > 1000  -- ERROR: Cannot use aggregate in WHERE
GROUP BY customer_id;
```

**Combining WHERE and HAVING:**
```sql
-- Sales per product in 2026, only products with >100 sales
SELECT product_id, SUM(quantity) AS total_sold
FROM order_items oi
JOIN orders o ON oi.order_id = o.order_id
WHERE o.order_date >= '2026-01-01'
GROUP BY product_id
HAVING SUM(quantity) > 100;
```

**Execution order:**
1. FROM/JOIN
2. WHERE (filter rows)
3. GROUP BY
4. Aggregation
5. HAVING (filter groups)
6. SELECT
7. ORDER BY

---

## **Advanced Grouping**

### **GROUPING SETS - Multiple GROUP BYs in One Query**
```sql
-- Sales by year, by month, and total
SELECT 
    EXTRACT(YEAR FROM order_date) AS year,
    EXTRACT(MONTH FROM order_date) AS month,
    SUM(total) AS revenue
FROM orders
GROUP BY GROUPING SETS (
    (year, month),  -- Monthly sales
    (year),         -- Yearly sales
    ()              -- Grand total
);
```

**Result:**
```
year | month | revenue
2026 | 1     | 50000   -- Jan 2026
2026 | 2     | 60000   -- Feb 2026
2026 | NULL  | 110000  -- 2026 total
NULL | NULL  | 110000  -- Grand total
```

---

### **CUBE - All Possible Combinations**
```sql
SELECT country, city, SUM(sales)
FROM sales
GROUP BY CUBE (country, city);
```

**Generates:**
- (country, city)  -- Each city in each country
- (country)        -- Each country total
- (city)           -- Each city across all countries
- ()               -- Grand total

---

### **ROLLUP - Hierarchical Aggregation**
```sql
SELECT year, quarter, month, SUM(sales)
FROM sales
GROUP BY ROLLUP (year, quarter, month);
```

**Generates:**
- (year, quarter, month)  -- Most granular
- (year, quarter)         -- Quarterly totals
- (year)                  -- Yearly totals
- ()                      -- Grand total

**Use case:** Hierarchical reports (year → quarter → month)

---

## **Real Scenario: Analytics Dashboard**

**Query:** Show total orders, revenue, avg order value per month
```sql
SELECT 
    DATE_TRUNC('month', order_date) AS month,
    COUNT(*) AS order_count,
    SUM(total) AS revenue,
    AVG(total) AS avg_order_value,
    COUNT(DISTINCT customer_id) AS unique_customers
FROM orders
WHERE order_date >= '2026-01-01'
GROUP BY DATE_TRUNC('month', order_date)
ORDER BY month;
```

**Performance:**
- Index on `order_date` → Fast filtering
- GROUP BY on expression → Computed during aggregation
- Result: 12 rows (one per month) even if millions of orders

---

# **SECTION 11: SET OPERATIONS**

## **UNION - Combine Results**
```sql
-- All customers from orders OR registrations
SELECT customer_id, name FROM customers
UNION
SELECT customer_id, name FROM prospects;
```

**UNION** removes duplicates
**UNION ALL** keeps duplicates (faster)
```sql
-- Keep duplicates (customer in both lists appears twice)
SELECT customer_id FROM orders
UNION ALL
SELECT customer_id FROM abandoned_carts;
```

---

## **INTERSECT - Common Rows**
```sql
-- Customers who both ordered AND subscribed to newsletter
SELECT customer_id FROM orders
INTERSECT
SELECT customer_id FROM newsletter_subscribers;
```

---

## **EXCEPT - Difference**
```sql
-- Customers who ordered but NOT subscribed
SELECT customer_id FROM orders
EXCEPT
SELECT customer_id FROM newsletter_subscribers;
```

**Use case:** Find inactive users, missing data, etc.

---

# **SECTION 12: SUBQUERIES**

## **Scalar Subquery - Returns Single Value**
```sql
-- Products priced above average
SELECT product_name, price
FROM products
WHERE price > (SELECT AVG(price) FROM products);
```

**Execution:**
1. Subquery calculates AVG(price) → 150
2. Main query filters WHERE price > 150

---

## **Subquery in FROM - Inline Views**
```sql
SELECT customer_id, total_orders
FROM (
    SELECT customer_id, COUNT(*) AS total_orders
    FROM orders
    GROUP BY customer_id
) AS customer_stats
WHERE total_orders > 5;
```

**Treat subquery as temporary table**

---

## **Correlated Subquery - References Outer Query**
```sql
-- Each product with its category's average price
SELECT product_name, price,
    (SELECT AVG(price) 
     FROM products p2 
     WHERE p2.category_id = p1.category_id) AS category_avg
FROM products p1;
```

**Executes subquery for EACH row** (can be slow!)

**Better with JOIN:**
```sql
SELECT p.product_name, p.price, cat_avg.avg_price
FROM products p
JOIN (
    SELECT category_id, AVG(price) AS avg_price
    FROM products
    GROUP BY category_id
) cat_avg ON p.category_id = cat_avg.category_id;
```

---

## **EXISTS - Check for Existence**
```sql
-- Customers who placed orders
SELECT name FROM customers c
WHERE EXISTS (
    SELECT 1 FROM orders o 
    WHERE o.customer_id = c.customer_id
);
```

**Optimization:** Stops at first match (doesn't count all)

**NOT EXISTS:**
```sql
-- Customers who never ordered
SELECT name FROM customers c
WHERE NOT EXISTS (
    SELECT 1 FROM orders o 
    WHERE o.customer_id = c.customer_id
);
```

---

## **IN vs EXISTS**
```sql
-- IN (subquery returns list)
SELECT * FROM products
WHERE category_id IN (SELECT category_id FROM categories WHERE active = true);

-- EXISTS (checks existence)
SELECT * FROM products p
WHERE EXISTS (
    SELECT 1 FROM categories c 
    WHERE c.category_id = p.category_id AND c.active = true
);
```

**Performance:**
- **IN:** Better when subquery returns small list
- **EXISTS:** Better when subquery returns large list (stops early)

---

## **ANY/ALL**
```sql
-- Products cheaper than ANY product in 'Premium' category
SELECT * FROM products
WHERE price < ANY (
    SELECT price FROM products WHERE category = 'Premium'
);

-- Products more expensive than ALL products in 'Budget' category
SELECT * FROM products
WHERE price > ALL (
    SELECT price FROM products WHERE category = 'Budget'
);
```

---

# **SECTION 13: COMMON TABLE EXPRESSIONS (CTE)**

## **WITH Clause - Named Subqueries**

**Instead of nested subqueries:**
```sql
-- Hard to read
SELECT * FROM (
    SELECT customer_id, SUM(total) as spent FROM orders GROUP BY customer_id
) AS customer_totals
WHERE spent > 1000;

-- Clear with CTE
WITH customer_totals AS (
    SELECT customer_id, SUM(total) as spent 
    FROM orders 
    GROUP BY customer_id
)
SELECT * FROM customer_totals WHERE spent > 1000;
```

**Multiple CTEs:**
```sql
WITH 
active_users AS (
    SELECT user_id FROM users WHERE is_active = true
),
recent_orders AS (
    SELECT * FROM orders WHERE order_date > NOW() - INTERVAL '30 days'
)
SELECT u.name, COUNT(o.order_id) 
FROM active_users au
JOIN users u ON au.user_id = u.user_id
JOIN recent_orders o ON u.user_id = o.customer_id
GROUP BY u.name;
```

---

## **Recursive CTE - Hierarchies & Graphs**

**Scenario: Organizational Chart (Employee → Manager)**
```sql
WITH RECURSIVE employee_hierarchy AS (
    -- Base case: Top-level employees (no manager)
    SELECT employee_id, name, manager_id, 1 AS level
    FROM employees
    WHERE manager_id IS NULL
    
    UNION ALL
    
    -- Recursive case: Find employees under current level
    SELECT e.employee_id, e.name, e.manager_id, eh.level + 1
    FROM employees e
    JOIN employee_hierarchy eh ON e.manager_id = eh.employee_id
)
SELECT * FROM employee_hierarchy ORDER BY level, name;
```

**Result:**
```
employee_id | name  | manager_id | level
1           | CEO   | NULL       | 1
2           | VP1   | 1          | 2
3           | VP2   | 1          | 2
4           | Mgr1  | 2          | 3
5           | Emp1  | 4          | 4
```

**Other Use Cases:**
- Bill of Materials (product → components → sub-components)
- File system traversal
- Social network (friends of friends)
- Graph shortest paths

---

# **SECTION 14: MODIFYING DATA**

## **INSERT**
```sql
-- Single row
INSERT INTO users (name, email) VALUES ('Alice', 'alice@email.com');

-- Multiple rows
INSERT INTO users (name, email) VALUES 
    ('Bob', 'bob@email.com'),
    ('Carol', 'carol@email.com');

-- From SELECT (copy data)
INSERT INTO archived_orders 
SELECT * FROM orders WHERE order_date < '2025-01-01';

-- With RETURNING (get inserted data)
INSERT INTO users (name, email) 
VALUES ('Dave', 'dave@email.com')
RETURNING user_id, created_at;
```

---

## **UPDATE**
```sql
-- Update single column
UPDATE products SET price = 99.99 WHERE product_id = 100;

-- Update multiple columns
UPDATE users 
SET last_login = NOW(), login_count = login_count + 1
WHERE user_id = 50;

-- Update with JOIN
UPDATE products p
SET price = price * 0.9
FROM categories c
WHERE p.category_id = c.category_id AND c.name = 'Clearance';

-- Conditional update
UPDATE orders 
SET status = CASE 
    WHEN shipped_date IS NOT NULL THEN 'shipped'
    WHEN payment_received THEN 'processing'
    ELSE 'pending'
END;
```

---

## **DELETE**
```sql
-- Delete specific rows
DELETE FROM orders WHERE order_date < '2020-01-01';

-- Delete with JOIN
DELETE FROM order_items oi
USING orders o
WHERE oi.order_id = o.order_id AND o.status = 'cancelled';

-- Delete all (use TRUNCATE instead!)
DELETE FROM logs; -- Slow, logs each deletion
TRUNCATE TABLE logs; -- Fast, resets table
```

---

## **UPSERT (INSERT ... ON CONFLICT)**

**Scenario: Insert if new, update if exists**
```sql
INSERT INTO user_stats (user_id, login_count, last_login)
VALUES (100, 1, NOW())
ON CONFLICT (user_id) DO UPDATE SET
    login_count = user_stats.login_count + 1,
    last_login = NOW();
```

**First time:** Inserts row
**Subsequent:** Updates existing row

---

## **MERGE (PostgreSQL 15+)**
```sql
MERGE INTO inventory i
USING incoming_shipment s ON i.product_id = s.product_id
WHEN MATCHED THEN
    UPDATE SET quantity = i.quantity + s.quantity
WHEN NOT MATCHED THEN
    INSERT (product_id, quantity) VALUES (s.product_id, s.quantity);
```

---

# **SECTION 15: TABLE MANAGEMENT**

## **Create Table**
```sql
CREATE TABLE orders (
    order_id SERIAL PRIMARY KEY,
    customer_id INT NOT NULL,
    order_date TIMESTAMP DEFAULT NOW(),
    total DECIMAL(10,2) CHECK (total >= 0),
    status VARCHAR(20) DEFAULT 'pending',
    FOREIGN KEY (customer_id) REFERENCES customers(customer_id)
);
```

## **Alter Table**
```sql
-- Add column
ALTER TABLE users ADD COLUMN phone VARCHAR(20);

-- Drop column
ALTER TABLE users DROP COLUMN middle_name;

-- Rename column
ALTER TABLE users RENAME COLUMN email TO email_address;

-- Change data type
ALTER TABLE products ALTER COLUMN price TYPE NUMERIC(12,2);

-- Add constraint
ALTER TABLE orders ADD CONSTRAINT fk_customer 
    FOREIGN KEY (customer_id) REFERENCES customers(customer_id);
```

## **Special Column Types**
```sql
-- SERIAL (auto-increment)
CREATE TABLE users (
    user_id SERIAL PRIMARY KEY -- Auto: 1, 2, 3...
);

-- IDENTITY (SQL standard)
CREATE TABLE products (
    product_id INT GENERATED ALWAYS AS IDENTITY PRIMARY KEY
);

-- Generated column (computed)
CREATE TABLE order_items (
    quantity INT,
    price DECIMAL,
    total DECIMAL GENERATED ALWAYS AS (quantity * price) STORED
);
```

---

# **SECTION 16: DATA TYPES**

## **Numeric**
```sql
INT, BIGINT, SMALLINT           -- Integers
DECIMAL(10,2), NUMERIC(10,2)    -- Exact decimals (money)
REAL, DOUBLE PRECISION          -- Floating point (scientific)
SERIAL, BIGSERIAL               -- Auto-increment
```

## **Text**
```sql
CHAR(10)      -- Fixed length (pads with spaces)
VARCHAR(100)  -- Variable length (up to 100)
TEXT          -- Unlimited length
```

## **Date/Time**
```sql
DATE          -- 2026-02-09
TIME          -- 14:30:00
TIMESTAMP     -- 2026-02-09 14:30:00
TIMESTAMPTZ   -- With timezone
INTERVAL      -- '2 days', '3 hours'
```

## **Boolean**
```sql
BOOLEAN       -- TRUE, FALSE, NULL
```

## **Special Types**
```sql
UUID          -- Universally unique identifier
JSON, JSONB   -- JSON data (JSONB is binary, faster)
ARRAY         -- INT[], TEXT[]
HSTORE        -- Key-value pairs
BYTEA         -- Binary data
ENUM          -- Custom type: CREATE TYPE mood AS ENUM ('sad', 'ok', 'happy');
```

---

# **SECTION 17: CONDITIONAL EXPRESSIONS**

## **CASE**
```sql
SELECT 
    name,
    age,
    CASE 
        WHEN age < 18 THEN 'Minor'
        WHEN age < 65 THEN 'Adult'
        ELSE 'Senior'
    END AS age_group
FROM users;

-- Simple CASE
SELECT 
    order_id,
    CASE status
        WHEN 'P' THEN 'Pending'
        WHEN 'S' THEN 'Shipped'
        WHEN 'D' THEN 'Delivered'
    END AS status_text
FROM orders;
```

## **COALESCE - First Non-NULL**
```sql
SELECT name, COALESCE(phone, email, 'No contact') AS contact
FROM users;
-- Returns phone if not NULL, else email, else 'No contact'
```

## **NULLIF - Return NULL if Equal**
```sql
SELECT price, NULLIF(discount, 0) AS discount
FROM products;
-- If discount is 0, returns NULL (avoid division by zero)
```

## **CAST - Type Conversion**
```sql
SELECT CAST('123' AS INT);
SELECT '2026-02-09'::DATE;  -- PostgreSQL shorthand
SELECT price::TEXT FROM products;
```

---

# **SECTION 18: VIEWS**

## **Create View - Virtual Table**
```sql
CREATE VIEW active_customers AS
SELECT customer_id, name, email
FROM customers
WHERE is_active = true;

-- Use like a table
SELECT * FROM active_customers;
```

**Benefits:**
- Simplify complex queries
- Security (hide columns)
- Consistent logic across applications

## **Materialized View - Cached Results**
```sql
CREATE MATERIALIZED VIEW sales_summary AS
SELECT 
    DATE_TRUNC('day', order_date) AS day,
    SUM(total) AS revenue
FROM orders
GROUP BY day;

-- Refresh cache
REFRESH MATERIALIZED VIEW sales_summary;
```

**Use case:** Expensive aggregations run hourly instead of per query

---

# **SECTION 19: INDEXES DEEP-DIVE**

## **Index Types Recap**
```sql
-- B-tree (default, 95% of cases)
CREATE INDEX idx_email ON users(email);

-- Partial index (subset)
CREATE INDEX idx_active ON users(email) WHERE is_active = true;

-- Multi-column
CREATE INDEX idx_name ON users(last_name, first_name);

-- Expression index
CREATE INDEX idx_lower_email ON users(LOWER(email));

-- Covering index
CREATE INDEX idx_email_covering ON users(email) INCLUDE (name, phone);

-- GIN (arrays, JSON)
CREATE INDEX idx_tags ON posts USING GIN(tags);

-- GiST (geometric, ranges)
CREATE INDEX idx_location ON places USING GIST(location);

-- BRIN (time-series, huge tables)
CREATE INDEX idx_created ON logs USING BRIN(created_at);
```

## **Monitoring**
```sql
-- Check if index is used
EXPLAIN SELECT * FROM users WHERE email = 'alice@email.com';
-- Look for "Index Scan using idx_email"

-- Find unused indexes
SELECT schemaname, tablename, indexname, idx_scan
FROM pg_stat_user_indexes
WHERE idx_scan = 0;

-- Index size
SELECT pg_size_pretty(pg_relation_size('idx_email'));
```

---

# **SECTION 20: TRIGGERS**

## **What Triggers Do**

Execute code automatically on INSERT/UPDATE/DELETE
```sql
-- Log all changes to users table
CREATE TABLE user_audit (
    audit_id SERIAL PRIMARY KEY,
    user_id INT,
    action VARCHAR(10),
    changed_at TIMESTAMP DEFAULT NOW()
);

CREATE OR REPLACE FUNCTION log_user_changes()
RETURNS TRIGGER AS $$
BEGIN
    INSERT INTO user_audit (user_id, action)
    VALUES (NEW.user_id, TG_OP);
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_user_audit
AFTER INSERT OR UPDATE OR DELETE ON users
FOR EACH ROW EXECUTE FUNCTION log_user_changes();
```

## **Trigger Timing**
```sql
-- BEFORE trigger (can modify NEW)
CREATE TRIGGER trg_validate_email
BEFORE INSERT ON users
FOR EACH ROW
EXECUTE FUNCTION validate_email_format();

-- AFTER trigger (cannot modify, used for logging)
CREATE TRIGGER trg_send_welcome_email
AFTER INSERT ON users
FOR EACH ROW
EXECUTE FUNCTION send_welcome_email();
```

## **Use Cases**

- **Audit trails:** Log who changed what and when
- **Validation:** Enforce complex business rules
- **Denormalization:** Auto-update summary tables
- **Cascading updates:** Update related records

**Warning:** Triggers slow down DML operations. Use sparingly.

---

# **SECTION 21: STORED PROCEDURES & FUNCTIONS (PL/pgSQL)**

## **Function vs Procedure**

**Function:** Returns value, can be used in SELECT
**Procedure:** Doesn't return value, called with CALL

## **Create Function**
```sql
CREATE OR REPLACE FUNCTION get_customer_orders(cust_id INT)
RETURNS TABLE(order_id INT, total DECIMAL) AS $$
BEGIN
    RETURN QUERY
    SELECT o.order_id, o.total
    FROM orders o
    WHERE o.customer_id = cust_id;
END;
$$ LANGUAGE plpgsql;

-- Use in query
SELECT * FROM get_customer_orders(100);
```

## **Create Procedure**
```sql
CREATE OR REPLACE PROCEDURE process_order(
    p_customer_id INT,
    p_product_id INT,
    p_quantity INT
)
LANGUAGE plpgsql AS $$
DECLARE
    v_order_id INT;
BEGIN
    -- Create order
    INSERT INTO orders (customer_id, total)
    VALUES (p_customer_id, 0)
    RETURNING order_id INTO v_order_id;
    
    -- Add item
    INSERT INTO order_items (order_id, product_id, quantity)
    VALUES (v_order_id, p_product_id, p_quantity);
    
    -- Update inventory
    UPDATE products 
    SET stock = stock - p_quantity
    WHERE product_id = p_product_id;
    
    COMMIT;
END;
$$;

-- Call procedure
CALL process_order(100, 500, 2);
```

## **Control Structures**
```sql
-- IF/ELSE
IF age < 18 THEN
    status := 'minor';
ELSIF age < 65 THEN
    status := 'adult';
ELSE
    status := 'senior';
END IF;

-- CASE
CASE status
    WHEN 'P' THEN RETURN 'Pending';
    WHEN 'S' THEN RETURN 'Shipped';
    ELSE RETURN 'Unknown';
END CASE;

-- LOOP
FOR i IN 1..10 LOOP
    INSERT INTO test VALUES (i);
END LOOP;

-- WHILE
WHILE counter < 100 LOOP
    counter := counter + 1;
END LOOP;

-- FOREACH (arrays)
FOREACH item IN ARRAY item_array LOOP
    RAISE NOTICE 'Item: %', item;
END LOOP;
```

## **Error Handling**
```sql
BEGIN
    UPDATE accounts SET balance = balance - 100 WHERE id = 1;
EXCEPTION
    WHEN insufficient_funds THEN
        RAISE NOTICE 'Insufficient funds';
        ROLLBACK;
    WHEN OTHERS THEN
        RAISE EXCEPTION 'Unknown error: %', SQLERRM;
END;
```

---

# **SECTION 22: ADVANCED FUNCTIONS**

## **Window Functions - Powerful Analytics**
```sql
-- Row number
SELECT 
    product_name,
    price,
    ROW_NUMBER() OVER (ORDER BY price DESC) AS rank
FROM products;

-- Running total
SELECT 
    order_date,
    total,
    SUM(total) OVER (ORDER BY order_date) AS running_total
FROM orders;

-- Partition by category
SELECT 
    category,
    product_name,
    price,
    AVG(price) OVER (PARTITION BY category) AS category_avg
FROM products;

-- Rank with ties
SELECT 
    name,
    score,
    RANK() OVER (ORDER BY score DESC) AS rank,
    DENSE_RANK() OVER (ORDER BY score DESC) AS dense_rank
FROM students;

-- Lag/Lead (previous/next row)
SELECT 
    date,
    sales,
    LAG(sales) OVER (ORDER BY date) AS previous_day,
    LEAD(sales) OVER (ORDER BY date) AS next_day
FROM daily_sales;
```

## **String Functions**
```sql
CONCAT('Hello', ' ', 'World')              -- 'Hello World'
UPPER('hello')                             -- 'HELLO'
LOWER('HELLO')                             -- 'hello'
LENGTH('hello')                            -- 5
SUBSTRING('hello' FROM 2 FOR 3)           -- 'ell'
REPLACE('hello world', 'world', 'there')  -- 'hello there'
SPLIT_PART('a,b,c', ',', 2)               -- 'b'
TRIM('  hello  ')                          -- 'hello'
LEFT('hello', 3)                           -- 'hel'
RIGHT('hello', 3)                          -- 'llo'
POSITION('lo' IN 'hello')                  -- 4
```

## **Date Functions**
```sql
NOW()                                      -- Current timestamp
CURRENT_DATE                               -- Today
CURRENT_TIME                               -- Current time
AGE('2026-02-09', '2000-01-01')           -- Interval
EXTRACT(YEAR FROM NOW())                   -- 2026
DATE_PART('month', NOW())                  -- 2
DATE_TRUNC('month', NOW())                 -- 2026-02-01 00:00:00
NOW() + INTERVAL '7 days'                  -- Next week
NOW() - INTERVAL '1 month'                 -- Last month
TO_CHAR(NOW(), 'YYYY-MM-DD')              -- '2026-02-09'
```

## **Math Functions**
```sql
ABS(-5)                                    -- 5
CEIL(4.3)                                  -- 5
FLOOR(4.9)                                 -- 4
ROUND(4.567, 2)                            -- 4.57
POWER(2, 3)                                -- 8
SQRT(16)                                   -- 4
RANDOM()                                   -- 0.0 to 1.0
MOD(10, 3)                                 -- 1
```

## **JSON Functions**
```sql
-- Extract field
SELECT data->>'name' FROM users; -- Get as text
SELECT data->'address'->'city' FROM users; -- Navigate nested

-- Array elements
SELECT jsonb_array_elements('[1,2,3]');

-- Build JSON
SELECT jsonb_build_object('name', 'Alice', 'age', 30);

-- Aggregate to JSON
SELECT jsonb_agg(name) FROM users;

-- Check existence
SELECT * FROM products WHERE data @> '{"color": "red"}';
```

---

# **SECTION 23: PERFORMANCE OPTIMIZATION**

## **EXPLAIN - Query Analysis**
```sql
EXPLAIN SELECT * FROM orders WHERE customer_id = 100;
-- Shows: Seq Scan or Index Scan

EXPLAIN ANALYZE SELECT * FROM orders WHERE customer_id = 100;
-- Actually runs query, shows real timing
```

**Read EXPLAIN output:**
- **Seq Scan:** Reading entire table (bad for large tables)
- **Index Scan:** Using index (good)
- **Bitmap Index Scan:** Using multiple indexes
- **Hash Join:** Joining via hash table
- **Nested Loop:** Joining via loops (slow for large tables)

## **Query Optimization Techniques**

**1. Use indexes on WHERE/JOIN columns**
**2. SELECT only needed columns (not `*`)**
**3. Filter early (WHERE before JOIN)**
**4. Use LIMIT for pagination**
**5. Avoid correlated subqueries (use JOINs)**
**6. Use UNION ALL instead of UNION when duplicates ok**
**7. Batch INSERTs (multi-row INSERT)**
**8. Use connection pooling**
**9. Partition huge tables**
**10. Use materialized views for expensive aggregations**

---

# **SECTION 24: REAL-WORLD SCENARIOS**

## **Scenario 1: High-Traffic E-commerce**

**Challenge:** 10,000 orders/sec, need to show inventory in real-time

**Solution:**
```sql
-- Denormalized product_stock table (updated by trigger)
CREATE TABLE product_stock (
    product_id INT PRIMARY KEY,
    available INT,
    reserved INT,
    last_updated TIMESTAMP
);

-- Fast read (no JOIN)
SELECT available FROM product_stock WHERE product_id = 100;

-- Write triggers update stock
CREATE TRIGGER trg_update_stock
AFTER INSERT ON order_items
FOR EACH ROW EXECUTE FUNCTION update_stock();
```

**Indexes:**
```sql
CREATE INDEX idx_orders_date ON orders(order_date) USING BRIN;
CREATE INDEX idx_customer ON orders(customer_id);
CREATE INDEX idx_product ON order_items(product_id);
```

---

## **Scenario 2: Social Media Feed (Instagram-like)**

**Challenge:** Show user's feed (posts from followed users) in <100ms

**Solution:**
```sql
-- Pre-computed feed table (fan-out on write)
CREATE TABLE user_feeds (
    user_id INT,
    post_id BIGINT,
    created_at TIMESTAMP,
    PRIMARY KEY (user_id, created_at, post_id)
);

-- On post create: Insert into all followers' feeds
CREATE OR REPLACE FUNCTION fanout_post()
RETURNS TRIGGER AS $$
BEGIN
    INSERT INTO user_feeds (user_id, post_id, created_at)
    SELECT follower_id, NEW.post_id, NEW.created_at
    FROM follows
    WHERE following_id = NEW.author_id;
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

-- Read feed (simple query, no JOINs)
SELECT post_id FROM user_feeds 
WHERE user_id = 12345 
ORDER BY created_at DESC 
LIMIT 50;
```

**Trade-off:** More writes (fan-out), but blazing fast reads

---

## **Scenario 3: Analytics Platform (1B events/day)**

**Challenge:** Store and query massive event logs

**Solution:**
```sql
-- Partitioned table (by month)
CREATE TABLE events (
    event_id BIGSERIAL,
    user_id INT,
    event_type VARCHAR(50),
    created_at TIMESTAMP NOT NULL
) PARTITION BY RANGE (created_at);

-- Create monthly partitions
CREATE TABLE events_2026_01 PARTITION OF events
FOR VALUES FROM ('2026-01-01') TO ('2026-02-01');

CREATE TABLE events_2026_02 PARTITION OF events
FOR VALUES FROM ('2026-02-01') TO ('2026-03-01');

-- BRIN index (tiny, perfect for time-series)
CREATE INDEX idx_events_created ON events USING BRIN(created_at);

-- Query only touches relevant partition
SELECT COUNT(*) FROM events 
WHERE created_at BETWEEN '2026-02-01' AND '2026-02-09';
-- Only scans events_2026_02 partition
```

---

## **Scenario 4: Banking System (ACID Critical)**

**Challenge:** Transfer money between accounts, zero tolerance for inconsistency

**Solution:**
```sql
CREATE OR REPLACE PROCEDURE transfer_money(
    from_account INT,
    to_account INT,
    amount DECIMAL
)
LANGUAGE plpgsql AS $$
BEGIN
    -- SERIALIZABLE isolation
    SET TRANSACTION ISOLATION LEVEL SERIALIZABLE;
    
    -- Lock both accounts (deterministic order to avoid deadlock)
    PERFORM 1 FROM accounts 
    WHERE account_id IN (from_account, to_account)
    ORDER BY account_id
    FOR UPDATE;
    
    -- Check balance
    IF (SELECT balance FROM accounts WHERE account_id = from_account) < amount THEN
        RAISE EXCEPTION 'Insufficient funds';
    END IF;
    
    -- Transfer
    UPDATE accounts SET balance = balance - amount WHERE account_id = from_account;
    UPDATE accounts SET balance = balance + amount WHERE account_id = to_account;
    
    -- Audit log
    INSERT INTO transfers (from_account, to_account, amount, timestamp)
    VALUES (from_account, to_account, amount, NOW());
    
    COMMIT;
EXCEPTION
    WHEN OTHERS THEN
        ROLLBACK;
        RAISE;
END;
$$;
```

---

## **Key Takeaways**

**1. Design Phase:**
- Model entities and relationships carefully (ER diagrams)
- Normalize to 3NF, denormalize selectively for performance
- Choose right data types (INT vs BIGINT, VARCHAR vs TEXT)

**2. Indexing Strategy:**
- Index all foreign keys
- Index WHERE/JOIN columns
- Use partial indexes for common filters
- Multi-column indexes for composite queries
- Monitor and drop unused indexes

**3. Query Optimization:**
- Filter early (WHERE before JOIN)
- Use indexes (check with EXPLAIN)
- Avoid SELECT *
- Use JOINs over subqueries
- Batch operations

**4. Concurrency:**
- Use appropriate isolation level
- Keep transactions short
- Lock in deterministic order (prevent deadlocks)
- Use optimistic locking for low contention

**5. Scaling:**
- Read replicas for read-heavy workloads
- Partitioning for huge tables
- Denormalization for critical queries
- Caching (Redis/Memcached) for hot data
- Connection pooling (PgBouncer)

**6. Maintenance:**
- VACUUM regularly (clean dead rows)
- ANALYZE to update statistics
- Monitor slow queries (pg_stat_statements)
- Archive old data
- Plan for backups and recovery

---

# **ESSENTIAL POSTGRESQL TOPICS FOR AI/ML ENGINEERS**

## **TIER 1: ABSOLUTE MUST-KNOW (Daily Use)**

### **1. Data Retrieval & Filtering**
- **SELECT, WHERE, ORDER BY, LIMIT**
  - Fetching training data, filtering datasets
  - `SELECT features, labels FROM training_data WHERE split = 'train' LIMIT 10000`

- **JOINs (INNER, LEFT)**
  - Combining features from multiple tables
  - `SELECT f.*, l.label FROM features f JOIN labels l ON f.id = l.id`

- **GROUP BY, Aggregate Functions (COUNT, AVG, SUM, MAX, MIN)**
  - Data exploration, statistics
  - `SELECT category, COUNT(*), AVG(price) FROM products GROUP BY category`

- **DISTINCT, DISTINCT ON**
  - Remove duplicates in datasets
  - `SELECT DISTINCT user_id FROM user_events`

### **2. Data Insertion & Updates**
- **INSERT (single & bulk)**
  - Loading predictions, storing results
  - `INSERT INTO predictions (model_id, prediction, confidence) VALUES (...)`

- **UPDATE**
  - Updating model versions, feature flags
  - `UPDATE experiments SET status = 'completed' WHERE experiment_id = 123`

- **UPSERT (ON CONFLICT)**
  - Critical for ML: Insert new predictions or update existing
```sql
  INSERT INTO model_metrics (model_id, accuracy, timestamp)
  VALUES (5, 0.95, NOW())
  ON CONFLICT (model_id) DO UPDATE SET accuracy = 0.95, timestamp = NOW();
```

### **3. Indexes - Performance Critical**
- **B-tree indexes on frequently queried columns**
```sql
  CREATE INDEX idx_user_id ON events(user_id);
  CREATE INDEX idx_timestamp ON events(created_at);
```

- **Multi-column indexes for composite queries**
```sql
  CREATE INDEX idx_user_date ON events(user_id, created_at);
```

- **EXPLAIN/EXPLAIN ANALYZE**
  - Debug slow queries fetching training data
```sql
  EXPLAIN ANALYZE SELECT * FROM features WHERE user_id = 12345;
```

### **4. Data Types**
- **Numeric**: INT, BIGINT, DECIMAL, REAL, DOUBLE PRECISION
- **Text**: VARCHAR, TEXT
- **Date/Time**: TIMESTAMP, DATE, INTERVAL
- **Arrays**: `INT[]`, `REAL[]` (store embeddings, feature vectors)
- **JSON/JSONB**: Store model configs, metadata
```sql
  CREATE TABLE models (
      model_id SERIAL PRIMARY KEY,
      config JSONB,
      metrics JSONB
  );
```

### **5. Window Functions**
- **ROW_NUMBER, RANK, DENSE_RANK**
  - Ranking predictions, finding top-K
```sql
  SELECT user_id, score, 
         RANK() OVER (PARTITION BY category ORDER BY score DESC) as rank
  FROM predictions;
```

- **LAG, LEAD**
  - Time-series features (previous/next values)
```sql
  SELECT timestamp, value,
         LAG(value) OVER (ORDER BY timestamp) as prev_value
  FROM sensor_data;
```

- **Running aggregations**
```sql
  SELECT date, revenue,
         SUM(revenue) OVER (ORDER BY date ROWS BETWEEN 6 PRECEDING AND CURRENT ROW) as rolling_7day
  FROM daily_sales;
```

---

## **TIER 2: VERY IMPORTANT (Weekly Use)**

### **6. CTEs (Common Table Expressions)**
- Clean, readable queries for complex data pipelines
```sql
WITH feature_stats AS (
    SELECT user_id, AVG(purchase_amount) as avg_purchase
    FROM transactions
    GROUP BY user_id
),
labeled_data AS (
    SELECT user_id, churned FROM users WHERE label_date IS NOT NULL
)
SELECT f.*, l.churned
FROM feature_stats f
JOIN labeled_data l ON f.user_id = l.user_id;
```

### **7. Subqueries**
- **IN, EXISTS** for filtering
- **Scalar subqueries** for feature engineering
```sql
SELECT user_id,
       (SELECT COUNT(*) FROM orders WHERE orders.user_id = users.user_id) as order_count
FROM users;
```

### **8. String Functions**
- **CONCAT, SUBSTRING, LOWER, UPPER, TRIM**
- **Text preprocessing for NLP**
```sql
SELECT LOWER(TRIM(review_text)) as cleaned_text FROM reviews;
```

- **Regular expressions (REGEXP_REPLACE)**
```sql
SELECT REGEXP_REPLACE(email, '@.*', '') as username FROM users;
```

### **9. Date/Time Functions**
- **NOW(), CURRENT_DATE, DATE_TRUNC, EXTRACT**
- **Feature engineering**: day of week, hour, month
```sql
SELECT EXTRACT(DOW FROM order_date) as day_of_week,
       EXTRACT(HOUR FROM order_timestamp) as hour,
       DATE_TRUNC('month', order_date) as month
FROM orders;
```

- **INTERVAL for time windows**
```sql
SELECT * FROM events 
WHERE created_at > NOW() - INTERVAL '7 days';
```

### **10. Transactions & Isolation**
- **BEGIN, COMMIT, ROLLBACK**
- Ensure consistency when writing model results
```sql
BEGIN;
    INSERT INTO model_runs (model_id, start_time) VALUES (5, NOW());
    UPDATE models SET status = 'running' WHERE model_id = 5;
COMMIT;
```

- **Isolation levels** (READ COMMITTED is default, usually sufficient)

### **11. Array Operations**
- **Storing embeddings/vectors**
```sql
CREATE TABLE embeddings (
    item_id INT,
    vector REAL[]  -- [0.1, 0.5, 0.3, ...]
);

-- Query
SELECT item_id FROM embeddings WHERE vector[1] > 0.5;
```

- **ARRAY_AGG** (collect values into array)
```sql
SELECT user_id, ARRAY_AGG(product_id ORDER BY purchase_date) as purchase_sequence
FROM purchases
GROUP BY user_id;
```

### **12. JSON/JSONB Operations**
- **Store model configs, hyperparameters, metadata**
```sql
CREATE TABLE experiments (
    experiment_id SERIAL PRIMARY KEY,
    hyperparams JSONB,
    results JSONB
);

-- Query
SELECT * FROM experiments 
WHERE hyperparams->>'learning_rate' = '0.001';

-- Update nested JSON
UPDATE experiments 
SET results = jsonb_set(results, '{metrics,accuracy}', '0.95')
WHERE experiment_id = 5;
```

---

## **TIER 3: IMPORTANT (Monthly Use / Specific Scenarios)**

### **13. Partitioning**
- **For huge event/log tables**
```sql
CREATE TABLE events (
    event_id BIGSERIAL,
    user_id INT,
    created_at TIMESTAMP
) PARTITION BY RANGE (created_at);

CREATE TABLE events_2026_01 PARTITION OF events
FOR VALUES FROM ('2026-01-01') TO ('2026-02-01');
```

### **14. BRIN Indexes**
- **For time-series data (logs, events)**
```sql
CREATE INDEX idx_created_brin ON events USING BRIN(created_at);
```
- Tiny index, perfect for append-only tables

### **15. Materialized Views**
- **Pre-compute expensive aggregations**
```sql
CREATE MATERIALIZED VIEW user_feature_summary AS
SELECT user_id,
       COUNT(*) as total_events,
       AVG(session_duration) as avg_session
FROM events
GROUP BY user_id;

-- Refresh periodically
REFRESH MATERIALIZED VIEW user_feature_summary;
```

### **16. GIN Indexes**
- **For JSONB queries**
```sql
CREATE INDEX idx_metadata ON experiments USING GIN(hyperparams);

SELECT * FROM experiments 
WHERE hyperparams @> '{"optimizer": "adam"}';
```

### **17. Sampling Data**
- **TABLESAMPLE for quick dataset exploration**
```sql
SELECT * FROM events TABLESAMPLE SYSTEM(1);  -- Random 1%
```

### **18. Recursive CTEs**
- **Graph traversal** (social networks, recommendation graphs)
```sql
WITH RECURSIVE friends AS (
    SELECT user_id, friend_id, 1 as depth
    FROM friendships WHERE user_id = 123
    UNION ALL
    SELECT f.user_id, fr.friend_id, friends.depth + 1
    FROM friends
    JOIN friendships fr ON friends.friend_id = fr.user_id
    WHERE depth < 3
)
SELECT * FROM friends;
```

### **19. COPY Command**
- **Bulk import/export (faster than INSERT)**
```sql
-- Export predictions to CSV
COPY predictions TO '/tmp/predictions.csv' CSV HEADER;

-- Import training data from CSV
COPY training_data FROM '/tmp/data.csv' CSV HEADER;
```

### **20. Stored Functions (for reusable logic)**
```sql
CREATE OR REPLACE FUNCTION calculate_churn_score(user_id INT)
RETURNS DECIMAL AS $$
DECLARE
    score DECIMAL;
BEGIN
    SELECT (days_since_last_login * 0.3 + order_count * -0.5)
    INTO score
    FROM user_features WHERE user_id = user_id;
    RETURN score;
END;
$$ LANGUAGE plpgsql;

-- Use in queries
SELECT user_id, calculate_churn_score(user_id) FROM users;
```

---

## **TIER 4: GOOD TO KNOW (Occasional Use)**

### **21. Triggers**
- Auto-update timestamps, derived columns
```sql
CREATE TRIGGER update_modified_time
BEFORE UPDATE ON experiments
FOR EACH ROW EXECUTE FUNCTION update_timestamp();
```

### **22. Full-Text Search**
- **For text data preprocessing**
```sql
CREATE INDEX idx_review_fts ON reviews USING GIN(to_tsvector('english', review_text));

SELECT * FROM reviews 
WHERE to_tsvector('english', review_text) @@ to_tsquery('excellent & product');
```

### **23. LATERAL Joins**
- **Correlated queries** (for each user, get top 3 products)
```sql
SELECT u.user_id, p.*
FROM users u
CROSS JOIN LATERAL (
    SELECT * FROM purchases 
    WHERE purchases.user_id = u.user_id
    ORDER BY purchase_date DESC
    LIMIT 3
) p;
```

### **24. Foreign Data Wrappers (FDW)**
- Query external databases/APIs from PostgreSQL
```sql
CREATE EXTENSION postgres_fdw;
CREATE SERVER remote_db FOREIGN DATA WRAPPER postgres_fdw 
OPTIONS (host 'remote.server.com', dbname 'production');

-- Query remote tables
SELECT * FROM remote_table;
```

### **25. Connection Pooling (PgBouncer)**
- Essential for high-concurrency ML services
- Reuse connections instead of creating new ones

---

## **PRACTICAL ML/AI USE CASES**

### **Use Case 1: Feature Engineering Pipeline**
```sql
-- Create features for churn prediction
WITH user_activity AS (
    SELECT user_id,
           COUNT(*) as login_count,
           MAX(login_date) as last_login,
           EXTRACT(EPOCH FROM (NOW() - MAX(login_date)))/86400 as days_since_login
    FROM logins
    WHERE login_date > NOW() - INTERVAL '90 days'
    GROUP BY user_id
),
purchase_features AS (
    SELECT user_id,
           COUNT(*) as purchase_count,
           AVG(amount) as avg_purchase,
           SUM(amount) as total_spent
    FROM purchases
    WHERE purchase_date > NOW() - INTERVAL '90 days'
    GROUP BY user_id
)
SELECT u.user_id,
       COALESCE(ua.login_count, 0) as login_count,
       COALESCE(ua.days_since_login, 999) as days_since_login,
       COALESCE(pf.purchase_count, 0) as purchase_count,
       COALESCE(pf.avg_purchase, 0) as avg_purchase,
       u.churned as label
FROM users u
LEFT JOIN user_activity ua ON u.user_id = ua.user_id
LEFT JOIN purchase_features pf ON u.user_id = pf.user_id
WHERE u.label_date IS NOT NULL;
```

### **Use Case 2: Storing Model Predictions**
```sql
-- Store batch predictions with metadata
CREATE TABLE predictions (
    prediction_id BIGSERIAL PRIMARY KEY,
    model_id INT,
    user_id INT,
    predicted_class INT,
    probability REAL,
    prediction_date TIMESTAMP DEFAULT NOW(),
    model_version VARCHAR(50)
);

-- Insert predictions (bulk)
INSERT INTO predictions (model_id, user_id, predicted_class, probability, model_version)
SELECT 5, user_id, pred_class, prob, 'v2.3.1'
FROM UNNEST(
    ARRAY[101, 102, 103],  -- user_ids
    ARRAY[1, 0, 1],        -- predictions
    ARRAY[0.95, 0.32, 0.87] -- probabilities
) AS t(user_id, pred_class, prob);
```

### **Use Case 3: A/B Test Analysis**
```sql
-- Compare conversion rates between control and treatment
SELECT 
    variant,
    COUNT(*) as users,
    SUM(CASE WHEN converted THEN 1 ELSE 0 END) as conversions,
    ROUND(AVG(CASE WHEN converted THEN 1 ELSE 0 END), 4) as conversion_rate
FROM experiments
WHERE experiment_id = 'homepage_redesign'
GROUP BY variant;
```

### **Use Case 4: Time-Series Feature Extraction**
```sql
-- Create rolling averages for time-series prediction
SELECT 
    date,
    sales,
    AVG(sales) OVER (ORDER BY date ROWS BETWEEN 6 PRECEDING AND CURRENT ROW) as ma_7day,
    AVG(sales) OVER (ORDER BY date ROWS BETWEEN 29 PRECEDING AND CURRENT ROW) as ma_30day,
    LAG(sales, 7) OVER (ORDER BY date) as sales_last_week,
    LAG(sales, 365) OVER (ORDER BY date) as sales_last_year
FROM daily_sales
ORDER BY date;
```

### **Use Case 5: Recommendation System Data Prep**
```sql
-- User-item interaction matrix
SELECT user_id, product_id, COUNT(*) as interaction_count
FROM events
WHERE event_type IN ('view', 'click', 'purchase')
  AND created_at > NOW() - INTERVAL '30 days'
GROUP BY user_id, product_id;

-- Item similarity (co-occurrence)
WITH user_products AS (
    SELECT user_id, ARRAY_AGG(DISTINCT product_id) as products
    FROM purchases
    GROUP BY user_id
)
SELECT p1.product_id as product_a,
       p2.product_id as product_b,
       COUNT(*) as co_occurrence
FROM user_products up
CROSS JOIN UNNEST(up.products) as p1(product_id)
CROSS JOIN UNNEST(up.products) as p2(product_id)
WHERE p1.product_id < p2.product_id
GROUP BY p1.product_id, p2.product_id
ORDER BY co_occurrence DESC;
```

---

## **PRIORITY LEARNING PATH**

### **Week 1-2: Foundation**
1. SELECT, WHERE, ORDER BY, LIMIT
2. JOINs (INNER, LEFT)
3. GROUP BY, aggregate functions
4. Basic indexes (CREATE INDEX)
5. INSERT, UPDATE, UPSERT

### **Week 3-4: Intermediate**
6. Window functions (ROW_NUMBER, LAG, LEAD, running totals)
7. CTEs (WITH clauses)
8. String and date functions
9. EXPLAIN for query optimization
10. JSON/JSONB basics

### **Week 5-6: Advanced**
11. Array operations
12. Transactions
13. Subqueries and EXISTS
14. Multi-column indexes
15. Materialized views

### **Week 7-8: Production**
16. Partitioning for big tables
17. BRIN indexes for time-series
18. GIN indexes for JSON
19. COPY for bulk import/export
20. Performance monitoring

---

## **ESSENTIAL POSTGRES COMMANDS TO MEMORIZE**
```sql
-- Connection
\c database_name
\dt                    -- List tables
\d table_name          -- Describe table

-- Performance
EXPLAIN ANALYZE query;
VACUUM ANALYZE table_name;

-- Indexes
CREATE INDEX idx_name ON table(column);
\di                    -- List indexes

-- Import/Export
\copy table FROM 'file.csv' CSV HEADER;
\copy (query) TO 'file.csv' CSV HEADER;

-- Useful queries
SELECT pg_size_pretty(pg_total_relation_size('table_name'));  -- Table size
SELECT * FROM pg_stat_user_indexes WHERE idx_scan = 0;        -- Unused indexes
```

---

## **WHAT YOU CAN SKIP (For AI/ML Context)**

1. ❌ **Advanced PL/pgSQL** (complex stored procedures) - Python handles ML logic
2. ❌ **Database administration** (replication, backup strategies) - DevOps handles this
3. ❌ **Low-level internals** (WAL, MVCC details) - Unless doing DB research
4. ❌ **Exotic index types** (Bloom, SP-GiST) - Rarely needed
5. ❌ **Triggers for complex business logic** - Keep logic in application/Python

---

# **5 CRITICAL AI/ML DATABASE SCENARIOS**

---

## **SCENARIO 1: REAL-TIME FEATURE STORE FOR RECOMMENDATION SYSTEM**

### **❓ PROBLEM STATEMENT**

You're building a product recommendation system for an e-commerce platform with **50M users** and **500K products**. 

**Requirements:**
- When a user visits, you need to fetch their features in <50ms to serve recommendations
- Features include: last 10 products viewed, purchase history stats, browsing patterns
- Features update in real-time as users interact
- Data: 1B user events per day

**Current (Bad) Implementation:**
```sql
-- This query runs on EVERY page load
SELECT 
    u.user_id,
    COUNT(DISTINCT p.product_id) as products_viewed,
    AVG(p.price) as avg_price_viewed,
    COUNT(DISTINCT pur.product_id) as products_purchased,
    ARRAY_AGG(DISTINCT p.product_id ORDER BY e.timestamp DESC LIMIT 10) as recent_products
FROM users u
LEFT JOIN events e ON u.user_id = e.user_id AND e.event_type = 'view'
LEFT JOIN products p ON e.product_id = p.product_id
LEFT JOIN purchases pur ON u.user_id = pur.user_id
WHERE u.user_id = $1
  AND e.timestamp > NOW() - INTERVAL '30 days'
GROUP BY u.user_id;
```

**Issues with current approach:**
- Query takes 800ms+ (multiple JOINs, aggregations on huge tables)
- Scans millions of event rows per request
- 10,000 concurrent users = database meltdown

---

### **✅ OPTIMIZED SOLUTION**

**Strategy: Pre-compute features + Incremental updates**

#### **Step 1: Create Feature Store Table (Materialized)**
```sql
CREATE TABLE user_features (
    user_id INT PRIMARY KEY,
    products_viewed_30d INT DEFAULT 0,
    avg_price_viewed DECIMAL(10,2),
    products_purchased_total INT DEFAULT 0,
    recent_products INT[],  -- Last 10 product IDs
    last_updated TIMESTAMP DEFAULT NOW()
);

-- Index for fast lookups
CREATE INDEX idx_user_features_updated ON user_features(last_updated);
```

#### **Step 2: Populate Initial Features (One-time)**
```sql
INSERT INTO user_features (user_id, products_viewed_30d, avg_price_viewed, products_purchased_total, recent_products)
SELECT 
    u.user_id,
    COUNT(DISTINCT e.product_id) FILTER (WHERE e.timestamp > NOW() - INTERVAL '30 days'),
    AVG(p.price) FILTER (WHERE e.timestamp > NOW() - INTERVAL '30 days'),
    (SELECT COUNT(DISTINCT product_id) FROM purchases WHERE user_id = u.user_id),
    ARRAY(
        SELECT product_id 
        FROM events 
        WHERE user_id = u.user_id AND event_type = 'view'
        ORDER BY timestamp DESC 
        LIMIT 10
    )
FROM users u
LEFT JOIN events e ON u.user_id = e.user_id
LEFT JOIN products p ON e.product_id = p.product_id
GROUP BY u.user_id;
```

#### **Step 3: Real-time Updates via Trigger (Incremental)**
```sql
CREATE OR REPLACE FUNCTION update_user_features()
RETURNS TRIGGER AS $$
BEGIN
    -- Update on new product view
    IF NEW.event_type = 'view' THEN
        UPDATE user_features
        SET 
            products_viewed_30d = products_viewed_30d + 1,
            recent_products = (
                ARRAY[NEW.product_id] || 
                recent_products[1:9]  -- Keep last 10
            ),
            last_updated = NOW()
        WHERE user_id = NEW.user_id;
    END IF;
    
    -- Update on purchase
    IF NEW.event_type = 'purchase' THEN
        UPDATE user_features
        SET 
            products_purchased_total = products_purchased_total + 1,
            last_updated = NOW()
        WHERE user_id = NEW.user_id;
    END IF;
    
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_update_features
AFTER INSERT ON events
FOR EACH ROW EXECUTE FUNCTION update_user_features();
```

#### **Step 4: Lightning-Fast Read Query**
```sql
-- Sub-10ms query (simple primary key lookup)
SELECT * FROM user_features WHERE user_id = $1;
```

#### **Step 5: Periodic Refresh (Cleanup stale data)**
```sql
-- Cron job runs hourly to recalculate 30-day window
UPDATE user_features
SET products_viewed_30d = (
    SELECT COUNT(DISTINCT product_id)
    FROM events
    WHERE user_id = user_features.user_id
      AND event_type = 'view'
      AND timestamp > NOW() - INTERVAL '30 days'
)
WHERE last_updated < NOW() - INTERVAL '1 hour';
```

---

### **📊 PERFORMANCE COMPARISON**

| Metric | Before | After |
|--------|--------|-------|
| Query time | 800ms | 8ms |
| Database load (10K users) | 95% CPU | 5% CPU |
| Scalability | Max 100 req/s | 10,000+ req/s |
| Data freshness | Real-time | Real-time (trigger) |

---

### **🎯 KEY INSIGHTS**

**Why This Works:**
1. **Denormalization**: Pre-compute expensive aggregations
2. **Single table lookup**: No JOINs at read time
3. **Incremental updates**: Trigger updates one row, not millions
4. **Array storage**: PostgreSQL arrays perfect for "recent items" lists
5. **Periodic cleanup**: Balance freshness vs. performance

**Trade-offs:**
- ✅ 100x faster reads
- ✅ Scales to millions of users
- ❌ Extra storage (feature table)
- ❌ Write overhead (trigger on every event)
- ❌ Eventual consistency (30-day window updated hourly)

**When to use:**
- Read-heavy workloads (recommendations, dashboards)
- Features computed from historical data
- Acceptable slight staleness (<1 hour)

---

---

## **SCENARIO 2: EFFICIENT MODEL PREDICTION LOGGING AT SCALE**

### **❓ PROBLEM STATEMENT**

Your ML model serves **1 million predictions per hour**. You need to log every prediction for:
- A/B testing analysis
- Model monitoring (drift detection)
- Debugging incorrect predictions
- Compliance (audit trail)

**Current (Bad) Implementation:**
```python
# Python code making 1M individual INSERT calls per hour
for user_id, prediction in predictions:
    cursor.execute("""
        INSERT INTO predictions (user_id, model_id, predicted_class, probability, timestamp)
        VALUES (%s, %s, %s, %s, NOW())
    """, (user_id, model_id, predicted_class, probability))
    conn.commit()
```

**Issues:**
- Each INSERT is a separate transaction (network round-trip)
- 1M transactions/hour = 278 transactions/second
- Database overwhelmed with connections
- Prediction latency increases from 20ms to 500ms
- Disk I/O saturated (fsync on every commit)

---

### **✅ OPTIMIZED SOLUTION**

**Strategy: Batch inserts + Partitioning + Async writes**

#### **Step 1: Design Partitioned Table (By Day)**
```sql
CREATE TABLE predictions (
    prediction_id BIGSERIAL,
    user_id INT NOT NULL,
    model_id INT NOT NULL,
    predicted_class INT,
    probability REAL,
    features JSONB,  -- Store input features for debugging
    timestamp TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    PRIMARY KEY (timestamp, prediction_id)  -- Partition key first
) PARTITION BY RANGE (timestamp);

-- Create daily partitions (automated via script/extension)
CREATE TABLE predictions_2026_02_09 PARTITION OF predictions
FOR VALUES FROM ('2026-02-09') TO ('2026-02-10');

CREATE TABLE predictions_2026_02_10 PARTITION OF predictions
FOR VALUES FROM ('2026-02-10') TO ('2026-02-11');

-- Indexes (created on each partition automatically)
CREATE INDEX idx_predictions_user ON predictions(user_id, timestamp);
CREATE INDEX idx_predictions_model ON predictions(model_id, timestamp);
```

#### **Step 2: Batch Insert (Python Application)**
```python
import psycopg2
from psycopg2.extras import execute_values

# Collect predictions in memory (buffer)
prediction_buffer = []

for user_id, prediction in predictions:
    prediction_buffer.append((
        user_id, 
        model_id, 
        predicted_class, 
        probability, 
        features_json
    ))
    
    # Flush buffer every 1000 predictions
    if len(prediction_buffer) >= 1000:
        cursor = conn.cursor()
        execute_values(
            cursor,
            """
            INSERT INTO predictions (user_id, model_id, predicted_class, probability, features)
            VALUES %s
            """,
            prediction_buffer
        )
        conn.commit()
        prediction_buffer = []
```

#### **Step 3: Even Better - Use COPY (Binary Protocol)**
```python
from io import StringIO
import csv

# Build CSV in memory
csv_buffer = StringIO()
writer = csv.writer(csv_buffer)

for user_id, prediction in predictions:
    writer.writerow([user_id, model_id, predicted_class, probability, features_json])

csv_buffer.seek(0)

# Bulk copy (10x faster than INSERT)
cursor.copy_from(
    csv_buffer, 
    'predictions',
    columns=['user_id', 'model_id', 'predicted_class', 'probability', 'features'],
    sep=','
)
conn.commit()
```

#### **Step 4: Async Write Queue (Production-Grade)**
```python
# Use message queue (RabbitMQ, Kafka, AWS SQS)
# Decouple prediction serving from logging

# Prediction service (synchronous)
prediction = model.predict(features)
queue.publish({
    'user_id': user_id,
    'prediction': prediction,
    'timestamp': time.time()
})
return prediction  # Return immediately, don't wait for DB

# Separate worker process (asynchronous)
while True:
    batch = queue.consume(batch_size=10000, timeout=5)  # Wait max 5s
    if batch:
        bulk_insert_to_postgres(batch)
```

#### **Step 5: Partition Management (Auto-create future partitions)**
```sql
-- Function to create next day's partition
CREATE OR REPLACE FUNCTION create_partition_for_tomorrow()
RETURNS void AS $$
DECLARE
    tomorrow DATE := CURRENT_DATE + 1;
    partition_name TEXT := 'predictions_' || TO_CHAR(tomorrow, 'YYYY_MM_DD');
BEGIN
    EXECUTE format(
        'CREATE TABLE IF NOT EXISTS %I PARTITION OF predictions
         FOR VALUES FROM (%L) TO (%L)',
        partition_name,
        tomorrow,
        tomorrow + 1
    );
END;
$$ LANGUAGE plpgsql;

-- Schedule daily (via cron or pg_cron extension)
SELECT cron.schedule('create-partition', '0 0 * * *', 'SELECT create_partition_for_tomorrow()');
```

#### **Step 6: Efficient Querying (Partition Pruning)**
```sql
-- Query only touches today's partition (fast)
SELECT * FROM predictions
WHERE timestamp >= '2026-02-09'
  AND timestamp < '2026-02-10'
  AND user_id = 12345;

-- PostgreSQL automatically scans only predictions_2026_02_09 partition
```

#### **Step 7: Archive Old Partitions**
```sql
-- Detach partition (make it standalone table)
ALTER TABLE predictions DETACH PARTITION predictions_2026_01_01;

-- Move to archive storage or S3
-- Then drop
DROP TABLE predictions_2026_01_01;
```

---

### **📊 PERFORMANCE COMPARISON**

| Metric | Individual INSERTs | Batched (1000) | COPY | Async Queue |
|--------|-------------------|----------------|------|-------------|
| Throughput | 500/sec | 50,000/sec | 200,000/sec | 500,000/sec |
| Latency impact | +480ms | +10ms | +5ms | 0ms (async) |
| DB connections | 100+ | 10 | 5 | 2 |
| Disk I/O | Very high | Medium | Low | Very low |

---

### **🎯 KEY INSIGHTS**

**Why This Works:**
1. **Batching**: 1000x fewer network round-trips
2. **COPY command**: PostgreSQL's fastest bulk load method
3. **Partitioning**: Each day = separate table (fast queries, easy archival)
4. **Async writes**: Prediction serving decoupled from logging
5. **Partition pruning**: Queries only scan relevant partitions

**Trade-offs:**
- ✅ 1000x higher throughput
- ✅ Zero latency impact (async)
- ✅ Easy to archive old data
- ❌ Slightly delayed logging (buffered)
- ❌ Need partition management automation
- ❌ More complex application logic

**When to use:**
- High-volume logging (millions/hour)
- Time-series data (logs, events, predictions)
- When slight delay acceptable (<5 seconds)

---

---

## **SCENARIO 3: TRAINING DATA EXTRACTION WITH COMPLEX FEATURE ENGINEERING**

### **❓ PROBLEM STATEMENT**

You need to extract **training data for a churn prediction model**:
- 10M users
- 500M events (clicks, purchases, logins) over 2 years
- 50+ features per user (aggregations, time-based, sequence features)

**Required features:**
1. User demographics (age, location)
2. Engagement metrics (logins last 30/60/90 days)
3. Purchase behavior (total spent, avg order value, recency)
4. Sequence features (days between purchases, trending up/down)
5. Categorical encoding (favorite category, most used device)

**Current (Bad) Implementation:**
```sql
-- Takes 45 minutes to run, database freezes
SELECT 
    u.user_id,
    u.age,
    u.country,
    COUNT(DISTINCT e1.event_id) FILTER (WHERE e1.event_type = 'login' AND e1.timestamp > NOW() - INTERVAL '30 days') as logins_30d,
    COUNT(DISTINCT e2.event_id) FILTER (WHERE e2.event_type = 'login' AND e2.timestamp > NOW() - INTERVAL '60 days') as logins_60d,
    COUNT(DISTINCT e3.event_id) FILTER (WHERE e3.event_type = 'login' AND e3.timestamp > NOW() - INTERVAL '90 days') as logins_90d,
    COUNT(DISTINCT p.purchase_id) as total_purchases,
    SUM(p.amount) as total_spent,
    AVG(p.amount) as avg_order_value,
    MAX(p.purchase_date) as last_purchase_date,
    EXTRACT(EPOCH FROM (NOW() - MAX(p.purchase_date)))/86400 as days_since_purchase,
    -- ... 40+ more features
FROM users u
LEFT JOIN events e1 ON u.user_id = e1.user_id
LEFT JOIN events e2 ON u.user_id = e2.user_id
LEFT JOIN events e3 ON u.user_id = e3.user_id
LEFT JOIN purchases p ON u.user_id = p.user_id
WHERE u.registration_date < NOW() - INTERVAL '6 months'  -- Need history
GROUP BY u.user_id;
```

**Issues:**
- Multiple self-joins on massive events table
- Scans 500M events rows multiple times
- No indexes on timestamp + event_type combination
- Everything computed in one monstrous query
- Runs out of memory on large aggregations

---

### **✅ OPTIMIZED SOLUTION**

**Strategy: Divide and conquer with CTEs + Parallel processing + Smart indexing**

#### **Step 1: Create Indexes (One-time)**
```sql
-- Composite indexes for filtered aggregations
CREATE INDEX idx_events_user_type_time ON events(user_id, event_type, timestamp);
CREATE INDEX idx_purchases_user_date ON purchases(user_id, purchase_date);

-- Partial index for recent events (most queries use this)
CREATE INDEX idx_events_recent ON events(user_id, timestamp)
WHERE timestamp > NOW() - INTERVAL '1 year';
```

#### **Step 2: Modular CTEs (Break down problem)**
```sql
WITH 
-- CTE 1: User demographics (fast, small table)
user_base AS (
    SELECT user_id, age, country, registration_date
    FROM users
    WHERE registration_date < NOW() - INTERVAL '6 months'
),

-- CTE 2: Login features (single pass on events)
login_features AS (
    SELECT 
        user_id,
        COUNT(*) FILTER (WHERE timestamp > NOW() - INTERVAL '30 days') as logins_30d,
        COUNT(*) FILTER (WHERE timestamp > NOW() - INTERVAL '60 days') as logins_60d,
        COUNT(*) FILTER (WHERE timestamp > NOW() - INTERVAL '90 days') as logins_90d,
        MAX(timestamp) as last_login
    FROM events
    WHERE event_type = 'login'
      AND timestamp > NOW() - INTERVAL '90 days'  -- Filter early
    GROUP BY user_id
),

-- CTE 3: Purchase features (single pass on purchases)
purchase_features AS (
    SELECT 
        user_id,
        COUNT(*) as total_purchases,
        SUM(amount) as total_spent,
        AVG(amount) as avg_order_value,
        MAX(purchase_date) as last_purchase_date,
        MIN(purchase_date) as first_purchase_date
    FROM purchases
    GROUP BY user_id
),

-- CTE 4: Purchase recency (derived from CTE 3)
recency_features AS (
    SELECT 
        user_id,
        EXTRACT(EPOCH FROM (NOW() - last_purchase_date))/86400 as days_since_purchase,
        EXTRACT(EPOCH FROM (last_purchase_date - first_purchase_date))/86400 as customer_lifetime_days
    FROM purchase_features
),

-- CTE 5: Sequence features (time between purchases)
purchase_intervals AS (
    SELECT 
        user_id,
        purchase_date,
        LAG(purchase_date) OVER (PARTITION BY user_id ORDER BY purchase_date) as prev_purchase_date
    FROM purchases
),

interval_stats AS (
    SELECT 
        user_id,
        AVG(EXTRACT(EPOCH FROM (purchase_date - prev_purchase_date))/86400) as avg_days_between_purchases,
        STDDEV(EXTRACT(EPOCH FROM (purchase_date - prev_purchase_date))/86400) as stddev_days_between_purchases
    FROM purchase_intervals
    WHERE prev_purchase_date IS NOT NULL
    GROUP BY user_id
),

-- CTE 6: Categorical features (mode category)
category_preferences AS (
    SELECT DISTINCT ON (user_id)
        user_id,
        category as favorite_category
    FROM purchases p
    JOIN products pr ON p.product_id = pr.product_id
    GROUP BY user_id, category
    ORDER BY user_id, COUNT(*) DESC
)

-- Final join (all small aggregated tables)
SELECT 
    ub.user_id,
    ub.age,
    ub.country,
    COALESCE(lf.logins_30d, 0) as logins_30d,
    COALESCE(lf.logins_60d, 0) as logins_60d,
    COALESCE(lf.logins_90d, 0) as logins_90d,
    EXTRACT(EPOCH FROM (NOW() - lf.last_login))/86400 as days_since_login,
    COALESCE(pf.total_purchases, 0) as total_purchases,
    COALESCE(pf.total_spent, 0) as total_spent,
    COALESCE(pf.avg_order_value, 0) as avg_order_value,
    COALESCE(rf.days_since_purchase, 999) as days_since_purchase,
    COALESCE(rf.customer_lifetime_days, 0) as customer_lifetime_days,
    COALESCE(ist.avg_days_between_purchases, 0) as avg_purchase_interval,
    COALESCE(ist.stddev_days_between_purchases, 0) as purchase_interval_variance,
    COALESCE(cp.favorite_category, 'unknown') as favorite_category,
    u.churned as label
FROM user_base ub
LEFT JOIN login_features lf ON ub.user_id = lf.user_id
LEFT JOIN purchase_features pf ON ub.user_id = pf.user_id
LEFT JOIN recency_features rf ON ub.user_id = rf.user_id
LEFT JOIN interval_stats ist ON ub.user_id = ist.user_id
LEFT JOIN category_preferences cp ON ub.user_id = cp.user_id
LEFT JOIN users u ON ub.user_id = u.user_id;
```

#### **Step 3: Export Efficiently**
```sql
-- Export directly to CSV (bypass Python)
\copy (SELECT * FROM training_query) TO '/tmp/training_data.csv' CSV HEADER;

-- Or use Python with chunking
import pandas as pd
from sqlalchemy import create_engine

engine = create_engine('postgresql://...')

# Process in chunks to avoid memory issues
chunk_size = 100000
for chunk in pd.read_sql(query, engine, chunksize=chunk_size):
    # Process each chunk
    chunk.to_parquet(f'training_data_{chunk_index}.parquet')
```

#### **Step 4: Parallel Execution (Advanced)**
```sql
-- PostgreSQL parallel query execution (for huge datasets)
SET max_parallel_workers_per_gather = 4;
SET parallel_setup_cost = 100;
SET parallel_tuple_cost = 0.01;

-- Query automatically parallelized across CPU cores
```

#### **Step 5: Materialized View (If features reused)**
```sql
-- If training data extracted frequently
CREATE MATERIALIZED VIEW training_features AS
SELECT 
    -- (Full query from above)
    ...
FROM user_base ub
...;

-- Refresh weekly
REFRESH MATERIALIZED VIEW training_features;

-- Export in seconds
SELECT * FROM training_features;
```

---

### **📊 PERFORMANCE COMPARISON**

| Metric | Original Query | Optimized CTEs | + Indexes | + Materialized View |
|--------|----------------|----------------|-----------|---------------------|
| Execution time | 45 min | 8 min | 2 min | 5 sec (refresh: 2 min) |
| Memory usage | 32GB (OOM) | 8GB | 4GB | 100MB |
| CPU usage | 100% | 80% | 60% | 5% |
| Database impact | Blocks other queries | Minimal | None | None (pre-computed) |

---

### **🎯 KEY INSIGHTS**

**Why This Works:**
1. **CTEs break complexity**: Each CTE is simple, optimized independently
2. **Single pass per table**: No multiple self-joins on same table
3. **Filter early**: WHERE clauses reduce data before aggregation
4. **Smart indexes**: Composite indexes match WHERE + GROUP BY patterns
5. **Parallel execution**: PostgreSQL uses multiple cores
6. **Materialized views**: Pre-compute for repeated use

**Trade-offs:**
- ✅ 20x faster execution
- ✅ Predictable memory usage
- ✅ Doesn't block production queries
- ❌ More complex SQL (but more maintainable)
- ❌ One-time index creation overhead
- ❌ Materialized view staleness (if used)

**Best Practices:**
1. Always use `EXPLAIN ANALYZE` to find bottlenecks
2. Create indexes on `(user_id, timestamp)` for time-series data
3. Use `FILTER` clause instead of multiple CTEs for simple counts
4. Export directly to CSV/Parquet (avoid Python overhead)
5. Consider partitioning events table by month

---

---

## **SCENARIO 4: HANDLING DATA DRIFT - DETECTING FEATURE DISTRIBUTION CHANGES**

### **❓ PROBLEM STATEMENT**

Your fraud detection model is deployed in production. You need to **monitor feature drift** to detect when model performance degrades:

**Requirements:**
- Compare current feature distributions to training data baseline
- Alert when distributions shift significantly (KS-test, PSI)
- Track per-feature statistics over time
- Minimal performance impact on production

**Current data:**
- Training data: 5M transactions (static, from 6 months ago)
- Production data: 100K transactions per day
- 30 numeric features + 10 categorical features

**Current (Bad) Implementation:**
```python
# Python computes statistics on entire dataset daily
# Downloads 5M training rows + 100K production rows

training_data = pd.read_sql("SELECT * FROM training_transactions", conn)
production_data = pd.read_sql("SELECT * FROM transactions WHERE date = CURRENT_DATE", conn)

for feature in features:
    # Compute KS statistic
    ks_stat, p_value = ks_2samp(training_data[feature], production_data[feature])
    if p_value < 0.05:
        send_alert(f"Drift detected in {feature}")
```

**Issues:**
- Downloads millions of rows daily (network bottleneck)
- Python computes statistics (slow, memory-intensive)
- No historical tracking (can't see trends)
- Runs as batch job (delayed alerts)

---

### **✅ OPTIMIZED SOLUTION**

**Strategy: Pre-compute statistics in database + Incremental comparison**

#### **Step 1: Store Baseline Statistics (Training Data)**
```sql
CREATE TABLE feature_baseline (
    feature_name VARCHAR(100) PRIMARY KEY,
    feature_type VARCHAR(20),  -- 'numeric' or 'categorical'
    mean DECIMAL,
    stddev DECIMAL,
    min_value DECIMAL,
    max_value DECIMAL,
    percentile_25 DECIMAL,
    percentile_50 DECIMAL,
    percentile_75 DECIMAL,
    top_categories JSONB,  -- For categorical: {'value': count}
    computed_at TIMESTAMP DEFAULT NOW()
);

-- Populate from training data (one-time)
INSERT INTO feature_baseline (feature_name, feature_type, mean, stddev, min_value, max_value, percentile_25, percentile_50, percentile_75)
SELECT 
    'transaction_amount' as feature_name,
    'numeric' as feature_type,
    AVG(transaction_amount) as mean,
    STDDEV(transaction_amount) as stddev,
    MIN(transaction_amount) as min_value,
    MAX(transaction_amount) as max_value,
    PERCENTILE_CONT(0.25) WITHIN GROUP (ORDER BY transaction_amount) as percentile_25,
    PERCENTILE_CONT(0.50) WITHIN GROUP (ORDER BY transaction_amount) as percentile_50,
    PERCENTILE_CONT(0.75) WITHIN GROUP (ORDER BY transaction_amount) as percentile_75
FROM training_transactions;

-- For categorical features
INSERT INTO feature_baseline (feature_name, feature_type, top_categories)
SELECT 
    'merchant_category' as feature_name,
    'categorical' as feature_type,
    jsonb_object_agg(category, count) as top_categories
FROM (
    SELECT merchant_category as category, COUNT(*) as count
    FROM training_transactions
    GROUP BY merchant_category
    ORDER BY count DESC
    LIMIT 20  -- Top 20 categories
) t;
```

#### **Step 2: Daily Statistics Computation (Incremental)**
```sql
CREATE TABLE feature_daily_stats (
    stat_id SERIAL PRIMARY KEY,
    date DATE NOT NULL,
    feature_name VARCHAR(100) NOT NULL,
    mean DECIMAL,
    stddev DECIMAL,
    min_value DECIMAL,
    max_value DECIMAL,
    percentile_25 DECIMAL,
    percentile_50 DECIMAL,
    percentile_75 DECIMAL,
    drift_score DECIMAL,  -- KS statistic or PSI
    is_drifted BOOLEAN DEFAULT FALSE,
    UNIQUE(date, feature_name)
);

-- Daily job (runs once per day)
INSERT INTO feature_daily_stats (date, feature_name, mean, stddev, min_value, max_value, percentile_25, percentile_50, percentile_75)
SELECT 
    CURRENT_DATE as date,
    'transaction_amount' as feature_name,
    AVG(transaction_amount) as mean,
    STDDEV(transaction_amount) as stddev,
    MIN(transaction_amount) as min_value,
    MAX(transaction_amount) as max_value,
    PERCENTILE_CONT(0.25) WITHIN GROUP (ORDER BY transaction_amount) as percentile_25,
    PERCENTILE_CONT(0.50) WITHIN GROUP (ORDER BY transaction_amount) as percentile_50,
    PERCENTILE_CONT(0.75) WITHIN GROUP (ORDER BY transaction_amount) as percentile_75
FROM transactions
WHERE transaction_date = CURRENT_DATE;
```

#### **Step 3: Drift Detection (PostgreSQL Function)**
```sql
CREATE OR REPLACE FUNCTION detect_drift()
RETURNS TABLE(feature_name VARCHAR, drift_score DECIMAL, is_drifted BOOLEAN) AS $$
BEGIN
    RETURN QUERY
    WITH today_stats AS (
        SELECT * FROM feature_daily_stats WHERE date = CURRENT_DATE
    ),
    baseline AS (
        SELECT * FROM feature_baseline
    )
    SELECT 
        ts.feature_name,
        -- Simple drift metric: Standardized difference in means
        ABS(ts.mean - b.mean) / NULLIF(b.stddev, 0) as drift_score,
        CASE 
            WHEN ABS(ts.mean - b.mean) / NULLIF(b.stddev, 0) > 2 THEN TRUE  -- >2 std devs = drift
            ELSE FALSE
        END as is_drifted
    FROM today_stats ts
    JOIN baseline b ON ts.feature_name = b.feature_name;
END;
$$ LANGUAGE plpgsql;

-- Run daily
SELECT * FROM detect_drift();
```

#### **Step 4: Advanced Drift - Population Stability Index (PSI)**
```sql
-- PSI for categorical features
CREATE OR REPLACE FUNCTION calculate_psi(feature_name_param VARCHAR)
RETURNS DECIMAL AS $$
DECLARE
    psi_value DECIMAL := 0;
    baseline_dist JSONB;
    current_dist JSONB;
BEGIN
    -- Get baseline distribution
    SELECT top_categories INTO baseline_dist
    FROM feature_baseline
    WHERE feature_name = feature_name_param;
    
    -- Get current distribution
    SELECT jsonb_object_agg(category, count::FLOAT / total)
    INTO current_dist
    FROM (
        SELECT 
            merchant_category as category,
            COUNT(*) as count,
            SUM(COUNT(*)) OVER () as total
        FROM transactions
        WHERE transaction_date = CURRENT_DATE
        GROUP BY merchant_category
    ) t;
    
    -- Calculate PSI (simplified)
    -- PSI = SUM((current_pct - baseline_pct) * LN(current_pct / baseline_pct))
    -- Threshold: PSI > 0.2 indicates significant drift
    
    -- (Full implementation would iterate through categories)
    
    RETURN psi_value;
END;
$$ LANGUAGE plpgsql;
```

#### **Step 5: Alerting (Automated)**
```sql
-- Monitor view
CREATE VIEW drift_alerts AS
SELECT 
    feature_name,
    date,
    drift_score,
    mean as current_mean,
    (SELECT mean FROM feature_baseline WHERE feature_baseline.feature_name = fds.feature_name) as baseline_mean
FROM feature_daily_stats fds
WHERE is_drifted = TRUE
ORDER BY drift_score DESC;

-- Application polls this view
SELECT * FROM drift_alerts WHERE date = CURRENT_DATE;
```

#### **Step 6: Visualization Query (Trend Over Time)**
```sql
-- Feature trend last 30 days
SELECT 
    date,
    mean,
    stddev,
    drift_score
FROM feature_daily_stats
WHERE feature_name = 'transaction_amount'
  AND date > CURRENT_DATE - INTERVAL '30 days'
ORDER BY date;

-- Multi-feature dashboard
SELECT 
    feature_name,
    AVG(drift_score) as avg_drift_30d,
    MAX(drift_score) as max_drift_30d,
    COUNT(*) FILTER (WHERE is_drifted) as days_drifted
FROM feature_daily_stats
WHERE date > CURRENT_DATE - INTERVAL '30 days'
GROUP BY feature_name
ORDER BY avg_drift_30d DESC;
```

---

### **📊 PERFORMANCE COMPARISON**

| Metric | Python Daily Batch | Database Incremental |
|--------|-------------------|----------------------|
| Data transferred | 5GB (5M rows) | 10MB (stats only) |
| Computation time | 20 minutes | 2 minutes |
| Memory usage | 16GB (Python) | 500MB (Postgres) |
| Alert latency | 24 hours | Real-time |
| Historical tracking | Manual export | Automatic (table) |

---

### **🎯 KEY INSIGHTS**

**Why This Works:**
1. **Pre-compute statistics**: Database aggregates natively (faster than Python)
2. **Incremental approach**: Only process new data daily
3. **Store baselines**: No need to reload training data
4. **Database-native math**: Use built-in `PERCENTILE_CONT`, `STDDEV`
5. **SQL functions**: Encapsulate drift logic, reusable

**Trade-offs:**
- ✅ 10x faster computation
- ✅ 500x less data transfer
- ✅ Historical tracking built-in
- ✅ Easy to visualize (SQL queries)
- ❌ More complex initial setup
- ❌ Limited to simple drift metrics (KS test needs more code)

**When to use:**
- Production ML monitoring
- High-volume prediction systems
- When drift detection is critical (fraud, healthcare)
- Need historical trend analysis

**Extensions:**
1. Add **per-segment drift** (e.g., by merchant category)
2. **Two-sample KS test** in PostgreSQL (via extensions or Python UDF)
3. **Automatic retraining trigger** when drift exceeds threshold
4. **Feature importance weighting** (weight drift by feature importance)

---

---

## **SCENARIO 5: EFFICIENT A/B TEST ANALYSIS WITH STATISTICAL SIGNIFICANCE**

### **❓ PROBLEM STATEMENT**

You're running **100 concurrent A/B tests** for a recommendation system:
- Each test: Control vs Treatment (2 variants)
- Metrics: Click-through rate (CTR), conversion rate, revenue per user
- 1M users, 10M events per day
- Need to compute statistical significance daily

**Requirements:**
- Track metrics per experiment and variant
- Compute confidence intervals, p-values
- Detect statistical significance (95% confidence)
- Support early stopping (sequential testing)

**Current (Bad) Implementation:**
```python
# Python script runs daily
experiments = db.query("SELECT DISTINCT experiment_id FROM experiments WHERE active = TRUE")

for exp_id in experiments:
    control_data = db.query(f"""
        SELECT user_id, converted 
        FROM events 
        WHERE experiment_id = '{exp_id}' AND variant = 'control'
    """)
    
    treatment_data = db.query(f"""
        SELECT user_id, converted 
        FROM events 
        WHERE experiment_id = '{exp_id}' AND variant = 'treatment'
    """)
    
    # Compute in Python
    control_rate = control_data['converted'].mean()
    treatment_rate = treatment_data['converted'].mean()
    
    # Statistical test
    z_stat, p_value = proportions_ztest([treatment_successes, control_successes], 
                                         [treatment_total, control_total])
    
    # Store results
    db.execute(f"INSERT INTO experiment_results VALUES (...)")
```

**Issues:**
- Downloads millions of rows per experiment (100 experiments = 100M+ rows)
- Python computes statistics (memory intensive)
- No incremental updates (recomputes everything daily)
- Slow (takes hours)

---

### **✅ OPTIMIZED SOLUTION**

**Strategy: Incremental aggregation + Database statistics + Materialized summaries**

#### **Step 1: Event Tracking Table (Efficient Schema)**
```sql
CREATE TABLE experiment_events (
    event_id BIGSERIAL PRIMARY KEY,
    user_id INT NOT NULL,
    experiment_id VARCHAR(100) NOT NULL,
    variant VARCHAR(50) NOT NULL,
    event_type VARCHAR(50),  -- 'impression', 'click', 'conversion'
    revenue DECIMAL(10,2),
    timestamp TIMESTAMPTZ DEFAULT NOW()
);

-- Indexes for fast aggregation
CREATE INDEX idx_exp_events_exp_var ON experiment_events(experiment_id, variant, timestamp);
CREATE INDEX idx_exp_events_user ON experiment_events(user_id, experiment_id);

-- Partition by date (for archival)
-- (Partitioning code similar to Scenario 2)
```

#### **Step 2: Incremental Summary Table**
```sql
CREATE TABLE experiment_daily_summary (
    summary_id SERIAL PRIMARY KEY,
    experiment_id VARCHAR(100),
    variant VARCHAR(50),
    date DATE,
    users INT,
    impressions BIGINT,
    clicks BIGINT,
    conversions BIGINT,
    revenue DECIMAL(15,2),
    UNIQUE(experiment_id, variant, date)
);

-- Daily aggregation (runs once per day)
INSERT INTO experiment_daily_summary (experiment_id, variant, date, users, impressions, clicks, conversions, revenue)
SELECT 
    experiment_id,
    variant,
    DATE(timestamp) as date,
    COUNT(DISTINCT user_id) as users,
    COUNT(*) FILTER (WHERE event_type = 'impression') as impressions,
    COUNT(*) FILTER (WHERE event_type = 'click') as clicks,
    COUNT(*) FILTER (WHERE event_type = 'conversion') as conversions,
    SUM(revenue) FILTER (WHERE event_type = 'conversion') as revenue
FROM experiment_events
WHERE DATE(timestamp) = CURRENT_DATE - INTERVAL '1 day'
GROUP BY experiment_id, variant, DATE(timestamp)
ON CONFLICT (experiment_id, variant, date) DO UPDATE SET
    users = EXCLUDED.users,
    impressions = EXCLUDED.impressions,
    clicks = EXCLUDED.clicks,
    conversions = EXCLUDED.conversions,
    revenue = EXCLUDED.revenue;
```

#### **Step 3: Cumulative Statistics View**
```sql
CREATE VIEW experiment_stats AS
SELECT 
    experiment_id,
    variant,
    SUM(users) as total_users,
    SUM(impressions) as total_impressions,
    SUM(clicks) as total_clicks,
    SUM(conversions) as total_conversions,
    SUM(revenue) as total_revenue,
    
    -- Metrics
    ROUND(SUM(clicks)::NUMERIC / NULLIF(SUM(impressions), 0), 4) as ctr,
    ROUND(SUM(conversions)::NUMERIC / NULLIF(SUM(users), 0), 4) as conversion_rate,
    ROUND(SUM(revenue) / NULLIF(SUM(users), 0), 2) as revenue_per_user
FROM experiment_daily_summary
GROUP BY experiment_id, variant;
```

#### **Step 4: Statistical Significance (PostgreSQL Function)**
```sql
CREATE OR REPLACE FUNCTION calculate_ab_test_significance(exp_id VARCHAR)
RETURNS TABLE(
    experiment_id VARCHAR,
    control_rate DECIMAL,
    treatment_rate DECIMAL,
    lift_percent DECIMAL,
    z_score DECIMAL,
    p_value DECIMAL,
    is_significant BOOLEAN
) AS $$
DECLARE
    control_conversions INT;
    control_users INT;
    treatment_conversions INT;
    treatment_users INT;
    p_control DECIMAL;
    p_treatment DECIMAL;
    p_pooled DECIMAL;
    se DECIMAL;
    z DECIMAL;
    p DECIMAL;
BEGIN
    -- Get control data
    SELECT total_conversions, total_users INTO control_conversions, control_users
    FROM experiment_stats
    WHERE experiment_stats.experiment_id = exp_id AND variant = 'control';
    
    -- Get treatment data
    SELECT total_conversions, total_users INTO treatment_conversions, treatment_users
    FROM experiment_stats
    WHERE experiment_stats.experiment_id = exp_id AND variant = 'treatment';
    
    -- Calculate rates
    p_control := control_conversions::DECIMAL / NULLIF(control_users, 0);
    p_treatment := treatment_conversions::DECIMAL / NULLIF(treatment_users, 0);
    
    -- Pooled proportion
    p_pooled := (control_conversions + treatment_conversions)::DECIMAL / 
                NULLIF(control_users + treatment_users, 0);
    
    -- Standard error
    se := SQRT(p_pooled * (1 - p_pooled) * (1.0/control_users + 1.0/treatment_users));
    
    -- Z-score
    z := (p_treatment - p_control) / NULLIF(se, 0);
    
    -- P-value (approximate, two-tailed)
    -- Using normal distribution approximation
    -- For production, use external stats library or pre-computed lookup table
    p := 2 * (1 - 0.5 * (1 + SIGN(z) * SQRT(1 - EXP(-2.0/PI() * z * z))));  -- Approximation
    
    RETURN QUERY
    SELECT 
        exp_id,
        p_control,
        p_treatment,
        ROUND((p_treatment - p_control) / NULLIF(p_control, 0) * 100, 2) as lift_percent,
        z,
        p,
        (p < 0.05 AND ABS(z) > 1.96) as is_significant;
END;
$$ LANGUAGE plpgsql;

-- Run for all active experiments
SELECT * FROM calculate_ab_test_significance('homepage_redesign');
```

#### **Step 5: Dashboard Query (All Experiments)**
```sql
-- A/B test dashboard
WITH test_results AS (
    SELECT 
        e.experiment_id,
        e.experiment_name,
        e.start_date,
        c.total_users as control_users,
        t.total_users as treatment_users,
        c.conversion_rate as control_rate,
        t.conversion_rate as treatment_rate,
        ROUND((t.conversion_rate - c.conversion_rate) / NULLIF(c.conversion_rate, 0) * 100, 2) as lift_pct
    FROM experiments e
    JOIN experiment_stats c ON e.experiment_id = c.experiment_id AND c.variant = 'control'
    JOIN experiment_stats t ON e.experiment_id = t.experiment_id AND t.variant = 'treatment'
    WHERE e.active = TRUE
)
SELECT 
    tr.*,
    sig.z_score,
    sig.p_value,
    sig.is_significant
FROM test_results tr
CROSS JOIN LATERAL calculate_ab_test_significance(tr.experiment_id) sig
ORDER BY sig.p_value;
```

#### **Step 6: Sequential Testing (Early Stopping)**
```sql
-- Track daily progress for early stopping
CREATE TABLE experiment_daily_results (
    result_id SERIAL PRIMARY KEY,
    experiment_id VARCHAR(100),
    date DATE,
    days_running INT,
    treatment_lift_pct DECIMAL,
    z_score DECIMAL,
    p_value DECIMAL,
    is_significant BOOLEAN,
    recommendation VARCHAR(50)  -- 'continue', 'stop_winner', 'stop_loser'
);

-- Daily update with early stopping logic
INSERT INTO experiment_daily_results (experiment_id, date, days_running, treatment_lift_pct, z_score, p_value, is_significant, recommendation)
SELECT 
    e.experiment_id,
    CURRENT_DATE,
    CURRENT_DATE - e.start_date as days_running,
    sig.lift_percent,
    sig.z_score,
    sig.p_value,
    sig.is_significant,
    CASE 
        WHEN sig.is_significant AND sig.lift_percent > 5 AND days_running >= 7 THEN 'stop_winner'
        WHEN sig.is_significant AND sig.lift_percent < -5 AND days_running >= 7 THEN 'stop_loser'
        WHEN days_running > 30 THEN 'stop_inconclusive'
        ELSE 'continue'
    END as recommendation
FROM experiments e
CROSS JOIN LATERAL calculate_ab_test_significance(e.experiment_id) sig
WHERE e.active = TRUE;
```

#### **Step 7: Confidence Intervals**
```sql
CREATE OR REPLACE FUNCTION calculate_confidence_interval(
    successes INT,
    trials INT,
    confidence_level DECIMAL DEFAULT 0.95
)
RETURNS TABLE(lower_bound DECIMAL, upper_bound DECIMAL) AS $$
DECLARE
    p DECIMAL;
    z DECIMAL;
    se DECIMAL;
BEGIN
    p := successes::DECIMAL / NULLIF(trials, 0);
    z := 1.96;  -- 95% confidence (for 99%, use 2.576)
    se := SQRT(p * (1 - p) / trials);
    
    RETURN QUERY
    SELECT 
        GREATEST(0, p - z * se) as lower_bound,
        LEAST(1, p + z * se) as upper_bound;
END;
$$ LANGUAGE plpgsql;

-- Use in dashboard
SELECT 
    experiment_id,
    variant,
    conversion_rate,
    ci.lower_bound,
    ci.upper_bound
FROM experiment_stats es
CROSS JOIN LATERAL calculate_confidence_interval(es.total_conversions, es.total_users) ci;
```

---

### **📊 PERFORMANCE COMPARISON**

| Metric | Python Daily Batch | Database Incremental |
|--------|-------------------|----------------------|
| Data processed daily | 100M rows | 100K rows (new events) |
| Computation time | 3 hours | 5 minutes |
| Memory usage | 64GB (Python) | 2GB (Postgres) |
| Scalability | 100 experiments max | 10,000+ experiments |
| Real-time updates | No | Yes (trigger-based) |

---

### **🎯 KEY INSIGHTS**

**Why This Works:**
1. **Incremental aggregation**: Only process new data daily
2. **Materialized summaries**: Pre-computed totals per experiment/variant
3. **Database statistics**: Use SQL for z-tests, p-values
4. **Views for metrics**: Computed on-the-fly from summaries
5. **Efficient schema**: Summary tables are tiny (100 experiments × 2 variants × 30 days = 6K rows)

**Trade-offs:**
- ✅ 36x faster (3 hours → 5 minutes)
- ✅ 100x fewer rows processed
- ✅ Supports 100x more experiments
- ✅ Real-time dashboard updates
- ❌ Initial setup complexity
- ❌ Approximate p-values (for exact, use Python UDF)

**Extensions:**
1. **Bayesian A/B testing**: Store posterior distributions
2. **Multi-armed bandits**: Thompson sampling using Postgres
3. **Segmented analysis**: Per-demographic lift (age, location)
4. **Multiple metrics**: Primary (conversion) + guardrail (revenue, engagement)
5. **Sample size calculator**: Pre-experiment power analysis

**Production Best Practices:**
1. Use **partitioning** for experiment_events (archive old experiments)
2. **Index** on (experiment_id, variant) for fast aggregation
3. **Materialized view** for experiment_stats (refresh hourly)
4. **Alert system**: Trigger when p < 0.05 detected
5. **Automatic stopping**: Update experiment status based on recommendations

---

---

## **🎯 SUMMARY: COMMON PATTERNS ACROSS ALL 5 SCENARIOS**

### **Performance Optimization Patterns:**

1. **Pre-compute & Materialize**
   - Don't recompute on every query
   - Store aggregations in separate tables
   - Use materialized views for complex queries

2. **Incremental Updates**
   - Process only new/changed data
   - Use triggers for real-time updates
   - Daily batch jobs for aggregations

3. **Partitioning**
   - Time-based partitions for logs/events
   - Easy archival (detach old partitions)
   - Query only relevant partitions

4. **Smart Indexing**
   - Multi-column indexes: (user_id, timestamp)
   - Partial indexes: WHERE active = TRUE
   - BRIN for time-series
   - GIN for JSON/arrays

5. **Batch Operations**
   - Bulk INSERT instead of row-by-row
   - COPY for massive data loads
   - Batch size: 1000-10000 rows

6. **Denormalization (Strategic)**
   - Pre-join for hot queries
   - Store computed features
   - Trade storage for speed

7. **Database-Native Computation**
   - Use SQL aggregations, not Python
   - Window functions for analytics
   - Built-in statistical functions

8. **Async/Decoupled Architecture**
   - Message queues for writes
   - Separate read replicas
   - Cache hot data (Redis)

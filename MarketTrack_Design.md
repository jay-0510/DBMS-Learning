# PostgreSQL Complete Learning Guide
## MarketTrack Database Design & Implementation
### From Schema Design to Performance Optimization

---

## Table of Contents

This guide covers all topics needed for the MarketTrack DBMS assignment with real-life scenarios, syntax examples, and optimization tips.

---

# PHASE 1: SETUP & FOUNDATION

## Step 1: Environment Setup

### 🎯 Real-Life Scenario
You're hired as a database engineer at MarketTrack, an e-commerce startup. Your first day requires setting up your local development environment before you can start designing their database.

### Installing PostgreSQL

PostgreSQL is an open-source, enterprise-grade relational database system. Think of it as the foundation where all your company's critical data will live.

```bash
# For Windows: Download installer from postgresql.org
# For macOS: 
brew install postgresql

# For Linux:
sudo apt-get install postgresql
```

**Why PostgreSQL?**
- ✅ Free and open-source
- ✅ Reliable and ACID-compliant
- ✅ Handles complex queries that MarketTrack will need for analytics
- ✅ Advanced features: window functions, CTEs, materialized views

**Alternative:** MySQL is lighter but lacks advanced features like window functions (which you'll need for the assignment).

### Connecting to Database via pgAdmin

pgAdmin is a graphical interface for PostgreSQL. It's like having a visual control panel for your database instead of typing commands blindly.

```sql
-- Default connection details:
-- Host: localhost
-- Port: 5432
-- Username: postgres
-- Database: postgres
```

💡 **Pro Tip:** Always create a separate database for each project. Don't clutter the default 'postgres' database.

```sql
-- Create MarketTrack database
CREATE DATABASE markettrack;
```

---

# PHASE 2: DATABASE DESIGN

## Step 3: Understanding Data Types

### 🎯 Real-Life Scenario
MarketTrack's product manager tells you: "We need to store user names, product prices, order dates, and inventory counts." You need to choose the right data type for each field to save space and ensure data accuracy.

### Numeric Data Types

| Data Type | Use Case | Example | Storage Size |
|-----------|----------|---------|--------------|
| `INTEGER` | Whole numbers (inventory, quantity) | `stock_quantity INTEGER` | 4 bytes |
| `SMALLINT` | Small whole numbers (ratings 1-5) | `rating SMALLINT` | 2 bytes |
| `BIGINT` | Very large numbers | `total_views BIGINT` | 8 bytes |
| `NUMERIC(p,s)` | Exact decimal values (money) | `price NUMERIC(10,2)` | Variable |
| `REAL` | Approximate decimals | `latitude REAL` | 4 bytes |
| `DOUBLE PRECISION` | High precision decimals | `scientific_value DOUBLE PRECISION` | 8 bytes |

#### ⚠️ Critical Money Rule

```sql
-- ❌ WRONG: Never use FLOAT or REAL for money
price REAL  -- Causes $9.99 to become $9.989999

-- ✅ CORRECT: Always use NUMERIC for money
price NUMERIC(10,2)  -- Exact: $9.99 stays $9.99
```

**Why?** FLOAT and REAL have rounding errors due to binary representation. NUMERIC stores exact decimal values.

#### 💡 Space Optimization Tip

```sql
-- For ratings (1-5), use SMALLINT instead of INTEGER
rating SMALLINT CHECK (rating BETWEEN 1 AND 5)  -- 2 bytes
-- vs
rating INTEGER CHECK (rating BETWEEN 1 AND 5)   -- 4 bytes (wastes 2 bytes per row)

-- With 1 million reviews, you save 2 MB using SMALLINT!
```

### Text Data Types

| Data Type | Use Case | Example | Max Size |
|-----------|----------|---------|----------|
| `CHAR(n)` | Fixed-length text | `country_code CHAR(2)` | n characters |
| `VARCHAR(n)` | Variable-length text with limit | `email VARCHAR(255)` | n characters |
| `TEXT` | Unlimited text | `review_text TEXT` | 1 GB |

#### When to Use What

```sql
-- ✅ VARCHAR for limited-length fields
email VARCHAR(255)      -- Emails rarely exceed 255 chars
username VARCHAR(100)   -- Usernames typically short
phone VARCHAR(20)       -- Phone numbers

-- ✅ TEXT for unlimited content
product_description TEXT
review_text TEXT
blog_post_content TEXT
```

💡 **Performance Note:** In PostgreSQL, `VARCHAR` and `TEXT` have similar performance. Use `VARCHAR` to enforce business rules (email can't be 10,000 characters).

### Date & Time Data Types

```sql
-- For dates only (no time)
birth_date DATE
created_date DATE

-- For exact timestamps (date + time)
created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
updated_at TIMESTAMP

-- For time zones (international business)
created_at TIMESTAMPTZ  -- Stores in UTC, converts to user timezone

-- For time intervals
delivery_estimate INTERVAL  -- '2 days 3 hours'
```

#### 🎯 MarketTrack Use Case

```sql
-- Order tracking needs precise timestamps
CREATE TABLE orders (
    order_id SERIAL PRIMARY KEY,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,  -- When order was placed
    shipped_at TIMESTAMP,                            -- When order shipped
    delivered_at TIMESTAMP                           -- When order delivered
);

-- Analytics queries use DATE for grouping
SELECT 
    DATE(created_at) AS order_date,
    COUNT(*) AS daily_orders
FROM orders
GROUP BY DATE(created_at);
```

### Boolean & Other Types

```sql
-- Boolean
is_active BOOLEAN DEFAULT TRUE
is_verified BOOLEAN

-- UUID (Universally Unique Identifier)
session_id UUID DEFAULT gen_random_uuid()

-- JSON (for flexible data)
metadata JSONB  -- JSONB is faster than JSON for queries

-- Arrays
tags TEXT[]  -- Array of tags: {'electronics', 'sale', 'new'}
```

---

## Step 4: Creating Tables (Schema Design)

### 🎯 Real-Life Scenario
The product team says: "We need to track users, products they sell, orders placed, and reviews left." You need to create tables that represent these real-world entities and their relationships.

### Basic Table Creation

```sql
-- Creating the Users table
CREATE TABLE users (
    user_id SERIAL PRIMARY KEY,
    username VARCHAR(100) NOT NULL UNIQUE,
    email VARCHAR(255) NOT NULL UNIQUE,
    password_hash VARCHAR(255) NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

**Breaking it down:**

- `SERIAL PRIMARY KEY`: Auto-generates unique IDs (1, 2, 3...). Every user gets a unique identifier.
- `NOT NULL`: This field is mandatory. You can't create a user without an email.
- `UNIQUE`: No two users can have the same email. Prevents duplicate accounts.
- `DEFAULT CURRENT_TIMESTAMP`: Automatically sets the current time when a row is inserted.

💡 **Why SERIAL over INTEGER?** 
SERIAL automatically handles ID generation. You don't need to track "what was the last ID?" manually.

```sql
-- SERIAL is equivalent to:
user_id INTEGER PRIMARY KEY DEFAULT nextval('users_user_id_seq')
-- PostgreSQL creates the sequence automatically
```

### Products Table

```sql
CREATE TABLE products (
    product_id SERIAL PRIMARY KEY,
    product_name VARCHAR(255) NOT NULL,
    description TEXT,
    category VARCHAR(100),
    price NUMERIC(10,2) NOT NULL CHECK (price > 0),
    stock_quantity INTEGER NOT NULL DEFAULT 0 CHECK (stock_quantity >= 0),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

**New Concept - CHECK Constraint:**

- `CHECK (price > 0)`: Prevents negative or zero prices. Business rule enforced at database level.
- `CHECK (stock_quantity >= 0)`: Can't have -5 items in stock. Prevents data corruption.

#### 🎯 Why Database-Level Validation?

Even if your app has bugs, the database prevents invalid data. Multiple apps can access the same database safely.

```sql
-- This INSERT will fail:
INSERT INTO products (product_name, price, stock_quantity)
VALUES ('Test Product', -10, 5);
-- ERROR: new row violates check constraint "products_price_check"

-- This INSERT will succeed:
INSERT INTO products (product_name, price, stock_quantity)
VALUES ('iPhone 15', 999.99, 50);
```

---

## Step 5: Constraints & Relationships

### 🎯 Real-Life Scenario
An order references a user and contains products. If someone deletes a user, what happens to their orders? If a product is deleted, what happens to order history? You need to define these relationships and rules.

### Primary Keys

A primary key uniquely identifies each row. Think of it as a social security number for database rows.

```sql
-- Single column primary key
user_id SERIAL PRIMARY KEY

-- Composite primary key (for junction tables)
CREATE TABLE order_items (
    order_id INTEGER,
    product_id INTEGER,
    PRIMARY KEY (order_id, product_id)
);
```

### Foreign Keys

Foreign keys create relationships between tables. They ensure data integrity - you can't reference something that doesn't exist.

```sql
CREATE TABLE orders (
    order_id SERIAL PRIMARY KEY,
    user_id INTEGER NOT NULL,
    order_status VARCHAR(50) NOT NULL DEFAULT 'created',
    total_amount NUMERIC(10,2) NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    
    -- Foreign key relationship
    FOREIGN KEY (user_id) REFERENCES users(user_id)
        ON DELETE CASCADE
        ON UPDATE CASCADE
);
```

**Understanding ON DELETE Options:**

| Option | Behavior | Use Case |
|--------|----------|----------|
| `CASCADE` | Delete child records automatically | User deleted → all their orders deleted |
| `RESTRICT` | Prevent deletion if children exist | Can't delete user with orders |
| `SET NULL` | Set foreign key to NULL | Product deleted → order_items.product_id = NULL |
| `SET DEFAULT` | Set to default value | Rare, specific business logic |
| `NO ACTION` | Similar to RESTRICT | Default behavior |

#### 🎯 MarketTrack Decision Matrix

```sql
-- Orders → Users: Use RESTRICT
-- Don't allow user deletion if they have orders (regulatory/audit reasons)
FOREIGN KEY (user_id) REFERENCES users(user_id) ON DELETE RESTRICT

-- Cart Items → Products: Use CASCADE  
-- If product discontinued, remove from all carts
FOREIGN KEY (product_id) REFERENCES products(product_id) ON DELETE CASCADE

-- Order Items → Products: Use RESTRICT
-- Can't delete products that have been ordered (need history)
FOREIGN KEY (product_id) REFERENCES products(product_id) ON DELETE RESTRICT

-- Reviews → Orders: Use CASCADE
-- Order deleted → reviews on that order deleted too
FOREIGN KEY (order_id) REFERENCES orders(order_id) ON DELETE CASCADE
```

### The Order-Product Relationship (Many-to-Many)

One order contains multiple products. One product appears in multiple orders. This needs a **junction table** (also called bridge table or associative entity).

```sql
CREATE TABLE order_items (
    order_item_id SERIAL PRIMARY KEY,
    order_id INTEGER NOT NULL,
    product_id INTEGER NOT NULL,
    quantity INTEGER NOT NULL CHECK (quantity > 0),
    price_at_purchase NUMERIC(10,2) NOT NULL,
    
    FOREIGN KEY (order_id) REFERENCES orders(order_id) ON DELETE CASCADE,
    FOREIGN KEY (product_id) REFERENCES products(product_id) ON DELETE RESTRICT,
    
    -- Prevent duplicate entries
    UNIQUE (order_id, product_id)
);
```

#### 💡 Why `price_at_purchase`?

Product prices change over time. You need to store the **historical price** for accurate order totals. This is called "denormalization for historical accuracy."

```sql
-- Bad approach: Reference current price
SELECT 
    o.order_id,
    SUM(oi.quantity * p.price) AS total  -- WRONG! Uses current price
FROM orders o
JOIN order_items oi ON o.order_id = oi.order_id
JOIN products p ON oi.product_id = p.product_id;

-- Good approach: Store price at purchase
SELECT 
    o.order_id,
    SUM(oi.quantity * oi.price_at_purchase) AS total  -- Correct historical price
FROM orders o
JOIN order_items oi ON o.order_id = oi.order_id;
```

### Complete MarketTrack Schema

```sql
-- Reviews Table
CREATE TABLE reviews (
    review_id SERIAL PRIMARY KEY,
    user_id INTEGER NOT NULL,
    product_id INTEGER NOT NULL,
    order_id INTEGER NOT NULL,  -- Verify purchase before allowing review
    rating SMALLINT NOT NULL CHECK (rating BETWEEN 1 AND 5),
    review_text TEXT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    
    FOREIGN KEY (user_id) REFERENCES users(user_id) ON DELETE CASCADE,
    FOREIGN KEY (product_id) REFERENCES products(product_id) ON DELETE CASCADE,
    FOREIGN KEY (order_id) REFERENCES orders(order_id) ON DELETE CASCADE,
    
    -- Business rule: One review per product per order
    UNIQUE (order_id, product_id)
);
```

**Key Design Decisions:**

1. **SMALLINT for rating:** Saves space (2 bytes vs 4 bytes). Ratings are always 1-5.
2. **order_id reference:** Ensures users can only review products they purchased (verified purchase).
3. **UNIQUE constraint:** Prevents duplicate reviews for the same product in the same order.
4. **updated_at:** Track when reviews are edited for audit purposes.

---

# PHASE 3: DATA POPULATION

## Step 6: Inserting Data

### 🎯 Real-Life Scenario
You've built the database structure. Now the business team gives you actual data to populate: user signups, product catalog, order history. You need to insert this data efficiently and correctly.

### Single Row Insert

```sql
INSERT INTO users (username, email, password_hash)
VALUES ('john_doe', 'john@example.com', 'hashed_password_here');

-- The user_id is auto-generated (SERIAL)
-- created_at is auto-filled (DEFAULT CURRENT_TIMESTAMP)
```

### Multiple Rows Insert (Efficient!)

```sql
INSERT INTO products (product_name, description, price, stock_quantity, category)
VALUES 
    ('iPhone 15', 'Latest Apple smartphone', 999.99, 50, 'Electronics'),
    ('MacBook Pro', '14-inch M3 laptop', 1999.99, 25, 'Electronics'),
    ('AirPods Pro', 'Noise-cancelling earbuds', 249.99, 100, 'Electronics'),
    ('Samsung TV', '55-inch 4K Smart TV', 699.99, 30, 'Electronics'),
    ('Nike Sneakers', 'Running shoes', 129.99, 200, 'Clothing');
    
-- Inserting 5 products in ONE database trip instead of 5 separate INSERTs
```

💡 **Performance Tip:** Always batch inserts when possible. 

- ❌ Bad: 1000 separate INSERT statements = 1000 network round trips
- ✅ Good: One INSERT with 1000 rows = 1 network round trip

**Performance comparison:**
```sql
-- Slow: 1000 separate inserts (1-2 seconds)
INSERT INTO products (...) VALUES (...);
INSERT INTO products (...) VALUES (...);
-- ... 998 more times

-- Fast: Batch insert (0.1 seconds)
INSERT INTO products (...) VALUES 
    (...),
    (...),
    -- ... 998 more rows
    (...);
```

### Inserting with Foreign Key References

```sql
-- First, create an order
INSERT INTO orders (user_id, order_status, total_amount)
VALUES (1, 'created', 1249.98)
RETURNING order_id;  
-- Returns the generated order_id, e.g., 42

-- Then, add products to that order
INSERT INTO order_items (order_id, product_id, quantity, price_at_purchase)
VALUES 
    (42, 1, 1, 999.99),   -- 1 iPhone
    (42, 3, 1, 249.99);   -- 1 AirPods
```

**RETURNING Clause:** Super useful! Immediately gives you the auto-generated ID so you can use it in the next query. No need for a separate SELECT.

```sql
-- Alternative: Get the last inserted ID
INSERT INTO orders (user_id, order_status, total_amount)
VALUES (1, 'created', 1249.98);

-- Get the ID that was just inserted
SELECT lastval();  -- Returns the last sequence value
```

### INSERT with SELECT (Copy from another table)

```sql
-- Copy all electronics products to a separate table
INSERT INTO featured_products (product_id, product_name, price)
SELECT product_id, product_name, price
FROM products
WHERE category = 'Electronics' AND price > 500;
```

---

# PHASE 4: OPERATIONAL QUERIES

## Step 8: Basic Querying

### 🎯 Real-Life Scenario
The support team needs to find customer information. The inventory manager wants to see low-stock products. The marketing team wants active users. You write SELECT queries to retrieve this data.

### SELECT - Retrieving Data

```sql
-- Get all users (not recommended for large tables!)
SELECT * FROM users;

-- Get specific columns (better practice!)
SELECT user_id, username, email FROM users;

-- Give columns readable names (aliases)
SELECT 
    user_id AS "User ID",
    username AS "Username",
    email AS "Email Address",
    created_at AS "Member Since"
FROM users;
```

⚠️ **Avoid SELECT \*:**
- Fetches ALL columns (wasteful if you need only 3 out of 50)
- Breaks if table structure changes
- Harder to understand query purpose
- More network bandwidth

```sql
-- Bad: Fetching 1 GB of data
SELECT * FROM products;  -- 50 columns, need only 3

-- Good: Fetching 60 MB of data
SELECT product_id, product_name, price FROM products;  -- Only what you need
```

### ORDER BY - Sorting Results

```sql
-- Most expensive products first
SELECT product_name, price 
FROM products 
ORDER BY price DESC;  -- DESC = Descending (high to low)

-- Alphabetical order
SELECT product_name 
FROM products 
ORDER BY product_name ASC;  -- ASC = Ascending (A to Z) - default

-- Multiple sort criteria
SELECT product_name, category, price, stock_quantity
FROM products
ORDER BY category ASC, price DESC;
-- Sorted by category A-Z, then within each category by price high-to-low
```

### DISTINCT - Remove Duplicates

```sql
-- How many unique users have placed orders?
SELECT DISTINCT user_id FROM orders;

-- How many different product categories exist?
SELECT DISTINCT category FROM products;

-- Combination of columns
SELECT DISTINCT user_id, DATE(created_at) AS order_date
FROM orders;
-- Shows each user once per day they ordered
```

#### 🎯 Business Use

"How many unique customers ordered this month?" vs "How many total orders this month?"

```sql
-- Unique customers (might be 500)
SELECT COUNT(DISTINCT user_id) FROM orders
WHERE created_at >= '2025-02-01';

-- Total orders (might be 2000 - some customers ordered multiple times)
SELECT COUNT(*) FROM orders
WHERE created_at >= '2025-02-01';
```

### LIMIT - Restricting Results

```sql
-- Top 10 most expensive products
SELECT product_name, price
FROM products
ORDER BY price DESC
LIMIT 10;

-- First 5 recent orders
SELECT * FROM orders
ORDER BY created_at DESC
LIMIT 5;

-- Pagination: Skip first 20, get next 10 (page 3)
SELECT * FROM products
ORDER BY product_id
LIMIT 10 OFFSET 20;
```

**Pagination Formula:**
```sql
-- Page N with page_size items per page:
LIMIT page_size OFFSET (page_number - 1) * page_size

-- Example: Page 3 with 10 items per page
LIMIT 10 OFFSET 20  -- (3-1) * 10 = 20
```

---

## Step 9: Filtering Data

### 🎯 Real-Life Scenario
The warehouse manager asks: "Show me products with less than 10 items in stock." The finance team needs: "All orders over $500 from last month." The customer service team wants: "Users who joined this year." You use WHERE clauses.

### WHERE - Basic Filtering

```sql
-- Low stock alert
SELECT product_name, stock_quantity
FROM products
WHERE stock_quantity < 10;

-- High-value orders
SELECT order_id, total_amount, created_at
FROM orders
WHERE total_amount > 500;

-- Specific order status
SELECT order_id, user_id, order_status
FROM orders
WHERE order_status = 'shipped';

-- String matching (case-sensitive)
SELECT * FROM users
WHERE username = 'john_doe';
```

### AND / OR - Multiple Conditions

```sql
-- Products that are expensive AND low stock (urgent restock)
SELECT product_name, price, stock_quantity
FROM products
WHERE price > 100 AND stock_quantity < 20;

-- Orders that are either cancelled OR refunded
SELECT order_id, order_status, total_amount
FROM orders
WHERE order_status = 'cancelled' OR order_status = 'refunded';

-- Complex conditions with parentheses
SELECT product_name, price, stock_quantity, category
FROM products
WHERE (price > 500 OR stock_quantity < 5) 
  AND category = 'Electronics';
-- High-value OR low-stock electronics
```

💡 **Operator Precedence:** AND has higher precedence than OR. Use parentheses to be explicit!

```sql
-- WITHOUT parentheses (might not be what you want):
WHERE A OR B AND C
-- Evaluated as: A OR (B AND C)

-- WITH parentheses (clear intent):
WHERE (A OR B) AND C
-- Evaluated as you expect
```

### IN - Multiple Values

```sql
-- Instead of multiple ORs
SELECT * FROM orders
WHERE order_status IN ('shipped', 'delivered', 'processing');

-- Equivalent to (but cleaner than):
WHERE order_status = 'shipped' 
   OR order_status = 'delivered' 
   OR order_status = 'processing'

-- Exclude certain statuses
SELECT * FROM orders
WHERE order_status NOT IN ('cancelled', 'refunded');

-- With numbers
SELECT * FROM products
WHERE product_id IN (1, 5, 10, 15, 20);

-- With subquery
SELECT * FROM products
WHERE category IN (
    SELECT DISTINCT category 
    FROM products 
    WHERE price > 1000
);
```

### BETWEEN - Range Queries

```sql
-- Products in a price range
SELECT product_name, price
FROM products
WHERE price BETWEEN 50 AND 200;
-- Includes both 50 and 200 (inclusive)

-- Equivalent to:
WHERE price >= 50 AND price <= 200

-- Orders from last month
SELECT * FROM orders
WHERE created_at BETWEEN '2025-01-01' AND '2025-01-31 23:59:59';

-- Products with stock between 10 and 100 (optimal range)
SELECT product_name, stock_quantity
FROM products
WHERE stock_quantity BETWEEN 10 AND 100;
```

⚠️ **BETWEEN is inclusive:** `BETWEEN 50 AND 200` includes both 50 and 200.

### LIKE - Pattern Matching

```sql
-- Products starting with 'iPhone'
SELECT product_name FROM products
WHERE product_name LIKE 'iPhone%';

-- Products containing 'Pro'
SELECT product_name FROM products
WHERE product_name LIKE '%Pro%';

-- Emails from Gmail
SELECT email FROM users
WHERE email LIKE '%@gmail.com';

-- Usernames starting with 'a' and ending with 'n'
SELECT username FROM users
WHERE username LIKE 'a%n';

-- Case-insensitive matching (PostgreSQL specific)
SELECT username FROM users
WHERE username ILIKE 'john%';  
-- Matches 'John', 'JOHN', 'john', 'JoHn'
```

**Wildcards:**
- `%` matches any number of characters (including zero)
- `_` matches exactly one character

```sql
-- Find products with 3-letter names
SELECT product_name FROM products
WHERE product_name LIKE '___';  -- Three underscores

-- Find phone numbers in format XXX-XXX-XXXX
SELECT phone FROM users
WHERE phone LIKE '___-___-____';
```

### IS NULL - Missing Data

```sql
-- Products without descriptions
SELECT product_name, price FROM products
WHERE description IS NULL;

-- Reviews without text (only rating)
SELECT review_id, rating FROM reviews
WHERE review_text IS NULL;

-- Products with descriptions
SELECT product_name FROM products
WHERE description IS NOT NULL;

-- Users who haven't set a profile picture
SELECT username FROM users
WHERE profile_picture_url IS NULL;
```

⚠️ **Never use = NULL:**

```sql
-- ❌ WRONG: This will never return results
SELECT * FROM products
WHERE description = NULL;

-- ✅ CORRECT: Use IS NULL
SELECT * FROM products
WHERE description IS NULL;
```

**Why?** NULL means "unknown". You can't compare unknown to unknown (NULL = NULL is false!). Always use `IS NULL` or `IS NOT NULL`.

---

# PHASE 5: ANALYTICAL QUERIES

## Step 10: Joining Tables

### 🎯 Real-Life Scenario
The CEO asks: "Show me all orders with customer names and product details." Data is split across users, orders, order_items, and products tables. You need to JOIN them together to create a complete report.

### INNER JOIN - Matching Records Only

INNER JOIN returns only rows where there's a match in BOTH tables.

```sql
-- Get orders with customer names
SELECT 
    o.order_id,
    u.username,
    u.email,
    o.total_amount,
    o.order_status,
    o.created_at
FROM orders o
INNER JOIN users u ON o.user_id = u.user_id;

-- Table aliases (o, u) make queries shorter and more readable
```

**What's happening:**
1. For each order, find the matching user (using user_id)
2. If an order has no matching user (orphaned data), it's excluded
3. Result: Only valid orders with customer information

### Multiple Joins - Complete Order Report

```sql
-- Full order details: Customer, Products, Prices
SELECT 
    o.order_id,
    u.username AS customer_name,
    u.email AS customer_email,
    p.product_name,
    p.category,
    oi.quantity,
    oi.price_at_purchase,
    (oi.quantity * oi.price_at_purchase) AS line_total,
    o.order_status,
    o.created_at AS order_date
FROM orders o
INNER JOIN users u ON o.user_id = u.user_id
INNER JOIN order_items oi ON o.order_id = oi.order_id
INNER JOIN products p ON oi.product_id = p.product_id
WHERE o.order_status = 'delivered'
ORDER BY o.created_at DESC;
```

**Join Order Matters!** 
- Start with the main table (orders)
- Then join related tables
- Each join builds on previous ones

**Reading the query:**
1. Start with orders
2. Find the user who placed each order
3. Find all items in each order
4. Find product details for each item

### LEFT JOIN - Include Missing Matches

LEFT JOIN returns ALL rows from the left table, even if there's no match in the right table (returns NULL for missing matches).

```sql
-- All products, including those with NO orders
SELECT 
    p.product_name,
    p.price,
    p.stock_quantity,
    COUNT(oi.order_item_id) AS times_ordered,
    COALESCE(SUM(oi.quantity), 0) AS total_units_sold
FROM products p
LEFT JOIN order_items oi ON p.product_id = oi.product_id
GROUP BY p.product_id, p.product_name, p.price, p.stock_quantity
ORDER BY times_ordered DESC;

-- Products with 0 orders will show: times_ordered = 0, total_units_sold = 0
```

#### 🎯 Business Use Case
"Show ALL products and their sales counts." If you used INNER JOIN, products without orders would be missing from the report!

```sql
-- INNER JOIN: Only shows products that have been ordered (might be 80 products)
SELECT p.product_name, COUNT(oi.order_item_id)
FROM products p
INNER JOIN order_items oi ON p.product_id = oi.product_id
GROUP BY p.product_name;

-- LEFT JOIN: Shows all products, even unsold ones (all 100 products)
SELECT p.product_name, COUNT(oi.order_item_id)
FROM products p
LEFT JOIN order_items oi ON p.product_id = oi.product_id
GROUP BY p.product_name;
```

### RIGHT JOIN & FULL OUTER JOIN

```sql
-- RIGHT JOIN (less common, just reversed LEFT JOIN)
SELECT 
    p.product_name,
    oi.quantity
FROM order_items oi
RIGHT JOIN products p ON oi.product_id = p.product_id;
-- Same as: LEFT JOIN with tables reversed

-- FULL OUTER JOIN (all rows from both tables)
SELECT 
    u.username,
    o.order_id
FROM users u
FULL OUTER JOIN orders o ON u.user_id = o.user_id;
-- Shows users with no orders AND orders with no user (orphaned)
```

### Anti-Join - Find Non-Matches

Find records in Table A that DON'T have matching records in Table B.

**Method 1: LEFT JOIN with IS NULL**

```sql
-- Products that have NEVER been ordered
SELECT 
    p.product_id,
    p.product_name,
    p.price,
    p.stock_quantity
FROM products p
LEFT JOIN order_items oi ON p.product_id = oi.product_id
WHERE oi.order_item_id IS NULL;
```

**Method 2: NOT EXISTS (often faster!)**

```sql
-- Products that have NEVER been ordered
SELECT 
    product_id,
    product_name,
    price,
    stock_quantity
FROM products p
WHERE NOT EXISTS (
    SELECT 1 
    FROM order_items oi 
    WHERE oi.product_id = p.product_id
);
```

💡 **Performance Tip:** NOT EXISTS can be faster than LEFT JOIN + IS NULL because:
- It stops searching once it finds one match
- Doesn't need to fetch data from the right table
- Use EXPLAIN to compare both methods

#### 🎯 Assignment Use Cases

```sql
-- Users who have NEVER left a review
SELECT u.user_id, u.username
FROM users u
WHERE NOT EXISTS (
    SELECT 1 FROM reviews r WHERE r.user_id = u.user_id
);

-- Products in inventory but never ordered (dead stock)
SELECT p.product_name, p.stock_quantity
FROM products p
LEFT JOIN order_items oi ON p.product_id = oi.product_id
WHERE oi.order_item_id IS NULL AND p.stock_quantity > 0;
```

### SELF JOIN - Comparing Rows Within Same Table

```sql
-- Find users who placed multiple orders on the same day
SELECT 
    u.username,
    o1.order_id AS first_order,
    o2.order_id AS second_order,
    DATE(o1.created_at) AS order_date,
    o1.total_amount + o2.total_amount AS combined_total
FROM users u
JOIN orders o1 ON u.user_id = o1.user_id
JOIN orders o2 ON u.user_id = o2.user_id
WHERE DATE(o1.created_at) = DATE(o2.created_at)
  AND o1.order_id < o2.order_id;  -- Avoid duplicates (1,2) vs (2,1)

-- Find products more expensive than average in their category
SELECT 
    p1.product_name,
    p1.price,
    p1.category,
    AVG(p2.price) AS category_avg_price
FROM products p1
JOIN products p2 ON p1.category = p2.category
GROUP BY p1.product_id, p1.product_name, p1.price, p1.category
HAVING p1.price > AVG(p2.price);
```

---

## Step 11: Aggregations

### 🎯 Real-Life Scenario
The finance team asks: "What's our total revenue this month?" Marketing wants: "How many orders per customer?" Warehouse needs: "Average order value by product category?" You use aggregation functions.

### GROUP BY - Aggregating Data

```sql
-- Total revenue per day
SELECT 
    DATE(created_at) AS order_date,
    COUNT(*) AS total_orders,
    SUM(total_amount) AS daily_revenue,
    AVG(total_amount) AS avg_order_value,
    MIN(total_amount) AS smallest_order,
    MAX(total_amount) AS largest_order
FROM orders
WHERE order_status = 'delivered'
GROUP BY DATE(created_at)
ORDER BY order_date DESC;
```

**Aggregation Functions:**

| Function | Purpose | Example |
|----------|---------|---------|
| `COUNT(*)` | Number of rows | `COUNT(*) AS total_orders` |
| `COUNT(column)` | Non-NULL values in column | `COUNT(review_text)` |
| `SUM(column)` | Total of numeric values | `SUM(total_amount)` |
| `AVG(column)` | Average value | `AVG(price)` |
| `MIN(column)` | Minimum value | `MIN(created_at)` |
| `MAX(column)` | Maximum value | `MAX(price)` |

#### 🎯 Real Business Queries

```sql
-- Top spending customers
SELECT 
    u.username,
    COUNT(o.order_id) AS order_count,
    SUM(o.total_amount) AS total_spent,
    AVG(o.total_amount) AS avg_order_value,
    MAX(o.created_at) AS last_order_date
FROM users u
JOIN orders o ON u.user_id = o.user_id
WHERE o.order_status = 'delivered'
GROUP BY u.user_id, u.username
ORDER BY total_spent DESC
LIMIT 10;

-- Products by category with sales stats
SELECT 
    category,
    COUNT(*) AS product_count,
    AVG(price) AS avg_price,
    MIN(price) AS cheapest,
    MAX(price) AS most_expensive,
    SUM(stock_quantity) AS total_inventory
FROM products
GROUP BY category
ORDER BY product_count DESC;
```

### HAVING - Filtering Aggregated Results

WHERE filters individual rows BEFORE grouping.  
HAVING filters grouped results AFTER aggregation.

```sql
-- Customers who spent more than $1000 total
SELECT 
    u.username,
    COUNT(o.order_id) AS order_count,
    SUM(o.total_amount) AS total_spent
FROM users u
JOIN orders o ON u.user_id = o.user_id
WHERE o.order_status = 'delivered'  -- Filter individual orders first
GROUP BY u.user_id, u.username
HAVING SUM(o.total_amount) > 1000   -- Filter grouped results
ORDER BY total_spent DESC;
```

**WHERE vs HAVING:**

```sql
-- WHERE: Filter BEFORE grouping
WHERE order_status = 'delivered'    -- Filters individual orders
WHERE price > 100                   -- Filters individual products

-- HAVING: Filter AFTER grouping
HAVING SUM(total_amount) > 1000     -- Filters aggregated totals
HAVING COUNT(*) >= 5                -- Filters groups with count
HAVING AVG(rating) >= 4.0           -- Filters by average
```

### GROUPING SETS - Multiple Groupings

```sql
-- Get totals by category, by month, and grand total
SELECT 
    category,
    DATE_TRUNC('month', created_at) AS month,
    SUM(total_amount) AS revenue
FROM orders o
JOIN order_items oi ON o.order_id = oi.order_id
JOIN products p ON oi.product_id = p.product_id
GROUP BY GROUPING SETS (
    (category),                          -- Total by category
    (DATE_TRUNC('month', created_at)),   -- Total by month
    ()                                    -- Grand total
);
```

### ROLLUP - Hierarchical Totals

```sql
-- Revenue by category and product, with subtotals
SELECT 
    p.category,
    p.product_name,
    SUM(oi.quantity * oi.price_at_purchase) AS revenue
FROM order_items oi
JOIN products p ON oi.product_id = p.product_id
GROUP BY ROLLUP(p.category, p.product_name)
ORDER BY p.category, p.product_name;

-- Produces:
-- Electronics | iPhone      | 50000
-- Electronics | MacBook     | 80000
-- Electronics | NULL        | 130000  ← Category subtotal
-- Clothing    | Jeans       | 20000
-- Clothing    | Shirt       | 15000
-- Clothing    | NULL        | 35000   ← Category subtotal
-- NULL        | NULL        | 165000  ← Grand total
```

#### 🎯 Business Use
Perfect for financial reports with subtotals and grand totals. ROLLUP creates a hierarchy of aggregations.

### CUBE - All Possible Combinations

```sql
-- All combinations of category and month
SELECT 
    category,
    DATE_TRUNC('month', created_at) AS month,
    SUM(total_amount) AS revenue
FROM orders o
JOIN order_items oi ON o.order_id = oi.order_id
JOIN products p ON oi.product_id = p.product_id
GROUP BY CUBE(category, DATE_TRUNC('month', created_at));

-- Produces aggregations for:
-- 1. Each category
-- 2. Each month
-- 3. Each category + month combination
-- 4. Grand total
```

---

## Step 12: Subqueries

### 🎯 Real-Life Scenario
Marketing asks: "Find users who spent more than the average customer." You need to first calculate the average, then find users above it. This requires nested queries - subqueries.

### Simple Subquery in WHERE

```sql
-- Products more expensive than average
SELECT 
    product_name, 
    price,
    price - (SELECT AVG(price) FROM products) AS price_above_avg
FROM products
WHERE price > (SELECT AVG(price) FROM products)
ORDER BY price DESC;

-- The subquery runs first, returns one value (avg price)
-- Then the main query filters using that value
```

### Subquery in SELECT (Scalar Subquery)

```sql
-- Show each user with their order count and total spent
SELECT 
    u.username,
    u.email,
    (SELECT COUNT(*) 
     FROM orders o 
     WHERE o.user_id = u.user_id) AS order_count,
    (SELECT COALESCE(SUM(total_amount), 0)
     FROM orders o 
     WHERE o.user_id = u.user_id) AS total_spent
FROM users u
ORDER BY order_count DESC;

-- Each subquery must return exactly ONE value per row
```

### Correlated Subquery

The subquery references columns from the outer query. Runs once for EACH row of the outer query.

```sql
-- Products priced above their category average
SELECT 
    p.product_name, 
    p.price, 
    p.category,
    (SELECT AVG(price) FROM products WHERE category = p.category) AS category_avg
FROM products p
WHERE p.price > (
    SELECT AVG(price)
    FROM products
    WHERE category = p.category  -- References outer query!
)
ORDER BY p.category, p.price DESC;

-- For each product:
--   1. Calculate avg price in ITS category
--   2. Compare that product's price to its category average
```

⚠️ **Performance Warning:** Correlated subqueries can be slow on large tables. The subquery runs thousands of times. Consider using JOINs or window functions instead.

```sql
-- Slow: Correlated subquery (runs N times)
SELECT p.product_name, p.price
FROM products p
WHERE p.price > (
    SELECT AVG(price) FROM products WHERE category = p.category
);

-- Fast: Window function (runs once)
WITH category_avg AS (
    SELECT 
        product_id,
        product_name,
        price,
        category,
        AVG(price) OVER (PARTITION BY category) AS cat_avg
    FROM products
)
SELECT product_name, price, category
FROM category_avg
WHERE price > cat_avg;
```

### EXISTS - Checking for Existence

```sql
-- Users who have written at least one review
SELECT 
    u.username, 
    u.email
FROM users u
WHERE EXISTS (
    SELECT 1 
    FROM reviews r 
    WHERE r.user_id = u.user_id
);

-- EXISTS returns TRUE/FALSE, doesn't return actual data
-- Stops searching after finding first match (efficient!)
```

💡 **Why SELECT 1?** EXISTS only cares if rows exist, not their content. `SELECT 1` is convention - could be `SELECT *` but that's wasteful.

#### 🎯 Assignment Use Cases

```sql
-- Products that have reviews with ratings below 3
SELECT p.product_name, p.price
FROM products p
WHERE EXISTS (
    SELECT 1
    FROM reviews r
    WHERE r.product_id = p.product_id
      AND r.rating < 3
);

-- Orders that contain at least one Electronics product
SELECT o.order_id, o.total_amount
FROM orders o
WHERE EXISTS (
    SELECT 1
    FROM order_items oi
    JOIN products p ON oi.product_id = p.product_id
    WHERE oi.order_id = o.order_id
      AND p.category = 'Electronics'
);
```

### ANY / ALL Operators

```sql
-- Products more expensive than ANY product in Electronics
SELECT product_name, price, category
FROM products
WHERE price > ANY (
    SELECT price FROM products WHERE category = 'Electronics'
);
-- Returns products more expensive than the cheapest Electronics item

-- Products more expensive than ALL products in Electronics
SELECT product_name, price, category
FROM products
WHERE price > ALL (
    SELECT price FROM products WHERE category = 'Electronics'
);
-- Returns products more expensive than the most expensive Electronics item
```

---

## Step 13: Common Table Expressions (CTEs)

### 🎯 Real-Life Scenario
You need to generate a complex revenue report with multiple calculations: monthly totals, running totals, year-over-year growth. Nested subqueries become unreadable. CTEs let you break the logic into named, reusable steps.

### Basic CTE

CTEs are temporary named result sets that exist only during query execution. Think of them as temporary views.

```sql
WITH high_value_orders AS (
    SELECT 
        order_id,
        user_id,
        total_amount,
        created_at
    FROM orders
    WHERE total_amount > 500
      AND order_status = 'delivered'
)
SELECT 
    u.username,
    COUNT(h.order_id) AS high_value_count,
    SUM(h.total_amount) AS total_spent,
    AVG(h.total_amount) AS avg_order_value
FROM high_value_orders h
JOIN users u ON h.user_id = u.user_id
GROUP BY u.user_id, u.username
ORDER BY total_spent DESC;

-- The CTE 'high_value_orders' is like a temporary table
-- Makes complex queries readable and maintainable
```

### Multi-Step CTE Chain (Assignment Requirement!)

```sql
-- Calculate comprehensive product performance metrics
WITH 
-- Step 1: Calculate order quantities per product
product_orders AS (
    SELECT 
        product_id,
        SUM(quantity) AS total_sold,
        COUNT(DISTINCT order_id) AS order_count,
        MIN(price_at_purchase) AS min_sale_price,
        MAX(price_at_purchase) AS max_sale_price
    FROM order_items
    GROUP BY product_id
),
-- Step 2: Calculate revenue per product
product_revenue AS (
    SELECT 
        product_id,
        SUM(quantity * price_at_purchase) AS total_revenue,
        AVG(quantity * price_at_purchase) AS avg_revenue_per_order
    FROM order_items
    GROUP BY product_id
),
-- Step 3: Get review stats
product_reviews AS (
    SELECT 
        product_id,
        COUNT(*) AS review_count,
        AVG(rating) AS avg_rating,
        MIN(rating) AS min_rating,
        MAX(rating) AS max_rating
    FROM reviews
    GROUP BY product_id
),
-- Step 4: Calculate reorder rate (customers who bought again)
repeat_purchases AS (
    SELECT 
        product_id,
        COUNT(DISTINCT user_id) AS unique_customers,
        COUNT(*) AS total_purchases
    FROM order_items oi
    JOIN orders o ON oi.order_id = o.order_id
    GROUP BY product_id
)
-- Final query: Combine all metrics
SELECT 
    p.product_id,
    p.product_name,
    p.category,
    p.price AS current_price,
    p.stock_quantity,
    COALESCE(po.total_sold, 0) AS units_sold,
    COALESCE(po.order_count, 0) AS times_ordered,
    COALESCE(pr.total_revenue, 0) AS total_revenue,
    COALESCE(pr.avg_revenue_per_order, 0) AS avg_revenue_per_order,
    COALESCE(prev.review_count, 0) AS review_count,
    COALESCE(prev.avg_rating, 0) AS avg_rating,
    COALESCE(rp.unique_customers, 0) AS unique_customers,
    CASE 
        WHEN rp.unique_customers > 0 
        THEN ROUND((rp.total_purchases::NUMERIC / rp.unique_customers - 1) * 100, 2)
        ELSE 0 
    END AS repeat_purchase_rate
FROM products p
LEFT JOIN product_orders po ON p.product_id = po.product_id
LEFT JOIN product_revenue pr ON p.product_id = pr.product_id
LEFT JOIN product_reviews prev ON p.product_id = prev.product_id
LEFT JOIN repeat_purchases rp ON p.product_id = rp.product_id
ORDER BY total_revenue DESC;
```

**Why This is Powerful:**
1. Each CTE has a descriptive name
2. Complex logic broken into digestible steps
3. Can test each CTE individually
4. Easy to add/modify steps
5. Much more readable than nested subqueries

**COALESCE Explanation:** Handles products with no orders/reviews (shows 0 instead of NULL)

```sql
COALESCE(po.total_sold, 0)  
-- If total_sold is NULL (no orders), return 0
```

### Recursive CTE (Advanced)

```sql
-- Generate a series of dates (for reports with no data gaps)
WITH RECURSIVE date_series AS (
    -- Base case: Start date
    SELECT DATE '2025-01-01' AS date
    
    UNION ALL
    
    -- Recursive case: Add one day
    SELECT date + INTERVAL '1 day'
    FROM date_series
    WHERE date < DATE '2025-01-31'
)
SELECT 
    ds.date,
    COALESCE(COUNT(o.order_id), 0) AS order_count,
    COALESCE(SUM(o.total_amount), 0) AS daily_revenue
FROM date_series ds
LEFT JOIN orders o ON DATE(o.created_at) = ds.date
GROUP BY ds.date
ORDER BY ds.date;
-- Shows all days in January, even days with zero orders
```

#### ✅ Why CTEs Over Subqueries?

| Aspect | CTEs | Subqueries |
|--------|------|------------|
| **Readability** | ✅ Named, descriptive | ❌ Nested, hard to read |
| **Reusability** | ✅ Reference multiple times | ❌ Copy-paste duplication |
| **Debugging** | ✅ Test each CTE individually | ❌ Hard to isolate issues |
| **Maintenance** | ✅ Easy to add/modify steps | ❌ Risky to change |
| **Performance** | ✅ Can be optimized | ✅ Can be optimized |

---

## Step 14: Window Functions

### 🎯 Real-Life Scenario
The executive dashboard needs: "Top 10 products by revenue PER CATEGORY", "Running total of daily sales", "Month-over-month growth rates". Regular GROUP BY can't do this. You need window functions.

**Key Concept:** Window functions perform calculations across rows related to the current row WITHOUT collapsing them into groups.

### ROW_NUMBER - Sequential Numbering

```sql
-- Rank products by revenue
SELECT 
    p.product_name,
    SUM(oi.quantity * oi.price_at_purchase) AS total_revenue,
    ROW_NUMBER() OVER (ORDER BY SUM(oi.quantity * oi.price_at_purchase) DESC) AS rank
FROM products p
JOIN order_items oi ON p.product_id = oi.product_id
GROUP BY p.product_id, p.product_name
ORDER BY rank;

-- ROW_NUMBER assigns unique sequential numbers (1, 2, 3, ...)
-- OVER clause defines the window (all rows, ordered by revenue)
```

### RANK vs DENSE_RANK vs ROW_NUMBER

```sql
-- Compare ranking methods
WITH product_revenue AS (
    SELECT 
        p.product_name,
        SUM(oi.quantity * oi.price_at_purchase) AS revenue
    FROM products p
    JOIN order_items oi ON p.product_id = oi.product_id
    GROUP BY p.product_id, p.product_name
)
SELECT 
    product_name,
    revenue,
    ROW_NUMBER() OVER (ORDER BY revenue DESC) AS row_num,
    RANK() OVER (ORDER BY revenue DESC) AS rank,
    DENSE_RANK() OVER (ORDER BY revenue DESC) AS dense_rank
FROM product_revenue
ORDER BY revenue DESC;

-- Example output with ties:
-- Product      | Revenue | row_num | rank | dense_rank
-- iPhone       | 50000   | 1       | 1    | 1
-- MacBook      | 50000   | 2       | 1    | 1  (tie)
-- AirPods      | 30000   | 3       | 3    | 2  (rank skips 2)
-- iPad         | 25000   | 4       | 4    | 3
```

#### 🎯 When to Use Which

| Function | Behavior | Use Case |
|----------|----------|----------|
| `ROW_NUMBER()` | Always unique (1, 2, 3, 4) | Pagination, need unique numbers |
| `RANK()` | Ties get same rank, gaps after (1, 1, 3, 4) | Traditional ranking (Olympics-style) |
| `DENSE_RANK()` | Ties get same rank, no gaps (1, 1, 2, 3) | Ranking without gaps |

### PARTITION BY - Ranking Within Groups

```sql
-- Top 3 products per category by revenue
WITH ranked_products AS (
    SELECT 
        p.category,
        p.product_name,
        SUM(oi.quantity * oi.price_at_purchase) AS revenue,
        RANK() OVER (
            PARTITION BY p.category 
            ORDER BY SUM(oi.quantity * oi.price_at_purchase) DESC
        ) AS category_rank
    FROM products p
    JOIN order_items oi ON p.product_id = oi.product_id
    GROUP BY p.category, p.product_id, p.product_name
)
SELECT 
    category,
    product_name,
    revenue,
    category_rank
FROM ranked_products
WHERE category_rank <= 3
ORDER BY category, category_rank;

-- PARTITION BY resets ranking for each category
-- Electronics: ranks 1, 2, 3
-- Clothing: ranks 1, 2, 3 (independent from Electronics)
-- Books: ranks 1, 2, 3
```

### Running Totals (Cumulative Sum)

```sql
-- Daily revenue with running total
WITH daily_sales AS (
    SELECT 
        DATE(created_at) AS order_date,
        SUM(total_amount) AS daily_revenue
    FROM orders
    WHERE order_status = 'delivered'
    GROUP BY DATE(created_at)
)
SELECT 
    order_date,
    daily_revenue,
    SUM(daily_revenue) OVER (
        ORDER BY order_date
        ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
    ) AS running_total,
    SUM(daily_revenue) OVER (
        ORDER BY order_date
    ) AS running_total_shorthand  -- Same as above
FROM daily_sales
ORDER BY order_date;

-- Shows cumulative revenue over time
-- Jan 1: $1000 | Running: $1000
-- Jan 2: $1500 | Running: $2500
-- Jan 3: $2200 | Running: $4700
```

**Window Frame Syntax:**
```sql
ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
-- From start of partition to current row (running total)

ROWS BETWEEN 6 PRECEDING AND CURRENT ROW
-- Last 7 rows including current (7-day rolling average)

ROWS BETWEEN 1 PRECEDING AND 1 FOLLOWING
-- Previous, current, and next row (3-row moving average)
```

### Rolling Averages (Moving Average)

```sql
-- 7-day moving average of daily orders
WITH daily_order_counts AS (
    SELECT 
        DATE(created_at) AS order_date,
        COUNT(*) AS order_count
    FROM orders
    GROUP BY DATE(created_at)
)
SELECT 
    order_date,
    order_count,
    AVG(order_count) OVER (
        ORDER BY order_date
        ROWS BETWEEN 6 PRECEDING AND CURRENT ROW
    ) AS rolling_7day_avg,
    AVG(order_count) OVER (
        ORDER BY order_date
        ROWS BETWEEN 29 PRECEDING AND CURRENT ROW
    ) AS rolling_30day_avg
FROM daily_order_counts
ORDER BY order_date;

-- Smooths out daily fluctuations
-- Each day shows average of past N days including today
-- Great for trend analysis
```

#### 🎯 Assignment Use
Use rolling averages for **rating volatility detection**. If a product's 30-day average rating changes dramatically, it might indicate quality issues.

```sql
-- Detect rating volatility
WITH daily_ratings AS (
    SELECT 
        product_id,
        DATE(created_at) AS review_date,
        AVG(rating) AS daily_avg_rating
    FROM reviews
    GROUP BY product_id, DATE(created_at)
)
SELECT 
    product_id,
    review_date,
    daily_avg_rating,
    AVG(daily_avg_rating) OVER (
        PARTITION BY product_id
        ORDER BY review_date
        ROWS BETWEEN 29 PRECEDING AND CURRENT ROW
    ) AS rolling_30day_avg,
    daily_avg_rating - AVG(daily_avg_rating) OVER (
        PARTITION BY product_id
        ORDER BY review_date
        ROWS BETWEEN 29 PRECEDING AND CURRENT ROW
    ) AS deviation_from_avg
FROM daily_ratings
ORDER BY product_id, review_date;
```

### LAG / LEAD - Access Previous/Next Row

```sql
-- Month-over-month revenue growth
WITH monthly_revenue AS (
    SELECT 
        DATE_TRUNC('month', created_at) AS month,
        SUM(total_amount) AS monthly_revenue
    FROM orders
    WHERE order_status = 'delivered'
    GROUP BY DATE_TRUNC('month', created_at)
)
SELECT 
    month,
    monthly_revenue,
    LAG(monthly_revenue) OVER (ORDER BY month) AS prev_month_revenue,
    monthly_revenue - LAG(monthly_revenue) OVER (ORDER BY month) AS revenue_change,
    ROUND(
        (monthly_revenue - LAG(monthly_revenue) OVER (ORDER BY month)) * 100.0 
        / NULLIF(LAG(monthly_revenue) OVER (ORDER BY month), 0), 
        2
    ) AS growth_percentage
FROM monthly_revenue
ORDER BY month;

-- LAG gets value from previous row (offset 1 by default)
-- LEAD gets value from next row
-- Perfect for trend analysis and comparisons
```

**LAG/LEAD with offset:**
```sql
-- Compare to same month last year (offset 12 months)
LAG(monthly_revenue, 12) OVER (ORDER BY month) AS same_month_last_year

-- Look ahead 3 months
LEAD(monthly_revenue, 3) OVER (ORDER BY month) AS three_months_ahead
```

### FIRST_VALUE / LAST_VALUE

```sql
-- Compare each product's price to cheapest and most expensive in category
SELECT 
    product_name,
    category,
    price,
    FIRST_VALUE(price) OVER (
        PARTITION BY category 
        ORDER BY price ASC
        ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
    ) AS cheapest_in_category,
    LAST_VALUE(price) OVER (
        PARTITION BY category 
        ORDER BY price ASC
        ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
    ) AS most_expensive_in_category,
    price - FIRST_VALUE(price) OVER (
        PARTITION BY category 
        ORDER BY price ASC
        ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
    ) AS price_above_cheapest
FROM products
ORDER BY category, price;
```

### NTILE - Divide into Buckets

```sql
-- Divide customers into quartiles by spending
WITH customer_spending AS (
    SELECT 
        u.user_id,
        u.username,
        COALESCE(SUM(o.total_amount), 0) AS total_spent
    FROM users u
    LEFT JOIN orders o ON u.user_id = o.user_id AND o.order_status = 'delivered'
    GROUP BY u.user_id, u.username
)
SELECT 
    username,
    total_spent,
    NTILE(4) OVER (ORDER BY total_spent DESC) AS spending_quartile
    -- 1 = Top 25% (VIP customers)
    -- 2 = 25-50%
    -- 3 = 50-75%
    -- 4 = Bottom 25%
FROM customer_spending
ORDER BY total_spent DESC;
```

---

# PHASE 6: REPORTING & VIEWS

## Step 16: Creating Views

### 🎯 Real-Life Scenario
The analytics team repeatedly runs the same complex query joining 5 tables. The security team wants to hide sensitive customer data from junior analysts. The executive dashboard needs pre-calculated metrics. You create views.

### What is a View?

A view is a **saved query** that acts like a virtual table. It doesn't store data - it runs the query each time you access it.

### Basic View

```sql
-- Create view for customer order summary
CREATE VIEW customer_orders AS
SELECT 
    u.user_id,
    u.username,
    u.email,
    COUNT(o.order_id) AS total_orders,
    COALESCE(SUM(o.total_amount), 0) AS total_spent,
    COALESCE(AVG(o.total_amount), 0) AS avg_order_value,
    MAX(o.created_at) AS last_order_date,
    MIN(o.created_at) AS first_order_date
FROM users u
LEFT JOIN orders o ON u.user_id = o.user_id AND o.order_status = 'delivered'
GROUP BY u.user_id, u.username, u.email;

-- Now use it like a table:
SELECT * FROM customer_orders WHERE total_spent > 1000;

-- Simple query, complex logic hidden!
SELECT username, total_orders FROM customer_orders ORDER BY total_orders DESC LIMIT 10;
```

**Benefits:**
1. **Simplifies complex queries** - hide complexity behind simple SELECT
2. **Reusability** - write once, use everywhere
3. **Security** - control what data users can see
4. **Abstraction** - change underlying tables without breaking queries

### Security with Views

```sql
-- Hide sensitive data from junior analysts
CREATE VIEW customer_summary_public AS
SELECT 
    user_id,
    username,
    -- email is hidden!
    total_orders,
    total_spent
FROM customer_orders;

-- Junior analysts can only see this view, not the full customer_orders view
-- They get the data they need without seeing sensitive email addresses
```

### Updatable Views

Some views allow INSERT, UPDATE, DELETE operations:

```sql
-- Simple view that maps 1:1 to a table
CREATE VIEW active_products AS
SELECT product_id, product_name, price, stock_quantity
FROM products
WHERE stock_quantity > 0;

-- You can update through this view:
UPDATE active_products SET price = 899.99 WHERE product_id = 1;
-- This updates the underlying products table

-- Views with aggregations or joins are NOT updatable
```

### View Management

```sql
-- Replace an existing view
CREATE OR REPLACE VIEW customer_orders AS
SELECT 
    u.user_id,
    u.username,
    COUNT(o.order_id) AS total_orders
    -- Modified definition
FROM users u
LEFT JOIN orders o ON u.user_id = o.user_id
GROUP BY u.user_id, u.username;

-- Drop a view
DROP VIEW IF EXISTS customer_orders;

-- List all views
SELECT table_name 
FROM information_schema.views 
WHERE table_schema = 'public';
```

---

## Materialized Views (Assignment Critical!)

### What is a Materialized View?

A materialized view **physically stores** the query results. It's faster than regular views but needs manual refreshing.

**Regular View vs Materialized View:**

| Aspect | Regular View | Materialized View |
|--------|--------------|-------------------|
| **Data Storage** | No storage, runs query each time | Stores results physically |
| **Freshness** | Always fresh (runs query) | Can be stale until refreshed |
| **Query Speed** | Depends on query complexity | Very fast (pre-computed) |
| **Disk Usage** | None | Uses disk space |
| **Use Case** | Simple queries, fresh data needed | Complex queries, dashboards |

### Creating Materialized Views

```sql
-- Create materialized view for analytics dashboard
CREATE MATERIALIZED VIEW daily_sales_summary AS
SELECT 
    DATE(o.created_at) AS sale_date,
    COUNT(DISTINCT o.order_id) AS order_count,
    COUNT(DISTINCT o.user_id) AS customer_count,
    SUM(o.total_amount) AS total_revenue,
    AVG(o.total_amount) AS avg_order_value,
    SUM(oi.quantity) AS items_sold,
    COUNT(DISTINCT p.category) AS categories_sold
FROM orders o
JOIN order_items oi ON o.order_id = oi.order_id
JOIN products p ON oi.product_id = p.product_id
WHERE o.order_status = 'delivered'
GROUP BY DATE(o.created_at);

-- Query it instantly (no computation!)
SELECT * FROM daily_sales_summary 
WHERE sale_date >= CURRENT_DATE - INTERVAL '30 days'
ORDER BY sale_date DESC;

-- Refresh when data changes
REFRESH MATERIALIZED VIEW daily_sales_summary;
```

### 🎯 When to Use Materialized Views

**Use materialized views when:**
- ✅ Heavy aggregations that take minutes to compute
- ✅ Dashboard metrics that don't need real-time accuracy
- ✅ Reports generated once per day/hour
- ✅ Read-heavy workloads
- ✅ Complex joins across many tables

**Don't use materialized views when:**
- ❌ Need real-time data
- ❌ Data changes constantly
- ❌ Simple queries that run fast anyway
- ❌ Limited disk space

### Refresh Strategies

```sql
-- Manual refresh (blocks reads during refresh)
REFRESH MATERIALIZED VIEW daily_sales_summary;

-- Concurrent refresh (doesn't lock the view, users can still query)
REFRESH MATERIALIZED VIEW CONCURRENTLY daily_sales_summary;
-- ⚠️ Requires unique index for concurrent refresh!

CREATE UNIQUE INDEX idx_daily_sales_date 
ON daily_sales_summary(sale_date);
```

**Refresh Schedule Options:**

```sql
-- 1. Manual (run when needed)
REFRESH MATERIALIZED VIEW daily_sales_summary;

-- 2. Scheduled (via cron job or pg_cron extension)
-- Run every night at 2 AM:
SELECT cron.schedule('refresh-sales', '0 2 * * *', 
    'REFRESH MATERIALIZED VIEW daily_sales_summary');

-- 3. Trigger-based (refresh on data changes)
CREATE OR REPLACE FUNCTION refresh_sales_summary()
RETURNS TRIGGER AS $$
BEGIN
    REFRESH MATERIALIZED VIEW CONCURRENTLY daily_sales_summary;
    RETURN NULL;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER refresh_sales_on_order
AFTER INSERT OR UPDATE ON orders
FOR EACH STATEMENT
EXECUTE FUNCTION refresh_sales_summary();
```

### Indexing Materialized Views

💡 **Best Practice:** Add indexes to materialized views! They're real tables, so they benefit from indexes.

```sql
-- Create materialized view
CREATE MATERIALIZED VIEW product_performance AS
SELECT 
    p.product_id,
    p.product_name,
    p.category,
    COUNT(DISTINCT o.order_id) AS order_count,
    SUM(oi.quantity) AS units_sold,
    SUM(oi.quantity * oi.price_at_purchase) AS total_revenue
FROM products p
LEFT JOIN order_items oi ON p.product_id = oi.product_id
LEFT JOIN orders o ON oi.order_id = o.order_id AND o.order_status = 'delivered'
GROUP BY p.product_id, p.product_name, p.category;

-- Add indexes for fast lookups
CREATE INDEX idx_product_perf_category ON product_performance(category);
CREATE INDEX idx_product_perf_revenue ON product_performance(total_revenue DESC);
CREATE UNIQUE INDEX idx_product_perf_id ON product_performance(product_id);
-- The unique index enables REFRESH MATERIALIZED VIEW CONCURRENTLY
```

---

# PHASE 7: DATA MANAGEMENT

## Step 18: Update & Delete Operations

### 🎯 Real-Life Scenario
Inventory levels need constant updates as products sell. Order statuses change as they ship and deliver. Cancelled orders must be marked or removed. You use UPDATE and DELETE statements.

### UPDATE - Modifying Data

```sql
-- Update single row
UPDATE products 
SET price = 899.99 
WHERE product_id = 1;

-- Update multiple columns
UPDATE products 
SET 
    price = 899.99,
    stock_quantity = stock_quantity - 5,
    updated_at = CURRENT_TIMESTAMP
WHERE product_id = 1;

-- Update based on condition
UPDATE orders 
SET order_status = 'shipped',
    shipped_at = CURRENT_TIMESTAMP
WHERE order_status = 'processing' 
  AND created_at < CURRENT_DATE - INTERVAL '2 days';

-- Bulk update
UPDATE products 
SET price = price * 1.10  -- 10% price increase
WHERE category = 'Electronics';
```

### UPDATE with JOIN

```sql
-- Update product stock after order is placed
UPDATE products p
SET stock_quantity = stock_quantity - oi.quantity
FROM order_items oi
WHERE p.product_id = oi.product_id
  AND oi.order_id = 12345;

-- Mark users as VIP if they spent over $5000
UPDATE users u
SET user_type = 'VIP'
FROM (
    SELECT user_id, SUM(total_amount) AS total_spent
    FROM orders
    WHERE order_status = 'delivered'
    GROUP BY user_id
    HAVING SUM(total_amount) > 5000
) spending
WHERE u.user_id = spending.user_id;
```

### DELETE - Removing Data

```sql
-- Delete single row
DELETE FROM products WHERE product_id = 999;

-- Delete with condition
DELETE FROM orders 
WHERE order_status = 'cancelled' 
  AND created_at < CURRENT_DATE - INTERVAL '1 year';

-- Delete all rows (careful!)
DELETE FROM temp_table;  -- Removes all data, but slower
TRUNCATE temp_table;     -- Faster, but can't be rolled back
```

### DELETE with JOIN

```sql
-- Delete reviews for products that no longer exist
DELETE FROM reviews r
USING products p
WHERE r.product_id = p.product_id
  AND p.stock_quantity = 0
  AND p.updated_at < CURRENT_DATE - INTERVAL '2 years';

-- Delete duplicate rows (keep oldest)
DELETE FROM users u1
USING users u2
WHERE u1.email = u2.email
  AND u1.user_id > u2.user_id;
-- Keeps user with lower ID, deletes duplicates with higher IDs
```

---

## Step 19: Transactions

### 🎯 Real-Life Scenario
A customer places an order: you need to create the order record, add order items, update inventory, and charge payment. If ANY step fails, ALL steps should be rolled back. You use transactions to ensure data consistency.

### What is a Transaction?

A transaction is a group of SQL statements that execute as a single unit. Either ALL succeed or ALL fail (atomicity).

**ACID Properties:**
- **A**tomicity: All or nothing
- **C**onsistency: Data remains valid
- **I**solation: Transactions don't interfere
- **D**urability: Committed data persists

### Basic Transaction

```sql
BEGIN;  -- Start transaction

    -- Create order
    INSERT INTO orders (user_id, order_status, total_amount)
    VALUES (1, 'created', 1249.98);
    
    -- Add order items
    INSERT INTO order_items (order_id, product_id, quantity, price_at_purchase)
    VALUES 
        (CURRVAL('orders_order_id_seq'), 1, 1, 999.99),
        (CURRVAL('orders_order_id_seq'), 3, 1, 249.99);
    
    -- Update inventory
    UPDATE products SET stock_quantity = stock_quantity - 1 WHERE product_id = 1;
    UPDATE products SET stock_quantity = stock_quantity - 1 WHERE product_id = 3;

COMMIT;  -- Save all changes

-- If anything fails, all changes are rolled back automatically
```

### Rollback on Error

```sql
BEGIN;

    UPDATE products SET stock_quantity = stock_quantity - 10 WHERE product_id = 1;
    
    -- Oops, this will fail (violates CHECK constraint)
    UPDATE products SET stock_quantity = -5 WHERE product_id = 2;
    
ROLLBACK;  -- Manually undo all changes

-- The first UPDATE is also undone!
-- Data remains unchanged
```

### Savepoints (Partial Rollback)

```sql
BEGIN;

    UPDATE products SET price = 899.99 WHERE product_id = 1;
    
    SAVEPOINT before_risky_operation;
    
    UPDATE products SET stock_quantity = 0 WHERE category = 'Electronics';
    -- Oops, too aggressive!
    
    ROLLBACK TO before_risky_operation;  -- Undo only this UPDATE
    -- The price update is still active
    
    UPDATE products SET stock_quantity = 0 WHERE product_id = 999;
    
COMMIT;  -- Save price update and last stock update, but not the aggressive one
```

### Real-World Example: Order Processing

```sql
BEGIN;

    -- 1. Create order
    INSERT INTO orders (user_id, order_status, total_amount)
    VALUES (123, 'processing', 299.98)
    RETURNING order_id INTO @order_id;
    
    -- 2. Reserve inventory
    UPDATE products 
    SET stock_quantity = stock_quantity - 2
    WHERE product_id = 45 AND stock_quantity >= 2;
    
    -- Check if update was successful (enough stock)
    IF (SELECT stock_quantity FROM products WHERE product_id = 45) < 0 THEN
        ROLLBACK;
        RAISE EXCEPTION 'Insufficient stock';
    END IF;
    
    -- 3. Add order items
    INSERT INTO order_items (order_id, product_id, quantity, price_at_purchase)
    VALUES (@order_id, 45, 2, 149.99);
    
    -- 4. Charge payment (simulated)
    -- If payment fails, the transaction rolls back automatically
    
COMMIT;  -- Everything succeeded!
```

---

# PHASE 8: PERFORMANCE OPTIMIZATION

## Step 21: Index Design

### 🎯 Real-Life Scenario
Users complain the product search page is slow. The order history page takes 10 seconds to load. The daily revenue report times out. The database has millions of rows. You need indexes to speed up queries.

### What is an Index?

An index is like a book's index - instead of reading every page to find a topic, you look it up quickly. Database indexes work the same way.

**Without Index:** Database scans every row (sequential scan) - SLOW  
**With Index:** Database jumps directly to matching rows (index scan) - FAST

### When to Create Indexes

Create indexes on columns that are:
- ✅ Frequently used in WHERE clauses
- ✅ Used in JOIN conditions (foreign keys)
- ✅ Used in ORDER BY clauses
- ✅ Used in GROUP BY clauses

Don't index:
- ❌ Columns rarely queried
- ❌ Small tables (< 1000 rows)
- ❌ Columns with low cardinality (few distinct values, like boolean)
- ❌ Columns updated frequently (index maintenance overhead)

### Basic Index Creation

```sql
-- Index on email for user lookups
CREATE INDEX idx_users_email ON users(email);

-- Now this query is fast:
SELECT * FROM users WHERE email = 'john@example.com';
-- Before: Scans all 1 million users
-- After: Finds user in microseconds

-- Index on order status for filtering
CREATE INDEX idx_orders_status ON orders(order_status);

-- Index on foreign key for JOINs (very important!)
CREATE INDEX idx_orders_user_id ON orders(user_id);
CREATE INDEX idx_order_items_order_id ON order_items(order_id);
CREATE INDEX idx_order_items_product_id ON order_items(product_id);

-- Index on timestamp for date range queries
CREATE INDEX idx_orders_created_at ON orders(created_at);
```

### Composite Indexes (Multi-Column)

When queries filter on multiple columns together, use a composite index:

```sql
-- Composite index for common query pattern
CREATE INDEX idx_orders_user_status 
ON orders(user_id, order_status);

-- Efficient for queries like:
SELECT * FROM orders 
WHERE user_id = 123 AND order_status = 'delivered';
-- Uses the composite index

-- Also helps queries filtering only on first column:
SELECT * FROM orders WHERE user_id = 123;
-- Can use the composite index (partial match)

-- Doesn't help queries filtering only on second column:
SELECT * FROM orders WHERE order_status = 'delivered';
-- Can't use idx_orders_user_status (need idx_orders_status instead)
```

🔑 **Column Order Matters!**

Put most selective column first (the one that filters the most rows):

```sql
-- Good: user_id is very selective (filters to one user's orders)
CREATE INDEX idx_orders_user_status ON orders(user_id, order_status);

-- Less optimal: status is less selective (many orders have same status)
CREATE INDEX idx_orders_status_user ON orders(order_status, user_id);
```

### Partial Indexes

Index only rows that match a condition - smaller, faster:

```sql
-- Only index active orders (not delivered/cancelled)
CREATE INDEX idx_active_orders 
ON orders(user_id, created_at) 
WHERE order_status IN ('created', 'processing', 'shipped');

-- Perfect for queries like:
SELECT * FROM orders 
WHERE user_id = 123 
  AND order_status = 'processing'
  AND created_at > '2025-01-01';

-- Smaller index = faster queries + less disk space
```

💡 **Use Case:** If 90% of orders are delivered and you only query active orders, why index delivered ones? Partial indexes save space and improve speed.

### Unique Indexes

```sql
-- Enforce uniqueness AND create index
CREATE UNIQUE INDEX idx_users_email_unique ON users(email);

-- Faster than regular index for lookups
-- Also prevents duplicate emails at database level
```

### Indexes on Expressions

```sql
-- Index on computed column
CREATE INDEX idx_products_lower_name 
ON products(LOWER(product_name));

-- Makes case-insensitive searches fast:
SELECT * FROM products 
WHERE LOWER(product_name) LIKE '%iphone%';

-- Index on date part
CREATE INDEX idx_orders_date 
ON orders(DATE(created_at));

-- Fast queries grouping by date:
SELECT DATE(created_at), COUNT(*) 
FROM orders 
GROUP BY DATE(created_at);
```

### Index Types

PostgreSQL supports several index types:

```sql
-- B-tree (default, most common)
CREATE INDEX idx_products_price ON products(price);
-- Good for: =, <, >, <=, >=, BETWEEN, ORDER BY

-- Hash (equality only)
CREATE INDEX idx_users_username ON users USING HASH (username);
-- Good for: = only (not ranges)
-- Rarely needed

-- GiST / GIN (full-text search, JSON, arrays)
CREATE INDEX idx_products_name_search 
ON products USING GIN (to_tsvector('english', product_name));
-- Good for: Full-text search

-- BRIN (block range index, for huge tables)
CREATE INDEX idx_orders_created_brin 
ON orders USING BRIN (created_at);
-- Good for: Large tables with naturally ordered data (timestamps)
```

### Index Maintenance

```sql
-- View all indexes on a table
SELECT 
    indexname, 
    indexdef 
FROM pg_indexes 
WHERE tablename = 'orders'
ORDER BY indexname;

-- Check index usage statistics
SELECT 
    schemaname,
    tablename,
    indexname,
    idx_scan AS times_used,
    idx_tup_read AS rows_read,
    idx_tup_fetch AS rows_fetched
FROM pg_stat_user_indexes
WHERE tablename = 'orders'
ORDER BY idx_scan DESC;

-- Find unused indexes (idx_scan = 0)
SELECT 
    schemaname,
    tablename,
    indexname,
    idx_scan
FROM pg_stat_user_indexes
WHERE idx_scan = 0
  AND indexrelname NOT LIKE 'pg_toast%';

-- Drop unused index
DROP INDEX IF EXISTS idx_old_unused_index;

-- Rebuild index (if corrupted or bloated)
REINDEX INDEX idx_orders_created_at;
REINDEX TABLE orders;  -- Rebuild all indexes on table
```

⚠️ **Index Overhead:** 
- Every index speeds up reads BUT slows down writes
- INSERT/UPDATE/DELETE must update all indexes
- Don't create indexes you don't need!
- Monitor index usage and drop unused ones

---

## Step 22: Query Optimization with EXPLAIN

### 🎯 Real-Life Scenario
A query takes 30 seconds. You need to understand WHY. Is it missing indexes? Doing full table scans? Inefficient joins? EXPLAIN shows you exactly what PostgreSQL is doing.

### Understanding EXPLAIN

```sql
-- See query execution plan (estimated)
EXPLAIN 
SELECT * FROM orders WHERE user_id = 123;

-- Output:
-- Seq Scan on orders  (cost=0.00..15000.00 rows=500 width=100)
--   Filter: (user_id = 123)
```

**Key Metrics:**
- **Seq Scan**: Sequential scan (reads entire table) - ⚠️ SLOW for large tables
- **Index Scan**: Uses index - ✅ FAST
- **cost**: Estimated processing cost (0.00..15000.00 = startup..total)
- **rows**: Estimated rows returned
- **width**: Average row size in bytes

### EXPLAIN ANALYZE - Actual Performance

```sql
-- Get actual execution time (runs the query!)
EXPLAIN ANALYZE
SELECT * FROM orders WHERE user_id = 123;

-- Output includes actual runtime:
-- Index Scan using idx_orders_user_id on orders
--   (cost=0.29..8.31 rows=5 width=100)
--   (actual time=0.015..0.018 rows=5 loops=1)
-- Planning Time: 0.123 ms
-- Execution Time: 0.045 ms
```

**EXPLAIN vs EXPLAIN ANALYZE:**

| Command | Behavior | Use Case |
|---------|----------|----------|
| `EXPLAIN` | Shows plan, doesn't run query | Safe, quick estimates |
| `EXPLAIN ANALYZE` | Runs query, shows actual times | Accurate, but runs query (can be slow!) |

### Reading Query Plans

#### Example 1: Missing Index

```sql
-- Bad: Sequential Scan
EXPLAIN ANALYZE
SELECT * FROM products WHERE category = 'Electronics';

-- Output:
-- Seq Scan on products  (cost=0.00..1500.00 rows=1000 width=100)
--                       (actual time=0.012..15.234 rows=1000 loops=1)
--   Filter: (category = 'Electronics')
--   Rows Removed by Filter: 9000
-- Execution Time: 15.456 ms

-- ⚠️ Scanned all 10,000 products, filtered to 1,000
```

**Solution: Add Index**

```sql
CREATE INDEX idx_products_category ON products(category);

-- Good: Index Scan
EXPLAIN ANALYZE
SELECT * FROM products WHERE category = 'Electronics';

-- Output:
-- Index Scan using idx_products_category on products
--   (cost=0.29..150.00 rows=1000 width=100)
--   (actual time=0.015..0.234 rows=1000 loops=1)
--   Index Cond: (category = 'Electronics')
-- Execution Time: 0.456 ms

-- ✅ 30x faster! (15ms → 0.45ms)
```

#### Example 2: Inefficient JOIN

```sql
-- Slow: Nested Loop without indexes
EXPLAIN ANALYZE
SELECT o.order_id, u.username
FROM orders o
JOIN users u ON o.user_id = u.user_id;

-- Output:
-- Nested Loop  (cost=0.00..500000.00 rows=10000 width=50)
--              (actual time=0.123..5234.567 rows=10000 loops=1)
--   -> Seq Scan on orders o  (cost=0.00..150.00 rows=10000 width=...)
--   -> Seq Scan on users u  (cost=0.00..50.00 rows=1 width=...)
--        Filter: (user_id = o.user_id)

-- ⚠️ For each of 10,000 orders, scans all users!
```

**Solution: Add Index on Foreign Key**

```sql
CREATE INDEX idx_orders_user_id ON orders(user_id);

-- Fast: Hash Join with index
EXPLAIN ANALYZE
SELECT o.order_id, u.username
FROM orders o
JOIN users u ON o.user_id = u.user_id;

-- Output:
-- Hash Join  (cost=150.00..350.00 rows=10000 width=50)
--            (actual time=1.234..12.345 rows=10000 loops=1)
--   Hash Cond: (o.user_id = u.user_id)
--   -> Seq Scan on orders o  (cost=0.00..150.00 rows=10000 width=...)
--   -> Hash  (cost=100.00..100.00 rows=5000 width=...)
--        -> Seq Scan on users u  (cost=0.00..100.00 rows=5000 width=...)

-- ✅ 400x faster! (5234ms → 12ms)
```

### Common Performance Problems

| Problem | Symptom in EXPLAIN | Solution |
|---------|-------------------|----------|
| **Missing Index** | Seq Scan on large table | CREATE INDEX |
| **Wrong Index** | Index Scan but still slow | Create better composite index |
| **SELECT \*** | High width value, slow | SELECT only needed columns |
| **Large JOIN** | Nested Loop with high cost | Add indexes on join columns |
| **OR conditions** | Multiple Seq Scans | Use UNION or IN clause |
| **Function in WHERE** | Can't use index | Create expression index |
| **LIKE '%term%'** | Seq Scan | Use full-text search (GIN index) |

### Query Optimization Examples

#### Problem: Slow Search

```sql
-- ❌ Slow: Can't use index (function on column)
SELECT * FROM products 
WHERE LOWER(product_name) LIKE '%iphone%';

-- Seq Scan (slow)
```

**Solutions:**

```sql
-- ✅ Option 1: Expression index
CREATE INDEX idx_products_lower_name ON products(LOWER(product_name));

-- ✅ Option 2: Full-text search (better for text searching)
CREATE INDEX idx_products_name_search 
ON products USING GIN (to_tsvector('english', product_name));

SELECT * FROM products 
WHERE to_tsvector('english', product_name) @@ to_tsquery('iphone');
```

#### Problem: OR conditions

```sql
-- ❌ Slow: Can't efficiently use indexes
SELECT * FROM orders 
WHERE user_id = 123 OR user_id = 456 OR user_id = 789;

-- Bitmap Heap Scan (still slower than IN)
```

**Solution:**

```sql
-- ✅ Fast: Can use index efficiently
SELECT * FROM orders 
WHERE user_id IN (123, 456, 789);

-- Index Scan or Bitmap Index Scan (fast)
```

### EXPLAIN Output Formats

```sql
-- Text format (default)
EXPLAIN SELECT * FROM orders;

-- JSON format (for tools/scripts)
EXPLAIN (FORMAT JSON) SELECT * FROM orders;

-- YAML format
EXPLAIN (FORMAT YAML) SELECT * FROM orders;

-- Detailed output
EXPLAIN (ANALYZE, BUFFERS, VERBOSE) 
SELECT * FROM orders WHERE user_id = 123;
```

---

# PHASE 9: SECURITY & GOVERNANCE

## Data Security Best Practices

### 🎯 Real-Life Scenario
You need to give the analytics team access to sales data without exposing customer emails or passwords. The finance team needs revenue data but not personal information. You use views and roles to control access.

### Identifying Sensitive Data

**Sensitive fields in MarketTrack:**
- User passwords (hashed)
- Email addresses
- Phone numbers
- Payment information
- Personal addresses

### Security with Views

```sql
-- Create public view without sensitive data
CREATE VIEW users_public AS
SELECT 
    user_id,
    username,
    -- email is hidden
    -- password_hash is hidden
    created_at
FROM users;

-- Analytics team can query this view
SELECT * FROM users_public;
-- They never see email or password_hash
```

### Row-Level Security (RLS)

```sql
-- Enable RLS on orders table
ALTER TABLE orders ENABLE ROW LEVEL SECURITY;

-- Policy: Users can only see their own orders
CREATE POLICY user_orders_policy ON orders
FOR SELECT
USING (user_id = current_user_id());

-- Policy: Analysts can see all delivered orders
CREATE POLICY analyst_orders_policy ON orders
FOR SELECT
TO analyst_role
USING (order_status = 'delivered');
```

### Data Retention Policies

```sql
-- Delete old cancelled orders (after 1 year)
DELETE FROM orders 
WHERE order_status = 'cancelled' 
  AND created_at < CURRENT_DATE - INTERVAL '1 year';

-- Archive old orders instead of deleting
INSERT INTO orders_archive 
SELECT * FROM orders 
WHERE created_at < CURRENT_DATE - INTERVAL '2 years';

DELETE FROM orders 
WHERE created_at < CURRENT_DATE - INTERVAL '2 years';
```

---

# SUMMARY: MarketTrack Implementation Checklist

## Required Deliverables

✅ **ER Diagram** showing all entities and relationships  
✅ **SQL DDL files** (CREATE TABLE statements with constraints)  
✅ **SQL files** with complex queries demonstrating:
   - Multiple JOINs (INNER, LEFT, anti-joins)
   - Subqueries and correlated subqueries
   - Multi-step CTEs (at least one chain)
   - Window functions (RANK, running totals, rolling averages)
✅ **SQL files** for views (regular and materialized)  
✅ **SQL files** for index creation  
✅ **EXPLAIN ANALYZE** outputs for key queries  
✅ **Written documentation** explaining design decisions  

---

## Key Performance Tips

💡 **Data Types:**
- Use NUMERIC for money, never FLOAT
- Use SMALLINT for small ranges (ratings 1-5) to save space
- Use VARCHAR with limits to enforce data quality
- Use TIMESTAMP for precise time tracking

💡 **Indexes:**
- Always add indexes on foreign keys
- Use composite indexes for common WHERE clause combinations
- Create partial indexes for frequently filtered subsets
- Drop unused indexes (check pg_stat_user_indexes)

💡 **Queries:**
- Use materialized views for expensive analytics queries
- Prefer NOT EXISTS over LEFT JOIN...IS NULL for anti-joins
- Use CTEs to break complex queries into readable steps
- Always run EXPLAIN ANALYZE before and after optimization

💡 **Joins:**
- Start with the main table, join related tables
- Add indexes on all join columns
- Use LEFT JOIN when you need all rows from left table
- Use window functions instead of correlated subqueries

---

## Module-Specific Features

### Module 1: Revenue Focus
- Window functions: RANK, SUM for revenue ranking
- Anti-joins for products never sold
- Rolling time windows for trend analysis
- Materialized views for daily/monthly aggregates

### Module 2: Inventory Focus
- Transactions for inventory updates
- CHECK constraints for stock >= 0
- Triggers for audit trails (optional)
- UPDATE with JOINs for stock management

### Module 3: Review Focus
- Subqueries with EXISTS for verified purchases
- Window functions for rolling averages (rating volatility)
- LEFT JOIN for products without reviews
- Composite indexes on (product_id, created_at)

### Module 4: Analytics Focus
- Materialized views for all dashboards
- Multi-step CTEs for complex metrics
- GROUPING SETS, ROLLUP, CUBE for hierarchical reporting
- Partitioned window functions for rankings

---

## Good Luck!

You now have all the knowledge needed to build a production-grade MarketTrack database from scratch! Remember:

1. **Start with ER diagram** - plan before coding
2. **Create schema with constraints** - enforce data integrity
3. **Insert sample data** - test your design
4. **Write operational queries** - master JOINs and filtering
5. **Build analytical queries** - CTEs, window functions
6. **Add views and materialized views** - simplify and optimize
7. **Create indexes** - speed up queries
8. **Use EXPLAIN** - verify performance
9. **Document everything** - explain your decisions

**The database is the product. Make it excellent!** 🚀
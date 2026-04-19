# **COMPLETE MONGODB FOR AI/ML ENGINEERS **

**Author's Note:** MongoDB queries look complex with all the brackets and JSON syntax, but they follow consistent patterns. This guide breaks down each concept with clear examples, visual patterns, and memory tricks.

---

# **TABLE OF CONTENTS**

1. [Why MongoDB? The NoSQL Revolution](#section-1-why-mongodb-the-nosql-revolution)
2. [MongoDB Fundamentals - The Building Blocks](#section-2-mongodb-fundamentals)
3. [CRUD Operations - The Basics](#section-3-crud-operations)
4. [Query Operators - The Power Tools](#section-4-query-operators)
5. [Understanding the Bracket Pattern](#section-5-understanding-the-bracket-pattern)
6. [Aggregation Pipeline - The Game Changer](#section-6-aggregation-pipeline)
7. [Indexing - Speed Optimization](#section-7-indexing)
8. [Data Modeling - The MongoDB Way](#section-8-data-modeling)
9. [Advanced Aggregation Patterns](#section-9-advanced-aggregation-patterns)
10. [Real-World ML/AI Scenarios](#section-10-real-world-mlai-scenarios)
11. [MongoDB vs SQL - Side by Side](#section-11-mongodb-vs-sql-comparison)
12. [Memory Tricks & Cheat Sheets](#section-12-memory-tricks--cheat-sheets)

---

# **SECTION 1: WHY MONGODB? THE NOSQL REVOLUTION**

## **The Problem with Traditional SQL (PostgreSQL)**

**Scenario:** You're building a user profile system.

**SQL Approach (Rigid):**
```sql
CREATE TABLE users (
    user_id INT PRIMARY KEY,
    name VARCHAR(100),
    email VARCHAR(100),
    phone VARCHAR(20)  -- What if user has 2 phones? Change schema!
);
```

**Problems:**
1. **Schema Changes Are Painful:** Adding a new field requires ALTER TABLE
2. **Can't Handle Variable Structure:** Some users have 1 address, some have 5
3. **JSON Data Awkward:** Storing flexible metadata requires JSONB hacks
4. **Horizontal Scaling Hard:** Sharding PostgreSQL is complex

---

## **The MongoDB Solution (Flexible)**

**MongoDB Approach:**
```javascript
// User 1: Basic info
{
    _id: ObjectId("507f1f77bcf86cd799439011"),
    name: "Alice",
    email: "alice@email.com"
}

// User 2: Has multiple phones (no schema change needed!)
{
    _id: ObjectId("507f1f77bcf86cd799439012"),
    name: "Bob",
    email: "bob@email.com",
    phones: ["555-0100", "555-0200"],
    addresses: [
        { type: "home", city: "NYC" },
        { type: "work", city: "LA" }
    ]
}

// User 3: Has custom fields (still works!)
{
    _id: ObjectId("507f1f77bcf86cd799439013"),
    name: "Carol",
    email: "carol@email.com",
    social_media: {
        twitter: "@carol",
        linkedin: "carol-smith"
    },
    preferences: {
        theme: "dark",
        language: "en"
    }
}
```

**Benefits:**
1. **No Schema Changes:** Add fields whenever needed
2. **Nested Data Natural:** Arrays and objects built-in
3. **Scales Horizontally:** Sharding is native
4. **Fast Development:** No migrations for every change

---

## **When to Use MongoDB vs PostgreSQL**

### **Use MongoDB When:**
- ✅ Schema evolves frequently (startups, experiments)
- ✅ Hierarchical/nested data (user profiles, product catalogs)
- ✅ High write throughput (logs, events, IoT data)
- ✅ Need horizontal scaling (millions of users)
- ✅ Flexible document structure per record
- ✅ Rapid prototyping

### **Use PostgreSQL When:**
- ✅ Complex joins across many tables
- ✅ ACID transactions critical (banking, payments)
- ✅ Fixed schema with referential integrity
- ✅ Complex analytical queries (SQL is more powerful)
- ✅ Data warehouse / business intelligence
- ✅ Strong consistency requirements

### **For ML/AI Projects:**
- **Training data extraction:** PostgreSQL (complex aggregations, joins)
- **Feature store:** MongoDB (flexible schema, fast reads)
- **Event logging:** MongoDB (high write volume, flexible structure)
- **Model metadata:** MongoDB (varying experiment configs)
- **Predictions storage:** Either (depends on query patterns)

---

# **SECTION 2: MONGODB FUNDAMENTALS**

## **Key Concepts - The Mental Model**

### **1. Database → Collection → Document**

**SQL Analogy:**
```
Database (PostgreSQL)
  └── Table (users)
        └── Row (user record)
```

**MongoDB Hierarchy:**
```
Database (mydb)
  └── Collection (users)
        └── Document (user record)
```

**Visual:**
```
mydb (database)
├── users (collection)
│   ├── { _id: 1, name: "Alice", age: 30 }  (document)
│   ├── { _id: 2, name: "Bob", age: 25 }    (document)
│   └── { _id: 3, name: "Carol", age: 28 }  (document)
│
├── orders (collection)
│   ├── { _id: 101, user_id: 1, total: 100 }
│   └── { _id: 102, user_id: 2, total: 200 }
│
└── products (collection)
    ├── { _id: 501, name: "Laptop", price: 1200 }
    └── { _id: 502, name: "Mouse", price: 25 }
```

---

## **2. Documents Are JSON (Actually BSON)**

**What You Write (JSON):**
```javascript
{
    name: "Alice",
    age: 30,
    email: "alice@email.com"
}
```

**What MongoDB Stores (BSON - Binary JSON):**
- More efficient storage
- Supports extra types (Date, ObjectId, Binary)
- Faster to parse

**Key Point:** You think JSON, MongoDB stores BSON. Same syntax for you!

---

## **3. The _id Field (Automatic Primary Key)**

**Every document gets automatic _id:**
```javascript
{
    _id: ObjectId("507f1f77bcf86cd799439011"),  // Auto-generated
    name: "Alice",
    age: 30
}
```

**ObjectId Structure:**
```
507f1f77bcf86cd799439011
├─┬─┘├─┬─┘├┬┘├──┬──┘
│ │  │ │  ││  │
│ │  │ │  ││  └─ Counter (random)
│ │  │ │  │└─ Process ID
│ │  │ │  └─ Machine ID
│ │  │ └─ Unix timestamp
│ └─ Unix timestamp
└─ Unix timestamp

4 bytes: Timestamp (seconds since epoch)
3 bytes: Machine identifier
2 bytes: Process ID
3 bytes: Counter
```

**Why ObjectId is Great:**
- **Globally unique** (no collisions across servers)
- **Sortable by creation time** (embedded timestamp)
- **No auto-increment bottleneck** (unlike SQL SERIAL)

**You can provide your own _id:**
```javascript
{
    _id: "user_alice_2026",  // Custom ID
    name: "Alice"
}
```

---

## **4. Data Types in MongoDB**

| Type | Example | SQL Equivalent |
|------|---------|----------------|
| **String** | `"Alice"` | VARCHAR, TEXT |
| **Number** | `30`, `3.14` | INT, DECIMAL |
| **Boolean** | `true`, `false` | BOOLEAN |
| **Date** | `ISODate("2026-02-09")` | TIMESTAMP |
| **Array** | `[1, 2, 3]` | ARRAY (PostgreSQL) or separate table (MySQL) |
| **Object** | `{ city: "NYC" }` | JSONB (PostgreSQL) |
| **Null** | `null` | NULL |
| **ObjectId** | `ObjectId("...")` | No equivalent (like UUID) |

**Key Difference:** Arrays and nested objects are first-class citizens in MongoDB!

---

## **5. Collections Don't Need Schema**

**SQL (Must define upfront):**
```sql
CREATE TABLE users (
    user_id INT,
    name VARCHAR(100),
    age INT
);
```

**MongoDB (Just insert):**
```javascript
// No CREATE needed, collection auto-created on first insert
db.users.insertOne({ name: "Alice", age: 30 })
```

**Different documents, same collection (totally fine!):**
```javascript
db.users.insertOne({ name: "Bob", age: 25, email: "bob@email.com" })
db.users.insertOne({ name: "Carol", hobbies: ["reading", "coding"] })
db.users.insertOne({ name: "Dave", age: 35, address: { city: "NYC" } })
```

---

# **SECTION 3: CRUD OPERATIONS**

## **The Pattern: db.collection.operation()**

**Mental Model:**
```javascript
db          // The database
  .users    // The collection (like a table)
  .find()   // The operation
```

---

## **CREATE (Insert Documents)**

### **Insert One Document**

**Pattern:**
```javascript
db.collection.insertOne({ field: value })
```

**Examples:**
```javascript
// Basic insert
db.users.insertOne({
    name: "Alice",
    age: 30,
    email: "alice@email.com"
})

// Returns:
{
    acknowledged: true,
    insertedId: ObjectId("507f1f77bcf86cd799439011")
}
```

**With Custom _id:**
```javascript
db.users.insertOne({
    _id: "user_001",
    name: "Bob",
    age: 25
})
```

**With Nested Data:**
```javascript
db.users.insertOne({
    name: "Carol",
    age: 28,
    address: {
        street: "123 Main St",
        city: "NYC",
        zip: "10001"
    },
    phones: ["555-0100", "555-0200"]
})
```

---

### **Insert Multiple Documents**

**Pattern:**
```javascript
db.collection.insertMany([
    { field: value },
    { field: value }
])
```

**Example:**
```javascript
db.users.insertMany([
    { name: "Alice", age: 30, city: "NYC" },
    { name: "Bob", age: 25, city: "LA" },
    { name: "Carol", age: 28, city: "Chicago" }
])

// Returns:
{
    acknowledged: true,
    insertedIds: {
        0: ObjectId("..."),
        1: ObjectId("..."),
        2: ObjectId("...")
    }
}
```

---

## **READ (Query Documents)**

### **Find All Documents**

**Pattern:**
```javascript
db.collection.find()
```

**Example:**
```javascript
db.users.find()

// Returns all documents (like SELECT * FROM users)
```

---

### **Find with Filter (WHERE clause)**

**Pattern:**
```javascript
db.collection.find({ field: value })
```

**Examples:**

**Equality:**
```javascript
// Find user named Alice
db.users.find({ name: "Alice" })

// SQL: SELECT * FROM users WHERE name = 'Alice'
```

**Multiple Conditions (AND):**
```javascript
// Users named Alice AND age 30
db.users.find({ 
    name: "Alice", 
    age: 30 
})

// SQL: SELECT * FROM users WHERE name = 'Alice' AND age = 30
```

**Comparison Operators:**
```javascript
// Users older than 25
db.users.find({ age: { $gt: 25 } })
// SQL: WHERE age > 25

// Users age 25 to 35
db.users.find({ age: { $gte: 25, $lte: 35 } })
// SQL: WHERE age BETWEEN 25 AND 35
```

---

### **Find One Document**

**Pattern:**
```javascript
db.collection.findOne({ field: value })
```

**Example:**
```javascript
db.users.findOne({ name: "Alice" })

// Returns single document (or null if not found)
// SQL: SELECT * FROM users WHERE name = 'Alice' LIMIT 1
```

---

### **Projection (Select Specific Fields)**

**Pattern:**
```javascript
db.collection.find(
    { filter },
    { field1: 1, field2: 1, _id: 0 }
)
```

**Understanding the Numbers:**
- `1` = Include this field
- `0` = Exclude this field

**Examples:**

**Include only name and age:**
```javascript
db.users.find(
    {},
    { name: 1, age: 1, _id: 0 }
)

// SQL: SELECT name, age FROM users

// Returns:
{ name: "Alice", age: 30 }
{ name: "Bob", age: 25 }
```

**Exclude _id (it's included by default):**
```javascript
db.users.find(
    {},
    { name: 1, _id: 0 }
)

// Only shows name
```

**Exclude specific fields:**
```javascript
db.users.find(
    {},
    { password: 0, ssn: 0 }
)

// Shows everything EXCEPT password and ssn
```

**⚠️ Rule:** Can't mix inclusion and exclusion (except for _id)
```javascript
// ❌ WRONG
db.users.find({}, { name: 1, age: 0 })  // Error!

// ✓ CORRECT
db.users.find({}, { name: 1, email: 1 })  // Include name, email
db.users.find({}, { password: 0 })  // Exclude password
```

---

### **Sorting**

**Pattern:**
```javascript
db.collection.find().sort({ field: 1 })  // 1 = ascending, -1 = descending
```

**Examples:**
```javascript
// Sort by age ascending
db.users.find().sort({ age: 1 })
// SQL: ORDER BY age ASC

// Sort by age descending
db.users.find().sort({ age: -1 })
// SQL: ORDER BY age DESC

// Multiple fields
db.users.find().sort({ city: 1, age: -1 })
// SQL: ORDER BY city ASC, age DESC
```

---

### **Limit & Skip (Pagination)**

**Pattern:**
```javascript
db.collection.find().limit(n).skip(n)
```

**Examples:**
```javascript
// First 10 users
db.users.find().limit(10)
// SQL: LIMIT 10

// Skip first 20, get next 10 (page 3)
db.users.find().skip(20).limit(10)
// SQL: LIMIT 10 OFFSET 20

// Combine with sorting
db.users.find()
    .sort({ age: -1 })
    .limit(5)
// Top 5 oldest users
```

---

### **Counting Documents**

**Pattern:**
```javascript
db.collection.countDocuments({ filter })
```

**Examples:**
```javascript
// Total users
db.users.countDocuments()
// SQL: SELECT COUNT(*) FROM users

// Users over 30
db.users.countDocuments({ age: { $gt: 30 } })
// SQL: SELECT COUNT(*) FROM users WHERE age > 30
```

---

## **UPDATE (Modify Documents)**

### **Update One Document**

**Pattern:**
```javascript
db.collection.updateOne(
    { filter },           // Which document to update
    { $set: { field: newValue } }  // What to change
)
```

**The $set Operator:**
```javascript
// Update Alice's age
db.users.updateOne(
    { name: "Alice" },        // Find Alice
    { $set: { age: 31 } }     // Set age to 31
)

// SQL: UPDATE users SET age = 31 WHERE name = 'Alice'
```

**⚠️ Common Mistake:**
```javascript
// ❌ WRONG - Replaces entire document!
db.users.updateOne(
    { name: "Alice" },
    { age: 31 }  // Missing $set!
)
// Result: { _id: ..., age: 31 }  (name is GONE!)

// ✓ CORRECT - Only updates specified field
db.users.updateOne(
    { name: "Alice" },
    { $set: { age: 31 } }
)
// Result: { _id: ..., name: "Alice", age: 31 }
```

**Update Multiple Fields:**
```javascript
db.users.updateOne(
    { name: "Alice" },
    { $set: { 
        age: 31, 
        city: "SF",
        updated_at: new Date()
    }}
)
```

---

### **Update Many Documents**

**Pattern:**
```javascript
db.collection.updateMany(
    { filter },
    { $set: { field: value } }
)
```

**Example:**
```javascript
// Set all users in NYC to active
db.users.updateMany(
    { city: "NYC" },
    { $set: { status: "active" } }
)

// SQL: UPDATE users SET status = 'active' WHERE city = 'NYC'
```

---

### **Update Operators**

**$set - Set field value:**
```javascript
{ $set: { age: 31 } }
```

**$unset - Remove field:**
```javascript
// Remove phone field
db.users.updateOne(
    { name: "Alice" },
    { $unset: { phone: "" } }  // Value doesn't matter
)
```

**$inc - Increment number:**
```javascript
// Increase age by 1
db.users.updateOne(
    { name: "Alice" },
    { $inc: { age: 1 } }
)

// Decrease by 5
db.users.updateOne(
    { name: "Alice" },
    { $inc: { login_count: -5 } }
)
```

**$mul - Multiply:**
```javascript
// Double the price
db.products.updateOne(
    { name: "Laptop" },
    { $mul: { price: 2 } }
)
```

**$rename - Rename field:**
```javascript
db.users.updateMany(
    {},
    { $rename: { "phone": "mobile" } }
)
```

**$currentDate - Set to current date:**
```javascript
db.users.updateOne(
    { name: "Alice" },
    { $currentDate: { last_login: true } }
)
```

---

### **Array Update Operators**

**$push - Add to array:**
```javascript
// Add tag to user
db.users.updateOne(
    { name: "Alice" },
    { $push: { tags: "premium" } }
)

// Before: { tags: ["user", "active"] }
// After:  { tags: ["user", "active", "premium"] }
```

**$pull - Remove from array:**
```javascript
// Remove tag
db.users.updateOne(
    { name: "Alice" },
    { $pull: { tags: "active" } }
)

// Before: { tags: ["user", "active", "premium"] }
// After:  { tags: ["user", "premium"] }
```

**$addToSet - Add if not exists:**
```javascript
// Add tag only if not already present
db.users.updateOne(
    { name: "Alice" },
    { $addToSet: { tags: "premium" } }
)
// Won't create duplicates
```

**$pop - Remove first/last element:**
```javascript
// Remove last element
db.users.updateOne(
    { name: "Alice" },
    { $pop: { tags: 1 } }  // 1 = last, -1 = first
)
```

---

### **Upsert (Insert if Not Exists)**

**Pattern:**
```javascript
db.collection.updateOne(
    { filter },
    { $set: { field: value } },
    { upsert: true }  // Insert if not found
)
```

**Example:**
```javascript
// Update Alice if exists, create if doesn't
db.users.updateOne(
    { name: "Alice" },
    { $set: { age: 30, email: "alice@email.com" } },
    { upsert: true }
)

// SQL equivalent:
// INSERT INTO users ... ON CONFLICT DO UPDATE
```

---

## **DELETE (Remove Documents)**

### **Delete One Document**

**Pattern:**
```javascript
db.collection.deleteOne({ filter })
```

**Example:**
```javascript
// Delete Alice
db.users.deleteOne({ name: "Alice" })

// SQL: DELETE FROM users WHERE name = 'Alice' LIMIT 1
```

---

### **Delete Many Documents**

**Pattern:**
```javascript
db.collection.deleteMany({ filter })
```

**Examples:**
```javascript
// Delete all users older than 60
db.users.deleteMany({ age: { $gt: 60 } })

// Delete all users in NYC
db.users.deleteMany({ city: "NYC" })

// SQL: DELETE FROM users WHERE city = 'NYC'
```

**⚠️ Delete Everything:**
```javascript
// Delete all documents in collection
db.users.deleteMany({})  // Empty filter = all documents

// SQL: DELETE FROM users
```

---

## **CRUD Summary - The Mental Map**

```javascript
// CREATE
db.users.insertOne({ name: "Alice" })
db.users.insertMany([{ name: "Bob" }, { name: "Carol" }])

// READ
db.users.find({ age: { $gt: 25 } })
db.users.findOne({ name: "Alice" })
db.users.find().sort({ age: -1 }).limit(10)

// UPDATE
db.users.updateOne({ name: "Alice" }, { $set: { age: 31 } })
db.users.updateMany({ city: "NYC" }, { $set: { status: "active" } })

// DELETE
db.users.deleteOne({ name: "Alice" })
db.users.deleteMany({ age: { $gt: 60 } })
```

---

# **SECTION 4: QUERY OPERATORS**

## **Understanding the Dollar Sign ($)**

**Mental Model:** `$` means "special MongoDB operator"

**Pattern:**
```javascript
{ field: { $operator: value } }
       ↑        ↑
   field name  operator
```

---

## **Comparison Operators**

### **The Big 6:**

| Operator | Meaning | SQL Equivalent |
|----------|---------|----------------|
| `$eq` | Equal | `=` |
| `$ne` | Not equal | `!=` |
| `$gt` | Greater than | `>` |
| `$gte` | Greater than or equal | `>=` |
| `$lt` | Less than | `<` |
| `$lte` | Less than or equal | `<=` |

**Examples:**

```javascript
// Age equals 30 (explicit)
db.users.find({ age: { $eq: 30 } })
// Shorthand (same thing)
db.users.find({ age: 30 })

// Age NOT equal to 30
db.users.find({ age: { $ne: 30 } })

// Age greater than 25
db.users.find({ age: { $gt: 25 } })

// Age between 25 and 35 (inclusive)
db.users.find({ age: { $gte: 25, $lte: 35 } })
// SQL: WHERE age BETWEEN 25 AND 35
```

---

### **$in and $nin (Match Any / None)**

**Pattern:**
```javascript
{ field: { $in: [value1, value2, ...] } }
```

**Examples:**

```javascript
// Users in NYC, LA, or Chicago
db.users.find({ 
    city: { $in: ["NYC", "LA", "Chicago"] } 
})
// SQL: WHERE city IN ('NYC', 'LA', 'Chicago')

// Users NOT in these cities
db.users.find({ 
    city: { $nin: ["NYC", "LA"] } 
})
// SQL: WHERE city NOT IN ('NYC', 'LA')

// Ages 25, 30, or 35
db.users.find({ 
    age: { $in: [25, 30, 35] } 
})
```

---

## **Logical Operators**

### **$and (All conditions must match)**

**Pattern:**
```javascript
{ $and: [
    { condition1 },
    { condition2 }
]}
```

**Examples:**

```javascript
// Age > 25 AND city = NYC
db.users.find({ 
    $and: [
        { age: { $gt: 25 } },
        { city: "NYC" }
    ]
})

// Shorthand (same thing - implicit AND)
db.users.find({ 
    age: { $gt: 25 },
    city: "NYC"
})
// Use explicit $and when same field has multiple conditions
```

**When to use explicit $and:**
```javascript
// Same field, different operators
db.users.find({ 
    $and: [
        { age: { $gt: 25 } },
        { age: { $lt: 35 } }
    ]
})
```

---

### **$or (At least one condition matches)**

**Pattern:**
```javascript
{ $or: [
    { condition1 },
    { condition2 }
]}
```

**Examples:**

```javascript
// Age > 30 OR city = NYC
db.users.find({ 
    $or: [
        { age: { $gt: 30 } },
        { city: "NYC" }
    ]
})
// SQL: WHERE age > 30 OR city = 'NYC'

// Name is Alice OR Bob
db.users.find({ 
    $or: [
        { name: "Alice" },
        { name: "Bob" }
    ]
})
// Better: use $in
db.users.find({ name: { $in: ["Alice", "Bob"] } })
```

---

### **$not (Negate condition)**

**Pattern:**
```javascript
{ field: { $not: { $operator: value } } }
```

**Examples:**

```javascript
// Age NOT greater than 30 (i.e., <= 30)
db.users.find({ 
    age: { $not: { $gt: 30 } } 
})
// SQL: WHERE NOT (age > 30)

// Email doesn't match pattern
db.users.find({ 
    email: { $not: { $regex: /@gmail\.com$/ } } 
})
```

---

### **$nor (None of the conditions match)**

**Pattern:**
```javascript
{ $nor: [
    { condition1 },
    { condition2 }
]}
```

**Example:**

```javascript
// NOT in NYC AND NOT age > 30
db.users.find({ 
    $nor: [
        { city: "NYC" },
        { age: { $gt: 30 } }
    ]
})
// SQL: WHERE city != 'NYC' AND age <= 30
```

---

## **Element Operators**

### **$exists (Field exists or not)**

**Pattern:**
```javascript
{ field: { $exists: true } }  // Has field
{ field: { $exists: false } } // Doesn't have field
```

**Examples:**

```javascript
// Users with phone field
db.users.find({ phone: { $exists: true } })

// Users without email field
db.users.find({ email: { $exists: false } })

// Combine with value check
db.users.find({ 
    phone: { $exists: true, $ne: null } 
})
// Has phone field AND it's not null
```

---

### **$type (Check data type)**

**Pattern:**
```javascript
{ field: { $type: "string" } }
```

**Common Types:**
- `"string"`, `"int"`, `"double"`, `"bool"`
- `"array"`, `"object"`, `"null"`
- `"date"`, `"objectId"`

**Examples:**

```javascript
// Find where age is a string (data quality issue)
db.users.find({ age: { $type: "string" } })

// Find documents where tags is an array
db.users.find({ tags: { $type: "array" } })
```

---

## **Array Operators**

### **$all (Array contains all elements)**

**Pattern:**
```javascript
{ field: { $all: [value1, value2] } }
```

**Example:**

```javascript
// Users with BOTH "ml" and "ai" tags
db.users.find({ 
    tags: { $all: ["ml", "ai"] } 
})

// tags: ["ml", "ai", "python"] ✓ matches
// tags: ["ml", "python"] ✗ doesn't match (missing "ai")
```

---

### **$elemMatch (Array element matches multiple conditions)**

**Pattern:**
```javascript
{ field: { $elemMatch: { condition1, condition2 } } }
```

**Example:**

```javascript
// Users with an order where quantity > 5 AND price < 100
db.users.find({ 
    orders: { 
        $elemMatch: { 
            quantity: { $gt: 5 },
            price: { $lt: 100 }
        }
    }
})

// Document structure:
{
    name: "Alice",
    orders: [
        { quantity: 10, price: 50 },  // This matches!
        { quantity: 2, price: 200 }
    ]
}
```

---

### **$size (Array length)**

**Pattern:**
```javascript
{ field: { $size: n } }
```

**Example:**

```javascript
// Users with exactly 3 tags
db.users.find({ tags: { $size: 3 } })

// ⚠️ Cannot use with ranges
// ❌ db.users.find({ tags: { $size: { $gt: 3 } } })  // Doesn't work!
```

---

## **String Operators**

### **$regex (Pattern matching)**

**Pattern:**
```javascript
{ field: { $regex: /pattern/ } }
{ field: { $regex: "pattern", $options: "i" } }  // Case-insensitive
```

**Examples:**

```javascript
// Email ends with @gmail.com
db.users.find({ 
    email: { $regex: /@gmail\.com$/ } 
})

// Name starts with "Al" (case-insensitive)
db.users.find({ 
    name: { $regex: /^Al/, $options: "i" } 
})
// Matches: "Alice", "albert", "ALEX"

// Contains "admin" anywhere
db.users.find({ 
    role: { $regex: /admin/i } 
})
```

**Common Regex Patterns:**
```javascript
/^abc/     // Starts with "abc"
/xyz$/     // Ends with "xyz"
/abc/i     // Contains "abc" (case-insensitive)
/^[A-Z]/   // Starts with capital letter
/\d{3}/    // Contains 3 digits
```

---

## **Query Operators - Quick Reference**

```javascript
// Comparison
{ age: { $gt: 25, $lt: 35 } }
{ city: { $in: ["NYC", "LA"] } }
{ status: { $ne: "inactive" } }

// Logical
{ $or: [{ age: { $gt: 30 } }, { city: "NYC" }] }
{ $and: [{ age: { $gt: 25 } }, { city: "NYC" }] }

// Element
{ phone: { $exists: true } }
{ age: { $type: "int" } }

// Array
{ tags: { $all: ["ml", "ai"] } }
{ tags: { $size: 3 } }

// String
{ email: { $regex: /@gmail\.com$/ } }
```

---

# **SECTION 5: UNDERSTANDING THE BRACKET PATTERN**

## **Why So Many Brackets? The Visual Guide**

**The Problem:** MongoDB queries look like this:
```javascript
db.orders.aggregate([
    { $match: { status: "completed" } },
    { $group: { _id: "$customer_id", total: { $sum: "$amount" } } }
])
```

**Looks confusing!** Let's break it down step by step.

---

## **Pattern 1: Object { }**

**Rule:** `{ }` defines a **key-value pair** (like a dictionary/map)

```javascript
{ key: value }
```

**Examples:**

```javascript
// Simple object
{ name: "Alice" }

// Nested object
{ 
    name: "Alice",
    address: { city: "NYC", zip: "10001" }
}

// With operator
{ age: { $gt: 25 } }
     ↑        ↑
    key    value (another object!)
```

---

## **Pattern 2: Array [ ]**

**Rule:** `[ ]` defines a **list of items**

```javascript
[ item1, item2, item3 ]
```

**Examples:**

```javascript
// Array of strings
["Alice", "Bob", "Carol"]

// Array of numbers
[1, 2, 3, 4, 5]

// Array of objects
[
    { name: "Alice", age: 30 },
    { name: "Bob", age: 25 }
]
```

---

## **Pattern 3: Combining { } and [ ]**

### **Array of Objects (Most Common in MongoDB)**

**Pattern:**
```javascript
[
    { key: value },
    { key: value }
]
```

**Example:**
```javascript
// Multiple filter conditions in $or
{ 
    $or: [
        { age: { $gt: 30 } },
        { city: "NYC" }
    ]
}

// Breaking it down:
{                          // Object start
    $or: [                 // Key: $or, Value: array
        { age: { $gt: 30 } },   // Array item 1 (object)
        { city: "NYC" }         // Array item 2 (object)
    ]
}
```

---

## **Visual Guide: Reading MongoDB Queries**

### **Example 1: Simple Query**

```javascript
db.users.find({ age: { $gt: 25 } })
```

**Breaking it down:**
```javascript
db.users.find(
    {                    // Filter object START
        age: {           // Field: age, Value: object
            $gt: 25      // Operator: $gt, Value: 25
        }
    }                    // Filter object END
)
```

**Visual tree:**
```
find()
 └─ { }                (filter object)
     └─ age: { }       (field with operator object)
         └─ $gt: 25    (operator and value)
```

---

### **Example 2: Logical Operator**

```javascript
db.users.find({ 
    $or: [
        { age: { $gt: 30 } },
        { city: "NYC" }
    ]
})
```

**Breaking it down:**
```javascript
{                              // Outer object START
    $or: [                     // Key: $or, Value: array
        { age: { $gt: 30 } },  // Array element 1
        { city: "NYC" }        // Array element 2
    ]
}                              // Outer object END
```

**Visual tree:**
```
{ }                    (filter object)
 └─ $or: [ ]           (operator with array value)
     ├─ { }            (condition 1)
     │   └─ age: { }
     │       └─ $gt: 30
     └─ { }            (condition 2)
         └─ city: "NYC"
```

---

### **Example 3: Aggregation Pipeline**

```javascript
db.orders.aggregate([
    { $match: { status: "completed" } },
    { $group: { _id: "$customer_id", total: { $sum: "$amount" } } }
])
```

**Breaking it down:**
```javascript
aggregate(
    [                                              // Array START (pipeline stages)
        { $match: { status: "completed" } },       // Stage 1 object
        { $group: {                                // Stage 2 object
            _id: "$customer_id",
            total: { $sum: "$amount" }
        }}
    ]                                              // Array END
)
```

**Visual tree:**
```
aggregate()
 └─ [ ]                          (pipeline array)
     ├─ { }                       (stage 1)
     │   └─ $match: { }
     │       └─ status: "completed"
     └─ { }                       (stage 2)
         └─ $group: { }
             ├─ _id: "$customer_id"
             └─ total: { }
                 └─ $sum: "$amount"
```

---

## **Memory Trick: The Bracket Rules**

### **Rule 1: { } for Structure**
```javascript
{ key: value }           // Single property
{ key1: val1, key2: val2 }  // Multiple properties
```

### **Rule 2: [ ] for Lists**
```javascript
[ value1, value2 ]       // List of values
[ { }, { } ]             // List of objects (common in MongoDB)
```

### **Rule 3: Nesting Pattern**
```javascript
{                        // Object containing
    key: [               // Array containing
        { key: value },  // Objects
        { key: value }
    ]
}
```

### **Rule 4: Operators Always Start with $**
```javascript
{ $match: ... }          // Aggregation stage
{ $gt: 25 }              // Query operator
{ $sum: "$amount" }      // Accumulator operator
```

---

## **Practice: Read These Queries Aloud**

### **Query 1:**
```javascript
db.users.find({ age: 30 })
```
**Read as:** "Find users where age equals 30"

---

### **Query 2:**
```javascript
db.users.find({ age: { $gte: 25, $lte: 35 } })
```
**Read as:** "Find users where age is greater than or equal to 25 AND less than or equal to 35"

---

### **Query 3:**
```javascript
db.users.find({ 
    $or: [
        { age: { $gt: 30 } },
        { city: "NYC" }
    ]
})
```
**Read as:** "Find users where (age is greater than 30) OR (city equals NYC)"

---

### **Query 4:**
```javascript
db.users.updateOne(
    { name: "Alice" },
    { $set: { age: 31 } }
)
```
**Read as:** "Update one user where name equals Alice, set age to 31"

---

## **Common Mistakes & Fixes**

### **Mistake 1: Forgetting Operators**
```javascript
// ❌ WRONG
db.users.find({ age > 25 })

// ✓ CORRECT
db.users.find({ age: { $gt: 25 } })
```

### **Mistake 2: Wrong Brackets**
```javascript
// ❌ WRONG
db.users.find([ { age: 30 } ])  // Array instead of object

// ✓ CORRECT
db.users.find({ age: 30 })
```

### **Mistake 3: Missing $set in Update**
```javascript
// ❌ WRONG (replaces entire document!)
db.users.updateOne({ name: "Alice" }, { age: 31 })

// ✓ CORRECT
db.users.updateOne({ name: "Alice" }, { $set: { age: 31 } })
```

### **Mistake 4: Commas**
```javascript
// ❌ WRONG (missing comma)
db.users.find({ 
    age: { $gt: 25 }
    city: "NYC"  // Syntax error!
})

// ✓ CORRECT
db.users.find({ 
    age: { $gt: 25 },
    city: "NYC"
})
```

---

# **SECTION 6: AGGREGATION PIPELINE - THE GAME CHANGER**

## **What is Aggregation Pipeline?**

**SQL Analogy:**
```sql
SELECT customer_id, SUM(total) as total_spent
FROM orders
WHERE status = 'completed'
GROUP BY customer_id
HAVING SUM(total) > 1000
ORDER BY total_spent DESC
```

**MongoDB Aggregation:**
```javascript
db.orders.aggregate([
    { $match: { status: "completed" } },
    { $group: { 
        _id: "$customer_id", 
        total_spent: { $sum: "$total" } 
    }},
    { $match: { total_spent: { $gt: 1000 } } },
    { $sort: { total_spent: -1 } }
])
```

---

## **Pipeline Concept: Data Flows Through Stages**

**Mental Model:**
```
Collection → Stage 1 → Stage 2 → Stage 3 → Result
  (input)    (filter)  (group)   (sort)   (output)
```

**Visual:**
```
orders collection (1000 documents)
    ↓
[ { $match: { status: "completed" } } ]  → 800 documents
    ↓
[ { $group: { _id: "$customer_id", total: { $sum: "$total" } } } ]  → 150 documents
    ↓
[ { $sort: { total: -1 } } ]  → 150 documents (sorted)
    ↓
Result
```

**Key Point:** Each stage transforms data and passes it to the next stage!

---

## **Basic Pattern**

```javascript
db.collection.aggregate([
    { stage1 },
    { stage2 },
    { stage3 }
])
       ↑
    Array of stage objects
```

---

## **Stage 1: $match (Filter Documents)**

**Purpose:** Like SQL `WHERE` - filter which documents to process

**Pattern:**
```javascript
{ $match: { field: value } }
```

**Examples:**

```javascript
// Filter completed orders
db.orders.aggregate([
    { $match: { status: "completed" } }
])

// SQL: WHERE status = 'completed'
```

```javascript
// Multiple conditions
db.orders.aggregate([
    { $match: { 
        status: "completed",
        total: { $gt: 100 }
    }}
])

// SQL: WHERE status = 'completed' AND total > 100
```

```javascript
// Date range
db.orders.aggregate([
    { $match: { 
        order_date: { 
            $gte: ISODate("2026-01-01"),
            $lt: ISODate("2026-02-01")
        }
    }}
])

// SQL: WHERE order_date >= '2026-01-01' AND order_date < '2026-02-01'
```

**💡 Pro Tip:** Always $match early to reduce documents in later stages!

---

## **Stage 2: $project (Select/Transform Fields)**

**Purpose:** Like SQL `SELECT` - choose which fields to keep/add

**Pattern:**
```javascript
{ $project: { 
    field1: 1,        // Include
    field2: 0,        // Exclude
    newField: "$existingField"  // Rename
}}
```

**Examples:**

**Include specific fields:**
```javascript
db.orders.aggregate([
    { $project: { 
        customer_id: 1,
        total: 1,
        _id: 0  // Exclude _id
    }}
])

// SQL: SELECT customer_id, total FROM orders
```

**Create computed fields:**
```javascript
db.orders.aggregate([
    { $project: { 
        customer_id: 1,
        total: 1,
        total_with_tax: { $multiply: ["$total", 1.1] }
    }}
])

// total_with_tax = total * 1.1
```

**Extract date parts:**
```javascript
db.orders.aggregate([
    { $project: { 
        customer_id: 1,
        year: { $year: "$order_date" },
        month: { $month: "$order_date" },
        day: { $dayOfMonth: "$order_date" }
    }}
])
```

**⚠️ Dollar Sign $ in Field References:**
```javascript
// ❌ WRONG
{ $project: { customer_id: 1, total_doubled: total * 2 } }

// ✓ CORRECT (use "$" to reference field)
{ $project: { customer_id: 1, total_doubled: { $multiply: ["$total", 2] } } }
```

---

## **Stage 3: $group (Aggregate Data)**

**Purpose:** Like SQL `GROUP BY` - aggregate documents together

**Pattern:**
```javascript
{ $group: { 
    _id: "$groupByField",
    newField: { $accumulator: "$sourceField" }
}}
```

**Key Concept:** `_id` defines the grouping key

---

### **Basic Grouping**

**Example: Total sales per customer**

```javascript
db.orders.aggregate([
    { $group: { 
        _id: "$customer_id",
        total_spent: { $sum: "$total" }
    }}
])

// SQL: 
// SELECT customer_id, SUM(total) as total_spent
// FROM orders
// GROUP BY customer_id
```

**Result:**
```javascript
{ _id: "cust_001", total_spent: 1500 }
{ _id: "cust_002", total_spent: 3200 }
{ _id: "cust_003", total_spent: 800 }
```

---

### **Aggregation Operators (Accumulators)**

**$sum - Sum values:**
```javascript
{ $group: { 
    _id: "$customer_id",
    total_spent: { $sum: "$amount" },
    order_count: { $sum: 1 }  // Count documents (like COUNT(*))
}}
```

**$avg - Average:**
```javascript
{ $group: { 
    _id: "$category",
    avg_price: { $avg: "$price" }
}}
```

**$max / $min - Maximum/Minimum:**
```javascript
{ $group: { 
    _id: "$customer_id",
    max_order: { $max: "$total" },
    min_order: { $min: "$total" }
}}
```

**$first / $last - First/Last value:**
```javascript
{ $group: { 
    _id: "$customer_id",
    first_order_date: { $first: "$order_date" },
    last_order_date: { $last: "$order_date" }
}}
```

**$push - Create array of values:**
```javascript
{ $group: { 
    _id: "$customer_id",
    all_order_ids: { $push: "$order_id" }
}}

// Result: { _id: "cust_001", all_order_ids: [101, 102, 103] }
```

**$addToSet - Unique array:**
```javascript
{ $group: { 
    _id: "$customer_id",
    unique_products: { $addToSet: "$product_id" }
}}

// Removes duplicates
```

---

### **Group by Multiple Fields**

**Pattern:**
```javascript
{ $group: { 
    _id: { 
        field1: "$field1",
        field2: "$field2"
    },
    total: { $sum: "$amount" }
}}
```

**Example: Sales by city and category**

```javascript
db.sales.aggregate([
    { $group: { 
        _id: { 
            city: "$city",
            category: "$category"
        },
        total_sales: { $sum: "$amount" }
    }}
])

// SQL:
// GROUP BY city, category
```

**Result:**
```javascript
{ _id: { city: "NYC", category: "Electronics" }, total_sales: 5000 }
{ _id: { city: "NYC", category: "Books" }, total_sales: 2000 }
{ _id: { city: "LA", category: "Electronics" }, total_sales: 3000 }
```

---

### **Group All Documents (No Grouping Key)**

**Pattern:**
```javascript
{ $group: { 
    _id: null,  // No grouping, aggregate everything
    total: { $sum: "$amount" }
}}
```

**Example: Overall statistics**

```javascript
db.orders.aggregate([
    { $group: { 
        _id: null,
        total_orders: { $sum: 1 },
        total_revenue: { $sum: "$total" },
        avg_order_value: { $avg: "$total" }
    }}
])

// SQL: SELECT COUNT(*), SUM(total), AVG(total) FROM orders
```

**Result:**
```javascript
{ 
    _id: null,
    total_orders: 1000,
    total_revenue: 150000,
    avg_order_value: 150
}
```

---

## **Stage 4: $sort (Order Results)**

**Pattern:**
```javascript
{ $sort: { field: 1 } }  // 1 = ascending, -1 = descending
```

**Examples:**

```javascript
// Sort by age ascending
db.users.aggregate([
    { $sort: { age: 1 } }
])

// Sort by multiple fields
db.users.aggregate([
    { $sort: { city: 1, age: -1 } }
])
// First by city A-Z, then by age high-low within each city
```

---

## **Stage 5: $limit & $skip (Pagination)**

**Pattern:**
```javascript
{ $limit: n }
{ $skip: n }
```

**Examples:**

```javascript
// Top 10 customers by spending
db.orders.aggregate([
    { $group: { 
        _id: "$customer_id",
        total_spent: { $sum: "$total" }
    }},
    { $sort: { total_spent: -1 } },
    { $limit: 10 }
])
```

```javascript
// Pagination (page 3, 20 per page)
db.orders.aggregate([
    { $sort: { order_date: -1 } },
    { $skip: 40 },   // Skip first 2 pages (20 * 2)
    { $limit: 20 }   // Get page 3
])
```

---

## **Stage 6: $addFields (Add New Fields)**

**Pattern:**
```javascript
{ $addFields: { 
    newField: expression
}}
```

**Examples:**

```javascript
// Add full_name field
db.users.aggregate([
    { $addFields: { 
        full_name: { $concat: ["$first_name", " ", "$last_name"] }
    }}
])
```

```javascript
// Add calculated fields
db.orders.aggregate([
    { $addFields: { 
        total_with_tax: { $multiply: ["$total", 1.1] },
        is_large_order: { $gt: ["$total", 1000] }
    }}
])
```

**Difference from $project:**
- **$project:** Replace all fields (only keeps what you specify)
- **$addFields:** Add to existing fields (keeps everything + adds new)

---

## **Stage 7: $count (Count Documents)**

**Pattern:**
```javascript
{ $count: "fieldName" }
```

**Example:**

```javascript
// Count completed orders
db.orders.aggregate([
    { $match: { status: "completed" } },
    { $count: "total_completed" }
])

// Result: { total_completed: 850 }
```

---

## **Real Example: Complete Pipeline**

**Task:** Find top 5 customers by total spending in 2026

```javascript
db.orders.aggregate([
    // Stage 1: Filter 2026 orders
    { $match: { 
        order_date: { 
            $gte: ISODate("2026-01-01"),
            $lt: ISODate("2027-01-01")
        }
    }},
    
    // Stage 2: Group by customer, sum spending
    { $group: { 
        _id: "$customer_id",
        total_spent: { $sum: "$total" },
        order_count: { $sum: 1 }
    }},
    
    // Stage 3: Sort by spending (descending)
    { $sort: { total_spent: -1 } },
    
    // Stage 4: Top 5
    { $limit: 5 },
    
    // Stage 5: Rename _id to customer_id
    { $project: { 
        _id: 0,
        customer_id: "$_id",
        total_spent: 1,
        order_count: 1
    }}
])
```

**Result:**
```javascript
{ customer_id: "cust_001", total_spent: 15000, order_count: 45 }
{ customer_id: "cust_002", total_spent: 12000, order_count: 38 }
{ customer_id: "cust_003", total_spent: 10500, order_count: 32 }
{ customer_id: "cust_004", total_spent: 9800, order_count: 29 }
{ customer_id: "cust_005", total_spent: 9200, order_count: 27 }
```

---

## **Aggregation Pipeline - Mental Checklist**

**Step 1:** What do I want? (Define output)
**Step 2:** Filter first ($match) - reduce data early
**Step 3:** Transform if needed ($project, $addFields)
**Step 4:** Group if aggregating ($group)
**Step 5:** Sort if ordering ($sort)
**Step 6:** Limit if needed ($limit, $skip)

---

# **SECTION 7: INDEXING**

## **Why Indexes Matter**

**Without Index:**
```
Collection: 1,000,000 documents
Query: db.users.find({ email: "alice@email.com" })
Process: Scans ALL 1,000,000 documents
Time: 5 seconds
```

**With Index:**
```
Collection: 1,000,000 documents
Index on email field
Query: db.users.find({ email: "alice@email.com" })
Process: Looks up in index (B-tree), finds in 3-4 comparisons
Time: 0.005 seconds (1000x faster!)
```

---

## **Creating Indexes**

### **Single Field Index**

**Pattern:**
```javascript
db.collection.createIndex({ field: 1 })
```

**Examples:**

```javascript
// Index on email
db.users.createIndex({ email: 1 })

// Now this query is fast:
db.users.find({ email: "alice@email.com" })
```

**1 vs -1:**
- `1` = Ascending order
- `-1` = Descending order
- For single-field index, usually doesn't matter

---

### **Compound Index (Multiple Fields)**

**Pattern:**
```javascript
db.collection.createIndex({ field1: 1, field2: -1 })
```

**Example:**

```javascript
// Index on city + age
db.users.createIndex({ city: 1, age: -1 })

// Fast queries:
db.users.find({ city: "NYC" })  ✓
db.users.find({ city: "NYC", age: { $gt: 25 } })  ✓
db.users.find({ city: "NYC" }).sort({ age: -1 })  ✓

// Slow query (doesn't use index efficiently):
db.users.find({ age: { $gt: 25 } })  ✗ (age is not first in index)
```

**Index Prefix Rule:**
```javascript
// Index: { city: 1, state: 1, zip: 1 }

// Uses index:
{ city: "NYC" }  ✓
{ city: "NYC", state: "NY" }  ✓
{ city: "NYC", state: "NY", zip: "10001" }  ✓

// Doesn't use index:
{ state: "NY" }  ✗ (city missing)
{ zip: "10001" }  ✗ (city and state missing)
```

---

### **Unique Index**

**Pattern:**
```javascript
db.collection.createIndex({ field: 1 }, { unique: true })
```

**Example:**

```javascript
// Ensure email is unique
db.users.createIndex({ email: 1 }, { unique: true })

// Now duplicate inserts fail:
db.users.insertOne({ name: "Alice", email: "alice@email.com" })  ✓
db.users.insertOne({ name: "Bob", email: "alice@email.com" })  ✗ Error!
```

---

### **Partial Index (Conditional)**

**Pattern:**
```javascript
db.collection.createIndex(
    { field: 1 },
    { partialFilterExpression: { condition } }
)
```

**Example:**

```javascript
// Index only active users
db.users.createIndex(
    { email: 1 },
    { partialFilterExpression: { status: "active" } }
)

// Smaller index, faster updates
// Only helps queries that match the condition:
db.users.find({ email: "alice@email.com", status: "active" })  ✓ uses index
db.users.find({ email: "alice@email.com" })  ✗ might not use index
```

---

### **Text Index (Full-Text Search)**

**Pattern:**
```javascript
db.collection.createIndex({ field: "text" })
```

**Example:**

```javascript
// Index article content for search
db.articles.createIndex({ content: "text" })

// Search for keywords
db.articles.find({ $text: { $search: "machine learning" } })

// Search with phrases
db.articles.find({ $text: { $search: "\"neural networks\"" } })
```

---

## **Index Management**

### **List Indexes**
```javascript
db.users.getIndexes()
```

### **Drop Index**
```javascript
db.users.dropIndex("email_1")
// Or
db.users.dropIndex({ email: 1 })
```

### **Drop All Indexes (Except _id)**
```javascript
db.users.dropIndexes()
```

---

## **Analyzing Query Performance**

### **Explain Query**

**Pattern:**
```javascript
db.collection.find({ filter }).explain("executionStats")
```

**Example:**

```javascript
db.users.find({ email: "alice@email.com" }).explain("executionStats")
```

**What to Look For:**

```javascript
{
    "executionStats": {
        "executionTimeMillis": 5,  // Query took 5ms
        "totalDocsExamined": 1,    // Scanned 1 document
        "totalKeysExamined": 1,    // Scanned 1 index entry
        "nReturned": 1             // Returned 1 document
    },
    "winningPlan": {
        "stage": "FETCH",
        "inputStage": {
            "stage": "IXSCAN",     // ✓ Using index!
            "indexName": "email_1"
        }
    }
}
```

**Bad Signs:**
- `"stage": "COLLSCAN"` (collection scan = no index used)
- `totalDocsExamined` >> `nReturned` (scanning too many docs)
- High `executionTimeMillis`

**Good Signs:**
- `"stage": "IXSCAN"` (index scan)
- `totalDocsExamined` ≈ `nReturned`
- Low `executionTimeMillis`

---

## **Indexing Best Practices**

### **1. Index Frequently Queried Fields**
```javascript
// If you often query by email
db.users.createIndex({ email: 1 })
```

### **2. Index Fields Used in Sorting**
```javascript
// If you often sort by created_at
db.posts.createIndex({ created_at: -1 })
```

### **3. Compound Indexes for Common Query Patterns**
```javascript
// If you often query: city + age
db.users.createIndex({ city: 1, age: -1 })
```

### **4. Don't Over-Index**
- Each index slows down writes
- Indexes take disk space
- Rule of thumb: 5-10 indexes per collection max

### **5. Index Selectivity Matters**
```javascript
// ✓ Good: High selectivity (email is unique)
db.users.createIndex({ email: 1 })

// ✗ Bad: Low selectivity (only 2 values)
db.users.createIndex({ gender: 1 })
```

---

## **Common Index Patterns for ML**

### **1. User Feature Lookup**
```javascript
db.user_features.createIndex({ user_id: 1 })

// Fast:
db.user_features.find({ user_id: "user_12345" })
```

### **2. Time-Series Data**
```javascript
db.events.createIndex({ user_id: 1, timestamp: -1 })

// Fast:
db.events.find({ 
    user_id: "user_12345",
    timestamp: { $gte: ISODate("2026-01-01") }
}).sort({ timestamp: -1 })
```

### **3. Experiment Tracking**
```javascript
db.experiments.createIndex({ experiment_id: 1, variant: 1 })

// Fast:
db.experiments.find({ 
    experiment_id: "exp_001",
    variant: "treatment"
})
```

---

# **SECTION 8: DATA MODELING**

## **The MongoDB Philosophy: Embed vs Reference**

**This is THE most important decision in MongoDB!**

---

## **Embedding (Denormalization)**

**Concept:** Store related data together in one document

**Example: Blog Post with Comments**

```javascript
{
    _id: ObjectId("..."),
    title: "Introduction to MongoDB",
    author: "Alice",
    content: "MongoDB is a NoSQL database...",
    comments: [
        { user: "Bob", text: "Great post!", date: ISODate("...") },
        { user: "Carol", text: "Very helpful!", date: ISODate("...") }
    ],
    tags: ["mongodb", "database", "tutorial"]
}
```

**Advantages:**
✅ **Fast reads** - Get everything in one query
✅ **Atomic updates** - Update post + comments together
✅ **No joins needed** - All data in one place

**Disadvantages:**
❌ **Document size limit** - 16MB max
❌ **Data duplication** - If Bob's name changes, update all his comments
❌ **Growing arrays** - Unbounded arrays slow down

---

## **Referencing (Normalization)**

**Concept:** Store related data in separate collections, link by ID

**Example: Blog Post with Comments (Referenced)**

```javascript
// posts collection
{
    _id: ObjectId("post_001"),
    title: "Introduction to MongoDB",
    author_id: ObjectId("user_alice"),
    content: "MongoDB is a NoSQL database..."
}

// comments collection
{
    _id: ObjectId("comment_001"),
    post_id: ObjectId("post_001"),
    user_id: ObjectId("user_bob"),
    text: "Great post!",
    date: ISODate("...")
}

{
    _id: ObjectId("comment_002"),
    post_id: ObjectId("post_001"),
    user_id: ObjectId("user_carol"),
    text: "Very helpful!",
    date: ISODate("...")
}
```

**Advantages:**
✅ **No data duplication** - User info stored once
✅ **No size limits** - Unlimited comments
✅ **Independent updates** - Change user without touching comments

**Disadvantages:**
❌ **Multiple queries** - Need to fetch post + comments separately
❌ **Application-level joins** - Must join in code (or use $lookup)

---

## **When to Embed vs Reference - Decision Tree**

### **Embed When:**

**1. One-to-Few Relationship**
```javascript
// User with addresses (usually 1-5)
{
    _id: ObjectId("..."),
    name: "Alice",
    addresses: [
        { type: "home", street: "123 Main St", city: "NYC" },
        { type: "work", street: "456 Office Blvd", city: "NYC" }
    ]
}
```

**2. Data Accessed Together**
```javascript
// Order with line items (always need together)
{
    _id: ObjectId("..."),
    customer_id: ObjectId("..."),
    order_date: ISODate("..."),
    items: [
        { product_id: "prod_001", quantity: 2, price: 50 },
        { product_id: "prod_002", quantity: 1, price: 100 }
    ],
    total: 200
}
```

**3. Child Data Doesn't Change Independently**
```javascript
// Product with specifications
{
    _id: ObjectId("..."),
    name: "Laptop",
    specs: {
        cpu: "Intel i7",
        ram: "16GB",
        storage: "512GB SSD"
    }
}
```

---

### **Reference When:**

**1. One-to-Many (Thousands)**
```javascript
// User with posts (could be thousands)
// users collection
{ _id: ObjectId("user_001"), name: "Alice" }

// posts collection (separate)
{ _id: ObjectId("..."), author_id: ObjectId("user_001"), title: "..." }
{ _id: ObjectId("..."), author_id: ObjectId("user_001"), title: "..." }
// ... thousands more
```

**2. Many-to-Many Relationships**
```javascript
// Students and Courses
// students collection
{ _id: ObjectId("student_001"), name: "Alice" }

// courses collection
{ _id: ObjectId("course_001"), name: "Database Systems" }

// enrollments collection (junction)
{
    student_id: ObjectId("student_001"),
    course_id: ObjectId("course_001"),
    grade: "A"
}
```

**3. Data Changes Independently**
```javascript
// User profile can change without affecting posts
// users collection
{ _id: ObjectId("user_001"), name: "Alice", email: "alice@email.com" }

// posts collection (references user)
{ _id: ObjectId("..."), author_id: ObjectId("user_001"), title: "..." }
```

---

## **Hybrid Approach (Best of Both Worlds)**

**Pattern:** Reference + Denormalize Frequently Accessed Fields

**Example: Orders with Customer Info**

```javascript
{
    _id: ObjectId("order_001"),
    customer_id: ObjectId("cust_001"),  // Reference (for joins)
    customer_name: "Alice Smith",       // Denormalized (fast display)
    customer_email: "alice@email.com",  // Denormalized (fast display)
    order_date: ISODate("2026-02-09"),
    total: 150,
    items: [
        { product_id: "prod_001", name: "Laptop", quantity: 1, price: 150 }
    ]
}
```

**Benefits:**
✅ Fast reads (no join needed for display)
✅ Can still join if need full customer data
✅ If customer changes name, update gradually (not critical)

**Trade-off:**
❌ Some data duplication
❌ Possible stale data (customer_name might be outdated)

---

## **Real-World Example: E-commerce Data Model**

### **Embedded Approach**

```javascript
// products collection
{
    _id: ObjectId("prod_001"),
    name: "Laptop",
    price: 1200,
    category: {
        _id: "cat_001",
        name: "Electronics"
    },
    reviews: [
        { user: "Bob", rating: 5, text: "Great laptop!" },
        { user: "Carol", rating: 4, text: "Good value" }
    ],
    inventory: {
        warehouse_a: 50,
        warehouse_b: 30
    }
}
```

### **Referenced Approach**

```javascript
// categories collection
{ _id: "cat_001", name: "Electronics" }

// products collection
{
    _id: ObjectId("prod_001"),
    name: "Laptop",
    price: 1200,
    category_id: "cat_001"
}

// reviews collection
{
    _id: ObjectId("..."),
    product_id: ObjectId("prod_001"),
    user_id: ObjectId("user_bob"),
    rating: 5,
    text: "Great laptop!"
}

// inventory collection
{
    product_id: ObjectId("prod_001"),
    warehouse_a: 50,
    warehouse_b: 30
}
```

### **Which to Choose?**

**Embed** if:
- Products have 0-20 reviews (bounded)
- Inventory doesn't change often
- Category info rarely changes

**Reference** if:
- Products have hundreds of reviews
- Inventory updates frequently
- Need to query reviews separately

---

## **Document Size Considerations**

**MongoDB Limit:** 16MB per document

**Typical Sizes:**
```javascript
// Typical user profile: ~1-5KB
{
    _id: ObjectId("..."),
    name: "Alice",
    email: "alice@email.com",
    preferences: { ... },
    addresses: [ ... ]
}

// Typical order: ~5-50KB
{
    _id: ObjectId("..."),
    customer: { ... },
    items: [ ... ],  // 10-100 items
    shipping: { ... }
}

// Large document: ~1-5MB
{
    _id: ObjectId("..."),
    dataset: [ ... ],  // Thousands of data points
    features: { ... }
}
```

**⚠️ Warning Signs:**
- Array with >1000 elements → Consider referencing
- Document >1MB → Definitely reference
- Unbounded growth (comments, logs) → Reference

---

## **Data Modeling Patterns**

### **Pattern 1: Attribute Pattern (Flexible Schema)**

**Problem:** Products have different attributes

**Bad:**
```javascript
// Need different fields for each product type
{ name: "Laptop", cpu: "i7", ram: "16GB" }
{ name: "Shirt", size: "M", color: "Blue" }
```

**Good:**
```javascript
// Flexible attributes array
{
    name: "Laptop",
    attributes: [
        { key: "CPU", value: "Intel i7" },
        { key: "RAM", value: "16GB" },
        { key: "Storage", value: "512GB" }
    ]
}

{
    name: "Shirt",
    attributes: [
        { key: "Size", value: "M" },
        { key: "Color", value: "Blue" },
        { key: "Material", value: "Cotton" }
    ]
}

// Index for querying
db.products.createIndex({ "attributes.key": 1, "attributes.value": 1 })

// Query
db.products.find({ 
    attributes: { 
        $elemMatch: { key: "CPU", value: "Intel i7" } 
    }
})
```

---

### **Pattern 2: Bucket Pattern (Time-Series)**

**Problem:** One document per sensor reading = millions of documents

**Bad:**
```javascript
// 1 document per reading
{ sensor_id: 123, timestamp: ISODate("..."), temp: 22.5 }
{ sensor_id: 123, timestamp: ISODate("..."), temp: 22.7 }
{ sensor_id: 123, timestamp: ISODate("..."), temp: 22.9 }
// ... millions more
```

**Good:**
```javascript
// Bucket: 1 document per hour (60 readings)
{
    sensor_id: 123,
    bucket_hour: ISODate("2026-02-09T14:00:00"),
    measurements: [
        { timestamp: ISODate("2026-02-09T14:01:00"), temp: 22.5 },
        { timestamp: ISODate("2026-02-09T14:02:00"), temp: 22.7 },
        { timestamp: ISODate("2026-02-09T14:03:00"), temp: 22.9 },
        // ... up to 60 measurements
    ],
    count: 60,
    avg_temp: 22.8,
    max_temp: 23.1,
    min_temp: 22.3
}
```

**Benefits:**
- 60x fewer documents
- Pre-computed aggregations (avg, max, min)
- Faster queries

---

### **Pattern 3: Computed Pattern (Cache Expensive Calculations)**

**Problem:** Expensive aggregations run on every query

**Bad:**
```javascript
// Count likes every time
db.posts.aggregate([
    { $match: { post_id: "post_001" } },
    { $lookup: {
        from: "likes",
        localField: "_id",
        foreignField: "post_id",
        as: "likes"
    }},
    { $addFields: { like_count: { $size: "$likes" } } }
])
```

**Good:**
```javascript
// Store pre-computed count
{
    _id: ObjectId("post_001"),
    title: "Introduction to MongoDB",
    content: "...",
    like_count: 150,  // Updated when likes change
    comment_count: 23,
    last_updated: ISODate("...")
}

// Update via trigger or increment
db.posts.updateOne(
    { _id: ObjectId("post_001") },
    { $inc: { like_count: 1 } }
)
```

---

## **Data Modeling for ML/AI**

### **Feature Store**

```javascript
{
    _id: ObjectId("..."),
    user_id: "user_12345",
    features: {
        // Demographics
        age: 30,
        country: "USA",
        
        // Behavioral
        logins_30d: 15,
        purchases_30d: 3,
        avg_session_duration: 320,
        
        // Derived
        days_since_last_login: 2,
        total_lifetime_value: 1500,
        churn_risk_score: 0.23
    },
    computed_at: ISODate("2026-02-09T10:00:00")
}

// Index for fast lookup
db.user_features.createIndex({ user_id: 1 })

// TTL index (auto-delete stale features after 24 hours)
db.user_features.createIndex(
    { computed_at: 1 },
    { expireAfterSeconds: 86400 }
)
```

---

### **Model Predictions**

```javascript
{
    _id: ObjectId("..."),
    user_id: "user_12345",
    model_id: "churn_v2.1",
    predicted_class: 1,  // Will churn
    probability: 0.87,
    features_snapshot: {  // Store features used for prediction
        logins_30d: 0,
        purchases_30d: 0
    },
    prediction_date: ISODate("2026-02-09"),
    model_version: "2.1.0"
}

// Compound index for querying
db.predictions.createIndex({ user_id: 1, prediction_date: -1 })

// TTL index (auto-delete old predictions after 30 days)
db.predictions.createIndex(
    { prediction_date: 1 },
    { expireAfterSeconds: 2592000 }
)
```

---

# **SECTION 9: ADVANCED AGGREGATION PATTERNS**

## **$lookup (JOIN in MongoDB)**

**Purpose:** Join two collections (like SQL JOIN)

**Pattern:**
```javascript
{ $lookup: {
    from: "otherCollection",
    localField: "fieldInThisCollection",
    foreignField: "fieldInOtherCollection",
    as: "outputArrayName"
}}
```

---

### **Basic $lookup Example**

**Scenario:** Join orders with customers

```javascript
// orders collection
{ _id: 1, customer_id: "cust_001", total: 100 }
{ _id: 2, customer_id: "cust_002", total: 200 }

// customers collection
{ _id: "cust_001", name: "Alice", email: "alice@email.com" }
{ _id: "cust_002", name: "Bob", email: "bob@email.com" }

// Join them
db.orders.aggregate([
    { $lookup: {
        from: "customers",
        localField: "customer_id",
        foreignField: "_id",
        as: "customer_info"
    }}
])
```

**Result:**
```javascript
{
    _id: 1,
    customer_id: "cust_001",
    total: 100,
    customer_info: [
        { _id: "cust_001", name: "Alice", email: "alice@email.com" }
    ]
}
```

**⚠️ Note:** Result is an array (even for one match)

---

### **Flattening $lookup Result**

**Use $unwind to convert array to single object:**

```javascript
db.orders.aggregate([
    { $lookup: {
        from: "customers",
        localField: "customer_id",
        foreignField: "_id",
        as: "customer_info"
    }},
    { $unwind: "$customer_info" },  // Flatten array
    { $project: {
        _id: 1,
        total: 1,
        customer_name: "$customer_info.name",
        customer_email: "$customer_info.email"
    }}
])
```

**Result:**
```javascript
{
    _id: 1,
    total: 100,
    customer_name: "Alice",
    customer_email: "alice@email.com"
}
```

---

## **$unwind (Flatten Arrays)**

**Purpose:** Convert array into separate documents

**Pattern:**
```javascript
{ $unwind: "$arrayField" }
```

**Example:**

**Before:**
```javascript
{
    _id: 1,
    customer: "Alice",
    items: ["Laptop", "Mouse", "Keyboard"]
}
```

**After $unwind:**
```javascript
db.orders.aggregate([
    { $unwind: "$items" }
])

// Results in 3 documents:
{ _id: 1, customer: "Alice", items: "Laptop" }
{ _id: 1, customer: "Alice", items: "Mouse" }
{ _id: 1, customer: "Alice", items: "Keyboard" }
```

---

### **Real Example: Order Items Analysis**

```javascript
// Order with embedded items
{
    _id: 1,
    customer_id: "cust_001",
    items: [
        { product_id: "prod_001", quantity: 2, price: 50 },
        { product_id: "prod_002", quantity: 1, price: 100 }
    ]
}

// Find total quantity sold per product
db.orders.aggregate([
    { $unwind: "$items" },  // One doc per item
    { $group: {
        _id: "$items.product_id",
        total_quantity: { $sum: "$items.quantity" },
        total_revenue: { $sum: { $multiply: ["$items.quantity", "$items.price"] } }
    }}
])
```

---

## **$facet (Multiple Aggregations in One Query)**

**Purpose:** Run multiple pipelines in parallel

**Pattern:**
```javascript
{ $facet: {
    pipeline1Name: [ { stage1 }, { stage2 } ],
    pipeline2Name: [ { stage1 }, { stage2 } ]
}}
```

**Example: Get Statistics and Top Products Simultaneously**

```javascript
db.products.aggregate([
    { $facet: {
        // Pipeline 1: Overall statistics
        statistics: [
            { $group: {
                _id: null,
                total_products: { $sum: 1 },
                avg_price: { $avg: "$price" },
                max_price: { $max: "$price" }
            }}
        ],
        
        // Pipeline 2: Top 5 expensive products
        top_products: [
            { $sort: { price: -1 } },
            { $limit: 5 },
            { $project: { name: 1, price: 1 } }
        ],
        
        // Pipeline 3: Price distribution
        price_ranges: [
            { $bucket: {
                groupBy: "$price",
                boundaries: [0, 100, 500, 1000, 5000],
                default: "5000+",
                output: { count: { $sum: 1 } }
            }}
        ]
    }}
])
```

**Result:**
```javascript
{
    statistics: [
        { _id: null, total_products: 1000, avg_price: 450, max_price: 3000 }
    ],
    top_products: [
        { _id: "...", name: "Laptop Pro", price: 3000 },
        { _id: "...", name: "Smartphone", price: 1200 },
        // ... 3 more
    ],
    price_ranges: [
        { _id: 0, count: 200 },      // $0-100
        { _id: 100, count: 400 },    // $100-500
        { _id: 500, count: 300 },    // $500-1000
        { _id: 1000, count: 80 },    // $1000-5000
        { _id: "5000+", count: 20 }  // $5000+
    ]
}
```

---

## **$bucket (Group by Ranges)**

**Purpose:** Group documents into ranges

**Pattern:**
```javascript
{ $bucket: {
    groupBy: "$field",
    boundaries: [0, 10, 20, 30],
    default: "Other",
    output: { count: { $sum: 1 } }
}}
```

**Example: Age Distribution**

```javascript
db.users.aggregate([
    { $bucket: {
        groupBy: "$age",
        boundaries: [0, 18, 30, 50, 100],
        default: "100+",
        output: {
            count: { $sum: 1 },
            names: { $push: "$name" }
        }
    }}
])
```

**Result:**
```javascript
{ _id: 0, count: 50, names: ["Alice", "Bob", ...] }     // 0-17
{ _id: 18, count: 200, names: ["Carol", "Dave", ...] }  // 18-29
{ _id: 30, count: 150, names: ["Eve", "Frank", ...] }   // 30-49
{ _id: 50, count: 80, names: [...] }                    // 50-99
{ _id: "100+", count: 2, names: [...] }                 // 100+
```

---

## **Conditional Operators in Aggregation**

### **$cond (If-Then-Else)**

**Pattern:**
```javascript
{ $cond: [ condition, valueIfTrue, valueIfFalse ] }
```

**Example:**

```javascript
db.orders.aggregate([
    { $addFields: {
        order_size: { 
            $cond: [ 
                { $gte: ["$total", 1000] },
                "large",
                "small"
            ]
        }
    }}
])
```

---

### **$switch (Multiple Conditions)**

**Pattern:**
```javascript
{ $switch: {
    branches: [
        { case: condition1, then: value1 },
        { case: condition2, then: value2 }
    ],
    default: defaultValue
}}
```

**Example:**

```javascript
db.users.aggregate([
    { $addFields: {
        age_group: {
            $switch: {
                branches: [
                    { case: { $lt: ["$age", 18] }, then: "minor" },
                    { case: { $lt: ["$age", 30] }, then: "young adult" },
                    { case: { $lt: ["$age", 50] }, then: "adult" },
                    { case: { $gte: ["$age", 50] }, then: "senior" }
                ],
                default: "unknown"
            }
        }
    }}
])
```

---

## **String Operations**

### **$concat (Concatenate Strings)**

```javascript
db.users.aggregate([
    { $addFields: {
        full_name: { 
            $concat: ["$first_name", " ", "$last_name"] 
        }
    }}
])
```

### **$split & $arrayElemAt**

```javascript
// Extract domain from email
db.users.aggregate([
    { $addFields: {
        email_domain: { 
            $arrayElemAt: [ 
                { $split: ["$email", "@"] }, 
                1 
            ]
        }
    }}
])

// alice@gmail.com → gmail.com
```

---

## **Date Operations**

### **Extract Date Parts**

```javascript
db.orders.aggregate([
    { $addFields: {
        year: { $year: "$order_date" },
        month: { $month: "$order_date" },
        day: { $dayOfMonth: "$order_date" },
        hour: { $hour: "$order_date" },
        day_of_week: { $dayOfWeek: "$order_date" }  // 1=Sunday, 7=Saturday
    }}
])
```

### **Date Difference**

```javascript
db.users.aggregate([
    { $addFields: {
        days_since_registration: {
            $dateDiff: {
                startDate: "$registration_date",
                endDate: new Date(),
                unit: "day"
            }
        }
    }}
])
```

---

# **SECTION 10: REAL-WORLD ML/AI SCENARIOS**

## **Scenario 1: Feature Engineering for Churn Prediction**

**Goal:** Extract user features for ML model

```javascript
db.events.aggregate([
    // Filter last 90 days
    { $match: {
        event_date: { $gte: new Date("2026-01-01") }
    }},
    
    // Group by user, create features
    { $group: {
        _id: "$user_id",
        
        // Count different event types
        login_count: { 
            $sum: { $cond: [{ $eq: ["$event_type", "login"] }, 1, 0] } 
        },
        purchase_count: { 
            $sum: { $cond: [{ $eq: ["$event_type", "purchase"] }, 1, 0] } 
        },
        view_count: { 
            $sum: { $cond: [{ $eq: ["$event_type", "view"] }, 1, 0] } 
        },
        
        // Total spent
        total_spent: { 
            $sum: { $cond: [{ $eq: ["$event_type", "purchase"] }, "$amount", 0] } 
        },
        
        // Last activity date
        last_activity: { $max: "$event_date" },
        
        // Unique products viewed
        unique_products_viewed: { $addToSet: "$product_id" }
    }},
    
    // Add derived features
    { $addFields: {
        days_since_activity: {
            $dateDiff: {
                startDate: "$last_activity",
                endDate: new Date(),
                unit: "day"
            }
        },
        avg_purchase_value: {
            $cond: [
                { $gt: ["$purchase_count", 0] },
                { $divide: ["$total_spent", "$purchase_count"] },
                0
            ]
        },
        unique_products_count: { $size: "$unique_products_viewed" }
    }},
    
    // Join with user demographics
    { $lookup: {
        from: "users",
        localField: "_id",
        foreignField: "user_id",
        as: "user_data"
    }},
    
    { $unwind: "$user_data" },
    
    // Final output
    { $project: {
        user_id: "$_id",
        login_count: 1,
        purchase_count: 1,
        view_count: 1,
        total_spent: 1,
        days_since_activity: 1,
        avg_purchase_value: 1,
        unique_products_count: 1,
        age: "$user_data.age",
        country: "$user_data.country",
        registration_days: {
            $dateDiff: {
                startDate: "$user_data.registration_date",
                endDate: new Date(),
                unit: "day"
            }
        },
        churned: "$user_data.churned"  // Label
    }},
    
    // Save to collection for export
    { $out: "ml_training_data" }
])
```

---

## **Scenario 2: Storing and Querying Predictions**

### **Insert Predictions**

```javascript
// Batch insert predictions
db.predictions.insertMany([
    {
        user_id: "user_001",
        model_id: "churn_v2.1",
        predicted_class: 1,
        probability: 0.87,
        features_snapshot: {
            logins_30d: 0,
            purchases_30d: 0
        },
        prediction_date: new Date(),
        model_version: "2.1.0"
    },
    // ... more predictions
])

// Create indexes
db.predictions.createIndex({ user_id: 1, prediction_date: -1 })
db.predictions.createIndex({ model_id: 1 })

// TTL index (auto-delete after 30 days)
db.predictions.createIndex(
    { prediction_date: 1 },
    { expireAfterSeconds: 2592000 }
)
```

### **Query Recent Predictions**

```javascript
// Get latest prediction for each user
db.predictions.aggregate([
    { $sort: { prediction_date: -1 } },
    { $group: {
        _id: "$user_id",
        latest_prediction: { $first: "$$ROOT" }
    }},
    { $replaceRoot: { newRoot: "$latest_prediction" } }
])
```

---

## **Scenario 3: A/B Test Analysis**

### **Store Experiment Events**

```javascript
db.experiment_events.insertMany([
    {
        user_id: "user_001",
        experiment_id: "homepage_redesign",
        variant: "control",
        event_type: "impression",
        timestamp: new Date()
    },
    {
        user_id: "user_001",
        experiment_id: "homepage_redesign",
        variant: "control",
        event_type: "click",
        timestamp: new Date()
    }
])
```

### **Calculate Metrics**

```javascript
db.experiment_events.aggregate([
    // Filter experiment
    { $match: { experiment_id: "homepage_redesign" } },
    
    // Group by variant
    { $group: {
        _id: "$variant",
        
        // Unique users
        users: { $addToSet: "$user_id" },
        
        // Count events
        impressions: {
            $sum: { $cond: [{ $eq: ["$event_type", "impression"] }, 1, 0] }
        },
        clicks: {
            $sum: { $cond: [{ $eq: ["$event_type", "click"] }, 1, 0] }
        },
        conversions: {
            $sum: { $cond: [{ $eq: ["$event_type", "conversion"] }, 1, 0] }
        }
    }},
    
    // Calculate rates
    { $addFields: {
        user_count: { $size: "$users" },
        ctr: { 
            $divide: ["$clicks", { $max: ["$impressions", 1] }] 
        },
        conversion_rate: { 
            $divide: ["$conversions", { $max: ["$user_count", 1] }] 
        }
    }},
    
    // Clean output
    { $project: {
        variant: "$_id",
        user_count: 1,
        impressions: 1,
        clicks: 1,
        conversions: 1,
        ctr: { $round: ["$ctr", 4] },
        conversion_rate: { $round: ["$conversion_rate", 4] },
        _id: 0
    }}
])
```

**Result:**
```javascript
{
    variant: "control",
    user_count: 10000,
    impressions: 50000,
    clicks: 6000,
    conversions: 1200,
    ctr: 0.12,
    conversion_rate: 0.12
}
{
    variant: "treatment",
    user_count: 10000,
    impressions: 50000,
    clicks: 6750,
    conversions: 1350,
    ctr: 0.135,
    conversion_rate: 0.135
}
```

---

## **Scenario 4: Time-Series Analysis (Sensor Data)**

### **Bucket Pattern for Efficiency**

```javascript
// Instead of millions of individual readings
// Store hourly buckets

{
    sensor_id: 123,
    bucket_hour: ISODate("2026-02-09T14:00:00"),
    measurements: [
        { timestamp: ISODate("2026-02-09T14:01:00"), temp: 22.5, humidity: 45 },
        { timestamp: ISODate("2026-02-09T14:02:00"), temp: 22.7, humidity: 46 },
        // ... up to 60 measurements
    ],
    count: 60,
    avg_temp: 22.8,
    max_temp: 23.1,
    min_temp: 22.3,
    avg_humidity: 45.5
}
```

### **Query Aggregate Statistics**

```javascript
db.sensor_data.aggregate([
    // Filter date range
    { $match: {
        sensor_id: 123,
        bucket_hour: {
            $gte: ISODate("2026-02-01"),
            $lt: ISODate("2026-02-09")
        }
    }},
    
    // Overall statistics
    { $group: {
        _id: null,
        overall_avg_temp: { $avg: "$avg_temp" },
        overall_max_temp: { $max: "$max_temp" },
        overall_min_temp: { $min: "$min_temp" },
        total_readings: { $sum: "$count" }
    }}
])
```

---

## **Scenario 5: Recommendation System - Item Co-occurrence**

### **Calculate Item Co-occurrence Matrix**

```javascript
// Find products often bought together
db.orders.aggregate([
    // Unwind items to get one row per item
    { $unwind: "$items" },
    
    // Group by order, collect product IDs
    { $group: {
        _id: "$order_id",
        products: { $push: "$items.product_id" }
    }},
    
    // Self-join to get pairs
    { $unwind: "$products" },
    { $project: {
        product_a: "$products",
        all_products: "$products"
    }},
    { $unwind: "$all_products" },
    
    // Remove self-pairs
    { $match: {
        $expr: { $ne: ["$product_a", "$all_products"] }
    }},
    
    // Count co-occurrences
    { $group: {
        _id: {
            product_a: "$product_a",
            product_b: "$all_products"
        },
        co_occurrence_count: { $sum: 1 }
    }},
    
    // Sort by frequency
    { $sort: { co_occurrence_count: -1 } },
    
    // Save results
    { $out: "product_recommendations" }
])
```

---

# **SECTION 11: MONGODB VS SQL COMPARISON**

## **Side-by-Side Query Comparison**

### **Simple SELECT**

**SQL:**
```sql
SELECT name, age FROM users WHERE age > 25;
```

**MongoDB:**
```javascript
db.users.find(
    { age: { $gt: 25 } },
    { name: 1, age: 1, _id: 0 }
)
```

---

### **Aggregation (GROUP BY)**

**SQL:**
```sql
SELECT city, COUNT(*) as user_count, AVG(age) as avg_age
FROM users
GROUP BY city
HAVING COUNT(*) > 100
ORDER BY user_count DESC;
```

**MongoDB:**
```javascript
db.users.aggregate([
    { $group: {
        _id: "$city",
        user_count: { $sum: 1 },
        avg_age: { $avg: "$age" }
    }},
    { $match: { user_count: { $gt: 100 } } },
    { $sort: { user_count: -1 } },
    { $project: {
        city: "$_id",
        user_count: 1,
        avg_age: 1,
        _id: 0
    }}
])
```

---

### **JOIN**

**SQL:**
```sql
SELECT o.order_id, o.total, c.name, c.email
FROM orders o
INNER JOIN customers c ON o.customer_id = c.customer_id;
```

**MongoDB:**
```javascript
db.orders.aggregate([
    { $lookup: {
        from: "customers",
        localField: "customer_id",
        foreignField: "_id",
        as: "customer"
    }},
    { $unwind: "$customer" },
    { $project: {
        order_id: "$_id",
        total: 1,
        name: "$customer.name",
        email: "$customer.email",
        _id: 0
    }}
])
```

---

### **Subquery**

**SQL:**
```sql
SELECT name, age
FROM users
WHERE age > (SELECT AVG(age) FROM users);
```

**MongoDB:**
```javascript
// Method 1: Two queries
const avgAge = db.users.aggregate([
    { $group: { _id: null, avg: { $avg: "$age" } } }
]).toArray()[0].avg

db.users.find({ age: { $gt: avgAge } }, { name: 1, age: 1 })

// Method 2: Single aggregation
db.users.aggregate([
    { $group: {
        _id: null,
        avg_age: { $avg: "$age" },
        all_users: { $push: "$$ROOT" }
    }},
    { $unwind: "$all_users" },
    { $match: {
        $expr: { $gt: ["$all_users.age", "$avg_age"] }
    }},
    { $replaceRoot: { newRoot: "$all_users" } },
    { $project: { name: 1, age: 1 } }
])
```

---

### **UPDATE**

**SQL:**
```sql
UPDATE users
SET age = age + 1
WHERE city = 'NYC';
```

**MongoDB:**
```javascript
db.users.updateMany(
    { city: "NYC" },
    { $inc: { age: 1 } }
)
```

---

### **DELETE**

**SQL:**
```sql
DELETE FROM users WHERE age < 18;
```

**MongoDB:**
```javascript
db.users.deleteMany({ age: { $lt: 18 } })
```

---

## **Concept Mapping**

| SQL Concept | MongoDB Equivalent |
|-------------|-------------------|
| Database | Database |
| Table | Collection |
| Row | Document |
| Column | Field |
| Primary Key | `_id` field |
| Foreign Key | Reference (manual) |
| Index | Index |
| JOIN | `$lookup` |
| GROUP BY | `$group` |
| HAVING | `$match` (after `$group`) |
| WHERE | `$match` |
| SELECT | `$project` |
| ORDER BY | `$sort` |
| LIMIT | `$limit` |
| OFFSET | `$skip` |
| COUNT(*) | `countDocuments()` or `{ $sum: 1 }` |
| DISTINCT | `distinct()` or `$group` |
| UNION | `$unionWith` |
| Subquery | Nested aggregation or variable |

---

# **SECTION 12: MEMORY TRICKS & CHEAT SHEETS**

## **The Bracket Decoder**

```javascript
{ }  →  Object (key: value pairs)
[ ]  →  Array (list of items)
$    →  MongoDB operator
```

**Pattern Recognition:**
```javascript
{ key: value }                     // Simple property
{ key: { $op: value } }           // Operator (comparison, logical)
{ $stage: { ... } }               // Aggregation stage
[ { }, { } ]                      // Array of objects (pipeline, $or, etc.)
```

---

## **CRUD Quick Reference**

```javascript
// CREATE
insertOne({ ... })
insertMany([{ }, { }])

// READ
find({ filter }, { projection })
findOne({ filter })

// UPDATE
updateOne({ filter }, { $set: { ... } })
updateMany({ filter }, { $set: { ... } })

// DELETE
deleteOne({ filter })
deleteMany({ filter })
```

---

## **Query Operators Cheat Sheet**

```javascript
// Comparison
{ age: { $eq: 30 } }        // Equal (or just { age: 30 })
{ age: { $ne: 30 } }        // Not equal
{ age: { $gt: 25 } }        // Greater than
{ age: { $gte: 25 } }       // Greater than or equal
{ age: { $lt: 35 } }        // Less than
{ age: { $lte: 35 } }       // Less than or equal
{ city: { $in: ["NYC", "LA"] } }      // In array
{ city: { $nin: ["NYC", "LA"] } }     // Not in array

// Logical
{ $and: [{ }, { }] }
{ $or: [{ }, { }] }
{ $not: { $gt: 30 } }
{ $nor: [{ }, { }] }

// Element
{ phone: { $exists: true } }
{ age: { $type: "int" } }

// Array
{ tags: { $all: ["ml", "ai"] } }
{ tags: { $size: 3 } }
{ tags: { $elemMatch: { $gt: 10 } } }

// String
{ email: { $regex: /@gmail\.com$/ } }
```

---

## **Update Operators Cheat Sheet**

```javascript
// Field
{ $set: { age: 31 } }              // Set value
{ $unset: { phone: "" } }          // Remove field
{ $inc: { age: 1 } }               // Increment
{ $mul: { price: 1.1 } }           // Multiply
{ $rename: { "phone": "mobile" } } // Rename field
{ $currentDate: { updated: true } }// Set to now

// Array
{ $push: { tags: "new" } }         // Add to array
{ $pull: { tags: "old" } }         // Remove from array
{ $addToSet: { tags: "unique" } }  // Add if not exists
{ $pop: { tags: 1 } }              // Remove last (1) or first (-1)
```

---

## **Aggregation Pipeline Cheat Sheet**

```javascript
[
    { $match: { filter } },                    // WHERE
    { $project: { field: 1 } },                // SELECT
    { $group: { _id: "$field", count: { $sum: 1 } } }, // GROUP BY
    { $sort: { field: -1 } },                  // ORDER BY
    { $limit: 10 },                            // LIMIT
    { $skip: 20 },                             // OFFSET
    { $lookup: { from: "col", ... } },         // JOIN
    { $unwind: "$array" },                     // Flatten array
    { $addFields: { new: "$old" } },           // Add fields
    { $count: "total" },                       // COUNT(*)
    { $out: "newCollection" }                  // Save results
]
```

---

## **Aggregation Accumulators**

```javascript
{ $sum: "$field" }         // SUM(field)
{ $sum: 1 }                // COUNT(*)
{ $avg: "$field" }         // AVG(field)
{ $max: "$field" }         // MAX(field)
{ $min: "$field" }         // MIN(field)
{ $first: "$field" }       // First value
{ $last: "$field" }        // Last value
{ $push: "$field" }        // Array of values
{ $addToSet: "$field" }    // Array of unique values
```

---

## **Common Patterns - Mental Templates**

### **Pattern 1: Filter → Group → Sort**
```javascript
db.collection.aggregate([
    { $match: { status: "active" } },
    { $group: { _id: "$category", count: { $sum: 1 } } },
    { $sort: { count: -1 } }
])
```

### **Pattern 2: Join → Unwind → Project**
```javascript
db.collection.aggregate([
    { $lookup: { from: "other", localField: "id", foreignField: "ref", as: "data" } },
    { $unwind: "$data" },
    { $project: { field1: 1, field2: "$data.field" } }
])
```

### **Pattern 3: Date Range → Group by Time**
```javascript
db.collection.aggregate([
    { $match: { date: { $gte: ISODate("2026-01-01") } } },
    { $group: { 
        _id: { $dateToString: { format: "%Y-%m", date: "$date" } },
        total: { $sum: "$amount" }
    }}
])
```

---

## **Memory Tricks**

### **Dollar Signs ($)**
- **$ in queries:** Operator (`$gt`, `$match`)
- **$ in field reference:** "Get this field" (`"$age"`)
- **$$ system variable:** (`$$ROOT`, `$$CURRENT`)

### **1 vs -1**
- **In sort:** `1` = ascending (A→Z, 0→9), `-1` = descending (Z→A, 9→0)
- **In projection:** `1` = include, `0` = exclude

### **Array [ ] vs Object { }**
- **Array `[ ]`:** List of things (pipeline stages, $or conditions, multi-insert)
- **Object `{ }`:** Single thing with properties (document, filter, stage)

---

## **Common Mistakes & Fixes**

### **1. Missing $ in Field Reference**
```javascript
// ❌ WRONG
{ $group: { _id: "city", total: { $sum: "amount" } } }

// ✓ CORRECT
{ $group: { _id: "$city", total: { $sum: "$amount" } } }
```

### **2. Forgot $set in Update**
```javascript
// ❌ WRONG (replaces document!)
db.users.updateOne({ name: "Alice" }, { age: 31 })

// ✓ CORRECT
db.users.updateOne({ name: "Alice" }, { $set: { age: 31 } })
```

### **3. Wrong Bracket Type**
```javascript
// ❌ WRONG
db.users.find([ { age: 30 } ])

// ✓ CORRECT
db.users.find({ age: 30 })
```

### **4. Operator Outside Object**
```javascript
// ❌ WRONG
db.users.find({ age $gt: 25 })

// ✓ CORRECT
db.users.find({ age: { $gt: 25 } })
```

---

## **Practice Reading Queries Aloud**

### **Query:**
```javascript
db.orders.aggregate([
    { $match: { status: "completed" } },
    { $group: { _id: "$customer_id", total: { $sum: "$amount" } } },
    { $sort: { total: -1 } },
    { $limit: 10 }
])
```

### **Read as:**
1. "Match orders where status equals completed"
2. "Group by customer_id, sum the amount as total"
3. "Sort by total descending"
4. "Limit to 10 results"

### **Result:**
"Top 10 customers by total spending on completed orders"

---

## **Final Tips for Mastery**

### **1. Start Simple**
- Begin with `find()`, `updateOne()`, `deleteOne()`
- Master basic operators (`$gt`, `$in`, `$regex`)
- Then move to aggregation

### **2. Use MongoDB Compass (GUI)**
- Visual query builder
- See results instantly
- Export to code

### **3. Think in Pipelines**
- Break complex queries into stages
- Test each stage independently
- Build up gradually

### **4. Practice Common Patterns**
- Filter → Group → Sort
- Join → Unwind → Project
- Match → Count

### **5. Use explain()**
- Check if indexes are used
- Find slow queries
- Optimize based on data

---

**You now have a complete guide to MongoDB! Focus on CRUD operations, aggregation pipeline, and indexing - these cover 90% of real-world ML/AI database work.**
---
title: "Slowly Changing Dimensions: Type 0 Through Type 7 Explained With Real Examples"
source: "https://medium.com/@reliabledataengineering/slowly-changing-dimensions-type-0-through-type-7-explained-with-real-examples-f9fe8348eb54"
author:
  - "[[Reliable Data Engineering]]"
published: 2026-01-19
created: 2026-04-16
description: "“” is published by Reliable Data Engineering."
tags:
  - "clippings"
---
Get unlimited access to the best of Medium for less than $1/week.[Become a member](https://medium.com/plans?source=upgrade_membership---post_top_nav_upsell-----------------------------------------)

[

Become a member

](https://medium.com/plans?source=upgrade_membership---post_top_nav_upsell-----------------------------------------)

## Understanding how to handle dimension changes in your data warehouse with practical implementations and when to use each type

![](https://miro.medium.com/v2/resize:fit:1400/format:webp/1*v0KRbBQARLa-JKpdU8TpEQ.png)

**Disclaimer:** The “best” SCD type depends on your business requirements: audit needs, storage constraints, query patterns, and regulatory compliance. The implementations shown are simplified examples — production systems may need additional considerations like ETL frameworks, error handling, and performance optimization. Storage and performance characteristics vary by platform and data volume.

## The Problem: Dimensions Change Over Time

You have a customer dimension in your data warehouse:

```c
CREATE TABLE dim_customers (
    customer_key INT PRIMARY KEY,
    customer_id VARCHAR(50),
    name VARCHAR(200),
    email VARCHAR(200),
    address VARCHAR(500),
    city VARCHAR(100),
    tier VARCHAR(20)  -- 'Bronze', 'Silver', 'Gold'
);
```
```c
-- Customer 123 on Jan 1, 2024
customer_key: 1
customer_id: 'C123'
name: 'John Doe'
email: 'john@email.com'
address: '123 Main St'
city: 'Boston'
tier: 'Silver'
```

**On March 15, 2024, things change:**

- Customer moves: new address is ‘456 Oak Ave’, new city is ‘Seattle’
- Customer upgraded: tier changes from ‘Silver’ to ‘Gold’

**The questions:**

1. How do you update this record?
2. Do you keep the old values?
3. Can you answer: “What was customer’s tier on Feb 1, 2024?”
4. Can you answer: “How much revenue from Gold tier customers in Q1?”
5. Do your historical reports change retroactively?

**Different SCD types give different answers.**

## SCD Type 0: Retain Original

**What it is:** Never change the value. Keep original value forever.

**Philosophy:** “This attribute should never change because it represents a fundamental truth.”

## Implementation

```c
CREATE TABLE dim_customers (
    customer_key INT PRIMARY KEY,
    customer_id VARCHAR(50),
    name VARCHAR(200),
    birth_date DATE,           -- Type 0: Never changes
    original_signup_date DATE, -- Type 0: Never changes
    first_purchase_date DATE   -- Type 0: Never changes
);
```
```c
-- On insert
INSERT INTO dim_customers VALUES 
(1, 'C123', 'John Doe', '1985-05-15', '2020-01-10', '2020-01-15');-- Later, when trying to update
-- ETL logic: Skip these columns entirely
UPDATE dim_customers
SET name = 'John Smith'  -- Name can change
-- birth_date, original_signup_date, first_purchase_date NEVER updated
WHERE customer_id = 'C123';
```

## Real-World Examples

**Use Type 0 for:**

```c
-- Birth date (biologically can't change)
birth_date DATE
```
```c
-- Initial account creation date (historical fact)
account_created_date DATE-- First transaction date (historical fact)
first_purchase_date DATE-- Original source system (how customer was acquired)
original_source VARCHAR(50)  -- 'web', 'mobile', 'retail'-- Social Security Number (shouldn't change in normal circumstances)
ssn_encrypted VARCHAR(100)
```

**ETL logic:**

```c
# When processing customer updates
def update_customer(new_data, existing_record):
    type_0_fields = ['birth_date', 'account_created_date', 'first_purchase_date']
    
    for field in new_data:
        if field in type_0_fields:
            # Verify it matches, but don't update
            if new_data[field] != existing_record[field]:
                log_error(f"Type 0 field {field} changed! Investigate data quality issue")
            continue  # Skip update
        else:
            existing_record[field] = new_data[field]  # Update other fields
```

## Pros and Cons

**Pros:**

- ✅ Simple to understand
- ✅ Protects immutable data
- ✅ No storage overhead
- ✅ No complexity in queries

**Cons:**

- ❌ Can’t handle legitimate changes (e.g., legal name change)
- ❌ No history tracking for these fields
- ❌ Requires manual intervention if value truly needs correction

## SCD Type 1: Overwrite

**What it is:** Just update the record. Don’t keep history.

**Philosophy:** “We only care about current state. History doesn’t matter.”

## Implementation

```c
CREATE TABLE dim_customers (
    customer_key INT PRIMARY KEY,
    customer_id VARCHAR(50),
    name VARCHAR(200),
    email VARCHAR(200),
    address VARCHAR(500),
    city VARCHAR(100),
    tier VARCHAR(20)
);

-- Initial state (Jan 1)
INSERT INTO dim_customers VALUES 
(1, 'C123', 'John Doe', 'john@email.com', '123 Main St', 'Boston', 'Silver');

-- Update (Mar 15) - Overwrites old values
UPDATE dim_customers
SET address = '456 Oak Ave',
    city = 'Seattle',
    tier = 'Gold'
WHERE customer_id = 'C123';

-- Result: Old values lost
customer_key: 1
customer_id: 'C123'
name: 'John Doe'
email: 'john@email.com'
address: '456 Oak Ave'    -- Changed
city: 'Seattle'           -- Changed
tier: 'Gold'              -- Changed
```
```c
-- Update (Mar 15) - Overwrites old values
UPDATE dim_customers
SET address = '456 Oak Ave',
    city = 'Seattle',
    tier = 'Gold'
WHERE customer_id = 'C123';-- Result: Old values lost
customer_key: 1
customer_id: 'C123'
name: 'John Doe'
email: 'john@email.com'
address: '456 Oak Ave'    -- Changed
city: 'Seattle'           -- Changed
tier: 'Gold'              -- Changed
```

**Historical queries are affected:**

```c
-- Query on April 1: "What was revenue by tier in Q1 2024?"
SELECT 
    c.tier,
    SUM(f.revenue) as total_revenue
FROM fact_sales f
JOIN dim_customers c ON f.customer_key = c.customer_key
WHERE f.date_key BETWEEN 20240101 AND 20240331
GROUP BY c.tier;

-- Shows customer C123 as 'Gold' for ALL of Q1
-- But they were actually 'Silver' in Jan-Feb, 'Gold' in Mar
-- Historical data is retroactively changed!
```

## When to Use Type 1

**Good candidates:**

```c
-- Corrections of errors
phone_number VARCHAR(20)  -- User entered wrong number, fixed it

-- Current contact information
email VARCHAR(200)  -- Only need current email for communication

-- Non-critical attributes
favorite_color VARCHAR(50)  -- Nobody cares about historical favorite colors

-- Attributes that don't affect analytics
marketing_opt_in BOOLEAN  -- Current preference is all that matters
```

**Bad candidates (should use Type 2 instead):**

```c
-- Customer segment (affects revenue analysis)
customer_tier VARCHAR(20)  -- Need history for "revenue by tier"

-- Geographic location (affects regional reporting)
country VARCHAR(100)  -- Need history for "sales by country"

-- Product pricing (affects margin analysis)
unit_price DECIMAL(10,2)  -- Need history for "what was price when sold"
```

## Pros and Cons

**Pros:**

- ✅ Simple to implement
- ✅ No storage overhead
- ✅ Always current data
- ✅ No duplicate rows
- ✅ Easy to understand

**Cons:**

- ❌ Loses all history
- ❌ Can’t do point-in-time analysis
- ❌ Historical reports change retroactively
- ❌ No audit trail
- ❌ Can’t answer “what was value on date X?”

## SCD Type 2: Add New Row

**What it is:** Keep full history by adding a new row for each change. Most common approach.

**Philosophy:** “Track everything. History matters.”

## Implementation

```c
CREATE TABLE dim_customers (
    customer_key INT PRIMARY KEY,        -- Surrogate key (unique per version)
    customer_id VARCHAR(50),              -- Business key (same across versions)
    name VARCHAR(200),
    email VARCHAR(200),
    address VARCHAR(500),
    city VARCHAR(100),
    tier VARCHAR(20),
    
    -- Type 2 tracking columns
    effective_date DATE,                  -- When this version became active
    expiration_date DATE,                 -- When this version expired (NULL = current)
    is_current BOOLEAN,                   -- Flag for current row
    
    -- Optional audit columns
    created_timestamp TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    created_by VARCHAR(100)
);

-- Initial insert (Jan 1, 2024)
INSERT INTO dim_customers VALUES 
(1, 'C123', 'John Doe', 'john@email.com', '123 Main St', 'Boston', 'Silver',
 '2024-01-01', NULL, TRUE, CURRENT_TIMESTAMP, 'etl_process');

-- Update (Mar 15, 2024) - Don't UPDATE, INSERT new row
-- Step 1: Expire old row
UPDATE dim_customers
SET expiration_date = '2024-03-14',
    is_current = FALSE
WHERE customer_id = 'C123' 
  AND is_current = TRUE;

-- Step 2: Insert new row
INSERT INTO dim_customers VALUES 
(2, 'C123', 'John Doe', 'john@email.com', '456 Oak Ave', 'Seattle', 'Gold',
 '2024-03-15', NULL, TRUE, CURRENT_TIMESTAMP, 'etl_process');

-- Result: Two rows for same customer
-- Row 1: customer_key=1, tier='Silver', effective_date='2024-01-01', expiration_date='2024-03-14', is_current=FALSE
-- Row 2: customer_key=2, tier='Gold', effective_date='2024-03-15', expiration_date=NULL, is_current=TRUE
```

## Querying Type 2 Dimensions

**Get current state:**

```c
SELECT * FROM dim_customers
WHERE customer_id = 'C123'
  AND is_current = TRUE;

-- Returns: customer_key=2, tier='Gold', current address
```

**Get historical state (point-in-time):**

```c
-- What was customer's tier on Feb 1, 2024?
SELECT * FROM dim_customers
WHERE customer_id = 'C123'
  AND effective_date <= '2024-02-01'
  AND (expiration_date > '2024-02-01' OR expiration_date IS NULL);

-- Returns: customer_key=1, tier='Silver', old address
```

**Join with facts (preserves history):**

```c
-- Revenue by tier in Q1 2024
SELECT 
    c.tier,
    SUM(f.revenue) as total_revenue
FROM fact_sales f
JOIN dim_customers c ON f.customer_key = c.customer_key  -- Uses surrogate key!
WHERE f.date_key BETWEEN 20240101 AND 20240331
GROUP BY c.tier;

-- Fact table contains customer_key (1 or 2)
-- January sales: linked to customer_key=1 (tier='Silver')
-- March sales: linked to customer_key=2 (tier='Gold')
-- Correctly shows revenue split between Silver and Gold
```

## ETL Pattern for Type 2

```c
def process_customer_update(customer_id, new_data):
    # Get current row
    current = db.query("""
        SELECT * FROM dim_customers 
        WHERE customer_id = ? AND is_current = TRUE
    """, customer_id)
    
    # Check if anything changed
    if current.matches(new_data):
        return  # No changes, skip
    
    # Check which fields changed
    type_2_fields = ['address', 'city', 'tier']
    if any(current[f] != new_data[f] for f in type_2_fields):
        # Type 2 change detected
        
        # Expire current row
        db.execute("""
            UPDATE dim_customers
            SET expiration_date = CURRENT_DATE - 1,
                is_current = FALSE
            WHERE customer_id = ? AND is_current = TRUE
        """, customer_id)
        
        # Insert new row
        next_key = get_next_surrogate_key()
        db.execute("""
            INSERT INTO dim_customers 
            VALUES (?, ?, ..., CURRENT_DATE, NULL, TRUE)
        """, next_key, customer_id, new_data)
        
        # Return new key for fact table updates
        return next_key
```

## Storage Impact

**Example: 5 million customers, 2 changes per customer over 3 years**

```c
Type 1 (overwrite):
- 5M rows
- 500 bytes per row
- Total: 2.5 GB

Type 2 (add new row):
- 5M × 3 versions = 15M rows
- 550 bytes per row (extra date columns)
- Total: 8.25 GB (3.3x larger)

Cost implication:
- Storage: +$0.05/GB/month × 5.75 GB = +$0.29/month
- Query overhead: Must filter on is_current or join on effective dates
```

## Pros and Cons

**Pros:**

- ✅ Complete history preserved
- ✅ Point-in-time analysis possible
- ✅ Audit trail built-in
- ✅ Historical reports stay consistent
- ✅ Can answer “what was X on date Y?”

**Cons:**

- ❌ 2–5x storage overhead
- ❌ Dimension tables grow continuously
- ❌ Queries must filter on is\_current
- ❌ Join complexity increases
- ❌ Surrogate key management required

## SCD Type 3: Add New Column

**What it is:** Keep limited history by adding columns for previous values.

**Philosophy:** “Track last change only. One level of history is enough.”

## Implementation

```c
CREATE TABLE dim_customers (
    customer_key INT PRIMARY KEY,
    customer_id VARCHAR(50),
    name VARCHAR(200),
    email VARCHAR(200),
    
    -- Current values
    address VARCHAR(500),
    city VARCHAR(100),
    tier VARCHAR(20),
    
    -- Previous values (Type 3)
    previous_address VARCHAR(500),
    previous_city VARCHAR(100),
    previous_tier VARCHAR(20),
    
    -- Change tracking
    tier_change_date DATE
);

-- Initial state (Jan 1)
INSERT INTO dim_customers VALUES 
(1, 'C123', 'John Doe', 'john@email.com',
 '123 Main St', 'Boston', 'Silver',
 NULL, NULL, NULL, NULL);

-- Update (Mar 15) - Shift values
UPDATE dim_customers
SET previous_address = address,     -- Save old value
    previous_city = city,
    previous_tier = tier,
    address = '456 Oak Ave',        -- Update to new value
    city = 'Seattle',
    tier = 'Gold',
    tier_change_date = '2024-03-15'
WHERE customer_id = 'C123';

-- Result: Single row with current + previous values
customer_key: 1
tier: 'Gold'           (current)
previous_tier: 'Silver' (what it was before)
tier_change_date: '2024-03-15'
```

## Querying Type 3

**Current state:**

```c
SELECT customer_id, tier
FROM dim_customers
WHERE customer_id = 'C123';
-- Returns: tier='Gold'
```

**Previous state:**

```c
SELECT customer_id, previous_tier
FROM dim_customers
WHERE customer_id = 'C123';
-- Returns: previous_tier='Silver'
```

**Customers who changed tier:**

```c
SELECT 
    customer_id,
    previous_tier,
    tier as current_tier,
    tier_change_date
FROM dim_customers
WHERE tier != previous_tier
  AND previous_tier IS NOT NULL;
```

**Comparing before/after:**

```c
-- Revenue from customers who upgraded
SELECT 
    SUM(CASE WHEN f.date_key < 20240315 THEN f.revenue ELSE 0 END) as revenue_before,
    SUM(CASE WHEN f.date_key >= 20240315 THEN f.revenue ELSE 0 END) as revenue_after
FROM fact_sales f
JOIN dim_customers c ON f.customer_key = c.customer_key
WHERE c.tier = 'Gold' 
  AND c.previous_tier = 'Silver';
```

## When to Use Type 3

**Good for:**

- Customer tier changes (current vs previous)
- Phone number changes (keep old for 90 days)
- Email changes (keep old for contact)
- Department transfers (current vs previous department)

**Real-world example:**

```c
-- Customer support use case
CREATE TABLE dim_customers (
    customer_key INT,
    customer_id VARCHAR(50),
    name VARCHAR(200),
    
    -- Current contact (Type 1)
    current_email VARCHAR(200),
    current_phone VARCHAR(20),
    
    -- Previous contact (Type 3 - keep for 90 days)
    previous_email VARCHAR(200),
    previous_phone VARCHAR(20),
    contact_changed_date DATE
);

-- Can reach customer at current OR previous contact info
-- Useful during transition period
```

## Pros and Cons

**Pros:**

- ✅ Simple to implement
- ✅ Minimal storage overhead (just extra columns)
- ✅ No duplicate rows
- ✅ Can compare current vs previous
- ✅ Good for “before/after” analysis

**Cons:**

- ❌ Only one level of history
- ❌ Loses older changes (3rd, 4th change overwrites)
- ❌ Can’t do point-in-time analysis beyond last change
- ❌ Need to add columns for each tracked attribute
- ❌ Gets messy with multiple changing attributes

## SCD Type 4: Add History Table

**What it is:** Keep current data in main dimension, move history to separate table.

**Philosophy:** “Fast queries on current data, preserve full history separately.”

## Implementation

```c
-- Main dimension (current state only)
CREATE TABLE dim_customers (
    customer_key INT PRIMARY KEY,
    customer_id VARCHAR(50) UNIQUE,
    name VARCHAR(200),
    email VARCHAR(200),
    address VARCHAR(500),
    city VARCHAR(100),
    tier VARCHAR(20),
    last_updated TIMESTAMP
);

-- History table (all changes)
CREATE TABLE dim_customers_history (
    history_key BIGINT PRIMARY KEY,
    customer_key INT,              -- FK to current dimension
    customer_id VARCHAR(50),
    name VARCHAR(200),
    email VARCHAR(200),
    address VARCHAR(500),
    city VARCHAR(100),
    tier VARCHAR(20),
    effective_date DATE,
    expiration_date DATE,
    change_type VARCHAR(20)        -- 'INSERT', 'UPDATE', 'DELETE'
);

-- Current state (Jan 1)
INSERT INTO dim_customers VALUES 
(1, 'C123', 'John Doe', 'john@email.com', '123 Main St', 'Boston', 'Silver', CURRENT_TIMESTAMP);

-- Also insert to history
INSERT INTO dim_customers_history VALUES
(1, 1, 'C123', 'John Doe', 'john@email.com', '123 Main St', 'Boston', 'Silver',
 '2024-01-01', NULL, 'INSERT');

-- Update (Mar 15)
-- Step 1: Close out history record
UPDATE dim_customers_history
SET expiration_date = '2024-03-14'
WHERE customer_id = 'C123' 
  AND expiration_date IS NULL;

-- Step 2: Insert new history record
INSERT INTO dim_customers_history VALUES
(2, 1, 'C123', 'John Doe', 'john@email.com', '456 Oak Ave', 'Seattle', 'Gold',
 '2024-03-15', NULL, 'UPDATE');

-- Step 3: Update current dimension
UPDATE dim_customers
SET address = '456 Oak Ave',
    city = 'Seattle',
    tier = 'Gold',
    last_updated = CURRENT_TIMESTAMP
WHERE customer_id = 'C123';

-- Result:
-- dim_customers: 1 row (current state)
-- dim_customers_history: 2 rows (full history)
```

## Querying Type 4

**Current state (fast):**

```c
-- Query current dimension (no filtering needed)
SELECT * FROM dim_customers
WHERE customer_id = 'C123';
-- Very fast, no is_current filter, no duplicate rows
```

**Historical state:**

```c
-- Query history table for point-in-time
SELECT * FROM dim_customers_history
WHERE customer_id = 'C123'
  AND effective_date <= '2024-02-01'
  AND (expiration_date > '2024-02-01' OR expiration_date IS NULL);
```

**Join facts with history (for historical accuracy):**

```c
-- Revenue by tier, historically accurate
SELECT 
    h.tier,
    SUM(f.revenue) as total_revenue
FROM fact_sales f
JOIN dim_customers_history h 
    ON f.customer_key = h.customer_key
    AND f.date_key BETWEEN h.effective_date AND COALESCE(h.expiration_date, '9999-12-31')
WHERE f.date_key BETWEEN 20240101 AND 20240331
GROUP BY h.tier;
```

**Join facts with current (for speed):**

```c
-- Recent sales, current customer info (much faster)
SELECT 
    c.name,
    c.tier,
    SUM(f.revenue) as total_revenue
FROM fact_sales f
JOIN dim_customers c ON f.customer_key = c.customer_key
WHERE f.date_key >= 20240301  -- Last month only
GROUP BY c.name, c.tier;
-- Uses small current table, not large history table
```

## Performance Comparison

**Query: Get current customer tier (10M customers)**

```c
Type 2 (add new row):
- Scan: 25M rows (includes all history)
- Filter: WHERE is_current = TRUE
- Time: 3.2 seconds

Type 4 (history table):
- Scan: 10M rows (current only)
- No filter needed
- Time: 0.8 seconds (4x faster)
```

**Query: Get tier on specific date**

```c
Type 2:
- Scan: 25M rows
- Filter: WHERE effective_date <= '2024-02-01' AND expiration_date > '2024-02-01'
- Time: 4.1 seconds

Type 4:
- Scan: 25M rows (history table)
- Filter: Same as Type 2
- Time: 4.3 seconds (similar)
```

## Pros and Cons

**Pros:**

- ✅ Fast queries on current data (main dimension is small)
- ✅ Complete history preserved (history table)
- ✅ Clear separation: current vs historical queries
- ✅ Can optimize each table differently
- ✅ Easier to partition history table

**Cons:**

- ❌ More complex ETL (update two tables)
- ❌ Queries need to know which table to use
- ❌ Historical queries still slow (join to history)
- ❌ Data duplicated (current exists in both tables)
- ❌ Synchronization risk between tables

## SCD Type 5: Mini-Dimension

**What it is:** Extract rapidly changing attributes into a separate mini-dimension.**Philosophy:** “Some attributes change fast, some change slow. Separate them.”

## Implementation

**Problem scenario:**

```c
-- Customer dimension with mixed change frequencies
CREATE TABLE dim_customers (
    customer_key INT,
    customer_id VARCHAR(50),
    name VARCHAR(200),           -- Rarely changes
    birth_date DATE,             -- Never changes
    address VARCHAR(500),        -- Changes occasionally (once per year)
    
    -- These change FREQUENTLY (weekly/monthly)
    credit_score INT,            -- Changes every month
    account_balance DECIMAL(15,2), -- Changes daily
    last_login TIMESTAMP         -- Changes constantly
);

-- Problem: Every balance update creates new Type 2 row
-- 10M customers × 365 balance changes/year = 3.65 BILLION rows per year!
```

**Solution: Extract to mini-dimension**

```c
-- Main dimension (slow-changing attributes)
CREATE TABLE dim_customers (
    customer_key INT PRIMARY KEY,
    customer_id VARCHAR(50),
    name VARCHAR(200),
    birth_date DATE,
    address VARCHAR(500),
    city VARCHAR(100),
    effective_date DATE,
    expiration_date DATE,
    is_current BOOLEAN
);

-- Mini-dimension (fast-changing attributes)
CREATE TABLE dim_customer_metrics (
    metrics_key INT PRIMARY KEY,      -- Surrogate key
    credit_score_band VARCHAR(20),    -- 'Excellent', 'Good', 'Fair', 'Poor'
    balance_band VARCHAR(20),         -- '<$1K', '$1K-$10K', '>$10K'
    activity_level VARCHAR(20)        -- 'High', 'Medium', 'Low'
);

-- Pre-populate combinations (small, controlled)
INSERT INTO dim_customer_metrics VALUES
(1, 'Excellent', '<$1K', 'High'),
(2, 'Excellent', '<$1K', 'Medium'),
(3, 'Excellent', '<$1K', 'Low'),
-- ... etc (maybe 100-200 rows total)

-- Fact table references BOTH dimensions
CREATE TABLE fact_transactions (
    transaction_key BIGINT,
    date_key INT,
    customer_key INT,          -- FK to dim_customers
    metrics_key INT,           -- FK to dim_customer_metrics (at time of transaction)
    amount DECIMAL(10,2)
);

-- When transaction happens:
-- 1. Look up current customer_key (slow-changing)
-- 2. Look up current metrics_key (based on current credit score, balance, activity)
-- 3. Record both in fact table
```

## Querying Mini-Dimensions

**Current customer profile:**

```c
-- Get current slow-changing attributes
SELECT * FROM dim_customers
WHERE customer_id = 'C123' AND is_current = TRUE;

-- Current metrics can be calculated from source or cached separately
```

**Historical analysis:**

```c
-- Revenue by credit score band and activity level
SELECT 
    m.credit_score_band,
    m.activity_level,
    COUNT(DISTINCT f.customer_key) as unique_customers,
    SUM(f.amount) as total_revenue
FROM fact_transactions f
JOIN dim_customer_metrics m ON f.metrics_key = m.metrics_key
WHERE f.date_key BETWEEN 20240101 AND 20240331
GROUP BY m.credit_score_band, m.activity_level;
-- Fast! Mini-dimension is tiny, no huge Type 2 dimension to scan
```

## Real-World Example: E-commerce

```c
-- Customer demographics (change slowly)
CREATE TABLE dim_customers (
    customer_key INT,
    customer_id VARCHAR(50),
    name VARCHAR(200),
    age_band VARCHAR(20),        -- Changes yearly
    location VARCHAR(100),       -- Changes rarely
    effective_date DATE
);

-- Customer behavior (changes constantly)
CREATE TABLE dim_customer_behavior (
    behavior_key INT PRIMARY KEY,
    recency VARCHAR(20),         -- 'Active', 'Lapsed', 'Inactive'
    frequency VARCHAR(20),       -- 'Heavy', 'Medium', 'Light'
    monetary VARCHAR(20)         -- 'High Value', 'Medium', 'Low'
);

-- Fact table
CREATE TABLE fact_orders (
    order_key BIGINT,
    date_key INT,
    customer_key INT,         -- Who they are (slowly changing)
    behavior_key INT,         -- How they behave (rapidly changing)
    order_amount DECIMAL(10,2)
);

-- Analysis: "High value but lapsed customers"
SELECT 
    c.location,
    COUNT(*) as customer_count,
    SUM(f.order_amount) as revenue
FROM fact_orders f
JOIN dim_customers c ON f.customer_key = c.customer_key
JOIN dim_customer_behavior b ON f.behavior_key = b.behavior_key
WHERE b.monetary = 'High Value'
  AND b.recency = 'Lapsed'
GROUP BY c.location;
```

## Pros and Cons

**Pros:**

- ✅ Prevents dimension explosion from frequent changes
- ✅ Fast queries (mini-dimension is small)
- ✅ Captures behavior patterns at transaction time
- ✅ Better than storing raw values in fact

**Cons:**

- ❌ Requires careful attribute selection (what goes where?)
- ❌ Loses precision (bands instead of exact values)
- ❌ More complex ETL logic
- ❌ Need to maintain mini-dimension combinations

## SCD Type 6: Hybrid (1+2+3)

**What it is:** Combine Type 1, 2, and 3. Add rows (Type 2), keep previous value (Type 3), and update current values (Type 1).

**Philosophy:** “I want everything! Current values always accessible, full history tracked, and last value quick to reference.”

## Implementation

```c
CREATE TABLE dim_customers (
    customer_key INT PRIMARY KEY,        -- Type 2: Unique per version
    customer_id VARCHAR(50),              -- Business key
    
    -- Type 2: Historical values (this version)
    name VARCHAR(200),
    address VARCHAR(500),
    city VARCHAR(100),
    tier VARCHAR(20),
    
    -- Type 3: Previous values (one level back)
    previous_tier VARCHAR(20),
    tier_change_date DATE,
    
    -- Type 1: Current values (always up-to-date)
    current_tier VARCHAR(20),
    current_address VARCHAR(500),
    current_city VARCHAR(100),
    
    -- Type 2 tracking
    effective_date DATE,
    expiration_date DATE,
    is_current BOOLEAN
);

-- Initial insert (Jan 1)
INSERT INTO dim_customers VALUES 
(1, 'C123', 'John Doe', '123 Main St', 'Boston', 'Silver',
 NULL, NULL,                              -- previous values (none yet)
 'Silver', '123 Main St', 'Boston',       -- current values
 '2024-01-01', NULL, TRUE);

-- First update (Mar 15)
-- Step 1: Update ALL rows (Type 1 behavior for current_ columns)
UPDATE dim_customers
SET previous_tier = current_tier,
    current_tier = 'Gold',
    current_address = '456 Oak Ave',
    current_city = 'Seattle'
WHERE customer_id = 'C123';

-- Step 2: Expire old row (Type 2 behavior)
UPDATE dim_customers
SET expiration_date = '2024-03-14',
    is_current = FALSE
WHERE customer_id = 'C123' 
  AND is_current = TRUE;

-- Step 3: Insert new row (Type 2 behavior)
INSERT INTO dim_customers VALUES 
(2, 'C123', 'John Doe', '456 Oak Ave', 'Seattle', 'Gold',
 'Silver', '2024-03-15',                  -- previous value
 'Gold', '456 Oak Ave', 'Seattle',        -- current value
 '2024-03-15', NULL, TRUE);

-- Result: Two rows for same customer
-- Old row: customer_key=1, tier='Silver', current_tier='Gold' (updated!)
-- New row: customer_key=2, tier='Gold', current_tier='Gold', previous_tier='Silver'
```

## Querying Type 6

**Current state (Type 1 approach):**

```c
-- Simple! Just use current_ columns from any row
SELECT current_tier, current_address, current_city
FROM dim_customers
WHERE customer_id = 'C123'
  AND is_current = TRUE;

-- OR even simpler (current_ columns are same on all rows)
SELECT current_tier, current_address, current_city
FROM dim_customers
WHERE customer_id = 'C123'
LIMIT 1;  -- Any row works
```

**Historical state (Type 2 approach):**

```c
-- What was tier on Feb 1?
SELECT tier
FROM dim_customers
WHERE customer_id = 'C123'
  AND effective_date <= '2024-02-01'
  AND (expiration_date > '2024-02-01' OR expiration_date IS NULL);
-- Returns: 'Silver'
```

**Previous state (Type 3 approach):**

```c
-- What was tier before current?
SELECT previous_tier, tier_change_date
FROM dim_customers
WHERE customer_id = 'C123'
  AND is_current = TRUE;
-- Returns: 'Silver', '2024-03-15'
```

**Use case: “Show current status with change history”**

```c
SELECT 
    c.customer_id,
    c.name,
    c.current_tier as tier_now,
    c.previous_tier as tier_before,
    c.tier_change_date,
    DATEDIFF(day, c.tier_change_date, CURRENT_DATE) as days_since_change
FROM dim_customers c
WHERE c.is_current = TRUE
  AND c.tier_change_date IS NOT NULL
ORDER BY c.tier_change_date DESC;
-- Shows recent tier changes with current context
```

## When to Use Type 6

**Perfect for:**

- Dashboard showing current state with change indicators
- Customer service needing current contact but seeing history
- Reports comparing current period vs previous period
- Audit requirements plus operational reporting

**Real example: Customer 360 view**

```c
SELECT 
    c.customer_id,
    c.name,
    c.current_tier,                    -- Current status (Type 1)
    c.current_address,
    c.previous_tier,                   -- Recent change (Type 3)
    c.tier_change_date,
    COUNT(h.customer_key) as version_count  -- Full history count (Type 2)
FROM dim_customers c
LEFT JOIN dim_customers h ON c.customer_id = h.customer_id
WHERE c.is_current = TRUE
GROUP BY c.customer_id, c.name, c.current_tier, c.current_address, 
         c.previous_tier, c.tier_change_date;
```

## Pros and Cons

**Pros:**

- ✅ Maximum flexibility
- ✅ Current values always accessible (no filtering)
- ✅ Full history preserved
- ✅ Recent changes easily queryable
- ✅ Supports multiple query patterns

**Cons:**

- ❌ Most storage overhead
- ❌ Most complex ETL (update all rows, then insert)
- ❌ Data redundancy (current\_ columns repeated)
- ❌ Confusion about which columns to use
- ❌ Update cost on every dimension change

## SCD Type 7: Hybrid with Date Dimension

**What it is:** Combine Type 2 with a date dimension to get both surrogate keys and date-based keys.

**Philosophy:** “Let me join on dates OR keys, whichever is more convenient.”

## Implementation

```c
-- Dimension with both key types
CREATE TABLE dim_customers (
    customer_key INT PRIMARY KEY,        -- Surrogate key
    customer_id VARCHAR(50),              -- Business key
    current_key INT,                      -- FK to current version
    
    name VARCHAR(200),
    address VARCHAR(500),
    city VARCHAR(100),
    tier VARCHAR(20),
    
    effective_date DATE,
    expiration_date DATE,
    is_current BOOLEAN
);

-- Fact table can reference either way
CREATE TABLE fact_sales (
    sale_key BIGINT,
    date_key INT,
    
    -- Option 1: Use customer_key (like Type 2)
    customer_key INT,         -- FK to specific version
    
    -- Option 2: Use customer_id + date (for lookups)
    customer_id VARCHAR(50),  -- Business key
    
    amount DECIMAL(10,2)
);

-- Initial insert
INSERT INTO dim_customers VALUES 
(1, 'C123', 1,  -- current_key points to self
 'John Doe', '123 Main St', 'Boston', 'Silver',
 '2024-01-01', NULL, TRUE);

-- Update
UPDATE dim_customers
SET expiration_date = '2024-03-14',
    is_current = FALSE
WHERE customer_key = 1;

INSERT INTO dim_customers VALUES 
(2, 'C123', 2,  -- New current_key
 'John Doe', '456 Oak Ave', 'Seattle', 'Gold',
 '2024-03-15', NULL, TRUE);

-- Update ALL rows to point to new current
UPDATE dim_customers
SET current_key = 2
WHERE customer_id = 'C123';
```

## Querying Type 7

**Join by surrogate key (Type 2 style):**

```c
-- Historically accurate
SELECT 
    c.tier,
    SUM(f.amount) as revenue
FROM fact_sales f
JOIN dim_customers c ON f.customer_key = c.customer_key
GROUP BY c.tier;
-- Each sale linked to tier at time of sale
```

**Join by date (date-based lookup):**

```c
-- Use effective/expiration dates
SELECT 
    c.tier,
    SUM(f.amount) as revenue
FROM fact_sales f
JOIN dim_customers c 
    ON f.customer_id = c.customer_id
    AND f.date_key BETWEEN c.effective_date AND COALESCE(c.expiration_date, '9999-12-31')
GROUP BY c.tier;
-- Same result, different join method
```

**Get current version easily:**

```c
-- Use current_key
SELECT c.*
FROM dim_customers c
WHERE c.customer_key = c.current_key;

-- OR
SELECT * FROM dim_customers
WHERE is_current = TRUE;
```

## Pros and Cons

**Pros:**

- ✅ Flexible joins (key-based or date-based)
- ✅ Can reconstruct relationships if keys lost
- ✅ Easy to get current version
- ✅ Supports multiple integration patterns

**Cons:**

- ❌ Most complex implementation
- ❌ Storage overhead (extra columns)
- ❌ Update complexity (maintain current\_key)
- ❌ Potential confusion about join method
- ❌ Rarely needed in practice

## Choosing the Right SCD Type

## Decision Matrix

Requirement Recommended Type No history needed Type 1 Corrections only Type 1 Complete history required Type 2 Compliance/audit trail Type 2 or Type 4 Current queries > historical Type 4 One level of history enough Type 3 Before/after comparison Type 3 Rapidly changing attributes Type 5 (mini-dimension) Complex reporting needs Type 6 Never change (immutable) Type 0

## By Industry

**Retail/E-commerce:**

- Customer tier: Type 2 (need history for segmentation analysis)
- Email/phone: Type 1 (only need current)
- Address: Type 2 (shipping history matters)
- Product price: Type 2 (need historical price at time of sale)

**Finance/Banking:**

- Account balance: Type 5 (mini-dimension for bands, changes daily)
- Customer segment: Type 2 (regulatory reporting requires history)
- Credit score: Type 5 or Type 2
- Address: Type 2 (Know Your Customer requirements)
- Account status: Type 2 (audit trail required)

**Healthcare:**

- Patient address: Type 2 (billing history)
- Insurance provider: Type 2 (claims tied to provider at time)
- Diagnosis codes: Type 0 (historical fact, never changes)
- Contact info: Type 1 (current only matters)

**SaaS:**

- Subscription tier: Type 2 (revenue recognition needs history)
- Feature flags: Type 1 (current state only)
- User profile: Mix (email Type 1, preferences Type 1, subscription Type 2)

## Common Mistakes

## Mistake #1: Using Type 2 for Everything

**Problem:**

```c
-- Making EVERYTHING Type 2
CREATE TABLE dim_customers (
    customer_key INT,
    customer_id VARCHAR(50),
    name VARCHAR(200),              -- Type 2 (rarely changes)
    email VARCHAR(200),             -- Type 2 (changes occasionally)
    phone VARCHAR(20),              -- Type 2 (changes occasionally)
    favorite_color VARCHAR(50),     -- Type 2 (WHO CARES?!)
    marketing_opt_in BOOLEAN,       -- Type 2 (current state only matters)
    last_login TIMESTAMP,           -- Type 2 (changes constantly!)
    account_balance DECIMAL(15,2),  -- Type 2 (updates daily!)
    effective_date DATE
);

-- Result: Dimension explodes
-- 1M customers, each changes 10 attributes 5 times = 50M rows
-- Queries slow, storage huge, maintenance nightmare
```

**Solution:** Mix types appropriately

```c
CREATE TABLE dim_customers (
    customer_key INT,
    customer_id VARCHAR(50),
    name VARCHAR(200),              -- Type 2 (changes, needs history)
    email VARCHAR(200),             -- Type 1 (current only)
    phone VARCHAR(20),              -- Type 1 (current only)
    -- Don't store: favorite_color (not analytical)
    marketing_opt_in BOOLEAN,       -- Type 1 (current only)
    -- Don't store: last_login (put in fact table or cache)
    -- account_balance → Type 5 mini-dimension
    effective_date DATE
);
```

## Mistake #2: Wrong Surrogate Key in Facts

**Problem:**

```c
-- Fact table stores business key, not surrogate key
CREATE TABLE fact_sales (
    sale_key BIGINT,
    customer_id VARCHAR(50),  -- Business key (WRONG!)
    product_id VARCHAR(50),   -- Business key (WRONG!)
    sale_amount DECIMAL(10,2)
);

-- Dimension is Type 2
CREATE TABLE dim_customers (
    customer_key INT,
    customer_id VARCHAR(50),
    tier VARCHAR(20),
    effective_date DATE,
    is_current BOOLEAN
);

-- Query tries to join
SELECT c.tier, SUM(f.sale_amount)
FROM fact_sales f
JOIN dim_customers c ON f.customer_id = c.customer_id
-- Which version of customer? All of them!
-- Returns multiple rows per sale if customer has multiple versions
-- Revenue double-counted!
```

**Solution:** Store surrogate keys

```c
CREATE TABLE fact_sales (
    sale_key BIGINT,
    customer_key INT,         -- Surrogate key (CORRECT!)
    product_key INT,          -- Surrogate key (CORRECT!)
    sale_amount DECIMAL(10,2)
);

-- ETL logic
def insert_sale(customer_id, product_id, amount, sale_date):
    # Look up customer_key that was valid on sale_date
    customer_key = db.query("""
        SELECT customer_key 
        FROM dim_customers 
        WHERE customer_id = ?
          AND effective_date <= ?
          AND (expiration_date > ? OR expiration_date IS NULL)
    """, customer_id, sale_date, sale_date)
    
    # Insert with surrogate key
    db.execute("""
        INSERT INTO fact_sales VALUES (?, ?, ?, ?)
    """, next_key, customer_key, product_key, amount)
```

## Mistake #3: Not Handling Late-Arriving Dimensions

**Problem:**

```c
-- Fact arrives first (Jan 15)
-- Customer dimension arrives later (Jan 20)

-- Fact load fails:
INSERT INTO fact_sales VALUES (1, 12345, 99.99, '2024-01-15');
-- Error: customer_key 12345 doesn't exist yet!

-- Or worse: uses wrong key
-- Uses customer_key from different customer
-- Data corruption!
```

**Solution:** Use late-arriving dimension pattern

```c
-- Create "unknown" placeholder row
INSERT INTO dim_customers VALUES (
    -1, 'UNKNOWN', 'Unknown Customer', NULL, NULL, NULL,
    '1900-01-01', NULL, TRUE
);

-- Fact load uses placeholder
INSERT INTO fact_sales VALUES (1, -1, 99.99, '2024-01-15');

-- When dimension arrives, update fact
UPDATE fact_sales
SET customer_key = 12345
WHERE customer_key = -1
  AND sale_date = '2024-01-15'
  AND /* other identifying criteria */;
```

## Performance Implications

**Query performance by type (10M customer dimension):**

```c
Query: Get current customer tier

Type 1:
- Rows: 10M
- Scan: Full table
- Filter: WHERE customer_id = 'C123'
- Time: 0.5s

Type 2:
- Rows: 30M (3 versions each)
- Scan: Full table
- Filter: WHERE customer_id = 'C123' AND is_current = TRUE
- Time: 1.8s (3.6x slower)

Type 4:
- Rows: 10M (current table)
- Scan: Current table only
- Filter: WHERE customer_id = 'C123'
- Time: 0.5s (same as Type 1)

Type 6:
- Rows: 30M
- Scan: Full table
- No filter needed (current_ columns)
- Time: 0.6s
```

**Storage comparison (10M customers, 3 changes each):**

```c
Type 1: 10M rows × 300 bytes = 3 GB
Type 2: 40M rows × 350 bytes = 14 GB (4.7x)
Type 3: 10M rows × 450 bytes = 4.5 GB (1.5x)
Type 4: 10M + 40M rows = 17.5 GB (5.8x)
Type 5: 10M + 200 rows mini = 3 GB (1x)
Type 6: 40M rows × 500 bytes = 20 GB (6.7x)
```

## Recommendations

**Start simple:**

1. Begin with Type 1 for everything
2. Add Type 2 for attributes that need history
3. Consider Type 4 if Type 2 dimensions get huge (>50M rows)
4. Use Type 5 for rapidly changing attributes (>1 change/day)
5. Avoid Type 6 unless you have a clear requirement

**Production best practices:**

```c
-- Always include these columns for any SCD type
effective_date DATE         -- When this version started
expiration_date DATE        -- When this version ended (NULL = current)
is_current BOOLEAN         -- Quick filter for current state
created_timestamp TIMESTAMP -- Audit: when row was created
created_by VARCHAR(100)    -- Audit: which ETL process created it
hash_diff VARCHAR(64)      -- Detect if attributes actually changed
```

**ETL framework:**

```c
def process_dimension_update(entity_id, new_data, scd_config):
    # scd_config defines which fields use which type
    # Example: {'email': 'type1', 'tier': 'type2', 'name': 'type2'}
    
    current = get_current_row(entity_id)
    
    type1_changed = False
    type2_changed = False
    
    for field, value in new_data.items():
        if current[field] == value:
            continue  # No change
            
        if scd_config[field] == 'type1':
            type1_changed = True
        elif scd_config[field] == 'type2':
            type2_changed = True
    
    if type1_changed and not type2_changed:
        # Just update existing row
        update_row(entity_id, new_data)
    
    elif type2_changed:
        # Expire old row, insert new row
        expire_current_row(entity_id)
        insert_new_row(entity_id, new_data)
    
    # else: no changes, skip
```

**Disclaimer:** The appropriate SCD type depends on your specific business requirements, technical constraints, and team capabilities. Start with the simplest approach that meets your needs, and add complexity only when justified by clear business requirements. Not every attribute needs history tracking, and not every history needs to be permanent. Focus on what your stakeholders actually need to analyze.

[![Reliable Data Engineering](https://miro.medium.com/v2/resize:fill:96:96/1*ewisWhJkTid55OnnFA0EmA.png)](https://medium.com/@reliabledataengineering?source=post_page---post_author_info--f9fe8348eb54---------------------------------------)[197 following](https://medium.com/@reliabledataengineering/following?source=post_page---post_author_info--f9fe8348eb54---------------------------------------)

[https://reliable-data-engineering.netlify.app/](https://reliable-data-engineering.netlify.app/)

## Responses (2)

Isaac Lee

What are your thoughts?

```c
Very detailed explanations.In my projects I additionally introduced SCD Type 0.5If column has this type it means that I do not want to trigger changes every time when the column is updated but I want to store the latest value for this column if other columns (with SCD >= 1 ) are updated.
```

2

```c
Well written article with detailed explanations!
```

5
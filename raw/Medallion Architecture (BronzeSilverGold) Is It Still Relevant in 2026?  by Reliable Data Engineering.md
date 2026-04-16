---
title: "Medallion Architecture (Bronze/Silver/Gold): Is It Still Relevant in 2026? | by Reliable Data Engineering"
source: "https://freedium-mirror.cfd/https://medium.com/@reliabledataengineering/medallion-architecture-bronze-silver-gold-is-it-still-relevant-in-2026-5e616fc03245"
author:
published:
created: 2026-04-16
description: "An analysis of the bronze/silver/gold pattern, its alternatives, and current adoption patterns in..."
tags:
  - "clippings"
---
[< Go to the original](https://medium.com/@reliabledataengineering/medallion-architecture-bronze-silver-gold-is-it-still-relevant-in-2026-5e616fc03245#bypass)

![Preview image](https://miro.medium.com/v2/resize:fit:700/1*uK3WmwUvH16z1Ssz9scCcw.png)

## Medallion Architecture (Bronze/Silver/Gold): Is It Still Relevant in 2026?

## An analysis of the bronze/silver/gold pattern, its alternatives, and current adoption patterns in modern data platforms

androidstudio · January 5, 2026 (Updated: January 6, 2026) · Free: No

Read for free:

## [Medallion Architecture (Bronze/Silver/Gold): Is It Still Relevant in 2026?](https://medium.com/@reliabledataengineering/medallion-architecture-bronze-silver-gold-is-it-still-relevant-in-2026-5e616fc03245)

### An analysis of the bronze/silver/gold pattern, its alternatives, and current adoption patterns in modern data platforms

medium.com

![None](https://miro.medium.com/v2/resize:fit:700/1*uK3WmwUvH16z1Ssz9scCcw.png)

**Disclaimer:** The adoption patterns and architectural recommendations discussed here reflect observations from data engineering communities, conference talks, and public discussions as of December 2025 - January 2026. Architecture choices are highly context-dependent and should be evaluated based on your specific organizational needs, data volumes, team skills, and existing infrastructure. No single architecture pattern is universally optimal.

### What is Medallion Architecture

Medallion architecture is a data design pattern that organizes data into three layers:

**Bronze Layer (Raw):**

- Data ingested in its original format
- Minimal transformation
- Append-only historical record
- No data quality enforcement

**Silver Layer (Refined):**

- Cleaned and conformed data
- Schema enforced
- Deduplicated
- Validated
- Still fairly granular

**Gold Layer (Curated):**

- Business-level aggregations
- Denormalized for consumption
- Optimized for specific use cases
- Analytics-ready

**Origin:** Popularized by Databricks around 2019-2020 as part of the lakehouse pattern.

### Example Implementation

### Bronze Layer

```python
# Ingest raw data with minimal transformation
from pyspark.sql import functions as F

# Read from source
raw_df = spark.read.format("json") \
    .load("s3://source-bucket/raw-orders/")

# Add ingestion metadata only
bronze_df = raw_df \
    .withColumn("ingestion_timestamp", F.current_timestamp()) \
    .withColumn("source_file", F.input_file_name())

# Write to bronze
bronze_df.write.format("delta") \
    .mode("append") \
    .save("/mnt/bronze/orders")
```

**Characteristics:**

- Preserves original data exactly as received
- Includes ingestion metadata for lineage
- Enables reprocessing if downstream logic changes

### Silver Layer

```python
# Clean and conform data
from pyspark.sql import functions as F
from pyspark.sql.types import *

# Read from bronze
bronze_df = spark.read.format("delta") \
    .load("/mnt/bronze/orders")

# Parse and validate
silver_df = bronze_df \
    .withColumn("order_id", F.col("id").cast(IntegerType())) \
    .withColumn("customer_id", F.col("customer").cast(IntegerType())) \
    .withColumn("order_date", F.to_date(F.col("date"), "yyyy-MM-dd")) \
    .withColumn("amount", F.col("total").cast(DecimalType(10,2))) \
    .filter(F.col("amount") > 0) \
    .filter(F.col("order_date").isNotNull()) \
    .dropDuplicates(["order_id"])

# Write to silver
silver_df.write.format("delta") \
    .mode("append") \
    .option("mergeSchema", "true") \
    .save("/mnt/silver/orders")
```

**Characteristics:**

- Standardized data types
- Data quality rules enforced
- Duplicate removal
- Still relatively granular

### Gold Layer

```bash
# Create business-level aggregations
silver_df = spark.read.format("delta") \
    .load("/mnt/silver/orders")

# Aggregate for specific use case
gold_df = silver_df \
    .groupBy(
        F.date_trunc("month", F.col("order_date")).alias("month"),
        "customer_id"
    ) \
    .agg(
        F.sum("amount").alias("total_spent"),
        F.count("order_id").alias("order_count"),
        F.avg("amount").alias("avg_order_value")
    )

# Write to gold
gold_df.write.format("delta") \
    .mode("overwrite") \
    .save("/mnt/gold/monthly_customer_metrics")
```

**Characteristics:**

- Business-focused aggregations
- Optimized for specific queries
- Often denormalized
- May have multiple gold tables for different use cases

### Current Adoption Patterns

### Databricks Ecosystem

Medallion architecture remains the recommended pattern in Databricks documentation and training materials. The Delta Live Tables product is explicitly designed around this pattern.

**Delta Live Tables example:**

```python
import dbt_expectations as de
from pyspark.sql import functions as F

# Bronze
@dlt.table(
    comment="Raw orders data"
)
def bronze_orders():
    return spark.readStream.format("json") \
        .load("s3://raw/orders")

# Silver
@dlt.table(
    comment="Cleaned and validated orders"
)
@dlt.expect_or_drop("valid_amount", "amount > 0")
@dlt.expect_or_fail("valid_date", "order_date IS NOT NULL")
def silver_orders():
    return dlt.read_stream("bronze_orders") \
        .select(
            F.col("id").cast("int").alias("order_id"),
            F.col("customer").cast("int").alias("customer_id"),
            F.to_date("date").alias("order_date"),
            F.col("total").cast("decimal(10,2)").alias("amount")
        )

# Gold
@dlt.table(
    comment="Monthly customer metrics"
)
def gold_monthly_metrics():
    return dlt.read("silver_orders") \
        .groupBy(
            F.date_trunc("month", "order_date").alias("month"),
            "customer_id"
        ) \
        .agg(
            F.sum("amount").alias("total_spent"),
            F.count("order_id").alias("order_count")
        )
```

### Alternative Patterns Observed

**Two-layer approach (Raw + Curated):**

```cpp
raw/          → Silver + Gold combined
curated/      → (skips explicit bronze)
```

Common in organizations that:

- Use managed ingestion tools (Fivetran, Airbyte)
- Don't need to reprocess from raw frequently
- Want simpler architecture

**Stage-based approach:**

```sql
staging/      → Light transformations (like bronze)
intermediate/ → Business logic transformations
marts/        → Final consumption layer
```

Common in dbt projects that predate medallion pattern.

**Domain-driven approach:**

```typescript
ingestion/
├── domain_a/
├── domain_b/
processing/
├── domain_a/
├── domain_b/
serving/
├── domain_a/
├── domain_b/
```

Common in organizations practicing data mesh principles.

### Advantages of Medallion Architecture

### 1\. Clear Separation of Concerns

Each layer has a distinct purpose:

- Bronze: Ingestion and historical preservation
- Silver: Data quality and standardization
- Gold: Business logic and optimization

**Benefit:** Team members understand which layer to work in for different tasks.

### 2\. Incremental Processing at Each Layer

```ruby
# Each layer can be processed incrementally
@dlt.table
def bronze_orders():
    return spark.readStream.format("json").load("s3://raw/orders")

@dlt.table
def silver_orders():
    return dlt.read_stream("bronze_orders").transform(...)

@dlt.table
def gold_metrics():
    return dlt.read("silver_orders").aggregate(...)
```

**Benefit:** Efficient processing of only changed data.

### 3\. Audit Trail and Reproducibility

Bronze layer preserves original data, enabling:

- Reprocessing if business logic changes
- Debugging data quality issues
- Compliance and audit requirements

**Example scenario:**

```sql
Day 1: Process data with logic v1
Day 30: Discover bug in logic v1
Action: Fix logic, reprocess from bronze with logic v2
Result: Corrected data without re-ingesting from source
```

### 4\. Schema Evolution Management

```
# Bronze: Accept any schema
bronze_df.write.format("delta") \
    .option("mergeSchema", "true") \
    .save("/bronze/orders")

# Silver: Enforce schema
silver_df.write.format("delta") \
    .option("mergeSchema", "false") \
    .save("/silver/orders")
```

**Benefit:** Catch schema changes at the silver layer rather than breaking downstream consumers.

### Disadvantages and Challenges

### 1\. Increased Storage Costs

Storing data in three layers multiplies storage requirements:

**Example calculation:**

- Source data: 100 GB
- Bronze layer: 100 GB (raw copy)
- Silver layer: 80 GB (cleaned, compressed)
- Gold layer: 20 GB (aggregated)
- **Total: 300 GB** (3x source data)

**Storage cost (S3 Standard at $0.023/GB/month):**

- 300 GB × $0.023 = $6.90/month

**Counterpoint:** Storage is relatively cheap compared to compute costs and engineering time.

### 2\. Increased Complexity

More layers mean:

- More ETL jobs to manage
- More monitoring and alerting
- More potential failure points
- Steeper learning curve for new team members

**Example: Pipeline failure complexity**

```
Source → Bronze → Silver → Gold
         ↓         ↓        ↓
      Fails?    Fails?   Fails?

3 potential failure points vs 1 in a single-layer approach
```

### 3\. Latency Addition

Each layer adds processing time:

**Measured latency example:**

- Source to Bronze: 5 minutes
- Bronze to Silver: 10 minutes
- Silver to Gold: 8 minutes
- **Total: 23 minutes** end-to-end

For a direct transformation: 15 minutes

**Trade-off:** Better data quality and flexibility vs faster data availability.

### 4\. Over-Engineering for Simple Use Cases

**Scenario:** Small organization with one data source and simple transformations.

**Medallion approach:**

```bash
bronze/shopify_orders
silver/shopify_orders_cleaned
gold/daily_revenue
```

**Simpler alternative:**

```bash
staging/shopify_orders
marts/daily_revenue
```

**Assessment:** Three layers may be unnecessary complexity for simple pipelines.

### When Medallion Architecture Makes Sense

### Use Case 1: Multiple Downstream Consumers

**Scenario:** Silver layer used by multiple gold tables.

```
Bronze (Orders)
    ↓
Silver (Orders Cleaned)
    ↓
    ├→ Gold (Daily Revenue)
    ├→ Gold (Customer Metrics)
    ├→ Gold (Product Analytics)
    └→ Gold (Inventory Tracking)
```

**Benefit:** Single source of clean data feeding multiple use cases.

### Use Case 2: Complex Data Quality Requirements

**Scenario:** Strict data validation and cleaning needed.

```
@dlt.table
@dlt.expect_or_drop("valid_email", "email RLIKE '^[^@]+@[^@]+\\.[^@]+$'")
@dlt.expect_or_fail("positive_amount", "amount > 0")
@dlt.expect("recent_date", "order_date >= current_date() - interval 5 years")
def silver_customers():
    return dlt.read_stream("bronze_customers").transform(...)
```

**Benefit:** Centralized data quality enforcement.

### Use Case 3: Frequent Reprocessing Needs

**Scenario:** Business logic changes frequently requiring historical reprocessing.

**Medallion advantage:** Reprocess from bronze without re-ingesting from source.

### Use Case 4: Large-Scale Data Platform

**Scenario:** 100+ data sources, 50+ data engineers, multiple business units.

**Medallion advantage:**

- Standardized approach across teams
- Clear ownership (bronze: data engineering, gold: analytics)
- Consistent patterns for onboarding

### When Alternative Approaches Make Sense

### Alternative 1: Two-Layer (Raw + Curated)

**When to use:**

- Small to medium data volumes (<1 TB)
- Managed ingestion tools handle initial quality
- Reprocessing rarely needed
- Team size <10 people

**Implementation:**

```typescript
raw/
├── source_a/
└── source_b/

curated/
├── metrics/
└── dimensions/
```

### Alternative 2: dbt Stage-Based

**When to use:**

- SQL-first transformation approach
- Using dbt as primary transformation tool
- Team familiar with dbt patterns

**Implementation:**

```sql
models/
├── staging/    (like bronze/silver combined)
├── intermediate/
└── marts/      (like gold)
```

### Alternative 3: Event Sourcing

**When to use:**

- Event-driven architecture
- Need complete audit trail
- Time-travel queries critical

**Implementation:**

```bash
events/         (immutable event log)
projections/    (materialized views of events)
```

### Migration Patterns

### From Single-Layer to Medallion

**Gradual approach:**

**Phase 1: Add Bronze**

```makefile
# Existing transformation
df = spark.read.json("s3://raw/orders").transform(...)

# New: Write to bronze first
bronze_df = spark.read.json("s3://raw/orders")
bronze_df.write.save("/bronze/orders")

# Existing transformation continues from bronze
df = spark.read.delta("/bronze/orders").transform(...)
```

**Phase 2: Separate Silver**

```makefile
# Move cleaning to silver
silver_df = spark.read.delta("/bronze/orders") \
    .transform_cleaning()
silver_df.write.save("/silver/orders")
```

**Phase 3: Create Gold**

```makefile
# Move aggregations to gold
gold_df = spark.read.delta("/silver/orders") \
    .aggregate()
gold_df.write.save("/gold/metrics")
```

### From Medallion to Simplified

**Consolidation approach:**

**Step 1: Analyze usage**

```sql
-- Check if bronze is actually used for reprocessing
SELECT 
    table_name,
    last_query_time
FROM information_schema.tables
WHERE table_schema = 'bronze'
ORDER BY last_query_time DESC;

-- If bronze tables rarely queried, consider removing
```

**Step 2: Merge layers**

```makefile
# Combine silver + gold if separation not providing value
curated_df = spark.read.json("s3://raw/orders") \
    .transform_cleaning() \
    .aggregate()
curated_df.write.save("/curated/metrics")
```

### Real-World Examples

### Example 1: E-commerce Company

**Architecture:** Full medallion (bronze/silver/gold) **Team size:** 15 data engineers **Data volume:** 10 TB **Rationale:** Multiple business units consuming data, frequent reprocessing needs

**Results:**

- Processing time: Bronze (5 min) + Silver (15 min) + Gold (10 min) = 30 min
- Storage cost: $230/month (10 TB × 3 layers × $0.023/GB)
- Team feedback: "Clear ownership and responsibilities"

### Example 2: SaaS Startup

**Architecture:** Two-layer (raw/curated) **Team size:** 3 data engineers **Data volume:** 500 GB **Rationale:** Simple use cases, managed ingestion via Fivetran

**Results:**

- Processing time: 12 minutes end-to-end
- Storage cost: $23/month (500 GB × 2 layers × $0.023/GB)
- Team feedback: "Simple to understand and maintain"

### Example 3: Financial Services

**Architecture:** Four-layer (raw/bronze/silver/gold) **Team size:** 30 data engineers **Data volume:** 50 TB **Rationale:** Regulatory requirements for audit trail

**Results:**

- Processing time: 45 minutes end-to-end
- Storage cost: $4,600/month (50 TB × 4 layers × $0.023/GB)
- Team feedback: "Necessary for compliance but operationally complex"

### Alternatives to Consider in 2026

### 1\. Data Vault 2.0

**Structure:**

```perl
raw_vault/
├── hubs/       (business keys)
├── links/      (relationships)
└── satellites/ (descriptive attributes)

business_vault/
└── (derived calculations)

information_marts/
└── (consumption layer)
```

**When to use:** Highly complex enterprise environments with many source systems.

### 2\. Modern Data Stack Pattern

**Structure:**

```typescript
Ingestion (Fivetran/Airbyte)
    ↓
Warehouse (Snowflake/BigQuery)
    ↓
Transformation (dbt)
    staging/
    intermediate/
    marts/
```

**When to use:** Cloud data warehouse-centric architecture.

### 3\. Event-Driven Architecture

**Structure:**

```java
Event Streams (Kafka)
    ↓
Stream Processing (Flink)
    ↓
State Stores (Delta Lake)
    ↓
Query Layer (Trino)
```

**When to use:** Real-time processing requirements.

### Best Practices (If Using Medallion)

### 1\. Clear Naming Conventions

```xml
bronze_<source>_<entity>     (e.g., bronze_salesforce_accounts)
silver_<domain>_<entity>     (e.g., silver_crm_accounts)
gold_<usecase>_<metric>      (e.g., gold_finance_monthly_revenue)
```

### 2\. Document Layer Boundaries

Create README.md in each layer:

```markdown
# Silver Layer

## Purpose
Cleaned and validated data ready for business logic.

## Quality Standards
- No duplicates
- No null primary keys
- Standardized data types
- Valid date ranges

## Schema Changes
Coordinate with data engineering team before making schema changes.
```

### 3\. Implement Data Quality Checks

```python
# Bronze: Minimal checks
assert bronze_df.count() > 0, "No data ingested"

# Silver: Comprehensive checks
assert silver_df.filter("order_id IS NULL").count() == 0
assert silver_df.filter("amount < 0").count() == 0
assert silver_df.select("order_id").distinct().count() == silver_df.count()

# Gold: Business rule validation
assert gold_df.filter("total_revenue < 0").count() == 0
```

### 4\. Monitor Layer Performance

```makefile
# Track processing time per layer
import time

start = time.time()
process_bronze()
bronze_time = time.time() - start

start = time.time()
process_silver()
silver_time = time.time() - start

start = time.time()
process_gold()
gold_time = time.time() - start

log_metrics({
    "bronze_processing_seconds": bronze_time,
    "silver_processing_seconds": silver_time,
    "gold_processing_seconds": gold_time
})
```

### Decision Framework

**Choose Medallion (Bronze/Silver/Gold) if:**

- Large organization (>20 data practitioners)
- Multiple data sources and consumers
- Complex data quality requirements
- Frequent reprocessing needs
- Using Databricks as primary platform

**Choose Two-Layer (Raw/Curated) if:**

- Small to medium organization (<20 people)
- Managed ingestion tools handle quality
- Simple transformation requirements
- Storage costs are a concern
- Quick time-to-value prioritized

**Choose dbt Stage-Based if:**

- SQL-first transformation approach
- Cloud data warehouse-centric
- Team familiar with dbt patterns
- Don't need raw data preservation

**Choose Alternative Architecture if:**

- Specific requirements (real-time, data mesh, etc.)
- Strong reasons to deviate from standard patterns
- Team expertise in alternative approaches

### The 2026 Verdict

**Medallion architecture remains relevant** for organizations that:

- Use Databricks or lakehouse platforms
- Have complex data quality requirements
- Need comprehensive audit trails
- Operate at enterprise scale

**However, it's not universal**:

- Simpler alternatives work well for smaller organizations
- Storage and complexity costs are real
- Modern tools (dbt, cloud warehouses) enable different patterns

**Key trend:** Teams are adapting the concept rather than following it rigidly. Common variations:

- Two layers instead of three
- Different naming (staging/intermediate/marts)
- Selective application (medallion for some pipelines, not all)

**Recommendation:** Evaluate based on your specific context:

- Start simple, add complexity only when needed
- Consider team size and expertise
- Assess actual reprocessing needs
- Calculate total cost of ownership (storage + compute + operations)

The pattern that works is the one your team can effectively operate and maintain.

**Disclaimer:** Architecture recommendations are context-dependent and evolve with technology changes. The patterns discussed reflect current practices as of December 2025 - January 2026 but may not be optimal for all situations. Always evaluate based on your specific requirements, constraints, and team capabilities. No architectural pattern is universally superior.

[#databricks](https://medium.com/tag/databricks "Databricks") [#snowflake](https://medium.com/tag/snowflake "Snowflake") [#azure](https://medium.com/tag/azure "Azure") [#google](https://medium.com/tag/google "Google") [#aws](https://medium.com/tag/aws "AWS")

- ### Discord Translator
	Translate messages in Discord
	[Add To Chrome](https://chromewebstore.google.com/detail/ibipjomdhljdfonmeiemgbbjpilidhne?utm_source=item-share-cb)
- ### WhatsApp Translator
	Translate messages in WhatsApp
	[Add To Chrome](https://chromewebstore.google.com/detail/ekaocdggcoffjhdaaddndakidgonodhe?utm_source=item-share-cb)
- ### Prompt Optimizer
	Optimize your prompts for AI models like ChatGPT, Claude, and Gemini.
	[Add To Chrome](https://chromewebstore.google.com/detail/ohobmnkjljbohbjhnhafkcamclkjjikd?utm_source=item-share-cb)
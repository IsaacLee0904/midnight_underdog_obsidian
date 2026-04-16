---
title: "4 Advanced Data Modelling Techniques Every Data Engineer Must Learn | by Khushbu Shah | in ProjectPro"
source: "https://freedium-mirror.cfd/https://medium.com/projectpro/4-advanced-data-modelling-techniques-every-data-engineer-must-learn-1113bcf7f5e9"
author:
published:
created: 2026-04-16
description: "These are the exact modeling techniques you must use to scale data pipelines in production."
tags:
  - "clippings"
---
[< Go to the original](https://medium.com/projectpro/4-advanced-data-modelling-techniques-every-data-engineer-must-learn-1113bcf7f5e9#bypass)

![Preview image](https://miro.medium.com/v2/resize:fit:700/0*Rg6OVFpNc_MkBew-)

## 4 Advanced Data Modelling Techniques Every Data Engineer Must Learn

## These are the exact modeling techniques you must use to scale data pipelines in production.

[![Khushbu Shah](https://miro.medium.com/v2/resize:fill:88:88/1*v6l6vVqNoAhbfMBioeveqQ.jpeg)](https://medium.com/@khushbu.shah_661 "I love writing at the intersection of AI, Data science, and Data Engineering to simplify complex topics that help readers confidently grow their careers.")

[Khushbu Shah](https://medium.com/@khushbu.shah_661 "I love writing at the intersection of AI, Data science, and Data Engineering to simplify complex topics that help readers confidently grow their careers.")[ProjectPro](https://medium.com/projectpro "ProjectPro helps data professionals build job-ready skills…")androidstudio ~8 min read · July 31, 2025 (Updated: July 31, 2025) · Free: Yes

If you're still writing `SELECT * FROM raw_logs` and hoping your dashboards run fast, it's time for a reality check. Real data engineering, the kind practiced at companies like Airbnb, Netflix, and Shopify, starts with rock-solid data models. And no, I'm not talking about basic star schemas you skimmed over in a YouTube tutorial. I'm talking about real modeling techniques that make data pipelines scale, queries fly, and data engineers happy.

In this blog, I'm walking you through 4 advanced data modeling techniques that real data engineers use when helping teams clean up their pipeline messes: from designing cumulative tables that make daily metrics effortless, to modeling graph relationships that power recommendations. These are patterns that every data engineer should master when building new systems from scratch or while refactoring legacy ETL. And if you want to get your hands dirty with production-grade projects that go beyond theory, the team at [ProjectPro](https://www.projectpro.io/) has put together a set of [real-world data engineering projects](https://www.projectpro.io/article/real-world-data-engineering-projects-/472) that focus on solving messy, end-to-end problems, the kind you'll face in production. Honestly, it's one of the few resources that consistently teaches this stuff at scale.

Let's get into it.

![None](https://miro.medium.com/v2/resize:fit:700/0*Rg6OVFpNc_MkBew-)

### 1\. Cumulative Table Design

If you've ever waited too long for a dashboard to load, chances are the underlying queries are doing too much work. That's exactly the problem cumulative tables solve. A cumulative table is not just about calculating totals, but it's about preparing data so it's always ready when you need it. Instead of recalculating metrics like revenue, active users, or sales every single time, you store those pre-aggregated values ahead of time in a separate table. This approach is commonly used in data platforms like BigQuery, Snowflake, or Redshift, where performance and cost really matter.

#### Why You Should Use Cumulative Tables

- **Faster Query Speeds**: Since the data is already summarized, dashboards and APIs can fetch results instantly.
- **Lower Compute Costs**: You don't need to scan your full dataset repeatedly so it's great for big data environments.
- **Scales Easily**: Works well with partitioned tables and lets you manage data growth more effectively.

Let's say the product team at ProjectPro wants a Daily Active Users (DAU) metric. Your raw events table logs hundreds of millions of records per day. Querying that directly every time is not ideal. Instead, we can create a table that stores just the DAU per date:

```
SELECT 
  event_date, 
  COUNT(DISTINCT user_id) AS daily_active_users,
  CURRENT_TIMESTAMP() AS last_updated
FROM events
WHERE event_type = 'login'
GROUP BY event_date;
```

This cumulative table only gets updated daily, and there is no need to touch historical data again and again. The dashboard becomes faster, and your data warehouse stays lean. When building cumulative tables, always include audit columns like `_last_updated` so that others know when the data was refreshed. Also, try to partition by date and cluster by common filters to keep your queries efficient.

### 2) Fact Data Modeling Fundamentals

If cumulative tables are your fast lanes, fact tables are the freeway itself. They power every metric in your system, but building them the right way isn't as simple as dumping numbers into a table. A well-structured fact table is the foundation of any scalable data warehouse. But if you're designing for event-scale systems with billions of rows a day, then you've got to treat fact table design as a performance discipline, not just a modeling exercise.

#### Key Considerations You Can't Skip

#### Grain Definition

This is non-negotiable. If your grain isn't clearly defined, everything downstream will break slowly, and expensively. Ask: *What does one row in this table represent?* Is it one page view, one transaction, or one user session? Your grain determines your joins, your aggregations, and your KPIs.

**Example**: If you define grain at the "user session" level, then each row might include `session_id`, `user_id`, `start_time`, and session-level metrics. But if you go more granular, say per event, you'll have millions more rows to process and store.

#### Surrogate Keys

Don't lean on natural or composite keys (like `user_id + timestamp`) for your joins. It's not just ugly, but it's slow. Instead, generate **surrogate keys that are** synthetic, usually auto-incremented or UUID-style values that serve as clean primary keys. These speed up joins and make your warehouse indexes perform like they're supposed to.

#### Partition Strategy

Here's where people blow up their Snowflake bill. If you're not partitioning by something smart like `event_date`, or a high-cardinality dimension, you're scanning way more data than needed.

**Do it right**:

- Time-based partitioning (`event_date`, `created_at`) for append-only datasets
- ID-based bucketing if your system supports it (BigQuery, Hive, etc.)
- Cluster on fields used most often in filters or group-bys

Your fact tables should never live alone. Pair them with well-designed dimension tables that carry all the slow-changing, descriptive attributes like product names, user types, and locations. And unless you're on a fully columnar system (like Snowflake or BigQuery), don't go crazy with denormalization. Wide, flat tables may feel convenient, but they'll crush your performance at scale.

### 3\. The Date List Data Type

Most engineers don't talk about this one, and that's exactly why it's a competitive edge. The date list is one of those advanced patterns you must explore if you are tired of row explosions and costly, unoptimized queries. The concept is simple: instead of creating one row per active date, you store an array of dates in a single row. That's it. But the downstream impact? Game-changing.

Let's say you're modeling ProjectPro's subscription data. A user signs up for a plan that lasts 14 days. Most beginner data models turn that into 14 rows, one for each day the user is active. That might work at 10K records. It won't scale at 10M. Storage bloat. Join hell. Query latency that makes your BI team hate you.

Here's how we do it instead:

```bash
{
  "user_id": "user_001",
  "active_dates": ["2025-07-01", "2025-07-02", ..., "2025-07-14"]
}
```

Now you've got one row per user per subscription, and `active_dates` is stored as an array (or a JSON field, depending on your warehouse). Use BigQuery's `REPEATED` fields, or Snowflake's `ARRAY` type, and you can still filter with ease:

```sql
WHERE '2025-07-05' IN UNNEST(active_dates)
```

#### Where It Shines

Here's where you will see the date list pattern pay off:

- **SaaS Usage**: Users active on non-consecutive days. Store only when they were active.
- **Hotel & Travel Bookings**: Multi-day stays. One row per reservation.
- **Event Scheduling**: Recurring events or irregular series. Avoid duplicating metadata across rows.

This model avoids duplicate rows, speeds up reads, and compresses beautifully in columnar storage.

#### Modeling Tip

When you store arrays like this, always track:

- `start_date` (helps with partitioning)
- `end_date` (quick filters)
- `active_dates` (full array for detail)

This gives you a fast entry point into the array and the ability to join with a date spine table when needed for reporting.

#### A few things you'll want to be careful about:

- Don't go array-crazy. If your consumers need flat tables, you'll have to `UNNEST`, which can be expensive if not done carefully.
- Index or cluster by `start_date` or `user_id` so you're not full-scanning giant arrays for every query.

The date list is a low-effort, high-impact model pattern for modern warehouses. It's perfect for semi-structured storage and aligns well with how tools like BigQuery, Snowflake, or even Delta Lake handle nested data.

### 4\. Graph Data Modeling

This is where most data engineers get uncomfortable, and that's exactly why you need to get good at it. Graph modeling isn't just for social networks or academia. If your data has complex, recursive, or relationship-heavy patterns, a graph structure can outperform a relational model by a mile.

I'm talking about:

- Multi-hop joins
- Real-time recommendations
- Dynamic dependency chains
- Fraud patterns
- Supply chain cascades

If that's the game you're playing, graphs are your best friend.

#### Why Graphs Matter in Data Engineering?

Relational databases are great when you know exactly what you're joining and how many hops it'll take. But once the relationships go beyond 2–3 degrees, like in "who influenced whom", "which system triggered which system", or "which customer knows which other customer" relational joins become a bottleneck. A well-modeled graph makes those traversals linear instead of combinatorial.

Let's say you're building a **fraud detection system** for a fintech product. You want to flag users who are:

- Using the same IP address
- Sharing bank accounts
- Referring each other suspiciously
- Using the same device fingerprint

Now imagine doing this with joins across five dimensions. You'll end up with a query plan that looks like a spaghetti diagram and runs for 30 minutes. In a graph? Each entity is a node, each shared connection is an edge, and you can write a `3-hop` traversal query that finds indirect connections in milliseconds.

#### When to Use Graph Models

Here's the litmus test you should use in production:

i. The relationships between things matter more than the things themselves

ii. You often ask "how is X connected to Y?"

iii. Your business logic depends on hops, influence, or connection strength

Common data domains where graphs win:

- **Recommendation systems**: people → items → tags → other people
- **Supply chain analysis**: item → vendor → factory → shipment
- **IT systems dependencies**: microservices → APIs → databases
- **Organizational structures**: manager → employee → project → team

Most engineers start by modeling the nodes (users, products, items). That's a trap. Start instead by listing out your edges: "what connects what?"

- A user referred another user
- A device was used by multiple accounts
- An item was bought with another item

If the relationships are frequent, dynamic, or query-critical, the right way to model them is as edges. Don't Use Graphs Just to Be Cool. Graphs are powerful, but they're not for everything. Don't migrate a perfectly working dimensional model to a graph just because Neo4j looked cool at a conference. Use graphs where:

- Relationships drive the business logic
- Queries are traversal-heavy
- Your current model is brittle under joins

Otherwise? Stick to columnar warehouses and optimize your star schema.

These four data modeling techniques: Cumulative Tables, Fact Table Fundamentals, Date List Data Types, and Graph Modeling aren't just concepts you learn and forget. They're foundational tools used daily by data engineers working at scale inside top-tier companies. Mastering them will set you apart.

If you're serious about becoming the kind of data engineer who builds for scale, reliability, and performance, and not just for a quick demo, then practice is non-negotiable. You need to find real-world data engineering use cases that put techniques like these into action, giving you the reps that matter. You can check out this [curated data engineering roadmap of real-world projects](https://www.projectpro.io/learning-paths/data-engineer-learning-path) that walk you through the exact workflows where these modeling patterns show up, from raw ingestion and transformation to designing performant data models in tools like BigQuery, Redshift, and Snowflake.

[#data-engineering](https://medium.com/tag/data-engineering "Data Engineering") [#artificial-intelligence](https://medium.com/tag/artificial-intelligence "Artificial Intelligence") [#data-science](https://medium.com/tag/data-science "Data Science") [#data-analysis](https://medium.com/tag/data-analysis "Data Analysis") [#machine-learning](https://medium.com/tag/machine-learning "Machine Learning")

- ### Discord Translator
	Translate messages in Discord
	[Add To Chrome](https://chromewebstore.google.com/detail/ibipjomdhljdfonmeiemgbbjpilidhne?utm_source=item-share-cb)
- ### WhatsApp Translator
	Translate messages in WhatsApp
	[Add To Chrome](https://chromewebstore.google.com/detail/ekaocdggcoffjhdaaddndakidgonodhe?utm_source=item-share-cb)
- ### Prompt Optimizer
	Optimize your prompts for AI models like ChatGPT, Claude, and Gemini.
	[Add To Chrome](https://chromewebstore.google.com/detail/ohobmnkjljbohbjhnhafkcamclkjjikd?utm_source=item-share-cb)
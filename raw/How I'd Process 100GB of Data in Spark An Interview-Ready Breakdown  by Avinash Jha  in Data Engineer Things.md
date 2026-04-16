---
title: "How I'd Process 100GB of Data in Spark: An Interview-Ready Breakdown | by Avinash Jha | in Data Engineer Things"
source: "https://freedium-mirror.cfd/https://blog.dataengineerthings.org/how-id-process-100gb-of-data-in-spark-an-interview-ready-breakdown-4a8924a486fd"
author:
published:
created: 2026-04-16
description: "Why This Spark Interview Question Isn't as Good as It Sounds"
tags:
  - "clippings"
---
[< Go to the original](https://blog.dataengineerthings.org/how-id-process-100gb-of-data-in-spark-an-interview-ready-breakdown-4a8924a486fd#bypass)

![Preview image](https://miro.medium.com/v2/resize:fit:700/1*pgPGfySbcLWNxa8I2mkA0w.jpeg)

## How I'd Process 100GB of Data in Spark: An Interview-Ready Breakdown

## Why This Spark Interview Question Isn't as Good as It Sounds

[![Avinash Jha](https://miro.medium.com/v2/resize:fill:88:88/1*R0fGA-naN8fzCCFUzs6Eqg.jpeg)](https://medium.com/@jha-avinash "Data Engineer | 5+ yrs experience | ex-Amazon | I build distributed data systems with Spark & cloud. I write about scaling, automation, and performance.")

[Avinash Jha](https://medium.com/@jha-avinash "Data Engineer | 5+ yrs experience | ex-Amazon | I build distributed data systems with Spark & cloud. I write about scaling, automation, and performance.")[Data Engineer Things](https://medium.com/data-engineer-things "Things learned in our data engineering journey and ideas on…")androidstudio ~4 min read · July 31, 2025 (Updated: July 31, 2025) · Free: No

### Why This Spark Interview Question Isn't as Good as It Sounds

> **"How would you process 100GB of data in Spark?"** **"How many executors will you use?"**

You'll often hear this question in data engineer interviews and it sounds technical, practical, and useful. But here's the problem:

> ***There is no single right answer.******It's a highly ambiguous question with too many assumptions.******It tests guessing more than real-world decision-making.***

#### Unless you're told:

1. What **cluster resources** are available?

2\. What **kind of data** you're processing (CSV, Parquet, text, etc.)?

3.What the **transformations** look like (joins, filters, shuffles?)?

4\. Whether it's **batch or streaming?**

5\. And what the **performance expectations** are…?

…you can't give a meaningful answer.

> It's like asking: *"* ***How much fuel do I need to drive 100 km****?"* Well, that depends are you in a motorcycle or a truck?

### A Better Way to Approach This Question

Since it's often asked anyway, the best approach is to:

**Step 1-> Requirement Gathering:**

#### Ask for these questions

**Type of Data**: Is it JSON, XML, CSV, Parquet, or coming from a database? *(e.g., Parquet is fast and columnar; JSON is slow to parse.)*

**Type of Transformation**: Are we just filtering and projecting? Or doing joins, aggregations, and UDFs?

**Single or Multiple Sources**: Reading from one file or combining data from multiple systems? This impacts read time and join complexity.

**Any SLA?** Is there a time expectation like finishing in 1 hour or 2 hours?

> Knowing these helps you design for performance, not just configure executors.

#### Step 2: Cluster Configuration

Once you understand the workload, asses your available infrastructure, ask the interviewer for the same. Sometime they might not give you exact details and may ask you to assume yourself.

#### Step 3: Now let's start calculation

Assume your node has **60 GB total RAM**.

#### You configure:

```java
15 GB RAM per executor
5 cores per executor
4 executors

Reserved Memory = 300 MB × 4 = 1.2 GB
So, usable memory = 60–1.2 = 58.8 GB ≈ 59 GB
```

Spark Memory Layout:

- **60% of 59 GB (≈36 GB)** → **Execution Memory** (for shuffles, joins, aggregations)
- **40% of 59 GB (≈24 GB)** → **Storage Memory** (for caching, broadcast vars)

> **Max memory usable in one iteration** = **36 GB**

### Caching Impact

If you cache data, part of the memory will be reserved for storage. So execution memory will drop:

> Range of available memory = **18 GB — 36 GB**, depending on how much data is cached.

### Estimating Iterations

Say your data size per shuffle stage is 140 GB.

If Spark can only process ~ **18–36 GB per iteration**, the number of iterations is:

```python
140 GB / 18 GB ≈ 7.7 → round up → ~8 iterations
```

> So you'll need to process data in **~7–8 waves** (based on **cache** + **memory** availability).

### Key Takeaway

Depending on your caching strategy, the number of iterations will range between:

```bash
4 (best case) → 7 (worst case)
```

This is a crucial mental model before running expensive jobs, it helps plan memory, stage failures, and optimize performance.

### Step 4: Debugging Mindset — What to Look For in Spark UI

After tuning memory and cluster config, **the next mindset shift** is about **debugging smartly**.

Use this checklist every time you open the Spark UI:

### Debugging Checklist

1. **Do I have skewed data?** Check **task time variance** in the **Spark UI's Stage tab**. Skew leads to long tails.
2. **What is the min and max executor utilization?** Use the **Executors tab** to identify **underutilized vs overloaded** executors.
3. **Which job/stage is slowing things down?** Go to the **Jobs tab** → drill into stages → check **duration vs shuffle read/write**.
4. **Do I need any optimization?**
- Large shuffles
- Wide stages (too many tasks)
- Long GC times

**Finally**

**Did I meet my SLA?** Compare **total job duration** with your SLA (e.g., finish in under 15 mins?).

### Step 5: What If You Still Don't Meet SLA?

Let's say your job **still doesn't meet the SLA** even after initial tuning and debugging. Here's your next move:

### Things to Consider

1. **Increase Executor Memory** If your tasks are memory-intensive (e.g., wide transformations, joins), increase executor memory.
2. **Use Caching and Repartitioning**
- **Caching** is useful if the same data is accessed repeatedly across stages.
- **Repartitioning** helps if you have **skewed partitions** or **too few/too many partitions**.

### New Optimized Configuration

Let's apply a different config setup to improve performance:

```sql
15 GB RAM per executor
5 cores per executor
5 executors total instead of 4.
```

This config assumes you've already optimized your code and are now scaling compute to meet tight deadlines.

### Trial-and-Error Based on Spark UI

Once the basic configurations are in place, the real tuning starts. This phase is all about **observing Spark UI**, identifying bottlenecks, and making **small, controlled changes**.

> ***"So all of this is trial and error now based on Spark UI response."***

You look for things like:

- Skewed tasks
- Uneven resource utilization
- Long GC times
- Slow shuffle reads or writes
- Failed/retried stages

> In the end, performance tuning is not just config tuning — it's a mindset shift.

To be honest, there's no fixed rule or perfect answer for these kinds of Spark optimization questions. It really depends on the problem you're trying to solve, the size and shape of the data, and the kind of resources you have like the **cluster**, **configs**, and **budget**.

> Most of the time, it's a mix of understanding the basics, making a few educated guesses, and then a **lot of trial and error.**

I've learned that the best way to tune things is by checking the Spark UI, seeing where the bottlenecks are, and making small changes one step at a time. It takes patience, but over time you start building that instinct. And that's honestly where the real learning happens.

[#data-engineering](https://medium.com/tag/data-engineering "Data Engineering") [#software-development](https://medium.com/tag/software-development "Software Development") [#apache-spark](https://medium.com/tag/apache-spark "Apache Spark") [#technology](https://medium.com/tag/technology "Technology")

- ### Discord Translator
	Translate messages in Discord
	[Add To Chrome](https://chromewebstore.google.com/detail/ibipjomdhljdfonmeiemgbbjpilidhne?utm_source=item-share-cb)
- ### WhatsApp Translator
	Translate messages in WhatsApp
	[Add To Chrome](https://chromewebstore.google.com/detail/ekaocdggcoffjhdaaddndakidgonodhe?utm_source=item-share-cb)
- ### Prompt Optimizer
	Optimize your prompts for AI models like ChatGPT, Claude, and Gemini.
	[Add To Chrome](https://chromewebstore.google.com/detail/ohobmnkjljbohbjhnhafkcamclkjjikd?utm_source=item-share-cb)
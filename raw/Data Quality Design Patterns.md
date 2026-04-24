---
title: "Data Quality Design Patterns"
source: "https://pipeline2insights.substack.com/p/data-quality-design-patterns-wap-awap"
author:
  - "[[Erfan Hesami]]"
published: 2025-12-02
created: 2026-04-25
description: "Overview of WAP, AWAP, TAP, and More with Implementation Examples"
tags:
  - "clippings"
---
In **[10 Pipeline Design Patterns for Data Engineers](https://pipeline2insights.substack.com/p/10-pipeline-design-patterns-for-data) [^1],** we explain what data pipelines are and walk through 10 key design patterns that help data engineers build scalable, efficient, and well-structured pipelines.

Now, let’s pause on a question every data engineer faces when designing a pipeline:

**How do we usually do our data quality checks?**

Do we:

- Load data into storage first, then validate it before transforming and publishing?
- Ingest, transform, then validate?
- Validate before ingestion, after ingestion, and again after transformation?
- Or ingest, transform, publish to production, *then* check?
- Or something similar, or completely different?

**We’d love to hear from you: what pattern do** ***you*** **use for data quality checks in your pipelines? Share your approach with us.**

As can be seen, there are many different approaches we use across the industry when implementing data quality checks, and we believe none of them is strictly right or wrong. The “best” pattern often depends on the type of quality issue, the use case, data size, platform limitations, SLAs, and many other factors.

> What’s common across all of them is this:  
> **We want to be proactive about data quality before it impacts downstream systems and undermines stakeholder trust.**

Whether we’re checking for row counts, growth rates, null patterns, schema changes, or something more advanced, the goal is always the same:

> Keep **bad data** out of **production**.

In this post, we share the patterns we’ve researched and seen used in the field, including how they work, when to use them, and their trade-offs:

- **WAP (Write–Audit–Publish)**
- **AWAP (Audit–Write–Audit–Publish)**
- **TAP (Transform–Audit–Publish)**
- **Signal Table Pattern**

These patterns reflect how modern data engineering teams protect production data, balance cost vs. safety, and design pipelines that scale. Let’s explore how each one works and when it makes sense to use it.

> **Note**: This post is part of our **[data quality mini-series](https://pipeline2insights.substack.com/t/data-quality)** [^2]**[,](https://pipeline2insights.substack.com/t/data-quality)** covering what data quality is, its impact on AI, key dimensions, how to build a data quality framework, and more

---

### WAP (Write–Audit–Publish)

![](https://substackcdn.com/image/fetch/$s_!HK3r!,w_1456,c_limit,f_webp,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2Feba687dc-e0d9-4bf0-8781-a1736b7f6ea2_1324x310.png)

The idea behind WAP actually comes from classic software engineering. Before shipping code to production, we deploy it to staging, run tests, validate behaviour, and only then publish it. WAP applies the exact same principle to data: we load new data into a safe, temporary area, check whether it’s valid, and only then make it available to production.

But this raises an interesting question: **didn’t we already have a similar concept or idea when designing the data warehouse?** We believe Kimball (and Inmon before him) described a *staging area* as an intermediate step in the data warehouse pipeline. Raw or semi-processed data is first landed in staging, then cleaned, validated, and transformed before being loaded into the warehouse’s dimensional model. So in a sense, data engineering often has a staging step.

What’s different today is the **intent**. In the modern definition, staging is not just an ETL convenience; it becomes an explicit environment boundary that protects production from bad data. WAP formalises this safety layer.

The pattern was popularised by **[Michelle Ufford](https://www.youtube.com/watch?v=fXHdeBnpXrg)** [at the 2017 DataWorks Summit](https://www.youtube.com/watch?v=fXHdeBnpXrg). She framed the core problem clearly: changing data pipelines can accidentally push bad data into production, breaking dashboards, machine-learning models, and downstream systems. The solution is to introduce a controlled gate in which new data is written to a temporary location, audited with quality checks, and promoted to production only if it passes. If it fails, it’s quarantined for review instead of corrupting production.

The name describes the flow:

- **Write**: write new data into a staging or temporary area
- **Audit**: run quality checks and validation rules
- **Publish**: promote the data to production only if it passes

In this post, [An Engineering Guide to Data Quality – A Data Contract Perspective (Part 2)](https://www.dataengineeringweekly.com/p/an-engineering-guide-to-data-quality) [^3], [Ananth Packkildurai](https://open.substack.com/users/3520227-ananth-packkildurai?utm_source=mentions) explains **two implementation patterns** for the WAP approach:

#### 1\. Two-Phase WAP

![](https://substackcdn.com/image/fetch/$s_!VF68!,w_1456,c_limit,f_webp,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2F5360816b-a7c7-4165-b010-fffe57afd37b_1803x748.png)

[Source](https://www.dataengineeringweekly.com/p/an-engineering-guide-to-data-quality)

This is the classic approach. It requires two physical copies of the table:

1. **Write**: write new data to a staging table.
2. **Audit**: validate the staged data.
3. **Publish**: if valid, copy it into the production table.

After publishing, staging is cleared out. This approach works with any system but involves extra storage and copying.

#### 2\. One-Phase WAP (Zero-Copy WAP)

![](https://substackcdn.com/image/fetch/$s_!lVRV!,w_1456,c_limit,f_webp,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2Fa3a3d543-c622-40c4-8cbf-f92d919062aa_1416x1131.png)

[Source](https://www.dataengineeringweekly.com/p/an-engineering-guide-to-data-quality)

Modern lakehouse table formats allow WAP without copying data.

Technologies like:

- **Apache Iceberg**
- **Apache Hudi**

support transactional branching or hidden snapshots. Iceberg, for instance, enables WAP simply through table properties like:

- `write.wap.enabled`
- `wap.id`

Here, the Write and Audit phases occur on an isolated snapshot or branch, and the Publish step is just a metadata commit, no deep copy required.

If you’re keen to learn more about these two approaches and how they work in **streaming**, please check out the full post [here](https://www.dataengineeringweekly.com/p/an-engineering-guide-to-data-quality) [^4].

**So how do we implement these patterns in real pipelines?**

---

### Different Ways to Implement WAP

As [Tomas Peluritis](https://open.substack.com/users/6780046-tomas-peluritis?utm_source=mentions) explained in his talk at PyCon Lithuania 2024, *[Write–Audit–Publish Pattern in Modern Data Pipelines](https://www.youtube.com/watch?v=-omlizzCALc)*, teams can implement WAP in multiple ways depending on their tech stack:

#### 1\. DIY Approach Using Pandas

This is the most flexible, but could be the most engineering-heavy approach. We manually implement the three WAP phases inside our pipeline code.

**How?**

**Write:**

![](https://substackcdn.com/image/fetch/$s_!hUuC!,w_1456,c_limit,f_webp,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2F8dbca21b-a3fc-42e4-a617-c5fd99eb9e1d_729x460.png)

[Source](https://dagster.io/blog/python-write-audit-publish)

**Audit:**

![](https://substackcdn.com/image/fetch/$s_!RDS-!,w_1456,c_limit,f_webp,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2F12316cf5-d64c-4414-b508-f6fa066dbcfd_735x157.png)

[Source](https://dagster.io/blog/python-write-audit-publish)

**Publish:**

![](https://substackcdn.com/image/fetch/$s_!0xkn!,w_1456,c_limit,f_webp,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2F85acb583-8609-43f8-9415-b661737f7c60_748x229.png)

[Source](https://dagster.io/blog/python-write-audit-publish)

This approach works with any compute engine and gives full control over the validation logic, making it easy to inspect and troubleshoot invalid data. Using DataFrames (e.g., Pandas or Spark) allows flexible, in-memory checks and quick prototyping of custom rules. However, it requires custom engineering, involves writing data twice (to staging and production), which increases compute and storage costs, can suffer from performance and memory limitations for large datasets, and lacks transactional guarantees when multiple tables need to be published together.

#### 2\. Snowflake zero-copy clones with dbt

![](https://substackcdn.com/image/fetch/$s_!RA19!,w_1456,c_limit,f_webp,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2F44bb1cd4-6aca-48e5-a0dc-0b75fc18339c_1584x518.png)

[Source](https://uncledata.substack.com/p/write-audit-publish-pattern-in-modern)

Snowflake supports a WAP pattern using zero-copy clones, allowing pipelines to create full copies of tables or schemas instantly without duplicating storage.

**How?**

**Write:**

- Transform all data into a development or staging schema (e.g., `RAW_WAP`).
- Run dbt models in this “sandbox” schema.

**Audit:**

- Run dbt tests or Snowflake-native quality checks on the cloned assets.
- Nothing touches production until tests are green.

**Publish:**

- If validation passes, use a **zero-copy clone** to promote the staged tables into the production schema.
- Promotion is nearly instantaneous because only metadata pointers change.

This approach ensures that production data is never corrupted during validation, providing a safe environment to test pipelines with full datasets. It also allows manual adjustments and re-testing before publishing changes. However, this approach requires maintaining custom orchestration code to manage clones and validations, adds some complexity to the pipeline (though less than DIY in-memory solutions), and still consumes additional compute and storage resources for transformations and testing.”

#### 3\. Apache Iceberg

Apache Iceberg introduces a **branching mechanism** that works like Git for data.  
This makes it a very powerful WAP implementation..

**How?**

1. **Write:**
	- Create a branch off the main table:  
		`CREATE BRANCH feature_run_123 FROM main`
		- Run transformations directly on that branch, not on the main dataset.
2. **Audit:**
	- Run DQ tests, checks, or manual inspection on the branch.
		- The branch reflects a full versioned snapshot of the data.
3. **Publish:**
	- If everything passes, “fast-forward” the branch into `main`.  
		This is atomic and consistent across all related tables.
		- If tests fail, simply delete the branch and start again.

**We also explored how we can implement the WAP pattern using:**

- **[dlt](https://dlthub.com/blog/write-audit-publish-wap)** [^5]**:**  
	Data is first extracted and written to a temporary location, then loaded into a dataframe for auditing, and finally written again to the destination after validation.

If you’re not familiar with dlt, check out this beginner-friendly post here:

- **[Airflow](https://ghostinthedata.info/posts/2025/2025-05-18-wap-data-pipelines/#:~:text=Enter%20the%20Write%2DAudit%2DPublish,of%20troubleshooting%20and%20emergency%20fixes.)** [^6]**:**  
	WAP is handled at the DAG level. Airflow manages the conditional logic and control flow, while the storage/compute layer performs the actual data reads, audits, and writes.
- **[dbt](https://www.getdbt.com/blog/testing-is-not-enough-transforming-data-quality-with-write-audit-publish)** [^7] **(**using `sdf build`**):**  
	The `sdf build` command abstracts the full WAP workflow by providing:
	- Creates `_draft` tables for all transformations.
		- Runs dbt tests and validations against the staged data.
		- Publish validated data to production without re-running the transformations.
		- Incremental models and snapshots work efficiently without losing performance.

Moreover, in this excellent course, ***[Data Quality: Transactions, Ingestions, and Storage](https://www.linkedin.com/learning/data-quality-transactions-ingestions-and-storage/introduction-to-the-write-audit-publish-wap-pattern-26845652?resume=false)*** [^8]**, [Mark Freeman](https://open.substack.com/users/139605027-mark-freeman?utm_source=mentions)** demonstrates a Python-based approach. He implements WAP in Python, extracting data from PostgreSQL, validating each record against Avro schemas, quarantining invalid rows, staging the clean data, and only then publishing it to production, ensuring high data quality while keeping production tables safe. The source code is available [here](https://github.com/LinkedInLearning/data-quality-transactions-ingestions-and-storage-3924005/blob/main/data_platform/etl/batch_postgres_minio_wap_etl.py).

![](https://substackcdn.com/image/fetch/$s_!MfmZ!,w_1456,c_limit,f_webp,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2F10ba6888-4d4d-461b-a852-657ed490a09d_2736x1502.png)

[Source](https://github.com/LinkedInLearning/data-quality-transactions-ingestions-and-storage-3924005/blob/main/data_platform/etl/batch_postgres_minio_wap_etl.py)

---

### AWAP

![](https://substackcdn.com/image/fetch/$s_!vkbA!,w_1456,c_limit,f_webp,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2F2a388fac-caf6-42dc-9d09-a979010304d0_1423x358.png)

In **[Data Engineering Design Patterns: Recipes for Solving the Most Common Data Engineering Problems](https://www.amazon.com.au/Data-Engineering-Design-Patterns-Problems/dp/1098165810)** [^9] by Bartosz Konieczny, we encounter the AWAP pattern. According to the author, AWAP is an evolution of the traditional WAP pattern, adding **more thorough checks on input data**. Unlike WAP, AWAP includes additional validation logic to perform **lightweight verification** on the incoming dataset before further processing.

1. **First Audit (Input Validation):**
	Validate the incoming raw data before any extraction.
	- **Checks may include:**
		- File format validation (CSV, JSON, Parquet, etc.)
				- Schema verification (columns present, correct types)
				- Basic metrics like row count, file size, or table size
		- **Goal:** Catch obvious issues early, e.g., missing columns or corrupted files, **before incurring compute costs in subsequent processing**.
2. **Write / Transform**
	- Apply transformations to the data and/or write it to a staging or intermediate location.
		- This step is similar to the WAP “Write” step, where we prepare production-ready data while keeping it separate from the live environment.
3. **Second Audit (Output Validation)**
	- Validate the transformed data to ensure transformations didn’t introduce errors.
		- **Checks may include:**
		- Row-level validations (e.g., no NULLs where not allowed)
				- Business rules (e.g., totals, ratios, unique constraints)
				- Aggregate metrics (e.g., distinct count, sum, averages)
		- **Goal:** Ensure the transformed dataset meets **production standards and business expectations**.
4. **Publish (Promote to Production)**
	- Only after both audit steps pass:
		- Move the validated data to the production environment (Data Lake, Warehouse, or Lakehouse).
				- Optionally, create backups of previous production tables for safety.

![](https://substackcdn.com/image/fetch/$s_!16D9!,w_1456,c_limit,f_webp,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2F9f8b4539-4675-4e67-a442-cbfe133d5bd5_730x547.png)

The AWAP pattern in a batch pipeline, page 335

The AWAP pattern helps ensure high data quality by checking both input and output datasets, catching errors introduced by sources or transformations, and reducing bad data in production. It extends unit tests to real data, allows flexible validation of records or entire datasets, and works for both batch and streaming pipelines. However, it comes with higher compute and storage costs, added pipeline complexity, potential redundancy, possible streaming delays, and isn’t completely foolproof, since rules can become outdated or trigger false positives.

Here’s my experience at Xero:

> - Implemented a **data quality framework** at Xero that applied checks at multiple stages of the pipeline:
> - **Pre-processing checks:** Verified source data in S3, ensuring tables existed, monitoring growth rates, row counts, and other metrics.
> - **During transformation:** Applied additional validations to maintain data accuracy.
> - **Post-loading checks:** After loading the target table and archiving the previous month’s data, I performed final validations before notifying stakeholders.
> - **Resource usage:** Not a primary concern; the focus was on **ensuring data accuracy and reliability**, even if it increased compute or storage requirements.

---

### TAP

![](https://substackcdn.com/image/fetch/$s_!Rlok!,w_1456,c_limit,f_webp,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2Fb5afe0ab-ee82-45ae-a449-044edf1e6a29_796x643.png)

[Daniel Beach](https://open.substack.com/users/21715962-daniel-beach?utm_source=mentions) in **[Introduction to Write-Audit-Publish Pattern](https://dataengineeringcentral.substack.com/p/introduction-to-write-audit-publish)** [^10]**[,](https://dataengineeringcentral.substack.com/p/introduction-to-write-audit-publish)** argues that while WAP has been effective, there is a simpler and more cost-effective alternative. WAP involves multiple read and write operations: writing transformed data to a staging table, reading it again for auditing, and then writing it to production.

In cloud environments, especially with TB-scale datasets on object stores like S3 or table formats like Delta/Iceberg, these extra I/O operations quickly become expensive. Every read, write, and extra file increases storage and metadata overhead, making WAP costly in modern lakehouses designed for object storage rather than traditional warehouses.

So he introduced TAP, which addresses this inefficiency by performing data quality checks directly in memory during transformation. The validated data is then written straight to production, eliminating unnecessary intermediate storage and reducing cloud I/O costs. This makes TAP faster, cheaper, and more aligned with modern cloud-native workflows.

What do you think?

---

### 2\. Signal Table Pattern

[Zach Wilson](https://open.substack.com/users/10367987-zach-wilson?utm_source=mentions) introduced the Signal Table Pattern in his post “ [Writing Data to Production Is a Contract That Isn’t Free!](https://blog.dataexpert.io/p/writing-data-to-production-is-a-contract)”, based on his experience working at Facebook. This pattern offers an alternative to the WAP approach for ensuring data quality.

The way this pattern works is:

- Write directly to the production table.
- Run our audits on production.
- If they pass, publish a signal that lets the downstream know the data is ready

![](https://substackcdn.com/image/fetch/$s_!_Bws!,w_1456,c_limit,f_webp,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2Ffb588e4c-8cba-47d6-bb13-28a1c7d6833a_2048x717.png)

[Source](https://blog.dataexpert.io/p/writing-data-to-production-is-a-contract)

**Pros:**

- Simpler to implement since there’s no staging table or partition exchange step.
- Faster data availability, reducing latency and helping meet SLAs.

**Cons / Risks:**

- Ad hoc queries on production data can see bad or incomplete data if analysts ignore the signal table.
- Less intuitive, as downstream consumers must explicitly wait for the signal table instead of relying on the production table itself.
- Bad data propagation is possible if the contract is violated, leading to costly fixes.

In short, the Single Table Pattern trades **some safety guarantees** for **simplicity and speed**, while WAP prioritises **data quality and reliability** over immediate availability.

---

### Conclusion

Good data quality is key to reliable pipelines. There are different patterns for checking and protecting data, such as WAP and AWAP, which use staging and audits to keep bad data out of production; TAP checks data in memory to save time and cost; and the Signal Table Pattern is simpler and faster but less safe. What matters most is catching errors early, keeping production data clean, and maintaining trust in your data. By understanding these patterns, data engineers can build pipelines that are safe, efficient, and scalable.

---

### We value your feedback

If you have any feedback, suggestions, or additional topics you’d like us to cover, please share them. We’d love to hear from you!

![](https://substackcdn.com/image/fetch/$s_!7ume!,w_1456,c_limit,f_webp,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2Fbd6c4369-5223-4820-94ae-f48d44af1898_489x274.png)

***Refer***

***3 friends: a 1-month free.***

***10 friends: a 3-month free.***

***25 friends: a 6-month free subscription.***

***Our way of saying thanks for helping grow the P2I community!***

[^1]: https://pipeline2insights.substack.com/p/10-pipeline-design-patterns-for-data

[^2]: https://pipeline2insights.substack.com/t/data-quality

[^3]: https://www.dataengineeringweekly.com/p/an-engineering-guide-to-data-quality

[^4]: https://www.dataengineeringweekly.com/p/an-engineering-guide-to-data-quality

[^5]: https://dlthub.com/blog/write-audit-publish-wap

[^6]: https://ghostinthedata.info/posts/2025/2025-05-18-wap-data-pipelines/#:~:text=Enter%20the%20Write%2DAudit%2DPublish,of%20troubleshooting%20and%20emergency%20fixes.

[^7]: https://www.getdbt.com/blog/testing-is-not-enough-transforming-data-quality-with-write-audit-publish

[^8]: https://www.linkedin.com/learning/data-quality-transactions-ingestions-and-storage/introduction-to-the-write-audit-publish-wap-pattern-26845652?resume=false

[^9]: https://www.amazon.com.au/Data-Engineering-Design-Patterns-Problems/dp/1098165810

[^10]: https://dataengineeringcentral.substack.com/p/introduction-to-write-publish
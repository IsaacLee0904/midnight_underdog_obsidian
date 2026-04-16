---
title: "I spent 5 hours learning Unity Catalog. Here's everything you need to know. | by Vu Trinh"
source: "https://freedium-mirror.cfd/https://blog.dataengineerthings.org/i-spent-5-hours-learning-unity-catalog-heres-everything-you-need-to-know-cebdfe8b0d69"
author:
published:
created: 2026-04-16
description: "The famous catalog service from Databricks, and it was open-sourced"
tags:
  - "clippings"
---
[< Go to the original](https://medium.com/@vutrinh274/i-spent-5-hours-learning-unity-catalog-heres-everything-you-need-to-know-cebdfe8b0d69#bypass)

![Preview image](https://miro.medium.com/v2/resize:fit:700/0*lbIZygnpZwyXWhRr.png)

## I spent 5 hours learning Unity Catalog. Here's everything you need to know.

## The famous catalog service from Databricks, and it was open-sourced

[![Vu Trinh](https://miro.medium.com/v2/resize:fill:88:88/1*Pn2C-qSJVHq3nURZKDmOzA.jpeg)](https://medium.com/@vutrinh274 "Follow for practical data engineering articles with self-created illustrations. No AI-writing content")

[Vu Trinh](https://medium.com/@vutrinh274 "Follow for practical data engineering articles with self-created illustrations. No AI-writing content")

androidstudio · January 21, 2026 (Updated: January 21, 2026) · Free: No

### Intro

In any database, a catalog is a central repository that stores metadata for all database objects, such as tables, columns, views, users, and relationships, acting as a directory to help the system find, understand, and manage data.

In the lakehouse world, there is also a concept of a catalog.

![None](https://miro.medium.com/v2/resize:fit:700/0*Ps8OVVWdKkNFbtCw.png)

It is the central metadata layer, serving as a unified directory to discover, govern, and manage data across diverse sources (such as data lakes and warehouses) by tracking tables, schemas, and access rules, enabling analytic engines and AI models to access data without moving it.

Databricks, known as the vendor that publicly introduced the concept of lakehouse to the world, has built and provided a robust catalog service on top of their offering.

It's called Unity Catalog. It has served as Databricks's proprietary product for a long time before the vendor decided to open-source it last year.

In this week's article, I share my insights on the Unity Catalog after reading the Databricks paper, [Unity Catalog: Open and Universal Governance for the Lakehouse and Beyond.](https://dl.acm.org/doi/abs/10.1145/3722212.3724459)

### Problems Unity Catalog was trying to solve

Following the paper, UC has been serving the 9000 Databricks customers since 2021 with some interesting numbers: 100 million tables under management, 400,000 machine learning (ML) models, and 60,000 API calls per second (yeah, you read it right, per second). A good piece of software always starts with the user's problems in mind. Here are challenges that Unity Catalog was designed to address:

- **Uniform access control:** A secure approach to let Lakehouse's users access the assets (e.g., the tables or the ML models) in both ways: via catalog or via a direct cloud storage path.
- **Support for diverse asset types:** The catalog must support a wide range of assets, adapting to the user's needs. In addition to tables, users also need to govern ML models.
- **External access:** Users require the data sharing capability done via the catalog; the key is that it must avoid data copying (which might increase storage cost and increase complexity when you must ensure data synchronization between the two copies).
- **Discovery support:** The assets must be discoverable and understandable. Their lifecycle and lineage must also be transparent
- **Performance:** Users don't want it to be slow, even though it only involves metadata-related operations.

### How does Unity Catalog fit into the Databricks lakehouse architecture?

The architecture consists of three key components:

- **The storage**: As you might know, the lakehouse paradigm separates the storage and compute for flexibility and interoperability. The storage could be object storage services from famous vendors like AWS or Google. In addition to the physical data stored in Parquet, there is a metadata layer on top of it that provides the table abstraction. Delta Lake and Iceberg are used for this purpose.
- **The Databricks runtimes**: The Databricks' core execution engine, which is a [fork](https://docs.github.com/en/pull-requests/collaborating-with-pull-requests/working-with-forks/fork-a-repo) of Apache Spark that provides the same interface but has enhancements for reliability and performance. Databricks also built the Photon engine, a library that integrates closely with DBR to enhance Spark analytical workloads further. The engine acts as a new set of physical operators inside the DBR. The query plan can use these operators as any other Spark query. Databricks' customers can continue to run their workloads without any changes and still benefit from Photon.

> Here is my previous article about [How is Databricks' Spark different from Open-Source Spark?](https://open.substack.com/pub/vutr/p/how-is-databricks-spark-different?utm_campaign=post-expanded-share&utm_medium=web)

- **The Unity Catalog service**:And finally, today's primary focus, the Unity Catalog service. It's a multi-tenant service that provides all UC functionality and APIs. The clients (in most cases, the query engines) use this service to deliver what the user needs.

### The SQL query's journey

After having a glimpse of Databricks lakehouse architecture, we will understand a typical SQL query journey:

- Upon DBR receiving the SQL query from the user, it parses the query to extract the related data asset references, such as the table or view names.
- DBR then sends a REST request to the Unity Catalog (UC) service to verify whether the user has the required permission on the data assets. Then the UC will return the assets' metadata, such as column definitions and constraints.
- The DBR will use the return metadata for the planning process.
- When the DBR needs to access the data in the object storage, it issues another request to fetch a temporary credential.
- The DBR can now access the data using the returned credentials.
- The query can now be processed, from scanning data to filtering, joining, and aggregating. The result will be sent back to the user.

### The designs

After understanding the high-level of the Unity Catalog, we will dive into its system design to see how Databricks resolved the challenges discussed at the beginning of the article.

### The disaggregation of the catalog and engine

First, Databricks decides that the Unity Catalog will be an independent service, not tied to any specific query engine. This helps two things:

- **Security:** The Unity Catalog can act as a gatekeeper; only those with authorized permission can access assets managed by the catalog.
- **Interoperability** (the spirit of the lakehouse architecture)**:** Different query engines can work with data managed by the catalog. The query engine accesses the metadata via the REST API, rather than replicating the metadata to each engine.

### Diverse Asset Types management

As mentioned, in addition to tables, users need the Unity Catalog to manage various data asset types.

The foundation of this ability lies in the entity-relationship (ER) data model, which backs all metadata operations.

The model exposes methods for tasks such as ID- or name-based lookup, parent-child relationship mapping, privilege grant management, and state management for resource provisioning and cleanup. Developers can extend the model for a specific asset type. This model is materialized in a relational database.

On top of the model is the adapter layer. The primary responsibility of this layer is to offer integration points for different asset types with various cloud providers.

A developer can register an asset type in the catalog by adding a manifest to UC's asset types registry. The manifest includes information such as its location in the model hierarchy, the support operations and privileges, the rules associated with each operation, and how the catalog will manage the asset lifecycle.

In this way, multiple asset types can be added to the Catalog.

Right on top of the adapter is the core features layer, which provides methods like namespace, lifecycle management, access control, and audit logging.

This layer exposes APIs for metadata management and credential requests.

### Access control

Databricks wants to ensure that data access is secure, regardless of how it is accessed: via the catalog or directly via the object storage path.

First, Databricks protects all the assets at two distinct levels: metadata and data. Metadata security is handled directly through UC's REST APIs, where the system validates a user's permissions based on the specific operation being performed.

Rather than giving clients direct access to cloud storage, administrators grant storage permissions exclusively to the Unity Catalog service. When a client needs to read or write data, it requests temporary credentials from the UC.

If the engine needs to access the data via the object storage path, the UC will resolve the path to the asset identifier, check whether the client has sufficient permissions, and finally return a temporary credential.

For more fine-grained access control, such as row or column security, the UC works with the engine to enforce a two-level access control. In addition to the mechanism above to protect both the metadata and the data of a table, the engine is now responsible for row and column security.

### A discoverable catalog

Recalled from the "Diverse Asset Types management" section, Unity Catalog is designed as a layered architecture, with the bottom layer the entity-relation model stored in the relational database, followed by the core feature layer and the APIs layer.

On top of the APIs layer is a layer with background functions, such as discovery, search, and data lineage. The service must be notified and updated whenever the metadata is changed.

Whenever changes occur, the core service sends change events so the background functions can consume them to update the indexes or lineage graphs. Databricks adopts this event-driven architecture to ensure that background functions stay up to date with the core service.

The separation between the core service and the background functions ensures both can be scaled independently, and the failure of a service won't impact the remaining services.

The core service also provides access control for background functions, ensuring that only authorized clients can discover the assets.

### Performance

As UC is the entry point for all the analytical workload in Databricks, its access latencies directly impact the user experience. Here are some optimizations that Databricks implements for the UC:

- Batching multiple asset requests.
- Caching immutable metadata (e.g., temporary credentials) at both the UC service and the query engines.
- For mutable metadata (e.g., table's name, columns, permissions…), Databricks implements a write-through cache for the relational database that backs the metadata. The relational database's data will be sharded across nodes; each node is responsible for its assigned subset of data in both the database's storage and the cache.
- To prevent the cache from growing indefinitely. Databricks implements two strategies for cache eviction. First, there are standard algorithms, such as LRU, that discard unpopular cached entities. Second, to cap the number of cached versions of popular entities, Databricks enforces a timeout for all UC API calls. They assume that when a request to an asset populates a new version of the asset into the cache, the existing cached versions will be used for processing requests for at most the timeout period.

> In a write-through cache, data is written simultaneously to the cache and the primary storage system. This approach ensures that the cache always contains the most recent data. — from [What is Read-Through vs Write-Through Cache? by Design Guru](https://www.designgurus.io/answers/detail/what-is-read-through-vs-write-through-cache?gad_source=1&gad_campaignid=23163907085&gbraid=0AAAAADME9yoSi4tAWruR5iWGGAGnQZcX-&gclid=Cj0KCQiAgbnKBhDgARIsAGCDdle2rdxFg-8SWJywUX7CAwptXKd4E6MaSTRchDZQdsrRhjrR_H-0U_oaAlzDEALw_wcB)

- UC uses database versions to provide snapshot isolation and serializable isolation.

> The idea of snapshot isolation (SI) is that each read transaction reads a consistent snapshot of the database. The transaction will only see all changes committed before it starts.

> Serializable is the strongest isolation level: although transactions can run in parallel, their effects are the same as if they were run serially (one at a time). There are several approaches to implement serializability, including serialized snapshot isolation (SSI).

> SSI is still based on the SI; the readings are still served with consistent snapshots. However, SSI has mechanisms to detect conflicts between writes. A common approach is to detect whether the reading snapshot is stale.

> For a given transaction, SI provides it with a consistent snapshot and ignores changes from ongoing or new transactions. With SSI, the database can check if any ignored changes were committed during the transaction. If so, the database will abort the transaction if it's not a read-only transaction — from [ACID For Data Engineers by Vu Trinh](https://open.substack.com/pub/vutr/p/acid-for-data-engineers?utm_campaign=post-expanded-share&utm_medium=web).

### Outro

In this article, I've shared my insights after reading the paper [Unity Catalog: Open and Universal Governance for the Lakehouse and Beyond](https://dl.acm.org/doi/abs/10.1145/3722212.3724459). From the problems that Unity Catalog is trying to solve, how it fits into the Databricks lakehouse architecture, the query life cycle with the Catalog as the entry point, and finally, the system designs that made it a reliable, secure, and performant lakehouse catalog

### Reference

*\[1\] Databricks,* *[Unity Catalog: Open and Universal Governance for the Lakehouse and Beyond](https://dl.acm.org/doi/abs/10.1145/3722212.3724459)**, 2025*

[#data-engineering](https://medium.com/tag/data-engineering "Data Engineering") [#data-lakehouse](https://medium.com/tag/data-lakehouse "Data Lakehouse") [#databricks](https://medium.com/tag/databricks "Databricks") [#unity-catalog](https://medium.com/tag/unity-catalog "Unity Catalog") [#delta-lake](https://medium.com/tag/delta-lake "Delta Lake")

- ### Discord Translator
	Translate messages in Discord
	[Add To Chrome](https://chromewebstore.google.com/detail/ibipjomdhljdfonmeiemgbbjpilidhne?utm_source=item-share-cb)
- ### WhatsApp Translator
	Translate messages in WhatsApp
	[Add To Chrome](https://chromewebstore.google.com/detail/ekaocdggcoffjhdaaddndakidgonodhe?utm_source=item-share-cb)
- ### Prompt Optimizer
	Optimize your prompts for AI models like ChatGPT, Claude, and Gemini.
	[Add To Chrome](https://chromewebstore.google.com/detail/ohobmnkjljbohbjhnhafkcamclkjjikd?utm_source=item-share-cb)
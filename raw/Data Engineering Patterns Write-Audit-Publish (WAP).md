---
title: "Data Engineering Patterns: Write-Audit-Publish (WAP)"
source: "https://lakefs.io/blog/data-engineering-patterns-write-audit-publish/"
author:
  - "[[Robin Moffatt]]"
published: 2023-05-30
created: 2026-04-19
description: "This blog explains the concept of Write-Audit-Publish, which is a pattern in data engineering to enforce data quality in data pipelines."
tags:
  - "clippings"
---
Write-Audit-Publish (WAP) is a pattern in data engineering that gives teams greater control over data quality. It was popularized by Netflix back in 2017 in [a talk](https://www.youtube.com/watch?v=fXHdeBnpXrg) by Michelle Winters at the DataWorks Summit called “ *Whoops the Numbers are wrong! Scaling Data Quality @ Netflix*.”

In this three-part blog series, I’m going to look at what WAP is, how it can be [implemented](https://lakefs.io/blog/how-to-implement-write-audit-publish/) with various technologies in use today, and then provide [a deep-dive walkthrough of WAP in action](https://lakefs.io/blog/write-audit-publish-with-lakefs/).

We’ll start off the series by looking at the Write-Audit-Publish pattern itself, examine how widely it’s used—and how widely it *should* be used—and how it has close similarities to the Blue-Green deployment pattern used in software engineering.

## What is Write-Audit-Publish?

At the heart of WAP is the intention to ensure that users of the data can trust it. This is done by checking the data after it’s been processed but before it’s available to consumers. We call it a pattern because the way it’s implemented is going to vary a lot on the specifics of your technology platforms, architecture designs, and other—potentially conflicting—requirements.

WAP is useful because it means that consumers of the data—whether end-users viewing the data in a dashboard or subsequent data processing jobs—can have faith in the data that they use. Much of [data quality](https://lakefs.io/data-quality/) is easy to programmatically enforce (for example, are there NULLs where there shouldn’t be? Are fields within expected ranges?). By doing so, we avoid the loss of trust that can occur if processed data is made available and then retrospectively withdrawn or fixed after we discover errors.

Here’s an abstracted illustration of how it works. I’m showing the user of the data as an analytics report, but in practice, this is useful anywhere that data flows downstream, including in other data processing jobs.

To start with, our users see just the currently available data. New data is waiting to be processed.

![New data for 18 May 2023 is available. Data for previous day is loaded in the database and shown in downstream use.](https://i0.wp.com/lakefs.io/wp-content/uploads/2023/05/write-audit-publish-01-e1684768735785.png?w=1440&quality=80&ssl=1)

New data for 18 May 2023 is available. Data for previous day is loaded in the database and shown in downstream use.

Now we use Write-Audit-Publish to make the data available in a controlled way. The name of the pattern describes it in a nutshell:

1. *Write* the data that you are processing to somewhere that is not read by consumers downstream. This could be a staging or temporary area, a branch, etc.![New data is loaded into a temporary staging area](https://i0.wp.com/lakefs.io/wp-content/uploads/2023/05/write-audit-publish-02.png?w=1440&quality=80&ssl=1)
	New data is loaded into a temporary staging area
2. *Audit* the data to make sure that it meets the data quality specifications.![Data in the temporary staging area is audited for quality. Users of the published data continue to only see the existing data](https://i0.wp.com/lakefs.io/wp-content/uploads/2023/05/write-audit-publish-03.png?w=1440&quality=80&ssl=1)
	Data in the temporary staging area is audited for quality. Users of the published data continue to only see the existing data
3. *Publish* the data by writing it to the place from which consumers downstream read it.![Following a successful audit, data is published and all downstream users now see it](https://i0.wp.com/lakefs.io/wp-content/uploads/2023/05/write-audit-publish-04.png?w=1440&quality=80&ssl=1)
	Following a successful audit, data is published and all downstream users now see it

“Publishing” is probably the term that caught me most unaware when I started looking at this since it’s not as self-explanatory as “Write” and “Audit”. When we talk about publishing data, it could be something like:

- Inserting data from a staging table into the main table against which users run their queries
- Merging a branch of data into the trunk, on platforms that support it (of this, more later!)
- Flipping a flag in a table so that users querying it now include that data in their results (perhaps using a view to effect this)

I’m going to explore in detail different ways of implementing WAP on various technologies in a subsequent blog.

## Is WAP widely used?????

I asked data folk on various social platforms “ *Do you use the write-audit-publish (WAP) pattern in your pipelines?*”.

I got the most responses on Reddit’s `r/dataengineering` with over a hundred votes, of whom the vast majority hadn’t even heard of WAP, let alone used it.

LinkedIn had the most positive response with nearly a third of respondents using it. You can probably read into these results something about the folk using each platform (as well as the number of people who follow me on each), but in general, even with a small sample size it’s probably fair to summarise that:

- Many people haven’t heard of WAP
- Of those who have heard of it, perhaps half make use of it

### LinkedIn

![28 votes / 29% Yep! / 14% No / 57% What's WAP?](https://i0.wp.com/lakefs.io/wp-content/uploads/2023/05/675a1a14beb199c15a7c96ac8ca2750c76870166.png?w=1440&quality=80&ssl=1)

28 votes / 29% Yep! / 14% No / 57% What's WAP?

### Reddit (r/dataengineering)

![96 votes / 5 Yep / 10 Nope / 96 What even is WAP?](https://i0.wp.com/lakefs.io/wp-content/uploads/2023/05/2d217bda19998b997680eb17c361a9db4ab9e197.png?w=1440&quality=80&ssl=1)

96 votes / 5 Yep / 10 Nope / 96 What even is WAP?

### Mastodon

![8 votes / 0 Yes / 12.5% Nope / 87.5% What's WAP](https://i0.wp.com/lakefs.io/wp-content/uploads/2023/05/ab6f6ace0d8e5f65d49130a22da4b3dda5e79c1f.png?w=1440&quality=80&ssl=1)

8 votes / 0 Yes / 12.5% Nope / 87.5% What's WAP

### Twitter

![20 votes / 10% Yep / 10% Nope / 80% What even is that?](https://i0.wp.com/lakefs.io/wp-content/uploads/2023/05/23e0318d85fe05684fc6a5a251dd8af536c065d0.png?w=1440&quality=80&ssl=1)

20 votes / 10% Yep / 10% Nope / 80% What even is that?

## Why WAP should be more widely used (⚠️ opinions ahead!)

The data engineering world has always lagged behind its software engineering brethren.

Concepts like source control were well established in software engineering for a decade or more before data engineers realised that there might just be something in the idea of not emailing around files called `DIM_DATE_V1_FINAL_REVISED_v2_PROD.sql`. (In fairness, it took a shift away from the old mindset of the established vendors too, in parallel with the emergence of the modern data stack for things to really click).

Write-Audit-Publish is very similar—or perhaps the same, if you squint—to the idea of Blue-Green deployments in the software engineering world (credit to Claus Herther who wrote about this [similarity](https://calogica.com/sql/bigquery/dbt/2020/05/24/dbt-bigquery-blue-green-wap.html)).

Blue-Green deployment was [popularised by Martin Fowler](https://martinfowler.com/bliki/BlueGreenDeployment.html) over ten years ago. In essence, when you deploy something new you don’t go “big bang”, but instead route some or all traffic to the new deployment with the option to flip back at any time.

![A user's making a request to an app server via a router. The router sends traffic to a 'green' set of web/app/db servers, with an alternative set of 'blue' servers.](https://i0.wp.com/lakefs.io/wp-content/uploads/2023/05/blue-green-apps.png?w=1440&quality=80&ssl=1)

A user's making a request to an app server via a router. The router sends traffic to a 'green' set of web/app/db servers, with an alternative set of 'blue' servers.

We can see how that’s very similar to what WAP is giving us but with datasets instead of application servers.

![A user queries data, logically routed from the 'blue' previously-published data](https://i0.wp.com/lakefs.io/wp-content/uploads/2023/05/wap-blue-data.png?w=1440&quality=80&ssl=1)

A user queries data, logically routed from the 'blue' previously-published data

The key thing is that there’s no actual router, but more a logical control that we need ourselves to implement over the data so that the user executes the same query against the same connection and receives the different data once we’re happy with it being released (a.k.a. “published”).

This could be with [branching](https://docs.lakefs.io/quickstart/branch.html). In the example below, the yellow box becomes the “main” trunk branch, and by merging into that the user sees the latest data. It could also take the form of a more manual step such as merging the data in from a staging table.

![A user queries data, logically routed from the 'green' newly-published data](https://i0.wp.com/lakefs.io/wp-content/uploads/2023/05/wap-green-data.png?w=1440&quality=80&ssl=1)

A user queries data, logically routed from the 'green' newly-published data

As data engineers we can learn a lot from established and proven patterns, and the Blue-Green one is a good example of this.

Why *wouldn’t* we want to adopt this, perhaps other than inertia and fear of something new? WAP is a perfect fit for both regular data pipelines as well as one-off data processing jobs.

In the [next article](https://lakefs.io/blog/how-to-implement-write-audit-publish/) in this series, I discuss several common technologies in use today and how WAP can be implemented using them. We’ll finish the series by rolling up our sleeves and looking at [a practical example of WAP in action](https://lakefs.io/blog/write-audit-publish-with-lakefs/). Stay tuned!
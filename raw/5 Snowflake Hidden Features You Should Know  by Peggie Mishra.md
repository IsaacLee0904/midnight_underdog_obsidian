---
title: "5 Snowflake Hidden Features You Should Know | by Peggie Mishra"
source: "https://freedium-mirror.cfd/https://blog.devgenius.io/5-snowflake-hidden-features-you-should-know-f5468f1ce0af"
author:
published:
created: 2026-04-16
description: "Dear Readers,"
tags:
  - "clippings"
---
[< Go to the original](https://peggie7191.medium.com/5-snowflake-hidden-features-you-should-know-f5468f1ce0af#bypass)

![Preview image](https://miro.medium.com/v2/resize:fit:700/1*DPm9gPv65TM8FFLF2P5DsQ.png)

## 5 Snowflake Hidden Features You Should Know

## Dear Readers,

[![Peggie Mishra](https://miro.medium.com/v2/resize:fill:88:88/1*pGEads0HAbyJn_S7YXLXpQ.jpeg)](https://medium.com/@peggie7191 "Storytelling at the crossroads of data, cloud, and modern life—unpacking the tech that shapes how we live, work, and connect.")

[Peggie Mishra](https://medium.com/@peggie7191 "Storytelling at the crossroads of data, cloud, and modern life—unpacking the tech that shapes how we live, work, and connect.")

androidstudio · June 18, 2025 (Updated: June 18, 2025) · Free: No

Snowflake offers many powerful features for data management. While some are well-known, others remain hidden gems that can simplify your queries and boost productivity. Let's dive into five lesser-known Snowflake features.

1. ***Extra Comma After Last Column:***

Unlike many other databases, Snowflake allows you to include an extra comma after the last column in a SELECT statement without throwing an error. This can be helpful when developers accidentally leave an extra comma or plan to add more columns later but forget the comma.

***Eg: Customer\_id has an additional comma, and the query runs fine.***

```sql
SELECT order_id,
       customer_id,
       FROM sales_data
```

***2\. Using Aliases Within the Same SELECT:***

Snowflake lets you reuse column aliases within the same ***SELECT*** clause. This reduces unnecessary lines and helps keep your queries clean and concise

***Eg: The alias SALE\_AMT is reused in a window function.***

```
SELECT REGION,
       ORDER_DATE,
       QUANTITY*UNIT_PRICE AS SALE_AMT,
       SUM(SALE_AMT) OVER (PARTITION BY REGION ORDER BY ORDER_DATE) AS RUNNING_TOTAL
FROM SALES_DATA
ORDER BY REGION,
         ORDER_DATE
```

***3\. Using ILIKE in SELECT:***

The ***ILIKE*** keyword is super useful when you're exploring a table and want to quickly find columns with names that match a pattern especially during debugging or ad hoc analysis.

***Eg: Following returns columns that include "id" in their names***

```sql
SELECT * ILIKE '%id%' FROM sales_data;
```

***4.EXCLUDE in SELECT:***

The ***EXCLUDE*** clause lets you remove specific columns from your ***SELECT\**** result. This is a time-saver when you want to avoid including unnecessary columns like audit timestamps.

***Eg: Exclude the ORDER\_TS column from your results***

```sql
SELECT * EXCLUDE ORDER_TS FROM sales_data;
```

5\. **RENAME in SELECT:**

Snowflake provides **RENAME** feature where a column can be renamed on the fly.

***Eg: This query renames customer\_id to customer\_number***

```
SELECT * RENAME customer_id AS customer_number FROM sales_data;
```

### Bonus Notes:

- ***EXCLUDE*** *can be used along with* ***RENAME****.*
- *You can also combine* ***ILIKE*** *with* ***RENAME****.*
- *These features —* ***EXCLUDE****,* ***ILIKE****, and* ***RENAME*** *—work seamlessly even in select used in* ***JOINS****.*

Ref: [https://docs.snowflake.com/en/sql-reference/sql/select](https://docs.snowflake.com/en/sql-reference/sql/select)

Hope these features help you write faster and better queries in Snowflake.

[#data](https://medium.com/tag/data "Data") [#data-science](https://medium.com/tag/data-science "Data Science") [#data-engineering](https://medium.com/tag/data-engineering "Data Engineering") [#data-analysis](https://medium.com/tag/data-analysis "Data Analysis") [#snowflake](https://medium.com/tag/snowflake "Snowflake")

- ### Discord Translator
	Translate messages in Discord
	[Add To Chrome](https://chromewebstore.google.com/detail/ibipjomdhljdfonmeiemgbbjpilidhne?utm_source=item-share-cb)
- ### WhatsApp Translator
	Translate messages in WhatsApp
	[Add To Chrome](https://chromewebstore.google.com/detail/ekaocdggcoffjhdaaddndakidgonodhe?utm_source=item-share-cb)
- ### Prompt Optimizer
	Optimize your prompts for AI models like ChatGPT, Claude, and Gemini.
	[Add To Chrome](https://chromewebstore.google.com/detail/ohobmnkjljbohbjhnhafkcamclkjjikd?utm_source=item-share-cb)
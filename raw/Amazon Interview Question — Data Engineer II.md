---
title: "Amazon Interview Question — Data Engineer II"
source: "https://towardsdev.com/amazon-interview-question-data-engineer-ii-e525045e1c37?gi=c81fb62eec10&source=email-1ed9e5037db-1769538360650-digest.reader-a648dc4ecb66-e525045e1c37----1-109------------------01ef26a1_5d9e_424c_ab24_62b87c8da812-1"
author:
  - "[[Mohit Daxini]]"
published: 2026-01-08
created: 2026-04-16
description: "Amazon Interview Question — Data Engineer II Valid Triangle Detection using SQL & PySpark Imagine you’re given side lengths of triangles stored in a table. Your task is to identify which …"
tags:
  - "clippings"
---
## Valid Triangle Detection using SQL & PySpark

![](https://miro.medium.com/v2/resize:fit:1400/format:webp/1*jqg8GzGQ02jitTCogVphkA.jpeg)

### Imagine you’re given side lengths of triangles stored in a table. Your task is to identify which combinations can actually form a triangle.

Link for my non-member readers — [Click here](https://medium.com/@mohitdaxini75/amazon-interview-question-data-engineer-ii-e525045e1c37?sk=ad5a451a398df52f3982d04ae5645d1c)

> Sounds easy? Let’s break it down — step by step, like a real data engineer would.👇

### 🔺 Problem Statement (In Simple Words)

You are given a table with:

![](https://miro.medium.com/v2/resize:fit:1400/format:webp/1*xmLqSxVAcV4hXFwlt-RRog.png)

Each CombinationID has exactly 3 rows → sides A, B, C.

🎯 **Goal**  
Return only those CombinationIDs that form a valid triangle.

**📐 Triangle Rule (The Golden Law)**  
A triangle is valid if and only if:  
A + B > C  
A + C > B  
B + C > A  
If any one of these fails → ❌ not a triangle.

📥 **Sample Input**
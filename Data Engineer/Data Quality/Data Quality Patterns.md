所有的以下 data quality solution 都稱為 patterns 是因為如何實現會很取決於採用者的資料架構與使用技術
## Write → Audit → Publish Pattern (WAP)

由 Netflix 的 Michelle Ufford 在 2017 年 DataWorks Summit 的演講[《Whoops the Numbers are wrong! Scaling Data Quality @ Netflix》](https://lakefs.io/blog/data-engineering-patterns-write-audit-publish/) 中分享了 Netflix 內部如何做 data quality check，類似於將軟體工程的藍綠部署(Blue-Green Deployment) 應用於資料工程
![[WAP_workflow|800]]
**Step1. Write**
Write the data that you are processing to somewhere that is not read by consumers downstream. This could be a staging or temporary area, a branch, etc.
![[Pasted image 20260419233740.png]]

**Step2. Audit**
Audit the data EX. not null, within expected ranges etc. to make sure that it meets the data quality specifications.
![[Pasted image 20260419233831.png]]

**Step3. Publish**
Publish the data by writing it to the place from which consumers downstream read it.
![[Pasted image 20260419233948.png]]
## Audit → Write → Audit → Publish Pattern (AWAP)

## Transform → Audit → Publish (TAP)

## Signal Table Pattern

## Two-Phase WAP / Fronting Kafka Pattern
## Dead-Letter Queue (DLQ) Pattern
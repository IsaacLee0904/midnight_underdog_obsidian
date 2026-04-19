所有的以下 data quality solution 都稱為 patterns 是因為如何實現會很取決於採用者的資料架構與使用技術
## Write → Audit → Publish Pattern (WAP)

由 Netflix 的 Michelle Ufford 在 2017 年 DataWorks Summit 的演講[《Whoops the Numbers are wrong! Scaling Data Quality @ Netflix》](https://lakefs.io/blog/data-engineering-patterns-write-audit-publish/) 中分享了 Netflix 內部如何做 data quality check，類似於將軟體工程的藍綠部署(Blue-Green Deployment) 應用於資料工程
![[WAP_workflow|800]]
#### Pattern
**Step1. Write**
將處理好的資料寫入下游使用者無法存取的暫存區或資料分支中
![[Pasted image 20260419233740.png]]

**Step2. Audit**
針對暫存區執行品質測試與驗證，EX. 檢查 Null 值、資料範圍、列數是否異常
![[Pasted image 20260419233831.png]]

**Step3. Publish**
若測試通過，將資料合併或切換到生產表供下游使用；若失敗，則將資料隔離並發出警告，避免污染生產環境，其中有三種 publish 的方法：
![[Pasted image 20260419233948.png]]
 1. 將資料從 staging table insert 到使用者可以查詢到的 main tabe
 2. 
 Publishing data could be something like:
- Inserting data from a staging table into the main table against which users run their queries
- Merging a branch of data into the trunk, on platforms that support it (of this, more later!)
- Flipping a flag in a table so that users querying it now include that data in their results (perhaps using a view to effect this)
#### Reference
* [[Data Engineering Patterns Write-Audit-Publish (WAP)]]

## Audit → Write → Audit → Publish Pattern (AWAP)

## Transform → Audit → Publish (TAP)

## Signal Table Pattern

## Two-Phase WAP / Fronting Kafka Pattern
## Dead-Letter Queue (DLQ) Pattern
所有的以下 data quality solution 都稱為 patterns 是因為如何實現會很取決於採用者的資料架構與使用技術
## Write → Audit → Publish Pattern (WAP)

由 Netflix 的 Michelle Ufford 在 2017 年 DataWorks Summit 的演講[《Whoops the Numbers are wrong! Scaling Data Quality @ Netflix》](https://lakefs.io/blog/data-engineering-patterns-write-audit-publish/) 中分享了 Netflix 內部如何做 data quality check，類似於將軟體工程的藍綠部署 (Blue-Green Deployment) 應用於資料工程
![[WAP_workflow|800]]
#### Pattern
**Step1. Write**
在 write 階段，資料可以從各種來源 extract，也可以透過 Spark、Kafka 等工具將資料進行轉換，將處理好的資料寫入下游使用者無法存取的暫存區或資料分支中
![[Pasted image 20260419233740.png]]

**Step2. Audit**
資料到達 staging 之後，會針對暫存區執行[[What Does Validate Mean|品質測試與驗證]]，EX. 檢查 Null 值、資料範圍、列數是否異常 比較好的做法是用一系列已經預先定義好的規則來確保完整性，data engineer 可能就會透過 python script 做以下檢查：
* 根據是先定義好的 schema 來檢查資料結構是否符合預期
* 利用統計方法做 [[What Is Anomaly Detection|anomaly detection]] 來檢測 outliers 或意外的資料週期
* 
![[Pasted image 20260419233831.png]]

**Step3. Publish**
若測試通過，將資料合併或切換到生產表供下游使用；若失敗，則將資料隔離並發出警告，避免污染生產環境，其中有三種 publish 的方法：
![[Pasted image 20260419233948.png]]
 1. 將資料從 <font color="#ff0000">staging table</font> insert 到使用者可以查詢到的 main table
 2. 利用支援 <font color="#ff0000">data branching 的工具</font> EX. lakeFS 將帶有新資料的 branch merge 進到 main
 3. 透過重新指定 <font color="#ff0000">view</font> 的方式讓資料可以被使用者看到

#### Pros and Cons

<mark style="background:#fff88f">Pros</mark>
1. Enhanced Data Integrity and Quality (增加資料正確性與品質)：audit 階段會仔細檢查資料的正確性、完整性與是否符合預定標準，適時地修正任何差異與異常狀況
2. Increased Data Security (增強資料安全性)：透過 WAP 得結構隔離原始資料與審核資料，<font color="#ff0000">保護敏感資料在未經驗證的時候洩漏</font> -> 牽涉的資料合規性
3. Improved Reliability (可靠性提升)：WAP 的多階段分離使得錯誤處理和復原機制更加完善，有助於 debug，從而增強 pipeline 的可靠性
4. Trust issue：確保下游 consumer 只看到通過驗證的資料，避免壞資料進入生產後才被發現、撤回的信任損失  -> 這很重要因為牽涉到 user 的信任問題
5. Operational Flexibility and Scalability (靈活性與可擴展性)：<font color="#ff0000">模組化</font>設計讓 Write、Audit、Publish 三個階段可以獨立調整與擴展

<mark style="background:#fff88f">Cons</mark>
1. 多步驟流程增加了資料延遲
2. 需要更複雜的 orchestration
3. 沒有真正的跨表 transaction 保證，若 pipeline 涉及多張表，publish 實際上是 best effort 而非 all-or-nothing

#### Reference
* [[Data Engineering Patterns Write-Audit-Publish (WAP)]]
* [[Write-Audit-Publish Pattern in Pipelines]]
* [[Write-Audit-Publish Safe Data Pipelines with Git-for-Data]]

## Audit → Write → Audit → Publish Pattern (AWAP)

## Transform → Audit → Publish (TAP)

## Signal Table Pattern

## Two-Phase WAP / Fronting Kafka Pattern
## Dead-Letter Queue (DLQ) Pattern
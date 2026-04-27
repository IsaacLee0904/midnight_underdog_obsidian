所有的以下 data quality solution 都稱為 patterns 是因為如何實現會很取決於採用者的資料架構與使用技術，正因如此沒有絕對「最好的」pattern，如何選擇則是取決於 type of quality issue, use case, data size, platform limitation, SLAs ...，但無論如何目的只有一個「Keep bad data out of production」，盡可能的在影響下游或損害 stakeholder 信任之前攔截問題
## Write → Audit → Publish Pattern (WAP)

由 Netflix 的 Michelle Ufford 在 2017 年 DataWorks Summit 的演講[《Whoops the Numbers are wrong! Scaling Data Quality @ Netflix》](https://lakefs.io/blog/data-engineering-patterns-write-audit-publish/) 中分享了 Netflix 內部如何做 data quality check，類似於將軟體工程的藍綠部署 (Blue-Green Deployment) 應用於資料工程
![[WAP_workflow|800]]
#### Pattern
**Step1. Write**
在 write 階段，資料可以從各種來源 extract，也可以透過 Spark、Kafka 等工具將資料進行轉換，將處理好的資料寫入下游使用者無法存取的暫存區 EX. log, staging table 或 資料分支 EX. [[Write-Audit-Publish Safe Data Pipelines with Git-for-Data|Bauplan]], LakeFS

![[Pasted image 20260419233740.png]]

**Step2. Audit**
資料到達 staging 之後，會針對暫存區執行[[What Does Validate Mean|品質測試與驗證]]，EX. 檢查 Null 值、資料範圍、列數是否異常 比較好的做法是用一系列已經預先定義好的規則來確保完整性，data engineer 可能就會透過 python script 做以下檢查：
* 根據是先定義好的 schema 來檢查資料結構是否符合預期
* 利用統計方法做 [[Anomaly Detection|anomaly detection]] 來檢測 outliers 或意外的資料週期
* 確保沒有 duplicate、NULL 或是格式錯誤的資料
* 驗證是否符合業務規則 EX. 產品價格不能是負數、age 不會有很奇怪的年齡 etc. 
> 這邊的 staging 與過去 Kimball 提出的 staging layer 差異在於，WAP 所認知的 staging 是一個<font color="#ff0000">明確的環境邊界</font>，用來保護避免 prod 的資料受到影響

![[Pasted image 20260419233831.png]]

**Step3. Publish**
若測試通過，將資料合併或切換到生產表供下游使用；若失敗，則將資料隔離並發出警告，避免污染生產環境，其中有三種 publish 的方法：
![[Pasted image 20260419233948.png]]
 1. 將資料從 <font color="#ff0000">staging table</font> insert 到使用者可以查詢到的 main table
 2. 利用支援 <font color="#ff0000">data branching 的工具</font> EX. lakeFS 將帶有新資料的 branch merge 進到 main
 3. 透過重新指定 <font color="#ff0000">view</font> 的方式讓資料可以被使用者看到
#### Two Implementation patterns of WAP

<mark style="background:#fff88f"> Two-Phase WAP in batching processing</mark>
傳統的方法，需要兩個實體的資料副本 table，並經歷 write、audit、publish 的步驟，publish 後清除 staging data，這種方法基本上沒有 infra 的限制，但涉及額外的儲存成本與複製成本
![[Pasted image 20260425150107.png]]

<mark style="background:#fff88f"> Two-Phase WAP in streaming processing</mark>
在 streaming data 中 WAP 依循前置佇列 (fronting queue) 模試，又稱為<font color="#ff0000"> Fronting Kafka pattern</font>，採用雙叢集 (two-cluster) 架構：
1. 類似 staging 環境負責接收 fronting Kafka 所有 event，不進行驗證
2. streaming consumer 通常是 Flink 或 Spark 的 streaming 處理框架實作從 fronting Kafka 消化 event 並執行 data contract 的驗證
EX. Netflix 的 [data mesh](https://netflixtechblog.com/data-mesh-a-data-movement-and-processing-platform-netflix-1288bcab2873)

![[Pasted image 20260426020405.png]]


<mark style="background:#fff88f"> One-Phase WAP in batching processing (Zero-Copy WAP) </mark>
在 data lakehouse 透過 Iceberg、Hudi 等 table format 技術讓 WAP 的過程<font color="#ff0000">不需要真的透過複製</font>來完成，以 Iceberg 為例，只需要透過 `write.wap.enabled` 和 `wap.id` 就能啟用 WAP，在 write 與 audit 階段在隔離的 branch 跟 snapshot 執行，publish 階段就只需要提交 metadata，Iceberg 的步驟如下：

Step1. Set `write.wap.enabled=true` in the <font color="#ff0000">table</font>
Step2. `Set spark.wap.id=<UUID>` in the <font color="#ff0000">Spark job </font>

![[Pasted image 20260425150421.png]]

<mark style="background:#fff88f"> One-Phase WAP in streaming processing (Zero-Copy WAP) </mark>
Streaming pipeline 中的 One-Phase WAP 減少了對中介佇列的需球，採用[訊息路由模式 (message router pattern)](https://www.enterpriseintegrationpatterns.com/patterns/messaging/MessageRouter.html) 的事件路由模式 (event router pattern)，這種做法有幾種特徵：
1. event router 能夠將相同的事件從一個源廣播到多個目的地 (one-to-many)
2. event router 通常不會修改事件的內容 (payload)
3. event router 可以在事件的信封 (envelope) 上附加額外的 metadata

![[Pasted image 20260427161721.png]]

#### Write-audit-publish Implementation

<mark style="background:#fff88f">DIY Approach Using Pandas </mark>
這種方式是最彈性最客製化的方案，基本上相容於各種 compute engine，並且能對驗證邏輯有完全的掌控，透過 panda、spark 等 DataFrame 框架能夠實現靈活的記憶體內檢查，然而缺點就是資料需要兩次的寫入 (staging、prod)，因此增加了運算與儲存成本，在處理大型資料的時候可能有 OOM 風險，同時如果要對多張 table 發布時會缺少交易性保證 (transactional guarantees)

**Write**
![[Pasted image 20260427164901.png]]

**Audit**
![[Pasted image 20260427164908.png]]

**Publish**
![[Pasted image 20260427164957.png]]

<mark style="background:#fff88f">Snowflake zero-copy clones with dbt </mark>

![[Pasted image 20260427174445.png]]
Snowflake 透過 zero-copy clones 支援 WAP 模式，讓 pipeline 能夠即時建立 table 或 schema 的完整副本，因此無需複製儲存資料

**Write**
* 將所有資料 load 到 dev 或 staging schema EX.`RAW_WAP`
* 在此 sandbox schema 中執行 dbt model 載入資料

**Audit**
* 針對 clone 的資料執行 dbt test 或 snowflake 的 DQ check
* 在 test 全部通過之前不會有任何資料留到 Prod

**Publish**
* 若驗證通過，使用 zero-copy clone 將 Staging 資料表提升至 Prod Schema
* 由於是指改動了 metadata pointer，因此變換幾乎是即時完成的

這種方式確保 Prod 的資料在驗證過程中不會遭到污染，提供了一個可使用完整資料集安全測試pipeline 的環境，同時也允許在發布變更前進行人工調整與重新測試

然而，此方法仍有幾項缺點：<font color="#ff0000">需要維護客製化的調度程式碼來管理 clone 與 test 流程</font>、<font color="#ff0000">增加 pipeline 的複雜度</font>，且在執行轉換與測試時，仍會消耗額外的運算與儲存資源

<mark style="background:#fff88f">Apache Iceberg</mark>

在 Iceberg 中使用了一種 分支機制 (branching mechanism)，類似於對 data 使用了 git，使得 WAP 實作

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
* [Write-Audit-Publish Pattern in Modern Data Pipelines](https://www.youtube.com/watch?v=-omlizzCALc)

## Audit → Write → Audit → Publish Pattern (AWAP)

由 Bartosz Konieczny 在其著作《Data Engineering Design Patterns》中提出，作為 WAP 的優化版

#### Reference
* [[Data Quality Design Patterns]]

## Transform → Audit → Publish (TAP)

## Signal Table Pattern

## Two-Phase WAP / Fronting Kafka Pattern
## Dead-Letter Queue (DLQ) Pattern
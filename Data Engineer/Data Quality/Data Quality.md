tags: #data-quality-check #etl #data-engineering
source: [DataExport](https://www.youtube.com/watch?v=JiedBnTFCeg&list=PLwUdL9DpGWU0lhwp3WCxRsb1385KFTLYE&index=13)

---
* How to build high trust in the data sets that you build
* How to build good data docs
* Data quality checks and how thet diff between facts and dims 
## Data Quality Basic

>Data quality refers to the development and implementation of activities that apply quality management techniques to data in order to ensure the data is fit to serve the specific needs of an organization in a particular context.

資料品質 (Data Quality) 是指資料的準確性、一致性、完整性和可靠性，高品質的資料能夠讓企業基於準確資訊做出有效的決策，而資料品質包含數個面向：

* <mark style="background: #BBFABBA6;">可發現性 (Discoverability)</mark>
  指的是當 user 想要做決策，他們能輕鬆知道資料是否存在並且要去哪裡取用它 -> 應該算是定義更嚴格的的資料可用性 (Data Availability)
  
* <mark style="background: #BBFABBA6;">資料的完整性</mark>
	* overview level：主要是關於整體資料的誤解與不完整定義 EX. Zillow 曾經以為他們有完美的模型可以預測房地產，結果因為沒有考慮到模型遺漏了哪些資料點，導致損失
	* table level：指的是單一的 table 有沒有全部欄位都同步到然後歷史資料也沒有缺漏，同時也包含不應該有 NULL 的欄位就不能有 NULL 值，然後不應該有 duplicate

* <mark style="background: #BBFABBA6;">資料的易用性</mark>
  table_name 跟 col_name 需要合理且明顯 EX. Airbnb 用 `m_` 來命名 metrics 相關的欄位、`dim_` 來密名維度欄位 同時也體現在使用者是否好找到這些資料 -> [[Data Quality#Data Spec and Data trust]]

* <mark style="background: #BBFABBA6;">指標定義一致性</mark>
  從不同 data pipeline 取得的指標定義要一致，最好符合 single source of truth 可以做 data pipeline 的整合，避免 DA 跟業務端計算 KPI 邏輯不一樣的狀況

* <mark style="background: #BBFABBA6;">來源可靠性</mark>
  資料品質會受到上游資料品質的疊加影響，要確保來源的可靠性，要仰賴資料血緣 (data lineage) 的追蹤，包含表和表之間的關聯 (table lineage)，以及欄位和欄位之間 (column linage) 的繼承關係

* <mark style="background: #BBFABBA6;">資料新鮮度 (Data Freshness)</mark>
  指資料的更新頻率和實效性，透過任務編排的執行頻率，以及資料倉儲對資料源歷程變化的記載方式共同決定，並且是 DE 與 user 之間的共識，可以減少收到無謂的需求

* <mark style="background: #BBFABBA6;">商業價值</mark>
  每一條你寫的資料管道要麼能產生更多收入，要麼能節省成本 EX. 測量在 AWS 上的花費 唯一的例外是提供戰略價值的資料集，這類資料主要用於重大高層決策 -> Metabase 上有很多用不到的 Dashboard

> 資料品質 = 資料信任 (data trust) + 資料影響力

## Data Spec and Data trust

理想狀況，在開發一個 data pipeline 之前，需要把所有的 downstream stakeholder 聚在一起然後做一些需求訪談，確保可以更清楚他們試圖想解決的問題，<span style="color:rgb(255, 0, 0)">額外需要思考「目前和未來的需求」，不要只為了一個單一問題去建立 pipeline，才能建立一個既可以回答他們今天問題，也能回答未來六個月問題的 data model</span>，最後產出一個規格書 (data spec) 並且被其他人 review EX. Airbnb 需要另外給 staff data engineer 來 reivew 

> [!HINT] **Business Impact 對 promotion 的影響**
> 延續上面說的要去釐清 stakeholder 現在和未來的需求，就是一種釐清商業影響力的表現，而這很重要是會很大的影響了個人的 performance reviews
> 
> EX. 可能會有 stakeholder 說他想要知道 2023 年巴西因為舉辦某個活動 Airbnb 房源出租狀況
> 千萬不要直接開始做，因為這麼 narrow down 的需求會反覆的出現

### Airbnb MIDAS process

在 Airbnb 當需要建立重要 data pipeline 的時候 (很重要會提供不少價值並且不會隨時變動)，有一個建立 data pipeline 的標準流程 [MIDAS](https://medium.com/airbnb-engineering/data-quality-at-airbnb-e582465f3ef7) 總共會有 9 個步驟：

![[Screenshot 2026-04-01 at 4.25.32 PM.png]]

Step1. Make a spec：說明要建立的 data pipeline 與做法

Step2. Spec review：需要給資料架構師、staff 以及 stakeholder review

Step3. Build and backfill：撰寫 Spark, Airflow code -> 但只 backfill 一個月的資料就好

Step4. SQL validation：由 DA 來驗證這一個月的資料是不是正確

Step5. Metrics validation：建立新的 metrics ( Minerva 是 Airbnb 存放 metrics 的 repo )

Step6. Data and code review：架構師會進來檢查 data model 或 code，確定 unit test 與 
	integration test
	
Step7. Metics migration

Step8. Final review：由 staff DS review

Step9. Launch PSA：宣布 dataset 正式上線

> [!note]  在建立 data pipeline 之前先把 stakeholder 拉進來，他們會覺得自己也與這個 pipeline 有關，能有效得減少未來的指責並且更早突顯出溝通問題

### Good Design Spec

一份好的規格書應該要包含 pipeline description、flow diagrams (from source to warehouse, modeling, metics, dashboard)、schemas (DDL of create table statements 然後應該要包含 `_fct`, `_dim`, `scd`, `agg` & column comments)、quality checks、metics definitions、example queries

![[Screenshot 2026-04-01 at 5.53.00 PM.png]]
<span style="color:rgb(184, 191, 193)">A example of good flow diagram </span>





[[Data Quality Patterns]]

## Reference
[《資料與程式碼的交鋒》Day 24 — 資料需求金字塔](https://shu-ting.medium.com/data-feat-programming-day-24-5f691450323f)

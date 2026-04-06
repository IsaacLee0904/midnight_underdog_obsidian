tags: #data-quality-check #etl #data-engineering
source: [DataExport](https://www.youtube.com/watch?v=JiedBnTFCeg&list=PLwUdL9DpGWU0lhwp3WCxRsb1385KFTLYE&index=13)

---
* How to build high trust in the data sets that you build
* How to build good data docs
* Data quality checks and how thet diff between facts and dims 
## Data Quality

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

### Three Type of Quality Check

#### Basic checks
例如確認有沒有資料、不該有 NULL、不應該有重複值以及 enums 枚舉所有可能的值
#### Intermediate checks
例如週與週的行數比較。一定要比對週與週，因為星期六和星期五的行為不同，但這種檢查在聖誕節等假日經常會因為季節性而失敗
#### Advanced checks
引入機器學習等技術，建立經季節性調整的行數檢查，減少品質檢查的誤報率
>[!cite] 在 Airbnb 這類的 ML DQC 甚至只是一行 yaml 的 code

### How to Decide Quality Check

#### Dimensional table
1. Grow or are flat day-over-day：維度表通常每天增長或持平，常見檢查像是表格正在成長
2. Don't grow sharply：百分比的變化不應該很劇烈
3. Complex relationship should be check EX. Facebook 好友不能超過 5000 人

#### Fact table
1. Row count：因為 fact table 會有<span style="color:rgb(255, 0, 0)">季節性因素</span>所以不要做 day-over-day 的檢查，可能<span style="color:rgb(255, 0, 0)">要做跨週、跨季甚至跨年的比較</span>
2. grow sharply：fact table 是有可能劇烈增長的
3. duplicate：更容易出現重複值 EX. 所以在 presto 可能要用 `approx_count_distinct`

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

#### Title 
首先會需要一個好的標題，明確的知道這個 pipeline spec 是什麼，並且清楚的描述這個 pipeline 做什麼、解決了什麼問題

```text
EcZachly Inc is a new startup that wants to messure its website traffic and user growth.

The goal of this pipeline is to answer the following questions:

- How many people are going to web_link and web_link on a daily basis ?
  - What is the geo and device break down of that traffic ?
  - Where are these people coming from ? Linkedin ? Youtube ?

- How many people are signing up with an account each day ?
  - Whar percentage of traffic is converting to signing up ?

- How many people are pyrchasing boot camps and course ?
	- What percentage of signups convert to paying ?
```

#### Business Metrics
1. 最好以<span style="color:rgb(255, 0, 0)">表格</span>的形式展現，包含 metric name、definition、is guardrail 等欄位
2. [護欄指標 (guardrail metrics)](https://www.statsig.com/blog/what-are-guardrail-metrics-in-ab-tests)：是<span style="color:rgb(255, 0, 0)">一種用來保護業務的衡量指標，當它發生顯著變化時，能像團隊發出「業務出現問題」的強烈信號</span>，可以想像成是避免翻車的安全網或護欄，目標是為了確保在某個領域的獲利，不會導致另一個冷遇的損失

> 一般來說我們會有 primary metrics 用來測量想積極推動的目標 EX. 新 UX 的 hit rate，而 guardrail metrics 則是負責守護產品整體健康的指標 EX. 頁面載入時間、使用者錯誤率
> - 如何選擇合適的 guardrail metrics ？
> 	- 依據核心業務：可以將<span style="color:rgb(255, 0, 0)">影響公司命脈的指標納入</span> EX. 無論進行什麼測試，都監控「使用者流失率 (churn)」或「工作階段時長 (session duration)」
> 	- **依據潛在的技術災難**：<span style="color:rgb(255, 0, 0)">預測這個功能上線可能會搞砸什麼</span>。經典的技術護欄指標包含：頁面載入時間變慢、使用者重試次數增加、或是 Bug 回報率飆升
> - 使用護欄指標的兩大限制與陷阱
> 	- **雜訊風險 (Risk of noise)**：護欄指標絕對<span style="color:rgb(255, 0, 0)">不是越多越好</span>。在 A/B 測試中，加入越多的指標，就越容易因為隨機機率而產生「統計結果看似顯著，但其實只是雜訊」的狀況。過多的指標反而會造成數據超載，干擾你做出明確決策 >> <span style="color:rgb(184, 191, 193)">就像統計模型丟一堆變數進去就會顯著</span>
> 	- **靈敏度不足 (Sensitivity)**：你的 A/B 測試樣本數通常是為了「主要指標」量身打造的 EX. 足以測量出 1% 的微幅提升。但是，<span style="color:rgb(255, 0, 0)">護欄指標的資料分佈可能完全不同</span><span style="color:rgb(255, 0, 0)">，這意味著護欄指標可能要發生 10%、20% 甚至 30% 的崩跌時，統計學上才抓得出來</span>。因此，你必須在事前根據業務邏輯定出「究竟多糟才算糟」的容忍閾值 EX. 網頁因為新功能變慢了 1%，這個代價是否值得？
> - guardrail metrics 的例子
> 	- Airbnb：在測試為了「增加訂房量」（主要指標）的改動時，他們會將「房客滿意度分數」設為護欄指標，確保業務成長不會犧牲客戶的滿意度
> 	- Netflix：在測試新的個人化推薦演算法時，他們的護欄指標是「影片開始播放的載入時間」與「緩衝比例」，確保推薦系統升級的同時，核心的「流暢觀影體驗」沒有受到損害
> 	- Uber：為了提升「行程轉換率」推出新配對演算法時，護欄指標是「乘客總發送請求數」**，這是為了確保轉換率的提升，不是因為系統把總請求數量砍掉而製造出來的數學假象

| Metrics Name             | Definition                            | Is Guardrail |
| ------------------------ | ------------------------------------- | ------------ |
| signup_conversion_rate   | COUNT(signups) / COUNT(website_hits)  | yes          |
| purchase_conversoin_rate | COUNT(purchases) / COUNT(signups)     | yes          |
| traffic_breakdown        | COUNT(website_hits) GROUP BY referrer | no           |
* `signup_conversion_rate` 是 guardrail 是因為如果下降代表登入頁面 (landing page) 有問題
* `purchase_conversion_rate` 是 guardrail 是因為如果下降代表結帳頁面 (checkout page) 有問題
* `traffic_breakdown` 不是 guardrail 是因為可能會在網站上做的 AB 測試，並不會影響流量是來自 LinkedIn 還是 Twitter
#### Flow Diagram
![[Screenshot 2026-04-06 at 8.12.12 PM.png]]
* IP Enrichment：那層用來處理原始資料，將 IP 地址傳送到某個 API，來查詢這個 IP 是哪裡的 IP
* User Agent Enrichment：用來取得設備的品牌資訊

#### Schema

<mark style="background: #BBFABBA6;">core.fct_website_events</mark>
* 這張表包含了 Exactly.com 的所有事件，並帶有地理與設備資料
* unique identifier 為 logged_out_user_id + event_timestamp

| col name           | col type              | col comment                                                                                                                                                             |
| ------------------ | --------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| user_id            | `BIGINT`              | This col is nullable for logged out events.<br>This col indicates the user who generated this event.                                                                    |
| logged_out_user_id | `BIGINT`              | This col is a hash of IP address and device information.                                                                                                                |
| dim_hostname       | `STRING`              | What is the host associated with this event (eczachly.com, zachwilson.tech etc)                                                                                         |
| dim_country        | `STRING`              | The country associated with the IP address of the request.                                                                                                              |
| dim_device_brand   | `STRING`              | The device brand associated with this request.                                                                                                                          |
| dim_action_type    | `STRING`              | This is an <span style="color:rgb(255, 0, 0)">enumerated list </span>of actions that a user could take on this website EX. signup, watch video, go to landing page etc. |
| event_timestamp    | `TIMESTAMP`           | The <span style="color:rgb(255, 0, 0)">UTC</span> timestamp for when this event occured.                                                                                |
| other_properties   | `MAP[STRING, STRING]` | Any other valid properties that are part of this request.                                                                                                               |
| ds                 | `STRING`              | This is the partition col for this table                                                                                                                                |

<mark style="background: #BBFABBA6;">core.agg_website_events</mark>
* 這是所有 website events 的聚合 view
* 

| col name           | col type              | col comment                                                                                                                                                             |
| ------------------ | --------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| user_id            | `BIGINT`              | This col is nullable for logged out events.<br>This col indicates the user who generated this event.                                                                    |
| logged_out_user_id | `BIGINT`              | This col is a hash of IP address and device information.                                                                                                                |
| dim_hostname       | `STRING`              | What is the host associated with this event (eczachly.com, zachwilson.tech etc)                                                                                         |
| dim_country        | `STRING`              | The country associated with the IP address of the request.                                                                                                              |
| dim_device_brand   | `STRING`              | The device brand associated with this request.                                                                                                                          |
| dim_action_type    | `STRING`              | This is an <span style="color:rgb(255, 0, 0)">enumerated list </span>of actions that a user could take on this website EX. signup, watch video, go to landing page etc. |
| event_timestamp    | `TIMESTAMP`           | The <span style="color:rgb(255, 0, 0)">UTC</span> timestamp for when this event occured.                                                                                |
| other_properties   | `MAP[STRING, STRING]` | Any other valid properties that are part of this request.                                                                                                               |
| ds                 | `STRING`              | This is the partition col for this table                                                                                                                                |
## Reference
[《資料與程式碼的交鋒》Day 24 — 資料需求金字塔](https://shu-ting.medium.com/data-feat-programming-day-24-5f691450323f)

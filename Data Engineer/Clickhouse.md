# Clickhouse

tags: #clickhouse #olap #database #data-engineering
source: [[Isaac's Note]]

---


#  ClickHouse Introduction
ClickHouse 是一個列為導向 (column-oriented) 儲存的資料庫系統 (database management system, DBMS)，並專為 OLAP (online analytical processing) 線上分析處理的查詢設計，支援基於 SQL 的聲明式查詢語言，在許多情況下與 ANSI SQL 標準相同
### ClickHouse Architecture


ClickHouse 的整體設計邏輯非常清晰：以高效能讀取為核心，透過分散式架構與儲存最佳化，讓秒級查詢在 PB 級數據中成為可能。 從資料寫入、儲存、索引到查詢回傳，ClickHouse 有著一套完全為 OLAP 場景最佳化的底層架構
```markdown
資料寫入 → 拆分成 Data Parts → Partition 劃分 → Primary Key 排序 → 壓縮 → Merge → 索引裁剪 → 向量化查詢 → 回傳結果
```
Step1. 排序 : 欄 (rows) 按照 key 排序 ex.town, street
Step2. 拆分 : 排序的資料再切分成不同行 (cols)
Step3. 壓縮 : 壓縮每一行 (cols)
Step4. 寫入硬碟


ClickHouse 的架構分為三個主要層
1. 查詢處理層 (Query processing layers) : 支援 SQL, PRQL, KQL
2. 儲存層 (Storage layer) : 由不同 table engines 組成，封裝了表資料的格式和位置，分為
1. MergeTree table engines : 主要的持久化格式，基於 LSM tree 將 table 水平分割
2. Special-purpose table engines : 用於加速或分散查詢執行 ex. distributed, external data, buffer, view, materializedview …
3. Virtual table engines : 用於與外部資料交換 ex. PostgreSQL, MySQL 訂閱系統 ex. Kafka, RabbitMQ 資料湖 ex. Iceberg, DeltaLake, Hudi 或 storage 中的檔案 ex. S3, GCS

1. 集成層 (Integration layer)
1. 推送式 : 透過 ETL 工具
2. 拉取式 : 直接連接外部資源


**OLAP vs OLTP**


| 分類
| OLTP (Online Transaction Processing)
| OLAP (Online Analytical Processing)
|


| 主要用途
| 交易處理 (CRUD 操作)，ACID 交易完整性
| 數據分析、統計報表
|


| 操作特性
| 少量資料的頻繁寫入
| 大量資料的批次查詢
|


| 查詢型態
| 單筆/少量記錄查詢
| 大範圍聚合查詢 (Aggregation)
|


| 儲存結構
| 行式存儲 (Row-based)
| 列式存儲 (Column-based)
|


| 代表產品
| MySQL, PostgreSQL, Oracle
| ClickHouse, Druid, Redshift
|


## ClickHouse Core Features


| **ClickHouse 技術特性**
| **說明**
|


| Columnar Storage
| 只讀取需要的欄位，避免不必要的 I/O
|


| Vectorized Execution
| 將資料轉成 SIMD 批次處理，加速 CPU 運算效率
|


| Compression
| 各種編碼方式 (LZ4, ZSTD, Delta Encoding) 提供高壓縮比，降低儲存成本
|


| Data Skipping Indexes
| 不需掃描全部資料，可根據索引直接跳過不相關的數據區塊，查詢更快
|


| MergeTree 儲存引擎
| 強大靈活的底層結構，支援分區、排序鍵、TTL 清理機制，適合大量數據分析
|


| Materialized Views
| 可將複雜查詢結果預先計算並實時更新，大幅加快查詢速度
|


| 分布式架構
| 支援 Sharding 與 Replica ，易於擴展到 PB 級數據處理規模
|


| Near-Real-Time Ingestion
| 支援高吞吐量寫入 (如 Kafka Stream)，數據可秒級查詢分析
|


### **列式儲存 (column-oriented)**
數據按列存儲，提升分析查詢效能
> **Why Column-Oriented Database work well as OLAP**
**Input / Output 討論**
- 對於一個分析的資料查詢來說，只會有一小部分的資料表欄位被讀取到，在列式導向儲存的資料庫中，我們可以只讀取我們需要的資料出來即可
ex. 如果我們需要在 100 的欄位中讀取 5 個欄位出來，則我們可以節省約 20 倍的硬碟的 I/O 量
- 自從資料從封包中進行讀取出來，這將會較容易進行資料壓縮，資料在列中也較容易進行壓縮，而且這可以更進一步的解少 I/O 的大小
- 因為節省了 I/O 大小，因此較多資料可以儲存在系統快取中
**優點**
- **查詢效能極佳（OLAP 場景）**：例如只需統計使用者年齡分佈時，系統僅讀取 Age 欄位，不需讀取 Name、Address 等無關欄位
- **壓縮率高**：同一欄位的資料型態一致，重複值多，能進行更有效率的編碼與壓縮（如 Run-Length Encoding、Delta Encoding）
- **向量化運算加速查詢**：資料欄位緊密排列，使得 CPU 可針對欄位資料進行 SIMD 向量化批次運算，加速查詢執行
**缺點**
- **單筆查詢效率低**：若查詢目標是完整一筆記錄，需從多個欄位位置讀取資料並組裝，延遲較高
- **寫入頻繁場景不適用**：寫入與修改操作成本較高，適合「讀多寫少」的應用情境
**Row-based 與 Column-based 適用場景總結**


| 類型
| Row-based database
| Column-based database
|


| 典型應用
| OLTP 系統（訂單交易、會員登入）
| OLAP 系統（數據分析、報表、指標監控）
|


| 查詢類型
| 單筆查詢 (主鍵查詢、多次頻繁寫入)
| 批次查詢 (聚合分析、大量讀取特定欄位)
|


| 資料寫入頻率
| 高頻寫入、隨時修改
| 寫入頻率低，以批次/流式匯入為主
|


| 查詢效率
| 單筆查詢延遲低、批次查詢效率差
| 單筆查詢較慢、批次查詢極速
|


| 代表資料庫
| MySQL、PostgreSQL、Oracle
| MySQL、PostgreSQL、Oracle
|


### **高壓縮比**
默認使用 [LZ4 算法](https://zh.wikipedia.org/zh-tw/LZ4) 壓縮，通過列式存儲和先進壓縮算法實現
\>\> 如果想要查詢速度變快，最簡單有效的方法就是減少資料掃瞄範圍與傳輸時的大小，在 ClickHouse 中列式儲存與數據壓縮實現了這兩點
> **為什麼壓縮對 OLAP 效能如此重要 ？**
在 OLAP 場景中，資料量動輒百萬、千萬筆，若未經良好壓縮處理，磁碟 I/O 將成為查詢效率瓶頸，ClickHouse 採用 Columnar Storage，讓每欄資料型態一致、重複性高，使得壓縮成效極佳
### **向量化執行**
可以簡單理解成消除程式中循環的優化(用 n 台果汁機只執行一次)，為了實現向量化執行，需要使用 CPU 的 single instructino multiple data (SIMD) 指令加速查詢處理，即是用單條指令操作多條數據，通過並行提高性能的一種實現方式

### **多主架構**
ClickHouse 與常見的 Master-Slave 分散式主從架構不同，採用了 Multi-Master 的多主架構，支援水平擴展和叢集部署，客戶端訪問任意一個節點都能得到一樣的效果，而這樣的架構也帶來系統架構簡單、集群所有節點功能一樣、規避單點故障問題等優點
### **分散式與多線程**
ClickHouse 的設計既利用多線程原理做到 分區 (Partition) 提升了縱向擴展可能性，也利用分散式原理做到 分片 (Shared) 提高橫向擴展


> **Partition (表分區)**在 ClickHouse 中 table 得資料可以按照指定的分區儲存，每個 partition 在 filesystem 都是以不同的 folder 形式存在，查詢時透過 `WHERE` 作為條件，可以有效的過濾掉大量的數據，具有以下特點 :1.  透過 `PARTITION BY` 語法建立 partition2. 不只可以按照某個字段做分區，也可以按照任意合法的表達式partition ex. toYYYYMM() 就可以按照月份來做 partition3. 支援對 partition 做 TTL 管理，淘汰過期的 partition 4. ClickHouse 中有專門對 partition 進行管理的 table `system.parts`**Shard (分片)**一個分片本身就是 ClickHouse 的一個實例節點，透過將一個全量的資料分成多份，從而降低單節點的資料掃瞄量，提高查詢性能
### **跳數索引 (Data Skipping Indexes)**
有別於傳統 SQL 資料庫使用多個次級索引來增加非索引查詢時的速度，ClickHouse 使用跳數索引來跳過讀取保證沒有匹配值的數塊，但要注意跳數索引只有支援 MergeTree 相關的 table engine


Skip Index Types
- minmax
- 儲存最大值與最小值的輕量化索引
- 不需要參數
- 適用於在按值松散排序的列
- 查詢處理成本最低
- set
- 接受一個參數
- 即每個區塊的值集合的 `max_size`（0 表示允許無限數量的離散值）
- 集合包含區塊中的所有值（如果值的數量超過 `max_size`，則為空）
- 索引的成本、性能和有效性取決於區塊內的基數
- Bloom Filter Types
- 是一種資料結構，允許以輕微的假正例機率為代價進行空間高效的集合成員測試


#  ClickHouse Table Engine
在 ClickHouse 中 Table engine 決定了資料的儲存方式、讀寫方式、併發控制、索引類型、查詢支援與複製機制，ClickHouse 能夠支撐高性能資料查詢的核心秘密之一，就是其強大的儲存引擎 — **MergeTree**，此外 ClickHouse 為不同目的提供了約 28 種表格引擎
EX. Log 系列用於小型表格資料分析，MergeTree 系列用於大容量資料分析，Integration 用於外部資料整合，複製表格 Replicated 和分散式表格 Distributed

## MergeTree Engine
MergeTree 是 ClickHouse 中最基礎的儲存引擎，負責將大量寫入資料有效儲存與管理，並支援高效的查詢與資料合併（Merge）操作
### MergeTree Engine Basic
**核心概念**
1. **分區 (Partitions)**：將資料依據指定欄位（如日期）切分成不同區塊，減少查詢時需掃描的資料量

2. **Primary Key (主鍵排序索引)**：定義資料在磁碟中的排序方式，讓查詢條件能快速定位資料範圍
3. **Data Parts (資料片段)**：每次資料寫入時，都會生成一個 Data Part，經過以下手續：Sorting、Splitting、Compression，最後 Writing to Disk
>
1. **Sorting**：資料根據 Sorting Key（如 town, street）進行排序，並產生一個稀疏主鍵索引（Sparse Primary Index）
2. **Splitting**：排序後的資料會被拆分成單獨的欄位
3. **Compression**：每個欄位分別進行壓縮處理，應用 LZ4、ZSTD 等壓縮算法

後續透過背景合併 (Merge) 將小片段整理成大型優化片段

**核心特性**
MergeTree 家族的引擎具備以下幾個特性：
1. **Primary Key 排序與稀疏索引 (Sparse Primary Index)**
表格的主鍵決定了每個資料片段 ( Data Part ) 內的排序方式（ Clustered Index ）。不過，這個索引並不指向單筆資料，而是以 8192 筆資料為單位的 Granule (粒度)，這種設計讓主鍵索引即便在超大資料量下仍能被保留在記憶體中，並且能有效快速地存取磁碟上的資料區塊
2. **靈活的分區機制 (Partitioning)**
使用任意表達式來劃分分區，並能透過 Partition Pruning 技術在查詢時自動跳過不相關的分區，避免不必要的 I/O
3. **高可用性與容錯 (Replication)**
資料可於多個 Cluster Nodes 間進行複製，支援高可用性、故障切換 (Failover)、以及無停機升級 (Zero Downtime Upgrade)
4. **統計與抽樣查詢 (Sampling & Statistics)**
MergeTree 支援各類型的統計與抽樣機制，可協助查詢優化器進行查詢路徑選擇與加速
>
**MergeTree 解決了什麼問題**
- **大規模寫入效能瓶頸**：透過將寫入資料分為小型 Data Parts 先行儲存，避免頻繁修改大型檔案帶來的 I/O 開銷
- **查詢效率提升**：依據 Partition 與 Primary Key 排序，能快速定位查詢資料區塊，避免全表掃描
- **資料壓縮與去重整合**：透過 Merge 操作合併資料時進行壓縮與去重，大幅降低儲存空間與查詢延遲

**程式範例**
Step1. Create Table
```sql
CREATE TABLE user_events
(
EventDate Date,
UserID UInt64,
EventType String,
EventValue Float32
) ENGINE = MergeTree()
PARTITION BY toYYYYMM(EventDate)
ORDER BY (EventDate, UserID);
```
Step2. Insert Data
```sql
INSERT INTO user_events VALUES ('2025-08-10', 1001, 'click', 1.0);
INSERT INTO user_events VALUES ('2025-08-10', 1002, 'view', 1.0);
INSERT INTO user_events VALUES ('2025-08-11', 1001, 'purchase', 299.99);
```
Step3. Query Data
```sql
SELECT *
FROM user_events
WHERE EventDate = '2025-08-10'
AND UserID = 1001;
```
### 指定索引 Granularity
若想更細緻控制 Primary Index 的粒度大小，可以透過 **`index_granularity`** 參數
```sql
ENGINE = MergeTree()
PARTITION BY toYYYYMM(EventDate)
ORDER BY (EventDate, UserID)
SETTINGS index_granularity = 8192;
```
8192 是預設 Granule 大小，若資料查詢行為較分散，也可以調整為更小的粒度 (如 4096) 來提升查詢命中率，但會增加索引體積
### MergeTree Famliy
ClickHouse 提供了各種衍生的 MergeTree Engine


| 儲存引擎
| 特性與應用場景
|


| ReplacingMergeTree
| 自動以指定欄位 (如 version 欄位) 替換重複資料，適合資料需去重的場景
|


| SummingMergeTree
| 在合併時自動將相同 Primary Key 的數值欄位進行加總，適用於資料匯總場景
|


| AggregatingMergeTree
| 針對 AggregateFunction 資料型別做更複雜的聚合運算，適合實時指標統計場景
|


| CollapsingMergeTree
| 透過 sign 欄位標記資料新增/刪除狀態，自動實現邏輯刪除與衝突解決
|


| VersionedCollapsingMergeTree
| 在 Collapsing 基礎上支援版本控制的資料去重
|


### **Merge 操作的原理與效能影響**
MergeTree 會在背景執行 Merge 操作，將多個小型 Data Part 合併成大型 Part，並在此過程中進行排序、壓縮與去重
- **Merge 頻率與效能平衡**：Merge 操作會佔用系統 I/O 資源，設定合理的 Merge 參數（如 **`max_parts_to_merge_at_once`**）可平衡查詢與寫入效能
- **Mutation (資料變異操作)**：ClickHouse 也支援在 Merge 階段進行 UPDATE / DELETE 操作，但屬於非即時性處理，適合資料分析場景
## **ReplacingMergeTree 與資料去重機制**
在大數據環境中，「資料重複」是常見且麻煩的問題，尤其是在 ETL Pipeline 或實時資料流匯入（如 Kafka Stream）時，重複資料會嚴重影響統計結果與查詢性能。ClickHouse 提供了一套簡單卻強大的去重機制：**ReplacingMergeTree**** 儲存引擎**
1. **運作邏輯**
ReplacingMergeTree 是 ClickHouse MergeTree 家族的一員，它能在背景 Merge 操作時，自動根據指定欄位 (如 version 欄位) 將重複資料去除，保留最新版本或最先寫入的那一筆
1. **程式範例**
```sql
CREATE TABLE user_profiles
(
user_id UInt64,
profile_version UInt32,
name String,
email String
) ENGINE = ReplacingMergeTree(profile_version)
ORDER BY user_id;
```
進行 `FINAL` 查詢語法後就可以確保查詢時讀到最新去重後的結果了
```sql
OPTIMIZE TABLE user_profiles FINAL;
```
>
**`OPTIMIZE`**：強制執行資料片段 (Data Parts) 合併動作，把分散的小 Data Parts 合併成大 Part，並同時觸發資料去重（ReplacingMergeTree）或聚合（SummingMergeTree）等邏輯。
**實質影響的是磁碟上的 Data Parts 資料**，合併結果是永久性（會寫入磁碟）

2. **ReplacingMergeTree 跟 Primary Key 的關係**
- **去重是基於 ****`Primary Key`**** 判斷的**，因此建表時的 **`ORDER BY`** 欄位（也就是 **`Primary Key`**）必須正確設計
- 若 **`ORDER BY`** 欄位無法唯一識別一筆記錄，則 **`ReplacingMergeTree`** 可能會保留錯誤的資料版本
3. **使用時機與適用場景**


| 場景
| 說明
|


| Kafka / Stream 資料流重放去重
| 多次 Consume、資料重放（At-least-once 保證）情境下自動去重
|


| 批次資料匯入過濾重複
| ETL 導入過程中重複載入相同資料，透過背景去重保證數據唯一性
|


| 具版本控制的資料歷史維護
| 保留最新版本數據，版本號較小的資料會在合併過程中被去除
|


| 數據補正修正
| 資料寫入後若有誤，透過補寫版本號較大的修正資料覆蓋錯誤記錄
|


4. **ReplacingMergeTree 與去重效能影響**
- `ReplacingMergeTree` 的寫入效能與 MergeTree 相近，因為去重是延遲處理的，不影響寫入吞吐量
- 背景合併 (Merge) 時會額外進行去重比對，Merge 負載會稍大，但對查詢來說，去重後反而能大幅減少掃描資料量
- 資料去重完成後，儲存空間可獲得顯著節省（取決於重複數據比例）
## **SummingMergeTree 進行資料彙總的應用場景**
需要大量的「數值加總」、「分組彙總統計」，例如每日活躍使用者數量、每小時流量統計、即時計數器 (Counter) 等，ClickHouse 提供了一個極致高效的資料彙總利器 —— **SummingMergeTree**
1. **基本介紹**
SummingMergeTree 是 ClickHouse MergeTree 系列的引擎變種，具備在資料合併 (Merge) 階段，自動將具有相同 **`Primary Key`** 的數值欄位進行加總 (SUM)，幫助你快速構建高效能的預彙總表 (Pre-Aggregated Table)
2. **特點**
- **自動數值欄位加總**：相同 Primary Key 的記錄在 Merge 時會自動執行加總
- **非即時性彙總**：數值加總發生於背景 Merge 階段
- **無需寫聚合函數邏輯**，簡單定義即可
3. **程式範例**
```sql
CREATE TABLE daily_metrics
(
date Date,
page String,
views UInt32,
clicks UInt32
) ENGINE = SummingMergeTree
PARTITION BY toYYYYMM(date)
ORDER BY (date, page);

INSERT INTO daily_metrics VALUES ('2025-08-01', 'Home', 100, 10);
INSERT INTO daily_metrics VALUES ('2025-08-01', 'Home', 200, 25);
INSERT INTO daily_metrics VALUES ('2025-08-01', 'Contact', 50, 5);
```
```sql
OPTIMIZE TABLE daily_metrics FINAL;
SELECT * FROM daily_metrics WHERE date = '2025-08-01';
```

4. **應用場景**


| 場景
| 說明
|


| 即時計數器 (Counter)
| 用於 API Call 次數、商品瀏覽次數等累加統計場景
|


| 批次數據預彙總 (Batch Aggregation)
| 將大數據表做成 Pre-Aggregated Table，避免即時計算壓力
|


| 流量/活動數據彙總 (Metrics Aggregation)
| 日誌、網站流量統計、使用者行為統計（例如 PV/UV 指標）
|


| 與 Materialized View 搭配
| 將 Raw Data 資料流寫入 Materialized View，實時聚合彙總結果至 SummingMergeTree 表
|


5. **GROUP BY 規則**
SummingMergeTree 的去重與彙總邏輯是以 **Primary Key 欄位為基準**，並對數值欄位進行加總
- ORDER BY 欄位 = GROUP BY Key
- 數值型欄位 (Int, Float) 才會自動 SUM
- 字串、日期等欄位僅參與分組，不會被聚合
6. **Materialized View 實時聚合**
```sql
CREATE TABLE raw_events
(
event_time DateTime,
page String,
views UInt32,
clicks UInt32
) ENGINE = MergeTree
ORDER BY (event_time, page);

CREATE MATERIALIZED VIEW mv_daily_metrics
TO daily_metrics
AS SELECT
toDate(event_time) AS date,
page,
sum(views) AS views,
sum(clicks) AS clicks
FROM raw_events
GROUP BY date, page;
```
7. **Best Practice**
1. **Primary Key 設計決定彙總效果**：**`ORDER BY`** 需謹慎設計，只包含用來 **`GROUP BY`** 的欄位
2. **數值欄位命名避免衝突**：非數值欄位不會被加總，但欄位型別設錯或命名混亂會導致統計錯誤
3. **資料寫入順序不影響結果**：Merge 階段才會進行彙總，因此資料流式寫入無需排序
4. **定期執行 OPTIMIZE FINAL**：若查詢即時性要求高，可定期強制執行合併去重，確保查詢返回彙總後的結果
5. **Materialized View 為即時彙總最佳搭配**：用以將原始大表流量，實時寫入 **`SummingMergeTree`** 進行高效查詢
## **CollapsingMergeTree 與邏輯刪除**
在傳統 OLTP 資料庫中，刪除與更新資料是家常便飯，但 ClickHouse 這類專為 OLAP 場警設計的資料庫中，「邏輯刪除」與「資料版本控制」則需要透過特別設計得儲存引擎實現，其中 **CollapsingMergeTree **就是 ClickHouse 提供的一種能夠自動處理資料新增與刪除標記（邏輯刪除）的特殊 MergeTree 引擎
1. **基本介紹**
CollapsingMergeTree 是 ClickHouse MergeTree 家族的一員，設計用來解決「資料新增與刪除標記」的場景，其運作邏輯是透過一個稱為 **`sign`** 的欄位標記資料的新增或刪除狀態，在背景 Merge 操作時，自動將 **`sign`** 相反的紀錄抵消 (collapse) 以實現邏輯刪除效果
2. **特點**
- 透過 **`sign`** 欄位標記資料狀態：**`1`** 代表新增，**`1`** 代表刪除
- Merge 階段會自動將相同 Primary Key 下的 **`1`** / **`1`** 抵消掉
- 資料刪除為「最終一致性」動作（**非即時刪除**）
3. **程式範例**
```sql
-- 建立 table
CREATE TABLE user_actions
(
user_id UInt64,
action String,
sign Int8
) ENGINE = CollapsingMergeTree(sign)
ORDER BY user_id;

-- 新增行為紀錄
INSERT INTO user_actions VALUES (1, 'login', 1);
INSERT INTO user_actions VALUES (2, 'purchase', 1);

-- 邏輯刪除 user_id = 2 的紀錄
INSERT INTO user_actions VALUES (2, 'purchase', -1);

--  query
OPTIMIZE TABLE user_actions FINAL;
SELECT * FROM user_actions;
```
**`user_id = 2`** 的紀錄將被抵消 (collapse) 掉，只剩下 **`user_id = 1`** 的資料
4. **運作原理**
1. **資料寫入時不進行即時去重**，所有資料（含 **`sign = -1`**）都會被寫入磁碟
2. **在背景合併 (Merge) 操作時**，ClickHouse 會根據 Primary Key 對應的資料進行配對：
- 1 筆 **`sign = 1`** 與 1 筆 **`sign = -1`** 的資料將被抵消 (collapse)
- 若 sign 數量不平衡 (如 **`sign = 1`** 多餘 **`sign = -1`**)，則會保留差值的紀錄
3. **查詢時不會自動隱藏尚未抵消的資料**，若要查詢已 collapse 的最終結果，可透過 **`FINAL`** 查詢保證一致性
4. **資料刪除是最終一致性動作**，在 Merge 完成前仍會顯示原始資料
5. **應用場景**


| 場景
| 說明
|


| 邏輯刪除 (Soft Delete)
| 需要保留刪除紀錄但不想讓刪除資料出現在查詢結果中
|


| 補資料與修正紀錄 (Data Correction)
| 錯誤資料寫入後可透過新增 sign = -1 的紀錄來抵銷錯誤紀錄
|


| Event Streaming 去重 (Event Deduplication)
| 用來消除重複事件紀錄，例如 Kafka Stream 資料去重
|


| 資料對帳與版本控制
| 結合版本欄位實現更精細的資料補正與對帳邏輯（推薦使用 VersionedCollapsingMergeTree）
|


6. **Best Practice**
1. 只對需要邏輯刪除的場景使用 CollapsingMergeTree
2. 設計 Primary Key 以「能唯一識別紀錄」為原則
3. 結合 Partition Key 做分區裁剪
4. 查詢大表時應避免全表 FINAL
5. 若需版本控制應考慮 VersionedCollapsingMergeTree
## **VersionedCollapsingMergeTree 與邏輯刪除**
在處理即時 Event Streaming 或高頻變動的資料場景時，僅靠 CollapsingMergeTree 的「新增/刪除」邏輯標記，往往無法應付複雜的資料狀態與版本管理需求。ClickHouse 為此提供了更進階的 **VersionedCollapsingMergeTree** 儲存引擎，透過 **`sign`** 與 **`version`** 雙欄位設計，實現更強大的資料去重與版本控制機制
1. **基本介紹**
VersionedCollapsingMergeTree 是在 CollapsingMergeTree 基礎上，加入版本欄位 (version) 的變種儲存引擎。 它能根據 Primary Key 判斷資料唯一性，並根據 **`sign`** 進行邏輯刪除，透過 **`version`** 來選擇保留最新版本的紀錄
2. **特點**
- **`sign`** 欄位：標記新增 (1) 或刪除 (-1)
- **`version`** 欄位：標記每筆資料的版本號，系統會保留最大版本
- 同一 Primary Key 下，當 **`sign`** 為相反數，且 **`version`** 值相同時，資料會被 collapse (抵消刪除)
- 當 **`version`** 值不同時，會保留最大版本的資料記錄
3. **程式範例**
```sql
-- 建立 table
CREATE TABLE user_profiles
(
user_id UInt64,
name String,
version UInt64,
sign Int8
) ENGINE = VersionedCollapsingMergeTree(sign, version)
ORDER BY user_id;

-- 初始新增紀錄
INSERT INTO user_profiles VALUES (1, 'Alice', 1, 1);
-- 更新版本紀錄
INSERT INTO user_profiles VALUES (1, 'Alice_updated', 2, 1);
-- 刪除紀錄 (針對 version = 2)
INSERT INTO user_profiles VALUES (1, 'Alice_updated', 2, -1);
```
執行背景 Merge (或 OPTIMIZE FINAL) 後：
- version = 2 的資料會被抵消。
- version = 1 的資料因為沒有對應的 -1 sign 紀錄，會被保留下來
4. **應用場景**


| 場景
| 說明
|


| 資料版本控制 (Data Versioning)
| 保留同一 Primary Key 的最新版本資料，並可透過補寫與刪除控制資料狀態
|


| 即時資料流去重與狀態管理
| 處理 Kafka、Message Queue 中的事件重放與修正，確保資料流中同一主鍵下只有一筆有效紀錄
|


| 補資料與刪除修正 (Data Correction)
| 當資料補寫/刪除需要依賴複雜的版本邏輯時，比單純的 CollapsingMergeTree 更合適
|


| IoT 或實時指標狀態更新場景
| 同一設備 (如 sensor_id) 狀態不斷更新，需保證資料只保留最新狀態值，並能正確處理刪除與修
|


## **AggregatingMergeTree 實時指標統計的進階應用**
在大型資料分析系統中，隨著資料規模與查詢複雜度提升，單純依賴 SELECT 聚合查詢（如 SUM、COUNT、AVG）將無法滿足即時回應的需求。ClickHouse 針對高效聚合查詢場景，提供了 **AggregatingMergeTree** 儲存引擎，透過預計算 (Pre-Aggregation) 與聚合函數壓縮 (AggregateFunction 型別)，大幅降低查詢延遲
1. **基本介紹**
AggregatingMergeTree 是 ClickHouse MergeTree 家族中的「聚合專用儲存引擎」，其特點是：
- 資料寫入時儲存為 **AggregateFunction 型別**（預計算的聚合狀態）
- Merge 階段進行 **狀態合併 (State Merging)**，將多筆相同 Primary Key 的聚合結果進行再彙總
- 查詢時透過 **聚合狀態還原 (State Finalization)**，將狀態轉為最終數值結果
這種設計特別適合高頻寫入 + 聚合查詢的場景，如：網站流量統計、實時 KPI 指標儀表板、IoT 裝置資料彙總
2. **特點**
- 支援聚合函數 : 支援所有 AggregateFunction (如 SUM, AVG, COUNT, MIN, MAX)
- 資料儲存型別 : AggregateFunction 型別 (儲存聚合狀態)
- Merge 時是否會聚合 : 會將相同 Primary Key 的聚合狀態進行合併
- 查詢時是否需特殊處理 : 查詢時需使用聚合函數將狀態還原（如 sumMerge(col), avgMerge(col)）
- 適用場景 : 複雜聚合運算（平均值、去重計數、分位數、標準差等統計需求）
3. **程式範例**
```sql
-- Source Table
CREATE TABLE visits (
StartDate DateTime64 NOT NULL,
CounterID UInt64,
Sign Nullable(Int32),
UserID Nullable(Int32)
) ENGINE = MergeTree ORDER BY (StartDate, CounterID);

-- AggregatingMergeTree Table
CREATE TABLE agg_visits (
StartDate DateTime64 NOT NULL,
CounterID UInt64,
Visits AggregateFunction(sum, Nullable(Int32)),
Users AggregateFunction(uniq, Nullable(Int32))
)
ENGINE = AggregatingMergeTree()
ORDER BY (StartDate, CounterID);

-- Materialized View Table
CREATE MATERIALIZED VIEW visits_mv TO agg_visits
AS SELECT
StartDate,
CounterID,
sumState(Sign) AS Visits,
uniqState(UserID) AS Users
FROM visits
GROUP BY StartDate, CounterID;
```
4. **運作流程**
Step1. **資料寫入 (AggregateFunction 狀態寫入) **
- 透過 **`sumState()`**, **`avgState()`**, **`uniqExactState()`** 等函數將資料寫入為「聚合狀態」
Step2. **背景 Merge 操作 (State Merging) **
- 當相同 Primary Key 的資料進行合併時，ClickHouse 會將 AggregateFunction 狀態進行「狀態合併」
- 這個過程發生於背景合併 (Merge)，不影響寫入吞吐量

Step3. **查詢時還原聚合結果 (State Finalization)**
- 查詢時需透過 **`sumMerge()`**, **`avgMerge()`**, **`uniqExactMerge()`** 等函數將聚合狀態還原為最終數值
5. **應用場景**


| 應用場景
| 說明
|


| 即時流量與使用者指標統計 (PV / UV 報表)
| 大量寫入使用者行為事件，彙總統計每天每頁面瀏覽量與不重複訪客數 (uniqExact)
|


| IoT 裝置資料彙總與即時狀態統計
| 實時收集 IoT 感測器資料，進行彙總平均值、最大最小值、分位數計算
|


| 行為事件流去重與複雜指標計算
| 如多指標 KPI 彙總、Session Duration 平均值等需進階統計場景
|


| 資料層級預聚合 (Pre-Aggregation)
| 針對查詢頻繁的聚合結果進行預先計算，減少查詢時的 CPU 負載與 I/O 操作
|


6. **優化 : 延遲聚合至 Merge 階段**
為了將聚合計算成本從 INSERT 時移至 Merge 階段，可以這麼做：
```sql
SET optimize_on_insert = 0;

CREATE MATERIALIZED VIEW visits_mv TO agg_visits
AS SELECT
StartDate,
CounterID,
initializeAggregation('sum', Sign) AS Visits,
initializeAggregation('uniqExact', UserID) AS Users
FROM visits;
```
效益：
1. **減少 INSERT 時的計算壓力 (極大化寫入吞吐)**
- 傳統 Materialized View 的預聚合邏輯：每次寫入 (INSERT) 都需要先做 Group By 聚合計算，這會對 CPU 與 Memory 造成即時壓力，寫入高峰時，會拖慢寫入速度
- initializeAggregation + optimize_on_insert = 0：每筆資料直接以「聚合狀態」寫入，無需即時計算聚合結果，讓 INSERT 成本變成「純寫檔案 + 生成狀態」的低成本操作
- 特別適合每秒數萬、數十萬筆的高頻寫入場景，如 IoT、網站點擊流、使用者行為追蹤
2. **聚合計算的成本被「平滑化」到背景 Merge**
- 傳統的預聚合方式會把所有聚合計算都集中在 INSERT 時進行，當寫入高峰時，資源使用率會瞬間飆升
- initializeAggregation 模式將聚合邏輯延遲到 Merge 階段，讓這些計算能夠由 ClickHouse 以 **批次、自動、分散式**的方式在背景進行。所以背景 Merge 可以根據系統負載動態調整，不會對線上寫入與查詢造成即時衝擊，更能彈性調整 Merge 策略 (如 TTL-based Merge、optimize_final 週期性合併)
>
### **為什麼 ****`initializeAggregation`**** 必須搭配 ****`optimize_on_insert = 0`****？**
當使用 **`initializeAggregation()`** 時，ClickHouse 會為每筆資料產生一個「尚未合併的 Aggregate State」，這讓每筆資料都能保留獨立的聚合狀態，直到 MergeTree 在背景進行合併 (Merge) 時，才會把相同 Primary Key 的資料進行實際聚合
只有當你將 **`optimize_on_insert`** 設為 0 時，ClickHouse 才會跳過這個 insert-time pre-aggregation 優化邏輯，將資料原封不動地寫入 AggregatingMergeTree，並將聚合計算延遲到 Merge 階段進行

#  **Materialized Views**
在 OLAP 系統中，「即時聚合」與「預先計算」是加速查詢、降低資源消耗的核心策略。ClickHouse 提供了強大的 **Materialized Views (物化視圖)**，能將複雜查詢結果實時寫入表中，並大幅減輕查詢時的運算壓力
## 什麼是 **Materialized Views**
Materialized View 是一種**帶有持久化儲存結果的查詢視圖**，當有資料寫入源表 (source table) 時，ClickHouse 會自動根據定義的 SELECT 查詢語句計算並將結果寫入目標表 (target table)
## 特點
- INSERT 寫入源表時，自動將計算結果寫入目標表
- 目標表通常是 **`SummingMergeTree`** / **`AggregatingMergeTree`** 等
- 只會計算「新寫入資料」的查詢結果，不會對舊資料重新掃描
```sql
# 建立 Source Table (Raw Data Table)
CREATE TABLE events
(
event_time DateTime,
page String,
user_id UInt64,
views UInt32
) ENGINE = MergeTree
ORDER BY (event_time, page);

# 建立 Target Table (Aggregate Table)
CREATE TABLE daily_page_views
(
date Date,
page String,
total_views UInt32
) ENGINE = SummingMergeTree
PARTITION BY toYYYYMM(date)
ORDER BY (date, page);

# 建立 Materialized View
CREATE MATERIALIZED VIEW mv_daily_page_views
TO daily_page_views
AS SELECT
toDate(event_time) AS date,
page,
sum(views) AS total_views
FROM events
GROUP BY date, page;
```
## 應用場景


| 應用場景
| 說明
|


| 高頻查詢彙總結果表 (Dashboard/BI 報表)
| 先將重計算的聚合結果寫入目標表，查詢時僅需掃小型表格
|


| 即時Event Streaming彙總 (如 PV/UV 統計)
| 結合 Kafka + MV，即時統計點擊數、瀏覽量等
|


| 指標統計與彙總 (Metrics Storage)
| 以 Materialized View 實時計算指標資料，適合 IoT、監控平台
|


| ReplacingMergeTree 或 SummingMergeTree 結合
| 目標表可以使用去重、聚合引擎進一步優化結果儲存
|


## **Materialized View 設計重點與限制**
1. **目標表 (TO Table) 必須先存在 : **Materialized View 是寫入目標表的「Trigger」，Target Table 必須先建立好
2. **只計算新增資料 : **MV 只會針對「新寫入」的資料進行聚合，不會針對舊資料補算，若源表有歷史資料變動需手動重算
3. **INSERT 觸發查詢，查詢性能依賴目標表設計 : **目標表應搭配適當的 MergeTree 引擎（**`Summing`**, **`Aggregating`**, **`Replacing`**）來對應資料處理需求
4. **無法直接支援 UPDATE/DELETE : **MV 只會針對 INSERT 事件觸發，若需進行補數據、資料刪除，需配合 **`ReplacingMergeTree`** 或 Mutation 處理
#  ClickHouse 的分散式架構
隨著資料量從 TB 級別成長至 PB 級別，單機架構顯然無法應付現代資料分析與即時查詢的需求，ClickHouse 透過 **Distributed Table 與分布式查詢架構**，讓資料能夠橫向擴展到**數十、數百台節點**，並在龐大資料量下依然維持秒級查詢回應
## Distributed Table
Distributed Table 並不是**實際存放資料的表**，而是 ClickHouse 提供的 **查詢路由代理 (Query Router)****，**它的作用是 :
1. 接收分布式查詢請求
2. 將查詢根據分片 (Shard) 規則分派至對應的資料節點 (Shard / Replica)
3. 蒐集各分片節點的查詢結果後，進行合併 (Merge) 回傳給使用者
### 核心概念


- Shared : 水平切分資料的單位，每個分片儲存資料的子集
- Replica : 同一 Shard 的多個副本，保證高可用性與讀取負載平衡
- Distributed Table : 路由器角色，負責將查詢請求分派至正確的 Shard/Replica 節點
- Remote Table : 真正儲存資料的表 (通常為 MergeTree 或其變種引擎)


### **語法與配置**
**1. 建立 Remote Table (實際儲存資料的表)**
```sql
CREATE TABLE cluster_shard.local_visits
(
Date Date,
UserID UInt64,
PageViews UInt32
) ENGINE = MergeTree()
ORDER BY (Date, UserID);
```
1. **建立 Distributed Table (分布式查詢路由表)**
```sql
CREATE TABLE [IF NOT EXISTS] [db.]table_name [ON CLUSTER cluster] AS [db2.]name2 ENGINE = Distributed(cluster, database, table[, sharding_key[, policy_name]]) [SETTINGS name=value, ...]

CREATE TABLE cluster_shard.distributed_visits
AS cluster_shard.local_visits
ENGINE = Distributed(cluster_name, 'cluster_shard', local_visits, rand());

-- cluster_name : 定義 clusets.xml 的 cluster 名稱
-- cluseter_shared : 資料庫名稱
-- local_vists : remote table 名稱 (目標 table)
-- rand() : 分片選擇策略，常見還有 sharding_key() 用於精確控制分片邏輯
--- 1. rand() : 隨機分片，適用於無明確切分邏輯、但需均衡寫入負載的場景
--- 2. cityHash64(UserID) % N : 根據 UserID 做 Hash 切分，保證相同 UserID 資料會落在同一 Shard 上
--- 3. toYYYYMM(Date) % N : 根據時間維度做切分，適合時間序列資料，如日誌分析、感測器資料
--- 4. Sharding Expression (任意欄位) : 可自由設定複合欄位作為分片鍵，根據業務邏輯優化查詢裁剪 (Data Skipping)
```
>
**設計建議**
- 盡可能避免查詢跨所有 Shard (如無分片條件的全表掃描)
- 分片鍵選擇應盡量與**查詢 WHERE 條件一致**，提升 Data Skipping 效率
- 維持分片數量與資料量均衡 (避免資料傾斜)

## Replicated Table
在實務中，資料庫節點可能因硬體故障、軟體升級或網路問題而離線。如何確保資料不遺失、查詢不中斷，並且能夠在線進行升級與維護，**高可用性 (High Availability, HA)** 架構成為核心需求
ClickHouse 提供了 **Replicated Tables** 機制，透過 **主從副本同步（Replication）與自動 Failover 機制**，實現資料一致性與讀取負載平衡，讓叢集具備「零停機升級」的能力
### 基本概念
Replicated Tables 是 ClickHouse 內建的一種資料複製技術，依賴 **ClickHouse Keeper / ****ZooKeeper** 作為協調器，實現以下功能 :
1. **資料副本同步 (Replication)**：將資料自動複製到多個節點（Replica）
2. **故障容錯 (Failover)**：當主節點故障時，自動切換至其他 Replica
3. **讀取負載平衡 (Load Balancing Reads)**：查詢時可自動將請求分散到多個 Replica 節點上
4. **無鎖定的線上擴容與升級 (Zero Downtime Scaling)**：允許新增副本節點、替換節點而不影響資料可用性

並且支援絕大多數的 MergeTree  Engine ，這些引擎會在建立表格時透過 **`Replicated*MergeTree`** 來定義，並指定 ZooKeeper/ClickHouse Keeper 路徑與 Replica 名稱來實現資料副本同步
### **語法與配置**
**建立 ReplicatedReplacingMergeTree Table**
```sql
CREATE TABLE table_name
(
EventDate DateTime,
CounterID UInt32,
UserID UInt32,
ver UInt16
)
ENGINE = ReplicatedReplacingMergeTree(
'/clickhouse/tables/{layer}-{shard}/table_name',
'{replica}',
ver
)
PARTITION BY toYYYYMM(EventDate)
ORDER BY (CounterID, EventDate, intHash32(UserID))
SAMPLE BY intHash32(UserID);

-- /clickhouse/tables/{layer}-{shard}/table_name : 這是 Zookeeper/ClickHouse Keeper 中的 Metadata 路徑，每個 Replica 會註冊到這個路徑下。 {layer} 與 {shard} 會從 macros.xml 讀取
-- {replica} : 代表該節點的 Replica 名稱 (例如 replica1, replica2)，同樣從 macros.xml 讀取
```
**macros.xml 每個節點需設定 shard 與 replica 名稱**
```xml


01
replica1


```
### 技術原理
1. **Replication 是「以資料表為單位」進行，不是以整個伺服器為單位**
- 一台 ClickHouse Server 上可以同時存在 **Replicated Tables** 與 **Non-Replicated Tables**
- 複製 (Replication) 與分片 (Sharding) 是兩個獨立的概念，Replication 只影響表格副本同步，不影響資料如何分片

1. **Sharding 與 Replication 的獨立性**
- 每個 Shard 之間的資料是獨立的，不會互相複製
- **Shard 內的副本 (Replica)** 會透過 ReplicatedMergeTree 系列引擎進行同步
- 分片是「水平切分資料量」，副本是「確保高可用性與故障容錯」
```sql
Cluster 配置：
┌─────────────────────────┐
│   Shard 1               │
│   ├─ Node 1 (Replica 1) │ ← user_id % 2 = 0 的資料
│   └─ Node 2 (Replica 2) │ ← 複製 Shard 1 的資料
├─────────────────────────┤
│   Shard 2               │
│   ├─ Node 3 (Replica 1) │ ← user_id % 2 = 1 的資料
│   └─ Node 4 (Replica 2) │ ← 複製 Shard 2 的資料
└─────────────────────────┘
```

1. **INSERT 與 ALTER 的操作會複製 (同步)**
- **INSERT**：寫入時資料會被同步到同一 Shard 內的所有 Replica 節點
- **ALTER**：如資料表結構變更 (ADD COLUMN 等)，也會將這些變更同步到 Replica
- **ALTER 操作支援非阻塞 (non-blocking) 機制**，能夠在線進行不影響線上查詢

1. **CREATE / DROP / ATTACH / DETACH / RENAME 等 DDL 操作不會自動複製**
- **CREATE TABLE**：
- 在某節點執行 CREATE 時，會在 Keeper/ZooKeeper 註冊為一個新的 Replica
- 若其他節點上已存在該表，則會自動建立一個新的 Replica 參與同步
- **DROP TABLE**：
- 只會刪除執行該指令的節點上的 Replica，不會影響其他 Replica
- **ATTACH / DETACH TABLE**：
- 操作的僅是該節點本機上的表，不會影響其他 Replica
- **RENAME TABLE**：
- RENAME 只會影響當前節點的表名稱，Replica 之間的表名可以不同（數據仍同步）

2. **Keeper / Zookeeper 為 Replica Metadata 協調服務**
- Replica 間的同步資訊（例如：目前有哪些 Replica、哪個是 Primary、同步進度、Leader 選舉）都會儲存在 ClickHouse Keeper 或 ZooKeeper 中
- ClickHouse 官方推薦使用 **ClickHouse Keeper**（ClickHouse 自家實作的輕量版協調器），相較於 ZooKeeper 更輕量穩定

1. **設定 Replication 需要配置 Keeper 區段**
在 ClickHouse 設定檔中，需要定義 ZooKeeper / Keeper 協調器
`config.xml`
```xml


zk1
2181


zk2
2181


zk3
2181


```

### **零停機升級 (Zero Downtime Upgrade) 流程**
**升級 Replica 節點步驟**
1. 將待升級節點標記為 **僅讀取流量**** (停止寫入該節點)**
2. 停止該 Replica 節點的 ClickHouse 服務
3. 升級 ClickHouse 版本，修改配置檔等作業
4. 重啟節點後，自動從其他副本同步缺失的資料 Part
5. 節點完成升級並重新加入叢集，恢復寫入與查詢
**完整升級叢集時**
1. 依序升級 Replica 節點
2. 確保任意時刻至少有一個 Primary 節點與副本處於線上狀態
3. 若需要升級 Shard 節點，應先將流量導向其他 Shard 或 Replica

### **Cluster 設計建議**
1. 使用奇數數量的 Keeper 節點 (3 或 5 節點) : 確保故障容忍度達到 n/2，可支撐單點或雙點故障仍維持叢集協調運作
2. 分散 Replica 於不同可用區 (AZ) : 提升容災能力，確保單一區域失效不會影響整體叢集可用性
3. 設定 insert_quorum=2 以強化寫入一致性 : 可設定 Quorum Write 機制，保證至少 N 個副本成功寫入才返回成功
4. 使用 Distributed Table + ReplicatedMergeTree : 確保資料分片與副本同步結合，實現橫向擴展與高可用性並存的架構設計
#  Integration
## Streaming Data with Kafka
在大規模資料場景下，企業越來越需要能夠 **實時處理與分析 Data Streaming** 的技術架構。 而 ClickHouse 天生與 Kafka 的整合，提供了一條高效能的 **Event Streaming → Real-Time Analytics** Data Pipeline，讓資料從產生到分析僅需「秒級延遲」

### 架構概念


```xml
┌─────────────┐
│  Producers  │
│             │
│ • Web       │
│ • IoT       │
│ • APM       │
└──────┬──────┘
│
▼
┌─────────────────┐
│  Kafka Topics   │
│                 │
│ • High throughput│
��� • Replayable    │
└────────┬────────┘
│
▼
┌──────────────────────────────┐
│ Kafka Connect /              │
│ ClickHouse Kafka Engine      │
│                              │
│ • Continuous ingestion       │
└─────────────┬────────────────┘
│
▼
┌───────────────────────────┐
│  ClickHouse Tables        │
│                           │
│ • Materialized Views      │
│ • Partitioning            │
│ • Real-time Analytics     │
└───────────────────────────┘
```


Stage 1. Producers
- Description: Upstream systems generating data streams
- Sources:
- Web behavior tracking
- IoT devices
- Application Performance Monitoring (APM)
Stage 2. Kafka Topics
- Description: Message Queue receiving data streams
- Key Features:
- High throughput
- Message replay capability
- Distributed architecture
- Data persistence
Stage 3. Kafka Connect / ClickHouse Kafka Engine
- Description: Data ingestion layer
- Responsibilities:
- Continuous data streaming from Kafka to ClickHouse
- Data transformation (if needed)
- Connection management
Stage 4. ClickHouse Tables
- Description: Storage and real-time analytics


### 整合方式
**\[ 方法一 \] Kafka Engine (ClickHouse 原生)**
- ClickHouse 內建支援 Kafka Engine，可將 Kafka Topic 當作虛擬表來查詢
- 適合 **低延遲****場景**，資料可以邊讀邊寫入 ClickHouse MergeTree 表格
**\[ 方法二 \] Kafka Connect + ClickHouse Sink Connector**
- 透過 Kafka Connect 架設 ETL Pipeline，使用 ClickHouse Sink Connector 自動將資料流寫入 ClickHouse
- 適合 **Data Streaming轉管道標準化需求** (與其他資料平台共用 Kafka Connect 架構)

### **語法與配置**
```sql
CREATE TABLE [IF NOT EXISTS] [db.]table_name [ON CLUSTER cluster]
(
name1 [type1] [ALIAS expr1],
name2 [type2] [ALIAS expr2],
...
) ENGINE = Kafka()
SETTINGS
kafka_broker_list = 'host:port',
kafka_topic_list = 'topic1,topic2,...',
kafka_group_name = 'group_name',
kafka_format = 'data_format'[,]
[kafka_security_protocol = '',]
[kafka_sasl_mechanism = '',]
[kafka_sasl_username = '',]
[kafka_sasl_password = '',]
[kafka_schema = '',]
[kafka_num_consumers = N,]
[kafka_max_block_size = 0,]
[kafka_skip_broken_messages = N,]
[kafka_commit_every_batch = 0,]
[kafka_client_id = '',]
[kafka_poll_timeout_ms = 0,]
[kafka_poll_max_batch_size = 0,]
[kafka_flush_interval_ms = 0,]
[kafka_thread_per_consumer = 0,]
[kafka_handle_error_mode = 'default',]
[kafka_commit_on_select = false,]
[kafka_max_rows_per_message = 1];

--- 參數設定
-- kafka_broker_list : Kafka Broker 位址與 Port，支援多節點（逗號分隔）
-- kafka_topic_list監聽的 Kafka Topic 名稱，支援多個 Topic（逗號分隔）
-- kafka_group_nameConsumer Group 名稱，ClickHouse 會以這個 Group 協調消費者進度
-- kafka_format資料格式，如 JSONEachRow、CSV、Avro、Protobuf 等
```

**Step1. Create  Table**
```sql
-- Main Events Table
CREATE TABLE IF NOT EXISTS default.user_events
(
UserID UInt64,
Action String,
EventDate DateTime,
) ENGINE = MergeTree()
PARTITION BY toYYYYMM(EventDate)
ORDER BY (UserID, EventDate);


-- Creat Kafka Engine Table : 本身不會儲存資料，而是作為 ClickHouse 消費 Kafka 訊息的入口
CREATE TABLE IF NOT EXISTS default.kafka_user_events
(
UserID UInt64,
Action String,
EventDate DateTime,
) ENGINE = Kafka()
SETTINGS
kafka_broker_list = 'kafka:29092',
kafka_topic_list = 'user_events_topic',
kafka_group_name = 'clickhouse_consumer_v3',
kafka_format = 'JSONEachRow',
kafka_num_consumers = 1,
kafka_thread_per_consumer = 1;
```
>
Kafka 表本身無法直接查詢，你必須透過 Materialized View 將資料寫入實體表才能存取


**Step2. Create MV **
這是整個 Streaming Pipeline 的關鍵橋樑，Materialized View 會監聽 kafka_user_events，並將其每筆資料**自動寫入**目標表 user_events
- 使用 **`TO default.user_events`** 表示這是一個「推送型」 MV
- SELECT 子句決定要寫入的資料欄位，需與目標表 schema 相符
- 不需要手動執行 INSERT，資料會**自動同步**
```sql
-- Materialized View to stream data from Kafka to main table
CREATE MATERIALIZED VIEW IF NOT EXISTS default.kafka_to_events_mv TO default.user_events AS
SELECT
UserID,
Action,
EventDate,
FROM default.kafka_user_events;
```

## Batching Data Best Practice
在實務的資料分析與數倉場景中，批次匯入（Batch Import）是 ClickHouse 最常見的資料導入方式之一。 根據資料量、來源與格式的不同，選擇合適的匯入方法與檔案格式，能大幅提升匯入速度並降低資源消耗，主要有三種方式 :
### CSV 匯入
1. 適用場景
- 來源系統匯出為純文字檔
- 跨平台通用、容易查看與修改
- **不追求極致效能**，但需要易於整合
2. 範例
```bash
clickhouse-client --query="INSERT INTO events_csv FORMAT CSV"  events.native
clickhouse-client --query="INSERT INTO events_native FORMAT Native"
## 如何優化查詢效能 ?
### **system.query_log**
system.query_log 是 ClickHouse 內建的查詢歷史紀錄表，它會紀錄每一筆查詢的：
- 啟動時間、執行耗時
- 資源使用量 (讀取行數、記憶體用量)
- 查詢錯誤與異常
- 使用者、來源 IP、Client 資訊
- 查詢使用的 Storage、Functions、Events
```sql
-- 如何找出慢查詢
SELECT
query_start_time,
query_duration_ms,
read_rows,
memory_usage,
query
FROM system.query_log
WHERE event_time > now() - INTERVAL 1 HOUR
AND type = 'QueryFinish'
AND query_duration_ms > 500  -- 大於 500ms
ORDER BY query_duration_ms DESC;
```
### **EXPLAIN**
ClickHouse 提供 EXPLAIN 語法，讓你在查詢前預測 **查詢路徑、掃描資料量、JOIN 策略** 等細節
```sql
EXPLAIN [AST | SYNTAX | QUERY TREE | PLAN | PIPELINE | ESTIMATE | TABLE OVERRIDE] [settings]
SELECT ...
```
### 優化一個慢查詢
Step1. 先用 system.query_log 找到最近的慢查詢Step2. 把該 SQL 用 EXPLAIN PLAN 預測路徑與資料量
Step3. 檢查
- 有全表掃描 (資料區塊過大)
- 有不必要的 JOIN → 可否轉 Materialized View
- 缺少 Partition Pruning、索引無法生效
Step4. 調整查詢條件

## Compressed Strategy
在 OLAP 場景中，資料量動輒百萬、千萬筆，若未經良好壓縮處理，磁碟 I/O 將成為查詢效率瓶頸。ClickHouse 採用 Columnar Storage，讓每欄資料型態一致、重複性高，使得壓縮成效極佳
### **ClickHouse  的壓縮優勢**
- **降低磁碟儲存空間需求**（通常壓縮比達 5\~10 倍以上）
- **減少磁碟 I/O 傳輸量**（更快讀取、更低延遲）
- **CPU 解壓縮效能優化**（使用輕量快速壓縮算法）
以下是以 ClickHouse 儲存 StackOverflow posts 資料表的壓縮統計，透過查詢 **`system.columns`** 取得各欄位壓縮前後的體積與壓縮比資料
```sql
SELECT name,
formatReadableSize(sum(data_compressed_bytes)) AS compressed_size,
formatReadableSize(sum(data_uncompressed_bytes)) AS uncompressed_size,
round(sum(data_uncompressed_bytes) / sum(data_compressed_bytes), 2) AS ratio
FROM system.columns
WHERE table = 'posts'
GROUP BY name

┌─name──────────────────┬─compressed_size─┬─uncompressed_size─┬───ratio────┐
│ Body                  │ 46.14 GiB       │ 127.31 GiB        │ 2.76       │
│ Title                 │ 1.20 GiB        │ 2.63 GiB          │ 2.19       │
│ Score                 │ 84.77 MiB       │ 736.45 MiB        │ 8.69       │
│ Tags                  │ 475.56 MiB      │ 1.40 GiB          │ 3.02       │
│ ParentId              │ 210.91 MiB      │ 696.20 MiB        │ 3.3        │
│ Id                    │ 111.17 MiB      │ 736.45 MiB        │ 6.62       │
│ AcceptedAnswerId      │ 81.55 MiB       │ 736.45 MiB        │ 9.03       │
│ ClosedDate            │ 13.99 MiB       │ 517.82 MiB        │ 37.02      │
│ LastActivityDate      │ 489.84 MiB      │ 964.64 MiB        │ 1.97       │
│ CommentCount          │ 37.62 MiB       │ 565.30 MiB        │ 15.03      │
│ OwnerUserId           │ 368.98 MiB      │ 736.45 MiB        │ 2          │
│ AnswerCount           │ 21.82 MiB       │ 622.35 MiB        │ 28.53      │
│ FavoriteCount         │ 280.95 KiB      │ 508.40 MiB        │ 1853.02    │
│ ViewCount             │ 95.77 MiB       │ 736.45 MiB        │ 7.69       │
│ LastEditorUserId      │ 179.47 MiB      │ 736.45 MiB        │ 4.1        │
│ ContentLicense        │ 5.45 MiB        │ 847.92 MiB        │ 155.5      │
│ OwnerDisplayName      │ 14.30 MiB       │ 142.58 MiB        │ 9.97       │
│ PostTypeId            │ 20.93 MiB       │ 565.30 MiB        │ 27         │
│ CreationDate          │ 314.17 MiB      │ 964.64 MiB        │ 3.07       │
│ LastEditDate          │ 346.32 MiB      │ 964.64 MiB        │ 2.79       │
│ LastEditorDisplayName │ 5.46 MiB        │ 124.25 MiB        │ 22.75      │
│ CommunityOwnedDate    │ 2.21 MiB        │ 509.60 MiB        │ 230.94     │
└───────────────────────┴─────────────────┴───────────────────┴────────────┘
```
- 高重複值欄位（如 FavoriteCount, ContentLicense）壓縮比可達數百甚至數千倍，這正是列式儲存結合專屬編碼的優勢
- 數值型欄位（Score, AcceptedAnswerId, PostTypeId）壓縮效果也非常顯著，Delta 編碼結合 LZ4/ZSTD 能大幅減少資料體積
- Body, Title 雖為文字型欄位，但仍有 2–3x 壓縮比，搭配 LowCardinality 設計將有更好空間優化空間
### **指定壓縮算法**
在 CREATE TABLE 的時候可以透過 `CODEC` 參數指定欄位使用的壓縮編碼
```sql
CREATE TABLE user_events (
event_date Date,
user_id UInt32,
event_type String CODEC(ZSTD),
event_value Float64 CODEC(Delta, LZ4)
) ENGINE = MergeTree
PARTITION BY toYYYYMM(event_date)
ORDER BY (event_date, user_id);
```
## Partition and **Partition Pruning**
當面對數億、數十億筆資料時，若每次查詢都必須掃描全表，效率勢必崩潰，ClickHouse 提供了靈活的 **分區**** (Partitioning)** 與 **分區裁剪**** (Partition Pruning)** 技術，讓你在查詢時僅需掃描「真正相關的資料區塊」，大幅減少 I/O 與查詢延遲
### Partition Basic
在 ClickHouse 中，**Partition **是一種邏輯資料切分單位，資料會根據指定的 Partition Key (表達式) 被切分成獨立的資料區塊（目錄），這些區塊在查詢時可以根據條件篩選，避免全表掃描
**重要特性**
- Partition 是 MergeTree 引擎的核心結構之一
- Partition Key 可以是任意表達式（如 toYYYYMM(date)、**`device_id`**、**`region_id`**）
- Partition 是「物理檔案目錄」級別的切分，對查詢優化效果非常明顯 \>\> 同理於 hdfs 的不同 folder
- Partition 切分範圍越小，查詢時能跳過的資料越多，但會增加小檔案數量與合併負擔 \>\> 換句話說 partition 需要切的很合理

**Partition  vs Primary Key**


| 比較項目
| Partition
| **Primary Key**
|


| 作用層級
| 資料目錄切分 (磁碟層級)
| 區塊內排序索引 (內部儲存層級)
|


| 查詢裁剪
| 查詢條件符合 Partition Key 時可直接跳過該區塊
| 查詢條件符合 Primary Key 時可精確定位資料區塊
|


| 設計依據
| 常用範圍查詢條件，如日期、業務區域
| 常用細粒度查詢條件，如 `user_id`、`order_id`
|


\*  兩者是互補關係，Partition 用於粗略範圍裁剪、Primary Key 用於精細定位
### Partition Pruning (分區剪裁) 原理
Partition Pruning 是 ClickHouse 在查詢時根據 WHERE 條件，自動判斷哪些 Partition 是不可能有資料的，並直接跳過讀取這些資料區塊
```sql
CREATE TABLE page_views
(
event_date Date,
user_id UInt64,
page String
) ENGINE = MergeTree
PARTITION BY toYYYYMM(event_date)
ORDER BY (event_date, user_id);

SELECT count() FROM page_views
WHERE event_date >= '2025-08-01' AND event_date
**Partition 與分片 (Sharding) 的關係**
兩者可結合設計，例如：
- 分片鍵 = **`user_id`**（負載均衡分散寫入）
- 分區鍵 = toYYYYMM(date)（加速範圍查詢）


| Partition (分區)
| Sharding (分片)
|


| 將單一表內的資料切分為多個資料區塊，便於裁剪查詢範圍
| 將整張表橫向切分到不同節點（叢集）上，提升分散式計算能力
|


| 分區僅影響單表查詢時的掃描範圍
| 分片影響資料的存放位置與分散式查詢執行路徑
|


| 分區裁剪靠 WHERE 條件判斷，查詢時減少掃描資料量
| 分片裁剪靠分片鍵與分布式查詢路由規則，決定哪些節點需參與查詢
|


### Best Practice
- 分區 Key 依據查詢模式設計 : 針對最常用的 WHERE 範圍條件（如日期、地區）設計分區裁剪維度
- 小心 Partition 數量爆炸 : 避免選用高基數欄位作為分區鍵，Partition 數量建議控制在數千 \~ 一萬以內
- 使用 system.parts 監控分區健康度 : 定期檢查 Active Parts 數量與大小，避免過多小檔案影響合併性能
- 結合 Primary Key 設計精細資料定位 : 分區負責粗篩，Primary Key 負責精細定位，兩者結合達到最佳查詢效率
- 不建議頻繁更改 Partition Key : Partition Key 變更等同於重建表，需謹慎設計與評估
## Primary Indexes
傳統資料庫採用 B-Tree 結構，ClickHouse 採用不同的設計方式，在 indexing 上也採用了不同的方式，primary indexes 不會為每一 row 都建立 index，而是每組 row (granules) 設置 index，這稱為稀疏索引 (sparse index)  ，因此它不保證資料唯一性，也不會自動加索引樹，ClickHouse 的 Primary Key 是用來決定資料在磁碟中的物理排序方式 **(Clustered Index)**，它是 MergeTree 引擎搜尋資料的首要索引依據
**密集索引 vs 稀疏索引**
```sql
-- 傳統 B-Tree（密集索引）
-- 8,870,000 rows → 8,870,000 index entries

-- ClickHouse 稀疏索引
-- 8,870,000 rows → ~1,083 index entries (assuming 8192 rows per granule)
```
**稀疏索引運作模式**
1. **每一個 granule 只記錄首筆資料的 Primary Key 值** : 例如預設 Granule 粒度 8192 筆，索引只會紀錄第一筆的主鍵值
2. **Sparse 索引非常精簡，能完全載入記憶體中**，即使資料量達到數百億筆，索引仍僅需占用少量記憶體空間
3. **每個 MergeTree 的 Data Part 都有獨立 Primary Index**，查詢時這些索引會分別比對以達到最佳裁剪效果
4. **查詢時，ClickHouse 根據 WHERE 條件與 Sparse Primary Index 比對 Granule 範圍**
- 條件範圍外的 Granule 會被直接跳過，不進行掃描
- 條件範圍內的 Granule 才會被讀取進行後續篩選
### Primary Indexes Basic
**特點**
- 決定資料的排序邏輯，並在查詢時作為區塊篩選的依據
- 與 Partition Key 互補，Partition 負責粗篩區塊，Primary Key 決定區塊內的排序與定位
- 可由一或多個欄位組成（ORDER BY 子句指定）

**程式範例**
```sql
CREATE TABLE orders
(
order_date Date,
user_id UInt64,
order_id UInt64,
amount Float64
) ENGINE = MergeTree
PARTITION BY toYYYYMM(order_date)
ORDER BY (user_id, order_date);
```
>
**Sorting Key**
- **Sorting Key = ORDER BY 子句指定的欄位組合**
- 在 ClickHouse，Sorting Key 就是 Primary Key，只是名稱層面上的不同（某些文件會混用這兩個詞）
- Sorting Key 決定了 Data Part 中資料的物理排序方式，並影響查詢範圍裁剪的效率


### **Granule**
Granule 是 ClickHouse 將資料拆分為查詢時可裁剪最小單位的「資料區塊」， 一個 Granule 會包含數千筆資料 (預設為 8192 rows)，系統會為每個 Granule 儲存該範圍內的 Sorting Key 最小值與最大值 (min-max 索引)


- 預設每個顆粒包含 8192 列
- 索引只為每個顆粒建立一個標記
- 大幅減少索引大小和記憶體使用

**Granule 的查詢流程**
1. 查詢時會根據 WHERE 條件比對 Granule 的 min-max 範圍
2. 若條件不在該 Granule 範圍內，則直接跳過讀取該 Granule
3. 這種跳過稱為 Primary Key 範圍裁剪 (Primary Key Indexing)

**Granule 的儲存結構**
- Granule ≈ 8192 rows (預設，可調整)
- 一個 Data Part 會包含多個 Granule
- Primary Key 索引是針對 Granule 粒度儲存的 Sparse Index

**測試案例**


Testing Query
### Primary Key 設計策略
1. 常查詢範圍條件放最前面 EX.  user_id、device_id 若常作為 WHERE 條件，應放排序鍵首位
2. 從高選擇性到低選擇性排序 EX. user_id → event_date，讓 Granule 範圍更集中，裁剪更精準
3. 避免將高變異但不查詢的欄位設為排序鍵 EX.  UUID、隨機 hash，排序無助於裁剪，只會造成合併成本上升
4. 結合 Partition 與 Sorting Key 設計 EX. Partition 粗裁剪、Primary Key 精裁剪，讓查詢僅需掃描極小範圍資料

>
**Primary Key vs Secondary Index**


| 比較項目
| Primary Key (範圍索引)
| Secondary Index (Data Skipping Index)
|


| 運作方式
| 資料寫入時排序，查詢時透過 Granule 索引範圍裁剪
| 查詢時依欄位值範圍 (min-max / bloom filter) 決定是否讀取
|


| 查詢效率
| 查詢條件若符合排序欄位 → 裁剪效率極佳
| 可支援非排序欄位的查詢過濾，但效率不如 Primary Key
|


| 建立方式
| 透過 ORDER BY 設定，與 MergeTree 強耦合
| 需額外建立 (ALTER TABLE ADD INDEX…)
|


| 適用查詢
| 範圍查詢、序列查詢、依排序邏輯為主的查詢
| 高基數欄位查詢（如特定 tag、keyword）
|


## Data Skip Indexing
**Data Skipping Indexes（資料跳過索引）** 是 ClickHouse 特有的查詢加速技術，核心原理是 :
> **在查詢時，只掃描必要的資料區塊，跳過無關區塊，降低 I/O 成本與查詢延遲**
這些索引不是傳統意義上的 B-Tree 索引，而是針對 MergeTree 資料片段內的「欄位統計資訊」建立的快速過濾機制
### **minmax Index**
每個 MergeTree Part 會為主鍵欄位自動建立 **`minmax`** 索引，例如：
```sql
SELECT * FROM orders WHERE order_date >= '2025-01-01' AND order_date momo 未來優化
#  Cost Managment
## **TTL 資料清理與儲存成本優化**
隨著時間的資料量成長，如何免去手工、使用自動化進行過期資料清理與儲存成本控制，成為大型數據系統設計中不可忽視的一環。ClickHouse 提供了 **TTL（Time To Live）資料清理機制**，不僅能自動刪除過期資料，還能將資料移動至冷儲存 (如 S3、HDD)，有效降低儲存成本
### 概念
TTL (Time To Live) 是指設定資料的「生命週期」，當資料達到指定條件時，ClickHouse 會自動進行清理（刪除）或儲存層級移動 (Move to Volume)
1. **整行資料 (Row TTL)** : 自動刪除過期資料
2. **欄位資料 (Column TTL)** : 針對指定欄位進行清理
3. **Volume TTL** : 將資料從 SSD 移至 HDD/S3 等不同 Volume 以降低儲存成本
### 分類
** Row TTL **
自動刪除過期資料
```sql
CREATE TABLE events
(
EventDate DateTime,
UserID UInt64,
Action String
) ENGINE = MergeTree
PARTITION BY toYYYYMM(EventDate)
ORDER BY (UserID, EventDate)
TTL EventDate + INTERVAL 7 DAY;
```
這會讓資料在 **`EventDate`** 超過 7 天後，自動被清除 **Column TTL **
指定欄位過期清除
```sql
CREATE TABLE logs
(
EventDate DateTime,
UserID UInt64,
Action String,
TempField String TTL EventDate + INTERVAL 1 DAY
) ENGINE = MergeTree
ORDER BY UserID;
```
**`TempField`** 欄位資料會在 1 天後被清除，但行資料仍會保留 **Volume TTL **
自動分層存儲 (Hot → Cold Storage)
```xml


/var/lib/clickhouse/ssd/


/var/lib/clickhouse/hdd/


disk_ssd


disk_hdd
5000000000


```
```sql
CREATE TABLE events_tiered
(
EventDate DateTime,
UserID UInt64,
Action String
) ENGINE = MergeTree
ORDER BY (UserID, EventDate)
SETTINGS storage_policy = 'tiered_policy'
TTL EventDate + INTERVAL 7 DAY TO VOLUME 'cold';
```
- 前 7 天資料會放在 SSD (Hot)
- 超過 7 天後資料會自動移動到 HDD (Cold)
TTL 不只是過期資料清理，更是**控制 ClickHouse 儲存資源分層 (SSD → HDD → S3)** 的重要利器。適當設計 TTL 策略，能幫助你在效能與成本之間取得最佳平衡
## **儲存政策（Storage Policies）與磁碟資源分層策略**
當你的 ClickHouse 資料規模從 GB、TB 成長到 PB 時，**如何妥善分配 SSD、HDD、甚至雲端冷儲存資源**，變得至關重要。ClickHouse 透過 **Storage Policies (儲存政策)**，提供了極為靈活的磁碟分層架構，不僅能優化查詢效能，也能大幅降低儲存成本
### Storage Policies
Storage Policies 是 ClickHouse 內部管理資料儲存位置與分層邏輯的配置機制。它將磁碟資源劃分為不同層級（Volumes），並根據資料大小、TTL、Merge 條件等動態將資料移動至不同層級磁碟
1. 熱資料存在 SSD，冷資料自動移至 HDD 或雲端 S3
2. 根據 Data Part 大小動態調度存儲位置
3. 搭配 TTL 策略，實現資料生命週期全自動管理
### **Storage Policies 結構**
```xml


/var/lib/clickhouse/ssd/


/var/lib/clickhouse/hdd/


s3
https://s3.amazonaws.com/your-bucket/
YOUR_KEY
YOUR_SECRET


disk_ssd


disk_hdd
5000000000


disk_s3


```
### **磁碟分層（Volumes）設計原則**


| 層級
| **磁碟類型**
| **適用資料**
| **說明**
|


| Hot
| SSD
| 近 7 天高頻查詢資料
| 保證讀取速度與低延遲
|


| Cold
| HDD
| 歷史數據或低頻查詢資料
| 儲存成本較低，適合冷資料
|


| Archive
| S3
| 歸檔資料、不常查詢但需長期保留
| 跨區域備份、儲存無上限、成本最低
|


其中可以透過 TTL 實現自動冷熱資料的分層


```sql
CREATE TABLE user_logs
(
EventDate DateTime,
UserID UInt64,
Action String
) ENGINE = MergeTree
ORDER BY (UserID, EventDate)
SETTINGS storage_policy = 'tiered_policy'
TTL EventDate + INTERVAL 7 DAY TO VOLUME 'cold',
EventDate + INTERVAL 30 DAY TO VOLUME 'archive';
```
### Best Practice
1. 分層設計要可以自動運作 ，不需要手動移動資料
2. 選擇適合的磁碟路徑與掛載點，SSD 用於熱資料、HDD 儲存冷資料、S3 用來歷史歸檔
3. 搭配 TTL 做時間序列資料的管理，進行自動清理或降層儲存
4. 定期檢查 Part 移動情況與執行效能


- 近 7 天資料會在 SSD。
- 7 天到 30 天資料移動到 HDD。
- 超過 30 天資料會自動移到 S3 儲存。
```sql
SELECT
name AS table_name,
disk_name,
count() AS parts
FROM system.parts
WHERE active AND table = 'user_logs'
GROUP BY table_name, disk_name;
```


#  Additional info
Introduction to Time Series Data
Row-oriented Database v.s. Column-oriented Database
## Reference
---
**Clickhouse official docs**

**article for clickhouse with Chinese**


**MergeTree Engine**

**Difference Table Engine in clickhouse**


**Comparsions between Clickhouse and BigQuery**

**Integrating dbt and ClickHouse**

**Partitioning in ClickHouse**

**Shards and Replication in ClickHouse**

**Multi-Node ClickHouse Cluster with Docker and Zookeeper**



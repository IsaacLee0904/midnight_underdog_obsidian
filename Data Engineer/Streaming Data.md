# Streaming Data

tags: #streaming #big-data #hadoop #data-engineering
source: [[Isaac's Note]]

---

## Introduction to Streaming Data

### What is Streaming Data?

Streaming Pipeline（串流數據管道）是實現 **Stream Processing** 的完整架構體系。它包含了從數據接收、處理、到輸出的整個流程，讓我們能夠建構出穩定可靠的即時數據處理系統。相比於傳統的 batch processing (批次處理)，stream processing 具有**消除延遲**與**提供即時決策能力**的優勢，並且具有以下的特色：

1. **即時處理**：數據一進入系統就立刻被處裡，延遲通常在秒級甚至毫秒級
2. **連續性**：持續不斷地處理數據流，而非等待批次的累積
3. **事件驅動**：基於事件觸發處理邏輯，適合實時決策場景
4. **狀態管理**：能維護處理過程中的狀態，支援複雜的聚合運算

### Streaming Data Technology

Streaming Pipeline 的技術架構：

1. **數據源（Data Sources）**：應用日誌、IoT 感測器、用戶行為事件、資料庫變更等
2. **消息佇列（Message Queue）**：Kafka、Pulsar 等，提供可靠的數據緩衝與分發
3. **流處理引擎（Stream Processing Engine）**：這是整個架構的核心，負責執行實際的 Stream Processing 邏輯，如 Flink、RisingWave、Spark Streaming 等
4. **狀態存儲（State Store）**：Stream Processing 需要維護狀態來支援 windowing、aggregation 等操作
5. **輸出端（Output Sinks）**：處理結果的目的地，如資料庫、數據倉儲、即時儀表板、告警系統等

### Stream Processing Engine 演進之路

大數據的技術可以分為四個層次：

1. **數據採集**：數據採集是大數據技術的基礎，這些數據的來源包括傳感器(IoT)、社交媒體、行動裝置以及日誌文件等。
2. **數據存儲與管理**：傳統的關聯數據庫缺乏水平擴展的能力，因此出現了分散式的存儲系統，根據數據存儲的層級，又可以分為**分散式檔案存儲系統 (Distributed File System)** 與**分散式資料庫 (Distributed Database)**。
3. **數據處理與分析**：採用分散式處理框架來進行**批處理**或**流處理**。數據經過 ETL 後，搭配統計學、機器學習等技術，即可從數據中提取出有價值的資訊。
4. **數據隱私與安全**：數據加密、權限控制、日誌監控等也是相當重要的一環。

---

## Big Data Compute: Batch Processing & Stream Processing

### MPP (Massively Parallel Processing)

**大規模並行處理**（MPP）是一種計算架構，專門設計用於處理大規模數據集和複雜分析工作負載。例如 Greenplum, Clickhouse, Presto/Trino, Impala。

MPP 系統的關鍵特性：
- **共享無（Shared-Nothing）架構**：每個節點擁有自己的資源，不與其他節點共享
- **數據分割（Data Partitioning）**：數據被水平分割並分佈到多個節點
- **並行執行**：查詢同時在多個節點上執行，然後合併結果
- **線性可擴展性**：增加節點可近乎線性地提高性能
- **高可用性**：某個節點失敗不會導致整個系統崩潰

### Bounded Data & Unbounded Data

在數據流處理中，**有界數據 (Bounded Data)** 和 **無界數據 (Unbounded Data)** 是兩個重要的概念，這裡的界是**時間邊界**，指數據流的時間終點。

- 有界數據的資料是**靜止不動**的，與之對應的處理方式稱為**批處理 (batch processing)**。
- 無界數據的資料是**連續**且**持續變動**的，與之對應的處理方式稱為**流處理 (stream processing)**。

### Batch Processing

批次處理是**先蒐集完數據再進行計算**，以**批 (batch)** 為單位處理資料，一般來說速度較慢，通常是使用在離線的數據處理上。

> 常見的批次處理計算：MapReduce、Spark

> **💡 注意**：Spark Streaming 屬於**微批處理**，本質上還是批處理，差別在於將每個 batch 切得更小，以此來加快每批數據的處理速度，進而達到近似流處理的效率。

### Stream Processing

流處理須兼顧無界數據「連續」且「持續變動」兩個特點，**數據的蒐集與計算是同時發生的**，通常是使用在及時的數據處理上，如交易系統、串流分析與日誌分析等，關注的重點在於**低延遲**。

> 常見的流處理計算：Flink、S4、Storm、Samza

> **💡 注意**：**Apache Kafka** 本身不屬於流處理計算，而是一個**分散式的訊息佇列**，用於可靠地接收、存儲和分發數據消息，因此常與流處理計算引擎搭配使用。

### Special Processing

- **圖計算**：針對圖結構數據的處理，常見技術有：Pregel、Graphx、PowerGraph、Hama、GoldenOrb 等
- **交互查詢計算**：針對數據存儲與查詢的處理，常見技術有：Dremel、Hive、Cassandra、Impala 等

---

## Introduction to Hadoop

> 詳細內容見 [[Hadoop]]

Hadoop 是一個開源的分散式存儲和處理框架，透過將多個機器結合成群集（cluster），充分利用機器的本地存儲與計算資源，以平行計算的方式來處理資料。

**Hadoop 生態系**：
- **HBase**：基於 HDFS 上 key-value pair 型態的 storage
- **YARN**：作為資源管理系統，允許任何分散式的程式在 Hadoop 集群上運行
- **HDFS**：分散式文件管理系統，支持超大型文件存儲（TB、PB 以上）
- **MapReduce**：分散式計算的程式設計模型，Divide and Conquer 概念應用於巨量數據處理

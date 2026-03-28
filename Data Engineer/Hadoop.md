# Hadoop

tags: #big-data #hadoop #data-engineering
source: [[Isaac's Note]]

---

## Introduction to Big Data

### What is Big Data？

大數據是指無法使用傳統數據處理應用在可接受的時間範圍內處理的數據集合。這些數據具有以下特徵，通常稱為 **4V 特性**：

1. **Volume（數量）**：數據規模巨大，從 TB 到 PB 甚至更多。例如，Facebook 每天處理超過 500TB 的數據；Walmart 每小時處理超過 1 百萬筆客戶交易。
2. **Variety（多樣性）**：數據類型多樣，包括結構化數據（如關聯數據庫中的表格數據）、半結構化數據（如 XML、JSON 文件）和非結構化數據（如圖像、視頻、社交媒體內容）。
3. **Velocity（速度）**：數據生成和處理的速度非常快。例如，Twitter 每秒產生超過 6,000 條推文；自動駕駛汽車每秒可產生約 1GB 的傳感器數據。
4. **Veracity（真實性）**：數據可能存在不確定性、不一致性或不完整性，需要專門的處理方法來確保數據質量。

大數據技術的主要目標是解決傳統數據處理技術在處理這類數據時面臨的挑戰，包括存儲、管理、處理和分析。

> **大數據下就會遇到相對應的挑戰**
> 1. 存儲空間的不足（硬體需要擴充越來越大）→ 如果硬碟壞掉可能會導致資料遺失 → 解決方式：**Replication** 的機制
> 2. 讀寫速度跟不上 → 任務上可能會有一些需要 combine 資料的時刻，這時候就會面臨挑戰 → 解決方式：Hadoop 的 **MapReduce**

### Big Data Technology

大數據的技術可以分為四個層次：

1. **數據採集**：數據採集是大數據技術的基礎，這些數據的來源包括傳感器（IoT）、社交媒體、行動裝置以及日誌文件等。
2. **數據存儲與管理**：傳統的關聯數據庫缺乏水平擴展的能力，因此出現了分散式的存儲系統，可分為**分散式檔案存儲系統（Distributed File System）**與**分散式資料庫（Distributed Database）**。
3. **數據處理與分析**：採用分散式處理框架來進行**批處理**或**流處理**。數據經過 ETL 後，搭配統計學、機器學習等技術，即可從數據中提取出有價值的資訊。
4. **數據隱私與安全**：數據加密、權限控制、日誌監控等也是相當重要的一環。

---

## Big Data Compute：Batch Processing & Stream Processing

### ETL

### MPP（Massively Parallel Processing）

**大規模並行處理**（MPP）是一種計算架構，**專門設計用於處理大規模數據集和複雜分析工作負載**。常見例子：Greenplum、Clickhouse、Presto/Trino、Impala。

**基本概念與架構：**
- MPP 系統由多個獨立的處理節點組成，每個節點有自己的 CPU、內存和存儲
- 節點之間通過高速互連網絡通信，協同工作
- 數據按照特定策略分佈在多個節點上
- 查詢被拆分為多個子任務，在多個節點上並行執行

**MPP 系統的關鍵特性：**
- **共享無（Shared-Nothing）架構**：每個節點擁有自己的資源，不與其他節點共享
- **數據分割（Data Partitioning）**：數據被水平分割並分佈到多個節點
- **並行執行**：查詢同時在多個節點上執行，然後合併結果
- **線性可擴展性**：增加節點可近乎線性地提高性能
- **高可用性**：某個節點失敗不會導致整個系統崩潰

---

### Bounded Data & Unbounded Data

在數據流處理中，**有界數據（Bounded Data）**和**無界數據（Unbounded Data）**是兩個重要的概念，這裡的「界」是**時間邊界**，指數據流的時間終點，可以把它想像成一個「閘門」：

- **有界數據**的資料是**靜止不動**的，對應的處理方式稱為**批處理（batch processing）**
- **無界數據**的資料是**連續**且**持續變動**的，對應的處理方式稱為**流處理（stream processing）**

---

### Batch Processing

前面提到有界數據是靜止不動的，因此我們在處理時會**先蒐集完數據再進行計算**，以**批（batch）**為單位處理資料，一般速度較慢，通常用於離線的數據處理上。

常見的批次處理計算：**MapReduce**、**Spark**

> **💡 注意**
> Spark Streaming 屬於**微批處理**，本質上還是批處理，差別在於將每個 batch 切得更小，以此來加快每批數據的處理速度，進而達到近似流處理的效率。

---

### Stream Processing

流處理須兼顧無界數據「連續」且「持續變動」兩個特點，**數據的蒐集與計算是同時發生的**，通常用於即時的數據處理，如交易系統、串流分析與日誌分析等，關注的重點在於**低延遲**。

常見的流處理計算：**Flink**、S4、**Storm**、Samza

> **💡 注意**
> **Apache Kafka** 本身不屬於流處理計算，而是一個**分散式的訊息佇列**，用於可靠地接收、存儲和分發數據消息，因此常與流處理計算引擎搭配使用。

---

### Special Processing

**圖計算**：針對圖結構數據的處理，常見技術：Pregel、Graphx、PowerGraph、Hama、GoldenOrb

**交互查詢計算**：針對數據存儲與查詢的處理，常見技術：Dremel、Hive、Cassandra、Impala

---

## Introduction to Hadoop

### Hadoop 簡介

Hadoop 是一個開源的分散式存儲和處理框架，提供了一個可靠可延展的平台，常用於巨量資料集的處理。**透過 Hadoop，我們能將多個機器結合成群集（cluster）**，充分利用機器的本地存儲與計算資源，並能夠以平行計算的方式來處理資料。透過將任務送到 MapReduce 來進行批次的處理（因此不適合做交互式的分析），也因此 Hadoop 衍伸出了各種生態系統的工具。

### Hadoop vs RDBMS

| | RDBMS | Hadoop |
|---|---|---|
| 儲存機制 | B-Tree | sort / Merge |
| 結構化機制 | **Schema on write** | **Schema on read** |
| 政策 | ACID | |
| 讀寫狀況 | 可以支援頻繁 read / write | read（**資料都會寫入硬碟**） |
| 資料量級 | GB | PB |

- **Schema on write**：要依循一定的 schema 來寫入，並且是結構式的。例：在寫入數據前要先定義欄位的名稱、datatype、長度等資訊
- **Schema on read**：Hadoop 並沒有要求必須是結構型的，可以是半結構型的資料，依賴於客戶端讀取數據時對數據進行類型上的處理

### Hadoop 生態系

- **HBase**：基於 HDFS 上 key-value pair 型態的 storage，提供了較好的交互式訪問系統
- **YARN**：Hadoop 2 之後出現的資源管理系統，減少了 MapReduce 的壓力，**允許任何分散式的程式在 Hadoop 集群上運行**，因此有了以下功能：
  - 交互 SQL 與**分散式查詢工具**：如 Impala、Hive、Tez
  - 循環反覆處理（如 ML）時可藉由 **Spark** 把中間步驟儲存在內存上
  - 流處理，達到即時的效果：如 Storm、Spark

---

## Hadoop 架構

### HDFS

HDFS 是基於**流式數據訪問**的**分散式文件管理系統**，支持超大型文件存儲（TB、PB 以上），透過網路進行連接，HDFS 作為一抽象層建構在集群網路上，對文件進行統一管理。優點：**高容錯性**、**高擴展性**以及**高可用性**。

架構上，HDFS 屬於典型的**主從架構**，主要由 3 種節點構成：

- **NameNode**
- **SecondaryNameNode**
- **DataNode**

HDFS 會將文件切成**數據塊（block）**，分散存儲在各個 DataNode，並透過 NameNode 來管理這些數據塊。

**Block**：是數據讀寫的最小單位，目的是為了要儲存大文件。相比於文件，block 更容易做 replication，也可以更好地管理 storage。

#### HDFS 的架構

**NameNode**
- 是主從架構中的主節點，負責管理整個檔案系統的目錄與文件，**儲存 metadata**
- 由兩個文件組成：**FSImage（File System Image）**與 **Edit Log**，前者是整個系統的完整快照，後者是文件系統的更新日誌
- 由於 FSImage 龐大，每次數據更新都直接更新 FSImage 非常耗時，所以會先更新至 Edit Log，之後再透過 SecondaryNameNode 合併

**SecondaryNameNode**
- 是 NameNode 的輔助節點，負責 FSImage 與 Edit Log 的合併
- **不是** NameNode 的熱備份，**不能**在 NameNode 故障時直接接管工作

**DataNode**
- 是主從架構中的從節點，負責數據塊的 I/O 操作
- 一個數據塊通常會被存放在不只一個 DataNode 中，當 client 將數據塊寫入一個 DataNode 後，該 DataNode 會將數據塊也寫入其他 DataNode
- DataNode 也會定期向 NameNode 匯報數據塊的情況，有助於快速檢測故障、數據恢復

#### HDFS Data Flow

**讀取流程：**
1. Client 發出請求去 filesystem
2. Filesystem 會去 NameNode 取得 block 的資訊
3. 再透過 inputstream 去 DataNode 提取資料

**寫入流程：**
1. 先從 NameNode 去申請閒置的 block
2. 將資料拆分成 packet（更小塊），放在 packet queue 中
3. 再寫入 DataNode 中

#### HDFS 訪問方式

1. CLI：`hdfs dfs -<option>`
2. Java API：`FileSystem.class`
3. HTTP（webhdfs）：REST call

---

### MapReduce

MapReduce 是一種**分散式計算的程式設計模型（programming model）**，也是 Hadoop 預設的計算模式。可以把它想成應用在巨量數據處理的**分而治之（Divide and Conquer）**，有 **Map** 與 **Reduce** 兩個主要階段。

**具體步驟：**

1. **Split**：Hadoop 會將數據**切割為等長的 input**（叫做 split），並對應到 MapReduce 的 task
2. **Map**：有一個 map function，從 split 中提取所需的資料，**進行初步的計算**，並將結果轉換為**鍵值對（key-value pair）**的形式
   - 例：map function 提取年份與溫度 → `(1996, 20), (1996, 30), (1997, 27), (1997, 29)`
   - input 與 output 的 key 可以不用一致
3. **Shuffle and Sort**：將前步驟的 map **根據框架進行分組與排序**，使**相同鍵的鍵值對能夠被連續處理**
   - 例：`(1996, [20, 30]), (1997, [27, 29])`
4. **Reduce**：對已分組的 map **進行聚合操作**，並產出結果
   - 例：reduce function 找最大值 → `(1996, 30), (1997, 29)`

#### Map Tasks

- MapReduce 的 job 可以被 Hadoop 拆分為許多 tasks（maptask、reducetask），這些 task 會被運行在不同的 node 上
- Input 會被 Hadoop 分割成等長的 input 項目（hadoop split 的概念），指派給不同的 task
- Split 與 task 是 1 對 1 的關係
- YARN 負責在不同 node 上面協調 task

> **💡 Data Locality Optimization（數據本地優化）**
> 指的是 Hadoop 如何讓這些 task 運行在不同的 node 上，又能得到想要的資料。因為本地的數據傳輸一定是最快的，所以會優先使用本地數據，盡量避免跨機器 → 跨 rack。

#### Reduce Tasks

- 不是必須的，如果 map task 已經是想要的結果了，就不一定會有 reduce task
- 1-to-many map tasks

**Combiner Function 預處理**：在 map task 與 reduce task 之間，在將資料儲存到 local disk 之前，先做一個取最大值的動作，透過這樣的方式**減少 reduce 時候的資料量**。

---

### YARN（Yet Another Resource Negotiator）

Hadoop 的資源管理原先由 MapReduce 負責，直到 Hadoop 2.0 後才獨立出 YARN。YARN 是 Hadoop 的**資源管理系統**，讓 MapReduce 能專心做數據相關的計算。

YARN 採用主從架構，主要由以下組件構成：

**ResourceManager（資源管理器）**
- 是主從架構中的主節點，**負責整個系統的資源管理與調度**，包含：
  - **ResourceScheduler**：負責資源的調度，將系統資源分配給應用程序
  - **ApplicationManager**：負責管理系統中的應用程序，包括提交、監控等

**NodeManager（節點管理器）**
- 是主從架構中的從節點，負責每個節點上的資源與任務管理

**ApplicationMaster（應用程式管理器）**
- 負責與 ResourceManager 協商資源，並與 NodeManager 溝通以執行所有任務
- 負責監控應用程式的執行狀況，並向 ResourceManager 回報狀態

**Container（容器）**
- 概念跟 Docker 中的容器差不多，將特定資源進行封裝，是節點中實際執行任務的地方

YARN 不只能與 MapReduce 搭配，也是一個通用的資源管理系統，多種大數據處理框架都能部署其上，如：**Spark on YARN**、**Flink on YARN**。

---

## Hive

### Hive Introduction

Hive 是一個運行在 HDFS 上的**數據倉庫（Data Warehouse）**，資料都儲存在 HDFS 上，透過 hive metastore 存儲有關表格結構、分區資訊等 metadata，將數據結構化為表達形式，使用 **HiveQL** 來進行數據的查詢與分析。

由於將 SQL 查詢轉換為 MapReduce 作業，因此查詢性能較慢（與 HBase 相比），多半用於大數據的批次處理上。

### Hive 架構

Hive 是建構在 HDFS 與 MapReduce 之上的應用，用戶可以透過 **Hive web UI**、**CLI** 或是 **Hive Client** 來操作 Hive。

**Hive Client**：外部應用程序與 Hive 建立遠程訪問的接口，支援多種程式語言，包含：
- **Thrift Server**：用來與其他程式語言溝通
- **JDBC Driver**：用來與 Java 溝通
- **ODBC Driver**：用來與其他資料庫溝通

**Hive Server**：接收 Hive Client 的請求並將其轉交給 Hive Driver

**Hive Driver**：接收 Hive Client、Hive web UI 與 CLI 的查詢並將這些查詢傳遞給編譯器

**Hive Metastore**：**用於存儲有關表格結構、分區信息、列和數據位置的元數據（metadata）**

---

## Reference

- YouTube：Hadoop 概念（中文）
- YouTube：MapReduce 概念（中文）
- Hadoop ecosystem tutorial

# Redshift

tags: #redshift #aws #data-warehouse #data-engineering
source: [[Isaac's Note]]

---

## Introduce to Redshift

### Redshift Basic

Amazon Redshift 是 AWS 提供的雲端級資料 data warehouse 服務，底層雖以 PostgreSQL 為基礎，但架構已針對 OLAP 的使用情境進行了改造，具備**行式儲存 (Columnar Storage)** 與**大規模平行處理 (MPP)** 的架構設計，Redshift 有 provisioned cluster 與 Serverless 兩種模式。

**Provisioned cluster (自行管理叢集)**

自行建立與管理 cluster，優點是高度的客製化（可調節 node 數量、大小等）並且有較高的資源掌控度方便成本計算，為 master-slave cluster 架構。

每一個 cluster 由多 nodes（leader node and compute node）所構成：
- **leader node**：負責 cluster 任務的協調與管理，例如接收使用者查詢、生成執行計畫，並分派給各個 Compute Node 執行
- **compute node**：是 cluster 處理資料與運算工作的核心
- **slices**：是 compute node 再細分成的數個單位，也就是 Redshift 中執行查詢的最小運算單位

**Serverless (無伺服器)**

AWS 全權管理資源配置與擴展，使用者只需專注於執行 SQL 查詢，優點是不需要維護底層架構，快速啟動、彈性高。

Serverless 的系統有兩個主要的服務：
- **workgroup**：用來處理運算資源的配置、網路連線與安全性設定相關工作
- **namespace**：用來**管理資料的虛擬容器**，其中包含多個資料表（tables）與綱要結構（schemas），主要用途是作為整合資料來源的邏輯單位

Serverless 架構做到了存算分離，將原本 compute node 把運算 CPU 與儲存硬碟綁在一起的做法優化：
1. 最小計價與執行單元為 RPU（Redshift Processing Unit）
2. 一個 Workgroup 可以對應多個 Namespace，但每個 Namespace 僅能連結到一個特定的 Workgroup
3. 這樣的架構讓 Serverless 模式具備高度彈性，並能在無需維護叢集資源的情況下，完成資料倉儲任務

> **💡 Amazon Redshift 主機類型**
> 目前主要有兩種節點類型可供選擇：**RA3** 與 **DC2**
>
> **DC2 (Dense Compute)**
> - DC2 是早期的 Redshift 節點類型，主要強調高效能與 I/O 密集的處理能力
> - 儲存與運算綁定在一起：每個節點的運算能力與儲存空間是固定綁定的
> - 成本較低、簡單好管理
>
> **RA3**（建議使用）
> - RA3 是目前建議使用的主流節點類型，設計上更為靈活與擴充
> - **儲存與運算分離架構 (Decoupled Storage and Compute)**，可以根據分析需求擴增計算節點，而儲存空間則動態管理於 Amazon Redshift Managed Storage (RMDS)
> - 適合大規模資料湖、彈性查詢量、與動態擴展需求的企業架構

---

## Architecture

Amazon Redshift 的底層技術與傳統資料庫密切相關，從底到上，Redshift 架構大致可以分為三層：**PostgreSQL 核心**、**PL/pgSQL 中介層**、以及 **Stored Procedure 應用層**。

### Stored Procedure (儲存程序)

為最上層直接給 user 互動的部分，可以視為是「打包並程式化一段資料流程」或者「一段可重複使用的小型程式」。

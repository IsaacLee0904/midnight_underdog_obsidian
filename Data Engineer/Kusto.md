# Kusto (KQL)

tags: #kusto #kql #azure #data-engineering
source: [[Isaac's Note]]

---

## What is Kusto？

Kusto（及其商業化版本 Azure Data Explorer，簡稱 ADX）是一種服務，可用於儲存和執行巨量資料的互動式分析。它**擁有海量數據的存儲和計算能力**，同時提供接近實時的**即時查詢能力**。

資料來源支援：
- 從數據湖中的大量日誌文件導入
- 與 `IoT Hub` 或 `Event Hub` 配合進行串流數據導入
- 直接查詢來自 `SQL Server` 或 `Cosmos DB` 中的結構化或非結構化數據

所有數據不管格式和來源，都可以用 **KQL（Kusto Query Language）** 進行即時分析。

### Kusto 特色

- 超大規模數據支援（存算分離，**分散式**叢集）
- 支援不同的數據來源（結構化、非結構化、半結構化）
- TB 級別數據量分鐘級別傳輸
- PB 級別數據查詢，秒級回應
- 超級好用的查詢語言（KQL）：啟發式和管道式、智慧提示、安全（查詢和管理命令分開）
- 原生的高級分析功能：內建**機器學習函數**（時序分析、異常檢測、回歸和預測等）、支援嵌入 Python / R 程式碼

---

## KQL 基本語法

### `count` — 計算筆數

```sql
StormEvents
| count

StormEvents
| where State == "NEW YORK"
| count
```

### `limit` / `take` — 限制回傳筆數

```sql
StormEvents
| limit 100

StormEvents
| take 100
```

### `project` — 篩選欄位

```sql
StormEvents
| take 10
| project StartTime, EndTime, EpisodeId
```

### `distinct` — 顯示唯一值

```sql
StormEvents
| distinct EventType
```

### `sort` — 排序（遞增加 `asc`）

```sql
StormEvents
| project StartTime, EndTime, State, EventType, DamageProperty
| where State == 'TEXAS' and EventType == 'Flood'
| sort by DamageProperty
```

### `where` — 條件篩選

```sql
StormEvents
| where State == 'TEXAS' and EventType == 'Flood'
| project StartTime, EndTime, State, EventType, DamageProperty
```

### `between` — 時間區間查詢

```sql
StormEvents
| where StartTime between (datetime(2007-08-01 00:00:00) .. datetime(2007-08-30 23:59:59))
| project State, EventType, StartTime, EndTime
| sort by StartTime asc
```

### `extend` — 新增計算欄位

```sql
StormEvents
| where State == 'TEXAS' and EventType == 'Flood'
| top 5 by DamageProperty desc
| extend Duration = EndTime - StartTime
```

### `let` — 定義變數

```sql
let sourceMapping = dynamic(
  {
    "Emergency Manager" : "Public",
    "Utility Company" : "Private"
  });
StormEvents
| where Source == "Emergency Manager" or Source == "Utility Company"
| project EventId, Source, FriendlyName = sourceMapping[Source]
```

---

## KQL 聚合語法

### `summarize`

```sql
StormEvents
| summarize TotalStorms = count() by State
```

### `render` — 可視化

```sql
StormEvents
| summarize TotalStorms = count() by State
| render barchart
```

### `countif` — 條件計數

```sql
StormEvents
| summarize StormsWithCropDamage = countif(DamageCrops > 0) by State
| top 5 by StormsWithCropDamage
```

### `bin` — 分組數值/時間

```sql
StormEvents
| where StartTime between (datetime(2007-01-01) .. datetime(2007-12-31))
    and DamageCrops > 0
| summarize EventCount = count() by bin(StartTime, 7d)
```

### `max` / `min` / `avg`

```sql
StormEvents
| where DamageCrops > 0
| summarize
    MaxCropDamage=max(DamageCrops),
    MinCropDamage=min(DamageCrops),
    AvgCropDamage=avg(DamageCrops)
    by EventType
| sort by AvgCropDamage
```

### `join`

```sql
StormEvents
| summarize PropertyDamage = sum(DamageProperty) by State
| join kind=innerunique PopulationData on State
| project State, PropertyDamagePerCapita = PropertyDamage / Population
| sort by PropertyDamagePerCapita
```

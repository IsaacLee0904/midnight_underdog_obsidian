# Pyspark

tags: #pyspark #spark #big-data #data-engineering
source: [[Isaac's Note]]

---

## What is Pyspark?

PySpark 顧名思義，也就是 Python 的一個 Spark Library，主要是利用 Python 語法結合 Spark 的框架，是現在很主流的一個處理**大量資料**的框架之一。

### What is Apache Hadoop?

Hadoop 是一個分散式的架構，是一個 cluster 的概念，針對大型的資料進行分散式的儲存與處理。你有一個**主要機器 (Master Node)**，搭配非常多的**小機器 (Slave Node)** 來儲存，當你機器一多，就可以將 File 分成好幾份，也能同時再複製幾份儲存在不同的小機器中，確保資料的可靠性。

### What is Apache Spark?

> Apache Spark 是用於大規模資料處理的整合數據分析引擎，內建 SQL、串流、機器學習和圖形處理等多種模組。

1. **Speed**: Run workloads 100x faster
2. **Ease of Use**: Write applications quickly in Java, Python, R, and SQL
3. **Generality**: Combine SQL, streaming, and complex analytics
4. **Runs Everywhere**: Spark runs on Hadoop, Apache Mesos, Kubernetes, standalone, or in the cloud

### Lazy Execution

與一般程式語言一行一行執行不一樣，在 PySpark 中，當您定義轉換操作時，它們不會立即執行，而是構建一個執行計劃，直到遇到一個 action 操作（如 `show()`、`collect()`、`count()`）時才真正執行，這就是惰性執行。

**好處**：
1. **優化執行計劃**：Spark 可以分析整個執行計劃，自動優化操作順序、合併操作，並消除不必要的步驟
2. **提高效能**：只計算最終結果所需的數據，避免中間結果物化，降低 I/O 操作
3. **記憶體管理優化**：只將必要的數據加載到記憶體中，避免 OOM 錯誤
4. **容錯與重算**：通過記錄操作而非數據，可以在節點失敗時重新計算
5. **分布式計算優化**：可以優化 shuffle 操作，減少數據在節點之間的移動

### DAG

Spark 的任務執行是遵循 **DAG (有向無環)** 的模式，每個事件的順序可以跟動作的不同階段相關，在這個單向的階段序列中不存在循環。

### Shuffle

Shuffle 在 Spark 中是一種**數據重分配機制**，是 Spark 集群的不同 node 上重新分配數據的過程，當執行 action 操作（`groupByKey`、`reduceByKey`、`join` 等），Spark 會觸發 shuffle 的機制。

Shuffle 分為兩個階段：
- **shuffle write（寫出階段）**：Map 任務處理輸入數據，並根據目標分區標準將輸出資料寫入本地硬碟
- **shuffle read（讀取階段）**：Reduce 任務從所有 map 任務 node 獲取屬於自己的資料，並進行聚合或處理

> **💡 Shuffle 優化策略**
> - **減少數據量**：在進行 join 之前先進行數據過濾，使用 `mapPartitions` 替代 map 減少對象創建
> - **調整分區數量**：分區過少會導致並行度不足，分區過多會產生過高的調度開銷
> - **使用高效的操作**：使用 `reduceByKey` 取代 `groupByKey`（前者在 map 端進行合併）

### RDD

RDD (Resilient Distributed Datasets) 是 Spark 架構中的重要抽象層，是一種不可變、分區、可平行處理的資料集合：
- **不可變（read-only）**：存儲的訊息無法被更改，如果要更改只能對現有 RDD 進行轉換操作
- **分區**：每個 RDD 都會被切分為一個或多個分區，存儲在群集中的多個節點上
- **平行處理**：各個節點上的數據可以被分開處理

**RDD 的依賴模式**：
- **Narrow Dependency 窄依賴**：轉換具有一對一的映射關係，不需要跨分區的數據溝通，如 `map()`、`filter()`
- **Wide Dependency 寬依賴**：部分操作會觸發 Shuffle 機制進行跨分區的數據溝通，如 `reduceByKey()`、`groupByKey()`

> 整體來說寬依賴的資源消耗是更嚴重的，因此在進行操作時應以窄依賴優先

**RDD 的操作方式**：
- **Transformation 轉換**：從現有 RDD 創建一個新的 RDD，屬於 Lazy Operation。常見的有：map、flatMap、filter、distinct、union、reduceByKey 等
- **Action 動作**：在對數據集運行計算後，返回一個值給驅動程序。常見的有：foreach、saveAsTextFile、collect、count、top、reduce 等

### Data Skew

Pyspark 是建構在平行運算的架構上，所謂資料傾斜就是指**資料放置位置不平均**。

**如何處理**：
1. 避免會造成 data skew 的操作：`distinct`、`groupByKey`、`reduceByKey`、`aggregateByKey`、`join`
2. 設計為一個 key 去做資料的讀取
3. 使用 `repartition` 來重新訂給分區位置

```python
import pyspark.sql.functions as F
df = df.withColumn('apple', F.rand())
df = df.repartition(5, 'apple')
```

---

## Create Spark Session

```python
from pyspark.sql import SparkSession
import pyspark.sql.functions as F

spark = SparkSession.builder \
    .appName("test") \
    .master("local[*]") \
    .getOrCreate()
```

`SparkSession.builder.master()` 選項：
- `local` - 在本地使用 1 個執行緒運行 Spark
- `local[k]` - 在本地使用 k 個執行緒運行 Spark
- `local[*]` - 在本地使用與 CPU 核心數量相同的執行緒運行 Spark
- `spark://HOST:PORT` - 連接到指定的 Spark 獨立集群主節點
- `yarn` - 連接到 YARN 集群
- `k8s://HOST:PORT` - 連接到 Kubernetes 集群

**記憶體相關 config**：

```python
.config("spark.driver.memory", "4g")
.config("spark.executor.memory", "4g")
.config("spark.memory.offHeap.enabled", "true")
.config("spark.memory.offHeap.size", "2g")
```

**執行相關 config**：

```python
.config("spark.executor.instances", "2")
.config("spark.executor.cores", "2")
.config("spark.default.parallelism", "4")
.config("spark.sql.shuffle.partitions", "100")
```

**性能調優 config**：

```python
.config("spark.sql.adaptive.enabled", "true")
.config("spark.sql.adaptive.skewJoin.enabled", "true")
.config("spark.dynamicAllocation.enabled", "true")
.config("spark.serializer", "org.apache.spark.serializer.KryoSerializer")
```

---

## DataFrame - Load DataFrame

### Read CSV

```python
# Method 1
df = spark.read.option("header", "true") \
    .option("inferSchema", "true") \
    .option("delimiter", ",") \
    .csv("/path/to/file.csv")

# Method 2
df = spark.read.option("delimiter", ",") \
    .csv("/path/to/file.csv", header=True, inferSchema=True)
```

### Read JSON

```python
df = spark.read.option("multiline", "true").json("/path/to/file.json")
```

### Load from DB

```python
spark = SparkSession.builder \
    .appName("PostgreSQL Connection") \
    .config("spark.jars", "/path/to/postgresql-42.6.0.jar") \
    .config("spark.driver.extraClassPath", "/path/to/postgresql-42.6.0.jar") \
    .getOrCreate()

url = "jdbc:postgresql://localhost:5432/mydb"
db_df = spark.read.format("jdbc") \
    .option("url", url) \
    .option("dbtable", "schema.table_name") \
    .option("user", "user") \
    .option("password", "password") \
    .option("driver", "org.postgresql.Driver") \
    .load()
```

### Load with RDD

```python
from pyspark.sql import SparkSession

spark = SparkSession.builder.appName("practice").getOrCreate()
sc = spark.sparkContext

rdd = sc.parallelize([("Carmen", 23, "吉普賽女郎"), ("Don José", 25, "青年衛兵下士")])
df = rdd.toDF(["name", "age", "role"])
df.show()
df.printSchema()
```

### Load with Python DataFrame

```python
spark_df = spark.createDataFrame(pandas_df)
```

### Load with List

```python
list_values = [["Carmen", 23, "吉普賽女郎"], ["Don José", 25, "青年衛兵下士"]]
list_spark_df = spark.createDataFrame(list_values, ["name", "age", "role"])
list_spark_df.show()
```

---

## DataFrame - Basic Operation

### show()

```python
df.show()              # 顯示前 20 筆（預設）
df.show(5)             # 顯示前 5 筆
df.show(truncate=50)   # 不截斷（指定最大長度）
df.show(truncate=False) # 完全不截斷
df.show(vertical=True) # 垂直顯示，適合欄位多的情況
```

### printSchema()

```python
df.printSchema()  # 顯示 schema 和 DataType
```

### describe()

```python
df.describe().show()  # 顯示每個 column 的統計基本資訊
```

### shape()

```python
# pyspark 沒有原生 shape，用以下替代
print((df.count(), len(df.columns)))
```

---

## DataFrame - Select

### select() / selectExpr()

```python
# 選取特定欄位
df.select("character_id", "login_time").show()

# 選取所有欄位
df.select("*").show()

# 計算欄位
df.select(df.Name, (df.Age + 10).alias('age')).show()
df.select(mean('Age'), avg('Age'), sum('Age'), max('Age'), min('Age')).show()

# 取得單一值
first_id = df.select("character_id").first()["character_id"]

# 使用 selectExpr
df.selectExpr("sum(case when Age >= 25 then 1 else 0 end) as ageOver25").show()
df.selectExpr('min(date) AS min_date', 'max(date) AS max_date').show()
```

---

## DataFrame - Filter

### filter()

```python
# 單一條件
df.filter(df.date == "2024-01-01").show()

# 多條件
df.filter((df.date == "2024-01-02") & (df.gameworldid == 2)).show()

# SQL-like
df.filter("date = '2024-01-02' AND gameworldid = 2").show()
```

### where()

```python
df.where(F.col('date') == "2024-01-02").show()
df.where((F.col('date') == "2024-01-02") & (F.col('gameworldid') == 2)).show()
```

### limit()

```python
df.limit(5).show()
```

---

## DataFrame - Sample

### sample()

```python
# withReplacement=False, fraction=0.1, seed=42
df.sample(withReplacement=False, fraction=0.1, seed=42).show()
```

### sampleBy()

```python
# 按特定欄位進行分層抽樣
df.sampleBy("gameworldid", fractions={1: 0.5, 2: 0.5}, seed=42).show()
```

---

## Data Cleaning

### Timestamp 轉換

```python
# String → Timestamp
df = df.withColumn("login_time", F.to_timestamp(F.col("login_time")))

# Timestamp → Date
df = df.withColumn("login_date", F.to_date(F.col("login_time")))
```

### String 操作

```python
# split()
df.select(F.split(F.col("character_name"), " ")[0].alias("first_name")).show()

# substr()
df.select(F.substr(F.col("first_name"), F.lit(1), F.lit(1)).alias("first_initial")).show()
```

### Regular Expression

```python
# regexp_extract()
df = df.withColumn(
    "first_name",
    F.regexp_extract(F.col("character_name"), "^([A-Za-z]+)", 0)
)

# regexp_replace()
df = df.withColumn(
    "clean_name",
    F.regexp_replace(F.col("character_name"), "\\s+(DVM|DDS|PhD|MD|Jr\\.|Sr\\.)\\s*$", "")
)

# rlike()
df.filter(F.col("first_name").rlike("^M")).show()
```

### Array Type

```python
# explode() - 將 array 的每個元素展開為獨立的一行
df.select(df.name, df.college, F.explode(df.positions)).show()
```

### JSON

```python
# get_json_object()
df.select(
    get_json_object(df.jstring, '$.f1').alias("c0"),
    get_json_object(df.jstring, '$.f2').alias("c1")
).show()
```

---

## Data Transform

### withColumn()

```python
# 新增或更新欄位
df = df.withColumn(
    "clean_name",
    F.regexp_replace(F.col("character_name"), "^(Mr\\.|Mrs\\.|Dr\\.)\\s+", "")
)
```

### withColumnRenamed()

```python
df = df.withColumnRenamed("old_name", "new_name")
```

---

## Join

Join 類型：
- **Inner join** - `"inner"`
- **Left Outer Join** - `"left"` / `"leftouter"` / `"left_outer"`
- **Right Outer Join** - `"right"` / `"rightouter"` / `"right_outer"`
- **Outer Join / Full Join** - `"full"` / `"outer"` / `"fullouter"`
- **Left Semi Join** - `"semi"` / `"leftsemi"` / `"left_semi"`
- **Left Anti Join** - `"anti"` / `"leftanti"` / `"left_anti"`
- **Cross Join** - `"cross"`

```python
# inner join
inner_join_df = employees_df.join(departments_df, "dept_id", "inner")

# left join
left_join_df = employees_df.join(departments_df, "dept_id", "left")

# cross join
cross_join_df = employees_df.crossJoin(departments_df)
```

**多 join 複雜情境（使用 SQL）**：

```python
orders_df.createOrReplaceTempView("orders")
users_df.createOrReplaceTempView("users")

spark.sql("""
SELECT o.*
FROM orders o
JOIN users d ON o.driver_id = d.user_id AND d.is_banned = false
JOIN users p ON o.passenger_id = p.user_id AND p.is_banned = false
""").show()
```

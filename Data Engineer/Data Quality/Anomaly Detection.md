tags: #data-quality-check #data-engineering #etl
source: [Dagster Glossary — Anomaly Detection](https://dagster.io/glossary/anomaly-detection)

---

> Anomaly detection 用來識別資料中的**離群值與異常**，這些異常可能導致分析結果偏斜，或揭示資料本身潛在的問題

在 [[Data Quality Patterns]] 的 WAP Pattern 中，Anomaly Detection 是 **Audit 階段**的核心技術之一，用統計方法檢測 outliers 或意外的資料週期

---

## Best Practices

1. **Understand the data**
   在套用任何演算法之前，先做 EDA 了解資料的分佈、趨勢與已知的異常。可以用 Pandas、NumPy、Matplotlib 協助探索

2. **Choose appropriate algorithms**
   演算法的選擇取決於資料的特性與使用情境，常見的有：
   - `IsolationForest`
   - `LocalOutlierFactor (LOF)`
   - `One-class SVM`
   → 可以用 `Scikit-learn` 或 `PyOD` 實作

3. **Evaluate performance**
   要避免偵測到的「異常」只是誤報，必須評估：
   - `Precision`：偵測到的異常中，真正是異常的比例
   - `Recall`：真實異常中，被成功偵測到的比例
   - `F1 score`：Precision 與 Recall 的調和平均

4. **Incorporate into your data pipeline**
   Anomaly Detection 應該**越靠近資料源越好**，可以透過 Kafka 或 AWS Kinesis 做串流，偵測到異常時觸發 alert 通知 DE 或 DS

---

## Common Algorithms

### IsolationForest

透過隨機切分資料來隔離離群值，需要切分次數越少的資料點，越可能是 anomaly

```python
import numpy as np
import matplotlib.pyplot as plt
from sklearn.ensemble import IsolationForest

np.random.seed(42)
X = 0.3 * np.random.randn(100, 2)
X_outliers = np.random.uniform(low=-4, high=4, size=(20, 2))
X = np.vstack([X, X_outliers])

clf = IsolationForest(random_state=42)
clf.fit(X)

scores = clf.decision_function(X)

plt.scatter(X[:, 0], X[:, 1], c=scores, cmap='coolwarm')
plt.colorbar()
plt.title('Anomaly Scores')
plt.show()
```

### LocalOutlierFactor (LOF)

比較某個資料點的**局部密度**與其鄰居的密度，密度明顯較低的點就是 anomaly

```python
from sklearn.neighbors import LocalOutlierFactor
from sklearn.datasets import load_iris

X, y = load_iris(return_X_y=True)

clf = LocalOutlierFactor(n_neighbors=20, contamination=0.1)
y_pred = clf.fit_predict(X)
# 輸出：1 = 正常，-1 = anomaly
```

### Evaluate with Precision / Recall / F1

```python
from sklearn.metrics import precision_score, recall_score, f1_score

precision = precision_score(y_true, y_pred, average='binary')
recall    = recall_score(y_true, y_pred, average='binary')
f1        = f1_score(y_true, y_pred, average='binary')
```

---

## Anomaly Detection in Dagster

Dagster 可以透過 [Asset Checks](https://docs.dagster.io/concepts/assets/asset-checks) 將 anomaly detection 整合進 pipeline，方法是把**目前資料與歷史 materialization 做比較**

以下範例：檢查當次 materialization 的 row count 是否在歷史平均的一個標準差以內

```python
from dagster import asset, asset_check, AssetCheckResult, MaterializeResult, EventRecordsFilter, DagsterEventType, AssetKey
import statistics

@asset
def asset1():
    num_rows = random.random() * 100
    return MaterializeResult(metadata={"num_rows": num_rows})

@asset_check(asset=asset1)
def num_rows_is_within_standard_deviations(context):
    records = context.instance.get_event_records(
        EventRecordsFilter(DagsterEventType.ASSET_MATERIALIZATION, asset_key=AssetKey("asset1")),
        limit=1000,
    )

    if len(records) >= 3:
        num_rows_values = [
            record.asset_materialization.metadata["num_rows"].value for record in records
        ]
        mean  = statistics.mean(num_rows_values[:-1])
        stdev = statistics.stdev(num_rows_values[:-1])

        return AssetCheckResult(
            passed=abs(num_rows_values[0] - mean) <= stdev,
            metadata={"Note": f"num_rows={round(num_rows_values[0],2)}, mean={round(mean,2)}, stdev={round(stdev,2)}"}
        )
    else:
        return AssetCheckResult(passed=True, metadata={"Note": "需要至少 3 次 materialization 才能啟用此 check"})
```

---

## 與 Data Quality 的關係

| 層級 | 對應 [[Data Quality]] 的概念 |
|---|---|
| Basic checks | NULL 檢查、重複值 → 不需要 anomaly detection |
| Intermediate checks | 週與週的行數比較 → 用統計方法做 |
| Advanced checks | 引入 ML，建立季節性調整的行數檢查，減少誤報 |

Anomaly detection 主要對應到 **Advanced checks** 這一層，特別適合用在 [[Data Quality#Fact table|Fact table]] 的行數檢查，因為 fact table 有季節性，不適合直接做 day-over-day 的比較

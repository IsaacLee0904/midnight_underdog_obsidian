tags: #design-pattern #etl #data-engineering #analytical-engineering #data-modeling
source: [DataExport](https://www.youtube.com/watch?v=JiedBnTFCeg&list=PLwUdL9DpGWU0lhwp3WCxRsb1385KFTLYE&index=13)

---

## Analytical Engineering Pattern

### Why we need Analytical Engineering

> [!attention] 核心價值：降低認知負荷，讓 data engineer 可以用更高維度去思考

與其每次都從 SQL 的 `GROUP BY` 層面思考，不如識別高階模式，讓 code 自然水到渠成，這在 LLM 時代尤其重要，他讓生產 SQL code 變得更容易，因此懂高階模式的人才能用 AI 工具發揮他的效率

### Common Analytical Engineering Patterns

在主資料層 (master data layer)，大部分 Data Engineer 會用到的 analytical pattern 都可以歸類成三種

| Pattern            | SQL statement                          | Note          |
| ------------------ | -------------------------------------- | ------------- |
| Aggregation-based  | `GROUP BY`                             | 維度切片、計數       |
| Accumulation-based | `FULL OUTER JOIN`                      | 今天 vs 昨天的狀態變化 |
| Window-based       | `OVER (PARTITION BY ... ORDER BY ...)` | 時間視窗內的變動率與趨勢  |
除此之外，還有一種富化模式 (enrichment pattern)，也就是做一個 `JOIN`，把其他欄位引入，但這屬於在 master data layer 上游進行的 pattern

#### Aggregation-based Pattern (聚合模式)

核心概念是將事物進行分組與計算，通常會引入一些維度
* Concept
	* 語法：`SUM`, `AVG`, `COUNT`, `MEDIAN`, `PERCENTILE`
	* 應用：root cause analysis (RCA)、維度拆解、A/B testing metics

> [!attention] Root Cause Analysis (RCA)
> Facebook 透過 RCA framework，可以插入任何 metrics，然後解釋他的變動，透過引入各種變數，可以得出造成這個趨勢的原因
> 
> EX. 有一個 week-over-week (WoW) 的變化，該指標週同比變化是增加 100 萬，這個框架會拆解這個變化，並得出原因「總共是增加 100 萬，但實際上他在美國是增加 150 萬，因為在印度是減少 50 萬 
> -> 某一個指標總體資料是正向的，或者朝某一個方向移動，並不意味著個體資料的維度切片 (dim cut) 也是朝同一個方向

* Best practice
	* 不要直接聚合 fact table，應該先建立聚合中間層 (daily grain)，再透過 `user_id` 將其他維度引入
	* `JOIN` 會變成單純的一對一，效能更好，且天然支援 A/B testing (可以很容易的把 user 分配到不同的實驗組)
* Gothas
	* 維度過多會回到 user grain，喪失聚合的意義
	* 高基數維度 EX. 國家、日期 可能會導致組合爆炸(超多類)，可以改成更粗顆粒度的資料 EX. day > week / month
	* 看百分比變化的指標一定要搭配絕對數量看，避免 1 變 0 但 100 % 下跌的誤判

#### Accumulation-based Pattern (累積模式)

與聚合模式的本質差異在於，「沒有資料」也是一種資料，也因此需要 `FULL OUTER JOIN`，這個模式建立在累積表設計 (cumulative table design) 基礎之上

<mark style="background:#fff88f">Growth Accounting (狀態轉換追蹤)</mark>

![[Screenshot 2026-06-16 at 3.37.22 PM.png]]

| status      | yesterday | today    |
| ----------- | --------- | -------- |
| New         | NULL      | Active   |
| Retained    | Active    | Active   |
| Churned     | Active    | Inactive |
| Resurrected | Inactive  | Active   |
| Stale       | Inactive  | Inactive |
| Deleted     | any       | NULL     |

* 增長公式 : **Net Growth = New (新用戶) + Resurrected (回流用戶) - Churned (流失用戶)**
* 這樣的 pattern 不只可以用在 user growth，也可以用在虛假帳號追蹤 (New/Reclassified/Declassified)、MLOps 模型健康程度追蹤 (分類器輸出狀態流動)、Airbnb 高風險房東追蹤
* DDL
```sql
CREATE TABLE users_growth_accounting (
    user_id           TEXT,
    first_active_date DATE, 
    last_active_date  DATE,
    daily_active_state  TEXT,   -- New/Retained/Churned/Resurrected/Stale
    weekly_active_state TEXT,
    dates_active      DATE[],
    date              DATE,
    PRIMARY KEY (user_id, date)
);
-- first_active_date : 該使用者第一次出現的日期，如果一開始在錯誤的日期開始累積，最終可能會得到錯誤的使用者首次活躍日期，而他們實際上很久以前就活躍了
-- last_active_date：這將是他們最後一次活躍的日期
-- daily_active_state (日活躍狀態) & weekly_active_state (週活躍狀態)：這將是我們增長會計的 value
-- dates_active (活躍日期陣列)：bit map


-- 每日累積查詢核心邏輯
--- get yesterday data from users_growth_accounting table 
WITH yesterday AS (
    SELECT * FROM users_growth_accounting
    WHERE date = '2023-02-28'
),
--- get today data and 
today AS (
    SELECT user_id,
           DATE_TRUNC('day', event_time::TIMESTAMP)::DATE as today_date
    FROM events
    WHERE DATE_TRUNC('day', event_time::TIMESTAMP)::DATE = '2023-03-01'
      AND user_id IS NOT NULL
    GROUP BY user_id, 2
)
SELECT
    COALESCE(t.user_id, y.user_id) as user_id,
    COALESCE(y.first_active_date, t.today_date) as first_active_date,  -- 有昨天取昨天，否則取今天
    COALESCE(t.today_date, y.last_active_date) as last_active_date,    -- 有今天取今天，否則保留昨天
    CASE
        WHEN y.user_id IS NULL AND t.user_id IS NOT NULL               THEN 'New'
        WHEN t.user_id IS NOT NULL
             AND y.last_active_date = t.today_date - INTERVAL '1 day'  THEN 'Retained'
        WHEN t.user_id IS NOT NULL
             AND y.last_active_date < t.today_date - INTERVAL '1 day'  THEN 'Resurrected' -- 回流
        WHEN t.user_id IS NULL
             AND y.last_active_date = y.date                           THEN 'Churned' -- 流失
        ELSE 'Stale' -- 沉睡
    END as daily_active_state
FROM today t
FULL OUTER JOIN yesterday y ON t.user_id = y.user_id;
```
要特別注意的是關於周活躍狀態 (weekly active state)，因為對於前 7 天留存和回流的定義會有所不同，在周活躍中的留存 (retained) 我們實際想看的是過去 7 天的任何一個時間點是否活躍，因此應該要使用如下
```SQL
CASE 
	WHEN t.user_id IS NULL THEN 'NEW'
	WHEN y.last_active_date < t.todat_date - INTERVAL '7 day' THEN 'Resurrected'
	WHEN y.last_active_date >= y.date - INTERVAL '7 day'  THEN 'Retained' -- 使用昨天的日期是因為今天可能不活躍
	WHEN t.today_date IS NULL AND y.last_active_date = y.date - INTERVAL '7 day'  THEN 'Churned'
	ELSE 'Stale'
END AS weekly_active_state
```
這邊的數學計算與邏輯蠻複雜的，因此 Facebook 實際上是使用 bit map 的方式進行日期的位移


<mark style="background:#fff88f">Survivor Analysis (存活分析)</mark>

> [!attention] **倖存者偏差 (Survivorship Bias)**
> 以二戰轟炸機的例子來看，我們看到的都是「活著回來」的飛機，反而彈孔最少的地方才是需要加強的地方 (因為這些地方被擊中就沒回來了)，所以在分析的時候應該要先反問「我的樣本是倖存者還是全體？」

![[Screenshot 2026-06-16 at 3.51.12 PM.png]]
留存率、J 曲線 (J-Curves) 或各領域的存活指標，基本上就是存活，由<font color="#ff0000">曲線 (curve)</font>、<font color="#ff0000">狀態檢查 (state check)</font> 與<font color="#ff0000">基準日期 (reference date)</font> 組成，可以看到事務基本上會有三種走向 (如上圖)，所有 user 都會從 100% (註冊那天開始)，隨著時間推移，狀態會發生變化，有些 user 留下來，有些則離開，這就是所謂的<font color="#ff0000">同群 (cohort) 分析</font>或稱為<font color="#ff0000">基準日期 (reference date) 分析</font> 


```SQL
-- 單一同期群的 J 曲線
SELECT
    date,
    COUNT(CASE WHEN daily_active_state IN ('New','Retained','Resurrected')
               THEN 1 END)::REAL / COUNT(1) as percent_active
FROM users_growth_accounting
WHERE first_active_date = '2023-03-01'
GROUP BY date
ORDER BY date;

-- 全同期群留存矩陣（Survivor Analysis Matrix）
SELECT
    first_active_date,
    date - first_active_date as days_since_first_active,
    COUNT(CASE WHEN daily_active_state IN ('New','Retained','Resurrected')
               THEN 1 END)::REAL / COUNT(1) as percent_active
FROM users_growth_accounting
GROUP BY 1, 2
ORDER BY 2;
```

>[!hint] Data Engineer Interview Important Thing
>

#### Window-based Pattern (時間視窗模式)

我們常常使用 window-based pattern 來處理同環比類型的東西 EX. DoDs, MoMs, YoYs ...，基本上就是一段時間內的變動率 (rate of change)，然而當然反過來做滾動加總 (rolling sum) 也是可以的

|                       | 數學概念 | SQL 效果 | 圖表特性     |
| --------------------- | ---- | ------ | -------- |
| DoD / WoW / MoM / YoY | 微分   | 變動率    | 更陡峭、更多雜訊 |
| Rolling Sum / Average | 積分   | 累積趨勢   | 更平滑、降低雜訊 |

Rolling 滾動式窗模板
```SQL
SUM(metric) OVER (
    PARTITION BY dimension 
    ORDER BY date 
    ROWS BETWEEN N PRECEDING AND CURRENT ROW
)
```
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

#### Aggregation-based (聚合模式)

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
	* `JOIN` 會變成單純的一對一，效能更好，且天然支援 A/B testing


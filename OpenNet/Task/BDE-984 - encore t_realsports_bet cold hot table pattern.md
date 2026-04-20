#### Basic Information
* related DAG
	* <font color="#548dd4">afbet_realsports.t_realsports_bet_reshard</font> : sync data from MySQL to Redshift **every 10 mins**
	* <font color="#548dd4">afbet_realsports.t_realsports_bet_cold_copy</font> : copy data from hot table to cold table **daily**
	* <font color="#548dd4">afbet_realsports.t_realsports_bet_hot_delete</font> : delete old data (older than 70 days) from hot table **daily**
- hot table : 70 days
- related country : ng, gh


Step1. Create Empty hot table
```sql
-- gh 
CREATE TABLE afbet_realsports_gh.t_realsports_bet_hot
(LIKE afbet_realsports_gh.t_realsports_bet);

-- ng
CREATE TABLE afbet_realsports_ng.t_realsports_bet_hot
(LIKE afbet_realsports_ng.t_realsports_bet);
```

Step2. Adjust the DAG
* <font color="#548dd4">afbet_realsports.t_realsports_bet_cold_copy</font>
	* 把原本只有針對 sporty cols 處理的邏輯加入 brand 判斷並且加入 encore cols
* <font color="#548dd4">afbet_realsports.t_realsports_bet_hot_delete</font>
	* 刪掉原本 brand = encore do nothing 的 brand guard 移除

Step3. Add Airflow Variables (country list)
* country_realsports_hot_copy
* 




Step3. Insert data to hot table 
打開 `afbet_realsports.t_realsports_bet_hot_copy.py`

> **Why use `create_time` not `index` ?**
> 
> 根據 `svv_table_info` 資訊 `index` 是 sort_key，然而根據 `EXPLAIN` 的結果發現實際上使用 `create_time` 與 `id` 都會是 full table scan
>
>* filter with create_time![[Screenshot 2026-03-27 at 5.48.29 PM.png]]
>
> * filter with index ![[Screenshot 2026-03-27 at 5.46.42 PM.png]]

```sql
EXPLAIN
SELECT * FROM afbet_realsports_gh.t_realsports_selection
WHERE create_time >= DATEADD(day, -10, GETDATE())
AND create_time < GETDATE();

EXPLAIN
SELECT *
FROM afbet_realsports_gh.t_realsports_selection
WHERE id >= CONCAT(TO_CHAR(DATEADD(day, -10, GETDATE()), 'YYMMDDHHMMSS'), '0000000000000');
```

>[!hint]
> 需要透過 `ANALYIZE` 來優化


Step3. Use <mark style="background: #FFB86CA6;">dba-redshift-executor</mark> rename tables  
```sql
-- gh
ALTER TABLE afbet_realsports_gh.t_realsports_selection RENAME TO t_realsports_selection_cold;
ALTER TABLE afbet_realsports_gh.t_realsports_selection_hot RENAME TO t_realsports_selection;

-- ng
ALTER TABLE afbet_realsports_ng.t_realsports_selection RENAME TO t_realsports_selection_cold;
ALTER TABLE afbet_realsports_ng.t_realsports_selection_hot RENAME TO t_realsports_selection;
```

分兩段  
1. 發 PR -> 等 approve -> merge
2. 按 action -> 等 approve -> SQL execute -> 檢查結果


Step4. Open DAG
* afbet_realsports.t_realsports_selection_reshard_v2
* afbet_realsports.t_realsports_selection_cold_copy
* afbet_realsports.t_realsports_selection_hot_delete

>[!WARNING] 需要額外創建 Airflow Variables

**Switch Time**
* gh - 2026-03-27 14:36
* ng - 2026-03-27 14:36
#### Basic Information
* related DAG
	* <font color="#548dd4">afbet_main.t_order_record</font> : sync data to Warehouse **every hour**
	* <font color="#548dd4">afbet_order.t_order_record_shard</font> : sync shard DB data to Warehouse **every 10 mins**
	* <font color="#548dd4">afbet_main.t_order_record_cold_copy</font> : copy data from hot table to cold table **daily**
	* <font color="#548dd4">afbet_main.t_order_record_hot_delete</font> : delete old data (older than 200 days) from hot table **daily**
- hot table : 200 days
- related country : ng, gh

Step0. Record the original row count
* ng : 2664889523
* gh : 774532368
```SQL
-- ng 
SELECT
	DATE(create_time) AS date,
	COUNT(*) AS row_count
FROM afbet_main_ng.t_order_record
WHERE create_time >= '2025-10-03T00:00:00' AND create_time < '2026-04-22T00:00:00'
GROUP BY DATE(create_time)
ORDER BY date;

-- gh
SELECT
	DATE(create_time) AS date,
	COUNT(*) AS row_count
FROM afbet_main_gh.t_order_record
WHERE create_time >= '2025-10-03T00:00:00' AND create_time < '2026-04-22T00:00:00'
GROUP BY DATE(create_time)
ORDER BY date;
```

Step1. Create Empty hot table
```sql
-- gh 
CREATE TABLE afbet_main_gh.t_order_record_hot
(LIKE afbet_main_gh.t_order_record);

-- ng
CREATE TABLE afbet_main_ng.t_order_record_hot
(LIKE afbet_main_ng.t_order_record);

-- Check the hot table schema and diskey
SELECT "schema", "table", diststyle, sortkey1, sortkey1_enc
FROM svv_table_info
WHERE "table" IN ('t_order_record_hot', 't_order_record')
ORDER BY "schema", "table";
```


Step2. Adjust the DAG
* <font color="#548dd4">afbet_realsports.t_realsports_bet_cold_copy</font>
	* 把原本只有針對 sporty cols 處理的邏輯加入 brand 判斷並且加入 encore cols
* <font color="#548dd4">afbet_realsports.t_realsports_bet_hot_delete</font>
	* 刪掉原本 brand = encore do nothing 的 brand guard 移除

Step3. Add Airflow Variables (country list)
* country_realsports_hot_copy -> replace with [ng, gh]
* country_realsports_cold_copy -> replace with [ng, gh]
* country_realsports_hot_delete -> replace with [ng, gh]

Step4. Insert data to hot table 

打開 `afbet_realsports.t_realsports_bet_hot_copy.py`
並且使用語法
```SQL
-- gh
-- Day 61-70 (2/9 - 2/19)
INSERT INTO afbet_realsports_gh.t_realsports_bet_hot                           
SELECT * FROM afbet_realsports_gh.t_realsports_bet                               
WHERE create_time >= '2026-02-09T00:00:00'                                       AND create_time < '2026-02-19T00:00:00';

-- Day 51-60 (2/19 - 3/1)                                                        
INSERT INTO afbet_realsports_gh.t_realsports_bet_hot
SELECT * FROM afbet_realsports_gh.t_realsports_bet                               
WHERE create_time >= '2026-02-19T00:00:00'
AND create_time < '2026-03-01T00:00:00';                                                                                                                          
-- Day 41-50 (3/1 - 3/11)                                                        
INSERT INTO afbet_realsports_gh.t_realsports_bet_hot
SELECT * FROM afbet_realsports_gh.t_realsports_bet                               
WHERE create_time >= '2026-03-01T00:00:00'
AND create_time < '2026-03-11T00:00:00';                                                                                                                          
-- Day 31-40 (3/11 - 3/21)                                                       
INSERT INTO afbet_realsports_gh.t_realsports_bet_hot
SELECT * FROM afbet_realsports_gh.t_realsports_bet                               
WHERE create_time >= '2026-03-11T00:00:00'
AND create_time < '2026-03-21T00:00:00';                                                                                                                          
-- Day 21-30 (3/21 - 3/31)
INSERT INTO afbet_realsports_gh.t_realsports_bet_hot                             SELECT * FROM afbet_realsports_gh.t_realsports_bet                               
WHERE create_time >= '2026-03-21T00:00:00'
AND create_time < '2026-03-31T00:00:00';                                                                                                                          

-- Day 11-20 (3/31 - 4/10)
INSERT INTO afbet_realsports_gh.t_realsports_bet_hot                             
SELECT * FROM afbet_realsports_gh.t_realsports_bet
WHERE create_time >= '2026-03-31T00:00:00'                                       
AND create_time < '2026-04-10T00:00:00';
                                                                                 
-- Day 1-10 (4/10 - 4/20)                                                        
INSERT INTO afbet_realsports_gh.t_realsports_bet_hot
SELECT * FROM afbet_realsports_gh.t_realsports_bet                               
WHERE create_time >= '2026-04-10T00:00:00'
AND create_time < '2026-04-20T00:00:00';    

-- ng
-- Day 61-70 (2/9 - 2/19)
INSERT INTO afbet_realsports_ng.t_realsports_bet_hot                           
SELECT * FROM afbet_realsports_ng.t_realsports_bet                               
WHERE create_time >= '2026-02-09T00:00:00'                                       AND create_time < '2026-02-19T00:00:00';

-- Day 51-60 (2/19 - 3/1)                                                        
INSERT INTO afbet_realsports_ng.t_realsports_bet_hot
SELECT * FROM afbet_realsports_ng.t_realsports_bet                               
WHERE create_time >= '2026-02-19T00:00:00'
AND create_time < '2026-03-01T00:00:00';                                                                                                                          
-- Day 41-50 (3/1 - 3/11)                                                        
INSERT INTO afbet_realsports_ng.t_realsports_bet_hot
SELECT * FROM afbet_realsports_ng.t_realsports_bet                               
WHERE create_time >= '2026-03-01T00:00:00'
AND create_time < '2026-03-11T00:00:00';                                                                                                                          
-- Day 31-40 (3/11 - 3/21)                                                       
INSERT INTO afbet_realsports_ng.t_realsports_bet_hot
SELECT * FROM afbet_realsports_ng.t_realsports_bet                               
WHERE create_time >= '2026-03-11T00:00:00'
AND create_time < '2026-03-21T00:00:00';                                                                                                                          
-- Day 21-30 (3/21 - 3/31)
INSERT INTO afbet_realsports_ng.t_realsports_bet_hot                             SELECT * FROM afbet_realsports_ng.t_realsports_bet                               
WHERE create_time >= '2026-03-21T00:00:00'
AND create_time < '2026-03-31T00:00:00';                                                                                                                          

-- Day 11-20 (3/31 - 4/10)
INSERT INTO afbet_realsports_gh.t_realsports_bet_hot                             
SELECT * FROM afbet_realsports_gh.t_realsports_bet
WHERE create_time >= '2026-03-31T00:00:00'                                       
AND create_time < '2026-04-10T00:00:00';
                                                                                 
-- Day 1-10 (4/10 - 4/20)                                                        
INSERT INTO afbet_realsports_ng.t_realsports_bet_hot
SELECT * FROM afbet_realsports_ng.t_realsports_bet                               
WHERE create_time >= '2026-04-10T00:00:00'
AND create_time < '2026-04-20T00:00:00';    
```

> **Why use `create_time` not `index` ?**
> 
> 根據 `svv_table_info` 資訊 `index` 是 sort_key，然而根據 `EXPLAIN` 的結果發現實際上使用 `create_time` 與 `id` 都會是 full table scan
>
>* filter with create_time![[Screenshot 2026-03-27 at 5.48.29 PM.png]]
>
> * filter with index ![[Screenshot 2026-03-27 at 5.46.42 PM.png]]

>[!hint]
> 需要透過 `ANALYIZE` 來優化


Step3. Use <mark style="background: #FFB86CA6;">dba-redshift-executor</mark> rename tables  
```sql
-- gh
ALTER TABLE afbet_realsports_gh.t_realsports_bet RENAME TO t_realsports_bet_cold;
ALTER TABLE afbet_realsports_gh.t_realsports_bet_hot RENAME TO t_realsports_bet;

-- ng
ALTER TABLE afbet_realsports_ng.t_realsports_bet RENAME TO t_realsports_bet_cold;
ALTER TABLE afbet_realsports_ng.t_realsports_bet_hot RENAME TO t_realsports_bet;
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
* gh - 2026-04-20 18:03
* ng - 2026-04-20 18:03
#### Basic Information
* related DAG
	* <font color="#548dd4">src_realsports_bet_breakdown_v12_p123</font> & <font color="#548dd4">src_realsports_bet_breakdown_v12_p4</font> : DAG for sync data to encore `bi_report`in DA Airflow
	* <font color="#548dd4">bi_realsports.src_realsports_all_orders_v12_cold_copy</font> : copy data from hot table to cold table **daily** (move data 85 days ago to cold table)
	* <font color="#548dd4">bi_realsports.src_realsports_all_orders_v12_hot_delete</font> : delete old data (older than 95 days) from hot table **daily**
- hot table : 90 days
- related country : ng, gh

Step0. Record the original row count
* row_count : 6445574610
```SQL
SELECT
  stat_date AS date,                                                             
  COUNT(*) AS row_count
FROM bi_realsports.src_realsports_all_orders_v12                        
WHERE stat_date >= '2026-01-27' AND stat_date < '2026-04-27'                     
GROUP BY stat_date
ORDER BY date;   
```

Step1. Create Empty hot table
```sql
-- SQL statement createe hot table
CREATE TABLE bi_realsports.src_realsports_all_orders_v12_hot
(LIKE bi_realsports.src_realsports_all_orders_v12);

-- Check the hot table schema and diskey
SELECT "schema", "table", diststyle, sortkey1, sortkey1_enc
FROM svv_table_info
WHERE "table" IN ('src_realsports_all_orders_v12_hot', 'src_realsports_all_orders_v12')
ORDER BY "schema", "table";
```


Step2. Adjust the DAG
* <font color="#548dd4">afbet_main.t_order_record_hot_delete</font>
	* 刪掉原本 brand = encore do nothing 的 brand guard 移除
* <font color="#548dd4">afbet_main.t_order_record_hot_copy</font>
	* 改成每小時執行，並且監控 afbet_main.t_order_record 是否執行完成

Step3. Add Airflow Variables (country list)
* country_realsports_hot_copy -> replace with [ng, gh]
* country_realsports_cold_copy -> replace with [ng, gh]
* country_realsports_hot_delete -> replace with [ng, gh]

Step4. Insert data to hot table 
並且使用語法
```SQL
-- gh
-- Day 61-70 (2/11 - 2/21)               
INSERT INTO afbet_realsports_gh.t_realsports_selection_source_hot     
SELECT * FROM afbet_realsports_gh.t_realsports_selection_source    
WHERE create_time >= '2026-02-11T00:00:00'          
AND create_time < '2026-02-21T00:00:00';                                                            
-- Day 51-60 (2/21 - 3/3)             
INSERT INTO afbet_realsports_gh.t_realsports_selection_source_hot
SELECT * FROM afbet_realsports_gh.t_realsports_selection_source   
WHERE create_time >= '2026-02-21T00:00:00'      
AND create_time < '2026-03-03T00:00:00';                                          
-- Day 41-50 (3/3 - 3/13)            
INSERT INTO afbet_realsports_gh.t_realsports_selection_source_hot  
SELECT * FROM afbet_realsports_gh.t_realsports_selection_source     
WHERE create_time >= '2026-03-03T00:00:00'          
AND create_time < '2026-03-13T00:00:00';                                          
-- Day 31-40 (3/13 - 3/23)                 
INSERT INTO afbet_realsports_gh.t_realsports_selection_source_hot  
SELECT * FROM afbet_realsports_gh.t_realsports_selection_source    
WHERE create_time >= '2026-03-13T00:00:00'
AND create_time < '2026-03-23T00:00:00';                                         					
-- Day 21-30 (3/23 - 4/2)                  
INSERT INTO afbet_realsports_gh.t_realsports_selection_source_hot    
SELECT * FROM afbet_realsports_gh.t_realsports_selection_source   
WHERE create_time >= '2026-03-23T00:00:00'
AND create_time < '2026-04-02T00:00:00';          																	  
-- Day 11-20 (4/2 - 4/12)
INSERT INTO afbet_realsports_gh.t_realsports_selection_source_hot     
SELECT * FROM afbet_realsports_gh.t_realsports_selection_source   
WHERE create_time >= '2026-04-02T00:00:00'
AND create_time < '2026-04-12T00:00:00';                                         	  
-- Day 1-10 (4/12 - 4/22)
INSERT INTO afbet_realsports_gh.t_realsports_selection_source_hot     
SELECT * FROM afbet_realsports_gh.t_realsports_selection_source     
WHERE create_time >= '2026-04-12T00:00:00'     
AND create_time < '2026-04-22T00:00:00';
										  
-- ng                                     
-- Day 61-70 (2/11 - 2/21)                   
INSERT INTO afbet_realsports_ng.t_realsports_selection_source_hot   
SELECT * FROM afbet_realsports_ng.t_realsports_selection_source
WHERE create_time >= '2026-02-11T00:00:00'                      
AND create_time < '2026-02-21T00:00:00';
										  
-- Day 51-60 (2/21 - 3/3)                    
INSERT INTO afbet_realsports_ng.t_realsports_selection_source_hot
SELECT * FROM afbet_realsports_ng.t_realsports_selection_source 
WHERE create_time >= '2026-02-21T00:00:00'           
AND create_time < '2026-03-03T00:00:00';																		  
-- Day 41-50 (3/3 - 3/13)                     
INSERT INTO afbet_realsports_ng.t_realsports_selection_source_hot
SELECT * FROM afbet_realsports_ng.t_realsports_selection_source     
WHERE create_time >= '2026-03-03T00:00:00'             
AND create_time < '2026-03-13T00:00:00';                 

-- Day 31-40 (3/13 - 3/23)                           
INSERT INTO afbet_realsports_ng.t_realsports_selection_source_hot    
SELECT * FROM afbet_realsports_ng.t_realsports_selection_source    
WHERE create_time >= '2026-03-13T00:00:00'
AND create_time < '2026-03-23T00:00:00';          																		
-- Day 21-30 (3/23 - 4/2)                   
INSERT INTO afbet_realsports_ng.t_realsports_selection_source_hot    
SELECT * FROM afbet_realsports_ng.t_realsports_selection_source        
WHERE create_time >= '2026-03-23T00:00:00'
AND create_time < '2026-04-02T00:00:00';                                                                                        			
-- Day 11-20 (4/2 - 4/12)
INSERT INTO afbet_realsports_ng.t_realsports_selection_source_hot     
SELECT * FROM afbet_realsports_ng.t_realsports_selection_source       
WHERE create_time >= '2026-04-02T00:00:00'
AND create_time < '2026-04-12T00:00:00';              																	  
-- Day 1-10 (4/12 - 4/22)
INSERT INTO afbet_realsports_ng.t_realsports_selection_source_hot        
SELECT * FROM afbet_realsports_ng.t_realsports_selection_source    
WHERE create_time >= '2026-04-12T00:00:00'          
AND create_time < '2026-04-22T00:00:00';
```

打開 `afbet_main_gh.t_realsports_selection_source_hot_copy.py`，確定跟上原本 pipeline 之後，檢查每個小時的紀錄
```SQL
-- gh
SELECT
	DATE_TRUNC('hour', create_time) AS hour,
	COUNT(*) AS row_count
FROM afbet_realsports_gh.t_realsports_selection_source
WHERE create_time >= '2026-04-22T00:00:00' AND create_time < '2026-04-23T00:00:00'
GROUP BY DATE_TRUNC('hour', create_time)
ORDER BY hour;

SELECT
	DATE_TRUNC('hour', create_time) AS hour,
	COUNT(*) AS row_count
FROM afbet_realsports_gh.t_realsports_selection_source_hot
WHERE create_time >= '2026-04-22T00:00:00' AND create_time < '2026-04-23T00:00:00'
GROUP BY DATE_TRUNC('hour', create_time)
ORDER BY hour;

-- ng
SELECT
	DATE_TRUNC('hour', create_time) AS hour,
	COUNT(*) AS row_count
FROM afbet_realsports_ng.t_realsports_selection_source
WHERE create_time >= '2026-04-22T00:00:00' AND create_time < '2026-04-23T00:00:00'
GROUP BY DATE_TRUNC('hour', create_time)
ORDER BY hour;

SELECT
	DATE_TRUNC('hour', create_time) AS hour,
	COUNT(*) AS row_count
FROM afbet_realsports_ng.t_realsports_selection_source_hot
WHERE create_time >= '2026-04-22T00:00:00' AND create_time < '2026-04-23T00:00:00'
GROUP BY DATE_TRUNC('hour', create_time)
ORDER BY hour;
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
ALTER TABLE afbet_realsports_gh.t_realsports_selection_source RENAME TO t_realsports_selection_source_cold;
ALTER TABLE afbet_realsports_gh.t_realsports_selection_source_hot RENAME TO t_realsports_selection_source;

-- ng
ALTER TABLE afbet_realsports_ng.t_realsports_selection_source RENAME TO t_realsports_selection_source_cold;
ALTER TABLE afbet_realsports_ng.t_realsports_selection_source_hot RENAME TO t_realsports_selection_source;
```

分兩段  
1. 發 PR -> 等 approve -> merge
2. 按 action -> 等 approve -> SQL execute -> 檢查結果


Step4. Open DAG
* afbet_main.t_order_record_shard
* afbet_main.t_order_record_cold_copy
* afbet_main.t_order_record_hot_delete

>[!WARNING] 需要額外創建 Airflow Variables

**Switch Time**
* gh - 2026-04-22 07:04 UTC
* ng - 2026-04-22 07:04 UTC
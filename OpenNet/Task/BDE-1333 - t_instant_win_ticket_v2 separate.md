#### Basic Information
* related DAG
	* <font color="#548dd4">afbet_instant_win.t_instant_win_ticket_v2</font> & <font color="#548dd4">afbet_instant_win.t_instant_win_ticket_v2_new</font> : DAG for sync data to warehouse
	* <font color="#548dd4">afbet_instant_win.t_instat_win_ticket_v2_app</font> : copy data from tz to zm
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

Step3. Insert data to hot table 
並且使用語法
```SQL
-- Day 91-95 (1/22 - 2/1)
INSERT INTO bi_report.bi_realsports.src_realsports_all_orders_v12_hot            
SELECT * FROM bi_report.bi_realsports.src_realsports_all_orders_v12
WHERE stat_date >= '2026-01-22' AND stat_date < '2026-02-01';                                                                                                     
-- Day 81-90 (2/1 - 2/11)                                                           
INSERT INTO bi_report.bi_realsports.src_realsports_all_orders_v12_hot            
SELECT * FROM bi_report.bi_realsports.src_realsports_all_orders_v12              
WHERE stat_date >= '2026-02-01' AND stat_date < '2026-02-11';                                                                                                     
-- Day 71-80 (2/11 - 2/21)                                                       
INSERT INTO bi_report.bi_realsports.src_realsports_all_orders_v12_hot   
SELECT * FROM bi_report.bi_realsports.src_realsports_all_orders_v12        
WHERE stat_date >= '2026-02-11' AND stat_date < '2026-02-21';
																			
-- Day 61-70 (2/21 - 3/3)     
INSERT INTO bi_report.bi_realsports.src_realsports_all_orders_v12_hot  
SELECT * FROM bi_report.bi_realsports.src_realsports_all_orders_v12   
WHERE stat_date >= '2026-02-21' AND stat_date < '2026-03-03';
																			
-- Day 51-60 (3/3 - 3/13)      
INSERT INTO bi_report.bi_realsports.src_realsports_all_orders_v12_hot 
SELECT * FROM bi_report.bi_realsports.src_realsports_all_orders_v12  
WHERE stat_date >= '2026-03-03' AND stat_date < '2026-03-13';	
			
-- Day 41-50 (3/13 - 3/23)                                    
INSERT INTO bi_report.bi_realsports.src_realsports_all_orders_v12_hot
SELECT * FROM bi_report.bi_realsports.src_realsports_all_orders_v12     
WHERE stat_date >= '2026-03-13' AND stat_date < '2026-03-23';                                                                                        
-- Day 31-40 (3/23 - 4/2)                                     
INSERT INTO bi_report.bi_realsports.src_realsports_all_orders_v12_hot       
SELECT * FROM bi_report.bi_realsports.src_realsports_all_orders_v12     
WHERE stat_date >= '2026-03-23' AND stat_date < '2026-04-02';                                                                                 
-- Day 21-30 (4/2 - 4/12)                                     
INSERT INTO bi_report.bi_realsports.src_realsports_all_orders_v12_hot 
SELECT * FROM bi_report.bi_realsports.src_realsports_all_orders_v12      
WHERE stat_date >= '2026-04-02' AND stat_date < '2026-04-12';

-- Day 11-20 (4/12 - 4/22)                             
INSERT INTO bi_report.bi_realsports.src_realsports_all_orders_v12_hot     
SELECT * FROM bi_report.bi_realsports.src_realsports_all_orders_v12    
WHERE stat_date >= '2026-04-12' AND stat_date < '2026-04-22';
			
-- Day 1-10 (4/22 - 4/27)                              
INSERT INTO bi_report.bi_realsports.src_realsports_all_orders_v12_hot       
SELECT * FROM bi_report.bi_realsports.src_realsports_all_orders_v12     
WHERE stat_date >= '2026-04-22' AND stat_date < '2026-04-27';
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
ALTER TABLE bi_report.bi_realsports.src_realsports_all_orders_v12 RENAME TO src_realsports_all_orders_v12_cold;
ALTER TABLE bi_report.bi_realsports.src_realsports_all_orders_v12_hot RENAME TO src_realsports_all_orders_v12;    
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
* 2026-04-28 10:36 UTC
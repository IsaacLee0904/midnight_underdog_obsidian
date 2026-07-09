#### Basic Information
* related DAG
	* <font color="#548dd4">afbet_instant_win.t_instant_win_ticket_v2</font> & <font color="#548dd4">afbet_instant_win.t_instant_win_ticket_v2_new</font> : DAG for sync data to warehouse
	* <font color="#548dd4">afbet_instant_win.t_instat_win_ticket_v2_app</font> : copy data from tz to zm

Step0. Record the original row count and min(create_time)
* row_count : 6445574610
```SQL
-- min(create_time) = 2025-06-20 00:00:10.000
select min(create_time)
from afbet_instant_win_tz.t_instant_win_ticket_v2
where country_code = 'zm'

-- 12211962
select count(*)
from afbet_instant_win_tz.t_instant_win_ticket_v2
where country_code = 'zm'
```

Step1. Dual Write + Backfill
PR : https://github.com/opennetltd/warehouse_engineer/pull/2672/changes#diff-d8e39bf16d3763f24dedea0a317d67879473a6b86f80a8b397f449330fd768e7
```markdown
### [BDE-1333](https://opennetltd.atlassian.net/browse/BDE-1333?atlOrigin=eyJpIjoiNWRkNTljNzYxNjVmNDY3MDlhMDU5Y2ZhYzA5YTRkZjUiLCJwIjoiZ2l0aHViLWNvbS1KU1cifQ) Separate TZ and ZM Data for Instant Win Ticket Tables in Redshift

#### Background

Currently, both TZ and ZM data are written into the TZ Redshift table, leaving the ZM table empty. However the ticket main goal is separated those data in Redshift make sure no one will confuse.

#### Workflow

Step1. **Dual Write + Backfill** ← _this PR_

- Update `_new` DAGs : ZM data writes to both `afbet_instant_win_tz.t_instant_ticket_v2` (existing, for backward compatibility) and `afbet_instant_win_zm.t_instant_ticket_v2` (new target)
- Create backfill `_app` DAG: copy historical ZM records from `afbet_instant_win_tz.t_instant_ticket_v2` into `afbet_instant_win_zm.t_instant_ticket_v2` to ensure ZM table is complete before DA migration
- Validate ZM table data is correct and up-to-date

Step2. **Migrate DA DAGs** :

- Update affected DA DAGs to query from the correct table per country

Step3. **Clean Up** :

- Disable `_new` DAGs and update original DAGs to include both `tz` and `zm` as target countries, each writing only to their respective table
```


Step2. Adjust DA DAGs
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
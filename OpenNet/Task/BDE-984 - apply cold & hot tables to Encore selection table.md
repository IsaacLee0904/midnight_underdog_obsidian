## Information

* **Jira Ticket** : [Link](https://opennetltd.atlassian.net/jira/software/projects/BDE/boards/117?jql=assignee%20%3D%20712020%3A4caf52e7-1e32-4e9b-ade5-8c262db7c7f0&selectedIssue=BDE-984)
* **Reference** : [Link](https://opennetltd.atlassian.net/wiki/spaces/DET/pages/3334045924/Redshift+-+Cold+Hot+Table+Structure)
* **Branch** : <span style="color:rgb(8, 186, 118)">feature/BDE-984encore_cold_hot_table</span>

## Requirement

Apply below table in Encore to cold / hot tables : 

| **DB**         | **Table**                                 | **Brand** | **Data**      |
| -------------- | ----------------------------------------- | --------- | ------------- |
| `bi_warehouse` | `t_realsports_selection`                  | Encore    | last 40 days  |
| `bi_warehouse` | `t_realsports_bet`                        | Encore    | last 70 days  |
| `bi_warehouse` | `t_order_record`                          | Encore    | last 200 days |
| `bi_warehouse` | `t_facts_sporty_uof_messages_odds_change` | Encore    | last 20 days  |
| `bi_warehouse` | `logs_patron_fe_behav`                    | Encore    | last 40 days  |
| `bi_warehouse` | `t_realsports_selection_source`           | Encore    | last 70 days  |
| `bi_report`    | `src_realsports_all_orders_v12`           | Encore    | last 90 days  |

## Implement

### t_realsports_selection 

Step1. Create Empty hot table
```sql
-- gh 
CREATE TABLE afbet_realsports_gh.t_realsports_selection_hot
(LIKE afbet_realsports_gh.t_realsports_selection);

-- ng
CREATE TABLE afbet_realsports_ng.t_realsports_selection_hot
(LIKE afbet_realsports_ng.t_realsports_selection);
```

Step2. Insert data to hot table 
```sql
-- gh
-- Day 1-10
INSERT INTO afbet_realsports_gh.t_realsports_selection_hot
SELECT * FROM afbet_realsports_gh.t_realsports_selection
WHERE create_time >= DATEADD(day, -40, GETDATE())
  AND create_time < DATEADD(day, -30, GETDATE());

-- Day 11-20
INSERT INTO afbet_realsports_gh.t_realsports_selection_hot
SELECT * FROM afbet_realsports_gh.t_realsports_selection
WHERE create_time >= DATEADD(day, -30, GETDATE())
  AND create_time < DATEADD(day, -20, GETDATE());

-- Day 21-30
INSERT INTO afbet_realsports_gh.t_realsports_selection_hot
SELECT * FROM afbet_realsports_gh.t_realsports_selection
WHERE create_time >= DATEADD(day, -20, GETDATE())
  AND create_time < DATEADD(day, -10, GETDATE());

-- Day 31-40
INSERT INTO afbet_realsports_gh.t_realsports_selection_hot
SELECT * FROM afbet_realsports_gh.t_realsports_selection
WHERE create_time >= DATEADD(day, -10, GETDATE())
  AND create_time < GETDATE();

-- ng
-- Day 1-10
INSERT INTO afbet_realsports_ng.t_realsports_selection_hot
SELECT * FROM afbet_realsports_ng.t_realsports_selection
WHERE create_time >= DATEADD(day, -40, GETDATE())
  AND create_time < DATEADD(day, -30, GETDATE());

-- Day 11-20
INSERT INTO afbet_realsports_ng.t_realsports_selection_hot
SELECT * FROM afbet_realsports_ng.t_realsports_selection
WHERE create_time >= DATEADD(day, -30, GETDATE())
  AND create_time < DATEADD(day, -20, GETDATE());

-- Day 21-30
INSERT INTO afbet_realsports_ng.t_realsports_selection_hot
SELECT * FROM afbet_realsports_ng.t_realsports_selection
WHERE create_time >= DATEADD(day, -20, GETDATE())
  AND create_time < DATEADD(day, -10, GETDATE());

-- Day 31-40
INSERT INTO afbet_realsports_ng.t_realsports_selection_hot
SELECT * FROM afbet_realsports_ng.t_realsports_selection
WHERE create_time >= DATEADD(day, -10, GETDATE())
  AND create_time < GETDATE();

```

> Why use create_time not


Step3. Use <mark style="background: #FFB86CA6;">dba-redshift-executor</mark> rename tables  
```sql
-- gh
ALTER TABLE afbet_realsports_gh.t_realsports_selection RENAME TO t_realsports_selection_cold;
ALTER TABLE afbet_realsports_gh.t_realsports_selection_hot RENAME TO t_realsports_selection;

-- ng
ALTER TABLE afbet_realsports_ng.t_realsports_selection RENAME TO t_realsports_selection_cold;
ALTER TABLE afbet_realsports_ng.t_realsports_selection_hot RENAME TO t_realsports_selection;
```

**Open DAG**
* afbet_realsports.t_realsports_selection_reshard_v2
* afbet_realsports.t_realsports_selection_cold_copy
* afbet_realsports.t_realsports_selection_hot_delete

**Switch Time**
* gh - 
* ng - 

### t_realsports_bet 
gh
```sql
-- Step1. 
CREATE TABLE afbet_realsports_gh.t_realsports_bet_hot AS
SELECT *
FROM afbet_realsports_gh.t_realsports_bet
WHERE created_at >= DATEADD(day, -70, GETDATE());

-- Step2.
ALTER TABLE afbet_realsports_gh.t_realsports_bet RENAME TO t_realsports_bet_cold;
ALTER TABLE afbet_realsports_gh.t_realsports_bet_hot RENAME TO t_realsports_bet;
```

ng
```sql
-- Step1. 
CREATE TABLE afbet_realsports_ng.t_realsports_bet_hot AS
SELECT *
FROM afbet_realsports_ng.t_realsports_bet
WHERE created_at >= DATEADD(day, -70, GETDATE());

-- Step2.
ALTER TABLE afbet_realsports_ng.t_realsports_bet RENAME TO t_realsports_bet_cold;
ALTER TABLE afbet_realsports_ng.t_realsports_bet_hot RENAME TO t_realsports_bet;
```

**Open DAG**
* afbet_realsports.t_realsports_bet_cold_copy
* afbet_realsports.t_realsports_bet_hot_delete

**Switch Time**
* gh
* ng

### t_order_record 
gh
```sql
-- Step1. 
CREATE TABLE afbet_main_gh.t_order_record_hot AS
SELECT *
FROM afbet_main_gh.t_order_record
WHERE created_at >= DATEADD(day, -200, GETDATE());

-- Step2.
ALTER TABLE afbet_main_gh.t_order_record RENAME TO t_order_record_cold;
ALTER TABLE afbet_main_gh.t_order_record_hot RENAME TO t_order_record;
```

ng
```sql
-- Step1. 
CREATE TABLE afbet_main_ng.t_order_record_hot AS
SELECT *
FROM afbet_main_ng.t_order_record
WHERE created_at >= DATEADD(day, -200, GETDATE());

-- Step2.
ALTER TABLE afbet_main_ng.t_order_record RENAME TO t_order_record_cold;
ALTER TABLE afbet_main_ng.t_order_record_hot RENAME TO t_order_record;
```

**Open DAG**
* afbet_main.t_order_record_cold_copy
* afbet_main.t_order_record_hot_delete

**Switch Time**
* gh
* ng
### t_realsports_selection_source 

gh
```sql
-- Step1. 
CREATE TABLE afbet_realsports_gh.t_realsports_selection_source_hot AS
SELECT *
FROM afbet_realsports_gh.t_realsports_selection_source
WHERE created_at >= DATEADD(day, -70, GETDATE());

-- Step2.
ALTER TABLE afbet_realsports_gh.t_realsports_selection_source RENAME TO t_realsports_selection_source_cold;
ALTER TABLE afbet_realsports_gh.t_realsports_selection_source_hot RENAME TO t_realsports_selection_source;
```

ng
```sql
-- Step1. 
CREATE TABLE afbet_realsports_ng.t_realsports_selection_source_hot AS
SELECT *
FROM afbet_realsports_ng.t_realsports_selection_source
WHERE created_at >= DATEADD(day, -70, GETDATE());

-- Step2.
ALTER TABLE afbet_realsports_ng.t_realsports_selection_source RENAME TO t_realsports_selection_source_cold;
ALTER TABLE afbet_realsports_ng.t_realsports_selection_source_hot RENAME TO t_realsports_selection_source;
```

**Open DAG**
* afbet_realsports.t_realsports_selection_source_cold_copy
* afbet_realsports.t_realsports_selection_source_hot_delete

**Switch Time**
* gh
* ng

### src_realsports_all_orders_v12 

gh
```sql
-- Step1. 
CREATE TABLE afbet_realsports_gh.src_realsports_all_orders_v12_hot AS
SELECT *
FROM afbet_realsports_gh.t_realsports_selection_source
WHERE created_at >= DATEADD(day, -70, GETDATE());

-- Step2.
ALTER TABLE afbet_realsports_gh.t_realsports_selection_source RENAME TO t_realsports_selection_source_cold;
ALTER TABLE afbet_realsports_gh.t_realsports_selection_source_hot RENAME TO t_realsports_selection_source;
```

ng
```sql
-- Step1. 
CREATE TABLE afbet_realsports_ng.t_realsports_selection_source_hot AS
SELECT *
FROM afbet_realsports_ng.t_realsports_selection_source
WHERE created_at >= DATEADD(day, -70, GETDATE());

-- Step2.
ALTER TABLE afbet_realsports_ng.t_realsports_selection_source RENAME TO t_realsports_selection_source_cold;
ALTER TABLE afbet_realsports_ng.t_realsports_selection_source_hot RENAME TO t_realsports_selection_source;
```

**Open DAG**
* afbet_realsports.t_realsports_selection_source_cold_copy
* afbet_realsports.t_realsports_selection_source_hot_delete

**Switch Time**
* gh
* ng


### t_facts_sporty_uof_messages_odds_change 

![[Screenshot 2026-03-26 at 4.20.45 PM.png]]

>[!WARNING] Encore Prod 有 table 但是空的

### logs_patron_fe_behav 

>[!WARNING] Encore Prod 沒有 table

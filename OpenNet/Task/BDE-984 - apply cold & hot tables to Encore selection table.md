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

### t_realsports_selection ok
gh
```sql
-- Step1. 
CREATE TABLE afbet_realsports_gh.t_realsports_selection_hot AS
SELECT *
FROM afbet_realsports_gh.t_realsports_selection
WHERE created_at >= DATEADD(day, -40, GETDATE());

-- Step2.
ALTER TABLE afbet_realsports_gh.t_realsports_selection RENAME TO afbet_realsports_gh.t_realsports_selection_cold;
ALTER TABLE afbet_realsports_gh.t_realsports_selection_hot RENAME TO afbet_realsports_gh.t_realsports_selection;
```

ng
```sql
-- Step1. 
CREATE TABLE afbet_realsports_ng.t_realsports_selection_hot AS
SELECT *
FROM afbet_realsports_ng.t_realsports_selection
WHERE created_at >= DATEADD(day, -40, GETDATE());

-- Step2.
ALTER TABLE afbet_realsports_ng.t_realsports_selection RENAME TO afbet_realsports_ng.t_realsports_selection_cold;
ALTER TABLE afbet_realsports_ng.t_realsports_selection_hot RENAME TO afbet_realsports_ng.t_realsports_selection;
```

**Open DAG**
* afbet_realsports.t_realsports_selection_cold_copy
* afbet_realsports.t_realsports_selection_hot_delete

**Switch Time**
* gh
* ng

### t_realsports_bet ok 
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
WHERE created_at >= DATEADD(day, -70, GETDATE());

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
WHERE created_at >= DATEADD(day, -70, GETDATE());

-- Step2.
ALTER TABLE afbet_main_ng.t_order_record RENAME TO t_order_record_cold;
ALTER TABLE afbet_main_ng.t_order_record_hot RENAME TO t_order_record;
```

**Open DAG**
* afbet_realsports.t_realsports_bet_cold_copy
* afbet_realsports.t_realsports_bet_hot_delete

**Switch Time**
* gh
* ng
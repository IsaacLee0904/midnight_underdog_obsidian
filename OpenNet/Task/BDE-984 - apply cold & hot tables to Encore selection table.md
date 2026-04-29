## Information

* **Jira Ticket** : [Link](https://opennetltd.atlassian.net/jira/software/projects/BDE/boards/117?jql=assignee%20%3D%20712020%3A4caf52e7-1e32-4e9b-ade5-8c262db7c7f0&selectedIssue=BDE-984)
* **Reference** : [Link](https://opennetltd.atlassian.net/wiki/spaces/DET/pages/3334045924/Redshift+-+Cold+Hot+Table+Structure)
* **Branch** : <span style="color:rgb(8, 186, 118)">feature/BDE-984encore_cold_hot_table</span>

## Requirement

Apply below table in Encore to cold / hot tables : 

| **DB**         | **Table**                                 | **Brand** | **Data**      | check |
| -------------- | ----------------------------------------- | --------- | ------------- | ----- |
| `bi_warehouse` | `t_realsports_selection`                  | Encore    | last 40 days  | ✅     |
| `bi_warehouse` | `t_realsports_bet`                        | Encore    | last 70 days  | ✅     |
| `bi_warehouse` | `t_order_record`                          | Encore    | last 200 days | ✅     |
| `bi_warehouse` | `t_facts_sporty_uof_messages_odds_change` | Encore    | last 20 days  | ❌     |
| `bi_warehouse` | `logs_patron_fe_behav`                    | Encore    | last 40 days  | ❌     |
| `bi_warehouse` | `t_realsports_selection_source`           | Encore    | last 70 days  | ✅     |
| `bi_report`    | `src_realsports_all_orders_v12`           | Encore    | last 90 days  | ✅     |

## Implement

>[!attention]
> 1. `cold_copy` 跟 `hot_delete` 要設定到前一天避免 Sporty Encore 之間互相影響
> 2. 完成後到 `bi_announcement` 公告
> 3. 去 `table_update_check` 刪除 _ hot 的紀錄，避免一直跳 alarm

### t_realsports_selection

![[BDE-984 - encore t_realsports_selection cold hot table pattern]]

### t_realsports_bet 

![[BDE-984 - encore t_realsports_bet cold hot table pattern]]

### t_order_record 

![[BDE-984 - encore t_order_record cold hot table pattern]]

### t_realsports_selection_source 

![[BDE-984 - encore t_realsports_selection_source cold hot table pattern]]

### src_realsports_all_orders_v12 




### t_facts_sporty_uof_messages_odds_change 

![[Screenshot 2026-03-26 at 4.20.45 PM.png]]

>[!WARNING] Encore Prod 有 table 但是空的

### logs_patron_fe_behav 

>[!WARNING] Encore Prod 沒有 table

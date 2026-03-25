## Information

* **Jira Ticket** : [Link](https://opennetltd.atlassian.net/jira/software/projects/BDE/boards/117?jql=assignee%20%3D%20712020%3A4caf52e7-1e32-4e9b-ade5-8c262db7c7f0&selectedIssue=BDE-984)
* **Reference** : [Link](https://opennetltd.atlassian.net/wiki/spaces/DET/pages/3334045924/Redshift+-+Cold+Hot+Table+Structure)

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


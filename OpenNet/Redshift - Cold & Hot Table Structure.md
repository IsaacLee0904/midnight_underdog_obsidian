## Hot Table Data

| **DB**         | **Table**                                 | **Brand** | **Data**      |
| -------------- | ----------------------------------------- | --------- | ------------- |
| `bi_warehouse` | `t_realsports_selection`                  | Sporty    | last 40 days  |
| `bi_warehouse` | `t_realsports_bet`                        | Sporty    | last 70 days  |
| `bi_warehouse` | `t_order_record`                          | Sporty    | last 200 days |
| `bi_warehouse` | `t_facts_sporty_uof_messages_odds_change` | Sporty    | last 20 days  |
| `bi_warehouse` | `logs_patron_fe_behav`                    | Sporty    | last 40 days  |
| `bi_warehouse` | `t_realsports_selection_source`           | Sporty    | last 70 days  |
| `bi_report`    | `src_realsports_all_orders_v12`           | Sporty    | last 90 days  

hot tables only save recent data and cold tables with suffix `_cold`have all data.





















[[BDE-984 - apply cold & hot tables to Encore selection table]]



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

## Project Background

隨著 Warehouse 資料量持續增長，部分資料表資料量過大，導致篩選與更新效率低落，EX. `afbet_realsports_ng.t_realsports_selection` 資料筆數已超過 **900 億筆** ，在 ETL pipeline 中我們會使用 `DELETE` 與 `INSERT` 來更新 warehouse 資料，而 `DELETE` 有時候要花非常久的時間才會完成造成 cluster 資源消耗極大影響其他排程

![[Pasted image 20260325182316.png]]

## Solution

為解決此問題，我們將一張龐大的資料表拆分為兩張：

- **冷資料表**：儲存全部資料，但使用頻率低
- **熱資料表**：僅儲存近期資料，使用頻率高


## Impletment
















[[BDE-984 - apply cold & hot tables to Encore selection table]]





**Step1. 先去查詢系統表 `svv_table_info`**

```sql
SELECT * 
FROM svv_table_info
WHERE "schema" LIKE 'afbet_realsports_%'
AND "table" IN ('t_realsports_selection');
```
![[Screenshot 2026-03-24 at 4.25.01 PM.png]]

這張系統表會顯示所有 table 相關的資訊包含 `sortkey`, `size`, `unsorted`, `stats_off` 等資訊，以下說明部分觀測效能會使用到的 columns

* `tbl_rows`: 估算的總 row 數 (含 ghost rows)
* `skew_rows`: Row 在各 slice 分佈的偏斜比例
* `skew_sortkey1` : Sort key column 值分佈的偏斜程度
* `unsorted`: 代表 table 中沒有被 sort 的比例，值越大代表越高比例沒有 sort ，效能會越差
* `stats_off`: 統計資訊過時程度 (越高越需要 `ANALYZE`)
* `vacuum_sort_benefit`: 執行 VACUUM SORT 預期能帶來的效益（0–100）

**Step2. 當遇到 table 查詢效能瓶頸的四種解決方式**

==Method 1 vacuum sort & vacuum cluster==

使用 `unsorted` 與 `vacuum_sort_benefit` 一起評估，前者代表沒有 sort 的比例，後者則代表 sort 可以帶來的優化提升
* vacuum sort : 對整張表進行 sort
* vacuum cluster : 針對較近期的資料進行 sort

>[!WARNING]
>由於我們的資料量都太大了， vacuum sorted 跟 vacuum cluster 都會吃掉太多效能不建議使用

==Method 2 ANALYZE==

`ANALYZE` 會掃描 table 的資料，**更新 query planner 用來產生執行計畫的統計資訊 (statistics)**，Redshift 的 query optimizer 是 cost-based，它會根據這些統計數字估算每種執行路徑的成本，然後選最便宜的那條走，可以透過看 `stats_off` 欄位來決定 `ANALYZE` 後的效能提升，數值越小越無效

 ```sql
 ANALYZE afbet_pocket_gh.t_pocket_pay_record_ch_request;
 ```

==Method 3 vacuum delete==

有些資料已經從 table 上 diable 了但還沒從 disk 上刪掉，會影響效能，這個有另外的排程在做

==Method 4 Walk around==

當我們不能使用 vacuum sort & vacuum cluster 的時候，有兩種 walk around 的方法

* Create 一個新的 table，重新設置 sort key，然後再把舊資料灌進去
* 做冷熱資料分離處理 : cold table & hot table


### Reference
[Redshift ANALYZE Docs](https://docs.aws.amazon.com/zh_tw/redshift/latest/dg/r_ANALYZE.html)
[Redshift VACUUM Docs](https://docs.aws.amazon.com/zh_tw/redshift/latest/dg/r_VACUUM_command.html)
[Redshift query plan Docs](https://docs.aws.amazon.com/zh_tw/redshift/latest/dg/c_data_redistribution.html)

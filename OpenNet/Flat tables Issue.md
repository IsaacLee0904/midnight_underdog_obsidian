有時候 DA 會有一些 long query 造成 Redshift 的效能問題，過去有針對常常被 JOIN 的 table 做成 Flat tables 可以建議他們去 JOIN 這些表，透過使用 Flat table 可以避免掉多張表的 heavy join，如果需要新增欄位時 @Dominykas Zenkevicius

>[!WARNING]
> 每張表留存的時間不一樣所以需要確保他們需要的資料區間還有在 Flat table 裡面，詳見以下 Exist Flat tables 章節每一個 table 的 data retention

### How it works

In our case, flat tables are created in Redshift to combine data from two or more individual RDS tables. The pipelines for populating Redshift flat tables with data work as follows : 

1. Data from individual RDS tables is joined together and written to Amazon S3
2. Combined data is loaded from Amazon S3 into a staging table in Redshift
3. The staging table is used to delete duplicate records from the flat table
4. Data from the staging table is inserted into the flat table

### Exist Flat tables

<mark style="background: #FFB86CA6;">`afbet_pocket_{country code}.t_pocket_pay_record_ch_request`</mark>

1. **basic info and pipeline overview** 
	The `afbet_pocket_{country code}.t_pocket_pay_record_ch_request` flat tables combine `afbet_pocket_{country code}.t_pocket_pay_record` and `afbet_pocket_{country code}.t_pocket_pay_ch_request` table data into one. The tables were created to:
	
	- **speed up queries that join these tables together** - slow running queries are an issue in Metabase dashboards, as they result in timeouts;
	- **decrease query costs** - since individual tables do not have a common distribution key that could be used when joining tables, they need to be redistributed every time they are joined together, which is a costly operation.
	
![[afbet_pocket.t_pocket_pay_record_ch_request_shard|800]]


2. **Schedule interval** : every 10 mins

3. **Pipeline** : [_afbet_pocket.t_pocket_pay_record_ch_request_shard_](http://172.31.71.234:8080/dags/afbet_pocket.t_pocket_pay_record_ch_request_shard/grid?tags=flat_tables&search=afbet_pocket.t_pocket_pay_record_ch_request_shard "http://172.31.71.234:8080/dags/afbet_pocket.t_pocket_pay_record_ch_request_shard/grid?tags=flat_tables&search=afbet_pocket.t_pocket_pay_record_ch_request_shard") 

4. **Distribution and sort keys**
	 The <span style="color:rgb(255, 192, 0)">`pay_id`</span> column is used as the distribution key in these flat tables, as it is used to delete duplicate records from the target table before inserting new records. The table uses <span style="color:rgb(255, 192, 0)">`pay_id`</span>, <span style="color:rgb(255, 192, 0)">`status`</span> and <span style="color:rgb(255, 192, 0)">`trade_code`</span> columns as a sort keys, since they are most often used to filter data in BI DAG queries that use `t_pocket_pay_record` and `t_pocket_pay_record_ch_request` Redshift tables.

5. **Data retention** : <mark style="background: #FF5582A6;">last 90 days</mark>

<mark style="background: #FFB86CA6;">`afbet_realsports_{country code}.t_realsports_bet_subbet_selection`</mark>

1. **basic info and pipeline overview**
	The `afbet_realsports_{country code}.t_realsports_bet_subbet_selection` flat tables combine `afbet_realsports_{country code}.t_realsports_bet`, `afbet_realsports_{country code}.t_realsports_subbet` and `afbet_realsports_{country code}.t_realsports_selection` table data into one. The tables were created to :
	
	- **decrease query costs** - since not all of the tables mentioned above have a common distribution key that could be used when joining tables, they need to be redistributed every time they are joined together, which is a costly operation;
	- **simplify queries** - using a single flat table instead of multiple individual realsports tables simplifies the query and makes it shorter and easier to read.
	
![[afbet_realsports.t_realsports_bet_subbet_selection|800]]

2. **Schedule interval** : every 10 mins

3. **Pipeline** : [_afbet_realsports.t_realsports_bet_subbet_selection_shard_](http://172.31.71.234:8080/dags/afbet_realsports.t_realsports_bet_subbet_selection_shard/grid?tags=flat_tables&search=afbet_realsports.t_realsports_bet_subbet_selection_shard "http://172.31.71.234:8080/dags/afbet_realsports.t_realsports_bet_subbet_selection_shard/grid?tags=flat_tables&search=afbet_realsports.t_realsports_bet_subbet_selection_shard") & [_afbet_realsports.t_realsports_bet_subbet_selection_u_shard_](http://172.31.71.234:8080/dags/afbet_realsports.t_realsports_bet_subbet_selection_u_shard/grid?tags=flat_tables&search=afbet_realsports.t_realsports_bet_subbet_selection_u_shard&tab=code "http://172.31.71.234:8080/dags/afbet_realsports.t_realsports_bet_subbet_selection_u_shard/grid?tags=flat_tables&search=afbet_realsports.t_realsports_bet_subbet_selection_u_shard&tab=code") 

4. **Distribution and sort keys**
	 The <span style="color:rgb(255, 192, 0)">`bet_id`</span> column is used as the distribution key in these flat tables, as it is used to delete duplicate records from the target table before inserting new records, and most of the queries that join other tables to `t_realsports_bet` table use `bet_id` as the join column. The table uses <span style="color:rgb(255, 192, 0)">`bet_id`</span>, <span style="color:rgb(255, 192, 0)">`subbet_id`</span> and <span style="color:rgb(255, 192, 0)">`selection_id`</span> columns as a sort keys, since they are most often used to filter data in BI DAG queries that use `t_realsports_bet`, `t_realsports_subbet` and `t_realsports_selection` Redshift tables.

5. **Data retention** : last 15 days

<mark style="background: #FFB86CA6;">`afbet_realsports_{country code}.t_realsports_bet_selection_extend`</mark>

1. **basic info and pipeline overview**
	The `afbet_realsports_{country code}.t_realsports_bet_subbet_selection` flat tables combine `afbet_realsports_{country code}.t_realsports_bet`, `afbet_realsports_{country code}.t_realsports_subbet` and `afbet_realsports_{country code}.t_realsports_selection` table data into one. The tables were created to :
	
	- **decrease query costs** - since not all of the tables mentioned above have a common distribution key that could be used when joining tables, they need to be redistributed every time they are joined together, which is a costly operation;
	- **simplify queries** - using a single flat table instead of multiple individual realsports tables simplifies the query and makes it shorter and easier to read.

![[afbet_realsports. t_realsports_bet_selection_extend|800]]
1. **Schedule interval** : every 10 mins

2. **Pipeline** : [_afbet_realsports.t_realsports_bet_subbet_selection_shard_](http://172.31.71.234:8080/dags/afbet_realsports.t_realsports_bet_subbet_selection_shard/grid?tags=flat_tables&search=afbet_realsports.t_realsports_bet_subbet_selection_shard "http://172.31.71.234:8080/dags/afbet_realsports.t_realsports_bet_subbet_selection_shard/grid?tags=flat_tables&search=afbet_realsports.t_realsports_bet_subbet_selection_shard") & [_afbet_realsports.t_realsports_bet_subbet_selection_u_shard_](http://172.31.71.234:8080/dags/afbet_realsports.t_realsports_bet_subbet_selection_u_shard/grid?tags=flat_tables&search=afbet_realsports.t_realsports_bet_subbet_selection_u_shard&tab=code "http://172.31.71.234:8080/dags/afbet_realsports.t_realsports_bet_subbet_selection_u_shard/grid?tags=flat_tables&search=afbet_realsports.t_realsports_bet_subbet_selection_u_shard&tab=code") 

3. **Distribution and sort keys**
	 The <span style="color:rgb(255, 192, 0)">`bet_id`</span> column is used as the distribution key in these flat tables, as it is used to delete duplicate records from the target table before inserting new records, and most of the queries that join other tables to `t_realsports_bet` table use `bet_id` as the join column. The table uses `bet_id`, `subbet_id` and `selection_id` columns as a sort keys, since they are most often used to filter data in BI DAG queries that use `t_realsports_bet`, `t_realsports_subbet` and `t_realsports_selection` Redshift tables.

4. **Data retention** : last 15 days

<mark style="background: #FFB86CA6;">`afbet_realsports_{country code}.t_realsports_bet_selection_source`</mark>

1. **basic info and pipeline overview**
	The `afbet_realsports_{country code}.t_realsports_bet_selection_source` flat tables combine `afbet_realsports_{country code}.t_realsports_selection_source` tables with a `bet_id` column from `afbet_realsports`.`t_realsports_bet_selection` production tables. The tables were created to :
	
	- **decrease query costs** - there are multiple pipelines where `t_realsports_bet_selection_source` tables are joined to `t_realsports_bet_subbet_selection` flat tables on the `selection_id` column, which results in data redistribution every time, which is a costly operation. Using `bet_id` column for joins instead would help avoid this.

![[afbet_realsports.t_realsports_bet_selection_source|800]]

2. **Schedule interval** : every 10 mins

3. **Pipeline** : [_afbet_realsports.t_realsports_bet_selection_source_](http://172.31.71.234:8080/dags/afbet_realsports.t_realsports_bet_selection_source/grid "http://172.31.71.234:8080/dags/afbet_realsports.t_realsports_bet_selection_source/grid") 

4. **Distribution and sort keys**
	 The <span style="color:rgb(255, 192, 0)">`bet_id`</span> column is used as the distribution key in these flat tables, as it is used for deleting duplicate records from the target table before inserting new records, and, as mentioned above, joining the new flat tables to `t_realsports_bet_subbet_selection` flat tables. <span style="color:rgb(255, 0, 0)">The table does not have any sort keys currently</span>

5. **Data retention** : last 61 days

#### Reference
[Flat tables Confluence](https://opennetltd.atlassian.net/wiki/spaces/BT/pages/2950758486/Flat+tables)
[[Ad hoc - missing data in t_pocket_pay_record_ch_request]]
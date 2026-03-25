## Pipeline Overview

![[sirtlow.png]]
DE team often get requests from the DA team to add new tables to our Data Warehouse, which means we need to sync data from a Production DB (usually MySQL) to BI Warehouse (Redshift) through Warehouse Airflow

## Workflow
#### Step1. Confirm Info
When get requests from DAs to add new tables to our data warehouse, we need to know some additional info
- Brand : Sporty / Encore
- Database
- Table (default is all the columns in the table)
![[Kichard Cheng UW-DA 405 PM.png]]

#### Step2. Some follow up questions
- How much historical data is needed ?
- How often should the tables be updated ? Intervals between data updates should be as big as possible to minimise load on our data warehouse
- Can the table experience data latency between records being created / updated in the application / backend and corresponding record being inserted/updated in the production tables ?  
> In some cases it could be that a record is created / updated in the application / backend at one time, but inserted / updated in the product on table after a short delay, resulting in data latency between the application/backend and a production table. Data latency is more common in cases where create_time and update_time values are set when a record is created or updated in the application/backend, instead of when it’s inserted or updated in the production table itself. 
   For example, let’s say that a record is created at 12:29:30 in the application/backend with a create_time value of 12:29:30, but, due to heavy workloads on the production database, is only inserted into the production table at 12:30:15. If the Airflow DAG that copies the data runs at 12:30:00 and collects data with create_time values ranging between 12:00 and 12:30, it will miss the record, as it’s haven’t yet been inserted into the production table at 12:30:00. If cases like this are a possibility, we usually make our DAG run schedule intervals overlap by up to a few minutes to make sure production table records that were created/updated with a delay and missed during the initial DAG run are collected during the next DAG run. In this example, having an overlap of 1 minute would ensure that the next DAG run, scheduled at 13:00:00, will collect records created between 12:29 and 13:00, picking up the record that was missed during the previous DAG run.

==Solution 1 overlap method==  
少數的 DAG 是利用這種方法，但本質上沒有解決問題，因為通常是 RDS 的寫入延遲  
      
==Solution 2 ask BE upgrade==  
比如說他們可能要加機器，或者說他們要做一些可能 workaround 的處理，暫時增加資料處理的速度，類似這樣。所以我們可能會需要去跟後端溝通這件事情

==Solution 3 add a timestamp==
也有可能在 Redshift 上新增一個爛位紀錄 timestamp 來讓 DA, DS 知道資料進來的時間，但如果資料很重要，那可能要請他們直接從 RDS 拿資料
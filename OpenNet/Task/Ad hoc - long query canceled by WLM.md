## Information
![[Screenshot 2026-05-05 at 10.56.19 AM.png]]

some of long query will canceled by WLM abort action of Query Monitoring Rule "abort_bi_dag_queries", this queue is setup for monitoring long query avoid warehouse loading increase 

## Solution
we have set 2 query queues on Spory Redshift for DA DAG : 

* **BI DAG queries** : normal queries, query timeout is 20 mins
- **BI DAG long queries** : complex and higher importance queries, query timeout is 60 mins

For most DAG queries, they run in first queue by default and should finish in 20 mins.  
only if DAG has `high_importance` tag, queries will be running in the second queue with 60 mins timeout.  
  
we don't want people frequently run long queries without optimization and increase cluster loading.
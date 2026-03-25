We have a [check_redshift_cs](https://airflow-warehouse-pub-prod-bi.on.sportybet2.com/dags/check_redshift_cs/grid) DAG monitoring WLMQueueLength of BI DAG queries queue. if it detects too many queued queries in it, it will enable Concurrency Scaling of it and send this alert.
![[Data 0.04.0. 10.0. 18.0.25.0. 26.0, 22.0.22.0. 21.0.23.0.png]]
#### What to do ? 

this is usually caused by high Redshift loading, go to AWS console [Redshift Query monitoring](https://eu-central-1.console.aws.amazon.com/redshiftv2/home?region=eu-central-1#/cluster-details?cluster=sporty-pub-prod-bi-warehouse&tab=queries) to check if there is hanging query or user trigger long query need to be killed.

when Redshift loading is back to normal, the same DAG will automatically disable CS for it.
![[Reason WMOururLength of BI DAG queries below 3 for 10 min.png]]

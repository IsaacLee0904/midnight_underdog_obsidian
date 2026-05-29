tags: #OpenNet  #data-engineering #infra  #airflow #kubernetes  

---
### Background

![[Screenshot 2026-04-02 at 11.36.26 AM.png]]

Rejection pipeline 跑的 Airflow Worker 會在 `de_alert4933` 的 channel 中反覆出現 EKS RAM (>85%) 的 Alarm，起因是因為 Rejection pipeline 有 memory leak 的問題，進到 [Grafana](https://grafana-pub-prod-misc.k8s.on.sportybet2.com/d/adk6avn6n2tc0aee/airflow-eks?orgId=1&from=now-30d&to=now&timezone=utc&var-datasource=P28ADB2B68CA29654&var-airflow_id=airflow-rejection&var-dag_id=$__all&var-airflow_pod=airflow-da-scheduler-658d68cfc7-zzvmg&viewPanel=panel-80)  可以看到這個問題 (如下圖)

![[Screenshot 2026-04-02 at 11.27.04 AM.png]]

Our Rejection Airflow workers experienced continuous memory leak issue, eventually triggering HPA (70% of 13Gi ≈ 9.1 GB). Manual redeploy was the only way to recover. The issue only appeared on Rejection Airflow, not other Airflow sharing the same version and configuration.

##### Request Reference
- [Slack Thread](https://opennetltd.slack.com/archives/C07347VTBB2/p1775101205797429)
- [Jira Ticket](https://opennetltd.atlassian.net/browse/BDE-1119?atlOrigin=eyJpIjoiZGNkMjU0MDYyZGM1NGNkZWEyMzg0MjU3MTZkYzQ1ZmIiLCJwIjoiaiJ9)
##### Environment
- Airflow 2.10.2, CeleryExecutor on EKS
- Worker pod : 13Gi memory limit
- EFS (NFS-backed) mounted at /opt/airflow/data for intermediate feather files
    `fs-08d1863cf0c4b6c88.efs.eu-central-1.amazonaws.com:/airflow/prod/sporty/rejection/data  /opt/airflow/data`
- Main pipeline runs every minute, processing small per-country DataFrames

### Temporary Solution

![[Rejection pipeline Memory Alert#Solution 定期重啟 Worker]]

### Investigation

#### Phase 1 : DB Connection Leak

**Hypothesis** : Unclosed DB connections function - `DatabaseManager.get_dataframe_by_sql_without_sharding_db` accumulate resources in long-lived worker processes.

**Test** : Applied conn.close() fix in the function. Ran 5,000 consecutive calls locally (result as below) no measurable effect on RSS, open file descriptors, or MySQL thread count.
![[Pasted image 20260416183023.png]]

**Result** : Fix did not address production memory leak issue but still necessary since could impact the DB workload.

### Phase 2 : Python High-Watermark Effect (Child Heap)

**Hypothesis** : CPython's memory allocator retains freed memory internally rather than returning it to the OS, causing RSS floor to rise.

*This was a known. behavior in long-lived Python process (CeleryExecutor). ref - [What we learned after running Airflow on Kubernetes for 2 years](https://medium.com/apache-airflow/what-we-learned-after-running-airflow-on-kubernetes-for-2-years-0537b157acfd)

**Test** : Designed a 3-phase experiment: small DataFrame (LIMIT 500) → large (LIMIT 1000) → small (LIMIT 500). Used production row counts downloaded from S3 to simulate actual pipeline workload.

**Local result** : RSS after Phase 3 remained at Phase 2 level, validating high-watermark behavior.
![[result.png]]

**Result** : Child process RSS stays flat at ~195 MB via ps aux over few days. Which means the memory leak was not from application heap.
![[Screenshot 2026-05-29 at 3.25.36 PM.png]]

### Phase 3 : Kernel-Level Investigation
**Hypothesis** : According to the result from phase 2, the memory growth might from kernel-level.

**Test** : After few data of observation, discovered that memory growth is not visible at Python process level (ps aut) but at cgroup / pod level (/sys/fs/cgroup/memory.stat).

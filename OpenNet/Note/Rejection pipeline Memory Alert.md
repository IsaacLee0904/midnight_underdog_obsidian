![[Screenshot 2026-04-02 at 11.36.26 AM.png]]

有時候 Rejection pipeline 跑的 Airflow 會在 `de_alert4933` 的 channel 中遇到 pod RAM 過高的 Alarm，起因是因為 Rejection pipeline 有 memory leak 的問題，進到 [Grafana](https://grafana-pub-prod-misc.k8s.on.sportybet2.com/d/adk6avn6n2tc0aee/airflow-eks?orgId=1&from=now-30d&to=now&timezone=utc&var-datasource=P28ADB2B68CA29654&var-airflow_id=airflow-rejection&var-dag_id=$__all&var-airflow_pod=airflow-da-scheduler-658d68cfc7-zzvmg&viewPanel=panel-80)  可以看到這個問題 (如下圖)

![[Screenshot 2026-04-02 at 11.27.04 AM.png]]

#### Solution : 定期重啟 Worker

**Step1. 開啟 [Rancher](https://rancher.on.sportybet2.com/dashboard/c/c-m-b2n7dcll/explorer/apps.deployment)** : 到 [sporty-pub-prod-bi](https://rancher.on.sportybet2.com/dashboard/c/c-m-b2n7dcll/explorer) 裡面選 airflow-rejection
![[Screenshot 2026-04-02 at 12.01.15 PM.png]]
![[Screenshot 2026-04-02 at 12.02.11 PM.png]]

**Step2. 重啟 workers**

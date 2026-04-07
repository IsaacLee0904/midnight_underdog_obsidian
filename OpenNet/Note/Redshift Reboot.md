Every Tuesday about 1 PM UTC+8 Redshift has a weekly reboot than there will be lots of DAG run fail alert as below example.

After reboot there are three things need to keep in mind :

1. Slack Channel alert : see if the fail task keep failing

2. [Airflow Grafana](https://grafana-pub-prod-misc.k8s.on.sportybet2.com/d/ddvknf88xc3y8d/airflow-alerts?orgId=1&from=now-1h&to=now&timezone=browser) : Monitoring whether the `Default pool scheduled slots` metric stays high — if so, it means pending tasks aren't being processed
![[Screenshot 2026-03-24 at 2.00.27 PM.png]]

3. [Airflow DAG Grafana](https://grafana-pub-prod-misc.k8s.on.sportybet2.com/d/adk6avn6n2tc0aee/airflow-eks?orgId=1&from=now-1h&to=now&timezone=utc&var-datasource=P28ADB2B68CA29654&var-airflow_id=airflow-warehouse&var-dag_id=$__all&var-airflow_pod=) : Monitoring whether the `DAG run schedule delay > 20 mins` metrics decrease

4. 然後在 Slack 群組傳送
```text
^ Redshift weekly reboot with patch version upgrade

^ Redshift is catching up
```

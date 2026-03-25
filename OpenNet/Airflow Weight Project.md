### Problem Statement

There are two core issues with the current `priority_weight` configuration in Airflow :

**1. No unified standard behind weight assignment** Without a consistent baseline, critical pipelines cannot be reliably prioritized by the scheduler when workers are buzzy — increasing the risk of SLA misses and delayed downstream reports.

**2. The default `weight_rule = "downstream"` causes weight distortion** By default, Airflow accumulates the weights of all downstream tasks onto each upstream task, artificially inflating the effective weight of upstream tasks. This results in inconsistent effective weights across tasks within the same DAG, and renders any cross-DAG priority comparison meaningless.

### Idea 1. Deal with the downstream issue

Using `weight_rule = "absolute"`
```python
default_args = { "priority_weight": 50, "weight_rule": "absolute", 
```
According to Kevin's, we are using Airflow's default config, which sets `weight_rule = "downstream"`. Under this rule, each task's effective weight is calculated by accumulating the weights of all its downstream tasks — meaning upstream tasks end up with artificially inflated effective weights. This makes priority comparison across DAGs unreliable and causes the `priority_weight` setting to lose its intended meaning.
### Idea 2. Custom Weight Rule  

```python
# plugins/priority_weight_plugin.py
from airflow.plugins_manager import AirflowPlugin
from airflow.listeners import hookimpl

TAG_WEIGHT_MAP = {
    "critical": 100,
    "important": 50,
    "normal": 10,
    "low": 1,
}
PRIORITY_TAGS = ["critical", "important", "normal", "low"] # for example
DEFAULT_WEIGHT = 10

class PriorityWeightListener:
    @hookimpl
    def on_dag_run_created(self, dag_run, session):
        dag = dag_run.dag
        weight = DEFAULT_WEIGHT
        for tag in PRIORITY_TAGS:  
            if tag in [t.name for t in dag.tags]:
                weight = TAG_WEIGHT_MAP[tag]
                break
        for task in dag.tasks:
            task.priority_weight = weight

class PriorityWeightPlugin(AirflowPlugin):
    name = "priority_weight_plugin"
    listeners = [PriorityWeightListener()]
```









### Reference
[Document of Airflow Weight](https://airflow.apache.org/docs/apache-airflow/stable/administration-and-deployment/priority-weight.html)
[Medium article About Airflow Weight](https://blog.devgenius.io/implementing-weighted-capacity-for-instance-fleets-in-apache-airflow-141349fbf6a6)

tags : #airflow #data-engineering #workflow #orchestration
source : Notion

---

## Airflow Basic

### Airflow 簡介

Apache Airflow 是一款 open source 工作排程工具，用於編排複雜的資料流和工作流程，它允許使用者以程式設計方式定義、安排和監控任務，確保它們以預定的順序和時間執行。Airflow 的核心思想是 **"workflow is code"**，這意味著可以使用 Python 程式碼來定義和管理工作流程。

### Airflow 核心架構

- **Web Server**：主要提供用戶操作使用，基於 Flask 框架建構，允許使用者通過 web 查看 DAG 執行狀況、log 等，也提供 API 方便與其他服務整合
- **Scheduler**：在 Airflow 架構中負責解析 DAG 並調度任務執行，會定期檢查 DAG 的變化，並根據任務相依關係和時間安排任務
- **Executor**：實際執行任務的 component，支援多種類型：
  - `SequentialExecutor`：順序執行任務，適用於開發和測試
  - `LocalExecutor`：在 local 執行任務
  - `CeleryExecutor`：使用 Celery 分布式任務列隊執行任務
  - `KubernetesExecutor`：在 K8s 集群中動態創建 Pod 執行任務
- **Metadata Database**：儲存了 Airflow 的所有 metadata，包含 DAG、任務狀態、變數等，常用的有 PostgreSQL、MySQL、SQLite 等
- **Worker**：分布式環境中實際執行任務的節點，當使用 CeleryExecutor 或 KubernetesExecutor 時，Worker 會從任務列隊中獲取任務並執行

### Airflow 核心概念

- **DAG**：代表了一個工作流，由一組 task 組成，用於定義任務之間依賴關係
- **TASK**：DAG 中的每個節點，可以是任何可執行操作，例如執行腳本、呼叫 API 或執行 SQL 查詢
- **Operator**：定義任務的特定行為，原生提供 `BashOperator`、`PythonOperator` 等
- **Scheduler**：負責解析 DAG 文件根據調度計畫觸發任務的執行
- **Executor**：負責任務實際執行的組件
- **XCom**：用於**任務之間傳遞數據的機制**，允許一個任務將資料存儲在資料庫中，另一個任務可以從資料庫中讀取

```python
from airflow.operators.python_operator import PythonOperator

def push_data(**kwargs):
    kwargs['ti'].xcom_push(key='my_key', value='my_value')

def pull_data(**kwargs):
    value = kwargs['ti'].xcom_pull(key='my_key')
    print(f'Received value: {value}')

push_task = PythonOperator(task_id='push_task', python_callable=push_data, provide_context=True, dag=dag)
pull_task = PythonOperator(task_id='pull_task', python_callable=pull_data, provide_context=True, dag=dag)

push_task >> pull_task
```

---

## Airflow 執行模式

### LocalExecutor

本地模式是最簡單的執行模式，適用於開發或測試環境，所有任務都在同一個機器上運作。

```bash
executor = LocalExecutor
```

使用情境：開發和測試、單機部署且工作規模較小

### CeleryExecutor

分散式執行模式，適用於正式環境，使用 Celery 作為任務列隊。

```bash
executor = CeleryExecutor
broker_url = redis://localhost:6379/0
result_backend = redis://localhost:6379/0
```

使用情境：生產環境、需要跨多個節點執行任務

### KubernetesExecutor

專為 Kubernetes 集群設計，**允許每個任務獨立在 Kubernetes Pod 中執行**，提供更高的資源隔離性和彈性。

```yaml
executor = KubernetesExecutor
```

使用情境：容器化環境、需要高資源隔離性

---

## Airflow CLI

```bash
# 確認 airflow 版本
airflow version

# 啟動各個 airflow 組件
airflow webserver
airflow scheduler

# 查看 DAG 列表
airflow dags list

# trigger DAG
airflow dags trigger <dag_id>

# 查看 log
airflow tasks logs <dag_id> <task_id> <execution_date>
# 例：airflow tasks logs example_dag task_1 2023-10-01

# 暫停和恢復 DAG
airflow dags pause <dag_id>
airflow dags unpause <dag_id>
```

---

## Airflow DAG Develop

### DAG 基本結構

```python
from airflow import DAG
from airflow.operators.dummy_operator import DummyOperator
from airflow.operators.python_operator import PythonOperator
from datetime import datetime

dag = DAG(
    'example_dag',
    description='DAG 範例',
    schedule_interval='@daily',
    start_date=datetime(2023, 1, 1),
    catchup=False
)

start_task = DummyOperator(task_id='start_task', dag=dag)
end_task = DummyOperator(task_id='end_task', dag=dag)

def print_hello():
    print("Hello, Airflow!")

hello_task = PythonOperator(task_id='hello_task', python_callable=print_hello, dag=dag)

start_task >> hello_task >> end_task
```

### DAG Config 主要參數

- `dag_id`：DAG 的唯一識別
- `description`：DAG 的描述訊息，會以 hover 的形式顯示在 UI 上
- `schedule`：DAG 的執行區間，可以是 `@daily`、`@hourly` 等，也可以用 cron 表示式
- `start_date`：DAG 的開始日期
- `catchup`：是否啟用 catchup，決定 DAG 啟動時是否執行過去沒有執行的任務

### 任務相依性

```python
task_a >> [task_c, task_d]   # task_c 和 task_d 都依賴 task_a
[task_c, task_d] >> task_e   # task_e 依賴 task_c 和 task_d
```

---

## Operators

### BashOperator

```python
# 執行 Bash 指令
list_files_task = BashOperator(task_id='list_files', bash_command='ls -l')

# 執行 Bash Script
run_script_task = BashOperator(task_id='run_script', bash_command='./script.sh')

# 傳遞參數
print_message_task = BashOperator(
    task_id='print_message',
    bash_command='echo "Message: {{ params.message }}"',
    params={'message': 'Hello, Airflow!'},
)
```

### PythonOperator

```python
def print_message(message):
    print(message)

message_task = PythonOperator(
    task_id='message_task',
    python_callable=print_message,
    op_kwargs={'message': 'Hello, Airflow with parameters!'},
    dag=dag,
)
```

---

## Sensors

Sensors 是一種特殊類型的 Operator，用於監控外部系統的狀態變化，透過持續**輪詢**某一個條件是否滿足直到條件成立或超時。常用於等待外部事件：文件傳遞、資料庫更新、API 完成等。

配置參數：
- `poke_interval`：兩次檢查之間的時間間隔
- `timeout`：sensor 等待條件滿足的最大時間
- `mode`：執行模式，可以是 `poke` 或 `reschedule`

```python
wait_for_file = FileSensor(
    task_id="wait_for_file",
    filepath="/path/to/your/file.txt",
    poke_interval=30,   # 每 30 秒檢查一次
    timeout=3600,       # 最多等待 1 小時
    mode="poke",
)
```

---

## Task Group

Airflow 提供任務分組（Task Group）功能，將相關的任務組織在一起簡化 DAG 的結構。

```python
from airflow.utils.task_group import TaskGroup

with DAG(dag_id="task_group_example", start_date=days_ago(1), schedule_interval=None) as dag:
    start = DummyOperator(task_id="start")

    with TaskGroup(group_id="group_1") as group_1:
        task_1 = DummyOperator(task_id="task_1")
        task_2 = DummyOperator(task_id="task_2")
        task_3 = DummyOperator(task_id="task_3")
        task_1 >> task_2 >> task_3

    end = DummyOperator(task_id="end")
    start >> group_1 >> end
```

## SubDAG

> **💡 注意**
> SubDAG 在 Airflow 1.x 時很流行，但在 2.x 後官方建議以 **TaskGroup 替代 SubDAG**，因為 TaskGroup 可以提供更好的性能與更乾淨的語法。

tags : #airflow #dbt #cosmos #data-engineering
source : Notion

---

## Why use Astronomer Cosmos？

在還沒有 Cosmos 之前，dbt 和 Airflow 基本上是兩個沒什麼相關的工具。過去如果要在 Airflow 中使用 dbt，通常就是用 `BashOperator`，這樣會導致 dbt 變成一個黑盒子，開發和維護上都很痛苦。

### Before Cosmos：dbt with `BashOperator`

```python
from airflow.operators.bash_operator import BashOperator

dbt_run = BashOperator(
    task_id='dbt_run',
    bash_command='dbt run',
    dag=dag
)
```

問題：
- `dbt_run` 會將整個 dbt 全部執行，中間如果有錯，就需要整個重新跑
- 需要 `dbt test` 又需要新增一個 `BashOperator`
- 想將每個 model 層級拆開，需要自行寫 `dbt run --models {model_folder}`

### After Cosmos：dbt with `DbtTaskGroup` and `DbtDAG`

使用 Astronomer Cosmos 之後，dbt 在 Airflow 中的執行不再是黑盒子，在 `TaskGroup` 中可以查看所有上下游的關係，即便有任一執行出錯，也可以單獨重新執行。

---

## Astronomer Cosmos Setup

### Install astro

```bash
brew install astro

astro dev init  # create a cosmos template
```

### Customized settings

**Dockerfile：**

```yaml
FROM quay.io/astronomer/astro-runtime:12.1.1

WORKDIR "/usr/local/airflow"

COPY requirements.txt .
RUN python -m virtualenv dbt_venv && source dbt_venv/bin/activate && \
    pip install --no-cache-dir -r requirements.txt && deactivate
```

**packages.txt（OS packages）：**

```
build-essential
libatlas-base-dev
tzdata
git
libkrb5-dev
```

**requirements.txt（Python packages）：**

```
astronomer-cosmos
apache-airflow-providers-apache-hive
dbt-core
dbt-hive
```

**docker-compose.override.yml（掛載 dbt 專案）：**

```yaml
version: "3.1"
services:
  scheduler:
    volumes:
      - ./dbt:/usr/local/airflow/dbt:rw
    networks:
      - hive_net

  webserver:
    volumes:
      - ./dbt:/usr/local/airflow/dbt:rw
    networks:
      - hive_net

  triggerer:
    volumes:
      - ./dbt:/usr/local/airflow/dbt:rw
    networks:
      - hive_net

networks:
  hive_net:
    name: hive_default
    external: true
```

### Restart

```bash
astro dev restart
```

---

## dbt Setup in Cosmos

### profile_config

Cosmos 需要透過在 dag 中設定 `profile_config` 的方式來與 dbt 連結。

**方法一：Using a profile mapping（使用 Cosmos 內建設定）**

```python
from cosmos.profiles import SnowflakeUserPasswordProfileMapping

profile_config = ProfileConfig(
    profile_name="my_profile_name",
    target_name="my_target_name",
    profile_mapping=SnowflakeUserPasswordProfileMapping(
        conn_id="my_snowflake_conn_id",
        profile_args={
            "database": "my_snowflake_database",
            "schema": "my_snowflake_schema",
        },
    ),
)

dag = DbtDag(profile_config=profile_config, ...)
```

**方法二：Using your own profiles.yml（傳統 dbt 做法）**

```python
from cosmos.config import ProfileConfig

profile_config = ProfileConfig(
    profile_name="my_snowflake_profile",
    target_name="dev",
    profiles_yml_filepath="/path/to/profiles.yml",
)

dag = DbtDag(profile_config=profile_config, ...)
```

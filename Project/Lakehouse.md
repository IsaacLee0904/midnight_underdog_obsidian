![[Pasted image 20261021140756.png|700]]


## Tech Stack Selection

### Write Engine


|                         | Athena | Glue job (spark) | EMR (spark) | Glue job (DuckDB) | Trino |
| ----------------------- | ------ | ---------------- | ----------- | ----------------- | ----- |
| Reading TSV             |        | `spark.read.csv` |             |                   |       |
| MERGE / UPDATE / DELETE |        |                  |             |                   |       |
| Table maintenance       |        |                  |             |                   |       |
| Partitioning (`day()`)  |        |                  |             |                   |       |
| Data volume limit       |        |                  |             |                   |       |
| Startup time            |        |                  |             |                   |       |
| Pricing                 |        |                  |             |                   |       |
| Airflow integration     |        |                  |             |                   |       |

### Storage + Data Catalog 

### Query Engine (Future)
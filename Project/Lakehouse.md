![[Pasted image 20261021140756.png|700]]


## Tech Stack Selection

### Write Engine


|                         | Athena                                                                     | Glue job (spark)            | EMR (spark)                 | Glue job (DuckDB) | Trino |
| ----------------------- | -------------------------------------------------------------------------- | --------------------------- | --------------------------- | ----------------- | ----- |
| Reading TSV             | OpenCSVSerDe reads every column as string, so types are CAST in the MERGE; | `spark.read.csv` sets types | `spark.read.csv` sets types |                   |       |
| MERGE / UPDATE / DELETE |                                                                            |                             |                             |                   |       |
| Table maintenance       |                                                                            |                             |                             |                   |       |
| Partitioning (`day()`)  |                                                                            |                             |                             |                   |       |
| Data volume limit       |                                                                            |                             |                             |                   |       |
| Startup time            |                                                                            |                             |                             |                   |       |
| Pricing                 |                                                                            |                             |                             |                   |       |
| Airflow integration     |                                                                            |                             |                             |                   |       |

### Storage + Data Catalog 

### Query Engine (Future)
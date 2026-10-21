![[Pasted image 20261021140756.png|700]]


## Tech Stack Selection

### Write Engine

|                                                                              | Athena                                                                                                                                                | Glue job (spark)                                                                              | EMR (spark)                                                                                   | Glue job (DuckDB)                                                                               | Trino                                                                                                                                                 |
| ---------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------- |
| Reading TSV (TSV carries no types, so every engine must define or cast them) | ⚠️ Types can't be taken at read time: columns are declared as string, and a CAST list generated from the Iceberg table schema is applied in the MERGE | ✅  Schema taken from the target table (`spark.table(target).schema`)                          | ✅  Schema taken from the target table (`spark.table(target).schema`),                         | ✅ Schema taken from the attached Iceberg table                                                  | ⚠️ Types can't be taken at read time: columns are declared as string, and a CAST list generated from the Iceberg table schema is applied in the MERGE |
| MERGE / UPDATE / DELETE                                                      | ✅ `MERGE INTO` in one commit (merge-on-read)                                                                                                          | ✅ `MERGE INTO` copy-on-write or merge-on-read is configurable                                 | ✅ `MERGE INTO` copy-on-write or merge-on-read is configurable                                 | ⚠️ `MERGE INTO`; merge-on-read only; docs state UPDATE / DELETE do not work on sorted tables    | ✅ `MERGE` in one commit (merge-on-read)                                                                                                               |
| Table maintenance                                                            | ✅ `OPTIMIZE` (compaction), `VACUUM` (expire snapshots, remove orphan files)                                                                           | ✅ `rewrite_data_files` (bin-pack / sort / z-order), `expire_snapshots`, `remove_orphan_files` | ✅ `rewrite_data_files` (bin-pack / sort / z-order), `expire_snapshots`, `remove_orphan_files` | ❌ Not supported; pair with S3 Tables automatic maintenance, the Glue table optimizer, or Athena | ✅ `OPTIMIZE` (compaction), `VACUUM` (expire snapshots, remove orphan files)                                                                           |
| Partitioning (`day()`)                                                       | ✅                                                                                                                                                     | ✅                                                                                             | ✅                                                                                             | ⚠️ Docs list only identity, bucket and truncate                                                 | ✅<br>                                                                                                                                                 |
| Pricing                                                                      | Per TB scanned (about $5/TB), 10 MB minimum per query;                                                                                                | Per DPU-hour, billed per second with a 1-minute minimum                                       | Per vCPU and memory used, billed per second                                                   | Per DPU-hour, billed per second with a 1-minute minimum                                         | EKS / EC2 instance hours (charged while idle)                                                                                                         |
| Airflow integration                                                          | AthenaOperator                                                                                                                                        | GlueJobOperator                                                                               | EmrServerlessStartJobOperator                                                                 | GlueJobOperator                                                                                 | SQLExecuteQueryOperator                                                                                                                               |


### Storage + Data Catalog 

|                                                                                                                            | A: S3 general purpose bucket + Glue Data Catalog                                                                                                                                                                 | B: S3 Tables (table bucket)                                                                                                                                                                                                                              |
| -------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Table maintenance**  <br>(Are compaction, snapshot expiry and orphan file cleanup automatic, or do we schedule them?)    | ⚠️ Not automatic by default. Enable the Glue table optimizers (compaction, snapshot retention, orphan file deletion), or schedule Athena / Spark maintenance jobs                                                | ✅ Automatic by default                                                                                                                                                                                                                                   |
| **Cost**  <br>(Besides storage, what else do we pay for?)                                                                  | billed per DPU-hour ($0.44, per second, 1-minute minimum) if enabled                                                                                                                                             | S3 Tables storage and requests, plus a per-object monitoring fee ($0.025 per 1,000 objects) and compaction fees ($0.002 per 1,000 objects and $0.005 per GB processed for bin-pack).                                                                     |
| **Engine compatibility**  <br>(Can Athena, Redshift Spectrum, Spark and DuckDB read and write? Can a future OLAP connect?) | ✅ Athena, Redshift Spectrum and Spark read and write natively through Glue. ⚠️ DuckDB's Glue catalog support is marked experimental. Future OLAP (Trino, StarRocks): native Glue connectors are widely available | ✅ Athena, Redshift Spectrum and EMR / Glue Spark through the Glue integration; DuckDB through `ENDPOINT_TYPE s3_tables` (shown in the AWS Glue + DuckDB blog). ⚠️ Future OLAP must support an Iceberg REST catalog with SigV4 signing; check each engine |
| **Access flexibility**  <br>(Can we move files with `aws s3 cp`, set lifecycle rules, or repair tables by hand?)           | ✅ Standard S3 bucket: `aws s3 cp`, S3 Lifecycle, replication and manual repair all work                                                                                                                          | ⚠️ Tables are written through Iceberg engines and the S3 Tables APIs; general S3 features such as Lifecycle rules and manual file operations are not the normal path (verify which S3 features apply before relying on them)                             |

### Query Engine (Future)





### Athena

  

- [MERGE INTO](https://docs.aws.amazon.com/athena/latest/ug/merge-into-statement.html)

- [OPTIMIZE](https://docs.aws.amazon.com/athena/latest/ug/optimize-statement.html)

- [VACUUM](https://docs.aws.amazon.com/athena/latest/ug/vacuum-statement.html)

- [OpenCSVSerDe for processing CSV](https://docs.aws.amazon.com/athena/latest/ug/csv-serde.html)

- [SELECT, including the `"$path"` pseudo-column](https://docs.aws.amazon.com/athena/latest/ug/select.html)

- [Creating Iceberg tables (partition transforms)](https://docs.aws.amazon.com/athena/latest/ug/querying-iceberg-creating-tables.html)

- [Service quotas (query timeouts)](https://docs.aws.amazon.com/athena/latest/ug/service-limits.html)

- [Athena pricing](https://aws.amazon.com/athena/pricing/)

  

### AWS Glue

  

- [Using the Iceberg framework in AWS Glue](https://docs.aws.amazon.com/glue/latest/dg/aws-glue-programming-etl-format-iceberg.html)

- [Job properties and worker types](https://docs.aws.amazon.com/glue/latest/dg/add-job.html)

- [AWS Glue pricing](https://aws.amazon.com/glue/pricing/)

- [AWS Big Data Blog: Cost-effective ETL with DuckDB and Amazon S3 Tables on AWS Glue](https://aws.amazon.com/blogs/big-data/cost-effective-etl-with-duckdb-and-amazon-s3-tables-on-aws-glue/)

  

### Spark and Iceberg

  

- [Spark CSV data source options](https://spark.apache.org/docs/latest/sql-data-sources-csv.html)

- [Iceberg Spark writes (MERGE INTO, UPDATE, DELETE)](https://iceberg.apache.org/docs/latest/spark-writes/)

- [Iceberg Spark procedures (rewrite_data_files, expire_snapshots, remove_orphan_files)](https://iceberg.apache.org/docs/latest/spark-procedures/)

- [Iceberg table configuration (copy-on-write / merge-on-read modes)](https://iceberg.apache.org/docs/latest/configuration/)

- [Iceberg table spec (partition transforms)](https://iceberg.apache.org/spec/)

  

### EMR Serverless

  

- [Pre-initialized capacity](https://docs.aws.amazon.com/emr/latest/EMR-Serverless-UserGuide/pre-init-capacity.html)

- [Amazon EMR pricing (includes EMR Serverless)](https://aws.amazon.com/emr/pricing/)

  

### DuckDB

  

- [Reading CSV files](https://duckdb.org/docs/current/data/csv/overview)

- [Reading faulty CSV files (rejects table)](https://duckdb.org/docs/current/data/csv/reading_faulty_csv_files)

- [Writing to Iceberg](https://duckdb.org/docs/current/core_extensions/iceberg/writing_to_iceberg)

- [New DuckDB-Iceberg features in v1.5.3 (MERGE INTO)](https://duckdb.org/2026/05/29/new-iceberg-features)

  

### Trino

  

- [Iceberg connector](https://trino.io/docs/current/connector/iceberg.html)

- [Hive connector (CSV tables)](https://trino.io/docs/current/connector/hive.html)

- [Fault-tolerant execution](https://trino.io/docs/current/admin/fault-tolerant-execution.html)

# Background

The Fraud team's **Multiple Account Auto Block (MAD)** actively scans the user base to detect clusters of linked accounts (sharing name / withdrawal account / deposit instrument / device, etc.) that are involved in risk flows, and applies block policies to them. (Currently Phase 1 only NG active)

The MAD rule engine runs its cluster-linkage evaluation on the **Fraud team's own RDS**. Several of the linkage lookups — e.g. _"users sharing the same account name / withdrawal account / device across the whole user base"_ — are expensive to compute directly on that RDS and hit performance limits.

To remove that bottleneck, **BI pre-computes the required identity / linkage lookup tables in the warehouse and delivers them into the Fraud RDS (reverse ETL)**, so the rule engine can query ready-made lookup tables instead of scanning raw data at evaluation time. This document describes that BI pipeline — how each table is built and how it is delivered.

For the full product requirements, conditions, and table schema, see the Fraud team's docs below :

- [Multiple Account Auto Block - Phase 1](https://opennetltd.atlassian.net/wiki/spaces/SPOR/pages/4249452564 "https://opennetltd.atlassian.net/wiki/spaces/SPOR/pages/4249452564") : Project document
- [DB table schema - MAD Phase 1](https://opennetltd.atlassian.net/wiki/spaces/SPOR/pages/4623794206 "https://opennetltd.atlassian.net/wiki/spaces/SPOR/pages/4623794206") : Schema & ownership reference
- [Build contract · MAD identity index (BI → Fraud sync)](https://opennetltd.atlassian.net/wiki/spaces/SPOR/pages/4623532130 "https://opennetltd.atlassian.net/wiki/spaces/SPOR/pages/4623532130") : Main development Spec
- [BDE-1578: BI - Multiple Account Auto Block Reverse ETLDone](https://opennetltd.atlassian.net/browse/BDE-1578)
- [BDE-1671: BI - Add new MAD pipeline t_fraud_name_holderDone](https://opennetltd.atlassian.net/browse/BDE-1671)

# Pipeline Overview

## Asset : `t_fraud_identity_asset` & `t_fraud_identity_asset_holder`
![[Pasted image 20260929153830.png]]

**Pipeline**
- orignal :
    - [bi_pocket.t_fraud_identity_asset](https://airflow-da-pub-prod-bi.on.sportybet2.com/dags/bi_pocket.t_fraud_identity_asset/grid "https://airflow-da-pub-prod-bi.on.sportybet2.com/dags/bi_pocket.t_fraud_identity_asset/grid")
    - [bi_pocket.t_fraud_identity_asset_holder](https://airflow-da-pub-prod-bi.on.sportybet2.com/dags/bi_pocket.t_fraud_identity_asset_holder/grid "https://airflow-da-pub-prod-bi.on.sportybet2.com/dags/bi_pocket.t_fraud_identity_asset_holder/grid")
        
- reverse
    - [reverse_etl_fraud.t_fraud_identity_asset](https://airflow-da-pub-prod-bi.on.sportybet2.com/dags/reverse_etl_fraud.t_fraud_identity_asset/grid?dag_run_id=scheduled__2026-09-16T06%3A15%3A00%2B00%3A00&base_date=2026-09-16T06%3A15%3A00Z&tab=logs&task_id=task_t_fraud_identity_asset.export_asset_ng "https://airflow-da-pub-prod-bi.on.sportybet2.com/dags/reverse_etl_fraud.t_fraud_identity_asset/grid?dag_run_id=scheduled__2026-09-16T06%3A15%3A00%2B00%3A00&base_date=2026-09-16T06%3A15%3A00Z&tab=logs&task_id=task_t_fraud_identity_asset.export_asset_ng")
    - [reverse_etl_fraud.t_fraud_identity_asset_holder](https://airflow-da-pub-prod-bi.on.sportybet2.com/dags/reverse_etl_fraud.t_fraud_identity_asset_holder/grid?dag_run_id=scheduled__2026-09-14T02%3A20%3A00%2B00%3A00&tab=logs&base_date=2026-09-14T02%3A20%3A00Z&task_id=task_t_fraud_identity_asset_holder.export_holder_ng "https://airflow-da-pub-prod-bi.on.sportybet2.com/dags/reverse_etl_fraud.t_fraud_identity_asset_holder/grid?dag_run_id=scheduled__2026-09-14T02%3A20%3A00%2B00%3A00&tab=logs&base_date=2026-09-14T02%3A20%3A00Z&task_id=task_t_fraud_identity_asset_holder.export_holder_ng")
        

**Purpose**
- Forward (`t_fraud_identity_asset`) : user → the withdrawal accounts they hold
- Holder (`t_fraud_identity_asset_holder`) : account → the users who hold it
    

**Scope**
`asset_type = 2` & `action = 2`. Plaintext `TRIM`, no hash.

### Source
`bi_warehouse.afbet_pocket_{country}.t_pocket_bank_asset` — synced from pocket MySQL RDS by `warehouse_engineer`, **hourly :50**. Filter: `asset_type = 2 AND action = 2 AND TRIM(account_number) <> ''`

| Column                  | Use                                               |
| ----------------------- | ------------------------------------------------- |
| `user_id`               | grain                                             |
| `asset_type` / `action` | filter (`2` / `2`); `action 2 → signal_type 2`    |
| `account_number`        | identifier — `TRIM`, string, never cast to number |
| `account_name`          | carried; never a key                              |
| `is_del`                | source soft-delete                                |
| `create_time`           | → `first_seen_at` (business time)                 |
| `update_time`           | recency guard + delta watermark                   |

### Schema design

```sql
-- base: one row per (country_code, user_id, signal_type, account_number)
CREATE TABLE bi_report.bi_pocket.t_fraud_identity_asset (
    country_code    VARCHAR(16)  NOT NULL,
    user_id         VARCHAR(64)  NOT NULL,
    signal_type     SMALLINT     NOT NULL,   -- 2 = WITHDRAWAL
    account_number  VARCHAR(64)  NOT NULL,   -- plaintext TRIM
    account_name    VARCHAR(128),
    is_del          SMALLINT     NOT NULL DEFAULT 0,
    first_seen_at   TIMESTAMP,               -- business time (source create_time)
    src_update_time TIMESTAMP,               -- staleness guard, not delivered
    created_at      TIMESTAMP    NOT NULL DEFAULT SYSDATE,  -- frozen on insert
    updated_at      TIMESTAMP    NOT NULL DEFAULT SYSDATE   -- reverse-ETL delta key
)
DISTKEY (account_number) SORTKEY (country_code, signal_type, account_number);

-- holder: one row per account -> active users
CREATE TABLE bi_report.bi_pocket.t_fraud_identity_asset_holder (
    country_code   VARCHAR(16)    NOT NULL,
    account_number VARCHAR(64)    NOT NULL,
    signal_type    SMALLINT       NOT NULL,
    user_count     INTEGER        NOT NULL,   -- true active count (uncapped)
    user_ids       VARCHAR(65535) NOT NULL,   -- JSON array, LISTAGG, capped to byte limit
    first_seen_at  TIMESTAMP,
    updated_at     TIMESTAMP      NOT NULL DEFAULT SYSDATE
)
DISTKEY (account_number) SORTKEY (country_code, signal_type, account_number);
```

>[!note] Business time = `first_seen_at`/`last_seen_at` (from source). `created_at`/`updated_at` = our warehouse `SYSDATE` (`created_at` frozen once; `updated_at` bumped each run). `src_update_time` = internal guard, not delivered. RDS column defaults fall back to the Fraud clock only if we omit them — we always send them.

### Update logic
- **Base** (`:00`) — window source event time `(create_time OR update_time)` + **2 h look-back**; dedup to earliest `create_time`; guarded upsert: `first_seen_at = LEAST`, `is_del`/`account_name` overwrite only if `src_update_time` is newer, `updated_at = SYSDATE`.
- **Holder** (`:10`) — recompute touched accounts (`base.updated_at ∈ window`) over active rows (`is_del = 0`); emptied → `user_count = 0, user_ids = '[]'`.
- **Reverse** (`:15` forward, `:20` holder) — delta on `updated_at` → `UNLOAD` → S3 (5 MB) → Aurora `REPLACE INTO`; `created_at` frozen; holder uses `LOAD_ESCAPED_BY = ""` for valid JSON.
### Key decisions

- `created_at` **frozen** : `REPLACE INTO` re-inserts rows, so a dynamic `SYSDATE` would reset it every delivery.
- `first_seen_at` **kept in holder** : lets the backfill chunk on business time; **not** delivered to RDS.
- **Zero-DELETE** : emptied accounts sent as `count=0, []`, overwritten via `REPLACE INTO`, no `DELETE`.
- **Guarded upsert** : out-of-order re-runs never overwrite newer data with older.
### Backfill
On initial build every row shares one `created_at`/`updated_at` (the build day) → can't window on `updated_at` (one giant batch → Aurora lag / holder OOM). So backfill windows on **business time** `first_seen_at`.


## Device : `t_fraud_identity_device` & `t_fraud_identity_device_holder`
![[Pasted image 20260929154747.png]]

**Pipeline**
- original
    - [bi_patron.t_fraud_identity_device](https://airflow-da-pub-prod-bi.on.sportybet2.com/dags/bi_patron.t_fraud_identity_device/grid "https://airflow-da-pub-prod-bi.on.sportybet2.com/dags/bi_patron.t_fraud_identity_device/grid")
    - [bi_patron.t_fraud_identity_device_holder](https://airflow-da-pub-prod-bi.on.sportybet2.com/dags/bi_patron.t_fraud_identity_device_holder/grid "https://airflow-da-pub-prod-bi.on.sportybet2.com/dags/bi_patron.t_fraud_identity_device_holder/grid")
        
- reverse
    - [reverse_etl_fraud.t_fraud_identity_device](https://airflow-da-pub-prod-bi.on.sportybet2.com/dags/reverse_etl_fraud.t_fraud_identity_device/grid "https://airflow-da-pub-prod-bi.on.sportybet2.com/dags/reverse_etl_fraud.t_fraud_identity_device/grid")
    - [reverse_etl_fraud.t_fraud_identity_device_holder](https://airflow-da-pub-prod-bi.on.sportybet2.com/dags/reverse_etl_fraud.t_fraud_identity_device_holder/grid?dag_run_id=scheduled__2026-09-19T22%3A20%3A00%2B00%3A00&task_id=end_etl&tab=logs&base_date=2026-09-19T22%3A20%3A00Z "https://airflow-da-pub-prod-bi.on.sportybet2.com/dags/reverse_etl_fraud.t_fraud_identity_device_holder/grid?dag_run_id=scheduled__2026-09-19T22%3A20%3A00%2B00%3A00&task_id=end_etl&tab=logs&base_date=2026-09-19T22%3A20%3A00Z")
        

**Purpose**
- **Forward** `t_fraud_identity_device`: user → login devices.
- **Holder** `t_fraud_identity_device_holder`: device → users (the cluster expansion).
    
**Scope**
native App only (ANDROID / IOS). Plaintext `TRIM`, case preserved, no hash. No `is_del`, no `signal_type` — `device_id` alone is the identifier and bindings are never removed.

### Source
`bi_warehouse.afbet_patron_{country}.t_patron_user_device` — synced from patron MySQL RDS by `warehouse_engineer`, **hourly :50**. Filter: `platform IN ('ANDROID','IOS') AND device_id IS NOT NULL AND CHAR_LENGTH(TRIM(device_id)) >= 16 AND LOWER(TRIM(device_id)) NOT IN ('undefined','null','none','unknown','nan','nil')`.

| Column             | Use                                             |
| ------------------ | ----------------------------------------------- |
| `user_id`          | grain                                           |
| `device_id`        | identifier — `TRIM`, case preserved             |
| `platform`         | filter (`ANDROID`/`IOS`) + carried              |
| `status`           | carried unfiltered (fraud filters at read time) |
| `create_time`      | fallback for `last_seen_at`                     |
| `update_time`      | recency guard + delta watermark                 |
| `last_active_time` | → `last_seen_at` (business time / liveness)     |

### Schema design
```sql
-- base: one row per (country_code, user_id, device_id)
CREATE TABLE bi_report.bi_patron.t_fraud_identity_device (
    country_code    VARCHAR(16)  NOT NULL,
    user_id         VARCHAR(64)  NOT NULL,
    device_id       VARCHAR(64)  NOT NULL,   -- plaintext TRIM, case preserved
    platform        VARCHAR(16),             -- ANDROID / IOS
    status          VARCHAR(16),             -- carried unfiltered
    last_seen_at    TIMESTAMP,               -- business time (source last_active_time), liveness
    src_update_time TIMESTAMP,               -- staleness guard, not delivered
    created_at      TIMESTAMP    NOT NULL DEFAULT SYSDATE,  -- frozen on insert
    updated_at      TIMESTAMP    NOT NULL DEFAULT SYSDATE   -- reverse-ETL delta key
)
DISTKEY (device_id) SORTKEY (country_code, device_id);

-- holder: one row per device -> all its users (all-time, append-only)
CREATE TABLE bi_report.bi_patron.t_fraud_identity_device_holder (
    country_code VARCHAR(16)    NOT NULL,
    device_id    VARCHAR(64)    NOT NULL,
    user_count   INTEGER        NOT NULL,   -- all-time distinct users
    user_ids     VARCHAR(65535) NOT NULL,   -- JSON array, LISTAGG, capped to byte limit
    last_seen_at TIMESTAMP,                 -- MAX; backfill chunking only, not delivered
    updated_at   TIMESTAMP      NOT NULL DEFAULT SYSDATE
)
DISTKEY (device_id) SORTKEY (country_code, device_id);
```

`DISTKEY(device_id)` so the holder rebuild groups co-located. Holder is all-time append-only — `user_count` only grows.

> [!note] `last_seen_at` is **source business time**. `created_at` / `updated_at` are the **BI (warehouse) clock** (`SYSDATE`) — `created_at` set once and frozen, `updated_at` bumped each upsert and drives the reverse-ETL delta. `src_update_time` is a BI-internal guard, not delivered. The RDS `DEFAULT`/`ON UPDATE` on `created_at`/`updated_at` falls back to the Fraud RDS clock only if the load omits the column — so the loader always supplies both.

### Update logic

- **Base** (`:00`) — window source event time `(create_time OR update_time)` + **2 h look-back**; dedup to latest state per `(user_id, device_id)`; guarded upsert: `last_seen_at = GREATEST`, `platform`/`status` overwrite only if `src_update_time` is newer, `updated_at = SYSDATE`, `created_at` frozen.
- **Holder** (`:10`) — recompute touched devices (`base.updated_at ∈ window`): `user_count`, `user_ids = LISTAGG(user_id)`, `last_seen_at = MAX`. No delete/emptied case (list never shrinks).
- **Reverse** (`:15` forward, `:20` holder) — delta on `updated_at` → `UNLOAD` → S3 (5 MB) → Aurora `REPLACE INTO`; `created_at` frozen; holder uses `LOAD_ESCAPED_BY = ""` for valid JSON. `last_seen_at` / `src_update_time` not delivered.
    

### Key decisions

- `created_at` **frozen** : `REPLACE INTO` re-inserts rows, so a dynamic `SYSDATE` would reset it every delivery.
- **No** `is_del` : bindings are never removed; the holder is all-time append-only and `last_seen_at` carries liveness (fraud applies its own recency th reshold at read time).
- `last_seen_at` **kept in holder** : used to chunk the backfill on business time; **not** delivered to RDS.
- **Guarded upsert** : out-of-order re-runs never overwrite newer data with older.
    
### Backfill
On initial build every row shares one `created_at`/`updated_at` (the build day) → can't window on `updated_at` (one giant batch → Aurora lag / holder OOM).



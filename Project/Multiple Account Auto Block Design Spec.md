
t_fraud_identity_asset & t_fraud_identity_asset_holder

PR : https://github.com/opennetltd/data_analysis/pull/10485

### [BDE-1578] Multiple Account Auto Block (MAD Phase 1) — Asset Binding & Reverse ETL Pipeline

#### Background
To support the Fraud team's Multiple Account Auto Block (MAD Phase 1) project, BI builds an end-to-end data pipeline that maintains withdrawal bank account bindings in Redshift and synchronizes both Forward (user $\to$ accounts) and Reverse Holder (account $\to$ users) tables into Fraud MySQL RDS.

ticket : https://opennetltd.atlassian.net/browse/BDE-1578?atlOrigin=eyJpIjoiZDUwYTI0ZDRhMjAzNDYwODg3YjUzNzc3NjIzOWYzNmEiLCJwIjoiaiJ9
spec : https://opennetltd.atlassian.net/wiki/spaces/SPOR/pages/4623532130/DB+table+schema+source+from+BI+-+MAD+Phase+1#4.-t_fraud_identity_asset-and-_holder

#### Data Flow & Architecture
```markdown
┌─────────────────────────────────────────────────────────────┐
│ Source DB (MySQL / DataShare)                               │
│ bi_warehouse.afbet_pocket_{country}.t_pocket_bank_asset     │
└──────────────────────────────┬──────────────────────────────┘
                               │ 
                               ▼
┌─────────────────────────────────────────────────────────────┐
│ Redshift Base Table: bi_pocket.t_fraud_identity_asset       │
│  - Grain: (country_code, user_id, 2, account_number)        │
│  - In-place upsert with LEAST(first_seen_at) & staleness guard│
└──────────────┬──────────────────────────────┬───────────────┘
               │                              │
               │                              │ 
               ▼                              ▼
┌──────────────────────────────┐  ┌──────────────────────────────┐
│ Redshift Holder Table        │  │ Fraud RDS Forward Table      │
│ t_fraud_identity_asset_holder│  │ t_fraud_identity_asset       │
│  - Grain: (account_number)   │  │ (Query: User's bank accounts)│
│  - LISTAGG active users      │  └──────────────────────────────┘
│  - 501-user truncation cap   │
└──────────────┬───────────────┘
               │ 
               ▼
┌──────────────────────────────┐
│ Fraud RDS Holder Table       │
│ t_fraud_identity_asset_holder│
│ (Query: Shared card graph)   │
└──────────────────────────────┘

```

#### Key Design Decisions
1. Persistent created_at in Redshift Base Table:
Why: The RDS write-back uses LOAD DATA ... REPLACE INTO, which deletes and re-inserts matching rows in MySQL. Dynamically generating SYSDATE AS created_at would reset the timestamp on every subsequent update.
Design: created_at is set once upon the initial INSERT in Redshift and remains untouched during UPDATEs, guaranteeing timestamp stability across repeated RDS deliveries.

2. first_seen_at in Redshift Holder Table:
Why: Historical backfill contains ~20M–30M rows. Loading all data in a single batch causes Aurora replication lag and lock spikes.
Design: Retaining MIN(first_seen_at) in the Redshift holder table allows clean, deterministic 90-day sliding window catchups (2018-03-26 to 2026-08-21) without hardcoding dates. (This column is omitted during RDS unload to keep the RDS schema strictly aligned with the Fraud spec).

3. Zero-DELETE Pattern for Emptied Accounts:
Accounts where all users are unlinked (is_del = 1) are exported as user_count = 0 and user_ids = []. This overwrites RDS via REPLACE INTO without requiring DELETE FROM statements or table locks.

4. Literal JSON Formatting (LOAD_ESCAPED_BY = ""):
Ensures UNLOAD outputs ["uid1", "uid2"] without stripping internal double quotes during Aurora S3 load, producing valid MySQL JSON.

#### Workflow & DAG Breakdown
Daily Incremental DAGs (Production)
1. `bi_pocket.t_fraud_identity_asset` 
- Purpose: Ingests daily bank binding changes from the source DB into the Redshift base table.
- Logic: Performs in-place upsert with LEAST(first_seen_at) to preserve historical binding times and freezes created_at on initial insert.

2. `bi_pocket.t_fraud_identity_asset_holder`
- Purpose: Computes and maintains the materialized holder mapping in Redshift.
- Logic: Sensor-waits on the base DAG, aggregates active users per touched account via LISTAGG (501-user cap), and handles emptied accounts as user_count=0, user_ids=[].

3. `reverse_etl_fraud.t_fraud_identity_asset`
- Purpose: Syncs daily forward binding deltas from Redshift to Fraud MySQL RDS.
- Logic: Sensor-waits on the base DAG, UNLOADs 5MB S3 slices, and loads via REPLACE INTO with frozen created_at.

4. `reverse_etl_fraud.t_fraud_identity_asset_holder`
- Purpose: Syncs daily reverse holder deltas from Redshift to Fraud MySQL RDS.
- Logic: Sensor-waits on the holder DAG, UNLOADs from the materialized holder table, and loads into RDS with LOAD_ESCAPED_BY="" to preserve valid JSON array formatting.

Historical Backfill DAGs
1. `bi_pocket.t_fraud_identity_asset_backfill`
- Purpose: Backfills the Redshift base table from historical source data.

2. `bi_pocket.t_fraud_identity_asset_holder_backfill`
- Purpose: Populates the Redshift holder table from the backfilled base table.

3. `reverse_etl_fraud.t_fraud_identity_asset_backfill`
- Purpose: Delivers the historical forward table from Redshift to Fraud MySQL RDS.

4. `reverse_etl_fraud.t_fraud_identity_asset_holder_backfill`
- Purpose: Delivers the historical holder table from Redshift to Fraud MySQL RDS.






t_fraud_identity_device & t_fraud_identity_device_holder

PR : https://github.com/opennetltd/data_analysis/pull/10510

### [BDE-1578] Multiple Account Auto Block (MAD Phase 1) — Device Binding & Reverse ETL Pipeline

#### Background
To support the Fraud team's Multiple Account Auto Block (MAD Phase 1) project, BI builds an end-to-end data pipeline that maintains user login-device bindings in Redshift and synchronizes both Forward (user $\to$ devices) and Reverse Holder (device $\to$ users) tables into Fraud MySQL RDS. This mirrors the Asset pipeline, with device-specific semantics (last-seen liveness, no soft-delete, native-app only).

ticket : https://opennetltd.atlassian.net/browse/BDE-1578
spec : https://opennetltd.atlassian.net/wiki/spaces/SPOR/pages/4623532130/DB+table+schema+source+from+BI+-+MAD+Phase+1#5.-t_fraud_identity_device-and-_holder

#### Data Flow & Architecture
```markdown
┌─────────────────────────────────────────────────────────────┐
│ Source DB (warehouse)                               │
│ bi_warehouse.afbet_patron_{country}.t_patron_user_device    │
└──────────────────────────────┬──────────────────────────────┘
                               │  platform IN (ANDROID, IOS), device_id valid
                               ▼
┌─────────────────────────────────────────────────────────────┐
│ Redshift Base Table: bi_patron.t_fraud_identity_device      │
│  - Grain: (country_code, user_id, device_id)  [no signal_type]│
│  - In-place upsert with GREATEST(last_seen_at) & staleness guard│
│  - No is_del: bindings are never removed                     │
└──────────────┬──────────────────────────────┬───────────────┘
               │                              │
               ▼                              ▼
┌──────────────────────────────┐  ┌──────────────────────────────┐
│ Redshift Holder Table        │  │ Fraud RDS Forward Table      │
│ t_fraud_identity_device_holder│  │ t_fraud_identity_device      │
│  - Grain: (device_id)        │  │ (Query: user's devices)      │
│  - LISTAGG all users (all-time,│  └──────────────────────────────┘
│    append-only, never shrinks)│
│  - user_count = true count;   │
│    user_ids capped at 2000    │
└──────────────┬───────────────┘
               │
               ▼
┌──────────────────────────────┐
│ Fraud RDS Holder Table       │
│ t_fraud_identity_device_holder│
│ (Query: shared device graph) │
└──────────────────────────────┘
```

#### Key Design Decisions
1. Persistent created_at in Redshift Base Table:
Why: The RDS write-back uses LOAD DATA ... REPLACE INTO, which deletes and re-inserts matching rows. Dynamically generating SYSDATE AS created_at would reset the timestamp on every subsequent update.
Design: created_at is set once on the initial INSERT and left untouched during UPDATEs, guaranteeing timestamp stability across repeated RDS deliveries.

2. last_seen_at in Redshift Holder Table (backfill windowing):
Why: Device is ~252M source rows (~48M after the ANDROID/IOS filter). A single-batch load causes Aurora replication lag and lock spikes.
Design: Retaining MAX(last_seen_at) in the Redshift holder table enables deterministic 90-day sliding-window catchups without hardcoding dates. (Column is omitted during RDS unload to keep the RDS schema aligned with the Fraud spec.) This is the device analog of Asset's first_seen_at.

3. All-time, append-only Holder (no is_del, no empty-device case):
The source t_patron_user_device carries no delete flag — a binding is never removed. The holder therefore aggregates all users of a device (INNER JOIN, no is_del filter) and never shrinks; there is no emptied-device case. Liveness is expressed by last_seen_at and narrowed at read time. (Contrast: the Asset holder filters is_del = 0 and emits emptied accounts as user_count=0, user_ids=[].)

4. user_count reports the true count; user_ids capped at 2000:
Why: Redshift LISTAGG/VARCHAR is bounded to 65535 bytes (~2340 ids), and per the spec a super-node (e.g. a public terminal shared by tens of thousands) must keep counting.
Design: user_count = COUNT(user_id) over the full partition (uncapped, exact), while user_ids is truncated at 2000 inside the LISTAGG. For super-nodes user_count > JSON_LENGTH(user_ids); the full member list remains available in the forward table, and the reconciliation check (forward COUNT = user_count) always passes.

#### Workflow & DAG Breakdown
Daily Incremental DAGs (Production)
1. `bi_pocket.t_fraud_identity_device` 
- Purpose: Ingests daily device-binding changes from the source DB into the Redshift base table.
- Logic: In-place upsert with GREATEST(last_seen_at) to keep the latest activity, src_update_time-guarded platform/status, and frozen created_at on initial insert.

2. `bi_pocket.t_fraud_identity_device_holder`
- Purpose: Computes and maintains the materialized holder mapping in Redshift.
- Logic: Sensor-waits on the base DAG (execution_delta=10min), aggregates all users per touched device via LISTAGG (user_ids cap 2000, user_count = true count).

3. `reverse_etl_fraud.t_fraud_identity_device`
- Purpose: Syncs daily forward binding deltas from Redshift to Fraud MySQL RDS.
- Logic: Sensor-waits on the base DAG (1h15m), UNLOADs 5MB S3 slices (window on updated_at), loads via REPLACE INTO with frozen created_at.

4. `reverse_etl_fraud.t_fraud_identity_device_holder`
- Purpose: Syncs daily reverse holder deltas from Redshift to Fraud MySQL RDS.
- Logic: Sensor-waits on the holder DAG (1h10m), UNLOADs from the materialized holder table, loads with LOAD_ESCAPED_BY="" to preserve valid JSON.

Historical Backfill DAGs
1. `bi_patron.t_fraud_identity_device_backfill`
- Purpose: Backfills the Redshift base from historical source (source create_time/update_time).

2. `bi_patron.t_fraud_identity_device_holder_backfill`
- Purpose: Populates the Redshift holder, windowed on MAX(last_seen_at) so each device lands in exactly one chunk.

3. `reverse_etl_fraud.t_fraud_identity_device_backfill`
- Purpose: Delivers the historical forward table, windowed on last_seen_at.

4. `reverse_etl_fraud.t_fraud_identity_device_holder_backfill`
- Purpose: Delivers the historical holder table, windowed on last_seen_at.

t_fraud_name_profile & t_fraud_name_token


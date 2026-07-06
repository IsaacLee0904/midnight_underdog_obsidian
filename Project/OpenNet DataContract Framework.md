
tags: #OpenNet  #data-engineering #data-contract #data-quality-check 

---
## Background

As our data platform grows, the number of ETL pipelines and downstream dependencies continues to increase, some pipelines even serve as application data sources. Today, when a DAG / table is modified — whether it's a transform logic change, a `distkey` adjustment, a column addition, or a data type alteration — there is no automated mechanism to notify downstream owners. Issues are typically only discovered when a consumer notices something unusual, or worse, when a business stakeholder raises the alarm first, damaging the credibility of the BI team.

This proposal introduces a lightweight **data contract** framework, inspired by [Airbnb's Metis platform](https://medium.com/airbnb-engineering/metis-building-airbnbs-next-generation-data-management-platform-d2c5219edf19 "https://medium.com/airbnb-engineering/metis-building-airbnbs-next-generation-data-management-platform-d2c5219edf19"), to address this gap. By ensuring every object (table & dashboard) has a registered owner in OpenMetadata, we can trigger an automated workflow to notify downstream owners whenever an upstream table is modified.

## Data Contract
### What is Data Contract ?
A data contract is a versioned, machine-readable agreement between a data producer and one or more data consumers that specifies :

- **Schema** : field names, data types, nullability, and acceptable value ranges
- **Semantics** : what the fields actually mean — the business definitions, not just the technical names
- **Service Level Agreements (SLAs)** : freshness guarantees, latency, availability expectations
- **Data Quality Rules** : completeness thresholds, referential integrity, format constraints
- **Ownership** : who is accountable for producing the data
-  **Stakeholder** : who is accountable for consuming it
- **Change Management** : how breaking changes are communicated, how versioning works, what constitutes a violation

### Case Study
This section includes how others company implement data contract and their framework.
#### PayPal
PayPal was one of the early large-scale adopters of data contracts, using them as part of their internal Data Mesh implementation. Their contract structure covers six sections: Fundamentals, Schema, Data Quality, SLA, Security & Stakeholders, and Custom Properties. They later open-sourced the framework they had been using internally, which has since evolved into what is now known as the [Open Data Contract Standard (ODCS)](https://github.com/bitol-io/open-data-contract-standard "https://github.com/bitol-io/open-data-contract-standard").

ref - [Github : data-contract-template](https://github.com/paypal/data-contract-template/ "https://github.com/paypal/data-contract-template/")
#### Miro
Miro adopted DataHub Cloud as their central metadata platform to address poor data product reliability. Their initial contracts lived inside Airflow code, making them too technical for business stakeholders to understand and act on. They migrated contract definitions to YAML files stored in their dbt repository, co-located with the analytics domain, with each data product having designated technical and business owners and clear SLAs. Through this approach, they reduced key pipeline downtime from 50% to approximately 1%.

ref - [Data Products Reliability: The Power of Metadata](https://miro.com/careers/life-at-miro/tech/data-products-reliability-the-power-of-metadata/ "https://miro.com/careers/life-at-miro/tech/data-products-reliability-the-power-of-metadata/")

#### GoCardless
GoCardless built their data contract system on top of their existing self-service infrastructure platform, treating data contracts as an API for data. Contracts are defined in code, specifying schema, privacy classifications, and deployment targets, and are automatically deployed once merged to Git. Within six months of scaling the initiative, they reached nearly 300 contracts in production across 28 teams, with 80% of their Pub/Sub topics managed through data contracts.

ref - [Improving Data Quality with Data Contracts](https://medium.com/gocardless-tech/improving-data-quality-with-data-contracts-238041e35698 "https://medium.com/gocardless-tech/improving-data-quality-with-data-contracts-238041e35698")
ref - [Data Contracts at GoCardless - 6 Months On](https://medium.com/gocardless-tech/data-contracts-at-gocardless-6-months-on-bbf24a37206e "https://medium.com/gocardless-tech/data-contracts-at-gocardless-6-months-on-bbf24a37206e")
ref - [Implementing Data Contracts at GoCardless](https://medium.com/gocardless-tech/implementing-data-contracts-at-gocardless-3b5c49074d13 "https://medium.com/gocardless-tech/implementing-data-contracts-at-gocardless-3b5c49074d13")
ref - [3 Things Our Software Engineers Love About Data Contracts](https://medium.com/gocardless-tech/3-things-our-software-engineers-love-about-data-contracts-3106e1f1602d "https://medium.com/gocardless-tech/3-things-our-software-engineers-love-about-data-contracts-3106e1f1602d")

### What We Can Learn ?
Across case studies above, two patterns stand out.
First, **ownership is the foundation of everything**. Before any automated notification or contract enforcement can work, every data asset must have an owner. Without this, a data contract is just documentation — not an agreement.
Second, **a centralized metadata platform is the enabler**. PayPal, Miro, and GoCardless all converged on a single source of truth for their data contracts — whether that is DataHub, an internal catalog, or an open standard. Having ownership, lineage, schema, and SLAs in one place is what makes automation and visibility possible.
These two principles directly inform our approach : we use OpenMetadata as the centralized metadata layer, and treat ownership registration as the first step. This proposal focuses on ownership and change notification as the starting point, with the flexibility to evolve toward schema enforcement and data quality contracts as the system matures.

## Implementation
### Proposal Architecture Overview
![[Datacontract.excalidraw|1500]]
### Prerequisites

Before the notification workflow can run, two conditions must be in place.

<mark style="background:#fff88f">Ownership Registration</mark>
Every table and dashboard in OpenMetadata must have a registered owner.

**Table**
`Airflow API -> DAG owner（name）-> mapping -> email -> Register to OM`

**Dashboards**
`Metabase API /api/dashboard/{id} → creator_id ↓ /api/user/{id} → email ↓ Register to OM`

<mark style="background:#fff88f">Lineage Registration</mark>
Every table in OpenMetadata must have lineage established — meaning OpenMetadata knows which DAG produces which table, and what is downstream.

For DA DAGs, lineage is parsed from SQL and pushed to OpenMetadata via a general function. However, not all DA DAGs currently adopt this function — this is a known gap that must be addressed before the notification workflow can run.
For DE DAGs, lineage is extracted by parsing the DAG files directly to identify the tables they maintain.

### Components
<mark style="background:#fff88f">data_keeper</mark>
[data_keeper](https://github.com/opennetltd/data_keeper "https://github.com/opennetltd/data_keeper") is an internal automation tool that manages and deploys configurations to OpenMetadata using version-controlled YAML files. It handles things like data quality tests, catalog services. In this project, data_keeper acts as the contract declaration layer to sync metadata like ownership, stakeholders, and consumers to OpenMetadata. Below is an example of a data contract definition :

![[Screenshot 2026-07-06 at 3.17.32 PM.png]]
```yaml
# example of yaml file
# config/data_contract/bi_warehouse/t_blog_rewrites_v2.yaml
contractedService:
  provider:
    name: t_blog_rewrites_v2
    team: Data Engineer Team
    contact: isaac.lee@opennet.tw
  consumer:
    - name: Encore Application
      team: Backend Team
      contact: danny.lin@football.com
```

reference : [Building a Data Contract: From Design to Deployment](https://cleandataarchitecture.substack.com/i/158709063/practical-guide-creating-a-data-contract-from-a-to-z)

<mark style="background:#fff88f">OpenMetadata</mark>
OpenMetadata is the central metadata layer. It keeps track of table and dashboard ownership, traces lineage across pipelines, and exposes a REST API so external systems (like GitHub Actions) can query downstream dependencies.

<mark style="background:#fff88f">GitHub Actions</mark>
GitHub Actions as the contract gateway — triggered on every PR merge to main. This follows common industry practice, where PR workflows are a natural place to enforce contract validation and send change notifications. It handles the full flow : parsing changed files, calling the OpenMetadata API, and sending Slack notifications.

<mark style="background:#fff88f">Metabase</mark>
Metabase is the source of dashboard ownership. During the initial setup, we use the [Metabase API](https://www.metabase.com/docs/latest/api#tag/apidashboard "https://www.metabase.com/docs/latest/api#tag/apidashboard") to fetch dashboard creators and register them as owners in OpenMetadata.

<mark style="background:#fff88f">Slack</mark>
Slack handles the notifications.

### Workflow

When a pull request is merged to main in any of the three monitored repositories — `warehouse_engineer`, `data_analysis`, or `dba-redshift-executor-prod` — a GitHub Actions workflow is triggered automatically.

**Step 1. Parse Changed Files**
The workflow analyzes the PR diff to identify which tables are affected by the change.
- For `warehouse_engineer` and `data_analysis`, the changed DAG files are identified and the affected table names are extracted by parsing the DAG content.
- For `dba-redshift-executor-prod`, the affected table names are extracted directly from the DDL statements in the changed SQL files.
    
In parallel, the existing Copilot PR summary is collected to provide a human-readable description of what changed and why.

**Step 2. Query OpenMetadata**
For each affected table, the workflow calls the [OpenMetadata Lineage API](https://docs.open-metadata.org/v1.12.x/api-reference/main-concepts/metadata-standard/apis "https://docs.open-metadata.org/v1.12.x/api-reference/main-concepts/metadata-standard/apis") to retrieve all downstream tables and dashboards, then resolves the owner of each downstream asset.

**Step 3. Send Slack Notification**
A Slack notification is sent to each identified downstream owners, including :
- The PR link
- A summary of what changed (according to Github Copilot)

## Roadmap

### Phase 0 : Foundation Setup

Before the notification workflow can operate, the following prerequisites must be done first.

**Ownership Registration**

All tables and dashboards in OpenMetadata must have a registered owner. For tables, ownership is derived from Airflow DAG owners via the Airflow API, mapped to email addresses, and registered in OpenMetadata. For dashboards, ownership is extracted directly from the Metabase API using the dashboard creator's email.

**Lineage Registration**

All pipelines must have table-level lineage established in OpenMetadata. For DA DAGs, all pipelines are required to adopt the standard general function to ensure consistent lineage coverage. For DE DAGs, lineage is extracted by parsing DAG files directly.

### Phase 1 : Automated Impact Notification

**Goal:** Trigger automated Slack notifications to downstream owners when a pipeline change is merged.

**Scope:**
- `warehouse_engineer` — data warehouse DAGs
- `data_analysis` — data analysis DAGs
- `dba-redshift-executor-prod` — DDL statements

**Pros:**
- Proactively notifies downstream owners before issues propagate
- Leverages existing infrastructure with no new tooling required
    
**Cons:**
- Notification quality depends on the completeness of ownership and lineage coverage established in Phase 0
    
### Phase 2 : Extend to Application Data Sources

**Goal:** Extend coverage to backend application data sources that feed into the data platform.

**Pros:**
- Broader coverage across the full data supply chain
### Phase 3 : Column-level & Data Quality Enforcement (Nice to have)

**Goal:** Evolve from table-level notification to column-level change detection and data quality contract enforcement. And those task could be define by [data_keeper](https://github.com/opennetltd/data_keeper "https://github.com/opennetltd/data_keeper").

**Pros:**
- More precise impact assessment — only notify owners whose downstream assets use the affected column
- Moves toward a fully enforceable data contract

**Cons:**
- Requires column-level lineage to be established in OpenMetadata
- Significantly higher implementation complexity
## Open Question

- Slack notification target : DM to downstream owners vs. dedicated alert channel (which channel)?
- Notification content : PR link, affected table, downstream impact scope (Copilot Summary)?
- Ownership coverage : How to using OpenMetadata to coverage all the owner and how do we handle tables without a registered owner (edge case)?
- Consumer / Stakeholder notification : How should consumers who are not part of the OM lineage graph (e.g. backend engineers, PMs) be notified? Three options under consideration :
	- **Option A — OM Follower** : Create OM accounts for consumers and let them self-register as Followers. GitHub Actions only needs to query OM. Downside: requires provisioning OM accounts for non-DE/DA users, which is likely impractical.
	- **Option B — data_keeper YAML** : Store consumer lists in `config/data_contract/` YAML files. GitHub Actions reads both OM lineage (downstream owners) and data_keeper YAML (consumers) and merges the two. Downside: GitHub Actions has two data sources, increasing complexity.
	- **Option C — OM Custom Property** : Store consumer lists in data_keeper YAML, but have data_keeper sync them to a custom property on each OM table entity. GitHub Actions queries only OM for both downstream owners and consumers. Keeps consumer data version-controlled while keeping GitHub Actions simple.





























### Reference
* https://soda.io/blog/guide-to-data-contracts
* https://medium.com/airbnb-engineering/metis-building-airbnbs-next-generation-data-management-platform-d2c5219edf19

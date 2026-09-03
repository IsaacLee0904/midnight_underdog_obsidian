
tags: #OpenNet  #data-engineering #data-contract #data-quality-check 

---
# Overview

## Background

As our data platform grows, an increasing number of non-BI teams are requesting access to data from our platform not just asking we query for them but sometime want to access directly or write back to application database. Currently, there is no standardized process for handling these requests — each case is handled ad hoc, leading to repeated discussions around the same questions : what table include the information they need, which data can be shared, under what conditions, and who is responsible for it.

This lack of a clear process is not only inefficient but also touches on broader data governance concerns, including data sensitivity, ownership accountability, and the terms of data usage agreements between producers and consumers.

These challenges point to the need for a data contract — a formal agreement between data producers and consumers that defines what data is available, how it can be used, and what both sides are responsible for.

## Objective

This document covers Phase 1 of the Data Contract project. This phase focuses on establishing the process and guidelines that the technical system will eventually enforce — defining how data access requests should be handled, what information each data asset should carry, and what the boundaries of BI data consumption look like.

## Scope

This phase covers the following :

- Define the data request process : how non-BI stakeholders apply for data access, the review flow involving PJM + DA/DE, and how agreements are documented

- Define the data catalog standard : what information each table in OpenMetadata should have (owner, column descriptions, SLA, sensitivity flags)
    
- Define CAN/CAN'Ts for consuming BI data
    
- Design the data contract and data catalog YAML structure in [data_keeper](https://github.com/opennetltd/data_keeper "https://github.com/opennetltd/data_keeper") : defining what fields are required for declaring table providers, consumers, column descriptions, and sensitivity tags. Actual population and sync to OpenMetadata will be carried out in Phase 2.
    

# Data Governance Guideline

This section defines the standards and agreements that govern how data is managed, described, and consumed within our platform. It covers two areas :

1. **Data Catalog** : Defines what metadata each data asset should carry.
    
2. **Data Contract** : Defines the terms of agreement between data producers and consumers.
    

## Data Catalog

A data catalog is the foundation of any data governance framework. Before we can enforce data contracts or manage access requests, every data asset needs to include information such as who owns it, what it contains, how fresh it is, and whether it carries any sensitivity constraints.

In this project, we use **OpenMetadata** as the central catalog. Each table is expected to carry a minimum set of metadata fields, defined in this section, which serve as the basis for ownership registration, lineage tracking, and downstream notification in later phases.

### Catalog Fields Standard

The following fields are defined based on industry practices and tailored to our current needs. Fields marked as **Tier 1** are required to be completed in Phase 2 — they are the minimum foundation for the data contract framework to function and to address more immediate data governance needs such as the data access request workflow.

![[Screenshot 2026-09-03 at 10.32.32 AM.png]]

### How to Populate Catalog Fields

Catalog metadata is managed through version-controlled YAML files in [data_keeper](https://github.com/opennetltd/data_keeper "https://github.com/opennetltd/data_keeper") under `config/data_catalog/`. This follows the **docs-as-code** principle — rather than manually editing fields in the OpenMetadata UI, metadata is declared alongside the data pipeline code, reviewed through pull requests, and automatically synced to OpenMetadata. This ensures catalog information is version-controlled, auditable, and consistent across tables.

#### Registration Strategy

The registration approach differs data layer by :

1. **RDS Source Tables**
Column descriptions are sourced directly from DDL comments maintained on the source RDS databases. These are automatically synced to OpenMetadata via the existing connector (currently about 80% coverage).

2. Warehouse Tables
Tables updated by DAGs in [warehouse_engineer](https://github.com/opennetltd/warehouse_engineer "https://github.com/opennetltd/warehouse_engineer") are registered through YAML files in [data_keeper](https://github.com/opennetltd/data_keeper "https://github.com/opennetltd/data_keeper") under `config/data_catalog/`. For the initial backfill, a script will cross-reference the source RDS DDL comments for columns that share the same name. For any new table created after, the catalog YAML must be included as part of the deployment to production.

```markdown

```


3. Data mart Tables
Tables updated by DAGs in [data_analysis](https://github.com/opennetltd/data_analysis "https://github.com/opennetltd/data_analysis") are registered through YAML files in [data_keeper](https://github.com/opennetltd/data_keeper "https://github.com/opennetltd/data_keeper") under `config/data_catalog/`. Since DA tables often involve complex SQL transformations, an LLM-assisted approach is expected to be used to generate the initial description draft. For any new table created after, the catalog YAML must be included as part of the deployment to production.

```markdown
DA pipeline SQL + warehouse schema 
        ↓ LLM generates description draft
Generated YAML draft stored in data_keeper 
        ↓ 
Register on OpenMetadata
```

#### YAML Template

Each field in the template maps to a specific location in OpenMetadata. The annotated screenshot below shows where each field appears in the OM UI.
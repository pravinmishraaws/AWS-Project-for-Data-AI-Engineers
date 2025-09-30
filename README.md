# AWS-Project-for-Data-AI-Engineers

This is a **project-based** course where you build, operate, and troubleshoot production-style AWS DATA/AI systems—exactly the kind you’ll encounter on the job. Instead of slideware and toy demos, you’ll ship five progressively complex projects: a lakehouse on S3 with Apache Iceberg, governance-as-code with Lake Formation, a Redshift Serverless warehouse with external tables and SCD2, an “on-call day” where you run and repair a batch data pipeline (backfills, late data, schema change), and a real-time streaming/CDC pipeline using Flink.

You’ll work from a shared, opinionated reference architecture so beginners aren’t blocked by AWS plumbing, while experienced engineers can go deeper with optional “pro” tasks. Every project includes **design choices, cost controls, data quality checks, security defaults (KMS, private networking, LF masking),** and **runbooks**—so you learn not only how to build pipelines, but also how to **operate** them reliably.

By the end, you’ll have hands-on experience across the modern AWS data stack (S3 + Iceberg, Glue/Athena, Lake Formation, Redshift Serverless, Kinesis/MSK, Flink, DMS, DynamoDB, QuickSight), plus AWS CLI commands together. If you’re aiming for an AWS Data/AI Engineering role, upskilling in your current job, or curating a portfolio that stands out, this course is designed to be both **practical** and **career-relevant**.

# Learning Outcomes

By the end of the course, you will be able to:

* **Design and implement a lakehouse on AWS** with S3 + Apache Iceberg, including partitioning, schema evolution, compaction, and data quality gates.
* **Apply governance as code** using Lake Formation (tag-based access, column masking, row filters, cross-account sharing) and validate policies in CI.
* **Model and load analytical warehouses on Redshift Serverless,** using external tables over Iceberg, SCD2 dimensions with MERGE, and performance/cost tuning.
* **Orchestrate, monitor, and support pipelines,** including retries, idempotency, backfills, late-data handling, incident response, and postmortems.
* **Build real-time streaming/CDC pipelines** (DMS → Kinesis/MSK → Flink → Iceberg/DynamoDB), with watermarks, exactly-once semantics, and savepoint-based rollbacks.

# Requirements / Prerequisites

* **Basic familiarity** with Python and SQL (comfortable reading/writing simple scripts and queries).
* **An AWS account** with permissions to create resources (S3, IAM, Glue, Lake Formation, Redshift, Kinesis/MSK, DMS, etc.).
* **Local tools:** Git, a code editor (VS Code), and a SQL client (we’ll suggest free options).

**Beginner-friendly?** Yes. I provide a ready-to-deploy AWS CLI commands. Each project has a **Core path** (must-have skills) and an optional **Pro path** (deeper operations/optimization) so experienced engineers can stretch without overwhelming newcomers.

# Target Audience

* **Aspiring DATA/AI engineers** who want practical AWS projects for their portfolio.
* **Software/analytics engineers** moving into cloud DATA/AI engineering.
* **Working data engineers** upskilling on lakehouse, governance, Redshift, streaming/CDC.
* **Career switchers** seeking hands-on, resume-ready AWS data experience.

---

## Project 1 — Lakehouse on AWS: S3 + Apache Iceberg

**Short Description**
Stand up a production-style lakehouse. Land raw data into S3, transform to Iceberg bronze/silver, implement partitioning and schema evolution, and gate publishes with data quality checks.

**Learning Objectives**

* Create Iceberg tables on S3 and manage partitioning, time travel, and schema changes.
* Build an ingestion/transform job (Glue/EMR) with **Great Expectations** DQ gates.
* Apply cost controls (S3 lifecycle, Intelligent-Tiering) and basic observability (CloudWatch).

**AWS Services Covered**
S3, Glue (Jobs/Studio/Catalog), Athena (Iceberg engine), IAM, KMS, CloudWatch.

**Estimated Time**
6–8 hours (Core), +3 hours (Pro: compaction, table evolution, lineage hooks).

---

## Project 2 — Governance as Code with Lake Formation

**Short Description**
Enforce fine-grained access to lakehouse data using Lake Formation. Implement tag-based policies, column masking, row filters, and cross-account data sharing—codified and tested in CI.

**Learning Objectives**

* Define LF tags/policies as code and apply them to databases, tables, and columns.
* Implement column masking and row-level filters for different personas.
* Share curated datasets to a consumer account using resource links; audit access.

**AWS Services Covered**
Lake Formation, Glue Catalog, IAM, KMS, CloudTrail + Athena (audit), S3.

**Estimated Time**
4–6 hours (Core), +2 hours (Pro: policy drift detection tests, access review scripts).

---

## Project 3 — Data Warehouse on Redshift Serverless (External + SCD2)

**Short Description**
Model a star schema and build curated marts on Redshift Serverless. Use **external tables over Iceberg** for the lakehouse interface, load facts/dimensions, implement **SCD2** with MERGE, and tune performance/cost.

**Learning Objectives**

* Configure Redshift Serverless namespaces/workgroups and workload isolation.
* Create external schemas over Iceberg; build SCD2 dims and incremental fact loads.
* Use materialized views, query monitoring rules, and basic performance benchmarks.

**AWS Services Covered**
Redshift Serverless, Spectrum (external schemas), S3 + Iceberg, Glue Catalog, IAM, CloudWatch.

**Estimated Time**
6–8 hours (Core), +3 hours (Pro: perf benchmarks, concurrency controls, MV refresh strategy).

---

## Project 4 — A Day in the Life of a Data Engineer (Batch Ops Simulation)

**Short Description**
Operate an end-to-end batch pipeline. Orchestrate ingest → DQ → publish, handle a broken run (schema change, late data), **backfill** the last N days, and write the incident postmortem.

**Learning Objectives**

* Build orchestration with **Step Functions or Airflow** (retries, idempotency, DLQs).
* Implement SLAs, job bookmarks, and parameterized backfills.
* Triage incidents, repair partitions, and control Athena cost via workgroups/limits.

**AWS Services Covered**
Step Functions **or** MWAA (Airflow), Glue Jobs, Athena, S3, CloudWatch, SQS (DLQ), IAM.

**Estimated Time**
5–7 hours (Core), +2 hours (Pro: late-data policy and automation, partition repair tooling).

---

## Project 5 — Real-Time CDC Streaming with Flink (Capstone)

**Short Description**
Build a near-real-time pipeline from **DMS (CDC)** → **Kinesis/MSK** → **Flink**. Apply event-time windows, watermarks, and exactly-once sinks to update **Iceberg upsert tables** and a DynamoDB serving store; publish a gold view to QuickSight. Includes savepoint-based rollback.

**Learning Objectives**

* Set up CDC from a relational source with AWS DMS and land change events to a stream.
* Implement Flink with watermarks/late data handling and exactly-once delivery semantics.
* Write dual sinks: upsert to Iceberg (batch/BI) and update a DynamoDB lookup table; visualize.
* Operate streaming jobs: checkpoints, savepoints, blue/green deploys, and alarms.

**AWS Services Covered**
AWS DMS, Kinesis **or** Amazon MSK, Managed Service for Apache Flink, Glue Schema Registry, S3 + Iceberg, DynamoDB, QuickSight, CloudWatch.

**Estimated Time**
7–10 hours (Core), +3 hours (Pro: stateful redeploy with savepoints, autoscaling, DLQ strategy).

---

### Notes on Progression

* **P1 → P2:** Secure what you built.
* **P2 → P3:** Serve governed data to analytics with Redshift (external + curated).
* **P3 → P4:** Operate and fix real failures in batch pipelines.
* **P4 → P5:** Extend to real-time and CDC, integrating lakehouse + serving + BI.


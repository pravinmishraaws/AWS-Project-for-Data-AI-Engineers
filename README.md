# AWS-Project-for-Data-AI-Engineers

This is a **project-based** course where you build, operate, and troubleshoot production-style AWS DATA/AI systems—exactly the kind you’ll encounter on the job. Instead of slideware and toy demos, you’ll ship five progressively complex projects: a lakehouse on S3 with Apache Iceberg, governance-as-code with Lake Formation, a Redshift Serverless warehouse with external tables and SCD2, an “on-call day” where you run and repair a batch data pipeline (backfills, late data, schema change), and a real-time streaming/CDC pipeline using Flink.

You’ll work from a shared, opinionated reference architecture so beginners aren’t blocked by AWS plumbing, while experienced engineers can go deeper with optional “pro” tasks. Every project includes **design choices, cost controls, data quality checks, security defaults (KMS, private networking, LF masking),** and **runbooks**—so you learn not only how to build pipelines, but also how to **operate** them reliably.

By the end, you’ll have hands-on experience across the modern AWS data stack (S3 + Iceberg, Glue/Athena, Lake Formation, Redshift Serverless, Kinesis/MSK, Flink, DMS, DynamoDB, QuickSight), plus AWS CLI commands together. If you’re aiming for an AWS Data/AI Engineering role, upskilling in your current job, or curating a portfolio that stands out, this course is designed to be both **practical** and **career-relevant**.

## Project Architecture

<img width="1432" height="806" alt="Screenshot 2025-09-30 at 14 38 18" src="https://github.com/user-attachments/assets/d752fe27-4c8c-4e84-ab79-1afa815891b9" />

## AWS Technical Architecture

<img width="1434" height="800" alt="Screenshot 2025-10-08 at 15 06 52" src="https://github.com/user-attachments/assets/0338e035-82c6-4139-88a1-e9616b04123a" />

---

## Project 1 — Lakehouse on AWS: S3 + Apache Iceberg

**Short Description**
Stand up a production-style lakehouse. Land raw data into S3, transform to Iceberg bronze/silver, implement partitioning and schema evolution, and gate publishes with data quality checks.

---

## Project 2 — DATA Governance with Lake Formation

**Short Description**
Enforce fine-grained access to lakehouse data using Lake Formation. Implement tag-based policies, column masking, row filters, and cross-account data sharing—codified and tested in CI.

---

## Project 3 — Data Warehouse on Redshift Serverless (External + SCD2)

**Short Description**
Model a star schema and build curated marts on Redshift Serverless. Use **external tables over Iceberg** for the lakehouse interface, load facts/dimensions, implement **SCD2** with MERGE, and tune performance/cost.

---

## Project 4 — A Day in the Life of a Data Engineer (Batch Ops Simulation)

**Short Description**
Operate an end-to-end batch pipeline. Orchestrate ingest → DQ → publish, handle a broken run (schema change, late data), **backfill** the last N days, and write the incident postmortem.

---

## Project 5 — AI/ML Readiness & Serving

**Short Description**
Curate ML-friendly/feature-like tables, ensure reproducible training sets using Iceberg snapshots/time travel, and (optional) integrate SageMaker/Athena for model workflows.


---

### Notes on Progression

* **P1 → P2:** Secure what you built.
* **P2 → P3:** Serve governed data to analytics with Redshift (external + curated).
* **P3 → P4:** Operate and fix real failures in batch pipelines.
* **P4 → P5:** Extend to real-time and CDC, integrating lakehouse + serving + BI.


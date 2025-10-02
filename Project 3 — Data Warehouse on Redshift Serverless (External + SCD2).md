## Data Warehouse on Redshift Serverless (External + SCD2 )

You’ll stand up a Redshift Serverless warehouse that: 

<img width="979" height="555" alt="Screenshot 2025-10-02 at 16 50 18" src="https://github.com/user-attachments/assets/d688e424-7b4c-4745-ba2e-f200e358b527" />


**1. Redshift Serverless Workgroup and Namespace** 
- A running Serverless endpoint you can connect to (psql, Query Editor v2). 

**2. External schema (Redshift Spectrum) pointing to Glue / Iceberg** 
- You can SELECT the Iceberg tables in place from Redshift (no copy). 

**3. Implements SCD Type 2 with a MERGE to track attribute history.** 
- A dimension table that keeps full attribute history over time. 

**4. Exposes a materialized view for BI.** 
- Low-latency, predictable query surfaces for dashboards. 

**5. Shows concurrency/cost controls via WLM & Query Monitoring Rules (QMR).** 
- Predictable performance and spend; noisy neighbors get constrained.

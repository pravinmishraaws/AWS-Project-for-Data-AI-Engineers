## DATA Governance
Enforce fine-grained access to lakehouse data. Implement tag-based policies, column masking, row filters, and cross-account data sharing—codified and tested in CI.

## What you’ll build

**Real-world org structures**

- Data Admin = platform governance
- Data Engineer = ingestion + pipelines
- Data Analyst = BI/Reporting (restricted view)
- Data Scientist = ML training (full access)


**Separation of duties**
- Admin governs metadata + grants.
- Engineers build pipelines but don’t own governance.
- Analysts consume governed views, no PII.
- Scientists need broad but auditable access.

> Matching the enterprise least-privilege and compliance.

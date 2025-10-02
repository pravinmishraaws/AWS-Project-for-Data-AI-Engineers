## DATA Governance
Enforce fine-grained access to lakehouse data. Implement tag-based policies, column masking, row filters, and cross-account data sharing—codified and tested in CI.

<img width="976" height="550" alt="Screenshot 2025-10-02 at 16 50 36" src="https://github.com/user-attachments/assets/757dce02-16ae-448d-a199-0445069d9742" />


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

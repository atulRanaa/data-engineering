# 8. Data Governance & Security

Ingesting petabytes of data is a technical problem. Ensuring that data doesn't violate a European privacy regulation resulting in a 20-million-euro fine is a business survival problem. 

!!! info "Historical Context: GDPR and the Shift to Privacy"
    In May 2018, the European Union enforced the **General Data Protection Regulation (GDPR)**, immediately followed by California's CCPA. Before 2018, Data Engineers rarely cared about Data Governance; they just dumped all marketing and database tracking logs into massive, unsecured Hadoop clusters. GDPR introduced the "Right to be Forgotten." Suddenly, companies had 30 days to systematically delete every single trace of "John Doe" across millions of unstructured JSON files floating in S3—a mathematically impossible task unless governance was built directly into the data pipelines.

---

## 8.1 Data Catalogs and Lineage

Before you can secure data, you must locate it. A **Data Catalog** (like Amundsen, DataHub, or Alation) is the "Google Search" for a company's internal databases.

### Lineage
If an executive looks at a Tableau dashboard and asks *"Where did this revenue number come from?"*, the Data Engineer must provide the **Data Lineage**.
Lineage is a massive graph tracking a specific column from its origin (the raw `stripe_webhook` Kafka topic) through 6 different `dbt` transformation SQL models, all the way to the final `dim_revenue` warehouse table. Lineage prevents Data Scientists from training ML models on biased or stale data.

---

## 8.2 Access Control and Role-Based Filtering

You cannot grant everyone `SELECT *`. Data Engineers apply security at three layers:

1. **Role-Based Access Control (RBAC):** Users are assigned to groups (e.g., `marketing_analyst`, `finance_lead`).
2. **Column-Level Security:** Instead of blocking access to the whole `users` table, the database dynamically replaces the `Social_Security_Number` column with `NULL` or `***-**-****` for users lacking the PII clearance role.
3. **Row-Level Security (RLS):** A Regional Sales Manager querying `SELECT * FROM sales` will transparently only see rows where `region = 'Europe'`, enforced natively by the Warehouse engine.

??? example "Snowflake Code: Column-level Masking Policies"
    Modern Cloud Data Warehouses use dynamic policies evaluated at query-time based on the user's role. No separate data copies are needed.
    ```sql
    -- 1. Create a dynamic masking policy
    CREATE OR REPLACE MASKING POLICY email_mask AS (val string) RETURNS string ->
        CASE
            WHEN current_role() IN ('HR_ADMIN', 'SYSADMIN') THEN val
            ELSE '***@***.com'
        END;

    -- 2. Apply it to the physical table
    ALTER TABLE employees 
    MODIFY COLUMN email 
    SET MASKING POLICY email_mask;

    -- If a standard Analyst runs `SELECT email FROM employees`, 
    -- the database engine intercepts the query and physically 
    -- replaces the string with exactly '***@***.com'.
    ```

---

## 8.3 Encryption & Anonymization

**Data Encryption:**
Data must be encrypted **In Transit** (TLS/HTTPS over the internet) and **At Rest** (AES-256 for physical hard drives/S3). 

**Data Anonymization (K-Anonymity):**
A dataset is considered $k$-anonymous if the information for any person cannot be distinguished from at least $k-1$ other individuals. If you publish a healthcare dataset, stripping out the "Name" isn't enough. If the dataset contains `[Zip Code, Gender, Date of Birth]`, that combination is often unique enough to mathematically re-identify the exact human. Governance teams must deliberately blur `Date of Birth` into `Decade of Birth` to protect privacy.

**Differential Privacy:**
The state-of-the-art approach used by companies like Apple and Google. Instead of just masking data, differential privacy mathematical injects calibrated "noise" into the dataset. This guarantees that querying the database will yield statistically accurate macro-trends (e.g., "average salary by city") without allowing the attacker to reverse-engineer the exact salary of any specific individual. 

### Hands-On Lab: PII Hashing & Masking
To understand how compliance translates to code, we must construct a reproducible isolation layer.
1. **Goal:** Mask sensitive PII data before analysts can query it.
2. **Implementation:** Use SQL hashing functions (`SHA256`) explicitly seeded with a salt to anonymize IDs tracking user behavior.

??? example "PostgreSQL Code: Data Masking with pgcrypto"
    ```sql
    -- Enable the crypto extension
    CREATE EXTENSION IF NOT EXISTS pgcrypto;

    -- Create an anonymized view over the raw user table for the Analytics team
    CREATE VIEW v_analytics_users AS
    SELECT 
        -- Hash the raw email address so it's impossible to reverse engineer,
        -- but the same user will generate the same hash, allowing JOINs.
        encode(digest(email || 'SALTY_STRING', 'sha256'), 'hex') AS user_hash_id,
        date_trunc('year', birth_date) AS birth_year, -- k-anonymity: mask exact birthday
        country
    FROM raw_users;
    ```

---

!!! abstract "References & Foundational Reading"
    - **General Data Protection Regulation (EU GDPR):** The driving legal force behind modern Data Governance architectures.
    - **Amundsen:** *How Lyft Discovers and Understands Data* (Lyft Engineering Blog). The inception of modern automated Data Catalogs.
    - **Data Mesh** (Zhamak Dehghani) - Central to Data Mesh is the concept of *Federated Computational Governance*, where security rules are enforced globally but pipelines are managed locally.

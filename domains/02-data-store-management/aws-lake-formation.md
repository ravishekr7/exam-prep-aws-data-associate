# AWS Lake Formation and Glue Data Catalog

Centralized governance, security, and cataloging for your data lake.

---

## Glue Data Catalog

Central metadata repository for all data assets.

### Components
- **Databases:** Logical grouping of tables
- **Tables:** Schema metadata pointing to data in S3, JDBC, etc.
- **Crawlers:** Auto-discover schema and populate catalog
- **Connections:** Credentials and network info for data sources

### Catalog as Hive Metastore
- Compatible with Apache Hive metastore
- Used by: Athena, Redshift Spectrum, EMR, Glue ETL
- Single source of truth for schema across services

### Partition Sync
- Crawlers can discover new partitions automatically
- `MSCK REPAIR TABLE` in Athena to sync partitions manually
- EventBridge + Lambda can add partitions on S3 object creation

---

## AWS Lake Formation

Specialized security and governance layer built on top of Glue Data Catalog.

### The Two-Layer Permission Model (Exam Critical)

This is the most tested Lake Formation concept. Lake Formation adds a second permission layer on top of IAM. Both must allow access — failing either one results in AccessDenied.

Evaluation order for a query (e.g., Athena querying a Lake Formation governed table):
1. IAM policy — does this principal have permission to call the AWS API (e.g., `athena:StartQueryExecution`, `glue:GetTable`)?
2. Lake Formation permission — does this principal have SELECT on this specific database/table/columns?

If EITHER check fails → AccessDenied.

**Key exam trap:** A user with `AdministratorAccess` IAM policy still gets AccessDenied on an Athena query hitting a Lake Formation governed table if Lake Formation has not explicitly granted them SELECT on that table. S3 admin access is irrelevant — Lake Formation intercepts data access before S3 is reached.

For S3 locations registered with Lake Formation:
- Lake Formation permissions become the primary access control for Glue/Athena/EMR queries
- S3 bucket policies and ACLs are effectively bypassed for governed data access
- IAM still controls which AWS APIs can be called, but NOT which data rows/columns are returned

Diagram:
```
User → Athena API call
         │
         ├─ IAM check: can this role call athena:StartQueryExecution? → DENY = stop here
         │
         └─ Lake Formation check: does this role have SELECT on this table?
                  │
                  ├─ DENY = AccessDenied (even with S3 full access in IAM)
                  │
                  └─ ALLOW → query proceeds, row/column filters applied automatically
```

Exam patterns:
- "User has S3 full access but gets AccessDenied querying Athena on a Lake Formation table" → Lake Formation SELECT permission is missing
- "Grant column-level access without modifying IAM or S3 bucket policies" → Lake Formation column security
- "Centrally manage data access across Glue, Athena, EMR, and Redshift Spectrum" → Lake Formation (single control plane for all these services)

### Data Lake Administrator Role

**Who can grant Lake Formation permissions?** By default, only two types of principals can grant LF permissions on a resource:
1. The **data lake administrator** (explicitly designated in LF settings)
2. The **resource creator** — but only on resources they created, and only if they have been granted `GRANT OPTION`

**Key exam trap:** An IAM administrator is NOT automatically a Lake Formation data lake administrator. An IAM admin who hasn't been designated as a Lake Formation admin cannot grant LF permissions to other users, even with full IAM privileges. This is a separate, explicit designation made in the Lake Formation console or CLI.

```bash
# Designate a principal as data lake admin
aws lakeformation put-data-lake-settings \
  --data-lake-settings '{
    "DataLakeAdmins": [
      {"DataLakePrincipalIdentifier": "arn:aws:iam::123456789:role/DataLakeAdminRole"}
    ]
  }'
```

Implications:
- Data lake admins can grant permissions on any database/table/column, regardless of who created it
- Without a designated data lake admin, only the resource creator can manage grants — this breaks down when teams rotate
- `GRANT OPTION` allows a non-admin grantee to re-grant their permissions to others (use carefully)

Exam patterns:
- "IAM admin cannot grant Lake Formation permissions to the analyst team" → Principal has not been designated as a data lake administrator
- "Table creator left the company — no one can manage permissions on their table" → No data lake admin was designated; fix by adding one in LF settings
- "Allow team lead to delegate access to their own team's tables" → Grant permissions with `GRANT OPTION`

### Fine-Grained Access Control
| Level | Description | Example |
|-------|-------------|---------|
| **Database** | Access to entire database | Analyst can see `analytics_db` |
| **Table** | Access to specific tables | Analyst can see `orders` table |
| **Column** | Hide specific columns | Hide `ssn` column from analysts |
| **Row** | Filter rows by condition | Show only rows where `country='US'` |
| **Cell** | Combination of row + column | Most granular |

### LF-Tags (Tag-Based Access Control) — Deep Dive

Resource-based grants (granting SELECT on a specific table) do not scale. When you have 500 tables across 20 departments, managing individual grants becomes unmanageable.

LF-Tags solves this with attribute-based access control:

How it works:
1. Define tag keys and values: e.g., key=`department`, values=`[finance, engineering, hr]`
2. Assign tags to databases, tables, or columns
3. Grant permissions based on tags: "Finance Analysts group can SELECT where `department=finance`"
4. Any NEW table tagged `department=finance` is automatically accessible to Finance Analysts — no new grants needed

Tag hierarchy:
- Tags assigned to a database are inherited by all tables in that database
- Tags assigned to a table are inherited by all columns in that table
- You can override inherited tags at a lower level (e.g., a specific column in a finance table can be tagged `sensitivity=restricted` even if the table is tagged `sensitivity=internal`)

Example:

```bash
# Tag the database
aws lakeformation add-lf-tags-to-resource \
  --resource '{"Database": {"Name": "finance_db"}}' \
  --lf-tags '[{"TagKey": "department", "TagValue": "finance"}]'

# Grant Finance Analysts team access to all resources tagged department=finance
aws lakeformation grant-permissions \
  --principal '{"DataLakePrincipalIdentifier": "arn:aws:iam::123456789:role/FinanceAnalysts"}' \
  --permissions '["SELECT"]' \
  --resource '{"LFTagPolicy": {"ResourceType": "TABLE", "Expression": [{"TagKey": "department", "TagValues": ["finance"]}]}}'
# Now Finance Analysts automatically access ALL tables in finance_db — including new ones added later
```

Exam patterns:
- "Company has 500 tables across 20 departments — how to manage access at scale with least operational overhead" → LF-Tags
- "New tables added weekly — access control should apply automatically without manual grants" → LF-Tags (tag the database, new tables inherit)
- "Different sensitivity levels (public, internal, confidential) across tables" → LF-Tags with a `sensitivity` tag key

### Row-Level and Column-Level Security — Implementation

**Column-Level Security:**

Exclude specific columns from a principal's view. Applied at the table level.

In the Lake Formation console or CLI: grant SELECT on the table but exclude specific columns.

Example: Analysts can see the `customers` table except the `ssn` and `credit_card_number` columns.

```bash
aws lakeformation grant-permissions \
  --principal '{"DataLakePrincipalIdentifier": "arn:aws:iam::123456789:role/Analyst"}' \
  --permissions '["SELECT"]' \
  --resource '{
    "TableWithColumns": {
      "DatabaseName": "customers_db",
      "Name": "customers",
      "ColumnWildcard": {
        "ExcludedColumnNames": ["ssn", "credit_card_number"]
      }
    }
  }'
```

**Row-Level Security (Data Filters):**

Create a named data filter with a row filter expression (a SQL WHERE clause). Assign it to a principal. That principal only sees rows matching the filter.

```bash
# Create a data filter: only rows where region = 'APAC'
aws lakeformation create-data-cells-filter \
  --table-data '{"TableCatalogId": "123456789", "DatabaseName": "sales_db", "TableName": "transactions"}' \
  --name "apac_only" \
  --row-filter '{"FilterExpression": "region = '\''APAC'\''"}' \
  --column-wildcard '{}'

# Grant this filter to the APAC team
aws lakeformation grant-permissions \
  --principal '{"DataLakePrincipalIdentifier": "arn:aws:iam::123456789:role/APACAnalysts"}' \
  --permissions '["SELECT"]' \
  --resource '{"DataCellsFilter": {"TableCatalogId": "123456789", "DatabaseName": "sales_db", "TableName": "transactions", "Name": "apac_only"}}'
```

The filter is enforced transparently — the APAC team's Athena queries return only APAC rows without needing to add a WHERE clause.

**Column Masking:**

Restrict a principal to a specific subset of columns by creating a data cells filter that enumerates only the allowed columns. The excluded columns are not returned — they do not appear as NULL; they are entirely absent from the result set.

Lake Formation does not natively replace column values with a hash or obfuscated string. If you need true value-level masking (e.g., show a column but display `****` instead of the real value), that requires a Glue or Athena view on top of the governed table.

```bash
# Allow analysts to see all columns EXCEPT email — omit email from ColumnNames
aws lakeformation create-data-cells-filter \
  --table-data '{"TableCatalogId": "123456789", "DatabaseName": "customers_db", "TableName": "customers"}' \
  --name "hide_email" \
  --row-filter '{"AllRowsWildcard": {}}' \
  --column-names '["customer_id", "name", "region", "signup_date"]'

# Grant this filter to analysts
aws lakeformation grant-permissions \
  --principal '{"DataLakePrincipalIdentifier": "arn:aws:iam::123456789:role/Analyst"}' \
  --permissions '["SELECT"]' \
  --resource '{"DataCellsFilter": {"TableCatalogId": "123456789", "DatabaseName": "customers_db", "TableName": "customers", "Name": "hide_email"}}'
```

Exam patterns:
- "Analysts should only see rows where region = 'APAC' — enforce this centrally without modifying their queries" → Lake Formation row filter (data cells filter)
- "Restrict which columns an analyst can see without modifying the table or creating a view" → Column-level security or data cells filter with `ColumnNames`
- "Different users should see different subsets of the same table based on their department" → Separate row filters per group assigned in Lake Formation

### Cross-Account Data Sharing

Lake Formation enables sharing governed Glue Data Catalog resources (databases and tables) across AWS accounts using AWS RAM (Resource Access Manager).

Step-by-step pattern (exam-testable):

1. **Account A (data producer):** Register S3 location with Lake Formation
2. **Account A:** Create Glue database and tables pointing to that S3 location
3. **Account A:** Grant Lake Formation permissions to Account A's own Glue jobs/Athena
4. **Account A:** Grant Lake Formation DESCRIBE + SELECT to **Account B's account ID** as a principal, then create a RAM Resource Share for the Glue database and add Account B as a principal. Both steps are required — RAM shares the catalog metadata; the LF grant controls data access.
5. **Account B:** Accept the RAM Resource Share in the Lake Formation console
6. **Account B:** Create a resource link (a local reference to Account A's shared database)
7. **Account B:** Grant Lake Formation SELECT on the resource link to Account B's Analyst role
8. **Account B:** Analysts query via Athena — row/column filters defined in Account A are enforced automatically

Key points:
- Row and column filters defined in Account A apply when Account B accesses the data — Account B cannot bypass them
- Account B pays for their own Athena query execution; Account A pays for S3 storage
- Account B cannot re-share data shared with them (no re-sharing by default)
- Resource links are local aliases — deleting a resource link in Account B does not affect Account A's data

Exam patterns:
- "Share a governed data lake table from Account A to Account B without copying data, while preserving row-level security" → Lake Formation cross-account sharing via RAM
- "Account B analysts query Account A's data — who pays for the Athena queries?" → Account B (query execution is Account B's cost)
- "Ensure Account B cannot see PII columns even when querying cross-account" → Column security defined in Account A's Lake Formation is enforced across accounts

### Registering S3 Locations and Governed Tables

**Registering S3 Locations:**

Before Lake Formation can govern data in S3, you must register the S3 path. Registration tells Lake Formation to take over access control for that path from S3 bucket policies.

```bash
aws lakeformation register-resource \
  --resource-arn arn:aws:s3:::my-data-lake-bucket/governed/ \
  --use-service-linked-role
```

After registration:
- The `AWSServiceRoleForLakeFormationDataAccess` service-linked role is used by Glue/Athena/EMR to access the data
- Existing S3 bucket policies must grant this service role access
- If you forget to grant the service role S3 access, all governed queries fail with S3 AccessDenied

Service roles that need Lake Formation permissions to work with governed data:
- **Glue ETL job role:** needs `lakeformation:GetDataAccess` in addition to IAM Glue permissions
- **EMR EC2 instance profile:** needs `lakeformation:GetDataAccess`
- **Athena:** uses the calling user's IAM role — that role needs Lake Formation SELECT

**Governed Tables:**

A Lake Formation feature that gives standard S3-backed Glue tables ACID transaction support:
- Multiple writers can insert/delete rows concurrently without conflicts
- Automatic compaction of small files (Lake Formation manages this)
- Supports row-level locking during writes

Governed Tables vs standard registered S3 tables:
- **Standard registered tables:** Lake Formation controls access, but S3 files are plain Parquet/ORC with no transactional guarantees
- **Governed Tables:** ACID transactions, automatic compaction, row-level conflict resolution

Exam pattern: "Need ACID transactions on a data lake table with automatic file compaction, managed entirely by AWS" → Lake Formation Governed Tables (or Amazon S3 Tables with Iceberg)

### IAMAllowedPrincipal and Legacy Mode (Migration Gotcha)

When Lake Formation is first enabled on an account that already uses Glue and Athena, all existing Glue tables have an implicit `IAMAllowedPrincipal` grant. This grant means:

> "Anyone with the appropriate IAM permissions can access this table — Lake Formation is not enforcing access control here."

This is **legacy mode**. In legacy mode, Lake Formation is present but effectively bypassed for those tables.

**What happens when you register an S3 location but don't revoke IAMAllowedPrincipal:**
- Athena queries continue to work for anyone with the right IAM permissions
- Column/row filters defined in Lake Formation are NOT enforced
- It looks like Lake Formation is configured, but it has no effect on data access

**To switch a table to full Lake Formation governance:**
```bash
# Revoke the IAMAllowedPrincipal grant — this hands control to Lake Formation
aws lakeformation revoke-permissions \
  --principal '{"DataLakePrincipalIdentifier": "IAM_ALLOWED_PRINCIPALS"}' \
  --permissions '["SELECT", "INSERT", "DELETE", "DESCRIBE"]' \
  --resource '{"Table": {"DatabaseName": "my_db", "Name": "my_table"}}'
```

After revocation, Lake Formation permissions become the sole arbiter of data access for that table. Anyone without an explicit LF grant will get AccessDenied immediately.

Exam patterns:
- "Lake Formation row filters are configured but Athena queries still return all rows" → IAMAllowedPrincipal grant has not been revoked; LF governance is not active on this table
- "Migrating existing Glue/Athena workload to Lake Formation — what must happen before LF permissions take effect?" → Register the S3 location AND revoke IAMAllowedPrincipal grants from existing tables
- "After enabling Lake Formation, existing users lose Athena access to some tables" → IAMAllowedPrincipal was revoked but explicit LF grants were not yet created for those users

### Blueprints
- Pre-built workflows to ingest data into the lake
- Sources: RDS, on-premises databases
- Handles: crawling, ETL, catalog registration

---

## SageMaker Catalog (Business Data Catalog)

- Business-oriented data catalog for discovery
- Data lineage tracking (SageMaker ML Lineage Tracking)
- Domain, domain units, and projects for organizing data assets
- Integrates with Lake Formation for governance

---

## Exam Gotchas

- **Lake Formation** = answer for "fine-grained access control on data lake" or "column/row level security"
- **LF-Tags (TBAC)** = scalable access control using tags; scales to 500+ tables without O(tables × teams) individual grants
- Lake Formation simplifies permissions that would otherwise need S3 policies + IAM + Glue policies
- Glue Data Catalog is the **metadata backbone** used by Athena, Spectrum, EMR, Glue
- Without Lake Formation, you'd need separate views/datasets per user group for column filtering
- Lake Formation integrates with **Macie** for PII identification
- **Both IAM AND Lake Formation must allow access.** There is no way to grant Lake Formation access that bypasses IAM, and no way to grant IAM access that bypasses Lake Formation for governed tables. Both gates must be open.
- **IAM admin ≠ Lake Formation data lake admin.** An IAM administrator cannot grant LF permissions unless explicitly designated as a data lake administrator in Lake Formation settings.
- **IAMAllowedPrincipal is the silent bypass.** Existing Glue tables have this grant by default. Until it is revoked, Lake Formation column/row filters are not enforced — a common migration pitfall.
- **The Lake Formation service-linked role (`AWSServiceRoleForLakeFormationDataAccess`) must have S3 GetObject/PutObject on registered S3 paths.** If S3 bucket policy doesn't grant this role access, all governed Glue and Athena queries fail with AccessDenied on S3 — a common misconfiguration.
- **Glue ETL jobs need `lakeformation:GetDataAccess` in addition to standard Glue IAM permissions** to read or write governed tables. Without this, the Glue job gets AccessDenied even if the job's IAM role has S3 full access.
- **LF-Tags scale better than resource-based grants for large organizations.** If you have 500+ tables and dozens of teams, resource-based grants require O(tables × teams) individual grants. LF-Tags reduce this to O(teams) grants using tag expressions.
- **Row filters are enforced transparently — users cannot bypass them.** A row filter applied to a Lake Formation principal applies to ALL query engines (Athena, Redshift Spectrum, Glue, EMR) — the user cannot write a query that retrieves filtered-out rows, even with full SQL knowledge.
- **Cross-account sharing requires both a RAM share AND an explicit LF grant to Account B's account ID.** RAM alone shares catalog metadata; the LF grant is what controls actual data access.
- **Column masking in Lake Formation excludes columns entirely — it does not replace values with NULL or a hash.** True value-level masking (e.g., showing `****`) requires a view built on top of the governed table.

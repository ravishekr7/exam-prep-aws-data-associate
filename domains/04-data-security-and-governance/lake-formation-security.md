# Lake Formation Security

Fine-grained access control for your data lake.

---

## Why Lake Formation?

Without Lake Formation, securing a data lake requires:
- S3 bucket policies
- IAM policies
- Glue Data Catalog policies
- Creating separate views/datasets per user group

Lake Formation **centralizes all of this** in one place.

---

## Lake Formation Permission Model

Lake Formation uses a **two-layer security model**:

### Layer 1: IAM — Who Can Call Which APIs
IAM controls which AWS API calls a principal can make. For example, an IAM policy must allow `glue:GetTable` and `athena:StartQueryExecution` for an analyst to run Athena queries.

### Layer 2: Lake Formation — What Data Can Be Accessed
Lake Formation controls which databases, tables, and columns within those tables a principal can actually see and query. Even if IAM allows the API calls, Lake Formation is the data access gatekeeper.

### Both Must Allow
For governed tables (tables registered in Lake Formation):
```
BOTH IAM AND Lake Formation must grant access.
IAM allows → Lake Formation denies → ACCESS DENIED
IAM denies → Lake Formation allows → ACCESS DENIED
IAM allows → Lake Formation allows → ACCESS GRANTED
```

### Lake Formation Supersedes S3 Bucket Policies (For Registered Locations)
When an S3 location is registered with Lake Formation, **Lake Formation permissions take precedence** over S3 bucket ACLs and bucket policies for query engines like Athena, Glue, and EMR that honor Lake Formation security.

**Critical gotcha:** A user with `AdministratorAccess` (S3 Admin) IAM policy **will still get AccessDenied** if they have not been granted Lake Formation table permissions. Lake Formation is additive on top of IAM — both must grant access.

```
Scenario: New analyst joins. Admin grants them:
  - IAM: AthenaFullAccess, GlueReadOnlyAccess
  - Forgets to add Lake Formation SELECT permission on the relevant tables

Result: Analyst gets AccessDenied when querying Athena. IAM alone is not enough.
```

---

## Access Control Levels

| Level | Example | How |
|-------|---------|-----|
| **Database** | Analyst sees `analytics_db` | Grant database permission |
| **Table** | Analyst sees `orders` table only | Grant table permission |
| **Column** | Hide `ssn` column from analysts | Column-level permission |
| **Row** | Show only rows where `region='US'` | Row filter expression |
| **Cell** | Combine row + column restrictions | Most granular |

---

## Tag-Based Access Control (TBAC) / LF-Tags

### Why LF-Tags?
Managing permissions at the individual table level doesn't scale. If you have 500 tables across 20 departments, you'd need to create and maintain 500 individual Lake Formation grants per user group.

LF-Tags solve this with attribute-based access control:

### How LF-Tags Work
1. **Define tags:** Create tag keys and values (e.g., `department` with values `finance`, `marketing`, `engineering`)
2. **Assign tags to resources:** Tag databases, tables, and columns with these attributes
3. **Grant based on tags:** "The FinanceAnalysts group can SELECT on all resources where `department=finance`"
4. **Automatic inheritance:** New tables tagged with `department=finance` automatically inherit the FinanceAnalysts permission — no manual grant needed

```
Tag Assignment:
  finance_db → department=finance
  finance_db.revenue_table → department=finance
  finance_db.payroll_table → department=finance, sensitivity=restricted

Permission Grant:
  FinanceAnalysts role:
    SELECT on department=finance → can query revenue_table
    Cannot query payroll_table (missing sensitivity=restricted grant)

  FinanceManagers role:
    SELECT on department=finance AND sensitivity=restricted → can query both tables
```

### Tag Hierarchy
Tags flow **downward**: database → table → column. A tag applied to a database is inherited by all tables in that database. A tag applied to a table is inherited by all columns. You can override at a lower level.

### Exam Pattern
"A company has 500 tables across 20 departments. They need to manage access at scale without manually maintaining permissions for every table." → **LF-Tags (Lake Formation Tag-Based Access Control)**.

---

## Row-Level and Column-Level Security

### Column-Level Security
Exclude specific columns from a user or role's view. Applied at the table level in Lake Formation.

```python
# Grant SELECT on all columns EXCEPT ssn and credit_card_number
lakeformation_client.grant_permissions(
    Principal={'DataLakePrincipalIdentifier': 'arn:aws:iam::123456789:role/AnalystRole'},
    Resource={
        'TableWithColumns': {
            'DatabaseName': 'customer_db',
            'Name': 'customers',
            'ColumnWildcard': {
                'ExcludedColumnNames': ['ssn', 'credit_card_number']
            }
        }
    },
    Permissions=['SELECT']
)
```

Result: When AnalystRole runs `SELECT * FROM customers`, the `ssn` and `credit_card_number` columns are not returned — even though the underlying S3 file contains them.

### Row-Level Security
Create a **data filter** with a SQL WHERE clause expression. Assign the filter to a principal. That principal only sees rows matching the filter.

```python
# Create a row filter: only show APAC region data
lakeformation_client.create_data_cells_filter(
    TableData={
        'TableCatalogId': '123456789012',
        'DatabaseName': 'sales_db',
        'TableName': 'global_sales',
        'Name': 'apac_only_filter',
        'RowFilter': {
            'FilterExpression': "region = 'APAC'"
        }
    }
)

# Grant with the filter applied
lakeformation_client.grant_permissions(
    Principal={'DataLakePrincipalIdentifier': 'arn:aws:iam::123456789:role/APACAnalystRole'},
    Resource={
        'DataCellsFilter': {
            'DatabaseName': 'sales_db',
            'TableName': 'global_sales',
            'Name': 'apac_only_filter'
        }
    },
    Permissions=['SELECT']
)
```

### Column Masking
Instead of excluding a column entirely, mask the value. The column is visible in the schema, but values are replaced with NULL or a hash. Useful for showing analysts that a field exists (for schema awareness) without revealing its content.

---

## Data Sharing

### Redshift Data Sharing
- Share live data between Redshift clusters without copying
- Cross-account and cross-region sharing
- Consumer pays for compute, producer pays for storage

### Lake Formation Cross-Account Data Sharing

#### Step-by-Step Process
1. **Register the S3 location** in Lake Formation in Account A (the data producer account)
2. **Create a RAM share** for the Glue database in Account A (using AWS Resource Access Manager)
3. **Accept the RAM share** in Account B (the data consumer account)
4. **Grant Lake Formation permissions** in Account A to Account B principals (or grant to Account B, which then manages its own users' access)
5. **Account B principals** query the shared tables using Athena, EMR, etc.

#### Row/Column Filters Preserved
When sharing tables cross-account, **row-level and column-level filters are applied on the producer side** — Account B principals only see the data their Lake Formation permissions allow, even though they're querying from a different account.

---

## Lake Formation + Other Services

| Service | Integration |
|---------|-------------|
| **Athena** | Enforces column/row-level security during queries |
| **Redshift Spectrum** | Enforces Lake Formation permissions on external tables |
| **EMR** | Row/column filtering when reading from catalog |
| **Glue** | Permissions on catalog objects |

---

## Exam Gotchas

- **"Column-level security"** or **"row-level security"** = Lake Formation
- **"Tag-based access control"** = Lake Formation TBAC
- Lake Formation replaces the need for complex S3 + IAM + Glue policy combinations
- Lake Formation permissions are enforced by Athena, Redshift Spectrum, and EMR
- **"Share data across accounts without copying"** = Redshift Data Sharing or Lake Formation cross-account

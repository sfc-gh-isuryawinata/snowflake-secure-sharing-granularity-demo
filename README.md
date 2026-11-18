# Snowflake Secure Data Sharing: Granularity Demo

A hands-on lab showing that Snowflake Secure Data Sharing is **not limited to whole databases or schemas**. A provider controls exactly what a consumer sees, down to individual tables, columns, rows, and even per-consumer rows, all from one share.

No data is copied. The consumer queries the provider's data live and read-only, using its own compute.

## What the demo proves

| # | Level | Mechanism | Object |
|---|---|---|---|
| 1 | Schema | Grant `USAGE` only on the schemas you want to expose | `HR` schema is never shared |
| 2 | Table | Grant `SELECT` per table | `SALES.ORDERS` shared, `SALES.INTERNAL_COSTS` not |
| 3 | Column | Secure view that leaves out columns | `SHARED.V_CUSTOMERS_NO_PII` (no email/phone) |
| 4 | Row (static) | Secure view with a `WHERE` filter | `SHARED.V_ORDERS_APAC` |
| 5 | Row (per consumer) | Secure view checked against an entitlement table using `CURRENT_ACCOUNT()` | `SHARED.V_ORDERS_ENTITLED` |

The entitlement table (`SHARED.CONSUMER_ENTITLEMENTS`) is **not** shared, so a consumer can't see what other consumers are entitled to.

## Repository contents

| File | Description |
|---|---|
| `secure_data_sharing_granularity_demo.ipynb` | Snowflake Notebook with every step, explanation, and the cleanup step |
| `README.md` | This file |

## Prerequisites

- Two Snowflake accounts: a **provider** and a **consumer**.
- Both in the **same cloud region**. A direct share can't cross regions. For a consumer in another region, publish the share through a listing with auto-fulfillment instead.
- On the provider: a role with `CREATE SHARE` and `CREATE DATABASE` (for example `ACCOUNTADMIN`), and a warehouse (the notebook uses `COMPUTE_WH`).
- On the consumer: a role with `IMPORT SHARE` / `CREATE DATABASE` (for example `ACCOUNTADMIN`).
- The consumer's **account identifier** (`<ORG_NAME>.<ACCOUNT_NAME>`) and **account locator**. Run this in the consumer account to get them:
  ```sql
  SELECT CURRENT_ORGANIZATION_NAME() || '.' || CURRENT_ACCOUNT_NAME() AS account_identifier,
         CURRENT_ACCOUNT() AS account_locator;
  ```

## How to run

1. Import `secure_data_sharing_granularity_demo.ipynb` into a Snowflake Workspace (or Snowflake Notebooks) in the **provider** account.
2. In **Step 0**, edit the configuration cell:
   ```python
   CONSUMER_ACCOUNT = "<CONSUMER_ORG>.<CONSUMER_ACCOUNT>"
   CONSUMER_LOCATOR = "<CONSUMER_LOCATOR>"
   ```
   The cell checks both values and stops if the placeholders are still there. The SQL cells pick the values up through Jinja (`{{CONSUMER_ACCOUNT}}`). The provider's own account is detected automatically.
3. Run the notebook from the top to the end of Step 6. It:
   - creates `SHARING_DEMO_DB` with dummy `SALES`, `HR` and `SHARED` schemas
   - builds the secure views
   - creates `SHARING_DEMO_SHARE` and grants only the selected objects
   - adds the consumer account
   - checks the share contents and previews the consumer's view with `SIMULATED_DATA_SHARING_CONSUMER`
4. Run the **Step 7** SQL in the **consumer** account. Replace `<PROVIDER_ORG>.<PROVIDER_ACCOUNT>` with the provider's identifier, mount the share, and confirm:
   - `HR` is not visible
   - only `ORDERS` is visible in `SALES`
   - `V_CUSTOMERS_NO_PII` has no `EMAIL` / `PHONE`
   - `V_ORDERS_APAC` and `V_ORDERS_ENTITLED` return only APAC rows
   - secure view definitions are hidden

### Expected result

| Querying account | `V_ORDERS_ENTITLED` regions |
|---|---|
| Provider | APAC, EMEA, AMER (9 orders) |
| Consumer | APAC only (4 orders) |

## Re-running

Every cell can be re-run safely:
- `CREATE ... IF NOT EXISTS` for the database, schemas, tables and share, so nothing is dropped and the share keeps its grants
- `INSERT OVERWRITE` for the sample data, so rows aren't duplicated
- `CREATE OR REPLACE SECURE VIEW ... COPY GRANTS` for the views, so the share keeps access to them
- `GRANT` and `ALTER SHARE ... ADD ACCOUNTS` do nothing if they're already in place

## Business Critical edition note

If the provider is on **Business Critical** and the consumer is on a lower edition, Snowflake blocks the share by default:

> Sharing is not allowed from a provider account on BUSINESS_CRITICAL+ edition to target accounts on lower edition…

This guardrail protects regulated data. The demo uses only dummy data, so it overrides the check **for this share only**:

```sql
ALTER SHARE SHARING_DEMO_SHARE ADD ACCOUNTS = <CONSUMER_ORG>.<CONSUMER_ACCOUNT> SHARE_RESTRICTIONS = FALSE;
```

This needs the `OVERRIDE SHARE RESTRICTIONS` privilege. Don't do this for real regulated data unless the consumer has equivalent security controls.

## Cleanup

**Step 8** in the notebook removes everything. Run it in this order:

1. Consumer account:
   ```sql
   DROP DATABASE IF EXISTS SHARING_DEMO_FROM_PROVIDER;
   ```
2. Provider account (the Step 8 cell):
   ```sql
   ALTER SHARE IF EXISTS SHARING_DEMO_SHARE REMOVE ACCOUNTS = <CONSUMER_ORG>.<CONSUMER_ACCOUNT>;
   DROP SHARE IF EXISTS SHARING_DEMO_SHARE;
   DROP DATABASE IF EXISTS SHARING_DEMO_DB;
   ```

Dropping the share revokes the consumer's access immediately.

## Going further

- **Masking policies** for value-level protection, for example partially masking an email column.
- **Row access policies** as a policy-based alternative to the entitlement view.
- **Listings** (internal or Marketplace) to share across regions with auto-fulfillment.

## References

- [Introduction to Secure Data Sharing](https://docs.snowflake.com/en/user-guide/data-sharing-intro)
- [Use secure objects to control data access](https://docs.snowflake.com/en/user-guide/data-sharing-secure-views)
- [GRANT … TO SHARE](https://docs.snowflake.com/en/sql-reference/sql/grant-privilege-share)

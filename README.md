> ## ⚠️ Current state: replicating nothing
>
> As of **2026-08-19**, `Included Table Regex` is `test\.Sales\..*`. **No source table matches it** —
> either the `test` database has no `Sales` schema, or nothing in it has Change Tracking enabled. The
> connector is RUNNING with 0 errors but has an empty scope.
>
> Switching scope away from `SalesLT` caused the connector to **drop the entire destination schema**
> `RAW_DATA.AZURE_SQL`, along with all 9 tables and their journals. This was done knowingly; recovery
> was declined. See [De-scoping is destructive](#de-scoping-is-destructive) and
> [Restoring the SalesLT pipeline](#restoring-the-saleslt-pipeline).

# Azure SQL → Snowflake Ingest (OpenFlow SQL Server CDC)

Change Tracking–driven replication from an Azure SQL Database into Snowflake using the OpenFlow
**SQL Server** connector (not native CDC).

Deployed and verified: **2026-08-18**. Last change: **2026-08-19** — scope switched from `SalesLT` to
`Sales`, which matched nothing; the `SalesLT` destination schema was dropped by the connector as a
result.

**Change history:** see [`changelog.md`](./changelog.md). This README and the changelog are both
published to <https://github.com/andrijama/tasty> (`main`) — that is the canonical copy. Every
change to this setup must be reflected in both and pushed in the session it was made: the README
kept current, and an append-only changelog entry recording when, by whom, what, why and the result.

## Data flow

```
Azure SQL Database (andrijasqlse, db "test")
  scope = test\.Sales\..*   →  0 tables matched
        │
        ▼
OpenFlow runtime "azure"  (SPCS, connector SQLServer v0.48.0-80ad996b)
        │  RUNNING, idle — nothing to replicate
        ▼
Snowflake  RAW_DATA.AZURE_SQL   — schema does not currently exist
```

When the scope matches tables, the shape is:
source table → snapshot → `<table>_JOURNAL_<hash>_<gen>` → MERGE via `MYWH` → destination table.

## Runtime and connector

| Item | Value |
|---|---|
| OpenFlow runtime | `azure` (Azure Runtime) |
| Deployment type | SPCS, Snowflake account `SFSEEUROPE-IE_DEMO99` |
| Runtime API | `https://of--sfseeurope-ie-demo99.snowflakecomputing.app/azure/nifi-api` |
| nipyapi profile | `ie_demo99_azure` |
| Registry / bucket | `ConnectorFlowRegistryClient` / `connectors` |
| Connector flow | `sqlserver-multidatabase` |
| Deployed version | `0.48.0-80ad996b` |
| Process group name | `SQLServer` |
| Process group id | `151d549e-01a0-1000-0000-0000240ac508` |
| Merge cadence | Every 15 minutes — `Merge Task Schedule CRON` = `0 0/15 * * * ?` |

This account is **Gen1** (the `OPENFLOW` SQL grammar is not enabled — `SHOW OPENFLOW CONNECTOR
DEFINITIONS` fails), so the connector is managed through the NiFi API via `nipyapi`, not SQL.

There is also a `sqlserver-multidatabase-cdc` flow in the registry. That is the native-CDC variant
and is **not** what is deployed here; this setup uses the Change Tracking connector.

## Snowflake destination

| Setting | Value |
|---|---|
| Destination database | `RAW_DATA` |
| Destination schema | `AZURE_SQL` |
| Destination Schema Pattern | `AZURE_SQL` (literal — overrides the connector default) |
| Object Identifier Resolution | `CASE_INSENSITIVE` |
| Snowflake Role | `OPENFLOW_ADMIN` |
| Authentication Strategy | `SNOWFLAKE_MANAGED` |
| Snowflake Warehouse | `MYWH` |

Notes:

- `SNOWFLAKE_MANAGED` means **no** private key, passphrase, or Snowflake username is set — those
  parameters are intentionally left blank.
- The connector default for `Destination Schema Pattern` is
  `${source.database.name}_${source.schema.name}`, which would have created `TEST_SALESLT`. It is
  pinned to the literal `AZURE_SQL` to satisfy the required destination schema. Consequence: every
  source schema collapses into one destination schema, so same-named tables from different source
  schemas would collide.
- `RAW_DATA.AZURE_SQL` is **owned by** `OPENFLOW_ADMIN`, and so are the tables it creates. Other
  roles — including `ACCOUNTADMIN` — get "Insufficient privileges" on SELECT until granted. Use
  `USE ROLE OPENFLOW_ADMIN` to query, or grant SELECT explicitly.

## Source

| Setting | Value |
|---|---|
| Server | `andrijasqlse.database.windows.net` |
| Port | `1433` |
| Database | `test` |
| Connection URL | `jdbc:sqlserver://andrijasqlse.database.windows.net:1433;encrypt=false;database=test;` |
| Username / password | Held only in the `SQLServer Source Parameters` context (sensitive). Not stored in this repo. |

### Ingested tables

Scope is defined by **regex**, not an explicit list:

| Parameter | Value |
|---|---|
| `Included Table Regex` | `test\.Sales\..*` |
| `Included Table Names` | *(empty)* |

The regex matches `database.schema.table`. **It currently matches nothing** — either `test` has no
`Sales` schema, or nothing in it has Change Tracking enabled. Note that `test\.Sales\..*` does *not*
match the `SalesLT` schema, because of the escaped dot after `Sales`.

Database-level Change Tracking on `test`: **enabled**, retention **2 days**, auto-cleanup on.

### Previously replicated (until 2026-08-19)

These 9 `SalesLT` tables were replicating and are the reference for a healthy pipeline. **Their
destination tables no longer exist.**

| Source table | Destination | Rows at load |
|---|---|---|
| `test.SalesLT.Address` | `RAW_DATA.AZURE_SQL.ADDRESS` | 450 |
| `test.SalesLT.Customer` | `RAW_DATA.AZURE_SQL.CUSTOMER` | 847 |
| `test.SalesLT.CustomerAddress` | `RAW_DATA.AZURE_SQL.CUSTOMERADDRESS` | 417 |
| `test.SalesLT.Product` | `RAW_DATA.AZURE_SQL.PRODUCT` | 295 |
| `test.SalesLT.ProductCategory` | `RAW_DATA.AZURE_SQL.PRODUCTCATEGORY` | 41 |
| `test.SalesLT.ProductDescription` | `RAW_DATA.AZURE_SQL.PRODUCTDESCRIPTION` | 762 |
| `test.SalesLT.ProductModel` | `RAW_DATA.AZURE_SQL.PRODUCTMODEL` | 128 |
| `test.SalesLT.SalesOrderDetail` | `RAW_DATA.AZURE_SQL.SALESORDERDETAIL` | 542 |
| `test.SalesLT.SalesOrderHeader` | `RAW_DATA.AZURE_SQL.SALESORDERHEADER` | 32 |

Before that, the `dbo` tables (`Orders` 568, `Products` 31, `Customers` 23, `OrderItems` 2) were
replicated; they were de-scoped earlier the same day and likewise dropped.

The stock AdventureWorksLT table `ProductModelProductDescription` never appeared — it either does not
exist in this database or has no Change Tracking. Not verified against the source.

### De-scoping is destructive

**Removing tables from scope deletes them in Snowflake.** Observed twice on 2026-08-19:

1. De-scoping the four `dbo` tables → the connector dropped `ORDERS`, `PRODUCTS`, `CUSTOMERS`,
   `ORDERITEMS` and their journal tables.
2. Switching the regex from `SalesLT` to `Sales` → the connector dropped all 9 tables **and the
   `AZURE_SQL` schema itself**.

In both cases no manual `DROP` was issued and no bulletin or warning was raised. The connector
documentation's claim that it "does not automatically delete data" does **not** hold for de-scoping
on this version (`0.48.0-80ad996b`).

Before narrowing scope, clone anything you want to keep
(`CREATE SCHEMA <backup> CLONE RAW_DATA.AZURE_SQL`). Within the 1-day retention window,
`UNDROP SCHEMA RAW_DATA.AZURE_SQL` also recovers it.

### A scope that matches nothing fails silently

An empty match set is treated as valid: the flow stays RUNNING, `bulletin_errors: 0`, and the Table
State Store simply goes empty. Nothing distinguishes "schema doesn't exist" from "typo in the regex"
from "tables exist but have no Change Tracking". Always check the state store after a scope change.

### Restoring the SalesLT pipeline

```bash
nipyapi --profile ie_demo99_azure ci configure_inherited_params \
  --process_group_id 151d549e-01a0-1000-0000-0000240ac508 \
  --parameters '{"Included Table Regex":"test\\.SalesLT\\..*"}'
```

The connector recreates `RAW_DATA.AZURE_SQL` and re-snapshots all 9 tables (~2 minutes). It will
**not** snapshot over an existing destination table, so if you `UNDROP SCHEMA` first to recover the
old data, rename that schema out of the way before reverting the regex.

### Change Tracking is not native CDC

This connector reads Change Tracking (`sys.change_tracking_tables` / `CHANGETABLE`). Running
`sys.sp_cdc_enable_table` enables the *native CDC* feature and has **no effect** here — the table
stays undiscovered and no bulletin is raised, so it fails silently. The correct statement is:

```sql
ALTER TABLE <schema>.<table> ENABLE CHANGE_TRACKING;
```

Confirmed on `SalesLT.Address` and `SalesLT.CustomerAddress`: `sp_cdc_enable_table` did nothing;
`ENABLE CHANGE_TRACKING` got them discovered and snapshotted within ~2 minutes, with no OpenFlow
change at all.

(`sqlserver-multidatabase-cdc` in the registry is the native-CDC variant; it is not what is deployed
here. Switching would require a redeploy and a full re-snapshot of every table.)

### Changing the ingest scope

```bash
# Regex scope (current approach) — whole schema, future-proof
nipyapi --profile ie_demo99_azure ci configure_inherited_params \
  --process_group_id 151d549e-01a0-1000-0000-0000240ac508 \
  --parameters '{"Included Table Names":"","Included Table Regex":"test\\.<schema>\\..*"}'

# Explicit list instead
nipyapi --profile ie_demo99_azure ci configure_inherited_params \
  --process_group_id 151d549e-01a0-1000-0000-0000240ac508 \
  --parameters '{"Included Table Regex":"","Included Table Names":"<full comma-separated FQN list>"}'
```

Rules that matter:

- Exactly one of `Included Table Names` / `Included Table Regex` needs a value; set the other to `""`.
- `Included Table Names` is a **full replacement**, not an append — always pass the complete list.
- FQNs and the regex are matched **case-sensitively** against the source (`test.SalesLT.Customer`).
- Changes apply **live**; no stop/start. With `Ingestion Type = full` the connector introspects,
  snapshots, and switches to incremental on its own (~60–120s per batch of tables).
- Anything dropping out of scope is **deleted in Snowflake** — see
  [De-scoping is destructive](#de-scoping-is-destructive).

Check what the connector actually thinks it is replicating:

```bash
nipyapi --profile ie_demo99_azure canvas get_controller_state 0e5a1fc2-555d-3868-bf67-a9ebf9d391a9
```

### Network access

Egress is already permitted — no change was needed:

- EAI `OPENFLOW_NETWORK_ACCESS_INTEGRATION`
- Network rule `DEMODB.PUBLIC.AZURESQL_NETWORK_RULES` includes
  `andrijasqlse.database.windows.net:1433`

## Parameter contexts

| Context | Id |
|---|---|
| `SQLServer Source Parameters` | `bf108151-8110-3a64-9b01-a2da69cf0202` |
| `SQLServer Destination Parameters` | `9aba99fb-714a-3b59-b893-b868a2df1032` |
| `SQLServer Ingestion Parameters` | `8c512dc0-fbfe-3a04-a832-dd3c14f02cc4` |

The `SQLServer Ingestion Parameters` context holds **`Merge Task Schedule CRON`** — the Quartz CRON
that drives the `Merge Journal to Destination` processor (`MultiDatabaseMergeSnowflakeJournalTable`,
id `883fb705-6717-312b-8fbe-00eae27018e0`) in `SQLServer → Incremental Load` via its `Merge Schedule`
property. Current value **`0 0/15 * * * ?`** (every 15 minutes). It is a NiFi/Quartz CRON, not a
Snowflake interval schedule. To change the cadence, update this parameter (nipyapi picks it up live;
the processor briefly stops and restarts).

### Parameter asset — JDBC driver

The connector does not bundle a JDBC driver, and the Microsoft download link is a **zip archive**,
not a JAR.

1. Download `https://go.microsoft.com/fwlink/?linkid=2356503`
2. Extract; the JARs are under `sqljdbc_13.4/enu/jars/`
3. Use **`mssql-jdbc-13.4.0.jre11.jar`** only — not the `jre8`, `-sources`, or `-javadoc` variants

Uploaded to the `SQLServer Source Parameters` context and linked to the `SQLServer JDBC Driver`
parameter:

| Field | Value |
|---|---|
| Asset name | `mssql-jdbc-13.4.0.jre11.jar` |
| Asset id | `d8a7448c-6f15-3b25-bb3e-7f05c94f3da9` |
| SHA-256 | `e36f5237c1267983e5b88dc2169f6b9d7e50eceec6dc1ca31018e3877e14af66` |

A local copy is kept at `./driver/mssql-jdbc-13.4.0.jre11.jar`.

## Flow definition

`azure_sql_ingest_flow_20260819_152233.json` — current flow (empty `Sales` scope).

Earlier exports kept for history: `azure_sql_ingest_flow_20260819_113207.json` (9 SalesLT tables — the
last known-good state), `azure_sql_ingest_flow_20260819_105453.json` (7 tables, regex scope),
`azure_sql_ingest_flow_20260819_104716.json` (five explicit tables),
`azure_sql_ingest_flow_20260819_095127.json` (four explicit tables).

Sensitive values are **not** present: all sensitive parameters export with `value: null`, and the
`SQLServer Username` parameter value was redacted to `***REDACTED***`. Re-apply the username and
password by hand if you import this definition elsewhere.

Note: the `SQLServer Username` parameter is declared **non-sensitive** by the connector, so it does
export in cleartext. Redact it manually after every export.

## Operating the connector

All commands use the `ie_demo99_azure` profile and process group id above.

```bash
PG=151d549e-01a0-1000-0000-0000240ac508
P=ie_demo99_azure

# Status / health
nipyapi --profile $P ci get_status --process_group_id "$PG"

# Errors
nipyapi --profile $P bulletins get_bulletin_board --pg_id "$PG"

# Stop (processors only — quick restart)
nipyapi --profile $P ci stop_flow --process_group_id "$PG"

# Stop fully (before config or version changes)
nipyapi --profile $P ci stop_flow --process_group_id "$PG" --disable_controllers

# Start (enables controllers, then processors)
nipyapi --profile $P ci start_flow --process_group_id "$PG"

# Validate configuration without starting
nipyapi --profile $P ci verify_config --process_group_id "$PG" --verify_controllers=false
```

A healthy state is `state: RUNNING`, 86 of 87 running processors (one intentionally disabled),
`invalid_processors: 0`, `bulletin_errors: 0`, 9 of 10 controllers enabled.

Verify data in Snowflake:

```sql
USE ROLE OPENFLOW_ADMIN;
SELECT TABLE_NAME, ROW_COUNT
FROM RAW_DATA.INFORMATION_SCHEMA.TABLES
WHERE TABLE_SCHEMA = 'AZURE_SQL' AND TABLE_NAME NOT LIKE '%_JOURNAL_%'
ORDER BY TABLE_NAME;
```

## Known issues and troubleshooting

**`Snowflake Private Key Service` shows INVALID / stays disabled.** Expected. That controller is
only used by `KEY_PAIR` auth; this connector uses `SNOWFLAKE_MANAGED`. It is the single failure in
`verify_config` and the one disabled controller of ten. No impact.

**JDBC driver version.** The Snowflake connector guidance recommends `12.10.0`; `13.x` was still
under validation (FLOW-10139). This deployment pins `13.4.0.jre11` as required by `AGENTS.md` and it
works. If you hit driver-level oddities, `12.10.0.jre11` from Maven Central is the fallback. On
`13.2+`, `VECTOR` columns may need `vectorTypeSupport=off` appended to the connection URL.

**Controllers stuck in ENABLING.** The JDBC driver asset is missing or unlinked. Re-upload it to the
source context and link it to `SQLServer JDBC Driver`.

**Insufficient privileges on SELECT.** The schema and tables belong to `OPENFLOW_ADMIN`. Switch role
or grant SELECT.

**Change Tracking retention expiry** is the main long-term risk. Retention is 2 days — the documented
minimum. If the connector is stopped for longer than that, SQL Server purges change records and the
affected tables need a full re-snapshot. For anything beyond demo use, raise it:

```sql
ALTER DATABASE test SET CHANGE_TRACKING = ON (CHANGE_RETENTION = 5 DAYS, AUTO_CLEANUP = ON);
```

**Journal tables** (`*_JOURNAL_*`) are not cleaned up automatically; that is by design, for auditing.

**Schema changes.** Added columns appear automatically. Renamed and dropped columns are retained
with a `__SNOWFLAKE_DELETED` suffix rather than being removed. Do not hand-edit the destination
table structure while replication is running.

**`Object Identifier Resolution` cannot be changed** after replication starts without a full reset
(stop flow, clear state, drop destination tables, restart).

## Secrets

No credentials are stored in this repository — not in the flow JSON, not here. The SQL Server
username and password live only in the OpenFlow `SQLServer Source Parameters` context. Parameters
were applied via a temporary file outside the repo, which was deleted immediately afterward.

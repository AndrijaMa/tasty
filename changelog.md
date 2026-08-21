# Changelog — Azure SQL → Snowflake Ingest (OpenFlow)

Change history for the OpenFlow SQL Server (Change Tracking) pipeline that lands
`andrijasqlse` / database `test` into `RAW_DATA.AZURE_SQL` on Snowflake account
`SFSEEUROPE-IE_DEMO99`.

Canonical location: <https://github.com/andrijama/tasty> (`main`), alongside `README.md`.
Setup documentation: `README.md`.

Append-only, newest first. Every change gets an entry — including ones that failed or were
reverted. No credentials, connection strings with credentials, or private keys in this file.

Entry fields: **When / Who / What / Why / Result**.

---

## 2026-08-21

- **When:** 2026-08-21T09:03+02:00
  **Who:** Andrija Marcic (`ANDRIJA`, GitHub `AndrijaMa`) — applied by Cortex Code via nipyapi
  (profile `ie_demo99_azure`).
  **What:** Changed the destination compute warehouse from `MYWH` to `OPENFLOW_WH`. Updated parameter
  `Snowflake Warehouse` in the `SQLServer Destination Parameters` context
  (`9aba99fb-714a-3b59-b893-b868a2df1032`), process group `SQLServer`
  (`151d549e-01a0-1000-0000-0000240ac508`), via
  `nipyapi ci configure_inherited_params` (dry-run first, then applied). Also updated the pinned
  value in `AGENTS.md` and `README.md`. Confirmed `OPENFLOW_WH` exists (STANDARD, X-Small,
  auto-resume) and that `OPENFLOW_ADMIN` already holds `USAGE`/`OPERATE`/`MONITOR` on it — no grant
  needed.
  **Why:** User requested the switch off `MYWH` to the dedicated `OPENFLOW_WH` warehouse.
  **Result:** Applied live — `parameters_updated: 1`, `contexts_modified: 1`. Re-export confirms
  `Snowflake Warehouse = OPENFLOW_WH`. No stop/start required. No new warehouse- or
  Snowflake-connection bulletins appeared. The connector remains in its **pre-existing idle state**
  unrelated to this change: `Included Table Regex = test\.Sales\..*` still matches nothing,
  `RAW_DATA.AZURE_SQL` does not exist, and the `Table State Service` controller
  (`0e5a1fc2-555d-3868-bf67-a9ebf9d391a9`) is disabled — hence the 20 invalid processors / 5 "Create
  Journal Table" bulletins, all carried over from 2026-08-19. Data-landing verification not possible
  until the ingest scope is restored (see README → *Restoring the SalesLT pipeline*); the new
  warehouse will take effect on the next snapshot/merge once tables are back in scope.

---

## 2026-08-20

- **When:** 2026-08-20T16:23+02:00
  **Who:** Andrija Marcic (`ANDRIJA`, GitHub `AndrijaMa`) — applied by Cortex Code.
  **What:** Changed the `AGENTS.md` changelog convention: the **When** field must now be a full
  ISO 8601 timestamp with timezone offset (`YYYY-MM-DDTHH:MM±HH:MM`), not just the date — a timestamp
  next to every change. Adopted the `When:` line format in this changelog going forward; existing
  entries left as-is (append-only).
  **Why:** Recording only the day loses the time-of-day ordering of multiple same-day changes.
  **Result:** `AGENTS.md` and `changelog.md` updated; pushed to `andrijama/tasty@main`.

- **Who:** Andrija Marcic (`ANDRIJA`, GitHub `AndrijaMa`) — applied by Cortex Code via nipyapi
  (profile `ie_demo99_azure`).
  **What:** Set the connector merge cadence to every 15 minutes. Changed parameter
  `Merge Task Schedule CRON` in the `SQLServer Ingestion Parameters` context
  (`8c512dc0-fbfe-3a04-a832-dd3c14f02cc4`) from `0 * * * * ?` (every minute) to
  `0 0/15 * * * ?`. This parameter feeds the `Merge Schedule` property of the
  `Merge Journal to Destination` processor (`MultiDatabaseMergeSnowflakeJournalTable`,
  `883fb705-6717-312b-8fbe-00eae27018e0`) in `SQLServer → Incremental Load`. Value is a NiFi/Quartz
  CRON, not a Snowflake interval schedule.
  **Why:** Reduce merge frequency from once a minute to once every 15 minutes.
  **Result:** Applied live via `nipyapi.parameters.update_parameter_in_context` (referencing
  processor briefly stopped/restarted). Merge processor back to RUNNING / VALID, 0 validation
  errors. README updated (Runtime table + Parameter contexts section).

- **Who:** Andrija Marcic (`ANDRIJA`, GitHub `AndrijaMa`) — applied by Cortex Code.
  **What:** Reversed the "README stays local" rule in `AGENTS.md`. `README.md` is now published to
  `andrijama/tasty@main` alongside `changelog.md`, and **must be pushed every time the solution
  changes** (parameters, table scope, driver, grants, network rules, source-side Change Tracking) in
  the same session as the matching changelog entry. Flow JSON exports remain local — they leak the
  source username. Added the secret-review-before-push reminder for both files and the
  `gh auth switch -u AndrijaMa` note.
  **Why:** The changelog alone does not describe the current state of the solution; a pushed
  changelog with a stale or absent README is incomplete for anyone reading the repo.
  **Result:** `AGENTS.md`, `README.md` and `changelog.md` updated; README published to the repo for
  the first time. Checked for secrets — the README documents the source server, database and port
  but no username or password.

- **Who:** Andrija Marcic (`ANDRIJA`, GitHub `AndrijaMa`) — applied by Cortex Code.
  **What:** Added a `Change log` section to `AGENTS.md` requiring an append-only `changelog.md`
  (When / Who / What / Why / Result per entry) whose canonical copy lives in
  `github.com/andrijama/tasty`; expanded the `README.md` requirements to cover end-to-end setup
  steps, prerequisites/tooling, and a pointer to the changelog. Created this file and pushed it
  to the repo.
  **Why:** Setup and change history were only in the README and in session memory, with no
  attributed, dated record of what changed.
  **Result:** `AGENTS.md` and `changelog.md` updated locally; `changelog.md` published to
  `andrijama/tasty@main`. `README.md` and the flow JSON exports remain local by design.

## 2026-08-19

- **Who:** Andrija Marcic — applied by Cortex Code.
  **What:** Narrowed scope by switching `Included Table Regex` from `test\.SalesLT\..*` to
  `test\.Sales\..*`.
  **Why:** Attempt to re-scope the ingest to a different source schema.
  **Result:** ⚠️ **Destructive.** The regex matched nothing (no `Sales` schema, or no Change
  Tracking on it). The connector dropped all 9 replicated tables *and the `AZURE_SQL` schema
  itself* — no manual DROP, no bulletin, flow stayed RUNNING with an empty state store.
  Pipeline left idle; recovery declined by the user. Flow export
  `azure_sql_ingest_flow_20260819_152233.json`. Recovery path: set the regex back to
  `test\.SalesLT\..*` (recreates schema + re-snapshots ~2 min), or `UNDROP SCHEMA` within the
  1-day retention window.

- **Who:** Andrija Marcic — applied by Cortex Code.
  **What:** Enabled Change Tracking at the source on `SalesLT.Address` and
  `SalesLT.CustomerAddress` (`ALTER TABLE … ENABLE CHANGE_TRACKING`). No OpenFlow changes.
  **Why:** Both tables were in regex scope but not replicating.
  **Result:** Picked up by the existing regex in ~2 min. `ADDRESS` 450 rows,
  `CUSTOMERADDRESS` 417 rows. Confirmed that `sp_cdc_enable_table` (native CDC) is a **silent
  no-op** for this connector — it reads Change Tracking, not CDC.

- **Who:** Andrija Marcic — applied by Cortex Code.
  **What:** Replaced explicit table lists with whole-schema scope: `Included Table Regex` =
  `test\.SalesLT\..*`, `Included Table Names` = `""`. This also **de-scoped the `dbo` tables**.
  **Why:** Pick up future `SalesLT` tables automatically instead of editing the list per table.
  **Result:** 7 tables snapshotted — `CUSTOMER` 847, `PRODUCT` 295, `PRODUCTCATEGORY` 41,
  `PRODUCTDESCRIPTION` 762, `PRODUCTMODEL` 128, `SALESORDERDETAIL` 542, `SALESORDERHEADER` 32.
  ⚠️ De-scoping proved destructive: the connector **dropped** `ORDERS`, `PRODUCTS`, `CUSTOMERS`,
  `ORDERITEMS` and their journal tables. Flow exports
  `azure_sql_ingest_flow_20260819_104716.json`, `…_20260819_105453.json`, `…_20260819_113207.json`.

- **Who:** Andrija Marcic — applied by Cortex Code.
  **What:** Added `test.dbo.OrderItems` and `test.SalesLT.ProductDescription` to
  `Included Table Names` (full replacement of the parameter, not an append), via
  `nipyapi ci configure_inherited_params`.
  **Why:** Extend the demo beyond the three initial tables and cover a non-`dbo` source schema.
  **Result:** Applied live, no stop/start; connector introspected, snapshotted and returned to
  incremental in ~60–90s. `ORDERITEMS` 2 rows, `PRODUCTDESCRIPTION` 762 rows, 0 bulletins.
  Noted: because `Destination Schema Pattern` is pinned to the literal `AZURE_SQL`, non-`dbo`
  source schemas land in the same destination schema — same-named tables would collide.
  Flow export `azure_sql_ingest_flow_20260819_095127.json`.

## 2026-08-18

- **Who:** Andrija Marcic — applied by Cortex Code.
  **What:** Initial build of the pipeline. Deployed flow `sqlserver-multidatabase` from
  `ConnectorFlowRegistryClient` bucket `connectors` to OpenFlow runtime `azure` (driven via the
  NiFi API / nipyapi profile `ie_demo99_azure`, since `SHOW OPENFLOW CONNECTOR DEFINITIONS` is
  unsupported on this pre-SOM account). Destination `RAW_DATA.AZURE_SQL`, role `OPENFLOW_ADMIN`,
  warehouse `MYWH`, `SNOWFLAKE_MANAGED` auth, `CASE_INSENSITIVE` identifier resolution.
  `Destination Schema Pattern` pinned to the literal `AZURE_SQL` (default
  `${source.database.name}_${source.schema.name}` would have created `TEST_DBO`).
  `Included Table Names` = `test.dbo.Orders, test.dbo.Products, test.dbo.Customers` (FQNs).
  Uploaded `mssql-jdbc-13.4.0.jre11.jar` (extracted from the Microsoft ZIP at
  `go.microsoft.com/fwlink/?linkid=2356503`, path `sqljdbc_13.4/enu/jars/`) as the
  `SQLServer JDBC Driver` parameter asset. JDBC URL uses `encrypt=false`. Network access via EAI
  `OPENFLOW_NETWORK_ACCESS_INTEGRATION` → rule `DEMODB.PUBLIC.AZURESQL_NETWORK_RULES`
  (`andrijasqlse.database.windows.net:1433`).
  **Why:** Stand up an Azure SQL → Snowflake CDC ingest demo.
  **Result:** Working. `ORDERS` 568, `PRODUCTS` 31, `CUSTOMERS` 23 rows; 86/87 processors
  running, 0 bulletin errors. `*_JOURNAL_*` tables stay at 0 rows after snapshot (by design,
  never auto-cleaned). Expected benign noise: `Snowflake Private Key Service` reports INVALID and
  pre-enable `verify_config` lists ~46 "Controller Service … is disabled" failures under
  `SNOWFLAKE_MANAGED`.

- **Who:** Andrija Marcic.
  **What:** Created `AGENTS.md` pinning the runtime, connector type, destination settings, JDBC
  driver version, connection-URL template, source prerequisites and post-build artifact rules.
  **Why:** Make the ingest setup reproducible and stop agents substituting defaults or
  re-guessing connection details.
  **Result:** In effect for all subsequent work in this project.

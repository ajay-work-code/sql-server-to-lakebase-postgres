# Two Notebooks Stage MOSAIC's Move to Lakebase

Both `.dbc` files are single-notebook Databricks archives: zipped JSON with cell source and metadata but no outputs. Together they move the MOSAIC applications from Azure SQL Database onto Lakebase, Databricks' managed Postgres. **Azure SQL to Lakebase Migration** is a copy tool driven by a table list (the "registry"). It reads 26 registered Azure SQL objects over Spark JDBC, generates Postgres DDL that has columns and types only, and reloads each table with TRUNCATE plus CSV `COPY`. That serves a "table-by-table atomic cutover" of about 3 GB of read/write data behind the POSIT apps on Posit Workbench and Connect. **Lakebase Schema Setup** creates a new `mosaic_redesign` schema with eight empty tables that the redesigned app writes to directly: an audit trail, AI-assistant logs, a cache, daily briefs and an e-mail request queue. Their DDL was translated by hand from T-SQL and Delta. Both notebooks connect to Lakebase exactly as the generally available (GA) Postgres API documents. The cutover strategy, a short scheduled downtime with no dual writes, suits a database this small. Neither notebook is safe to run unattended as written. The migration notebook can truncate a table that apps already write to on Lakebase, and it leaves a table empty when its load fails. It also corrupts binary columns, drops every key, default and index, and never fails a run. The setup notebook swallows DDL errors, so a run that rolled back still finishes green, and it grants the apps no access. Both should be converted to `.ipynb` or `.py` in Git and fixed. After that, the migration notebook becomes a parameterized refresh job and a cutover runner for one app's tables at a time. The setup notebook becomes a one-time bootstrap, kept as the first script of a versioned schema-migration tool.

## Each archive is one zipped notebook with its outputs stripped

A `.dbc` is Databricks' "DBC Archive (notebooks only)" export, which the documentation describes as "a binary format that includes metadata and notebook command outputs" ([Databricks: Import and export notebooks](https://learn.microsoft.com/azure/databricks/notebooks/notebook-export-import#export-notebooks)). Opened as ZIP files, each archive holds **exactly one JSON entry and nothing else**: no folder, manifest or companion file ([migration archive](<../context/Azure SQL to Lakebase Migration.dbc>); [setup archive](<../context/Lakebase Schema Setup.dbc>)).

- `Azure SQL to Lakebase Migration.python`: 38,981 bytes of JSON, 8 cells.
- `Lakebase Schema Setup.python`: 24,429 bytes, 6 cells.

**All 14 cells have empty `results`**, every run timestamp is zeroed, and the last editor appears only as "a user". The archives therefore cannot show which compute ran them, how long runs took, or whether any run succeeded.

The metadata still says a good deal:

- Both files declare `NotebookV1`, Python and `reposExportFormat: "JUPYTER"`, the `.ipynb` format Databricks now uses by default ([Databricks: Manage notebook format](https://learn.microsoft.com/azure/databricks/notebooks/notebook-format)).
- Neither pins compute. **Neither defines widgets, so neither can accept job parameters without code edits.**
- `environmentMetadata: {"client": "5"}` uses the deprecated field name for a serverless environment version ([Databricks: Bundle resources](https://learn.microsoft.com/azure/databricks/dev-tools/bundles/resources#job)). Environment 5 ships Python 3.12.3 and databricks-sdk 0.67.0 but not psycopg 3 ([Databricks: Environment version 5](https://learn.microsoft.com/en-us/azure/databricks/release-notes/serverless/environment-version/five)). That is why both notebooks reinstall packages in every new session.
- The two notebooks were very likely created or imported together, from one template rather than edited over time. Their IDs are consecutive (…200 and …201), the cell IDs come from one counter, and every cell is at `commandVersion: 1`. The setup notebook's dependency and connection cells also duplicate the migration notebook's ([migration archive](<../context/Azure SQL to Lakebase Migration.dbc>); [setup archive](<../context/Lakebase Schema Setup.dbc>)).
- The code requires `databricks-sdk>=0.118.0`, a version released on 18 June 2026 ([PyPI: databricks-sdk](https://pypi.org/project/databricks-sdk/#history)), so it was written in mid-2026 or later.

What the archives leave out matters as much. They carry no job definitions, schedules, secret values or permissions. **They also do not contain the Genie Code skill** (`/Users/<user>/.assistant/skills/azure-sql-to-lakebase/SKILL.md`). The migration notebook says this skill holds its type map, cutover workflow and "10 documented gotchas from the POC". Skills are workspace files, not notebooks ([Databricks: Extend Genie Code with agent skills](https://learn.microsoft.com/azure/databricks/genie-code/skills)), so a DBC export never includes them, and this one could not be reviewed.

DBC is still a supported import and export format, through the UI and `databricks workspace import … --format DBC` ([Databricks CLI: workspace commands](https://learn.microsoft.com/azure/databricks/dev-tools/cli/reference/workspace-commands)). Git folders, however, recognize notebooks only as `.ipynb` files or as source files with the `# Databricks notebook source` marker ([Databricks: Supported asset types in Git folders](https://learn.microsoft.com/azure/databricks/repos/supported-artifact-types)). These files are a way to move notebooks between workspaces, not something to commit to Git. Without a workspace you can still read them: unzip, load the JSON, sort `commands` by `position`, and read each `command` field.

| | Azure SQL to Lakebase Migration | Lakebase Schema Setup |
|---|---|---|
| Cells | 8 (1 markdown, 7 Python) | 6 (1 markdown, 5 Python) |
| Job | Copy 26 registered Azure SQL objects into Lakebase | Create schema `mosaic_redesign` with 8 empty app tables |
| Input | Azure SQL Database over Spark JDBC | T-SQL and Delta DDL translated by hand into the code |
| Target | Lakebase project, `production` branch, `databricks_postgres` | Same project, branch and database |
| Writes | Generated DDL plus a full data reload per table | 22 DDL statements, no data |
| Verification | Row-count comparison; never fails the run | Prints table and column counts; asserts nothing |

## The migration notebook reloads 26 Azure SQL objects for a table-by-table cutover

The opening "Project Plan" cell defines the job. It moves application-serving data ("OLTP reads + writes, NOT analytical", about **3 GB** in production) from Azure SQL into a Lakebase `production` branch. The consumers are "POSIT apps — Workbench (dev), Connect (production serving)" ([migration archive, cell 1](<../context/Azure SQL to Lakebase Migration.dbc>)).

The strategy is a **"Table-by-Table Atomic Cutover"** with "No dual-write period allowed", under one rule: "For any given table, ALL readers and writers must be on the same database at the same time." The plan justifies this by noting that the apps use no cross-table joins and no foreign keys, and have low traffic. It runs in three phases:

- **Phase 1:** register a table, run the notebook, review the discovered schema, and test reads from Workbench.
- **Phase 2:** give each table a five-minute window. Apps are drained, the notebook runs "one final time", and apps are redeployed with Lakebase connection strings.
- **Phase 3:** monitor, then decommission the Azure SQL table.

That shape matches mainstream guidance, which warns that dual writes from the client are "two separate transactions" that can fail between the two commits ([Google Cloud: Database migration concepts and principles](https://docs.cloud.google.com/architecture/database-migration-concepts-principles-part-1)).

| Cell | Title | What it does |
|---|---|---|
| 1 | Project Plan (markdown) | Strategy, phases, a 12-table "Tables in Scope" list, and a pointer to the skill |
| 2 | Configuration | Source settings (an Entra user account plus a password from an Azure Key Vault-backed secret scope), the Lakebase project, branch and database, and the 26-entry `TABLE_MIGRATION_REGISTRY` |
| 3 | Dependencies | Installs `psycopg>=3.1.0` and `databricks-sdk>=0.118.0` with pip through `subprocess` if missing |
| 4 | Dependency Validation | Prints versions, queue counts and "Python restart required: no" |
| 5 | Establish Connections | Builds a JDBC URL with `Authentication=ActiveDirectoryPassword`. Lists Lakebase endpoints, takes `endpoints[0]`, gets a 1-hour OAuth token, and connects with psycopg over SSL |
| 6 | Schema Discovery and DDL Generation | Reads `INFORMATION_SCHEMA.COLUMNS` and maps 28 SQL Server types (anything else becomes `text`). Converts names to snake_case and builds `CREATE TABLE IF NOT EXISTS` with columns and types only |
| 7 | Migration Loop | For each table: JDBC read and `count()`, commits TRUNCATE or DROP/CREATE, `toPandas()`, CSV with `\N` for NULL, `COPY … FROM STDIN`, then a `COUNT(*)` check. Reconnects after 45 minutes |
| 8 | Summary Report | Displays results and counts, closes the connection, raises nothing |

**The registry controls what the notebook does.** Each entry has three fields:

- `cutover_status`: `pre_cutover`, `cutover_complete` or `paused`. Only `pre_cutover` entries are refreshed.
- `reload_mode`: `truncate` or `recreate`.
- `enabled`.

Today **all 26 entries are `pre_cutover`, `truncate` and enabled** ([migration archive, cell 2](<../context/Azure SQL to Lakebase Migration.dbc>)). They span four schemas: 12 tables in `mosaicai`, 12 in `mosaicaiderm`, and one `FIELDLENS_METRICS` table in each of `fieldlensonc` and `fieldlensderm`.

The markdown's scope table lists only the 12 `mosaicaiderm` tables, each marked "Migrated" ([migration archive, cell 1](<../context/Azure SQL to Lakebase Migration.dbc>)). Together they hold about **1.6 million rows**, led by `CALL_DETAIL_TM31` (743,400), `ADMARKET_LONG_TM31` (568,084) and `HCP_ATTRIBUTES_TM31` (213,671). Here "Migrated" means copied, not cut over.

The registry also includes objects that do not belong:

- **A view, `vw_MOSAICAI_HO_ALL`.** `INFORMATION_SCHEMA.COLUMNS` lists a view's columns just like a table's ([Microsoft: COLUMNS](https://learn.microsoft.com/en-us/sql/relational-databases/system-information-schema-views/columns-transact-sql)). It would land in Lakebase as a static table that goes stale after cutover.
- **Two backups:** `MOSAICAI_PROMPTS_EDIT_BAK_20260106` and `RTE_REQUESTS_BAK`.
- **A probable typo:** `MOASIAC_PROMPTS_TRACKING` next to `MOSAICAI_PROMPTS_TRACKING`.
- **`RTE_REQUESTS`**, which duplicates a table the setup notebook creates.

**The Lakebase side of the notebook is current and correct.** It calls `list_endpoints`, reads `status.hosts.host`, calls `generate_database_credential`, and connects with psycopg using `sslmode=require`. That is the documented SDK pattern ([Lakebase: Manage computes](https://learn.microsoft.com/en-us/azure/databricks/oltp/projects/manage-computes); [Lakebase: Authentication](https://learn.microsoft.com/en-us/azure/databricks/oltp/projects/authentication)). These Postgres APIs became **generally available on 14 August 2026** ([Lakebase release notes](https://learn.microsoft.com/en-us/azure/databricks/release-notes/lakebase/)). The notebooks use `w.postgres`, not the retired Provisioned `w.database` API. `w.postgres` first appeared in SDK 0.75.0 ([databricks-sdk changelog](https://github.com/databricks/databricks-sdk-py/blob/main/CHANGELOG.md)), but Databricks documents 0.81.0 as the minimum for this credential pattern ([Lakebase: Connect a Databricks app](https://learn.microsoft.com/azure/databricks/oltp/projects/tutorial-databricks-apps-autoscaling)). The notebooks' 0.118.0 floor is therefore stricter than the platform requires.

The type map preserves values correctly for most types:

- integers;
- `bit` to `boolean`;
- `decimal` to `numeric(p,s)`;
- `money` to `numeric(19,4)`;
- `uniqueidentifier` to `uuid`;
- `(max)` string types to `text`.

`varchar(n)` lengths cannot overflow, because SQL Server's *n* counts bytes or byte-pairs while Postgres counts characters ([Microsoft: nchar and nvarchar](https://learn.microsoft.com/en-us/sql/t-sql/data-types/nchar-and-nvarchar-transact-sql?view=sql-server-ver17); [PostgreSQL: Character types](https://www.postgresql.org/docs/current/datatype-character.html)). Using `\N` as the NULL marker keeps NULLs and empty strings distinct under COPY's CSV rules ([PostgreSQL: COPY](https://www.postgresql.org/docs/current/sql-copy.html)).

The plan's text is not a reliable description of the code: reading the two side by side turns up **16 discrepancies** ([migration archive, cells 1–4](<../context/Azure SQL to Lakebase Migration.dbc>)). The most important concerns the target itself:

- **The header says** the source is the `POSIT_DEV` database and the target is the `posit-dev-testdb` Lakebase project.
- **The configuration** reads from `POSIT`, with a stale comment saying 'Change to "POSIT" for production'. It writes to project `posit`, the same project the setup notebook uses.

A reader who follows the header would think this is a dev copy. In fact the code copies what its own comment calls production into the production branch. Other discrepancies:

- The plan says the notebook is "idempotent: DROP + CREATE", but the default mode is `truncate`.
- The plan's cell map has six cells; the notebook has eight.
- The plan says to edit `TABLES_TO_MIGRATE`, a variable that is derived from the registry and never used.
- The plan says `%pip`, but the code runs pip through `subprocess`.
- The plan says to "verify type mappings" from the discovery output, but that output prints only column renames.

## Eight high-severity defects make the migration unsafe to schedule

**The most dangerous defect is in the registry design.** Its statuses fail open: any entry without a status defaults to `pre_cutover`. The configuration comment invites people to use the list to "control scheduled sync behavior". Phase 2 never says to change a table to `cutover_complete` after its apps move ([migration archive, cells 1–2](<../context/Azure SQL to Lakebase Migration.dbc>)).

Suppose the notebook runs on a schedule and someone forgets that edit. The next run then **truncates the Lakebase table, which is now the live copy, and reloads stale Azure SQL data**. Every app write since cutover is erased, with no warning.

The load sequence makes this worse. `prepare_target_table()` commits the TRUNCATE (or DROP and CREATE) in its own transaction. Only then does the loop do a second full source read, the pandas conversion and the COPY. Postgres TRUNCATE takes an `ACCESS EXCLUSIVE` lock and "is not MVCC-safe", but it "is transaction-safe" if it stays inside the same transaction as the load ([PostgreSQL: TRUNCATE](https://www.postgresql.org/docs/current/sql-truncate.html)). By committing early, the notebook gives up that protection:

- **Readers see an empty table for the whole reload.**
- **Any later failure leaves the table empty.** The rollback only discards the failed COPY ([psycopg: COPY](https://www.psycopg.org/psycopg3/docs/basic/copy.html); [psycopg: Transactions](https://www.psycopg.org/psycopg3/docs/basic/transactions.html)).

**The data path keeps most values intact but mangles some, and row counts would not notice.** NULLs, empty strings, embedded quotes and newlines, booleans, dates, decimals and Unicode survive. The following do not:

- **Binary columns.** They reach pandas as Python `bytes` ([Databricks: Environment version 5](https://learn.microsoft.com/en-us/azure/databricks/release-notes/serverless/environment-version/five)), and the CSV writer converts non-string values with `str()` ([Python: csv](https://docs.python.org/3/library/csv.html)). The result is text such as `b'...'`. Postgres `bytea` input rejects a backslash not followed by octal digits ([PostgreSQL: Binary types](https://www.postgresql.org/docs/current/datatype-binary.html)). Tables with non-printable bytes therefore fail outright, while **values that are all printable are stored as that literal text**.
- **`rowversion`, `geography` and `geometry` columns.** These fall back to `text` and are stored as the same `b'...'` strings, without any error.
- **Large integers in columns with NULLs.** Arrow turns integer columns that contain NULLs into `float64` ([Apache Arrow: pandas integration](https://arrow.apache.org/docs/python/pandas.html)). **`bigint` values above 2^53 are rounded** before the notebook's `Int64` fix runs.
- **A literal `\N` string** becomes NULL.
- **`datetimeoffset`** loses its offset, because `timestamptz` does not keep the input time zone ([PostgreSQL: Date/time types](https://www.postgresql.org/docs/current/datatype-datetime.html)).
- **`datetime2(7)`** loses its seventh fractional digit ([Microsoft: datetime2](https://learn.microsoft.com/en-us/sql/t-sql/data-types/datetime2-transact-sql?view=sql-server-ver17)).
- **Placeholder dates outside 1677–2262** (such as 9999-12-31) cannot be represented by pandas' nanosecond timestamps ([pandas: Timestamp limitations](https://pandas.pydata.org/docs/user_guide/timeseries.html#timestamp-limitations)).

**The table structure is copied even less faithfully: the DDL generator copies names and types only.** It reads `IS_NULLABLE` and then ignores it. It carries over no primary keys, identity columns, defaults, unique or check constraints, indexes, computed-column logic or triggers. Microsoft warns against exactly this: "Don't use INFORMATION_SCHEMA views to determine the schema of an object" ([Microsoft: COLUMNS](https://learn.microsoft.com/en-us/sql/relational-databases/system-information-schema-views/columns-transact-sql)). After cutover:

- App inserts that relied on `IDENTITY`, `GETDATE()` or `NEWID()` store NULLs. Postgres fills an omitted column with "its declared default value or null if there is none" ([PostgreSQL: INSERT](https://www.postgresql.org/docs/current/sql-insert.html)).
- Any `ON CONFLICT` upsert fails, because there is no unique index for it to use.
- Lookups by key on the 743,400-row call-detail table become full table scans.

**String comparisons change too.** Azure SQL Database defaults to the case-insensitive collation `SQL_LATIN1_GENERAL_CP1_CI_AS` ([Azure SQL: Logical servers](https://learn.microsoft.com/en-us/azure/azure-sql/database/logical-servers?view=azuresql#overview)), and it treats `'abc'` and `'abc '` as equal ([Microsoft: String comparison](https://learn.microsoft.com/en-us/sql/t-sql/language-elements/string-comparison-assignment?view=sql-server-ver17)). Lakebase defaults to `C.UTF-8`, which compares strings byte for byte ([Lakebase: Postgres compatibility](https://learn.microsoft.com/azure/databricks/oltp/projects/compatibility)). A lookup by user name or e-mail in tables such as `PACE_AI_USERS` can therefore return nothing, without any error. Converting names to snake_case would also rename any mixed-case column (`PromptText` would become `prompt_text`), breaking app SQL that still uses the old name.

**Operationally, the notebook cannot fail.** Errors are caught per table and printed, and the summary never raises. A scheduled job would therefore report success even if every table failed, mismatched, or was skipped. Skips are easy to miss: an entry whose capitalization differs from the source catalog is dropped by Spark's case-sensitive filter after SQL Server's case-insensitive lookup has already found it.

**Source authentication uses a deprecated method.** The notebook uses `ActiveDirectoryPassword`. Microsoft has deprecated it as "incompatible with mandatory Microsoft Entra multifactor authentication (MFA)" and adds: "Don't use a Microsoft Entra user account as a service account" ([Microsoft: JDBC Entra authentication](https://learn.microsoft.com/en-us/sql/connect/jdbc/connecting-using-azure-active-directory-authentication?view=sql-server-ver17#connect-using-activedirectorypassword-authentication-mode)). A personal account used as a service account is exactly what the hard-coded user is. It is also unverified whether this generic JDBC path runs on serverless compute at all. Databricks' serverless documentation requires Unity Catalog for external sources and lists the bundled `sqlserver` connector, not generic JDBC, among the supported read formats ([Databricks: Serverless limitations](https://learn.microsoft.com/en-us/azure/databricks/compute/serverless/limitations); [Databricks: Spark data sources](https://learn.microsoft.com/en-us/azure/databricks/connect/spark-data-sources)).

**There is no way to target one table.** With no parameters, a Phase 2 window reloads all 26 entries, and the notebook reads each one from the source twice.

Two medium risks round this out:

- **Endpoint selection.** Endpoints are typed `ENDPOINT_TYPE_READ_WRITE` or `ENDPOINT_TYPE_READ_ONLY`, and `endpoints[0]` will send writes to a read-only endpoint once a read replica exists ([Lakebase: Manage computes](https://learn.microsoft.com/en-us/azure/databricks/oltp/projects/manage-computes)). The fix is to filter on the read-write type.
- **Reconnection.** The 45-minute reconnect is harmless, but it does nothing for the open connection, because "Token expiration is enforced only at login" ([Lakebase: Authentication](https://learn.microsoft.com/en-us/azure/databricks/oltp/projects/authentication)). Token refresh matters for each *new* connection, which is where Databricks' refresh guidance applies. Real disconnections are not handled. Idle connections close after 24 hours, a lifetime beyond 24 hours is not guaranteed, and a compute that has scaled to zero must wake before it accepts a connection. The loop therefore needs retry logic.

| ID | Defect | Consequence | Fix |
|---|---|---|---|
| H1 | Statuses default to `pre_cutover`; no step to change `cutover_status` | A scheduled run truncates a cut-over table and reloads stale data | Control table with explicit statuses; refuse to truncate a table written since the last sync; revoke the sync identity's write access on cut-over tables |
| H2 | TRUNCATE or DROP committed before COPY | Readers see empty tables; a failed load leaves the table empty | TRUNCATE, COPY with an explicit column list, and verification in one transaction; or load a staging table and swap names |
| H3 | Bytes written as `b'...'` text | `bytea` loads fail or store garbage; `rowversion` and spatial values stored as that text | psycopg `copy.write_row()` or binary COPY; explicit casts in the source query; fail on unknown types |
| H4 | No NOT NULL, keys, identity, defaults, constraints or indexes | NULL keys, failing upserts and full table scans after cutover | Generate DDL from SQL Server's `sys.*` catalog views; identity columns plus `setval` after each load; create indexes after the load |
| H5 | Errors printed, never raised; skipped tables vanish from the summary | Jobs report success on failure | Record skips; raise at the end; write results to an audit table |
| H6 | `ActiveDirectoryPassword` with a personal account | Breaks once MFA is enforced; tied to one person | Service principal or managed identity, ideally through a Unity Catalog SQL Server connection |
| H7 | No parameters; every run reloads all 26 entries | The five-minute window rewrites unrelated tables | Job parameters for table group, reload mode and dry run |
| H8 | Case- and trailing-space-insensitive matching becomes exact; mixed-case names renamed | Lookups silently miss; app SQL breaks | Per column: `citext`, `lower()` indexes or a case-insensitive ICU collation; lower-case names without renaming |

How severe H3 and H4 really are depends on facts the archive cannot show. H3 matters only if some in-scope column is binary, `rowversion`, spatial or `datetimeoffset`. H4 depends on how heavily the apps rely on identity columns and defaults. The cheapest way to re-rank both is to query `sys.columns`, `sys.types` and `sys.identity_columns` for the 26 objects.

## The schema notebook builds eight app tables, then hides its own failures

**The setup notebook's header states its purpose directly:** "Create empty tables in Lakebase Postgres for the MOSAIC Redesign app. These tables are written to directly by the app — they are **not** migrated from Azure SQL." It promises to be "Self-contained" and "Idempotent: safe to re-run" ([setup archive, cell 1](<../context/Lakebase Schema Setup.dbc>)). The remaining cells:

- **Cells 2 and 3** reuse the migration notebook's package installer and Lakebase connection.
- **Cell 4** runs 17 statements translated from a T-SQL `MOSAIC_REDESIGN` schema.
- **Cell 5** runs five statements translated from Delta tables in a development catalog's `gold_mosaic` schema.
- **Cell 6** prints table and column counts.

In total it creates **one schema, 8 tables with 136 columns, 19 indexes, 6 identity sequences, one CHECK constraint and 11 column defaults** ([setup archive, cells 4–5](<../context/Lakebase Schema Setup.dbc>)). The 19 indexes are 12 explicit ones plus 7 created automatically for primary keys. It targets the same hard-coded project, `production` branch and `databricks_postgres` database as the migration code. The table roles below are inferred from column names, since no app code was available.

| Table | Source DDL | Columns | Key | Likely role |
|---|---|---|---|---|
| `mosaic_audit_log` | T-SQL | 42 | Identity `audit_id`; unique `event_uid` | Request and event audit trail, with 11 B-tree indexes |
| `mosaic_audit_heartbeat` | T-SQL | 9 | `instance_id` | One row per running app process |
| `mosaic_audit_viewer` | T-SQL | 7 | Identity `viewer_id`; CHECK that a person is named | Who may open the audit-log page |
| `recent_search` | T-SQL | 8 | Identity `id` | Each user's recently viewed entities |
| `assistant_query_log` | Delta | 16 | Identity `log_id` | One row per AI-assistant request (natural language to SQL) |
| `assistant_query_cache` | Delta | 9 | Identity `cache_id` | Assistant response cache |
| `assistant_daily_brief` | Delta | 24 | Identity `brief_id` | AI-generated daily briefings per employee and scope |
| `rte_requests` | Delta | 21 | **None** | Queue of outbound e-mail requests, probably rep-triggered e-mail |

**Most translations preserve meaning or widen safely, but three change behaviour without saying so:**

- **`IDENTITY(1,1)` became `GENERATED ALWAYS AS IDENTITY`.** This rejects explicit IDs unless the INSERT says `OVERRIDING SYSTEM VALUE`. COPY, however, accepts explicit IDs without advancing the sequence ([PostgreSQL 17: CREATE TABLE](https://www.postgresql.org/docs/17/sql-createtable.html)). Any backfill of history therefore needs a sequence reset afterwards.
- **`SYSUTCDATETIME()` became `CURRENT_TIMESTAMP` in `timestamp` columns.** `CURRENT_TIMESTAMP` returns the transaction's start time as a time-zone-aware value ([PostgreSQL 17: Date/time functions](https://www.postgresql.org/docs/17/functions-datetime.html)). Postgres stores it as local time in the writer's session `TimeZone` ([PostgreSQL 17: Date/time types](https://www.postgresql.org/docs/17/datatype-datetime.html)). Lakebase documents no default for that setting ([Lakebase: Postgres compatibility](https://learn.microsoft.com/azure/databricks/oltp/projects/compatibility)). R's RPostgres package sets its sessions to UTC by default ([RPostgres: Postgres()](https://rpostgres.r-dbi.org/reference/Postgres.html)). psycopg, like other libpq clients, sends a time zone only when `PGTZ` is set. As a result, **columns meant to hold UTC can hold values hours apart, depending on whether an R or a Python app wrote them**.
- **`IGNORE_DUP_KEY=ON` became a plain unique index.** SQL Server skipped only the duplicate rows ([Microsoft: CREATE INDEX](https://learn.microsoft.com/sql/t-sql/statements/create-index-transact-sql)). In Postgres, every writer must add `ON CONFLICT (event_uid) DO NOTHING` ([PostgreSQL 17: INSERT](https://www.postgresql.org/docs/17/sql-insert.html)). Otherwise it loses the whole batch, on exactly the retries the deduplication exists to absorb.

On the Delta side:

- **Text lengths.** The header claims `STRING → text`, but **34 of the 37 string columns are `varchar(n)`** ([setup archive, cell 5](<../context/Lakebase Schema Setup.dbc>)). If the source columns were `STRING`, which accepts "any length" ([Databricks: STRING type](https://learn.microsoft.com/azure/databricks/sql/language-manual/data-types/string-type)), long prompts or JSON will now make inserts fail.
- **Timestamps.** Delta `TIMESTAMP` became `timestamp`, although Databricks' own Lakebase writer maps it to `timestamptz` ([Databricks: Lakebase data type mappings](https://learn.microsoft.com/azure/databricks/structured-streaming/lakebase#data-type-mappings)).
- **UUIDs.** `uuid()` → `gen_random_uuid()` is correct; the function has been built into Postgres since version 13 ([PostgreSQL 13 release notes](https://www.postgresql.org/docs/13/release-13.html)).

**The notebook's error handling defeats its own verification.** Cells 4 and 5 wrap each statement in `try/except` inside one psycopg transaction, then call `commit()` once at the end ([setup archive, cells 4–5](<../context/Lakebase Schema Setup.dbc>)). When one statement fails:

1. The server "will refuse further commands, until a rollback() is called" ([psycopg: Transactions](https://www.psycopg.org/psycopg3/docs/basic/transactions.html)).
2. Every later statement in the cell prints an error.
3. The final COMMIT does not raise. The server completes it as a ROLLBACK ([PostgreSQL source: utility.c](https://github.com/postgres/postgres/blob/REL_17_STABLE/src/backend/tcop/utility.c)), and psycopg reports that as success ([psycopg source](https://github.com/psycopg/psycopg/blob/master/psycopg/psycopg/_connection_base.py)).

**The cell finishes green, yet nothing from it persists, including statements that printed "OK".** Cell 6 then simply reports however many tables happen to exist.

The notebook's "idempotent" claim holds only narrowly. `IF NOT EXISTS` offers "no guarantee that the existing relation is anything like the one that would have been created" ([PostgreSQL 17: CREATE TABLE](https://www.postgresql.org/docs/17/sql-createtable.html)), so a changed DDL statement is silently ignored on re-run. Postgres also checks privileges before it checks whether an object exists ([PostgreSQL source: schemacmds.c](https://github.com/postgres/postgres/blob/REL_17_STABLE/src/backend/commands/schemacmds.c)). A re-run by anyone other than the original owner therefore fails, and because errors are swallowed it still looks successful.

**The notebook also leaves ownership and access unresolved.** "The table will be owned by the user issuing the command" ([PostgreSQL 17: CREATE TABLE](https://www.postgresql.org/docs/17/sql-createtable.html)). The notebook connects as the person running it and grants nothing. A new Lakebase role starts with "no database permissions" ([Lakebase: Manage database permissions](https://learn.microsoft.com/en-us/azure/databricks/oltp/projects/manage-roles-permissions)). **The Posit Connect identity therefore cannot read or write these tables until someone adds grants.** Moving ownership later requires a workaround through a temporary role, which starts by connecting as the current owner ([Lakebase: Transfer object ownership](https://learn.microsoft.com/en-us/azure/databricks/oltp/projects/transfer-object-ownership)). That has to happen before the creator leaves the company.

The table design has further gaps:

- **`rte_requests` has no key.** Duplicate requests, export-flag updates that hit several rows, and impossible `ON CONFLICT` upserts are all real risks.
- **Audit inserts are expensive.** Every insert into the audit log updates 11 B-tree indexes.
- **Nothing enforces retention.** These tables hold e-mail addresses, IP addresses and request bodies. `pg_cron` and `pg_partman` are not offered on Lakebase ([Lakebase: Extensions](https://learn.microsoft.com/azure/databricks/oltp/projects/extensions)), so purges must run as Databricks jobs.
- **`mosaic_audit_viewer` starts empty**, so nobody can open the audit page until someone adds rows.

All eight tables are empty today. Switching to `timestamptz`, adding keys and widening free-text columns therefore costs a DDL edit now, but a data migration later.

**The two notebooks complement each other.** The migration notebook keeps the legacy schema names (`mosaicai`, `mosaicaiderm`, `fieldlens*`), while the setup notebook creates new tables in `mosaic_redesign`. No names collide, and either notebook can run first.

They meet at **`rte_requests`**. The migration copies Azure SQL's `mosaicai.RTE_REQUESTS` into `mosaicai.rte_requests`, while the setup notebook creates an empty `mosaic_redesign.rte_requests` from Delta DDL ([migration archive, cell 2](<../context/Azure SQL to Lakebase Migration.dbc>); [setup archive, cell 5](<../context/Lakebase Schema Setup.dbc>)). If the legacy app and the redesigned app write to different copies, an exporter that reads only one copy will miss requests. The team must pick one copy as the system of record.

"Not migrated" also means that existing rows in the Azure SQL `MOSAIC_REDESIGN` schema stay behind unless someone deliberately backfills them. That includes the audit history and the audit-viewer access list.

## Once hardened, each notebook has a distinct role in the cutover

**Both notebooks should first be converted out of the archive format.**

- **Import and convert.** Import each `.dbc` into a Git folder (`databricks workspace import … --format DBC`). Export it either as `.ipynb`, which keeps cell titles and the environment ([Databricks: Manage notebook format](https://learn.microsoft.com/azure/databricks/notebooks/notebook-format)), or as `.py` source for cleaner diffs ([Databricks CLI: workspace commands](https://learn.microsoft.com/azure/databricks/dev-tools/cli/reference/workspace-commands)). Commit the result, and export SKILL.md separately.
- **Make functions testable.** Pure functions such as `map_sql_type` and `to_snake_case` belong in an importable module with unit tests. Databricks recommends keeping functions and their tests outside notebooks ([Databricks: Unit testing for notebooks](https://learn.microsoft.com/azure/databricks/notebooks/testing#organize-functions-and-unit-tests)).
- **Parameterize.** Hard-coded hosts, project, branch and registry should move into bundle variables and job parameters. Deploy them with Declarative Automation Bundles, the March 2026 rename of Databricks Asset Bundles, which keeps the same CLI and YAML ([Databricks: Bundles release notes](https://learn.microsoft.com/azure/databricks/release-notes/dev-tools/bundles)).
- **Pin dependencies.** Replace the runtime pip calls with pinned versions in the serverless environment. Databricks recommends "moving away from unpinned `%pip install` commands" on serverless compute ([Databricks: Notebook-scoped libraries](https://learn.microsoft.com/en-us/azure/databricks/libraries/notebooks-python-libraries)). It also says an upgraded SDK needs a Python restart before it takes effect ([Databricks SDK for Python](https://learn.microsoft.com/en-us/azure/databricks/dev-tools/sdk-python)), which the current cells claim they do not need. Replace plain `psycopg` with `psycopg[binary]`, because the pure-Python build needs a local libpq library ([psycopg: Installation](https://www.psycopg.org/psycopg3/docs/basic/install.html)).
- **Run safely.** Jobs should run as a service principal ([Databricks: Bundle run_as](https://learn.microsoft.com/azure/databricks/dev-tools/bundles/run-as)) with a matching Lakebase OAuth role, which is named by the principal's client ID ([Lakebase: Create Postgres roles](https://learn.microsoft.com/en-us/azure/databricks/oltp/projects/postgres-roles)). They should keep Databricks' default of one concurrent run per job ([Databricks: Run a job](https://learn.microsoft.com/azure/databricks/jobs/run-now)) and select the `ENDPOINT_TYPE_READ_WRITE` endpoint explicitly. Each job should end with an assertion that fails the run when anything went wrong.

### The migration notebook becomes a refresh job and a cutover runner

Fix the notebook in this priority order:

1. Make cutover control fail closed.
2. Make each table's load a single transaction.
3. Use a type-safe data path, such as psycopg's `copy.write_row()`, which adapts values "as they would be in normal querying" ([psycopg: COPY](https://www.psycopg.org/psycopg3/docs/basic/copy.html)).
4. Generate DDL from the SQL Server catalog.
5. Make failures fail the run.
6. Add parameters.
7. Authenticate to the source through a Unity Catalog SQL Server connection, which supports Entra service principals ([Databricks: SQL Server federation with Entra ID](https://learn.microsoft.com/azure/databricks/query-federation/sql-server-entra)).

After that, the notebook supports three uses:

- **Pre-cutover refresh.** It should write to a Lakebase child branch rather than `production`. Branches are instant copy-on-write clones, and their changes never reach the parent ([Lakebase: Branches](https://learn.microsoft.com/en-us/azure/databricks/oltp/projects/branches)).
- **Cutover runner.** It should load one app's tables per run, in a single transaction, and then change their status in a control table. The unit is an app's tables, not one table, because changing an app's connection string moves every table that app touches at once.
- **Template for future Azure SQL moves.** Its type map, DDL extraction and loader belong in a shared Python module, which the Genie Code skill references instead of restating.

**The runbook around the notebook needs controls the plan omits:**

- **Enforce the write freeze.** "Stop/drain apps" should become `DENY INSERT, UPDATE, DELETE` on each table for every account that writes to it. Note that "DENY does not apply to object owners or members of the sysadmin fixed server role" ([Microsoft: DENY](https://learn.microsoft.com/sql/t-sql/statements/deny-transact-sql?view=sql-server-ver17)), so those accounts need separate handling.
- **Prove no writes arrived late.** Use Change Tracking's `CHANGETABLE(CHANGES …)` from the version recorded at the final copy ([Microsoft: CHANGETABLE](https://learn.microsoft.com/sql/relational-databases/system-functions/changetable-transact-sql?view=sql-server-ver17)).
- **Validate more than row counts.** Read both databases into Spark, hash every row by key, and compare the hashes. Add a handful of "golden" app queries run against both databases, to catch collation changes that no data comparison can see.
- **Prepare rollback.** Rehearse a reverse copy from Lakebase to Azure SQL and set a time limit after which rollback is no longer an option. For the migration period, raise Azure SQL's point-in-time restore window from its seven-day default toward the 35-day maximum ([Azure SQL: Automated backups](https://learn.microsoft.com/azure/azure-sql/database/automated-backups-overview?view=azuresql)).
- **Plan app authentication.** The built-in PgBouncer connection pooler does not accept OAuth. Password connections, the alternative, are disabled by default for projects created since 21 May 2026 ([Lakebase: Authentication](https://learn.microsoft.com/en-us/azure/databricks/oltp/projects/authentication); [Lakebase release notes](https://learn.microsoft.com/en-us/azure/databricks/release-notes/lakebase/)).

**No packaged migration tool is a better starting point than this notebook:**

- Azure Database Migration Service lists no SQL Server-to-PostgreSQL scenario ([Azure DMS: Supported scenarios](https://learn.microsoft.com/azure/dms/resource-scenario-status)).
- Lakebridge converts code only into Databricks SQL, Spark SQL or Python notebooks ([Lakebridge: Transpile](https://databrickslabs.github.io/lakebridge/docs/transpile/)).
- pgloader's latest stable release dates from October 2024 ([pgloader releases](https://github.com/dimitri/pgloader/releases)). It also documents only username-and-password SQL authentication ([pgloader: MS SQL](https://pgloader.readthedocs.io/en/latest/ref/mssql.html)), while the source currently signs in with Entra.

Platform features can still shrink the notebook's scope. Read-mostly reference tables such as call detail, HCP attributes and ad-market data may already exist in the lakehouse. If so, Snapshot synced tables could load them at "up to 2,000 rows per second per CU", about six minutes for 743,400 rows on one capacity unit (CU). Synced tables are read-only in Postgres: only reads, index creation and `DROP TABLE` are allowed. They are owned by their sync pipeline, and each needs a primary key ([Lakebase: Synced tables](https://learn.microsoft.com/en-us/azure/databricks/oltp/projects/sync-tables)). The notebook therefore remains the path for tables that apps write to. Lakebase Change Data Feed, in Public Preview since May 2026 ([Lakebase release notes](https://learn.microsoft.com/en-us/azure/databricks/release-notes/lakebase/)), can stream those app-written tables back into Delta for analytics ([Lakebase: Change Data Feed](https://learn.microsoft.com/en-us/azure/databricks/oltp/projects/lakebase-cdf)). Databricks' own Lakebase agent skill calls this feature "Lakehouse Sync" and labels it Beta, so confirm its status in the workspace before relying on it.

### The schema notebook becomes migration V0001

**The setup notebook should run once per branch, to bootstrap the schema.** Then fix its 22 statements while the tables are empty. Keep them as the first migration ("V0001") in a SQL-first versioned migration tool such as dbmate, Sqitch or Flyway, or Alembic if the Python apps already use SQLAlchemy.

Fix the error handling as follows:

- Run all DDL inside `with conn.transaction():`, which rolls back and raises on any failure ([psycopg: Transactions](https://www.psycopg.org/psycopg3/docs/basic/transactions.html)).
- Set a short `lock_timeout` first.
- End with an assertion that checks the expected tables and columns in the catalog.

Run the bootstrap as a Databricks **group** role. Lakebase lets every member of a group log in as that role with their own token ([Lakebase: Create Postgres roles](https://learn.microsoft.com/en-us/azure/databricks/oltp/projects/postgres-roles)), so ownership does not depend on one person. After it, run a grant script that gives read-write and read-only roles (roles that cannot log in themselves) usage on the schema, insert/update/delete/select on the tables, and default privileges for future tables. Then add the first rows to the audit-viewer list.

Every later change becomes a numbered migration. Test it on a branch forked from production that expires after two to four hours ([Lakebase: Manage branches](https://learn.microsoft.com/azure/databricks/oltp/projects/manage-branches)). Then apply it to production by running it again there, because in Lakebase "Promotion is not automatic" ([Lakebase: Branch-based development workflow](https://learn.microsoft.com/azure/databricks/oltp/projects/dev-workflow-tutorial)).

**The skill both notebooks rely on should move out of one person's home folder.** Put it in `Workspace/.assistant/skills/` or a registered Git folder, where Genie Code can load it for the whole team ([Databricks: Extend Genie Code with agent skills](https://learn.microsoft.com/azure/databricks/genie-code/skills)). Genie Code is the March 2026 rename of the Databricks Assistant ([Databricks release notes, March 2026](https://learn.microsoft.com/azure/databricks/release-notes/product/2026/march#databricks-assistant-is-now-genie-code)).

The skill suits design-time work, such as reviewing type mappings and drafting registry entries or migrations. It does not suit unattended data movement. Genie Code's Agent mode runs code and sends table data samples and cell outputs to the model ([Databricks: Genie Code Agent mode](https://learn.microsoft.com/azure/databricks/genie-code/agent-mode)). With HCP (healthcare professional) commercial data involved, using it is a governance decision.

## Conclusion

The notebooks get the hard architectural decisions right: short scheduled downtime instead of dual writes, a per-table registry, and the current GA Lakebase API. Where they go wrong is in what they silently carry over from SQL Server, and **almost every serious failure is silent**:

- a live table truncated because of a forgotten status flag;
- a table left empty by a failed load;
- binary data stored as `b'...'` text;
- NULL identity keys;
- lookups that differ only in letter case returning nothing;
- a timestamp default that depends on whether R or Python wrote the row;
- a DDL run that rolls back while printing "OK".

The only check either notebook performs, counting rows or tables, catches none of these. Readiness should therefore be judged by whether Lakebase *behaves* like Azure SQL for each app, not by whether the copy ran.

Two facts make now the cheapest time to act:

- **Every registry entry is still `pre_cutover`.** No table has yet reached the point where H1 can destroy app writes.
- **Every `mosaic_redesign` table is still empty.** Fixing keys and types costs a DDL edit rather than a data migration.

The metadata suggests both notebooks were captured in one pass as a proof of concept, alongside a personal skill. The decisive step is to move all three into a reviewed, parameterized, version-controlled repository before anyone puts them on a schedule.

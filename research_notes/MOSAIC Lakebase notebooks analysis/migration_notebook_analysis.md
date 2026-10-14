# Azure SQL to Lakebase Migration notebook: contents, workflow fit, value fidelity and defects (as of 2026-10-07)

Evidence conventions: "[Cn]" links cite the notebook archive `context/Azure SQL to Lakebase Migration.dbc`, where cell n is the command with `position` n.0. A readable dump of that archive (metadata plus full source) sits in the session scratchpad. "[Setup Cn]" cites `context/Lakebase Schema Setup.dbc`. The docs were read on 2026-10-06/07 at these versions:

- Apache Spark / PySpark 4.2.0
- PostgreSQL 18.6. The behaviours cited have been stable for many major versions; Lakebase examples create PG 17 projects.
- psycopg 3.3.7.dev1
- pandas 3.0.6 (serverless environment 5 actually ships pandas 2.2.3)
- Python 3.14.8 (environment 5 ships Python 3.12.3)
- Microsoft Learn: SQL Server "ver17" pages, Azure SQL and Azure Databricks pages

Nothing was executed against any database or workspace. One local, stdlib-only CSV-serialization probe was run with Python 3.14.7 ([probe]).

## Q1. What does each of the 8 cells do, what configuration/prerequisites does it need, what does it output, and what does the notebook-level metadata reveal?

### Takeaway
The notebook has 1 markdown cell (the project plan) and 7 Python cells. The code cells do the following:
- Configure a 26-entry table registry.
- Install and validate psycopg 3 and databricks-sdk without restarting Python.
- Open a Spark JDBC source using Entra *password* authentication, and a psycopg connection to a Lakebase endpoint using a 1-hour OAuth token.
- Generate column-only Postgres DDL from `INFORMATION_SCHEMA.COLUMNS`.
- Reload every queued table (TRUNCATE or DROP/CREATE, then CSV COPY) and compare row counts.

The metadata shows no widgets (so no job parameters), no saved compute preference, `.ipynb` storage, and a serverless environment-version-5 marker. No outputs are embedded, so the archive contains no evidence of which compute ran it or what it produced.

### Cited Findings

**Notebook-level metadata**
- Keys and values:
  - `version: "NotebookV1"`, `name: "Azure SQL to Lakebase Migration"`, `language: "python"`, `origId: 296454260807200`, `guid` present.
  - `dashboards: []`, `globalVars: {}`, `inputWidgets: {}`.
  - `notebookMetadata: {"pythonIndentUnit": 4}`, `iPythonMetadata: {"nbformat": 4, ...}`, `reposExportFormat: "JUPYTER"`.
  - `environmentMetadata: {"client": "5"}`.
  - `computePreferences: null`, `inputWidgetPreferences: null`, `environmentSerializationPreferences: null`.
  - "Cell count: 8", "Cells with embedded results/outputs: 0".

  — [C1–C8 metadata][dump]
- Fields common to every cell:
  - `commandType: "auto"`, `state: "finished"`, `resultDbfsStatus: "INLINED_IN_TREE"`.
  - `startTime`, `submitTime` and `finishTime` all `0`.
  - `latestUser: "a user"`, `latestUserId: null`, `commentThread: []`, `showCommandTitle: true`.
  - Titles: "Project Plan", "Configuration", "Dependencies", "Dependency Validation", "Establish Connections", "Schema Discovery and DDL Generation", "Migration Loop", "Summary Report".

  — [dump]
- Command origIds are consecutive: 6543445814429833–…840 here and 6543445814429849–…854 in the schema-setup notebook. That notebook's own origId is 296454260807201, one more than this notebook's. — [dump]; [setup]
- Azure Databricks: "By default, new notebooks are saved in `.ipynb` format. If your notebook is saved in source format, serverless metadata might not be captured correctly". — [Serverless limitations][dbx-serverless-limits]
- A serverless notebook's environment exports as `environment_version = "5"` for "Standard v5". — [Configure the serverless environment][dbx-serverless-deps]
- Contents of serverless **environment version 5**:
  - Ubuntu 24.04.3, Python 3.12.3, Databricks Connect 18, JDK 17.
  - Standard base environment libraries: pandas 2.2.3, pyarrow 21.0.0, numpy 2.1.3, **databricks-sdk 0.67.0** and **psycopg2 2.9.11**. psycopg 3 is not listed.
  - "BinaryType consistently maps to Python bytes."
  - The Py4J gateway is disabled for `dbutils`.

  — [Environment version 5][dbx-env5]
- Release dates on PyPI:

  | databricks-sdk version | Released |
  |---|---|
  | 0.67.0 | 2025-09-25 |
  | 0.79.0 | 2026-01-22 |
  | 0.118.0 | 2026-06-18 |
  | 0.147.0 (latest) | 2026-10-06 |

  — [PyPI release history][pypi-sdk]

**Cell 1: markdown "Project Plan"**
- **Overview table:**
  - Source: Azure SQL Database, database `POSIT_DEV`.
  - Target: Lakebase project `posit-dev-testdb`, branch `production`, with a hard-coded endpoint hostname on port 5432.
  - Database `databricks_postgres`.
  - "Preserves source schema (e.g., `mosaicaiderm` -> `mosaicaiderm`)".
  - Consumers: "POSIT apps — Workbench (dev), Connect (production serving)".
  - "Application-serving (OLTP reads + writes), NOT analytical".
  - "~3 GB total across all tables (prod)".
  - Auth: "OAuth tokens via `databricks-sdk` → `w.postgres.generate_database_credential()` (1-hour lifetime)".

  — [C1][dump]
- **Strategy:**
  - "Table-by-Table Atomic Cutover".
  - "Apps both READ and WRITE to these tables. No dual-write period allowed."
  - Rule: "For any given table, ALL readers and writers must be on the same database at the same time."
  - Feasibility rests on: "No cross-table joins in apps", "No foreign keys", "Low-traffic apps".

  — [C1][dump]
- **Phases:**
  - Phase 1 (Prep): add the table to `TABLES_TO_MIGRATE` "in Cell 2" → run the notebook → "Review the Schema Discovery output (Cell 5) to verify type mappings and column renames" → test reads from Workbench → re-run ("notebook is idempotent: DROP + CREATE").
  - Phase 2 (5-minute window per table): stop/drain the apps → "Run this notebook one final time (catches any writes since last sync)" → deploy the apps with Lakebase connection strings → enable the apps.
  - Phase 3: monitor, then decommission the table from Azure SQL.

  — [C1][dump]
- **"This Notebook" table:** six rows numbered 1–6 (Configuration, Dependencies "`%pip install` … (run once per session)", Establish Connections, Schema Discovery, Migration Loop "JDBC read → DROP/CREATE → buffer COPY → verify row count", Summary Report). "No manual DDL translation or column mapping needed." — [C1][dump]
- **Skill file:** `/Users/<user>/.assistant/skills/azure-sql-to-lakebase/SKILL.md` is described as holding:
  - a T-SQL→Postgres type map, case-folding rules, the connection pattern, buffer- and row-by-row COPY patterns, the migration loop, token refresh, the cutover workflow, and "10 documented gotchas from the POC";
  - the claim: "The Databricks Assistant reads this skill automatically".

  — [C1][dump]
- **"Tables in Scope":** 12 rows, all in `mosaicaiderm`, each marked "Migrated (N rows)":

  | Table | Rows |
  |---|---|
  | COMPLIANCE_CONFIRMATIONS | 5,729 |
  | PROMPTS_EDIT | 42 |
  | PROMPTS_TRACKING | 14,692 |
  | CALL_DETAIL_TM31 | 743,400 |
  | PROMPTS | 0 |
  | DERM_ALERTS_TM31 | 40,241 |
  | HCP_ATTRIBUTES_TM31 | 213,671 |
  | ADMARKET_LONG_TM31 | 568,084 |
  | PACE_AI_USERS | 15 |
  | PACE_AI_KNOWN_USERS | 4 |
  | PROMPTS_EDIT_IMPERSONATE | 7,000 |
  | PROMPTS_FAVORITES | 986 |

  — [C1][dump]

**Cell 2: Configuration**
- `JDBC_CONFIG` holds:
  - the server hostname;
  - `"database": "POSIT",          # Change to "POSIT" for production`;
  - port 1433;
  - a hard-coded Entra UPN in `user`;
  - `secret_scope: "akv-delta-access"` and `secret_key: "bis-azure-pwd"`.

  `LAKEBASE_CONFIG = {"project_id": "posit", "branch": "production", "database": "databricks_postgres"}`. — [C2][dump]
- `TABLE_MIGRATION_REGISTRY` has 26 dicts. Each has `source_table`, `cutover_status`, `reload_mode` and `enabled`, and all 26 are `"pre_cutover"`, `"truncate"`, `True`. By schema:
  - 12 in `mosaicai`, including `vw_MOSAICAI_HO_ALL`, `MOSAICAI_PROMPTS_EDIT_BAK_20260106`, `RTE_REQUESTS_BAK`, `RTE_REQUESTS`, and both `MOASIAC_PROMPTS_TRACKING` and `MOSAICAI_PROMPTS_TRACKING`;
  - 1 in `fieldlensonc`;
  - 1 in `fieldlensderm`;
  - 12 in `mosaicaiderm`.

  — [C2][dump]
- Registry semantics are documented only in comments:
  - "`pre_cutover` tables are refreshed from Azure SQL; anything else is skipped".
  - `truncate` "keeps the target table object and reloads the data"; `recreate` "drops and recreates it".
  - "Edit this list to control scheduled sync behavior."

  — [C2][dump]
- `normalize_table_registry_entry`:
  - accepts bare strings, which become `pre_cutover`/`truncate`/`enabled`;
  - defaults missing `cutover_status` to `"pre_cutover"` and missing `reload_mode` to `"truncate"`;
  - lower-cases both values;
  - enforces exactly one dot in `source_table`;
  - validates the values against `{"pre_cutover","cutover_complete","paused"}` and `{"truncate","recreate"}`.

  It then derives `TABLE_REGISTRY`, `ACTIVE_TABLE_REGISTRY`, `MIGRATION_TABLE_CONFIGS`, `SKIPPED_TABLE_CONFIGS` and `TABLES_TO_MIGRATE`. `TABLES_TO_MIGRATE` is never used by later cells. Output: four counts, which are 26/26/26/0 with the current registry. — [C2][dump]

**Cell 3: Dependencies**
- Checks `importlib.metadata` versions against `psycopg>=3.1.0` and `databricks-sdk>=0.118.0`. If either is missing or older, it runs `subprocess.check_call([sys.executable, "-m", "pip", "install", "--quiet", *packages_to_install])`.
- The comment reads "Install only when needed; no Python restart required". The cell prints the installed versions.

— [C3][dump]

**Cell 4: Dependency Validation** (this cell is not in the markdown's cell table)
- Imports `pandas`, `psycopg` and `WorkspaceClient`, prints versions via `md.version()`, then prints "Python restart required: no" and the queue counts.

— [C4][dump]

**Cell 5: Establish Connections**
- **JDBC URL:** `jdbc:sqlserver://{host}:{port};database={db};Authentication=ActiveDirectoryPassword;`.
- **Connection properties:** `user`, a password from `dbutils.secrets.get(...)`, and `driver: com.microsoft.sqlserver.jdbc.SQLServerDriver`.
- **Lakebase connection steps:**
  1. `w = WorkspaceClient()`.
  2. `endpoints = list(w.postgres.list_endpoints(parent=f"projects/{project_id}/branches/{branch}"))`, then `ep = endpoints[0]` and `host = ep.status.hosts.host`.
  3. `username = w.current_user.me().user_name`.
  4. `token = w.postgres.generate_database_credential(endpoint=ep.name).token`, recording `token_issued_at = time.time()`.
  5. `psycopg.connect(host=…, port=5432, dbname=…, user=username, password=token, sslmode="require")`.
- No Spark JDBC call happens here; Azure SQL is first contacted in Cell 6. Output: source/target host and the "Connected as" user.

— [C5][dump]

**Cell 6: Schema Discovery and DDL Generation**
- **`map_sql_type`:** a 28-key dict covering varchar/nvarchar, char/nchar, the integer types, bit, decimal/numeric, float/real, the datetime family, date, time, uniqueidentifier, the binary types, money/smallmoney, xml, text/ntext. Anything else falls back to `"text"`.
- **Naming helpers:**
  - `to_snake_case` is two regexes plus `.lower()`.
  - `generate_target_name` lower-cases the schema and snake-cases the table.
- **Catalog query:** one JDBC subquery over `INFORMATION_SCHEMA.COLUMNS`, filtered by an OR of `(TABLE_SCHEMA = '{schema}' AND TABLE_NAME = '{table}')` literals. It selects `IS_NULLABLE`, which is never used.
- **Per-table processing:**
  - A Spark `.filter(f"TABLE_SCHEMA = '{schema}' AND TABLE_NAME = '{table}'").orderBy("ORDINAL_POSITION").collect()` selects the columns.
  - If nothing matches, it prints "⚠️ … No columns found in INFORMATION_SCHEMA. Skipping." and `continue`s.
  - Otherwise it builds `CREATE TABLE IF NOT EXISTS {schema.table} (col type, …)` with no constraints, plus a source→target `column_map`.
- Output: per table "✓ source → target (N columns, reload_mode=…)" and the column-name mapping. **Types and DDL are not printed.** It ends with "N table(s) ready for migration."

— [C6][dump]

**Cell 7: Migration Loop**
- **`refresh_connection_if_needed()`:** if more than 2,700 s (45 min) have passed since the token was issued, it closes `conn`, mints a new token and reconnects.
- **`prepare_target_table()`:**
  - `recreate` → `DROP TABLE IF EXISTS`, CREATE, `commit()`.
  - `truncate` → `CREATE TABLE IF NOT EXISTS`, `TRUNCATE TABLE`, `commit()`.
- **`migrate_table()`:**
  1. `spark.read.jdbc(url, table=source, properties)`, then `source_count = source_df.count()`.
  2. Token refresh check.
  3. `prepare_target_table` (commits).
  4. `pdf = source_df.toPandas()`.
  5. Rename the columns via `column_map`.
  6. A loop over datetime columns: `pdf[col].where(pdf[col].notna(), None)`.
  7. Integer columns that arrive as `float64` are cast to `"Int64"`.
  8. `pdf.to_csv(StringIO, index=False, header=False, na_rep="\\N", quoting=csv.QUOTE_MINIMAL)`.
  9. `cur.copy("COPY {target} FROM STDIN WITH (FORMAT csv, NULL '\N')")`, written in 8,192-char chunks, then `commit()`.
  10. `SELECT COUNT(*)`, compare to `source_count`, and set "success" or "mismatch".
- **Main block:**
  - `CREATE SCHEMA IF NOT EXISTS` for each target schema, then commit. This runs outside any try.
  - A per-table `try/except Exception` that calls `conn.rollback()` and records `status: "failed"`.
  - Results go to a list. Output is per-step progress lines.

— [C7][dump]

**Cell 8: Summary Report**
- `display(pd.DataFrame(results))`, then counts (processed/succeeded/mismatched/failed/skipped by cutover status), the problem rows, and `conn.close()`. Nothing is raised on failures or mismatches.

— [C8][dump]

**Prerequisites and permissions, from the docs**
- **Compute and source connector:**
  - "Databricks Runtime includes bundled connectors for common databases such as PostgreSQL, SQL Server, MySQL…". — [Connect to external databases][dbx-federation]
  - On serverless, "You must use Unity Catalog to connect to external data sources". The serverless read-source list names `SQLSERVER` and `POSTGRESQL` but not generic `JDBC`. — [Serverless limitations][dbx-serverless-limits]
  - A Unity Catalog JDBC connection (`.option("databricks.connection", …)`) is "supported across Unity Catalog compute types, including serverless". It requires DBR 17.3 LTS+ on standard/dedicated clusters and is GA from DBR 19. — [JDBC connection][dbx-jdbc-conn]; [Spark data sources][dbx-spark-sources]
  - Bundled connectors are called by format name (`"sqlserver"`, `"postgresql"`). "Reads and writes are supported on dedicated and standard compute", and "Writes on serverless compute are supported for PostgreSQL, SQL Server, MySQL, Snowflake, and Redshift". "Custom third-party Spark JARs are not supported on Databricks SQL, serverless, or standard access mode compute. For those compute types, use bundled connectors or JDBC Unity Catalog connections." — [Spark data sources][dbx-spark-sources]
  - Limits of a UC JDBC connection:
    - "The memory limit for the JDBC driver is 400 MiB".
    - No arbitrary DML: reads, appends and whole-table overwrites only.
    - "The target database system and the Azure Databricks workspace cannot be in the same VNet".
    - SQL Server identifiers in `query` need `[...]` quoting.

    — [JDBC connection][dbx-jdbc-conn]
- **Source permissions:** `INFORMATION_SCHEMA.COLUMNS` "Returns one row for each column that can be accessed by the current user in the current database". The Entra account therefore needs object-level access to each table and view, or it silently sees no columns. — [COLUMNS (Transact-SQL)][ms-infoschema-columns]
- **Secrets:** Databricks recommends storing JDBC credentials in secrets and shows `dbutils.secrets.get(scope=…, key=…)`. The notebook follows that pattern. — [Query databases using JDBC][dbx-jdbc]
- **Lakebase database access:**
  - OAuth tokens expire after one hour, and "Token expiration is enforced only at login".
  - SSL is required.
  - "The project owner's OAuth role is created automatically. To enable OAuth authentication for other Azure Databricks identities, you must create their Postgres roles using the `databricks_auth` extension".

  — [Lakebase authentication][lb-auth]
- **Lakebase project permissions:** the grantable levels are `CAN_USE` and `CAN_MANAGE`. Managing roles requires CAN MANAGE or CAN USE. — [Lakebase API guide][lb-api]; [Manage roles][lb-manage-roles]
- **Role behaviour:**
  - `databricks_create_role()` creates a role with LOGIN only; privileges must be granted separately.
  - A service principal's role name is its application ID (UUID).
  - The project owner's role owns `databricks_postgres` and is a member of `databricks_superuser`.

  — [Create Postgres roles][lb-pg-roles]
- **Python packages:**
  - A pure-Python `pip install psycopg` "still requires the libpq library installed locally"; `psycopg[binary]` bundles it. — [psycopg install][psycopg-install]
  - The Lakebase token-rotation example installs `psycopg[binary,pool]`. — [Lakebase authentication][lb-auth]

### Inferences
- The consecutive notebook and command origIds suggest the two notebooks were created or imported together in a single operation (for example an import of a folder or archive).
- `latestUser: "a user"` with a null id, together with zeroed timestamps, suggests the export anonymised or stripped run history.
- `environmentMetadata.client = "5"` most likely records serverless environment version 5. The export format names it `environment_version`, and `client` appears to be the older field name.
- If so, a fresh session finds databricks-sdk 0.67.0 (below the 0.118.0 floor) and no psycopg 3, so Cell 3 installs on every new session.
- `reposExportFormat: "JUPYTER"` means the notebook is stored as `.ipynb` (consistent with the serverless guidance above).
- `inputWidgets: {}` means the notebook cannot receive job/task parameters as written. Selecting tables requires editing code.
- The required `databricks-sdk>=0.118.0` (released 2026-06-18) and the `_BAK_20260106` registry entry imply the code was last edited after mid-June 2026. The "Tables in Scope" row counts are an older static snapshot.
- The exactly-round 7,000 rows for `MOSAICAI_PROMPTS_EDIT_IMPERSONATE` is worth a sanity check against the source; it may be coincidence.

### Gaps
- No cell outputs are embedded, so the following are unknown:
  - which compute ran the notebook (serverless or a classic DBR cluster);
  - actual run durations, peak memory and errors;
  - whether the typo table exists.
- Whether `spark.read.jdbc(...)` with an explicit `driver` class runs on serverless environment 5 is not established by the docs. This is generic JDBC: neither the bundled `sqlserver` format nor a UC connection.
  - The docs list supported read formats without `jdbc`. The documented serverless paths are the bundled `sqlserver` connector or a UC JDBC connection.
  - If the notebook ran on serverless, as the env-5 marker suggests, it worked despite this. If it ran on a classic dedicated cluster, the marker is incidental.
- The SKILL.md file and its "10 documented gotchas" are not in the archive and could not be reviewed.
- Which Lakebase project permission level is minimally required for `list_endpoints`/`generate_database_credential` was not confirmed.
- Network prerequisites were not checked: the Azure SQL firewall or private endpoint for classic VNet or serverless egress.

## Q2. What workflow is it built for (registry fields, Phase 1/2/3), and does the code implement what the markdown describes?

### Takeaway
The code implements a repeatable **full-refresh copy** of every enabled `pre_cutover` table, gated by a hand-edited Python registry. That is adequate for Phase 1 refreshes and as a Phase 2 "final copy". However:
- it cannot target a single table;
- it has no safeguard against reloading a table that has already been cut over;
- there is no Phase 3 tooling;
- several documented steps (cell numbers, the variable to edit, DROP+CREATE, `%pip`, "verify type mappings") no longer match the code.

### Cited Findings
- **Registry fields:**
  - `cutover_status` selects which tables are refreshed (`pre_cutover` only); `cutover_complete`/`paused` are skipped and listed.
  - `reload_mode` picks TRUNCATE+reload vs DROP+CREATE.
  - `enabled=False` leaves an entry inert.
  - Missing fields default to `pre_cutover`/`truncate`/`enabled`.

  — [C2][dump]
- **Phase 2 says** "Run this notebook one final time (catches any writes since last sync)". The code reloads **all** enabled `pre_cutover` entries (26 today), each with two full source reads (`count()` and then `toPandas()`). — [C1][dump]; [C7][dump]
- **The Phase 2 procedure has no step to set `cutover_status` to `cutover_complete`** after the apps switch. Only the Cell 2 comment explains that non-`pre_cutover` tables are skipped. — [C1][dump]; [C2][dump]
- **Truncate mode**, the default for all entries, does `TRUNCATE TABLE {target}` and then reloads from Azure SQL. — [C7][dump]
- **No widgets or parameters exist** (`inputWidgets: {}`). — [dump]
- **Phase 1 step 3** says to verify type mappings from the Schema Discovery output, but Cell 6 prints only `src → tgt` column names; neither the DDL nor the Postgres types are printed. — [C6][dump]
- **The registry includes a view, `mosaicai.vw_MOSAICAI_HO_ALL`.** `INFORMATION_SCHEMA.COLUMNS` covers columns of objects the user can access, which includes views. The notebook would create and fill a static table from it. — [C2][dump]; [COLUMNS (Transact-SQL)][ms-infoschema-columns]
- **Lakebase branches are a better place to test.** They are "instant, isolated copies of your database for development, testing, or recovery", copy-on-write, and "changes you make to a child branch don't affect its parent". Snapshots and point-in-time restore (2–30 day window, default 7) provide restore points. Manual snapshots are allowed on root branches only, up to 10 per project, the snapshot API is Beta, and snapshot storage is billable from 2026-06-01. — [Branches][lb-branches]; [Snapshots][lb-snapshots]; [Point-in-time restore][lb-pitr]
- **Apps need an authentication approach that survives token expiry.** OAuth suits apps "that can implement token rotation". Native password roles suit "Applications that cannot refresh credentials hourly", but "Password connections are disabled by default for new Lakebase projects". The built-in PgBouncer pooler "does not support OAuth authentication". — [Lakebase authentication][lb-auth]

### Inferences
- **High risk (H1):** the safety of a cut-over table depends on someone remembering to edit a Python literal, and the defaults fail open. If the notebook is scheduled ("scheduled sync behavior") and a table is cut over without flipping its status, the next run will TRUNCATE the now-authoritative Lakebase table and reload stale Azure SQL data. Every write the apps made since cutover would be lost, with no warning. Nothing checks whether the target has received writes since the last sync.
- **Phase 2 timing:** in a 5-minute window, a full 26-table run (two reads of 743K- and 568K-row tables, plus pandas/CSV/COPY) is unlikely to be the fastest path. It also rewrites tables unrelated to the one being cut over. A single-table run is needed.
- **The view breaks the independence assumption.** By its name, `vw_MOSAICAI_HO_ALL` probably combines the `MOSAICAI_HO1/2/3_DATA` tables, so the "tables are independent" premise does not hold for those four objects. They must be cut over together, or the view recreated as a Postgres view.
- **Missing workflow pieces:**
  - no reverse path (Lakebase → Azure SQL) if a cut-over table must roll back after apps have written to Lakebase;
  - no detection of a straggler app still writing to Azure SQL after cutover.
- **Testing on `production` exposes testers to the reload window.** Phase 1 testing and refreshes write to the `production` branch, and Workbench testers can observe the empty-table windows described in Q6. A child branch would isolate testing.
- **Cross-reference to the schema-setup notebook:**
  - It targets the same project/branch/database and creates `mosaic_redesign.rte_requests`, which it says is "written to directly by the app — … **not** migrated from Azure SQL", sourced from Delta DDL in `bis_dev.gold_mosaic`.
  - This notebook separately migrates `mosaicai.RTE_REQUESTS` and `mosaicai.RTE_REQUESTS_BAK`.
  - Two `rte_requests` tables in one database invite apps to point at the wrong one. — [Setup C1, C5][setup]; [C2][dump]

### Gaps
- Which apps touch which tables could not be checked; this determines whether "table-by-table" cutover is actually possible for each group.
- The definition of `vw_MOSAICAI_HO_ALL` could not be checked.
- Whether a scheduled job already exists for this notebook is unknown.

## Q3. How faithful is the SQL Server → Postgres type mapping in `map_sql_type`?

### Takeaway
The DDL-level mappings are sound for the integer, bit, decimal/numeric, money, float/real, date, varchar/nvarchar(n) and (max)→text cases. Where they fall short:
- `datetime2(7)` precision and `datetimeoffset` offsets are lost.
- `char(n)` carries blank-padded semantics that Postgres discourages.
- Several types fall silently to `text` or fail at read time: `rowversion`/`timestamp`, `geography`, `geometry`, `hierarchyid`, `sql_variant`, and anything new.
- The binary types map correctly in the DDL but are corrupted by the data path (Q4).

### Cited Findings
- **Notebook mapping table** — [C6][dump]:

| SQL Server `DATA_TYPE` | Notebook DDL | Spark read type (Spark 4.2.0 docs) | Assessment |
|---|---|---|---|
| varchar(n), nvarchar(n) | varchar(n) | VarcharType(n) / StringType | SQL Server n is **bytes** (varchar) or **byte-pairs** (nvarchar) ([char/varchar][ms-char]; [nchar/nvarchar][ms-nchar]). Postgres n is **characters** ([PG character types][pg-char]). OK, but see the collation and trailing-space notes in Q5. |
| varchar(max), nvarchar(max) | text (because `CHARACTER_MAXIMUM_LENGTH = -1`) | VarcharType / StringType | Correct. INFORMATION_SCHEMA reports "-1 for xml and large-value type data" ([COLUMNS][ms-infoschema-columns]). |
| char(n), nchar(n) | char(n) | CharType(n) / StringType | Padding is kept. Postgres: "trailing spaces are treated as semantically insignificant … removed when converting", and "In most situations text or character varying should be used instead" ([PG character types][pg-char]). |
| text, ntext | text | StringType | OK. |
| int, bigint, smallint | integer, bigint, smallint | IntegerType, LongType, ShortType | OK. |
| tinyint | smallint | ShortType | OK. |
| bit | boolean | BooleanType | OK. Postgres accepts `True`/`False` because "case does not matter" ([PG boolean][pg-boolean]). |
| decimal(p,s), numeric(p,s) | numeric(p,s) | DecimalType(p,s) | OK. |
| money, smallmoney | numeric(19,4), numeric(10,4) | DecimalType(19,4), DecimalType(10,4) | Consistent with Spark's own mapping ([Spark JDBC][spark-jdbc]). |
| float | double precision | DoubleType for 25 ≤ p ≤ 53 (float(p≤24) is real) | OK. |
| real | real | FloatType | OK. |
| datetime | timestamp | TimestampType (or TimestampNTZType with `preferTimestampNTZ=true`) | OK. Accuracy 0.00333 s, range 1753–9999 ([date/time types][ms-datetime-types]). |
| datetime2 | timestamp | TimestampType / NTZ | Default precision is 7 digits (100 ns) ([datetime2][ms-datetime2]). Postgres precision is 0–6 (1 µs) ([PG datetime][pg-datetime]), so sub-microsecond digits are lost. |
| smalldatetime | timestamp | TimestampType / NTZ | OK (1-minute accuracy). |
| datetimeoffset | timestamptz | TimestampType | SQL Server stores UTC and "The time zone offset is preserved in the database for retrieval" ([datetimeoffset][ms-datetimeoffset]). Postgres timestamptz "is stored internally as UTC, and the originally stated or assumed time zone is not retained" ([PG datetime][pg-datetime]). **Offset lost.** |
| date | date | DateType | OK. |
| time | time | **TimestampType** (TimestampNTZType with `preferTimestampNTZ`) | Spark adds a date part. Postgres time input: "You can also specify a date but it will be ignored" ([PG datetime][pg-datetime]). The 100 ns → 1 µs precision loss also applies. |
| uniqueidentifier | uuid | not listed in Spark's table | Postgres accepts "use of upper-case digits", and "Output is always in the standard form" (lower-case) ([PG uuid][pg-uuid]). |
| binary, varbinary, image | bytea | BinaryType (Spark's table lists `image` as StringType) | DDL OK. **Data path broken; see Q4.** |
| xml | text | not listed (StringType expected) | Acceptable; loses the xml typing. |
| text fallback: `timestamp` (= rowversion) | text | rowversion → BinaryType | T-SQL "**timestamp** is the synonym for the **rowversion** data type"; it "isn't a date or time data type" ([rowversion][ms-rowversion]; [date/time types][ms-datetime-types]). Lands in text as a Python bytes repr (Q4). |
| text fallback: geography, geometry | text | BinaryType | Lands in text as a Python bytes repr. |
| text fallback: sql_variant | text | "UNRECOGNIZED_SQL_TYPE error raised" | The whole table read fails ([Spark JDBC][spark-jdbc]). |
| text fallback: hierarchyid / anything else | text | not documented | Silent fallback; no warning is printed. |

- The notebook selects `CHARACTER_MAXIMUM_LENGTH, NUMERIC_PRECISION, NUMERIC_SCALE`. It does not read `DATETIME_PRECISION` or `COLUMN_DEFAULT`, both of which `INFORMATION_SCHEMA.COLUMNS` provides. — [C6][dump]; [COLUMNS][ms-infoschema-columns]
- Microsoft warns: "Don't use INFORMATION_SCHEMA views to determine the schema of an object. INFORMATION_SCHEMA views only represent a subset of the metadata… The only reliable way… is to query the `sys.objects` catalog view." — [COLUMNS][ms-infoschema-columns]
- `DATA_TYPE` is the "System-supplied data type", so user-defined alias types report their base type. `DOMAIN_NAME` carries the UDT name. — [COLUMNS][ms-infoschema-columns]

### Inferences
- **No length overflow:** SQL Server lengths count bytes or byte-pairs and every character uses at least one unit, while Postgres counts characters. A value that fits the source therefore fits the target varchar(n)/char(n).
- **datetimeoffset:** the instant survives only if Spark's session time zone and Postgres's session `TimeZone` agree (Q4). The original offset is gone in every case, and apps that display "local time as entered" will change behaviour.
- **Silent text fallbacks:** together with the fact that Cell 6 prints no types, `rowversion`/spatial/`hierarchyid` columns can migrate as nonsense text without anyone noticing at review time.
- **Mapping maintenance:** the mapping should fail loudly on unknown types and map explicitly:
  - `rowversion` → bytea, or drop the column;
  - spatial types → PostGIS geometry/geography, or WKT text via `STAsText()` in the source query;
  - `hierarchyid` → text via `ToString()`;
  - `sql_variant` → excluded or cast in the source query.

### Gaps
- How Spark maps `uniqueidentifier` and `hierarchyid` is not documented in the Spark 4.2.0 table.
- Spark's table lists `image` → StringType, which looks like a doc oddity; it was not verified empirically.
- Whether any in-scope table actually uses the risky types (binary, rowversion, spatial, datetimeoffset, time, sql_variant) is unknown without access to the source catalog.
- Newer Azure SQL types such as `json`/`vector` would hit the `text` fallback; their exact `DATA_TYPE` strings and status were not verified in this session.
- The Lakebase project's Postgres major version and whether PostGIS is available were not verified.

## Q4. How faithful are values end-to-end along Spark JDBC read → `toPandas()` → `to_csv(na_rep='\N', QUOTE_MINIMAL)` → `COPY … FORMAT csv, NULL '\N'`?

### Takeaway
These survive correctly:
- NULLs and empty strings, and their distinction;
- commas, quotes and newlines in text;
- booleans, dates, decimals and Unicode text;
- integers up to 2^53.

These do not:
- **Binary data**: it becomes Python `b'...'` repr text, which either fails COPY or is stored as garbage.
- **Literal `\N` strings**: they become NULL.
- **Large bigint values in columns that contain NULLs**: they are silently altered beyond 2^53.
- **Timestamps**: they depend on time-zone consistency and on pandas' 1677–2262 nanosecond range.
- **`datetimeoffset`**: its offset is lost.

### Cited Findings
- **Pipeline facts:**
  - `toPandas()` "results in the collection of all records in the DataFrame to the driver". Arrow is on by default. Timestamps are "converted to nanoseconds and each column will be converted to the Spark session time zone then localized to that time zone, which removes the time zone". — [PySpark Arrow guide 4.2.0][pyspark-arrow]
  - The serverless default session time zone is `Etc/UTC`. — [Serverless limitations][dbx-serverless-limits]
  - Arrow → pandas conversions: "(U)INT{8,16,32,64} *with nulls* → `float64`"; BOOL with nulls → object (True/False/None); BINARY → "object arrays of bytes objects"; DATE32 → `datetime.date` objects; DECIMAL → `decimal.Decimal` objects. — [Arrow pandas integration][arrow-pandas]
  - Environment 5: "BinaryType consistently maps to Python bytes". — [env5][dbx-env5]
  - pandas `na_rep` is the "Missing data representation" (default `''`) and `quoting` defaults to `csv.QUOTE_MINIMAL`. — [pandas to_csv][pandas-to-csv]
  - Python csv: QUOTE_MINIMAL quotes "only those fields which contain special characters such as *delimiter*, *quotechar*, `'\r'`, `'\n'` or any of the characters in *lineterminator*". "All other non-string data are stringified with `str()` before being written". — [Python csv 3.14][py-csv]
  - Postgres COPY: NULL is "`\N` … in text format, and an unquoted empty string in CSV format". "A NULL is output as the NULL parameter string and is not quoted, while a non-NULL value matching the NULL parameter string is quoted… Reading values follows similar rules". "CSV format will both recognize and produce CSV files with quoted values containing embedded carriage returns and line feeds". — [PG COPY][pg-copy]
  - Postgres bytea escape input: a backslash must be followed by three octal digits or another backslash, otherwise "invalid input syntax for type bytea". "non-backslash printable characters … are stored as-is". — [PG bytea][pg-bytea]
- **Local probe results** (Python 3.14.7 stdlib `csv.writer`, QUOTE_MINIMAL, `lineterminator="\n"`) — [probe]:
  - empty string → `x,,y` (unquoted empty field);
  - literal two-character string `\N` → `x,\N,y` (**unquoted**);
  - `a,b` → `"a,b"`;
  - `say "hi"` → `"say ""hi"""`;
  - LF and lone CR → quoted;
  - `b"\x00\x01ab"` → `b'\x00\x01ab'`;
  - `b"abc"` → `b'abc'`;
  - `bytearray(b"\x00")` → `bytearray(b'\x00')`;
  - `True` → `True`;
  - `Decimal("0E-8")` → `0E-8`;
  - `date(2024,1,2)` → `2024-01-02`;
  - Unicode written as-is;
  - a single-field row containing `""` → `""`;
  - `int(float(2**53+1))` → 9007199254740992, so precision is lost.
- **Timestamp range:** pandas nanosecond timestamps span `1677-09-21 00:12:43.145224193` to `2262-04-11 23:47:16.854775807`. — [pandas timestamp limitations][pandas-ts-limits] SQL Server `datetime2`/`datetimeoffset` range is 0001-01-01 to 9999-12-31. — [date/time types][ms-datetime-types]
- **Time-zone interpretation in Postgres:** for timestamptz input without a zone, the value "is assumed to be in the time zone indicated by the system's TimeZone parameter". For `timestamp without time zone`, "PostgreSQL will silently ignore any time zone indication". — [PG datetime][pg-datetime]
- **Notebook's own type fixes:** it casts float64 → `Int64` only for Spark Integer/Long/Short/Byte columns, and its datetime loop does `pdf[col].where(pdf[col].notna(), None)`. — [C7][dump]

**Outcome by value class** (fact plus the chain of cited behaviours above)
- **NULL in any column:** pandas NA → unquoted `\N` → NULL. ✓
- **Empty string `''`:** an unquoted empty field does not match the NULL string `\N`, so it loads as `''`. ✓ (A single-column row is written `""`, which also loads as `''`.)
- **A literal `\N` text value:** written unquoted, so it loads as **NULL**. ✗ Low severity (rare, but it is silent corruption).
- **Commas, double quotes, LF/CRLF inside text:** quoted, with doubled quotes. ✓
- **bit/boolean:** `True`/`False`, or NA → `\N`. ✓
- **Integers with no NULLs:** native int dtype. ✓
- **Integers with NULLs:** float64, then the notebook casts back to Int64. ✓ for |x| ≤ 2^53. ✗ for larger bigint values, which are silently rounded *before* the fix runs. Medium severity for hashed or snowflake-style IDs.
- **decimal/money:** `Decimal` → `str()`, which may use exponent form (`0E-8`). ✓ (see Inferences).
- **date:** ISO `YYYY-MM-DD`. ✓
- **datetime/datetime2/smalldatetime:**
  - ✓ when the Spark session TZ is UTC (the serverless default) or matches the JVM's.
  - ✗ for values outside pandas' 1677–2262 ns range (for example `9999-12-31` or `0001-01-01` sentinels): the conversion cannot represent them.
- **datetimeoffset:** the offset is lost, and naive strings are re-interpreted in the Postgres session `TimeZone`.
- **time:** Spark gives TimestampType, which becomes a "date + time" string, and Postgres ignores the date. ✓ normally.
- **uniqueidentifier:** loads; the case changes on output (Postgres outputs lower-case).
- **binary/varbinary/image → bytea:**
  - values with non-printable bytes → `b'\x..'` → "invalid input syntax for type bytea", so the table fails;
  - all-printable values → **stored as the literal repr bytes** `b'abc'`, silently.
  - ✗ **High severity.**
- **rowversion/geography/geometry (/hierarchyid) → text:** stored as Python repr strings such as `b'\x00…'`, silently. ✗ High severity where present.
- **sql_variant:** the Spark read fails, so the table fails. ✗ Loud.

### Inferences
- **The datetime loop does nothing:** a `datetime64[ns]` Series cannot hold Python `None`, so pandas coerces it back to `NaT`. NaT is then written as `na_rep`, which is correct anyway.
- **Not all failures are loud:** binary columns with any non-printable byte fail loudly, but all-printable binary or text-fallback binary columns corrupt silently, and the row-count check still passes.
- **Postgres `numeric` should accept exponent-form strings like `0E-8`.** This is standard numeric input, but it was not re-verified here.
- **NUL characters (`\x00`) in source strings** cannot be stored in Postgres text types, so such a table would fail COPY. Not verified in this session.
- **A SQL Server `time` column whose non-null values are all `00:00:00`** may be written by pandas as date-only strings, which Postgres `time` input might reject. Unverified edge case.
- **Simplest robust fix:** skip pandas and CSV entirely. Use psycopg `copy.write_row()`, which adapts values "as they would be in normal querying" (bytes→bytea, None→NULL, Decimal, datetime, UUID) ([psycopg COPY][psycopg-copy]). Alternatively use binary COPY with `set_types()` (Postgres "will apply no cast rules" in binary mode ([psycopg COPY][psycopg-copy])). Feed it from Arrow batches or `toLocalIterator()`, read with `preferTimestampNTZ=true` ([Spark JDBC][spark-jdbc]), and apply explicit source-side casts for datetimeoffset (to an ISO string with offset), spatial (to WKT) and rowversion (to binary/hex).

### Gaps
- **Lone carriage returns on Python 3.12 (untested).**
  - The Python 3.12 docs (3.12.15) define QUOTE_MINIMAL as quoting fields that contain "*delimiter*, *quotechar* or any of the characters in *lineterminator*", with no mention of `'\r'`/`'\n'`. The 3.14 docs list "`'\r'`, `'\n'`" explicitly. — [Python csv 3.12][py-csv-312]; [Python csv 3.14][py-csv]
  - On environment 5 (Python 3.12.3, pandas `lineterminator` `'\n'` on Linux), a text value containing a lone carriage return may therefore be written **unquoted**.
  - Postgres's CSV parser would then treat it as an end of line, causing a COPY error and a failed table, which Q6 shows is left emptied.
  - CRLF text contains `\n` and is quoted either way. Low severity.
  - This was not tested on 3.12, so it remains an inference.
- Whether PySpark 4.x `toPandas()` raises or overflows on out-of-range timestamps was not verified. Either outcome is a defect: a failed table or wrong values.
- Whether the JVM default time zone on the compute used differs from `spark.sql.session.timeZone` is unknown.
- How `str(numpy.float64)` is formatted at round-trip precision in pandas 2.2.3 `to_csv` was not verified. It is expected to be shortest round-trip repr.

## Q5. Which schema elements are not carried over, and what breaks once apps INSERT/UPDATE on Lakebase after cutover?

### Takeaway
Only column names and base types are recreated. Dropped:
- nullability;
- primary keys and identity/sequences;
- defaults;
- unique and check constraints;
- indexes;
- computed-column logic;
- triggers;
- view definitions.

Collation semantics also change, from case-insensitive with trailing-space-insensitive equality to byte-exact. Lower-casing and snake-casing names forces app SQL changes for CamelCase columns. The first app INSERTs, upserts and keyed lookups after cutover are where breakage will show.

### Cited Findings
- **The generated DDL is column/type only:** `CREATE TABLE IF NOT EXISTS {target_table} (\n col type, … )`, and `IS_NULLABLE` is fetched but unused. — [C6][dump]
- **What Postgres does with the missing pieces:**
  - "Each column not present in the explicit or implicit column list will be filled with a default value, either its declared default value or null if there is none." — [PG INSERT][pg-insert]
  - With `ON CONFLICT`, "If an attempt at inference is unsuccessful, an error is raised" (a unique index or constraint is required). — [PG INSERT][pg-insert]
  - Identity columns: `GENERATED BY DEFAULT AS IDENTITY` lets "the user-specified value take precedence", attaches "an implicit sequence", and "Such a column is implicitly NOT NULL". — [PG CREATE TABLE][pg-createtable]
  - TRUNCATE's default is `CONTINUE IDENTITY` ("Do not change the values of sequences"). — [PG TRUNCATE][pg-truncate]
- **Collation and comparison semantics change:**
  - Azure SQL Database's default collation is `SQL_LATIN1_GENERAL_CP1_CI_AS`, where "`CI` is case-insensitive". — [Logical server][ms-logical-server]
  - SQL Server compares strings with padding, so "Transact-SQL considers the strings `'abc'` and `'abc '` to be equivalent for most comparison operations" (LIKE excepted). — [String comparison][ms-string-compare]
  - Postgres deterministic collations consider "strings to be equal only if they consist of the same byte sequence". Case-insensitive comparison needs a nondeterministic ICU collation, which has "a performance penalty" and makes some pattern matching impossible. — [PG collation][pg-collation]
  - In Postgres, "Trailing spaces are semantically significant in `character varying` and `text` values". — [PG character types][pg-char]
- **Identifier case-folding:** Postgres folds unquoted identifiers to lower case ("`FOO`, `foo`, and `"foo"` are considered the same … but `"Foo"` and `"FOO"` are different"). — [PG lexical][pg-lexical]
- **The schema-setup notebook shows the missing translation knowledge:** for its new tables it hand-writes PKs, `GENERATED ALWAYS AS IDENTITY`, `DEFAULT CURRENT_TIMESTAMP`, a unique index for `ON CONFLICT` deduplication, a CHECK constraint and secondary indexes. — [Setup C4, C5][setup]

### Inferences
- **Breakage after cutover:**
  - **Inserts that relied on identity:** app inserts that relied on `IDENTITY` or `DEFAULT GETDATE()/NEWID()` will insert NULL ids or timestamps instead of failing. This produces duplicate or NULL "keys" and NULL audit columns.
  - **Upserts:** any `INSERT … ON CONFLICT` (the natural Postgres replacement for MERGE/IGNORE_DUP_KEY patterns) will error, because there is no unique index.
  - **Keyed access:** UPDATE/DELETE/SELECT by key becomes sequential scans. On `CALL_DETAIL_TM31` (~743K rows) and `ADMARKET_LONG_TM31` (~568K rows) this is noticeable for OLTP latency.
  - **Constraints:** none (NOT NULL, uniqueness, CHECK) are enforced, so bad writes are accepted silently.
  - **Computed columns:** they become plain columns frozen at copy time, and new rows get NULL unless the app supplies a value.
  - **Triggers:** no trigger logic (audit or derived fields) runs.
- **Adding identity later:** if identity is added to migrated tables, each reload needs `setval()` to max(id) afterwards. Otherwise sequences collide with copied ids. TRUNCATE does not reset sequences by default.
- **Name changes break app SQL:**
  - All-caps-with-underscore names keep working unquoted, because they fold to the same lower-case names.
  - CamelCase columns are renamed by `to_snake_case` (`PromptText` → `prompt_text`), so app SQL referencing them breaks.
  - App code that quotes upper-case identifiers (`"MOSAICAI_PROMPTS"`) breaks.
- **Behaviour changes to check per table:** email/user lookups (for example the PACE_AI_USERS tables) can return no rows after cutover where SQL Server matched case-insensitively or ignored trailing spaces. Lower-case uuid output can also break case-sensitive string comparisons of GUIDs in app code.
- **The view:** `vw_MOSAICAI_HO_ALL` becomes a static table. After cutover, writes to the HO1/2/3 tables are not reflected in it.

### Gaps
- The source catalog could not be inspected for which tables have identity columns, defaults, PKs, indexes, computed columns or triggers (`sys.identity_columns`, `sys.default_constraints`, `sys.indexes`, `sys.computed_columns`, `sys.triggers`).
- Whether the R/Python app code uses quoted identifiers, `OUTPUT INSERTED`, `SCOPE_IDENTITY()`, `MERGE` or other T-SQL constructs that need rewriting was not reviewed.
- The Lakebase database's default collation/locale was not verified.

## Q6. What are the transaction semantics of the truncate and recreate modes, and is a row-count comparison enough?

### Takeaway
Both modes **commit the destructive step first**, in its own transaction. After that the notebook re-reads the source, builds the CSV and runs COPY in a second transaction. As a result:
- readers see an empty table, or block, for the whole reload;
- any failure after the first commit leaves the table empty;
- the row-count check compares two different source reads against the target, and cannot detect value corruption.

### Cited Findings
- **Order of operations:** `prepare_target_table()` runs `CREATE TABLE IF NOT EXISTS` + `TRUNCATE` + `commit()` (or `DROP` + `CREATE` + `commit()`). Only then does `migrate_table()` call `source_df.toPandas()`, build the CSV, run COPY and `commit()`. The except block runs `conn.rollback()`. — [C7][dump]
- **psycopg transactions:**
  - "By default, any database operation will start a new transaction".
  - After an error "the server will refuse further commands, until a `rollback()` is called".
  - Closing a connection with a transaction open sends no COMMIT.

  — [psycopg transactions][psycopg-tx]
- **psycopg COPY:** "If an exception is raised inside the block, the operation is interrupted and the records inserted so far are discarded." — [psycopg COPY][psycopg-copy]
- **TRUNCATE locking and visibility** — [PG TRUNCATE][pg-truncate]:
  - "acquires an `ACCESS EXCLUSIVE` lock on each table it operates on, which blocks all other concurrent operations on the table".
  - "is not MVCC-safe. After truncation, the table will appear empty to concurrent transactions, if they are using a snapshot taken before the truncation occurred".
  - "is transaction-safe … the truncation will be safely rolled back if the surrounding transaction does not commit".
  - "You must have the `TRUNCATE` privilege".
- **Failed COPY:** "If the command fails, these rows are left in a deleted state; these rows will not be visible, but still occupy disk space… `VACUUM` should be used". `COPY … FREEZE` applies only if the table "has been created or truncated in the current subtransaction". — [PG COPY][pg-copy]
- **No safeguard on table shape:** `CREATE TABLE IF NOT EXISTS`: "there is no guarantee that the existing relation is anything like the one that would have been created". COPY without a column list copies "all columns of the table except generated columns". — [PG CREATE TABLE][pg-createtable]; [PG COPY][pg-copy]
- **Azure SQL reads are only statement-consistent:** new Azure SQL databases enable RCSI by default, which presents "each statement with a transactionally consistent snapshot of the data as it existed at the start of the statement". — [Row versioning guide][ms-rcsi]; [Deadlocks doc][ms-rcsi-default]
- **Serverless cannot cache:** `df.cache()`/`persist()` "are not supported on serverless compute". — [Serverless limitations][dbx-serverless-limits]

### Inferences
- **What concurrent readers see during a reload:**
  - TRUNCATE takes ACCESS EXCLUSIVE, so it waits for in-flight readers and queues new ones behind it.
  - From its commit until the COPY commit, readers see an **empty** table.
  - That window covers a second full JDBC read, the pandas conversion, CSV building and the COPY itself (minutes for the largest tables).
  - In recreate mode, the DROP and CREATE commit together, so readers wait and then see an empty new table. Grants, indexes and any manual additions on the old table are gone.
- **If COPY fails** (bytea error, type mismatch, dropped connection), the rollback restores only the COPY transaction. The table stays **empty**, with dead tuples left over.
  - In Phase 1 this breaks Workbench testing.
  - In Phase 2 it leaves the cut-over table empty right before apps are enabled, unless someone reads the summary.
- **Schema drift in truncate mode:**
  - A new source column makes COPY fail ("extra data after last expected column") after the TRUNCATE has committed.
  - A target recreated by hand with the same column count but a different order loads values into the **wrong columns** silently whenever the types are compatible.
- **Correct pattern:** one transaction per table: `BEGIN; TRUNCATE; COPY (explicit column list) [FREEZE]; verify; COMMIT`. Readers then block but never see an empty table, and a failure restores the old data. For no blocking, load a staging table and swap names in a short transaction (`ALTER TABLE … RENAME`), and re-apply grants and indexes to the staging table first.
- **Row counts are not enough:**
  - `source_count` comes from a different statement than the data loaded (`count()` vs `toPandas()`), so during Phase 1, while apps are still writing, a mismatch or a coincidental match is possible.
  - Counts cannot detect value corruption (bytes repr, `\N`→NULL, precision, time zones).
  - Add per-column checks: NULL counts, min/max, sums of numeric columns, an order-independent hash of key plus row content, and a sampled row-level diff (for example a Spark `exceptAll` between a source read and a Postgres read) against **one** materialised source snapshot.

### Gaps
- How long the empty-table window lasts per table is unknown, because there are no outputs or timings.

## Q7. How reliable and scalable is it (memory, partitioning, token refresh, connection handling, partial failure)?

### Takeaway
It is workable at today's ~3 GB on an adequately sized driver. However:
- each table sits in driver memory several times over (Arrow result, pandas DataFrame, one big CSV string);
- each table is read twice over a single JDBC connection;
- the 45-minute token refresh is unnecessary for an open connection, since expiry matters only at login;
- there is no reconnect or retry for real failures;
- it never fails the run, so a scheduled job will report success even when tables failed, mismatched, or were silently skipped.

### Cited Findings
- **Driver memory:** `toPandas()` collects all records to the driver and "should be done on a small subset of the data". — [PySpark Arrow guide][pyspark-arrow]
- **Single-threaded reads:** "By default, the JDBC driver queries the source database with only a single thread". Parallel reads need `partitionColumn`/`lowerBound`/`upperBound`/`numPartitions`, and Databricks warns against high values for application databases ("Be wary of setting this value above 50"). — [Query databases using JDBC][dbx-jdbc]; [Spark JDBC][spark-jdbc]
- **What the code does** — [C7][dump]:
  - one full JDBC read for `count()` and another for `toPandas()`;
  - the entire CSV in an `io.StringIO`;
  - `refresh_connection_if_needed()` called once per table, before `prepare_target_table`;
  - no retries;
  - `conn.rollback()` inside `except` without its own try;
  - no exit code or raise in Cell 8.
- **Lakebase connection rules** — [Lakebase authentication][lb-auth]:
  - "Token expiration is enforced only at login. Open connections remain active even after the token expires".
  - Connections face a "24-hour idle timeout" and a "3-day maximum connection life".
  - "Design your applications to handle connection timeouts gracefully by implementing connection retry logic".
  - The documented rotation pattern mints a fresh token per new connection (psycopg pool `CustomConnection.connect`) or refreshes using `credential.expire_time`.
- **Endpoints can be read-only:** a branch can have a primary compute (`ENDPOINT_TYPE_READ_WRITE`) and read replicas (`ENDPOINT_TYPE_READ_ONLY`), and the code takes `endpoints[0]` without checking the type. — [Lakebase API guide][lb-api]; [C5][dump]
- **psycopg connection states:** "A broken connection is always closed, but wasn't closed in a clean way". The context manager commits or rolls back and closes on exit. — [psycopg connections][psycopg-conn]
- **Serverless jobs:** "By default, serverless jobs have no query execution timeout", and runs are capped at 7 days. — [Serverless limitations][dbx-serverless-limits]

### Inferences
- **Memory:**
  - The peak for the 743K-row table is roughly the Arrow buffers plus the pandas frame (object columns are heavy) plus the full CSV text, all held at once.
  - It also scales with string widths; the `(max)` columns are unknown.
  - Cell 6's `columns_df` is not cached (and caching is unsupported on serverless), so each per-table `.collect()` can re-query `INFORMATION_SCHEMA` (26 small queries).
- **Token refresh:** because expiry is enforced only at login, the reconnect every 45 minutes buys nothing for an open connection. It is harmless, but it is the only reconnect logic. A dropped connection (network blip, compute restart or failover) is not handled.
- **Failure handling:**
  - If the connection is broken, `conn.rollback()` inside `except` will itself raise. That aborts the loop and leaves later tables untouched, with no summary and the earlier tables committed.
  - `conn` is never closed if a cell errors.
- **Partial failure:**
  - Tables before a failure are committed.
  - The failing table is left **empty** (Q6).
  - Later tables continue.
  - Tables whose `INFORMATION_SCHEMA` lookup returns nothing are skipped with a print and never appear in `results`, so the summary under-reports them.
- **Spark filtering is case-sensitive:** Spark string equality in `columns_df.filter(...)` is case-sensitive by default, while SQL Server's WHERE (CI collation) is not. A registry entry whose case differs from the catalog's stored case is fetched by SQL Server and then dropped by Spark, so the table is silently skipped.
- **Endpoint choice:** if a read replica is ever added to the branch and listed first, `endpoints[0]` points writes at a read-only endpoint, and every table fails.

### Gaps
- Driver memory size, run times, and whether any run has exceeded 45 minutes (triggering the refresh path) are unknown.
- Whether `list_endpoints` returns a stable order is not documented.
- What psycopg raises on `rollback()` of a closed or broken connection is not stated in the docs read.
- Lakebase scale-to-zero behaviour (suspend and wake latency, and whether a suspend drops idle sessions) was not checked.

## Q8. What are the security issues (auth mode, hard-coded user, secrets, f-string SQL, object ownership)?

### Takeaway
- **Source authentication** is deprecated and Microsoft says it is incompatible with mandatory MFA. Microsoft also says not to use a user account as a service account, which is what the hard-coded UPN is.
- **Target authentication** follows the documented OAuth and SSL pattern.
- **Missing pieces:**
  - no ownership or role design for jobs and apps;
  - identifiers are composed with unquoted f-strings (low injection risk today, but fragile);
  - internal hostnames and paths are embedded in a shareable `.dbc`.

### Cited Findings
- **ActiveDirectoryPassword status** — [JDBC Entra auth][ms-jdbc-entra]; [JDBC 13.2 release notes][ms-jdbc-relnotes]; [Connection properties][ms-jdbc-props]:
  - "The ActiveDirectoryPassword authentication option (Microsoft Entra ID password authentication) is deprecated in the Microsoft SQL drivers. This high-risk authentication flow is incompatible with mandatory Microsoft Entra multifactor authentication (MFA) and might not work in tenants where MFA is enforced. Plan to migrate…".
  - It is based on "the OAuth 2.0 Resource Owner Password Credentials (ROPC) grant".
  - Recommended alternatives for unattended scenarios: managed identity (ActiveDirectoryMSI/ManagedIdentity), else ActiveDirectoryServicePrincipal (preferring certificates).
  - "Don't use a Microsoft Entra user account as a service account."
  - JDBC driver 13.2 "Added deprecation warning for Microsoft Entra ID password authentication method", and connection-property docs label it "ActiveDirectoryPassword [DEPRECATED]".
- **Microsoft's OLE DB driver docs:** "Plan to migrate to a more secure authentication method before this option is removed." — [MSOLEDBSQL deprecated methods][ms-oledb-deprecated]
- **What the code embeds:** the UPN in `JDBC_CONFIG["user"]`, the password from `dbutils.secrets.get(...)`, and both kept in a module-level `connection_properties` dict. — [C2, C5][dump]
- **Databricks guidance:** Databricks recommends secrets for JDBC credentials, and Lakehouse Federation where it supports the source ("The legacy query federation documentation has been retired…"). — [Query databases using JDBC][dbx-jdbc]
- **Unity Catalog JDBC connections:** "Connection credentials are hidden from the querying user". OAuth M2M is in Beta. — [JDBC connection][dbx-jdbc-conn]
- **Identifiers are built as unquoted f-strings:**
  - Postgres: `DROP TABLE IF EXISTS {target}`, `TRUNCATE TABLE {target}`, `COPY {target} …`, `SELECT COUNT(*) FROM {target}`, `CREATE SCHEMA IF NOT EXISTS {schema_name}`, and the generated column names.
  - SQL Server: string literals in the `INFORMATION_SCHEMA` filter.
  - Spark: `.filter()` string expressions.

  — [C6, C7][dump]
- **Postgres reserved words and identifier rules:**
  - USER, ORDER, GROUP, DESC, END, LIMIT, OFFSET, DEFAULT, CHECK, COLUMN, TABLE, PRIMARY, WHEN, ONLY, FROM and TO are reserved. Reserved words "are not allowed as column or table names" unless quoted. — [PG keywords][pg-keywords]
  - Unquoted identifiers must begin with a letter or underscore. Longer names are truncated to 63 bytes. — [PG lexical][pg-lexical]
- **psycopg supports safe composition:** "You can compose a COPY statement dynamically by using objects from the `psycopg.sql` module" (`sql.SQL`, `sql.Identifier`). — [psycopg COPY][psycopg-copy]
- **Lakebase roles and ownership** — [Create Postgres roles][lb-pg-roles]:
  - The project owner's role owns `databricks_postgres` and is in `databricks_superuser`.
  - Other identities need a `databricks_create_role()` role, which has LOGIN only, and then explicit GRANTs.
  - A role "cannot be dropped if it owns database objects" unless ownership is reassigned.
  - "If a user's email or group display name changes … authentication and existing database grants break."
  - Group roles let any member authenticate as the group.

### Inferences
- **Who owns what:** every schema and table is owned by whoever runs the notebook, a human user when run interactively. This causes problems in three ways:
  - A scheduled job running as a service principal cannot TRUNCATE (without a grant) or DROP (owner only) those tables, so truncate or recreate mode fails.
  - The person leaving or being renamed complicates ownership.
  - Recreate mode silently drops any GRANTs made to the app roles.
- **Fix for ownership:** create a dedicated owner role (a Databricks group role, or a service-principal role) and `ALTER DEFAULT PRIVILEGES`. Grant SELECT/INSERT/UPDATE/DELETE to the app roles, and run the job as the service principal.
- **Injection risk:** low while the registry is hard-coded. It becomes real if the registry moves to widgets or a control table (Q10).
- **Quoting today is mainly a correctness problem:**
  - Columns named `User`, `Order`, `Group`, `Desc` or `End` (common in app tables) make `CREATE TABLE` fail.
  - Names with spaces, hyphens, `#`, `$` at the start, or leading digits fail.
  - Two source columns that snake-case to the same name (`UserID` and `User_ID`) collide.
- **Secret exposure:** `connection_properties` keeps the plaintext password in a notebook global, where it could be printed or serialised.
- **Information disclosure:** the `.dbc` exposes the SQL Server hostname, a Lakebase endpoint hostname, a UPN, secret scope and key names, and a personal workspace path (Cell 1, Cell 2). That is low-severity, but relevant if the archive is shared.
- **App authentication after cutover:** Posit Connect apps must use either OAuth with hourly token rotation (no PgBouncer) or native password roles. Password roles are disabled by default for new projects and must be enabled per project or compute.

### Gaps
- Whether the tenant already enforces MFA or Conditional Access on the `bis-azure` account is unknown, as is whether a service principal or managed identity is available for Azure SQL.
- Whether Databricks output redaction would mask the password if it were printed was not verified.
- The minimal project ACL for job identities was not confirmed.

## Q9. Where do the documentation and code disagree?

### Takeaway
There are at least 16 discrepancies. Most are stale documentation:
- dev vs production targets;
- DROP+CREATE vs truncate;
- cell numbering;
- `TABLES_TO_MIGRATE`;
- `%pip`;
- "verify type mappings";
- 12 vs 26 tables.

Others reflect real scope problems:
- backups;
- a view;
- a probable typo;
- duplicate `rte_requests` tables;
- a missing cutover-status step.

### Cited Findings
1. **Database: dev vs production.** The header names source database `POSIT_DEV`. The config has `"database": "POSIT",          # Change to "POSIT" for production`, so the value already *is* the production name by its own comment, and the comment is stale. — [C1, C2][dump]
2. **Lakebase project.** The header names project `posit-dev-testdb`. The config uses `project_id: "posit"`, as does the schema-setup notebook (`posit` / `production` / `databricks_postgres`). — [C1, C2][dump]; [Setup C1, C3][setup]
3. **Endpoint.** The header hard-codes an endpoint hostname. The code discovers endpoints and uses `endpoints[0]`. — [C1, C5][dump]
4. **Reload mode.** The markdown says "notebook is idempotent: DROP + CREATE" and "Migration Loop | For each table: JDBC read → DROP/CREATE → buffer COPY". The default `reload_mode` is `truncate` for all 26 entries. — [C1, C2, C7][dump]
5. **Cell numbering:**
   - The markdown cell table lists 6 cells numbered 1–6, with Configuration as 1.
   - The actual notebook has 8 cells: 1 markdown, 2 Configuration, 3 Dependencies, 4 Dependency Validation (undocumented), 5 Connections, 6 Schema Discovery, 7 Migration, 8 Summary.
   - Phase 1 says "Cell 2" for configuration, which is correct only if the markdown counts, and "Cell 5" for Schema Discovery, which is actually Cell 6.
   
   — [C1][dump]
6. **Install method.** The markdown says "`%pip install` for psycopg + databricks-sdk (run once per session)". The code uses `subprocess` pip, with "no Python restart required" in Cells 3 and 4. — [C1, C3, C4][dump]
7. **Variable to edit.** The markdown says "Add source table name to `TABLES_TO_MIGRATE`" (twice). The real input is `TABLE_MIGRATION_REGISTRY` (dicts); `TABLES_TO_MIGRATE` is derived and unused. — [C1, C2][dump]
8. **Type review.** The markdown says "verify type mappings and column renames" from the discovery output. Only the renames are printed. — [C1, C6][dump]
9. **Table count.** "Tables in Scope" lists 12 `mosaicaiderm` tables marked "Migrated (N rows)". The registry has 26 entries, and the 14 others are undocumented: 12 `mosaicai`, `fieldlensonc.FIELDLENS_METRICS` and `fieldlensderm.FIELDLENS_METRICS`. The 12 `mosaicaiderm` entries match the markdown list one-for-one. — [C1, C2][dump]
10. **Backup tables in scope:** `mosaicai.MOSAICAI_PROMPTS_EDIT_BAK_20260106` and `mosaicai.RTE_REQUESTS_BAK`. — [C2][dump]
11. **A view in a table list:** `mosaicai.vw_MOSAICAI_HO_ALL`. It is materialised as the table `mosaicai.vw_mosaicai_ho_all`. — [C2, C6][dump]
12. **Probable typo:** `mosaicai.MOASIAC_PROMPTS_TRACKING` sits next to `mosaicai.MOSAICAI_PROMPTS_TRACKING`. — [C2][dump]
13. **Duplicate `rte_requests`.** `mosaicai.RTE_REQUESTS` is migrated into `mosaicai.rte_requests`. The schema-setup notebook creates a separate, empty, app-owned `mosaic_redesign.rte_requests` ("not migrated from Azure SQL"; DDL from Delta). — [C2][dump]; [Setup C1, C5][setup]
14. **Missing cutover step.** The Phase 2 procedure never says to flip `cutover_status` to `cutover_complete`, although the registry comment frames the notebook as a scheduled sync. — [C1, C2][dump]
15. **Restart guidance.** Cell 4 prints "Python restart required: no". Databricks docs say that after `%pip` installs or upgrades the SDK, "you must restart Python to make the installed or upgraded library available to the notebook", and recommend `dbutils.library.restartPython()` after upgrading a runtime-included package. — [C4][dump]; [Databricks SDK for Python][dbx-sdk-python]; [Restart Python][dbx-restart-python]
16. **Skill behaviour.** The markdown says "The Databricks Assistant reads this skill automatically". Current Databricks docs describe skills under "Genie Code":
    - user skills live at `/Users/{username}/.assistant/skills/` and workspace skills at `Workspace/.assistant/skills/`;
    - skills are picked up automatically, or `@`-mentioned to force use;
    - "When you edit an existing skill, start a new chat to apply your changes";
    - workspace skills "codify shared standards … so every user gets consistent output".

    — [C1][dump]; [Genie Code skills][dbx-genie-skills]

### Inferences
- **Dev vs prod confusion:** items 1–2 mean a reader following the header would believe this targets a dev database and dev project, while the code would copy from what its own comment calls production into project `posit`. Either the header is stale or the config is wrong. Confirm before any scheduled run.
- **Typo entry:** item 12 is either a nonexistent table, silently skipped (Q7), or a real misspelt legacy table that will be created as `mosaicai.moasiac_prompts_tracking`. Both outcomes need a decision.
- **Status labels:** "Migrated" in the scope table contrasts with `pre_cutover` in the registry. "Copied, not cut over" is the accurate state.
- **Personal skill:** as a user skill in one person's folder, the skill helps only that user unless promoted to the workspace folder. Its type map can drift from `map_sql_type`, because they are two sources of truth.
- **Cross-notebook observation:** the schema-setup cells wrap each DDL in `try/except` and continue after printing "ERR". In Postgres an error aborts the transaction until rollback ([psycopg transactions][psycopg-tx]), so later statements in that cell would also fail, and the final `commit()` would not persist them. The schema-setup researcher should confirm this.

### Gaps
- Whether the header or the config reflects the intended target is unknown.
- Whether `MOASIAC_PROMPTS_TRACKING` exists in the source is unknown.

## Q10. How could the notebook be used going forward (scheduled refresh job, reusable template, with the skill), and which changes make it production-grade, in priority order?

### Takeaway
It can serve as:
- a pre-cutover refresh job;
- a single-table cutover runner;
- a reusable Azure SQL → Lakebase template paired with a workspace-level Genie Code skill.

It needs these changes first:
- fail-safe cutover control;
- atomic reloads;
- a type-safe data path;
- full DDL extraction;
- loud failures;
- parameters;
- service-principal authentication.

### Cited Findings
- **Serverless environment and dependencies:**
  - Serverless jobs can use notebook-scoped libraries, a job "Environment", widgets for "job and task parameters", and job-level environment variables (Beta). — [Serverless limitations][dbx-serverless-limits]; [Configure the serverless environment][dbx-serverless-deps]
  - Databricks: "We recommend moving away from unpinned `%pip install` commands as they are not compatible with Databricks Serverless compute architecture. Use pin to `==<version>` instead". `%pip` "do[es] not automatically restart the Python process". — [Notebook-scoped libraries][dbx-nb-libs]
- **Connection options:**
  - The Unity Catalog JDBC connection works on serverless and dedicated/standard compute, hides credentials, and supports OAuth M2M (Beta). — [JDBC connection][dbx-jdbc-conn]
  - Lakehouse Federation is recommended over raw JDBC "If Lakehouse Federation supports your source database". — [Query databases using JDBC][dbx-jdbc]
  - Bundled-connector writes on serverless "are supported for PostgreSQL, SQL Server, MySQL, Snowflake, and Redshift". Custom third-party JARs are not supported on serverless. — [Spark data sources][dbx-spark-sources]
- **Safety nets:** Lakebase snapshots, point-in-time restore and child branches are documented restore and test mechanisms (details in Q2). — [Snapshots][lb-snapshots]; [Point-in-time restore][lb-pitr]; [Branches][lb-branches]
- **Skill format:** Genie Code skills need a `SKILL.md` with `name`/`description` frontmatter. Extra skill folders, such as a Git folder, can be registered, and docs advise version-controlling skills in Git folders. — [Genie Code skills][dbx-genie-skills]
- **The documented token-rotation pattern** mints a token per new connection. — [Lakebase authentication][lb-auth]

### Inferences
**Usage modes:**
1. **Scheduled pre-cutover refresh:** acceptable only after changes 1–5 below. Run as a service principal, with max concurrency 1 and an alert on failure, writing to a child branch or with a snapshot first.
2. **Cutover runner:** one table group per run, chosen by parameter, in a single transaction, ending with the status flip in the control table.
3. **Template for other Azure SQL → Lakebase moves:** factor `map_sql_type`, the DDL extraction and the loader into a workspace Python module. The notebook and the SKILL.md then reference one source of truth.
4. **Skill:** move it to `Workspace/.assistant/skills/azure-sql-to-lakebase/`, or register a Git folder, so the whole team gets it. Keep its type table generated from, or checked against, the module.

**Production-grade changes, in priority order:**
1. **Fail-safe cutover control (fixes H1):**
   - Move the registry to a control table in Delta/UC or in Lakebase, and make the defaults fail-closed (no default `pre_cutover`; explicit statuses required).
   - Refuse to TRUNCATE a table that has received writes since the last sync. Track `pg_stat_user_tables.n_tup_ins/upd/del` or a max(`updated_at`) watermark per table.
   - Revoke the sync identity's TRUNCATE/DML on cut-over tables.
   - Snapshot the root branch before each run.
2. **Atomic per-table reload (H2):**
   - `BEGIN; TRUNCATE; COPY (explicit column list); verify; COMMIT`, or load a staging table and rename-swap.
   - Detect drift by comparing source and target column lists before touching the target.
3. **Type-safe data path (H3, M-items):**
   - Replace pandas/CSV with psycopg `copy.write_row()` (or binary COPY with `set_types`) fed from Arrow batches.
   - Use `preferTimestampNTZ=true` and an explicit UTC session time zone.
   - Apply source-side casts for datetimeoffset, spatial types and rowversion, and fail loudly on unknown types.
   - Alternative to evaluate: Spark's bundled `postgresql` connector (writes are supported on serverless) writing straight to a Lakebase staging table, followed by a transactional swap. This keeps data off the driver; check its type handling and options first.
4. **Full DDL extraction from `sys.*` catalog views:**
   - NOT NULL, PKs, unique and check constraints, defaults (translated), and identity → `GENERATED BY DEFAULT AS IDENTITY` plus `setval` after load.
   - Indexes, created after the load.
   - Computed columns → generated columns where translatable.
   - Views → Postgres views.
   - Exclude backups, fix or remove the typo entry, and decide which `rte_requests` is canonical.
5. **Loud failures and observability:**
   - Record "not_found" skips, and raise at the end if any table failed, mismatched or was missing.
   - Write run results to an audit table, and alert.
   - Verify with column-level aggregates and hashes against one source snapshot.
6. **Parameterisation:**
   - Widgets or job parameters for table list/group, mode and dry run (DDL diff only).
   - One job per cutover group, with max concurrent runs = 1.
7. **Source authentication:** replace ActiveDirectoryPassword with a service principal (certificate preferred), a managed identity, or a UC JDBC connection or Lakehouse Federation connection. Remove the hard-coded UPN.
8. **Identity and privileges on Lakebase:**
   - Run as a service principal with a `databricks_create_role` role.
   - Use an owner group role, GRANTs to app roles, and `ALTER DEFAULT PRIVILEGES`.
   - Design app authentication (OAuth rotation vs native password roles plus PgBouncer).
9. **Dependencies:** a pinned `%pip install "psycopg[binary]==…" "databricks-sdk==…"` followed by `dbutils.library.restartPython()`, or pinned serverless environment dependencies. Validate the *imported* version (`databricks.sdk.version.__version__`), not just the installed distribution metadata.
10. **Robust connections:**
    - A connection factory that mints a token per connection.
    - Retry with backoff.
    - Select the endpoint by `ENDPOINT_TYPE_READ_WRITE` and fail if there is none.
    - `with psycopg.connect(...)` blocks.
    - `psycopg.sql.Identifier` for every identifier.
11. **Scale:** stream instead of `toPandas()` plus a whole-CSV buffer; read once (count from the loaded rows); optionally partition reads on an indexed numeric key with a modest `numPartitions`.
12. **App-semantics readiness:** inventory the case-insensitive and trailing-space-sensitive lookups. Use `citext`, a nondeterministic ICU collation, or `lower()`-based indexes where needed. Normalise GUID case comparisons. Rewrite T-SQL constructs in the apps.
13. **Environments:** test on Lakebase child branches, not `production`. Keep `production` for the real cutover, with a snapshot or point-in-time restore plan.
14. **Documentation:** fix every Q9 item, and generate the "Tables in Scope" table from the audit table instead of hand-typing it.

### Gaps
- Whether Lakebase "synced tables" (Delta → Postgres) could replace any of this was not evaluated. They are intended for Unity Catalog data and would not suit app-written OLTP tables, but this was not verified here.
- The Jobs defaults (max concurrent runs, retries) were not checked against current docs.

## Q11. Consolidated, severity-ranked issue register with fixes

### Takeaway
There are 8 High, 14 Medium and 10 Low issues. The two most dangerous combine: a fail-open, hand-edited cutover registry with a commit-before-copy TRUNCATE can erase post-cutover app writes or leave tables empty, and binary or rowversion/spatial data is silently or loudly corrupted by the CSV path.

### Cited Findings
| ID | Sev | Issue (basis) | Evidence | Fix |
|---|---|---|---|---|
| H1 | High | A fail-open, hand-edited registry; truncate mode reloads stale Azure SQL data over a cut-over table if its status wasn't flipped; Phase 2 omits the flip (code + inference) | [C1, C2, C7][dump] | Control table, fail-closed defaults, a write-since-last-sync tripwire, revoked DML for the sync identity, a snapshot per run |
| H2 | High | TRUNCATE (or DROP/CREATE) committed before the re-read and COPY; readers see an empty table; failures leave it empty (code + docs) | [C7][dump]; [PG TRUNCATE][pg-truncate]; [psycopg tx][psycopg-tx] | Single transaction per table, or staging + rename swap |
| H3 | High | Binary values serialised as Python `b'...'` repr: bytea COPY errors, or silent garbage for printable values and for text-fallback types (rowversion, spatial) (code + docs + probe) | [C6, C7][dump]; [env5][dbx-env5]; [py csv][py-csv]; [PG bytea][pg-bytea]; [probe] | `write_row`/binary COPY, or hex-encode as `\x…`; explicit type casts |
| H4 | High | No PK, NOT NULL, identity, defaults, unique/check constraints or indexes; post-cutover inserts get NULL keys, ON CONFLICT errors, scans on 500K–750K-row tables (code + docs) | [C6][dump]; [PG INSERT][pg-insert]; [PG CREATE TABLE][pg-createtable] | Extract full DDL from `sys.*`; identity + `setval`; indexes after the load |
| H5 | High | Silent skips (not found, or a case mismatch between the Spark filter and SQL Server) and no failure signal; jobs report success on failed/mismatched tables (code + inference) | [C6, C8][dump] | Track not-found tables; raise at the end; audit table and alerts |
| H6 | High | ActiveDirectoryPassword is deprecated, incompatible with mandatory MFA, and uses a user account as a service account (docs) | [JDBC Entra auth][ms-jdbc-entra]; [JDBC 13.2 notes][ms-jdbc-relnotes] | Service principal (certificate), managed identity, or a UC JDBC/Federation connection |
| H7 | High | No single-table targeting; a Phase 2 run reloads all 26 entries inside a 5-minute window (code + inference) | [C1, C2][dump] | Widgets/job params; per-group jobs; dry run |
| H8 | High (for user-keyed tables) / Medium otherwise | CI collation and trailing-space-insensitive equality become byte-exact; CamelCase columns renamed; uuid case changes (docs + inference) | [Logical server][ms-logical-server]; [String comparison][ms-string-compare]; [PG collation][pg-collation]; [PG uuid][pg-uuid] | Inventory app predicates; citext, ICU nondeterministic collation, or `lower()` indexes; app SQL updates |
| M1 | Med | Schema drift ignored (`IF NOT EXISTS`); COPY without a column list maps positionally (docs) | [PG CREATE TABLE][pg-createtable]; [PG COPY][pg-copy] | Pre-load column diff; explicit COPY column list |
| M2 | Med | Two separate source reads (`count()` and `toPandas()`), so the count check compares different snapshots; double source load (code + docs) | [C7][dump]; [RCSI][ms-rcsi] | Materialise once; count the loaded rows |
| M3 | Med | Row count is the only verification (code) | [C7][dump] | Column aggregates, hashes, sampled diffs |
| M4 | Med | Bigint values above 2^53 in NULL-containing columns rounded before the Int64 fix (docs + probe) | [Arrow pandas][arrow-pandas]; [probe] | Avoid float conversion (Arrow-typed iteration / `write_row`) |
| M5 | Med | Timestamps outside 1677–2262 can't be represented as datetime64[ns]; time-zone dependence; datetimeoffset offset lost (docs) | [PySpark Arrow][pyspark-arrow]; [pandas limits][pandas-ts-limits]; [PG datetime][pg-datetime]; [datetimeoffset][ms-datetimeoffset] | `preferTimestampNTZ`, UTC session, no pandas; offset-preserving casts |
| M6 | Med | Silent `text` fallback for unknown types; sql_variant fails the read; types not printed for review (code + docs) | [C6][dump]; [Spark JDBC][spark-jdbc] | Explicit mapping; fail on unknown types; print DDL |
| M7 | Med | Unquoted f-string identifiers: reserved words, special characters, collisions, 63-byte truncation (code + docs) | [C6, C7][dump]; [PG keywords][pg-keywords]; [PG lexical][pg-lexical] | `psycopg.sql.Identifier`; validation and collision checks |
| M8 | Med | Dependency install without restart; env 5 ships sdk 0.67.0, so an upgrade happens every session; validation reads disk metadata; pure psycopg needs libpq (docs) | [C3, C4][dump]; [env5][dbx-env5]; [SDK docs][dbx-sdk-python]; [psycopg install][psycopg-install] | Pinned `%pip` + `restartPython()`, or an environment spec; `psycopg[binary]` |
| M9 | Med | `endpoints[0]` may be a read-only replica; no empty check (code + docs) | [C5][dump]; [Lakebase API][lb-api] | Filter on `ENDPOINT_TYPE_READ_WRITE` |
| M10 | Med | No retry or reconnect; `rollback()` on a broken connection can abort the loop; connections not closed on error (code + inference) | [C7][dump]; [Lakebase auth][lb-auth] | Connection factory, retries, context managers |
| M11 | Med | Ownership tied to whoever runs it; no grants for apps; recreate drops grants; a job service principal cannot TRUNCATE/DROP user-owned tables (docs + inference) | [Create Postgres roles][lb-pg-roles]; [PG TRUNCATE][pg-truncate] | Owner group role, grants, default privileges, run as service principal |
| M12 | Med | A view is materialised as a static table and goes stale after cutover (code + docs) | [C2][dump]; [COLUMNS][ms-infoschema-columns] | Exclude it; recreate as a Postgres view |
| M13 | Med | Backup tables, the probable typo, and duplicate `rte_requests` targets in scope (code) | [C2][dump]; [setup] | Prune the registry; decide the canonical table |
| M14 | Med | Phase 1 tests and refreshes run on the `production` branch (code + docs) | [C2][dump]; [Branches][lb-branches] | Child branches for testing; snapshots before cutover |
| L1 | Low | 45-minute token refresh is unnecessary for open connections (expiry checked only at login) (docs) | [Lakebase auth][lb-auth] | Mint a token per connection; use `expire_time` |
| L2 | Low | A literal `\N` string becomes NULL (docs + probe) | [PG COPY][pg-copy]; [probe] | `write_row` or binary COPY |
| L3 | Low | `char(n)` padding semantics; Postgres recommends varchar/text (docs) | [PG character types][pg-char] | Map to varchar/text after an app check |
| L4 | Low | datetime2/time precision 100 ns → 1 µs (docs) | [datetime2][ms-datetime2]; [PG datetime][pg-datetime] | Accept or store as text if it matters |
| L5 | Low | Datetime `where(..., None)` loop is a no-op (inference) | [C7][dump] | Remove |
| L6 | Low | Memory: whole table in Arrow + pandas + CSV string; single-threaded read (docs) | [PySpark Arrow][pyspark-arrow]; [Databricks JDBC][dbx-jdbc] | Stream; partition modestly |
| L7 | Low | Failed COPY leaves dead tuples (docs) | [PG COPY][pg-copy] | Autovacuum/VACUUM; transactional design |
| L8 | Low | Internal hostnames, UPN, secret names and personal path embedded in a shareable `.dbc` (code) | [C1, C2][dump] | Move to config or job params; scrub the markdown |
| L9 | Low | Documentation drift (16 items in Q9) (code) | [C1–C4][dump] | Regenerate the docs from code and the audit table |
| L10 | Low | On Python 3.12 (env 5), a lone `\r` in text may not be quoted (3.12 docs omit `\r`/`\n` from QUOTE_MINIMAL), so COPY fails and the table is left empty (docs + inference) | [Python csv 3.12][py-csv-312]; [PG COPY][pg-copy] | `write_row`/binary COPY; or `lineterminator="\r\n"` plus a test |

### Inferences
- **Severity calibration:**
  - H3, M5 and M6 drop to Low if the source catalog shows no binary, rowversion, spatial, datetimeoffset or out-of-range temporal data.
  - H4's impact depends on how much the apps rely on IDENTITY and defaults.
  - Checking `sys.columns`/`sys.types` for the 26 objects is the cheapest way to re-rank these.

### Gaps
- The register's real-world impact needs the source catalog (types, constraints, identity, indexes), app SQL, and run history. None of these was available.

[dump]: <file:///C:/Users/Ajay%20T/Desktop/Projects/Incyte/Data%20Engineering/Mosaic%20Developments/context/Azure%20SQL%20to%20Lakebase%20Migration.dbc>
[setup]: <file:///C:/Users/Ajay%20T/Desktop/Projects/Incyte/Data%20Engineering/Mosaic%20Developments/context/Lakebase%20Schema%20Setup.dbc>
[probe]: <file:///C:/Users/AJAYT~1/AppData/Local/Temp/claude/c--Users-Ajay-T-Desktop-Projects-Incyte-Data-Engineering-Mosaic-Developments/510a4c41-bb82-4e88-8228-5bdefea90c58/scratchpad/csv_experiment/csv_probe.py>
[ms-jdbc-entra]: https://learn.microsoft.com/en-us/sql/connect/jdbc/connecting-using-azure-active-directory-authentication?view=sql-server-ver17#connect-using-activedirectorypassword-authentication-mode
[ms-jdbc-relnotes]: https://learn.microsoft.com/en-us/sql/connect/jdbc/release-notes-for-the-jdbc-driver?view=sql-server-ver17#132
[ms-jdbc-props]: https://learn.microsoft.com/en-us/sql/connect/jdbc/setting-the-connection-properties?view=sql-server-ver17#properties
[ms-oledb-deprecated]: https://learn.microsoft.com/en-us/sql/connect/oledb/major-version-differences?view=sql-server-ver17#deprecated-authentication-methods
[ms-infoschema-columns]: https://learn.microsoft.com/en-us/sql/relational-databases/system-information-schema-views/columns-transact-sql
[ms-nchar]: https://learn.microsoft.com/en-us/sql/t-sql/data-types/nchar-and-nvarchar-transact-sql?view=sql-server-ver17
[ms-char]: https://learn.microsoft.com/en-us/sql/t-sql/data-types/char-and-varchar-transact-sql?view=sql-server-ver17
[ms-datetime-types]: https://learn.microsoft.com/en-us/sql/t-sql/functions/date-and-time-data-types-and-functions-transact-sql?view=sql-server-ver17#date-and-time-data-types
[ms-datetime2]: https://learn.microsoft.com/en-us/sql/t-sql/data-types/datetime2-transact-sql?view=sql-server-ver17
[ms-datetimeoffset]: https://learn.microsoft.com/en-us/sql/t-sql/data-types/datetimeoffset-transact-sql?view=sql-server-ver17
[ms-rowversion]: https://learn.microsoft.com/en-us/sql/t-sql/data-types/rowversion-transact-sql?view=sql-server-ver17
[ms-string-compare]: https://learn.microsoft.com/en-us/sql/t-sql/language-elements/string-comparison-assignment?view=sql-server-ver17
[ms-logical-server]: https://learn.microsoft.com/en-us/azure/azure-sql/database/logical-servers?view=azuresql#overview
[ms-rcsi]: https://learn.microsoft.com/en-us/sql/relational-databases/sql-server-transaction-locking-and-row-versioning-guide?view=sql-server-ver17
[ms-rcsi-default]: https://learn.microsoft.com/en-us/azure/azure-sql/database/analyze-prevent-deadlocks?view=azuresql#how-deadlocks-occur
[spark-jdbc]: https://spark.apache.org/docs/latest/sql-data-sources-jdbc.html
[pyspark-arrow]: https://spark.apache.org/docs/latest/api/python/tutorial/sql/arrow_pandas.html
[arrow-pandas]: https://arrow.apache.org/docs/python/pandas.html
[pandas-to-csv]: https://pandas.pydata.org/docs/reference/api/pandas.DataFrame.to_csv.html
[pandas-ts-limits]: https://pandas.pydata.org/docs/user_guide/timeseries.html#timestamp-limitations
[py-csv]: https://docs.python.org/3/library/csv.html
[py-csv-312]: https://docs.python.org/3.12/library/csv.html
[pg-copy]: https://www.postgresql.org/docs/current/sql-copy.html
[pg-truncate]: https://www.postgresql.org/docs/current/sql-truncate.html
[pg-datetime]: https://www.postgresql.org/docs/current/datatype-datetime.html
[pg-char]: https://www.postgresql.org/docs/current/datatype-character.html
[pg-bytea]: https://www.postgresql.org/docs/current/datatype-binary.html
[pg-lexical]: https://www.postgresql.org/docs/current/sql-syntax-lexical.html
[pg-keywords]: https://www.postgresql.org/docs/current/sql-keywords-appendix.html
[pg-collation]: https://www.postgresql.org/docs/current/collation.html
[pg-createtable]: https://www.postgresql.org/docs/current/sql-createtable.html
[pg-insert]: https://www.postgresql.org/docs/current/sql-insert.html
[pg-uuid]: https://www.postgresql.org/docs/current/datatype-uuid.html
[pg-boolean]: https://www.postgresql.org/docs/current/datatype-boolean.html
[psycopg-tx]: https://www.psycopg.org/psycopg3/docs/basic/transactions.html
[psycopg-copy]: https://www.psycopg.org/psycopg3/docs/basic/copy.html
[psycopg-install]: https://www.psycopg.org/psycopg3/docs/basic/install.html
[psycopg-conn]: https://www.psycopg.org/psycopg3/docs/api/connections.html
[lb-auth]: https://learn.microsoft.com/en-us/azure/databricks/oltp/projects/authentication
[lb-api]: https://learn.microsoft.com/en-us/azure/databricks/oltp/projects/api-usage
[lb-manage-roles]: https://learn.microsoft.com/en-us/azure/databricks/oltp/projects/manage-roles
[lb-pg-roles]: https://learn.microsoft.com/en-us/azure/databricks/oltp/projects/postgres-roles
[lb-branches]: https://learn.microsoft.com/en-us/azure/databricks/oltp/projects/branches
[lb-snapshots]: https://learn.microsoft.com/en-us/azure/databricks/oltp/projects/snapshots
[lb-pitr]: https://learn.microsoft.com/en-us/azure/databricks/oltp/projects/point-in-time-restore
[dbx-serverless-limits]: https://learn.microsoft.com/en-us/azure/databricks/compute/serverless/limitations
[dbx-serverless-deps]: https://learn.microsoft.com/en-us/azure/databricks/compute/serverless/dependencies
[dbx-env5]: https://learn.microsoft.com/en-us/azure/databricks/release-notes/serverless/environment-version/five
[dbx-jdbc]: https://learn.microsoft.com/en-us/azure/databricks/connect/external-systems/jdbc
[dbx-jdbc-conn]: https://learn.microsoft.com/en-us/azure/databricks/connect/jdbc-connection
[dbx-spark-sources]: https://learn.microsoft.com/en-us/azure/databricks/connect/spark-data-sources
[dbx-federation]: https://learn.microsoft.com/en-us/azure/databricks/query-federation/
[dbx-sdk-python]: https://learn.microsoft.com/en-us/azure/databricks/dev-tools/sdk-python
[dbx-restart-python]: https://learn.microsoft.com/en-us/azure/databricks/libraries/restart-python-process
[dbx-nb-libs]: https://learn.microsoft.com/en-us/azure/databricks/libraries/notebooks-python-libraries
[dbx-genie-skills]: https://learn.microsoft.com/en-us/azure/databricks/genie-code/skills
[pypi-sdk]: https://pypi.org/project/databricks-sdk/#history

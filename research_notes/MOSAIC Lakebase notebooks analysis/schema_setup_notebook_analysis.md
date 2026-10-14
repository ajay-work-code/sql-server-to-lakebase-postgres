# "Lakebase Schema Setup" notebook: cell-by-cell analysis, translation review and defect list

Research date: 2026-10-07. STATUS: COMPLETE.

Doc versions used:
- PostgreSQL 17 docs (Lakebase's default major version, platform Q1), plus PG 18 for the UUID functions and the PG 13 release notes.
- PostgreSQL source from the `REL_17_STABLE` branch on GitHub.
- psycopg 3 docs (v3.3.x) and psycopg `master` source.
- Microsoft Learn T-SQL docs (SQL Server 2025 / `sql-server-ver17` view).
- Azure Databricks and Lakebase docs.
- All accessed 2026-10-07.

Evidence conventions:
- [SETUP] = `context/Lakebase Schema Setup.dbc` (6 cells), read from its text dump. [MIG] = `context/Azure SQL to Lakebase Migration.dbc` (8 cells). Neither notebook was executed, and no database or workspace was contacted.
- Internal identifiers (hostnames, e-mails, user IDs, Lakebase project/endpoint names) are left out on purpose. The Lakebase project ID is written as `<project>`.
- Sibling notes are cited as "platform Qn" = [lakebase_platform_2026.md](lakebase_platform_2026.md) and "ops Qn" = [operationalizing_notebooks.md](operationalizing_notebooks.md).

---

## Q1. What does each of the 6 cells do, what does it need, and what does it output? What does the notebook-level metadata reveal?

### Takeaway
The notebook runs six steps in order: a markdown header; a runtime pip check; an OAuth connection to Lakebase as the human user running it; two DDL cells (17 + 5 statements, each loop inside one transaction with per-statement try/except); and a verification query that prints table and column counts. There are no parameters, widgets, assertions, grants or outputs. The metadata shows a serverless (environment v5), IPYNB-default notebook with no stored results, created right after the migration notebook.

### Cited Findings
**Cell 1 — markdown, title "Lakebase Schema Setup"** ([SETUP](<../../context/Lakebase Schema Setup.dbc>) cell 1)
- Text: "Create empty tables in Lakebase Postgres for the MOSAIC Redesign app. These tables are written to directly by the app — they are **not** migrated from Azure SQL."
- Claims: "Self-contained: establishes its own Lakebase connection. Idempotent: safe to re-run (`CREATE TABLE IF NOT EXISTS`)."
- Target: Lakebase project `<project>` / `production` branch / `databricks_postgres` database; schema `mosaic_redesign`.
- Needs nothing; renders as markdown.

**Cell 2 — "Dependencies"** ([SETUP] cell 2; identical to [MIG] cell 3)
- Uses `importlib.metadata` and `packaging.version` to compare installed versions against the floors `psycopg>=3.1.0` and `databricks-sdk>=0.118.0`.
- If a package is missing or older, runs `subprocess.check_call([sys.executable, "-m", "pip", "install", "--quiet", ...])`. Otherwise it prints "Required packages already satisfy minimum versions."
- Prints `package==version` for both packages.
- Needs: outbound access to PyPI or a mirror, and the `packaging` module.
- Installs plain `psycopg` (no `[binary]` extra) with unpinned `>=` floors.

**Cell 3 — "Establish Lakebase Connection"** ([SETUP] cell 3; same pattern as [MIG] cell 5)
- Imports `time` (unused in this notebook), `psycopg` and `WorkspaceClient`.
- `LAKEBASE_CONFIG = {"project_id": "<project>", "branch": "production", "database": "databricks_postgres"}`, hard-coded.
- `w = WorkspaceClient()` uses the notebook's own identity (the user who runs it, or the job's run-as identity).
- `endpoints = list(w.postgres.list_endpoints(parent=f"projects/{...}/branches/{...}"))`, then `ep = endpoints[0]` with no filter on endpoint type, and `host = ep.status.hosts.host`.
- `username = w.current_user.me().user_name`; `token = w.postgres.generate_database_credential(endpoint=ep.name).token`.
- `psycopg.connect(host=host, port=5432, dbname="databricks_postgres", user=username, password=token, sslmode="require")`. Default psycopg settings apply: autocommit off, no `connect_timeout`, no `application_name`, no `options`.
- Output: "Connected to: <host> / databricks_postgres" and "Authenticated as: <user e-mail>". Both would be stored in the notebook if it is exported with outputs.
- The API pattern matches the documented, GA Lakebase Postgres API; see platform Q1. Token lifetime and login-time-only expiry: platform Q2.

**Cell 4 — "Create mosaic_redesign Tables (T-SQL source)"** ([SETUP] cell 4)
- A header comment lists the T-SQL → Postgres translations applied (see Q3) and names the source as the "MOSAIC_REDESIGN schema (T-SQL)".
- `MOSAIC_REDESIGN_DDL` holds 17 statements: `CREATE SCHEMA IF NOT EXISTS mosaic_redesign`; 4 `CREATE TABLE IF NOT EXISTS` (`mosaic_audit_log`, `mosaic_audit_heartbeat`, `mosaic_audit_viewer`, `recent_search`); 12 `CREATE [UNIQUE] INDEX IF NOT EXISTS`.
- Prints "T-SQL source DDL statements: 17" and the table list.
- Loop: `with conn.cursor() as cur: for i, ddl in enumerate(...): try: cur.execute(ddl); print("OK ...") except Exception as e: print("ERR ...", e)`, then a single `conn.commit()` after the loop.
- Output: one "OK"/"ERR" line per statement, labelled with the statement's first line (truncated to 80 characters).

**Cell 5 — "Create mosaic_redesign Tables (Delta DDL source)"** ([SETUP] cell 5)
- Header comment: source is "Databricks SQL (Delta tables from `<dev catalog>`.gold_mosaic)", with the Delta → Postgres translations applied (see Q4).
- `MOSAIC_REDESIGN_APP_DDL` holds 5 statements: `CREATE SCHEMA IF NOT EXISTS mosaic_redesign` again (commented "Schema already created by previous cell, but IF NOT EXISTS is safe"), plus `assistant_query_log`, `assistant_query_cache`, `assistant_daily_brief` and `rte_requests`. No indexes.
- Same try/except loop and single `conn.commit()`.

**Cell 6 — "Verify All Tables"** ([SETUP] cell 6)
- Runs `SELECT table_name, (SELECT count(*) FROM information_schema.columns c WHERE ...) AS column_count FROM information_schema.tables t WHERE t.table_schema = 'mosaic_redesign' ORDER BY t.table_name`.
- Prints "Tables in mosaic_redesign: N" and one line per table, then `conn.close()`.
- It checks nothing: no expected count, no assertion, no index or constraint check.
- If every statement succeeded, the expected output, worked out from the DDL, is 8 tables: `assistant_daily_brief` 24 columns, `assistant_query_cache` 9, `assistant_query_log` 16, `mosaic_audit_heartbeat` 9, `mosaic_audit_log` 42, `mosaic_audit_viewer` 7, `recent_search` 8, `rte_requests` 21 (136 columns in total). Counted by parsing the cell source.

**Notebook-level metadata** ([SETUP] dump header; meaning of the keys per ops Q1)
- `version: NotebookV1`, `language: python`, `name: "Lakebase Schema Setup"`, `dashboards: []`, `globalVars: {}`, `inputWidgets: {}` (no widgets, so no parameters), `notebookMetadata.pythonIndentUnit: 4`.
- `reposExportFormat: "JUPYTER"`: the IPYNB default format (ops Q1).
- `environmentMetadata: {"client": "5"}`: serverless environment version 5, using the deprecated key name `client`, with no dependency list recorded (ops Q1).
- `computePreferences: null` (no pinned cluster); `iPythonMetadata.nbformat: 4`.
- "Cells with embedded results/outputs: 0": exported without outputs, or with outputs cleared.
- Per-cell metadata: all 6 cells have `state: "finished"`, `commandVersion: 1`, zeroed `startTime`/`submitTime`/`finishTime`, `latestUser: "a user"`, empty `commentThread`, and titles shown (`showCommandTitle: true`).
- Notebook `origId` is 296454260807201; [MIG]'s is 296454260807200. Cell `origId`s run 6543445814429849–854 here and 6543445814429833–840 in [MIG].

### Inferences
- **Created together with the migration notebook.** The two notebooks' `origId`s are consecutive, and their cell IDs come from the same counter, a few IDs apart. They were probably created in the same session, perhaps generated or imported in one pass. Every cell has `commandVersion: 1`, which fits a single import better than iterative interactive editing. The sharing of cells 2 and 3 with [MIG] (identical dependency code; same connection code without the token-refresh timestamp) points to a common template. The unused `import time` in cell 3 is a leftover from [MIG]'s refresh logic.
- **Runtime prerequisites**
  - Serverless or classic compute with PyPI egress.
  - A workspace identity that can list endpoints and mint credentials on the project (exact project ACL unverified; platform Q2 Gaps).
  - A Postgres OAuth role for that identity on the `production` branch, with `CREATE` on database `databricks_postgres` (for `CREATE SCHEMA`) and on the schema. The project owner's role has both by default (platform Q4).
  - Network reach to the endpoint on port 5432.
- **It always targets production.** Branch and project are hard-coded, so there is no dev/test path without editing code (also noted for [MIG] in ops Q2).
- **The output wording can mislead.** Cell 4/5 print "OK" whenever `execute()` does not raise. On a re-run every statement prints "OK" although Postgres only issued "already exists, skipping" notices, and in a failed transaction earlier "OK" lines are rolled back (see Q5).

### Gaps
- Databricks does not document the DBC JSON schema (`NotebookV1`/`CommandV1`, `commandVersion`, `origId`), so the provenance reading above is an inference from these two files (ops Q1 Gaps).
- Whether serverless environment v5 already ships `psycopg` and `libpq` was not verified.
  - It matters because the pure-Python `pip install psycopg` "will need the `libpq` PostgreSQL client library installed on your client".
  - `psycopg[binary]` is "a self-contained package with all the libraries needed" ([psycopg 3: Installation](https://www.psycopg.org/psycopg3/docs/basic/install.html)).
  - Cell 2 installs the plain package, so on an image without libpq, `import psycopg` in cell 3 would fail.
  - The current psycopg docs list support for Python 3.10–3.15 and PostgreSQL 10–18 ([psycopg 3: Installation](https://www.psycopg.org/psycopg3/docs/basic/install.html)).

---

## Q2. Inventory of every object created, and the likely role of each table

### Takeaway
The notebook creates one schema, 8 tables (136 columns) and 19 indexes: 12 explicit plus 7 implicit primary-key indexes. It also creates 6 identity sequences, 1 named CHECK constraint and 11 column defaults. Seven tables are append-mostly app telemetry, cache and log tables, and one (`rte_requests`) is a request queue with no key. The audit log carries 11 B-tree indexes.

### Cited Findings
All from [SETUP](<../../context/Lakebase Schema Setup.dbc>) cells 4–5 (counts from parsing the DDL text).

**Schema**: `mosaic_redesign` (created by statement 1 of cell 4 and again by statement 1 of cell 5).

**Tables, keys, defaults and constraints**

| Table (cell) | Cols | Primary key | Identity | Explicit NOT NULL | Defaults | Other constraints |
|---|---|---|---|---|---|---|
| `mosaic_audit_log` (4) | 42 | `audit_id` bigint | `audit_id` GENERATED ALWAYS | event_time, stored_at, category, severity, action, module, event_uid | `stored_at` CURRENT_TIMESTAMP | unique index on `event_uid` |
| `mosaic_audit_heartbeat` (4) | 9 | `instance_id` varchar(40) | — | instance_id, host_name, started_at, last_seen | — | — |
| `mosaic_audit_viewer` (4) | 7 | `viewer_id` integer | `viewer_id` GENERATED ALWAYS | is_active, granted_at | `is_active` true, `granted_at` CURRENT_TIMESTAMP | `ck_mosaic_audit_viewer_who CHECK (emp_user_id IS NOT NULL OR emp_id IS NOT NULL)` |
| `recent_search` (4) | 8 | `id` bigint | `id` GENERATED ALWAYS | emp_id, entity_type, entity_id, viewed_at | — | — |
| `assistant_query_log` (5) | 16 | `log_id` bigint | `log_id` GENERATED ALWAYS | create_date | `create_date` CURRENT_TIMESTAMP | — |
| `assistant_query_cache` (5) | 9 | `cache_id` bigint | `cache_id` GENERATED ALWAYS | cache_key, norm_prompt, response_json, create_date | `create_date` CURRENT_TIMESTAMP | — |
| `assistant_daily_brief` (5) | 24 | `brief_id` bigint | `brief_id` GENERATED ALWAYS | 15 columns (all but user_id, source_freshness_json, prev_brief_id, changed_json, errors, row_count_in, model, elapsed_ms) | `window_days` 56, `is_unchanged` false, `degraded` false, `brief_date` CURRENT_DATE, `generated_at` CURRENT_TIMESTAMP | — |
| `rte_requests` (5) | 21 | **none** | — | none | `external_request_id` gen_random_uuid() | — |

**Explicit indexes (12)**
1. `ux_mosaic_audit_log_event_uid` UNIQUE (event_uid) — the dedup guard replacing `IGNORE_DUP_KEY=ON`
2. `ix_mosaic_audit_log_event_time` (event_time DESC) INCLUDE (severity, action, module, status_code)
3. `ix_mosaic_audit_log_trace` (trace_id, event_time)
4. `ix_mosaic_audit_log_session` (session_id, event_time)
5. `ix_mosaic_audit_log_request` (request_id)
6. `ix_mosaic_audit_log_parent` (parent_request_id)
7. `ix_mosaic_audit_log_emp` (emp_id, event_time DESC)
8. `ix_mosaic_audit_log_severity` (severity, event_time DESC)
9. `ix_mosaic_audit_log_action` (action, event_time DESC)
10. `ix_mosaic_audit_log_module` (module, event_time DESC)
11. `ix_mosaic_audit_heartbeat_open` (stopped_at, last_seen)
12. `ix_recent_search_user` (emp_id, entity_type, viewed_at DESC)

**Column-type mix (all 8 tables)**
- 76 `varchar(n)`, 6 `text`, 1 `char(32)`, 18 `bigint`, 11 `integer`, 1 `smallint`, 7 `boolean`, 12 `timestamp` (without time zone), 3 `date` and 1 `uuid` (sum 136).
- No `jsonb`, `timestamptz`, `numeric` or foreign keys.

**Implicit objects** (Postgres semantics)
- "Adding a `PRIMARY KEY` constraint will automatically create a unique btree index on the column or group of columns used in the constraint. That index has the same name as the primary key constraint." — [PostgreSQL 17: CREATE TABLE](https://www.postgresql.org/docs/17/sql-createtable.html)
- An identity column "will have an implicit sequence attached to it … Such a column is implicitly `NOT NULL`." — [PostgreSQL 17: CREATE TABLE](https://www.postgresql.org/docs/17/sql-createtable.html)

### Inferences
**Implicit object counts**
- 7 PK indexes and 6 identity sequences. Their default names are normally `<table>_pkey` and `<table>_<column>_seq`; the naming convention is not quoted from the docs.
- `mosaic_audit_log` therefore has 11 B-tree indexes: the PK, the `event_uid` unique index and 9 secondary indexes.
- Identity primary keys are NOT NULL even where the DDL does not say so.

**Likely role of each table** (inferred from column names; confirm with the app team)
- **`mosaic_audit_log`**: the app's request/event audit trail, probably written by HTTP middleware and app-level logging.
  - Who: emp_id, emp_user_id, user_name, user_email, emp_type, team_name, auth_via.
  - What: category, severity, action, module, description.
  - Request: http_method, path, route, query_string, request_body, status_code, duration_ms, response_bytes, response_message.
  - Client: ip_address, user_agent, client_route.
  - Error: error_type, error_message, stack_trace.
  - Database cost: db_query_count, db_ms.
  - Correlation: trace_id, session_id, request_id, parent_request_id.
  - Deployment: host_name, instance_id, app_env, app_version.
  - `event_time` (when it happened) is separate from `stored_at` (when the database received it). Together with a client-generated 32-character `event_uid` and a "silent dedup" index, this suggests events are buffered and flushed in batches with at-least-once retries.
  - The table holds personal data (e-mail, IP address, user agent) and possibly sensitive request bodies and query strings.
  - The audit-log page queries it newest-first, with filters by severity, action, module, employee, trace, session and request (index comments in cell 4).
- **`mosaic_audit_heartbeat`**: one row per running app process (`instance_id`). It is upserted at start-up, its `last_seen` is updated periodically, and `stopped_at`/`stop_reason` are set on shutdown. The `(stopped_at, last_seen)` index serves "which instances are alive or stale" queries. Its pairing with `instance_id` in the audit log lets the app spot gaps or crashes in audit coverage.
- **`mosaic_audit_viewer`**: the access-control list for the audit-log page. It names who may view it (by `emp_user_id` and/or `emp_id`), who granted access and when, and whether access is still active (`is_active`). The CHECK makes sure each row names someone. Because the table starts empty, nobody can view audit logs until rows are seeded.
- **`recent_search`**: a per-user "recently viewed or searched entities" list. `entity_type` is 3 characters (an entity code such as an HCP/HCO/territory type; unknown). The `(emp_id, entity_type, viewed_at DESC)` index serves "my last N items of type X".
- **`assistant_query_log`**: one row per AI-assistant (natural-language → SQL) request. It records the prompt, generated SQL, status, row count, timings (elapsed and DB), LLM `model`, whether the answer was `degraded` or `cached`, the error text, and the page and session it came from.
- **`assistant_query_cache`**: a response cache for the assistant. `cache_key` (20 characters, probably a truncated hash of the normalized prompt and scope) plus `norm_prompt` and the scope columns (team_id, page, prof_id, org_id) map to `response_json`. `create_date` supports age-based expiry.
- **`assistant_daily_brief`**: AI-generated daily briefings per employee and scope.
  - Scope and subject: `scope_level`, `scope_hash`, `subject_type`/`subject_id`, `field`.
  - Data used: `data_version` (snapshot of the source data) and `source_freshness_json`, over a 56-day (8-week) default window.
  - Change tracking against the previous brief: `prev_brief_id`, `is_unchanged`, `changed_json`.
  - Content and run details: `payload_json`, `degraded`/`errors`, `row_count_in`, `model`, `elapsed_ms`, `brief_date`, `generated_at`.
- **`rte_requests`**: a queue of outbound e-mail requests. "RTE" most likely means *rep-triggered email*, a pharma field-force pattern; this is an inference.
  - Who and what: contact, sales rep, territory, product, brand, subject line, salutation and closing.
  - Delivery and context: e-mail source, requested delivery date, call engagement type.
  - Status flags set by a downstream export process: `export_flag`/`export_date`, `pending_flag`, `prod_dev_flag`.
  - `content_rec_indication_top_3_flag` probably ties the request to the content-recommendation feature.

### Gaps
- The original T-SQL and Delta DDL scripts are not in the archive, so source-side details (clustered index choice, Delta column types, any VARCHAR(n) declarations) cannot be compared directly.
- Table roles are inferred from column names. No app code was available.

---

## Q3. Are the T-SQL → Postgres translations correct?

### Takeaway
The type translations are faithful, or safely wider. Three translations change behaviour, and the notebook's comments do not say so:
- **(1) `SYSUTCDATETIME()` → `CURRENT_TIMESTAMP` stored in `timestamp`.** UTC is no longer guaranteed: the stored value is the transaction start time expressed in whatever session `TimeZone` the writer has, and Lakebase does not document its default.
- **(2) `IGNORE_DUP_KEY=ON` → plain unique index.** Silent dedup moves from the database into every INSERT statement (`ON CONFLICT (event_uid) DO NOTHING`). Any writer that forgets it gets a hard error.
- **(3) `IDENTITY(1,1)` → `GENERATED ALWAYS`.** Explicit-ID backfills need `OVERRIDING SYSTEM VALUE`, while `COPY` bypasses ALWAYS. Either way the sequence must be reset after a backfill.

Also check, outside the type list: Lakebase's default `C.UTF-8` collation (case-sensitive, byte-order) versus Azure SQL Database's case-insensitive default `SQL_Latin1_General_CP1_CI_AS`, and app code that binds 0/1 integers to former `BIT` columns.

### Cited Findings
**IDENTITY(1,1) → `GENERATED ALWAYS AS IDENTITY`** (6 columns: `audit_id`, `viewer_id`, `id`, plus 3 Delta-side keys in Q4)
- "In an `INSERT` command, if `ALWAYS` is selected, a user-specified value is only accepted if the `INSERT` statement specifies `OVERRIDING SYSTEM VALUE`. If `BY DEFAULT` is selected, then the user-specified value takes precedence. … (In the `COPY` command, user-specified values are always used regardless of this setting.)" — [PostgreSQL 17: CREATE TABLE](https://www.postgresql.org/docs/17/sql-createtable.html)
- "In an `UPDATE` command, if `ALWAYS` is selected, any update of the column to any value other than `DEFAULT` will be rejected." Identity columns are "implicitly `NOT NULL`". — [PostgreSQL 17: CREATE TABLE](https://www.postgresql.org/docs/17/sql-createtable.html)
- "For an identity column defined as `GENERATED ALWAYS`, it is an error to insert an explicit value (other than `DEFAULT`) without specifying either `OVERRIDING SYSTEM VALUE` or `OVERRIDING USER VALUE`." `OVERRIDING USER VALUE` means supplied values "are ignored and the default sequence-generated values are applied". — [PostgreSQL 17: INSERT](https://www.postgresql.org/docs/17/sql-insert.html)
- "using `BY DEFAULT` results in a behavior more similar to default values … whereas `ALWAYS` provides some more protection against accidentally inserting an explicit value." Also: "Partitions inherit identity columns from the partitioned table." — [PostgreSQL 17: Identity Columns](https://www.postgresql.org/docs/17/ddl-identity-columns.html)
- Identity defaults are evaluated by `ExecEvalNextValueExpr`, which calls `nextval_internal(op->d.nextvalueexpr.seqid, false)`. The second argument means the sequence privilege check is skipped, so plain INSERT privilege on the table is enough. This closes the platform Q4 gap. — [postgres `execExprInterp.c`, REL_17_STABLE](https://github.com/postgres/postgres/blob/REL_17_STABLE/src/backend/executor/execExprInterp.c)

**DATETIME2(3)/DATETIME → `timestamp`**
- T-SQL `datetime2`: precision "0 to 7 digits, with an accuracy of 100 nanoseconds (100 ns)", default 7; "Time zone offset aware and preservation: No". — [datetime2 (Transact-SQL)](https://learn.microsoft.com/sql/t-sql/data-types/datetime2-transact-sql)
- T-SQL `datetime`: "Rounded to increments of `.000`, `.003`, or `.007` seconds"; not offset-aware. — [datetime (Transact-SQL)](https://learn.microsoft.com/sql/t-sql/data-types/datetime-transact-sql)
- Postgres: "`time`, `timestamp`, and `interval` accept an optional precision value *p* which specifies the number of fractional digits retained in the seconds field. By default, there is no explicit bound on precision. The allowed range of *p* is from 0 to 6." Also: "writing just `timestamp` [is] equivalent to `timestamp without time zone`". — [PostgreSQL 17: Date/Time Types](https://www.postgresql.org/docs/17/datatype-datetime.html)

**SYSUTCDATETIME() → `CURRENT_TIMESTAMP` (in a `timestamp without time zone` column)**
- `SYSUTCDATETIME` returns **datetime2**. It is "a nondeterministic function", and the underlying Windows API precision "is fixed at 100 nanoseconds". — [SYSUTCDATETIME (Transact-SQL)](https://learn.microsoft.com/sql/t-sql/functions/sysutcdatetime-transact-sql)
- Postgres: `CURRENT_TIMESTAMP` → `timestamp with time zone`; `LOCALTIMESTAMP` → `timestamp`. "Since these functions return the start time of the current transaction, their values do not change during the transaction." `clock_timestamp()` "changes even within a single SQL statement". `timestamptz AT TIME ZONE zone` → `timestamp without time zone`. — [PostgreSQL 17: Date/Time Functions](https://www.postgresql.org/docs/17/functions-datetime.html)
- "Conversions between `timestamp without time zone` and `timestamp with time zone` normally assume that the `timestamp without time zone` value should be taken or given as `timezone` local time." And: "In a value that has been determined to be `timestamp without time zone`, PostgreSQL will silently ignore any time zone indication." — [PostgreSQL 17: Date/Time Types](https://www.postgresql.org/docs/17/datatype-datetime.html)
- `TimeZone`: "The built-in default is `GMT`, but that is typically overridden in `postgresql.conf`; initdb will install a setting there corresponding to its system environment." — [PostgreSQL 17: Client Connection Defaults](https://www.postgresql.org/docs/17/runtime-config-client.html)
- It can be changed per session with `SET TIME ZONE`, and "The `PGTZ` environment variable is used by libpq clients to send a `SET TIME ZONE` command to the server upon connection." — [PostgreSQL 17: Date/Time Types § Time Zones](https://www.postgresql.org/docs/17/datatype-datetime.html)
- Lakebase's compatibility page does **not** document a default `TimeZone`. It says parameters with `user` context can be set "at the session, database, or role level" (`ALTER DATABASE databricks_postgres SET …`, `ALTER USER "…" SET …`), but "not at the instance level". It also states that the default collation is `C.UTF-8` with UTF8 encoding. — [Lakebase: Postgres compatibility](https://learn.microsoft.com/azure/databricks/oltp/projects/compatibility)
- psycopg 3: "Python `datetime` objects are converted to PostgreSQL `timestamp` (if they don't have a `tzinfo` set) or `timestamptz` (if they do)". `timestamptz` values come back in the connection's `TimeZone`. — [psycopg 3: Adapting basic Python types](https://www.psycopg.org/psycopg3/docs/basic/adapt.html)

**NVARCHAR(n) → `varchar(n)`, NVARCHAR(MAX) → `text`**
- T-SQL: "in **nchar(*n*)** and **nvarchar(*n*)**, the *n* defines the string length in **byte-pairs** (0-4,000). *n* never defines numbers of characters". Supplementary characters "use two byte-pairs". `nvarchar(max)` storage is up to "2^31-1 characters (2 GB)". — [nchar and nvarchar (Transact-SQL)](https://learn.microsoft.com/sql/t-sql/data-types/nchar-and-nvarchar-transact-sql)
- Postgres:
  - `varchar(n)`/`char(n)` store "up to *n* characters (not bytes)".
  - Inserting a longer string is an error unless the excess is all spaces; an explicit cast to `varchar(n)` truncates silently.
  - The maximum string is about 1 GB, and the maximum *n* is 10,485,760.
  - "There is no performance difference among these three types, apart from increased storage space when using the blank-padded type, and a few extra CPU cycles to check the length…"
  - Source: [PostgreSQL 17: Character Types](https://www.postgresql.org/docs/17/datatype-character.html)

**IGNORE_DUP_KEY=ON → unique index + `ON CONFLICT` at insert time**
- T-SQL `IGNORE_DUP_KEY`:
  - "applies only to insert operations after the index is created or rebuilt", with no effect on `CREATE INDEX`, `ALTER INDEX` or `UPDATE`.
  - ON: "A warning message occurs when duplicate key values are inserted into a unique index. Only the rows violating the uniqueness constraint aren't inserted."
  - OFF: "The entire `INSERT` statement is rolled back."
  - It cannot be ON for non-unique or filtered indexes.
  - The warning text is "Msg 3604 … Duplicate key was ignored."
  - Source: [CREATE INDEX (Transact-SQL)](https://learn.microsoft.com/sql/t-sql/statements/create-index-transact-sql)
- Postgres `ON CONFLICT`:
  - It "specifies an alternative action to raising a unique violation or exclusion constraint violation error".
  - "For `ON CONFLICT DO NOTHING`, it is optional to specify a *conflict_target*". With a target such as `(event_uid)`, Postgres infers the matching unique index as the arbiter.
  - The "cardinality violation" (same row affected twice) rule is stated only for `DO UPDATE`.
  - "all forms of `ON CONFLICT` also require `SELECT` privilege on any column whose values are read."
  - Source: [PostgreSQL 17: INSERT](https://www.postgresql.org/docs/17/sql-insert.html)
- The notebook's own comment says: "SQL Server IGNORE_DUP_KEY=ON has no Postgres equivalent on the index. Use ON CONFLICT (event_uid) DO NOTHING at insert time for silent dedup." — [SETUP](<../../context/Lakebase Schema Setup.dbc>) cell 4

**Identifiers, collation and client time-zone defaults**
- "Key words and unquoted identifiers are case-insensitive." Also: "Quoting an identifier also makes it case-sensitive, whereas unquoted names are always folded to lower case." For example, `FOO`, `foo` and `"foo"` are the same, but `"Foo"` differs. Identifiers longer than 63 bytes are truncated. — [PostgreSQL 17: Lexical Structure](https://www.postgresql.org/docs/17/sql-syntax-lexical.html)
- Azure SQL Database collation is, "By default, `SQL_Latin1_General_CP1_CI_AS`", and cannot be changed after deployment. — [Limitations in SQL database in Microsoft Fabric (Azure SQL Database column)](https://learn.microsoft.com/fabric/database/sql/limitations)
- `SQL_Latin1_General_CP1_CI_AS` "is a case-insensitive, accent-sensitive collation". — [EF Core: Collations and case sensitivity](https://learn.microsoft.com/ef/core/miscellaneous/collations-and-case-sensitivity)
- `SYSUTCDATETIME` returns a datetime2 whose "date and time is returned as UTC time (Coordinated Universal Time)". — [SYSUTCDATETIME (Transact-SQL)](https://learn.microsoft.com/sql/t-sql/functions/sysutcdatetime-transact-sql)
- RPostgres `dbConnect(timezone=)`: "Sets the timezone for the connection. The default is `"UTC"`. If `NULL` then no timezone is set, which defaults to the server's time zone." — [RPostgres: Postgres()](https://rpostgres.r-dbi.org/reference/Postgres.html)

### Inferences
**Identity**
- Correct and stricter, matching SQL Server's default `IDENTITY` behaviour of rejecting explicit values. (SQL Server `SET IDENTITY_INSERT` was not re-verified this session.)
- Consequences:
  - (a) A one-off backfill of history from Azure SQL (old audit rows, existing audit-viewer ACL rows) with original IDs needs `INSERT … OVERRIDING SYSTEM VALUE`.
  - (b) Loading through `COPY`, which is what the migration notebook uses, silently accepts explicit IDs but never advances the sequence. The next app INSERT then collides with an existing PK (duplicate-key error) until the sequence is moved past `max(id)`. Inferred from defaults only calling `nextval()` when no value is supplied; not tested.
  - (c) An ORM or app layer that sends `id = NULL` or explicit IDs fails under ALWAYS.
- Fixes:
  - Keep ALWAYS for app-generated keys.
  - For a planned backfill, either use `OVERRIDING SYSTEM VALUE` plus a sequence reset (`SELECT setval(pg_get_serial_sequence('mosaic_redesign.mosaic_audit_log','audit_id'), (SELECT max(audit_id) FROM mosaic_redesign.mosaic_audit_log));`), or declare `BY DEFAULT` for tables that are seeded from history.
  - No sequence GRANT is needed for plain INSERTs, but apps should read new keys with `INSERT … RETURNING audit_id`, which needs SELECT on that column, not `currval()`.

**Datetime precision**
- `timestamp` (microseconds) is a lossless widening of DATETIME2(3) (milliseconds) and DATETIME (about 3.33 ms).
- Both sides are offset-naive, so the type mapping is semantically faithful.
- Use `timestamp(3)` only if exact millisecond parity with old data matters, for example equality joins during a parallel run. Low impact.

**SYSUTCDATETIME → CURRENT_TIMESTAMP is the real semantic change**
- The column default yields the `timestamptz` transaction-start instant, converted to `timestamp` in the session's `TimeZone`.
- So `stored_at`, `granted_at`, `create_date` and `generated_at` hold UTC only if every writer's session runs in UTC.
- Under `America/New_York`, for example, the same instant is stored four or five hours different.
- Rows written by different clients (app vs notebook vs ad-hoc SQL) can then disagree. Nothing in the data records which zone was used.
- This is a live risk on Posit:
  - R apps on RPostgres get a UTC session by default.
  - Python apps on psycopg get the server or database default, unless `PGTZ` or `options='-c TimeZone=UTC'` is set.
  - This notebook runs with no explicit zone.
  - The same column can therefore receive UTC from R and something else from Python if the Lakebase default is not UTC.
- The app also supplies `event_time` itself:
  - If it binds an *aware* Python datetime, psycopg sends `timestamptz`, and Postgres converts it to session-local time on assignment.
  - If it binds a *naive* one, the value is stored as-is.
  - Either way, correctness depends on client conventions.
- Second, smaller difference: `CURRENT_TIMESTAMP` is per *transaction*. A batch of audit inserts in one long transaction all get the same `stored_at`. SQL Server evaluates `SYSUTCDATETIME()` per statement; that per-statement behaviour of T-SQL defaults was not re-verified.
- Fixes, in order of preference:
  - (1) Use `timestamptz` for every point-in-time column, with `DEFAULT now()` (Q6).
  - (2) If `timestamp` stays, use `DEFAULT (now() AT TIME ZONE 'UTC')`, or `clock_timestamp() AT TIME ZONE 'UTC'` for per-row receipt time. That makes the default independent of the session.
  - (3) Pin `ALTER DATABASE databricks_postgres SET TimeZone = 'UTC'` on every branch as a guard rail. The Lakebase docs allow database- and role-level settings.
  - (4) Run `SHOW TimeZone;` on the target branch to learn the current default.

**NVARCHAR(n) → varchar(n)**
- Safe: Postgres counts characters (code points), while SQL Server counts UTF-16 code units. Any string that fit in `NVARCHAR(n)` fits in `varchar(n)`.
- The comment "(Postgres is Unicode-native)" holds for Lakebase's UTF8 encoding.
- The real difference is **collation**. Lakebase defaults to `C.UTF-8`, which is case-sensitive with code-point ordering. Azure SQL Database defaults to the case-insensitive `SQL_Latin1_General_CP1_CI_AS`. Confirm the source with `SELECT DATABASEPROPERTYEX(DB_NAME(), 'Collation')`.
- Equality lookups on `emp_user_id`, `user_email`, `entity_id`, `cache_key` and the like become case-sensitive. The `mosaic_audit_viewer` permission check (`WHERE emp_user_id = :login`) is the riskiest: a case mismatch silently denies access.
- Fix: normalize to lower case on write and read, or declare those columns with an ICU non-deterministic, case-insensitive collation. Lakebase supports ICU.
- Also: Postgres raises an error on over-length strings, much like SQL Server's truncation error, so app code that relied on silent truncation will now fail.

**NVARCHAR(MAX) → text and BIT → boolean**
- `text` is correct. Its ~1 GB per-value cap is below SQL Server's 2 GB, which is irrelevant for stack traces, request bodies or JSON, but the app should cap `request_body` and `stack_trace` anyway (Q6).
- `boolean` is correct. App code that binds Python/R integers 0/1, the habit with `BIT`, should be expected to fail with "column … is of type boolean but expression is of type integer". Postgres has no implicit integer → boolean assignment cast. This is from general Postgres knowledge, not re-verified this session.
- Fix: bind native booleans.

**IGNORE_DUP_KEY**
- The translation is the only possible one, but it changes where duplicates are handled.
- **Before:** any client, bulk API or ad-hoc INSERT got silent dedup from the index.
- **After:** every INSERT path into `mosaic_audit_log` must say `INSERT … ON CONFLICT (event_uid) DO NOTHING`.
- App changes required:
  - (a) Update the audit-flush code and any retry or replay tool.
  - (b) Bulk paths cannot rely on `COPY`, which has no ON CONFLICT and fails the whole batch on one duplicate. Use multi-row `INSERT … VALUES … ON CONFLICT DO NOTHING`, or COPY into a temporary/staging table followed by `INSERT … SELECT … ON CONFLICT DO NOTHING`.
  - (c) Exception handling that looked for SQL Server 2601/2627/3604 must map to SQLSTATE `23505` (`unique_violation`).
  - (d) The affected-row count now excludes skipped duplicates, so any "rows written" telemetry changes meaning.
  - (e) The app role needs SELECT on `event_uid` (granted anyway in Q7).
- Optional: write `ON CONFLICT ON CONSTRAINT …` against a named UNIQUE *constraint* instead of a bare unique index, which makes the arbiter explicit.

**Lower-case snake_case identifiers**
- Correct, and required for clean Postgres use, since unquoted identifiers fold to lower case (verified above).
- App SQL ported from T-SQL that *quotes* mixed-case names (for example `"AuditId"`, or `DBI::Id(schema = "MOSAIC_REDESIGN")`) will fail.
- Every column rename (for example `AuditId` → `audit_id`) also needs a matching change in app code, ORM mappings and R `dplyr` code.

### Gaps
- The original T-SQL DDL is not in the archive, so which columns were DATETIME vs DATETIME2(3), and the exact original names, cannot be checked.
- The source Azure SQL database's actual collation was not confirmed. Only the Azure SQL Database default (`SQL_Latin1_General_CP1_CI_AS`) is documented.
- Lakebase's default `TimeZone` is undocumented. Run `SHOW TimeZone` on each branch.
- Client time-zone behaviour was not verified for the R `odbc` package with a PostgreSQL ODBC driver, or for JDBC clients. RPostgres (UTC default) and libpq/psycopg (no zone unless `PGTZ`) are verified above.
- SQL Server's `SET IDENTITY_INSERT` semantics, and whether T-SQL column defaults evaluate `SYSUTCDATETIME()` per statement, were not re-verified this session.

---

## Q4. Are the Databricks SQL (Delta) → Postgres translations correct?

### Takeaway
Several mappings are exact: `INT → integer`, BIGINT, BOOLEAN, DATE, `CAST(current_timestamp() AS DATE) → CURRENT_DATE`, and `uuid() → gen_random_uuid()` (core since PG 13, so available on all Lakebase versions 16/17/18). Four points need attention:
- **(1) The header says `STRING → text`, but the DDL uses `varchar(n)` for 34 of the 37 string columns.** Either the Delta DDL declared VARCHAR(n), which Delta supports and length-checks, or the lengths were re-imposed from elsewhere. If the source columns were STRING, the Postgres tables now reject values that Delta accepted.
- **(2) Delta `TIMESTAMP` → `timestamp`.** Delta `TIMESTAMP` is session-time-zone/instant semantics. Databricks' own Lakebase tooling maps it to `timestamptz` and maps only `TIMESTAMP_NTZ` to `timestamp`.
- **(3) `uuid()` STRING → native `uuid` type.** This is stricter and better, but the column stays nullable and non-unique.
- **(4) Same-time-zone dependence for defaults as in Q3.**

### Cited Findings
- Cell 5 header: "STRING → text; current_timestamp() → CURRENT_TIMESTAMP; CAST(current_timestamp() AS DATE) → CURRENT_DATE; uuid() → gen_random_uuid(); INT → integer; USING DELTA → removed". The source is "Delta tables from `<dev catalog>`.gold_mosaic". — [SETUP](<../../context/Lakebase Schema Setup.dbc>) cell 5
- Of cell 5's 37 string columns, 34 are `varchar(n)` (lengths 4 to 2000) and only 3 are `text` (`sql_text`, `response_json`, `payload_json`). Counted from the DDL. — [SETUP] cell 5
- Databricks `STRING`: "The type supports character sequences of any length greater or equal to 0." — [STRING type](https://learn.microsoft.com/azure/databricks/sql/language-manual/data-types/string-type)
- Databricks types:
  - `TIMESTAMP` "Represents values comprising values of fields year, month, day, hour, minute, and second, with the session local timezone."
  - `TIMESTAMP_NTZ`: "All operations are performed without taking any time zone into account."
  - `INT` is a "4-byte signed integer".
  - Source: [Data types](https://learn.microsoft.com/azure/databricks/sql/language-manual/sql-ref-datatypes)
- Delta enforces CHAR/VARCHAR lengths. It has an error class: "Value "`<value>`" exceeds char/varchar type length limitation." — [Error conditions in Azure Databricks](https://learn.microsoft.com/azure/databricks/error-messages/error-classes)
- Databricks' Lakebase streaming sink maps types when it creates tables: `StringType → text`, `VarcharType(n) → varchar(n)`, `CharType(n) → char(n)`, `TimestampType → timestamptz`, `TimestampNTZType → timestamp`, `IntegerType → integer`, `LongType → bigint`, `BooleanType → boolean`, `DateType → date`. — [Connect to Lakebase (Structured Streaming): data type mappings](https://learn.microsoft.com/azure/databricks/structured-streaming/lakebase#data-type-mappings)
- Synced tables map `STRING` to `TEXT` by default, with an optional `varchar(n)` override, and `TIMESTAMP → TIMESTAMP WITH TIME ZONE` (platform Q5). — [Serve lakehouse data with synced tables](https://learn.microsoft.com/azure/databricks/oltp/projects/sync-tables#data-types-and-compatibility)
- The Lakeflow Connect SQL Server connector lands `NVARCHAR`/`VARCHAR` as `STRING`, `DATETIME`/`DATETIME2` as `TIMESTAMP`, and `UNIQUEIDENTIFIER` as `STRING`. — [SQL Server connector reference](https://learn.microsoft.com/azure/databricks/ingestion/lakeflow-connect/sql-server-reference)
- Databricks `uuid()` "Returns … A `STRING` formatted as a canonical UUID 36-character string. The function is non-deterministic." — [uuid function](https://learn.microsoft.com/azure/databricks/sql/language-manual/functions/uuid)
- `current_date()` and `current_timestamp()` return a DATE and a TIMESTAMP "at the start of query evaluation". `current_timezone()` "Returns the current session local timezone." — [current_date](https://learn.microsoft.com/azure/databricks/sql/language-manual/functions/current_date); [current_timestamp](https://learn.microsoft.com/azure/databricks/sql/language-manual/functions/current_timestamp); [current_timezone](https://learn.microsoft.com/azure/databricks/sql/language-manual/functions/current_timezone)
- PostgreSQL 13 (released 2020-09-24): "Add function `gen_random_uuid()` to generate version-4 UUIDs … Previously UUID generation functions were only available in the external modules uuid-ossp and pgcrypto." — [PostgreSQL 13 release notes](https://www.postgresql.org/docs/13/release-13.html)
- PostgreSQL 18 documents `gen_random_uuid()` and `uuidv4()` ("Generates a version 4 (random) UUID") and `uuidv7()` ("version 7 (time-ordered) UUID"). — [PostgreSQL 18: UUID Functions](https://www.postgresql.org/docs/18/functions-uuid.html)
- Lakebase: "The `pg_uuidv7` extension is not available on Lakebase. On Postgres 18, use the built-in `uuidv7()` and `uuidv4()` … On Postgres 16 and Postgres 17, use `gen_random_uuid()`". — [Lakebase: Postgres extensions](https://learn.microsoft.com/azure/databricks/oltp/projects/extensions)
- The Postgres `uuid` type is a 128-bit value. Input may omit hyphens (e.g. `a0eebc999c0b4ef8bb6d6bb9bd380a11`) or use upper case or braces, but "Output is always in the standard form". — [PostgreSQL 17: UUID Type](https://www.postgresql.org/docs/17/datatype-uuid.html)
- psycopg adapts Python `uuid.UUID` to and from Postgres `uuid`. — [psycopg 3: Adapting basic Python types](https://www.psycopg.org/psycopg3/docs/basic/adapt.html)

### Inferences
**Where the `varchar(n)` lengths probably came from**
- (a) The Delta DDL declared `VARCHAR(n)`, which Delta length-checks. The author kept those lengths and applied `STRING → text` only to the three true STRING columns. In that case the header is incomplete rather than wrong.
- (b) The gold tables originate in Azure SQL. Ingestion paths such as Lakeflow Connect turn NVARCHAR into STRING and drop lengths, so the author may have re-imposed SQL Server lengths. This is plausible for `rte_requests`: its narrow `varchar(4)` `request_type` and `varchar(255)` `subject_line` look like SQL Server DDL, and a same-named table is migrated from Azure SQL by [MIG].
- (c) The lengths were chosen by hand.
- Which applies cannot be decided from the archive. Run `SHOW CREATE TABLE <dev catalog>.gold_mosaic.<table>` for all four tables.
- If the sources are STRING, Postgres now rejects over-length values that Delta accepted. A long user prompt (`prompt_text varchar(1000)`, `norm_prompt varchar(1000)`), a long error (`error_text varchar(500)`) or JSON in `changed_json varchar(2000)` would make the INSERT fail, and the log or cache row is lost.
- Fix: use `text` for free-form and JSON columns, and keep `varchar(n)` only where a business rule really bounds the value. Postgres has no performance difference between `varchar(n)` and `text` (Q3).

**TIMESTAMP**
- If the Delta columns are `TIMESTAMP`, not `TIMESTAMP_NTZ`, the faithful target is `timestamptz`. Databricks itself uses that for synced tables and the streaming sink.
- Mapping to `timestamp` drops the instant semantics and makes stored values depend on the writer's session zone (Q3).
- It would also mismatch any future synced copy or Lakebase→Delta feed of the same entities.
- Fix: map `TIMESTAMP → timestamptz` (`create_date`, `data_version`, `generated_at`, `request_date`) and `TIMESTAMP_NTZ → timestamp`.

**`uuid()` → `uuid DEFAULT gen_random_uuid()`**
- A good translation: a native 16-byte validated type replaces a 36-character string.
- Caveats:
  - (a) Writers that send non-UUID strings are rejected. Writers that send upper-case, braced or hyphen-less forms get normalized output, so string comparisons in app code may change.
  - (b) The default fires only when the column is omitted. The column is nullable and not unique, so explicit NULLs and duplicates are possible (Q6).
  - (c) A v4 random UUID as a future primary key spreads inserts across the B-tree. That is irrelevant at this table's size. `uuidv7()` exists only on PG 18, and `pg_uuidv7` is unavailable.
- No extension is needed on PG 16–18 (platform Q1).

**INT → integer, BIGINT/BOOLEAN/DATE**
- These are exact.

**`CAST(current_timestamp() AS DATE)` → `CURRENT_DATE`**
- Equivalent in intent. Databricks evaluates it at query start in the Spark session zone; Postgres evaluates `CURRENT_DATE` in the Postgres session `TimeZone`.
- `brief_date` agrees with Databricks-side computations only if both sessions run in the same zone, preferably UTC (Q3).

**"USING DELTA → removed" and "identifiers already lowercase"**
- Both correct.
- Delta-side informational constraints, table properties and column comments, if any, are not carried over. Add `COMMENT ON` for documentation.

### Gaps
- The Delta DDL itself is not in the archive. Whether the source columns were STRING or VARCHAR(n), and TIMESTAMP or TIMESTAMP_NTZ, is unverified.
- Databricks' default Spark session time zone on serverless was not verified this session.

---

## Q5. Transaction and error handling, idempotency and "safe to re-run"

### Takeaway
- Each DDL cell runs as one psycopg transaction. If any statement fails, every later statement in that cell fails with "current transaction is aborted", and the final `conn.commit()` silently becomes a ROLLBACK. The cell still finishes green, and nothing from it persists, including statements that printed "OK".
- "Idempotent / safe to re-run" holds only in a narrow sense: the same DDL, re-run by the same owning role. `IF NOT EXISTS` never reconciles a changed definition.
- A re-run by any other identity fails, because Postgres checks privileges and ownership before the existence check.
- A re-run against a live database briefly takes write-blocking `SHARE` locks on three app tables.

### Cited Findings
**Notebook code**
- Cells 4 and 5 both run `try: cur.execute(ddl) ... except Exception as e: print(...)` per statement, then call `conn.commit()` once after the loop. No `rollback()`, no re-raise, no savepoints, and `autocommit` is left at its default. — [SETUP](<../../context/Lakebase Schema Setup.dbc>) cells 3–5

**psycopg 3** (docs v3.3.x, accessed 2026-10-07)
- "By default, any database operation will start a new transaction. As a consequence, changes made by any cursor of the connection will not be visible until `Connection.commit()` is called, and will be discarded by `Connection.rollback()`." — [psycopg 3: Transactions management](https://www.psycopg.org/psycopg3/docs/basic/transactions.html)
- "If a database operation fails, the server will refuse further commands, until a `rollback()` is called." The resulting error is "InFailedSqlTransaction: current transaction is aborted, commands ignored until end of transaction block". — [psycopg 3: Transactions management](https://www.psycopg.org/psycopg3/docs/basic/transactions.html)
- `with conn.transaction()` commits on exit or rolls back on exception, and "Transaction blocks can be also nested (internal transaction blocks are implemented using SAVEPOINT)". Autocommit "may be required to run operations that cannot be executed inside a transaction". — [psycopg 3: Transactions management](https://www.psycopg.org/psycopg3/docs/basic/transactions.html)
- In psycopg's source, `_commit_gen` returns early only `if self.pgconn.transaction_status == IDLE`. Otherwise it sends `COMMIT`. `_exec_command` raises only when `result.status != COMMAND_OK and result.status != TUPLES_OK`. There is no special handling of the INERROR state. — [psycopg `_connection_base.py` (master, accessed 2026-10-07)](https://github.com/psycopg/psycopg/blob/master/psycopg/psycopg/_connection_base.py)

**PostgreSQL server**
- For `COMMIT`, PostgreSQL 17 runs `if (!EndTransactionBlock(stmt->chain)) { /* report unsuccessful commit in qc */ ... SetQueryCompletion(qc, CMDTAG_ROLLBACK, 0); }`. A COMMIT of an aborted transaction therefore completes normally, with command tag `ROLLBACK` instead of an error. — [postgres `src/backend/tcop/utility.c`, REL_17_STABLE](https://github.com/postgres/postgres/blob/REL_17_STABLE/src/backend/tcop/utility.c)
- `CREATE TABLE IF NOT EXISTS`: "Do not throw an error if a relation with the same name already exists. A notice is issued in this case. Note that there is no guarantee that the existing relation is anything like the one that would have been created." — [PostgreSQL 17: CREATE TABLE](https://www.postgresql.org/docs/17/sql-createtable.html)
- `CREATE INDEX IF NOT EXISTS`: "...there is no guarantee that the existing index is anything like the one that would have been created. Index name is required when `IF NOT EXISTS` is specified." — [PostgreSQL 17: CREATE INDEX](https://www.postgresql.org/docs/17/sql-createindex.html)
- `CREATE SCHEMA ... IF NOT EXISTS`:
  - `CreateSchemaCommand` first checks `object_aclcheck(DatabaseRelationId, MyDatabaseId, saved_uid, ACL_CREATE)` and `check_can_set_role(...)`, and only then looks the schema up for `if_not_exists`.
  - The code comment reads: "We could do this before making the permissions checks, but since CREATE TABLE IF NOT EXISTS makes its creation-permission check first, we do likewise."
  - Source: [postgres `src/backend/commands/schemacmds.c`, REL_17_STABLE](https://github.com/postgres/postgres/blob/REL_17_STABLE/src/backend/commands/schemacmds.c)
- `CREATE INDEX` processing starts with `lockmode = stmt->concurrent ? ShareUpdateExclusiveLock : ShareLock; relid = RangeVarGetRelidExtended(stmt->relation, lockmode, 0, RangeVarCallbackOwnsRelation, NULL);`. This happens before `DefineIndex()`, so the table lock and the table-ownership check come before any "already exists" check. — [postgres `src/backend/tcop/utility.c`, REL_17_STABLE](https://github.com/postgres/postgres/blob/REL_17_STABLE/src/backend/tcop/utility.c)
- "Normally PostgreSQL locks the table to be indexed against writes and performs the entire index build with a single scan of the table." Also: "A regular `CREATE INDEX` command can be performed within a transaction block, but `CREATE INDEX CONCURRENTLY` cannot." — [PostgreSQL 17: CREATE INDEX](https://www.postgresql.org/docs/17/sql-createindex.html)

### Inferences
**Failure walk-through: statement *k* of cell 4 fails**
- Possible causes include permission denied, a lock or statement timeout, or a type or syntax error after someone edits the DDL.
- Statements *k+1…17* each raise `InFailedSqlTransaction`, which is caught and printed as `ERR … current transaction is aborted`.
- `conn.commit()` sends COMMIT and the server rolls back. psycopg sees `COMMAND_OK` and returns normally.
- Result: none of cell 4's objects exist, although lines 1…*k−1* printed "OK", and the cell (or a job task) is reported as successful.
- Cell 5 then starts a fresh transaction on the now-idle connection and may succeed, leaving a half-built schema (4 of 8 tables).
- Cell 6 prints "Tables in mosaic_redesign: 4" and asserts nothing.
- Exception: if the failure is a dropped connection (for example scale-to-zero or failover), `commit()` itself raises, so the cell fails loudly.

**All-or-nothing DDL is good; swallowing the error is the defect**
- Postgres DDL is transactional, so one transaction for the whole bootstrap is the right unit.
- Fix: delete the try/except, or re-raise.
- Run all 22 statements inside `with conn.transaction():`, so any error rolls everything back and raises.
- Run `SET LOCAL lock_timeout = '5s'` first, so a busy table fails fast instead of queueing.
- End with an assertion against the catalogs (see Q9, item H2).

**Idempotency, as actually delivered**
- Same DDL, same owner role: a true no-op. Postgres emits "already exists, skipping" notices, which the notebook does not show, and the notebook prints "OK" for every statement.
- Changed DDL text:
  - A changed column list, type, NOT NULL, default or CHECK is silently ignored.
  - An index whose definition changed but whose name did not is silently ignored.
  - A renamed index creates a second, duplicate index.
  - Cell 6 cannot detect any of these, because it only counts columns.
  - Schema evolution therefore needs versioned migrations (ops Q4).
- Re-run by a different identity (another engineer, or a job running as a service principal):
  - Statement 1 (`CREATE SCHEMA IF NOT EXISTS`) fails without `CREATE` on `databricks_postgres`.
  - Each `CREATE TABLE IF NOT EXISTS` fails without `CREATE` on the schema.
  - Each `CREATE INDEX IF NOT EXISTS` fails unless the identity owns, or is a member of the role that owns, the table.
  - Combined with the swallowed errors, a "verification re-run" by a non-owner prints ERR lines but still ends green.
- Re-run on a live database: each of the 12 `CREATE INDEX IF NOT EXISTS` statements takes a `SHARE` lock on its table (`mosaic_audit_log`, `mosaic_audit_heartbeat`, `recent_search`) and holds it until the cell's single commit.
  - Normally that lasts milliseconds.
  - Behind a long-running app transaction, the DDL waits, and new app INSERTs and UPDATEs queue behind it. This follows from standard Postgres lock queuing; not tested on Lakebase.
  - Hence `lock_timeout`, and a rule of running the notebook only in a maintenance window or never against production.

### Gaps
- The psycopg source was read from `master` (3.3.x dev). The `_commit_gen` logic is assumed unchanged since 3.1, the notebook's floor, but older tags were not checked.
- Not verified on a live Lakebase endpoint, since nothing was executed. Lakebase is described as standard Postgres 16/17/18 (platform Q1), so community-Postgres semantics are assumed.

---

## Q6. Design review for an app-facing OLTP database on Lakebase

### Takeaway
The schema works for a first release, but it is tuned for SQL Server habits rather than for an append-heavy Postgres OLTP store on Lakebase:
- Every audit insert maintains **11 B-tree indexes**.
- There is no retention or partitioning plan. `pg_cron` and `pg_partman` are not offered on Lakebase, so retention must run from outside, for example as a Databricks job.
- `rte_requests` has **no key at all**.
- Upsert-style tables (cache, daily brief, recent search, viewer ACL) lack the unique keys their access patterns need.
- JSON is stored as length-capped `varchar`/`text`.
- Time columns are `timestamp` rather than `timestamptz`.

None of this blocks bootstrap, and most of it can be added later with versioned migrations (ops Q4).

### Cited Findings
**Indexes**
- `INCLUDE`:
  - "A non-key column cannot be used in an index scan search qualification".
  - "an index-only scan can return the contents of non-key columns without having to visit the index's table".
  - Supported for B-tree, GiST and SP-GiST.
  - "It's wise to be conservative about adding non-key columns … non-key columns duplicate data from the index's table and bloat the size of the index".
  - Source: [PostgreSQL 17: CREATE INDEX](https://www.postgresql.org/docs/17/sql-createindex.html)
- An index-only scan "will be a win only if a significant fraction of the table's heap pages have their all-visible map bits set. But tables in which a large fraction of the rows are unchanging are common enough". — [PostgreSQL 17: Index-Only Scans and Covering Indexes](https://www.postgresql.org/docs/17/indexes-index-only-scans.html)
- A B-tree "can also be scanned backward, producing output satisfying `ORDER BY x DESC`… In single-column indexes the options are indeed redundant, but in multicolumn indexes they can be useful." — [PostgreSQL 17: Indexes and ORDER BY](https://www.postgresql.org/docs/17/indexes-ordering.html)
- `CONCURRENTLY` builds "without taking any locks that prevent concurrent inserts, updates, or deletes", but "requires more total work" and can leave an "invalid" index on failure.
- Partial indexes (`WHERE`) contain "entries for only a portion of a table".
- For unique indexes, "The default is that [nulls] are distinct, so that a unique index could contain multiple null values"; `NULLS NOT DISTINCT` changes that.
- Source: [PostgreSQL 17: CREATE INDEX](https://www.postgresql.org/docs/17/sql-createindex.html)
- HOT updates require that "The update does not modify any columns referenced by the table's indexes, not including summarizing indexes" and enough page free space. Lowering `fillfactor` increases the likelihood. — [PostgreSQL 17: Heap-Only Tuples](https://www.postgresql.org/docs/17/storage-hot.html)
- Lakebase: cumulative statistics "are not saved when a compute … scales to zero". — [Lakebase: Postgres compatibility](https://learn.microsoft.com/azure/databricks/oltp/projects/compatibility)

**Partitioning and retention**
- "Dropping an individual partition using `DROP TABLE`, or doing `ALTER TABLE DETACH PARTITION`, is far faster than a bulk operation. These commands also entirely avoid the `VACUUM` overhead caused by a bulk `DELETE`."
- Rule of thumb: partitioning pays off when "the size of the table should exceed the physical memory of the database server".
- "To create a unique or primary key constraint on a partitioned table, … the constraint's columns must include all of the partition key columns."
- Rows with no matching partition raise an error, and "an appropriate partition must be added manually".
- Source: [PostgreSQL 17: Table Partitioning](https://www.postgresql.org/docs/17/ddl-partitioning.html)
- Lakebase's extension list includes `citext`, `pg_jsonschema`, `pg_stat_statements`, `moddatetime` and `btree_gin`. It does **not** list `pg_cron` or `pg_partman`. — [Lakebase: Postgres extensions](https://learn.microsoft.com/azure/databricks/oltp/projects/extensions)
- Lakebase CDF needs `REPLICA IDENTITY FULL` on each table and does not support partitioned tables (platform Q5). — [Lakebase Change Data Feed](https://learn.microsoft.com/azure/databricks/oltp/projects/lakebase-cdf)

**Keys, replication and the no-PK table**
- Replica identity:
  - "A published table must have a *replica identity* configured in order to be able to replicate `UPDATE` and `DELETE` operations … By default, this is the primary key".
  - `FULL` "should only be used as a fallback".
  - "If a table without a replica identity is added to a publication that replicates `UPDATE` or `DELETE` operations then subsequent `UPDATE` or `DELETE` operations will cause an error on the publisher. `INSERT` operations can proceed regardless".
  - Source: [PostgreSQL 17: Logical Replication — Publication](https://www.postgresql.org/docs/17/logical-replication-publication.html)
- `REPLICA IDENTITY DEFAULT` "Records the old values of the columns of the primary key, if any". `ADD … PRIMARY KEY USING INDEX` can promote an existing unique index; this needs a full scan if the columns are not yet NOT NULL. — [PostgreSQL 17: ALTER TABLE](https://www.postgresql.org/docs/17/sql-altertable.html)
- Lakebase: "Replicating data to or from a Lakebase database using native Postgres logical replication is not yet available." — [Lakebase: Postgres compatibility](https://learn.microsoft.com/azure/databricks/oltp/projects/compatibility)
- Synced tables need a primary key (platform Q5). `ON CONFLICT (cols)` arbiters are inferred only from unique indexes containing exactly those columns. — [PostgreSQL 17: INSERT](https://www.postgresql.org/docs/17/sql-insert.html)
- [MIG]'s plan states "No foreign keys (no migration order constraints)" for the app tables. — [MIG](<../../context/Azure SQL to Lakebase Migration.dbc>) cell 1

**Types**
- JSON types:
  - They "enforce that each stored value is valid according to the JSON rules".
  - `jsonb` is "slightly slower to input" but "significantly faster to process" and supports GIN indexing.
  - "most applications should prefer to store JSON data as `jsonb`, unless there are quite specialized needs, such as legacy assumptions about ordering of object keys."
  - Source: [PostgreSQL 17: JSON Types](https://www.postgresql.org/docs/17/datatype-json.html)
- psycopg requires `Json`/`Jsonb` wrappers to dump Python objects as JSON. — [psycopg 3: Adapting basic Python types](https://www.psycopg.org/psycopg3/docs/basic/adapt.html)
- `character(n)` is blank-padded, and trailing spaces are insignificant in comparisons. "character(n) is usually the slowest of the three … In most situations text or character varying should be used instead." — [PostgreSQL 17: Character Types](https://www.postgresql.org/docs/17/datatype-character.html)
- "All timezone-aware dates and times are stored internally in UTC" and are displayed in the session `TimeZone`. — [PostgreSQL 17: Date/Time Types](https://www.postgresql.org/docs/17/datatype-datetime.html)
- CHECK: "Expressions evaluating to TRUE or UNKNOWN succeed." — [PostgreSQL 17: CREATE TABLE](https://www.postgresql.org/docs/17/sql-createtable.html)
- Column-level privileges: SELECT, INSERT, UPDATE and REFERENCES can be granted per column, for example `GRANT SELECT (col1) … ON mytable`. — [PostgreSQL 17: Privileges](https://www.postgresql.org/docs/17/ddl-priv.html)

### Inferences
**Write amplification on `mosaic_audit_log`**
- Each INSERT updates 11 B-trees: the PK, the `event_uid` unique index and 9 secondaries. Together the indexes will likely exceed the heap in size: the `event_time` index alone carries 4 payload columns, and `event_uid` is 33+ bytes per entry.
- Expect several times the WAL and I/O of the bare row, plus more vacuum work. Under Lakebase's separated storage this means more WAL shipped to the safekeepers. No Lakebase-specific benchmark was found.
- The page probably needs fewer indexes:
  - Keep `(event_time DESC)` for newest-first browsing; the `DESC` is harmless but redundant for a single-column key.
  - Keep `(emp_id, event_time)`, `(trace_id, event_time)` and `request_id`.
  - Consider replacing the low-cardinality `severity`/`action`/`module` composites with one partial index for the common triage view, e.g. `(event_time) WHERE severity IN ('ERROR','WARN')`, or rely on the `event_time` index plus filtering while the table is small.
  - `parent_request_id` and `session_id` may be rarely used.
- Decide from `pg_stat_user_indexes.idx_scan` after a few weeks. Lakebase resets these counters on scale-to-zero, so sample them regularly or keep production from scaling to zero.
- Add or drop indexes later with `CREATE INDEX CONCURRENTLY`, outside a transaction block, so the app is not blocked.

**The INCLUDE covering index**
- Fine in principle. An append-only table is the ideal case for index-only scans, because pages become all-visible once vacuumed.
- It only pays off if the list query reads exactly `event_time, severity, action, module, status_code`. A typical audit grid also shows `user_name`, `description`, `path` and so on, which forces heap visits anyway.
- Either widen the INCLUDE list deliberately, keeping wide columns out, or drop the INCLUDE.

**Retention and partitioning**
- `mosaic_audit_log`, `assistant_query_log`, `recent_search` and `assistant_query_cache` grow without bound, and the audit and assistant tables hold personal data (e-mail, IP address, user agent, prompts).
- Define a retention period with compliance and implement it as a scheduled **Databricks job**, since `pg_cron` is not available.
- At today's scale (about 3 GB for the whole estate), batched `DELETE … WHERE event_time < now() - interval '…'` is enough.
- Monthly range partitioning on `event_time` becomes attractive only once the table outgrows compute memory. It has costs:
  - (a) Uniqueness must include the partition key, so dedup becomes `UNIQUE (event_uid, event_time)` and the PK becomes `(audit_id, event_time)`. This is safe only if a retried event always carries the same `event_time`.
  - (b) Partitions must be pre-created by the same job (no `pg_partman`), with a DEFAULT partition as a safety net.
  - (c) Lakebase CDF does not support partitioned tables.
- If audit analytics in Delta are wanted, prefer CDF or UC registration (platform Q4/Q5) plus short retention in Lakebase over partitioning.

**`rte_requests` with no primary key**
- The notebook comment calls it a "heap table", which is SQL Server terminology. Every Postgres table is a heap, so the comment really means "no key".
- Consequences:
  - (a) Nothing prevents duplicate rows: an explicit duplicate `external_request_id`, a replayed insert or a re-run backfill. For an outbound-e-mail queue, duplicates can mean duplicate sends; the use is inferred.
  - (b) The export process's `UPDATE … SET export_flag … WHERE external_request_id = …` has no index, and may hit several rows.
  - (c) Idempotent writes (`ON CONFLICT (external_request_id)`) are impossible, because there is no arbiter index.
  - (d) If the table ever joins a publication that replicates UPDATE/DELETE, every app UPDATE and DELETE starts failing. Native logical replication is not yet offered on Lakebase, but Postgres-CDC tools generally rely on it.
  - (e) Lakebase CDF needs `REPLICA IDENTITY FULL` anyway, which logs whole old rows on every update.
  - (f) ORM and dplyr row-update helpers need a key.
- Fix:
  - `ALTER TABLE mosaic_redesign.rte_requests ALTER COLUMN external_request_id SET NOT NULL, ADD CONSTRAINT rte_requests_pkey PRIMARY KEY (external_request_id);`. Do this now, while the table is empty, or after de-duplicating any backfill.
  - If external systems may legitimately reuse IDs, add a surrogate `rte_request_id bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY` and a unique `(external_request_id)` where it is NOT NULL.
  - Add a partial index for the export queue, e.g. `WHERE pending_flag AND NOT coalesce(export_flag,false)`.

**JSON in `text`/`varchar`**
- Affected: `details_json`, `request_body`, `response_json`, `payload_json`, `source_freshness_json` (1000), `changed_json` (2000).
- With these types nothing validates JSON, nothing can be indexed, and the varchar caps can reject legitimate payloads.
- Recommendation:
  - `jsonb` for JSON the app or analysts filter on (`details_json`, `payload_json`, `changed_json`, `source_freshness_json`).
  - Keep `text` where the value is only stored and returned verbatim and key order matters, e.g. a cached `response_json`. Optionally add a JSON-validity or `pg_jsonschema` CHECK.
  - App code must bind JSON with psycopg's `Jsonb(...)`, or pass a serialized string.

**`char(32)` `event_uid`**
- Works if the app always sends exactly 32 characters, e.g. `uuid4().hex`. A shorter ID is silently space-padded; a 36-character hyphenated UUID is rejected.
- If it is a UUID, the native `uuid` type accepts the 32-hex form, stores 16 bytes and shrinks the unique index by roughly half. Note that it outputs the hyphenated form.
- Otherwise use `text` with `CHECK (event_uid ~ '^[0-9a-f]{32}$')`.

**`timestamp` vs `timestamptz`**
- For an audit trail correlated across hosts, instances, Databricks jobs and Delta, `timestamptz` removes the session-zone ambiguity described in Q3/Q4.
- It applies to `event_time`, `stored_at`, `started_at`, `last_seen`, `stopped_at`, `granted_at`, `viewed_at`, `create_date`, `data_version`, `generated_at` and `request_date`. Keep `date` for business dates.
- Cost: Python code receives aware datetimes. This is cheap to change now, while the tables are empty.

**`mosaic_audit_viewer`**
- The CHECK is correct (`IS NOT NULL` never yields UNKNOWN), but nothing stops duplicate or conflicting grants for the same person.
- Fix:
  - `CREATE UNIQUE INDEX … (lower(emp_user_id)) WHERE emp_user_id IS NOT NULL` and `… (emp_id) WHERE emp_id IS NOT NULL`.
  - Seed the table: it starts empty, so no one can open the audit page until rows are inserted.
  - Give the app SELECT only, unless it has an admin UI (Q7).

**Missing unique and lookup keys elsewhere**
- `assistant_query_cache`:
  - Needs `UNIQUE (cache_key, team_id, page, prof_id, org_id) NULLS NOT DISTINCT` (PG ≥ 15) plus `INSERT … ON CONFLICT … DO UPDATE` for safe concurrent cache fills.
  - Needs an index on `create_date` for expiry.
  - A 20-character key suggests a truncated hash. Prefer a full SHA-256 hex digest, 64 characters.
- `assistant_daily_brief`: a natural key such as `UNIQUE (scope_hash, subject_type, subject_id, field, brief_date)`, plus `(emp_id, brief_date DESC)` for "today's brief".
- `recent_search`: if the semantics are "latest view per entity", use `UNIQUE (emp_id, entity_type, entity_id)` with upserts on `viewed_at`, plus trimming.
- `assistant_query_log`: `(emp_id, create_date)` if the app reads it.

**Foreign keys**
- Their absence matches [MIG]'s "no foreign keys" stance, and it suits an audit log: an FK to `mosaic_audit_heartbeat` would couple write ordering.
- The only natural candidate is the self-reference `assistant_daily_brief.prev_brief_id → brief_id` (`ON DELETE SET NULL`, so retention deletes still work).
- `emp_id`/`team_id` point to dimensions that live in the lakehouse, not in Lakebase.

**Column sizes**
- Postgres errors on over-length input, so an audit row with an unusually long `user_agent` (500), `path` (1000), `query_string` (2000), `error_message` (4000) or `response_message` (2000) fails the entire audit INSERT.
- Fix: truncate in the app's audit writer (`left(x, n)`), or use `text`.
- Inconsistent lengths for the same concept (`team_name` 100 vs 64) and tight enumerations (`page varchar(10)`, `status varchar(12)`) will surface as errors. Add CHECK constraints or enums where values are a closed set (`severity`, `category`, `scope_level`, `request_type`).

**`mosaic_audit_heartbeat`**
- `last_seen` is indexed, so every heartbeat UPDATE is non-HOT: it adds a new index entry and leaves dead tuples. The table is tiny, so this is low impact.
- Optionally drop the index (seq scans over a few rows are fine) or set `fillfactor`.

**Personal data**
- `request_body`, `query_string` and `stack_trace` can carry tokens or personal data, and `user_email`, `ip_address` and `user_agent` are personal data.
- Redact at the source, and consider column-level `GRANT SELECT (...)` so that only an audit-admin role sees the sensitive columns.

### Gaps
- No Lakebase-specific guidance was found on WAL or storage cost per index or on autovacuum tuning for append-only tables.
- How the app actually queries these tables is unknown, so the index recommendations need validation with `EXPLAIN` and `pg_stat_statements`, which Lakebase supports.

---

## Q7. Ownership and permissions: who owns the objects, and what grants let a Posit Connect service principal read and write?

### Takeaway
- Whoever runs the notebook becomes the owner of `mosaic_redesign` and all 8 tables. Today that is a human's OAuth role, connected as `w.current_user.me().user_name`. The notebook grants nothing to anyone.
- Recommended setup:
  - (1) DDL owned by a Databricks **group** role, which Lakebase supports for OAuth login with each member's own token.
  - (2) A NOLOGIN app role holding schema `USAGE` plus table DML, granted to the Posit Connect service principal's OAuth role.
  - (3) `ALTER DEFAULT PRIVILEGES FOR ROLE <owner group>` so that future tables inherit the grants.
- If the creating person leaves before ownership is moved, the tables keep serving the app, but owner-only DDL (ALTER, CREATE INDEX, DROP) is stranded. Lakebase's documented transfer procedure requires connecting as the current owner.

### Cited Findings
- "The table will be owned by the user issuing the command." — [PostgreSQL 17: CREATE TABLE](https://www.postgresql.org/docs/17/sql-createtable.html)
- Cell 3 connects as `user=w.current_user.me().user_name`. No cell issues GRANT, ALTER DEFAULT PRIVILEGES, `SET ROLE` or `ALTER ... OWNER`. — [SETUP](<../../context/Lakebase Schema Setup.dbc>) cells 3–6
- Lakebase pre-created roles:
  - The project owner role has LOGIN, CREATEDB, CREATEROLE, REPLICATION and BYPASSRLS, and owns `databricks_postgres`.
  - `databricks_superuser` is NOLOGIN, "Administrative role with all privileges (with grant) on all databases, schemas, tables, and sequences", and "should not be used in automated applications".
  - "By default, a newly created Postgres role has no database permissions."
  - Source: [Lakebase: Manage database permissions](https://learn.microsoft.com/en-us/azure/databricks/oltp/projects/manage-roles-permissions)
- Lakebase documents these grant patterns: `GRANT USAGE ON SCHEMA`, `GRANT SELECT|INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA`, and `ALTER DEFAULT PRIVILEGES [FOR ROLE table_creator_role_name] IN SCHEMA … GRANT … ON TABLES TO …`. The last one comes with the note "only members of table_creator_role_name can run this command". — [Lakebase: Manage database permissions](https://learn.microsoft.com/en-us/azure/databricks/oltp/projects/manage-roles-permissions)
- `ALTER DEFAULT PRIVILEGES` "does not affect privileges assigned to already-existing objects". In addition, "at object creation time, new object permissions are only affected by the default privileges of the current role, and are not inherited from any roles in which the current role is a member." — [PostgreSQL 17: ALTER DEFAULT PRIVILEGES](https://www.postgresql.org/docs/17/sql-alterdefaultprivileges.html)
- Ownership transfer in Lakebase:
  - "use a temporary shared role as an intermediate step … You can't do this directly with a standard `ALTER TABLE ... OWNER TO` command."
  - Prerequisites: the target Databricks group exists, "The current object owner is a member of the target group", and the group has a Postgres role.
  - Step 1 is "Connect as the `<current_owner>` role".
  - `REASSIGN OWNED` "Runs in the current database context" and "does not change existing GRANT permissions or default privileges".
  - Source: [Lakebase: Transfer Postgres object ownership](https://learn.microsoft.com/en-us/azure/databricks/oltp/projects/transfer-object-ownership)
- Identity mapping:
  - Group members can log in as the group's Postgres role with their own OAuth token, and membership is checked at login (platform Q4).
  - A service principal's Postgres role name is its application (client) ID, created with `databricks_create_role('<id>','SERVICE_PRINCIPAL')` (platform Q2/Q4).
  - Renaming a user e-mail or group breaks authentication and grants (platform Q4).
  - "Service principals don't inherit default schema permissions" (ops Q2).
- INSERT needs the `INSERT` privilege; `ON CONFLICT DO UPDATE` also needs `UPDATE`; "all forms of `ON CONFLICT` also require `SELECT` privilege on any column whose values are read". — [PostgreSQL 17: INSERT](https://www.postgresql.org/docs/17/sql-insert.html)
- Identity defaults call `nextval_internal(seqid, false)`, which skips the sequence permission check. INSERT on an identity table therefore needs no sequence GRANT. This resolves the platform Q4 gap. — [postgres `execExprInterp.c`, REL_17_STABLE](https://github.com/postgres/postgres/blob/REL_17_STABLE/src/backend/executor/execExprInterp.c)
- Ownership: "The right to modify or destroy an object is inherent in being the object's owner, and cannot be granted or revoked in itself". That right "can be inherited by members of the owning role". PUBLIC gets no default privileges on tables, sequences or schemas, but does get `CONNECT` and `TEMPORARY` on databases. — [PostgreSQL 17: Privileges](https://www.postgresql.org/docs/17/ddl-priv.html)
- `ALTER TABLE`: "You must own the table to use ALTER TABLE". To change the owner, "You must be able to `SET ROLE` to the new owning role, and that role must have `CREATE` privilege on the table's schema." — [PostgreSQL 17: ALTER TABLE](https://www.postgresql.org/docs/17/sql-altertable.html)

### Inferences
**Concrete grant script.** Run once per branch, because roles and grants are branch-scoped (ops Q4). Run it as the project owner or another `databricks_superuser` member over an OAuth session. `<…>` are placeholders.
```sql
-- 0. Identities (OAuth roles)
CREATE EXTENSION IF NOT EXISTS databricks_auth;
SELECT databricks_create_role('<mosaic-db-owners group display name>', 'GROUP');      -- DDL owners (engineers + deploy SP as members)
SELECT databricks_create_role('<posit-connect-sp application id>', 'SERVICE_PRINCIPAL'); -- prod app
-- (repeat for a Workbench/dev SP or a dev group on the dev branch)

-- 1. Functional NOLOGIN roles
CREATE ROLE mosaic_redesign_rw NOLOGIN;
CREATE ROLE mosaic_redesign_ro NOLOGIN;
GRANT mosaic_redesign_rw TO "<posit-connect-sp application id>";
GRANT CONNECT ON DATABASE databricks_postgres TO mosaic_redesign_rw, mosaic_redesign_ro;
GRANT CREATE  ON DATABASE databricks_postgres TO "<mosaic-db-owners group>";   -- only needed to create the schema

-- 2. Run the bootstrap DDL *as the group role* (psycopg user="<mosaic-db-owners group>", password=<own token>)
--    so the schema and tables are owned by the group, not a person:
CREATE SCHEMA IF NOT EXISTS mosaic_redesign;   -- owner = group role
-- ... 8 tables + indexes ...

-- 3. Grants on existing objects
GRANT USAGE ON SCHEMA mosaic_redesign TO mosaic_redesign_rw, mosaic_redesign_ro;
GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA mosaic_redesign TO mosaic_redesign_rw;
GRANT SELECT ON ALL TABLES IN SCHEMA mosaic_redesign TO mosaic_redesign_ro;
-- Optional least-privilege for the audit trail:
REVOKE UPDATE, DELETE ON mosaic_redesign.mosaic_audit_log FROM mosaic_redesign_rw;
REVOKE INSERT, UPDATE, DELETE ON mosaic_redesign.mosaic_audit_viewer FROM mosaic_redesign_rw;  -- if grants are admin-managed

-- 4. Future objects created by the owner group
ALTER DEFAULT PRIVILEGES FOR ROLE "<mosaic-db-owners group>" IN SCHEMA mosaic_redesign
  GRANT SELECT, INSERT, UPDATE, DELETE ON TABLES TO mosaic_redesign_rw;
ALTER DEFAULT PRIVILEGES FOR ROLE "<mosaic-db-owners group>" IN SCHEMA mosaic_redesign
  GRANT SELECT ON TABLES TO mosaic_redesign_ro;
ALTER DEFAULT PRIVILEGES FOR ROLE "<mosaic-db-owners group>" IN SCHEMA mosaic_redesign
  GRANT USAGE, SELECT ON SEQUENCES TO mosaic_redesign_rw;   -- only if app code calls nextval/currval/setval explicitly
```
- **Why `FOR ROLE <owner group>` matters:** default privileges apply only to objects created by that exact role. If engineers keep connecting as themselves, defaults set for the group never fire. Either connect as the group role, or `SET ROLE "<group>"` before DDL. Session-level `SET ROLE` to a group was not verified on Lakebase.
- **Already-created tables:**
  - If the notebook has already run as a person, use the documented temporary-role and `REASSIGN OWNED` procedure to move ownership to the group, while that person can still log in.
  - Then apply step 3. The transfer does not move or create grants.
  - The schema object itself is also owned by the person and must be moved too (`REASSIGN OWNED` covers schemas).
- **If the creating user leaves:**
  - Tables, data and existing grants remain, so the app keeps working.
  - The person's role can no longer log in (no OAuth identity).
  - Every owner-only operation stalls:
    - `ALTER TABLE`, for example new columns from a migration
    - `CREATE INDEX` on these tables
    - `DROP`/`TRUNCATE` (unless granted)
    - `ALTER … OWNER`, `REINDEX`
    - granting on new objects that the person's default privileges do not cover
  - `databricks_superuser`'s "all privileges (with grant)" is described in terms of privileges; ownership-only DDL is a separate matter in Postgres.
  - Lakebase's transfer procedure starts by connecting as the current owner, so ownership must be moved before off-boarding.
- **A job running as a service principal** with the notebook unchanged would own anything it creates, but would fail on objects a human created earlier (Q5). That is another reason for group ownership.
- **Pharma audit trail hygiene:** the app role should be append-only on `mosaic_audit_log` (INSERT + SELECT only), so the application cannot rewrite history. Retention purges then run under a separate maintenance role.

### Gaps
- Undocumented: the recovery path when the owner's Databricks identity is already deleted. It is unclear whether the project owner, as CREATEROLE, can take membership in or reassign from a departed user's OAuth role, given PG 16+ CREATEROLE/ADMIN rules.
- Not verified that `SET ROLE "<group role>"` works for an OAuth-authenticated member, as an alternative to logging in as the group role.

---

## Q8. How should this notebook be used (per branch/environment, CI/CD, migration tool), and how does it relate to the migration notebook?

### Takeaway
- Treat the notebook as a **one-time bootstrap** of the `mosaic_redesign` schema. Run it per database or branch, by the owner role (ideally the owner group), before the redesign app is deployed.
- Then freeze it as the **baseline migration (V0001)** of a SQL-first migration tool. All later changes (grants, keys, type fixes, new columns) become numbered migrations, tested on short-lived Lakebase branches (ops Q4).
- It is complementary to the migration notebook: legacy tables are copied into their original schema names, and new app tables live in `mosaic_redesign`.
- The two meet at **`rte_requests`**. One copy is migrated from Azure SQL into `mosaicai.rte_requests`; this notebook creates an empty `mosaic_redesign.rte_requests` from Delta DDL. The team must choose one system of record.

### Cited Findings
- [SETUP] targets project `<project>` / `production` / `databricks_postgres` with values hard-coded in cell 3, and has no widgets (`inputWidgets: {}`). Cell 1 says its tables "are **not** migrated from Azure SQL". — [SETUP](<../../context/Lakebase Schema Setup.dbc>) cells 1, 3; metadata
- Cell 4's DDL source is "MOSAIC_REDESIGN schema (T-SQL)". Cell 5's source is "Delta tables from `<dev catalog>`.gold_mosaic". — [SETUP] cells 4–5
- [MIG] registry:
  - Includes `mosaicai.RTE_REQUESTS` and `mosaicai.RTE_REQUESTS_BAK`, both `pre_cutover`/`truncate`/enabled, and no `MOSAIC_REDESIGN.*` entry.
  - Its markdown says it "Preserves source schema".
  - `generate_target_name` lower-cases the schema and snake-cases the table, so `mosaicai.RTE_REQUESTS` → `mosaicai.rte_requests`.
  - Its generated DDL carries no PK, identity, default or index (ops Q2).
  - Source: [MIG](<../../context/Azure SQL to Lakebase Migration.dbc>) cells 1, 2, 6
- [MIG]'s code connects to the same hard-coded project and `production` branch as [SETUP]. Its markdown names a different project (doc drift; ops Q2). — [MIG] cells 1–2, 5
- Lakebase branches are copy-on-write. The isolation "includes Postgres roles, databases and GRANT/REVOKE state". Promotion "is not automatic" (re-run the same migration). CI branches with 2–4 h TTL are suggested. — ops Q4, citing [Branches](https://learn.microsoft.com/azure/databricks/oltp/projects/branches) and [Branch-based development workflow](https://learn.microsoft.com/azure/databricks/oltp/projects/dev-workflow-tutorial)
- `production` is the root branch of a project and cannot be deleted. `databricks_postgres` is the default database (platform Q1).

### Inferences
**Ways to use it, in order of maturity**
1. **As-is, interactive and one-time**
   - The project owner opens the notebook in a workspace with serverless environment v5 and runs it once against `production`.
   - Then they run the Q7 grant script and seed `mosaic_audit_viewer`.
   - Acceptable for a first bootstrap only, and only if the cell outputs are read carefully, because errors are swallowed (Q5).
2. **Per environment**
   - If dev, test and prod are *branches of one project*: bootstrap `production` (plus roles and grants) once. Branches forked afterwards inherit schema, roles and grants at fork time.
   - If they are *separate projects*, as [MIG]'s markdown suggests for dev/test: parameterize `project_id`, `branch` and `database` with widgets or job parameters, and run once per project.
   - Never let the default target be production. Require an explicit confirmation parameter for it.
3. **As a job**
   - Wrap it in a Lakeflow Job, triggered manually, `run_as` a deploy service principal that is a member of the owner group.
   - Pin the environment instead of the pip cell, make it fail fast, and end with a catalog assertion task (Q9).
   - Good for repeatable bootstrap of new branches and projects.
4. **Replaced by migrations**, the recommended end state (ops Q4)
   - V0001 = the 22 statements, with the fixes chosen from Q9.
   - V0002 = roles and grants plus default privileges.
   - V0003+ = keys, `timestamptz` and `jsonb` changes, retention support.
   - Run from CI with a just-minted OAuth token, on a `ci-<pr>` branch first, then on `production`.
   - The notebook then becomes either a thin runner for the tool or is retired.

**Relationship to the migration notebook**
- **Separation of concerns.** [MIG] moves *existing* Azure SQL tables, keeping their schema names (`mosaicai`, `mosaicaiderm`, `fieldlens*`). [SETUP] creates *new* app-owned tables in `mosaic_redesign`. No object names collide, and no FKs link them, so the two can run in either order. [SETUP] must finish, with grants, before the redesign app goes live.
- **`rte_requests` exists in two lineages.**
  - (a) Azure SQL `mosaicai.RTE_REQUESTS` → [MIG] → `mosaicai.rte_requests`: data copied, but no key, defaults or indexes, and truncate-and-reload until cutover.
  - (b) Delta `<dev catalog>.gold_mosaic.rte_requests` → [SETUP] DDL → empty `mosaic_redesign.rte_requests`, with a `gen_random_uuid()` default.
  - The Delta table's snake_case column names look like a lakehouse replica of the Azure SQL table; this is unverified.
  - If the legacy MOSAIC AI app keeps writing `mosaicai.rte_requests` while the redesign app writes `mosaic_redesign.rte_requests`, any downstream export must read both. Otherwise requests are missed, or, if both feed one export, duplicated.
  - Decide one system of record:
    - (i) The redesign app writes `mosaicai.rte_requests` after cutover; add the PK and default there; drop the empty copy.
    - (ii) Both apps use `mosaic_redesign.rte_requests`: backfill with `INSERT … SELECT` with casts, retire `mosaicai.rte_requests` after cutover, and stop [MIG]'s refresh for it, otherwise the next run truncates live data (ops Q2 cutover hazard).
    - (iii) Keep two tables only with an explicit union in the exporter.
  - Also drop `RTE_REQUESTS_BAK` from the registry unless it is needed.
- **"Not migrated from Azure SQL" needs a decision per table.** The four T-SQL-sourced tables come from an Azure SQL `MOSAIC_REDESIGN` schema that [MIG] does not migrate, so any rows there stay behind.
  - The audit history.
  - The **audit-viewer ACL**: without seeding, nobody can open the audit page.
  - Recent searches.
  - The same applies to the Delta assistant tables: query log, cache and daily briefs. Losing these is harmless for the cache, but breaks the `prev_brief_id` change chain and history continuity.
  - For any table that is backfilled, use `OVERRIDING SYSTEM VALUE` (or COPY) and reset the identity sequence (Q3).
- **Shared code.** Cells 2–3 duplicate [MIG] cells 3 and 5. Factor connection handling into one module, with endpoint-type filtering, retry and an optional group-role user, and use it from both notebooks and from the app's own connection code.

### Gaps
- Whether dev/test use separate Lakebase projects or branches of one project could not be confirmed. The markdown and code disagree.
- Which app(s) read and write each `rte_requests` copy, and what consumes `export_flag`/`export_date`, is unknown.
- Whether the Azure SQL `MOSAIC_REDESIGN` tables and the dev-catalog Delta tables hold data worth keeping is unknown.

---

## Q9. Severity-ranked issue list with concrete fixes

### Takeaway
- **Five high-severity issues** should be fixed before the redesign app writes to these tables:
  - (H1) swallowed DDL errors that roll back silently;
  - (H2) no ownership or grant model;
  - (H3) time-zone-dependent "UTC" timestamps;
  - (H4) deduplication moved into application code;
  - (H5) a key-less, duplicated `rte_requests`.
- **Eleven medium and eight low items** cover re-run safety, verification, environment targeting, dependencies, column sizing, collation, keys, indexing and retention, Delta type mapping, data seeding, personal-data handling and documentation drift.
- The cheapest moment to fix the type and key choices is now, while all 8 tables are empty.

### Cited Findings
Evidence for each item is in the sections above, with source links:
- Transaction and idempotency semantics: Q5
- Translation semantics: Q3, Q4
- Grants and ownership: Q7
- Design: Q6
- Notebook relationships: Q8

Cell references point to [SETUP](<../../context/Lakebase Schema Setup.dbc>).

### Inferences
Severity scale:
- **High**: can lose or corrupt data, block the app, or make a failed deployment look successful.
- **Medium**: likely operational pain, latent correctness risk, or a security/compliance weakness.
- **Low**: hygiene.

**High**
- **H1 — Errors are swallowed and the commit silently rolls back** (correctness/reliability; cells 4–5)
  - Problem: a failing statement aborts the transaction. Every later statement fails, and `commit()` becomes a ROLLBACK that psycopg does not raise.
  - Result: the cell is green, objects from that cell are missing, and earlier lines said "OK" (Q5).
  - Fix:
    - Drop the per-statement `try/except`.
    - Run all DDL in `with conn.transaction():` after `SET LOCAL lock_timeout = '5s'` (and `statement_timeout`), and let exceptions propagate so the cell or job fails.
    - If per-statement reporting is wanted, use `with conn.transaction():` per statement (savepoints) and re-raise at the end when any statement failed.
- **H2 — No ownership or grant model** (security/operations; cells 3–6)
  - Problem: everything is owned by the individual who ran the notebook. The Posit Connect service principal has no USAGE or DML, so the app cannot read or write. Owner-only DDL is stranded if that person leaves, because ALTER/DROP "cannot be granted" ([PostgreSQL 17: Privileges](https://www.postgresql.org/docs/17/ddl-priv.html)).
  - Fix:
    - Connect as a Databricks **group** role (or `SET ROLE`) for DDL.
    - Create `mosaic_redesign_rw`/`_ro` roles; grant USAGE, DML and `ALTER DEFAULT PRIVILEGES FOR ROLE <owner group>`.
    - Grant `rw` to the Connect SP's OAuth role, and make the audit log append-only for the app.
    - Move any already-created objects with Lakebase's temporary-role `REASSIGN OWNED` procedure while the creator can still log in (Q7 script).
- **H3 — "UTC" timestamps depend on the session time zone** (correctness; cells 4–5)
  - Problem: `timestamp … DEFAULT CURRENT_TIMESTAMP` stores the transaction-start instant in the writer's session `TimeZone`.
    - Lakebase documents no default.
    - psycopg leaves it unset unless `PGTZ` or `options` is set.
    - RPostgres defaults the session to `"UTC"` ([RPostgres `Postgres()`](https://rpostgres.r-dbi.org/reference/Postgres.html)).
    - App-supplied aware datetimes are converted to session-local time.
    - So R and Python writers, notebooks and ad-hoc SQL can disagree (Q3).
  - Fix, in order of preference:
    - (1) Switch every point-in-time column to `timestamptz DEFAULT now()` while the tables are empty.
    - (2) Otherwise use `DEFAULT (now() AT TIME ZONE 'UTC')`.
    - (3) In all cases run `ALTER DATABASE databricks_postgres SET TimeZone = 'UTC'` per branch, and verify with `SHOW TimeZone`.
    - (4) Standardize app code on aware UTC datetimes.
- **H4 — Silent dedup moved from the index into application code** (correctness/data loss; cell 4)
  - Problem: under SQL Server, `IGNORE_DUP_KEY=ON` dropped duplicate rows and kept the rest of the INSERT. In Postgres, any writer that omits `ON CONFLICT (event_uid) DO NOTHING` gets `23505` and loses the *whole* batch, typically on the very retries the dedup exists for. `COPY` cannot dedup at all (Q3).
  - Fix:
    - Change the audit writer to multi-row `INSERT … ON CONFLICT (event_uid) DO NOTHING`, or COPY into a temporary table followed by `INSERT … SELECT … ON CONFLICT DO NOTHING`.
    - Map exceptions to SQLSTATE 23505.
    - Add an integration test that replays a batch twice on a Lakebase branch.
- **H5 — `rte_requests` has no key and two competing copies** (design/correctness; cell 5 + [MIG] cell 2)
  - Problem: nothing enforces uniqueness. Export-flag updates may hit several rows or scan the table, ON CONFLICT is impossible, and UPDATE/DELETE would fail if the table is ever published for logical replication.
  - Meanwhile [MIG] maintains `mosaicai.rte_requests` from Azure SQL, so writes and exports can split between two tables (Q6, Q8).
  - Fix:
    - Choose one system of record.
    - Add `PRIMARY KEY (external_request_id)` (NOT NULL), or a surrogate identity PK plus a unique external ID.
    - Add a partial index for the export queue.
    - Backfill from the chosen source.
    - Remove the other table from the [MIG] registry at cutover.

**Medium**
- **M1 — "Idempotent / safe to re-run" is overstated** (reliability/docs; cells 1, 4–5)
  - Problem:
    - Re-runs by any identity without CREATE on the database or schema, or without ownership of the tables, fail, because permission and ownership checks precede the `IF NOT EXISTS` checks.
    - Re-runs on live tables take `SHARE` locks.
    - Changed DDL is silently ignored (Q5).
  - Fix:
    - Reword cell 1 to "bootstrap; re-run only as the owner role; does not apply changes".
    - Add `lock_timeout`.
    - Move future changes to a migration tool (Q8).
- **M2 — The verification cannot fail** (reliability; cell 6)
  - Problem: it prints counts only. `information_schema.tables` shows "Only those tables and views … that the current user has access to" ([PostgreSQL 17: information_schema.tables](https://www.postgresql.org/docs/17/infoschema-tables.html)), so a non-owner sees fewer objects.
  - Fix: assert against an expected manifest built from `pg_catalog`, covering table names, column names, types, nullability and defaults, plus index names and definitions (`pg_indexes`), constraints and owners. `raise` on any diff.
- **M3 — Production is hard-coded; endpoint selection and connection are fragile** (safety/reliability; cells 1, 3)
  - Problem:
    - Project, branch and database are literals, with no widgets.
    - `endpoints[0]` may be a read-only replica.
    - There is no `connect_timeout` or retry for a scale-to-zero wake-up (ops Q2).
  - Fix:
    - Use widgets or job parameters with a non-production default.
    - Filter for the read-write endpoint (or address `…/endpoints/primary`).
    - Set `connect_timeout` and retry with backoff.
    - Set `application_name='mosaic_schema_setup'`.
- **M4 — The runtime pip cell is unpinned and assumes libpq** (reliability; cell 2)
  - Problem:
    - `psycopg>=3.1.0` and `databricks-sdk>=0.118.0` float.
    - Plain `psycopg` is the pure-Python build, which "will need the `libpq` PostgreSQL client library installed", and the docs recommend `psycopg[binary]` otherwise ([psycopg 3: Installation](https://www.psycopg.org/psycopg3/docs/basic/install.html)).
    - An upgraded but already-imported SDK needs a Python restart (ops Q2).
  - Fix: declare pinned `psycopg[binary]==<tested>` and `databricks-sdk==<tested>` in the notebook or job environment (serverless v5), and delete the cell.
- **M5 — Length caps reject data the sources accepted** (correctness; cells 4–5)
  - Problem: `varchar(n)` on Delta STRING sources (if confirmed), on free-form audit fields and on JSON columns makes the whole INSERT fail on over-length values (Q3, Q4, Q6).
  - Fix: use `text` for free-form and JSON columns, and truncate deliberately in the writer where a cap is wanted.
- **M6 — Collation and case-sensitivity change** (correctness; cell 4)
  - Problem: Lakebase defaults to `C.UTF-8`, while Azure SQL Database defaults to `SQL_Latin1_General_CP1_CI_AS` ([Fabric/Azure SQL limitations table](https://learn.microsoft.com/fabric/database/sql/limitations); "a case-insensitive, accent-sensitive collation" per [EF Core: Collations](https://learn.microsoft.com/ef/core/miscellaneous/collations-and-case-sensitivity)). The source database's actual collation is not confirmed.
  - User and e-mail matches in `mosaic_audit_viewer` and the lookups become case-sensitive.
  - Fix: lower-case on write and compare with `lower()` plus expression indexes, or use `citext` (available on Lakebase) or an ICU case-insensitive collation.
- **M7 — Upsert-style tables lack unique or lookup keys** (design; cells 4–5)
  - Problem: the cache, daily brief, recent searches and viewer ACL all lack them, so duplicates are possible and lookups scan the table.
  - Fix: add the unique keys listed in Q6 (using `NULLS NOT DISTINCT` for nullable scope columns, PG ≥ 15) and `ON CONFLICT` upserts.
- **M8 — Audit-log indexing and retention** (performance/operations/compliance; cell 4)
  - Problem: 11 B-trees per insert, an INCLUDE payload that may not match the real list query, and no retention for tables holding personal data. `pg_cron` and `pg_partman` are not available on Lakebase.
  - Fix:
    - Start with about 4 indexes and add more with `CREATE INDEX CONCURRENTLY` based on `pg_stat_user_indexes`, keeping in mind that counters reset on scale-to-zero.
    - Run a scheduled Databricks retention job.
    - Partition only when the table exceeds memory: the dedup key must then include `event_time`, and CDF does not support partitioned tables.
- **M9 — Delta timestamp mapping and header drift** (correctness/docs; cell 5)
  - Problem: Delta `TIMESTAMP` maps to `timestamp`, unlike Databricks' own `timestamptz` mapping, and the header's "STRING → text" contradicts the `varchar(n)` DDL.
  - Fix: confirm the source types with `SHOW CREATE TABLE`, map TIMESTAMP → `timestamptz`, and document the actual mapping table in the cell header.
- **M10 — No seed or backfill plan** (operations; cells 1, 4–5)
  - Problem:
    - `mosaic_audit_viewer` starts empty, so nobody can open the audit page.
    - History in Azure SQL `MOSAIC_REDESIGN` and in the Delta assistant tables is left behind.
    - Backfills with explicit IDs need `OVERRIDING SYSTEM VALUE` plus a sequence reset.
  - Fix: a reviewed seed script per environment (viewer ACL at minimum), plus a documented backfill procedure with `setval(pg_get_serial_sequence(...), max(id))`.
- **M11 — Sensitive content stored without redaction or column-level control** (security/privacy; cell 4)
  - Problem: `request_body`, `query_string`, `stack_trace`, `user_email`, `ip_address` and `user_agent` are stored, and every app-role reader can read every column.
  - Fix: redact secrets and personal data in the writer, cap sizes, and use column-level `GRANT SELECT (…)` so that only an audit-admin role sees the sensitive columns ([PostgreSQL 17: Privileges](https://www.postgresql.org/docs/17/ddl-priv.html)). Pair this with the retention job (M8).

**Low**
- **L1 — JSON stored as text/varchar** (cells 4–5). Fix: `jsonb` for queried JSON; `text` plus a validity check for verbatim blobs (Q6).
- **L2 — `char(32)` `event_uid`** (cell 4). Fix: native `uuid` (accepts 32-hex input) or `text` plus a regex CHECK.
- **L3 — Identifiers printed into outputs** (cell 3): the endpoint host and the user's e-mail are printed and can leak through exports that include outputs. Fix: print masked values, and clear outputs before export.
- **L4 — Observability and cleanup** (cells 3–5)
  - "OK" does not distinguish *created* from *skipped*, and server NOTICEs are not shown.
  - `CREATE SCHEMA` runs twice, `import time` is unused, and the label formats are inconsistent.
  - Fix: check `to_regclass()` before and after each statement, or capture notices; tidy the code.
- **L5 — Heartbeat updates are non-HOT** (cell 4): `last_seen` is indexed. Fix: drop the index (tiny table) or set `fillfactor`.
- **L6 — Index and constraint nits** (cells 4–5): `DESC` on a single-column key is redundant (a backward scan suffices); there are no CHECKs or enums for closed value sets; an optional self-FK on `prev_brief_id` (`ON DELETE SET NULL`). Fix as described.
- **L7 — `CURRENT_TIMESTAMP` is per-transaction**: a batch written in one transaction shares one `stored_at`. Fix: `clock_timestamp()` if per-row receipt time matters.
- **L8 — Documentation drift** (cells 1, 4–5)
  - The "heap table" wording is a SQL Server term.
  - "Idempotent: safe to re-run" (see M1).
  - "STRING → text" (see M9).
  - The translation lists omit the time-zone and collation caveats.
  - Fix: rewrite the header comments to state the real semantics and the prerequisites (owner role, `CREATE` on the database, environment).

**Suggested order of work**
1. While the tables are empty, apply H3/M9 (`timestamptz`), H5 (keys), M5/L1/L2 (types) and M7 (unique keys) as a corrected V0001.
2. H1/M1/M2/M3/M4: make the runner fail fast, parameterized and pinned.
3. H2: owner group, roles, grants and default privileges per branch.
4. H4 and M6: app-side ON CONFLICT and case handling, with branch-based integration tests.
5. M8/M10/M11: seed data, retention job, redaction and column grants.

### Gaps
- Severity assumes the inferred table roles: a rep-triggered-email queue, and an audit log used for oversight. If `rte_requests` is not an outbound queue, H5 drops to Medium.
- The app code for the audit writer, cache and exporter was not available, so H4 and M7 could not be confirmed against actual SQL.

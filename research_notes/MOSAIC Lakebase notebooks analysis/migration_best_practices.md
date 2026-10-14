# Azure SQL Database → Databricks Lakebase (Postgres): migration best practices and pitfalls, benchmarked against the MOSAIC notebooks

Research date: 2026-10-07. STATUS: COMPLETE (Q1–Q6 filled). Sources are primary vendor and project docs, accessed 2026-10-07:
- PostgreSQL 17/18 docs and wiki;
- Microsoft Learn for Azure SQL / T-SQL and Azure Databricks Lakebase;
- Spark and pandas docs;
- psycopg 3 and RPostgres docs;
- Google Cloud architecture guidance;
- AWS DMS docs;
- Databricks Labs Lakebridge and pgloader docs.

Statements labeled "search-result summary" were not read from the full page.

Scope: small OLTP app workloads (~3 GB, ~26 registered tables, low traffic, R/Python apps on Posit Workbench/Connect). Databricks-native data movement (synced tables, Lakeflow Connect, Federation) is covered in `lakebase_platform_2026.md` Q5, OAuth token behavior in its Q2, and Jobs/bundles/schema-migration tools/testing in `operationalizing_notebooks.md`. This note does not repeat them; it cites them as [PLAT] and [OPS].

Notebook citations (local, read-only; no code was run, no workspace contacted; internal hostnames, e-mails, user IDs and project/endpoint names are deliberately left out):
- [MIG] = `context/Azure SQL to Lakebase Migration.dbc` (8 cells)
- [SETUP] = `context/Lakebase Schema Setup.dbc` (6 cells)
- [PLAT] = `research_notes/MOSAIC Lakebase notebooks analysis/lakebase_platform_2026.md`
- [OPS] = `research_notes/MOSAIC Lakebase notebooks analysis/operationalizing_notebooks.md`

What the approach being benchmarked actually does (from reading the notebooks):
- **Copy path** [MIG cell 7]: `spark.read.jdbc(table=<schema.TABLE>)` → `count()` → `toPandas()` → rename columns via `column_map` → NaT→None for datetimes, float→`Int64` for nullable int columns → `to_csv(na_rep="\\N", QUOTE_MINIMAL)` into a `StringIO` → `COPY <target> FROM STDIN WITH (FORMAT csv, NULL '\N')` via psycopg 3 → `commit()` → `SELECT COUNT(*)` compared to the Spark count. Target prep (`TRUNCATE` or `DROP`+`CREATE`) is committed *before* the COPY ([OPS] already flags this).
- **DDL generation** [MIG cell 6]: reads only `TABLE_SCHEMA, TABLE_NAME, COLUMN_NAME, ORDINAL_POSITION, DATA_TYPE, CHARACTER_MAXIMUM_LENGTH, NUMERIC_PRECISION, NUMERIC_SCALE, IS_NULLABLE` from `INFORMATION_SCHEMA.COLUMNS`; emits `CREATE TABLE IF NOT EXISTS schema.table (col type, ...)` with **no** NOT NULL (IS_NULLABLE unused), PK, unique, index, default, identity, computed-column, check or collation handling. Type map: `varchar/nvarchar(n)`→`varchar(n)` (MAX→`text`); `char/nchar(n)`→`char(n)`; `tinyint`→`smallint`; `bit`→`boolean`; `decimal/numeric`→`numeric(p,s)`; `float`→`double precision`; `datetime/datetime2/smalldatetime`→`timestamp` (no precision); `datetimeoffset`→`timestamptz`; `time`→`time`; `uniqueidentifier`→`uuid`; `(var)binary/image`→`bytea`; `money`→`numeric(19,4)`; `smallmoney`→`numeric(10,4)`; `xml/text/ntext`→`text`; anything else → `text`.
- **Renaming** [MIG cell 6]: schema lower-cased; table and column names passed through a CamelCase→snake_case regex then `lower()`; identifiers are not quoted.
- **Cutover** [MIG cell 1]: table-by-table "atomic" cutover; rule "for any given table, ALL readers and writers must be on the same database at the same time"; feasibility rests on "no cross-table joins in apps", "no foreign keys", "low-traffic apps". Phase 2 (5-minute window per table): stop/drain apps that touch the table → run notebook once more → deploy apps with Lakebase connection strings → enable apps. Phase 3: monitor, then decommission the table in Azure SQL. No source write-freeze step, no proof-of-no-late-writes step, no rollback step is documented.
- **Verification** [MIG cells 7–8]: row count only (Spark `count()` vs Postgres `COUNT(*)`), reported as success/mismatch/failed; nothing raises.
- **Registry** [MIG cell 2]: 26 entries across 4 source schemas, all `pre_cutover`/`truncate`/enabled, including a `vw_`-prefixed view, two backup (`_BAK`) tables and a likely misspelled duplicate ([OPS] Q2 flags these).
- **App-owned tables** [SETUP cells 4–5]: 8 empty tables in `mosaic_redesign` translated by hand from T-SQL and Delta DDL: `IDENTITY(1,1)`→`GENERATED ALWAYS AS IDENTITY`, `DATETIME2(3)/DATETIME`→`timestamp`, `SYSUTCDATETIME()`→`CURRENT_TIMESTAMP`, `IGNORE_DUP_KEY=ON`→plain unique index + "use ON CONFLICT at insert time", `uuid()`→`gen_random_uuid()`, one heap table (`rte_requests`, no PK).

---

## Q1. Schema-conversion pitfalls that change app behavior after cutover

### Takeaway
- **Value fidelity is mostly sound.** The notebook's type map preserves values well: `money`→`numeric(19,4)`, `bit`→`boolean`, `uniqueidentifier`→`uuid`, `datetimeoffset`→`timestamptz`, and `varchar(n)` with character-count semantics. Its `\N` NULL marker keeps NULL and empty string distinct.
- **Table behavior beyond types is dropped.** DDL comes only from `INFORMATION_SCHEMA.COLUMNS` names and types, so everything SQL Server attaches to a table besides types is silently lost. The pitfalls most likely to change app behavior after cutover, in rough order:
  1. **Case- and trailing-space-insensitive comparison and sorting.** Azure SQL's `SQL_Latin1_General_CP1_CI_AS` gives way to Lakebase's code-point `C.UTF-8`.
  2. **Identity and default constraints are lost.** Inserts that omit those columns get NULLs or collide.
  3. **NOT NULL, PK and unique constraints are lost.** Duplicates are accepted silently, and `ON CONFLICT` has no arbiter.
  4. **A view becomes a static table.**
  5. **Timestamp time-zone and precision semantics change.**
  6. **Computed columns become plain columns.**

### Cited Findings
**Case-insensitivity and sort order (collation)**
- Azure SQL Database defaults:
  - "the default database collation is `SQL_Latin1_General_CP1_CI_AS`". It is set at database creation and can't be updated; individual columns can carry their own collation. — [Features comparison: Azure SQL Database and Managed Instance](https://learn.microsoft.com/azure/azure-sql/database/features-comparison?view=azuresql); [Fabric SQL database limitations (Azure SQL DB column)](https://learn.microsoft.com/fabric/database/sql/limitations)
  - Collation governs "Sorting and comparison rules in `=`, `JOIN`, `ORDER BY` and other operators that compare textual data". — [Set or change the server collation](https://learn.microsoft.com/sql/relational-databases/collations/set-or-change-the-server-collation?view=sql-server-ver17)
- Lakebase collation:
  - "By default, Lakebase uses the `C.UTF-8` collation." The `icu` provider is also supported, with a full set of predefined ICU locales usable per database or per column (e.g., `COLLATE "ru-x-icu"`).
  - "In Postgres, you can't change a database's encoding or collation after it has been created."
  - Source: [Lakebase Postgres compatibility](https://learn.microsoft.com/azure/databricks/oltp/projects/compatibility)
- In PostgreSQL's builtin provider, `C.UTF-8` "uses the code point values only" for collation, and its case mapping is the "simple" variant. — [PostgreSQL 18 docs: Locale support](https://www.postgresql.org/docs/current/locale.html)
- Deterministic vs nondeterministic collations:
  - "A deterministic collation … considers strings to be equal only if they consist of the same byte sequence". "All standard and predefined collations are deterministic".
  - A case-insensitive alternative: `CREATE COLLATION case_insensitive (provider = icu, locale = 'und-u-ks-level2', deterministic = false);`.
  - Drawbacks: "a performance penalty … B-tree cannot use deduplication with indexes that use a nondeterministic collation … certain operations are not possible with nondeterministic collations, such as some pattern matching operations."
  - Source: [PostgreSQL 18 docs: Collation support](https://www.postgresql.org/docs/current/collation.html) (page version 18.6)
- PostgreSQL 18 (released 2025-09-25):
  - "Allow `LIKE` with nondeterministic collations" and "Allow text position search functions with nondeterministic collations … These used to generate an error."
  - Adds `casefold()` and the `PG_UNICODE_FAST` builtin collation.
  - Source: [PostgreSQL 18 release notes](https://www.postgresql.org/docs/18/release-18.html)
- Lakebase Postgres versions:
  - Lakebase supports Postgres 16, 17 and 18; **17 is the default**, and 18 arrived on 2026-06-18.
  - Lakebase "does not upgrade a project to a new major Postgres version in place"; upgrading needs a new project plus `pg_dump`/`pg_restore`.
  - Source: [Postgres version support](https://learn.microsoft.com/azure/databricks/oltp/projects/postgres-version-support); [Lakebase release notes](https://learn.microsoft.com/azure/databricks/release-notes/lakebase/)
- `citext` is available on Lakebase (1.6 on PG 16/17, 1.8 on PG 18), as are `pg_trgm`, `unaccent` and `uuid-ossp`. — [Lakebase Postgres extensions](https://learn.microsoft.com/azure/databricks/oltp/projects/extensions)
- PostgreSQL's own `citext` page:
  - "Consider using nondeterministic collations … instead of this module."
  - `citext`'s case folding "depends on the `LC_CTYPE` setting", "is not truly case-insensitive in the terms defined by the Unicode standard", and is "not as efficient as `text`".
  - The traditional approach, `WHERE lower(col) = LOWER(?)`, "won't use an index, unless you create a functional index using `lower`".
  - Source: [PostgreSQL docs: citext](https://www.postgresql.org/docs/current/citext.html)
- The PostgreSQL wiki's SQL Server migration page (last edited 2024-07-16) still says "PostgreSQL is case sensitive and there is no option to change this behaviour" and advises forcing "data to upper or lower on both sides of text comparisons". Nondeterministic collations make this out of date. — [PG wiki: Microsoft SQL Server to PostgreSQL Migration by Ian Harding](https://wiki.postgresql.org/wiki/Microsoft_SQL_Server_to_PostgreSQL_Migration_by_Ian_Harding)

**Trailing spaces**
- SQL Server `=` follows ANSI SQL-92 padding: "Transact-SQL considers the strings `'abc'` and `'abc '` to be equivalent for most comparison operations. The only exception … is the LIKE predicate". "`SET ANSI_PADDING` … affects storage but not comparisons." — [= (String comparison or assignment)](https://learn.microsoft.com/sql/t-sql/language-elements/string-comparison-assignment?view=sql-server-ver17)
- In Azure SQL Database "`ANSI_PADDING` is always set to `ON`", so trailing blanks are stored, not trimmed. — [SET ANSI_PADDING](https://learn.microsoft.com/sql/t-sql/statements/set-ansi-padding-transact-sql?view=sql-server-ver17)
- SQL Server `LIKE` treats pattern-side trailing spaces as significant but ignores trailing blanks in the matched expression. With Unicode (`nvarchar`) arguments, trailing blanks are significant. — [LIKE (Transact-SQL)](https://learn.microsoft.com/sql/t-sql/language-elements/like-transact-sql?view=sql-server-ver17)
- Forum answer (Microsoft Q&A; lower authority) on unique keys: a unique constraint on `varchar` rejects `'VAT-01'` and `'VAT-01   '` as duplicates, because "trailing spaces are not significant when comparing strings". — [Microsoft Q&A](https://learn.microsoft.com/answers/a/1902474)
- PostgreSQL character types:
  - "trailing spaces *are* semantically significant in `character varying` and `text` values, and when using pattern matching".
  - For `character(n)`, trailing spaces "are treated as semantically insignificant", and they "are removed when converting a `character` value to one of the other string types".
  - `varchar(n)` stores "up to *n* characters (not bytes)". An over-length insert errors "unless the excess characters are all spaces", in which case it is truncated.
  - "the character with code zero (sometimes called NUL) cannot be stored."
  - Source: [PostgreSQL docs: Character types](https://www.postgresql.org/docs/current/datatype-character.html)

**NULL vs empty string (load path)**
- COPY CSV: "A `NULL` is output as the `NULL` parameter string and is not quoted, while a non-`NULL` value matching the `NULL` parameter string is quoted". `FORCE_NOT_NULL` and `FORCE_NULL` change how empty and quoted values match the null string. — [PostgreSQL docs: COPY](https://www.postgresql.org/docs/current/sql-copy.html)

**Identity columns and sequences**
- With `GENERATED ALWAYS`, "a user-specified value is only accepted if the `INSERT` statement specifies `OVERRIDING SYSTEM VALUE`". With `BY DEFAULT`, "the user-specified value takes precedence". — [PostgreSQL docs: Identity columns](https://www.postgresql.org/docs/current/ddl-identity-columns.html)
- "An identity column, however, does not guarantee uniqueness … Uniqueness would need to be enforced using a `PRIMARY KEY` or `UNIQUE` constraint." — same page
- "For identity columns, the `COPY FROM` command will always write the column values provided in the input data, like the `INSERT` option `OVERRIDING SYSTEM VALUE`." — [PostgreSQL docs: COPY](https://www.postgresql.org/docs/current/sql-copy.html)
- `setval` semantics:
  - `setval('myseq', 42)` makes the next `nextval` return 43; `setval('myseq', 42, false)` makes it return 42.
  - "sequence state changes made by `setval` … are not undone if the calling transaction rolls back."
  - Source: [PostgreSQL docs: Sequence functions](https://www.postgresql.org/docs/current/functions-sequence.html)
- SQL Server keeps identity metadata in `sys.identity_columns` (`seed_value`, `increment_value`, `last_value`). `sys.columns` flags `is_identity` and `is_computed`. `sys.computed_columns` and `sys.default_constraints` hold the expressions. — [sys.identity_columns](https://learn.microsoft.com/sql/relational-databases/system-catalog-views/sys-identity-columns-transact-sql?view=sql-server-ver17); [sys.columns](https://learn.microsoft.com/sql/relational-databases/system-catalog-views/sys-columns-transact-sql?view=sql-server-ver17); [Object catalog views](https://learn.microsoft.com/sql/relational-databases/system-catalog-views/object-catalog-views-transact-sql?view=sql-server-ver17)
- `INFORMATION_SCHEMA.COLUMNS` has a `COLUMN_DEFAULT` column, and Microsoft warns that "INFORMATION_SCHEMA views only represent a subset of the metadata of an object". The notebook does not select `COLUMN_DEFAULT` [MIG cell 6]. — [COLUMNS (Transact-SQL)](https://learn.microsoft.com/sql/relational-databases/system-information-schema-views/columns-transact-sql?view=sql-server-ver17)
- [OPS] Q2 already records that the generator emits no PK, identity, default, unique or index DDL and never applies `IS_NULLABLE`.

**Date and time**
- SQL Server date/time types:
  - `datetime` values "are rounded to increments of `.000`, `.003`, or `.007` seconds", with range 1753–9999.
  - `datetime2` has 0–7 fractional digits, default 7 (100 ns).
  - `smalldatetime` has 1-minute accuracy.
  - `datetimeoffset` carries a `+HH:mm` offset.
  - Source: [datetime](https://learn.microsoft.com/sql/t-sql/data-types/datetime-transact-sql?view=sql-server-ver17); [datetime2](https://learn.microsoft.com/sql/t-sql/data-types/datetime2-transact-sql?view=sql-server-ver17); [Date and time types](https://learn.microsoft.com/sql/t-sql/functions/date-and-time-data-types-and-functions-transact-sql?view=sql-server-ver17)
- Azure SQL Database time zone (sources conflict):
  - "Azure SQL Database … follow[s] UTC" for `GETDATE()`. `SET TIME ZONE` docs say the instance time zone "is always UTC and can't be changed today".
  - The `CURRENT_TIMESTAMP` and `CURRENT_TIMEZONE_ID` pages say Azure SQL Database "support[s] changing the default time zone from … UTC" at database or session level.
  - Source: [GETDATE](https://learn.microsoft.com/sql/t-sql/functions/getdate-transact-sql?view=sql-server-ver17); [SET TIME ZONE](https://learn.microsoft.com/sql/t-sql/statements/set-time-zone-transact-sql?view=fabric-sqldb); contradicted by [CURRENT_TIMESTAMP](https://learn.microsoft.com/sql/t-sql/functions/current-timestamp-transact-sql?view=sql-server-ver17)
- PostgreSQL current-time functions:
  - `CURRENT_TIMESTAMP`/`now()` "return the start time of the current transaction, their values do not change during the transaction"; `clock_timestamp()` changes even within a statement.
  - `CURRENT_TIMESTAMP` delivers values *with* time zone; `LOCALTIMESTAMP` delivers values *without* time zone.
  - Source: [PostgreSQL docs: Date/time functions](https://www.postgresql.org/docs/current/functions-datetime.html)
- Spark JDBC mapping changes:
  - "Since Spark 4.0, MsSQL Server JDBC datasource will read DATETIMEOFFSET as TimestampType, while in Spark 3.5 and previous, read as StringType". The legacy flag is `spark.sql.legacy.mssqlserver.datetimeoffsetMapping.enabled`.
  - Since Spark 4.0, TINYINT is read as ShortType.
  - Source: [Spark SQL migration guide (4.2.0)](https://spark.apache.org/docs/latest/sql-migration-guide.html)

**money, bit, uniqueidentifier**
- PG wiki on `money`: "get away from MONEY … Use NUMERIC(19,4)". The notebook already does this. — [PG wiki (Ian Harding)](https://wiki.postgresql.org/wiki/Microsoft_SQL_Server_to_PostgreSQL_Migration_by_Ian_Harding)
- PostgreSQL boolean:
  - Accepts the strings `true/yes/on/1` and `false/no/off/0` (case-insensitive). "The key words `TRUE` and `FALSE` are the preferred (SQL-compliant) method".
  - Output is always `t` or `f`.
  - Source: [PostgreSQL docs: Boolean type](https://www.postgresql.org/docs/current/datatype-boolean.html)
- SQL Server `uniqueidentifier`:
  - "ordering is not implemented by comparing the bit patterns". SQL Server treats "the last six bytes of a value [as] most significant".
  - Source: [uniqueidentifier](https://learn.microsoft.com/sql/t-sql/data-types/uniqueidentifier-transact-sql?view=sql-server-ver17); [Comparing GUID and uniqueidentifier values](https://learn.microsoft.com/sql/connect/ado-net/sql/compare-guid-uniqueidentifier-values?view=sql-server-ver17)
- PostgreSQL `uuid`:
  - "Output is always in the standard form", which is "lower-case hexadecimal digits".
  - Upper-case, braced or hyphen-less input is accepted.
  - Source: [PostgreSQL docs: UUID type](https://www.postgresql.org/docs/current/datatype-uuid.html)
- `pg_uuidv7` is not available on Lakebase. Postgres 18 has built-in `uuidv7()`/`uuidv4()`; on 16/17 use `gen_random_uuid()`. — [Lakebase Postgres extensions](https://learn.microsoft.com/azure/databricks/oltp/projects/extensions)

**Computed columns and defaults**
- PostgreSQL generated columns:
  - "There are two kinds of generated columns: stored and virtual … A generated column is by default of the virtual kind" (PG 18).
  - "A generated column cannot be written to directly." Expressions "can only use immutable functions", and virtual ones only built-in functions and types.
  - Source: [PostgreSQL docs: Generated columns](https://www.postgresql.org/docs/current/ddl-generated-columns.html); [PG 18 release notes](https://www.postgresql.org/docs/18/release-18.html)
- COPY: "If no column list is specified, all columns of the table except generated columns will be copied." — [PostgreSQL docs: COPY](https://www.postgresql.org/docs/current/sql-copy.html)

**Clustered index vs heap**
- "Clustering is a one-time operation: when the table is subsequently updated, the changes are not clustered." — [PostgreSQL docs: CLUSTER](https://www.postgresql.org/docs/current/sql-cluster.html)
- "The CLUSTER keyword in PostgreSQL is not the same as the CLUSTERED keyword in a MS SQL Server index". — [PG wiki (Ian Harding)](https://wiki.postgresql.org/wiki/Microsoft_SQL_Server_to_PostgreSQL_Migration_by_Ian_Harding)

**Views**
- [OPS] Q2 and [PLAT] Q9 already note that the registry's `vw_`-prefixed entry will be copied by JDBC as a static table.

### Inferences
1. **Collation is the #1 behavior risk, and the notebook does nothing about it.**
   - Every migrated string column inherits the database default (`C.UTF-8`), so equality, `IN`, `GROUP BY`, `DISTINCT`, joins and `ORDER BY` all become case-sensitive with code-point ordering ('Z' sorts before 'a'). Before, they were case-insensitive and dictionary-ordered.
   - Typical symptoms in low-traffic apps:
     - a login or user lookup that no longer matches after a user types different case;
     - "known users" or "allowed users" tables (the registry has such tables) that silently stop matching;
     - drop-down lists sorting differently;
     - `GROUP BY` producing extra groups.
   - Remedy options, for a ~26-table app:
     - (a) **Inventory first.** List string columns that apps filter, join or sort on with user-entered or cross-system values (e-mails, user IDs, names, codes).
     - (b) **Normalize where the data permits.** Store lower-case values (or `casefold()` on PG 18) and add `CHECK (col = lower(col))`. Alternatively, keep `text` and add `lower(col)` expression indexes, rewriting app predicates.
     - (c) **Use `citext` for identifier-like columns.** It is simplest, works on PG 16/17/18, and supports `LIKE`. The trade-offs are that PostgreSQL's own docs prefer nondeterministic collations, and `citext` is slower.
     - (d) **Or use a nondeterministic ICU collation (`und-u-ks-level2`).** Before choosing it:
       - Confirm the project is on PG 18; on PG 16/17 (17 is Lakebase's default) `LIKE` on such columns errors.
       - Lakebase documents the `icu` provider but does not explicitly mention nondeterministic collations, so test on a branch.
     - (e) **For UI sort order,** add `COLLATE "en-US-x-icu"` (deterministic, linguistic order) on the column or in `ORDER BY`.
2. **Trailing spaces.**
   - In SQL Server, `WHERE code = 'ABC'` matches `'ABC  '`, and `WHERE col = ''` matches whitespace-only strings. In Postgres neither does.
   - Profile each string column on the source for trailing blanks. `col <> RTRIM(col)` is useless in T-SQL because of padding; use `col LIKE '% '` (non-Unicode) or `DATALENGTH(col) <> DATALENGTH(RTRIM(col))`.
   - Then either `rtrim` during load (a deliberate data change, to be signed off) or fix the app predicates.
   - Under SQL Server, a unique key that relied on padding equality would have prevented `'X'` and `'X '` duplicates. Postgres accepts both.
3. **NULL vs empty string is handled correctly by the copy path.**
   - With `na_rep='\N'` and `NULL '\N'`, NULLs load as NULL, and empty strings load as empty strings because an empty field doesn't match `\N`.
   - Residual edge case: a genuine string value `\N` would become NULL.
   - SQL Server and Postgres both distinguish `''` from NULL (unlike Oracle), so app semantics don't change except through the trailing-space rule (`'' = ' '` is true only in SQL Server).
   - Pre-scan for `CHAR(0)`: Postgres cannot store NUL, so COPY would fail on such rows.
4. **Identity re-seeding is the most likely cause of first-insert failures after cutover.** The notebook creates no identity, default or PK. For each source identity column (from `sys.identity_columns`):
   - create the column as `GENERATED BY DEFAULT AS IDENTITY` (or `ALWAYS`; COPY overrides either way) plus a `PRIMARY KEY`;
   - after **every** final load, run `SELECT setval(pg_get_serial_sequence('<schema>.<table>','<col>'), COALESCE(max(<col>),0)+1, false) FROM <schema>.<table>;` or `ALTER TABLE … ALTER COLUMN … RESTART WITH <max+1>`. Note that `TRUNCATE` (the notebook's default mode) does not reset sequences unless `RESTART IDENTITY` is given, and `recreate` mode builds a fresh sequence starting at 1.
   - Without a default, app inserts that omit the ID store **NULL IDs silently**, because there is no NOT NULL or PK.
   - With an identity but no re-seed, inserts collide with existing IDs: a duplicate-key error if a PK exists, silent duplicates if not.
   - For the `mosaic_redesign` tables ([SETUP]), `ALWAYS` is fine because they start empty. Backfilling history into them would need `OVERRIDING SYSTEM VALUE` or COPY, followed by `setval`.
5. **Defaults must be translated, not dropped.** Typical translations:
   - `GETDATE()`/`SYSDATETIME()` → `LOCALTIMESTAMP` or `(now() AT TIME ZONE 'UTC')` for `timestamp` columns (Azure SQL DB returns UTC unless a database-scoped TIME_ZONE was set; check `sys.database_scoped_configurations`)
   - `NEWID()` → `gen_random_uuid()`
   - `((0))` on a `bit` column → `false` (an integer default on a boolean column is expected to be rejected; verify)
   - `''` → `''`
   - Two side notes:
     - `now()` is the *transaction* start time, so rows inserted in one transaction share a timestamp. That matters only if apps order by insert time within a transaction.
     - The [SETUP] notebook's `timestamp DEFAULT CURRENT_TIMESTAMP` depends on the session `TimeZone` ([OPS] notes this too). R/Python sessions in Posit with a non-UTC TimeZone would write local time. Prefer `timestamptz`, or `DEFAULT (now() AT TIME ZONE 'UTC')`.
6. **DATETIMEOFFSET and time zones on the load path.**
   - The notebook header says "client 5" (a serverless environment version). On Spark ≥ 4.0, DATETIMEOFFSET arrives as `TimestampType`. `toPandas()` renders it as naive wall-clock time in `spark.sql.session.timeZone`, and COPY into `timestamptz` interprets that naive text in the Lakebase session `TimeZone`. If the two differ, values shift silently.
   - On Spark 3.5 the value arrives as a string with its offset, which Postgres parses correctly.
   - In both cases the original offset is not preserved: `timestamptz` normalizes to UTC. This is standard Postgres behavior; the datatype page was not re-fetched.
   - Remedy: set `spark.sql.session.timeZone=UTC` and run `SET TIME ZONE 'UTC'` on the loader connection. If apps display the original offset, add an explicit offset column.
   - `datetime`→`timestamp` loses nothing (microsecond resolution holds .003/.007). `datetime2(7)`→`timestamp` loses the 7th digit, rounding to µs via Spark. Both are negligible for app logic, but they can break exact-match validation (see Q3).
7. **`bit`→`boolean` changes SQL, not data.**
   - R `odbc` and pyodbc already return logical/bool, so data frames look the same.
   - App SQL that compares or inserts integers (`WHERE is_active = 1`, inserting `0/1`) is expected to fail in Postgres with a type error. Rewrite to `TRUE/FALSE`, or bind booleans. Verify on a branch.
8. **`uniqueidentifier`→`uuid`.**
   - Postgres always returns lower-case text. App code that compares GUID strings case-sensitively (e.g., R `==` against an upper-case value cached from SQL Server output) will stop matching.
   - `ORDER BY` on GUIDs changes order (SQL Server weights the last 6 bytes).
   - Also unverified: whether SQL Server drivers return GUIDs upper-case by default. SQL Server tooling usually displays them that way.
9. **Computed columns become frozen copies.** `INFORMATION_SCHEMA.COLUMNS` lists computed columns like any other, so the notebook creates a plain column and copies today's values; after cutover new rows get NULL. Detect them with `sys.columns.is_computed`, and recreate them as `GENERATED ALWAYS AS (…) STORED` (or virtual on PG 18). COPY then needs an explicit column list that excludes them.
10. **Clustered index vs heap.** SQL Server results "look" ordered by the clustered key even without `ORDER BY` (never guaranteed). Postgres heap order follows load and update history. Any app code taking "the first row" or relying on implicit order (e.g., R `head()`, `df.iloc[0]`) needs explicit `ORDER BY`. Recreate PKs and needed indexes; `CLUSTER` is one-time only and not a substitute.
11. **Views and routines.**
    - Drop the `vw_` entry from the copy registry. After its base tables cut over, recreate it as a Postgres `VIEW` (or `MATERIALIZED VIEW` with a refresh plan); otherwise it silently freezes at the last copy.
    - Inventory stored procedures, functions and triggers that the apps call (`sys.objects` / `sys.sql_modules`, type P, FN, IF, TF, TR). Rewrite them in PL/pgSQL or move them into app code.
    - Whether any exist for these schemas is unknown (see Gaps).

### Gaps
- Not known from the notebooks:
  - which source columns are identity, computed or defaulted;
  - which columns carry non-default collations;
  - whether any app relies on case-insensitive matching;
  - whether stored procedures or triggers exist on these schemas.
  - All of these require querying the source catalog (`sys.*`), which this research did not do.
- The Lakebase project's Postgres major version is unknown. It decides whether nondeterministic collations support `LIKE` (PG 18 only).
- Lakebase docs list the ICU provider but do not explicitly confirm `CREATE COLLATION … deterministic = false`. This needs a branch test.
- Not verified:
  - the exact Spark version behind serverless environment version 5, and therefore which DATETIMEOFFSET mapping applies;
  - Lakebase's default session `TimeZone` (also a gap in [OPS]).
- The PostgreSQL datatype-datetime page was not fetched in this session. The "timestamptz does not retain the input offset" and "1 µs resolution" statements in the Inferences come from established PostgreSQL behavior, not a quote captured here.
- Two expected errors were not captured from a live database: `boolean = integer` comparisons and an integer default on a boolean column.

---

## Q2. Application-side changes for the R/Python apps on Posit (dialect, identifier case, drivers, placeholders, isolation)

### Takeaway
- **Mechanical changes.** Every app touching a migrated table needs a driver swap and some T-SQL rewrites (`TOP`, `ISNULL`, `+`, brackets, `GETDATE`, `OUTPUT`/`SCOPE_IDENTITY`, `MERGE`, `IGNORE_DUP_KEY`). These are mostly mechanical, and syntax errors surface on first run.
- **Silent semantic breaks.** These matter more:
  1. Result-set column names come back lower-case (and CamelCase names are actually *renamed*), so R `df$COL` and pandas `df["COL"]` break.
  2. Postgres aborts the whole transaction on any error, whereas SQL Server by default rolls back only the failing statement.
  3. The Q1 case- and trailing-space comparison changes surface inside app `WHERE` clauses.
  4. Upserts and dedup need unique constraints that the copy notebook never creates.
- **Isolation is broadly equivalent.** Azure SQL's default RCSI and Postgres READ COMMITTED both give statement-level snapshots where readers don't block writers.

### Cited Findings
**Dialect**
- PG wiki examples: "MS uses square brackets around all identifiers" while "PostgreSQL uses double quotes instead". There is no `dbo` owner qualification. "isnull() has a PostgreSQL counterpart, coalesce()". "datepart() becomes date_part()". — [PG wiki (Ian Harding), last edited 2024-07-16](https://wiki.postgresql.org/wiki/Microsoft_SQL_Server_to_PostgreSQL_Migration_by_Ian_Harding)
- `INSERT … RETURNING` "is primarily useful for obtaining values that were supplied by defaults, such as a serial sequence number" (the replacement for `SCOPE_IDENTITY()`/`OUTPUT inserted.*`). For `GENERATED ALWAYS` identities, an explicit value needs `OVERRIDING SYSTEM VALUE` (the analogue of `IDENTITY_INSERT`). — [PostgreSQL docs: INSERT](https://www.postgresql.org/docs/current/sql-insert.html)
- PostgreSQL 18 adds `OLD`/`NEW` aliases in `RETURNING` for INSERT/UPDATE/DELETE/MERGE. This is the closest match to T-SQL `OUTPUT deleted.*, inserted.*`. — [PG 18 release notes](https://www.postgresql.org/docs/18/release-18.html)
- `ON CONFLICT` behavior:
  - Arbiter unique indexes are inferred from the conflict target; "If an attempt at inference is unsuccessful, an error is raised."
  - "For `ON CONFLICT DO NOTHING`, it is optional to specify a *conflict_target*; when omitted, conflicts with all usable constraints (and unique indexes) are handled."
  - "`ON CONFLICT DO UPDATE` guarantees an atomic `INSERT` or `UPDATE` outcome … even under high concurrency."
  - Source: [PostgreSQL docs: INSERT](https://www.postgresql.org/docs/current/sql-insert.html)
- `MERGE` under concurrency: "If a repeated action is an `INSERT`, this will cause a uniqueness violation". "You may also wish to consider using `INSERT ... ON CONFLICT` as an alternative … they are not interchangeable." — [PostgreSQL docs: MERGE](https://www.postgresql.org/docs/current/sql-merge.html)
- PostgreSQL 17 (released 2024-09-26) added `RETURNING` for `MERGE` (with `merge_action()`), `WHEN NOT MATCHED BY SOURCE`, and `MERGE` on updatable views. — [PG 17 release notes](https://www.postgresql.org/docs/17/release-17.html)
- AWS's SQL Server→PostgreSQL playbook (search-result summary of the official page, not fetched in full): AWS SCT "uses object names in lowercase for PostgreSQL", and objects whose names differ only by case need quoting or renaming. — [AWS playbook: case sensitivity](https://docs.amazonaws.cn/dms/latest/sql-server-to-aurora-postgresql-migration-playbook/chap-sql-server-aurora-pg.sql.casesensitivity.html)

**Identifier case-folding**
- PostgreSQL folding rules:
  - "Quoting an identifier also makes it case-sensitive, whereas unquoted names are always folded to lower case … `FOO`, `foo`, and `"foo"` are considered the same … but `"Foo"` and `"FOO"` are different".
  - "If you want to write portable applications you are advised to always quote a particular name or never quote it."
  - Identifiers are limited to 63 bytes; "longer names … will be truncated".
  - Source: [PostgreSQL docs: Lexical structure](https://www.postgresql.org/docs/current/sql-syntax-lexical.html)

**Drivers and placeholders**
- psycopg 3 binding:
  - Psycopg 3 "sends the query and the parameters to the server separately". Server-side binding "doesn't work with `SET` or with `NOTIFY`" or "any data definition statement"; workarounds are `set_config()`, `psycopg.sql`, or `ClientCursor`.
  - With parameters, several `;`-separated statements in one `execute()` fail with "cannot insert multiple commands into a prepared statement".
  - "`IN` cannot be used with a tuple as single parameter"; use `= ANY(%s)` with a list.
  - "using `with connection` will close the connection at the end of the `with` block".
  - Source: [psycopg 3: Differences from psycopg2](https://www.psycopg.org/psycopg3/docs/basic/from_pg2.html)
- pandas 3.0.6 `read_sql` accepts an "ADBC Connection, SQLAlchemy connectable, str, or sqlite3 connection … If a DBAPI2 object, only sqlite3 is supported." Parameter syntax "is database driver dependent" (PEP 249 paramstyles). — [pandas.read_sql](https://pandas.pydata.org/docs/reference/api/pandas.read_sql.html)
- RPostgres 1.4.10 `dbConnect` defaults:
  - `timezone` "default is `"UTC"`"; `timezone_out` "defaults to `timezone`".
  - `bigint` maps by default to `bit64::integer64`.
  - Source: [RPostgres: Postgres()](https://rpostgres.r-dbi.org/reference/Postgres.html)
- Posit Professional Drivers:
  - They include a PostgreSQL ODBC connector, are tested with DBI + odbc, and are usable only with Posit professional products (Workbench, Connect).
  - Source: [Posit Professional Drivers](https://docs.posit.co/pro-drivers/) (search-result summary); [Posit Solutions: PostgreSQL](https://solutions.posit.co/connections/db/databases/postgresql/)
- Lakebase OAuth tokens, pooling and Posit connectivity are covered in [PLAT] Q2, Q3 and Q9 and not repeated here.

**Isolation and error semantics**
- Azure SQL Database isolation defaults:
  - "New databases in Azure SQL Database enable read committed snapshot (RCSI) by default." Under RCSI, readers and writers don't block each other.
  - Snapshot isolation is also enabled but must be requested explicitly.
  - Source: [Analyze and prevent deadlocks in Azure SQL Database](https://learn.microsoft.com/azure/azure-sql/database/analyze-prevent-deadlocks?view=azuresql); [Transaction locking and row versioning guide](https://learn.microsoft.com/sql/relational-databases/sql-server-transaction-locking-and-row-versioning-guide?view=sql-server-ver17)
- With optimized locking plus RCSI, update conflicts under READ COMMITTED "are handled and retried automatically". Under SNAPSHOT, update conflicts "must be handled and retried by the application". — [Transaction locking and row versioning guide](https://learn.microsoft.com/sql/relational-databases/sql-server-transaction-locking-and-row-versioning-guide?view=sql-server-ver17)
- PostgreSQL isolation levels:
  - "Read Committed is the default isolation level in PostgreSQL". A query "sees a snapshot of the database as of the instant the query begins". For a concurrently updated target row, UPDATE/DELETE re-evaluate the `WHERE` clause against the updated version.
  - Repeatable Read uses a transaction-level snapshot and can fail with "could not serialize access due to concurrent update". The application "should abort the current transaction and retry". Serialization failures use SQLSTATE `40001`.
  - Source: [PostgreSQL docs: Transaction isolation](https://www.postgresql.org/docs/current/transaction-iso.html)
- SQL Server error semantics:
  - "When SET XACT_ABORT is OFF, in some cases only the Transact-SQL statement that raised the error is rolled back and the transaction continues processing … OFF is the default setting in a T-SQL statement".
  - On a constraint violation, "the statement is ended. However, when `SET XACT_ABORT` is set to OFF, the transaction … continues to be processed."
  - Source: [SET XACT_ABORT](https://learn.microsoft.com/sql/t-sql/statements/set-xact-abort-transact-sql?view=sql-server-ver17); [CREATE TABLE (constraint notes)](https://learn.microsoft.com/sql/t-sql/statements/create-table-transact-sql?view=sql-server-ver17)
- In Postgres/psycopg, after a failed statement "the server will refuse further commands, until a rollback() is called". — psycopg docs, as cited in [PLAT] Q9 ([psycopg 3: Transactions](https://www.psycopg.org/psycopg3/docs/basic/transactions.html))

### Inferences
1. **What actually breaks after the lower-snake-case rename.**
   - **Keeps working:** unquoted SQL that references ALL_CAPS_WITH_UNDERSCORES names (`SELECT CALL_DATE FROM MOSAICAIDERM.CALL_DETAIL_TM31`). Postgres folds it to the lower-case names the notebook created, and the table registry's names are mostly in this style.
   - **Breaks:**
     - (a) bracket-quoted identifiers (`[CALL_DATE]`, a syntax error);
     - (b) double-quoted upper-case identifiers (`"CALL_DATE"`, relation or column not found);
     - (c) result-set column names, which come back lower-case, so R `df$CALL_DATE`, `dplyr::select(CALL_DATE)` and pandas `df["CALL_DATE"]` fail;
     - (d) CamelCase or mixed names that `to_snake_case` *renamed* (e.g., `HcpId` → `hcp_id`, `TM31Flag` → `tm31_flag`), where even unquoted SQL no longer matches (`HcpId` folds to `hcpid`).
   - **Also check:** names longer than 63 bytes are truncated silently and can collide (SQL Server allows 128 characters).
   - **Remedy:** publish the notebook's `column_map` per table, which cell 6 already prints, as a rename manifest for app teams. During transition, a thin compatibility layer can upper-case data-frame names after each read (`names(df) <- toupper(names(df))`; `df.columns = df.columns.str.upper()`); this only covers names that weren't renamed. Make an explicit decision about CamelCase sources (keep the original lower-cased name without inserting underscores, or rename and fix the apps).
2. **T-SQL → Postgres rewrites, all syntax-level:**
   - `SELECT TOP n` → `… LIMIT n` (or `FETCH FIRST n ROWS ONLY`)
   - `ISNULL(a,b)` → `COALESCE(a,b)`
   - `'a' + b` → `'a' || b` or `concat()` (NULL handling: `+` and `||` both propagate NULL, `concat()` ignores it)
   - `GETDATE()`/`SYSDATETIME()` → `LOCALTIMESTAMP` or `now()`; `SYSUTCDATETIME()`/`GETUTCDATE()` → `now() AT TIME ZONE 'UTC'` (see Q1 #5)
   - `[x]` → `x` (or `"x"` only if truly needed)
   - `OUTPUT inserted.id` / `SCOPE_IDENTITY()` / `@@IDENTITY` → `RETURNING id`
   - `MERGE` → `INSERT … ON CONFLICT … DO UPDATE` for upserts (atomic under concurrency). Keep `MERGE` (PG 15+; PG 17 adds `RETURNING` and `NOT MATCHED BY SOURCE`) only for multi-action syncs, where its uniqueness-violation risk under concurrency is acceptable.
   - `IGNORE_DUP_KEY = ON` → `ON CONFLICT DO NOTHING`
   - `WITH (NOLOCK)` → delete it (no dirty reads in Postgres)
   - `NEWID()` → `gen_random_uuid()`
   - `DATEADD`/`DATEDIFF`/`CONVERT(…, style)` → interval arithmetic, `date_part`/`age`, `to_char`
   - `LEN()` → `length()` (note: `LEN` ignores trailing spaces)
   - `#temp` → `CREATE TEMP TABLE`
   - `bit` literals `1/0` → `TRUE/FALSE`
   - A grep of the R/Python sources for these tokens is a cheap way to scope the work.
3. **Upserts depend on Q1.** `ON CONFLICT (event_uid)` in the redesign app works only because [SETUP] created the unique index. For migrated tables, the notebook creates no PK or unique index, so any upsert, or any `IGNORE_DUP_KEY` replacement, fails with an inference error until constraints are added.
4. **Error-handling change (high risk, invisible in happy-path tests).** Many R/Python apps wrap inserts in a transaction and catch duplicate-key errors (`tryCatch(dbExecute(...), error = ...)`, `try/except pyodbc.IntegrityError`), then keep going. That works on SQL Server's default `XACT_ABORT OFF`. On Postgres the transaction is poisoned, and every later statement fails until rollback. Fixes:
   - use `ON CONFLICT DO NOTHING`;
   - wrap the risky statement in a savepoint (psycopg `with conn.transaction():` nested blocks; R `DBI::dbWithTransaction` plus explicit `SAVEPOINT`);
   - or use autocommit for independent statements.
   Add a negative test for each write path.
5. **Isolation needs little code change.** RCSI maps to Postgres READ COMMITTED; both are MVCC with statement-level snapshots. Watch three things:
   - code that sets `SNAPSHOT` explicitly should become `REPEATABLE READ` with retry on SQLSTATE `40001`;
   - `NOLOCK` and `READ UNCOMMITTED` hints are moot;
   - lock hints (`UPDLOCK`, `ROWLOCK`) → `SELECT … FOR UPDATE`.
6. **Driver choices on Posit.**
   - **Python:** pyodbc (`?` placeholders) → psycopg 3 (`%s` / `%(name)s`, server-side binding; `IN (%s)` with tuples must become `= ANY(%s)`). For pandas, give `read_sql` a SQLAlchemy 2 engine on psycopg, or an ADBC connection (raw psycopg connections are "not supported").
   - **R, option 1:** `odbc` + SQL Server driver → `RPostgres` (native DBI backend; it numbers placeholders `$1, $2, …` — see Gaps).
   - **R, option 2:** keep `odbc` and switch only the driver to Posit's Pro PostgreSQL ODBC driver. Fewer code changes, because it keeps `odbc`/DBI code paths and `?` placeholders.
   - **Prefer** `DBI::sqlInterpolate`/`glue::glue_sql` or bound parameters over string pasting, because quoting rules differ.
7. **Type surprises in data frames:**
   - `bigint` IDs become `integer64` in RPostgres (odbc also defaults to integer64). Some R code (`jsonlite`, `%in%` against numerics) behaves differently; set `bigint = "integer"` or `"numeric"` if IDs fit.
   - `timestamp` vs `timestamptz` comes back naive or aware in pandas.
   - RPostgres uses UTC by default for session and output, so displayed local times may shift relative to the old odbc behavior.
   - Booleans come back `TRUE/FALSE` in both drivers, so data frames are unaffected; only SQL literals change.

### Gaps
- The app source code (R/Python on Posit) was not available, so which T-SQL constructs, quoted identifiers or error-handling patterns are actually used is unknown.
- The RPostgres reference page fetched did not show placeholder syntax. The `$1`-style positional placeholders are well-known RPostgres/libpq behavior but are not quoted from a fetched source here.
- `MERGE` arriving in PostgreSQL 15 is well known but was not quoted from the PG 15 release notes in this session; the PG 17 additions are cited.
- Not verified:
  - whether the Posit Pro PostgreSQL ODBC driver supports Lakebase's OAuth-token-as-password with `sslmode=require` (expected, since the token is a password, but untested);
  - its exact placeholder and boolean mapping behavior.

---

## Q3. Validation beyond row counts: what a minimal-but-sound suite looks like for ~26 tables

### Takeaway
- **Row counts alone miss the failures this migration is most likely to produce:** truncated or rounded timestamps, time-zone shifts, mangled `bytea`, trailing-space or case changes, and NULL/empty-string swaps.
- **A sound suite for ~3 GB fits in one Spark job** and has four layers:
  1. schema/contract checks;
  2. per-column profiles (null counts, min/max, sums, length sums, distinct-key counts) computed identically on both sides;
  3. a full keyed row-hash diff (cheap at this size: no sampling needed) for every table the apps write, and ideally all of them;
  4. a handful of app-level "golden queries" run against both databases to catch semantic changes (collation, trailing spaces) that no data diff can see.
- **Off-the-shelf tools mostly don't fit this pair:** Lakebridge Reconcile targets Databricks, data-diff is archived, and DMS validation needs AWS. Google's open-source DVT is the one that supports both MSSQL and Postgres.

### Cited Findings
- **Lakebridge Reconcile** (Databricks Labs):
  - Report types: `schema` (column names and types), `row` ("Hash of each row (no join key needed)"), `data` ("Row and column values via join columns") and `all`.
  - Sources: Oracle, Snowflake, SQL Server, Redshift, Teradata, BigQuery and Databricks. It compares a source "against the Databricks target", which requires Unity Catalog, `USE CONNECTION` on the source connection, and DBR 17.3+ for classic clusters. Postgres/Lakebase is not listed as a target.
  - Source: [Lakebridge Reconcile docs](https://databrickslabs.github.io/lakebridge/docs/reconcile/)
- **Google Data Validation Tool (DVT)**:
  - "an open sourced Python CLI tool based on the Ibis framework" for heterogeneous source/target comparison. Connections include MSSQL and Postgres; Azure SQL is not explicitly listed.
  - Validation types:
    - column validation with `COUNT, AVG, SUM, MIN, MAX, STDDEV_SAMP` and grouping;
    - row validation by "hash/checksum, concat, or comparison fields", which needs "one or more columns to uniquely identify each row";
    - schema validation;
    - custom-query validation.
  - The repo shows ongoing activity (1,164 commits on develop); no release number or date was captured.
  - Source: [GoogleCloudPlatform/professional-services-data-validator](https://github.com/GoogleCloudPlatform/professional-services-data-validator)
- **data-diff and its forks:**
  - Datafold archived open-source **data-diff on 2024-05-17**. **reladiff** is a drop-in fork that uses hash-based bisection across databases (and joins within one database), and a further fork, namidiff, exists.
  - Reladiff's docs page confirms Postgres but did not explicitly list SQL Server.
  - Source: [reladiff docs](https://reladiff.readthedocs.io/en/latest/); [search summary incl. Datafold OSS notice](https://docs.datafold.com/reference/open_source/)
- **AWS DMS data validation** (reference practice):
  - It "compares each row in the source with its corresponding row at the target … and reports any mismatches".
  - Limitations:
    - It "requires that the table has a primary key or unique index".
    - Rows "being continuously modified during validation" can't be validated.
    - "AWS DMS doesn't support data validation of views".
    - A target modified by applications during validation gives inaccurate results.
    - Sort order of string PKs matters: a PostgreSQL target PK not using "C" collation broke validation against Oracle.
  - A "full load validation only" task is recommended "just before or during a maintenance window production cut-over".
  - "Enhanced" validation supports SQL Server→PostgreSQL (engine 3.5.4; needs `pgcrypto` on the target).
  - Source: [AWS DMS data validation](https://docs.aws.amazon.com/dms/latest/userguide/CHAP_Validating.html)
- **SQL Server hashing functions:**
  - `CHECKSUM` "ignores the nchar and nvarchar dash character … a hash collision is **guaranteed** for any two strings where the only differences are dashes". It "trims trailing spaces from nchar and nvarchar strings" and "depends on the collation". Microsoft recommends `HASHBYTES` when changes must not be missed.
  - `BINARY_CHECKSUM` only covers "up to 255 characters of type nvarchar(n)/nvarchar(max)".
  - `CHECKSUM_AGG` is order-independent, but "a small possibility exists that the calculated checksum will not change".
  - `HASHBYTES` algorithms other than SHA2_256/SHA2_512 are deprecated from SQL Server 2016.
  - Source: [CHECKSUM](https://learn.microsoft.com/sql/t-sql/functions/checksum-transact-sql?view=sql-server-ver17); [BINARY_CHECKSUM](https://learn.microsoft.com/sql/t-sql/functions/binary-checksum-transact-sql?view=sql-server-ver17); [CHECKSUM_AGG](https://learn.microsoft.com/sql/t-sql/functions/checksum-agg-transact-sql?view=sql-server-ver17); [HASHBYTES](https://learn.microsoft.com/sql/t-sql/functions/hashbytes-transact-sql?view=sql-server-ver17)
- **Notebook findings already recorded in [OPS] Q2:**
  - The notebook's count compares two separate JDBC scans (`count()` and `toPandas()`), which can disagree while the source is live. Use `len(pdf)`.
  - Add checksums (null counts, min/max of keys and dates).
  - Binary columns are written as Python `bytes` reprs, which COPY won't decode as `bytea`.

### Inferences
1. **Why a data diff is needed here specifically.** The copy path has several lossy or transforming hops: JDBC→Spark types, `toPandas()`, float→`Int64` coercion, CSV text, COPY parsing. Each can change values without changing row counts:
   - `datetime2(7)`→µs truncation;
   - DATETIMEOFFSET time-zone shifts on Spark 4 (Q1 #6);
   - `bytea` corruption ([OPS]);
   - very large `numeric` or float formatting;
   - NUL characters;
   - a literal `\N` string becoming NULL.
   Row counts can't detect any of these.
2. **Do hashing in one engine, not two.** Hashing on each side with native SQL (`HASHBYTES` vs `md5()`/`sha256()`) needs byte-identical canonical text across dialects: datetime formats, float rendering, boolean `1/0` vs `t/f`, GUID case, collation-dependent `CHECKSUM`. That is fragile. With ~3 GB, the robust pattern is:
   - read **both** sides into Spark: source via the existing SQL Server JDBC options, target via Postgres JDBC/psycopg, or via UC if Lakebase is registered there ([PLAT] Q5);
   - apply the same canonicalization: cast to string with fixed formats, normalize timestamps to UTC µs, hex-encode binary, keep strings untrimmed;
   - compute `sha2(concat_ws('\u0001', <cols with explicit NULL marker>), 256)` per row;
   - full-outer-join on the PK into `missing_in_target` / `extra_in_target` / `changed`;
   - for tables without a PK, compare multisets in Spark (`exceptAll`-style set difference both ways).
   This reuses the notebook's existing JDBC reads and adds about one cell.
3. **Minimal-but-sound suite for ~26 tables.** Effort: about 1–2 days to build as a shared module plus a results table.
   - **L0 Contract (every run, every table).**
     - Target columns, types, nullability, PK/unique/identity and defaults match an expected manifest generated from source `sys.*` metadata.
     - Sequences sit above `max(id)`.
     - Grants exist for the app role.
     - Fail fast on drift.
   - **L1 Counts (every run).** Compare `len(pdf)` (same snapshot as the load) with target `count(*)`, inside the load transaction ([OPS]).
   - **L2 Column profiles (every run; cheap).** Per column on both sides, through Spark:
     - non-null count and null count;
     - min/max (numbers, dates, string length);
     - `sum` of numerics cast to `decimal(38,s)`;
     - sum of string lengths, using raw length because `LEN` ignores trailing spaces;
     - count of distinct PK values.
     Store the results in an audit table.
   - **L3 Full keyed row-hash diff (final pre-cutover run and the cutover run; all tables at this size).** Zero tolerance for writable tables, with a documented tolerance list (e.g., `datetime2(7)`→µs). Persist mismatching keys and sample 20 mismatches field-by-field for triage.
   - **L4 Behavioral golden queries (once per app, before cutover).** For each app, 5–10 representative SQL statements run against Azure SQL and Lakebase with results diffed. Include case-varied lookups, `ORDER BY` on text, trailing-space keys, and "latest N" queries. Only this layer catches Q1's collation and padding changes.
   - **L5 Post-cutover invariants (daily for 1–2 weeks).**
     - No NULL PKs.
     - Sequence > max(id).
     - Error rate from app logs.
     - Row growth plausible.
     - Source tables unchanged since freeze (Q4).
4. **Why not sampling.** Sampling (say 1% of keys field-compared) is a fallback for multi-TB tables. At ~3 GB and ≤ ~750k rows per table, full hashing costs minutes of serverless compute, so sampling gives up certainty without saving anything meaningful.
5. **Tool fit:**
   - **Lakebridge Reconcile** fits only if the target is presented as Databricks, e.g., a UC catalog over the Lakebase database or a Delta snapshot. That adds a hop, and it is untested for Postgres-backed catalogs.
   - **DVT** fits the pair natively (MSSQL + Postgres) and gives aggregate and row-hash validation for free. Azure SQL with Entra service-principal authentication and Lakebase OAuth tokens would need testing, and it is another tool to install on Databricks.
   - **reladiff** suits incremental re-checks of large tables, but MSSQL support is unconfirmed.
   - Verdict: a ~100-line Spark validation cell in the existing notebook is the lowest-friction option. DVT is the best off-the-shelf alternative if the team prefers a CLI with reports.

### Gaps
- Not confirmed:
  - whether DVT supports Azure SQL with Entra service-principal authentication, or a Postgres password that is a short-lived OAuth token;
  - DVT's latest release number and date;
  - whether reladiff supports MSSQL in its current release;
  - whether Lakebridge Reconcile can treat a UC-registered Lakebase catalog as a "Databricks" source or target.
- The PySpark `exceptAll` page could not be read in this session. Its multiset-difference semantics are standard Spark behavior, not quoted here.

---

## Q4. Cutover and rollback (strategy choice, write freeze, proof of no late writes, rollback, monitoring, decommissioning)

### Takeaway
- **The strategy is right.** For ~3 GB of low-traffic, read/write tables, a short scheduled-downtime cutover without dual writes is the established choice. Google's migration guidance explicitly warns against client dual writes, and CDC-based near-zero downtime is unjustified complexity at this size: there is no native SQL Server→Postgres CDC path on Databricks, and AWS DMS can't CDC from Azure SQL Database.
- **The runbook is missing the controls that make that strategy safe:**
  1. cut over by **app group**, i.e., connected components of the app↔table graph, not by single table;
  2. an **enforced and verified source write-freeze** (DENY/REVOKE plus session kill, not just "stop/drain apps");
  3. **proof that no writes landed after the final copy** (Change Tracking version or equivalent);
  4. a **go/no-go gate**: validation diff, sequence re-seed and grants before apps are re-enabled;
  5. a **rollback plan with a point of no return and a rehearsed reverse copy**;
  6. **switching off the scheduled pre-cutover refresh** for that table ([OPS] hazard);
  7. post-cutover monitoring and a decommission sequence that ends in a final backup.

### Cited Findings
**Strategy guidance**
- Google Cloud, "Database migration: concepts and principles (Part 1)" (last updated 2025-04-29):
  - When "significant downtime is acceptable", export/import or backup/restore are "perfect approaches", but "you need to quiesce the source database to prevent updates before you export", and you must "clearly establish that the acceptable downtime is long enough for the database migration and follow-up testing."
  - "you can start your test clients in read-only mode against the target databases long before you switch the clients over."
  - "While it's unrealistic to achieve zero downtime during a switch over, you can minimize the downtime…"
  - On dual writes: "This approach has several pitfalls … Two database writes are two separate transactions; you might encounter a failure after the first finishes and before the second finishes", and clients "do not know the source database transaction commit order".
  - On fallback: "Sometimes you keep the source databases as a fallback measure … after the target databases are reliably operating, you eventually delete the source databases."
  - Validation should verify that all specified data is migrated and that "Each piece of data is migrated once, and only once."
  - Source: [Google Cloud: Database migration concepts and principles, Part 1](https://docs.cloud.google.com/architecture/database-migration-concepts-principles-part-1)

**Freezing writes on Azure SQL Database**
- Database-level freeze:
  - `ALTER DATABASE … SET READ_ONLY`: "Users can read data from the database but not modify it." Changing it "requires exclusive access", and the documented pattern is `SET SINGLE_USER WITH ROLLBACK IMMEDIATE`, then `SET READ_ONLY`, then `SET MULTI_USER`.
  - Source: [ALTER DATABASE SET options](https://learn.microsoft.com/sql/t-sql/statements/alter-database-transact-sql-set-options?view=sql-server-ver17)
- Object-level freeze with DENY:
  - "DENY does not apply to object owners or members of the sysadmin fixed server role." The permission check is bypassed entirely "if the login is a member of the sysadmin fixed server role or if the user is the dbo user".
  - "A table-level DENY does not take precedence over a column-level GRANT."
  - Members of `db_owner` can grant or deny any permission in the database.
  - Source: [DENY (Transact-SQL)](https://learn.microsoft.com/sql/t-sql/statements/deny-transact-sql?view=sql-server-ver17); [Permissions (Database Engine)](https://learn.microsoft.com/sql/relational-databases/security/permissions-database-engine?view=sql-server-ver17); [DENY Object Permissions](https://learn.microsoft.com/sql/t-sql/statements/deny-object-permissions-transact-sql?view=sql-server-ver17)
- "Denying or revoking permissions to a principal will not affect requests that have passed authorization and are currently running. To restrict access immediately, you must cancel active requests or kill current sessions." This is from the Synapse/PDW edition of the GRANT/DENY/REVOKE page, so its applicability to Azure SQL Database is inferred. — [Permissions: GRANT, DENY, REVOKE](https://learn.microsoft.com/sql/t-sql/statements/permissions-grant-deny-revoke-azure-sql-data-warehouse-parallel-data-warehouse?view=azure-sqldw-latest)

**Proving no writes after the final copy**
- Change Tracking is supported on Azure SQL Database:
  - `CHANGE_TRACKING_CURRENT_VERSION()` "Returns a version that is associated with the last committed transaction".
  - `CHANGETABLE(CHANGES <table>, <last_sync_version>)` "Returns tracking information for all changes to a table that have occurred since the version" given.
  - Enable it with `ALTER DATABASE … SET CHANGE_TRACKING = ON (CHANGE_RETENTION = 2 DAYS, AUTO_CLEANUP = ON)` plus `ALTER TABLE … ENABLE CHANGE_TRACKING`.
  - Source: [CHANGE_TRACKING_CURRENT_VERSION](https://learn.microsoft.com/sql/relational-databases/system-functions/change-tracking-current-version-transact-sql?view=sql-server-ver17); [CHANGETABLE](https://learn.microsoft.com/sql/relational-databases/system-functions/changetable-transact-sql?view=sql-server-ver17); [ADF change-tracking tutorial](https://learn.microsoft.com/azure/data-factory/tutorial-incremental-copy-change-tracking-feature-portal); [ALTER DATABASE SET options](https://learn.microsoft.com/sql/t-sql/statements/alter-database-transact-sql-set-options?view=sql-server-ver17)
- `sys.change_tracking_tables.begin_version` "is reset if the table is truncated". — [sys.change_tracking_tables](https://learn.microsoft.com/sql/relational-databases/system-catalog-views/change-tracking-catalog-views-sys-change-tracking-tables?view=sql-server-ver17)
- Databricks' SQL Server ingestion setup enables Change Tracking "for tables with primary keys" (and uses CDC otherwise). — [Lakeflow Connect: SQL Server utility objects script](https://learn.microsoft.com/azure/databricks/ingestion/lakeflow-connect/sql-server-utility); see also [PLAT] Q5
- CDC on Azure SQL Database is available "for S3 tier and above. Basic, S0, S1, S2 aren't supported." — [Features comparison](https://learn.microsoft.com/azure/azure-sql/database/features-comparison?view=azuresql)
- `sys.dm_db_index_usage_stats`:
  - `user_updates` counts insert, update and delete operations, and `last_user_update` is the "Time of last user update".
  - But "The counters are initialized to empty whenever the database engine is started", and rows are removed when a database is shut down.
  - Source: [sys.dm_db_index_usage_stats](https://learn.microsoft.com/sql/relational-databases/system-dynamic-management-objects/sys-dm-db-index-usage-stats-transact-sql?view=sql-server-ver17)

**Rollback and decommissioning**
- Azure SQL Database retention:
  - It "retains sufficient backups to allow PITR within the last seven days by default", configurable to 1–35 days (Basic: 1–7).
  - Long-term retention keeps full backups up to 10 years, and LTR backups can be configured as immutable.
  - Source: [Automated backups in Azure SQL Database](https://learn.microsoft.com/azure/azure-sql/database/automated-backups-overview?view=azuresql); [Long-term retention](https://learn.microsoft.com/azure/azure-sql/database/long-term-retention-overview?view=azuresql)
- AWS DMS supports "only the full load option for Azure SQL Database" (no CDC). This is a search-result summary of AWS docs/blog. — [Using SQL Server as a source for AWS DMS](https://docs.aws.amazon.com/dms/latest/userguide/CHAP_Source.SQLServer.html); [AWS Database Blog: Azure SQL Database connector](https://aws.amazon.com/blogs/database/introducing-amazon-s3-and-microsoft-azure-sql-database-connectors-in-aws-database-migration-service/)
- Already covered elsewhere:
  - Lakebase PITR, branching and HA: [PLAT] Q6.
  - Lakebase Change Data Feed to Delta (Public Preview): [PLAT] Q5.
  - The risk that a scheduled `pre_cutover` refresh TRUNCATEs a live, cut-over table: [OPS] Q2.

### Inferences
1. **Strategy fit:**
   - **Big-bang (all 26 tables in one window)** is acceptable at this size: total copy time is minutes, and one runbook and one rollback is simpler. It is risky only if many app teams must change code simultaneously.
   - **Phased by app group (recommended).** Each group is a connected component of the app↔table graph. A single table can switch alone only if every reader and writer can be pointed at it independently. Posit apps typically have one connection string, so the realistic unit is "all tables an app touches". Any table shared with another app pulls that app's tables into the group.
   - **Rejected:**
     - dual writes, for Google's atomicity and ordering pitfalls, and because the notebook's own rule forbids them;
     - CDC near-zero downtime, because Lakeflow Connect CT/CDC lands in Delta and synced tables make Lakebase tables read-only ([PLAT] Q5), so it would not serve writable tables. A third-party CDC tool would add more risk than a 5-minute window removes.
   - **"No cross-table joins" ≠ "independent".** The notebook's feasibility argument covers SQL-level coupling but not app-level coupling. Build the app×table matrix from code search plus, for evidence, Azure SQL Auditing or Query Store over a representative period. Include Posit Connect scheduled reports and Workbench users' ad-hoc scripts that write.
2. **Freeze recipe for one cutover group** (adds ~30–60 minutes of scripted steps to the 5-minute window):
   - **T−7 days.** Enable Change Tracking on the group's tables (needs a PK; for heaps such as `rte_requests`-style tables, use the T−0 hash plus the index-usage fallback). Rehearse the whole runbook on a Lakebase branch.
   - **T−0, freeze:**
     - Stop the Posit Connect content and schedules that touch the group.
     - `DENY INSERT, UPDATE, DELETE ON OBJECT::<schema>.<table> TO <every writing principal>`.
     - Confirm none of those principals is the table owner, `dbo`, `db_owner` or the Entra admin, because DENY doesn't bind them. If one is, disable that login or rotate its secret instead.
     - Kill the remaining sessions from app hosts.
     - For the final group only, `ALTER DATABASE … SET READ_ONLY` is the simpler whole-database freeze.
   - **Copy:**
     - Record `v0 = CHANGE_TRACKING_CURRENT_VERSION()`.
     - Run the final copy with a transactional TRUNCATE+COPY ([OPS]).
     - Run the Q3 L3 diff; zero mismatches is the go criterion.
     - Re-seed sequences, apply grants, and set `cutover_status` so the scheduled refresh skips the table.
   - **Switch.** Deploy apps with the Lakebase connection, run golden-query smoke tests, then re-enable.
   - **Proof (T+0 and T+1 day).** `SELECT COUNT(*) FROM CHANGETABLE(CHANGES <t>, v0) AS c` = 0 for every table in the group. Also re-run the L2 column profile on the source and expect it unchanged. As a weaker extra signal, check that `last_user_update` in `sys.dm_db_index_usage_stats` is earlier than the freeze, unless the engine restarted.
   - Keep the DENY in place until decommission. It turns any forgotten writer into a visible error instead of silent split-brain.
3. **Rollback plan.**
   - **Before the point of no return** (Lakebase not yet accepting app writes): rollback is trivial. Remove the DENY and repoint the apps; nothing is lost.
   - **After go-live:** every Lakebase write must be carried back. Prepare and rehearse a reverse copy before cutover:
     - read the group's tables from Lakebase in Spark;
     - invert the `column_map`;
     - write them to staging tables in Azure SQL;
     - swap in one transaction, using `SET IDENTITY_INSERT ON` for identity columns.
   - Writes after cutover can alternatively be replayed from audit columns (`created_at`/`updated_at`) or from Lakebase CDF history tables (Preview) where enabled.
   - **Set explicit rollback triggers** (e.g., data diff failures, app error rate, a failed golden query) **and a time box** (e.g., 48 hours). After the time box, fix forward on Lakebase, using Lakebase PITR or branches ([PLAT] Q6) for bad-write recovery.
   - Raise Azure SQL PITR retention to ≥14–35 days for the migration period.
4. **Post-cutover monitoring (1–2 weeks per group):**
   - Posit Connect app logs and error rates.
   - Lakebase `pg_stat_statements` (available extension) for slow queries or missing indexes. Expect them: the notebook creates no indexes.
   - Daily L5 invariants (Q3): no NULL PKs, sequences above max, plausible row growth.
   - Confirm daily that the source shows no changes since `v0`.
   - Watch for OAuth-token and connection-drop errors in long-lived app sessions ([PLAT] Q2/Q9).
5. **Decommissioning sequence:**
   1. After sign-off and at least one full business cycle, set `cutover_status='cutover_complete'` and `enabled=False` in the registry.
   2. Take a final immutable LTR backup (or export to a Delta archive in UC).
   3. Rename the source tables (e.g., `zz_migrated_<t>`) for a grace period, so that hidden readers fail loudly rather than read stale data.
   4. Drop the tables, then the logins, firewall rules and secrets.
   5. Finally, remove the Azure SQL dependency from the Posit apps' configuration.
   - Record-retention requirements for a pharma company's commercial data may require keeping the final backup for a defined period (see Gaps).

### Gaps
- Not known:
  - which principals and apps write each table;
  - whether any app connects as `dbo`, `db_owner` or the Entra admin (which would defeat DENY);
  - the Azure SQL service tier (CDC needs S3+; Change Tracking availability by tier was not checked);
  - whether the source tables have PKs (Change Tracking needs one).
- No authoritative source was found for a formal "point of no return" gate or for reverse-replication rollback as an established pattern. Both are practitioner convention and are presented here as inference.
- The company's record-retention and validation SOPs (e.g., any GxP or computer-system-validation expectations for these commercial-ops apps) are unknown and could dictate decommissioning evidence.
- AWS DMS's Azure SQL Database "full load only" statement comes from search-result summaries of AWS pages, not a full-page read.

---

## Q5. Tooling check: would pgloader, Lakebridge, Azure DMS or AWS SCT/DMS beat the custom notebook here?

### Takeaway
- **No off-the-shelf tool beats the hardened custom notebook for Azure SQL Database → Lakebase at ~3 GB.**
  - **Azure DMS** has no SQL Server→PostgreSQL path.
  - **Lakebridge** transpiles and reconciles only toward Databricks SQL/Delta.
  - **AWS DMS/SC** targets RDS/Aurora, needs AWS infrastructure, and can't CDC from Azure SQL Database.
  - **pgloader** is the closest functional fit (catalog-driven tables, PKs, indexes, FKs), but its last stable release is from October 2024, it documents only SQL authentication, and its default casts are coarse.
- **What's worth borrowing:**
  - AWS DMS Schema Conversion's SQL Server→PostgreSQL settings, as a checklist (citext-for-all-strings or automatic `LOWER()`, lower-case-only folding, unique index names, procedures→functions);
  - pgloader's catalog-driven DDL (PK/index) as the model for fixing the notebook's generator;
  - Google DVT (Q3) if an off-the-shelf validator is wanted.

### Cited Findings
- **pgloader** (docs, "latest" = v4):
  - Connection: v4 "uses the official Microsoft JDBC driver (mssql-jdbc) so no FreeTDS or ODBC installation is required". It supports `jdbc:sqlserver://…;encrypt=true;…` URLs, with an Azure SQL TLS example; the native `mssql://` URI "sets `encrypt=false` automatically". Only user/password authentication is documented.
  - What it migrates: schema, data, indexes, primary keys and foreign keys. Views can be migrated "as if they were base tables". Identity, defaults, computed columns and procedures are not mentioned.
  - Default casts:
    - `char/nchar/varchar/nvarchar/xml` → `text` (drop typemod)
    - `datetime/datetime2` → `timestamptz`
    - `money/smallmoney` → `numeric` "using float-to-string"
    - `bit` → `boolean`
    - `uniqueidentifier` → `uuid`
    - `(var)binary` → `bytea`
    - `hierarchyid/geography` → `bytea`
  - Source: [pgloader: Migrating a MS SQL database to PostgreSQL](https://pgloader.readthedocs.io/en/latest/ref/mssql.html)
- **pgloader releases:**
  - Latest stable is **3.6.9 (2024-10-24)**, which includes "Add default support for MSSQL auto-incrementing bigint and smallint".
  - A **v4 development build** is dated 2024-06-20.
  - No 2025–2026 releases appeared on the releases page as fetched.
  - Source: [pgloader releases](https://github.com/dimitri/pgloader/releases)
- **Lakebridge Transpile:**
  - It converts "SQL/ETL sources to Databricks-compatible code". Sources include `mssql` ("Microsoft SQL Server, Azure SQL Database, Azure SQL Managed Instance…") and `synapse`.
  - Outputs are **DBSQL, SparkSql or a Python notebook only**, with no PostgreSQL output.
  - Engines: BladeBridge and Morpheus are deterministic; Switch is LLM-based, for "stored procedures and business logic".
  - Source: [Lakebridge Transpile](https://databrickslabs.github.io/lakebridge/docs/transpile/)
- **Lakebridge Reconcile** compares a source "against the Databricks target" (see Q3). — [Lakebridge Reconcile](https://databrickslabs.github.io/lakebridge/docs/reconcile/)
- **Azure Database Migration Service:**
  - Supported offline scenarios are SQL Server → Azure SQL DB/MI/VM, Oracle → Azure SQL via SSMA (preview), MongoDB → Cosmos DB, and MySQL → Azure MySQL. For "Azure Database for PostgreSQL flexible server" from PostgreSQL, DMS shows "No"; that path belongs to a separate migration service.
  - **There is no SQL Server → PostgreSQL scenario.**
  - Source: [Azure DMS supported scenarios](https://learn.microsoft.com/azure/dms/resource-scenario-status)
- The **migration service in Azure Database for PostgreSQL** accepts only PostgreSQL sources (on-premises, Azure VM, RDS, Aurora, Cloud SQL). — [Migration service overview](https://learn.microsoft.com/azure/postgresql/migrate/migration-service/overview-migration-service-postgresql)
- **AWS DMS Schema Conversion** (SQL Server → PostgreSQL settings):
  - Settings "apply when the source is Microsoft SQL Server and the target is Amazon RDS for PostgreSQL or Amazon Aurora PostgreSQL."
  - Options:
    - `CaseSensitivityNames` (default `false`: names "folded to lower case without quoting")
    - `UniqueIndexGeneration` (default `true`, because "In PostgreSQL, all index names within a schema must be unique")
    - `ConvertProceduresToFunction`
    - `AvoidCastingToLowerCase` (default `false`: "adds `LOWER()` casts where needed to normalise case comparisons")
    - `UseCitextForAllStringDatatypes` (maps all character types to `CITEXT`, "when migrating from a case-insensitive SQL Server collation and you want to preserve that behaviour … without modifying queries")
    - `ConvUnsupportedBuiltinsToStubs`
  - Source: [AWS DMS Schema Conversion: SQL Server to PostgreSQL settings](https://docs.aws.amazon.com/dms/latest/userguide/schema-conversion-sql-server-postgresql.html)
- **AWS DMS data movement and validation:**
  - Data movement supports "only the full load option for Azure SQL Database" (Q4).
  - Data validation supports SQL Server sources and PostgreSQL targets; "enhanced" validation covers SQL Server→PostgreSQL (Q3).
  - Source: [AWS DMS SQL Server source](https://docs.aws.amazon.com/dms/latest/userguide/CHAP_Source.SQLServer.html); [AWS DMS data validation](https://docs.aws.amazon.com/dms/latest/userguide/CHAP_Validating.html)

### Inferences
1. **Verdict per tool:**
   - **pgloader → No, as the main path; optionally a "second-opinion" DDL generator.**
     - Pros: one command handles tables, PKs, indexes and FKs.
     - Cons:
       - The source uses Entra authentication, and pgloader documents only SQL authentication.
       - It must run outside Databricks serverless (a container or VM that can reach both endpoints), adding networking and identity work.
       - Releases have been stale since 2024.
       - `datetime`→`timestamptz` silently changes semantics unless the session TimeZone is pinned.
       - `money` via float-to-string risks precision. The notebook's `numeric(19,4)` is better.
       - Identity, defaults and computed columns are not documented.
       - Lakebase OAuth tokens would work only for runs under one hour.
   - **Lakebridge → No, for this path.**
     - Transpile can't emit Postgres, so it won't convert app T-SQL or procedures for Lakebase.
     - Reconcile needs a Databricks target.
     - Its Analyzer might still help size any T-SQL views or procedures (not researched).
   - **Azure DMS → Not applicable.** There is no heterogeneous SQL Server→PostgreSQL support, and Lakebase is not Azure Database for PostgreSQL anyway.
   - **AWS SCT / DMS SC / DMS → No.** DMS SC targets RDS/Aurora only, the data path would need an AWS replication instance reaching both Azure endpoints, and there is no CDC from Azure SQL Database. Its conversion settings remain a valuable checklist.
2. **Why the notebook is the right base:**
   - It already runs where identity and networking work: a Databricks secret scope or UC connection for the source, and Lakebase OAuth for the target.
   - It reuses Spark JDBC, and its registry-driven, per-table design matches the cutover plan.
   - Its weaknesses (no PK, identity, default or index DDL; weak validation; non-atomic load) are fixable in roughly a few days (Q6). Adopting any external tool would also require fixing the case-sensitivity and app-side issues, which no tool solves for you.
3. **Renaming: the notebook goes further than the tools.** AWS's default lower-cases names *without* inserting underscores (`CaseSensitivityNames=false`), so unquoted app SQL keeps working. The notebook's CamelCase→snake_case insertion renames identifiers, which forces app changes (Q2 #1). Recommendation: lower-case only, unless a deliberate naming clean-up is wanted and budgeted.
4. **Case-insensitivity: established tooling treats it as a schema-conversion decision, not an afterthought.** AWS offers either citext for all strings or automatic `LOWER()` casts. That confirms Q1's #1 risk is mainstream and should be decided per column before cutover.

### Gaps
- Not verified:
  - whether pgloader v4's JDBC path accepts `authentication=ActiveDirectoryServicePrincipal` (it would need msal4j);
  - whether any pgloader commits or releases appeared in 2025–2026 beyond the releases page.
- Whether AWS SCT (desktop) supports a generic, non-RDS PostgreSQL target was not checked. DMS Schema Conversion's page names only RDS and Aurora.
- Lakebridge Analyzer capabilities for Azure SQL Database code were not fetched.

---

## Q6. Benchmark: best practice vs. the notebooks (done / partial / missing), remedy and rough effort

### Takeaway
- **Already right:**
  - the overall strategy (scheduled downtime, no dual writes, per-table registry);
  - value-level type mapping (`numeric(19,4)` for money, `uuid`, `boolean`, `timestamptz`);
  - NULL-vs-empty-string handling in the COPY path;
  - nullable-int coercion;
  - token refresh;
  - the hand-written `mosaic_redesign` DDL: identity, PKs, a unique dedup index plus `ON CONFLICT`, and named indexes.
- **The gaps cluster in three places:**
  1. The DDL generator copies only names and types: no NOT NULL, PK, identity, default, computed column, index or collation.
  2. Verification stops at row counts.
  3. The cutover runbook has no enforced freeze, no proof of no late writes, no go/no-go gate and no rollback.
- **Top five fixes before the first production cutover:**
  1. Generate PK, identity, NOT NULL and default DDL, and re-seed sequences.
  2. Make a per-column case-insensitivity decision (citext, `lower()` or ICU) and run app golden queries.
  3. Make the load transactional and add a keyed row-hash diff.
  4. Enforce the DENY-based freeze with a Change Tracking proof.
  5. Write and rehearse rollback and reverse-copy scripts.
- **Effort.** Roughly 2–3 engineer-weeks of platform work, plus per-app code changes (~1–3 days per app; inference).

### Cited Findings
**Notebook facts (read directly from the archives; no code run)**
- [MIG cell 6]:
  - DDL comes from `INFORMATION_SCHEMA.COLUMNS` and emits `CREATE TABLE IF NOT EXISTS` with name and type only. `IS_NULLABLE` is fetched but unused; `COLUMN_DEFAULT` is not fetched.
  - Unknown types fall back to `text`.
  - `to_snake_case` inserts underscores at case boundaries and lower-cases names. Identifiers are unquoted, with no collision or 63-byte check.
- [MIG cell 7]:
  - TRUNCATE (or DROP+CREATE) is committed before COPY.
  - COPY has no column list and uses CSV text with `\N` as the NULL marker.
  - NaT becomes None, and float columns become `Int64`.
  - Verification is `SELECT COUNT(*)` against a separate Spark `count()`.
  - Errors are caught per table, and nothing raises.
- [MIG cell 2]: a registry of 26 entries with `cutover_status`, `reload_mode` and `enabled`, all `pre_cutover`/`truncate`/enabled.
- [MIG cell 1]:
  - Phase 1 tests *reads* from Posit Workbench.
  - Phase 2 stops or drains apps, re-runs the notebook, then redeploys and enables apps.
  - Phase 3 says "monitor … decommission".
  - There is no freeze enforcement, no late-write proof and no rollback step.
- [SETUP cells 4–5]:
  - Identity, PKs, the `event_uid` unique index (with an "ON CONFLICT at insert time" note) and secondary indexes are present.
  - `timestamp DEFAULT CURRENT_TIMESTAMP` replaces `SYSUTCDATETIME()`.
  - One table has no PK.
- [SETUP cell 6]: verification prints table and column counts but asserts nothing.

**Best-practice sources:** see Q1–Q5:
- PostgreSQL docs on identity, COPY, `setval`, collation, `citext`, character types, MERGE/ON CONFLICT and lexical structure;
- Microsoft docs on collation, padding, DENY, Change Tracking and backups;
- the Google Cloud migration principles;
- AWS DMS validation and Schema Conversion settings;
- Lakebridge, pgloader and Azure DMS docs.

### Inferences
Statuses are judged against the notebook facts above. "Effort" is the rough hands-on time for one engineer who knows the notebooks; it excludes app-team work unless stated. Practice sources are in the Q-section named in the second column.

| # | Best practice | Basis | Notebook status | Recommended remedy | Effort |
|---|---|---|---|---|---|
| **A. Schema conversion** | | | | | |
| A1 | Value-preserving type map (`money`→`numeric(19,4)`, `bit`→`boolean`, GUID→`uuid`, `datetimeoffset`→`timestamptz`, char-length `varchar(n)`) | Q1 | **Done** [MIG 6] | Make unknown types (`sql_variant`, `hierarchyid`, `geography`) fail loudly instead of falling back to `text` | <0.5 d |
| A2 | NOT NULL carried over | Q1, [OPS] | **Missing** (`IS_NULLABLE` unused) [MIG 6] | Emit `NOT NULL` from `IS_NULLABLE` | <0.5 d |
| A3 | PK / unique constraints / indexes recreated (needed for upserts, validation, Change Tracking, performance) | Q1–Q4 | **Missing** for migrated tables; **Done** in [SETUP] | Generate from `sys.key_constraints`, `sys.indexes` and `sys.index_columns`; create after COPY. `IGNORE_DUP_KEY` → unique index + app `ON CONFLICT DO NOTHING` | 1 d |
| A4 | Identity → `GENERATED … AS IDENTITY` + sequence re-seed after each final load | Q1 | **Missing** [MIG 6–7]; **Done** for empty [SETUP] tables | Read `sys.identity_columns`; use `BY DEFAULT AS IDENTITY`; run `setval(pg_get_serial_sequence(…), max+1, false)` after COPY | 0.5 d |
| A5 | Default constraints translated (`GETDATE`→`LOCALTIMESTAMP` or `now() AT TIME ZONE 'UTC'`, `NEWID`→`gen_random_uuid`, `((0))`→`false`) | Q1 | **Missing** [MIG 6]; **Partial** [SETUP] (`timestamp` + `CURRENT_TIMESTAMP` depends on session TZ) | Read `sys.default_constraints` through a translation table and review by hand. Prefer `timestamptz` or explicit UTC defaults | 0.5–1 d |
| A6 | Computed columns → generated columns; exclude them from COPY | Q1 | **Missing** (silently frozen) | Detect `sys.columns.is_computed`; `GENERATED ALWAYS AS (…) STORED`; explicit COPY column list | 0.5 d if any exist |
| A7 | Case-insensitive comparison and sort preserved where apps rely on it (CI_AS → `C.UTF-8`) | Q1, Q5 (AWS offers citext or `LOWER()`) | **Missing** (both notebooks) | Per-column decision: `citext` (PG 16–18), `lower()` + expression index, or ICU `und-u-ks-level2` nondeterministic (needs PG 18 for `LIKE`; test on Lakebase). `COLLATE "en-US-x-icu"` for UI sorts | 1–3 d incl. app review |
| A8 | Trailing-space semantics handled | Q1 | **Missing** | Profile the source (`LIKE '% '`, `DATALENGTH`); either `rtrim` on load (signed-off data change) or fix app predicates | 0.5 d |
| A9 | Identifier policy: lower-case, no collisions, ≤63 bytes, quoted safely | Q2, Q5 | **Partial**: lower-casing matches tool defaults, but snake_case insertion renames CamelCase names; no collision, length or quoting safety | Lower-case only (or a deliberate, published rename manifest); collision and length checks; `psycopg.sql.Identifier` | 0.5 d |
| A10 | Time-zone-safe timestamp handling | Q1 | **Partial**: `datetime`→`timestamp` OK; DATETIMEOFFSET path depends on Spark 4 and session TZ; TZ not pinned | `spark.sql.session.timeZone=UTC` + `SET TIME ZONE 'UTC'` on the loader; document `datetime2(7)`→µs | <0.5 d |
| A11 | Binary, NUL and edge-value safe load | Q1, Q3, [OPS] | **Missing** (`bytes` repr breaks `bytea`; NUL fails COPY; a literal `\N` string becomes NULL) | Hex-encode `bytea` or switch to psycopg typed COPY (`write_row`, or binary format; psycopg COPY docs not fetched here). Pre-scan for `CHAR(0)` | 0.5 d |
| A12 | NULL vs empty string preserved | Q1 | **Done** (`\N` sentinel) | Keep it; add the literal-`\N` pre-scan (A11) | — |
| A13 | Views recreated as views, not copied as tables | Q1, [OPS], [PLAT] | **Missing** (a `vw_` entry is in the registry) | Remove from the registry; `CREATE VIEW` after the base tables cut over | 0.5 d |
| A14 | Procedures, functions and triggers inventoried and rewritten | Q1, Q5 | **Unknown / not addressed** | Query `sys.objects`/`sys.sql_modules` for the 4 schemas; rewrite in PL/pgSQL or app code | 0.5 d inventory; rewrite TBD |
| A15 | No reliance on implicit (clustered) order | Q1 | **Not addressed** (app concern) | Add `ORDER BY` in app queries; recreate PK and needed indexes (A3) | in app review |
| **B. Data movement** | | | | | |
| B1 | Bulk load via COPY from a single source read | Q3, [OPS] | **Partial**: COPY ✓; count and data come from two JDBC scans | Use `len(pdf)` as the source count | trivial |
| B2 | Per-table load is atomic (TRUNCATE + COPY + checks in one transaction) | [OPS] | **Missing** | One transaction, or a staging table + rename swap | <0.5 d |
| B3 | Idempotent, registry-driven reruns that preserve grants | [OPS] | **Partial**: registry and `truncate` ✓; `recreate` drops grants; doc drift | Default to `truncate`; re-apply grants after `recreate` | <0.5 d |
| B4 | Guard against refreshing a table after cutover | Q4, [OPS] | **Partial**: `cutover_status` skip exists; no target-side guard | Target-side marker (control table or COMMENT) checked before TRUNCATE | <0.5 d |
| B5 | Failures surface (job fails, results persisted) | [OPS] | **Missing** | Raise on mismatch or failure; write results to an audit table | <0.5 d |
| B6 | Token or credential refresh for long runs | [PLAT] Q2, [OPS] | **Done** (45-minute refresh) | Add reconnect-on-drop | <0.5 d |
| **C. Validation** | | | | | |
| C1 | Row counts from the same snapshot | Q3 | **Partial** (see B1) | `len(pdf)` vs `count(*)` inside the load transaction | trivial |
| C2 | Schema-contract check (types, nullability, PK, identity, defaults, grants, sequence > max) | Q3 | **Missing** ([SETUP 6] prints only) | Assert against a manifest generated from `sys.*` | 0.5 d |
| C3 | Column profiles (nulls, min/max, sums, length sums, distinct keys) on both sides | Q3 (DVT, DMS practice) | **Missing** | Spark cell computing identical profiles for both sides | 0.5 d |
| C4 | Full keyed row-hash diff for cutover sign-off | Q3 (DMS, Lakebridge, DVT practice) | **Missing** | Spark: canonicalize → `sha2` per row → full-outer-join on PK (multiset diff for heaps). Optional alternative: Google DVT | 1 d |
| C5 | Behavioral golden queries per app on both DBs | Q3 | **Missing** | 5–10 queries per app, including case-varied lookups, text `ORDER BY` and trailing-space keys | 0.5 d per app |
| C6 | Post-cutover invariants (daily, 1–2 weeks) | Q3, Q4 | **Missing** | Scheduled checks plus alerting | 0.5 d |
| **D. Application changes** (outside the notebooks; app-team work) | | | | | |
| D1 | Published rename manifest | Q2 | **Partial** ([MIG 6] prints `column_map`) | Persist to a table or CSV per app; link from the runbook | <0.5 d |
| D2 | T-SQL → Postgres rewrites (`TOP`, `ISNULL`, `+`, `[ ]`, `GETDATE`, `OUTPUT`/`SCOPE_IDENTITY`→`RETURNING`, `MERGE`→`ON CONFLICT`, `IGNORE_DUP_KEY`, `NOLOCK`, bit literals) | Q2 | **Partial** ([SETUP] documents `IGNORE_DUP_KEY`→`ON CONFLICT`, `SYSUTCDATETIME`→`CURRENT_TIMESTAMP`) | Grep-driven checklist per app; test on a Lakebase branch | 1–3 d per app |
| D3 | Driver and placeholder swap (pyodbc→psycopg 3 + SQLAlchemy/ADBC for pandas; R odbc→RPostgres or odbc + Posit Pro PostgreSQL driver) | Q2 | **Not in scope of the notebooks** | Choose one pattern per language; shared connection helper with OAuth refresh ([PLAT]) | 1–2 d shared + per app |
| D4 | Transaction error-semantics review (Postgres aborts the whole transaction) | Q2 | **Missing** | Savepoints or `ON CONFLICT`; negative tests per write path | in D2 |
| D5 | Isolation equivalence (RCSI ≈ Postgres READ COMMITTED) | Q2 | **No action needed**, except where code uses SNAPSHOT or lock hints | Map SNAPSHOT→REPEATABLE READ with retry on 40001 | small |
| **E. Cutover and rollback** | | | | | |
| E1 | Scheduled-downtime cutover, no dual writes | Q4 (Google) | **Done** [MIG 1] | — | — |
| E2 | Cutover unit = app group (app↔table connected components) | Q4 | **Partial** (per-table rule; relies on "no cross-table joins") | Build an app×table matrix (code + Azure SQL Auditing / Query Store); include Connect schedules and Workbench writers | 0.5–1 d |
| E3 | Enforced source write-freeze | Q4 | **Missing** ("stop/drain apps" only) | `DENY INSERT/UPDATE/DELETE` per table to writer principals (not effective for owners, dbo or sysadmin); kill sessions; DB `READ_ONLY` for the last group | 0.5 d scripts |
| E4 | Proof of no writes after the final copy | Q4 | **Missing** | Change Tracking `v0` + `CHANGETABLE(CHANGES …)` = 0 at T+0 and T+1 day; profile re-check; index-usage stats as a weak extra signal | 0.5 d |
| E5 | Go/no-go gate (C2–C5 pass, sequences re-seeded, grants applied) before enabling apps | Q3, Q4 | **Missing** (row count only) | Checklist cell that must pass before redeploying apps | 0.5 d |
| E6 | Rollback plan: point of no return, triggers, time box, rehearsed reverse copy | Q4 | **Missing** | Lakebase→Azure SQL reverse-copy script (inverse `column_map`, `IDENTITY_INSERT`), rehearsed once; raise Azure SQL PITR retention during the migration | 1 d |
| E7 | Rehearsal on a non-production target | Q4, [OPS] | **Partial** (read tests only, against the production branch) | Full dry run per group on a Lakebase branch: load, validate, app smoke tests, rollback | 0.5 d per group |
| E8 | Post-cutover monitoring | Q4 | **Partial** ("monitor app behavior") | `pg_stat_statements`, app error logs, daily invariants, source-unchanged check | 0.5 d |
| E9 | Decommission with evidence (final immutable backup, rename-then-drop, registry and secret clean-up) | Q4 | **Partial** ("decommission that table") | Final LTR or Delta archive; `zz_migrated_` grace period; drop; registry `cutover_complete`/disabled | 0.5 d |
| **F. Tooling** | | | | | |
| F1 | Use a managed or standard tool where it fits | Q5 | **Justified custom approach**: no tool fits Azure SQL DB → Lakebase better | Borrow AWS SC settings as a checklist; optionally DVT for validation | — |

**Additional inferences:**
- **Sequencing.**
  - Do A2–A6 and B2/B5 first: they change what the tables *are* and make every later pre-cutover refresh trustworthy.
  - Then do C2–C4, then A7/A8 and D2–D4 with the app teams, then E2–E7 per group.
  - The `mosaic_redesign` tables need only the [OPS] fixes (error handling, grants) and the timestamp-default decision, because they hold no migrated data.
- **Biggest single risk if nothing else changes.** A cut-over app inserts rows and gets NULL or duplicate IDs (no identity or PK), or a case-varied user lookup silently returns nothing. Neither is caught by row counts.

### Gaps
- Effort figures are judgment-based estimates, not sourced.
- Statuses for A6, A13–A15, D2–D5 and E2 depend on source-catalog and app-code facts not available to this research: computed columns, procedures and triggers, which apps write which tables, and the constructs used in app SQL.
- psycopg 3's COPY `write_row`/binary-format behavior (A11 alternative) was not re-verified from psycopg docs in this session.

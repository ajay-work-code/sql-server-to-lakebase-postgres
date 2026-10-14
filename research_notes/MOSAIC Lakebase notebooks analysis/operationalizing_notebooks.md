# Operationalizing the two MOSAIC Lakebase notebooks (.dbc) on Azure Databricks

Research date: 2026-10-06. Sources are current Microsoft Learn pages for Azure Databricks, accessed on that date, plus vendor docs for the migration tools. Facts about the notebooks come from reading the two `.dbc` archives in `context/` and their text dumps. No notebook code was run and no workspace was contacted. Internal identifiers (hostnames, e-mails, user IDs, project/endpoint names, secret-scope names) are deliberately left out of these notes. Notebook citations point to the local archives:
- [MIG] = `Azure SQL to Lakebase Migration.dbc` (8 cells)
- [SETUP] = `Lakebase Schema Setup.dbc` (6 cells)

Naming changes that the report writer should use (all 2026):
- **Databricks Assistant → Genie Code** (Mar 11, 2026).
- **Databricks Asset Bundles → Declarative Automation Bundles** (Mar 16, 2026; same `bundle` CLI, same YAML).
- Workflows are now **Lakeflow Jobs**.

---

## Q1. What is a .dbc archive, what is inside these two, is it still supported in 2026, and how do you open, import, export and convert it?

### Takeaway
A `.dbc` ("DBC Archive") is Databricks' notebooks-only export format: a ZIP of JSON notebook documents carrying cell source, per-cell metadata and, optionally, outputs. As of Oct 2026 it is still a supported import/export format in the UI, CLI and API, but it is not a version-control format. Databricks recommends IPYNB (the default) or source format for Git. The practical path is to import each `.dbc` into a Git folder, or import it and export it as `.ipynb`/`.py`, and commit that. Both archives hold exactly one Python notebook with all outputs empty. The Genie Code skill file the migration notebook depends on is **not** inside either archive.

### Cited Findings

#### Format and support status
- Databricks lists five import/export formats for notebooks: source file (`.py/.scala/.sql/.r`), HTML, "Databricks `.dbc` archive", IPython (`.ipynb`) and RMarkdown. — [Import and export Databricks notebooks](https://learn.microsoft.com/azure/databricks/notebooks/notebook-export-import)
- Folder download options:
  - **"DBC Archive (notebooks only)"** is described as "a binary format that includes metadata and notebook command outputs".
  - **"Zip - Source (notebook + files only)"** is described as usable "in a CI/CD pipeline". Notebook command outputs are not included.
  — [Import and export Databricks notebooks](https://learn.microsoft.com/azure/databricks/notebooks/notebook-export-import#export-notebooks)
- Outputs are included in HTML, IPYNB and DBC exports unless they were cleared. The export dialog has an **Include outputs** checkbox for those three formats. — [Import and export Databricks notebooks](https://learn.microsoft.com/azure/databricks/notebooks/notebook-export-import#export-notebooks)
- Importing a DBC or ZIP archive recreates its folder structure in the current folder and imports each notebook. — [Import and export Databricks notebooks](https://learn.microsoft.com/azure/databricks/notebooks/notebook-export-import)
- Size limits for import/export:
  - DBC, HTML, R Markdown and source notebooks: up to **10 MB**.
  - IPYNB: up to **100 MB**.
  - Individual cell input: 6 MB.
  — [Known limitations of Databricks notebooks](https://learn.microsoft.com/azure/databricks/notebooks/notebook-limitations)
- New notebooks are created in `.ipynb` by default. The **File → Notebook format** menu offers "**Jupyter (.ipynb) (Recommended)**" or "Source (.scala, .py, .sql, .r)". IPYNB "captures source code, notebook environment, visualization definitions, notebook widgets, and optional outputs", and GitHub/GitLab can render rich diffs of it. — [Manage Databricks notebook format](https://learn.microsoft.com/azure/databricks/notebooks/notebook-format)
- Source-format notebooks begin with the comment `# Databricks notebook source`. Cells are delimited by `# COMMAND ----------`. A file with that first-line marker and a `.py/.sql/.scala/.r` extension is treated as a notebook. — [Import and export Databricks notebooks](https://learn.microsoft.com/azure/databricks/notebooks/notebook-export-import#convert-a-file-to-a-notebook)
- Git folders recognize notebooks only by the `.ipynb` extension or by the `# Databricks notebook source` marker. Supported Git-folder asset types are files, notebooks and folders, plus queries, dashboards and alerts (those three in Public Preview). DBC is not a Git-folder notebook format. — [Supported asset types in Git folders](https://learn.microsoft.com/azure/databricks/repos/supported-artifact-types)
- By default, admins do not allow `.ipynb` outputs to be committed from Git folders. "Allow Git folders to Export IPYNB outputs" must be switched on, and per-notebook control lives in `.databricks/commit_outputs`. — [Manage Databricks notebook format](https://learn.microsoft.com/azure/databricks/notebooks/notebook-format#manage-ipynb-notebook-output-commits)
- Source-format notebooks don't store notebook dashboards. Use `.ipynb` to keep them. — [Git folder limits](https://learn.microsoft.com/azure/databricks/repos/limits#source-management)
- For serverless Python notebooks, the environment (version plus dependencies) can be embedded in source exports as a PEP 723 header. This is controlled by the "Include in source file exports" toggle. Example: `# /// script` / `# [tool.databricks.environment]` / `# environment_version = "5"`. — [Configure the serverless environment](https://learn.microsoft.com/azure/databricks/compute/serverless/dependencies#include-the-environment-in-source-file-exports)

#### What the two archives contain (local, read-only inspection)
- Each `.dbc` is a standard ZIP (deflate) containing **one entry and no folder or manifest**. — local inspection of [MIG](<../../context/Azure SQL to Lakebase Migration.dbc>) and [SETUP](<../../context/Lakebase Schema Setup.dbc>)
  - [MIG]: entry `Azure SQL to Lakebase Migration.python`, 9,396 bytes compressed / 38,981 bytes JSON.
  - [SETUP]: entry `Lakebase Schema Setup.python`, 5,265 / 24,429 bytes.
- Each JSON has 16 top-level keys: `commands, computePreferences, dashboards, environmentMetadata, environmentSerializationPreferences, globalVars, guid, iPythonMetadata, inputWidgetPreferences, inputWidgets, language, name, notebookMetadata, origId, reposExportFormat, version`. — local inspection, [MIG]/[SETUP]
- Each command (cell) object has 57 keys. These include:
  - `command` (source) and `position` (order)
  - `commandTitle`/`showCommandTitle` (cell titles)
  - `state`, `guid`/`origId`/`nuid`
  - `commentThread`, `latestUser`
  - `startTime/submitTime/finishTime`, `results`
  — local inspection; [notebook dumps](<../../context/Azure SQL to Lakebase Migration.dbc>)
- Every command has a `results` key, and in all 14 cells it is empty. The archives were exported without outputs, or with outputs already cleared. — local inspection
- Notebook-level values in both archives:
  - `version: NotebookV1`, `language: python`
  - `reposExportFormat: "JUPYTER"`
  - `environmentMetadata: {"client": "5"}`
  - `iPythonMetadata.nbformat: 4`
  - `computePreferences: null`
  - `inputWidgets: {}` (no widgets), `dashboards: []`
  — [MIG](<../../context/Azure SQL to Lakebase Migration.dbc>), [SETUP](<../../context/Lakebase Schema Setup.dbc>)
- In the serverless environment spec, `client` is the **deprecated** name for the environment version. The current required field is `environment_version`. — [Bundle resources: job.environments.spec](https://learn.microsoft.com/azure/databricks/dev-tools/bundles/resources#job)
- Serverless **environment version 5** is current, and version 6 also exists. v5 added:
  - `%uv pip` (May 7, 2026)
  - MLflow 3.8.1 and Arrow-optimized Python UDFs (Feb 25, 2026)
  — [Environment version 5](https://learn.microsoft.com/azure/databricks/release-notes/serverless/environment-version/five); [Environment version 6](https://learn.microsoft.com/azure/databricks/release-notes/serverless/environment-version/six)

#### Import, export and inspection paths
- **CLI `databricks workspace export SOURCE_PATH --format <fmt> --file <local>`**
  - Formats: `AUTO, DBC, HTML, JUPYTER, RAW, R_MARKDOWN, SOURCE`.
  - Exporting a directory is supported only for DBC, SOURCE and AUTO.
  - Oversized exports return `MAX_NOTEBOOK_SIZE_EXCEEDED`.
  — [workspace command group](https://learn.microsoft.com/azure/databricks/dev-tools/cli/reference/workspace-commands)
- **CLI `databricks workspace import TARGET_PATH --file <local> --format DBC`** (also `--content` base64, `--overwrite`, `--language`)
  - Importing a directory works with DBC, or with SOURCE when the language is unset.
  - A single SOURCE file requires `--language`.
  - "Zip files within directories are not supported."
  — [workspace command group](https://learn.microsoft.com/azure/databricks/dev-tools/cli/reference/workspace-commands)
- **CLI `import-dir` / `export-dir`** copy whole directories recursively. "Notebooks will have their extensions stripped" on `import-dir`. — [workspace command group](https://learn.microsoft.com/azure/databricks/dev-tools/cli/reference/workspace-commands)
- **UI**: Workspace → right-click a folder → **Import**, then browse to the `.dbc`. To change format later, use **File → Export** or **File → Notebook format**. — [Import and export Databricks notebooks](https://learn.microsoft.com/azure/databricks/notebooks/notebook-export-import); [Manage Databricks notebook format](https://learn.microsoft.com/azure/databricks/notebooks/notebook-format#change-the-default-notebook-format-setting)
- A July 2021 release note shows the history behind the recommendation. Before bulk source export existed, folders could be exported only as HTML or `.dbc`, and the `.dbc` "would include notebook command results unless the notebook was cleared before exporting". Source ZIPs were introduced as the CI/CD-friendly option. — [Release notes, July 2021](https://learn.microsoft.com/azure/databricks/release-notes/product/2021/july#bulk-import-and-export-notebooks-in-a-folder-as-source-files)

### Inferences
- **"How could each file be used" (direct answer)**
  - Treat both `.dbc` files as transport containers only.
  - Import each into the workspace, preferably straight into a Git folder:
    - `databricks workspace import /Workspace/Users/<me>/mosaic-lakebase/migration --file "Azure SQL to Lakebase Migration.dbc" --format DBC`
  - Convert to a diffable format:
    - `databricks workspace export <path> --format JUPYTER --file azure_sql_to_lakebase_migration.ipynb`
    - or `--format SOURCE --file ....py`
  - Commit the result. `reposExportFormat: "JUPYTER"` suggests the authors already used IPYNB as their default, so in a Git folder these would serialize as `.ipynb` without any change.
- **IPYNB vs source**
  - Prefer IPYNB if the team wants notebook environments and cell titles round-tripped. Cell titles carry meaning in both notebooks.
  - Prefer `.py` source for the cleanest PR diffs and for importing functions into pytest. A hybrid also works: logic in `.py` modules, thin `.ipynb` driver notebooks.
- **Offline inspection without a workspace**
  - Open the `.dbc` as a ZIP and read the single `<name>.python` JSON.
  - Sort `commands` by `position` and read `command`. A leading `%md` marks a markdown cell, and `commandTitle` gives the cell title.
  - A small script can emit a source-format `.py`: the `# Databricks notebook source` header plus `# COMMAND ----------` separators.
  - Round-tripping through the workspace is more faithful (titles, magics, environment), because those conventions are only partly documented (see Gaps).
- **Why `.dbc` should not be committed**
  - It is opaque binary, so Git cannot diff it.
  - The JSON inside is noisy: per-cell GUIDs, `origId`, timestamps, `latestUser`, comment threads.
  - It is not recognized as a notebook in Git folders.
  - It can silently carry outputs (data samples). That is a real risk here: the migration notebook's summary cell `display()`s per-table results, and other cells print column maps and table names.
- **What a `.dbc` does not carry**
  - Cluster/serverless configuration, beyond the `environmentMetadata` version pointer, which holds no dependency list in these two archives.
  - Job definitions and schedules.
  - Secret values.
  - Permissions/ACLs.
  - Git metadata.
  - Workspace files: the skill folder (`/Users/<user>/.assistant/skills/azure-sql-to-lakebase/SKILL.md`) is a workspace file, not a notebook, so it was never in the archive. It must be exported separately, e.g. `databricks workspace export /Users/<user>/.assistant/skills/azure-sql-to-lakebase/SKILL.md --format AUTO --file SKILL.md`, or with `export-dir` on the skill folder.
- **What `environmentMetadata {"client":"5"}` means in practice**
  - The notebooks were last attached to serverless environment v5.
  - Neither archive records any declared dependencies.
  - This is consistent with dependencies being installed at runtime by `subprocess` pip ([MIG] cell 3, [SETUP] cell 2) rather than through the Environment side pane.

### Gaps
- No Databricks page found that formally documents the internal DBC JSON schema (key meanings, `NotebookV1`/`CommandV1`). The description above comes from inspecting these two files.
- The exact source-format conventions are not verified in the docs fetched this session:
  - `# MAGIC` prefixes for `%md` cells
  - `# DBTITLE 1,<title>` for cell titles
- Not verified whether importing a `.dbc` directly into a Git folder behaves differently from importing into a normal folder.
- No deprecation notice for DBC was found, and no statement saying it is "not recommended". The recommendation to avoid it for Git comes from the IPYNB "(Recommended)" label and the CI/CD wording on source ZIPs.

---

## Q2. How should the two notebooks be productionized (Git folders, bundles, Jobs, environments, JDBC driver, secrets, run_as, notifications/retries/timeouts, idempotency), given what they actually do?

### Takeaway
Both notebooks work interactively, but they have issues that will bite once scheduled:
- **Hard-coded config**: environment-specific hosts, DB names and the Lakebase project/branch, plus a 26-entry table registry in code.
- **Runtime installs**: packages are installed with `subprocess` pip.
- **Deprecated source auth**: the Azure SQL login is a human Entra account using **deprecated** `ActiveDirectoryPassword` auth.
- **Ambiguous endpoint**: `endpoints[0]` is taken without checking that it is the read-write endpoint.
- **Errors swallowed**: a job run would report success even when tables fail.
- **Non-atomic loads**: TRUNCATE is committed before COPY.
- **Cutover hazard**: once a table is cut over, a scheduled refresh would overwrite live Lakebase data unless the registry is updated first.

The recommended shape:
- Git folder plus a **Declarative Automation Bundle** with dev and prod targets.
- A **Lakeflow Job** with job parameters (optionally a `For each` task over tables).
- Serverless environment v5 with pinned dependencies.
- A UC connection (SQL Server federation with Entra OAuth M2M), or SP-based JDBC, for the source.
- An Azure Key Vault-backed scope readable only by the job's service principal.
- `run_as` the service principal, with a matching Lakebase Postgres OAuth role.
- `max_concurrent_runs: 1`, retries and timeouts on idempotent tasks, failure notifications, and a final assertion that fails the run on any table failure or count mismatch.

### Cited Findings

#### Platform naming and versions (2026)
- "Databricks Asset Bundles has been renamed to Declarative Automation Bundles" (Mar 16, 2026). The rename "is non-breaking. The `bundle` CLI command and all of your existing configuration does not need to be modified." — [Bundles release notes](https://learn.microsoft.com/azure/databricks/release-notes/dev-tools/bundles); [Bundles FAQ](https://learn.microsoft.com/azure/databricks/dev-tools/bundles/faqs)
- Bundle resources for **Lakebase Postgres projects, branches and endpoints** (`postgres_project`, `postgres_branch`, `postgres_endpoint`) arrived Feb 5, 2026 with CLI 0.287.0. — [Bundles release notes](https://learn.microsoft.com/azure/databricks/release-notes/dev-tools/bundles)
- The bundle **direct deployment engine became GA** on June 10, 2026 (CLI 1.3.0). It is the default for new bundles instead of Terraform. — [Bundles release notes](https://learn.microsoft.com/azure/databricks/release-notes/dev-tools/bundles)
- Databricks recommends bundles for CI/CD and recommends parameterizing environment-specific settings "instead of hardcoding". — [CI/CD workflows on Databricks](https://learn.microsoft.com/azure/databricks/dev-tools/ci-cd/flows#core-principles-of-ci-cd)
- Bundles can include the Lakebase project, branch and endpoint as resources. Bundle deploys put files under `/Workspace/Users/<user>/.bundle/<bundle>/<target>` by default. — [Bundle resources](https://learn.microsoft.com/azure/databricks/dev-tools/bundles/resources); [CI/CD with Jenkins](https://learn.microsoft.com/azure/databricks/dev-tools/ci-cd/jenkins#deploy-the-bundle-to-your-workspace)
- Databricks Repos are now "Git folders". They can live anywhere under `/Workspace/Users/<user>/`. Databricks recommends Databricks Runtime 15.0+ for consistent behavior. — [What happened to Databricks Repos?](https://learn.microsoft.com/azure/databricks/repos/what-happened-repos)

#### Jobs: parameters, For each, retries, timeouts, notifications, concurrency
- Notebook task parameters are key-value pairs read in the notebook with `dbutils.widgets`. Job-level parameters are pushed to all tasks that accept key-value parameters. — [Notebook task](https://learn.microsoft.com/azure/databricks/jobs/tasks/notebook#configure-compute-and-dependent-libraries); [Configure and edit Lakeflow Jobs](https://learn.microsoft.com/azure/databricks/jobs/configure-job#select-a-job-to-edit-in-the-workspace)
- A **`For each`** task iterates over a JSON array of inputs. **Concurrency** defaults to 1. The nested task reads `{{input}}`, or `{{input.<key>}}` when iterating over objects. — [For each task](https://learn.microsoft.com/azure/databricks/jobs/tasks/for-each#add-the-%60for-each%60-task-to-a-job)
- When tasks use dynamic values, job parameters are limited to 10,000 characters. A job can have up to 1,000 tasks. — [Lakeflow Jobs limitations](https://learn.microsoft.com/azure/databricks/jobs/#monitoring-and-observability)
- Retries:
  - By default most configurations do not retry.
  - **Serverless jobs auto-optimize retries by default**, and this can be disabled.
  - If both a timeout and retries are set, "the timeout applies to each retry".
  - Run-duration thresholds can trigger a warning event, and a **Timeout** marks the task "Timed Out".
  — [Configure tasks: advanced settings](https://learn.microsoft.com/azure/databricks/jobs/configure-task#copy-a-task-path)
- Bundle task keys include `max_retries`, `min_retry_interval_millis` (default 0 = immediate), `health` rules and `email_notifications` (`on_start`, `on_success`, `on_failure`, `on_duration_warning_threshold_exceeded`). — [Bundle task settings](https://learn.microsoft.com/azure/databricks/dev-tools/bundles/job-task-types#task-settings)
- Notification caveats:
  - Job-level notifications are **not** sent when failed tasks are retried. Use task notifications for per-attempt alerts.
  - At most 3 system destinations per event type.
  - "Succeeded with failures" counts as success.
  — [Add notifications on a job](https://learn.microsoft.com/azure/databricks/jobs/notifications#configure-notifications-on-a-job)
- Jobs support email, Slack, webhook and other destinations. System tables record job runs for monitoring. — [Lakeflow Jobs](https://learn.microsoft.com/azure/databricks/jobs/#monitoring-and-observability)
- "Databricks recommends using the default settings that enforce a single concurrent run for each job." — [Trigger a single job run](https://learn.microsoft.com/azure/databricks/jobs/run-now)

#### Dependencies: serverless environments instead of runtime pip
- Serverless doesn't support compute policies or init scripts, so dependencies go in the notebook's **Environment** side pane, in any `requirements.txt`-valid form. **Apply** installs them and restarts Python. The virtual environment is cached, including across job tasks that share the same dependency set. — [Configure the serverless environment](https://learn.microsoft.com/azure/databricks/compute/serverless/dependencies#add-dependencies-to-the-notebook)
- Serverless CPU architecture is not guaranteed (`aarch64` or `x86_64`), so dependencies must support both. — [Configure the serverless environment](https://learn.microsoft.com/azure/databricks/compute/serverless/dependencies#add-dependencies-to-the-notebook)
- Serverless notebook tasks use the **notebook's environment** by default. This can be overridden with a job-level environment ("Jobs Environment"). — [Configure the environment for job tasks](https://learn.microsoft.com/azure/databricks/compute/serverless/dependencies#configure-environment-for-job-tasks)
- In bundles, `environments` define serverless specs (`environment_version` required, `dependencies` list). For serverless notebook tasks "the environment is accessible in the notebook environment panel". `environment_key` is required only for Python script, wheel and dbt tasks. A serverless notebook job can simply omit cluster settings. — [Bundle resources: job](https://learn.microsoft.com/azure/databricks/dev-tools/bundles/resources#job); [Bundle examples](https://learn.microsoft.com/azure/databricks/dev-tools/bundles/examples#job-configuration)
- On serverless notebook tasks Databricks says to install libraries "using the Environment pane or using `%pip install`". On other compute, use **Dependent libraries**. — [Notebook task](https://learn.microsoft.com/azure/databricks/jobs/tasks/notebook#configure-compute-and-dependent-libraries)
- Classic compute can also use "Environments" dependency mode. This is **Beta**, needs DBR 19 (Beta)+ and standard access mode, and disables init scripts and cluster libraries. — [Manage dependencies using environments](https://learn.microsoft.com/azure/databricks/compute/environments-mode)

#### Source connectivity: SQL Server driver, auth, serverless vs classic
- Microsoft has **deprecated `ActiveDirectoryPassword`** in its SQL drivers:
  - It relies on the ROPC grant, "is incompatible with mandatory Microsoft Entra multifactor authentication (MFA) and might not work in tenants where MFA is enforced".
  - For unattended services, Microsoft recommends managed identity, or `ActiveDirectoryServicePrincipal` (preferably with a certificate), and says "Don't use a Microsoft Entra user account as a service account."
  — [Connect using Microsoft Entra authentication (JDBC)](https://learn.microsoft.com/sql/connect/jdbc/connecting-using-azure-active-directory-authentication?view=sql-server-ver17#connect-using-activedirectorypassword-authentication-mode)
- mssql-jdbc 13.2 added a deprecation warning for `ActiveDirectoryPassword`. — [JDBC driver release notes 13.2](https://learn.microsoft.com/sql/connect/jdbc/release-notes-for-the-jdbc-driver?view=sql-server-ver17#132)
- mssql-jdbc supports `ActiveDirectoryServicePrincipal` (9.2+), `ActiveDirectoryServicePrincipalCertificate` (12.4+) and `ActiveDirectoryManagedIdentity`/`MSI`. Entra modes need "additional jars as dependencies". — [Set the connection properties](https://learn.microsoft.com/sql/connect/jdbc/setting-the-connection-properties?view=sql-server-ver17#properties); [Entra auth driver support](https://learn.microsoft.com/fabric/data-warehouse/entra-id-authentication#driver-support-for-microsoft-entra-authentication)
- Databricks Runtime bundles a **SQL Server connector** (`spark.read.format("sqlserver")` with `host`, `database`, `query`/`dbtable` options).
  - Reads and writes are supported on dedicated and standard compute.
  - Serverless writes are supported for SQL Server and PostgreSQL, among others.
  - "Custom third-party Spark JARs can only be installed on dedicated compute. For serverless or standard compute, use bundled connectors or JDBC Unity Catalog connections."
  — [Spark data sources](https://learn.microsoft.com/azure/databricks/connect/spark-data-sources)
- A **JDBC Unity Catalog connection** holds driver, URL and credentials as a UC securable that hides credentials from users.
  - Works on serverless and on DBR 17.3 LTS+ standard/dedicated: GA on DBR 19+, Public Preview on 17.3 LTS–18.
  - Driver JARs must be in a UC volume.
  - Supports static credentials and OAuth M2M.
  - The JDBC driver has a 400 MiB memory limit.
  — [JDBC connection](https://learn.microsoft.com/azure/databricks/connect/jdbc-connection#limitations)
- For databases that query federation supports, including SQL Server, Databricks recommends **Lakehouse Federation** over a JDBC connection (table-level governance, pushdown). Federation does not support writes. — [JDBC connection: JDBC vs query federation](https://learn.microsoft.com/azure/databricks/connect/jdbc-connection#jdbc-versus-query-federation)
- SQL Server federation supports Entra ID **U2M and M2M (service principal)** OAuth. On the SQL side: `CREATE LOGIN [<app>] FROM EXTERNAL PROVIDER; CREATE USER ...; ALTER ROLE db_datareader ADD MEMBER ...`. — [Configure Microsoft Entra ID for SQL Server federation](https://learn.microsoft.com/azure/databricks/query-federation/sql-server-entra)
- Lakeflow Connect also has a managed SQL Server ingestion connector. It uses Change Tracking or CDC, supports Azure SQL and U2M/M2M Entra or password auth, and lands data in Delta, not Postgres. — [SQL Server connector concepts](https://learn.microsoft.com/azure/databricks/ingestion/lakeflow-connect/sql-server-concepts)

#### Lakebase connection, availability and identity
- Lakebase release history on Azure:
  - Beta on Azure: Feb 4, 2026.
  - **GA on Azure: Mar 2, 2026**, with autoscaling, scale-to-zero, branching and instant restore, plus high availability the same day.
  - GA with the compliance security profile (HIPAA/C5/TISAX/None): Mar 4, 2026.
  - **Lakebase Postgres REST API, CLI and SDKs GA: Aug 14, 2026.**
  - New projects get **password auth disabled by default** (May 21, 2026) and **scale to zero after 24 h idle** by default (May 13, 2026).
  — [Lakebase release notes](https://learn.microsoft.com/azure/databricks/release-notes/lakebase/)
- OAuth tokens for Lakebase:
  - They expire after one hour, but "**Token expiration is enforced only at login. Open connections remain active even after the token expires.**"
  - Tokens are workspace-scoped and SSL is required.
  - The built-in PgBouncer pooler doesn't support OAuth.
  - All connections close after 24 h idle and may close after 3 days.
  — [Lakebase authentication](https://learn.microsoft.com/azure/databricks/oltp/projects/authentication)
- Tokens can be minted with the CLI (`databricks postgres generate-database-credential projects/<p>/branches/<b>/endpoints/<e> --output json`) or with `w.postgres.generate_database_credential(endpoint=...)`, for users (U2M) or service principals (M2M). The docs show psycopg 3 pool and SQLAlchemy `do_connect` token-rotation patterns. — [Lakebase authentication](https://learn.microsoft.com/azure/databricks/oltp/projects/authentication)
- Endpoint types:
  - A branch normally has one read-write endpoint (`ENDPOINT_TYPE_READ_WRITE`) and optionally read replicas (`ENDPOINT_TYPE_READ_ONLY`).
  - Every branch gets an implicit read-write endpoint named `primary`.
  - Writes on a read replica fail.
  — [Computes and endpoints](https://learn.microsoft.com/azure/databricks/oltp/projects/computes-and-endpoints); [Terraform for Lakebase](https://learn.microsoft.com/azure/databricks/oltp/projects/automate-with-terraform#quickstart-manage-a-lakebase-project-with-terraform); [Manage read replicas](https://learn.microsoft.com/azure/databricks/oltp/projects/manage-read-replicas#create-and-manage-read-replicas)
- The Python SDK reads the host from `endpoint.status.hosts.host`. — [Manage computes](https://learn.microsoft.com/azure/databricks/oltp/projects/manage-computes#create-and-manage-computes)
- Each non-owner identity needs a Postgres **OAuth role**, created with `CREATE EXTENSION IF NOT EXISTS databricks_auth; SELECT databricks_create_role('<identity>', '<type>')` or in the UI/REST API. For a service principal the role name is its **client ID (UUID)**, which is case-sensitive. — [Create Postgres roles](https://learn.microsoft.com/azure/databricks/oltp/projects/postgres-roles#create-postgres-roles); [Connect external app using SDK](https://learn.microsoft.com/azure/databricks/oltp/projects/external-apps-connect#1-create-service-principal-with-oauth-secret)
- "Service principals don't inherit default schema permissions." Grants must be explicit. — [Connect a Databricks app to Lakebase](https://learn.microsoft.com/azure/databricks/oltp/projects/tutorial-databricks-apps-autoscaling#step-1-create-your-app-and-database)
- Lakebase uses a two-layer model: workspace-level project permissions, plus a separate Postgres-role layer for schema, table and sequence access. — [CI/CD for Databricks Apps agents](https://learn.microsoft.com/azure/databricks/agents/custom-agents/cicd-agent-app#grant-postgres-permissions-for-lakebase-memory-templates)
- The first connection after scale-to-zero may take longer, so implement retry logic. — [Connect external app using API: troubleshooting](https://learn.microsoft.com/azure/databricks/oltp/projects/external-apps-manual-api#troubleshooting)
- External apps (relevant to Posit Connect):
  - Databricks Python SDK v0.89.0+ (Java 0.73.0+, Go 0.109.0+).
  - An SP OAuth secret of up to 730 days, with "Workspace access" enabled.
  - Call `generate_database_credential()` when opening each pooled connection.
  — [Connect external app using SDK](https://learn.microsoft.com/azure/databricks/oltp/projects/external-apps-connect)

#### Secrets (Azure Key Vault-backed scopes)
- Azure Key Vault-backed scopes are read-only views of the vault. They **support only the "Vault access policy" permission model; Azure RBAC is not supported**.
  - Creating one requires Key Vault Contributor, Contributor or Owner on the vault.
  - Creation grants Get/List on the vault to the Azure Databricks application.
  - Create through `#secrets/createScope` or the CLI (`--scope-backend-type AZURE_KEYVAULT`).
  — [Secret management](https://learn.microsoft.com/azure/databricks/security/secrets/#manage-secret-scopes); [secrets CLI](https://learn.microsoft.com/azure/databricks/dev-tools/cli/reference/secrets-commands)
- Secret ACLs apply at scope level. With an AKV-backed scope, anyone granted the scope can read **all secrets in that vault**, so use separate vaults to restrict access. Grant with `databricks secrets put-acl <scope> <principal> READ`; a service principal is identified by its `applicationId`. — [Secret management: permissions](https://learn.microsoft.com/azure/databricks/security/secrets/#manage-secret-scope-permissions)
- Bundles can declare `secret_scopes` with `backend_type: AZURE_KEYVAULT` (`resource_id`, `dns_name`) and `permissions`. — [Bundle resources: secret_scope](https://learn.microsoft.com/azure/databricks/dev-tools/bundles/resources#secret%5C_scope)

#### Running as a service principal
- Users need the account-level **Service Principal User** role to set a job's **Run as** to a service principal. Service Principal Manager does not include it. Running as an SP "prevents jobs from failing if a user leaves your organization". — [Roles for managing service principals](https://learn.microsoft.com/azure/databricks/security/auth/access-control/service-principal-acl); [Run a job with a Microsoft Entra ID service principal](https://learn.microsoft.com/azure/databricks/jobs/how-to/run-jobs-with-service-principals)
- In bundles, `run_as` takes `user_name` or `service_principal_name` (the application ID), either top-level or per target. If the deploying identity differs from `run_as`, only jobs and pipelines are supported. A service principal for staging and production is described as "the most secure way". — [Specify a run identity for a bundle workflow](https://learn.microsoft.com/azure/databricks/dev-tools/bundles/run-as)
- For CI/CD authentication, Databricks recommends **workload identity federation** because it removes the need for stored secrets. — [CI/CD workflows](https://learn.microsoft.com/azure/databricks/dev-tools/ci-cd/flows#core-principles-of-ci-cd)

### Inferences

#### Migration notebook: what it does, and what to fix before scheduling
Source for all items: [MIG] cells 1–8.
- **Config is hard-coded and has drifted**
  - Cell 2 hard-codes the source host/DB/user, the secret scope/key, the Lakebase project/branch/database, and the 26-entry `TABLE_MIGRATION_REGISTRY`.
  - The markdown in cell 1 has drifted from the code:
    - It names a different source database and Lakebase project than cell 2.
    - Its cell map is off by one.
    - It says "%pip install" while the code uses `subprocess`.
    - It says the notebook is idempotent via "DROP + CREATE" while the default `reload_mode` is `truncate`.
    - It lists 12 "Migrated" tables (largest about 743k rows) while the registry has 26 entries across 4 schemas.
  - The database value in cell 2 carries a contradictory "change to … for production" comment.
  - Fix: move environment values to bundle **target variables → job parameters → `dbutils.widgets.get()`**. Move the registry to a version-controlled YAML/JSON file in the Git folder, or to a control table, and pass an optional `tables` filter parameter.
- **Registry hygiene**
  - Before scheduling, review entries that look like:
    - a misspelled duplicate (transposed letters)
    - a view (`vw_` prefix; `INFORMATION_SCHEMA.COLUMNS` includes views, so it would be **materialized as a table**)
    - two backup tables (`_BAK` and `_BAK_<date>`)
  - All 26 entries are currently `pre_cutover` / `truncate` / enabled.
- **Dependencies**
  - Cell 3 pip-installs `psycopg>=3.1.0` and `databricks-sdk>=0.118.0` into `sys.executable` at runtime and claims "no restart required".
  - If an older `databricks-sdk` is already imported in the Python process, an in-place upgrade won't take effect until Python restarts.
  - Unpinned `>=` floors also make runs non-reproducible.
  - Fix: declare pinned versions in the notebook Environment panel (saved with the notebook, and in PEP 723 headers for source exports) or in a job-level environment with `environment_version: "5"`, then delete cells 3–4. On classic compute, use task libraries.
- **Source authentication**
  - Cell 5 uses `Authentication=ActiveDirectoryPassword` with a **human user principal** and a password from a scope whose `akv-` prefix suggests it is AKV-backed. That is deprecated, MFA-incompatible, and tied to one person.
  - Options, in order of fit:
    - (a) A UC **SQL Server connection with Entra OAuth M2M** plus a foreign catalog. Read with `spark.table("<fed_catalog>.<schema>.<table>")` or `remote_query`. No secrets in code, works on serverless, read-only (which is all this notebook needs).
    - (b) The bundled `sqlserver` connector or a UC JDBC connection with SP credentials.
    - (c) On dedicated classic compute only, raw JDBC with `Authentication=ActiveDirectoryServicePrincipal` and a client ID and secret from a dedicated AKV-backed scope.
  - Whether the current raw `spark.read.jdbc` + `ActiveDirectoryPassword` path even runs on serverless is unverified (see Gaps). Test before relying on it.
- **Lakebase endpoint selection**
  - Cells 5 and 7 (and [SETUP] cell 3) take `endpoints[0]`. Once a read replica exists, that could be a **read-only endpoint**, and TRUNCATE/COPY would fail.
  - Fix: filter on `status.endpoint_type == ENDPOINT_TYPE_READ_WRITE` (or address `…/endpoints/primary`), and wrap `psycopg.connect` in retry/backoff for scale-to-zero wake-ups.
- **Identity in Postgres**
  - The code connects as `w.current_user.me().user_name`. Under `run_as` an SP, the code would connect as the SP.
  - That needs `databricks_create_role('<sp-client-id>','SERVICE_PRINCIPAL')` plus grants: CONNECT; CREATE on the database for `CREATE SCHEMA`; USAGE/CREATE on the target schemas; and TRUNCATE/INSERT/SELECT, or ownership for DROP in `recreate` mode.
  - Tables created earlier by a human's role are owned by that role. Transfer ownership to a shared owner role, or the SP's TRUNCATE/DROP will fail.
- **Token handling is adequate for batch**
  - The 45-minute refresh check before each table is fine, because expiry is only enforced at login.
  - A single very long COPY would not be cut off by token expiry.
- **Schema discovery limitations (cell 6)**
  - Builds `INFORMATION_SCHEMA` filters by f-string interpolation. That is unsafe once table names become job parameters, so validate them against the registry and a strict regex.
  - Fetches `IS_NULLABLE` but never uses it, so every target column is nullable.
  - Migrates **no primary keys, identity/defaults, unique constraints or indexes**.
    - For OLTP tables the apps will write to after cutover, inserts that relied on SQL Server `IDENTITY` will break.
    - Lookups will be slow without indexes.
    - Fix: generate PK/identity/index DDL from `sys.*` or `INFORMATION_SCHEMA` constraint views, and reset identity sequences to MAX+1 after the final load.
  - Leaves identifiers unquoted. Reserved words, spaces or mixed punctuation will break the DDL, and two source names can collide after snake_case. Use `psycopg.sql.Identifier` and a collision check.
  - Maps `varbinary/binary/image` to `bytea`. However, pandas `to_csv` writes Python `bytes` reprs, which COPY CSV will not decode as bytea hex, so binary columns need explicit `\x…` hex encoding.
  - Maps `float(n≤24)` to `double precision` rather than `real` (harmless).
  - Sends unknown types (`sql_variant`, `hierarchyid`, `geography`) to `text`.
- **Load is not atomic (cell 7)**
  - `prepare_target_table` **commits** TRUNCATE (or DROP+CREATE) before the COPY. A failure mid-COPY therefore leaves an **empty** target table, because the rollback only undoes the COPY.
  - Fix: do TRUNCATE + COPY + count check in **one transaction**. TRUNCATE is transactional in Postgres. Alternatively, COPY into a staging table and swap names in one transaction.
  - Also give COPY an explicit column list. With `CREATE TABLE IF NOT EXISTS` plus `truncate`, a source column added later will make COPY fail (count mismatch), or misalign columns if the order differed. Detect schema drift and switch that table to `recreate` before cutover.
- **Verification can raise false alarms**
  - `source_df.count()` and `toPandas()` are two separate JDBC scans. While apps are still writing to Azure SQL before cutover, counts can legitimately differ and be reported as "mismatch".
  - Use `len(pdf)` as the source count, and add checksums (null counts, min/max of keys and dates) for cutover sign-off.
- **The job can report success when everything failed**
  - Errors are caught per table and only printed. Cell 8 never raises, so a scheduled run can finish "Succeeded" with every table failed, and failure notifications would never fire.
  - Fix: end with `if failed or mismatched: raise RuntimeError(...)`. Also persist `results` to an audit table (Delta or Lakebase) for traceability, which is likely valuable in a regulated pharma setting.
- **Cutover hazard: the biggest operational risk of scheduling this notebook**
  - After a table is cut over, Lakebase becomes the system of record.
  - If `cutover_status` is not flipped *before* the next scheduled run, the job will TRUNCATE the live table and overwrite app writes with stale Azure SQL data.
  - Mitigations:
    - (i) A Lakebase-side control table (or table COMMENT marker) that the job checks, refusing to touch tables marked cut over.
    - (ii) A separate, manually triggered "final cutover sync" job with a confirmation parameter.
    - (iii) Pause or remove the scheduled refresh job per table, or entirely after the last cutover.
- **Concurrency and parallelism**
  - Keep `max_concurrent_runs: 1` so a manual run cannot interleave TRUNCATE/COPY with a scheduled one.
  - For parallelism, use a `For each` task over the active tables (concurrency 2–4) with one connection per iteration. At about 3 GB total and under 1M rows per table, the driver-side `toPandas()` path is acceptable.
  - Each iteration is an independent retryable unit once the load is made transactional.
- **Target branch**
  - Both notebooks point at the **production** branch even for pre-cutover test refreshes.
  - Parameterize the branch so testing from Posit Workbench can use a `dev`/`staging` branch, with production touched only by cutover runs.

#### Schema setup notebook: what it does, and what to fix
Source for all items: [SETUP] cells 1–6.
- **What it creates**: schema `mosaic_redesign`, 8 empty app-owned tables and 12 indexes, through 22 `IF NOT EXISTS` statements.
  - From T-SQL DDL: `mosaic_audit_log`, `mosaic_audit_heartbeat`, `mosaic_audit_viewer`, `recent_search`.
  - From Delta DDL: `assistant_query_log`, `assistant_query_cache`, `assistant_daily_brief`, `rte_requests`.
- **Error handling swallows failures**
  - Each statement's exception is caught and printed, then `conn.commit()` runs once at the end.
  - In Postgres, the first failed statement aborts the transaction. Later statements also error, and the final commit cannot persist anything, yet the run still looks successful.
  - Fix: let exceptions propagate (or use autocommit with per-statement checks), and turn cell 6's printout into an **assertion**: 8 tables with the expected column counts.
- **`IF NOT EXISTS` only bootstraps**
  - It never evolves a table: a column added to the DDL list is silently skipped on existing tables.
  - Same-named indexes with changed definitions are also skipped.
  - Once the apps are live, schema changes need versioned migrations (Q4).
- **No GRANTs**
  - Tables will be owned by whoever ran the notebook (today a human OAuth role).
  - Add explicit grants for the app identity, which will connect from Posit Connect as an SP or a password role: USAGE on the schema, DML on the tables.
  - Add `ALTER DEFAULT PRIVILEGES`.
  - Use a dedicated owner role.
- **Design details to confirm with the app team**
  - `rte_requests` has no primary key. That affects updates/ORMs and any future Lakebase CDF; Lakebase CDF requires `REPLICA IDENTITY FULL`, see [Lakebase CDF](https://learn.microsoft.com/azure/databricks/oltp/projects/lakebase-cdf#set-up-lakebase-cdf).
  - The migration registry also contains an `RTE_REQUESTS` table in another schema. Confirm which one the redesigned app uses.
  - `GENERATED ALWAYS AS IDENTITY` rejects explicit IDs unless `OVERRIDING SYSTEM VALUE` is used. This matters if historical rows are ever backfilled.
  - `timestamp DEFAULT CURRENT_TIMESTAMP` replaced T-SQL `SYSUTCDATETIME()`. The stored value follows the session `TimeZone`, so storing UTC explicitly (or using `timestamptz`) avoids client-dependent values.
  - SQL Server's `IGNORE_DUP_KEY` became a unique index, so the app must use `INSERT … ON CONFLICT (event_uid) DO NOTHING`.

#### Suggested target layout (sketch, not validated)
- **Repo** (Git folder in the workspace + Azure DevOps/GitHub remote):
  - `databricks.yml` with `dev`/`prod` targets
  - `notebooks/azure_sql_to_lakebase_sync.ipynb` and `notebooks/lakebase_schema_bootstrap.ipynb`
  - `src/mosaic_lakebase/` holding pure functions: registry normalization, type mapping, naming, DDL generation, endpoint selection, CSV encoding
  - `config/table_registry.yaml`
  - `migrations/` (Q4), `tests/`
  - `skills/azure-sql-to-lakebase/SKILL.md` (Q3)
- **Jobs**:
  - (1) `lakebase_table_sync`: job parameters `source_database`, `lakebase_project`, `lakebase_branch`, `tables` (default `ALL`) and `dry_run`. Schedule PAUSED in prod by default. `max_concurrent_runs: 1`, `max_retries: 1` with a 5-minute interval (only after the load is transactional), `timeout_seconds`, a duration-warning health rule, `on_failure` notifications to a team DL, and a final assert task.
  - (2) `lakebase_schema_migrate`: runs migrations (Q4) and is triggered from CI on merge.
  - (3) A manual `final_cutover_sync` that takes one table per run and requires a confirmation parameter.
- **Identity**: `run_as: service_principal_name: <application-id>` in `prod`. The SP holds:
  - READ ACL on a dedicated AKV-backed scope, only if any secret remains
  - USE CONNECTION on the UC SQL Server connection
  - Lakebase project permission plus a Postgres OAuth role with explicit grants
  - Service Principal User for the deployers or CI identity
- **Environment**: serverless, `environment_version: "5"`, pinned `psycopg[binary]` and `databricks-sdk` (≥ the 0.89.0 Lakebase minimum; keep the team's tested version). Use dedicated classic compute only if the raw JDBC driver path must be kept.

### Gaps
- Not verified whether `spark.read.jdbc` with an explicit `com.microsoft.sqlserver.jdbc.SQLServerDriver` URL and `Authentication=ActiveDirectoryPassword` works on **serverless** today. Which Entra-auth dependency JARs (e.g., msal4j) ship with serverless or DBR 18.x is also unverified. Docs only state that bundled connectors or UC JDBC connections are the supported paths on serverless/standard compute.
- Not verified whether `WorkspaceClient().current_user.me().user_name` returns the SP application ID when a job runs as a service principal. That must match the Lakebase OAuth role name (the client ID), so test it.
- Exact YAML for passing a task value into `For each` inputs (e.g., `{{tasks.<key>.values.<name>}}`) and for setting `environment_key` on a serverless *notebook* task was not confirmed. Run `databricks bundle validate`.
- The current latest `databricks-sdk` PyPI version was not checked. The notebook's floor of 0.118.0 is above the documented Lakebase minimum of 0.89.0.
- Lakebase's default session `TimeZone` was not confirmed.
- Unknown whether the customer workspace uses the compliance security profile/HIPAA, which affects Lakebase and Genie Code defaults.

---

## Q3. What is the Databricks Assistant (now Genie Code) skill mechanism, how are skills discovered and used, and how could the migration notebook and its skill file be used together?

### Takeaway
The `.assistant/skills/<name>/SKILL.md` file referenced by the migration notebook is a **Genie Code agent skill** (Databricks Assistant was renamed Genie Code on Mar 11, 2026). Skills follow the open Agent Skills spec:
- YAML frontmatter with `name` and `description`, then Markdown instructions, optionally with scripts or reference files.
- Genie Code loads a skill automatically when the request matches its description, or when the skill is `@`-mentioned.
- User skills (`/Users/{username}/.assistant/skills/`) are private. Workspace skills (`Workspace/.assistant/skills/`) are shared.

The skill currently lives in one person's home folder and is not in the `.dbc`. Recommended next steps:
- Export it, put it under Git next to the notebook code, and register that folder (or promote it to a workspace skill).
- Use it with Agent mode as a design-time helper for onboarding new tables: review type mappings, propose registry entries and PK/identity DDL, and draft migrations.
- Keep the actual data movement in the deterministic, reviewed job.

### Cited Findings
- "Databricks Assistant is now Genie Code" (Mar 11, 2026), and "Genie Code Agent mode is now generally available for data science, data engineering, and dashboard authoring." — [Release notes, March 2026](https://learn.microsoft.com/azure/databricks/release-notes/product/2026/march#databricks-assistant-is-now-genie-code)
- Skills timeline:
  - Jan 6, 2026: "You can now create skills to extend Databricks Assistant in Agent mode… User skills follow the open Agent Skills standard and are automatically loaded when relevant." — [Release notes, January 2026](https://learn.microsoft.com/azure/databricks/release-notes/product/2026/january)
  - April 2026: "Workspace skills for Genie Code are now available." — [Platform release notes index](https://learn.microsoft.com/azure/databricks/release-notes/product/)
- Skill mechanics:
  - Skill types and locations: **Workspace skills** at `Workspace/.assistant/skills/`, available to everyone; admins create them and can grant others access to add more. **User skills** at `/Users/{username}/.assistant/skills/`, available only to the owner and useful "for prototyping a skill before promoting it".
  - Each skill needs its own folder containing `SKILL.md` with frontmatter `name` and `description`, followed by Markdown instructions. Optional scripts and reference files use relative paths.
  - Loading: "Genie Code automatically loads skills when relevant, based on your request and the skill's description. You can also manually invoke skills by `@` mentioning them."
  - Edits apply only in a **new chat**.
  - Built-in skills exist for notebooks, Unity Catalog exploration, dashboards, pipelines and MLflow.
  — [Extend Genie Code with agent skills](https://learn.microsoft.com/azure/databricks/genie-code/skills)
- Skill folders and maintenance:
  - Extra skill folders can be registered from the Genie Code pane (Settings → Add folder), for example a **Git folder** "next to the project code". Registered skills can be toggled on or off.
  - Best practices include focused scope, clear name and description, examples and edge cases, scripts for repeatable actions, and "Version control your skills" with Git folders.
  — [Extend Genie Code with agent skills](https://learn.microsoft.com/azure/databricks/genie-code/skills)
- Agent Skills spec constraints as summarized by Microsoft: `name` is at most 64 lowercase letters, digits or hyphens and **must match the parent directory name**. `description` is at most 1,024 characters and "should include keywords that help agents identify relevant tasks". — [Agent Skills (Microsoft Agent Framework)](https://learn.microsoft.com/agent-framework/agents/skills#skill-structure)
- Agent mode:
  - It "automates multi-step workflows, plans solutions, retrieves relevant assets, runs code, uses cell outputs to improve results, and fixes errors automatically".
  - Prompts and context sent to the model include table/column names, code, **data samples from tables, and cell outputs**, while respecting UC permissions.
  - It requires partner-powered AI features and a supported Geo. Without partner-powered AI, Genie Code works without agentic capabilities.
  — [Genie Code Agent mode](https://learn.microsoft.com/azure/databricks/genie-code/agent-mode)
- Agent mode has been on by default for compliance-security-profile workspaces with HIPAA controls since May 25, 2026. — [Release notes, May 2026](https://learn.microsoft.com/azure/databricks/release-notes/product/2026/may)
- Genie Code beyond chat:
  - Can run as a **Lakeflow Jobs task (Beta, Aug 27, 2026)**. — [Release notes, August 2026](https://learn.microsoft.com/azure/databricks/release-notes/product/2026/august)
  - **Scheduled tasks are GA (Sep 1, 2026).** — [Release notes, September 2026](https://learn.microsoft.com/azure/databricks/release-notes/product/2026/september)
  - Genie Code usage is billed pay-as-you-go. — [AI/BI and Genie One release notes 2026](https://learn.microsoft.com/azure/databricks/ai-bi/release-notes/2026#july-2026)
- Separately, `databricks aitools install` installs Databricks skills and plugins into external coding agents (Claude Code, Copilot, Cursor). The skills follow the same Agent Skills standard. — [Agent skills for AI coding assistants](https://learn.microsoft.com/azure/databricks/agent-skills/)
- What [MIG] says about its skill (cell 1 markdown):
  - It lives in the author's user skills folder under `azure-sql-to-lakebase/`.
  - It contains the T-SQL→Postgres type map, snake_case rules, the Lakebase SDK connection/OAuth pattern, buffer and row-by-row COPY patterns, the full migration loop, token refresh, the cutover workflow and "10 documented gotchas".
  - Its stated purpose: "The Databricks Assistant reads this skill automatically when asked about Azure SQL → Lakebase migrations."
  — [MIG](<../../context/Azure SQL to Lakebase Migration.dbc>) cell 1

### Inferences
- **Get the skill out of a personal folder**
  - As a user skill it is invisible to teammates, and it is at risk if the author leaves.
  - Export it, commit it to the project repo (`skills/azure-sql-to-lakebase/SKILL.md` plus `references/` and `scripts/`), and either register the Git folder as an extra skill folder for each engineer or have an admin copy it to `Workspace/.assistant/skills/`.
  - Keep `name: azure-sql-to-lakebase`, matching the folder.
  - Write a keyword-rich `description`: Azure SQL, SQL Server, T-SQL, Lakebase, Postgres, migrate table, cutover, type mapping, COPY, OAuth token.
- **Remove duplication**
  - The type map and naming rules now exist in both the notebook and the skill, and they will drift.
  - Put them in `src/mosaic_lakebase/*.py` and have the skill *reference* that module (relative path or workspace path) instead of restating it.
  - Add the notebook's newly found issues to the skill's gotchas: `endpoints[0]` vs read-write, non-atomic TRUNCATE, view materialization, bytea CSV encoding, missing PK/identity, the cutover overwrite hazard.
- **How the notebook and skill work together for future tables**
  - (1) In a Genie Code chat on the repo's notebook, `@azure-sql-to-lakebase` + "onboard table X". The agent profiles source columns, flags unmapped types, collisions and reserved words, proposes PK/identity/index DDL, and proposes a registry YAML entry.
  - (2) The engineer reviews and opens a PR.
  - (3) CI tests on a throwaway Lakebase branch (Q4/Q5).
  - (4) The scheduled job picks the table up through the registry.
  - (5) At cutover, a person runs the manual cutover job and flips the status.
  - The same skill can serve app-owned schema work (like [SETUP]): drafting `CREATE`/`ALTER` migrations in the team's chosen tool.
- **Guardrails**
  - Don't let Agent mode run migration cells against production. Agent mode executes code and sends samples and outputs to the model.
  - Pharma governance should review partner-powered AI settings, because these tables include HCP-related data.
  - Prefer running the agent against a dev branch or non-production catalog.
  - The Beta "Genie Code task" in Jobs is not appropriate for unattended production data movement. It could be used for low-risk recurring review tasks, e.g., "summarize last night's sync results and flag mismatches".

### Gaps
- Not found: an explicit GA or preview label on the skills page itself. The release notes show user skills (Jan 2026) and workspace skills (Apr 2026) "available", and Agent mode GA (Mar 2026). Whether skills carry a separate preview status, and whether they are available in the customer's region/Geo and compliance profile, should be confirmed in the workspace.
- The content of the referenced SKILL.md itself was not available, because it is not in the `.dbc` files. Its frontmatter validity (e.g., `name` matching the folder) is unknown.
- Not verified whether skills registered from extra folders are per-user settings or can be set workspace-wide.

---

## Q4. How should schema changes on Lakebase Postgres be managed (idempotent DDL notebooks vs. migration tools), run from Databricks or CI with OAuth tokens, and tested on Lakebase branches?

### Takeaway
`CREATE … IF NOT EXISTS` notebooks are fine for one-time bootstrap, but they cannot express or record change (ALTERs, renames, backfills, rollbacks).

Because Lakebase is standard Postgres (PG 17 default), any wire-compatible migration tool works. Pick a SQL-first tool with a history table. For a team without an ORM, dbmate, Sqitch or Flyway Community are good fits. Alembic suits SQLAlchemy app code, and Atlas suits declarative and linting needs. Seed it with:
- a baseline (V1 = the [SETUP] DDL)
- for the 26 migrated tables, a baseline taken after cutover with `pg_dump --schema-only`

Run migrations from CI or a Databricks job as a service principal: mint a 1-hour OAuth token immediately before running and pass it as the password, over SSL, directly to the endpoint (not PgBouncer). Test every change first on a short-TTL Lakebase branch forked from production, using schema diff, then replay the same migration on production. Lakebase does not merge branches.

### Cited Findings
- Lakebase branches:
  - Copy-on-write: creation is instant regardless of size and has no performance impact on production.
  - "Changes to a child branch don't affect its parent". The isolation **includes Postgres roles, databases and GRANT/REVOKE state**.
  - **Branch reset** refreshes a child from its parent.
  — [Branches](https://learn.microsoft.com/azure/databricks/oltp/projects/branches#how-branches-work)
- Branch creation and cleanup:
  - Create from current data or from a past point in time.
  - Expiration (TTL) can be 1 h, 1 d, 7 d, custom up to 30 days, or never. The docs suggest **2–4 hours for CI/CD pipelines**.
  - Expiring branches are billed only for changed data; permanent branches are billed for full size.
  - SDK: `w.postgres.create_branch(parent=…, branch=Branch(spec=BranchSpec(source_branch=…, ttl=Duration(...))), branch_id=…).wait()`.
  — [Manage branches](https://learn.microsoft.com/azure/databricks/oltp/projects/manage-branches#understanding-branches)
- CLI: `databricks postgres create-branch projects/<p> <id> --json '{"spec":{"source_branch":"projects/<p>/branches/production", ...}}'`. An expiration policy is mandatory (`ttl`, `expire_time` or `no_expiry`). The branch automatically gets a primary read-write endpoint. `delete-branch` returns immediately but deletes asynchronously. Branch protection is set with `update-branch … spec.is_protected`. — [Databricks CLI for Lakebase](https://learn.microsoft.com/azure/databricks/oltp/projects/cli#manage-branches)
- **Schema diff** compares a child branch with its parent side by side as DDL: tables, columns, indexes, constraints. — [Manage branches: compare schemas](https://learn.microsoft.com/azure/databricks/oltp/projects/manage-branches#create-and-manage-branches)
- Databricks' branch-based workflow tutorial:
  - Generate migrations "with your framework (Prisma, Alembic, Django, etc.)", run them on a personal branch, review with schema diff.
  - "Promotion is not automatic. To promote your changes, run the same migration against your `development` branch… There's no Lakebase-specific step involved."
  — [Tutorial: Branch-based development workflow](https://learn.microsoft.com/azure/databricks/oltp/projects/dev-workflow-tutorial#develop-your-feature)
- Lakebase does not upgrade major Postgres versions in place. A new major version means a new project plus `pg_dump`/`pg_restore`. New projects default to Postgres 17. — [Postgres version support](https://learn.microsoft.com/azure/databricks/oltp/projects/postgres-version-support#major-version-upgrades); [Connect a Databricks app to Lakebase](https://learn.microsoft.com/azure/databricks/oltp/projects/tutorial-databricks-apps-autoscaling#step-1-create-your-app-and-database)
- Authentication for tools:
  - Tokens come from `databricks postgres generate-database-credential <endpoint> --output json` (with `token` and `expire_time`) or the SDK.
  - Expiry is enforced only at login. SSL is required. PgBouncer does not support OAuth.
  - The docs give an SQLAlchemy `do_connect` hook that injects a fresh token. That is the same mechanism Alembic's `env.py` engine can use.
  — [Lakebase authentication](https://learn.microsoft.com/azure/databricks/oltp/projects/authentication)
- Lakebase can also be managed with Terraform (`databricks_postgres_branch`, `databricks_postgres_endpoint`) and with bundles (`postgres_branch` and related resources). — [Terraform for Lakebase](https://learn.microsoft.com/azure/databricks/oltp/projects/automate-with-terraform#quickstart-manage-a-lakebase-project-with-terraform); [Bundles release notes](https://learn.microsoft.com/azure/databricks/release-notes/dev-tools/bundles)
- **Alembic** (SQLAlchemy):
  - Autogenerate compares ORM metadata with the live schema. It detects added/removed tables and columns, nullability, and basic index, unique and FK changes. Type and server-default comparison are configurable options, and their defaults vary by Alembic version (verify for the pinned version).
  - It **cannot detect table or column renames**, which come out as drop + add, or unnamed constraints.
  - "It is *always* necessary to manually review and correct the candidate migrations."
  — [Alembic autogenerate](https://alembic.sqlalchemy.org/en/latest/autogenerate.html)
- **Flyway** (Redgate):
  - Community, Teams and Enterprise editions.
  - Community requires hand-written versioned scripts and lacks undo migrations and dry runs, which are paid features.
  — secondary sources: [FitGap: Redgate Flyway](https://us.fitgap.com/products/redgate-flyway); [Redgate Flyway docs (pipelines)](https://documentation.red-gate.com/fd/flyway-development-and-deployment-pipelines-180715693.html)
- **Liquibase**: Liquibase Community moved to the **Functional Source License (FSL) starting with 5.0** (announced Sep 30, 2025). Free production use continues, and each version reverts to Apache 2.0 two years after release. — [Liquibase blog](https://www.liquibase.com/blog/liquibase-community-for-the-future-fsl); [BusinessWire](https://www.businesswire.com/news/home/20250930639100/en/Liquibase-Unveils-Liquibase-Secure-and-Liquibase-5.0-A-New-Era-for-Mission-Critical-Database-Change)
- **Sqitch**:
  - A plan file declares dependencies between changes, with no sequence numbers and a Merkle-tree integrity check.
  - Scripts are native SQL. It is "not tied to any framework, ORM, or platform".
  - It keeps a deployment registry in the target database, supports PostgreSQL, and is MIT-licensed.
  — [Sqitch on GitHub](https://github.com/sqitchers/sqitch)
- **Atlas**:
  - Declarative mode (`schema apply` against desired state) or versioned mode (`migrate diff` to generate migrations, `migrate apply`).
  - `migrate lint` flags destructive, locking or data-loss changes.
  — [Atlas migrations summary (secondary)](https://tessl.io/registry/testland/atlas-migrations)
  - Atlas needs a **dev database** to validate and normalize. It must be empty ("Atlas rejects a dev-database that already contains objects"), must match the target's engine, version and extensions, and is typically `docker://postgres/<ver>/dev`. — [Atlas dev database](https://atlasgo.io/concepts/dev-database)
- **dbmate**:
  - A lightweight, framework-agnostic migrator: plain SQL files with required up and down sections.
  - Connects through `DATABASE_URL`.
  - Runs each migration in a transaction by default.
  - Maintains a `schema.sql` dump and records versions in `schema_migrations`.
  - Supports PostgreSQL, MIT-licensed.
  — [dbmate on GitHub](https://github.com/amacneil/dbmate)

### Inferences
- **Recommended approach for MOSAIC**
  - The apps are on Posit; R/Python apps probably use DBI/psycopg rather than an ORM. Use a **SQL-first versioned tool**.
  - **dbmate** is the simplest: single binary, `DATABASE_URL`, transactional, `schema.sql` diffable in PRs.
  - **Sqitch** suits explicit dependencies plus deploy/verify/revert.
  - **Flyway Community** suits a JVM-centric team.
  - Choose **Alembic** only if the Python app code already uses SQLAlchemy models.
  - Choose **Atlas** if declarative schema-as-code and `migrate lint` in CI are wanted. Its dev DB can be a local Docker Postgres 17; a Lakebase branch is not empty, so it cannot serve.
  - Liquibase is viable, but legal should review the FSL change.
- **Seeding the tool**
  - `V0001__bootstrap_mosaic_redesign.sql` = the 22 statements from [SETUP]. Keep `IF NOT EXISTS` so the baseline can be applied to the already-bootstrapped production branch, or mark it applied with the tool's baseline feature.
  - Add a later migration that adds GRANTs and default privileges.
  - For each migrated legacy schema, after final cutover take `pg_dump --schema-only` as that schema's baseline. Every later app-driven change, including PKs/identity and indexes the copy didn't create, becomes a numbered migration.
- **Where migrations run**
  - Preferred: CI (Azure DevOps or GitHub Actions).
    - Authenticate the Databricks CLI as the deployment SP via workload identity federation.
    - Run `databricks postgres generate-database-credential projects/<p>/branches/<b>/endpoints/primary --output json`.
    - Export the token as `PGPASSWORD` or inside `DATABASE_URL`, with `sslmode=require` and the SP client ID as the user.
    - Run the tool.
  - Alternative inside Databricks: a bundle job task (Python) that mints the token with `WorkspaceClient().postgres.generate_database_credential()` and invokes Alembic, or shells out to a pinned dbmate/sqitch binary, running as the SP.
  - Because expiry is checked only at login, one token per run is enough. For tools that open several connections over a long run, mint the token right before each connection, or use the `do_connect` hook.
- **Branch-based gate per PR**
  - (1) Create branch `ci-<pr>` from `production` with `ttl` of 2–4 h.
  - (2) Resolve its `primary` endpoint.
  - (3) Apply pending migrations.
  - (4) Optionally run the data-sync job against the branch for the affected tables (job parameter `lakebase_branch=ci-<pr>`) and the app's smoke tests.
  - (5) Inspect schema diff, or `pg_dump --schema-only` diff, against the parent.
  - (6) Let the branch expire or delete it.
  - Promotion = running the same migration on `production` after merge, never merging branches.
  - The CI SP needs a Postgres role on the branch. Roles and grants made on a child don't propagate, so create roles on `production` once; children inherit them at fork time.
- **Data migrations and cutover rehearsals**
  - Fork `production` into `rehearsal` and run the full sync plus the app's write tests against it, so the cutover runbook is proven against real-size data at zero risk to production.
  - Keep point-in-time branching as the rollback story if a production migration misbehaves.
- **Protect production**
  - Mark the `production` branch protected (`spec.is_protected`).
  - Restrict who holds DDL rights: the owner role is used only by the migration SP. App roles get DML only.

### Gaps
- Could not retrieve Redgate's current official edition/feature matrix for Flyway (page 404). The Flyway edition claims above rely on secondary sources; verify before choosing Flyway.
- The Atlas lint/declarative description comes from a secondary summary. The dev-database facts are from atlasgo.io.
- Not verified whether Lakebase's built-in schema diff can be called through the API/CLI for CI. It is documented only as a UI feature.
- Not checked whether any migration tool needs Postgres privileges or extensions that Lakebase restricts. Advisory locks, `CREATE SCHEMA` and similar are expected to work as standard Postgres, but this was not tested.

---

## Q5. How should the notebooks be tested: unit tests for pure functions, integration tests on a throwaway Lakebase branch, and CI patterns for Databricks notebooks?

### Takeaway
Move the pure logic out of the notebooks into an importable `src/` package and unit-test it with pytest on every PR. The functions are `normalize_table_registry_entry`, `map_sql_type`, `to_snake_case`, `generate_target_name`, plus new helpers for endpoint selection, DDL generation and CSV/bytea encoding.

Then gate with `databricks bundle validate`/`deploy` to a non-production target. Run integration tests in which a Lakebase branch with a 2–4 h TTL is forked from production, migrations and a sync of a few representative tables are applied, and row counts, checksums and schema expectations are asserted. CI authenticates as a service principal via workload identity federation.

### Cited Findings
- For Python and R notebooks, "Databricks recommends storing functions and their unit tests outside of notebooks". The workspace has tooling to discover and run Python unit tests. — [Unit testing for Databricks notebooks](https://learn.microsoft.com/azure/databricks/notebooks/testing#organize-functions-and-unit-tests)
- Tests can also run inside notebooks: `unittest` in a cell, a widget-driven "Test/Normal" mode, `%run` of a shared-code notebook, code in Git folders, the web terminal, or scheduled test jobs with notification e-mails. — [Test Databricks notebooks](https://learn.microsoft.com/azure/databricks/notebooks/test-notebooks)
- Databricks' three testing layers:
  - (1) Unit tests on importable `src/` modules with pytest, on every PR.
  - (2) Bundle validation: `bundle validate` locally, and preferably `bundle deploy` to a non-production workspace in CI.
  - (3) Integration tests in staging with "row-count or schema expectations".
  - "All tests pass on the main branch and in staging" is the promotion gate.
  — [Developer best practices: testing and observability](https://learn.microsoft.com/azure/databricks/developers/best-practices#testing-and-observability)
- Further CI/CD guidance:
  - Run pytest for business logic and `bundle validate` in CI.
  - Use Azure DevOps (or GitHub Actions) with bundles on Azure.
  - Keep dev, staging and production environments separate.
  - Use workload identity federation for authentication.
  — [CI/CD workflows on Databricks](https://learn.microsoft.com/azure/databricks/dev-tools/ci-cd/flows#core-principles-of-ci-cd)
- For GitHub Actions running notebooks, Databricks discourages personal access tokens and recommends a Microsoft Entra ID service principal, because notebooks run with the token identity's permissions. — [Software engineering best practices for notebooks](https://learn.microsoft.com/azure/databricks/notebooks/best-practices#optional-step-6-set-up-the-repo-to-test-the-code-and-run-the-notebook-automatically-whenever-the-code-changes)
- Unit tests should not run against production data; use synthetic data that resembles it. — [Unit testing for Databricks notebooks: write unit tests](https://learn.microsoft.com/azure/databricks/notebooks/testing#write-unit-tests)
- Ephemeral Lakebase branches:
  - TTL-based expiration exists for "CI/CD environments" and "automated testing", with 2–4 hours suggested for pipelines. — [Manage branches](https://learn.microsoft.com/azure/databricks/oltp/projects/manage-branches#understanding-branches)
  - Branches can be created and deleted from the CLI and SDK, and each new branch gets a primary read-write endpoint. — [Databricks CLI for Lakebase](https://learn.microsoft.com/azure/databricks/oltp/projects/cli#manage-branches)

### Inferences
- **Unit-test targets and cases** (derived from [MIG] cells 2 and 6):
  - `normalize_table_registry_entry`:
    - accepts plain strings
    - rejects `a.b.c` and invalid status/mode values
    - lowercases status and mode
    - trims whitespace
  - Add a test that the registry has no case-insensitive duplicates and no `vw_`/`_BAK` entries unless explicitly allowed.
  - `to_snake_case`:
    - `MOSAICAI_PROMPTS_EDIT→mosaicai_prompts_edit`
    - `EmpID→emp_id`
    - `HCPAttributes→hcp_attributes`
    - `userID→user_id`
    - names with spaces or symbols should fail validation
  - Add a collision test over each table's column set, and a reserved-word check (`user`, `order`, `group`, …) that forces quoting.
  - `map_sql_type`:
    - `nvarchar` with length −1 → `text`
    - `varchar(50)` → `varchar(50)`
    - `decimal(18,2)` → `numeric(18,2)`
    - `bit` → `boolean`
    - `datetimeoffset` → `timestamptz`
    - `uniqueidentifier` → `uuid`
    - `money` → `numeric(19,4)`
    - unknown → `text`
  - Parametrize from a CSV kept in the skill's `references/`, so the skill and the code share one truth.
  - `generate_target_name`: schema is lowercased and the table is snake-cased.
- **Integration-test targets**, against a throwaway Lakebase branch or locally against Docker Postgres 17 for speed:
  - CSV/COPY round-trip of tricky values:
    - NULL vs empty string
    - embedded quotes, commas and newlines
    - a literal `\N` string
    - nullable ints (Int64)
    - booleans
    - timestamps and datetimeoffset
    - **bytea**
    - non-ASCII (NVARCHAR) text
  - TRUNCATE+COPY atomicity: inject a failure mid-COPY and assert the previous data survives.
  - Endpoint selection when a read replica exists.
  - Expired-token reconnect.
  - Schema drift: add a source column and assert the job fails loudly or recreates as designed.
  - Final assertion: the job fails if any table fails or mismatches.
- **CI pipeline sketch** (Azure DevOps or GitHub Actions; SP via workload identity federation):
  - On each PR: `pytest` for the `src/` unit tests (no Databricks needed), then `databricks bundle validate -t ci`.
  - Create Lakebase branch `ci-<pr>` with `ttl` of 2–4 h.
  - Apply schema migrations (Q4).
  - `databricks bundle deploy -t ci`, then `databricks bundle run lakebase_table_sync -t ci` with `--lakebase_branch=ci-<pr> --tables=<2–3 small representative tables>`. The source side reads a non-production Azure SQL database or a federated catalog.
  - Assertions (counts, checksums, `information_schema` column/PK expectations, the [SETUP] 8-table check).
  - Delete the branch, or let it expire.
  - On merge to main: deploy to `prod` (the job stays paused or manual for cutover operations) and run migrations against `production`.
- **Notebook-level smoke test**: keep a `dry_run` job parameter. When true, the job validates the registry, connects to both sides, discovers schema, generates and prints DDL, and **writes nothing**. This is useful both in CI and before every manual cutover run.

### Gaps
- No Databricks-published, end-to-end example was found that combines CI with ephemeral Lakebase branches and integration tests. The pipeline above is a synthesis of the documented pieces: branch TTL guidance for CI, CLI branch commands, and the testing layers.
- Not verified whether the Lakebase schema-diff feature has an API usable as an automated CI assertion. The fallback is diffing `pg_dump --schema-only` output.

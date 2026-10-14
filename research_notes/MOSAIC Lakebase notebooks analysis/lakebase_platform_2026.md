# Databricks Lakebase on Azure Databricks: platform state as of October 2026, and fit with the two MOSAIC notebooks

Research date: 2026-10-06. All learn.microsoft.com and docs.posit.co pages were accessed on 2026-10-06. Where a page shows its own date, it is given. Release-note entries carry their own dates. Status labels follow the sources: **GA**, **Public Preview**, **Beta**, **Private Preview**, **Deprecated**. Internal identifiers from the notebooks (hosts, projects, users) were never used in any search or external request.

Notebook facts this research was aimed at (read from the local dumps, never executed):
- Both notebooks use `WorkspaceClient().postgres.list_endpoints(parent="projects/<id>/branches/production")`, then take `endpoints[0]`, read `ep.status.hosts.host` and call `w.postgres.generate_database_credential(endpoint=ep.name).token`.
- They connect with psycopg 3 on port 5432 with `sslmode=require` to `databricks_postgres`, as `w.current_user.me().user_name`, which is the human user running the notebook.
- They require `databricks-sdk>=0.118.0` and `psycopg>=3.1.0`.
- The migration notebook reads Azure SQL over Spark JDBC with `Authentication=ActiveDirectoryPassword`, using a user account and a password from a secret scope. The path is JDBC → `toPandas()` → CSV buffer → `COPY`, with `truncate` or `recreate` reload modes. It reconnects with a new token after 45 minutes.
- The migration notebook's first markdown cell shows a regional host of the form `<uid>.database.eastus2.azuredatabricks.net`.
- The schema-setup notebook creates schema `mosaic_redesign` and 8 tables. It uses identity columns, `gen_random_uuid()` and covering (`INCLUDE`) indexes, runs each DDL statement in a per-statement try/except inside a single transaction, then calls `commit()`.

---

## Q1. What is Lakebase today (Provisioned vs Autoscaling, Neon-based architecture, GA status on Azure and East US 2, Postgres versions and extensions), which API/SDK do `w.postgres.list_endpoints` and `generate_database_credential` belong to and from which SDK version, and is `databricks_postgres` the default database?

### Takeaway
Lakebase is Databricks' managed Postgres. The current version, called "Lakebase Autoscaling" (projects → branches → computes/endpoints → databases/roles, managed through the Postgres API `w.postgres`), has been **GA on Azure since 2026-03-02**. `eastus2` was one of the three original Azure Beta regions (2026-02-04). The older "Lakebase Provisioned" (database instances, `w.database`) is closed to new creation (since 2026-03-12), and all existing instances were moved onto the Autoscaling platform by July 2026. The notebooks' calls are exactly the documented, now-**GA** (2026-08-14) Postgres API pattern. `w.postgres` first shipped in databricks-sdk v0.75.0 (2025-12-17), so the notebooks' `>=0.118.0` floor is more than enough. `databricks_postgres` is the default database.

### Cited Findings
**Product variants and timeline**
- Two Lakebase database types exist. **Lakebase Autoscaling** "organizes resources as projects containing branches and databases". **Lakebase Provisioned** "organizes resources as database instances". "You can't create new Provisioned databases after March 12, 2026, but you can add existing ones as app resources." — [Add a Lakebase resource to a Databricks app](https://learn.microsoft.com/azure/databricks/dev-tools/databricks-apps/lakebase)
- Since 2026-03-12, new instances are created with autoscaling, scale-to-zero, branching and instant restore by default, whether through the UI, the Database instance API, Terraform or the SDKs. — [Lakebase release notes (Azure)](https://learn.microsoft.com/en-us/azure/databricks/release-notes/lakebase/)
- "Azure Databricks has upgraded all Lakebase Provisioned instances to the Lakebase Autoscaling platform. The upgrade completed in July 2026."
  - Existing connection strings, API calls, bundles and Terraform keep working.
  - The Provisioned UI stayed available until **September 1, 2026**.
  - Upgraded instances appear in both the Database instance API and the Postgres API.
  - Source: [Upgrade to Autoscaling](https://learn.microsoft.com/azure/databricks/oltp/upgrade-to-autoscaling)
- Provisioned private-preview features are being retired:
  - The "Forward ETL (private preview)" feature on Provisioned "is no longer supported"; use Lakebase Change Data Feed instead.
  - The PostgREST REST API (private preview) is "no longer supported" and is replaced by the Data API on Autoscaling.
  - Source: [Upgrade to Autoscaling](https://learn.microsoft.com/azure/databricks/oltp/upgrade-to-autoscaling)
- Billing separates the two products. `DATABASE` = "Lakebase Provisioned database instances, the original version of Lakebase". `LAKEBASE` = "Lakebase, the current version of Lakebase". — [Billable usage system table reference](https://learn.microsoft.com/azure/databricks/admin/system-tables/billing)
- Dated Azure timeline, all from the [Lakebase release notes (Azure)](https://learn.microsoft.com/en-us/azure/databricks/release-notes/lakebase/):
  - 2026-01-15: Lakebase is managed from the "Lakebase App".
  - 2026-02-04: Azure **Beta** in `eastus2`, `westeurope` and `westus`.
  - 2026-03-02: **GA on Azure**, plus 11 more regions; high availability added the same day.
  - 2026-03-04: GA with compliance security profile.
  - 2026-03-12: OAuth roles can be created from the UI and REST API; budget policies and tags added.
  - 2026-03-16: Lakebase can be a Databricks Apps resource.
  - 2026-05-12: Lakebase CDF enters **Public Preview**.
  - 2026-05-13: new instances scale to zero by default after 24 hours.
  - 2026-05-21: password authentication is off by default for new projects.
  - 2026-06-18: Postgres 18 support.
  - 2026-08-14: **Lakebase Postgres APIs GA**.
  - 2026-08-31: SLA introduced.
  - 2026-10-01: LTAP Direct Writes GA.
- AWS GA was 2026-02-03. This date comes from a search-result summary that cites [DBTA](https://www.dbta.com/Editorial/News-Flashes/Databricks-Lakebase-is-Now-Generally-Available-Delivering-Reliability-Performance-and-Governance-173362.aspx) and the [AWS Lakebase release notes](https://docs.databricks.com/aws/en/release-notes/lakebase/); neither page was opened.

**Architecture and Neon origin**
- "Lakebase separates stateless Postgres compute from a durable storage layer of safekeepers, pageservers, and cloud object storage". The hierarchy is Project → Branch (e.g., `production`) → Compute (read-write) + Database (e.g., `databricks_postgres`). — [Core concepts](https://learn.microsoft.com/azure/databricks/oltp/projects/core-concepts)
- Databricks announced on 2025-05-14 that it would acquire Neon, a serverless Postgres company, for about $1B. Neon "split compute from storage, paused to zero when idle, and let developers create instant database branches". — [SiliconANGLE, 2025-05-14](https://siliconangle.com/2025/05/14/databricks-buys-serverless-database-startup-neon-reported-1b/); [TechRepublic](https://www.techrepublic.com/article/news-databricks-neon-acquisition/)

**Regions**
- Supported Azure regions (page ms.date 2026-09-11): `eastus`, **`eastus2`**, `centralus`, `northcentralus`, `southcentralus`, `westus`, `westus2`, `canadacentral`, `brazilsouth`, `northeurope`, `uksouth`, `westeurope`, `francecentral`, `germanywestcentral`, `australiaeast`, `centralindia`, `southeastasia`, `eastasia`, `japaneast`. "Your Lakebase project is created in your Databricks workspace region." — [Manage projects](https://learn.microsoft.com/en-us/azure/databricks/oltp/projects/manage-projects)

**Postgres versions and extensions**
- Postgres 16, 17 and 18 are supported, with 17 as the default. Projects created through the Database instance API or the Provisioned UI use Postgres 16. — [Postgres compatibility](https://learn.microsoft.com/azure/databricks/oltp/projects/compatibility); [Upgrade to Autoscaling](https://learn.microsoft.com/azure/databricks/oltp/upgrade-to-autoscaling)
- Supported minor versions are 16.15, 17.11 and 18.6 (2026-08-24). Minor updates apply at the next compute restart. — [Release notes](https://learn.microsoft.com/en-us/azure/databricks/release-notes/lakebase/)
- Available extensions include `uuid-ossp`, `vector` (pgvector 0.8.x), `unaccent`, `tcn` and PostGIS. `pg_uuidv7` is **not** available. "On Postgres 16 and Postgres 17, use `gen_random_uuid()`". On Postgres 18, `uuidv7()` and `uuidv4()` are built in. — [Postgres extensions](https://learn.microsoft.com/azure/databricks/oltp/projects/extensions)
- Lakebase Search (`lakebase_vector`, `lakebase_text`) became GA on 2026-09-18. — [Release notes](https://learn.microsoft.com/en-us/azure/databricks/release-notes/lakebase/)
- Differences from standard Postgres:
  - Scale-to-zero closes idle connections, so session context is lost (temporary tables, prepared statements, advisory locks, LISTEN/NOTIFY).
  - Unlogged tables are not kept across restarts or scale-to-zero.
  - Local temp disk is 20 GiB or 15 GiB × the maximum CU, whichever is larger.
  - "Access to Postgres logs is not supported." This conflicts with the Beta `system.lakebase.postgres_logs` table (see Q5).
  - Source: [Postgres compatibility](https://learn.microsoft.com/azure/databricks/oltp/projects/compatibility)

**API and SDK surface used by the notebooks**
- REST management happens under `/api/2.0/postgres/` (projects, branches, endpoints). `databricks postgres list-endpoints projects/<p>/branches/<b>` lists an endpoint's computes. "In the SDK, access the host via `endpoint.status.hosts.host` (not `endpoint.status.host`)". The Python sample is `list(w.postgres.list_endpoints(parent="projects/my-project/branches/production"))`. — [Manage computes](https://learn.microsoft.com/en-us/azure/databricks/oltp/projects/manage-computes) (ms.date 2026-09-25)
- `w.postgres.generate_database_credential(endpoint="projects/.../branches/.../endpoints/...")` returns `token` and `expire_time` (1 hour). The REST equivalent is `POST /api/2.0/postgres/credentials` with body `{"endpoint": ...}`. — [About authentication](https://learn.microsoft.com/en-us/azure/databricks/oltp/projects/authentication); [Connect external app using API](https://learn.microsoft.com/en-us/azure/databricks/oltp/projects/external-apps-manual-api)
- The Postgres APIs became **GA on 2026-08-14** across REST, CLI and the Python, Java and Go SDKs. They cover projects, branches, endpoints, databases, roles, credentials, synced tables and catalogs. — [Release notes](https://learn.microsoft.com/en-us/azure/databricks/release-notes/lakebase/)
- databricks-sdk-py changelog:
  - v0.75.0 (2025-12-17): "Add `databricks.sdk.service.postgres` package. Add w.postgres workspace-level service."
  - v0.79.0 (2026-01-22): "Add `generate_database_credential()` method for [w.postgres]".
  - v0.118.0 (2026-06-18) exists and added the Data API methods (`create_data_api()` and related).
  - The latest release is v0.147.0 (2026-10-06).
  - The older `w.database` (Provisioned) service had credential and role methods by v0.58.0 (2025-07-09).
  - Source: [databricks-sdk-py CHANGELOG](https://github.com/databricks/databricks-sdk-py/blob/main/CHANGELOG.md), read through its raw file in two partial reads.
- The Databricks docs give slightly different minimum versions:
  - A tutorial says "Version 0.81.0 or later includes the `generate_database_credential()` method" — [Connect a custom Databricks app](https://learn.microsoft.com/azure/databricks/oltp/projects/tutorial-databricks-apps-autoscaling)
  - The external-app guide requires "Python v0.89.0+" — [Connect external app using SDK](https://learn.microsoft.com/azure/databricks/oltp/projects/external-apps-connect)
  - The SQLAlchemy example pins `databricks-sdk==0.79.0` — [About authentication](https://learn.microsoft.com/en-us/azure/databricks/oltp/projects/authentication)

**Default database and branch**
- A new project includes a `production` root branch, which cannot be deleted. It also includes a primary read-write compute, a database "named `databricks_postgres`", and a Postgres role for the creator's Databricks identity, which owns that database. — [Manage projects](https://learn.microsoft.com/en-us/azure/databricks/oltp/projects/manage-projects); [Create Postgres roles](https://learn.microsoft.com/en-us/azure/databricks/oltp/projects/postgres-roles)

### Inferences
- The notebooks target the current product and the current, GA API. Nothing in their connection code is legacy Provisioned-only (`w.database`).
- Lakebase Autoscaling is very likely built on Neon technology. The docs describe "safekeepers" and "pageservers", which are Neon's storage terms, and Databricks acquired Neon. The pages read here do not name Neon, so treat this as an inference.
- The notebooks need no `uuid-ossp` extension, because `gen_random_uuid()` is built in on Postgres 16, 17 and 18.

### Gaps
- No Data + AI Summit 2025/2026 announcement pages were opened, so the original (2025) Public Preview dates of Lakebase Provisioned are not cited here.
- The changelog was only partly read (offsets 0–200k of ~528k characters). The exact first release of `w.database` may be earlier than v0.58.0.
- The Postgres version of the notebooks' project is unknown. It is 16 if the project was created through the Database instance API and 17 by default otherwise.

---

## Q2. Authentication: OAuth credential lifetime and refresh, whether sessions survive token expiry, service principals (M2M) for apps, native Postgres password roles, and token-rotation patterns for long-running apps and pools

### Takeaway
- OAuth database credentials last **1 hour**, but expiry is checked **only at login**. An open connection keeps working until the platform's own limits: a 24-hour idle timeout, a 3-day maximum connection life, scale-to-zero, failover or restart.
- Apps should use a **service principal (M2M OAuth)** with a matching Postgres role, plus a pool that asks for a new credential every time it opens a connection.
- Native Postgres **password roles** are supported and are the only option for the built-in PgBouncer pooler. Since **2026-05-21** they are **disabled by default for new projects**.

### Cited Findings
- How tokens work:
  - "Token lifetime: OAuth tokens expire after one hour."
  - "Expiration enforcement: Token expiration is enforced only at login. Open connections remain active even after the token expires."
  - "OAuth tokens are workspace-scoped and must belong to the same workspace that owns the project. Cross-workspace token authentication is not supported."
  - "SSL required … (typically `sslmode=require`)".
  - Source: [About authentication](https://learn.microsoft.com/en-us/azure/databricks/oltp/projects/authentication)
- All connections, whatever the authentication method, are subject to a "24-hour idle timeout" and a "3-day maximum connection life". — [About authentication](https://learn.microsoft.com/en-us/azure/databricks/oltp/projects/authentication)
- Pooling and OAuth: "The built-in PgBouncer connection pooler does not support OAuth authentication. Use native Postgres password roles for pooled connections." — [About authentication](https://learn.microsoft.com/en-us/azure/databricks/oltp/projects/authentication)
- Recommended rotation patterns:
  - Rotate before the hourly expiry, either by checking the expiry on each use or by refreshing from a background thread.
  - psycopg 3 example: a `psycopg_pool.ConnectionPool` with a custom `connection_class` whose `connect()` calls `w.postgres.generate_database_credential(endpoint=...)` for every new connection.
  - SQLAlchemy example: a `do_connect` event listener that refreshes the token within 2 minutes of expiry.
  - Source: [About authentication](https://learn.microsoft.com/en-us/azure/databricks/oltp/projects/authentication)
- Service-principal setup for external apps:
  - The service principal needs an OAuth secret "with 730-day max lifetime", the "Workspace access" entitlement, and its client ID (UUID), which becomes the Postgres role name and `PGUSER`.
  - Create its role with `databricks_create_role('{client-id}','SERVICE_PRINCIPAL')` and grant CONNECT, USAGE and SELECT/INSERT/UPDATE/DELETE.
  - "Both the workspace OAuth token and database credential expire after 60 minutes."
  - SDK minimum: "Python v0.89.0+".
  - Source: [Connect external app using SDK](https://learn.microsoft.com/azure/databricks/oltp/projects/external-apps-connect)
- Token scope: "Database credentials are workspace-scoped. While the `endpoint` parameter is required, the returned token can access any database or project in the workspace that the service principal has permissions for." — [Connect external app using API](https://learn.microsoft.com/en-us/azure/databricks/oltp/projects/external-apps-manual-api)
- Two-step REST flow for languages without an SDK:
  1. `POST {host}/oidc/v1/token` with `grant_type=client_credentials&scope=all-apis`.
  2. `POST {host}/api/2.0/postgres/credentials` with `{"endpoint": ...}`.
  - "First connection after idle may take longer as Lakebase starts compute from zero."
  - Source: [Connect external app using API](https://learn.microsoft.com/en-us/azure/databricks/oltp/projects/external-apps-manual-api)
- Native password roles:
  - Passwords "do not expire automatically" and are managed by Postgres, with no workspace integration.
  - They "support the built-in PgBouncer connection pooler".
  - "Password connections are disabled by default for new Lakebase projects". This changed on 2026-05-21; existing projects are unaffected.
  - Source: [About authentication](https://learn.microsoft.com/en-us/azure/databricks/oltp/projects/authentication); [Release notes](https://learn.microsoft.com/en-us/azure/databricks/release-notes/lakebase/)
- Password rules: at least 12 characters with mixed character classes, "validated at creation time to verify 60-bit entropy". They are stored as SCRAM-SHA-256 verifiers. SDK, REST and CLI role creation does not return the password; use SQL `CREATE ROLE … PASSWORD` or the UI "Reset password". — [Create Postgres roles](https://learn.microsoft.com/en-us/azure/databricks/oltp/projects/postgres-roles)
- "**Copy OAuth Token** generates a token for your current login identity only, not for service principals or other users." — [Quickstart](https://learn.microsoft.com/azure/databricks/oltp/projects/connect-overview)

### Inferences
- **Match:** the notebooks' use of 1-hour OAuth tokens, `sslmode=require`, and reconnecting before 60 minutes (at 45 minutes) is consistent with the documentation.
- The refresh is **conservative rather than necessary**, because an already-open session survives token expiry. The real need is a new token for each **new** connection.
- Long-running sync jobs are actually limited by the 3-day maximum connection life, the 24-hour idle timeout, scale-to-zero suspension and HA failover. That argues for retry-on-disconnect logic in addition to the 45-minute refresh.
- The notebooks run as the **human user**. For a scheduled sync job, the documented pattern is a service principal (M2M) with its own OAuth role, so the job does not depend on an individual's identity.
- For app connection pools on Posit Connect, there are two practical choices:
  - OAuth with per-connection token minting. This needs code, and the pooler is not available.
  - A password role through PgBouncer (`-pooler` host). This needs password connections enabled in project settings, plus secret storage and manual rotation.

### Gaps
- No documentation was found on rate limits for `generate_database_credential` calls.
- No documentation was found on whether a specific project ACL (for example CAN USE) is needed to mint credentials, beyond workspace membership and the "Workspace access" entitlement.

---

## Q3. Connecting from outside Databricks (Posit Workbench and Posit Connect; R and Python), network options, connection limits per size, built-in pooling, scale-to-zero and cold start

### Takeaway
- Lakebase is reachable from external apps over standard Postgres drivers on port 5432, with TLS, at a regional host (`*.database.<region>.azuredatabricks.net`).
- Posit Workbench (managed Databricks OAuth credentials) and Posit Connect (Databricks OAuth integrations: Viewer, Service Account, Workload Identity) can both supply the Databricks token that Lakebase credential minting needs. No Posit page mentions Lakebase, so the Lakebase step (calling the credential API) is custom code.
- If the workspace uses Private Link, external Postgres clients need the separate "Inbound Private Link for performance-intensive services" endpoint, which is in **Public Preview**.
- Connection limits scale with CU, from 105 at 0.5 CU up to 3,993.
- PgBouncer pooling, in transaction mode for up to 10,000 clients, works only with password roles.
- Scale-to-zero resume takes "a few hundred milliseconds".

### Cited Findings
**Posit**
- Posit Workbench "has a native integration for Databricks that includes support for managed Databricks OAuth credentials and a dedicated pane in the RStudio Pro IDE". Managed credentials "work with the Databricks CLI, most official packages and SDKs, and database drivers for Python and R" and "are refreshed automatically while your session is active". — search summary of [Posit Workbench user guide](https://docs.posit.co/ide/server-pro/user/posit-workbench/guide/databricks.html)
- Workbench-managed credentials (doc v2026.09.0):
  - "The Workbench-managed `.databrickscfg` file has a profile called `workbench` containing the short-lived tokens".
  - Available in RStudio Pro, VS Code and Positron Pro.
  - Documented as compatible with `sparklyr` (1.8.4+), `odbc` (1.4.0+), the Databricks R SDK, the Databricks SDK for Python, `pyodbc`, Databricks Connect and the Databricks CLI.
  - The page does not mention Lakebase, Postgres, Azure or Entra.
  - Source: [Workbench-managed Databricks Credentials](https://docs.posit.co/ide/server-pro/user/posit-workbench/managed-credentials/databricks.html)
- Posit Connect (Admin Guide v2026.09.0):
  - Databricks OAuth integration types are **Viewer** (the content viewer's identity), **Service Account** (a service principal with a stored OAuth secret) and **Workload Identity** (a service principal without stored secrets).
  - For Workload Identity, "Connect also sets `CONNECT_DATABRICKS_INTEGRATION_GUID` in the content environment so that the Databricks SDK can discover the integration and obtain tokens without additional configuration."
  - Example scopes: `all-apis`, `sql`, `offline_access`, `openid`, `profile`, `email`.
  - The page does not mention Lakebase, Postgres, Azure or Entra.
  - Source: [Posit Connect: Databricks OAuth integration](https://docs.posit.co/connect/admin/integrations/oauth-integrations/databricks/)
- Posit's overview pages also say "Content can run using the viewer's identity, a service account, or workload identity" and "Workbench handles token refreshes in the background". — [Posit: Databricks configuration for Workbench and Connect](https://docs.posit.co/data-sources/admin/databricks.html); [Posit partnerships: Databricks](https://docs.posit.co/partnerships/databricks/)
- Known issue: a GitHub issue reports that the Databricks OAuth U2M flow "cannot complete on Posit Workbench" in Positron Data Connections; the browser is redirected to localhost. The issue's date was not retrieved. — [posit-dev/positron#16409](https://github.com/posit-dev/positron/issues/16409)

**Network**
- Private Link for Lakebase:
  - Inbound Private Link (port 443) is required for REST/API traffic.
  - "Inbound Private Link for performance-intensive services (Service Direct Private Link, port 5432) is required … when your applications connect to Lakebase from outside the Azure Databricks workspace … [with] a regional connection string". Postgres clients use a "dedicated high-performance regional ingress" at `*.database.<region>.azuredatabricks.net`.
  - Apps running inside the workspace (Databricks Apps, Feature Store) don't need it.
  - Source: [Private Link for Lakebase](https://learn.microsoft.com/azure/databricks/oltp/projects/private-link)
- Inbound private connectivity to performance-intensive services (Zerobus Ingest, Lakebase Autoscaling) "is in Public Preview". — [Private Link concepts](https://learn.microsoft.com/azure/databricks/security/network/concepts/privatelink-concepts)
- Provisioned-era connection strings without a region still work over the standard inbound Private Link. The new regional strings need Service Direct Private Link when the workspace uses Private Link and clients are outside it. — [Upgrade to Autoscaling](https://learn.microsoft.com/azure/databricks/oltp/upgrade-to-autoscaling)
- Workspace IP access lists (Premium plan) control access "to the workspace or workspace-level APIs". The page does not say whether they apply to Lakebase Postgres (5432) connections. — [Manage IP access lists](https://learn.microsoft.com/azure/databricks/security/network/front-end/ip-access-list)

**Connection limits and sizing** (from [Manage computes](https://learn.microsoft.com/en-us/azure/databricks/oltp/projects/manage-computes), ms.date 2026-09-25)
- Maximum connections by compute size:

| CU | RAM | Max connections |
| --- | --- | --- |
| 0.5 | ~1 GB | 105 |
| 1 | ~2 GB | 218 |
| 2 | ~4 GB | 443 |
| 3 | ~6 GB | 668 |
| 4 | ~8 GB | 894 |
| 8 | ~16 GB | 1,795 |
| 16 | ~32 GB | 3,597 |
| 24+ | — | 3,993 |

- "When autoscaling is enabled, the maximum number of connections is determined by the smaller of your maximum CU and 8× your minimum CU."
- The docs conflict on connection figures:
  - The pooling page cites 1,678 connections for 8 CU and 3,357 for 16 CU. — [Use connection pooling](https://learn.microsoft.com/en-us/azure/databricks/oltp/projects/connection-pooling)
  - The synced-tables page says "Lakebase Postgres supports up to 1,000 concurrent connections". — [Synced tables](https://learn.microsoft.com/en-us/azure/databricks/oltp/projects/sync-tables)

**Built-in pooling** (from [Use connection pooling](https://learn.microsoft.com/en-us/azure/databricks/oltp/projects/connection-pooling))
- Settings: PgBouncer in `pool_mode=transaction`, `max_client_conn=10000`, `default_pool_size=0.9 × max_connections` per user/database pair, `max_prepared_statements=1000` and `query_wait_timeout=120`.
- Access: hostname `<endpoint-id>-pooler.<region>…` (or `-ro-pooler` for reads); password roles only.
- Not available through the pooler: SQL `PREPARE`, session `SET`, temp tables held across transactions, `WITH HOLD` cursors, advisory locks and LISTEN/NOTIFY. Use a direct connection for `pg_dump` and schema migrations.

**Scale-to-zero and cold start** (from [Scale to zero](https://learn.microsoft.com/en-us/azure/databricks/oltp/projects/scale-to-zero) and [Manage computes](https://learn.microsoft.com/en-us/azure/databricks/oltp/projects/manage-computes))
- Default timeout: 24 hours. Range: 60 seconds to 7 days.
- Resume: "within a few hundred milliseconds". "Applications should implement connection retry logic".
- Suspension resets session context (cache, temp tables, prepared statements, connection pools).
- Scale-to-zero is only available for computes of ≤32 CU, and not at all with HA.

### Inferences
- **Python apps on Posit Connect or Workbench:**
  - Use `WorkspaceClient()`, which picks up Workbench's `workbench` profile, or Connect's Service Account / Workload Identity integration through the SDK.
  - Combine it with the documented psycopg `ConnectionPool` custom connection class or the SQLAlchemy `do_connect` hook. This should work without stored Postgres passwords.
  - This combination is not documented end-to-end by either vendor.
- **R apps (DBI/RPostgres/pool):**
  - No Databricks R SDK call for Lakebase credentials was found.
  - R apps would call `POST /api/2.0/postgres/credentials` themselves (for example with httr2), using a Workbench or Connect token, and must ensure each **new** pooled connection gets a fresh credential. One way is to rebuild the `pool` every less than 60 minutes. Note that existing connections survive expiry.
  - The alternative is a password role, through PgBouncer or directly.
- **Viewer-identity content on Connect** would need Postgres roles for each viewer. A Databricks **group** role lets every member authenticate with their own token (see Q4), which keeps this manageable.
- If the workspace uses Private Link, Posit servers outside the workspace VNet will need the Public Preview Service Direct Private Link endpoint on 5432. Firewalls must allow outbound 443 (token minting) and 5432 (Postgres).
- For a low-traffic app on a 0.5–1 CU compute, the 105–218 connection ceiling matters. Count Connect processes × pool size, plus up to 16 connections per synced-table pipeline (see Q5).

### Gaps
- No Posit documentation or blog post about Lakebase was found.
- Whether Connect's Databricks integrations support Azure Databricks with Microsoft Entra ID tokens, rather than Databricks OAuth app connections, is not stated in the pages read.
- How R content on Connect retrieves the integration token (for example the `connectapi` function names) was not retrieved.
- Whether IP access lists or context-based ingress rules cover port-5432 Lakebase traffic is undocumented in the pages read.

---

## Q4. Postgres roles and permissions: mapping of Databricks users, groups and service principals; privileges to create schemas and tables; ownership and grant practice; Unity Catalog integration

### Takeaway
- Databricks identities become Postgres **OAuth roles**. You create them with the `databricks_auth` extension's `databricks_create_role(name, 'USER'|'SERVICE_PRINCIPAL'|'GROUP')`, or through the UI, REST or SDK.
- The role name is the email (users), the application ID (service principals) or the display name (groups). New roles get only LOGIN, and privileges are standard `GRANT`s.
- Only the project owner's role is created automatically, and it owns `databricks_postgres`. Objects belong to whoever creates them.
- Moving ownership to a group needs a documented workaround through a temporary role.
- A Lakebase database can be registered in Unity Catalog as a **read-only** catalog, queryable only from Serverless SQL warehouses.

### Cited Findings
**Default and system roles**
- Project creation adds:
  - a role for the owner's identity, which owns `databricks_postgres` and is a member of `databricks_superuser`;
  - `databricks_superuser`, which inherits `pg_read_all_data`, `pg_write_all_data` and `pg_monitor`;
  - system roles: `cloud_admin`, `databricks_control_plane`, `databricks_monitor`, `databricks_replicator`, `databricks_writer_<dbid>` and `databricks_reader_<dbid>`.
  - Source: [Create Postgres roles](https://learn.microsoft.com/en-us/azure/databricks/oltp/projects/postgres-roles)

**Creating OAuth roles**
- SQL path: `CREATE EXTENSION IF NOT EXISTS databricks_auth;` (per database), then `SELECT databricks_create_role('identity_name','identity_type');`.
- It requires "`CREATE` and `CREATE ROLE` permissions on the database" and a session authenticated by OAuth. "Native Postgres authenticated sessions cannot create OAuth roles".
- The function "creates a Postgres role with `LOGIN` permission only".
- SDK and REST: `w.postgres.create_role(parent="projects/…/branches/…", role=Role(spec=RoleRoleSpec(identity_type=…, postgres_role=…)))`. `postgres_role` is the email, application ID (UUID) or group display name.
- Source: [Create Postgres roles](https://learn.microsoft.com/en-us/azure/databricks/oltp/projects/postgres-roles)
- Group-based authentication:
  - Any direct or indirect member of a Databricks group can log in as the group's role, using the group role name as the user and their own OAuth token.
  - Group membership is validated only at login.
  - The group must be assigned to the workspace, and names are case-sensitive.
  - Renaming a user email or group "break[s]" authentication and grants.
  - Role names are limited to 63 characters.
  - Source: [Create Postgres roles](https://learn.microsoft.com/en-us/azure/databricks/oltp/projects/postgres-roles)
- "Any role created in the Lakebase UI is created with `databricks_superuser` privileges. There is a limit of 500 roles per branch." — [Manage projects](https://learn.microsoft.com/en-us/azure/databricks/oltp/projects/manage-projects)
  - This conflicts with the roles page, where the UI OAuth flow is followed by explicit grants and adding `databricks_superuser` is optional for password roles. — [Create Postgres roles](https://learn.microsoft.com/en-us/azure/databricks/oltp/projects/postgres-roles)

**Privileges and ownership**
- "By default, a newly created Postgres role has no database permissions." Use standard `GRANT`:
  - `GRANT CREATE ON SCHEMA` and `GRANT USAGE ON SCHEMA`;
  - `GRANT SELECT/INSERT/UPDATE/DELETE ON ALL TABLES IN SCHEMA`;
  - `ALTER DEFAULT PRIVILEGES [FOR ROLE creator] IN SCHEMA … GRANT … ON TABLES TO …`.
  - Source: [Manage database permissions](https://learn.microsoft.com/en-us/azure/databricks/oltp/projects/manage-roles-permissions)
- `databricks_superuser` "should not be used in automated applications". Database owners have full control and own the objects they create. — [Manage database permissions](https://learn.microsoft.com/en-us/azure/databricks/oltp/projects/manage-roles-permissions)
- Ownership transfer: "use a temporary shared role as an intermediate step … You can't do this directly with a standard `ALTER TABLE ... OWNER TO` command". `REASSIGN OWNED BY` moves all objects and must run in each database. — [Transfer Postgres object ownership](https://learn.microsoft.com/en-us/azure/databricks/oltp/projects/transfer-object-ownership)
- Adding a Lakebase resource to a Databricks App creates a role named after the app service principal's client ID and grants it `CONNECT` and `CREATE` on the database, which "let the app create schemas and tables". — [Add a Lakebase resource to a Databricks app](https://learn.microsoft.com/azure/databricks/dev-tools/databricks-apps/lakebase)

**Project permissions (control plane, separate from database access)**
- Levels are CAN CREATE, CAN USE and CAN MANAGE.
- Defaults: the owner and workspace admins get CAN MANAGE; workspace users get CAN CREATE.
- "Project permissions and database access are separate".
- Source: [Manage projects](https://learn.microsoft.com/en-us/azure/databricks/oltp/projects/manage-projects)

**Unity Catalog integration**
- Registering a database creates "a read-only Unity Catalog catalog that mirrors your Postgres database structure".
- Requirements and limits:
  - `CREATE CATALOG` privilege, and a **Serverless** SQL warehouse (Pro and Classic return `PERMISSION_DENIED`).
  - One database per catalog; metadata is cached; branched databases cannot be registered separately.
- "Permissions set in Unity Catalog control query access through SQL warehouses. Direct connections to your Lakebase database use Postgres roles and permissions independently."
- SDK: `w.postgres.create_catalog(...)`.
- Source: [Register a Lakebase database in Unity Catalog](https://learn.microsoft.com/en-us/azure/databricks/oltp/projects/register-uc)

### Inferences
- **Notebook fit:**
  - Both notebooks create schemas and tables as the human who runs them, so that person's role owns the 8 `mosaic_redesign` tables and every migrated table.
  - Unless that person is the project owner, someone must first create their OAuth role and grant CREATE on `databricks_postgres`, or add them to `databricks_superuser`.
  - The Posit app identity (ideally one service principal per environment, or a group role) then needs:
    - explicit `USAGE` on each schema;
    - `SELECT/INSERT/UPDATE/DELETE` on the tables;
    - `ALTER DEFAULT PRIVILEGES` for tables created later.
- **Group ownership recommended:** making a Databricks group (for example the data-engineering team) the owner of the app schemas avoids objects being tied to one person's role. That matters when someone leaves the company or their email changes. Use the documented temporary-role and `REASSIGN OWNED` workaround.
- **`recreate` mode loses grants:** in the migration notebook, `DROP TABLE` + `CREATE` throws away any grants given to app roles on that table, and only the owner can drop. Under standard Postgres semantics, `truncate` mode keeps the table and its grants, which makes it the safer default once app grants exist. This is an inference from Postgres semantics, not a Lakebase-specific document.
- **Unity Catalog registration** gives governed, read-only analytics access from serverless SQL to the app-written MOSAIC tables, for example to join audit or prompt logs with lakehouse data. It does not govern Posit's direct Postgres connections.

### Gaps
- Not verified: whether `INSERT` on a table with `GENERATED ALWAYS AS IDENTITY` needs any extra privilege on the identity sequence in Lakebase.
- No Lakebase document was found on row-level security on native tables, beyond the note that it cannot be set on synced tables.

---

## Q5. Data movement: synced tables, Lakebase → Delta CDC, Lakehouse Federation, Lakeflow Connect for SQL Server, and what could replace or complement the JDBC truncate-and-reload

### Takeaway
- **Synced tables** (UC Delta/Iceberg/views → Lakebase; Snapshot, Triggered or Continuous) are the managed reverse-ETL path. They suit **read-only** reference tables that already exist in Delta, such as HCP attributes, call details and ad-market data. They produce read-only, pipeline-owned Postgres tables and need a primary key.
- **Lakebase Change Data Feed** (**Public Preview**, 2026-05-12) streams changes from app-written Lakebase tables back into UC Delta history tables.
- **Lakeflow Connect's SQL Server connector** (GA 2025-08-26) and **Lakehouse Federation for SQL Server** (GA 2024-08-01) are governed alternatives to the notebook's raw JDBC reads from Azure SQL.

### Cited Findings
**Synced tables** (from [Serve lakehouse data with synced tables](https://learn.microsoft.com/en-us/azure/databricks/oltp/projects/sync-tables))
- Sources: managed and external Delta tables, managed and external Iceberg tables, views and materialized views. Each sync produces a UC synced table plus "A Postgres table in Lakebase (read-only…)".
- "In Postgres, the Unity Catalog schema name becomes the Postgres schema name".
- Sync modes:
  - **Snapshot** is a full copy each run, "10x more efficient if modifying >10% of source data".
  - **Triggered** applies incremental inserts, updates and deletes, and is expensive at intervals under 5 minutes.
  - **Continuous** is streaming with a minimum interval of 15 seconds.
  - Triggered and Continuous need a change data feed on the source, either write-time CDF or Automatic CDF.
- Primary key rules:
  - The primary key is required, and its columns are not nullable. "Rows with nulls in primary key columns are excluded from the sync."
  - Duplicate keys make the pipeline fail unless a **Timeseries Key** is configured for deduplication.
- Capacity:
  - "Each synced table uses up to 16 connections", which count toward the compute's limit.
  - The total synced data counts toward the branch storage quota. Databricks recommends ≤1 TB for tables that need refreshes.
  - A source table can have up to 20 synced tables.
  - Names must match `[A-Za-z0-9_]+`.
  - Only additive schema changes are allowed for Triggered and Continuous. The definition can't be changed in place; changing it means re-creating the synced table.
- Throughput: "approximately 150 rows per second per Capacity Unit (CU)" for Continuous and Triggered, and "up to 2,000 rows per second per CU" for Snapshot.
- Ownership: synced tables are owned by `databricks_writer_<dbid>`. The creator gets `SELECT`, `DELETE` and `TRUNCATE`, but **not** `INSERT` or `UPDATE`; "direct writes are overwritten on the next refresh". Only read queries, index creation and `DROP` are recommended.
- Type mapping: `TIMESTAMP` → `TIMESTAMP WITH TIME ZONE`, `TIMESTAMP_NTZ` → `TIMESTAMP WITHOUT TIME ZONE`, and `STRING` → `TEXT`, which can be overridden to `varchar(n)` through `type_overrides`. ARRAY, MAP and STRUCT become JSONB. GEOGRAPHY, GEOMETRY and VARIANT are not supported.
  - The TIMESTAMP → TIMESTAMPTZ mapping applies to synced tables created after 2025-08-25. — [Azure Databricks release notes, Aug 2025](https://learn.microsoft.com/azure/databricks/release-notes/product/2025/august)
- Scheduling: the first snapshot runs automatically. After that, Snapshot and Triggered runs come from a "Database Table Sync pipeline" task in Lakeflow Jobs, using a table-update trigger or a cron schedule.
- **LTAP Direct Writes** (**GA 2026-10-01**) writes the initial load and Snapshot refreshes straight to the storage layer, bypassing the compute endpoint. It requires Postgres 16, 17 or 18 and can only be chosen when the synced table is created. — [Release notes](https://learn.microsoft.com/en-us/azure/databricks/release-notes/lakebase/); [Synced tables](https://learn.microsoft.com/en-us/azure/databricks/oltp/projects/sync-tables)

**Lakebase → Delta (CDC)** (from [Lakebase Change Data Feed](https://learn.microsoft.com/en-us/azure/databricks/oltp/projects/lakebase-cdf), **Public Preview**)
- Every insert, update and delete is captured from the WAL by the `wal2delta` extension into UC managed Delta tables named `lb_<table>_history`, "batched and flushed every ~15 seconds". Each row carries `_pg_change_type`, `_pg_lsn`, `_pg_xid` and `_timestamp`.
- Requirements and setup:
  - A workspace admin must enable the preview.
  - Postgres 16, 17 or 18.
  - `REPLICA IDENTITY FULL` on each table.
  - The feed is configured per schema; new tables join automatically.
  - A table must contain at least one row before it appears.
- Not supported: partitioned tables, destination catalogs on default storage, and destination storage reachable only through a private endpoint.
- Schema changes trigger a re-snapshot of the affected table.

**SQL Server sources**
- Lakeflow Connect's Microsoft SQL Server connector became **GA on 2025-08-26**. — [Azure Databricks release notes, Aug 2025](https://learn.microsoft.com/azure/databricks/release-notes/product/2025/august)
- How it works:
  - It supports Azure SQL Database and Managed Instance and uses Change Tracking (recommended when the table has a primary key) or CDC.
  - It needs an ingestion gateway plus a pipeline, Unity Catalog and serverless.
  - Maximum 250 tables per pipeline.
  - Source: [SQL Server connector concepts](https://learn.microsoft.com/azure/databricks/ingestion/lakeflow-connect/sql-server-concepts); [SQL Server connector overview](https://learn.microsoft.com/azure/databricks/ingestion/lakeflow-connect/sql-server-overview); [Ingest data from SQL Server](https://learn.microsoft.com/azure/databricks/ingestion/lakeflow-connect/sql-server-pipeline)
- Conflict on authentication methods:
  - The concepts page lists "User-to-machine (U2M) OAuth with Microsoft Entra ID", "Machine-to-machine (M2M) OAuth with Microsoft Entra ID" and "Username and password".
  - The overview page's authentication table marks OAuth U2M and M2M as "Not supported" and only basic username/password as supported.
- Query-based connectors (no gateway, scheduled, cursor column) also support SQL Server. — [Query-based connectors](https://learn.microsoft.com/azure/databricks/ingestion/lakeflow-connect/query-based-overview)
- Lakehouse Federation connectors, including PostgreSQL and Microsoft SQL Server, became **GA on 2024-08-01**, with SSO for SQL Server and Private Link from serverless. — [Azure Databricks release notes, Aug 2024](https://learn.microsoft.com/azure/databricks/release-notes/product/2024/august)
- "In addition to the SQL Server ingestion connector …, Databricks offers a zero-copy connector in Lakehouse Federation." — [SQL Server connector concepts](https://learn.microsoft.com/azure/databricks/ingestion/lakeflow-connect/sql-server-concepts)
- For third-party reverse ETL, the docs point to Partner Connect tools such as Census and Hightouch. — [Synced tables](https://learn.microsoft.com/en-us/azure/databricks/oltp/projects/sync-tables)

### Inferences
- **Read-mostly reference tables that already exist in Delta** (e.g., `hcp_attributes_*`, `call_detail_*`, `admarket_*`, `*_HO*_DATA`, `fieldlens_metrics`):
  - A Snapshot synced table on a nightly or table-update trigger, ideally with LTAP Direct Writes, would replace the JDBC copy from Azure SQL.
  - Lakebase would be fed straight from the governed gold layer, removing the dependency on Azure SQL for those tables.
  - **Snapshot throughput:** about 2,000 rows/s/CU, so a ~743k-row table on a 1 CU compute takes roughly 6 minutes.
  - **Caveats:**
    - (a) The app can never write to these tables.
    - (b) A unique, non-null primary key, or a timeseries key, is needed.
    - (c) The Postgres schema equals the UC schema name, so app SQL that expects `mosaicaiderm.*` either needs a matching UC schema name or must be repointed.
    - (d) Timestamps arrive as `timestamptz` rather than the notebook's `timestamp`.
    - (e) Each synced table can use up to 16 connections during a sync.
- **Tables the apps write to** (prompts, edits, favorites, tracking, compliance confirmations, the `mosaic_redesign` audit and assistant tables):
  - Keep them as native Lakebase tables, as the notebooks do.
  - Optionally add Lakebase CDF (Preview) to land change history in UC for analytics and audit. That needs `REPLICA IDENTITY FULL` and at least one row.
- **One-time migration from Azure SQL:** the notebook's JDBC → `COPY` approach is reasonable at ~3 GB. Two governed alternatives exist:
  - A Lakehouse Federation SQL Server connection, which keeps credentials in a UC connection and allows zero-copy reads.
  - The Lakeflow Connect SQL Server connector, which lands Delta copies with CT/CDC; a synced table would then push them to Lakebase.
  - Both add a hop. A truncate-and-COPY under a short maintenance window remains the simplest final cutover for read/write tables.
- **Pre-cutover refresh needs a lock:** the migration notebook commits `TRUNCATE` and `COPY` in separate transactions, so readers can briefly see an empty table. That is fine while apps still use Azure SQL. During cutover it is safe only because apps are drained, and the "pre_cutover" refresh must be switched off once a table is cut over. The notebook's `cutover_status` registry is designed for that.

### Gaps
- No preview or GA label was found on the Autoscaling synced-tables page itself, so synced-table GA status on Azure is unconfirmed.
- The Data API (PostgREST replacement) status was not checked.
- Whether Lakehouse Federation's PostgreSQL connector is a supported way to reach Lakebase, as opposed to UC registration, was not checked.

---

## Q6. Operations: HA and readable secondaries, backups and point-in-time restore, branching, monitoring, storage and connection limits, compute sizing, maintenance, SLA

### Takeaway
- **HA:** 1 primary + 1–3 secondaries across availability zones, automatic failover, and an unchanged connection string. There is no scale-to-zero under HA. The SLA is 99.99% with zone-redundant HA and 99.9% for a single compute, since 2026-08-31.
- **Recovery:** point-in-time history of 2–30 days (default 7), snapshots, branching, and soft-delete with 7-day recovery.
- **Monitoring:** built-in dashboards, plus Beta telemetry in `system.lakebase`.
- **Maintenance:** automatic minor-version and patch updates with a brief restart.

### Cited Findings
**High availability** (from [High availability](https://learn.microsoft.com/en-us/azure/databricks/oltp/projects/high-availability))
- An HA endpoint has exactly one primary and one to three secondaries in different availability zones.
- On failover a secondary is promoted, "Failover preserves all committed transactions", and the primary host routes to the new primary. "Existing connections are terminated during failover and must reconnect."
- Readable secondaries are optional, through the `{endpoint-id}-ro…` host. With only one readable secondary, reads are fully interrupted during a failover.
- Limits: 2–4 compute instances, (max − min) ≤ 16 CU, and "Scale to zero is not available".
- "Lakebase storage is highly available regardless of your HA setting."
- SLA: "99.99% Monthly Uptime Percentage SLA for Lakebase Endpoints configured in zone-redundant high availability mode, and a 99.9% … for Single Compute Lakebase Endpoints." The release notes date this to 2026-08-31. — [High availability](https://learn.microsoft.com/en-us/azure/databricks/oltp/projects/high-availability); [Release notes](https://learn.microsoft.com/en-us/azure/databricks/release-notes/lakebase/)

**Recovery, branching and project limits** (from [Manage projects](https://learn.microsoft.com/en-us/azure/databricks/oltp/projects/manage-projects))
- History window: 2–30 days, default 7. It enables point-in-time restore, point-in-time queries and branching from past states. Extending it increases storage.
- Project limits:
  - 20 concurrently active computes per project (the default branch is exempt);
  - 6 read replicas per branch;
  - 500 branches per project; 500 roles per branch; 500 databases per branch;
  - 1,000 projects per workspace;
  - 1 protected branch; 3 root branches;
  - 10 manual snapshots;
  - maximum history retention 30 days.
- Storage quota: per branch, "Adjustable on request". When it is reached, write performance drops. Only table and index data counts, not PITR history.
- Soft delete: a deleted project is retained for 7 days and can be recovered (`undelete_project`). The project ID is reserved for 7 days.
- New-project defaults: `production` compute "8 - 16 CU", HA off, autoscaling on, scale-to-zero on at 24 hours. Separately, "Compute defaults" for new computes are "2 ↔ 4 CU".
  - These conflict with the bundles guide, which says the default production branch is "Created automatically with 1 CU primary compute". — [Manage Lakebase with bundles](https://learn.microsoft.com/azure/databricks/oltp/projects/manage-with-bundles)
- Upgraded Provisioned instances map capacity 1 to a 4–8 CU range. — [Upgrade to Autoscaling](https://learn.microsoft.com/azure/databricks/oltp/upgrade-to-autoscaling)
- Branching: a child branch "inherits all databases, roles, and data from its parent branch at the time of creation". Later parent changes don't propagate. The `production` root branch can't be deleted. — [Manage projects](https://learn.microsoft.com/en-us/azure/databricks/oltp/projects/manage-projects)
- Snapshots and backups, all from the [Release notes](https://learn.microsoft.com/en-us/azure/databricks/release-notes/lakebase/):
  - Snapshot storage has been billed since 2026-06-01.
  - Snapshots API (**Beta**, 2026-09-15): create, list or delete snapshots, and restore by creating a branch.
  - Backup schedule API (**Beta**, 2026-09-22): daily, weekly or monthly cadences with per-cadence retention.

**Compute sizing** (from [Manage computes](https://learn.microsoft.com/en-us/azure/databricks/oltp/projects/manage-computes) and [Autoscaling](https://learn.microsoft.com/azure/databricks/oltp/projects/autoscaling))
- 1 CU is about 2 GB RAM plus proportional CPU and local SSD.
- Autoscaling runs from 0.5 to 64 CU (64 since 2026-05-20). Fixed sizes go up to 112 CU.
- The autoscaling range must satisfy max − min ≤ 16 CU. Scaling within the range doesn't drop connections, but changing min or max "may cause a brief interruption".
- "Set your minimum compute size large enough to cache your working set in memory".

**Maintenance**
- Updates (Postgres minor versions, security patches, features) need "a brief compute restart that takes a few seconds" in a preferred window you can choose. — [Manage projects](https://learn.microsoft.com/en-us/azure/databricks/oltp/projects/manage-projects)
- Planned updates are posted at least 3 days in advance (changed from 7 on 2026-07-10). — [Release notes](https://learn.microsoft.com/en-us/azure/databricks/release-notes/lakebase/)
- Restarting a compute "interrupts any active connections". A disabled compute can't be woken by a connection; it must be re-enabled through the API. — [Manage computes](https://learn.microsoft.com/en-us/azure/databricks/oltp/projects/manage-computes)

**Monitoring**
- Built-in tools: a metrics dashboard (CPU, RAM, connections, database size), active queries, query performance, system operations, `pg_stat_statements`, and external pgAdmin or PgHero. Host agents are not supported. Password roles are recommended for monitoring tools. — [Observability](https://learn.microsoft.com/azure/databricks/oltp/projects/monitor); [External monitoring tools](https://learn.microsoft.com/azure/databricks/oltp/projects/external-monitoring-tools)
- `system.lakebase` tables are **Beta** with 7-day retention. They include `active_session_history`, `wait_event_counters`, `plan_history`, `pg_stat_statements_counters`, `ddl_history`, database and compute counters and gauges, and `postgres_logs`. — [Lakebase system table reference](https://learn.microsoft.com/azure/databricks/admin/system-tables/lakebase)
- AI Insights and Genie troubleshooting have been **Beta** since 2026-07-24. — [Release notes](https://learn.microsoft.com/en-us/azure/databricks/release-notes/lakebase/)

### Inferences
- **Branching for dev/test:** a `dev` branch per developer, or for the Posit Workbench environment, gives a zero-copy clone of production data. With a short scale-to-zero timeout it costs almost nothing when idle. It also lets the schema-setup and migration notebooks be tested against a branch before they run on `production`. Note that HCP data copied into branches inherits production's sensitivity.
- **Sizing for MOSAIC (~3 GB):**
  - A working set of ~3 GB fits in memory at about 2 CU (4 GB).
  - 0.5–1 CU (1–2 GB) will page from storage but may be fine at low traffic.
  - The documented new-project default of 8–16 CU is far more than this workload needs and drives cost (see Q7). Check the actual setting with `get_endpoint` (`autoscaling_limit_min_cu`/`max_cu`).
- **HA trade-off:** HA would remove scale-to-zero and multiply compute cost by the number of instances. A single compute with 99.9% SLA, PITR and snapshots may be enough for low-traffic internal apps. That is a business decision.

### Gaps
- No documented recovery-time objective for failover, and no restore time for PITR.
- No documented cold-start time other than "a few hundred milliseconds" for scale-to-zero resume.
- The per-branch storage quota default value is not published ("Adjustable on request").

---

## Q7. Cost: pricing model and conceptual comparison with Azure SQL Database for a ~3 GB, low-traffic OLTP app

### Takeaway
- Lakebase Autoscaling bills **compute per CU-hour**, under the Database Serverless SKU at 0.213 DBU per CU-hour, only while the compute is active.
- **Storage is billed per GB-month** under the Databricks Storage SKU. The multipliers are 15 DSU for database storage, 8.7 for PITR history and 3.91 for snapshots.
- The indicative US list prices were **$0.111/CU-hour, $0.35/GB-month storage, $0.20/GB-month PITR and $0.09/GB-month snapshots**, labeled "indicative" for AWS US East, Enterprise tier.
- For ~3 GB, storage is negligible, and **compute minimum CU × hours active** dominates. Scale-to-zero, a small minimum CU and no HA make Lakebase cheap. Leaving the documented 8–16 CU default always-on does not.
- Azure dollar rates were not retrievable; the official pricing page renders prices dynamically.

### Cited Findings
- Azure Databricks SKUs and multipliers:
  - Database Serverless Compute SKU: "Lakebase Autoscaling Compute | 0.213X" DBU multiplier.
  - Databricks Storage SKU: "Database Storage | 15X", "Point in time restore (PITR) | 8.7X" and "Snapshots | 3.91X" DSU multipliers.
  - Source: [Serverless DBU consumption by SKU (Azure Databricks)](https://learn.microsoft.com/azure/databricks/resources/pricing)
- Indicative prices: "Compute: $0.111 per CU-hour; Database storage: $0.35 per GB-month; Instant restore (PITR) storage: $0.20 per GB-month; Branch snapshot storage: $0.09 per GB-month". "Billing for Lakebase Autoscaling usage begins in January 2026". "Prices … indicative of the Lakebase Autoscaling Enterprise Tier in AWS US East (N. Virginia)".
  - This text is from the search-indexed version of [docs.databricks.com/aws/en/oltp/projects/pricing](https://docs.databricks.com/aws/en/oltp/projects/pricing), which now returns 301 to [databricks.com/product/pricing/lakebase](https://www.databricks.com/product/pricing/lakebase). That page loads prices dynamically, and none could be extracted on 2026-10-06.
- The Databricks pricing page notes that "Premium tier on Azure Databricks corresponds to the Enterprise tier on AWS and GCP" and that Azure Databricks has unified Azure billing. — [Lakebase pricing](https://www.databricks.com/product/pricing/lakebase)
- After the upgrade, "Lakebase GA pricing applies … most customers see a reduction in compute costs. Lakebase Autoscaling also includes always-on compute pricing." — [Upgrade to Autoscaling](https://learn.microsoft.com/azure/databricks/oltp/upgrade-to-autoscaling)
- Usage appears in `system.billing.usage` with `billing_origin_product = 'LAKEBASE'`. Projects support serverless usage (budget) policies and custom tags for attribution (2026-03-12). — [Billable usage system table](https://learn.microsoft.com/azure/databricks/admin/system-tables/billing); [Manage projects](https://learn.microsoft.com/en-us/azure/databricks/oltp/projects/manage-projects)
- Scale-to-zero: "You pay only for active compute time, not for idle periods". "A development database used 8 hours per day costs one-third as much as an always-active compute." — [Scale to zero](https://learn.microsoft.com/en-us/azure/databricks/oltp/projects/scale-to-zero)
- Azure SQL Database serverless, for comparison:
  - Compute is billed per second as `max(min vCores, vCores used, min memory GB × 1/3, memory used × 1/3)`; only storage is billed while paused.
  - General Purpose auto-pause delay: minimum 15 minutes, default 60 minutes. Default minimum is 0.5 vCores.
  - Worked example: "Suppose the compute unit price is $0.000145/vCore/second". Business Critical costs about 2.7× General Purpose.
  - Source: [Serverless compute tier billing](https://learn.microsoft.com/azure/azure-sql/database/serverless-tier-billing?view=azuresql); [Serverless compute tier overview](https://learn.microsoft.com/azure/azure-sql/database/serverless-tier-overview?view=azuresql); [vCore purchasing model](https://learn.microsoft.com/azure/azure-sql/database/service-tiers-sql-database-vcore?view=azuresql)

### Inferences
- **Multipliers and prices agree:**
  - 0.213 DBU × about $0.52/DBU ≈ $0.111/CU-hour.
  - 15 / 8.7 / 3.91 DSU × about $0.023/DSU ≈ $0.35 / $0.20 / $0.09 per GB-month.
  - So the Azure multipliers and the US-indicative dollar prices describe the same rate card. Actual Azure (Premium) East US 2 rates and any negotiated discounts must be confirmed in the Azure pricing calculator or with the account team.
- **Rough monthly compute cost** at the indicative $0.111/CU-hour, always on (730 h):

| Compute | Approx. per month |
| --- | --- |
| 0.5 CU | ~$41 |
| 1 CU | ~$81 |
| 2 CU | ~$162 |
| 8 CU (the documented new-project minimum) | ~$648 |
| 8 CU, 2-instance HA | ~$1,296 |

- Storage for 3 GB is about $1/month, plus PITR history proportional to write churn over the 7-day default window.
- With scale-to-zero (for example a 5–15 minute timeout), a business-hours-only app could pay a fraction of these figures. Scale-to-zero is unavailable with HA.
- **Comparison with Azure SQL serverless General Purpose:**
  - At Microsoft's example price ($0.000145/vCore-second ≈ $0.52/vCore-hour), an always-online database at the 0.5-vCore minimum (~2.1 GB minimum memory, billed as 0.7 vCore) costs about $267/month in compute.
  - With auto-pause (minimum 15 minutes), costs fall in a similar way to Lakebase scale-to-zero (minimum 60 seconds).
  - Units differ: 1 Lakebase CU ≈ 2 GB RAM, while Azure SQL serverless normalizes 3 GB per vCore.
  - DTU tiers (Basic or Standard) can be cheaper for tiny databases, but no prices for them were retrieved.
  - **Net:** at ~3 GB and low traffic, Lakebase at 0.5–2 CU with scale-to-zero is likely comparable to or cheaper than Azure SQL serverless GP. That depends on (a) right-sizing away from the 8–16 CU default and (b) not enabling HA. The current Azure SQL tier and bill are unknown.

### Gaps
- Official Azure dollar rates for the Database Serverless DBU and DSU in East US 2 (Premium) were not retrievable from a primary source on 2026-10-06.
- No data was found on Lakebase egress or networking charges for external clients. The `NETWORKING` billing product mentions "egress back to clients for performance-intensive products", but no rates were found ([Billable usage system table](https://learn.microsoft.com/azure/databricks/admin/system-tables/billing)).
- Service Direct Private Link costs were not found.

---

## Q8. Security and compliance: encryption, audit logging, compliance security profile / HIPAA eligibility and certifications relevant to pharma commercial data

### Takeaway
- Lakebase encrypts data at rest and in transit, with TLS required, and supports customer-managed keys for **new** projects.
- It is supported under the **compliance security profile**: HIPAA, PCI-DSS, HITRUST, C5, TISAX, ISMAP or None. IRAP and UK Cyber Essentials Plus are supported in specific regions. It has been GA with the compliance security profile since 2026-03-04 and on by default in those workspaces since 2026-08-05. It is **SOC 2 Type 2** compliant (2026-07-14).
- With the HIPAA standard, Lakebase adds **pgaudit**-based SQL audit logging to `system.access.audit`, with 365-day retention.

### Cited Findings
- Encryption and HIPAA:
  - Under HIPAA, Databricks provides "Encryption of data at rest and in transit" and "Audit logging for HIPAA-enabled projects".
  - The customer must keep PHI out of object names and query text, use least-privilege roles and manage audit retention.
  - A BAA is required for PHI.
  - Source: [HIPAA compliance for Lakebase](https://learn.microsoft.com/en-us/azure/databricks/oltp/projects/hipaa-compliance)
- Customer-managed keys encrypt project data at rest with a customer KMS key. They arrived on 2026-04-14 and reached all AWS and Azure Lakebase regions on 2026-05-20, and are "available for new Lakebase projects". — [Release notes](https://learn.microsoft.com/en-us/azure/databricks/release-notes/lakebase/)
- Data protection features: CMK, protected branches and Private Link. — [Data protection](https://learn.microsoft.com/en-us/azure/databricks/oltp/projects/data-protection)
- Compliance standards: "Lakebase is enabled by default for workspaces with the compliance security profile when the compliance standard is set to HIPAA, PCI-DSS, HITRUST, C5, TISAX, ISMAP, or None". Region-specific: IRAP (`australiaeast`) and UK Cyber Essentials Plus (`uksouth`, `ukwest`). "Other compliance standards are not supported." SOC 2 Type 2 reports are available from the account team. — [Data protection](https://learn.microsoft.com/en-us/azure/databricks/oltp/projects/data-protection)
  - The HIPAA page lists only "HIPAA, C5, TISAX, or None", which looks like an older page. — [HIPAA compliance for Lakebase](https://learn.microsoft.com/en-us/azure/databricks/oltp/projects/hipaa-compliance)
- Dated compliance milestones, from the [Release notes](https://learn.microsoft.com/en-us/azure/databricks/release-notes/lakebase/):
  - 2026-03-04: GA with the compliance security profile (HIPAA, C5, TISAX, None).
  - 2026-07-14: SOC 2 Type 2.
  - 2026-08-05: enabled by default in compliance security profile workspaces.
  - 2026-08-10: PCI-DSS and HITRUST ("On Azure, all regions where Lakebase is available are supported").
  - 2026-08-21: ISMAP, IRAP and UK Cyber Essentials Plus on Azure.
- HIPAA audit logging:
  - Events are console, API and Postgres (pgaudit) events in `system.access.audit` with `service_name = 'lakebase'`.
  - The fixed configuration is `pgaudit.log = all, -misc`, `log_parameter = off`, `log_catalog = off`.
  - `action_name` takes forms like `ddl.create_table`. `request_params` includes `statement`, `objectName` and `endpointId`.
  - Retention is 365 days. "HIPAA is the only compliance standard that enables extended audit logging for Lakebase."
  - Source: [HIPAA audit logging](https://learn.microsoft.com/azure/databricks/oltp/projects/hipaa-audit-logging); [HIPAA compliance for Lakebase](https://learn.microsoft.com/en-us/azure/databricks/oltp/projects/hipaa-compliance)
- Credential security: TLS is required for all connections. Password roles are stored as SCRAM-SHA-256 verifiers, with KMS-encrypted control-plane copies for generated passwords. OAuth tokens are workspace-scoped and can reach any database the identity has rights on. — [About authentication](https://learn.microsoft.com/en-us/azure/databricks/oltp/projects/authentication); [Create Postgres roles](https://learn.microsoft.com/en-us/azure/databricks/oltp/projects/postgres-roles); [Connect external app using API](https://learn.microsoft.com/en-us/azure/databricks/oltp/projects/external-apps-manual-api)
- Governance: Unity Catalog registration adds UC permissions, lineage and audit for reads made through SQL warehouses. — [Register in Unity Catalog](https://learn.microsoft.com/en-us/azure/databricks/oltp/projects/register-uc)

### Inferences
- **Data classification:** HCP attributes and sales-call details concern prescribers, not patients, so they are usually personal or commercial-confidential data rather than PHI. Whether the workspace needs the HIPAA profile is a compliance-team call. This is an inference.
- **Audit coverage depends on the HIPAA profile:** pgaudit audit logs are documented only for HIPAA-profile workspaces. Without HIPAA, SQL-level audit depends on the app's own audit tables (as in `mosaic_redesign.mosaic_audit_log`), Lakebase CDF history tables, and the Beta `system.lakebase` telemetry (7-day retention).
- **CMK must be chosen at creation:** CMK applies only to **new** projects. If CMK is required, the target project must be created with it, possibly a fresh project instead of the one the notebooks currently point at.

### Gaps
- Not verified: whether Lakebase on Azure is covered by specific GxP or 21 CFR Part 11 attestations relevant to pharma. No such statement was found; most commercial-operations data is outside GxP.
- No Lakebase-specific data-residency or backup-geo-replication statement was found.

---

## Q9. Cross-check: where do the notebooks' assumptions match or conflict with current platform behavior?

### Takeaway
- **Matches:** the notebooks' connection pattern matches the current GA Lakebase Postgres API and documented examples almost line for line: `list_endpoints` → `status.hosts.host` → `generate_database_credential` → psycopg with `sslmode=require` on `databricks_postgres`, with 1-hour tokens. East US 2 is a GA region.
- **Main conflicts or risks:**
  1. `endpoints[0]` could pick a read replica.
  2. Using a human identity and ownership for app objects needs explicit grants or an ownership plan.
  3. The JDBC source uses the **deprecated** `ActiveDirectoryPassword` Entra mode.
  4. App-side pooling with OAuth (PgBouncer requires password roles, which are off by default for new projects).
  5. Private Link and scale-to-zero behavior for external Posit clients.

### Cited Findings
**Matches**
- **API calls:** `w.postgres.list_endpoints(parent="projects/<p>/branches/<b>")`, `endpoint.status.hosts.host` and `w.postgres.generate_database_credential(endpoint=...)` are the documented Python SDK calls. — [Manage computes](https://learn.microsoft.com/en-us/azure/databricks/oltp/projects/manage-computes); [About authentication](https://learn.microsoft.com/en-us/azure/databricks/oltp/projects/authentication)
- **API status:** the Postgres APIs became GA on 2026-08-14. — [Release notes](https://learn.microsoft.com/en-us/azure/databricks/release-notes/lakebase/)
- **SDK floor:** `w.postgres` appeared in v0.75.0 and `generate_database_credential` in v0.79.0. The docs ask for 0.81.0 or 0.89.0, and the notebooks' floor is 0.118.0 (2026-06-18). — [databricks-sdk-py CHANGELOG](https://github.com/databricks/databricks-sdk-py/blob/main/CHANGELOG.md); [Connect external app using SDK](https://learn.microsoft.com/azure/databricks/oltp/projects/external-apps-connect)
- **Defaults and TLS:** `databricks_postgres` is the default database and `production` the default root branch. TLS is required. — [Manage projects](https://learn.microsoft.com/en-us/azure/databricks/oltp/projects/manage-projects); [About authentication](https://learn.microsoft.com/en-us/azure/databricks/oltp/projects/authentication)
- **User name:** for users, the Postgres user is the Databricks email; the docs' example uses `user="your.email@company.com"`. — [About authentication](https://learn.microsoft.com/en-us/azure/databricks/oltp/projects/authentication)
- **Host pattern:** the `*.database.eastus2.azuredatabricks.net` host shown in the migration notebook is Lakebase's regional Postgres ingress pattern. — [Private Link for Lakebase](https://learn.microsoft.com/azure/databricks/oltp/projects/private-link)
- **Schema-setup DDL** is valid on Postgres 16, 17 and 18: `gen_random_uuid()` is built in (no `uuid-ossp` needed). — [Postgres extensions](https://learn.microsoft.com/azure/databricks/oltp/projects/extensions)

**Conflicts or risks**
- **Read replicas in the endpoint list:** a branch can have "1 endpoint: Primary read-write compute only" or "2+ endpoints: Primary compute plus one or more read replicas". Endpoints carry `status.endpoint_type` (READ_WRITE or READ_ONLY). — [Manage computes](https://learn.microsoft.com/en-us/azure/databricks/oltp/projects/manage-computes); [CLI guide](https://learn.microsoft.com/azure/databricks/oltp/projects/cli)
- **Ownership:** objects are owned by the creating identity's role. New roles have no privileges. Ownership transfer needs a temporary-role workaround. — [Manage database permissions](https://learn.microsoft.com/en-us/azure/databricks/oltp/projects/manage-roles-permissions); [Transfer Postgres object ownership](https://learn.microsoft.com/en-us/azure/databricks/oltp/projects/transfer-object-ownership)
- **Deprecated source auth:**
  - "ActiveDirectoryPassword is deprecated". It "is incompatible with mandatory Microsoft Entra multifactor authentication (MFA) and might not work in tenants where MFA is enforced".
  - Microsoft's guidance: "Don't use a Microsoft Entra user account as a service account"; use ActiveDirectoryServicePrincipal or MSI.
  - Microsoft JDBC Driver 13.2 added a deprecation warning.
  - Source: [JDBC: Connect using Microsoft Entra authentication](https://learn.microsoft.com/sql/connect/jdbc/connecting-using-azure-active-directory-authentication?view=sql-server-ver17); [JDBC driver release notes](https://learn.microsoft.com/sql/connect/jdbc/release-notes-for-the-jdbc-driver?view=sql-server-ver17)
- **Pooling for apps:** the PgBouncer pooler requires password roles, and new projects have password connections disabled by default (2026-05-21). — [Use connection pooling](https://learn.microsoft.com/en-us/azure/databricks/oltp/projects/connection-pooling); [Release notes](https://learn.microsoft.com/en-us/azure/databricks/release-notes/lakebase/)
- **Connections are dropped by the platform:** scale-to-zero closes idle connections (default 24 hours on `production`). Restarts, updates and failover terminate connections. Connections have a 3-day maximum life. — [Scale to zero](https://learn.microsoft.com/en-us/azure/databricks/oltp/projects/scale-to-zero); [Manage projects](https://learn.microsoft.com/en-us/azure/databricks/oltp/projects/manage-projects); [About authentication](https://learn.microsoft.com/en-us/azure/databricks/oltp/projects/authentication)
- **psycopg error handling:** by default, psycopg 3 opens a transaction implicitly. "If a database operation fails, the server will refuse further commands, until a rollback() is called" (`InFailedSqlTransaction`). `autocommit=True` or `with conn.transaction()` changes this. — [psycopg 3: Transactions management](https://www.psycopg.org/psycopg3/docs/basic/transactions.html)

### Inferences
1. **`endpoints[0]` (both notebooks).** This is safe only while the branch has a single compute. Once a read replica is added, the list may return a read-only endpoint first. Filter on `ep.status.endpoint_type == ENDPOINT_TYPE_READ_WRITE`, or address the primary endpoint by its full resource name. HA secondaries live inside the same endpoint (`-ro` host), so HA by itself does not add list entries.
2. **Identity (both notebooks).** The notebooks connect as the human running them. That works only if that user has an OAuth Postgres role; only the project owner gets one automatically. It also means:
   - the person who ran the notebook owns the 8 `mosaic_redesign` tables and all migrated tables;
   - the Posit app's service principal or group role needs `USAGE` on each schema plus DML grants and default privileges.
   - **Recommended:** run scheduled syncs as a service principal (M2M), and have a group role own the app schemas.
   - Whether `w.current_user.me().user_name` returns a service principal's application ID (needed if the notebook runs as a service principal) was not verified.
3. **Token refresh (migration notebook).** The refresh at 45 minutes is harmless but not required for an open connection, because expiry is enforced at login only. Add reconnect/retry for platform-initiated disconnects, which the current loop does not handle mid-table.
4. **Reload modes (migration notebook).** `recreate` (DROP + CREATE) drops app grants and requires ownership. `truncate` keeps them. The first markdown cell still describes the notebook as "DROP + CREATE", which is inconsistent with the registry's `truncate` default. That is internal documentation drift, not a platform issue.
5. **Source authentication (migration notebook).** `Authentication=ActiveDirectoryPassword`, with an Entra user and a secret-scope password, is deprecated and may fail under enforced MFA. Alternatives:
   - an Entra service principal (`ActiveDirectoryServicePrincipal`);
   - a UC SQL Server connection (Lakehouse Federation or Lakeflow Connect), with the caveat that the authentication-method docs conflict (see Q5).
6. **Schema-setup error handling.** The per-statement try/except does not isolate failures. After the first failing DDL, every later statement in that cell fails with `InFailedSqlTransaction`, and the final `commit()` cannot keep the work done in the aborted transaction. This follows Postgres semantics. Use `autocommit=True` for idempotent DDL, or a `conn.transaction()` savepoint per statement.
7. **Pooling and sizing for the apps (outside the notebooks).** Posit apps cannot use the Lakebase pooler with OAuth. Either mint a token per connection in the app pool, or enable password connections and use `-pooler` hosts. On small computes, keep the 105 (0.5 CU) to 443 (2 CU) connection ceilings in mind.
8. **Data types.** The notebook maps SQL Server `datetime`/`datetime2` → `timestamp` (without time zone). If some tables later move to synced tables, their Delta `TIMESTAMP` columns arrive as `timestamptz`. App code should tolerate the change or use type overrides where available. Only `varchar`/`vector`/`halfvec` overrides are documented, so timestamp types cannot currently be overridden.
9. **Scope of what gets copied.** The migration registry includes a view (`vw_*`) and backup tables (`*_BAK*`). JDBC will copy the view as a static table, and backups will be copied as live tables. Whether these belong in Lakebase is an app decision, not a platform one.
10. **Networking for Posit.** If the workspace uses front-end Private Link, Posit Workbench and Connect servers reaching the regional host from outside the workspace need the Public Preview Service Direct Private Link endpoint. The notebooks themselves run inside Databricks, but the docs do not explicitly say notebooks on serverless count as "within the workspace" for this purpose.

### Gaps
- The notebooks' actual project settings (Postgres version, CU range, scale-to-zero, HA, whether password connections are enabled, and creation date relative to 2026-05-21) cannot be determined from the notebook files.
- Whether the workspace uses the compliance security profile or Private Link is unknown.
- The migration notebook's skill file, referenced in its markdown, was not available to this research.

# multi-platform-orchestrator-iac

Terraform for the [multi-platform orchestrator](https://github.com/bendfeldt/multi-platform-orchestrator):
Azure infrastructure, identities and platform grants. The solution repo deploys onto it and
reads only the environment contract this repo publishes (no shared code, no shared state).

Entra ID everywhere, no secrets: every pipeline and workload logs in with a federated
credential or a managed identity.

## Layout

| Path | What | Run by |
|---|---|---|
| `bootstrap/` | Shared state and contract storage, the shared Front Door Premium profile, per environment: resource group, image policy, and the two federated deployers | A person, once |
| `environment/` | One environment (dev, test or prod): network, Container Apps environment, Postgres, ACR, logs, identities, sign-in apps, Front Door endpoints and WAF, image policy, domains, contract | The pipelines |
| `modules/domain_access/` | One domain: its managed identity and its grants on Databricks, Fabric and Snowflake | via `environment/` |
| `domains/<env>/<name>.yaml` | What each domain may read and write. File name = the code location's name | People, by pull request |
| `alerts/<env>.yaml` | Who is notified about which Dagster alerts, and how | People, by pull request |
| `snowflake/bootstrap.sql` | The infra pipeline's Snowflake login (workload identity) | A Snowflake admin, once per account |
| `snowflake/cost_access.sql` | Lets mpo_ops read Snowflake's cost attribution | A Snowflake admin, once per environment |
| `fabric/capacity_operations.kql` | The function the cost report reads Fabric capacity events through | A KQL database admin, once |
| `scripts/` | `ci.sh`, `apply.sh`, `install-tools.sh`, called by both CI systems; `check-kql.sh` parses the alert queries | |

## Set up (once)

1. **Bootstrap** (Owner on the subscriptions; Entra rights to create apps and grant the Graph
   app role `Application.ReadWrite.OwnedBy`):
   ```bash
   cd bootstrap
   cp bootstrap.tfvars.example bootstrap.tfvars   # tenant, subscriptions, GitHub and/or ADO
   terraform init && terraform apply -var-file=bootstrap.tfvars
   # then switch versions.tf to `backend "azurerm"` and:
   terraform init -migrate-state -backend-config=backend.hcl
   ```
   One subscription per environment or one shared subscription both work: give the same ID
   three times. `location` is optional (default `westeurope`).
2. **Platforms**, once per environment, by their admins:
   - Snowflake: run `snowflake/bootstrap.sql` with the infra deployer's object ID.
   - Databricks: add the infra deployer's service principal to the workspace as admin; catalog
     owners give it `MANAGE` on the catalogs domains use. Job owners give each domain's service
     principal `CAN_MANAGE_RUN` on its jobs (Terraform doesn't, because
     `databricks_permissions` would replace each job's whole permission list).
   - Fabric: allow service principals to use Fabric APIs for a security group containing the
     deployers and domain identities; make the infra deployer Admin of the domains' workspaces.
   - Cost report (`mpo_ops`), after the first apply: a Snowflake ACCOUNTADMIN runs
     `snowflake/cost_access.sql`; a Databricks metastore admin runs, with mpo_ops's client ID
     (contract `identities.domains.mpo_ops.client_id`):
     ```sql
     GRANT USE CATALOG ON CATALOG system TO `<client-id>`;
     GRANT USE SCHEMA, SELECT ON SCHEMA system.billing TO `<client-id>`;
     ```
     Both read cost metadata only. Set `snowflake.credit_price_usd` in `envs/<env>.tfvars` for
     cost in USD. `databricks_cost_warehouse_id` reuses an existing SQL warehouse (its owner
     gives mpo_ops `CAN_USE`); otherwise Terraform creates a 2X-Small serverless one that stops
     10 minutes after the daily query.
   - Fabric cost (optional): a capacity admin creates an Eventhouse in a workspace of its own
     and an eventstream from Real-Time hub "Fabric capacity operation events" into table
     `FabricCapacityOperations` (not in Terraform); run `fabric/capacity_operations.kql` in
     that KQL database; set `fabric_capacity_events` (Query URI, database, price per CU hour)
     in `envs/<env>.tfvars`; uncomment the Viewer grant in `domains/<env>/mpo_ops.yaml`. The
     grant is on the whole workspace, so keep only the monitoring Eventhouse there.
   - Purview (optional): set `purview_endpoint` (the tenant's existing account) in
     `envs/<env>.tfvars`. In the Purview portal, a Purview admin gives mpo_ops's identity
     Data Curator on a collection and runs the solution's `orchestrator_core.purview typedefs`
     once.
3. **Environment values**: copy the bootstrap outputs into `environment/envs/<env>.tfvars`
   (and the dev file's `acr_pull_principal_ids` with the test and prod app deployers).
4. **CI**: environments `dev`, `test`, `prod` (GitHub) or `mpo-infra-<env>` (ADO) with approvals
   on test and prod. Solution repo, for pull-request previews in dev: GitHub environments
   `preview` (required reviewers) and `preview-cleanup` (deployment branch `main` only), both
   with dev's app-deployer variables; GitHub variables `AZURE_CLIENT_ID`, `AZURE_TENANT_ID`,
   `AZURE_SUBSCRIPTION_ID`; ADO service connections `mpo-infra-<env>` with workload identity
   federation.
5. The solution repo needs each environment's contract URL (`bootstrap` output
   `shared.contract_urls`) and app deployer client ID.

## Day to day

- Pull requests run `scripts/ci.sh`: format, validate, `terraform test` (mocked providers,
  including the grant tests in `modules/domain_access/tests`), the alert, workbook and
  Fabric queries through Microsoft's Kusto parser, and a Trivy scan. No cloud login.
- Merging to `main` applies dev, then test and prod after approval.
- **Add a domain**: add `domains/<env>/<name>.yaml` for each environment. Terraform creates
  its identity and exactly the listed grants, adds it to the contract, and creates the
  domain's Dagster roles (`Dagster.Launcher.<name>`, `Dagster.Editor.<name>`) for the groups
  in `launchers:` / `editors:`.
- **Who may do what in Dagster**: groups in `viewer_group_ids` / `admin_group_ids`
  (`envs/<env>.tfvars`) and the domain files. Group-to-role assignment needs Entra ID P1 or
  P2.
- **Alerts**: `alerts/<env>.yaml` names action groups (email, SMS, voice, Azure app push, Azure
  role, Entra-secured webhook, Event Hub, ITSM, and webhook, Logic App, Function and runbook)
  and routes alerts to them by code location, kind and severity. One log alert rule per route
  and severity reads the solution's alert lines from Log Analytics as a dedicated identity with
  Log Analytics Reader; another fires when the 5-minute health check has not reported for 15
  minutes. The URL of a webhook, Logic App, Function or runbook receiver holds a key, so the file
  only names a Key Vault secret; its owner stores the URL there and gives the infra deployer
  `Key Vault Secrets User` on it. Prefer `secure_webhook` (Entra) where the target supports it.
- **Cost and insights**: `mpo_ops` sends daily rows to `MpoCost_CL` and `MpoInsights_CL`
  (columns in `environment/insights/tables.json`) through a data collection rule it alone may
  use; the workbook "Dagster cost and reliability" shows them (queries in
  `environment/insights/*.kql`).

## Not yet verified against Azure (Phase 0 spikes)

- Front Door Private Link approval on the Container Apps environment (`frontdoor.tf`).
- The Azure Policy aliases for Container Apps images (`bootstrap/environment/main.tf`).
- The Snowflake provider's workload identity login from a CI runner.
- Container Apps sign-in without a client secret (implicit ID tokens in `entra.tf`).
- Whether an Event Hub receiver needs a role on the hub (Microsoft documents only the preview
  managed-identity variant, which the azurerm provider does not expose yet).
- The alert lines arriving unchanged in `Log_s` (the KQL is parsed, not run, in CI).
- Whether the infra role's `MANAGE GRANTS` may grant `SNOWFLAKE.USAGE_VIEWER` (then
  `cost_access.sql` moves into Terraform).
- The workbook JSON rendering as intended (only its queries are checked in CI).
- Querying the Eventhouse as mpo_ops's managed identity with workspace Viewer, and the
  eventstream's table matching `fabric/capacity_operations.kql` (CI parses it against the
  documented event shape).

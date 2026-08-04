# Standing Up a Separate ANA Instance

## Status after the 2026-08-04 meeting

The 2026-08-04 discussion separated two concerns that had been tangled together: **where the software runs**
(hosting) and **who issues the tokens** (identity). They are independent axes and should be decided
independently.

**Settled at the 2026-07-31 meeting, still holding:**
- The driver is **isolation**. A new instance is fully separated, in its own Azure subscription.
- The goal is **minimum work to launch**.
- **Multi-tenancy (one deployment serving many clients) is off the table.** Shared Search/SQL/OpenAI means a
  shared blast radius, the opposite of what was asked.

**New direction from 2026-08-04:**
- **Hosting** — migrate compute to **AKS**, with three placement options for compute vs. data. See [§3](#3-hosting-model--where-compute-and-data-live)
  and [§4](#4-migrating-compute-to-aks).
- **Identity** — support **Azure Entra ID only**. Customers federate Okta or any other IdP into their own
  Entra tenant; ANA integrates with Entra and nothing else. See [§5](#5-identity--entra-id-only).

**Verdict up front:** the identity decision is sound and cheap — it is the correct scope boundary and costs
little to implement. The hosting direction is viable but only in the order 1 → 2 → 3, and AKS is not what
makes options 2 and 3 possible. The blocker for customer-tenant hosting is that ~80% of the platform is
managed Azure services (OpenAI, Cognitive Search, SQL, SignalR, Storage) that AKS does not touch, plus a
configuration layer that currently hardcodes endpoints, keys, and a tenant ID. See [§4](#4-migrating-compute-to-aks)
for what AKS does and does not buy.

**Still open:**
- Which hosting option is the launch target (§3).
- Whether to commit to full IaC now — option 3 makes this mandatory rather than optional (§8).

---

## 1. What "an instance" consists of

One environment (QA or Prod) spans roughly:

| Layer | Resources | Containerizable? |
|---|---|---|
| Compute | App Service (API), App Service (primary FE), App Service (voice BE), Static Web App (voice FE), 5 Function Apps (Confluence/Slack/Mail/TextMerge/ProcessContent) + plans | **Yes** |
| Data | SQL Database, Azure Blob Storage (content), Azure Table Storage (sync state), Azure Cognitive Search (index + skillset + indexers) | No — managed services |
| AI | Azure OpenAI account + deployments (chat, embeddings, realtime/transcribe) | No — managed, quota-bound |
| Platform | Azure SignalR, Application Insights ×N, Log Analytics workspace | No — managed |
| Network/DNS | custom domains + certs for API and FEs (`*.altusnova.ai` / `*.engineering.altusnova.com`) | N/A |
| Identity | 6 Entra app registrations (see [entra-app-registrations.md](entra-app-registrations.md)) | N/A |
| Pipelines | 8 Azure DevOps pipelines + variable groups + release definitions | N/A |

The "Containerizable?" column is the single most important thing to absorb before discussing AKS: **only the
first row moves.** Everything expensive, stateful, or quota-bound is a managed Azure service and stays one
under every option below.

Resource names are largely env-suffixed already (`-qa` / `-Prod`), so a third suffix mostly works — but
several are global-unique (storage, Search, OpenAI, SWA) and a few are hardcoded (§7).

---

## 2. What QA and Prod share today

This is what "fully separated" actually costs. Every row marked *Shared* has to become a dedicated resource —
and several are pinned in the **base** config file rather than a per-environment one, so a third environment
silently inherits QA's values unless explicitly overridden.

| Resource | Shared? | Evidence |
|---|---|---|
| **Azure subscription** | **Shared** — `cd249234-d6b9-4108-b17c-96ed8ab0ad90`, hardcoded in both the QA and Prod stages | `altus-ai-assistant-voice/azure-pipelines-infra.yml` |
| **Azure OpenAI account** | **Shared** — `aitests1`, one `gpt-5.2` deployment | `appsettings.json` → `Settings:GptDeployments`; voice `infra/main.bicep` |
| **Azure Cognitive Search** | **Shared** — `searchaitests2`, and hardcoded a second time in code | `appsettings.json`; `IndexingService.cs:295` |
| **Application Insights** | **Shared connection string in base config** (`268188ee-…`) | `appsettings.json` → `ApplicationInsights:ConnectionString` |
| **Log Analytics workspace** | **Shared** — ProcessContent's Bicep takes it as an *existing* resource in another RG | `AltusAiProcessContent/infra/main.bicep` |
| **Entra tenant** | **Shared** — `e99365e9-…`, hardcoded in three places in the base config | `appsettings.json:42,47,54` |
| **Entra app registrations 1–5** | **Shared** — API resource server, FE SPA, MCP client, Graph service identity, mail daemon | [entra-app-registrations.md](entra-app-registrations.md) §1–5 |
| **Voice API registration (#6)** | **Not shared** — per-environment by design | [voice-backend-jwt-auth.md](voice-backend-jwt-auth.md) |
| Blob/content storage | Not shared — `storageaiassistantqa` vs `altusaiassistant` | `appsettings.QA.json` / `appsettings.Prod.json` |
| SQL database | Not shared — connection string per environment | app settings (`env-var`) |
| App Services / Function Apps / SWA | Not shared | env-suffixed names |
| Custom domains | Not shared | `*.qa.altusnova.ai` vs `*.altusnova.ai` |

Three consequences worth stating plainly:

- **The base `appsettings.json` is effectively the QA config.** `appsettings.Prod.json` overrides only
  `BlobAccountName`, `Cors`, and `McpSettings:ResourceUri` — everything else in Prod comes from the base file
  or from App Service settings. **Confirm in the portal which base values Prod actually overrides via app
  settings** before assuming a third env file is sufficient.
- **Search index names carry no environment marker** — `IndexingService.GetIndexName()` returns
  `{clientUniqueId}-index`. QA and Prod coexist on one Search service only because their client GUIDs differ.
  Do not plan to share a Search service.
- **All Azure data-plane access is key-based.** Eight `AzureKeyCredential` constructions, a blob connection
  string handed to the Search indexer data source, and raw SQL connection strings. No managed identity
  anywhere. This matters enormously in §3 — see the cross-tenant auth discussion there.

### Also tenant/org-bound, not transferable

Confluence, Slack, and Grain integration credentials, and any Exchange mail source. Each needs its own setup
per instance regardless of the hosting or identity decision.

---

## 3. Hosting model — where compute and data live

Three options were put forward. They form a maturity ladder, not a menu — each is a superset of the previous
one's requirements, and the same build artifacts serve all three **if and only if** the configuration layer is
fully parameterized (§7). Treat 1 → 2 → 3 as sequencing, not as alternatives to choose between now.

### First: what actually counts as "data at rest"

Option 2's entire value proposition is "customer content stays under customer control at rest." That promise
is only meaningful if it survives a security review, so the list of places customer content physically lands
today has to be exact:

| Where | What lands there | Movable to customer tenant? |
|---|---|---|
| Blob Storage | Source documents, verbatim | **Yes** — clean boundary |
| SQL Database | Chat history, contacts, prompts, taxonomy, config | **Yes** — clean boundary |
| **Cognitive Search index** | **Chunked document text + 1536-dim embeddings** | Yes, but see below |
| Azure Table Storage | Sync state and metadata | Yes |
| Azure OpenAI | Prompts + retrieved chunks in transit; Azure abuse-monitoring retains up to 30 days unless the subscription is granted an exemption | Only by moving the OpenAI account |
| Azure SignalR | Streamed response text transits the service | Only by moving/self-hosting SignalR |
| App Insights / Log Analytics | **Unverified** — depends on what the API logs. Must be audited; prompt or content fragments in telemetry defeat the whole exercise | Yes, once audited |

**The Search index is the crux.** It holds a chunked, searchable copy of every indexed document plus its
embeddings. If the Search service stays in the Altus Nova tenant, the customer's data is at rest in our tenant
in the form that matters most, and option 2 fails its own premise the first time someone asks "where does the
document text physically live?" So option 2 must be read as **blob + SQL + Table + Search in the customer
tenant**, not just blob + SQL.

That reading has a knock-on cost, because Search is not a passive store — it makes three outbound calls:

- → **Blob storage**, via the indexer data source connection (`IndexingService.CreateBlobDataSource`)
- → **Azure OpenAI**, directly, for embeddings (`AzureOpenAIEmbeddingSkill`)
- → **the TextMerge function**, as a `WebApiSkill` custom skill

Moving Search into the customer tenant drags the first along naturally (blob moves too) but turns the other
two into cross-tenant calls: their Search service calling our OpenAI account, and their Search service calling
into our AKS ingress. Both are solvable; neither is free.

### Option 1 — compute and data both in the Altus Nova tenant

Today's architecture, in a new subscription. This is what §6's isolation checklist describes.

- **Viability: high.** No new trust boundaries, no cross-tenant auth, no customer dependency. Quota, cost, and
  operations all stay under our control.
- Only isolates data, compute, and cost — not identity (which §5 now handles separately and deliberately).
- **This is the right launch target.** Options 2 and 3 are commercial differentiators; option 1 is what proves
  the parameterization works. Building 1 with everything properly parameterized turns 2 and 3 into placement
  decisions rather than rewrites.

### Option 2 — compute ours, data in the customer tenant

The pitch is real: customers who will not hand over their corpus will often accept processing-in-place if the
data never leaves their subscription at rest. But the engineering is the most awkward of the three, because it
puts a tenant boundary through the middle of the indexing pipeline rather than around it.

**What it requires beyond option 1:**

- **Cross-tenant auth.** Ironically, today's key-based access makes the naive version easy — keys and
  connection strings are just secrets and cross tenants without any Entra trust relationship. But customers
  sophisticated enough to demand option 2 are exactly the ones enforcing `allowSharedKeyAccess=false` on
  storage and `disableLocalAuth` on Search via Azure Policy. When that lands, the naive version stops working
  and you need a multi-tenant app registration consented in their tenant plus RBAC role assignments they grant.
  **Migrating off key-based auth to managed identity is therefore a prerequisite for option 2, not a nice-to-have.**
- **Network connectivity.** Same customers typically also disable public network access and require Private
  Endpoints. Reaching their storage/SQL/Search from our AKS then needs VNet peering, Private Link, or a VPN —
  per customer.
- **The `WebApiSkill` callback.** Their Search service must reach our TextMerge endpoint. Public ingress works;
  if they require private, Azure AI Search shared private links support `Microsoft.Network/privateLinkServices`,
  so an internal load balancer plus a Private Link Service in front of AKS is the path. Workable, extra work.
- **Operational blast radius inverted.** They hold the keys and the resource locks; they can rotate a key,
  apply a policy, or hit a quota ceiling and break the service — while we own the SLA. This needs to be written
  into the contract, not just the architecture.
- **Latency and egress** on every indexer run and every retrieval. Tolerable in-region, material cross-region.

**Viability: medium.** Genuinely sellable, but it is the option with the worst effort-to-clarity ratio: the
half-measure (blob theirs, Search ours) is cheap and does not deliver the promise, and the honest version costs
nearly as much as option 3 while still leaving compute and the OpenAI account on our side of the line. If a
customer is pushing this hard, check whether what they actually want is option 3.

### Option 3 — compute and data both in the customer tenant, deployed by us

Most sellable, hardest to operate, and the only one that fully answers a data-residency objection.

**What it requires:**

- **Delegated access.** [Azure Lighthouse](https://learn.microsoft.com/azure/lighthouse/) is the right
  mechanism and worth naming explicitly — it grants us RBAC into their subscription cross-tenant without guest
  accounts, scoped per resource group, revocable by them, and auditable. The alternative (a service principal
  they create for us) works but is worse on all four counts.
- **The customer must qualify.** They need an Azure subscription, and critically **Azure OpenAI access with
  sufficient TPM quota in-region**. That is approved per subscription and takes days; some customers will not
  have it at all. This disqualifies a real fraction of prospects and should be a sales qualification question,
  not a discovery-during-implementation surprise.
- **Fleet management.** N independent stacks, each drifting, each needing upgrades, each with its own quota,
  cost, and incident surface. Observability has to aggregate across tenants — App Insights lands in *their*
  subscription, so either Lighthouse-delegated queries or a shipping path to a central workspace is needed.
- **Full IaC becomes mandatory.** Option 3 executed by hand is not a deployment model, it is a growing pile of
  snowflakes. This is the strongest argument in the document for §8's option B or C.

**Viability: low for launch, high as a target.** Nothing about it is technically unreasonable, but it is a
fleet-management product rather than a deployment, and the current codebase — hardcoded search endpoint at
`IndexingService.cs:295`, a plaintext OpenAI key in `appsettings.json`, a hardcoded tenant ID in three places,
no seeding mechanism — is several steps away from being able to support even one instance of it cleanly.

### Recommendation

Commit to **option 1 for launch, built as if for option 3**: full parameterization, no hardcoded endpoints or
tenant IDs, managed identity instead of keys, IaC covering the whole stack. Sell option 3 as the enterprise
tier once one instance has been provisioned from templates end to end. Offer option 2 only if a specific
customer demands it and rejects option 3 — it is the awkward middle, and its cost is closer to option 3's than
its value is.

---

## 4. Migrating compute to AKS

### What it costs

Nothing is containerized today — there is not a single Dockerfile in any of the three repos. The migration
covers 3 App Services, 1 Static Web App, and 5 Function Apps.

**The good news is the Function Apps, which look like the hard part and are not.** All five run the
**isolated worker model** (Functions Worker 2.0, .NET 8), and the trigger inventory is:

- 6 × `TimerTrigger`
- 4 × `HttpTrigger` at `AuthorizationLevel.Function`
- **No Durable Functions, no queue triggers, no blob triggers, no Event Grid triggers.**

That is close to the cheapest possible Functions→Kubernetes profile. Timer triggers map to KEDA's `cron`
scaler or plain Kubernetes `CronJob`s; the function-key HTTP endpoints become ordinary services behind ingress.
Microsoft ships base images for the isolated worker host, so the Functions runtime can be kept as-is inside a
container rather than rewritten.

**Two gotchas that will bite if unplanned:**

1. **Timer singleton locking still needs a storage account.** The Functions host uses blob leases to stop two
   replicas firing the same timer. Keep `AzureWebJobsStorage` wired, or run those deployments at replica 1.
   Several triggers use `RunOnStartup = true`, which means every pod restart, rollout, and node drain fires the
   sync immediately — currently harmless on always-on App Service plans, noticeably less so under Kubernetes
   rescheduling. Audit those six timers for idempotency before the cutover.
2. **The TextMerge HTTP endpoint is called by Azure Cognitive Search**, not by our own code. Its ingress must
   remain reachable from the Search service under every hosting option (see §3).

**SignalR:** `Program.cs` uses `AddAzureSignalR`, so the app makes an outbound connection to the managed
service — this keeps working unchanged on AKS and is the low-effort choice. Self-hosting SignalR to remove the
PaaS dependency means a Redis backplane, sticky sessions, and WebSocket-capable ingress. Don't do it as part of
the migration.

**Front ends:** both SPAs become nginx containers behind ingress. The voice FE loses Static Web Apps' built-in
CDN, routing, and free certs; ingress-nginx plus cert-manager plus a CDN in front replaces them.

### What it buys — and what it doesn't

AKS makes **compute** portable and uniform. It does not make the *platform* portable, because Azure OpenAI,
Cognitive Search, SQL, Storage, and SignalR are managed services that stay managed services in every option.
On a generous count the containerizable surface is one row of the §1 table.

This matters for how AKS is justified:

- **AKS does not enable options 2 or 3.** Those are unblocked by parameterized configuration, managed identity,
  cross-tenant auth, and IaC — every one of which is needed whether the compute runs on AKS, App Service, or
  Container Apps. Adopting AKS *and* claiming it delivers customer-tenant hosting would be crediting it for
  work it doesn't do.
- **AKS's density argument only applies to option 1.** One cluster with a namespace per customer is where
  Kubernetes genuinely beats N sets of App Service plans. Options 2 and 3 imply a cluster per customer, which
  pays the control-plane and node-pool floor per customer while getting none of the bin-packing benefit.
- **What it does buy, honestly:** uniform deploy artifacts across the .NET API, the NestJS voice backend, and
  both SPAs; one ingress/cert/secrets story instead of four; portability if a customer ever demands non-Azure
  or on-prem; and a credible answer to enterprise buyers who ask for a Kubernetes-shaped deployment.

### Consider Azure Container Apps first

If the goal is "portable, uniform, per-customer compute" rather than "Kubernetes specifically," **Azure
Container Apps** delivers most of it at a fraction of the operational cost: it is KEDA-based (so the same cron
scalers work), has managed ingress and certs, scales to zero, and needs no cluster upgrades, node pools, or
control-plane tier. It runs the same container images, so choosing it is not a dead end — moving ACA→AKS later
is a deployment-manifest change, not a rebuild.

Pick AKS over ACA deliberately, for one of: customers contractually requiring Kubernetes; a need for non-Azure
targets; or workloads needing cluster-level control (GPUs, custom CNI, service mesh, Private Link Service
plumbing for §3's `WebApiSkill` path). If none of those apply, ACA is the better fit for "minimum work to
launch."

**Viability: high, but re-sequence it.** Containerization is worth doing and the Functions profile makes it
unusually cheap. It is not on the critical path for a first isolated instance, and putting a from-scratch
Kubernetes migration in front of the launch inverts the "minimum work to launch" constraint agreed on
2026-07-31. Containerize the workloads (which is the reusable part), ship on ACA or App Service, and move to
AKS when a concrete requirement demands it.

---

## 5. Identity — Entra ID only

**Decision: ANA integrates with Azure Entra ID and no other identity provider.** Customers who use Okta, Ping,
ADFS, or Google federate those into their own Entra tenant; from ANA's perspective the token issuer is always
Entra.

**Viability: high. This is the strongest decision to come out of the meeting** — it is the correct scope
boundary, it is cheap to implement, and it does not corner us.

Why it holds up:

- Entra federation into third-party IdPs is a standard, supported enterprise pattern, not a workaround.
  Okta-primary organizations commonly run Entra federated for M365 already, and Entra supports SAML/WS-Fed
  direct federation for external identities. The customer's SSO, MFA, and lifecycle policies keep working;
  ANA never sees Okta.
- It collapses an unbounded integration surface (SAML quirks per IdP, SCIM provisioning, per-IdP claim
  mapping, N sets of certificates to rotate) into one well-documented protocol surface we already use.
- MSAL is already in both front ends and `AddMicrosoftIdentityWebApi` in the API. Nothing is thrown away.

**The one real caveat:** it assumes the customer *has* an Entra tenant. A prospect on Google Workspace + Okta
with no Microsoft footprint does not, and "create an Entra tenant to use our product" is genuine friction — a
free Entra tenant with federation configured is a small but non-zero ask, and it needs an owner on their side.
Entra External ID's OIDC federation is the escape hatch if this comes up. Make "do you have an Entra tenant?"
a sales qualification question.

### What this costs in code

The platform is single-tenant today and hardcoded to the Altus Nova tenant:

- `appsettings.json:42,47,54` — tenant ID `e99365e9-…` in three separate config sections
- `Program.cs:41` — `AddMicrosoftIdentityWebApi` bound to that single-tenant authority
- `msalConfig.ts` — authority already comes from `VITE_MSAL_AUTHORUTY`, so the FE side is env-var-only

There are two ways to serve a customer's Entra tenant, and the cheaper one is not the obvious one:

| Model | How | Cost |
|---|---|---|
| **Per-instance, single-tenant** (recommended) | App registrations live in the customer's tenant; authority and client IDs are per-instance config | Config-only in the API; the FE authority is already an env var. No consent framework, no multi-issuer validation |
| **Multi-tenant registration** | One registration, `/common` authority, admin consent per customer | Requires multi-issuer token validation, a consent flow, and tenant allow-listing to stop arbitrary tenants signing in |

Since every instance serves exactly one customer (multi-tenancy is off the table), **per-instance
single-tenant registrations are strictly cheaper** and fit option 3 hosting naturally. Take that path.

Knock-on effects to plan for:

- **Graph service identity and the mail-ingestion daemon** need admin consent in the customer's tenant, and
  mail ingestion only works if they have Exchange Online.
- **MCP OAuth** requires the API's Application ID URI to be on a domain verified in the tenant that holds the
  registration — `*.azurewebsites.net` cannot be an identifier URI. Per-instance registrations mean this is
  now a per-customer DNS/verification step. Budget for it or launch without MCP. See
  [mcp-claude-oauth-entra.md](mcp-claude-oauth-entra.md).
- **Roles and scopes** are defined per registration — see [entra-scopes-and-roles.md](entra-scopes-and-roles.md).
  Recreating them per instance is scriptable and should be scripted, not clicked.

### Relationship to hosting

Identity and hosting are independent, but not every combination is coherent:

| Hosting | Identity | Coherent? |
|---|---|---|
| Option 1 | Altus Nova tenant | Yes — today's internal model |
| Option 1 | Customer tenant | Yes — our infrastructure, their users. The likely near-term shape |
| Option 2 / 3 | Customer tenant | Yes — the natural fit |
| Option 2 / 3 | Altus Nova tenant | Incoherent — their data, our users. Don't offer it |

---

## 6. Isolation checklist for a dedicated instance

Everything marked *Shared* in §2 becomes a new dedicated resource:

- New Azure subscription + resource groups (customer's subscription under §3 option 3).
- New Azure OpenAI account + chat/embedding/realtime/transcribe deployments.
- New Azure Cognitive Search service.
- New App Insights + Log Analytics workspace (do **not** point at the existing shared workspace).
- New SQL server/DB, storage accounts, SignalR.
- New compute — AKS/ACA workloads or App Services + Function Apps, per §4.
- New custom domains + certs, or accept default hosts (see the MCP caveat in §5).
- New ADO service connection; new variable groups; new release stages.

**Lead-time risk — start this before anything else:** a new subscription has its own quota. Azure OpenAI model
capacity (TPM) and compute quota are granted per subscription and per region, and requests take days. This is
the long pole under every option, and under option 3 it is the *customer's* long pole, which makes it a
schedule risk you do not control. The existing platform is in `eastus2`.

---

## 7. Work required regardless of approach

Hard blockers whether the instance is built by hand or from templates, on AKS or App Service.

| # | Change | Where |
|---|---|---|
| 1 | Add a third environment: `appsettings.<Inst>.json` overriding `BlobAccountName`, `Cors:Origins`, `McpSettings:ResourceUri` | `AltusAiAssistantApi/` |
| 2 | Override the base file's shared values per environment — `SearchServiceEndpoint`, `GptDeployments:*:Endpoint`/`Key`, `ApplicationInsights:ConnectionString` — via app settings or move them out of the base file entirely | `appsettings.json` |
| 3 | **Remove the plaintext Azure OpenAI key** from `appsettings.json` (`b657e441…`). It is committed. Cloning environments multiplies the exposure; rotate it as part of this work | `appsettings.json:30` |
| 4 | **Remove the hardcoded Search endpoint in code** — `new Uri("https://searchaitests2.search.windows.net")` bypasses config entirely and will silently point a new instance at the shared QA/Prod Search service | `IndexingService.cs:295` |
| 5 | **Parameterize the tenant ID** in all three config sections; make the authority per-instance (§5) | `appsettings.json:42,47,54` |
| 6 | Widen `@allowed(['QA','Prod'])` on `environmentName` to accept the new value | `AltusAiProcessContent/infra/main.bicep` |
| 7 | Parameterize the hardcoded OpenAI endpoint, Search endpoint, and tenant ID in the voice Bicep | `altus-ai-assistant-voice/infra/main.bicep` |
| 8 | New subscription + service connection in the voice infra pipeline (currently hardcoded twice) | `azure-pipelines-infra.yml` |
| 9 | Add a `build:<inst>` script + Vite mode + env file for the new FE environment | `altus-ai-assistant-fe/package.json` |
| 10 | New variable groups and release stages for all 8 pipelines | Azure DevOps |
| 11 | Seed reference data — clients, users/roles, taxonomy, custom prompts, content sources, contact-role configs. **No seeding mechanism exists**; manual via admin UI/SQL today | — |

Additional, required only for §3 options 2 and 3:

| # | Change | Why |
|---|---|---|
| 12 | **Migrate from key-based auth to managed identity** — 8 × `AzureKeyCredential`, the blob connection string on the Search indexer data source, and raw SQL connection strings | Customer tenants commonly enforce `allowSharedKeyAccess=false` / `disableLocalAuth` by policy |
| 13 | Containerize all workloads (no Dockerfiles exist today) | §4 |
| 14 | **Audit App Insights / Log Analytics telemetry for customer content** | If prompts or document fragments are logged, telemetry becomes a data-residency leak that undoes options 2 and 3 |
| 15 | Cross-tenant network path for Search → TextMerge and Search → OpenAI | §3 |

Items 3, 4, and 5 are fixes rather than additions and are worth doing regardless of this project — item 4 in
particular is a live hazard that will misdirect any new instance. Item 11 is the most commonly underestimated:
it is pure manual effort today and repeats in full for every future instance.

**To verify:** the FE pipeline runs `npm run build:$(Env)` on a hosted `ubuntu-latest` agent, but `.env.qa` /
`.env.production` are gitignored and therefore absent at checkout. Either the variable group supplies `VITE_*`
as process env vars (which Vite picks up) or QA/Prod builds are getting default values. Confirm before adding
a third mode, or the new FE will silently build against the wrong API.

---

## 8. Provisioning approach (open)

Current automation coverage is roughly **35%**, and the automated part is the easy part:

**Automated** — app/Function build + publish via 8 `azure-pipelines.yml` files; SQL schema via an EF Core
idempotent migration script generated in-pipeline; Search index/skillset creation from application code
(`IndexingService`) + helper scripts; two Bicep templates (voice stack, ProcessContent function).

**Manual** — API, primary FE, and the four original Function Apps; SQL Server/DB, Storage, Cognitive Search,
Azure OpenAI + model deployments, SignalR, Log Analytics; custom domains, certificates, DNS; all Entra
registrations, scopes, pre-authorizations, consent; variable groups, secrets, release definitions; deploy
stages (build pipelines only publish artifacts — deployment lives in classic ADO releases outside the repos);
every seed/reference data row.

### Option A — Manual clone, documented

Build the new instance by hand in the portal, write a runbook as you go.

- **Fastest to first working instance**, no new tooling, no retrofitting risk.
- No drift protection, high error rate across ~40 resources, and instance #3 costs the same again.
- Only defensible if this is genuinely a one-off.

### Option B — Incremental Bicep

Extend Bicep to cover the whole stack, parameterized by instance + environment, one `main.bicep` per instance
with `.bicepparam` files, plus an infra pipeline. Entra stays scripted-plus-runbook.

- Builds on what already exists and can be staged (data → compute → networking) without a freeze.
- The new instance becomes the *proof* — provision it from templates rather than the portal, so the work is
  validated by the launch instead of added on top of it.
- Adds calendar time before first deploy; retrofitting the hand-made QA/Prod resources needs care.

### Option C — Full IaC (Terraform / Azure Developer CLI)

One tool for Azure + Entra (Terraform's `azuread` provider creates app registrations), remote state, `azd
up`-style provisioning.

- Strongest end state — genuinely one-command instances, Entra registrations included, which matters a lot
  under §5's per-instance-registration model.
- Cross-tenant deployment (§3 option 3) is a first-class scenario: Terraform providers take an explicit
  `tenant_id`/`subscription_id`, so a Lighthouse-delegated fleet is a variable file per customer.
- Largest up-front effort, abandons the existing Bicep, introduces state management.

### How to choose

**The §3 direction has largely answered this.** The old framing — "is there a fourth instance?" — is settled
the moment customer-tenant hosting is on the roadmap: options 2 and 3 mean one instance per customer, and
manual provisioning of a customer fleet is not a viable operating model. Option A survives only as the
launch-day tactic for the *first* instance, not as the approach.

Recommendation: **hybrid now, option B or C next.** Use the two existing Bicep templates where they already
cover ground (§7 items 6 and 7 make them instance-ready cheaply), do the rest by hand for instance #1, and
write the runbook as the specification for the IaC that follows. If customer-tenant hosting is committed to
rather than aspirational, skip B and go to C — Terraform's ability to create Entra registrations and target
arbitrary tenants is exactly the gap that B leaves open, and doing B first means writing the infrastructure
twice.

---

## 9. Sequence

Steps 1–3 are on the critical path and do not depend on the §8 decision — start them now.

1. **Pick the launch hosting option** (§3). Recommendation: option 1, built parameterized for option 3.
2. Request the new subscription; file OpenAI + compute quota requests in `eastus2` **immediately** — the long
   pole under every option.
3. Fix the config hazards: hardcoded Search endpoint (`IndexingService.cs:295`), plaintext OpenAI key,
   hardcoded tenant ID ×3, shared App Insights connection string (§7 items 2–5). Prerequisite for cloning
   environments under any option, and item 4 is a live misconfiguration risk today.
4. **Decide the provisioning approach** (§8), informed by whether customer-tenant hosting is committed.
5. Decide instance naming and hostnames; start DNS/cert work early — per-instance Entra registrations make
   domain verification a per-customer step if MCP is in scope (§5).
6. Provision the resources in §6, by the chosen approach.
7. Apply the remaining §7 config/pipeline changes; create the service connection, variable groups, and release
   stages; run one full deploy.
8. Entra work: create per-instance registrations in the target tenant, script them (§5).
9. Seed reference data (§7 item 11) and configure the third-party integrations with new credentials.

### Parallel track — containerization

Independent of steps 1–9 and safe to run alongside them, since it changes packaging rather than architecture:

1. Dockerfiles for the API, voice BE, and both SPAs.
2. Dockerfiles for the 5 Function Apps on the isolated-worker base image; audit the six timer triggers for
   idempotency and `RunOnStartup` behaviour under rescheduling (§4).
3. Deploy the containers to ACA or App Service for Containers first. Move to AKS when a requirement in §4's
   list actually appears.

### Worth doing eventually, out of scope for launch

Add deploy stages to the YAML pipelines so releases stop living in classic ADO outside the repos; add a
seed/bootstrap path for reference data (EF seeding or an idempotent script); migrate data-plane auth from keys
to managed identity (§7 item 12 — required for options 2 and 3, and correct regardless). All three are pure
cost at launch and pure savings on every instance after.

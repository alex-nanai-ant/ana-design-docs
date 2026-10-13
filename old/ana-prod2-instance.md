# ANA prod2 — Second ANA Production Instance

**Status:** in preparation · 2026-10-02. Subscription created. Code is prepared on
`proposal/ana-prod2` branches in `altus-ai-terragrunt` and `altus-ai-gitops`, not yet applied or
merged. Open questions are in [§8](#8-open-questions).

## 1. What we are launching

A second, separate ANA production instance at **`assistant2.altusnova.ai`**.

It is the same product as `ana-prod`: same builds, ANA branding, default prompts. It runs in its own
Azure subscription so that its usage can be **tracked and billed to the customer using it**. The
first customer is Emerie, but nothing in the instance is Emerie-specific. Another customer could use
it later without renaming anything.

It is **not** related to the Emerie-branded instance (`ana-emerie-qa` / `ana-emerie-prod`), which is
a different edition (Emerie branding, `PromptInstance=emerie`) serving Emerie's clients.

| | Users | Edition | Azure subscription |
|---|---|---|---|
| `ana-qa` / `ana-prod` | Altus Nova | ANA | old shared sub (`aitests1`, `searchaitests2`) |
| `ana-emerie-qa` / `ana-emerie-prod` | Emerie's clients | Emerie | "ANA for Emerie" (`e1c7ee8e…`) |
| **`ana-prod2`** | whichever customer it's billed to (currently Emerie) | **ANA** | **"ANA for Emerie Users"** (`602dfaba…`) |

Background: [old/new-instance-provisioning.md](old/new-instance-provisioning.md) covers the general
"new instance" problem. Much of it is out of date, because compute now runs on the shared `altus-k8s`
AKS cluster, releases go through Kargo, and PaaS is provisioned with Terragrunt. This doc covers only
this instance, using the current tooling.

---

## 2. Naming (decided 2026-10-02)

The instance is named by **what it is** (a second ANA prod), not by who uses it, so the name stays
correct if the customer changes.

| Thing | Value |
|---|---|
| Instance ID (Kargo stage, Argo app + AppProject, namespace, KV secret prefix, chart `namePrefix`) | `ana-prod2` |
| UI host | `assistant2.altusnova.ai` |
| API host | `api.assistant2.altusnova.ai` |
| Indexer skills host | `index-skills.assistant2.altusnova.ai` |
| Azure subscription | `ANA for Emerie Users` (`602dfaba-e95d-4fde-987f-fabe611dcdb6`) |
| Resource group | `rg-ana-prod2-eastus` |
| Azure resources | `stanaprod2eastus`, `searchanaprod2`, `oaianaprod2eastus`, `signalranaprod2`, `sqlsrv-ana-prod2-cus` (centralus, see §6) |
| Entra apps / group | `ANA-Prod2-SPA/API/Worker`, `ANA Prod2 Users` |
| Terragrunt tree | `live/ana/prod2/` (own `env.hcl`, own state) |
| Tags | `environment=prod2`, `product=ana`, `instance=ana-prod2`, `managed_by=terragrunt`, `project=altus-ai-infra`. No new tag keys; billing separation comes from the subscription |

Why `ana-prod2` and not `ana-prod-2`: Key Vault secrets are named `<prefix>-<key>`. With `ana-prod-2`,
every secret would start with `ana-prod-`, so prefix searches and wildcards would match both prod
instances. `ana-prod2-` can't be confused with `ana-prod-`.

The hosts are in the parent `altusnova.ai` zone, which ExternalDNS already manages. The records are
created automatically from the HTTPRoutes. (The legacy `assistant.altusnova.ai` needed a manual CNAME
only because of its App Service migration.)

---

## 3. Isolation and billing boundary

| Layer | Where it lives | Billed separately? |
|---|---|---|
| **Azure OpenAI** (chat, mini, embeddings) | New subscription | **Yes** |
| AI Search, Blob/Table Storage, SQL, SignalR | New subscription | **Yes** |
| Compute (API, FE, sync/processing workers) | Shared `altus-k8s`, namespace `ana-prod2` | No, shared cluster |
| Secrets | Shared KV `kv-altus-k8s-eastus`, prefix `ana-prod2-*` | No (negligible) |
| Container images / charts | Shared ACR `acraltusdeveastus` (every stage pulls from it) | No (negligible) |
| Terraform state | New subscription (own bootstrap) | Yes (negligible) |

**Dedicated subscription for all PaaS, shared cluster for compute**, the same model as the
Emerie-branded instance. Every OpenAI and Search charge lands on the subscription's invoice line,
with no tag filtering needed.

Compute is the only cost that doesn't split cleanly, and it's small. The `ana-prod` pods request about
0.4 vCPU in total. If the customer has to be billed for compute too, AKS cost analysis breaks cost
down per namespace.

The ACR sits in the "ANA for Emerie" subscription for historical reasons, so that subscription pays for
the registry every stage uses. It's a Basic SKU, so the cost doesn't matter, but it's worth knowing if
that subscription is ever re-billed.

---

## 4. Azure OpenAI

The same setup as the other instances (decided 2026-09-30). The `modules/assistant-services` module is
used unchanged: a `kind = "OpenAI"` account with `gpt-5.2` (chat), `gpt-5-mini` (identity extraction)
and `text-embedding-ada-002` (1536 dimensions, which must match the index schema). The account shows
up in the Azure AI Foundry portal like any Azure OpenAI resource.

**Region: `eastus`**, the same region as `altus-k8s` and the rest of this instance's services.
Quota is per subscription, so the eastus embedding shortage in the Emerie subscription doesn't apply.
If the eastus quota request is refused, fall back to `eastus2`.

**Capacity:** 10K TPM per deployment, the same as the other instances. That is a floor, not a sizing
figure. Check the quota as soon as the subscription is available and size it to expected usage (§8).

---

## 5. Identity

The standing decision (2026-08-04) is **Entra ID only**. This instance uses the same model as
`ana-emerie-prod`: a multi-tenant SPA/API/Worker trio in the Altus hub tenant (`modules/assistant-entra`,
`product_label = "ANA"`, `AzureADMultipleOrgs`), assignment-required, with the access group
`ANA Prod2 Users` (becomes `Settings__UsersDirectoryGroupId`).

The customer's users sign in with their own Entra accounts as B2B guests in the hub. They have to be
guests and members of the group before they can sign in. The module's default group members (four
Altus staff) are also added, which gives Altus support access. No code changes are needed.

The Worker (Graph) and mail-reading apps only matter for the integrations the customer actually uses
(§6).

---

## 6. Content sources and data

The instance starts **empty**: new Search service, storage, and SQL. Nothing is copied from other
instances.

- **Integrations belong to the customer.** Confluence, Slack and Grain need credentials for the
  customer's workspaces, and mail sync needs an app with `Mail.Read` in whichever tenant holds the
  mailbox. All four workers ship with `enabled: false`; turning one on later is a payload edit plus its
  Key Vault secret.
- **Search tier depends on client count.** Every ANA client gets its own index (`{clientId}-index`).
  Basic allows **15 indexes**, S1 allows 50. For more than about 12 clients, start on S1, because a
  tier change means creating a new service.
- **Seeding is still manual:** first admin user, client list, taxonomy, any custom prompts,
  contact-role configs. `PromptInstance` stays unset (ANA default prompts).
- **SQL region: `centralus`.** Both eastus and eastus2 refused new SQL servers in this subscription
  (`ProvisioningDisabled`, 2026-10-02), the same restriction the Emerie subscription has. centralus
  is about 25–30 ms from the cluster, the same setup Emerie QA/prod already run with. Each refused
  attempt left a name reservation that isn't visible in the resource group, so the server name carries
  a region suffix: `sqlsrv-ana-prod2-cus`. A support request (issue type "Service and subscription
  limits") could open eastus/eastus2 later if the latency matters.

---

## 7. Work breakdown

Items marked ✅ are prepared on the `proposal/ana-prod2` branches, not applied or merged. Blanks are
marked `TODO-`; `grep TODO-` in both repos lists what's left.

### Terragrunt (`altus-ai-terragrunt`)

1. ✅ `live/ana/prod2/env.hcl`: subscription ID filled in, names from §2, own tfstate account
   (`TODO-`: the state account name, generated at bootstrap). It overrides `live/ana/env.hcl`
   because Terragrunt uses the nearest `env.hcl`.
2. ✅ Stacks, in apply order: `_tfstate-bootstrap` (subscription pinned in its provider) →
   `resource-group` → `assistant-services` → `sql` (firewall includes the `altus-k8s` NAT) → `entra`.
   No module changes.
3. ✅ READMEs: `live/ana/prod2/README.md` (first-time setup, fallbacks), `live/ana/README.md`, root
   apply order.

### Secrets

4. ✅ `scripts/set-ana-prod2-kv-secrets.ps1` writes the 10 `ana-prod2-*` secrets to
   `kv-altus-k8s-eastus` from Terragrunt outputs. The chart's ExternalSecret fails if any key is
   missing, so unused integrations get a `disabled` value.

### GitOps (`altus-ai-gitops`)

5. ✅ `apps/cluster/appproject-ana-prod2.yaml`.
6. ✅ `apps/payloads/ana-prod2.yaml`, copied from `ana-prod.yaml`:
   - new hosts, CORS, endpoints, `namePrefix`, ANA branding
   - `migration.enabled: true`
   - Confluence, Slack, mail and Grain sync disabled
   - `ASPNETCORE_ENVIRONMENT: Production`, not `Prod`, because `Prod` loads `appsettings.Prod.json`
     with ana-prod's blob, CORS and MCP values
   - `TODO-`: the six Entra outputs
7. ✅ `apps/kargo/ana/stage-ana-prod2.yaml` (freight from `ana-qa`) and its `projectconfig.yaml`
   entry.
8. ✅ Three gateway listeners and certs in `public-gateway.yaml`, `DOMAINS.md`, README diagrams.

**Freight source:** `ana-qa` (decided 2026-10-02). `ana-prod` and `ana-prod2` are siblings: any
build verified on QA can be promoted to either one independently. prod2 doesn't wait for `ana-prod`,
and `ana-prod` doesn't wait for prod2.

**Don't merge the gitops branch early.** The `payloads` app-of-apps syncs every file in
`apps/payloads/`, so merging would deploy an app with missing secrets and request TLS certs for hosts
that have no backend yet.

### App repos

9. None.

### Launch

10. Bootstrap state → fill in the state account name → apply the stacks → run the secrets script →
    fill in the Entra `TODO-` values → merge both branches.
11. Promote QA-verified freight to `ana-prod2` (the build `ana-prod` runs is a safe first choice). Smoke-test with tiers 0–4 of
    [old/aks-migration-smoke-test.md](old/aks-migration-smoke-test.md), especially T2.2 (SignalR),
    T4.5 (TextMerge reachable from Search) and T4.6 (end-to-end indexing).
12. Seed data (§6), invite the customer's users as guests and add them to `ANA Prod2 Users`.

Prerequisites for whoever runs the applies:
- Terraform ≥ 1.15.
- **Storage Blob Data Contributor** on the new state account. Owner on the subscription doesn't
  include it.

---

## 8. Open questions

1. **Identity.** Does Emerie use Entra ID (Microsoft 365), so its users can sign in as B2B guests?
2. **Content sources.** Which of Confluence, Slack, Grain and mail are needed? For mail, which mailbox
   and tenant?
3. **Scale.** How many users and clients? This sets the OpenAI TPM and the Search tier (15-index Basic
   limit).
4. **Compute billing.** Is billing the subscription enough, or does the customer also pay a share of
   the shared cluster?
5. **Scope.** MCP connector for Claude and ANDI are assumed out of scope for launch. Correct?

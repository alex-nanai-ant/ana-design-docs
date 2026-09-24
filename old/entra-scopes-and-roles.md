# Entra Scopes & App Roles — AiAssistantApi

> **Scope:** the authoritative list of delegated **scopes** and app-only **app roles**
> exposed by the `AiAssistantApi` resource server (`4279cecc-…`), the naming model, who
> holds each, and the cross-surface authorization gotchas. Companion to
> [entra-app-registrations.md](entra-app-registrations.md) (the registrations themselves),
> [andi-mcp-auth.md](andi-mcp-auth.md) (the ANDI auth design), and
> [andi-mcp-ana-api-implementation.md](andi-mcp-ana-api-implementation.md) (the API cut).
> **Status:** rows are marked **current** (built) or **proposed** (from the ANDI /
> resource-naming work, nothing built yet).

Tenant: `e99365e9-089f-4355-a598-bd139993cd1e` (Altus Nova).
Resource: `AiAssistantApi`, client id `4279cecc-ad27-4e90-81be-af8e4c038897`,
App ID URI `api://4279cecc-ad27-4e90-81be-af8e4c038897`.

---

## 1. Scopes vs. app roles — the two token shapes

Entra carries a caller's permissions in **two different claims**, depending on how the
token was obtained. This distinction is load-bearing for every gate below.

| | **Delegated scope** | **App role** |
|---|---|---|
| Claim | `scp` (space-delimited string) | `roles` (JSON array) |
| Flow | a human signs in — auth-code / OBO | client-credentials — no human |
| Means | "this **user**, via this app" | "this **app**, on its own behalf" |
| Example holders | Ana FE, Claude, ANDI's OBO picker | ANDI's SP, future services |
| `allowedMemberTypes` | n/a | **`["Application"]`** — never `User` (see below) |

`roles` is **not** app-only in general: an app role whose `allowedMemberTypes` includes
`User` lands in a *delegated* token's `roles`. Every app role here is
`["Application"]`-only, so its presence in `roles` unambiguously means app-only — and a
user can never be assigned it and trip the app branch. The one claim that reliably means
"user present" is **`scp`**. (Ana's own `Admin`/`User` roles are *not* Entra app roles —
they're DB-backed via `RoleClaimsMiddleware`, arriving in `ClaimTypes.Role`, a different
claim entirely, so the two role systems never collide.)

### Naming

- **Delegated users get one coarse MCP scope, `Mcp.Invoke`**, granting *all* MCP tools.
  A delegated token is already user-bounded (it can only reach what the user can), so there
  is no least-privilege reason to split it per tool. Both Claude and ANDI's OBO picker use it.
- **App roles are per-resource, `Mcp.<Resource>.<Operation>`** — following Microsoft's
  `Resource.Operation[.Constraint]` convention — because least privilege *does* matter for
  headless services: a service should be grantable document-read without, say, expensive
  chat-completion. Today that means one role, `Mcp.Documents.Read` (search / fetch).
- **Breadth is not in the name — no `.All`.** `.All` means "every resource," but every
  identity here is *scoped*: a user to their `UserClient` rows, an app to its `ServiceApp`
  allowlist. That breadth lives in the allowlist, not the role, and varies per app (ANDI =
  all clients, another integration = a subset) while the role definition is uniform. So the
  constraint is omitted; `.All` would only fit a role that *bypasses* the allowlist, and none
  exists.
- **`Mcp.` is the surface namespace** (its origin is `Mcp.Invoke` = "invoke the MCP
  endpoint"), keeping MCP permissions distinct from REST ones. The *unprefixed* `Clients.Read`
  is reserved for a possible future REST-surface scope (§7), never the MCP tool.
- **Avoid case-twins.** A `Mcp.invoke` scope beside a `Mcp.Invoke` role would be a silent
  transcription trap (matching is ordinal / case-sensitive). The §2 rename killed that one.

Ana's token validation (`AddMicrosoftIdentityWebApi`) admits either claim; the default
requires **`scp` or `roles`** be present. A policy that inspects only `scp` rejects every
app-only token regardless of roles — that is why admitting ANDI's SP needs a policy change,
not just a role assignment.

### The two axes that must not be conflated

1. **Capability** — *may this identity use this tool at all?* → the scope/role it holds.
2. **Result scoping** — *which clients' data comes back?* → intrinsic to the **token shape**,
   decided by `ClientScopeResolver`: a delegated/OBO token → that **user's** `UserClient`
   rows; an app-only token → the **app's own allowlist**. An app-only token *cannot* return
   "a user's clients" — there is no user in it. This is why client-listing is a delegated
   activity (§4).

---

## 2. Delegated scopes (`scp`)

Exposed on `AiAssistantApi`, requested by client apps on behalf of a signed-in user.

| Scope value | Scope `id` (GUID) | Held by | Purpose | Status |
|---|---|---|---|---|
| `user-access` | `5ee826c9-46ab-441a-9ec7-52bb99bed16b` | Ana FE, Voice FE (shared reg); **ANDI config UI (OBO)** | REST API access for the signed-in user — includes the clients list | **current** |
| `Mcp.Invoke` | `3b3e9efa-0089-4847-91ac-3045e4bc3416` | Claude MCP client | Coarse: invoke **all** `/mcp` tools as the user (incl. `list_clients`) | **current** |

`Mcp.Invoke` is **kept coarse** — it stays the single delegated MCP scope covering every tool
for a signed-in user. It is not split into per-tool scopes: the per-resource split lives on
the *app-role* side (§1, §3), where least privilege matters.

> **Rename recorded:** this scope was `Mcp.invoke` (irregular casing), renamed to `Mcp.Invoke`
> while MCP had no consumers, so no migration cost. The scope **`id` (`3b3e9efa-…`) is
> unchanged**, so client API-permission references survived — only the value string and
> `McpSettings:RequiredScope` changed.

**ANDI's config picker uses `user-access` on REST — the FE's path.** ANDI exchanges the
signed-in user's token via OBO for `user-access` and calls the existing REST clients endpoint
(`/api/clients`), exactly as the Ana FE does; the result is that user's own clients (§4). Its
data plane separately calls `search_documents` / `fetch_document` on `/mcp` via its SP with the
`Mcp.Documents.Read` app role (§3). So ANDI uses REST (`user-access`, OBO) for config-time
listing and `/mcp` (SP) for the data plane — it does **not** call MCP `list_clients`.

**Separate resource server:** the voice API (`31d01534-…`, per-environment) exposes its own
`Voice.Access` scope — see [entra-app-registrations.md §6](entra-app-registrations.md). Not on
`AiAssistantApi`, out of scope here.

---

## 3. App roles (`roles`)

Exposed on `AiAssistantApi`, assigned to service principals for client-credentials
(app-only) calls. **Proposed** — nothing built yet.

There is **one** app role. A service invokes the data plane against a client it already
knows; it does not enumerate clients (§4), so no app role grants `list_clients`.

| App role value | `allowedMemberTypes` | Gates tools | Assigned to | Status |
|---|---|---|---|---|
| `Mcp.Documents.Read` | `["Application"]` | `search_documents`, `fetch_document` | ANDI SP; any future service | **proposed** |

At the Entra layer `Mcp.Documents.Read` grants the document tools; the **per-client bound is
Ana's `ServiceApp` allowlist**, not the role (an app may hold a broad permission yet be
limited by application logic). The `clientId` on each call is required and validated against
that allowlist by `ClientScopeResolver`.

Config: binds to `McpSettings:RequiredAppRole` (= `Mcp.Documents.Read`,
[McpSettings.cs](../altus-ai-assistant/AltusAiAssistantApi/Models/McpSettings.cs), mirroring
`RequiredScope`).

> **No app role grants `list_clients` — deliberately, on principle.** A service always operates
> on a known/pinned client (ANDI injects the workshop's `anaClientId`), so it never needs to
> enumerate; and an app-only list would return the app's whole allowlist, a backdoor if
> surfaced to a user (§4). If a future service genuinely must enumerate, define a role then —
> it is not pre-built.
>
> A future `Mcp.Chat.Create` **app** role would be defined *only if* a headless integration
> needs Ana-as-a-service completions — a separate grant from document read, because it is far
> more expensive (LLM + MAF workflow) and a different capability class (§5).

---

## 4. Client-listing: a delegated-user activity

The crux the ANDI thread settled: **listing clients is something humans do, not services.** A
service always knows the client it is working on (ANDI pins the workshop's `anaClientId`), so
it has no reason to enumerate — and if it did, an app token would only ever return the app's
whole allowlist, a backdoor the moment it reaches a user. So `list_clients` is a
**delegated-user** operation; no app role admits it.

**MCP `list_clients` tool** (delegated-only):

| Caller | Token / claim on `list_clients` | Allowed? | Returns |
|---|---|---|---|
| Claude | delegated, `Mcp.Invoke` scp | ✅ | the calling user's clients |
| Any service (incl. ANDI SP) | app-only, `Mcp.Documents.Read` role | ❌ | **403 / tool filtered out of `tools/list`** |

**REST clients endpoint** (`/api/clients`, gated by `user-access`) serves the client picker for
both the **Ana FE** and **ANDI's config UI** — the latter via an OBO `user-access` token (§2).
ANDI does not call MCP `list_clients`. The **result-scoping axis** (§1) governs everywhere: a
delegated claim always yields the caller's own clients, never more.

**Why services are denied listing — and why it's Ana-enforced.** (1) *Principle:* a service
operates on a known client; enumeration is a config-time, human activity. (2) *Safety:* an app
token has no user, so a service list could only return the app's whole allowlist — surfacing
that to a user would let them see clients beyond their own access. Because the gate lives in
Ana (`list_clients` requires a delegated `scp`), the boundary is **structural**: an app token
gets a 403 no matter what — not something ANDI has to get right.

**Implementation.** `list_clients` keeps its per-tool `[Authorize]` requiring a delegated
`Mcp.Invoke` scope, so it is filtered out of `tools/list` for any app token — this is the
`McpDelegatedUser` gate in [andi-mcp-ana-api-implementation.md §4](andi-mcp-ana-api-implementation.md),
which stands. The data tools admit *delegated `Mcp.Invoke` **or** app-role `Mcp.Documents.Read`*.

---

## 5. The resource family (extensibility)

Delegated users reach all tools through the coarse `Mcp.Invoke` scope. **Service** access is
per-resource, and each new MCP tool gets a **new sibling app role** (never folded into an
existing one), so a service can be granted exactly what it needs:

| Tool(s) | Delegated (user) | App role (service) | Status |
|---|---|---|---|
| `search_documents`, `fetch_document` | `Mcp.Invoke` | `Mcp.Documents.Read` | proposed |
| `list_clients` | `Mcp.Invoke` | — (services never list) | current / n/a |
| *(future)* chat completion (RAG + taxonomy + contacts + MAF) | `Mcp.Invoke` (or a new scope) | `Mcp.Chat.Create` (only if needed) | future |

`Mcp.Chat.Create` is a **different resource and operation** than `Mcp.Documents.Read` (it
*creates* a synthesized answer, it doesn't *read* a document), so a service that wants it must
be granted it separately — for cost control (LLM + workflow spend, the
[andi-mcp-auth.md §11](andi-mcp-auth.md) runaway-cost concern) and because its audience differs
(ANDI runs its own agent over raw documents and does *not* want Ana's completion; an "ask Ana"
integration wants the opposite). If the delegated side is ever split for cost gating, the list
capability would become `Mcp.Clients.Read` and documents `Mcp.Documents.Read` as *scopes* too —
not planned now.

---

## 6. Capability × identity matrix

| Identity | `user-access` (scp) | `Mcp.Invoke` (scp) | `Mcp.Documents.Read` (role) |
|---|---|---|---|
| Ana FE (user) | ✅ | — | — |
| Claude MCP client (user) | — | ✅ | — |
| ANDI — config picker (OBO, user) | ✅ | — | — |
| ANDI — data plane (SP) | — | — | ✅ |
| Future headless service (SP) | — | — | ✅ |

---

## 7. Cross-surface gotcha: REST does not enforce scopes

**Load-bearing fact, currently true in code.** REST controllers gate on **authentication and
DB roles only** — `[Authorize]` / `[Authorize(Roles="Admin")]` — and check **no `scp`**
([UserController.cs](../altus-ai-assistant/AltusAiAssistantApi/Controllers/UserController.cs),
etc.). Scope enforcement (`RequiredScope`) exists **only** on `/mcp`. So `user-access` is not
actually enforced; any validly-audienced, authenticated token reaches any REST endpoint
(subject to roles). Two consequences:

1. **A narrow scope restricts nothing on its own.** Adding a `Clients.Read` REST scope (or any
   granular scope) does *not* limit a caller until the *other* endpoints require `user-access`.
   Least-privilege by scope needs a baseline everywhere, not one endpoint.
2. **MCP tokens are usable at REST.** MCP and REST share the one `AiAssistantApi` registration
   and audience (`4279cecc`, [Program.cs](../altus-ai-assistant/AltusAiAssistantApi/Program.cs)
   line 41). Since REST ignores scopes, a delegated MCP token (`scp: Mcp.Invoke`, no
   `user-access`) passes REST `[Authorize]` and `RoleClaimsMiddleware` even stamps the user's
   role — so a token consented only for "invoke MCP" is effectively a full REST token for that
   user (Admin endpoints included, if they are Admin). (ANDI's OBO picker token legitimately
   carries `user-access`, so it is a proper REST caller — the concern is tokens that *lack* it.)
   A future ANDI **SP** token also clears REST `[Authorize]` (no DB role, so smaller blast
   radius, but not cleanly rejected). The boundary is asymmetric: `Mcp.Invoke` gates *into*
   `/mcp`, but nothing gates REST by scope.

**Remediation (one change fixes all of it).** Add a **fallback authorization policy** requiring
`scp` contains `user-access` on all REST endpoints. Then MCP-only tokens (Claude's MCP token,
ANDI's SP) are rejected from REST; the FE and **ANDI's OBO picker** keep working (both carry
`user-access`). **Audit every non-FE caller first** — anything relying on "authenticated is
enough" will start getting 403s. The heavier
alternative is a separate registration/audience for MCP so its tokens are cryptographically
unusable at REST.

---

## 8. References

- [entra-app-registrations.md](entra-app-registrations.md) — the registrations, GUIDs, redirect URIs, MCP OAuth flow.
- [andi-mcp-auth.md](andi-mcp-auth.md) — §1 (allowlist ceiling), §6 (ANDI wrappers), §9 (escalations). Reconciled to this permission model: one `Mcp.Documents.Read` app role, no app-only listing, OBO picker on the REST clients endpoint (`user-access`).
- [andi-mcp-ana-api-implementation.md](andi-mcp-ana-api-implementation.md) — §4 (policies); its delegated-only `McpDelegatedUser` gate on `list_clients` stands.
- Code: [McpSettings.cs](../altus-ai-assistant/AltusAiAssistantApi/Models/McpSettings.cs) (`RequiredScope` = `Mcp.Invoke`, proposed `RequiredAppRole` = `Mcp.Documents.Read`), [Program.cs](../altus-ai-assistant/AltusAiAssistantApi/Program.cs) (`McpInvoke` policy line 45, `/mcp` mapping line 404), [RoleClaimsMiddleware.cs](../altus-ai-assistant/AltusAiAssistantApi/Middleware/RoleClaimsMiddleware.cs) (DB roles → `ClaimTypes.Role`).

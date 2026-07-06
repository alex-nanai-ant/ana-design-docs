# ANDI → Ana MCP: Authentication & Authorization

> **Scope:** how **ANDI** (a separate AI workshop-facilitation app, its own FE + BE) authenticates to Ana's `/mcp` endpoint, and how each workshop is bound to a **single Altus client** by a token **Ana itself issues**.
> **Companion docs:** [mcp-claude-oauth-entra.md](mcp-claude-oauth-entra.md) (the per-user delegated flow Claude uses) and [entra-app-registrations.md](entra-app-registrations.md) (the registrations that front `/mcp`).
> **Status:** design / proposal. Nothing below is built yet; it describes the intended shape and the concrete changes Ana needs.

---

## 1. The core tension — ANDI is not Claude

Ana's MCP endpoint today serves **one** access pattern: an interactive human, signed in through Claude, holding a **delegated** token whose client-scoping is derived **server-side from that user's `UserClient` assignments**.

ANDI breaks every assumption in that sentence:

| Assumption in the Claude flow | Reality for ANDI |
|---|---|
| A **human** drives each call, interactively | The **workshop AI runs autonomously** in ANDI's backend — no human in the loop at query time |
| The caller is an **Ana user** with `UserClient` rows | Workshop participants are **not Ana users**; they have no `UserClient` assignments |
| Scoping is **user → client** | Scoping is **workshop → client** — a domain concept Ana has never heard of |
| Token carries `scp = Mcp.invoke` (delegated) | ANDI has no user token at all |

**Decisions taken:**

1. **Same tenant.** ANDI's backend lives in the Altus Nova Entra tenant (`e99365e9-…`); its own workshop participants authenticate to ANDI and never reach Ana.
2. **App-only.** ANDI's backend acts on its **own service identity** — no user identity flows to Ana.
3. **Ana enforces hard isolation, per workshop, via a token Ana issues.** For each workshop, ANDI asks Ana to **mint a workshop token bound to exactly one Altus client**. That token is what the workshop AI uses to call `/mcp`. Because the client is baked into the token **at mint time by Ana**, the workshop AI **cannot** reach any other client — there is no `clientId` for it to tamper with.

That third decision is the heart of this document. It is stronger than "ANDI passes a `clientId` and Ana checks it against an allow-list": here the workshop-runtime credential **has no ability to name a client at all**. The client is a property of the token, set by Ana.

---

## 2. The two layers

The design splits into a privileged **control plane** and a scoped **data plane**:

```
        CONTROL PLANE (mint, rare, privileged)          DATA PLANE (invoke, frequent, scoped)
        ─────────────────────────────────────           ────────────────────────────────────
ANDI backend ──Entra app token (WorkshopToken.Mint)──▶  workshop AI ──Ana workshop token──▶ /mcp
             "mint a token for clientId=X"                          bound to exactly clientId=X
             Ana checks X ∈ AppClient(ANDI)                         Ana pins clientId from the token
             Ana returns ana_ws_… (single-use record)              no clientId parameter is trusted
```

- **Control plane** — ANDI's backend, authenticated with its **Entra app-only identity**, calls a new Ana endpoint to mint a workshop token for a given client. Ana bounds *which* clients ANDI may mint for via a server-side allow-list (`AppClient`, §5.3). This is where the strong Entra identity is spent — rarely, per workshop.
- **Data plane** — the workshop AI holds only the **Ana-issued `ana_ws_…` token**, which is bound to one client. Every `/mcp` call it makes is pinned to that client by Ana. This credential is cheap, single-client, expirable, and revocable when the workshop ends.

Splitting them means the credential that lives inside the workshop runtime (the most exposed place) is the **least** powerful one: it can touch one client and nothing else, and it dies with the workshop.

---

## 3. The actors

| Actor | Identity | Role |
|---|---|---|
| **ANDI backend (minter)** | new confidential client app reg — *"Ana MCP - ANDI Service"* | Control-plane client. Holds a secret/cert, calls the **mint** endpoint on its Entra app identity. |
| **ANDI service principal** | the SP of that app reg in tenant `e99365e9-…` (`oid`/`appid`) | The subject Ana authorizes **to mint**. The `AppClient` allow-list is keyed on this. |
| **Workshop token** | `ana_ws_…`, a record in Ana's DB | Data-plane credential. **Bound to one `clientId`.** Ana issues, validates, expires, and revokes it. |
| **Workshop AI** | runs in ANDI's backend, holds a workshop token | Calls `/mcp` tools (`list_clients` / `search_documents` / `fetch_document`) for one workshop. |
| **MCP server** | `/mcp` on the Ana API | OAuth 2.0 protected resource. Same endpoint Claude uses; now also accepts workshop tokens. |
| **Resource app reg** | `AiAssistantApi` — `4279cecc-…` | Gains an app role `WorkshopToken.Mint` that guards the mint endpoint. |
| **Authorization server** | Entra ID v2.0, tenant `e99365e9-…` | Issues the **control-plane** app token only. It has nothing to do with workshop tokens. |

---

## 4. The end-to-end flow

### 4.1 Control plane — mint a workshop token (once per workshop)

```
ANDI backend                    Entra ID                        Ana API
     │                             │                               │
     │ 1. client-credentials: scope=api://4279cecc-…/.default      │
     ├────────────────────────────▶                               │
     │ 2. app token, roles=["WorkshopToken.Mint"], oid=ANDI SP     │
     ◀────────────────────────────┤                               │
     │ 3. POST /mcp/workshop-tokens  Authorization: Bearer <app token>
     │    { "clientId": X, "workshopRef": "...", "ttl": "8h" }      │
     ├────────────────────────────────────────────────────────────▶
     │ 4. Ana: policy requires WorkshopToken.Mint role             │
     │        Ana: X ∈ AppClient(ANDI SP)?  no → 403               │
     │        Ana: create WorkshopToken{ hash, clientId=X, exp }   │
     │ 5. { "token": "ana_ws_…", "expiresAt": "…" }  (raw token, once)
     ◀────────────────────────────────────────────────────────────┤
```

ANDI stores `ana_ws_…` against the workshop and hands it to that workshop's AI runtime.

### 4.2 Data plane — the workshop AI calls MCP (many times)

```
Workshop AI (in ANDI)                                  Ana /mcp
     │  MCP call, Authorization: Bearer ana_ws_…           │
     ├─────────────────────────────────────────────────────▶
     │  Ana: recognize workshop-token scheme (ana_ws_ prefix)
     │  Ana: look up hash → { clientId=X, not expired, not revoked }
     │  Ana: build principal carrying clientId=X claim      │
     │  McpInvoke policy: workshop-token principal → allow  │
     │  ClientScopeResolver: clientId is PINNED to X        │
     │       (a clientId arg, if sent, must equal X, else 403)
     │  → search/fetch run against client X only            │
     ◀─────────────────────────────────────────────────────┤
```

The workshop AI never presents an Entra token and never chooses a client. It could try to pass `clientId=Y` to `search_documents` — Ana ignores/rejects it in favor of the token's bound client. **There is no code path by which that runtime reaches another client.**

---

## 5. What Ana must build

Five additive pieces; none touches the existing delegated (Claude/FE) path.

### 5.1 An app role `WorkshopToken.Mint` on `AiAssistantApi`

Add an app role with `allowedMemberTypes: ["Application"]` — `WorkshopToken.Mint` — on `4279cecc-…`. It guards **only** the mint endpoint. ANDI's SP is granted it (admin consent). Note this role is *not* `Mcp.invoke`: ANDI's Entra identity may **mint**, but it may not itself **invoke** `/mcp` — only workshop tokens do that.

### 5.2 A mint (and revoke) endpoint

- `POST /mcp/workshop-tokens` — guarded by a policy requiring the `WorkshopToken.Mint` role. Body `{ clientId, workshopRef, ttl? }`. Validates `clientId ∈ AppClient(callerSpObjectId)`, creates a `WorkshopToken` record, returns the raw token **once**.
- `POST /mcp/workshop-tokens/{id}/revoke` (or `DELETE`) — same guard; marks the record revoked. ANDI calls this when a workshop ends.

These live under `/mcp/*` but, like `/mcp/status` and `/mcp/revoke` today, are **excluded from `McpChallengeMiddleware`** — they use plain Entra JWT, not the OAuth-resource challenge flow ([McpChallengeMiddleware.cs](../altus-ai-assistant/AltusAiAssistantApi/Middleware/McpChallengeMiddleware.cs)).

### 5.3 An `AppClient` allow-list — the mint-time bound

The machine-caller equivalent of `UserClient`; it caps which clients ANDI may mint tokens for:

| Column | Meaning |
|---|---|
| `ServicePrincipalObjectId` (or `AppId`) | which minter (ANDI's SP) |
| `ClientId` | an Altus client it may mint tokens for |
| audit fields | per Ana's `BaseEntity` convention |

Admin-provisioned out-of-band, exactly like `UserClient` rows are seeded — never self-serve by ANDI.

### 5.4 A `WorkshopToken` entity + storage

| Column | Meaning |
|---|---|
| `Id` | opaque record id (used for revoke) |
| `TokenHash` | **hash** of the raw token — store the hash, never the token (API-key hygiene) |
| `ClientId` | the single Altus client this token is bound to |
| `IssuedToAppId` | which minter created it (audit) |
| `WorkshopRef` | ANDI's workshop identifier (audit / bulk-revoke) |
| `ExpiresAt`, `RevokedAt` | lifecycle |

Token format: high-entropy random with a recognizable prefix, e.g. `ana_ws_<random>`. **Opaque reference token, not a JWT** — chosen so a workshop ending can revoke it instantly by DB flag; a signed JWT would need a denylist to revoke early. (JWT is the alternative if statelessness ever outweighs easy revocation.)

### 5.5 A `WorkshopToken` authentication scheme on `/mcp`

`/mcp` currently authenticates via Entra JWT only (`AddMicrosoftIdentityWebApi`, [Program.cs](../altus-ai-assistant/AltusAiAssistantApi/Program.cs)). Add a second **authentication scheme** that:

1. recognizes `Authorization: Bearer ana_ws_…` by prefix (a policy-scheme selector forwards `ana_ws_`-prefixed bearers to this handler, everything else to JwtBearer),
2. hashes and looks up the token; rejects if missing/expired/revoked,
3. builds a `ClaimsPrincipal` carrying a `clientId` claim = the bound client.

Then the two consuming pieces branch on caller type:

- **`McpInvoke` policy** ([Program.cs:45-75](../altus-ai-assistant/AltusAiAssistantApi/Program.cs)) — today it only inspects `scp` for `Mcp.invoke`. Extend it to **also** allow a principal authenticated by the workshop-token scheme. (Delegated Claude path unchanged.)
- **`ClientScopeResolver`** ([ClientScopeResolver.cs](../altus-ai-assistant/AltusAiAssistantApi/Mcp/ClientScopeResolver.cs)) — today it reads the user's `oid`, looks up `UserClient`s, and validates the requested `clientId` ∈ that set. Add a branch: if the principal is a workshop token, **pin the `clientId` from its claim**; if the tool call also carried a `clientId` argument, require it to equal the bound client, else `ForbiddenAccessException`. `list_clients` returns just that one client. The tools already accept a `clientId` parameter ([McpService.cs:89,175](../altus-ai-assistant/AltusAiAssistantApi/Services/McpService.cs)) — that parameter simply stops being a *trusted input* for workshop callers.

> As-is, a service principal's `oid` doesn't resolve to a user, so the current resolver already **fails closed** for any non-user caller — the new branch is purely additive and there's no risk of an app caller accidentally inheriting user scope.

---

## 6. New Entra app registration

A fifth client alongside the four in [entra-app-registrations.md](entra-app-registrations.md):

| Field | Value |
|---|---|
| Display name | `Ana MCP - ANDI Service` |
| Type | **Confidential client** (has a credential; no redirect URI — no user flow) |
| Credential | **Federated credential or certificate preferred** over a shared secret; if a secret, store in Key Vault and rotate |
| API permission | `AiAssistantApi` → **`WorkshopToken.Mint`** (Application permission) — **admin consent required** |
| Grant used | `client_credentials`, scope `api://4279cecc-…/.default` |

No redirect URIs, no delegated permissions, no Graph `User.Read`, and deliberately **no `Mcp.invoke`** — this identity mints; it does not invoke.

---

## 7. Where the isolation boundary sits — and why this is strong

This design makes the **workshop → single client** boundary something **Ana enforces on every data-plane call**, not something ANDI is trusted to respect:

- **Runtime credential is minimal.** The workshop AI holds a token that *is* a client binding. It cannot express "a different client." The most-exposed credential is the least-privileged one.
- **Mint is bounded and audited.** ANDI can only mint for clients in its `AppClient` allow-list; every mint is an authenticated, logged control-plane action tied to a `workshopRef`.
- **Blast radius is one client and one workshop.** A leaked `ana_ws_…` reaches one client until it expires or is revoked — not ANDI's whole client set.

**Residual trust (be honest):** ANDI's *minter* identity can mint for **any** client in its allow-list, so a compromised ANDI backend could mint tokens across that whole set. That's inherent — someone has to be trusted to say "this workshop is for client X." We shrink that surface by (a) keeping the `AppClient` allow-list tight, (b) short workshop-token TTLs, (c) treating the mint credential as high-value (federated/cert, Key Vault), and (d) auditing mints. If even that is too broad, the escalation is **one minter identity per Altus client** (§9) — but that reintroduces credential sprawl and is not recommended unless a shared minter is judged unacceptable.

---

## 8. Token lifecycle

- **Issue** — at workshop start (or first AI turn). ANDI mints with a TTL matched to the workshop (hours, not days).
- **Use** — the workshop AI calls `/mcp` with the token for the workshop's duration.
- **Expire** — `ExpiresAt` passes → Ana rejects; ANDI re-mints if the workshop runs long.
- **Revoke** — workshop ends, or a token is suspected leaked → ANDI calls revoke, or an admin bulk-revokes by `WorkshopRef`. Because tokens are DB-backed reference tokens, revocation is immediate (no denylist propagation).

---

## 9. Failure modes & how to read them

| Symptom | Likely cause | Where to look |
|---|---|---|
| `403` on mint | ANDI SP lacks `WorkshopToken.Mint` app role / no admin consent | Entra → ANDI reg → API permissions (§6) |
| `403` on mint for a specific `clientId` | No `AppClient` row for (ANDI SP, that client) | Provisioning gap (§5.3) — expected, not a bug |
| `401` on `/mcp` with `ana_ws_…` | Workshop-token scheme not wired, or token expired/revoked | §5.5; check `ExpiresAt`/`RevokedAt` |
| Workshop AI reaches the wrong client | Should be impossible — investigate immediately | Resolver not pinning `clientId` from the token (§5.5) |
| `403` when the AI passes a `clientId` arg | Arg ≠ the token's bound client (working as intended) | Have the AI omit `clientId`; the token pins it |
| Mint token `aud` invalid | `accessTokenAcceptedVersion` ≠ 2 on `AiAssistantApi` | Same knob as the Claude flow ([companion §4.3](mcp-claude-oauth-entra.md)) |

---

## 10. Why not the alternatives

- **ANDI passes `clientId` on its own app-only token, Ana checks an allow-list.** Simpler, but the runtime credential can name *any* client in ANDI's allow-list — the single-client boundary is only as good as ANDI's correctness. This design deliberately removes `clientId` from the runtime's control by baking it into an Ana-issued token.
- **On-behalf-of / delegated per participant.** Participants aren't Ana users and have no `UserClient`s, and the scoping we need is *workshop → client*, not *user → client*. OBO solves a problem we don't have.
- **One Entra app reg per Altus client.** True per-client hard isolation, but N credentials to manage. The workshop-token model gets the same runtime guarantee (a credential that can touch one client) without minting N Entra identities — Ana issues the per-client credential itself.
- **A plain REST endpoint instead of MCP.** Viable if ANDI only needs retrieval, but ANDI's workshop facilitation is an **LLM agent** that benefits from the existing tool contract, and reusing `/mcp` keeps one data-plane surface.

---

## 11. Open questions

- **Mint trigger & re-mint.** Does ANDI mint eagerly at workshop start, or lazily on first AI turn? How are long workshops (past TTL) re-minted — silent refresh or explicit?
- **One minter or per-client minters?** Drives the §7 residual-trust trade-off. Depends on how many clients ANDI serves and the acceptable blast radius.
- **Credential type for the minter.** Federated identity credential (no secret to rotate) if ANDI's backend runs on Azure with a managed identity; else certificate; secret last.
- **Provisioning UX for `AppClient`.** Admin screen or migration/script to start (mirrors however `UserClient` is seeded).
- **Telemetry.** `McpTelemetry` logs `userObjectId` / `userPrincipalName`. For workshop callers, log `workshopRef` + bound `clientId` + minter `appId` instead; confirm that satisfies audit.
- **Rate/size caps.** An autonomous agent calls far harder than a human — do the `McpSettings` caps need workshop-caller-specific limits?

---

## 12. References

- **Companion:** [mcp-claude-oauth-entra.md](mcp-claude-oauth-entra.md) (delegated flow), [entra-app-registrations.md](entra-app-registrations.md) (registrations).
- **Code to change:** [Program.cs](../altus-ai-assistant/AltusAiAssistantApi/Program.cs) (`McpInvoke` policy, second auth scheme + policy-scheme selector, mint-endpoint authz), [ClientScopeResolver.cs](../altus-ai-assistant/AltusAiAssistantApi/Mcp/ClientScopeResolver.cs) (workshop-token branch), [McpService.cs](../altus-ai-assistant/AltusAiAssistantApi/Services/McpService.cs) + tools under [Mcp/Tools/](../altus-ai-assistant/AltusAiAssistantApi/Mcp/Tools/) (`list_clients` pinned to one client), [McpChallengeMiddleware.cs](../altus-ai-assistant/AltusAiAssistantApi/Middleware/McpChallengeMiddleware.cs) (exclude the mint endpoints), new `AppClient` + `WorkshopToken` entities + EF configs under [Data/](../altus-ai-assistant/AltusAiAssistantApi/Data/).
- **Specs:** OAuth 2.0 Client Credentials Grant (RFC 6749 §4.4); Microsoft identity platform — app roles / application permissions, `.default` scope, client-credentials with certificates & federated credentials, access-token versions (`accessTokenAcceptedVersion`). Workshop tokens are an **Ana-proprietary** reference-token scheme, not an OAuth artifact.
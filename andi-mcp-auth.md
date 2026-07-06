# Ana MCP: User-Issued, Client-Scoped Access Tokens (for ANDI & other third-party integrations)

> **Scope:** how a third-party application — **ANDI** (a separate AI workshop-facilitation app) or any future integration — gets a credential to call Ana's `/mcp` endpoint, bound to a **specific set of Altus clients** chosen at issuance (often just one, but not limited to one), **without any Entra setup of its own**. An Ana user who already has access to those clients generates the token from the Ana UI and hands it to the consuming application.
> **Companion docs:** [mcp-claude-oauth-entra.md](mcp-claude-oauth-entra.md) (the per-user delegated flow Claude uses — token issuance below reuses this same user identity) and [entra-app-registrations.md](entra-app-registrations.md) (the registrations that front `/mcp`; this design adds none).
> **Status:** design / proposal. Supersedes an earlier draft where ANDI's backend minted its own tokens via a dedicated Entra app registration — that approach is kept below only as a rejected alternative (§9). Nothing below is built yet.

---

## 1. The core idea

ANDI's workshop AI runs autonomously in ANDI's backend — no human in the loop at query time — and needs to touch a bounded set of Altus clients per workshop, never more than it's explicitly granted. Usually that's a single client, but the mechanism shouldn't have to be redesigned the day a workshop (or another integration) legitimately spans a handful of clients. The earlier draft solved single-client scoping by having ANDI's backend mint its own scoped token via a new Entra app registration and an admin-provisioned allow-list. That works, but it means **every new third-party integration needs its own Entra app registration, app role grant, and allow-list rows** before it can receive a single token.

Simpler framing: **the client-scoping decision doesn't need a new authorization mechanism — Ana already has one.** Every Ana user who can see a client's data interactively already has a `UserClient` row saying so. If that same user generates the token, through the Ana UI, using the session they already have, then:

- Issuance is authorized by something that already exists (`UserClient`) — no new allow-list to provision or maintain.
- The consuming application (ANDI, or anyone else) never touches Entra at all. It receives an opaque bearer string and uses it like an API key.
- Onboarding a new third-party integration is a UI action, not an Entra admin-consent workflow.

The data-plane guarantee is unchanged from the original design: **the token itself is the client binding.** Whoever holds it cannot express access to a client outside the set baked into it — a `clientId` parameter, if present, is checked for membership in that set, not treated as free-form input.

---

## 2. The two layers

```
        CONTROL PLANE (issue, rare, human-driven)          DATA PLANE (invoke, frequent, scoped)
        ───────────────────────────────────────            ────────────────────────────────────
Ana user ──existing session/user auth──▶ Ana UI/API       ANDI (or any 3rd party) ──ana_cat_…──▶ /mcp
          "generate a token for clientIds={X,Y,…}"                    bound to that fixed client set
          Ana checks {X,Y,…} ⊆ UserClient(user)                       Ana restricts clientId to the set
          Ana returns ana_cat_… (shown once)                          no clientId outside the set is trusted
                     │
                     └─ user copies the token, pastes it into ANDI's (or another app's) configuration
```

- **Control plane** — an Ana user, signed in through the Ana web app exactly as they already are today, opens a "generate token" screen, picks one of the clients they have access to, and gets back a raw token once. This is not a new authentication surface: it's the same delegated user identity described in [mcp-claude-oauth-entra.md](mcp-claude-oauth-entra.md), authorizing against the same `UserClient` table that already scopes that user's own interactive access.
- **Data plane** — unchanged from the original design. The consuming application holds only the `ana_cat_…` token, bound to one client, cheap, single-client, expirable, and revocable at any time.

Splitting them still means the credential living inside the most-exposed place (a third party's backend) is the least powerful one. What's different is *how it gets there*: a human hands it over deliberately, rather than a service identity minting it on demand.

---

## 3. The actors

| Actor | Identity | Role |
|---|---|---|
| **Ana user (issuer)** | an existing Ana user, signed in via the normal delegated flow | Generates a token for one of the clients they already have a `UserClient` row for, via the Ana UI. |
| **Ana UI / API (issuance)** | existing Ana web app + API | Hosts the "generate token" feature. Authorizes issuance against the caller's own `UserClient` rows — no new allow-list, no new authorization server. |
| **Client access token** | `ana_cat_…`, a record in Ana's DB | Data-plane credential. **Bound to one or more `clientId`s**, fixed at issuance. Ana issues, validates, expires, and revokes it. |
| **Consuming application** | ANDI's backend, or any other third-party service the user chooses to hand the token to | Holds the raw token as a plain configuration value. Calls `/mcp` tools (`list_clients` / `search_documents` / `fetch_document`) for that one client. **No Entra identity, no app registration, no admin consent required.** |
| **MCP server** | `/mcp` on the Ana API | OAuth 2.0 protected resource. Same endpoint Claude uses; now also accepts client access tokens. |

Notably absent compared to the earlier draft: no dedicated Entra app registration for the consumer, no new app role, no service-principal allow-list. Issuance authorization piggybacks entirely on the user auth and `UserClient` data Ana already has.

---

## 4. The end-to-end flow

### 4.1 Control plane — issue a token (once per integration, via the Ana UI)

```
Ana user (signed in)              Ana UI                              Ana API
     │                               │                                    │
     │ 1. Open "Integrations" / "API tokens", pick one or more clients they can access
     ├──────────────────────────────▶│                                    │
     │                               │ 2. POST /api/client-tokens   Authorization: <existing user session>
     │                               │    { "clientIds": [X, Y], "label": "ANDI workshops", "ttl": "8h" }
     │                               ├───────────────────────────────────▶
     │                               │ 3. Ana: {X, Y} ⊆ UserClient(caller)?  no → 403
     │                               │        Ana: create ClientAccessToken{ hash, exp } + one
     │                               │        ClientAccessTokenClient row per id in {X, Y}
     │                               │ 4. { "token": "ana_cat_…", "expiresAt": "…" }  (raw token, shown once)
     │                               ◀───────────────────────────────────┤
     │ 5. UI displays the token once; user copies it                     │
     ◀──────────────────────────────┤                                    │
     │ 6. user pastes the token into ANDI's (or another app's) config     │
```

This is an ordinary, already-authenticated Ana API call — not a new Entra client-credentials grant. It lives outside `/mcp/*` (e.g. `/api/client-tokens`) and is not touched by `McpChallengeMiddleware`.

### 4.2 Data plane — the consuming app calls MCP (many times)

```
Consuming app (e.g. ANDI's workshop AI)                Ana /mcp
     │  MCP call, Authorization: Bearer ana_cat_…          │
     ├─────────────────────────────────────────────────────▶
     │  Ana: recognize client-access-token scheme (ana_cat_ prefix)
     │  Ana: look up hash → { clientIds={X,Y,…}, not expired, not revoked }
     │  Ana: build principal carrying one clientId claim per bound client
     │  McpInvoke policy: token principal → allow            │
     │  ClientScopeResolver: clientId is RESTRICTED to the bound set
     │       (a clientId arg, if sent, must be a member, else 403;
     │        if omitted and the set has exactly one client, default to it)
     │  → search/fetch run against the resolved client only  │
     ◀─────────────────────────────────────────────────────┤
```

The consuming app never presents a user or Entra credential and never chooses a client outside what the token allows. It could try to pass `clientId=Z` to `search_documents` — Ana rejects it unless `Z` is a member of the token's bound set. **There is no code path by which it reaches a client outside that set.**

---

## 5. What Ana must build

Four additive pieces; none touches the existing delegated (Claude/FE) path, and — unlike the earlier draft — **none requires a new Entra app registration or app role.**

### 5.1 A `ClientAccessToken` entity + a client join table

| Column | Meaning |
|---|---|
| `Id` | opaque record id (used for revoke) |
| `TokenHash` | **hash** of the raw token — store the hash, never the token (API-key hygiene) |
| `CreatedByUserId` | the Ana user (`oid`) who generated it — audit, and the basis for any future cascade-revoke |
| `Label` | free-text, user-supplied, e.g. `"ANDI workshops - Acme Q3"` |
| `ExpiresAt`, `RevokedAt` | lifecycle |

The clients a token is bound to live in a separate join table, deliberately mirroring `UserClient`:

`ClientAccessTokenClient` — `(TokenId, ClientId)` composite key, one row per client the token grants access to. A token scoped to a single client is just a table with one row; nothing about the single-client case is special-cased in the schema.

Token format: high-entropy random with a recognizable prefix, `ana_cat_<random>`. Opaque reference token, not a JWT — same reasoning as before: a DB flag revokes it instantly, no denylist needed.

### 5.2 An issuance UI + API in the existing Ana web app

- A new "Integrations" / "API tokens" screen — an ordinary authenticated page, no new sign-in flow. The client picker is multi-select.
- `POST /api/client-tokens` — body `{ clientIds: [...], label?, ttl? }`. Validates every id in `clientIds` is a member of `UserClient(callerUserId)`, creates a `ClientAccessToken` record plus one `ClientAccessTokenClient` row per id, returns the raw token **once**.
- `GET /api/client-tokens` — lists the caller's own tokens (id, bound clients, label, createdAt, expiresAt, revoked) for management. Never returns the raw value again.
- `POST /api/client-tokens/{id}/revoke` — revokes a token the caller created.

These are regular Ana API endpoints authenticated the same way as the rest of the Ana FE — not under `/mcp/*`, not guarded by any new Entra role.

### 5.3 A `ClientAccessToken` authentication scheme on `/mcp`

`/mcp` currently authenticates via Entra JWT only (`AddMicrosoftIdentityWebApi`, [Program.cs](../altus-ai-assistant/AltusAiAssistantApi/Program.cs)). Add a second **authentication scheme** that:

1. recognizes `Authorization: Bearer ana_cat_…` by prefix (a policy-scheme selector forwards `ana_cat_`-prefixed bearers to this handler, everything else to JwtBearer),
2. hashes and looks up the token; rejects if missing/expired/revoked,
3. builds a `ClaimsPrincipal` carrying a `clientId` claim = the bound client.

### 5.4 Policy + resolver changes

- **`McpInvoke` policy** ([Program.cs:45-75](../altus-ai-assistant/AltusAiAssistantApi/Program.cs)) — today it only inspects `scp` for `Mcp.invoke`. Extend it to **also** allow a principal authenticated by the client-access-token scheme. (Delegated Claude path unchanged.)
- **`ClientScopeResolver`** ([ClientScopeResolver.cs](../altus-ai-assistant/AltusAiAssistantApi/Mcp/ClientScopeResolver.cs)) — today it reads the user's `oid`, looks up `UserClient`s, and validates the requested `clientId` ∈ that set. Add a branch: if the principal is a client access token, resolve its bound set from the `clientId` claims; if the tool call carried a `clientId` argument, require it ∈ that set, else `ForbiddenAccessException`; if omitted and the set has exactly one client, default to it (preserving today's single-client convenience); if omitted and the set has more than one, require the caller to specify. `list_clients` returns the token's full bound set, not just one.

> As before, a token principal's claims don't resolve to a real `UserClient` set, so the existing resolver already **fails closed** for it — the new branch is purely additive.

---

## 6. Where the isolation boundary sits — and why this is still strong

- **Runtime credential is minimal.** The consuming app holds a token that *is* a client binding. It cannot express "a different client." The most-exposed credential is the least-privileged one — unchanged from the original design.
- **Issuance is bounded by something that already exists.** A user cannot generate a token for a client they don't already have `UserClient` access to. There's nothing new to provision or keep in sync — it's the same table that already scopes their own interactive access.
- **Blast radius is one client**, until expiry or revocation.

**Residual trust (be honest):** any user with access to a client can hand a live credential for it to whatever external application they choose — Ana has no way to know or control which third parties end up holding one of its tokens, beyond trusting the issuing user's judgment. That's not a new class of privileged identity, though: it's exactly the trust already placed in that user for their own interactive access, not an escalation. It's also *narrower* than the rejected design's shared minter identity (§9), which could mint across its entire allow-list rather than a set of clients a specific human vouches for. If a client's data is sensitive enough that self-service issuance is unacceptable, the escalation path is an admin-approval gate on issuance for that specific client (§10).

Supporting multiple clients on one token widens the blast radius of a leak from "one client" to "however many clients the issuer chose to include" — up to their entire accessible set if they select all of them. Ana only enforces that every included client is already in the issuer's `UserClient` set; it doesn't enforce minimality beyond that. The UI should nudge toward narrow, deliberate selection (no one-click "select all") so the common case stays small (§10).

---

## 7. Token lifecycle

- **Issue** — a user clicks "generate token" in the Ana UI for one or more clients they can access, and hands the raw value to the consuming application out of band.
- **Use** — the consuming app calls `/mcp` with the token for as long as it's valid.
- **Expire** — `ExpiresAt` passes → Ana rejects; the user generates a new one.
- **Revoke** — the creating user revokes it from the UI, or a token is suspected leaked. Revocation is all-or-nothing for the token (not per-client); because tokens are DB-backed reference tokens, revocation is immediate (no denylist propagation).
- **Open**: if the creating user's own `UserClient` access to one of a token's bound clients is later removed, the existing token does **not** automatically shrink or revoke unless that cascade is explicitly built (§10).

---

## 8. Failure modes & how to read them

| Symptom | Likely cause | Where to look |
|---|---|---|
| `403` on generate | caller lacks a `UserClient` row for the requested `clientId` | expected, not a bug |
| `401` on `/mcp` with `ana_cat_…` | token scheme not wired, or token expired/revoked | §5.3; check `ExpiresAt`/`RevokedAt` |
| Consuming app reaches the wrong client | should be impossible — investigate immediately | resolver not pinning `clientId` from the token (§5.4) |
| `403` when the caller passes a `clientId` arg | arg ∉ the token's bound client set (working as intended) | have the caller omit `clientId` (if the token is single-client) or pass one that's in the set |
| Token still works after the issuing user leaves or loses client access | cascade-revoke not implemented | decide the policy explicitly (§10) — not a bug in the current design, but worth resolving before launch |

---

## 9. Why not the alternatives

- **The consuming app mints its own token via an Entra app registration + admin-provisioned allow-list** (the earlier draft of this doc). More conventional in one sense — an admin explicitly approves which app may reach which client — but it means standing up a new Entra app registration, app role grant, and allow-list table for *every* third-party integration before it can receive a single token, plus a shared minter identity whose blast radius spans its whole allow-list (worse than this design's per-issuance, per-user scoping). Reusing access a human already has removes an entire authorization surface without loosening anything: that user could already see the client's data interactively.
- **On-behalf-of / delegated per participant.** Workshop participants aren't Ana users and have no `UserClient`s, and the scoping needed is *integration → client*, not *participant → client*. OBO solves a problem we don't have.
- **One Entra app registration per Altus client.** True per-client hard isolation, but N Entra identities to manage — and now doubly unnecessary, since no app registration is needed for issuance at all under this design.
- **A single static shared secret per client**, handed out instead of individually generated tokens. Rejected: no per-issuance audit trail (who created it, for what) and no way to revoke one integration without breaking every other consumer of that client's secret.
- **Keep tokens strictly single-client; issue several tokens if an integration needs several clients.** Simpler schema (no join table), but pushes the cost onto every consuming app: it now has to manage, configure, and rotate N credentials for what is conceptually one integration. A multi-client token has the same blast radius as holding N single-client tokens at once, with none of the bookkeeping — so the join-table approach (§5.1) was chosen instead.
- **A plain REST endpoint instead of MCP.** Viable if a consumer only needs retrieval, but ANDI's workshop facilitation is an **LLM agent** that benefits from the existing tool contract, and reusing `/mcp` keeps one data-plane surface for every consumer, not just ANDI.

---

## 10. Open questions

- **Cascade revoke.** If the issuing user's own `UserClient` access to one of a token's bound clients is removed (they leave, or lose access), should that client be dropped from the token, should the whole token be revoked, or is this an accepted gap (as with e.g. GitHub PATs)?
- **Client-admin visibility.** Should someone other than a token's creator — a client owner/admin — be able to see and revoke tokens touching their client, in case the creator is unreachable or a token needs shutting down urgently?
- **Self-service opt-out.** Should certain clients be able to disable self-service token issuance entirely, falling back to an admin-approval step? This is the escalation path referenced in §6 if a client's data is judged too sensitive for any user with access to self-issue integration tokens.
- **Caps.** A max number of active tokens per user/client, a max number of clients per token, and a default/max TTL.
- **UI placement and defaults.** Standalone "Integrations" / "API tokens" page, or a tab within each client's own settings? And should the multi-select client picker default to none-selected with no "select all" shortcut, to keep the common token narrowly scoped (§6)?
- **Telemetry.** `McpTelemetry` logs `userObjectId` / `userPrincipalName` for the delegated path. For token callers, log `CreatedByUserId` + bound `clientId` + `label` instead; confirm that satisfies audit.
- **Rate/size caps.** An autonomous agent (e.g. ANDI's workshop AI) calls far harder than a human — do `McpSettings` caps need token-caller-specific limits?

---

## 11. References

- **Companion:** [mcp-claude-oauth-entra.md](mcp-claude-oauth-entra.md) (delegated flow — token issuance reuses this same user identity), [entra-app-registrations.md](entra-app-registrations.md) (registrations fronting `/mcp`; unchanged by this design).
- **Code to change:** [Program.cs](../altus-ai-assistant/AltusAiAssistantApi/Program.cs) (`McpInvoke` policy, second auth scheme + policy-scheme selector), [ClientScopeResolver.cs](../altus-ai-assistant/AltusAiAssistantApi/Mcp/ClientScopeResolver.cs) (client-access-token branch resolving a *set* of clients), [McpService.cs](../altus-ai-assistant/AltusAiAssistantApi/Services/McpService.cs) + tools under [Mcp/Tools/](../altus-ai-assistant/AltusAiAssistantApi/Mcp/Tools/) (`list_clients` returning the token's bound set), new `/api/client-tokens` controller (issuance, list, revoke — standard Ana API auth, not under `/mcp/*`), new `ClientAccessToken` entity + `ClientAccessTokenClient` join table + EF configs under [Data/](../altus-ai-assistant/AltusAiAssistantApi/Data/).
- **Specs:** none new on the issuance side — it rides the existing delegated user auth. Client access tokens are an **Ana-proprietary** reference-token scheme, not an OAuth artifact, same as the earlier draft's workshop tokens.
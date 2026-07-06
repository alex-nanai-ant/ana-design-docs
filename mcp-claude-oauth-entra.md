# How Claude's MCP OAuth Works Against Entra ID

> **Scope:** the remote MCP server on the primary backend ([altus-ai-assistant](../altus-ai-assistant/)), branch `feat/mcp`, and the Entra ID app registrations that front it.
> **Companion doc:** [mcp-branch-maf-migration.md](mcp-branch-maf-migration.md) — what the branch changes and how it lands on the MAF flow.
> **Status:** reference / explainer. Describes the OAuth design as configured in QA (tenant `e99365e9-…`).

This document explains, end to end, how an MCP client like Claude authenticates to Ana's `/mcp` endpoint, why Entra ID makes that flow non-standard, and exactly which app-registration and config values make it work.

---

## 1. TL;DR — the core tension

The MCP authorization spec layers on top of OAuth 2.1 and assumes a **zero-configuration** client bootstrap: the client discovers the server's auth metadata (RFC 9728 + RFC 8414) and then **dynamically registers itself** (RFC 7591) to obtain a `client_id` on the fly.

**Microsoft Entra ID does not implement open Dynamic Client Registration.** So the zero-config path dead-ends. The workaround — and what is already configured in this tenant — is to **pre-register a client app** (`Ana MCP - Claude Client`) and hand its `client_id` (plus a secret, if it's a confidential client) to Claude manually via the connector's advanced OAuth settings. Everything else (discovery, the auth-code+PKCE exchange, the bearer-token call) is standard.

```
MCP spec default:   discover → DCR (auto client_id) → auth-code+PKCE → call
With Entra ID:      discover → [DCR unavailable] → use PRE-REGISTERED client_id → auth-code+PKCE → call
```

---

## 2. The actors

| Actor | Identity | Role |
|---|---|---|
| **MCP server** | `/mcp` on the Ana API | OAuth 2.0 **protected resource** (RFC 9728). Validates bearer tokens, exposes tools. |
| **Resource / API app reg** | `AiAssistantApi` — appId `4279cecc-…`, App ID URI `https://ana-api-qa.engineering.altusnova.com/mcp` | Defines the **`Mcp.invoke`** delegated scope ("Use Ana via Claude"). The token's audience. |
| **Client app reg** | `Ana MCP - Claude Client` | The OAuth **client** Claude authenticates as. Holds delegated permission to `Mcp.invoke` + Graph `User.Read`. |
| **Authorization server** | Entra ID v2.0 endpoint for tenant `e99365e9-…` | Issues tokens. `https://login.microsoftonline.com/e99365e9-…/v2.0`. |
| **Claude** | Claude.ai / Claude Desktop custom connector, or Claude Code | The MCP client driving the flow. |

Two **separate** app registrations matter here — a resource and a client. Conflating them is the most common point of confusion.

---

## 3. The end-to-end flow

```
Claude (MCP client)            Ana /mcp (resource)        Entra ID (auth server)
      │                              │                            │
      │ 1. POST /mcp (no token)      │                            │
      ├─────────────────────────────▶                            │
      │ 2. 401 + WWW-Authenticate:   │                            │
      │    Bearer resource_metadata= │                            │
      │    ".../.well-known/oauth-   │                            │
      │     protected-resource"      │                            │
      ◀─────────────────────────────┤                            │
      │ 3. GET that discovery URL    │                            │
      ├─────────────────────────────▶                            │
      │ 4. { resource,               │                            │
      │      authorization_servers,  │                            │
      │      scopes_supported }      │                            │
      ◀─────────────────────────────┤                            │
      │ 5. GET AS metadata (openid-configuration)                 │
      ├──────────────────────────────────────────────────────────▶
      │ 6. { authorization_endpoint, token_endpoint, … }          │
      ◀──────────────────────────────────────────────────────────┤
      │ 7. [spec: dynamic client registration] ── NOT SUPPORTED by Entra
      │    → instead use the PRE-REGISTERED client_id (+ secret)   │
      │ 8. Authorization-code + PKCE: browser → sign-in → consent  │
      ├──────────────────────────────────────────────────────────▶
      │    ("Use Ana via Claude")                                  │
      │ 9. authorization code → token_endpoint → access_token      │
      ◀──────────────────────────────────────────────────────────┤
      │ 10. POST /mcp  Authorization: Bearer <access_token>        │
      ├─────────────────────────────▶                            │
      │ 11. server validates aud/iss/scope, runs the tool          │
      ◀─────────────────────────────┤                            │
```

### Step-by-step

1. **Unauthenticated probe.** Claude calls `/mcp` with no token.
2. **Challenge.** The server returns `401` with `WWW-Authenticate: Bearer resource_metadata="<discovery URL>"`. This is added by [McpChallengeMiddleware.cs](../altus-ai-assistant/AltusAiAssistantApi/Middleware/McpChallengeMiddleware.cs) for `/mcp*` paths (deliberately **excluding** `/mcp/status` and `/mcp/revoke`, which use plain JWT, not this OAuth-resource flow).
3–4. **Protected-resource discovery (RFC 9728).** Claude GETs `/.well-known/oauth-protected-resource`, served by [McpDiscoveryController.cs](../altus-ai-assistant/AltusAiAssistantApi/Controllers/McpDiscoveryController.cs). It returns:
   - `resource` = `McpSettings:ResourceUri` (`https://ana-api-qa.engineering.altusnova.com/mcp`),
   - `authorization_servers` = `[McpSettings:AuthorizationServer]`,
   - `scopes_supported` = `["{McpSettings:AppIdUri}/{RequiredScope}"]`,
   - `bearer_methods_supported` = `["header"]`.
5–6. **Authorization-server discovery (RFC 8414).** Claude fetches Entra's metadata (`…/v2.0/.well-known/openid-configuration`) to learn the `authorization_endpoint` and `token_endpoint`.
7. **Client identity.** The spec says *register dynamically here*. Entra can't, so Claude uses the **pre-registered** `Ana MCP - Claude Client` `client_id` you supply.
8. **Authorization code + PKCE.** Browser opens Microsoft sign-in; the user consents to the `Mcp.invoke` scope. The consent prompt text is the scope's user-consent display name — **"Use Ana via Claude."** Because `Mcp.invoke` has *Admin consent required: No*, an ordinary user can consent; no tenant-admin grant is needed.
9. **Token.** Claude exchanges the code at the `token_endpoint` for an access token scoped to `Mcp.invoke`.
10–11. **Authenticated call.** Claude resends the request to `/mcp` with `Authorization: Bearer <token>`; the server validates it and runs the tool.

---

## 4. Where Entra diverges from the MCP spec

### 4.1 No Dynamic Client Registration (RFC 7591)
Entra ID exposes no open `registration_endpoint`. The fix is a **pre-registered client app** — `Ana MCP - Claude Client` — and entering its `client_id`/secret into Claude's connector advanced OAuth fields. This is why a second app registration exists at all.

### 4.2 The scope string must match exactly
Entra issues a token for the scope **`https://ana-api-qa.engineering.altusnova.com/mcp/Mcp.invoke`** — the resource's **App ID URI** + `/Mcp.invoke`. But the server advertises `scopes_supported` as `"{McpSettings:AppIdUri}/{RequiredScope}"`. The committed config sets `McpSettings:AppIdUri = api://4279cecc-…`, which would advertise `api://4279cecc-…/Mcp.invoke` — **a different string than the scope Entra actually knows**. Claude would request a scope Entra rejects.

**Resolution:** set `McpSettings__AppIdUri` on the QA App Service to `https://ana-api-qa.engineering.altusnova.com/mcp`, so discovery advertises the scope clients can actually request.

### 4.3 Token audience depends on `accessTokenAcceptedVersion`
The server pins accepted audiences in [Program.cs](../altus-ai-assistant/AltusAiAssistantApi/Program.cs) (`PostConfigure<JwtBearerOptions>`):

```
ValidAudiences = [ apiClientId (4279cecc-… GUID),  McpSettings:AppIdUri ]
ValidIssuers   = [ https://login.microsoftonline.com/{tenant}/v2.0,
                   https://sts.windows.net/{tenant}/ ]
```

The `aud` claim Entra emits depends on the API app's manifest:
- `accessTokenAcceptedVersion = 2` → `aud` = the resource **appId GUID** (`4279cecc-…`) → validates via `apiClientId`.
- `accessTokenAcceptedVersion = 1` / `null` → `aud` = the **App ID URI** (`https://ana-api-qa…/mcp`) → validates **only if** `McpSettings:AppIdUri` is set to that same https value.

Either way, aligning `McpSettings:AppIdUri` with the App ID URI (§4.2) is what keeps both the advertised scope **and** the v1-token audience consistent.

### 4.4 Scope is enforced by policy, not just authentication
Reaching `/mcp` requires the **`McpInvoke`** authorization policy ([Program.cs](../altus-ai-assistant/AltusAiAssistantApi/Program.cs)), which requires an authenticated user **and** a `Mcp.invoke` value in the `scp` (or legacy `…/identity/claims/scope`) claim. A valid Entra token *without* that scope still gets `403`.

---

## 5. The configured state (QA, tenant `e99365e9-…`)

**Resource app — `AiAssistantApi` (`4279cecc-…`)**
- App ID URI: `https://ana-api-qa.engineering.altusnova.com/mcp`
- Exposed scope **`Mcp.invoke`** — admin display "Invoke Ana MCP tools", user display "Use Ana via Claude", *Who can consent: Admins and users*, **Enabled**.
- (Also exposes `user-access`, used by the Ana FE SPA — unrelated to MCP.)

**Client app — `Ana MCP - Claude Client`**
- Delegated permission **`AiAssistantApi / Mcp.invoke`** (admin consent **not** required).
- Delegated **Microsoft Graph `User.Read`** (sign-in).
- Redirect URI determines the supported Claude surface (see §6).

**Server config — `McpSettings`** ([McpSettings.cs](../altus-ai-assistant/AltusAiAssistantApi/Models/McpSettings.cs))
- `ResourceUri`, `AuthorizationServer`, `RequiredScope = Mcp.invoke`, `AppIdUri` (see §4.2 caveat), plus result/size caps.

---

## 6. Connecting Claude

First confirm the deploy actually serves MCP (anonymous, no token):
- `GET …/mcp/health` → `{"status":"ok"}`
- `GET …/.well-known/oauth-protected-resource` → and verify `scopes_supported` is the **`https://…/mcp/Mcp.invoke`** form (§4.2).

Then pick the surface that matches the client app's **redirect URI**:

**Claude.ai / Claude Desktop** (redirect URI `https://claude.ai/api/mcp/auth_callback`):
1. Settings → **Connectors → Add custom connector**.
2. URL: `https://ana-api-qa.engineering.altusnova.com/mcp`.
3. In advanced OAuth, supply the `Ana MCP - Claude Client` **client ID** (and **secret** if it's a confidential/Web client; none if it's a SPA/public client using PKCE), the **authorization/token endpoints** from `…/v2.0/.well-known/openid-configuration`, and **scope** `https://ana-api-qa…/mcp/Mcp.invoke`.
4. Connect → Microsoft sign-in → consent ("Use Ana via Claude") → the tools (`list_clients`, `search_documents`, `fetch_document`) appear.

**Claude Code** (redirect URI is a `http://localhost:…` loopback):
```bash
claude mcp add --transport http ana-qa https://ana-api-qa.engineering.altusnova.com/mcp
# run the OAuth flow, then verify:
/mcp
```
Or skip OAuth entirely with a static token (handy for a quick functional check):
```bash
claude mcp add --transport http ana-qa https://ana-api-qa.engineering.altusnova.com/mcp \
  --header "Authorization: Bearer <MCP_INVOKE_TOKEN>"
```

---

## 7. Failure modes & how to read them

| Symptom | Likely cause | Where to look |
|---|---|---|
| Claude can't start OAuth / "registration failed" | Expecting DCR; no `client_id` provided | Supply the pre-registered client ID (§4.1) |
| Entra error on consent: invalid scope | Advertised scope ≠ real scope | `McpSettings__AppIdUri` mismatch (§4.2) |
| `401` *after* a successful sign-in | Token `aud` not accepted | `accessTokenAcceptedVersion` vs `ValidAudiences` (§4.3) |
| `403` with a valid token | Token lacks `Mcp.invoke` in `scp` | Client app's API permission / the scope requested (§4.4) |
| Redirect/callback mismatch in browser | Wrong Claude surface for the registered redirect URI | Client app **Authentication** blade (§6) |
| Tools list but return "no clients" | Auth is fine; the user has no assigned `UserClient`s | Not an auth problem — expected for an unprovisioned user |
| `/mcp/*` returns `404` | MCP build not deployed | Deploy `feat/mcp`; both QA hostnames are the same App Service |

---

## 8. References

- **Code:** [Program.cs](../altus-ai-assistant/AltusAiAssistantApi/Program.cs) (JWT pinning, `McpInvoke` policy, `MapMcp`), [McpDiscoveryController.cs](../altus-ai-assistant/AltusAiAssistantApi/Controllers/McpDiscoveryController.cs), [McpChallengeMiddleware.cs](../altus-ai-assistant/AltusAiAssistantApi/Middleware/McpChallengeMiddleware.cs), [McpSettings.cs](../altus-ai-assistant/AltusAiAssistantApi/Models/McpSettings.cs), tools under [Mcp/Tools/](../altus-ai-assistant/AltusAiAssistantApi/Mcp/Tools/).
- **Specs:** MCP Authorization; OAuth 2.0 Protected Resource Metadata (RFC 9728); Authorization Server Metadata (RFC 8414); Dynamic Client Registration (RFC 7591); Bearer Token Usage (RFC 6750); PKCE (RFC 7636).
- **Entra:** Microsoft identity platform — access token versions (`accessTokenAcceptedVersion`), exposing an API / scopes, and the documented absence of open dynamic client registration.

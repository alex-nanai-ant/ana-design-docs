# Entra App Registrations — Ana

Tenant ID: `e99365e9-089f-4355-a598-bd139993cd1e` (Altus Nova)

All registrations are **shared across QA and Prod** — there are no environment-specific app registration IDs. Environment differences are handled via `ResourceUri` / redirect URI configuration.

---

## Registrations

### 1. AiAssistantApi — Resource Server

| Field | Value |
|---|---|
| Client ID | `4279cecc-ad27-4e90-81be-af8e4c038897` |
| Application ID URI | `api://4279cecc-ad27-4e90-81be-af8e4c038897` |
| Role | JWT bearer resource — validates tokens for the API and the MCP endpoint |

**Exposed scopes:**

| Scope | Value | Scope `id` (GUID) | Consumers |
|---|---|---|---|
| `user-access` | `user-access` | `5ee826c9-46ab-441a-9ec7-52bb99bed16b` | Ana FE (MSAL login) |
| `Mcp.invoke` | `Mcp.invoke` | `3b3e9efa-0089-4847-91ac-3045e4bc3416` | Ana MCP - Claude Client |

The full scope string a client requests is `<App ID URI>/<value>` — e.g. `api://4279cecc-.../user-access` or `https://ana-api-qa.engineering.altusnova.com/mcp/user-access`. Both resolve to the same scope; the issued token's `scp` claim is just the value (`user-access`). Other registrations reference these scopes by the GUID `id`, not the string.

**Pre-authorized client applications** (`api.preAuthorizedApplications`) — a pre-authorized client skips the user consent prompt for the listed scopes:

| Client | Scope | Consent prompt? |
|---|---|---|
| Ana FE (`d341df90-...`) | `user-access` | **No** — pre-authorized, so FE users are never prompted |
| Ana MCP - Claude Client (`61e3c182-...`) | `Mcp.invoke` | **Yes** — *not* pre-authorized; Claude users hit the interactive consent screen (by design for the MCP OAuth flow) |

**App ID URIs registered** (required for Azure AD to accept `resource=` in token requests):

| Environment | URI | Registered? |
|---|---|---|
| All | `api://4279cecc-ad27-4e90-81be-af8e4c038897` | Yes ✓ |
| QA | `https://ana-api-qa.engineering.altusnova.com/mcp` | Yes ✓ |
| Prod | `https://ana-api.engineering.altusnova.com/mcp` (custom domain) | Yes ✓ |

Prod now uses a custom domain (`ana-api.engineering.altusnova.com`) on the App Service, mirroring the QA pattern, so its Application ID URI is a verified-domain subdomain instead of the `*.azurewebsites.net` hostname. See resolved issue below.

**Config reference:** `appsettings.json` → `AzureAdApi` section; `Program.cs` → `.AddMicrosoftIdentityWebApi(builder.Configuration.GetSection("AzureAdApi"))`

---

### 2. Ana FE — React Frontend MSAL Client

| Field | Value |
|---|---|
| Client ID | `d341df90-110e-45c5-9417-1681597e1dd3` |
| Application ID URI | None (client app, not a resource) |
| Role | Public client — MSAL popup/redirect login for the React SPA |

> **Shared with the voice frontend.** The `altus-ai-assistant-voice` frontend reuses this same registration and the same `user-access` scope (its Azure DevOps pipeline sets `VITE_MSAL_CLIENT_ID = d341df90-…`). That is why the two `azurestaticapps.net` hosts below — the voice FE's QA/prod Azure Static Web Apps — are registered here. There is no separate app registration for voice.

**Redirect URIs** (all SPA platform type). Each host is registered both bare and with the `/auth-redirect.html` suffix that MSAL redirects to:

| App | Environment | Bare URI | `/auth-redirect.html` variant |
|---|---|---|---|
| Primary FE | Development | `http://localhost:3000` | ✓ |
| Primary FE | QA (App Service) | `https://ai-assistant-fe-qa-ckcwc7ffb0fjbyb7.eastus2-01.azurewebsites.net` | ✓ |
| Primary FE | Prod (App Service) | `https://ai-assistant-fe-a5c0eba9awhnb7hn.eastus2-01.azurewebsites.net` | ✓ |
| Primary FE | Prod (custom domain) | `https://ana.engineering.altusnova.com` | — |
| Voice FE | Static Web App (QA) | `https://calm-sky-093bdb50f.1.azurestaticapps.net` | ✓ |
| Voice FE | Static Web App (Prod) | `https://victorious-dune-027c4fb0f.3.azurestaticapps.net` | ✓ |

Voice FE deploys to Static Web Apps `ai-assistant-voice-qa-fe` / `ai-assistant-voice-prod-fe` (see `altus-ai-assistant-voice/azure-pipelines-fe.yml`); `calm-sky` is QA, `victorious-dune` is prod.

**Requested scopes:** `api://4279cecc-ad27-4e90-81be-af8e4c038897/user-access`

**Config reference:** `altus-ai-assistant-fe/.env.*` → `VITE_MSAL_CLIENT_ID` / `VITE_MSAL_SCOPES` / `VITE_MSAL_REDIRECT_URI`

---

### 3. Ana MCP - Claude Client — MCP OAuth Client

| Field | Value |
|---|---|
| Client ID | `61e3c182-d699-4fae-90f2-3444860ec8fd` |
| Application ID URI | None (client app, not a resource) |
| Role | Confidential/public OAuth client used by Claude.ai and Claude Desktop to acquire tokens for the MCP endpoint |

**API permissions (delegated):**

| API | Scope | Purpose |
|---|---|---|
| AiAssistantApi | `Mcp.invoke` | Invoke Ana MCP tools |
| Microsoft Graph | `User.Read` | Sign in and read user profile |

**Redirect URIs:**

| Platform | URI | Used by |
|---|---|---|
| Web | `https://claude.ai/api/mcp/auth_callback` | Claude.ai remote MCP |
| Web | `https://claude.com/api/mcp/auth_callback` | Claude.ai remote MCP (alternate domain) |
| Mobile/desktop | `http://127.0.0.1/oauth/callback` | Claude Desktop / local MCP clients |
| Mobile/desktop | `http://localhost/oauth/callback` | Claude Desktop / local MCP clients |
| Mobile/desktop | `http://127.0.0.1/callback` | Additional local callback variant |
| Mobile/desktop | `http://localhost/callback` | Additional local callback variant |

**Config reference:** Not referenced in `appsettings.json` — this client ID is provided externally to MCP consumers (e.g., when configuring Claude.ai's remote MCP integration).

---
 
### 4. AzureAd — API Service Identity

| Field | Value |
|---|---|
| Client ID | `eed2ebe5-5083-41bf-b5d0-8c922faa6331` |
| Entra display name | `AiAssistantApiGraphClient` ("AzureAd" is only the `appsettings.json` section name) |
| Role | Daemon / service-to-service identity used by the API for outbound calls (e.g., OBO flows, Graph calls) |

**Config reference:** `appsettings.json` → `AzureAd` section

---

### 5. MailReadingApp — Mail Sync Daemon

| Field | Value |
|---|---|
| Client ID | `f81e2ce9-6a0b-47bb-be6b-7af5264254aa` |
| Entra display name | `AnaMailIngestionService` ("MailReadingApp" is only the `appsettings.json` section name) |
| Role | Confidential daemon client for reading Exchange mailboxes via Microsoft Graph (used by `AltusAiMailSync` Azure Function) |

**Config reference:** `appsettings.json` → `MailReadingApp` section

---

## MCP OAuth Flow

When Claude.ai or Claude Desktop connects to Ana's MCP endpoint, the OAuth sequence is:

```
MCP Client (Claude)
  │
  ├─ GET {mcp-server-url}/.well-known/oauth-protected-resource
  │     → resource: <McpSettings:ResourceUri>
  │     → authorization_servers: ["https://login.microsoftonline.com/{tenant}/v2.0"]
  │     → scopes_supported: ["api://4279cecc-.../Mcp.invoke"]
  │
  ├─ Redirect user to Azure AD authorize endpoint
  │     client_id = 61e3c182-...  (Ana MCP - Claude Client)
  │     scope     = api://4279cecc-.../Mcp.invoke
  │     redirect_uri = https://claude.ai/api/mcp/auth_callback  (or localhost variant)
  │
  ├─ Exchange auth code for token (PKCE)
  │     → Azure AD returns access token
  │     → token audience: api://4279cecc-...
  │     → token scp claim: Mcp.invoke
  │
  └─ Call MCP server with Bearer token
        → API validates via AzureAdApi registration (4279cecc-...)
        → McpInvoke policy checks scp == "Mcp.invoke"
```

**Implementation refs:**
- Discovery endpoint: [McpDiscoveryController.cs](../altus-ai-assistant/AltusAiAssistantApi/Controllers/McpDiscoveryController.cs)
- Token validation: `Program.cs` → `.AddMicrosoftIdentityWebApi` + `McpInvoke` authorization policy
- WWW-Authenticate challenge: [McpChallengeMiddleware.cs](../altus-ai-assistant/AltusAiAssistantApi/Middleware/McpChallengeMiddleware.cs)

---

## MCP ResourceUri per Environment

The `McpSettings:ResourceUri` is what is advertised as `resource` in the OAuth discovery document. Azure AD must have this URL registered as an Application ID URI on `AiAssistantApi` (`4279cecc-...`) for token requests using it as the `resource` parameter to succeed.

| Environment | ResourceUri | Registered on AiAssistantApi? |
|---|---|---|
| Development | `https://localhost:5001/mcp` | N/A (local only) |
| QA | `https://ana-api-qa.engineering.altusnova.com/mcp` | Yes ✓ |
| Prod | `https://ana-api.engineering.altusnova.com/mcp` | Yes ✓ |

**Config refs:** `appsettings.json` (base) + `appsettings.QA.json` / `appsettings.Prod.json` override `McpSettings:ResourceUri`.

---

## Resolved Issue — Prod MCP Now Working

**Former root cause:** The prod API hostname (`https://ai-assistant-api-prod-g7e7ewcab7ekb8bk.eastus2-01.azurewebsites.net`) was not registered as an Application ID URI on the `AiAssistantApi` registration. When MCP clients sent a token request using this URL as the `resource` parameter, Azure AD couldn't resolve it to any known app and rejected the request.

**Why the raw hostname couldn't be registered:** Entra rejects any `https://` Application ID URI that isn't on a **verified domain of the tenant** (or a subdomain). The verified domain here is `altusnova.com` (the app's `publisherDomain`). `*.azurewebsites.net` is Microsoft's shared domain, not one the org owns, so it could never be an identifier URI.

**The fix (applied):**
1. Configured a custom domain, `ana-api.engineering.altusnova.com`, on the prod App Service — mirrors the QA pattern (`ana-api-qa.engineering.altusnova.com`).
2. Registered `https://ana-api.engineering.altusnova.com/mcp` as an Application ID URI on `AiAssistantApi` → Expose an API.
3. Updated `appsettings.Prod.json` → `McpSettings:ResourceUri` to `https://ana-api.engineering.altusnova.com/mcp`.

Prod MCP OAuth now resolves correctly, consistent with QA.

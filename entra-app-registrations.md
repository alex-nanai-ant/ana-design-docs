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

| Scope | Full ID | Consumers |
|---|---|---|
| `user-access` | `api://4279cecc-.../user-access` | Ana FE (MSAL login) |
| `Mcp.invoke` | `api://4279cecc-.../Mcp.invoke` | Ana MCP - Claude Client |

**App ID URIs registered** (required for Azure AD to accept `resource=` in token requests):

| Environment | URI |
|---|---|
| All | `api://4279cecc-ad27-4e90-81be-af8e4c038897` |
| QA | `https://ana-api-qa.engineering.altusnova.com` |
| Prod | `https://ai-assistant-api-prod-g7e7ewcab7ekb8bk.eastus2-01.azurewebsites.net` ⚠️ see note below |

**Config reference:** `appsettings.json` → `AzureAdApi` section; `Program.cs` → `.AddMicrosoftIdentityWebApi(builder.Configuration.GetSection("AzureAdApi"))`

---

### 2. Ana FE — React Frontend MSAL Client

| Field | Value |
|---|---|
| Client ID | `d341df90-110e-45c5-9417-1681597e1dd3` |
| Application ID URI | None (client app, not a resource) |
| Role | Public client — MSAL popup/redirect login for the React SPA |

**Redirect URIs:**

| Environment | URI |
|---|---|
| Development | `http://localhost:3000` |
| QA | `https://ai-assistant-fe-qa-ckcwc7ffb0fjbyb7.eastus2-01.azurewebsites.net` |
| Prod | `https://ai-assistant-fe-a5c0eba9awhnb7hn.eastus2-01.azurewebsites.net` |

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
| Role | Daemon / service-to-service identity used by the API for outbound calls (e.g., OBO flows, Graph calls) |

**Config reference:** `appsettings.json` → `AzureAd` section

---

### 5. MailReadingApp — Mail Sync Daemon

| Field | Value |
|---|---|
| Client ID | `f81e2ce9-6a0b-47bb-be6b-7af5264254aa` |
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
| Prod | `https://ai-assistant-api-prod-g7e7ewcab7ekb8bk.eastus2-01.azurewebsites.net/mcp` | No ⚠️ |

**Config refs:** `appsettings.json` (base) + `appsettings.QA.json` / `appsettings.Prod.json` override `McpSettings:ResourceUri`.

---

## Outstanding Issue — Prod MCP Not Working

**Root cause:** The prod API hostname (`https://ai-assistant-api-prod-g7e7ewcab7ekb8bk.eastus2-01.azurewebsites.net`) is not registered as an Application ID URI on the `AiAssistantApi` registration. When MCP clients send a token request using this URL as the `resource` parameter, Azure AD cannot resolve it to any known app and rejects the request.

**Fix options:**

**Option A — Register the prod hostname (quickest):**
In the `AiAssistantApi` app registration → Expose an API → add:
```
https://ai-assistant-api-prod-g7e7ewcab7ekb8bk.eastus2-01.azurewebsites.net
```

**Option B — Set up a custom domain for prod (mirrors QA pattern):**
1. Configure a custom domain on the prod App Service (e.g., `https://ana-api.engineering.altusnova.com`)
2. Register that domain as an Application ID URI on `AiAssistantApi`
3. Update `appsettings.Prod.json` → `McpSettings:ResourceUri` to use the custom domain

Option B is preferred for stability — the raw Azure App Service hostname changes if the service is recreated, while a custom domain is durable.

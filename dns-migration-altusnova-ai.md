# DNS Migration — `altusnova.ai`

Migrates all Ana hosts off the current mix of `*.engineering.altusnova.com`, `*.azurewebsites.net`, and `*.azurestaticapps.net` onto the `{app}.{env}.altusnova.ai` scheme.

Source of truth for the naming scheme: [DNS](https://altusnova.atlassian.net/wiki/spaces/AIP/pages/4967661570/DNS) (Confluence, space AIP).

Related: [entra-app-registrations.md](entra-app-registrations.md), [mcp-claude-oauth-entra.md](mcp-claude-oauth-entra.md). Both describe current state and are updated **after** cutover, not before.

---

## Target hosts

Prod uses the **shorthand** form in all configuration — the `.prod.` segment exists in DNS but nothing points at it. The shorthand exists specifically so prod URLs don't say "prod".

| App | Azure resource | QA | Prod (canonical) | Prod (full, alias) |
|---|---|---|---|---|
| ANA portal | FE App Service | `assistant.qa.altusnova.ai` | `assistant.altusnova.ai` | `assistant.prod.altusnova.ai` |
| ANA API + MCP | API App Service | `assistant.api.qa.altusnova.ai` | `assistant.api.altusnova.ai` | `assistant.api.prod.altusnova.ai` |
| ANA Voice FE | Static Web App | `voice-assistant.qa.altusnova.ai` | `voice-assistant.altusnova.ai` | `voice-assistant.prod.altusnova.ai` |
| ANA Voice BE | App Service | `voice-assistant.api.qa.altusnova.ai` | — | `voice-assistant.api.prod.altusnova.ai` |

Voice BE has no shorthand — it isn't user-facing.

---

## Prerequisite — verify `altusnova.ai` in Entra

**This gates every Entra change and therefore all MCP work.**

Entra rejects any `https://` Application ID URI whose domain isn't a verified tenant domain or a subdomain of one, with:

> Values of identifierUris property must use a verified domain of the organization or its subdomain.

The tenant currently has exactly three domains — `altusnova.com` (verified, default), `altusnova.onmicrosoft.com`, `puruagrawal.onmicrosoft.com`. The previous MCP work stayed inside `altusnova.com`, so `ana-api-qa.engineering.altusnova.com` inherited that verification for free and no domain step was needed. `altusnova.ai` is a separate registrable domain and inherits nothing.

Add it: Entra ID → **Custom domain names** → **+ Add custom domain** → `altusnova.ai` → add the returned `MS=ms…` TXT at the zone apex → **Verify**. Requires **Domain Name Administrator** or **Global Administrator**.

Bare verification is sufficient — no Exchange or federation setup needed. Once verified, every subdomain is accepted automatically; individual hosts are not verified separately.

This TXT is Entra's and is distinct from the `asuid.<host>` TXT records App Service requires for custom domain binding.

---

## Work items

### DNS (`altusnova.ai` zone)

CNAME to the Azure default hostname plus `asuid.<host>` TXT per App Service host; SWA hosts use their own CNAME validation.

- QA: `assistant.qa`, `assistant.api.qa`, `voice-assistant.qa`, `voice-assistant.api.qa`
- Prod full: `assistant.prod`, `assistant.api.prod`, `voice-assistant.prod`, `voice-assistant.api.prod`
- Prod shorthand: `assistant`, `assistant.api`, `voice-assistant`

### Azure

Bind custom domain + managed TLS cert on 6 App Services and 2 Static Web Apps. Prod resources take both the full and shorthand hostname.

### Entra app registrations

Only two of the five need changes. The MCP Claude client, `AzureAd` service identity, and `MailReadingApp` are unaffected — their redirect URIs are all `claude.ai` / localhost.

**AiAssistantApi** (`4279cecc-…`) — add Application ID URIs:
- `https://assistant.api.qa.altusnova.ai/mcp`
- `https://assistant.api.altusnova.ai/mcp`

**Ana FE** (`d341df90-…`) — add SPA redirect URIs, each bare and with the `/auth-redirect.html` suffix:
- `https://assistant.qa.altusnova.ai`
- `https://assistant.altusnova.ai`
- `https://voice-assistant.qa.altusnova.ai`
- `https://voice-assistant.altusnova.ai`

Both are additive. Old URIs stay registered until cutover completes.

### Committed config — done, not yet deployed

| File | Change |
|---|---|
| `altus-ai-assistant/AltusAiAssistantApi/appsettings.json` | QA CORS origins — new hosts added alongside old |
| `altus-ai-assistant/AltusAiAssistantApi/appsettings.Prod.json` | Prod CORS origins added alongside old |
| `altus-ai-assistant-fe/env.development` + `.env.development` / `.env.qa` / `.env.production` | `VITE_MSAL_REDIRECT_URI`, `VITE_MSAL_BASE_URI` |
| `altus-ai-assistant-voice/fe/public/staticwebapp.config.json` | CSP `connect-src` → `voice-assistant.api.qa.altusnova.ai` |

CORS is additive because both old and new hostnames serve the same App Service during transition. The MSAL URIs are single-valued and flip at deploy.

The FE `.env.*` files are **tracked in git** despite `CLAUDE.md` describing them as gitignored, and the pipeline builds from the repo checkout — so an env change only reaches a build once committed and pushed.

### Deliberately deferred — waits on domain verification

`McpSettings:ResourceUri` in `appsettings.QA.json` / `appsettings.Prod.json` still points at the `engineering.altusnova.com` hosts. It must not flip until `altusnova.ai` is verified and the matching identifierUri exists on `AiAssistantApi` — otherwise discovery advertises a resource Entra can't resolve and MCP breaks. The portal migration is not gated on this, so the two are deployed separately.

Flip `ResourceUri` (both files) together with the QA App Service's `McpSettings__AppIdUri` app setting, in a dedicated post-verification deploy.

### Config outside the repos

- **QA App Service** app setting `McpSettings__AppIdUri` → `https://assistant.api.qa.altusnova.ai/mcp`. Must move in lockstep with the App ID URI, or discovery advertises a scope Entra rejects — see [mcp-claude-oauth-entra.md](mcp-claude-oauth-entra.md).
- **Azure DevOps variable groups** for the voice FE pipeline — `VITE_MSAL_REDIRECT_URI`, `VITE_MSAL_BASE_URI`, `VITE_BACKEND_URL`.

---

## Cutover order

1. Verify `altusnova.ai` in Entra.
2. DNS records, custom domain bindings, certs.
3. Add new App ID URIs and redirect URIs (additive — nothing breaks yet).
4. Deploy QA. Validate login, chat, SignalR streaming, MCP.
5. Deploy prod. Same validation. `web.config` carries both the `Canonical Host QA` and `Canonical Host Prod` rewrite rules; each is keyed to its own legacy hostname, so the shared file is inert in the other environment. `assistant.altusnova.ai` is bound and serving (verified), so the prod rule is safe to ship.
6. Announce MCP breakage and re-onboard.
7. Remove old `engineering.altusnova.com` App ID URIs, `*.azurewebsites.net` / `*.azurestaticapps.net` redirect URIs and CORS origins.
8. Update [entra-app-registrations.md](entra-app-registrations.md) and [mcp-claude-oauth-entra.md](mcp-claude-oauth-entra.md) to the new state.

Do not deploy ahead of step 2 — CORS and MSAL would point at hosts that don't resolve.

## Legacy hostname bookmarks

`VITE_MSAL_REDIRECT_URI` is baked into the bundle at build time, so the deployed app has exactly one valid MSAL origin. A user arriving on the legacy `*.azurewebsites.net` hostname opens a popup that lands cross-origin on the canonical host; the opener can't read it and the popup hangs blank. No error is surfaced — it just never logs in.

Handled with an IIS rewrite in `altus-ai-assistant-fe/web.config` that 302-redirects the legacy hostname to the canonical one before any script runs, preserving path and query. Keyed to the exact legacy hostname so the shared file is inert in the other environment.

It is deliberately a **302, not a 301** — browsers cache 301s indefinitely and ignore later corrections, which is unrecoverable if a hostname is wrong. Switch to `redirectType="Permanent"` once the redirect is proven, so bookmarks actually update.

The alternative — deriving `redirectUri` from `window.location.origin` so any host works — was rejected: it keeps both hostnames live indefinitely and requires every host registered in Entra, which is the opposite of the migration's goal.

## MCP is a breaking change

Changing `McpSettings:ResourceUri` changes the resource identity every existing Claude.ai and Claude Desktop connection was consented against. Every MCP user must remove and re-add the connector and re-consent. Schedule and announce it rather than shipping quietly.

## Open items

- **Voice SWA CSP ships QA-only.** `staticwebapp.config.json` is a single file deployed to both QA and prod SWAs, and its `connect-src` names only the QA backend. This predates the migration and is masked by the `https:` / `wss:` wildcards already in the same directive, which is why prod works. Not fixed here — flagged for a separate change.
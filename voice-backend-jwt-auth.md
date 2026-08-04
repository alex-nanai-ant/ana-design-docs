# Voice Backend — JWT Authentication

Status: implemented (2026-07-21) — pending the Entra registrations below, which are manual portal work

Repo: `altus-ai-assistant-voice` (`be/` + `fe/`)

## Problem

The voice backend has no authentication. `handleConnection` in `be/src/voice.gateway.ts` accepts any socket; `main.ts` calls `app.enableCors()` with no origin list. The gateway's `cors.origin: FRONTEND_URL` is a browser convention, not an access control — any non-browser client can open `wss://<host>/voice`, emit `initializeHybridVoice`, and consume Azure OpenAI Realtime quota.

Related hole, same code path: `initializeHybridVoice` accepts `searchIndexId`, `prompt`, and `model` from the client. Authentication alone still leaves an authenticated user able to pass another client's `uniqueId` and read that client's search index.

## Scope

1. Authenticate the WebSocket handshake against Entra.

Out of scope for now — everything that protects an *authenticated* user from another:

- **Search index authorization.** `searchIndexId` is still taken from the client, so a signed-in user can pass another client's `uniqueId` and read that index. Deferred because the check needs a main-API-audience token, which the per-environment voice registration deliberately does not issue; closing it requires either an on-behalf-of exchange (client secret on the voice registration) or the FE sending a second token.
- **Server-side `prompt` and `model` allowlist.** Both remain client-supplied, so an authenticated user can steer the assistant's instructions and pick the model.

This change closes anonymous access only.

## Approach

### Handshake validation

Enforce in Socket.IO middleware (`server.use()` in `afterInit`), not a Nest guard — guards run per-message, after the connection is open and `handleConnection` has allocated a session.

Token arrives via `socket.handshake.auth.token`. Validate signature against the tenant JWKS, plus `iss`, `aud`, `exp`/`nbf`, and required scope in `scp`. Reject on any failure, including JWKS fetch errors. Attach claims to `socket.data.user`.

Only the v2 issuer is accepted, so the registration's `api.requestedAccessTokenVersion` must be `2`. Both `aud` spellings are accepted (`api://<client-id>` for v1, bare client ID for v2) since they name the same registration.

Libraries: `jwks-rsa` + `jsonwebtoken`.

### Token expiry — deliberately not handled

The token is checked once, at the handshake. A socket authenticated at connect stays connected past its token's `exp`.

An expiry timer was built and then removed before release, to observe real session lengths first. It disconnected at `exp`, which tears down the upstream Azure OpenAI Realtime session (`closeSession`) and loses the assistant's conversation context — an amnesia event roughly hourly for a long meeting. That cost is not obviously worth paying given Entra access tokens cannot be revoked mid-lifetime anyway.

What to watch before deciding: `handleConnection` logs each token's `exp` and remaining lifetime. If sessions routinely outlive their token, add in-band refresh (FE sends a fresh token before `exp`; BE re-verifies and updates `socket.data.user`) rather than reinstating the disconnect.

Known gap this leaves: `openSocket` connects the socket before awaiting `setupMicrophone`, and the mic-failure path does not disconnect. A failed mic therefore leaves a live socket and a live, billable GPT realtime connection receiving no audio. Nothing currently reaps it. `SessionInfo.lastActivity` is already maintained but unread — an idle timeout built on it would target this directly, and more precisely than a token timer did.

### Frontend

`getToken()` in `fe/src/helpers/axiosInstance.ts` was module-private; it moved to `helpers/auth.ts` as `acquireToken(scope)` with `acquireApiToken` / `acquireVoiceToken` wrappers. The token rides in `io(url, { auth: { token } })` in `useVoice.ts`, making `createSocketConnection` async — `openSocket` was already async, so `Dashboard.tsx` is unaffected. `connect_error` with `unauthorized` retries once with a fresh token, then signs out.

## Entra registrations

Existing state (see `entra-app-registrations.md`): all registrations are shared across QA and Prod. The voice FE reuses the primary FE SPA registration `d341df90`.

**These registrations do not exist yet — they are manual portal work and the voice app cannot authenticate until they are created.** For each environment: expose scope `Voice.Access`, and pre-authorize SPA client `d341df90-110e-45c5-9417-1681597e1dd3` so users see no consent prompt.

Decision — split on the resource axis, not the client axis:

| Decision | Rationale |
|---|---|
| **New registration for the voice API** (not reuse `4279cecc`) | The `aud` claim is only a boundary if the voice BE has its own audience. Reuse means any main-API token is accepted by voice and vice versa. Cost: FE needs a second `acquireTokenSilent` — MSAL can't put two resources in one token. |
| **Split the voice API registration per environment** | With one shared registration a QA token is byte-identical to a prod token and passes prod validation — that reopens at the environment level the hole being closed. Greenfield, so the cost is one extra registration and one env var. |
| **Do not split the SPA registration** `d341df90` | Splitting means migrating redirect URIs and re-consenting users; a public client holds no secret, so the gain is small. Wrinkle: `d341df90` must be pre-authorized on both voice API registrations. |

Whether the main ANA app should also move to per-environment registrations is a separate migration, out of scope here.

## Config

New BE settings: `AZURE_TENANT_ID`, `VOICE_API_AUDIENCE`, `VOICE_API_REQUIRED_SCOPE`. `VOICE_API_AUDIENCE` differs per environment and comes from the `QA` / `Prod` Azure DevOps variable groups — **these must be populated before the next deploy or the backend will not start.** `EntraTokenService` fails fast on a missing value rather than starting with validation silently disabled.

Wired into `be/.env.example`, `infra/main.bicep` `appSettings`, and both the QA and Prod `AzureAppServiceSettings@1` blocks in `azure-pipelines-be.yml`. FE build takes `VITE_VOICE_API_SCOPE` in both `Build_QA` and `Build_Prod` in `azure-pipelines-fe.yml`.

`app.enableCors()` now takes an explicit origin list parsed from `FRONTEND_URL` (comma-separated). No `/health` route was added — nothing in `main.bicep` configures an App Service health probe, so it would have been unused.

## Open questions

- Where the server-side prompt lives — config, or fetched from the main API per client.
- For the deferred index check: on-behalf-of exchange (client secret on the voice registration) or a second token from the FE.

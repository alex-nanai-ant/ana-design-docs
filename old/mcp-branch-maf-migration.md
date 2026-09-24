# MCP Branch — Review & Migration onto the MAF Flow

> **Scope:** primary backend only ([altus-ai-assistant](../altus-ai-assistant/)), branch `feat/mcp`.
> **Companion doc:** [maf-adoption-plan.md](maf-adoption-plan.md) — the MAF migration this branch must align with.
> **Status:** analysis / migration plan. No migration code written yet.

This document describes the changes on the `feat/mcp` branch, how to land them cleanly on top of the now-implemented MAF completion flow, and improvements worth making to the branch itself.

---

## 0. Git situation

`feat/mcp` already contains all of `main`. The MAF refactor (`91c497b ANA-123 Refactor complete chat. Add MAF for agentic. Feature toggled.`) landed on `main` and was pulled into the branch via `b5adfbd "Merge branch 'main' into feat/mcp"`. **The MCP code and the MAF flow (`Services/Maf/`, `CompleteChatService`) already coexist in `feat/mcp`**, and the merge resolved cleanly — no conflict markers remain in tracked source.

**"Migrate MCP onto MAF" means re-pointing couplings, not re-merging.** The MCP code was authored against the pre-MAF `ChatService` and still binds to the legacy completion/retrieval surface that the MAF plan removes in Phase 4. The migration work is to re-point those bindings at the MAF abstractions before the legacy code is deleted.

---

## 1. What the branch changes

The branch adds a **remote MCP server** (`/mcp`) that exposes Ana's corpus to external MCP clients (Claude Desktop / Web / Code), plus the auth, observability, and retrieval plumbing it needs. ~32 new files, ~2,700 insertions over `main`.

### 1.1 MCP protocol surface
- New packages `ModelContextProtocol` + `ModelContextProtocol.AspNetCore` (`0.4.1-preview.1`) in `AltusAiAssistantApi.csproj`.
- `app.MapMcp("/mcp").RequireAuthorization("McpInvoke")` and `AddMcpServer(...).WithHttpTransport().WithTools<...>()` in `Program.cs`.
- Three **registered** tools:
  - `Mcp/Tools/ListClientsTool.cs` — enumerate the caller's allowed clients.
  - `Mcp/Tools/SearchDocumentsTool.cs` — vector search scoped to one client.
  - `Mcp/Tools/FetchDocumentTool.cs` — fetch a full source document by `documentId` (index `parent_id`), paginated by chunk.
- A fourth tool, `Mcp/Tools/AskAnaTool.cs`, previously existed (disabled via commit `ec1ca8b "feat: turn off ask ana tool"`) as the only piece that called full chat completion. It has since been **deleted** along with `McpService.AskAna`, its input/result DTOs, and `AnaSourceDto`. The MCP surface is now exactly the three registered retrieval tools.

### 1.2 Auth / security (OAuth protected resource, RFC 9728)
- `Controllers/McpDiscoveryController.cs` serves `/.well-known/oauth-protected-resource`.
- `Middleware/McpChallengeMiddleware.cs` attaches `WWW-Authenticate: Bearer resource_metadata="..."` on 401s for MCP paths (and deliberately *excludes* the `/mcp/status` and `/mcp/revoke` REST siblings, which use plain JWT).
- `Program.cs` JWT hardening: explicit `ValidAudiences` (`apiClientId` + `McpSettings:AppIdUri`) and `ValidIssuers` pinning; an `McpInvoke` authorization policy that requires the `Mcp.invoke` scope (`scp` / scope claim).
- Stricter per-route CORS for the `/mcp` REST surface (origin allow-list, no `AllowAnyHeader/Method`); a request-body-size cap on `/mcp` (`McpSettings:MaxRequestBodyBytes`, default 1 MB) layered over the 100 MB Kestrel global.
- `Controllers/McpStatusController.cs` (`GET /mcp/status`, `POST /mcp/revoke`) backed by `Services/McpGrantService.cs`, which reads/deletes AAD `oauth2PermissionGrants` via Microsoft Graph, with a 60s per-user `IMemoryCache` TTL.

### 1.3 Tenancy / scope resolution
- `Mcp/ClientScopeResolver.cs` maps AAD `oid` → app user → allowed `UserClient`s and enforces that every tool call is scoped to exactly one client.
- `Services/UserService.cs` gains `GetUserIdByObjectId` and a **read-only** `GetUserClientsReadOnly` — deliberately *without* the admin auto-grant side effect of `GetUserClients`, so an MCP read can never silently re-create a previously revoked tenancy.
- Typed exceptions: `Mcp/ForbiddenAccessException.cs`, `Mcp/NoClientsAvailableException.cs`, `Mcp/DocumentNotFoundException.cs`.

### 1.4 Retrieval surface (the MAF-relevant part)
- New `Services/IAnaSearchService.cs` — `Search(...)` + `FetchByDocumentId(...)`, returning `DocumentHit` / `DocumentFetchResult` records — **implemented by `ChatService`** (the `ChatService : IChatService, IAnaSearchService` additions).
  - `Search` reuses the **legacy** `GetNextGptDeploymentParams()` + `GetEmbedding(query, gptParams)` and re-implements vector search inline.
  - `FetchByDocumentId` is genuinely new logic: counts chunks for a `parent_id`, then pages them in `chunk_id` order, concatenating up to `maxChars`/`maxChunks`.
- `Services/McpService.cs` orchestrates the three retrieval tools. (The former `AskAna` orchestration, which called `_chatService.CompleteChat` / `CompleteChatWithPlanning`, has been removed.)
- DI (`Program.cs`): `ChatService` is registered as its own type, with **both** `IChatService` and `IAnaSearchService` resolving to that same instance.

### 1.5 Observability + PII
- `Telemetry/PiiScrubbingProcessor.cs` — strips `Authorization`/`Cookie`/`Set-Cookie` from telemetry property bags and caps message lengths; runs first in the chain.
- `Mcp/McpTelemetry.cs` — `McpToolInvocation` custom event (tool name, user oid, resolved client, duration, success, error category, MCP client user-agent).
- Adaptive sampling reconfigured to **exclude** the `Event` type so MCP invocations keep full fidelity.
- `Middleware/ErrorHandlerMiddleware.cs` switched to the `(Exception, template, args)` logging overload (no string-interpolated exceptions that would leak query/SQL fragments).
- In `ChatService`: removed `Console.WriteLine` debug spam and **stopped logging raw query / planner text** (length-only) to avoid leaking confidential client content.

### 1.6 Config & models
- `Models/McpSettings.cs` + a `McpSettings` block in every `appsettings.*.json`, validated `OnStart` (rejects missing or `PLACEHOLDER` values).
- DTO / record files under `Models/Mcp/` (tool inputs/results, `OAuthProtectedResourceMetadataDto`, `McpStatusResponse`, `McpHealthResponseDto`, etc.). The `ask_ana` DTOs and `AnaSourceDto` were removed with the tool.

---

## 2. Migration onto the MAF flow

**It already runs.** With `UseMaf` on, `IChatService.CompleteChat` delegates to `CompleteChatService`, and the MAF service handles `SignalRConnectionId == null` (skips streaming, still persists and returns the `ChatResponseDto`). So even `AskAna` would work today. The migration work is removing bindings to **legacy code that Phase 4 deletes**.

### a. Move `IAnaSearchService` off `ChatService` onto the MAF retrieval layer — primary item
The current implementation lives on `ChatService` and calls `GetNextGptDeploymentParams()` — explicitly on the Phase-4 deletion list (Decision 9) — and re-implements vector search inline, duplicating `Services/Maf/IKnowledgeRetriever.cs`, which already exposes `GetEmbedding(string)` and `GetFromSearchIndex(...)` on the single configured deployment.

- Implement `IAnaSearchService` on a small standalone service (or fold it into `KnowledgeRetriever`) using the single-deployment config + the shared search-client construction.
- Add `FetchByDocumentId` as a new `KnowledgeRetriever` method — it is the natural sibling of `ReadWholeDocument`.
- Change DI from `AddScoped<IAnaSearchService>(sp => sp.GetRequiredService<ChatService>())` to the new type; drop `ChatService : IAnaSearchService` and the double-registration.
- **Outcome:** `search_documents` / `fetch_document` no longer depend on anything Phase 4 removes.

### b. `ask_ana` — removed (was: re-point at `ICompleteChatService`)
**Resolved by deletion.** `ask_ana` was the only MCP coupling to the completion surface (`IChatService.CompleteChat` / `CompleteChatWithPlanning`) and the only consumer of `chatResponse.MessageContext` (via the now-deleted `McpService.MapSources`). With the tool removed, the MCP layer no longer touches completion at all, so the earlier concerns — re-pointing `AskAna` at `ICompleteChatService`, the triage-relocation sequencing (MAF **Deviation 4**), and the `MessageContext` parity check for the non-streaming MCP caller — no longer apply to MCP.

### e. Housekeeping before merging to `main`
- Delete `Program.cs.orig`, `appsettings.QA.json.orig`, `.vscode/`.
- Remove the unused `ILogger<McpService>` constructor parameter.

---

## 3. Improvements to the branch itself

### 🔴 Critical — secrets in the working tree
The uncommitted `appsettings.Development.json` and `appsettings.json` contain **real plaintext secrets** replacing the `env-var` placeholders: SQL connection, `BlobAccessKey`, `AzureSignalRConnectionString`, `SlackBotToken`, `SearchServiceKey`, `AzureAd:ClientSecret`, and the GPT deployment `Key`. `appsettings.json` is committed (not gitignored), so this is one `git add` from leaking live QA credentials.

**Action: rotate these keys and revert both files to `env-var` before any commit.** Unrelated to MCP correctness, but the highest-priority finding.

### Content-exclusion parity gap
The chat retrieval path applies the `IContentExclusionService` exclusion filter; the new `IAnaSearchService.Search` / `FetchByDocumentId` do **not**. `search_documents` / `fetch_document` can therefore surface chunks the in-product chat would suppress for that client — a governance hole on an externally exposed tool. Thread the same exclusion filter through (the MAF `MafRunContext.ExclusionFilter` is the natural source once retrieval moves to the MAF layer).

### Transient session leak in `AskAna` — resolved by deletion
`AskAna` used to create a `ChatSession` per call (`CreateStandardSession(...)` + `SaveChanges`), leaking orphan sessions and returning a misleading `SessionId: 0`. Removing the tool eliminates this. (It was also the sole reason the MCP layer needed `UserClient.Id`; with it gone, the whole MCP surface is scoped purely by `Client.Id`.)

### Duplicated Azure Search client construction
`Search` and `FetchByDocumentId` each new up `SearchClient` / endpoint / credential inline — the same construction `KnowledgeRetriever` already performs, plus a third copy of the index-name / embedding logic. Consolidating (migration item **a**) removes the duplication.

### Minor
- `"UseMaf": true,` has a trailing comma before `}` in dev settings — valid only because the config JSON reader allows trailing commas; clean it up.
- The `McpInvoke` policy hand-rolls scope parsing that `Microsoft.Identity.Web`'s `RequireScope` / `[RequiredScope]` already provides — fewer moving parts with the built-in.
- `FetchByDocumentId`'s `Truncated` / offset math counts empty chunks toward the page (`chunkOffset + chunksIncluded`), which can slightly over-report progress vs. content actually returned — defensible, but worth a comment or a test given there is no test coverage here.

---

## 4. Suggested sequencing

1. **Security first:** rotate leaked keys; revert `appsettings*.json` to `env-var`; delete `.orig` / `.vscode` artifacts.
2. **Migration item (a):** extract `IAnaSearchService` onto the MAF retrieval layer with exclusion-filter parity; re-point DI. This decouples the live MCP tools from Phase-4 deletions.
3. **`ask_ana` (b):** done — deleted, so the MAF completion re-pointing and `MessageContext` parity check no longer apply to MCP.
4. Housekeeping (e) and the minor items.

After steps 1–2, the registered MCP tools (`list_clients`, `search_documents`, `fetch_document`) are safe to ship on the MAF flow. They are the entire MCP surface now that `ask_ana` is removed.
# Identity Discovery v1 — `AltusAiProcessContent` Azure Function

Companion to [identity-discovery-design.md](identity-discovery-design.md) and [identity-discovery-v0-plan.md](identity-discovery-v0-plan.md).

> **Status:** draft — to be refined before implementation.

## Context

v0 ([identity-discovery-v0-plan.md](identity-discovery-v0-plan.md)) is a manually-triggered API endpoint (`POST /api/clients/{clientId}/identities/discover`) that full-scans every Slack/Grain/Email blob each run. Fine at small volume, but it re-pays the GPT cost for unchanged blobs and only runs on demand.

v1 makes discovery run **automatically**, process **only blobs that actually changed**, and moves it into a **new Azure Function project, `AltusAiProcessContent`** — joining the existing sync-worker family (`AltusAiSlackSync`, `AltusAiMailSync`, `AltusAiConfluenceSync`, `AltusAiTextMergeSkill`). The v0 API endpoint is **removed**; its logic moves into the Function.

As a Function, it follows the established sync-worker conventions rather than the API's:
- References **`AltusAiShared` only** — no reference to `AltusAiAssistantApi`, no EF.
- **Dapper + raw SQL** over `Microsoft.Data.SqlClient` for DB access (client lookup + identity upsert), `SqlConnectionString` from an environment variable — as [AltusAiSlackSync/Function.cs](../../altus-ai-assistant/AltusAiSlackSync/Function.cs) does.
- Config from **environment variables** (`local.settings.json` locally, App Settings in Azure), not `appsettings.json` / bound `AppSettings`.
- DI wired manually as singletons in `Program.cs` ([pattern](../../altus-ai-assistant/AltusAiSlackSync/Program.cs)).

### Decisions
1. **Hosting:** new Azure Function `AltusAiProcessContent` — named generically ("process content") so future content-processing functions can join it; identity discovery is its first function.
2. **DB access:** **Dapper**, matching the sync workers. The v0 `UpsertObservedIdentity` check-then-insert is reimplemented in SQL; the API's EF copy of the discovery path is removed.
3. **Manual/backfill trigger:** an **HTTP-triggered function in the same project** (like `SlackSync`'s `DeepSyncHttp`), replacing the deleted API endpoint.
4. **Shared types:** `IdentitySource` (and the `ExtractedIdentity` record) **move to `AltusAiShared`** so the API's `ContactIdentity` entity and the Function's extraction logic reference one definition.
5. **Per-blob identity list in Table Storage:** **provenance only** — record blob→identities for idempotency/diagnostics; never auto-remove identities (the reviewer still owns assign/merge/delete).
6. **Source scope:** Slack / Grain / Email (same as v0). Confluence excluded.
7. **Change detection:** two-tier. Blob `LastModified` (not ETag) decides whether to *read* a blob; a **SHA256 content hash** decides whether to *re-extract* (the GPT-cost step). A metadata-only write bumps `LastModified` without changing content, so gating GPT on the content hash avoids re-paying extraction cost when only metadata moved.

---

## Part A — Shared library changes (`AltusAiShared`)

### A1. Move `IdentitySource` into `AltusAiShared`
[IdentitySource.cs](../../altus-ai-assistant/AltusAiAssistantApi/Models/Contacts/IdentitySource.cs) currently lives in `AltusAiAssistantApi.Models.Contacts` and carries a `[JsonConverter(typeof(StringEnumConverter))]` attribute. `AltusAiShared` already references `Newtonsoft.Json` 13.0.3, so the attribute moves cleanly.

- Move the enum to `AltusAiShared` (e.g. namespace `AltusAiShared.Contacts`). Values unchanged: `Unknown, Grain, Slack, Confluence, Email, Phone`.
- **Namespace fallout in the API:** every reference (`ContactIdentity`, `ContactsService`, the contact DTOs, `ContactIdentityConfiguration`, `IdentityMentionsService`, etc.) needs its `using` updated from `AltusAiAssistantApi.Models.Contacts` to the new shared namespace. Mechanical but repo-wide — a build surfaces every site.

### A2. Move `ExtractedIdentity` into `AltusAiShared`
The `record ExtractedIdentity(string SourceKey)` (today in [IIdentityExtractionService.cs](../../altus-ai-assistant/AltusAiAssistantApi/Services/IIdentityExtractionService.cs)) moves to `AltusAiShared` alongside `IdentitySource`, since the extraction service moves to the Function (B3).

### A3. `ListFilesWithProperties` on `BlobStorageService`
`BlobStorageService.ListFiles` returns only `(name, metadata)` and has callers in `SlackSyncService` and `AdminService`. Add a **new** method beside it to avoid touching those:

- [BlobStorageService.cs](../../altus-ai-assistant/AltusAiShared/BlobStorageService.cs): `ListFilesWithProperties(containerName)` → `List<BlobItemInfo>`. Introduce a small **named type** `BlobItemInfo` (in `AltusAiShared`) rather than a tuple, so callers can grow without breaking the signature:

  ```csharp
  public class BlobItemInfo
  {
      public string Name { get; init; } = string.Empty;
      public IDictionary<string, string> Metadata { get; init; } = new Dictionary<string, string>();
      public DateTimeOffset? LastModified { get; init; }
      // Future: ETag, ContentLength, ContentHash, etc. — add fields without touching call sites.
  }
  ```

  `GetBlobsAsync(BlobTraits.Metadata)` already exposes `blobItem.Properties.LastModified` (and `.ETag`) — no extra round-trip. Adding `ETag` here later is the cheap hedge noted in the LastModified-vs-ETag discussion.

> `ContainerExists`, `GetBlobStream`, and `AzureTableClientHelper.InitializeTableClient` already live in `AltusAiShared` and are usable from the Function as-is.

---

## Part B — New Function project (`AltusAiProcessContent`)

Scaffold the project to match the existing workers ([AltusAiSlackSync.csproj](../../altus-ai-assistant/AltusAiSlackSync/AltusAiSlackSync.csproj) is the template):

- `Microsoft.NET.Sdk`, `net8.0`, `AzureFunctionsVersion v4`, `OutputType Exe`.
- Package refs from the SlackSync template (`Microsoft.Azure.Functions.Worker*`, `...Extensions.Http`, `...Extensions.Timer`, `ApplicationInsights.WorkerService`), **plus** the GPT client packages the API uses for `IChatClient`: `Azure.AI.OpenAI`, `Microsoft.Extensions.AI`, `Microsoft.Extensions.AI.OpenAI`.
- `ProjectReference` → `AltusAiShared` only.
- `host.json`, `local.settings.json` (gitignored secrets), `azure-pipelines.yml`.
- Add the project to `AltusAiAssistantApi.sln`.

### B1. `Program.cs` — DI wiring
Mirror [AltusAiSlackSync/Program.cs](../../altus-ai-assistant/AltusAiSlackSync/Program.cs): App Insights worker service, then register as singletons reading from `Environment.GetEnvironmentVariable`:

- `BlobStorageService` — from `BlobAccountName` + `BlobStorageAccountKey` (same construction as SlackSync).
- `IChatClient` — built the same way the API does ([Program.cs](../../altus-ai-assistant/AltusAiAssistantApi/Program.cs) lines 50–69): `new AzureOpenAIClient(endpoint, ApiKeyCredential(key)).GetChatClient(deployment).AsIChatClient()`, wrapped in a `ChatClientBuilder`. Endpoint/key/deployment come from env vars (e.g. `GptEndpoint`, `GptKey`, `GptDeployment`).
- `IdentityExtractionService` (B3), `IdentityDiscoveryStateRepository` (B2), `IdentityUpsertRepository` (B4), `IdentityDiscoveryRunner` (B6).
- `SqlConnectionString` is read per-call inside the repositories (as SlackSync does), not registered.

### B2. Table Storage state (provenance + change detection)
Use [AzureTableClientHelper.InitializeTableClient](../../altus-ai-assistant/AltusAiShared/Confluence/AzureTableClientHelper.cs) (`BlobAccountName` + `BlobStorageAccountKey` env vars):

- `IdentityDiscoveryStateEntity : ITableEntity`
  - `PartitionKey` = clientId; `RowKey` = `SHA256(blobPath)` hex (blob paths contain `/`, illegal in RowKey)
  - `BlobPath` (readable), `Source`, `LastModified` (DateTimeOffset, last processed), `ContentHash` (hex SHA256 of last-extracted content), `SourceKeysJson` (JSON `string[]` = per-blob provenance list), `LastProcessedAt`, `FailureCount`, `LastError?`
- `IdentityDiscoveryStateRepository` — `Get(clientId, blobPath)`, `Upsert(entity)`. Table name from env var (default `IdentityDiscoveryState`).

### B3. `IdentityExtractionService` (moved from the API)
Move [IdentityExtractionService.cs](../../altus-ai-assistant/AltusAiAssistantApi/Services/IdentityExtractionService.cs) + its interface into the Function project verbatim — it depends only on `IChatClient` (Microsoft.Extensions.AI) and the now-shared `IdentitySource` / `ExtractedIdentity`. The prompt, `SourceNoun`/`SourceHint`, fence-stripping, and JSON parsing are unchanged. **Delete the API's copy** (the API no longer extracts).

### B4. `IdentityUpsertRepository` — Dapper upsert (replaces the EF `UpsertObservedIdentity`)
Reimplement the v0 check-then-insert ([ContactsService.cs](../../altus-ai-assistant/AltusAiAssistantApi/Services/ContactsService.cs) lines 188–203) in SQL:

```csharp
Task<bool> UpsertObservedIdentity(int clientId, IdentitySource source, string sourceKey, CancellationToken ct);
// returns true when a new row was inserted
```

- Insert into `ContactIdentities` with `ContactId = null` (unassigned). Store `Source` as its int enum value.
- **Populate `BaseEntity` audit columns manually** — EF's audit interceptor did this automatically; Dapper does not. Set `CreatedDate`/`UpdatedDate = UtcNow`, `CreatedBy`/`UpdatedBy` = a system marker (e.g. `"IdentityDiscovery"`). Confirm the exact column set against [ContactIdentityConfiguration.cs](../../altus-ai-assistant/AltusAiAssistantApi/Data/ContactIdentityConfiguration.cs).
- **Make the upsert race-safe at the DB:** the table has a **unique index on `(ClientId, Source, SourceKey)`**. Use an atomic `INSERT ... WHERE NOT EXISTS (...)` (or `MERGE`) and report "created" from rows-affected, and/or catch SQL error 2601/2627 and treat as not-created. This is safer than the EF check-then-insert v0 shipped, which had no catch (see open items).

### B5. Client lookup (Dapper)
Active clients to scan: `SELECT Id, UniqueId FROM Clients WHERE InactiveDate IS NULL` (mirrors the SlackSync active filter). The HTTP trigger's single-client lookup filters by `UniqueId`.

### B6. `IdentityDiscoveryRunner` — the shared work, made incremental
The per-client scan, ported from [IdentityDiscoveryService.cs](../../altus-ai-assistant/AltusAiAssistantApi/Services/IdentityDiscoveryService.cs) and upgraded to incremental. Both triggers call it.

```csharp
Task<IdentityDiscoveryResultDto> RunDiscovery(int clientId, string clientUniqueId, DateTimeOffset modifiedSince, bool force, CancellationToken ct);
Task RunAllClients(DateTimeOffset modifiedSince, CancellationToken ct); // loops B5 clients, per-client try/catch
```

`modifiedSince` is **required** and acts as a **time-window prefilter** *before* any Table Storage work, so old blobs are cheap to ignore. No "process everything" magic value — to scan all blobs, pass a sufficiently old date (e.g. `2000-01-01`).

- **Guard the container first:** `BlobStorageService.ContainerExists(clientUniqueId)` → empty result if missing, so clients with no synced sources don't throw a 404 out of `GetBlobsAsync`.
- Use `ListFilesWithProperties`. Keep the `.md`/`.html` + `TryMapSource` filtering (the `sourceType` metadata → `IdentitySource` map ported from the API service).
- **If `blob.LastModified < modifiedSince` → skip immediately** (no read, no Table lookup). Count as `BlobsOutsideWindow` (or fold into `BlobsSkipped`).
- Per remaining candidate: load state row. If `!force && state != null && state.LastModified >= blob.LastModified` → **skip unchanged** (`BlobsUnchanged`), no read, no GPT.
- Otherwise read content, compute `SHA256(content)` (hex). Reading is cheap; GPT is the cost we protect.
  - **If `!force && state != null && state.ContentHash == hash`** → content identical, only `LastModified` advanced (metadata patch). **Skip extraction**, advance the state row's `LastModified`/`LastProcessedAt`, leave `SourceKeysJson` as-is, count as `BlobsUnchanged`.
  - **Else** → extract, `UpsertObservedIdentity` each (Dapper, B4), then **write the state row** with new `LastModified` + `ContentHash` + JSON list of extracted `sourceKey`s + `LastProcessedAt`.
- On read/extraction failure: log, **do not** advance `LastModified` (next tick retries), bump `FailureCount`/`LastError`; skip once `FailureCount` exceeds a small cap (poison guard, design §8). Provenance-only ⇒ never delete identities whose sourceKey disappeared.
- `force` bypasses both the per-blob `LastModified` and `ContentHash` comparisons (full re-extract) — used by the HTTP trigger's `forceUnchanged=true`.

### B7. Result DTO
Move [IdentityDiscoveryResultDto.cs](../../altus-ai-assistant/AltusAiAssistantApi/Models/Contacts/IdentityDiscoveryResultDto.cs) into the Function (or `AltusAiShared`), and add `int BlobsUnchanged` (and `int BlobsOutsideWindow` if tracked separately from `BlobsSkipped`).

### B8. Function triggers (`Function.cs`)
Two functions in the project, both delegating to `IdentityDiscoveryRunner`:

- **`[Function("IdentityDiscovery")]` — TimerTrigger.** `[TimerTrigger("0 */15 * * * *")]` (every 15 min), **`RunOnStartup = false`** so it does **not** fire at boot — the schedule-only requirement, native to Functions timers (unlike the SlackSync incremental, which uses `RunOnStartup = true`). Per tick: `RunAllClients(DateTimeOffset.UtcNow.AddDays(-DiscoveryLookbackDays), ct)`. Default `DiscoveryLookbackDays = 3`. No custom enable flag — to turn the schedule off, set the Azure app setting **`AzureWebJobs.IdentityDiscovery.Disabled = true`**, which disables **only this timer** and leaves the HTTP function callable.
- **`[Function("IdentityDiscoveryHttp")]` — HttpTrigger** (`AuthorizationLevel.Function`, GET), replacing the deleted API endpoint. The on-demand / backfill trigger for a single client. Query params:
  - **`clientUniqueId` (required)** — which client to scan (resolve `Id` via B5).
  - **`modifiedSince=<ISO date>` (required)** — same window prefilter. Pass an old date (e.g. `2000-01-01`) for the initial full backfill.
  - **`forceUnchanged=true` (optional, default false)** — also bypass the per-blob Table Storage comparison, re-extracting even unchanged blobs (the runner's `force`).
  - A separate function from the timer, so it stays callable even when the schedule is disabled via `AzureWebJobs.IdentityDiscovery.Disabled`.

### B9. Configuration (env vars / App Settings)
No `appsettings.json` binding — Functions read env vars (`local.settings.json` `Values` locally; App Settings in Azure). Required:
`BlobAccountName`, `BlobStorageAccountKey`, `SqlConnectionString`, `GptEndpoint`, `GptKey`, `GptDeployment`, `IdentityDiscoveryStateTable` (default `IdentityDiscoveryState`), `DiscoveryLookbackDays` (default `3`), plus `AzureWebJobsStorage` / `APPLICATIONINSIGHTS_CONNECTION_STRING` per the worker template. The timer is turned off (when desired) via the `AzureWebJobs.IdentityDiscovery.Disabled` app setting, not a custom flag.

---

## Part C — API cleanup (`AltusAiAssistantApi`)

Discovery no longer runs in the API; remove the now-dead code:

- **Delete the endpoint** — the `DiscoverIdentities` action in [ContactsController.cs](../../altus-ai-assistant/AltusAiAssistantApi/Controllers/ContactsController.cs) (lines 124–129) and the `IIdentityDiscoveryService` constructor dependency (lines 14, 20, 25).
- **Delete** `IIdentityDiscoveryService` / `IdentityDiscoveryService` and `IIdentityExtractionService` / `IdentityExtractionService` from the API, and their `AddScoped` registrations in `Program.cs`.
- **`ContactsService.UpsertObservedIdentity`** is now unused by the API (the Function owns discovery writes). Remove it for leanness, or leave it as harmless documentation — see open items.
- **Keep** everything else in the contacts review surface (`ContactIdentity` entity, `GET /identities?unassigned=true`, assignment) — only the *discovery* path leaves the API.
- `IdentityDiscoveryResultDto` moved out (B7); fix/remove its API references.

> The `ContactIdentity` entity, its EF config, and the unique index stay in the API — the SQL table is shared between the API (reads/review) and the Function (Dapper writes). No migration is needed; the Function writes the same table the API already owns.

---

## Verification
1. `dotnet build AltusAiAssistantApi.sln` — no errors (API cleanup + shared move + new project all compile).
2. `dotnet test tests/AltusAiShared.Tests/AltusAiShared.Tests.csproj` — existing tests pass.
3. `cd AltusAiProcessContent && func start`: confirm the timer does **not** fire at startup (`RunOnStartup = false`) and fires on the next 15-min boundary, running `RunAllClients`; the HTTP trigger works on demand.
4. Set `AzureWebJobs.IdentityDiscovery.Disabled=true`: confirm the timer never fires, but `IdentityDiscoveryHttp` still works on demand.
5. **Backfill:** call the HTTP trigger with `?clientUniqueId=<id>&modifiedSince=2000-01-01`. Unassigned identities appear via `GET /api/clients/{clientId}/identities?unassigned=true`; a state row per processed blob exists in `IdentityDiscoveryState` with populated `SourceKeysJson`.
6. **Second run, no blob changes:** `BlobsUnchanged` = candidate count, `IdentitiesCreated` = 0, no GPT calls.
7. **Changed content:** re-upload one blob with edited content: only that blob re-extracts; its state row's `LastModified`, `ContentHash`, `SourceKeysJson` update.
8. **Metadata-only change:** patch a blob's metadata without touching content: next run reads it but makes **no GPT call**; state row's `LastModified` advances while `ContentHash`/`SourceKeysJson` stay; counts as `BlobsUnchanged`.
9. **Window:** a blob older than 3 days, never processed, is ignored by the timer run; the HTTP trigger with `modifiedSince=2000-01-01` picks it up.
10. **Upsert race:** confirm the Dapper insert is idempotent under the unique index — a concurrent timer tick + backfill for the same client does not throw on `(ClientId, Source, SourceKey)`.
11. **Logging:** App Insights `traces` from one run share an `operation_Id` (the Functions invocation id) and carry `operation_Name`; filtering `operation_Name == "IdentityDiscovery"` returns scheduled runs only (manual runs are `IdentityDiscoveryHttp`).

---

## Logging & observability
Functions log via `ILogger<T>` into the App Insights worker service already wired by the template — these land in `traces` (a `TimerTrigger` produces no HTTP request; the `HttpTrigger` does). No custom logging scope is needed: the Functions host already wraps each invocation in a scope carrying the **function name** (`operation_Name`) and a unique **invocation id** (`operation_Id`), so all logs from one run are correlated, and scheduled vs manual is just the function name (`IdentityDiscovery` vs `IdentityDiscoveryHttp`).

- One Information **start** line and one Information **summary** line per client run (clientId + `IdentityDiscoveryResultDto` counts); per-blob read/extraction failures at Warning; poison-skip at Warning/Error.
- KQL (on `traces`), using the native fields:
  ```kql
  // Scheduled job only
  traces | where operation_Name == "IdentityDiscovery" | order by timestamp asc
  // One run, end to end
  traces | where operation_Id == "<invocation-id>" | order by timestamp asc
  ```

---

## Open items to refine
- **Upsert race vs. the `(ClientId, Source, SourceKey)` unique index** — addressed by B4: the Dapper upsert is made atomic at the DB (`INSERT ... WHERE NOT EXISTS` / `MERGE`, or catch 2601/2627). Two concurrent runs still last-write-wins on the same Table Storage state row (benign — provenance only). A per-client single-flight lock remains optional.
- **Active-client query** — confirmed shape: `Clients WHERE InactiveDate IS NULL`. Independent of this, `RunDiscovery` guards with `ContainerExists` so a container-less client is a cheap no-op.
- **Keep `ContactsService.UpsertObservedIdentity` in the API?** — now unused by the API. Remove for leanness, or leave as harmless dead code. (Lean toward remove.)
- **GPT config in the Function** — single `GptEndpoint`/`GptKey`/`GptDeployment` env trio vs. mirroring the API's multi-deployment `GptDeployments` map. Single deployment is simpler for a worker; revisit if load-balancing across deployments matters.
- **`IdentitySource` namespace** — proposed `AltusAiShared.Contacts`; confirm before the repo-wide `using` sweep.
- **Where the Function-local types live** (`IdentityDiscoveryStateEntity/Repository`, `IdentityDiscoveryResultDto`, extraction service) — in `AltusAiProcessContent` vs. `AltusAiShared`. Default to the Function project unless another worker will need them.
- **Interval / cron** — 15-min `TimerTrigger` cron (`0 */15 * * * *`). A clock-aligned cron is native to Functions timers, so no boot-relative drift.
- **Poison cap** — value for `FailureCount` max and whether to surface an alert (design §8).
- **Provenance reads** — whether any UI/endpoint surfaces `SourceKeysJson`, or it stays internal.
- **Deploy pipeline** — new `azure-pipelines.yml` for `AltusAiProcessContent` modeled on [AltusAiSlackSync/azure-pipelines.yml](../../altus-ai-assistant/AltusAiSlackSync/azure-pipelines.yml); a new Function App resource + App Settings to provision.
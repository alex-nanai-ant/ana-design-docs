# Identity Discovery v1 — Incremental Discovery Plan

Companion to [identity-discovery-design.md](identity-discovery-design.md) and [identity-discovery-v0-plan.md](identity-discovery-v0-plan.md).

> **Status:** draft — to be refined before implementation.

## Context

v0 ([identity-discovery-v0-plan.md](identity-discovery-v0-plan.md)) is a manually-triggered endpoint that full-scans every Slack/Grain/Email blob each run. Fine at small volume, but it re-pays the GPT cost for unchanged blobs and only runs on demand. v1 makes discovery run automatically and process only blobs that actually changed.

The original design ([identity-discovery-design.md](identity-discovery-design.md) §3, §8) proposed a self-contained **Azure Function** with **ETag** change detection and a SQL `ContactIdentityEvidence` table. Investigation of the codebase changed that:

- **All identity code already lives in the API project** — `IdentityDiscoveryService`, `IdentityExtractionService`, `ContactsService`, EF `AiAssistantDbContext`. The sync Functions reference only `AltusAiShared` + Dapper and cannot reach that EF code without relocating it.
- **The API already runs scheduled background work** — `GrainSynchronizer` is a `BackgroundService` on a `PeriodicTimer`, the established in-process scheduled-job pattern, with full scoped/EF access.

### Decisions
1. **Hosting:** in-API `BackgroundService` (mirror `GrainSynchronizer`), not a new Azure Function. Maximum reuse, no code relocation.
2. **Per-blob identity list in Table Storage:** **provenance only** — record blob→identities for idempotency/diagnostics; never auto-remove identities (the reviewer still owns assign/merge/delete).
3. **Source scope:** Slack / Grain / Email (same as v0). Confluence excluded.
4. **Change detection:** two-tier. Blob `LastModified` (not ETag) decides whether to *read* a blob; a **SHA256 content hash** decides whether to *re-extract* (the GPT-cost step). A metadata-only write bumps `LastModified` without changing content, so gating GPT on the content hash avoids re-paying extraction cost when only metadata moved.

---

## Implementation (all in `AltusAiAssistantApi`, reusing `AltusAiShared`)

### 1. Blob listing with LastModified
`BlobStorageService.ListFiles` returns only `(name, metadata)` and has 3 callers (`SlackSyncService`, `AdminService`, `IdentityDiscoveryService`). Add a **new** method beside it to avoid touching the others:

- [BlobStorageService.cs](../../altus-ai-assistant/AltusAiShared/BlobStorageService.cs): `ListFilesWithProperties(containerName)` → `List<BlobItemInfo>`. Introduce a small **named type** `BlobItemInfo` (class or record, in `AltusAiShared`) rather than a tuple, so callers can grow without breaking the signature:

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

### 2. Table Storage state (provenance + change detection)
Reuse [AzureTableClientHelper.InitializeTableClient](../../altus-ai-assistant/AltusAiShared/Confluence/AzureTableClientHelper.cs) (blob account name + key from `AppSettings.BlobAccountName` / `BlobAccessKey`).

- New `Services/IdentityDiscoveryStateEntity.cs : ITableEntity`
  - `PartitionKey` = clientId; `RowKey` = `SHA256(blobPath)` hex (blob paths contain `/`, illegal in RowKey)
  - `BlobPath` (readable), `Source`, `LastModified` (DateTimeOffset, last processed), `ContentHash` (hex SHA256 of the last-extracted content), `SourceKeysJson` (JSON `string[]` = per-blob provenance list), `LastProcessedAt`, `FailureCount`, `LastError?`
- New `Services/IdentityDiscoveryStateRepository.cs` — `Get(clientId, blobPath)`, `Upsert(entity)`. Table name from a new `AppSettings` value (default `IdentityDiscoveryState`).

### 3. Make `IdentityDiscoveryService` incremental
Refactor [IdentityDiscoveryService.cs](../../altus-ai-assistant/AltusAiAssistantApi/Services/IdentityDiscoveryService.cs). New signature:

```csharp
Task<IdentityDiscoveryResultDto> RunDiscovery(int clientId, DateTimeOffset modifiedSince, CancellationToken ct);
```

`modifiedSince` is **required** (non-nullable) and acts as a **time-window prefilter** applied *before* any Table Storage work, so old blobs are cheap to ignore. There is no "process everything" magic value — to scan all blobs the caller passes a sufficiently old date (e.g. `2000-01-01`).
- **Guard the container first:** call `BlobStorageService.ContainerExists(client.UniqueId)` and return an empty result if it's missing — clients without a container (or with no synced sources yet) must not throw a 404 out of `GetBlobsAsync` on every tick. (See also the active-client filtering open item — this is the cheap defensive backstop regardless of how that query is scoped.)
- Use `ListFilesWithProperties`. Keep existing `.md`/`.html` + `TryMapSource` filtering.
- **If `blob.LastModified < modifiedSince` → skip immediately** (don't read content, don't even look up Table Storage). Count these as `BlobsOutsideWindow` (or fold into `BlobsSkipped`).
- Per remaining candidate: load state row. If `state != null && state.LastModified >= blob.LastModified` → **skip unchanged** (new `BlobsUnchanged` counter), no read, no GPT.
- Otherwise read content and compute `SHA256(content)` (hex). Reading is cheap; the GPT call is the cost we are protecting.
  - **If `state != null && state.ContentHash == hash`** → content is identical, only `LastModified` advanced (e.g. a metadata patch). **Skip extraction** (no GPT), advance the state row's `LastModified`/`LastProcessedAt`, leave `SourceKeysJson` as-is, and count as `BlobsUnchanged`.
  - **Else** → extract, `UpsertObservedIdentity` each (unchanged logic), then **write the state row** with new `LastModified` + new `ContentHash` + JSON list of extracted `sourceKey`s + `LastProcessedAt`.
- On read/extraction failure: log, **do not** update `LastModified` (next tick retries), bump `FailureCount`/`LastError`; skip the blob once `FailureCount` exceeds a small cap (poison guard, design §8). Provenance-only ⇒ never delete identities whose sourceKey disappeared.
- Optional `bool force` param to ignore state (full re-scan) for the manual endpoint.

### 4. Result DTO
[IdentityDiscoveryResultDto.cs](../../altus-ai-assistant/AltusAiAssistantApi/Models/Contacts/IdentityDiscoveryResultDto.cs): add `int BlobsUnchanged` (and `int BlobsOutsideWindow` if tracked separately from `BlobsSkipped`).

### 5. Scheduled runner + manual trigger
Split the **schedule** from the **work** so the same logic serves both the timer and the endpoint:

- New `IdentityDiscoveryRunner` (scoped service) with `RunAllClients(DateTimeOffset modifiedSince, ct)` — loads active clients from `AiAssistantDbContext` and loops them, calling `IdentityDiscoveryService.RunDiscovery(clientId, modifiedSince, ct)` in a per-client try/catch so one client's failure doesn't abort the run. This is the shared entry point.
- New `IdentityDiscoveryBackgroundService.cs : BackgroundService`, modeled on [GrainSynchronizer.cs](../../altus-ai-assistant/AltusAiAssistantApi/GrainSynchronizer.cs) but **schedule-only — no run on startup**:
  - `PeriodicTimer(TimeSpan.FromMinutes(15))`, loop on `await timer.WaitForNextTickAsync(ct)` so the **first** execution is one interval after boot (drop GrainSynchronizer's initial pre-loop `RunSync` call).
  - Gate on `appSettings.DiscoverIdentities` (early return when false), same as `SyncGrain`.
  - Per tick: `CreateScope()` → resolve `IdentityDiscoveryRunner` → `RunAllClients(DateTimeOffset.UtcNow.AddDays(-appSettings.DiscoveryLookbackDays), ct)`. Default `DiscoveryLookbackDays = 3`, so the scheduled job only considers blobs modified in the **last 3 days**.
- The schedule gate (`DiscoverIdentities`) only disables the **timer**; the manual endpoint stays callable regardless, so discovery can be triggered on demand even when the schedule is off.

### 6. Wiring
- [AppSettings.cs](../../altus-ai-assistant/AltusAiAssistantApi/AppSettings.cs): add `bool DiscoverIdentities`, `string IdentityDiscoveryStateTable`, and `int DiscoveryLookbackDays` (default `3`).
- [Program.cs](../../altus-ai-assistant/AltusAiAssistantApi/Program.cs): register `IdentityDiscoveryStateRepository` and `AddHostedService<IdentityDiscoveryBackgroundService>()` next to the existing `AddHostedService<GrainSynchronizer>()`. `BlobStorageService` + `IIdentityDiscoveryService` are already registered.
- `appsettings*.json`: add `"DiscoverIdentities": false` to Development/QA, `true` where desired in prod (mirror `SyncGrain` defaults).
- Manual `POST /api/clients/{clientId}/identities/discover` endpoint stays — the **on-demand trigger** for a single client, runs the same incremental `RunDiscovery`, and works independently of the schedule flag. Query params:
  - **`modifiedSince=<ISO date>` (required)** — same window prefilter as the service. Pass an old date (e.g. `2000-01-01`) for the initial full backfill.
  - **`forceUnchanged=true` (optional, default false)** — also bypass the per-blob Table Storage comparison, re-extracting even blobs whose `LastModified` is unchanged (maps to the service's `force` behavior).

### 7. Logging & observability
Logging reuses the existing **Serilog → Application Insights (traces)** sink ([Program.cs](../../altus-ai-assistant/AltusAiAssistantApi/Program.cs)); every class uses `ILogger<T>`, so `customDimensions.SourceContext` (the class name) is already filterable. A `BackgroundService` produces **no HTTP request**, so these land in `traces`, not `requests` — there's no `operation_Name` to filter on; use a logging scope instead.

- **Tag each run with a scope** at the two shared entry points (`IdentityDiscoveryRunner.RunAllClients` for scheduled, the endpoint for manual). Serilog promotes scope properties to `customDimensions`, and all child logs (per-client, per-blob) inherit them:

  ```csharp
  using (_logger.BeginScope(new Dictionary<string, object>
  {
      ["Job"] = "IdentityDiscovery",
      ["Trigger"] = "Scheduled",   // endpoint passes "Manual"
      ["JobRunId"] = Guid.NewGuid(),
  }))
  { /* run */ }
  ```

- **Log lines:** one Information **start** line and one Information **summary** line per client run (clientId + the `IdentityDiscoveryResultDto` counts); per-blob read/extraction failures at Warning; poison-skip at Warning/Error.
- **App Insights queries** (Logs / KQL on the `traces` table):

  ```kql
  // Scheduled job only
  traces
  | where customDimensions.Job == "IdentityDiscovery" and customDimensions.Trigger == "Scheduled"
  | order by timestamp asc

  // One run, end to end
  traces | where customDimensions.JobRunId == "<guid>" | order by timestamp asc

  // Summary lines only
  traces | where customDimensions.Job == "IdentityDiscovery" and message startswith "Identity discovery complete"
  ```

---

## Verification
1. `dotnet build AltusAiAssistantApi.sln` — no errors.
2. `dotnet test tests/AltusAiShared.Tests/AltusAiShared.Tests.csproj` — existing tests pass.
3. Set `DiscoverIdentities: true` locally, run the API. Confirm the background service does **not** run at startup, and fires for the first time ~15 min later. With the flag `false`, confirm the timer never runs but `POST .../identities/discover` still works on demand.
4. First run: unassigned identities appear via `GET /api/clients/{clientId}/identities?unassigned=true`; a state row per processed blob exists in `IdentityDiscoveryState` with populated `SourceKeysJson`.
5. Second run, no blob changes: `BlobsUnchanged` = candidate count, `IdentitiesCreated` = 0, no GPT calls.
6. **Changed content:** re-upload one blob with edited content (bumps `LastModified`, changes the hash): only that blob re-extracts next tick; its state row's `LastModified`, `ContentHash`, and `SourceKeysJson` update.
6a. **Metadata-only change:** patch a blob's metadata without touching content (bumps `LastModified`, hash unchanged): next tick reads it but makes **no GPT call**; the state row's `LastModified` advances while `ContentHash`/`SourceKeysJson` stay the same. Counts as `BlobsUnchanged`.
7. **Window:** a blob older than 3 days that was never processed is ignored by the scheduled run (no Table Storage lookup, no GPT). Calling the endpoint with `?modifiedSince=2000-01-01` picks it up — confirm the initial full backfill works this way.
8. **Logging:** locally (console sink) confirm each run emits start + summary lines carrying `Job`/`Trigger`/`JobRunId`; in App Insights confirm `traces | where customDimensions.Trigger == "Scheduled"` returns only timer runs and excludes endpoint-triggered ones.

---

## Open items to refine
- **Concurrent runs vs. the `(ClientId, Source, SourceKey)` unique index** *(known issue — to be resolved separately, not in this plan)*. `UpsertObservedIdentity` is a check-then-insert with no catch ([ContactsService.cs](../../altus-ai-assistant/AltusAiAssistantApi/Services/ContactsService.cs)), but `ContactIdentity` has a **unique index** on `(ClientId, Source, SourceKey)` ([ContactIdentityConfiguration.cs](../../altus-ai-assistant/AltusAiAssistantApi/Data/ContactIdentityConfiguration.cs)). v1 introduces a realistic overlap — a long manual backfill (`modifiedSince=2000-01-01`) running while a 15-min timer tick fires for the same client — where two runs extract the same person, both miss the existing-row check, and the second `SaveChanges` throws on the constraint. Two concurrent runs also last-write-wins on the same Table Storage state row. Not addressed here; resolution (per-client single-flight lock, and/or catching `DbUpdateException` + re-query in `UpsertObservedIdentity`) is tracked separately.
- **Active-client query** — confirm the exact source of "clients to scan" (all `Clients` with `InactiveDate IS NULL`, or only those with relevant blob containers). Independent of this, `RunDiscovery` already guards with `ContainerExists` (§3) so a container-less client is a cheap no-op rather than a per-tick 404.
- **Interval** — 15 min fixed vs. configurable (`appsettings`). Decided: **no run-on-startup**; first run is one interval after boot. If a clock-aligned schedule (e.g. on the quarter-hour) is wanted instead of a boot-relative interval, that needs a cron mechanism rather than `PeriodicTimer`.
- **Poison cap** — value for `FailureCount` max and whether to surface an alert (design §8 mentions alerting).
- **`force` full-rescan** — keep as query param on the manual endpoint, or omit for now.
- **Window param shape** — `modifiedSince=<date>` (chosen, required, matches "any modification date") vs. a `lookbackDays=N` int (closer to the existing `lookbackHours` param on `SlackSync` `DeepSyncHttp`).
- **Provenance reads** — whether any UI/endpoint will surface `SourceKeysJson`, or it stays internal for now.
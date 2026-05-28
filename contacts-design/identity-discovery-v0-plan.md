# Identity Discovery v0 — Implementation Plan

Companion to [identity-discovery-design.md](identity-discovery-design.md).

## Context

V0 is a manually-triggered API endpoint that scans all markdown blobs whose `sourceType` metadata is `Slack`, `Grain`, or `Email`, extracts people via GPT, and creates unassigned `ContactIdentity` rows for user review in the existing unassigned panel.

No date filtering. No state tracking. Idempotency comes entirely from `UpsertObservedIdentity` — running the endpoint twice produces the same result. Scope is limited to Slack, Grain, and Email — the sources where identity extraction from content adds the most value (Slack message authors are inline only; Grain transcripts carry speaker names throughout; Email bodies carry sender/recipient names and signatures).

No changes to `BlobStorageService` are needed — the existing `ListFiles(containerName)` returns all blobs with metadata, which is sufficient.

---

## Files to Modify or Create

### 1. `AltusAiAssistantApi/Services/IContactsService.cs` + `ContactsService.cs`

Add `UpsertObservedIdentity`:

```csharp
Task<ContactIdentityDto> UpsertObservedIdentity(
    int clientId,
    IdentitySource source,
    string sourceKey,
    string? displayLabel,
    CancellationToken ct);
```

Implementation:
1. Query for existing `ContactIdentity` by `(clientId, source, sourceKey)`.
2. If found: update `DisplayLabel` only if the new label is longer (prefer "Alex Johnson" over "Alex"); save and return.
3. If not found: call existing `CreateIdentity` with `ContactId = null` (unassigned).

---

### 2. NEW `AltusAiAssistantApi/Services/IIdentityExtractionService.cs` + `IdentityExtractionService.cs`

Single responsibility: given markdown content and a source type, call GPT and return extracted identities.

```csharp
public record ExtractedIdentity(string SourceKey);

public interface IIdentityExtractionService {
    Task<List<ExtractedIdentity>> ExtractIdentities(
        string markdownContent, IdentitySource source, CancellationToken ct);
}
```

Uses the Azure OpenAI client already in DI (same as `ChatService`). Prompt: see [identity-discovery-design.md §4](identity-discovery-design.md). Parses the JSON array response; logs and returns empty list on parse failure (blob is counted as skipped).

---

### 3. NEW `AltusAiAssistantApi/Services/IIdentityDiscoveryService.cs` + `IdentityDiscoveryService.cs`

Orchestrates one discovery run for a client:

```csharp
Task<IdentityDiscoveryResultDto> RunDiscovery(int clientId, CancellationToken ct);
```

Steps:
1. Look up `client.UniqueId` to get the blob container name.
2. Call existing `BlobStorageService.ListFiles(containerName)` — returns all blobs with metadata.
3. For each blob:
   - Read `sourceType` from blob metadata; map to `IdentitySource` (`Slack`/`Grain`/`Email` → corresponding enum value; `SlackFile`/`EmailFile` and anything else → skip).
   - Skip non-`.md` files.
   - Read content via `BlobStorageService.GetFile`.
   - Call `IdentityExtractionService.ExtractIdentities`.
   - For each extracted identity: call `ContactsService.UpsertObservedIdentity`.
4. Accumulate counts; return `IdentityDiscoveryResultDto`.

---

### 4. NEW `AltusAiAssistantApi/Models/Contacts/IdentityDiscoveryResultDto.cs`

```csharp
public class IdentityDiscoveryResultDto {
    public int BlobsScanned { get; set; }      // Slack/Grain/Email .md blobs processed
    public int BlobsSkipped { get; set; }      // unknown sourceType, non-.md, or GPT parse errors
    public int IdentitiesFound { get; set; }   // total extracted by GPT across all blobs
    public int IdentitiesCreated { get; set; } // net new ContactIdentity rows inserted
}
```

---

### 5. `AltusAiAssistantApi/Controllers/ContactsController.cs`

Add one action:

```
POST /api/clients/{clientId}/identities/discover
```

- No query parameters.
- Runs synchronously; returns `200 IdentityDiscoveryResultDto`.
- No background queuing in v0 — synchronous is acceptable at bounded blob counts.

---

### 6. `AltusAiAssistantApi/Program.cs`

Register new services:

```csharp
services.AddScoped<IIdentityExtractionService, IdentityExtractionService>();
services.AddScoped<IIdentityDiscoveryService, IdentityDiscoveryService>();
```

---

## What Is NOT in v0

- No `ContactIdentityEvidence` table (deferred).
- No date / ETag filtering — full scan every run; idempotency via upsert.
- No timer-based or EventGrid automatic runs (Azure Function, v1).
- Confluence blobs not in scope — too little identity signal in their content for v0.

---

## Verification

1. `dotnet build AltusAiAssistantApi.sln` — no errors.
2. `dotnet test` — existing tests pass.
3. Manual: POST the endpoint for a client that has Slack or Grain blobs. Confirm unassigned identities appear under `GET /api/clients/{clientId}/identities?unassigned=true`.
4. POST again — `IdentitiesCreated` = 0, `IdentitiesFound` unchanged (upsert idempotency).

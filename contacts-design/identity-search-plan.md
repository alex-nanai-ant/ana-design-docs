# Identity Search — Implementation Plan

Companion to [identity-discovery-design.md](identity-discovery-design.md), [contacts-api.md](contacts-api.md), and [contacts-entity-model.md](contacts-entity-model.md).

## Context

The discovery flow creates a `ContactIdentity` per person it sees in ingested content. From the unassigned panel and the contact detail view the user needs to answer **"where in the content was this person mentioned?"** — scoped to a single identity, to verify a discovery before assigning it and to inspect the evidence behind any identity attached to a contact. Contact-level "show me everything across all identities" is deferred; the FE can fan out per-identity calls if it needs the union.

We're solving this with plain search over the existing Azure Cognitive Search index — not RAG, and not by capturing extraction-time evidence. Rationale and the search-vs-RAG trade-off live in conversation history; the short version:

- Manual identities have no extraction event to attach evidence to — search covers both auto and manual uniformly.
- The current discovery prompt doesn't return quotes; adding that means prompt redesign + hallucination validation.
- The content is already indexed per-client; reusing the same `{clientUniqueId}-index` is free.
- Verification UX needs ground-truth snippets, not paraphrased answers — LLM-in-the-loop is the wrong tool here.

The FE will reuse the chat-context layout ([Chat-context.tsx](../../altus-ai-assistant-fe/src/components/dashboard/right-panel/chat-context/Chat-context.tsx) + [getContextGroup](../../altus-ai-assistant-fe/src/libs/helpers/file.ts)), so the DTO shape mirrors `MessageContextItemDto` and adds a `Highlights` field — the missing piece that makes "find the name in a long Grain transcript" actually usable.

---

## Endpoints

One route, hung off `ContactsController` (base route `api/clients/{clientId:int}`).

| Method | Route | Returns | Notes |
|---|---|---|---|
| GET | `/api/clients/{clientId}/identities/{identityId}/mentions` | `IdentityMentionDto[]` | Mentions of a single identity. |

Query parameters:
- `?sourceType=Slack,Grain,Email,Confluence` — optional filter (comma-separated `DocumentSourceType` values).
- `?take=` — default 50, max 200. No `skip` in v1; results are ranked, and pagination is deferred.

Status codes: 200 on success (empty array if no hits), 404 if the identity isn't in the client, 403 if the caller isn't a member of `{clientId}`.

---

## DTO

`IdentityMentionDto` is shape-compatible with [`MessageContextItemDto`](../AltusAiAssistantApi/Models/ChatResponseDto.cs) on the shared fields so the FE can reuse `getContextGroup` and the existing chunk-card layout. The one new field is `Highlights` — the actual snippet to render.

```csharp
public class IdentityMentionDto {
    public string Content { get; set; } = string.Empty;        // full chunk text (kept for "expand" UX, hidden by default)
    public string FileUrl { get; set; } = string.Empty;
    public string? ExternalUrl { get; set; }
    public DocumentSourceType? SourceType { get; set; }
    public string? Author { get; set; }
    public string? SlackChannelName { get; set; }

    public List<string> Highlights { get; set; } = [];          // Azure Search @search.highlights — text with <em>…</em> markers
    public double Score { get; set; }                           // search score, for FE sort/debug
}
```

Do not reuse the literal `MessageContextItemDto` type — keeping the types distinct prevents the chat contract and the identity-search contract from co-evolving accidentally. Shape match is intentional; identity match is not.

---

## Service surface

```csharp
public interface IIdentityMentionsService {
    Task<List<IdentityMentionDto>> FindMentionsForIdentity(
        int clientId, int identityId,
        List<DocumentSourceType>? sourceTypes, int take,
        CancellationToken ct);
}
```

Implementation outline:

1. Load the identity from the DB and verify `clientId` membership; 404 otherwise.
2. Build a `SearchQuery` for the identity (see next section).
3. Execute the Azure Search call against the existing per-client `{clientUniqueId}-index`.
4. Map each `SearchResult<SearchDocument>` to `IdentityMentionDto`, populating `Highlights` from `result.Highlights["content"]` (plus `title`/`author` if present).
5. Sort by `Score` descending, truncate to `take`.

DI registration in [Program.cs](../AltusAiAssistantApi/Program.cs):
```csharp
services.AddScoped<IIdentityMentionsService, IdentityMentionsService>();
```

---

## Search query construction

Per identity, build:

**Query text.** Combine `displayLabel` and `sourceKey` if both present and different:
```
"Alex Johnson" OR "alex.johnson@altus.com"
```
Use quoted phrase matching to avoid breaking multi-word names into independent tokens. If only `sourceKey` is present, use it alone. If `sourceKey` is opaque (e.g., a Slack user ID like `U07ABC123`), the metadata filter below carries the search — the content query may return nothing, which is fine.

**Source-type filter.** If the caller passed `sourceType`, OR them into an OData filter:
```
sourceType eq 'Slack' or sourceType eq 'Grain'
```

**Metadata filters (additive, OR-ed with content match via a follow-on query).** For identities whose `sourceKey` matches index metadata, run a *second* search restricted to metadata equality:

| `IdentitySource` | Filter |
|---|---|
| `Email` | `author eq '{sourceKey}'` or `mailboxAddress eq '{sourceKey}'` |
| `Slack` | `author eq '{sourceKey}'` (Slack author is the display name in the indexed metadata; v0 accepts this is best-effort for Slack IDs) |
| `Grain` / `Confluence` / `Phone` / `Unknown` | no metadata filter; content match only |

Merge the two result sets by `chunk_id`, summing scores so metadata-confirmed hits rank above content-only hits.

**SearchOptions.** Mirror the chat-time options ([ChatService.GetFromSearchIndex](../AltusAiAssistantApi/Services/ChatService.cs)) with two changes:
- `QueryType = SearchQueryType.Simple` — keyword search only; **do not run a vector query**. Vector search adds latency, an embedding call, and noise for a name-finding workload where the query terms are already literal.
- `HighlightFields = "content,title,author"`, `HighlightPreTag = "<em>"`, `HighlightPostTag = "</em>"` — produces the snippets the FE renders.
- `Size = take` (capped at 200).
- Keep the existing `ScoringProfile = "CustomScoringProfile"` so freshness boost still applies — newer mentions surface first.

---

## Files to modify or create

### 1. NEW `AltusAiAssistantApi/Models/Contacts/IdentityMentionDto.cs`
Shape above.

### 2. NEW `AltusAiAssistantApi/Services/IIdentityMentionsService.cs` + `IdentityMentionsService.cs`
Interface + implementation as outlined. Takes `IOptions<AppSettings>`, `AiAssistantDbContext`, `IContactsService`, `ILogger<IdentityMentionsService>` via DI. Constructs `SearchClient` per call the same way `ChatService` does.

### 3. `AltusAiAssistantApi/Controllers/ContactsController.cs`
Add one action to the existing controller (base route `api/clients/{clientId:int}`) — inject `IIdentityMentionsService` alongside the existing services. Route is relative to the controller route and lives in the `#region Identities` block:

```csharp
[HttpGet("identities/{identityId:int}/mentions")]
public Task<IActionResult> GetIdentityMentions(int clientId, int identityId, [FromQuery] string? sourceType, [FromQuery] int take = 50, CancellationToken ct = default);
```

Parses `sourceType` from CSV → `List<DocumentSourceType>`, clamps `take` to `[1, 200]`, calls the service, returns `Ok(result)`.

### 4. `AltusAiAssistantApi/Program.cs`
Register the new service.

### 5. No changes to `IndexingService`
The index schema already has `chunk_id`, `content`, `sourceType`, `author`, `mailboxAddress`, `slackChannelName`, `title`, `url`, `externalUrl`, `date` — everything we need. Highlighting works on `SearchableField` out of the box; no schema rebuild required.

---

## What is NOT in v1

- **No vector / hybrid search.** Keyword is sufficient and cheaper. Revisit only if there's a concrete miss in production (e.g., a person consistently referenced via nicknames the discovery prompt didn't catch).
- **No "find all other mentions" UI affordance on chat citations.** Out of scope; this is a contacts-panel feature.
- **No caching.** Searches run on every request. Add a short-TTL cache (`(identityId, sourceType, take) → results`, 60s) if the panel turns out to trigger repeat calls.
- **No skip/take pagination.** Top-N with `take` only. Add when the panel needs "show more."
- **No contact-level mentions route.** The FE fans out per-identity calls if it needs the union across a contact's identities. Add a server-side `/contacts/{id}/mentions` only if that fan-out causes a real problem (request count, dedupe complexity).
- **No FE work in this plan.** The FE deltas are tracked separately — minimum is a new right-panel view that calls the endpoint and renders `IdentityMentionDto[]` through an adapted `Chat-context.tsx` that prefers `Highlights` over `Content`.

---

## Verification

1. `dotnet build AltusAiAssistantApi.sln` — no errors.
2. `dotnet test` — existing tests pass.
3. Manual: in a client with known Slack/Grain/Email content for an identifiable person:
   - `GET /clients/{clientId}/identities/{identityId}/mentions` returns hits with non-empty `Highlights` and the expected `SourceType` mix.
   - `?sourceType=Slack` reduces the set to Slack-only.
4. Manual: query for an identity whose `sourceKey` is an opaque Slack user ID and confirm 200 with empty results (no crash, no 500).
5. Cost / latency check: each call is one Azure Search call — no GPT calls. Confirm p95 latency under ~300ms.

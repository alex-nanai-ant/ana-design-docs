# Contact / Company Description Generation — API Design (Draft)

LLM-backed "Generate description" affordance for the Contact and Company forms. Extends the directory API in [contacts-api.md](contacts-api.md); follows repo conventions (DTO suffix, `[Authorize]`, no `Async` method suffix).

## Goal

The Contact and Company configuration forms each carry a free-text `Description` (Contact) / `RelationDescription` (Company) field. Authoring that text from scratch is the friction point — the consultant typically *has* the source material (Grain calls, Slack threads, Confluence pages, emails mentioning the person or company) but doesn't want to summarize it by hand.

A "Generate description" button calls this endpoint, which retrieves grounded snippets from the indexed corpus and asks the LLM for a draft. The endpoint **does not mutate the entity** — it returns suggested text that the FE drops into the form textarea. The user reviews/edits and saves through the normal PUT in [contacts-api.md](contacts-api.md).

One endpoint per entity type. The request body discriminates between the two flows:
- **Existing entity** — caller sets `ContactId` (or `CompanyId`); the server loads the row, reads its identities (contact) or stored name (company), and searches.
- **Pre-save / create flow** — caller leaves the id null and provides the typed-in fields directly (`Name` + optional `Role` / `CompanyId` / `IdentityIds` for contact; `Name` for company). Useful when the user wants a draft to seed the form before clicking Save.

## Endpoints

| Method | Route | Body | Returns | Notes |
|---|---|---|---|---|
| POST | `/api/clients/{clientId}/contacts:generateDescription` | `GenerateContactDescriptionDto` | `GeneratedDescriptionDto` | Either `ContactId` (existing contact — server reads its identities) or `Name` + optional `IdentityIds` (draft). 422 if no identities resolved or search returns nothing. |
| POST | `/api/clients/{clientId}/companies:generateDescription` | `GenerateCompanyDescriptionDto` | `GeneratedDescriptionDto` | Either `CompanyId` (existing company — server reads its stored name) or `Name` (draft). 422 if no content matches. |

Action-style suffix (`:generateDescription`) over `POST .../generate-description` — this is an RPC, not a new sub-resource. Collection-scoped (no `{id}` in the path) because the action covers both existing rows and not-yet-persisted drafts; the discriminator lives in the body. The path stays the same regardless of flow, so the FE doesn't need to branch on URL.

## DTOs

```csharp
public class GenerateContactDescriptionDto {
    // Discriminator: exactly one of (ContactId) vs (Name [+ optional Role/CompanyId/IdentityIds]) must be provided.
    public int? ContactId { get; set; }         // existing contact — server reads its identities from DB

    public string? Name { get; set; }           // draft mode: required when ContactId is null
    public string? Role { get; set; }           // draft mode: optional, used in the system prompt
    public int? CompanyId { get; set; }         // draft mode: optional, used in the system prompt
    public List<int> IdentityIds { get; set; } = []; // draft mode: identities the user has already picked in the form

    public string? Instructions { get; set; }   // optional steering, e.g. "focus on technical expertise" or "keep to one sentence"
    public int? MaxSources { get; set; }        // default 10, hard cap 20
}

public class GenerateCompanyDescriptionDto {
    // Discriminator: exactly one of (CompanyId) vs (Name) must be provided.
    public int? CompanyId { get; set; }         // existing company — server reads its stored name
    public string? Name { get; set; }           // draft mode: required when CompanyId is null

    public string? Instructions { get; set; }
    public int? MaxSources { get; set; }
}

public class GeneratedDescriptionDto {
    public string Description { get; set; } = string.Empty;
    public List<DescriptionSourceDto> Sources { get; set; } = []; // documents used to ground the draft, for FE attribution
}

public class DescriptionSourceDto {
    public string DocumentId { get; set; } = string.Empty;
    public string Title { get; set; } = string.Empty;
    public DocumentSourceType SourceType { get; set; }            // Confluence | Grain | Slack | Email | ...
    public string Snippet { get; set; } = string.Empty;           // ~200 chars; the chunk actually passed to the model
}
```

`DocumentSourceType` is the existing enum from [CLAUDE.md](../CLAUDE.md). No new types.

## Behavioral details

1. **No persistence.** Each call is a fresh draft; calling twice is safe and will likely yield different text. No server-side caching in v1 — repeated clicks burn tokens, so the FE should debounce / disable the button while a request is in flight.
2. **Source retrieval.**
   - **Contact:** identities are resolved first — when `ContactId` is set, loaded from the contact's stored assignments; otherwise taken from `IdentityIds` in the body. Azure Cognitive Search query is then built from those identities — `senderEmail in (...)`, `speakerName in (...)`, `slackUserId in (...)`, etc., per the indexed fields exposed in [IndexingService.cs](../AltusAiAssistantApi/Services/IndexingService.cs). All identities are unioned (OR across identity values). Top-*N* chunks by hybrid score (semantic + keyword), capped at ~6K input tokens of grounded context (consistent with `Constants` max-context guidance — see [CLAUDE.md](../CLAUDE.md)).
   - **Company:** name is resolved first — when `CompanyId` is set, loaded from the row; otherwise taken from `Name` in the body. Text search for the company name across indexed content; same chunk cap. Optional refinement: also pull contacts whose `CompanyId` matches and union their identity-keyed content (only viable in the existing-entity flow — skipped for drafts).
3. **LLM seam.** Same `IChatClient` (`Microsoft.Extensions.AI`) wiring as [IdentityExtractionService.cs](../AltusAiAssistantApi/Services/IdentityExtractionService.cs). System prompt embeds name + role + company (for a contact) or name (for a company), the retrieved snippets, and any caller `Instructions`. Output capped at ~500 tokens (one paragraph). No function-calling.
4. **Synchronous response.** Plain JSON, not streamed. Description generation is bounded and short enough (~2–5s) that SignalR streaming isn't worth the wiring. The form just shows a spinner.
5. **422 vs 500.** 422 means "the entity exists but the index has nothing to draw on" — the FE renders "no content yet to summarize, fill in manually". 500 stays for unexpected LLM / search failures.
6. **No mutation.** The endpoint never writes to the contact or company row. Save flows through the normal PUT.
7. **Cost control.** `MaxSources` lets the FE shrink the prompt for cheap re-rolls. No per-client rate limit in v1; revisit if a user spam-clicks.
8. **Auth.** `[Authorize]` + the same `clientId` membership check as the rest of the directory API. A user who can't see the contact / company can't generate a description for it.

## Status codes

| Code | When |
|---|---|
| 200 | Successful generation |
| 400 | Validation failure: neither nor both of (id / name) provided, draft missing `Name`, `MaxSources` out of range, etc. |
| 401 / 403 | No auth / not a member of `{clientId}` |
| 404 | `ContactId` / `CompanyId` provided but not found in client (only possible in the existing-entity flow) |
| 422 | No identities resolved (contact, either flow) / no indexed content matches — nothing to summarize |
| 500 | LLM or search failure |

## Service surface

Lives on `IContactsDirectoryService` (already defined in [contacts-api.md](contacts-api.md)) — composes existing search + chat-client seams; nothing here justifies a new top-level service.

```csharp
Task<GeneratedDescriptionDto> GenerateContactDescription(
    int clientId, GenerateContactDescriptionDto input, string userId, CancellationToken ct);

Task<GeneratedDescriptionDto> GenerateCompanyDescription(
    int clientId, GenerateCompanyDescriptionDto input, string userId, CancellationToken ct);
```

The service inspects `input` and branches: if `ContactId` / `CompanyId` is set, load the row (404 on miss) and pull identities/name from it; otherwise use the draft fields directly. Both branches converge on the same retrieval + LLM call.

`userId` is threaded through purely for logging / audit trail of who asked for the draft — it does not influence the prompt.

## Open questions

- **Caching.** Whether to cache the last generated draft per (clientId, entityId) for ~5 min so a user reopening the form sees the same suggestion without a re-spend. Probably not worth it; flag if token cost climbs.
- **Streaming.** Whether to switch to SignalR streaming so the user sees text appear progressively. Only if 2–5s synchronous latency proves jarring in testing.
- **Company refinement.** Whether company description generation should also pull content via the identities of contacts assigned to that company (richer signal) or stay name-match only (simpler).
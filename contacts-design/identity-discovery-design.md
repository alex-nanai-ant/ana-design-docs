# Automatic Identity Discovery — Design

Companion to [contacts-proposal.md](contacts-proposal.md), [contacts-entity-model.md](contacts-entity-model.md), and [contacts-api.md](contacts-api.md).

## 1. Problem

The sync workers write markdown blobs to Azure Blob Storage but do not capture identity data systematically:

| Source | What's available in metadata/structure | What's missing |
|---|---|---|
| Slack messages | Authors embedded inline in markdown (`**[ts] Name:** msg`) | Nothing in metadata |
| Slack files | `author` metadata = uploader display name | Other participants |
| Confluence | No author/contributor data at all | Everything |
| Email | `author` metadata = From address | To, CC, names in body |
| Grain | Unknown (transcripts are manually uploaded markdown) | Speakers, attendees |

Since these blobs are exactly what gets searched for RAG, identity discovery must work from the blob content itself. Format differences between sources don't matter — AI extraction is format-agnostic.

**Goal:** after a markdown blob is created or updated, run AI extraction over the content and surface every identified person as an unassigned `ContactIdentity` for user review. This is the primary (and only) mechanism for populating the unassigned identities panel.

---

## 2. Architecture Overview

```
Blob created/updated (any source)
      │
      ▼
AltusAiIdentityDiscovery  (new Azure Function)
      │
      ├─ Determine: clientId + IdentitySource from blob path/metadata
      ├─ Skip if content hash unchanged since last run
      ├─ Call GPT: extract persons from markdown content
      │
      └─ For each extracted person:
            ContactsService.UpsertObservedIdentity(
                clientId, source, sourceKey, displayLabel, ct)
            → inserts new (unassigned) ContactIdentity, or returns existing row
            + insert IdentityEvidence row (snippet + blob path)
```

The discovery function is a new caller of the existing `ContactsService` (which already has `CreateIdentity`/`UpdateIdentity`). It adds one new method — `UpsertObservedIdentity` — to avoid forcing the function to handle 409s from race conditions.

---

## 3. Trigger Mechanism

**Option: timer-based scan (recommended for v1).** Every 15 minutes, scan the blob container for items whose `LastModified` is newer than the last successful run timestamp. Simpler to deploy than EventGrid subscriptions, and identity review is not time-critical enough to need real-time triggering.

**Option: EventGrid (v2).** If near-real-time discovery becomes needed, EventGrid on the blob containers feeds a Storage Queue; the function consumes from the queue. Handles backpressure during bulk initial loads.

**Path-to-metadata mapping.** The blob path and blob metadata (`sourceType` field) are used to determine `clientId` and `IdentitySource`. Blob path conventions from existing sync workers:

```
{containerName}  ←  maps to clientId via existing container→client lookup
  Slack/{channel}/{chunk}.md             → IdentitySource.Slack
  Slack/{channel}/{date}/{file}          → IdentitySource.Slack
  Email/{date} ({from})/{subject}/...   → IdentitySource.Email
  {SpaceKey}/{page}.md                   → IdentitySource.Confluence
  {anything else}                        → IdentitySource.Unknown (skip or treat as Grain)
```

Grain blobs are manually uploaded — their path convention needs to be established. Until then, treat unrecognized paths as `Unknown`.

**Change detection.** A `IdentityDiscoveryState` table in Azure Table Storage (PartitionKey = clientId, RowKey = SHA256(blobPath)) stores the last-processed blob ETag. If ETag unchanged, skip. This mirrors the pattern already used by the existing sync workers.

---

## 4. AI Extraction

**One GPT call per blob.** The entire markdown content is sent for extraction. For blobs exceeding a configurable token limit (default 24K tokens), split at `## ` section boundaries and merge results by deduplicating on `sourceKey`.

### Extraction Prompt

Extraction targets **participants only** — people who personally spoke, wrote, sent, or received the item. People who are merely talked about (even real, work-relevant colleagues or clients), public figures, passing/small-talk mentions, generic roles, and non-person names (products, tools, companies, places) are all excluded. Restricting to participants — anchored by the structural signals each source provides — was the change that cut the false-positive noise; "people discussed in the work" proved too broad in practice. `{SourceNoun}` and `{SourceHint}` are filled per `IdentitySource` so the prompt names the document type and points at where participants appear (Grain `**Name:**` speaker labels, Slack `**[ts] Name:**` authors, Email From/To/CC headers).

```
You are extracting the participants of a {SourceNoun} — the people who personally
spoke, wrote, sent, or received it. Return ONLY a JSON array — no markdown, no explanation.

Each array element is an object with one field:
  "sourceKey" – exactly as this person appears most often in the document
                (e.g. "Alex Johnson", "Alex", "alex@example.com", "+15125551234")

{SourceHint}

INCLUDE only participants — a person who personally contributed to this {SourceNoun}
(spoke, authored a message, or was a sender or recipient).

EXCLUDE everyone else, even if they are real people relevant to the work:
  - Anyone who is only talked about, quoted, or referenced but did not themselves
    participate (e.g. a colleague or client the participants discuss).
  - Public or well-known figures, and people mentioned in jokes, small talk,
    hypotheticals, or off-topic asides.
  - Generic or anonymous references with no specific named person
    ("the team", "everyone", "the client", "the Secretary of Defense").
  - Names that are not people: products, companies, tools, models, or places
    (e.g. "ANA", "FinMed", "Optimizer", "Ollama", "Llama 3", "Amazon Bedrock", "AWS").

For every name, confirm the person actually participated before adding it. When
unsure whether someone participated or was merely mentioned, leave them out.

Rules:
  - Consolidate all mentions of the same person into one entry.
  - If a participant appears only as a phone number or email with no associated
    name, use it as the sourceKey.

Document type: {sourceType}
---
{markdownContent}
```

Note: `email`, `phone`, and `context` are intentionally omitted from the v0 extraction. `context` will be added back when `ContactIdentityEvidence` is built (see §6).

### Source Key Strategy

`(ClientId, IdentitySource, SourceKey)` is the natural key on `ContactIdentity` that prevents duplicates across documents.

| What the AI found | SourceKey |
|---|---|
| Person always shown by email | `alex@example.com` |
| Person shown by full name | `Alex Johnson` |
| Person shown by first name only | `Alex` |
| Phone only | `+15126945346` |

`sourceKey` is always the most common appearance in the document — no normalization or inference. The same person appearing as "Alex" in one document and "Alex Johnson" in another produces two separate `ContactIdentity` rows, which the reviewer can merge via assignment.

When the same person appears in multiple documents, subsequent runs hit the existing `ContactIdentity` row (via `UpsertObservedIdentity`) and only add a new `IdentityEvidence` record pointing to the new document. No duplicate identities are created.

---

## 5. Changes to `ContactsService`

Add one method to `IContactsService` (the existing service interface):

```csharp
// Internal — called by AltusAiIdentityDiscovery. NOT exposed via HTTP.
Task<ContactIdentityDto> UpsertObservedIdentity(
    int clientId,
    IdentitySource source,
    string sourceKey,
    string? displayLabel,
    CancellationToken ct);
```

Implementation:
1. Try to load existing `ContactIdentity` by `(clientId, source, sourceKey)`.
2. If found: update `DisplayLabel` if the new label is longer/more specific; return existing row.
3. If not found: insert new `ContactIdentity` with `ContactId = null` (unassigned).

This is the only addition to the existing contacts service required for discovery. The rest of the identity lifecycle (review, assign, merge) already exists or is covered by the API design.

---

## 6. Entity Model Additions

### New table: `ContactIdentityEvidence`

Stores the snippets that tell the user *why* an identity was discovered. One identity can appear across many documents — this table is one-to-many from `ContactIdentity`.

```csharp
public class ContactIdentityEvidence : BaseEntity {
    public int ContactIdentityId { get; set; }
    public ContactIdentity ContactIdentity { get; set; } = null!;

    public string BlobPath { get; set; } = string.Empty;
    public IdentitySource Source { get; set; }              // denormalized for display
    public string ExtractedContext { get; set; } = string.Empty;  // ≤ 120-char snippet
    public DateTime DiscoveredAt { get; set; }
}
```

**Why separate from `ContactIdentity`?** An identity in three Grain transcripts produces three evidence rows. Embedding in `ContactIdentity` would require a JSON column or parallel columns.

**Deferrable:** The unassigned panel works without evidence. Defer this table if the review UX is acceptable with just `displayLabel` + `source`. Add it when user feedback shows reviewers need context to recognize who an identity is.

### `IdentityDiscoveryState` (Azure Table Storage)

Not a SQL table — same approach as sync workers. Stores last-processed ETag per blob to make the timer scan idempotent.

---

## 7. API Additions

### Evidence on identity reads

Add optional evidence embedding to `ContactIdentityDto`, returned only when `?includeEvidence=true` is passed to `GET /identities`. Avoids inflating the default unassigned panel response.

```csharp
public class ContactIdentityEvidenceDto {
    public string BlobPath { get; set; } = string.Empty;
    public string ExtractedContext { get; set; } = string.Empty;
    public DateTime DiscoveredAt { get; set; }
}

// Added to ContactIdentityDto (nullable — absent unless requested):
public List<ContactIdentityEvidenceDto>? Evidence { get; set; }
```

No other API changes are required. The existing `GET /identities?unassigned=true` already drives the review panel. Assignment flow (`PUT /identities/{id}` with a `contactId`) is unchanged.

---

## 8. New Azure Function: `AltusAiIdentityDiscovery`

New project following the same structure as `AltusAiConfluenceSync`, `AltusAiSlackSync`, etc.

```
AltusAiIdentityDiscovery/
  IdentityDiscoveryFunction.cs       — timer-triggered entry point
  IdentityExtractionService.cs       — GPT call + JSON response parsing
  BlobPathParser.cs                  — blob path → (clientId, IdentitySource)
  DiscoveryStateRepository.cs        — Table Storage ETag tracking
```

The function takes `IContactsService` via DI — the same interface used by the API controllers.

**Error handling:** GPT parse failures (malformed JSON, refusal) are logged and the blob ETag is NOT updated, so the next timer run retries. After 3 consecutive failures on the same blob, write a poison marker to Table Storage and alert (same pattern as existing sync workers).

---

## 9. Cost and Rate Considerations

**Per-file cost.** One GPT-4o call per changed blob. Typical Grain transcript ~3K tokens input + ~500 tokens output ≈ $0.01–0.03 per file at current rates.

**Initial load.** All existing blobs processed on first deploy. Throttle via `maxConcurrentFunctions` setting. 500 blobs × $0.02 ≈ $10 one-time cost.

**Steady-state.** Only blobs with changed ETags are processed. Low frequency after initial load.

**Large blobs.** Confluence pages can be large. If content exceeds token limit, split at `## ` boundaries and merge. Each split = one additional GPT call. Log split counts to estimate actual costs in production.

---

## 10. Open Questions

- **Grain blob path convention.** Grain transcripts are manually uploaded — their container/path structure isn't defined yet. Need to establish this before the `BlobPathParser` can map them to `IdentitySource.Grain`.
- **`IdentityEvidence` in v1 or deferred.** Build `UpsertObservedIdentity` and extraction first; add evidence table if reviewers say they can't recognize identities without context snippets.
- **Timer vs. EventGrid.** Timer scan every 15 minutes is simpler for v1. Revisit if the delay is unacceptable in practice.
- **Display label update policy.** When `UpsertObservedIdentity` finds an existing row, should it update `DisplayLabel` to the new value, keep the existing, or prefer longer? Current proposal: prefer the longer/more specific label (e.g., "Alex Johnson" beats "Alex").

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
Scheduled tick (in-API BackgroundService, every 15 min)
      │
      ▼
For each active client → IdentityDiscoveryService.RunDiscovery
      │
      ├─ List blobs; map IdentitySource from `sourceType` metadata
      ├─ Skip blobs outside the modified-since window
      ├─ Skip if blob LastModified unchanged since last run (Table Storage state)
      ├─ Call GPT: extract participants from content
      │
      └─ For each extracted person:
            ContactsService.UpsertObservedIdentity(
                clientId, source, sourceKey, displayLabel, ct)
            → inserts new (unassigned) ContactIdentity, or returns existing row
            + record the blob's sourceKeys in the Table Storage state row (provenance)
```

Discovery is a new caller of the existing `ContactsService` (which already has `CreateIdentity`/`UpdateIdentity`). It adds one method — `UpsertObservedIdentity` — which inserts an unassigned identity or returns the existing row (and reports whether it created one), so callers never have to handle 409s from race conditions.

---

## 3. Trigger Mechanism

**Scheduled scan (v1).** An in-API `BackgroundService` on a `PeriodicTimer` (every 15 min) — the same in-process scheduled-job pattern as `GrainSynchronizer`, with full EF/scoped access. It is **schedule-only** (no run at startup); the first run is one interval after boot. A run loops active clients and calls `IdentityDiscoveryService.RunDiscovery` per client. The scan only considers blobs modified within a window (default last 3 days) so most history is ignored without any per-blob work. The existing manual endpoint can trigger the same logic on demand with a caller-supplied window.

**Option: EventGrid (later).** If near-real-time discovery is ever needed, EventGrid on the blob containers feeds a Storage Queue consumed by a worker. Handles backpressure during bulk initial loads.

**Source mapping.** `IdentitySource` is derived from the blob's `sourceType` metadata (`Slack`/`Grain`/`Email` → matching enum; anything else → skip), not from the blob path. The client is the blob container.

**Change detection.** A `IdentityDiscoveryState` table in Azure Table Storage (PartitionKey = clientId, RowKey = SHA256(blobPath)) stores the last-processed blob `LastModified` plus the list of `sourceKey`s found in that blob (provenance). If the blob's current `LastModified` is not newer, skip. This mirrors the timestamp-tracking pattern already used by the existing sync workers.

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
Task<(ContactIdentityDto Dto, bool Created)> UpsertObservedIdentity(
    int clientId,
    IdentitySource source,
    string sourceKey,
    string? displayLabel,
    CancellationToken ct);
```

Implementation:
1. Try to load existing `ContactIdentity` by `(clientId, source, sourceKey)`.
2. If found: return it (unchanged), flagged as not created.
3. If not found: insert new `ContactIdentity` with `ContactId = null` (unassigned), flagged as created.

The `Created` flag lets the discovery run count net-new identities.

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

Not a SQL table — same approach as sync workers. Stores, per blob, the last-processed `LastModified` (makes the scan idempotent) and the list of `sourceKey`s found in that blob. That per-blob `sourceKey` list is the lightweight **provenance** record used in place of the deferred `ContactIdentityEvidence` table — it is read-only context, never used to auto-remove identities.

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

## 8. Runner: in-API `BackgroundService`

Discovery runs inside the API process, not as a separate Azure Function — all the identity code (`IdentityDiscoveryService`, `IdentityExtractionService`, `ContactsService`, EF context) already lives there, and the API already hosts scheduled work via `GrainSynchronizer`. Components:

```
IdentityDiscoveryBackgroundService  — PeriodicTimer (15 min), schedule-only, gated by config flag
IdentityDiscoveryRunner             — loops active clients → RunDiscovery per client
IdentityDiscoveryService            — per-client scan + GPT extraction + upsert (already exists)
IdentityDiscoveryStateRepository    — Table Storage LastModified + sourceKey tracking
```

The background service and the existing manual endpoint share the same `RunDiscovery` logic. Source is mapped from blob `sourceType` metadata; there is no blob-path parser.

**Error handling:** GPT parse failures (malformed JSON, refusal) and read errors are logged and the blob's `LastModified` is NOT recorded, so the next run retries. After a few consecutive failures on the same blob, mark it poison in Table Storage and alert (same pattern as existing sync workers).

---

## 9. Cost and Rate Considerations

**Per-file cost.** One GPT-4o call per changed blob. Typical Grain transcript ~3K tokens input + ~500 tokens output ≈ $0.01–0.03 per file at current rates.

**Initial load.** The backfill is a one-time manual endpoint call with an old `modifiedSince` so all existing blobs are processed. 500 blobs × $0.02 ≈ $10 one-time cost.

**Steady-state.** Only blobs modified within the window whose `LastModified` advanced are processed. Low frequency after initial load.

**Large blobs.** Confluence pages can be large. If content exceeds token limit, split at `## ` boundaries and merge. Each split = one additional GPT call. Log split counts to estimate actual costs in production.

---

## 10. Open Questions

- **`ContactIdentityEvidence` (SQL) deferred.** The Table Storage per-blob `sourceKey` list covers lightweight provenance for now; add the SQL evidence table + `?includeEvidence` API only if reviewers need richer context snippets.
- **Timer vs. EventGrid — resolved.** In-API `BackgroundService` on a 15-min timer for v1. Revisit EventGrid only if near-real-time becomes necessary.
- **Display label update policy — resolved.** `UpsertObservedIdentity` does not mutate `DisplayLabel` on an existing row (keep first-seen). Reviewers consolidate variants via assignment/merge.

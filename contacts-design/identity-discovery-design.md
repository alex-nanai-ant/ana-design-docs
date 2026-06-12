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
Timer tick (AltusAiProcessContent Azure Function, every 15 min)
      │
      ▼
For each active client → IdentityDiscoveryRunner.RunDiscovery
      │
      ├─ List blobs; map IdentitySource from `sourceType` metadata
      ├─ Skip blobs outside the modified-since window
      ├─ Skip if blob LastModified unchanged since last run (Table Storage state)
      ├─ Skip GPT if content hash unchanged (metadata-only write)
      ├─ Call GPT: extract participants from content
      │
      └─ For each extracted person:
            IdentityUpsertRepository.UpsertObservedIdentity(
                clientId, source, sourceKey, ct)   // Dapper INSERT, race-safe on the unique index
            → inserts new (unassigned) ContactIdentity, or no-ops on the existing row
            + record the blob's sourceKeys in the Table Storage state row (provenance)
```

Discovery runs in a dedicated Azure Function (`AltusAiProcessContent`), not in the API. Following the sync-worker convention, it references `AltusAiShared` only and writes the `ContactIdentities` table directly via **Dapper** rather than EF. The upsert inserts an unassigned identity or no-ops on the existing row (reporting whether it created one), and is made idempotent at the DB by the `(ClientId, Source, SourceKey)` unique index — so concurrent runs never throw a constraint violation. The API retains ownership of the identity-review surface (read, assign, merge); only the *discovery write* moves out.

---

## 3. Trigger Mechanism

**Scheduled scan (v1).** A `TimerTrigger` function in the `AltusAiProcessContent` Azure Function (every 15 min via cron `0 */15 * * * *`) — the same Azure Functions scheduled-worker pattern as `AltusAiSlackSync`/`AltusAiMailSync`. It is **schedule-only** (`RunOnStartup = false`), so it never fires at boot. A run loops active clients and calls `IdentityDiscoveryRunner.RunDiscovery` per client. The scan only considers blobs modified within a window (default last 3 days) so most history is ignored without any per-blob work. A companion **HTTP-triggered function** in the same project triggers the same logic on demand for a single client with a caller-supplied window (the backfill path), replacing the v0 API endpoint.

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

## 5. Identity upsert (Dapper, in the Function)

The discovery write goes through a small Dapper repository in `AltusAiProcessContent`, not the API's EF `ContactsService`:

```csharp
Task<bool> UpsertObservedIdentity(
    int clientId,
    IdentitySource source,
    string sourceKey,
    CancellationToken ct);   // returns true when a new row was inserted
```

Implementation:
1. Insert a new `ContactIdentity` with `ContactId = null` (unassigned), `Source` stored as its int enum value, and the `BaseEntity` audit columns populated manually (EF's interceptor isn't in the loop).
2. Make the insert atomic against the `(ClientId, Source, SourceKey)` unique index — `INSERT ... WHERE NOT EXISTS` / `MERGE`, or catch SQL 2601/2627 — so a row that already exists is a no-op.
3. Report "created" from rows-affected.

The created flag lets the discovery run count net-new identities. The API's contacts service is **not** modified for discovery; the v0 `ContactsService.UpsertObservedIdentity` is removed along with the API endpoint. The rest of the identity lifecycle (review, assign, merge) stays in the API as designed.

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

## 8. Runner: `AltusAiProcessContent` Azure Function

Discovery runs in a dedicated Azure Function, `AltusAiProcessContent` (its first function; the project is named generically so later content-processing functions can join it). It follows the sync-worker convention — references `AltusAiShared` only, reads config from environment variables, uses Dapper for SQL, and builds `IChatClient` against Azure OpenAI in `Program.cs`. Components:

```
Function.cs
  ├─ [TimerTrigger 0 */15 * * * *, RunOnStartup=false]  — schedule-only; disable via AzureWebJobs.IdentityDiscovery.Disabled
  └─ [HttpTrigger]                                       — on-demand single-client / backfill run
IdentityDiscoveryRunner             — loops active clients → RunDiscovery per client (incremental scan + extraction)
IdentityExtractionService           — GPT extraction (moved here from the API; depends only on IChatClient)
IdentityUpsertRepository            — Dapper INSERT into ContactIdentities, race-safe on the unique index
IdentityDiscoveryStateRepository    — Table Storage LastModified + content hash + sourceKey tracking
```

Both triggers share the same `IdentityDiscoveryRunner` logic. `IdentitySource` and the `ExtractedIdentity` record move to `AltusAiShared` so the API entity and the Function's extraction service share one definition. Source is mapped from blob `sourceType` metadata; there is no blob-path parser.

**Error handling:** GPT parse failures (malformed JSON, refusal) and read errors are logged and the blob's `LastModified` is NOT recorded, so the next run retries. After a few consecutive failures on the same blob, mark it poison in Table Storage and alert (same pattern as existing sync workers).

---

## 9. Cost and Rate Considerations

**Per-file cost.** One GPT-4o call per changed blob. Typical Grain transcript ~3K tokens input + ~500 tokens output ≈ $0.01–0.03 per file at current rates.

**Initial load.** The backfill is a one-time call to the HTTP-triggered function with an old `modifiedSince` so all existing blobs are processed. 500 blobs × $0.02 ≈ $10 one-time cost.

**Steady-state.** Only blobs modified within the window whose `LastModified` advanced are processed. Low frequency after initial load.

**Large blobs.** Confluence pages can be large. If content exceeds token limit, split at `## ` boundaries and merge. Each split = one additional GPT call. Log split counts to estimate actual costs in production.

---

## 10. Open Questions

- **`ContactIdentityEvidence` (SQL) deferred.** The Table Storage per-blob `sourceKey` list covers lightweight provenance for now; add the SQL evidence table + `?includeEvidence` API only if reviewers need richer context snippets.
- **Timer vs. EventGrid — resolved.** A `TimerTrigger` in the `AltusAiProcessContent` Function on a 15-min cron for v1. Revisit EventGrid only if near-real-time becomes necessary.
- **Display label update policy — resolved.** `UpsertObservedIdentity` does not mutate `DisplayLabel` on an existing row (keep first-seen). Reviewers consolidate variants via assignment/merge.

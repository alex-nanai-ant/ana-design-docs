# BlobDocument Content Exclusion — Technical Design

_Date: 2026-05-01_
_Companion to: [content-exclusion-query-time-design.md](content-exclusion-query-time-design.md)_

## Goal

Add a second `TargetKind` to the existing exclusion-rule machinery: **`BlobDocument`**, which excludes a single indexed blob (any source — Slack chunk, Email, Curated Answer, Manual upload, Grain, Confluence page, …) by matching its Cognitive Search `parent_id`.

Slots into the tagged-union schema from §10 of the companion doc — no migration, no index schema change, no reindex.

## 1. Matching rule

```
doc.parent_id == rule.targetValue
```

`parent_id` is already filterable on every chunk ([IndexingService.cs:68](../AltusAiAssistantApi/Services/IndexingService.cs#L68)) — set by the IndexProjectionSelector at [IndexingService.cs:228-258](../AltusAiAssistantApi/Services/IndexingService.cs#L228-L258).

## 2. Rule schema

Reuse `ContentExclusionRules`. Add one enum value:

```csharp
public enum ContentExclusionTargetKind
{
    ConfluenceSubtree,
    BlobDocument      // TargetValue = Cognitive Search parent_id
}
```

`TargetValue` holds the parent_id string (URL-safe base64, ≤ ~512 chars in practice). No DB migration, no index schema change.

`ValidateTargetValue` for `BlobDocument`: non-empty, ≤ 512 chars, matches `^[A-Za-z0-9_-]+$` (URL-safe base64 alphabet, no padding). The regex is the **only** thing standing between user input and an unescaped OData filter clause — it must be tight.

## 3. parent_id computation (Option A — local)

`parent_id` is the parent indexer document's key. The blob data source has no explicit `documentKey` mapping ([IndexingService.cs:138-151](../AltusAiAssistantApi/Services/IndexingService.cs#L138-L151), [:281-288](../AltusAiAssistantApi/Services/IndexingService.cs#L281-L288)), so Azure Search's default applies: `metadata_storage_path` passed through `base64Encode` (URL-safe base64, no padding).

Single helper, lives in `AltusAiShared` next to `BlobMetadata`:

```csharp
public static string ComputeParentId(string blobUrl)
{
    var bytes = Encoding.UTF8.GetBytes(blobUrl);
    return Convert.ToBase64String(bytes)
        .Replace('+', '-')
        .Replace('/', '_')
        .TrimEnd('=');
}
```

`blobUrl` is the same string already constructed at [AdminService.cs:213](../AltusAiAssistantApi/Services/AdminService.cs#L213) — `{baseUrl}/{containerId}/{path}`.

### 3.1. Encoding-correctness verification

Local computation is brittle: any divergence between the input `blobUrl` and the actual `metadata_storage_path` Azure Search saw will silently produce a wrong parent_id, and the exclusion rule will no-op. Pin this down with an xUnit test in `AltusAiShared.Tests`:

- Pull a representative `(parent_id, url)` sample from a real client index covering: ASCII path, path with spaces, path with parens (Email folder format), path with non-ASCII (Slack channel name), long path, deeply nested path.
- Assert `ComputeParentId(url) == parent_id` for each.
- Lock the sample as a fixture so future Azure Search behavior changes break the build, not production rules.

If the test ever fails on a new sample, fall back to Option B (index-as-source-of-truth) — see companion doc.

## 4. BlobItemDto

Add one field, populated for every non-folder node:

```csharp
public string? ParentId { get; set; }
```

Set at [AdminService.cs:228-248](../AltusAiAssistantApi/Services/AdminService.cs#L228-L248):

```csharp
ParentId = isFolder ? null : ParentIdHelper.ComputeParentId(url),
```

Folders never get a ParentId — exclusion is single-blob only. Folder-level exclusion is a separate kind (out of scope here).

## 5. Filter construction

Extend `BuildSearchFilter` at [ContentExclusionService.cs:196-217](../AltusAiAssistantApi/Services/ContentExclusionService.cs#L196-L217). One additional clause, AND-joined with the existing Confluence clause:

```csharp
var blobDocumentIds = rules
    .Where(r => r.TargetKind == ContentExclusionTargetKind.BlobDocument)
    .Select(r => r.TargetValue)
    .Distinct()
    .ToList();

if (blobDocumentIds.Count > 0)
    clauses.Add($"not search.in(parent_id, '{string.Join(",", blobDocumentIds)}', ',')");
```

`search.in` with `,` separator is safe because §2's regex prohibits commas in `TargetValue`.

Empty-set short-circuit: if no `BlobDocument` rules, omit the clause entirely.

## 6. Admin UI flow

1. Admin browses the file tree (`GET /Admin/GetBlobItems/{clientId}`). Each leaf node now carries `parentId`.
2. "Exclude" action on a node POSTs `{ targetKind: "BlobDocument", targetValue: node.parentId, displayTitle: node.path }`.
3. Caps (§7 of companion doc) and authorization (§4) are unchanged — the existing `CreateRule` flow handles `BlobDocument` identically to `ConfluenceSubtree`.

## 7. Caps

`BlobDocument` rules need their own cap, separate from `ConfluenceSubtree`. A `parent_id` is base64 of the full blob URL (`https://{account}.blob.core.windows.net/{guid}/{path}`), typically **150–350 chars** post-encoding — Email body paths with date+sender+subject folders push the upper end. At ~250 chars + 1 separator per ID, the 500-rule cap reused from `ConfluenceSubtree` would produce ~125 KB of filter — about 4× over Azure Search's 32 KB POST ceiling.

| Scope | Soft cap | Hard cap |
|---|---|---|
| Per-user BlobDocument (active) | 15 | 30 |
| Global BlobDocument (active) | 40 | 70 |

Independent caps that sum to 100 by construction — no separate combined-per-query check is needed because the worst case is `70 + 30 = 100` BlobDocument rules in a single filter (~25 KB), leaving headroom under the POST ceiling for the existing `ConfluenceSubtree` clause.

Rationale for the 70/30 split: global rules apply to every user in the client and are the natural growth area (compliance exclusions, departed-employee Slack content, draft pages, etc.) — they get the larger share. Per-user rules cover personal "I don't want to see this" preferences; if a user is past 30 they're likely misusing exclusion as a curation system, and the soft cap at 15 nudges them before that.

- **Soft cap** → UI warning. Request still succeeds.
- **Hard cap** → API returns `400 Bad Request`.

## 8. Out of scope

- Folder/prefix exclusion (would need a filterable path field — separate doc).
- Per-source kinds (`SlackChannel`, `Mailbox`) — slot in the same way using the existing typed metadata fields.
- Bulk "exclude every result currently shown" UI — additive on top of this.

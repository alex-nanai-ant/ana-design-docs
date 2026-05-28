# Confluence Page Hierarchy — Technical Design

_Date: 2026-04-20_
_Companion to: [confluence-indexing-and-hierarchy.md](confluence-indexing-and-hierarchy.md)_

## Goal

Preserve Confluence page hierarchy through the sync pipeline by encoding it in
a single **structured blob metadata** key, and render it as a nested tree in
[`AdminController.GetBlobItems`](../AltusAiAssistantApi/Controllers/AdminController.cs#L98).

Blob paths stay flat (`{SpaceKey}/{PageTitle}.md`) — hierarchy lives in metadata
only. This keeps renames/moves cheap (metadata update, no blob relocation) and
avoids orphaning blobs when Confluence titles change.

Folders (Confluence `type: "folder"` containers) are **first-class members of
the ancestor chain** — not a display-only carve-out. They carry IDs in metadata,
get synthetic nodes in the admin tree, and participate in any downstream
feature (notably subtree exclusion) on the same footing as pages. Pages and
folders share one matching rule: "is this target ID anywhere in my ancestor
chain?"

## 1. Fetch ancestors from Confluence

[`ConfluenceService.GetContent`](../AltusAiShared/Confluence/ConfluenceService.cs#L64)
currently requests `expand=body.storage`. Add `ancestors`:

```
expand=body.storage,ancestors
```

Extend [`ConfluenceGetContentResponse`](../AltusAiShared/Confluence/ConfluenceGetContentResponse.cs)
with an `Ancestors` array (ordered root → immediate parent):

```csharp
public class ConfluenceAncestor
{
    [JsonProperty("id")]    public string Id { get; set; }
    [JsonProperty("title")] public string Title { get; set; }
    [JsonProperty("type")]  public string Type { get; set; } // "page" or "folder"
}
```

Root pages return an empty array. The array is kept **as-is** — no filtering
by type. Pages and folders both flow through to blob metadata, sync state, and
the admin tree.

The CQL filter (`type=page` at
[ConfluenceSyncService.cs:46](../AltusAiShared/Confluence/ConfluenceSyncService.cs#L46))
still governs which *pages* are synced. Folders have no blob of their own —
they exist only as entries in the ancestor chains of pages that live under
them.

## 2. Write hierarchy into blob metadata

Extend [`BlobMetadata`](../AltusAiShared/BlobMetadata.cs) with two keys:

| Key | Value | Purpose |
|---|---|---|
| `pageId` | This page's own Confluence ID | Direct identity — tree node key, exclusion target match |
| `ancestors` | JSON array of `{id, title, type}` objects, root → immediate parent | Tree rendering, move detection, subtree matching |

Example `ancestors` value (stored as a JSON string):

```json
[
  {"id":"123",  "title":"Engineering",   "type":"page"},
  {"id":"f456", "title":"Drafts",        "type":"folder"},
  {"id":"789",  "title":"Design Review", "type":"page"}
]
```

Everything the previous three-key design (`parentPageId`, `ancestorIds`,
`breadcrumb`) delivered, downstream code derives from `ancestors` in one pass:

| Derived value | Expression |
|---|---|
| Immediate parent ID (page or folder) | last element's `id` |
| All ancestor IDs (pages + folders) | `ancestors.Select(a => a.id)` |
| Breadcrumb for display | `string.Join("/", ancestors.Select(a => a.title))` |

Extend `BlobMetadata.Create(...)` with optional `pageId` and `ancestors`
parameters. Call sites that don't supply hierarchy (Slack, Email, Manual,
Grain, CuratedAnswer) stay unchanged.

Update the upload call at
[ConfluenceSyncService.cs:177-182](../AltusAiShared/Confluence/ConfluenceSyncService.cs#L177-L182)
to serialize the ancestor array and pass it through alongside `pageId`.

Pages with no body (skipped by the `< 50 char` rule at
[ConfluenceSyncService.cs:169](../AltusAiShared/Confluence/ConfluenceSyncService.cs#L169))
produce no blob and therefore contribute no `ancestors` metadata of their own,
but they still appear in *other* pages' ancestor chains — so the admin tree can
still surface them as synthetic nodes (see §4).

## 3. Persist hierarchy in sync state

Add one column to
[`SyncPageTableEntity`](../AltusAiShared/Confluence/SyncPageTableEntity.cs):
`Ancestors` (JSON string, same shape as the blob metadata value).

Incremental sync compares the stored `Ancestors` JSON to the current one.
A difference signals a **move** (page re-parented under a different page or
folder) and triggers a metadata refresh on the existing blob without
re-downloading content. Structural comparison correctly catches folder
re-parents too — something a page-only ancestor list would silently miss when
only an intermediate folder's parent changes.

Table Storage requires no schema migration — the new property appears
automatically.

## 4. Build the hierarchical tree in `GetBlobItems`

Today [`AdminService.GetBlobItems`](../AltusAiAssistantApi/Services/AdminService.cs#L125)
builds the tree from the **blob path** (`parts = item.Split('/')`). For
Confluence blobs, replace that with a metadata-driven pass that treats pages
and folders uniformly:

1. Partition `allItems` into Confluence (`SourceType == Confluence` or
   `ExternalConfluence`) and non-Confluence.
2. Run the existing path-splitting loop for non-Confluence blobs (Slack, Email,
   Manual, etc.).
3. For Confluence blobs within a space folder:
   - Deserialize `ancestors` and read `pageId` for each blob.
   - Build a dictionary keyed by Confluence ID, aggregating every node
     encountered — both the pages with their own blobs and the folder (and
     body-less page) entries seen in ancestor chains. Each dictionary entry
     records `id`, `title`, `type`, and (for pages with a blob) the backing
     blob path.
   - Emit one `BlobItemDto` per entry. Page entries with a blob set `Path` to
     the existing flat blob path so
     [`AdminService.DeleteBlobItems`](../AltusAiAssistantApi/Services/AdminService.cs)
     keeps working unchanged. Folder entries and body-less page entries have
     no `Path`, no size, and are not deletable — but they are real,
     clickable tree nodes.
   - Link each node to its immediate parent using the last entry of that
     node's ancestor chain. Roots (empty chain) attach to the space-key
     folder node.

Add to [`BlobItemDto`](../AltusAiAssistantApi/Models/BlobItemDto.cs):
`ConfluenceId`, `ConfluenceNodeType` (`"page"` or `"folder"`),
`ParentConfluenceId`, and `Breadcrumb`. The FE uses `ConfluenceNodeType` to
pick an icon and to block destructive actions on folder nodes.

**No collapse-up behavior.** Every folder in the raw ancestor chain becomes a
visible node. A page nested under a folder sits directly under that folder in
the tree — same shape the user sees in Confluence.

## 5. Rollout

1. Ship the sync changes (steps 1-3). New and updated pages get `pageId` and
   `ancestors` metadata on the next incremental run; untouched pages retain
   flat metadata.
2. Trigger a **Full** sync per Confluence space to backfill metadata on all
   existing blobs. No reindex of Cognitive Search is required for the
   `GetBlobItems` feature — the admin file tree reads blob metadata directly,
   not the search index.
3. Update `GetBlobItems` (step 4). Confluence blobs that still lack metadata
   (e.g. if full sync hasn't run yet) fall back to the existing flat rendering
   — detect by absence of `ancestors` and treat as root attached to the
   space-key folder.

## Out of scope

- Promoting `ancestors` (or any projection of it, such as the flat ID list) to
  a Cognitive Search index field. Needed only if a downstream feature wants
  hierarchy-aware retrieval or query-time subtree filtering — see the
  exclusion design for one example. The admin UI works entirely from blob
  metadata.
- Moving the blob path to include the hierarchy. Explicitly rejected — see
  tradeoff in the investigation doc.
- Synthetic folder blobs (uploading empty placeholder blobs so folders have
  "real" storage presence). Rejected — pollutes the search index with empty
  content; folders don't need blobs to be first-class tree nodes.
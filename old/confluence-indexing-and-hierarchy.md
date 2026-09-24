# Confluence Indexing & Page Hierarchy — Investigation

_Date: 2026-04-16_

## 1. Where Confluence pages are indexed

Indexing happens in two stages: a background Function writes Markdown blobs, then an Azure Cognitive Search indexer vectorizes them on a schedule.

### Stage 1 — Fetch & upload to Blob Storage

Handled by the `AltusAiConfluenceSync` Azure Function.

- **Timer triggers** in [AltusAiConfluenceSync/Function.cs](../AltusAiConfluenceSync/Function.cs):
  - `IncrementalSync` — every 15 minutes ([Function.cs:56](../AltusAiConfluenceSync/Function.cs#L56))
  - `DeletedPagesSync` — every 60 minutes ([Function.cs:110](../AltusAiConfluenceSync/Function.cs#L110))
  - `IncrementalSyncHttp` — manual HTTP trigger with date param ([Function.cs:35](../AltusAiConfluenceSync/Function.cs#L35))
- **Orchestration**: [`ConfluenceSyncService.RunSyncJob()`](../AltusAiShared/Confluence/ConfluenceSyncService.cs#L37)
  - Fetches pages via CQL search API ([ConfluenceSyncService.cs:92](../AltusAiShared/Confluence/ConfluenceSyncService.cs#L92))
  - Tracks sync state in Azure Table Storage (`SyncPageTableEntity`, `SyncRunTableEntity`)
  - Supports incremental and full sync modes
- **Content transformation** ([ConfluenceSyncService.cs:152-182](../AltusAiShared/Confluence/ConfluenceSyncService.cs#L152-L182)):
  - HTML tables → text via [`HtmlContentHelper.TransformHtmlTablesToText()`](../AltusAiShared/HtmlContentHelper.cs#L30)
  - HTML → Markdown via `ReverseMarkdown.Converter` ([ConfluenceSyncService.cs:167](../AltusAiShared/Confluence/ConfluenceSyncService.cs#L167))
  - Skipped if content < 50 chars
- **Blob upload** ([ConfluenceSyncService.cs:177-182](../AltusAiShared/Confluence/ConfluenceSyncService.cs#L177-L182)):
  - Container: `{clientUniqueId}` (safe name via `CreateSafeContainerName`)
  - Path: `{SpaceKey}/{DocumentTitle}.md`
  - Metadata: `DocumentSourceType`, `LastModified`, `ConfluencePageUrl`

### Stage 2 — Blob → Azure Cognitive Search

Provisioned via `POST /indexing/createindex/{clientId}` at [IndexingController.cs:22](../AltusAiAssistantApi/Controllers/IndexingController.cs#L22), built by [`IndexingService`](../AltusAiAssistantApi/Services/IndexingService.cs) ([IndexingService.cs:31-54](../AltusAiAssistantApi/Services/IndexingService.cs#L31-L54)):

- **Data source**: the blob container
- **Skillset**:
  - `SplitSkill` — chunks content into 800-token pages ([IndexingService.cs:147-170](../AltusAiAssistantApi/Services/IndexingService.cs#L147-L170))
  - `WebApiSkill` → `AltusAiTextMergeSkill` — prepends filename to chunks ([IndexingService.cs:173-204](../AltusAiAssistantApi/Services/IndexingService.cs#L173-L204))
  - `AzureOpenAIEmbeddingSkill` — 1536-dim vectors ([IndexingService.cs:309-340](../AltusAiAssistantApi/Services/IndexingService.cs#L309-L340))
- **Indexer**: runs on a 30-minute schedule ([IndexingService.cs:262](../AltusAiAssistantApi/Services/IndexingService.cs#L262))
- **Index fields**: `chunk_id`, `content`, `contentVector`, `title`, `externalUrl`, `date`, `sourceType`

## 2. Page hierarchy tracking

**Not tracked today.** The pipeline is flat end-to-end.

| Layer | Hierarchy support | Location |
|---|---|---|
| Confluence API request | No — only `expand=body.storage` | [ConfluenceService.cs:64](../AltusAiShared/Confluence/ConfluenceService.cs#L64) |
| DTOs | No `ancestors` / `parentId` / `parent` | [ConfluenceSearchResponse.cs:18-37](../AltusAiShared/Confluence/ConfluenceSearchResponse.cs#L18-L37), `ConfluenceGetContentResponse.cs` |
| Blob path | Flat `{SpaceKey}/{PageTitle}.md` | [ConfluenceSyncService.cs:177-182](../AltusAiShared/Confluence/ConfluenceSyncService.cs#L177-L182) |
| Blob metadata | Only source type, modified, URL | same |
| Sync state table | `SyncPageTableEntity` has no parent fields | `AltusAiShared/Confluence/SyncPageTableEntity.cs` |
| Search index | `parent_id` exists but is the chunk→document link only | [IndexingService.cs:220-243](../AltusAiAssistantApi/Services/IndexingService.cs#L220-L243) |

> Note: the `parent_id` field in the Cognitive Search index is **not** a Confluence parent — it comes from `SplitSkill` and links a chunk back to its source document.

## 3. What it would take to add hierarchy

1. **API**: request `expand=ancestors` (and optionally `space`) from the Confluence content API; add `AncestorIds` / `ParentId` to [ConfluenceSearchResponse.cs](../AltusAiShared/Confluence/ConfluenceSearchResponse.cs) and `ConfluenceGetContentResponse.cs`.
2. **Sync state**: add `ParentPageId` + `AncestorPath` columns to `SyncPageTableEntity` so incremental sync can reconcile moves/renames.
3. **Blob layer**: encode ancestors as **blob metadata** (e.g. `ParentPageId`, `Breadcrumb`) rather than into the path. Metadata is robust under rename/move; path-encoded hierarchy orphans blobs when Confluence titles change.
4. **Search index**: add `parentPageId` (filterable) and `breadcrumb` (searchable) fields in [IndexingService.cs](../AltusAiAssistantApi/Services/IndexingService.cs), mapped from blob metadata via the indexer's `fieldMappings`. No skillset changes needed. Requires a reindex.

### Tradeoff

- **Path-encoded hierarchy** — intuitive, visible in Blob Storage, but fragile under renames/moves (produces orphan blobs and re-uploads).
- **Metadata-encoded hierarchy** — robust under moves, queryable via Cognitive Search filters, but requires an index schema change and a one-time reindex.

Recommended: **metadata-encoded**, with the blob path left flat.
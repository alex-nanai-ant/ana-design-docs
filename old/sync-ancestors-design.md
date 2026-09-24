# SyncAncestors — Technical Design

_Date: 2026-04-23_
_Companion to: [confluence-hierarchy-design.md](confluence-hierarchy-design.md)_

## Problem

Confluence's `lastmodified` timestamp does **not** change when a page is moved
or its parent is renamed. The incremental sync filters pages server-side via
`cql ... AND lastmodified >= X`, so moved/re-parented pages never enter the
result set and their ancestors drift out of sync in local Table Storage and
blob metadata.

## Solution

A new Azure Function `SyncAncestors` that runs on a timer, independently of
`IncrementalSync` and `DeletedPagesSync`. Per client Confluence link:

1. Fetch all pages in the space with ancestors expanded in a single paginated
   call: `/rest/api/content?spaceKey={key}&expand=ancestors&limit=100`.
   New method on `ConfluenceSyncService`, e.g. `GetAllPagesWithAncestors`.
2. Iterate local `SyncPageTableEntity` rows for that space/client.
3. For each page still present in Confluence, compare
   `SerializeAncestors(remote.Ancestors)` against `localPage.Ancestors`. If
   equal, skip.
4. On drift:
   - `BlobClient.SetMetadata` on the existing markdown blob with new
     `confluenceAncestorIds` / `confluenceAncestorNames` — **no re-upload**,
     no HTML→MD conversion, no grain-link reprocessing.
   - Upsert the table entity with the new `Ancestors` JSON. Do **not** bump
     `LastSyncTime` — that field means "when the page's content was last
     synced," and ancestor reconciliation does not sync content. The table
     row's auto-populated `Timestamp` already records when ancestors changed.

## Why a separate Function

- Keeps `IncrementalSync` and `DeletedPagesSync` single-purpose.
- Runs on its own cadence (moves are rare — daily is likely enough).
- Independent failure domain: if ancestor reconciliation breaks, content sync
  and deletion detection keep working.

## No `SyncRunTableEntity` write

Like `DeletedPagesSync`, this Function does **not** create a `SyncRunTableEntity`
row. That table records content-sync runs (full/incremental); ancillary
reconciliation jobs stay out of it so the sync-run history keeps one meaning.
Ancestor drift events are observable via logs and the per-page row `Timestamp`.

## Out of scope

- Page-content drift (handled by `IncrementalSync` via `lastmodified`).
- Page renames of the page itself (Confluence bumps `lastmodified` on rename —
  already handled).
- Deletions (handled by `DeletedPagesSync`).

## Downstream effect

`SetMetadata` bumps the blob's `Last-Modified`, so the Azure Cognitive Search
blob indexer re-picks up the blob on its next run and refreshes any fields
mapped from `confluenceAncestorIds` / `confluenceAncestorNames`.
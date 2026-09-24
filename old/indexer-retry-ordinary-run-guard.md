# Indexer retry: require an ordinary run between retry batches

_Date: 2026-09-20_  
_Status: implemented 2026-09-21 on `feature/ANA-178-retry-docs-index`; QA verification outstanding_

## Problem

`AltusAiIndexerRetry` must give ordinary indexing a turn between Reset Docs retry batches. An ordinary run only needs to start; it does not need to finish or succeed before another retry batch may be selected.

The current guard is:

```csharp
if (lastResult?.StatusDetail == IndexerExecutionStatusDetail.ResetDocs)
{
    return "last indexer run processed a reset selection; waiting for a normal scheduled run before executing more retries";
}
```

This does not enforce the requirement. `StatusDetail == ResetDocs` describes the Reset Docs request entry, rather than the later indexer execution that processes the selected documents.

## Observed execution history

The QA indexer history exposes `mode` on each execution. The observed sequence distinguishes the operations clearly:

| History entry | `status` | `statusDetail` | `mode` |
|---|---|---|---|
| Reset Docs request | `reset` | `resetDocs` | `indexingResetDocs` |
| Execution that processes the reset selection | `success` | null | `indexingResetDocs` |
| Ordinary indexer execution | `success` | null | `indexingAllDocs` |

The reset-processing executions can contain failed items even though their top-level status is `success`, because the indexer tolerates document failures. Success, failure count, duration, and tracking-state changes therefore cannot reliably identify the execution type.

The latest captured status provided the decisive example:

```text
lastResult.status       = success
lastResult.statusDetail = null
lastResult.mode         = indexingResetDocs
currentState.mode       = indexingAllDocs
```

At that point, reset processing had finished and Azure was ready for ordinary indexing, but an ordinary execution had not started. The current `StatusDetail` guard would allow another retry batch immediately.

`currentState.mode` is also insufficient for this policy. It describes what the indexer is currently prepared to do and returns to `indexingAllDocs` after reset processing. It does not prove that an ordinary execution has started.

## Proposed rule

After handling an outstanding reset selection, allow a new retry batch only when the most recent or in-progress execution has mode `indexingAllDocs`.

With a typed SDK model, the replacement guard is:

```csharp
public static bool IsWaitingForOrdinaryRun(SearchIndexerStatus status) =>
    status.LastResult?.Mode != IndexingMode.AllDocuments;
```

`IndexingMode` is an extensible enum (a `readonly struct`, not a `System.Enum`), so this
comparison relies on the lifted `!=` against `IndexingMode?`. A null mode blocks.

This deliberately permits all of the following ordinary runs to satisfy the requirement:

- an in-progress ordinary run;
- a completed ordinary run with no changed documents;
- an ordinary run containing document errors;
- an ordinary run with a top-level failure.

The policy is about giving normal high-water-mark processing a turn, not requiring it to finish successfully. Missing or unknown mode blocks retry dispatch conservatively.

The existing outstanding-selection guard remains before this guard. If Reset Docs metadata still contains selected keys or reset mode, the worker requests a run and does not submit another selection.

## SDK upgrade

The project currently references `Azure.Search.Documents` `11.7.0-beta.1`. That version's typed `IndexerExecutionResult` does not expose the execution `Mode`, even though the service returns it in JSON.

Upgrade only `AltusAiIndexerRetry` to:

```xml
<PackageReference Include="Azure.Search.Documents" Version="11.8.0-beta.1" />
```

This is the narrowest suitable upgrade:

- it exposes `IndexerExecutionResult.Mode` through the typed model;
- it supports the `2025-11-01-preview` service version used to inspect and validate Reset Docs;
- it retains typed Reset Docs operations;
- it avoids the larger 12.x migration.

No GA release is a candidate. Both `11.7.0` and `12.0.0` strip the entire preview surface this
worker is built on — `Mode`, `IndexingMode`, `IndexerExecutionStatusDetail`, `IndexerState` /
`CurrentState`, `ResetDocumentsAsync` and `ResetDocumentOptions` are all absent. Moving to a GA
package would not merely block this guard; it would fail to compile the existing worker. The only
other version carrying the surface is `12.1.0-beta.2`, which is the eventual destination but moves
`ResetDocumentOptions` to `Indexes.Models` and reorders `ResetDocumentsAsync`.

The main API is a separate application and does not need its package reference changed for this
retry-worker behavior. Nothing shares the SDK between them: `AltusAiShared`, which both reference,
has no `Azure.Search.Documents` dependency, there is no central package management, and the two
projects resolve the package independently. Only the retry worker's pin is load-bearing — the API
uses no preview-only surface.

Implementation required one mechanical adjustment in addition to the guard: reading outstanding
selection state from `SearchIndexerStatus.CurrentState`, since 11.8.0-beta.1 drops `CurrentState`
from `IndexerExecutionResult`. That was the only compile break across the whole solution. The
typed `ResetDocumentsAsync` call needed no change; its signature is identical in 11.7.0-beta.1 and
11.8.0-beta.1.

## Expected flow

```text
ordinary execution starts
  -> retry worker may submit one eligible Reset Docs batch
  -> reset selection is outstanding
  -> a run processes the reset selection (mode: indexingResetDocs)
  -> retry worker waits
  -> ordinary execution starts (mode: indexingAllDocs)
  -> retry worker may submit the next eligible retry batch
```

Submitting a reset selection after an ordinary run has started is acceptable. The in-progress ordinary execution continues with the state captured at its start; the new selection is consumed by a subsequent run. The worker's accompanying Run request may receive `409`, which the existing code already treats as expected.

## Verification

Done:

1. ✅ Restore and build `AltusAiIndexerRetry` with `Azure.Search.Documents` 11.8.0-beta.1; the full
   solution and all 256 tests pass.
2. ✅ A completed reset-processing execution with `Mode == ResetDocuments` blocks another batch, and
   an ordinary run permits one — covered by `IndexerRetryGuardTests`, including an in-progress
   ordinary run, a failed ordinary run, a null mode and `IndexingResync`.
3. ✅ An outstanding reset selection still takes precedence and only triggers the existing Run
   request; that guard reads `status.CurrentState` and is covered by the same tests.

Still required against QA, since no test can prove what the service actually sends:

4. Confirm the status response populates both `status.LastResult.Mode` and `status.CurrentState`.
   This is the one assumption the unit tests cannot check — they build models through
   `SearchModelFactory` rather than observing the wire.
5. Confirm an ordinary run is observed as `Mode == AllDocuments` end to end, releasing the next batch.

The raw-JSON/custom-DTO approach remains a fallback if the upgraded typed client does not populate
`Mode` against the live service.

Note that the guard blocks on `IndexingMode.IndexingResync` as well, by the same
conservative rule. If resync is ever adopted for these indexers, retries would stall until an
ordinary run followed, and this policy would need revisiting.

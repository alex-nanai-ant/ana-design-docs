# Indexer Retry Worker — Retry Attempt Tracking

_Created: 2026-09-08._
_Status: approved design; implementation not started. Supersedes Part 3 (Retry worker) of [indexer-skip-metadata-and-failure-thresholds.md](indexer-skip-metadata-and-failure-thresholds.md); Parts 1, 2 and 4 of that document still stand._

## Decision

`AltusAiIndexerRetry` recovers source documents that client indexers tolerated as failures (`maxFailedItems: -1`). It observes completed indexer executions, records each document error as a retry attempt in Azure Table Storage, and resubmits eligible documents through the Azure AI Search **Reset Docs** preview API.

The worker **schedules retries; it does not adjudicate outcomes.** An observed failure is a trigger, and each trigger produces at most one reset submission. A document that succeeds generates no further trigger and is simply never resubmitted. Success is therefore never detected, inferred, or stored.

This is a deliberate reduction from the earlier design, which specified document states (`Pending`, `Submitted`, `AwaitingVerification`, `Recovered`, `Exhausted`, `NeedsAttention`, `Superseded`, `SourceGone`), an immutable reset batch, an indexer lease, a `NormalRunRequired` phase, batch deadlines and a watchdog. None of that is retained.

## Why the model changed

The earlier design's machinery exists to answer one question: *did a retried document succeed?*

Azure AI Search does not answer it. Reset Docs returns `204` acknowledging selection only; no API reports the outcome of a selection. Microsoft's own guidance is to query the index afterwards and inspect the documents. Every attempt to infer the answer from indexer execution history produced defects. Two were found in the implementation written against the earlier design, both of which mark a still-failing document as `Recovered`:

1. **Success inferred from a run that started before submission.** The code treated "absent from the errors of any execution that *ended* after dispatch" as proof of recovery. An execution already in flight when Reset Docs was called has fixed its reset state at start, so it ends after dispatch having retried nothing.
2. **Evidence discarded by the checkpoint.** Reconciliation returned early when a newer execution was in progress, but the observation checkpoint advanced regardless, retiring the completed execution that held the batch's verdict. The next pass then judged the batch against an unrelated later run.

Both are inference bugs. Neither is reachable in the model below, because nothing infers anything.

The consequence of getting this wrong is severe and silent: an indexer does not revisit an unchanged failed blob on subsequent normal runs, so a falsely-recovered document is dropped permanently and reported as fixed.

## Reset Docs contract

Confirmed against [Run or reset indexers — How to reset docs (preview)](https://learn.microsoft.com/en-us/azure/search/search-howto-run-reset-indexers#how-to-reset-docs-preview) and the [Reset Docs REST reference](https://learn.microsoft.com/en-us/rest/api/searchservice/indexers/reset-docs?view=rest-searchservice-2025-11-01-preview).

**Selection, not execution.** Reset Docs writes to indexer metadata and returns `204`:

> "Invoking the API adds the document keys or data source document identifiers to reset to the indexer metadata. **On the next scheduled or on-demand run of the indexer, the indexer processes only the reset documents.**"

**A reset run performs no ordinary ingestion:**

> "If you specify the reset parameters, they solely determine what gets processed, regardless of other changes in the underlying data. For example, if 20 blobs were added or updated since the last indexer run, but you only reset one document, the indexer processes only that document."

This is why the worker issues an explicit Run rather than waiting for the schedule — see [Run is explicit](#run-is-explicit) below.

**Invalid identifiers are silently ignored.** "Invalid document keys or document keys that don't exist are ignored." A wrong reset identifier format can therefore make a retry process no documents; the QA tests reported a ResetDocs warning for unmatched identifiers. This is the single most important thing to validate before enabling dispatch.

**A pending selection can be cleared.** Sending an empty `documentKeys` list "results in the indexer resuming regular indexing based on the high water mark" — the operational abort switch, requiring no code.

**`overwrite` is a query parameter** in `2025-11-01-preview` (`true` replaces the pending selection, `false` appends). The how-to article shows it as a body field in `2026-08-01-preview`. The .NET SDK's `ResetDocumentsAsync(name, overwrite, options, ct)` abstracts the difference.

**Cost.** The indexer holds `indexingResetDocs` mode "until it processes all the document keys provided", during which "no other indexer jobs execute while the operation is progressing", and locating documents may require cracking each one in the container. The worker submits `documentKeys`, so this cost must be measured during rollout.

**Identifiers (validated in QA Sandbox, 2026-09-19).** Submit `documentKeys` computed with `AltusAiShared.ParentIdHelper.ComputeParentId(BlobUrl)`, the same helper used by content exclusion rules. It canonicalizes the URL, encodes its UTF-8 bytes as Base64, replaces `+` and `/` with `-` and `_`, strips `=` padding and appends the padding count. These are source parent IDs, not projected `chunk_id` values. They can be derived even when a file failed before any searchable chunk existed. Manual tests with the two original documents found both sources: the marker file failed enrichment as intended and the other produced projection warnings. Raw URLs, relative paths and encoded `localId` values through `datasourceDocumentIds`, and a projected chunk key through `documentKeys`, did not resolve in this test.

## Model

Two Table Storage tables, both owned solely by this worker. State is authoritative; no retry metadata is written to blobs.

**Document row** — partition key is the client unique id, row key is a SHA-256 hash of the blob URL (blob paths contain characters Table Storage keys reject):

| Field | Purpose |
|---|---|
| `BlobUrl` | Canonical identity; converted to a parent document key with `ParentIdHelper.ComputeParentId` for Reset Docs |
| `BlobPath` | Path within the client container |
| `SourceETag` | The blob revision this budget belongs to. Compared for equality only |
| `SourceLastModified` | Diagnostic |
| `Attempts` | Reset submissions charged for this revision |
| `LastFailureAt` | End time of the execution that most recently reported this document as failed |
| `LastError` | Truncated diagnostic from that error |
| `LastDispatchedAt` | When this document was last submitted to Reset Docs |
| `SkipDispatchReason` | Why the last examination declined to submit it — currently only `SourceGone`. Null otherwise |

There is **no lifecycle state.** `Exhausted` is a query (`Attempts >= MaxAttempts`), not a stored value. `SkipDispatchReason` is a diagnostic rather than a state: nothing transitions through it, only its emptiness is queried, and a newly observed failure clears it.

**Coordinator row** — partition key constant, row key the client unique id: `IndexerName` and `LastSeenExecutionEnd` (the end time of the newest execution already turned into observations).

### Eligibility

A document is eligible for submission when:

```
(LastDispatchedAt is null or LastFailureAt > LastDispatchedAt)  and  Attempts < MaxAttempts  and  SkipDispatchReason is null
```

A document becomes eligible when a *newly observed* failure post-dates its last submission, and stops being eligible the moment it is submitted. Recording a failure also clears any `SkipDispatchReason`: the failure is evidence the indexer is processing the document again, so whatever held it out no longer applies.

This self-paces with no additional rule. A failure is observed exactly once — the checkpoint retires the execution that reported it — so a second worker pass fifteen minutes later finds no trigger. Attempts therefore advance at one per indexer execution, because an indexer execution is the only thing that can produce a new failure.

It is also self-limiting for ordinary ingestion. Because a reset run processes only the reset documents, no *new* failures can be discovered during a retry episode; the eligible set only shrinks as documents stop failing. A document that fails every time costs `MaxAttempts` reset runs and then goes quiet. No cooldown state is needed to bound this.

### Retry budget belongs to a blob revision

`SourceETag` is what makes the budget belong to a revision rather than a path. When recording a failure, the worker reads the current blob properties: if the ETag differs, it records the revision and resets `Attempts` to zero before eligibility is evaluated. This lets even an exhausted document receive retries when a re-upload fails normal indexing. A missing blob leaves the existing budget unchanged; dispatch handles `SourceGone`. The worker also rechecks the revision immediately before submission to cover changes since observation. Reset Docs does not write the blob, so repeated attempts against an unchanged revision retain the same ETag and the same budget.

The first sighting of a document needs no special case: `SourceETag` is empty, so it differs, so the current revision is adopted and the reset of an already-zero counter does nothing. A changed blob is submitted in the same pass rather than held back for ordinary change detection — change detection would indeed have indexed it anyway, but skipping to save one reset run is an optimisation for a problem nobody has observed, and it cost a branch and a rule to explain.

## Worker pass

Each invocation reconciles state and exits; a long indexer execution is never awaited inside a process. The job holds a global lock, because deciding whether an indexer is idle and owning the reset selection it creates cannot be done by two instances independently.

```
1. status = GetIndexerStatus(indexerName)                    // absent indexer → skip client
2. observations = ObserveExecutions(status, LastSeenExecutionEnd)
3. RecordFailures(observations)
4. LastSeenExecutionEnd = newest observation's end time
5. SaveCoordinator()
6. Dispatch()
```

**Observe.** Every retained completed execution newer than the checkpoint, oldest first, with document errors resolved to source blob URLs. One source blob fans out to many chunks, so the first error for a blob carries the cause. Diagnostics retained: a warning when execution history no longer reaches the checkpoint (runs were never observed), a count of errors whose key resolves to no source, and an error log when an execution fails carrying no document errors.

**Record.** Per resolved error: reject keys that do not name a blob in this client's container, skip paths excluded by convention (private-data folders, folder stubs), read the current blob revision and reset its retry budget if the ETag changed, then upsert `LastFailureAt` and `LastError`, creating the row if absent. `LastDispatchedAt` is preserved so an old failure does not become a new trigger merely because the blob changed. No outcome is inferred.

**Dispatch.** Guarded, then a single submission — see below.

## Dispatch

Three guards.

1. Dispatch is enabled and this client is in the configured allowlist
2. **No reset selection is outstanding** — neither `mode == indexingResetDocs` nor a non-empty `resetDatasourceDocumentIds` / `resetDocumentKeys`. The state is set when Reset Docs is called and persists *while the indexer is idle*, until a run has processed every identifier in the selection. Adding to a selection that has not been consumed would let a second trigger ride along with work already in flight, and muddy which run answers for which document. Rather than merely waiting, the worker requests a run — see [Clearing an outstanding selection](#clearing-an-outstanding-selection)
3. **The last execution was not itself a reset run** — `StatusDetail == IndexerExecutionStatusDetail.ResetDocs` identifies one. Guard 2 clears the moment a selection is fully processed, so without this a second selection could go out immediately and an ordinary run would never get a turn. The signal is read from the execution, not remembered in a phase field

### Two guards that were removed

Earlier drafts also blocked dispatch while an execution was in progress, and while the last execution carried a top-level `errorMessage`. Both were dropped because **both latch**: a condition assumed to be transient becomes permanent exactly where the worker is most needed.

An indexer whose runs approach its own 30-minute period is never observed idle, so the in-progress guard would block every pass forever. An indexer that times out on every run carries an `errorMessage` on every execution, so that guard would too. In both cases retries stop silently on the busiest, most backlogged clients — the ones generating the failures.

Neither was protecting correctness. Submitting while a run is in flight is harmless: that execution fixed its reset state at start, so the next run consumes the new selection, and the accompanying Run request simply returns `409`. And the backpressure the `errorMessage` guard was reaching for already exists in the model — dispatch needs a trigger, a trigger needs an observed document error, and an execution that fails at run level without document errors produces none.

What the `errorMessage` guard did cost is real but bounded: a run that produces document errors *and* then dies on `maxRunTime` will now have a reset selection injected, so the following run does reset documents rather than backlog. The normal high-water mark is retained, so the backlog resumes on the run after, and guard 3's alternation already caps this at one reset per two runs. `errorMessage` is still logged at error level whenever present.

Then every eligible row is revalidated against its blob. Exactly one outcome excludes a document: the source is gone (`SourceGone`). Anything else is included, adopting the current revision on the way through.

An earlier draft also excluded blobs carrying `AzureSearch_Skip = true`. That branch is unreachable: the flag is only ever applied at creation time — by `CreateFolder` on the folder stub, and by the deep-dive copy in `ChatService` — never added to an already-indexed blob. A blob created with it is skipped by the indexer, so it never errors and never gets a row, and the folder stub is rejected by the excluded-path check before that. Should an operator ever set the flag by hand during an incident, the worker will submit the document and spend its attempts on resets the indexer ignores — bounded at `MaxAttempts`, then silent.

An excluded document has its `SkipDispatchReason` recorded, which holds it out of the eligible set. Without that, nothing about its row would ever change again — `Attempts` only moves on submission — so a deleted blob would stay eligible forever, costing a blob-properties call and, in the skip-metadata case, a repeated warning on every pass. The reason both drains the set and answers "why was this never retried?" from the table itself.

`LastDispatchedAt` and the `Attempts` increment apply only to rows actually submitted, so one increment per accepted submission remains the sole way the counter moves. Both are written **after** the Reset Docs call returns, not before. Charging first would mean a transient Search failure consumed the trigger and spent an attempt on a retry that never happened — and nothing would re-trigger the document, since the indexer does not revisit an unchanged blob. The opposite risk, a crash between the accepted call and the write, costs at most one uncounted attempt and one redundant identifier in a later selection, because the selection is a set rather than a queue.

Finally one `ResetDocuments` call carrying the computed parent document key for every included blob URL, followed by Run. Identifiers are **appended** (`overwrite: false`), not replaced. Guard 2 means there is never a selection of ours to replace, and appending cannot destroy one an operator created by hand — including in a race where guard 3 read the status a moment before that selection appeared.

There is no batch cap. The whole eligible set ships in one selection; splitting it would multiply the per-run reset cost described above. If a client ever accumulates enough failures to approach a request-size limit this needs revisiting, but a cap chosen before observing real volumes would be a guess.

### Clearing an outstanding selection

When guard 2 finds a selection outstanding, the worker calls Run and returns without dispatching. Waiting for the schedule would leave the selection unconsumed for up to a full cycle, and clearing it is a precondition for the next retry anyway.

No check is made for a run already being in flight: Run then returns `409`, which is handled as benign, and the selection stands. Spending one refused API call is cheaper than a guard that can latch.

Note the worker cannot tell its own outstanding selection from another operator's, because no batch record is kept. It does not need to: running is not replacing. The state is in practice almost always one the worker itself created, since a `409` or `429` on its own Run leaves exactly this. Nothing this worker does can discard someone else's selection — submissions **append** (`overwrite: false`) rather than replace, so even if guard 3 were bypassed by a race the two selections would merge rather than one being lost.

The earlier design's separate "a pending reset selection this worker did not create" guard is therefore dropped. It only worked because the phase field proved the worker had nothing outstanding, making any selection found necessarily foreign — an inference that no longer holds.

This deliberately does **not** extend to guard 3. Launching an ordinary run to satisfy the "normal run between selections" rule sooner would roughly double indexer executions to hurry along work that is best-effort by nature, and on S3 HD and Serverless services on-demand runs draw down the shared 24-hour cumulative runtime quota. Retries wait for the schedule there.

### Run is explicit

Waiting for the schedule would work — the next scheduled run consumes the selection — but because a reset run performs no ordinary ingestion, it would spend a scheduled slot on reset-only work and delay normal indexing by a full 30-minute cycle. Calling Run immediately places the reset work *between* scheduled runs, leaving the normal cadence undisturbed.

`409` (an execution had already started) and `429` (throttled) are both caught and logged as benign: the selection remains pending and the next run consumes it. Under the earlier design a `409` was dangerous, because the code assumed the already-running execution consumed the selection and later inferred success from it. Nothing infers anything now.

## What the status API reveals about a truncated run

There is **no** documented "the indexer finished everything it had to do" flag — no `moreDocumentsAvailable`, no completion percentage. `IndexerExecutionStatus` has four values (`success`, `transientFailure`, `inProgress`, `reset`), and a run cut short by `maxRunTime` gets no distinct value.

Per-execution fields available from `executionHistory` ([Get Indexer Status](https://learn.microsoft.com/en-us/rest/api/searchservice/indexers/get-status?view=rest-searchservice-2025-11-01-preview)):

| Field | Meaning | Used |
|---|---|---|
| `errorMessage` | "The error message indicating the top-level error, if any" — distinct from item-level `errors[]` | Logged only |
| `statusDetail` | `resetDocs` / `resync` — identifies an execution that was processing a reset selection | Guard 3 |
| `mode` | `indexingAllDocs` / `indexingResetDocs` per execution in REST, but **not surfaced per execution by the pinned SDK** — see below | — |
| `itemsProcessed`, `itemsFailed` | Volume worked; a large `itemsProcessed` suggests a backlog being drained | — |
| `initialTrackingState`, `finalTrackingState` | The documented way to observe change-tracking position | — |

`finalTrackingState` is the closest thing to "how far did it get", but its **content is an opaque string**. The `LastFullEnumerationStartTime` / `NameHighWaterMark` shape appearing in Microsoft's examples is observed, not contracted. Comparing initial against final for equality is safe; parsing the JSON inside it is not, and this design does not.

**The SDK's object model differs from the wire format**, verified against `Azure.Search.Documents 11.7.0-beta.1` as pinned by the project. REST places `mode` and `statusDetail` on each execution result and `currentState` on the status object. The SDK instead exposes `CurrentState` on `IndexerExecutionResult` — "all of the state that defines and dictates the indexer's current execution" — and `SearchIndexerStatus` has no `CurrentState` at all, so the indexer's outstanding mode is reached through `status.LastResult.CurrentState.Mode`, as the existing code already does. There is **no `Mode` property on `IndexerExecutionResult`** in this version, so per-execution mode is unavailable; `StatusDetail` (`IndexerExecutionStatusDetail.ResetDocs`, "indicates that the reset that occurred was for a call to ResetDocs") is what guard 5 uses instead. Revisit if the package is ever upgraded to v12, where the shape follows REST.

Guard 3 gives ordinary indexing its turn between reset selections using only documented fields. The other signals here are recorded rather than acted on: a guard built on `errorMessage` latches on a persistently timing-out indexer, which is why it was removed.

## Configuration

Read from environment variables, following the other console workers.

| Setting | Default | Purpose |
|---|---|---|
| `IndexerRetry__Enabled` | `false` | Controls new submissions only. Observation and failure recording run regardless |
| `IndexerRetry__Clients` | empty | Client unique ids allowed to dispatch; empty means every active client |
| `IndexerRetry__MaxAttempts` | `5` | Reset submissions per blob revision |

Retry delays, backoff and batch size are not configurable because they no longer exist. Pacing comes from the indexer's own 30-minute schedule; the worker's CronJob runs every 15 minutes.

## Rollout

### Controlled failure fixture

`AltusAiIndexerSkills.TextMerge.Merge` throws `InvalidOperationException` when the filename or extracted page content contains the exact, case-sensitive marker `IndexerError_af6aeb8daaf04e56a493326f0908f3b1`. The filename check provides an easy failure fixture. The content check allows overwriting the same blob path with changed content to test a new ETag and retry-budget reset; retain the marker to keep failing, or remove it to allow normal processing. Existing private-folder and empty-stub exclusions take precedence.

The exception propagates as HTTP 500 for the entire custom-skill request, including other records in its batch. Use a dedicated test client with disposable documents. Confirm document errors in the indexer execution history before checking retry state.

### Rollout stages

Two stages, each revertible through chart values alone.

1. **Collect only** (`Enabled: false`). Observation and failure recording run; dispatch returns before touching Search. No risk to indexing. Left in place for several days it produces a real inventory of what fails and how often, which no amount of design can substitute for.
2. **Live, one client** (`Enabled: true`, `Clients` set to one id). Validate the recorded `BlobUrl` identifiers against the source blobs before enabling dispatch, since invalid identifiers are silently ignored. Confirm that `Attempts` advances once per indexer execution rather than once per worker pass, that the eligible set shrinks across an episode, and that ordinary ingestion continues throughout. Compare a reset run's duration against a normal run to settle the container-scan question. Clear `Clients` to allow dispatch for all active clients after the pilot.

Abort at any stage by submitting an empty `documentKeys` list with `overwrite=true`.

**Deployment configuration:** the umbrella chart disables `indexerRetry` by default. All five ANA GitOps payloads (QA, QA sandbox, production, Emerie QA and Emerie production) explicitly enable the CronJob for collection only (`IndexerRetry__Enabled: "false"`), each using the same storage account, Search endpoint and shared secret as its API. Retry dispatch remains a separate rollout setting. Publishing the chart and promoting its version are separate rollout steps.

## Open questions

**Does a normal run with tolerated document failures report `success` or `transientFailure`?** Client indexers are created with `MaxFailedItems = -1`, which should mean item failures do not fail the run, but the existing implementation assumes the opposite. Stage 1 answers this within days. It matters because guard 6 must stay keyed on `errorMessage` — which stays null when item failures are tolerated — and must never be widened to `status`, which would otherwise block every dispatch permanently.

**Does `datasourceDocumentIds` avoid the full container scan?** Microsoft documents document-cracking for `documentKeys`. If the scan happens regardless, reset episodes are expensive and the case for one large selection over several small ones gets stronger still.

**Does an execution that fully processed its reset selection always carry `StatusDetail = ResetDocs`?** The value is documented as indicating "the reset that occurred was for a call to ResetDocs", which reads as covering the whole reset execution, but it is preview surface and guard 3 depends on it. If it turns out to be unset on some reset executions, guard 3 can fall back to comparing the execution's `EndTime` against the worker's own last submission time.

## Accepted limitations

- **A document that succeeds leaves a row that looks unfinished** — `Attempts` short of the maximum, `LastDispatchedAt` newer than `LastFailureAt`. That combination *is* the success signal, read after the fact by a person rather than acted on by the worker.
- **A permanently broken document is visible only by query** (`Attempts >= MaxAttempts`), not by an alert. The final attempt is logged at error level so it reaches Application Insights.
- **Recovery is not verified.** This pipeline writes many chunks per blob, so even an affirmative index query proves only partial progress without an expected-chunk manifest. The earlier design's `AwaitingVerification` state existed for this and is not retained; the worker makes no recovery claim at all, which is weaker but honest.
- **A re-uploaded blob gets a fresh budget only when it fails again.** Ordinary change detection handles the successful case, so nothing is lost.

## Deferred

- **Test harness.** The worker has no end-to-end coverage. `SearchModelFactory` can construct `SearchIndexerStatus`, `IndexerExecutionResult` and `SearchIndexerError`, so no recorded HTTP is needed; a small indexer gateway seam, a blob-properties seam and an injected clock make the runner testable. Highest-value scenarios: one attempt per execution rather than per worker pass; a failure observed once is submitted once; an ETag change resets the budget; each dispatch guard.
- **Affirmative verification.** If the table proves hard to read operationally, querying the index for a document's chunks is a check rather than a guess, and would allow retries to stop early.
- **A skip reason that no longer applies is cleared only by a new failure.** That is the intended behaviour — a blob unskipped or restored is reindexed by change detection, and only a fresh failure means the retry worker has anything to do — but it means the table can hold a stale `SkipDispatchReason` indefinitely for a document that is now perfectly healthy.

# Asynchronous Chat Image Uploads — Design

_Updated: 2026-09-11. Phase 1 implemented locally; Phase 2 cleanup and Phase 3
scale hardening remain deferred._

## 1. Goal

Start uploading an image as soon as the user selects or pastes it, while the user continues writing the message. Sending the message should transfer only text and references to already uploaded images, reducing the delay between pressing **Send** and model generation.

This supersedes the upload lifecycle in [chat-image-attachments-design.md](chat-image-attachments-design.md). The current image work is not committed or deployed, so its schema and API can be changed directly: there is no existing image data or released multipart contract to preserve. The new flow needs a temporary, session-owned image that does not yet belong to a message.

The scope remains static PNG, JPEG, and WebP chat attachments. Indexed documents, generated images, and deep-dive files are unchanged.

## 2. Recommended flow

```text
Select/paste image
  -> FE shows a local preview and starts POST /Chat/{sessionId}/Images
  -> API authorizes the active session, validates and stores the image
  <- API returns an opaque image ID and validated metadata

Continue typing while uploads run

Press Send
  -> FE waits for every selected image to be ready
  -> POST /Chat/CompleteChat/{sessionId} as JSON with text + ordered image IDs
  -> API atomically attaches those images to the new user message
  -> existing standard/agentic model and response flow continues
```

“Asynchronous” here means upload is a separate request running concurrently with message composition. The initial implementation should remain API-mediated rather than issuing browser write SAS URLs. This preserves the current server-side byte/format/dimension/animation validation, keeps storage private, avoids a second finalize protocol, and is sufficient for the current 5 MiB-per-image limit. Direct browser-to-Blob upload can be considered later if API bandwidth becomes a measured bottleneck.

## 3. Phase 1 — Separate upload and send

### 3.1 Data model

Adapt `ChatImage` to represent either an uploaded draft or an attached image:

| Field | Change |
| --- | --- |
| `ChatSessionId` | Add required FK and index. This is the ownership boundary before a message exists and avoids deriving the blob path/authorization through a nullable message. |
| `ChatMessageId` | Make nullable. `null` means uploaded but not sent; non-null means attached to its permanent owning user message. |
| `Position` | Make nullable until attachment, or keep a non-semantic default. Recommended: nullable and set from the order of IDs in the completion request. |
| Existing metadata | Keep `UniqueId`, sanitized `FileName`, validated `ContentType`, `SizeBytes`, `Width`, `Height`, and audit fields. |

Add relationships `ChatSession.ChatImages` and optional `ChatImage.ChatMessage`. Configure the direct session FK with `Restrict`/`NoAction` deletion behavior to avoid a second cascade path through `ChatMessage`; sessions use soft deletion in normal operation anyway. Once attached, require the message to be a user message in the same session. Domain methods should express the transition (`CreateUploaded` and `AttachTo`) rather than allowing callers to set foreign keys directly. `AttachTo` must fail if the image is already attached, inactive, or belongs to another session. Tests must enforce the redundant session/message-session invariant.

Revise the existing uncommitted `ChatImage` migration and model snapshot to create this final schema directly. Do not add a backfill or transitional migration: there are no deployed `ChatImage` rows to preserve. The blob path remains `{sessionUniqueId}/{imageUniqueId}`.

A persisted `Uploading` state is not required for Phase 1 because a `ChatImage` row is created only after Blob upload succeeds. Consequently, SQL-save failures can still leave an untracked blob, as they can today. Phase 2 addresses that reconciliation. The upload response is not returned until both Blob upload and SQL persistence succeed, so every ID visible to the FE is ready to attach.

### 3.2 Upload API

Add:

```http
POST /Chat/{sessionId}/Images
Content-Type: multipart/form-data

image=<one binary file>
```

Success: `201 Created` with a `ChatImageUploadDto`:

```json
{
  "uniqueId": "1b35fef7-2765-4b3b-83ab-68d22a612fd8",
  "fileName": "architecture.png",
  "contentType": "image/png",
  "sizeBytes": 123456,
  "width": 1200,
  "height": 800
}
```

The endpoint must:

- authorize the caller against the active session before accepting bytes;
- apply the same configured byte, pixel, decoded-format, and animation checks as completion does today;
- accept one image per request so images can upload concurrently and retry independently;
- generate the image ID on the API, upload with `If-None-Match: *`, then save an unattached `ChatImage` row;
- return no write credential and no SAS read URL. The FE can retain its local object URL until completion returns the normal `ChatImageDto` read link;
- rely on normal POST caching semantics and avoid logging bodies, filenames where
  unnecessary, or signed URLs;
- apply authenticated endpoint rate limiting if the deployment already has a standard mechanism; cleanup is not a substitute for abuse control.

The configured maximum image count is enforced again at send time. It cannot safely be enforced as “total drafts in session” because abandoned drafts from earlier composer activity may exist.

### 3.3 Completion API

Add an ordered field to `CompleteChatDto`:

```json
{
  "userMessage": "Compare these diagrams",
  "imageIds": [
    "1b35fef7-2765-4b3b-83ab-68d22a612fd8",
    "83bccbe7-53d6-4a1c-a987-d7ac7da6eecb"
  ],
  "reasoning": "agentic",
  "signalRConnectionId": "..."
}
```

`imageIds` determines display/model order. Reject duplicate IDs, too many IDs, nonexistent IDs, and any image that is inactive, already attached, or owned by another session. Return one generic `400` attachment error for these cases so the API does not disclose whether an ID exists elsewhere. Session authorization remains mandatory before resolving IDs.

Within one EF transaction, create the user message, attach each selected image with its zero-based position, update the session name if needed, and save. Commit before model generation, preserving the existing retry/restore behavior. Concurrency must be enforced in SQL, not only by an in-memory check: use a conditional update/concurrency token or equivalent so two completion requests cannot claim the same draft. A failed claim rolls back the message and all image claims.

After attachment, history construction, SAS generation/renewal, standard chat, agentic chat, restore, and completion responses keep their current contracts. Image-only sends remain valid; a send with neither non-whitespace text nor image IDs remains invalid.

Replace the uncommitted multipart completion action rather than retaining two image-submission contracts. Completion uses the existing JSON action with `imageIds`; image bytes are accepted only by the separate upload endpoint. Text-only JSON sends remain unchanged. This avoids maintaining multipart binding, duplicate validation paths, and an ambiguous request containing both files and uploaded IDs.

Do not automatically replay completion after an uncertain HTTP failure. As today, restore and reconcile first because the user message may already have been persisted.

### 3.4 Frontend behavior

Replace `File[]` composer state with ordered attachment items containing a stable client key, the `File`, local preview URL, status (`uploading`, `ready`, or `failed`), progress where available, and returned image ID/metadata.

- Start upload immediately after upload-dialog confirmation or image paste. Upload up to a small fixed concurrency (recommended 2) to avoid competing requests exhausting browser/API connections.
- Preserve selection order independently of completion order. Send IDs in selection order.
- Show per-image uploading, ready, failed, retry, and remove states. A failed upload must not silently disappear.
- Disable **Send** while any selected attachment is uploading. If one failed, require retry or removal before sending. Text editing stays enabled throughout.
- Removing an uploading image aborts the browser request where possible. Removing a ready image removes it from the future `imageIds`; Phase 1 intentionally leaves its server copy abandoned.
- Keep the local preview until completion supplies the durable image DTO, then revoke its object URL. Also revoke URLs on remove, session navigation, and component unmount.
- On session change, abort outstanding requests and never attach IDs uploaded for the previous session.
- Apply client-side count/size/type checks for quick feedback, while treating server validation as authoritative.
- On send success, reconcile the current optimistic user message with returned `images`. SignalR still carries assistant streaming only.

Uploading starts on explicit selection/confirmation, not merely when the file picker opens. The user may press Send immediately after selecting a file; waiting for the already-running upload is acceptable and should be represented as “Finishing image upload…” rather than appearing unresponsive.

### 3.5 Errors and idempotency

Phase 1 does not need an upload idempotency key: retrying a failed/uncertain upload may create a second draft, but only the ID included in completion is attached and Phase 2 will remove the other. The FE must not retry automatically after an uncertain successful upload unless the user asks it to; it should surface retry and tolerate the stale duplicate.

Suggested status behavior:

| Status | Case |
| --- | --- |
| `201` | Upload validated, stored, and draft row persisted |
| `400` | Invalid image or invalid/ineligible `imageIds` on completion |
| `401` / `403` | Existing authentication/access behavior |
| `404` | Session inactive or inaccessible, following the renewal endpoint's non-disclosing behavior |
| `413` | API/proxy body limit |
| `500` / network failure | Upload outcome uncertain; offer retry, and rely on later cleanup for duplicates/orphans |

Stable machine-readable error codes would improve FE behavior but are not required solely for this change; if introduced, apply them consistently to both upload and completion validation.

### 3.6 Phase 1 tests and rollout

Backend tests:

- the revised initial migration creates the final session-owned, optionally message-attached schema and can roll forward and back on an empty database;
- upload authorization, inactive sessions, all existing validation limits, sanitized filenames, and storage failure behavior;
- only ready/unattached images from the same session can be claimed;
- request ordering becomes persisted `Position` ordering;
- duplicate IDs and concurrent double-claim attempts fail without partial messages;
- message plus image claims are transactional;
- both chat modes and history receive uploaded images exactly as before;
- multipart completion rejects image submission, while separate upload plus JSON completion and text-only JSON completion work.

Frontend verification:

- typing remains responsive while multiple images upload;
- selection order survives out-of-order upload completion;
- send waits for uploads; failure/retry/remove and navigation cancellation work;
- image-only send, optimistic reconciliation, SignalR/HTTP races, and restore work;
- local object URLs are revoked and no SAS/upload identifiers enter durable browser storage or analytics.

Deploy the backend and frontend as one coordinated, still-disabled feature release, then enable image uploads only after both are present. Keep `ChatImages__Enabled` as the shared feature gate: when disabled, the separate upload endpoint rejects files while text-only chat continues. Observe upload duration/failure rate and time from Send to model invocation before considering the phase complete. Phase 1 creates intentionally unbounded abandoned storage, so Phase 2 should follow promptly; do not treat Phase 1 alone as the final production operating model.

## 4. Phase 2 — Abandoned and stale image cleanup

An abandoned image is a `ChatImage` with `ChatMessageId = null`. It can result from removal after upload, navigating away, closing the browser, retrying an uncertain upload, or never sending the draft.

### 4.1 Retention policy

Add a configurable retention period, recommended default **24 hours**, based on the row's UTC creation timestamp. Cleanup must never delete an attached image, regardless of age. Do not use SAS expiry as the retention clock; SAS links are access credentials, not ownership state.

Run cleanup as a scheduled, single-run console job following the repository's existing sync/job hosting pattern, not inside API request processing. Use bounded batches and an overlap-safe query. Multiple runners must be harmless.

For each stale unattached row:

1. re-check that it is still unattached at claim/delete time;
2. delete the exact blob `{sessionUniqueId}/{imageUniqueId}` with “not found” treated as success;
3. delete or soft-delete the SQL row according to the repository's audit convention;
4. record counts and failures without logging SAS URLs.

Blob deletion before row deletion is deliberate: a transient Blob failure leaves a row for retry. A crash after Blob deletion but before SQL deletion is also safe because the next run treats a missing blob as success. A concurrent completion claim must win over cleanup; use a conditional row claim/concurrency check before deleting bytes so cleanup cannot delete an image that has just been attached.

### 4.2 Orphan blob reconciliation

Rows do not cover a Blob upload that succeeded before its SQL insert failed. Add a less frequent reconciliation pass that lists blobs under valid session GUID prefixes, compares them to image IDs in SQL, and deletes unreferenced blobs older than the same safety window. Blob `LastModified`, not filename parsing alone, supplies the age check. Restrict parsing to the exact expected `{sessionGuid}/{imageGuid}` shape and never touch other containers or unexpected paths automatically; count/report unexpected entries for investigation.

At the expected scale, a batched list-and-compare job is sufficient. If volume grows, use a temporary DB table/set-based comparison or Blob Inventory rather than loading the whole container into memory.

### 4.3 Optional explicit discard

After scheduled cleanup is proven, optionally add:

```http
DELETE /Chat/{sessionId}/Images/{imageId}
```

It may delete only an unattached image owned through that active session and should be idempotent from the FE's perspective. The FE can call it when a ready image is removed, reducing waste quickly. This is an optimization, not the correctness mechanism: browsers disappear and requests fail, so scheduled cleanup remains required.

### 4.4 Cleanup tests and operations

- retain drafts younger than the threshold and every attached image;
- clean old unattached rows/blobs, tolerate missing blobs, and retry transient failures;
- prove the attach-versus-cleanup race cannot delete a claimed image;
- reconcile old untracked blobs but retain recent blobs that may be between upload and SQL save;
- scope all operations to the configured private chat-image container and expected path shape;
- emit metrics for scanned, deleted, retained, malformed, missing, and failed items, plus oldest pending age;
- alert on repeated failures or sustained growth of stale drafts/orphans;
- initially run in report-only mode in QA, then delete in QA, then enable per production environment.

Session soft deletion remains unchanged: attached images are retained with chat history under the existing policy. Unattached images in an inactive session are still eligible after the draft retention period.

## 5. Phase 3 — Optional scale hardening

Only pursue this phase based on telemetry:

- direct browser-to-Blob uploads using a narrowly scoped, short-lived create-only SAS plus a finalize endpoint;
- upload idempotency keys to eliminate duplicates after uncertain retries;
- resumable/block uploads and richer progress for materially larger limits;
- antivirus/content-safety scanning if the security policy requires it;
- per-user/session draft quotas and throttling to control intentional storage abuse.

Direct upload makes validation and lifecycle more complex: the API must create an upload reservation, issue a blob-specific credential, finalize by reading and validating the completed blob, reject unfinalized IDs at send time, and clean expired reservations. It should not be the Phase 1 default for 5 MiB images.

## 6. Decisions and non-goals

- Upload ownership is the chat session, not merely the user/client. This makes cross-session attachment impossible and keeps blob paths unchanged.
- Completion accepts opaque image IDs, never client-supplied blob URLs or paths.
- Only completion attaches an uploaded image to a message. Uploading alone does not create chat history or invoke the model.
- Uploaded IDs are single-use: an image belongs to exactly one user message. Reusing an older image continues to mean reattaching the source file or sending a text follow-up while it remains in model history.
- Phase 1 deliberately accepts temporary abandoned storage; Phase 2 supplies eventual cleanup.
- No new SignalR upload events are needed. Upload state belongs to the browser request that initiated it.
- Existing display SAS renewal, model capacity checks, history window, and generation-failure semantics remain unchanged after an image is attached.

## 7. Implementation footprint

Expected changes span:

- `ana-design-docs`: this lifecycle/API design;
- backend domain model, EF configuration/migration, upload and attachment services, controller/DTO contract, tests, and later cleanup job/chart scheduling;
- primary frontend composer state/UI, upload mutation, completion request, optimistic reconciliation, and manual/build verification;
- backend API documentation and deployment notes after implementation.

No storage-account or container change is required for Phase 1. Phase 2 needs job deployment/configuration and Blob delete/list permissions in addition to the API's current upload/read/SAS permissions; verify the exact identity used by the scheduled job before granting them.

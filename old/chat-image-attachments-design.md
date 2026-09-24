# Chat Image Attachments — Conceptual Design

_Updated: 2026-09-11. Backend phases 1–5 implemented; deployment preparation done,
live Phase 6 verification pending. See the implementation checkpoint below._

> **Upload contract superseded:** image bytes are now uploaded before send using the
> separate session-scoped flow in
> [async-chat-image-uploads-design.md](async-chat-image-uploads-design.md). The multipart
> completion contract described below is retained as historical context and is not
> supported by the current implementation.

## Goal

Users can attach images, compare them, and ask follow-up questions in chat.
Images go directly into the main conversation in both standard and agentic
chat, including Planner and Synthesizer. Repeated image-token costs are accepted
for the current small user base.

## Storage

- Use a separate **private blob container in the same storage account as indexing data**.
- Chat images do not enter the search index or client knowledge base.
- Blob path: `{sessionUniqueId}/{imageId}` inside the attachment container.
  Ownership and access come from SQL; no user or client ID in the path.
- Image bytes live in Blob Storage. SQL stores image IDs, metadata, and message
  ownership. Derive the blob path from the session and image unique IDs.
- Session deletion sets `InactiveDate`. Retain its image rows and blobs;
  do not issue new SAS links while the session is inactive. Existing links
  remain valid until expiry. No automatic image deletion for v1.

## SQL table

Add `ChatImage`, with one row per image:

| Column | Purpose |
| --- | --- |
| `Id` | Internal primary key, following existing conventions |
| `UniqueId` | API-generated image GUID; used in the blob path and API responses |
| `ChatMessageId` | Required foreign key to the owning user message |
| `Position` | Required image order within the message |
| `FileName` | Sanitized user-provided basename, used for display and model references |
| `ContentType` | Validated image media type |
| `SizeBytes` | Stored image size |
| `Width`, `Height` | Image dimensions |
| Audit fields | Existing `BaseEntity` fields |

Resolve the session through the message; no separate `ChatSessionId` or stored
blob path is needed. Make `UniqueId` unique and index `ChatMessageId`.

## Basic flow

1. **Send:** FE submits message text and image files in one multipart API request.
   The API checks session access, validates the images, assigns their unique IDs,
   and uploads them to Blob Storage. No separate image upload operation.
2. **Persist:** Save the user message and its `ChatImage` rows together in one
   SQL transaction before calling the model. Every image has a required message
   link from creation; no pending-image state is needed.
3. **Model input:** The API generates fresh read-only SAS URLs and includes each
   image in the conversation immediately after a synthetic text label containing
   its one-based order number and sanitized filename. The label is generated only
   for model input and is not appended to the saved user message. Azure OpenAI
   fetches the images directly.
   Save the assistant response after generation. No base64 embedding, generated
   descriptions, or separate image-inspection tool.
4. **Display:** The API returns short-lived SAS URLs for FE to display images
   directly from Blob Storage, including when restoring chat history.
5. **Follow-up:** Images follow the existing history window: the first message
   and the last 10 messages. Include images only when their owning message is
   included, preserving image order and generating fresh SAS URLs. Images outside
   this window remain stored and visible in the UI, but the model cannot see
   them; users may need to attach them again. No separate image history window
   for v1.

If generation fails, retain the saved user message and images, including
unanswered user messages in the normal history window. The user can send
"retry" as a new message; the model receives the earlier request and images
while they remain in that window. No image re-upload, special retry endpoint,
or generation-status system is needed for v1. If a connection is lost, FE can
restore session history to check what was saved. FE can show local previews
while sending.

Orphaned blob cleanup is deferred; blobs left by failed SQL saves may remain
in storage for now.

## Access

The storage endpoint is reachable by browsers and Azure OpenAI; the container
remains private. The API authorizes access before issuing a read-only SAS for
an individual image.

Use the same backend SAS-generation service for FE and model requests.
FE sends image files with new messages; the API resolves stored images for
history and retries. SAS URLs are temporary
and are not stored in SQL or logs; expired display links can be renewed through
the API.

## V1 defaults

- **Images:** static PNG, JPEG, and WebP; up to 4 images per message, 5 MiB and
  20 megapixels per image. Validate actual content and dimensions server-side;
  reject animated or unsupported files. Keep limits configurable.
- **Send contract:** multipart `message` contains the existing completion DTO
  as JSON; repeated `images` parts contain files in display order. Continue
  accepting existing JSON requests for text-only chat. Allow text, images, or
  both, but reject empty sends. Preserve existing response fields and include
  the saved user message ID and image references.
- **SAS links:** default lifetime of 1 hour for both display and model input.
  Return each display URL with its expiry; provide an authorized endpoint to
  renew it. Generate model links afresh per completion and renew before a retry
  if needed. Keep model runs within the link lifetime.
- **Capacity:** enforce the deployed model's image-count and context limits
  across the whole selected history. If exceeded, return a clear error asking
  the user to start a new session; do not silently omit images.

## Implemented API specification for frontend work

This section describes the current backend implementation, not a proposed API.
Paths below are relative to the configured API base URL. All API calls use the
existing authenticated request helper and `Authorization: Bearer <access-token>`.
Session IDs and message IDs are SQL integers; image IDs are UUID strings.
JSON property names are camelCase. Dates are ISO 8601 strings; treat `timestamp`
as UTC and parse `expiresAt` with its supplied offset.

### Endpoints

| Method and path | Request | Success response |
| --- | --- | --- |
| `POST /Chat/CompleteChat/{sessionId}` | JSON for text-only sends, or multipart for sends with files | `200`, one `ChatResponseDto` after generation finishes |
| `GET /Chat/RestoreSession/{sessionId}` | No body | `200`, one `ChatSessionResponseDto` containing saved messages and fresh image links |
| `POST /Chat/{sessionId}/Images/{imageId}/Renew` | No body; `imageId` is the image's `uniqueId` | `200`, one `ChatImageDto` with a fresh URL and expiry |

Use the existing session creation/list/deletion endpoints; images do not introduce
a new session type. There is no separate upload, attachment-delete, batch-renewal,
or retry endpoint. The send and restore responses are **objects, not arrays**,
even where older controller Swagger annotations imply an array for restore.
The cacheable restore endpoint sets `Cache-Control: no-store` because its response
may contain temporary SAS URLs. Image POST responses rely on normal HTTP semantics.

### Send request

The completion JSON has these fields:

| Field | Type | Default / behavior |
| --- | --- | --- |
| `userMessage` | `string` or `null`, optional | Text to send. Omit or use `""` for image-only sends. Whitespace-only text with no files is rejected. |
| `reasoning` | `"low"`, `"medium"`, `"high"`, or `"agentic"`, optional | Defaults to `"low"`; unrecognized values fall back to low. Send lowercase values. File/deep-dive sessions always use standard chat, even with `"agentic"`. |
| `signalRConnectionId` | `string` or `null`, optional | Existing connection ID for streaming. Without it, the HTTP request still completes normally. |
| `useTaxonomy` | `boolean`, optional | Defaults to `false`; existing behavior. |
| `useContactsContext` | `boolean`, optional | Defaults to `false`; existing behavior. |
| `focusMode` | `boolean`, optional | Defaults to `false`; restricts retrieval using existing focus rules. |

For text-only sends, continue to use `Content-Type: application/json`:

```json
{
  "userMessage": "What differs from the first image?",
  "reasoning": "agentic",
  "signalRConnectionId": "current-connection-id"
}
```

For sends with images, use `multipart/form-data`:

| Part name | Content | Required |
| --- | --- | --- |
| `message` | A **string** containing the JSON object above, serialized with `JSON.stringify` | Yes, including image-only sends |
| `images` | One binary `File` per part; repeat the same part name in display order | At least one for an image send; text-only multipart also works |

Do not send the JSON as a file/Blob part. Do not send `images[]`, base64, image
URLs, or existing image IDs. The JSON DTO ignores an `images` property; only
multipart file parts attach new images. The backend derives each saved image's
zero-based `position` from multipart file order.

Browser request example (use the application's existing token acquisition;
the token below is already acquired):

```ts
const body = new FormData();
body.append("message", JSON.stringify({
  userMessage: text, // "" is valid when files are attached
  reasoning,
  signalRConnectionId,
  useTaxonomy,
  useContactsContext,
  focusMode,
}));
for (const file of selectedFiles) body.append("images", file, file.name);

const response = await fetch(`${apiBaseUrl}/Chat/CompleteChat/${sessionId}`, {
  method: "POST",
  headers: { Authorization: `Bearer ${accessToken}` },
  body,
});
// Handle non-2xx responses before reading the success DTO.
```

Let the browser generate `Content-Type` and its multipart boundary. When adapting
the existing axios helper, remove any default JSON content-type for `FormData`.
Do not automatically resend this POST after an error or network timeout.

Client-side selection defaults are 4 images per send, 5,242,880 bytes per image,
and 20,000,000 pixels (`width * height`) per image. Accept static PNG, JPEG, and
WebP (`image/png,image/jpeg,image/webp`); reject animated images. These are
configurable server limits, and server validation remains authoritative: it
checks decoded bytes rather than trusting the filename or MIME header.
The backend reduces each multipart filename to a basename, removes control
characters, and limits it to 255 characters before persistence. Filenames are
display/model metadata only and never determine the decoded image format.

### Response types and image ownership

```ts
interface ChatImageDto {
  uniqueId: string; // UUID; stable key and renewal identifier
  position: number; // zero-based within its owning user message
  fileName: string; // sanitized user-provided basename
  contentType: "image/png" | "image/jpeg" | "image/webp";
  sizeBytes: number;
  width: number;
  height: number;
  url: string;      // temporary, opaque HTTPS SAS URL
  expiresAt: string; // ISO 8601 DateTimeOffset
}

interface ChatResponseDto {
  id: number;
  images: ChatImageDto[];
  message: string | null;
  messageType: 0 | 1 | 2; // User=0, Assistant=1, AssistantCurated=2
  messageContext: MessageContextItemDto[] | null; // existing citation DTO
  timestamp: string;
  curatedResponse: CuratedResponseDto | null; // existing curation DTO
}

interface ChatSessionResponseDto {
  id: number;
  userClientId: number;
  name: string | null;
  chatResponses: ChatResponseDto[] | null;
  files: FileUrlDto[] | null; // existing deep-dive documents, not chat images
  isFileSession: boolean;
  startTime: string;
}
```

Reuse the existing citation, curation, and deep-dive file DTOs unchanged.
`messageType` is serialized as a **number**, not `"User"` or `"Assistant"`.
The new `images` array is empty when there are no attachments. For compatibility
with an older deployed API, the FE may normalize missing `images` to `[]`.

**Completion response:** `id`, `message`, `messageType`, `messageContext`, and
`timestamp` describe the newly saved **assistant** response. `images` belongs to
the saved user message from this send, not to the assistant. Map the images to
the optimistic user message for the current request, then create/update the
assistant message using `id`.

Example `200` response for a one-image send (URL is illustrative):

```json
{
  "id": 902,
  "images": [{
    "uniqueId": "1b35fef7-2765-4b3b-83ab-68d22a612fd8",
    "position": 0,
    "fileName": "architecture.png",
    "contentType": "image/png",
    "sizeBytes": 123456,
    "width": 1200,
    "height": 800,
    "url": "https://storage.example/chat-attachments/session/image?SIGNED_QUERY",
    "expiresAt": "2026-09-10T13:00:00+00:00"
  }],
  "message": "The image shows a chart with three series.",
  "messageType": 1,
  "messageContext": null,
  "timestamp": "2026-09-10T12:00:00Z",
  "curatedResponse": null
}
```

**Restore response:** `chatResponses` contains saved messages in ascending message
ID order. Each item's `id` is that item's own message ID. The user item (`id=901`,
`messageType=0` in this example) carries `images`; the assistant item (`id=902`,
`messageType=1`) has `images: []`. Image-only user messages can have `message: ""` and
must still be rendered. Unanswered user messages are included. Existing curated
responses remain nested under `curatedResponse` on the associated message.

### Display and renewal

Render images and their filenames directly using the returned URL, ordered by
`position`. A user-facing ordinal is `position + 1`; for example,
`Image 1: architecture.png`. Use
`uniqueId` as the stable image key; URLs change on restore/renewal. Keep the URL
opaque, including its query string. Do not reconstruct storage paths or append
the API bearer token to blob requests. An `<img src={image.url}>` needs no API
authorization header. Avoid recording SAS URLs in analytics, logs, or durable
client storage; keep them in current session state with `expiresAt`.

Renew a visible image shortly before its expiry or once when loading fails due
to expiry:

```http
POST /Chat/42/Images/1b35fef7-2765-4b3b-83ab-68d22a612fd8/Renew
Authorization: Bearer <access-token>
```

No request body is required. The response is the same `ChatImageDto` shape shown
inside `images`, with a fresh `url` and `expiresAt`. Default lifetime is one hour;
use the returned expiry rather than hardcoding it. Replace the matching image
reference and retry display once; avoid an unbounded renewal loop. Deduplicate
concurrent renewal requests for the same image. Renewal does not call the model
or create a message. Deleted/inaccessible sessions and missing images return
`404`; stop renewing and refresh session state or show an unavailable image.
Previously issued links remain usable until expiry after session soft deletion.

### Streaming, failures, and retry UX

Keep the existing SignalR handlers. `ReceiveStreamCompleted` carries the existing
`messageId` for the **assistant** and `sessionId` as a string. It does **not**
carry images or URLs. Get those from the HTTP completion response
or restore. There is no new image-upload-completed or user-message-saved event.
Do not assume `ReceiveStreamError` fires for every failure: validation and some
agentic failures return through HTTP only.

Recommended FE sequence:

1. Check selection limits, show local object-URL previews, and submit text plus
   files once. Disable duplicate sends while that request is in progress.
2. Stream the assistant through existing handlers. On HTTP success, replace the
   current optimistic user's previews with returned image references. Reconcile
   the assistant by `id` to avoid creating
   it twice when both SignalR and HTTP completion arrive. Revoke local object URLs
   when replaced or when files are removed.
3. On generation failure or uncertain delivery, restore before deciding whether
   to resend. The user message and images may already be saved, and even an
   assistant response can have been saved before a later delivery failure.
   An error body does not include the saved message ID. There is no idempotency
   key or exactly-once guarantee across separate POST requests.
4. When restore shows an unanswered saved request, let the user send a **new**
   text message such as `retry`, with no file parts. This adds a new user message
   and uses the earlier request/images while they remain in model history.
   Reattach files only for a genuinely new attachment or when the old images
   have fallen outside the model history window.

The model window includes the first eligible message plus the last 10 eligible
messages (user and assistant), including the newly saved request and unanswered
requests, without duplicates. Restored/displayed history is not limited to this
window. Do not imply that every image visible in the UI is still visible to the
model. No API field reports whether an image is in the current model window.

### Error contract and rollout constraints

Application errors normally use `{ "message": "..." }`, with no stable error
code field. ASP.NET model-binding errors may instead return Problem Details with
`title`/`errors`. Auth/proxy errors may have empty or non-JSON bodies. Unexpected
backend errors currently return `500` with the text `Unexpected error occurred`,
despite a JSON content-type; do not unconditionally JSON-parse error responses.

| Status | Current cases relevant to images | FE handling |
| --- | --- | --- |
| `400` | Empty send; invalid multipart JSON; uploads disabled; image count/byte/pixel/format/animation validation; model capacity; generation failure mapped to an application error | Show the returned message. Validation occurs before persistence; capacity/generation errors can occur after persistence, so restore before retrying. |
| `401` / `403` | Missing/expired authentication or access denial; existing send/restore session checks use `401` for an unauthorized caller | Use existing auth/access handling. |
| `404` | Renewal: inactive/inaccessible session or image not found; route mismatch | Do not keep renewing. |
| `413` | Request rejected by server/proxy body limits | Explain that the upload is too large; reduce files/size. |
| `415` | Unsupported send content-type | Send JSON or browser-generated multipart. |
| `500` or network/proxy timeout | Unexpected failure or uncertain delivery | Restore/reconcile; never blindly replay the upload. |

Useful current application messages include:

- `Image attachments are not enabled.`
- `Enter a message or attach an image.`
- `Attach at most 4 images per message.` (count follows configured limits)
- `Animated images are not supported. Attach a static image.`
- `This conversation exceeds the model's image or context limit. Start a new session and attach the images you need.`
- `Session not found or access denied.` / `Image not found.` (renewal)

The existing send/restore session lookup can return `500` for a missing or
soft-deleted session; the new renewal endpoint explicitly returns `404`. Do not
assume identical not-found behavior across these endpoints. Error text is for
display, not a versioned machine-readable discriminator.

There is currently **no capability/limits endpoint** and no feature-enabled field
in the response. Coordinate FE feature gating and configured limits with deployment;
the backend defaults to `ChatImages__Enabled=false`. Keep text-only chat working
when image submission is unavailable. This API is implemented locally; Phase 6
live environment verification remains pending.

Implementation references: [controller](../altus-ai-assistant/AltusAiAssistantApi/Controllers/ChatController.cs),
[completion DTO](../altus-ai-assistant/AltusAiAssistantApi/Models/CompleteChatDto.cs),
[response DTO](../altus-ai-assistant/AltusAiAssistantApi/Models/ChatResponseDto.cs),
[image DTO](../altus-ai-assistant/AltusAiAssistantApi/Models/ChatImageDto.cs),
[turn persistence](../altus-ai-assistant/AltusAiAssistantApi/Services/Maf/ChatTurnStore.cs),
[image access service](../altus-ai-assistant/AltusAiAssistantApi/Services/ChatImages/ChatImageService.cs),
and [error middleware](../altus-ai-assistant/AltusAiAssistantApi/Middleware/ErrorHandlerMiddleware.cs).

## Implementation phases

Implement sequentially, with a separate diff/commit for each phase. Each phase
builds on the previous ones. Per the implementation request, continue without
review pauses and report completed phases. Include focused tests and verification
summaries at the checkpoints.

| Phase | Scope | Review checkpoint |
| --- | --- | --- |
| 1. SQL model | Add `ChatImage`, message relationship, EF mapping, indexes, and migration. | Required message ownership and schema match this design; no chat behavior changes. |
| 2. Storage and SAS | Add private-container storage, image validation, derived paths, SAS generation, and configurable limits. | Valid images can be stored and read through scoped SAS links; invalid images are rejected. |
| 3. Message lifecycle | Separate saving the user message from saving the assistant response in both chat modes. | Text-only chat still works; failed generation retains one unanswered message, including during internal retries. |
| 4. Standard chat and API | Add multipart send, image persistence, standard model image input, history selection, restore metadata, and SAS renewal endpoint. | Upload, answer, history display data, and retry-by-new-message work; existing JSON requests remain compatible. |
| 5. Agentic chat | Carry image content through Planner and Synthesizer, preserving it through intermediate stages. | Both stages receive images; comparison and follow-up questions work without forced retrieval. |
| 6. Deployment readiness | Prepare container/identity permissions, application configuration, ingress limits, and integration checks. | Verify both chat modes against Azure, access restrictions, soft-deleted sessions, SAS expiry, and model limits; deploy separately. |

Keep image submission disabled until both chat modes are ready; use tests to
review intermediate phases. Frontend UI implementation remains separate.
Orphaned blob cleanup is outside v1 scope.

Filename and ordinal support was added after the initial phases: `ChatImage`
stores the sanitized filename, `ChatImageDto.fileName` exposes it to the FE, and
model history inserts `Image {position + 1} of {message image count}` plus the
JSON-escaped filename immediately before each corresponding image content part.
Existing rows receive deterministic `image-N` filenames during migration.

This design covers the backend and its frontend contract. Frontend UI work,
images from indexed content/documents, and image generation are separate work.

## Implementation checkpoint — Phase 1 (2026-09-10)

Phase 1 was completed first and committed as `377be3f` in the backend repository.
The later phases continued without review pauses, as requested; see below.

- Added `ChatImage`, `ChatMessage.ChatImages`, the context set, and EF mapping.
  The constructor requires a user message and generates a GUID. `Position` is
  zero-based; `SizeBytes` uses `long`/SQL `bigint`; `ContentType` is required
  with a 100-character maximum. Actual image validation belongs to Phase 2.
- Added migration `20260910000244_Add_Table_ChatImages` and updated the model
  snapshot. The table has a required message foreign key, unique `UniqueId`
  index, `ChatMessageId` index, and existing `BaseEntity` audit fields/filter.
  Physical message deletion follows the existing cascade convention; session
  soft deletion retains messages and images. No blob paths or SAS URLs are stored.
- Verification: `dotnet test tests/AltusAiAssistantApi.Tests/AltusAiAssistantApi.Tests.csproj --no-restore --verbosity quiet`
  passed all 47 tests. New tests cover schema, unsaved message/image relationship
  tracking, user-message ownership, metadata guards, soft-delete retention,
  migration/model consistency, and forward/rollback SQL generation.
- `git diff --check` passed. Migration generation and tests did not connect to
  SQL; the migration has not been applied to a database.

## Implementation checkpoint — continued phases (2026-09-10)

| Phase | Status | Backend commit / verification |
| --- | --- | --- |
| 1. SQL model | Complete locally | `377be3f`; 47 API tests passed |
| 2. Storage and SAS | Complete locally | `a75d3ab`; 57 API tests passed |
| 3. Message lifecycle | Complete locally | `af88828`; 61 API tests passed |
| 4. Standard chat and API | Complete locally | `6bb2a82`; 73 API tests passed |
| 5. Agentic chat | Complete locally | `24ef9ff`; 75 API tests passed |
| 6. Deployment readiness | Prepared; live verification pending | 77 API tests passed after deployment-related hardening; Terraform formatting and integration script syntax passed |

Implemented multipart completion, actual PNG/JPEG/WebP decoding with static-image
and configurable limits, required image persistence before model generation,
fresh model/display SAS links, authorized renewal, ordered history including
unanswered requests, image input in both modes, direct image-answer planning,
and model image/context capacity checks. Responses include image references;
restore places images on their owning user message.

Phase 6 adds private-container provisioning and optional workload identity roles
in `altus-ai-terragrunt/modules/assistant-services/chat-images.tf`, disabled-by-default
Helm configuration, and appropriate gateway timeouts. Model-specific image limits,
context capacity, and worst-case image token cost must be configured before enabling
uploads. Budgeting is conservative; it may reject before the exact model limit.

See the [integration script](../altus-ai-assistant/scripts/Test-ChatImages.ps1) for the
remaining checks. The script creates and soft-deletes
test sessions and calls both models; it has **not** been run against Azure.

Remaining Phase 6 verification: SQL Server migration, Linux native decoder/image
build, Helm rendering, Terraform validation/plan, and live Azure checks for both
chat modes, real SAS expiry, cross-user access, browser CORS, configured model limits,
and telemetry. Local Helm is unavailable, Docker's daemon is unavailable, and
Terraform 1.12.2 is below the module's >=1.15 requirement. No environment was deployed,
no infrastructure was applied, and no database migration was run.

Frontend UI remains outside this design's implementation scope. Uploads remain
disabled by default until environment verification. No automatic orphan cleanup.

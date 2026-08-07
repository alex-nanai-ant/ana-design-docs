# Chat Image Attachments — Technical Design

_Date: 2026-08-05_
_Scope: `altus-ai-assistant` (API) only. No frontend work — see §11._
_Related: a separate design will cover images arriving through **indexed content** (Slack files, email attachments, Confluence). This document deliberately shares its ID and tool contract with that future work — see §10._

## Goal

Let a user attach an image to a chat message and have the assistant reason about
it — including follow-up questions several turns later — **without image tokens
ever entering the main conversation**.

The API exposes upload, attachment retrieval, and the chat contract. Rendering,
paste capture, and thumbnails are frontend concerns and are out of scope.

## 1. Core decision: the main conversation stays text-only

Images are **never** placed into the message list sent to the main model loop.
Instead each image is described once, and the description travels as ordinary
text; the pixels are reachable only through a tool that makes its own isolated
vision call (§5).

This is the decision everything else follows from, and it is worth stating why
it beats the obvious alternative of putting image content parts into the user
message.

**Cost.** Images in history are re-sent on every model call, and a "call" is not
once per turn. `FunctionInvokingChatClient` loops — a turn with three tool calls
is four round trips, each carrying the full message list. The agentic path adds
Planner and Synthesizer as separate invocations on top. Worse,
[`ChatTurnStore.BuildHistory`](../altus-ai-assistant/AltusAiAssistantApi/Services/Maf/ChatTurnStore.cs#L63-L67)
**pins the first message of a session forever** once history exceeds
`MaxContextMessages`. An image pasted in the first message would be re-billed on
every model call for the entire life of the session, without bound. Passing a
SAS URL instead of bytes does not help: Azure OpenAI fetches and tokenizes the
image identically, so the token cost is unchanged and only the request payload
shrinks.

**Blast radius.** Because the main conversation stays text:

- [`CompleteChatService`](../altus-ai-assistant/AltusAiAssistantApi/Services/CompleteChatService.cs#L85-L90) message construction is unchanged
- `BuildHistory` stays synchronous and in-memory — no blob reads per turn
- `PlannerInput` and the MAF workflow types need no widening; the agentic path works unmodified
- SharpToken counting stays accurate, because the conversation really is all text
- the "image parts are only valid on user messages" constraint of the Chat Completions API never applies

Image bytes exist in exactly **one** code path — the tool's sub-call. That is a
far smaller thing to get right than multimodal message assembly across two chat
paths and a workflow graph.

**What we accept.** The model never sees pixels in the main conversation, so it
cannot volunteer what nobody asked about ("by the way, the error in the corner
says…"). Descriptions are lossy in unpredictable directions. This is a
deliberate trade, mitigated by §4.3 and §5.

## 2. Storage

Images go to a **single, private, shared blob container**, separate from the
per-client content containers.

**Separate container.** The search indexer's data source binds to a container
named exactly `client.UniqueId`
([`CreateBlobDataSource`](../altus-ai-assistant/AltusAiAssistantApi/Services/IndexingService.cs#L138-L150)).
Any other container is *structurally* un-indexable — there is no filter for
anyone to forget later. Chat images must never enter the client knowledge base.

**Shared, not per-user.** Container boundaries only isolate when something
untrusted holds container-scoped credentials; here the API holds the connection
string and nothing else touches storage directly. Per-user containers would buy
no real isolation while costing: a container creation on the interactive upload
path, N chances to get public-access wrong, and — the deciding factor — the loss
of a single Azure lifecycle rule for retention. Pasted images accumulate
indefinitely otherwise.

**Partitioned by client, not user.** The confidentiality boundary in this
platform is the client, not the consultant; one user works across several
clients. Path layout:

```
<chat-attachments>/{clientUniqueId}/{sessionUniqueId}/{attachmentId}
```

This gives client offboarding, user deletion, and session deletion each a prefix
delete.

**Do not create this container via `BlobStorageService.CreateContainer`** — it
hardcodes `PublicAccessType.Blob`
([BlobStorageService.cs:209](../altus-ai-assistant/AltusAiShared/BlobStorageService.cs#L209)),
which is why existing content blobs are anonymously readable by URL. This
container is created once, with `PublicAccessType.None`.

Consequence: there is **no SAS infrastructure in the codebase today**. Serving
these images back requires either streaming through the API or minting
short-lived blob-scoped SAS after an authorization check. Either is acceptable;
the API must never hand out a raw blob URL for this container.

## 3. Schema

A new `ChatMessageAttachment` table, not a JSON column on `ChatMessage` — the
tool resolver looks attachments up by ID, and descriptions are written after the
row is created.

Fields, conceptually: opaque `AttachmentId`, owning `ChatMessageId` (nullable
until the turn is persisted, see §4.1), `ChatSessionId`, blob path, media type,
byte size, `Description`, and the usual `BaseEntity` audit fields.

**`AttachmentId` is an opaque string.** For chat attachments a GUID is
sufficient. The tool contract (§5) accepts an opaque ID specifically so that the
content-image work can later introduce content-hash IDs — which double as a
dedup and description-cache key for high-volume ingested images — without
changing the tool signature or the prompt protocol.

## 4. Lifecycle of an attachment

### 4.1. Upload

Upload is a **separate call from the chat completion**, not multipart on the
chat request: it validates and describes, which is slow enough that it should
not sit inside the send path.

Validation is by **content sniffing, not extension** — magic bytes, an allowlist
of media types, a max byte size, and a max pixel-dimension guard against
decompression bombs. Note that the existing manual-upload path performs no
server-side extension validation at all
([AdminService.cs:467](../altus-ai-assistant/AltusAiAssistantApi/Services/AdminService.cs#L467));
this endpoint must not repeat that.

The upload returns an `AttachmentId`. The row exists before any message does;
it is bound to a `ChatMessageId` when `PersistTurn` writes the turn. Rows never
bound to a message (user attached, then never sent) are swept by the retention
rule.

### 4.2. Description

Generated **once**, at upload, stored on the row.

The description prompt matters more than anything else in this design, and it
has one unusual requirement: **descriptions must state their own limits.** Not
"a bar chart of quarterly revenue" but "bar chart, 12 categories; axis labels
transcribed verbatim; individual bar values not read precisely." A description
that advertises what it does *not* contain is what tells the model when to reach
for the tool. Without this the model will answer confidently from an inadequate
description — the primary failure mode of the whole design.

Guidance for how the model should treat descriptions and when to call the tool
belongs in [`ChatPromptBuilder`](../altus-ai-assistant/AltusAiAssistantApi/Services/Maf/ChatPromptBuilder.cs).

### 4.3. The arrival turn

The common case is paste-and-ask-immediately, where a description written before
anyone knew the question is a poor substitute — and relying on the model to
choose to call the tool adds a round trip and a gamble to the most important
turn.

So: on the turn an attachment is **first** sent, the API makes a
question-conditioned vision call server-side *before* the main completion,
passing the user's message as the question. Its answer is injected into that
turn's user message as text alongside the stored description.

No tool-choice dependency, no extra loop iteration, and the main conversation is
still text-only. One vision call can produce both artifacts — the generic
description for storage and the question-conditioned answer for this turn.

### 4.4. History

`BuildHistory` renders a past user message as its text plus the descriptions of
its attachments. Because history is text, the pinned-first-message problem
dissolves by construction: a first-message image costs a few hundred fixed
tokens forever instead of a full image payload per call.

Two adjacent fixes belong in this change: `BuildHistory` uses `.First()` /
`.TakeLast()` over an **unordered** EF navigation collection — it works today
only because materialization follows the Id clustered index, and attachments
would inherit that implicit assumption. Add an explicit ordering. Separately,
the pinning at
[ChatTurnStore.cs:65](../altus-ai-assistant/AltusAiAssistantApi/Services/Maf/ChatTurnStore.cs#L65)
pins the first user message but not its assistant reply, so the model sees an
opening question with no answer — pre-existing, but more conspicuous when that
message is "here is a diagram".

## 5. The `InspectImage` tool

`InspectImage(attachmentId, question) -> string`.

The tool **does not return the image**. On the Chat Completions API used by
`GetChatClient(...).AsIChatClient()`, image parts are valid only on user
messages and tool results are text-only; `FunctionResultContent.Result` is
JSON-serialized, so returned bytes would land as a base64 string the model reads
as text. Instead the tool makes its own vision call — image + the specific
question + a slice of conversation context — and returns **text**. The protocol
constraint stops applying entirely.

This is what makes description-only viable: full fidelity stays available at
turn 50 as easily as turn 2, and image cost is paid only when a question
actually needs pixels.

**Wiring.** Tools are built per run by
[`ChatToolProvider.CreateChatTools(ctx)`](../altus-ai-assistant/AltusAiAssistantApi/Services/Maf/ChatToolProvider.cs#L42-L71)
from `MafRunContext`; the session's attachments extend that context.
`DescriptionGenerationService` is the existing precedent for a service making
its own self-contained completion.

**Use the non-keyed `IChatClient`,** not the `"notifying"` one — the keyed
client exists to stream tool-call and preamble events to the user's SignalR
connection, and a sub-call must not emit UI events as though it were main-loop
activity.

**Authorization is not optional.** An ID reaching the tool has been through the
model, so it can be hallucinated or carried forward in history. The resolver
must confirm the attachment belongs to a session the caller may access under the
current run's client — never trust that the ID appeared in context.

**Set `reasoning_effort` explicitly** on the sub-call rather than inheriting;
the Synthesizer hardcodes `"high"`, which is wasteful for "read the third column
of this table". Set a timeout appropriate to a vision call — not the 5-second
`NetworkTimeout` copied from
[`GetCuratedFileTitle`](../altus-ai-assistant/AltusAiAssistantApi/Services/ChatService.cs#L421-L431).

## 6. Usage accounting and caps

**Accounting.** `PersistTurn` records token counts from the main response only,
so vision sub-call tokens — the expensive ones — would be invisible to
`UsageStatsService`, systematically under-reporting exactly the cost this design
exists to control. Sub-call usage must be accumulated into the turn total. The
agentic path already solves this shape with `MafRunState.AddUsage`; the
non-agentic path has no equivalent and needs one.

**Caps.** Each `InspectImage` invocation re-sends the image to the sub-model, so
a turn needs a maximum invocation count plus a per-turn memo so an identical
(id, question) pair does not re-fire. Also cap attachments per message and per
session.

**Quota.** One turn can now be: main call + N tool round trips + M vision
sub-calls, all against a single Azure OpenAI deployment — the single-deployment
rule is enforced at startup
([Program.cs:61-62](../altus-ai-assistant/AltusAiAssistantApi/Program.cs#L61-L62)),
so there is no spillover. Worth sizing against TPM before load reveals it.

## 7. API surface

- **`POST /chat/sessions/{sessionId}/attachments`** — multipart upload; validates, stores, describes; returns `AttachmentId` and the media type.
- **`CompleteChatDto`** gains a list of attachment IDs for the current turn. `UserMessage` stays a string.
- **`GET /chat/attachments/{attachmentId}/content`** — authorized read-back for rendering, streamed or via short-lived SAS redirect. Never a raw blob URL.
- **`RestoreSession`** returns attachment references per message so a restored session can render its images.
- **`DELETE`** on an attachment, for removal before send.

## 8. Agentic path

Descriptions flow into the agentic path for free, because history and the user
message are text.

The tool does **not**, and this ship does not change that.
[`SynthesizerExecutor`](../altus-ai-assistant/AltusAiAssistantApi/Services/Maf/Executors/SynthesizerExecutor.cs#L57-L58)
deliberately builds a no-tools agent and the Planner receives only
`ProbeSearch`. So in `reasoning=agentic` mode the model has descriptions but
cannot inspect further. This is a **documented limitation of v1**, not an
oversight: granting the Synthesizer a tool turns synthesis into a tool loop and
interacts with response streaming, which is a separate change and should be
made on its own merits.

## 9. Failure modes to handle explicitly

- **Blob missing** after the row exists — degrade to the stored description, do not fail the turn.
- **Description generation fails at upload** — accept the upload, mark the row, and let the arrival turn (§4.3) or the tool supply detail.
- **Vision sub-call fails or times out** mid-turn — return a tool error string the model can reason about, never abort the completion.
- **Unknown or unauthorized `attachmentId`** — a clean "not available" tool result, not an exception.

## 10. Forward compatibility with content images

The follow-on work — images arriving through Slack, email, and Confluence — is
deliberately kept able to reuse this design:

- **`InspectImage` is one tool over an opaque ID.** A second resolver for indexed images requires no change to the signature or the prompt protocol.
- **Descriptions as findable pointers.** For indexed images the description is what gets embedded and retrieved; short descriptions plus on-demand detail suit that far better than long self-sufficient ones, and `SplitSkill` chunks at 800 tokens, which a long description would fragment.
- **Content-hash IDs** for ingested images give dedup and description caching for free at a volume where that matters.

Nothing in this ship should assume attachments are chat-only beyond the resolver
itself.

## 11. Out of scope

- **All frontend work** — paste capture, thumbnails, removal UI, rendering restored sessions. The API contracts in §7 are the deliverable.
- **Images in indexed content** — Slack files, email attachments, Confluence attachments. `AllowedIndexFileExtensions` stays unchanged; no image reaches a client content container as a result of this ship.
- **Granting the agentic Synthesizer the tool** (§8).
- **Image output** — the assistant does not generate or return images.
- **Images embedded inside documents** (charts in PDFs, figures in docx).
- **Multimodal embeddings / image similarity search.**

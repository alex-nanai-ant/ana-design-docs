# Content Sources — Fetch, Index, and Chat Usage

_Date: 2026-04-16_

Catalogue of every value in [`DocumentSourceType`](../AltusAiShared/DocumentSourceType.cs), describing — for each source — how content is fetched, indexed, and consumed during chat completion.

> Confluence and ExternalConfluence fetch/ingest details are already covered in [confluence-indexing-and-hierarchy.md](confluence-indexing-and-hierarchy.md); this document covers them only at a summary level.

---

## 0. Shared pipeline (applies to every source)

All sources write to the **same per-client blob container** (`{clientUniqueId}`) and are picked up by a **single unified indexer** per client. Source-specific behaviour is captured entirely in the blob path and metadata.

### Blob metadata
Constructed via [`BlobMetadata.Create()`](../AltusAiShared/BlobMetadata.cs) and attached on upload. Common keys:
- `sourceType` — the `DocumentSourceType` enum value
- `date` — document date (received / modified / uploaded)
- `externalUrl` — link back to the origin (Confluence page, Grain recording, Slack permalink…)
- `author` — Slack user / email sender (optional)
- `slackChannelName` — Slack channel (optional)
- `mailboxAddress` — mailbox the email was pulled from (optional)

### Indexer & skillset
Created by [`IndexingService.CreateIndex()`](../AltusAiAssistantApi/Services/IndexingService.cs#L31):
- **Data source**: the blob container
- **Skillset** ([IndexingService.cs:147-250](../AltusAiAssistantApi/Services/IndexingService.cs#L147-L250)):
  - `SplitSkill` — 800-token chunks
  - `WebApiSkill` → `AltusAiTextMergeSkill` — prepends filename to each chunk
  - `AzureOpenAIEmbeddingSkill` — 1536-dim vectors (`text-embedding-ada-002`)
- **Schedule**: every 30 minutes ([IndexingService.cs:262](../AltusAiAssistantApi/Services/IndexingService.cs#L262))
- **Scoring**: 1.5× freshness boost for documents from the last 720 days

### Retrieval during chat
- [`ChatService.GetFromSearchIndex()`](../AltusAiAssistantApi/Services/ChatService.cs#L840) runs vector search against the per-client index.
- Exposed to the LLM as the `SearchInternalDocuments` tool ([ChatService.cs:984](../AltusAiAssistantApi/Services/ChatService.cs#L984)); **not** registered when `session.IsFileSession == true` ([ChatService.cs:197](../AltusAiAssistantApi/Services/ChatService.cs#L197-L204)) — file sessions answer purely from uploaded file contents.
- Friendly source names in the user prompt are expanded to enum values inside [`SearchInternalDocumentsFunc`](../AltusAiAssistantApi/Services/ChatService.cs#L1037-L1064) — e.g. `Slack` → `Slack + SlackFile`, `Confluence` → `Confluence + ExternalConfluence`, `Email` → `Email + EmailFile`.
- Supported filters: `sourceType` (OData eq/or) and `date` range ([ChatService.cs:860-870](../AltusAiAssistantApi/Services/ChatService.cs#L860-L870)).
- Citations rendered with `sourceType`, `author`, and `slackChannelName` metadata ([ChatService.cs:946-963](../AltusAiAssistantApi/Services/ChatService.cs#L946-L963)).

---

## 1. Confluence

See [confluence-indexing-and-hierarchy.md](confluence-indexing-and-hierarchy.md).

**Summary**: `AltusAiConfluenceSync` Function, 15-min incremental + 60-min deleted-page scan, HTML→Markdown via `ReverseMarkdown`, blob path `{SpaceKey}/{PageTitle}.md`. Retrieved via the shared vector search; user can ask "from Confluence" (expanded to include `ExternalConfluence`).

---

## 2. ExternalConfluence

Same pipeline as Confluence — the only difference is the source-type tag, which is decided per-configuration:

### Fetch
Same code path as internal Confluence in [`ConfluenceSyncService`](../AltusAiShared/Confluence/ConfluenceSyncService.cs). The distinction is made at upload time based on the configured base URL:

```
confluenceApiSettings.BaseUrl.Contains(Constants.ConfluenceBaseUrl)
    ? DocumentSourceType.Confluence
    : DocumentSourceType.ExternalConfluence;
```
([ConfluenceSyncService.cs:175](../AltusAiShared/Confluence/ConfluenceSyncService.cs#L175))

### Index
Identical to Confluence — same container, path, metadata schema, indexer, and skillset. Only the `sourceType` metadata value differs.

### Chat usage
Retrieved via the shared index. `sourceTypes = 'Confluence'` from the LLM is expanded to both `Confluence` and `ExternalConfluence` ([ChatService.cs:1052-1054](../AltusAiAssistantApi/Services/ChatService.cs#L1052-L1054)) — users do not distinguish the two.

---

## 3. Slack (messages)

### Fetch
- **Function**: [`AltusAiSlackSync/Function.cs`](../AltusAiSlackSync/Function.cs).
  - `IncrementalSync` — every 10 min ([Function.cs:27](../AltusAiSlackSync/Function.cs#L27))
  - `DeepSync` — daily at 00:00 ([Function.cs:34](../AltusAiSlackSync/Function.cs#L34))
  - `DeepSyncHttp` — manual per-channel trigger with `lookbackHours` param ([Function.cs:40-71](../AltusAiSlackSync/Function.cs#L40-L71))
- **Source-of-truth registry**: `SlackLinks` SQL table (joined against `Clients`) — each active link is `(channelId, channelName, clientUniqueId)` ([Function.cs:84-87](../AltusAiSlackSync/Function.cs#L84-L87)).
- **Orchestration**: [`SlackSyncService.RunSyncJob()`](../AltusAiShared/Slack/SlackSyncService.cs#L62):
  - Fetches channel history via `conversations.history`; filters bot/system subtypes and empty-text messages ([SlackSyncService.cs:99](../AltusAiShared/Slack/SlackSyncService.cs#L99)).
  - Fetches thread replies for messages with `reply_count > 0` via `conversations.replies` ([SlackSyncService.cs:507-521](../AltusAiShared/Slack/SlackSyncService.cs#L507-L521), 1.2s delay between calls to respect Slack Tier-3 rate limits).
  - Resolves user IDs to display names ([SlackSyncService.cs:524-540](../AltusAiShared/Slack/SlackSyncService.cs#L524-L540)).
  - If an incremental sync is requested but no full sync has completed yet, it auto-upgrades to Full ([SlackSyncService.cs:71-80](../AltusAiShared/Slack/SlackSyncService.cs#L71-L80)).
- **Chunking strategy** ([SlackSyncService.cs:18](../AltusAiShared/Slack/SlackSyncService.cs#L18), [cs:160-211](../AltusAiShared/Slack/SlackSyncService.cs#L160-L211)):
  - Target 750 tokens per chunk; messages are never split mid-message (each message + its thread replies is atomic).
  - When a chunk exceeds 750 tokens it is **sealed** (uploaded as final); a new chunk starts.
  - The **last chunk stays open** and is appended to on subsequent incremental syncs.
  - The `DeepSync` path rebuilds recently-touched chunks to pick up retroactive thread replies that would otherwise be missed on messages in sealed chunks ([SlackSyncService.cs:567-642](../AltusAiShared/Slack/SlackSyncService.cs#L567-L642)).
- **State** (all in Azure Table Storage):
  - `SlackSyncRunTableEntity` — one row per sync-job execution (used to check full-sync completion)
  - `SlackSyncChannelStateEntity` — per-channel: last synced message TS, current open-chunk metadata, last file TS
  - `SlackChunkStateEntity` — per-chunk boundaries (first/last message TS, token count) — used to target deep-sync rebuilds

### Index
- **Blob path**: `Slack/{channelName}/{chunkIndex}.md` (via `SlackHelper.GetChunkBlobName`)
- **Metadata** ([SlackSyncService.cs:454-458](../AltusAiShared/Slack/SlackSyncService.cs#L454-L458)):
  - `sourceType = Slack`
  - `date = newest message timestamp in the chunk`
  - `externalUrl = Slack permalink to the chunk's first message`
  - `slackChannelName = <channelName>`
- Picked up by the shared indexer on its next 30-min tick.

### Chat usage
- Vector search via the shared pipeline.
- User-facing source name `Slack` expands to **both** `Slack` and `SlackFile` ([ChatService.cs:1045-1046](../AltusAiAssistantApi/Services/ChatService.cs#L1045-L1046)).
- Citations render `slackChannelName` alongside source type ([ChatService.cs:953-959](../AltusAiAssistantApi/Services/ChatService.cs#L953-L959)).

---

## 4. SlackFile

Files (PDFs, docs, images…) shared in Slack channels, indexed as standalone documents rather than chunks.

### Fetch
Runs inside the same `SlackSyncService.RunSyncJob()` call as messages, via [`SyncFiles()`](../AltusAiShared/Slack/SlackSyncService.cs#L232):
- Lists files via `files.list` ([SlackService.GetChannelFiles](../AltusAiShared/Slack/)).
- Filters by `Constants.AllowedIndexFileExtensions` ([SlackSyncService.cs:270](../AltusAiShared/Slack/SlackSyncService.cs#L270)).
- Skips files already synced (`timestamp <= LastSyncedFileTs`) ([SlackSyncService.cs:277](../AltusAiShared/Slack/SlackSyncService.cs#L277)).
- Downloads the file via its private Slack URL using the bot token ([SlackSyncService.cs:310](../AltusAiShared/Slack/SlackSyncService.cs#L310)).
- Tracks max seen file timestamp in `SlackSyncChannelStateEntity.LastSyncedFileTs`.
- [`CleanupDeletedFiles()`](../AltusAiShared/Slack/SlackSyncService.cs#L376) removes blobs whose Slack file IDs are no longer in the channel listing.

### Index
- **Blob path**: `Slack/{channelName}/{yyyyMMdd}/{safeFileName}_{fileId}{ext}` (see [`SlackHelper.GetFilesFolderName`](../AltusAiShared/Slack/) and [SlackSyncService.cs:289-295](../AltusAiShared/Slack/SlackSyncService.cs#L289-L295)).
- **Metadata** ([SlackSyncService.cs:329-334](../AltusAiShared/Slack/SlackSyncService.cs#L329-L334)):
  - `sourceType = SlackFile`
  - `date = file.created`
  - `externalUrl = file.permalink`
  - `author = <resolved Slack user display name>`
  - `slackChannelName`
- Indexed by the shared indexer. File extraction (PDF/DOCX text) is handled by the built-in Cognitive Search content extraction before `SplitSkill`.

### Chat usage
Retrieved the same way as Slack messages; `sourceTypes = 'Slack'` matches both ([ChatService.cs:1045-1046](../AltusAiAssistantApi/Services/ChatService.cs#L1045-L1046)). Citations include channel and author.

---

## 5. Email

### Fetch
- **Function**: [`AltusAiMailSync/Function.cs`](../AltusAiMailSync/Function.cs).
  - `MailIncrementalSync` — every 10 min ([Function.cs:22-27](../AltusAiMailSync/Function.cs#L22-L27))
  - `MailFullSyncHttp` — manual per-mailbox full resync ([Function.cs:29-52](../AltusAiMailSync/Function.cs#L29-L52))
- **Registry**: `MailboxLinks` SQL table joined against `Clients` ([Function.cs:63-66](../AltusAiMailSync/Function.cs#L63-L66)).
- **Orchestration**: [`MailSyncService.RunSyncJob()`](../AltusAiShared/Mail/MailSyncService.cs#L41):
  - Uses Microsoft Graph via `ClientSecretCredential` ([MailSyncService.cs:37-38](../AltusAiShared/Mail/MailSyncService.cs#L37-L38)).
  - Reads the `Inbox` mail folder, selecting `id, subject, from, receivedDateTime, body, hasAttachments` ([MailSyncService.cs:128-135](../AltusAiShared/Mail/MailSyncService.cs#L128-L135)).
  - Incremental uses OData filter `receivedDateTime ge <lastSynced+1s>` ([MailSyncService.cs:121-126](../AltusAiShared/Mail/MailSyncService.cs#L121-L126)); full sync first deletes the `Email/` blob folder ([MailSyncService.cs:53](../AltusAiShared/Mail/MailSyncService.cs#L53)).
  - Paginates via `OdataNextLink` ([MailSyncService.cs:142-146](../AltusAiShared/Mail/MailSyncService.cs#L142-L146)).
- **State**: Azure Table `MailSyncMailboxState` keyed by `(clientUniqueId, mailboxAddress)`, storing `LastSyncedDateTime` and last error ([MailSyncService.cs:253-264](../AltusAiShared/Mail/MailSyncService.cs#L253-L264)).
- **Body transform**: Plain-text header (`From / Date / Subject`) is prepended; Graph's HTML body is kept as-is (no HTML→Markdown conversion) ([MailSyncService.cs:172-181](../AltusAiShared/Mail/MailSyncService.cs#L172-L181)).

### Index
- **Blob path**: `Email/{yyyy-MM-dd} ({from})/{sanitizedSubject}/{sanitizedSubject}.html` ([MailSyncService.cs:159-181](../AltusAiShared/Mail/MailSyncService.cs#L159-L181)).
- **Metadata** ([MailSyncService.cs:163-169](../AltusAiShared/Mail/MailSyncService.cs#L163-L169)):
  - `sourceType = Email`
  - `date = receivedDateTime`
  - `author = <sender email>`
  - `mailboxAddress = <inbox owner>`
- Picked up by the shared indexer.

### Chat usage
- Vector search via shared pipeline.
- `sourceTypes = 'Email'` expands to **both** `Email` and `EmailFile` ([ChatService.cs:1049-1050](../AltusAiAssistantApi/Services/ChatService.cs#L1049-L1050)).
- Citations render sender and mailbox via the standard `author` field.

---

## 6. EmailFile

Email attachments. Indexed as standalone documents, colocated with the parent email.

### Fetch
Runs inside `MailSyncService.SyncMessage()` when `message.HasAttachments == true`, via [`SyncAttachments()`](../AltusAiShared/Mail/MailSyncService.cs#L194):
- Lists via Graph `/users/{mbx}/messages/{id}/attachments`.
- Only handles `FileAttachment` — skips `ItemAttachment` and `ReferenceAttachment` ([MailSyncService.cs:200](../AltusAiShared/Mail/MailSyncService.cs#L200)).
- Filters by `Constants.AllowedIndexFileExtensions` ([MailSyncService.cs:205](../AltusAiShared/Mail/MailSyncService.cs#L205)).
- Reads `attachment.ContentBytes` directly (inline, no separate download step) ([MailSyncService.cs:210](../AltusAiShared/Mail/MailSyncService.cs#L210)).

### Index
- **Blob path**: same folder as the email body — `Email/{yyyy-MM-dd} ({from})/{sanitizedSubject}/{originalFileName}` ([MailSyncService.cs:211](../AltusAiShared/Mail/MailSyncService.cs#L211)).
- **Metadata**: inherits email metadata (`date`, `author`, `mailboxAddress`) with `sourceType` overridden to `EmailFile` ([MailSyncService.cs:186-189](../AltusAiShared/Mail/MailSyncService.cs#L186-L189)).

### Chat usage
Retrieved the same way as email bodies; `sourceTypes = 'Email'` matches both.

---

## 7. Grain

Grain meeting-recording transcripts. Fetch is a two-stage handoff to Zapier because the Grain API does not expose transcripts directly.

### Fetch
1. **Registration**: A `GrainLink` row is created (via the frontend / admin API) containing the Grain URL but a `null` `BlobPath` — it is unprocessed.
2. **Background poller**: [`GrainSynchronizer`](../AltusAiAssistantApi/GrainSynchronizer.cs) is a hosted `BackgroundService`.
   - Runs on a 10-minute `PeriodicTimer` ([GrainSynchronizer.cs:34](../AltusAiAssistantApi/GrainSynchronizer.cs#L34)).
   - Gated by `appSettings.SyncGrain` ([GrainSynchronizer.cs:63](../AltusAiAssistantApi/GrainSynchronizer.cs#L63)).
   - Selects `GrainLinks` with `BlobPath == null` and `SyncAttemptsCount < Constants.GrainSyncAttemptsCount` ([GrainSynchronizer.cs:66-69](../AltusAiAssistantApi/GrainSynchronizer.cs#L66-L69)).
   - POSTs `{Id, ClientId}` to `appSettings.ZapierWebhookUrl` ([GrainSynchronizer.cs:76-82](../AltusAiAssistantApi/GrainSynchronizer.cs#L76-L82)) and increments `SyncAttemptsCount` regardless of Zapier's response code.
3. **Zapier webhook**: Zapier extracts the transcript from Grain and calls back **anonymously** to [`POST /Integration/ImportGrain`](../AltusAiAssistantApi/Controllers/IntegrationController.cs#L29) — protected by a hard-coded `X-Api-Key` header ([IntegrationController.cs:31-34](../AltusAiAssistantApi/Controllers/IntegrationController.cs#L31-L34)).
   - `multipart/form-data`: `file` = transcript, `data` = JSON `ImportGrainLinkRequestDto` (`GrainId`, `ClientId`, `Title`, `Date`, `Url`).
4. **Ingest**: [`AdminService.ImportGrainLink()`](../AltusAiAssistantApi/Services/AdminService.cs#L1016):
   - Looks up the matching `GrainLink` row. If `BlobPath` already set, bails out (idempotent).
   - Uploads to blob and sets `BlobPath` in the DB to seal it ([AdminService.cs:1031-1038](../AltusAiAssistantApi/Services/AdminService.cs#L1031-L1038)).

### Index
- **Blob path**: `{Constants.GrainFolderName}/{title}_{yyyy-MM-dd}.md` ([AdminService.cs:1033](../AltusAiAssistantApi/Services/AdminService.cs#L1033)).
- **Metadata** ([AdminService.cs:1036](../AltusAiAssistantApi/Services/AdminService.cs#L1036)):
  - `sourceType = Grain`
  - `date = dto.Date` (meeting date)
  - `externalUrl = <Grain recording URL>`
- Picked up on the next 30-min indexer tick (no explicit `RunIndexer` call in this path).

### Chat usage
- Retrieved via shared vector search.
- In the system prompt, Grain is framed as "meetings" — `sourceTypes = 'Grain'` or user phrases like "from meetings" map to it ([ChatService.cs:127](../AltusAiAssistantApi/Services/ChatService.cs#L127), [ChatService.cs:1057](../AltusAiAssistantApi/Services/ChatService.cs#L1057)).
- The system prompt explicitly instructs the model: *"File titles represent calls and meetings. Use these titles for context regarding inquiries about calls and meetings."* ([ChatService.cs:81](../AltusAiAssistantApi/Services/ChatService.cs#L81)).

---

## 8. Manual

Arbitrary files uploaded by admins through the web UI.

### Fetch
No sync — synchronous upload through the API:
- **Endpoint**: `POST /Admin/UploadBlobItems/{clientId}` via [`AdminController`](../AltusAiAssistantApi/Controllers/AdminController.cs) → [`AdminService.UploadBlobItems()`](../AltusAiAssistantApi/Services/AdminService.cs#L307).
- Accepts a list of `(fileName, fileContent)` plus a target `Path` and an `IsOverwrite` flag.

### Index
- **Blob path**: `{user-supplied Path}/{fileName}` — caller controls the folder hierarchy ([AdminService.cs:315](../AltusAiAssistantApi/Services/AdminService.cs#L315)).
- **Metadata** ([AdminService.cs:318](../AltusAiAssistantApi/Services/AdminService.cs#L318)):
  - `sourceType = Manual`
  - `date = DateTime.UtcNow` at upload
- **Indexer is triggered explicitly** immediately after upload (`await _indexingService.RunIndexer(client.Id);` at [AdminService.cs:321](../AltusAiAssistantApi/Services/AdminService.cs#L321)), so manual uploads don't wait for the 30-min tick.

### Chat usage
- Vector search via the shared pipeline, unless the session is a **file session** (see below).
- `sourceTypes = 'Manual'` → `DocumentSourceType.Manual` ([ChatService.cs:1060](../AltusAiAssistantApi/Services/ChatService.cs#L1060)).
- Citations include source type and date only — no author/channel metadata.

#### File session bypass
When a user uploads files directly into a chat session (`ChatSession.IsFileSession == true`), [`ChatService.CompleteChat`](../AltusAiAssistantApi/Services/ChatService.cs#L76-L78) short-circuits:
- The system prompt embeds `session.FilesContent` verbatim.
- The `SearchInternalDocuments` tool is **not** registered ([ChatService.cs:197-204](../AltusAiAssistantApi/Services/ChatService.cs#L197-L204)) — so the index (including prior Manual uploads) is not searched for that session.

---

## 9. CuratedAnswer

User-authored corrections/refinements to prior assistant responses. Indexed so that future queries benefit from them.

### Fetch
No external source — authored in the chat UI:
- **Endpoint**: `POST /Chat/CreateCuratedMessage/{sessionId}` → [`ChatService.CreateCuratedMessage()`](../AltusAiAssistantApi/Services/ChatService.cs#L752).
- Takes `OriginalMessageId` (the assistant message being refined), `Text`, and optional `Title`.

### Index
- **Deduplication**: If a curated message already exists for the same parent, the old DB row is soft-deleted and its blob is removed first ([ChatService.cs:760-767](../AltusAiAssistantApi/Services/ChatService.cs#L760-L767)).
- **Title generation**: If no title is supplied, a short one is generated via a cheap `GPT-35-Turbo` summarization ([ChatService.cs:793-838](../AltusAiAssistantApi/Services/ChatService.cs#L793-L838)).
- **Blob path**: `{Constants.CuratedAnswersFolderName}/{title}_{yyyy-MM-dd_HH-mm-ss}.md` ([ChatService.cs:780](../AltusAiAssistantApi/Services/ChatService.cs#L780)). The folder is created on first use ([ChatService.cs:769-773](../AltusAiAssistantApi/Services/ChatService.cs#L769-L773)).
- **Metadata** ([ChatService.cs:783](../AltusAiAssistantApi/Services/ChatService.cs#L783)):
  - `sourceType = CuratedAnswer`
  - `date = DateTime.UtcNow`
- **DB record**: A `ChatMessage` with `ParentMessageId` linking back to the original assistant message and `CuratedMessageFileUrl` pointing at the blob ([ChatService.cs:786](../AltusAiAssistantApi/Services/ChatService.cs#L786)).
- **Indexer is triggered explicitly** after save (`await _indexingService.RunIndexer(userClient.ClientId);` at [ChatService.cs:790](../AltusAiAssistantApi/Services/ChatService.cs#L790)).

### Chat usage
- Retrieved like any other indexed document — **no preferential boosting** beyond the standard 1.5× freshness rule.
- `sourceTypes = 'CuratedAnswer'` ([ChatService.cs:1062-1063](../AltusAiAssistantApi/Services/ChatService.cs#L1062-L1063)).
- The system prompt does not give curated answers special treatment; their influence is purely through higher retrieval score from being tightly written and topical.

---

## Summary table

| Source | Fetch trigger | Cadence | Blob path | Unique metadata | Indexer triggered by | `sourceTypes` friendly name |
|---|---|---|---|---|---|---|
| **Confluence** | Timer (`AltusAiConfluenceSync`) | 15 min incr / 60 min deleted | `{SpaceKey}/{Title}.md` | externalUrl | Schedule (30m) | `Confluence` |
| **ExternalConfluence** | Same as Confluence | Same | Same | externalUrl | Schedule (30m) | `Confluence` (merged) |
| **Slack** | Timer (`AltusAiSlackSync`) | 10 min incr / daily deep | `Slack/{channel}/{chunk}.md` | slackChannelName, externalUrl | Schedule (30m) | `Slack` |
| **SlackFile** | Same as Slack (`SyncFiles`) | 10 min incr / daily deep | `Slack/{channel}/{yyyyMMdd}/{file}_{id}` | slackChannelName, author, externalUrl | Schedule (30m) | `Slack` (merged) |
| **Email** | Timer (`AltusAiMailSync`) | 10 min incr / HTTP full | `Email/{date}_({from})/{subject}/{subject}.html` | author, mailboxAddress | Schedule (30m) | `Email` |
| **EmailFile** | Same as Email (`SyncAttachments`) | 10 min incr / HTTP full | `Email/{date}_({from})/{subject}/{file}` | author, mailboxAddress | Schedule (30m) | `Email` (merged) |
| **Grain** | `POST /Integration/ImportGrain` via Zapier; driven by `GrainSynchronizer` poller (10 min) | On demand | `Grain/{title}_{date}.md` | externalUrl | Schedule (30m) | `Grain` |
| **Manual** | `POST /Admin/UploadBlobItems/{clientId}` | On demand | `{userPath}/{file}` | — | **Explicit** (`RunIndexer` call) | `Manual` |
| **CuratedAnswer** | `POST /Chat/CreateCuratedMessage/{sessionId}` | On demand | `CuratedAnswers/{title}_{timestamp}.md` | — | **Explicit** (`RunIndexer` call) | `CuratedAnswer` |

### Key cross-source observations

- **One index per client, not per source.** Source separation exists only as blob metadata and an OData filter on `sourceType`.
- **Three fetch archetypes.** Scheduled Azure Function (Confluence, Slack, Email), user-triggered REST (Manual, CuratedAnswer), and hybrid poller+webhook (Grain).
- **File-session chats skip the index entirely.** Manual uploads at the client level feed the index; files attached to a specific chat session do not — they flow into the system prompt as raw text.
- **The LLM sees a simplified source vocabulary.** `Slack`/`Email`/`Confluence` are each presented as one concept even though each expands to two underlying `DocumentSourceType` values.
- **Only Manual and CuratedAnswer kick the indexer synchronously.** Everything else waits up to 30 minutes for the schedule to tick.
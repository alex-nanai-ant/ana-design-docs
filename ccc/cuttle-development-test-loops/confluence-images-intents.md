# Confluence images intents (snapshot)

**Status:** Snapshot of Cuttle intents for reference during review. Not the source of truth, and not an approval record.

**Exported:** 2026-10-19 with `cuttle intents get <id>`. Descriptions and plans below are copied verbatim; only plan heading levels are shifted so they nest in this file. The live intents in Cuttle may change after this date.

**Approval state at export:** all four intents are `planned`; `approvedBy` and `signedOffAt` are empty; no work branch or PR exists.

**Related:** [Ralph test approach](ralph-test-approach.md) uses Stage A as its primary example. [Options design](design.md). [Content image indexing design](../content-image-indexing-iteration-1/design.md).

| Role | Title | Intent ID | Status | Created | Plan written | Dependency / order |
| --- | --- | --- | --- | --- | --- | --- |
| Parent | [Support images in content](#parent-support-images-in-content) | `c56ad8b5-2f1d-4d1b-8f58-33476dd9ae6e` | planned | 2026-10-15 | 2026-10-16 | Coordination record; children ship A → B → C |
| Stage A | [Stage A: Confluence images](#stage-a-confluence-images) | `39a0be03-276e-47ee-875f-38006b90c52c` | planned | 2026-10-16 | 2026-10-16 | First delivery stage. B and C reuse this infrastructure after its PR is integrated; do not implement their inputs here. |
| Stage B | [Stage B: Validated Confluence attachments](#stage-b-validated-confluence-attachments) | `87ca7bd4-4f0a-467b-ad5f-b101cf9d5549` | planned | 2026-10-16 | 2026-10-16 | Depends on Stage A (39a0be03-276e-47ee-875f-38006b90c52c) being integrated. Re-read A's final design/implementation before implementing B and preserve A's JPEG/PNG behavior. |
| Stage C | [Stage C: Standalone images](#stage-c-standalone-images) | `faf98799-33f5-4569-b404-0c95a19dd3b3` | planned | 2026-10-16 | 2026-10-16 | Third planned PR, after A and B. Reuse integrated A infrastructure. B is the preceding delivery milestone; do not couple standalone admission to PDF/Word internals. Re-read earlier stage outcomes before implementation. Stage B: 87ca7bd4-4f0a-467b-ad5f-b101cf9d5549. Stage A: 39a0be03-276e-47ee-875f-38006b90c52c. |

---

## Parent: Support images in content

| Field | Value |
| --- | --- |
| Intent ID | `c56ad8b5-2f1d-4d1b-8f58-33476dd9ae6e` |
| Status | planned |
| Parent | — |
| Priority | medium |
| Created | 2026-10-15T18:44:53.485Z |
| Updated | 2026-10-16T09:21:29.600Z |
| Plan written | 2026-10-16T09:21:29.597Z |
| Approved by | — |

### Description

Make images in ANA's knowledge content searchable and usable in grounded answers, with citations to their sources.

Build on API branch feature/content-image-indexing-poc and specs/content-image-indexing-iteration-1/design.md. Frontend base: main.

Reduced scope confirmed by the owner on 2026-10-16:
- Stage A: Confluence JPEG/PNG images only. Ingest image attachments, including images displayed in pages. Support image-based questions in both chat modes with citations opening the parent Confluence page, retaining correct sources after conversation reload. Validate the image-only Search path internally; standalone knowledge-source admission waits for C.
- Stage B: Confluence PDF/Word attachments only for formats and embedded-image extraction demonstrated by validation. Reuse A's transport, ownership and lifecycle; do not advertise unvalidated Office support.
- Stage C: standalone knowledge images from manual uploads, Slack files and ordinary non-inline email attachments, initially validated JPEG/PNG. Limit FE changes to the existing knowledge-upload flow.

Image previews, new admin/status UI, remote/macro renderers, broader formats, inline email images and image-aware file sessions are outside these stages unless separately agreed.

Historical Confluence coverage requires explicitly started, bounded backfill for selected clients, links and spaces/pages, with progress and resume checkpoints. Opt-in must not start a historical scan or reset Search. Ongoing reconciliation must respect enrolled scope.

Preserve client scoping, content rules, caching and numeric ingestion/cost controls. Use synthetic or explicitly non-sensitive pilot fixtures; real client content waits for storage/access-policy review. Validate the two-hour healthy-service target for updates and confirmed deletions within enrolled scope, including inventory detection and Search processing. Required lifecycle/citation/access checks ship with each stage.

Retain at least 80% correct-source retrieval and 70% correct answers on a fixed corpus of at least 20 answerable image questions, reporting both chat modes separately. Report abstentions without counting them as correct answers. Require correct citations for every answered image question and zero observed client/content-rule bypasses in planned tests. Move the full PDF/Word extraction corpus to Stage B.

Produce human-readable designs and review numeric budgets and storage/revocation policy before implementing each stage. Status: Draft - for human review. This scope reduction does not approve a design or authorize implementation.

Delivery is split into child intents, one shippable PR each, in A -> B -> C order:
- Stage A: Confluence images: 39a0be03-276e-47ee-875f-38006b90c52c
- Stage B: Validated Confluence attachments: 87ca7bd4-4f0a-467b-ad5f-b101cf9d5549
- Stage C: Standalone images: faf98799-33f5-4569-b404-0c95a19dd3b3
The parent is the coordination/scope record. Children remain planned; research and implementation were not started by this split.

### Plan

### Support images in ANA knowledge content

_Status: Draft - for human review. Scope refined with the owner on 2026-10-16. These intent choices do not approve stage designs or authorize implementation._

#### Overview

Build on the existing content-image indexing POC. Limit Stage A to Confluence JPEG/PNG image attachments, including images displayed in pages, using the existing Azure AI Search pipeline. Stage B adds only validated Confluence PDF/Word attachments. Stage C adds standalone knowledge images from manual uploads, Slack files and ordinary email attachments. Validate image-only source documents in Stage A because Confluence image attachments use that technical path; this does not admit standalone sources before Stage C. Preserve the existing description cache, client opt-in, retrieval filters, and separate indexing model deployment. Make source citations reliable across both chat modes before releasing the Confluence slice. Stage A delivers grounded text answers with citations that open the parent Confluence page. In-ANA image previews, new admin/status UI, and broader format/extraction work are outside these three delivery stages unless separately agreed.

##### Delivery ownership

The owner approved splitting this work on 2026-10-16. Each sub-intent owns one separately shippable PR, in A -> B -> C order. This parent retains the cross-stage scope/research ledger; execute stage work through the children rather than starting duplicate implementation here. Shared cleanup, security, evidence/citation and rollout checks ship with each affected child.

| Stage | Sub-intent | Intent ID |
| --- | --- | --- |
| A | Stage A: Confluence images | `39a0be03-276e-47ee-875f-38006b90c52c` |
| B | Stage B: Validated Confluence attachments | `87ca7bd4-4f0a-467b-ad5f-b101cf9d5549` |
| C | Stage C: Standalone images | `faf98799-33f5-4569-b404-0c95a19dd3b3` |

Children were created with --parent to inherit this intent's base. They are planned, with research stopped and automatic plan approval disabled. A must reconcile the prepared API baseline with feature/content-image-indexing-poc before implementation; inherited base metadata does not mean that the POC is present. Designs still require review.

Deliver in reviewable stages:

| Stage | Outcome | Release gate |
| --- | --- | --- |
| A: Confluence images | Confluence JPEG/PNG image attachments become searchable evidence for text answers in both chat modes, with parent-page citations | Image-only pipeline validation for these attachments, isolation, lifecycle, persisted citations and a bounded Confluence pilot; PDF/Word and standalone sources excluded |
| B: Validated Confluence attachments | Add Confluence PDF/Word attachments only for formats and embedded-image extraction demonstrated by validation | Reviewed attachment design; measured format coverage, source ownership, lifecycle and chat/citation acceptance; no new source connectors or preview UI |
| C: Standalone images | Admit validated standalone knowledge images from manual uploads, Slack files and ordinary non-inline email attachments | Reviewed source-admission design, bounded source replay, provenance, isolation, lifecycle and source-specific acceptance; start with JPEG/PNG |

This is research and an implementation proposal, not a statement that the feature is deployed or validated. No repository files were changed, builds run, cloud resources modified, or live client data inspected during research.

##### Scope choices confirmed by the owner (2026-10-16)

- **Stage A user outcome:** users can ask image-based questions in both chat modes and open the cited parent Confluence page. Image previews inside ANA are outside the reduced delivery scope. Persisted citations must still resolve to the correct source after conversation reload.
- **Stage boundaries (revised by the owner):** A = Confluence JPEG/PNG images only; B = validated Confluence PDF/Word attachments; C = standalone knowledge images. Stage A still validates JPEG/PNG as image-only Search source documents internally. PDF/Word validation and the document extraction corpus move to B; manual/Slack/mail image admission moves to C. Broader Office formats, remote images, macro renderers, inline email images, vector/PDF rendering, image-aware file sessions, image previews and admin/status UI are deferred outside this plan.
- **Historical content:** backfill must be explicitly started and bounded. Opt-in alone must not start a historical Confluence scan or reset a Search indexer. The backfill operation must declare its client, source link, selected spaces/pages, workload limits and resume checkpoint. Scheduled reconciliation must not silently expand this historical scope; the Stage A design must define and test the distinction between backfill enrollment and normal new/changed-page sync.
- **Pilot data:** use synthetic or explicitly non-sensitive fixtures. Real client content waits for the storage/access-policy review. This pilot choice does not approve public blob access or settle the final private-access architecture.
- **Freshness target:** validate that changed or confirmed-deleted attachments within the enrolled scope are reflected in Search within two hours while services are healthy. This is a proposed acceptance target, not an observed service guarantee. Include inventory detection and Search processing in the measured interval.
- **Pilot quality:** retain at least 80% retrieval and 70% correct answers on the fixed image-question corpus. Require correct source citations for every answered image question and zero client/content-rule bypasses. Report abstentions separately; they must not inflate the correctness score.
- **Review state:** these owner-confirmed intent choices still require a human-reviewed Stage A spec/design/plan before implementation. Ingestion/cost budgets, precise attachment selection, and storage/revocation policy remain open for that design. None of those open decisions is approval to ingest sensitive content.

##### Stage A acceptance and evidence

The owner selected these pilot boundaries on 2026-10-16. Future validation must record results; this interview did not run the pilot. The stage design must carry these criteria into its acceptance matrix.

| Area | Acceptance criterion | Proposed validation and evidence |
| --- | --- | --- |
| Data boundary | The initial pilot uses synthetic or explicitly non-sensitive fixtures. Real client content waits for storage/access-policy review. | Review the fixture manifest before ingestion. Document the access-policy decision and affected download/preview paths before later real-content use. |
| Freshness | Every tested update or confirmed deletion in the enrolled pilot scope is reflected in Search within 120 minutes under healthy services. The interval includes inventory detection and Search processing. | Use controlled source mutation times as the start, rather than starting the clock after discovery. For updates, verify the new revision is retrievable. For deletions, verify old attachment chunks are absent from new retrievals. Record source change, inventory observation, blob update/delete and Search observation times. Cover attachment changes without page edits. Log breaches and service failures separately; do not silently exclude them from the report. |
| Retrieval | At least 80% of a fixed corpus of at least 20 answerable image-only questions retrieves the correct permitted source. | Agree on questions, expected source occurrences and ground-truth answers before scoring. Cover the advertised Stage A formats, Confluence diagrams/screenshots and image-only pages. Report results separately for ordinary and agentic chat so a passing mode cannot hide a failing mode. Apply the meaningful-image marker and decorative-suppression checks to the JPEG/PNG pilot. Run the full iteration 1 document extraction corpus and its 90% marker/90% decorative-suppression gates in Stage B; PDF/Word coverage does not block A. |
| Correct answers | At least 70% of the fixed answerable-question corpus receives a reviewer-rated correct answer. | Score all corpus questions, including questions that receive an abstention. An abstention is reported separately and does not count as a correct answer to an answerable question. Report each chat mode separately. Keep unsupported/unanswerable negative cases in a separate fallback assessment. |
| Citations | Every answered image question has the correct parent-page citation for its image evidence, in both chat modes and after conversation reload. | Check page URL and source occurrence, including repeated searches, several image chunks and identical images in different documents. An incorrect or missing citation fails this gate independently of the answer-correctness percentage. No in-ANA image preview is required. |
| Isolation | Zero observed client/content-rule bypasses across the planned negative-access cases. | Cover cross-client IDs, nonmembers, excluded sources, Focus Mode, private paths and fabricated image markers. Do not describe this finite test result as proof that every possible access path is secure. |
| Bounded historical work | Backfill starts only by explicit operator action and stays within its declared client/link/space/page and workload scope. | Exercise cancel/resume, unchanged replay, opt-in without a backfill and reconciliation of enrolled pages. Verify successful unchanged work does not repeat uploads or model calls. |

For the freshness criterion, healthy services and pilot capacity must be defined in the Stage A design and logged during validation. The source currently configures Confluence page sync every 15 minutes in `charts/altus-ai-assistant/charts/altus-ai-confluence-sync/values.yaml` and Search every 30 minutes in `IndexingService.BuildIndexer`. These are inspected configuration values, not evidence of deployed schedules, completed runs or a two-hour guarantee. Initial historical backfill completion time is a separate estimate; the two-hour target applies to ongoing changes within enrolled scope.

A deletion is eligible for destructive reconciliation only after a complete, successful, ownership-scoped inventory confirms its absence. Authentication errors, loss of connector visibility, incomplete traversals and empty suspicious responses must follow the reviewed failure/revocation policy. The two-hour freshness target does not define permission-revocation latency or retroactive removal of saved answers; both remain explicit design questions.

##### Research baseline

All research used the four supplied prepared repository paths. The API worktree is on `ccc/intent/c56ad8b5` at `db2c2a2f544ff0ac65faff1e43affcf7dd705310`; it does **not** contain the requested POC or design. The requested local and remote-tracking refs both resolve to `feature/content-image-indexing-poc` at `74f900b25082ca6205888db375f9fba276025cd3`. The POC code, tests, and `specs/content-image-indexing-iteration-1/{design,testing}.md` were therefore read with `git show`/`git cat-file` from that prepared API repository, without checking out or copying files. That POC merges the prepared API commit and is the implementation baseline for this plan. Future implementation must start from the POC ref, not accidentally rebuild the feature from the prepared worktree HEAD.

Refinement check on 2026-10-16: the registered API checkout at `C:\\git\\ana\\altus-ai-assistant` is clean, on `feature/content-image-indexing-poc` at `74f900b25082ca6205888db375f9fba276025cd3`. The Cuttle intent still reports its separate prepared API worktree at `db2c2a2f544ff0ac65faff1e43affcf7dd705310`; that metadata is not the registered checkout's branch. No checkout was switched and no worktree was rebased in this interview. The registered FE checkout has an existing edit to `public/runtime-config.js`; Terragrunt has an untracked `scripts/set-ana-content-images-skill-code.ps1`. Neither file was opened or modified. These are separate from the prepared worktree baseline below.

The FE prepared HEAD is `99ebaa5cd366e38ba304abd16422a975b7065e90`, matching its local `main`. GitOps HEAD is `aeb5a7818cda0394c425dff9e4ecaf3c0adb7bcc`; Terragrunt HEAD is `67b3be249de4f188242a4137023fc06d3c0c269b`. Configuration in Git is evidence of intended configuration, not evidence of the running environment.

#### Reasoning

##### Existing capabilities and validation gaps

| Area | Present in the POC | Validation gap or remaining work |
| --- | --- | --- |
| PDF/Word/Office extraction | Opted-in indexers use `generateNormalizedImages`; `DescribeImages` produces markers; Merge inserts them before split/embedding | Actual extraction coverage, ordering, scanned PDFs, older Word image formats, and normalized-byte stability are not established by unit tests |
| Search and answers | Image descriptions become ordinary searchable chunk text; `ViewContentImage` makes a separate vision call and returns text | Image-only documents and real end-to-end citation quality remain unverified |
| Model access | Tool is available in ordinary chat and agentic synthesis, only for opted-in non-file sessions | Multiple searches, probe evidence, chunk deduplication, duplicate image IDs, and persisted citations need integration coverage |
| Client scoping | Separate index/container per client; image cache partitioned by client; tool resolves only IDs registered from same-client retrieved citations | Preview endpoints must enforce membership and applicable content rules themselves; existing blob URLs are public-capable |
| Caching and cost | SHA-256-derived image ID; cache keyed by client/image with prompt version; 100-pixel size threshold; 1,200-character description cap; decorative suppression | Stable normalized bytes must be measured; concurrent misses can call the model more than once; no aggregate cross-client rate limit |
| Failure handling | Record-level skill errors; transient failure limit 3; throttling backoff; unprocessable images cached; configuration failures do not consume the image failure budget | Cloud retry behavior and batch isolation need controlled fault injection; retry worker defaults to disabled and must actually be enabled |
| Storage lifecycle | JPEG copies under `{document}.images/{imageId}.jpg`, `AzureSearch_Skip=true`, reserved-path guards; file deletion removes copies; moves remove old copies | Removed/replaced embedded images leave stale copies; copies reappear at a moved document's new path only after indexing |
| Opt-in | `Client.ContentImagesEnabled`, additive migration, admin PUT endpoint; enable/disable rewrites Search configuration without resetting | Client read DTO omits the flag; no FE toggle; Search writes plus SQL flag are not an atomic transaction |
| Confluence | Storage HTML becomes Markdown with page ID, ancestors, source type, date and external URL | No attachment listing/download; images/macros do not become image evidence; pages below 50 Markdown characters are skipped |
| Other inputs | Existing supported documents from manual, Slack and mail flow through the same blob indexer | FE knowledge uploads and `AllowedIndexFileExtensions` exclude image formats; chat-image attachments are a separate existing feature |
| UI | Citation text and source links already render; normal file previews exist | No content-image status, description lookup, image-file preview, or image-marker affordance |
| Deployment | POC adds skill-host settings/secrets and telemetry; sandbox GitOps config names a description deployment and a 60-second gateway timeout | Model existence/quota, applied migration, Search resources, actual skill image, and successful test runs were not inspected |

Existing image tests cover describer cache/isolation, filtering and failures; skill source-account guards and record errors; Search configuration shape; migration/opt-in behavior; reserved paths and copy deletion; and mocked tool behavior. They are useful foundations, not proof of actual Azure document cracking, retrieval quality, authorization across HTTP routes, or a successful deployment. The Confluence tests primarily exercise job orchestration through fake services; the real attachment transport and sync lifecycle need additional seams and tests.

##### Keep the shared indexing path

Use option 1 from iteration 1's design: ingest Confluence attachments as source files and let Search call the existing describer. Do not put model calls into Confluence sync for the first release. This retains one cache, failure policy, prompt, storage convention, and rate-control path. Stage A ingests only JPEG/PNG image attachments. Stage B reuses this path for Confluence PDF/Word attachments after validating their extraction behavior.

The expected flow is:

```text
Confluence page + complete attachment inventory
  -> bounded, authenticated attachment downloads
  -> source blobs carrying page/link provenance
  -> existing normalized-image / describe / merge / embed pipeline
  -> filtered search results and stable citation references
  -> grounded answer, optionally using ViewContentImage
```

Each attachment is initially a separate search document. Keep its filename and parent-page URL; inherit page ID and ancestors so Confluence subtree exclusion and Focus Mode continue to work. The cost is weaker surrounding-page context. Evaluate it with page-plus-diagram questions before considering in-page descriptions. Include an explicit unresolved-image inventory so downloading attachments is not mistaken for complete coverage of remote images or diagram macros.

Microsoft documents standalone and embedded image enrichment and JPEG normalized-image output, which supports this architecture; it does not establish that ANA's current merge/split/projection configuration handles an otherwise empty image document correctly. That remains a release gate. [Azure AI Search image processing](https://learn.microsoft.com/en-us/azure/search/cognitive-search-concept-image-scenarios).

For Confluence Cloud, use a typed attachment reader around the page attachment endpoint, cursor pagination, attachment versions and download links. The documented API requires visibility of the page and its space. Test the actual linked credentials and download redirects rather than assuming the existing JSON `GetByUrl` method can download binary content. [Atlassian attachment API](https://developer.atlassian.com/cloud/confluence/rest/v2/api-group-attachment/), [Atlassian pagination](https://developer.atlassian.com/cloud/confluence/rest/v2/intro/).

##### Preserve ownership and evidence, not just image bytes

The existing image ID identifies normalized bytes, not a source occurrence. Retain client + source document + image ID when granting access or rendering a citation. Identical images in two documents may share descriptions, but they must keep distinct source ownership. The first permitted occurrence must not authorize another document with different rules.

There are concrete integration hazards in the current code:

- `AdminService.BuildConfluenceHierarchy` places one node per `confluencePageId`. Giving several attachments the same page ID without changing tree construction can hide attachments or cause an attachment to occupy the page node.
- `ChatToolProvider.SearchInternalDocuments` registers image IDs from every search but overwrites `LastSearchContext`. `CompleteChatService` persists only that last context. An image read from an earlier search can therefore lack its source in the saved answer.
- Agentic `JoinExecutor` keeps the first citation per file URL. Other chunks, including the only image marker or an end marker, can be discarded from synthesis evidence.
- `SyncPageTableEntity` keys are space key + page ID; client is only a property/query filter. Identical source spaces linked to two clients can overwrite each other's rows, and different sites can collide. New attachment state must not repeat this pattern.
- Confluence page state is advanced before processing, and failure rows persist the current modified time/version. A later unchanged-page check can skip a failed operation. Attachment success must have a separate, durable checkpoint.

Use stable attachment IDs and link/site identity for source state and paths. Recommended attachment layout: `Confluence - {space}/attachments/{linkId}/{pageId}/{attachmentId}/{safeFilename}`. Retain the current page Markdown layout for compatibility. Attachment paths must never use the reserved `.images` suffix; `.images` remains exclusively for derived JPEG copies. Preserve a distinct logical page identity even when the title-derived Markdown path changes.

Use existing Confluence subtree metadata for parent-page rules. Before adding fields, test legacy document-target rules as well: `BlobDocument` matches only Search `parent_id`, so a rule against a page blob does not inherently cover separately indexed attachments. If those rules are supported for page ownership, add a nullable, filterable `contentOwnerParentId` provenance field and project it to attachment chunks, with matching filter tests. Ordinary documents keep their current behavior. Verify `ParentIdHelper` against real Search values before deriving owner IDs from it.

##### Define the security and rollout boundary honestly

Retain ANA's current client membership and content-rule semantics. The connector authenticates as a configured Confluence account; the inspected pipeline does not implement per-ANA-user Confluence restriction synchronization. Attachment support must not imply that such synchronization exists. Restriction changes, loss of connector access, and cached historical answers require an explicit retention/revocation policy in the stage A design.

`BlobStorageService.CreateContainer` uses `PublicAccessType.Blob`, and Terragrunt permits public nested items. A SAS URL does not make a publicly readable blob private. Protect new API operations with authorization, and settle the storage policy before ingesting sensitive additional images. A switch to private containers requires compatible document previews/downloads and whole-document reads; it cannot safely be an isolated one-line storage change. The researched `GetAllClients` anonymous exception and connector-field exposure remain a separate existing issue; new settings/toggle APIs are deferred, so this scope does not expand that route.

Keep opt-in free of implicit full reindexing or Confluence rescans. Disable stops new image extraction/tool use after successful Search reconfiguration; it does not remove already indexed descriptions, stored copies, or caches. Any purge is a separate, explicitly scoped operation. Per-indexer parallelism 2 is not a deployment-wide spending cap when many clients index simultaneously.

#### Affected Files

Paths below are repository-relative. Existing POC files refer to the requested feature ref; proposed new files are marked **new**. Final filenames for review documents and small service abstractions should follow the repository's Spec Kit workflow.

| Repository / files | Intended change |
| --- | --- |
| API: `specs/content-image-indexing-iteration-1/design.md`, `testing.md` | Preserve iteration 1 decisions; record measured outcomes and corrected assumptions |
| API: `specs/content-image-indexing-iteration-2/{spec,design,plan,tasks,testing}.md`, `contracts/` (**new**) | Canonical Confluence scope, provenance, security, state, API contracts and acceptance gates |
| API: `AltusAiShared/Confluence/{ConfluenceService,ConfluenceGetContentResponse,ConfluenceSyncService,SyncPageTableEntity,IConfluenceSyncService}.cs` | Typed attachment access, reference inventory, sequencing and scoped success checkpoints |
| API: `AltusAiShared/Confluence/ConfluenceAttachment*.cs`, attachment sync/state interfaces (**new**) | Attachment DTOs, identity/path rules, inventory reconciliation and transport abstraction |
| API: `AltusAiConfluenceSync/{ConfluenceLinkDbEntity,ConfluenceLinkRepository,Program}.cs`, `AzureTableConfluencePageStateStore.cs`, attachment state store (**new**) | Load opt-in/link identity, inject dependencies, isolate durable state |
| API: `AltusAiConfluenceSync/Jobs/Confluence{Incremental,Ancestors,DeletedPages}SyncJob.cs`, `IConfluenceDocumentStore.cs`, `BlobConfluenceDocumentStore.cs` | Attachment refresh/reconciliation, metadata propagation and deletion under coordinated client locks |
| API: `AltusAiShared/{BlobMetadata,Constants,BlobStorageService,ParentIdHelper}.cs` | Provenance metadata, verified source formats, content types, bounded IO and lifecycle operations |
| API: `AltusAiShared/ContentImages/{ContentImageOptions,ContentImageDescriber,ContentImageCache,ContentImageStorage,ContentImageTelemetry,ThrottlingContentImageModel}.cs` | Reuse existing behavior; targeted safeguards, recovery and cleanup hooks only as justified |
| API: `AltusAiIndexerSkills/{DescribeImages,ContentImageModel,Program}.cs`, `AltusAiAssistantApi/Services/IndexingService.cs` | Validate standalone documents; minimal merge fallback if needed; provenance projection and configuration reconciliation |
| API: `AltusAiAssistantApi/Services/Maf/{ChatToolProvider,AgenticWorkflowTypes,KnowledgeRetriever}.cs`, `Executors/{SearchExecutor,JoinExecutor,SynthesizerExecutor}.cs` | Stable evidence/citation registry, preserve marker-bearing chunks, image-aware whole-document handling |
| API: `AltusAiAssistantApi/Services/ContentImages/{ContentImageRun,ContentImagePathGuard}.cs`, `CompleteChatService.cs`, `Maf/ChatTurnStore.cs` | Source-bound tool access and persistence of the evidence actually used |
| API: `AltusAiAssistantApi/Services/{AdminService,ContentRuleService}.cs`, relevant provenance/tree DTOs | Stage A minimum attachment/page ownership, tree identity and source-rule corrections. New settings, status and preview APIs are deferred |
| API: `AltusAiShared/{Slack/SlackSyncService,Mail/MailSyncService}.cs` | Stage C standalone image admission and bounded source-specific replay; inline mail remains deferred |
| API: `tests/AltusAi{Shared,IndexerSkills,AssistantApi,ConfluenceSync,SlackSync,MailSync}.Tests/` | Extend existing xUnit/Moq tests; add transport/sync/provenance integration coverage |
| FE: `src/libs/constants/supported-files-types.ts`, upload validation/helpers and `src/libs/shared/UploadDialog.tsx` | Stage C only: admit validated standalone knowledge images using the existing upload flow |
| API charts: `charts/altus-ai-assistant/charts/altus-ai-{assistant-api,indexer-skills,confluence-sync}/values.yaml` and relevant templates | Required configuration, limited secrets, timeouts, resources and reconciliation scope |
| GitOps: `apps/payloads/ana-qa-sandbox.yaml`, promoted environment payloads, `config/altus-k8s/ana-qa-sandbox/secrets/externalsecret.yaml` and environment equivalents | Pilot wiring first; controlled promotion after validation; no secret literals in new settings |
| Terragrunt: `modules/assistant-services/{main,variables,outputs}.tf`, applicable environment `env.hcl` and `assistant-services/terragrunt.hcl` | Optional separately budgeted description deployment; reviewed storage policy where needed |

The existing `20261009102228_Add_Client_ContentImagesEnabled` migration is reused. Add a new additive SQL migration only if the reviewed contract genuinely requires new SQL fields; attachment inventory can use the existing Azure Table storage system. Do not introduce a new service or a second image-description pipeline.

#### Implementation Steps

1. Review the Confluence ingestion design and confirm the POC baseline <!-- step:c0d0be -->
   - Start implementation from API `feature/content-image-indexing-poc` at the researched revision and FE `main`. Recheck differences if either branch advanced.
   - Produce human-readable stage A `spec.md`, `design.md`, `plan.md`, `tasks.md`, contracts and an acceptance matrix under `specs/content-image-indexing-iteration-2/`. Include a data-flow diagram, example attachment paths/metadata, state transitions, source-link examples, security boundaries, rollback and cost estimates. Obtain review before implementing stage A, as required by both the intent and API constitution.
   - Choose attachment-first indexing explicitly. Stage A accepts only Confluence JPEG/PNG image attachments. PDF/Word attachments move to Stage B after validation; standalone source admission moves to C. Other formats and renderers remain deferred. Inventory attachment-backed `ac:image`/`ri:attachment`, standard image elements, remote images, cross-page references and diagram macros; distinguish supported downloads from unresolved forms.
   - Define whether attachment ingestion is gated by `ContentImagesEnabled`; recommended initial behavior is opt-in-only attachment ingestion, leaving existing text-page sync unchanged. Disabling preserves existing content and does not purge it.
   - Apply the owner-confirmed Stage A boundary: grounded text answers and parent-page citations in both chat modes; in-ANA image previews and new admin/status controls are outside the reduced delivery scope. Verify citation source identity after conversation reload, including page/image-only cases.
   - Design an explicitly started, bounded historical Confluence backfill with client/link/space/page scope, progress, cancellation, restart/resume and workload limits. Keep this separate from an Azure Search reset and from opt-in. Define how scheduled reconciliation respects the enrolled scope so it cannot silently start a historical backfill.
   - Use synthetic or explicitly non-sensitive pilot fixtures; real client content waits for storage/access-policy review. Resolve public-blob handling and connector restriction/revocation semantics before sensitive rollout. Keep the confirmed two-hour freshness target and 80% retrieval/70% correct-answer pilot gates in the acceptance matrix. Review numeric ingestion/cost budgets and reconciliation capacity rather than leaving them implicit in code.

2. Validate Confluence image attachments through the existing image-only pipeline <!-- step:f7028e -->
   - Extend `DescribeImagesTests` and Search configuration tests with JPEG/PNG sources and empty document text. Add a real-Azure acceptance procedure using a disposable test client; local mocks cannot prove document cracking.
   - Verify normalized output, merge with empty/missing source text or offsets, nonempty chunks/embeddings, image-only semantic/keyword retrieval, source URL/date/type, marker splitting, and `ViewContentImage` in both chat modes.
   - Keep normalized JPEG copies at `photo.png.images/{id}.jpg` for compatibility. Do not optimize away the second copy until access and source-format behavior justify a separate reviewed change.
   - If empty-content merge fails, add the smallest image-only text fallback before splitting. Preserve the ordinary PDF/Office and opted-out paths; do not replace the pipeline or immediately add sync-time model calls.
   - Advertise only Confluence JPEG/PNG image support in Stage A. Test rejected/deferred attachment types without admitting PDF/Word, other Office formats, manual uploads, Slack images or mail images. Keep regression checks for existing POC document behavior; new document extraction qualification belongs to Stage B.
   - Use a Stage A Confluence JPEG/PNG corpus with at least 20 answerable image questions. Retain the 80% retrieval and 70% correct-answer pilot gates and applicable meaningful/decorative image checks. Move the full iteration 1 document extraction corpus (10 documents, 30 meaningful images, two Word files and a 10-page scan) and its 90% marker/90% decorative-suppression gates to Stage B. Score retrieval and answer correctness separately for each chat mode on a fixed corpus; abstentions on answerable questions remain in the denominator and are reported separately. Require correct source citations for every answered image question, including after conversation reload.
   - Reprocess unchanged bytes and require cache hits without new model calls or copy names. Measure actual dimensions, reasoning settings, latency and peak skill memory; reproduce transient, unprocessable and configuration errors with fault injection. Record actual results, not assumed passes.

3. Define scoped attachment identities and durable sync state <!-- step:07e1a3 -->
   - Add typed attachment records and path construction with client, normalized source site/link, page ID, attachment ID, version, filename, media type, size, source URL and successful sync revision. Sanitize names and reserve `.images`; distinguish duplicate filenames by attachment ID.
   - Partition new attachment state by client and stable source/link identity, with page/attachment IDs in row identity. Preserve source site identity in metadata and any cache/tree keys; identical IDs on two sites must remain separate.
   - Correct the existing page-state collision risk with an additive scoped-key format or new table. Use a documented compatibility reader/migration for legacy rows that verifies ownership; never adopt or delete another client's row. Keep old state available for rollback until migration is verified.
   - Separate last-attempt/error fields from last-successful page and attachment versions. A failed download/list/upload must remain eligible for retry even if the page timestamp is unchanged. Reuse storage interfaces and inject state/transport dependencies into the shared sync service so real sync behavior is testable.
   - Carry page provenance via `BlobMetadata.Create`: retain `Confluence`/`ExternalConfluence`, `confluencePageId`, ancestor IDs/names, date, and a canonical page URL. Add attachment/link/version metadata without adding enum values unnecessarily.

4. Add secure Confluence attachment listing and downloads <!-- step:3fd31e -->
   - Extend `ConfluenceService` or a small typed attachment reader with current-page attachment enumeration and streamed binary download. Follow every page of results; do not use the existing string-deserialization helper for bytes.
   - Use request-specific authorization for the selected link instead of mutating shared `DefaultRequestHeaders`. Validate pagination origins and binary download redirects; never forward Basic credentials to another host. Allow documented trusted download hosts explicitly and do not fetch arbitrary page-provided URLs with connector credentials.
   - Enforce configured per-file bytes, page/run attachment and byte budgets, request deadlines, cancellation and bounded concurrency. Verify actual bytes/media type rather than trusting the extension or an HTML login response.
   - Apply bounded Retry-After-aware backoff to 429/temporary errors. Distinguish access denial, missing attachment, unsupported type, oversized input and incomplete traversal in sync results; none is an instruction to delete the page's inventory.
   - Add fake-HTTP tests for pagination, relative URLs, redirects, multiple credentials/sites, cancellation, malformed payloads, login responses, 401/403/404/429 and maximum-size streams. Verify secrets and signed URLs do not appear in logs.

5. Ingest Confluence attachments with page provenance <!-- step:09f4f3 -->
   - Load the client's opt-in in `ConfluenceLinkRepository`/`ConfluenceLinkDbEntity` and pass explicit link identity into the sync operation. Do not add model credentials to the Confluence worker.
   - Enumerate attachments and ingest only validated JPEG/PNG images before applying the 50-character Markdown test. Image-only pages must retain their attachment evidence even if no page Markdown is uploaded; create a synthetic page tree node where needed.
   - Upload changed/new source attachments at deterministic paths with valid content types and page provenance. Compare successful version/checksum state before uploading so unchanged syncs do not change blob timestamps and trigger unnecessary extraction/embedding.
   - Keep derived copies exclusively under `ContentImageStorage`; only they carry `AzureSearch_Skip`. Source attachments must be indexable, subject to the client's opt-in and existing filters.
   - Preserve useful page links/captions in Markdown and record which references resolve to which attachment IDs. In stage A, page-local image elements referencing downloaded attachments are covered; arbitrary external URLs and unrendered macros are reported as coverage gaps.
   - Add shared-service tests for meaningful/short/image-only pages, attachment-only pages, repeated references, identical images across files, duplicate filenames, same IDs across clients/sites, and partial success. Maintain existing Grain-link and table conversion behavior.

6. Reconcile attachment changes and removals safely <!-- step:ce20c2 -->
   - Do not rely on page `lastmodified` alone. Add a bounded attachment inventory sweep for opted-in links within the existing Confluence job host, independent of unchanged-page skipping. Persist a continuation/checkpoint so large spaces progress within run budgets. Design for the owner-confirmed target of no more than two hours from a source attachment update or confirmed deletion to the corresponding Search result change within enrolled scope under healthy services. Budget that interval across inventory traversal, sync and Search processing; do not start its clock only after discovery. Record actual lag and capacity limits in the pilot.
   - Compare complete remote inventories against successfully owned local attachment state. Download new versions, update renamed paths/provenance, and remove confirmed deleted attachments and their derived copies. Use complete successful traversals as the prerequisite for deletion.
   - Extend deleted-page and unlink/client-cleanup paths to remove owned attachment prefixes and state. Delete blobs before retiring state, or persist a tombstone, so a crash cannot permanently orphan attachments. Respect Search's soft-delete detection window and verify projected chunks disappear.
   - Propagate ancestor moves and page URL/title changes to every attachment's metadata. Keep stable attachment paths across page title changes; update any owner-page path mapping needed by legacy rules.
   - Coordinate incremental, ancestors, deletion and attachment reconciliation with the same client/link locking boundary; existing deletion/ancestor jobs have different global locks and must not race new attachment work.
   - Test replacement without page edits, failed enumeration, empty results, lost permissions, rename, ancestor move, unlink, interrupted deletion and replay. Permission revocation must follow the reviewed policy, not be treated as an empty successful inventory.

7. Preserve attachment tree ownership (Stage A) <!-- step:ef0f80 -->
   - Separate page identity from attachment children and derived `.images` folders; never deduplicate attachments by page ID. Support synthetic page nodes when image-only pages have no Markdown blob.
   - Reuse Confluence subtree metadata for exclusions/Focus Mode. Where legacy blob-document ownership requires it, add the reviewed `contentOwnerParentId` projection/filter support before enabling attachment queries. Apply schema changes before queries reference new fields.
   - Test several attachments, duplicate page IDs across links/sites, ancestor moves and legacy page rules. Preserve `AzureSearch_Skip` and reserved-path guards. Restrict changes to ownership/correctness; no new image description/status/preview contract or frontend tree controls.

8. Preserve image evidence and citations across both chat modes <!-- step:5bf311 -->
   - Introduce a per-run evidence registry or equivalent stable reference mapping used by search output, image authorization and final persistence. Ordinary chat must retain sources used by earlier searches instead of persisting only `LastSearchContext`.
   - Assign stable `[docN]` references across repeated searches; preserve compatibility with `ChatMessageContext`/`MessageContextItemDto`. For agentic synthesis, merge or preserve distinct chunks from the same document rather than dropping all but its first chunk in `JoinExecutor`.
   - Make probe registration deliberate: probe-only IDs must not become usable uncitable evidence in the answer. Promote the source into synthesis/persisted evidence before image viewing, or keep probe IDs unavailable.
   - Preserve the image-to-source occurrence map and source citation when returning tool text. Reject unknown IDs and mismatched clients; never accept a model-supplied storage URL. Cover identical IDs in multiple permitted documents and concurrent searches without changing document ownership.
   - Keep retrieved image markers when whole-document raw text is added; avoid trying to text-decode standalone images. File sessions remain a documented separate limitation until an image-aware design is reviewed.
   - Test an image from search A followed by search B, several image chunks in one document, split end markers, source exclusions and Focus Mode, prompt-injection text, missing copies, timeouts and later-turn re-retrieval. Reload saved conversations and verify each image-based claim opens the correct source page/file.

9. Qualify and review the Confluence pilot <!-- step:36f6d8 -->
   - Use a disposable opted-in client and an opted-out control with synthetic or explicitly non-sensitive fixtures. Real client content waits for storage/access-policy review. Include internal/external Confluence links, PNG/JPEG diagrams, screenshots, image-only pages, paginated inventories, attachment version changes, deleted pages and ancestor moves.
   - Start the historical pilot backfill explicitly for selected client/link/space/page scope. Stop and resume it at configured limits without repeating successful unchanged uploads or model calls. Verify opt-in alone starts neither a historical scan nor a Search reset; record which historical pages are outside the selected scope.
   - Verify text answers cite the correct parent Confluence page in both chat modes and after conversation reload. PDF/Word attachment qualification, standalone source admission and preview UI must not block this milestone.
   - Require no client/rule bypass in retrieval or viewing; unchanged inventories must cause no new downloads/uploads/model calls; derived copies must never become Search documents. Test legacy page-document rules as well as Confluence subtree rules.
   - Compare page-plus-image questions against the attachment-first tradeoff. Document incorrect answers and missing page context with examples before choosing a more complex design.
   - Record the fixed question corpus and ground truth before scoring. Require at least 80% correct-source retrieval and 70% reviewer-rated correct answers separately for each chat mode. Report abstentions without removing answerable questions from the denominator. Require correct citations for every answered image question and zero observed client/content-rule bypasses in the planned access tests.
   - Test controlled updates and confirmed deletions against the 120-minute end-to-end target, including changes without page edits. Record healthy-service conditions, source/discovery/blob/Search timestamps and all breaches. Keep initial backfill duration and permission-revocation behavior distinct from this target.
   - Capture coverage counts, unresolved image forms, retrieval/citation correctness, cost by client, retry exhaustion, memory and reconciliation delay. Review the results before moving beyond the pilot; add targeted recovery commands for failed documents rather than resetting every client.

10. Review and validate Confluence document attachments (Stage B) <!-- step:67f8c4 -->
   - Produce a human-readable Stage B design and acceptance matrix before implementation. Limit the scope to PDF/Word attachments from the existing Confluence connector; reuse Stage A transport, scoped state, provenance, backfill and reconciliation.
   - Run the iteration 1 extraction corpus: at least 10 documents, 30 meaningful images, two Word files and a 10-page scan. Require 90% meaningful-image markers and 90% decorative suppression; distinguish document admission from actual embedded-image extraction. Advertise only individual formats/extraction behaviors demonstrated by validation.
   - Validate ordering, scanned documents, normalized-byte stability, merge/split behavior, cache reuse, model cost and retry isolation. Preserve the JPEG/PNG path and existing text-document behavior.
   - Enable validated attachment types only within the reviewed Confluence allowlist and bounded operator-selected scope. Test update/delete/rename/move/replay and the two-hour enrolled-scope freshness target; no automatic historical scan.
   - Report retrieval and answer correctness separately for both chat modes using fixed questions and ground truth, retaining at least 80% correct-source retrieval, 70% correct answers, correct parent-page citations for every answered image question (including reload) and zero observed client/content-rule bypasses. Keep abstentions in the correctness denominator.
   - No standalone uploads, Slack/mail admission, new settings/status/preview APIs or preview UI in this stage.

11. Review standalone image source admission (Stage C) <!-- step:6f88fa -->
   - Produce a human-readable Stage C design/contracts and focused FE upload tasks. Limit new source coverage to standalone knowledge images from manual uploads, Slack files and ordinary non-inline email attachments, initially JPEG/PNG validated in A.
   - Define per-source provenance, MIME/signature/size limits, client opt-in, source/content-rule checks, lifecycle and explicit bounded replay/backfill. Reuse the existing image describer/cache and source citation behavior; chat-message image attachments remain a separate existing feature.
   - Keep changes to the existing knowledge-upload UI limited to admitting the validated image formats and useful validation errors. New admin toggles, status UI and in-ANA image previews remain deferred.
   - Keep a deferred coverage ledger for remote Confluence images/macros, cross-page forms, inline mail/CID images, broader Office/raster formats, selective PDF/vector rendering and image-aware file sessions; do not implement those extensions in these stages.

12. Admit standalone images from the agreed knowledge sources (Stage C) <!-- step:b5ca4b -->
   - Extend manual knowledge upload validation, the FE upload allowlist and `Constants.AllowedIndexFileExtensions` only with agreed, validated JPEG/PNG inputs. Do not treat chat-file support as proof of Search support or expand Office allowlists.
   - Apply bounded size and MIME/signature validation to manual, Slack and mail inputs. Preserve Slack channel/file URLs and email attachment provenance. Skip/report inline signatures and CID images explicitly.
   - Add explicit scoped source replay/backfill; allowlist changes alone do not revisit Slack/mail items behind existing checkpoints. Enabling image processing must not initiate broad historical work.
   - Validate upload/import through indexing, retrieval, correct source citations and reload for each source, plus opted-out behavior, content-rule isolation, update/delete, rejected formats and unchanged replay/cache reuse. Report both chat modes separately and measure quality/cost before rollout.

13. Add only stage-required cleanup and operational recovery <!-- step:a37a98 -->
   - Deliver cleanup required for each stage's accepted update/deletion behavior with that stage. Reconcile obsolete source/image occurrences only after a completed owned revision or complete inventory, respecting active runs, source moves and reviewed retention/revocation policy. Retain shared description rows when an individual document is deleted.
   - Keep caches client-isolated and versioned. Use targeted recovery for failed attachment versions and exhausted image work; ordinary deployment must not invalidate all clients or reset Search.
   - Preserve low per-request parallelism and the dedicated description deployment. Define numeric run/cost budgets before rollout; add deployment-wide concurrency limits only if measured quota requires them.
   - Test cleanup isolation, crash replay, unchanged reindexing and retained references. Broad historical orphan sweeps, new renderers and extraction optimization projects are deferred.

14. Promote each stage with measured deployment and rollback controls <!-- step:da174e -->
   - Apply these release checks to A, then B, then C. Start with a disposable client using synthetic or explicitly non-sensitive fixtures and unique containers/indexes. Real client content waits for storage/access-policy review; QA namespace/SQL separation alone does not isolate shared Search/storage.
   - Verify the dedicated description deployment, limited skill-host secrets, migration, Search configuration, retry worker, request/resource limits, telemetry and numeric ingestion/reconciliation budgets. Prepare reviewed Helm/GitOps changes and optional environment-specific deployment resources only as required; no secret literals.
   - Smoke-test both chat modes for opted-out clients, then opt-in, partial configuration failure/retry, disable, changed content and explicitly selected backfill. Reconcile Search resources and SQL flags after partial failure; emergency stop is complete only when Search image processing is disabled.
   - Run affected .NET tests for the stage's image/indexer/API/Confluence paths. Run Slack/mail tests and FE build/browser upload checks in C when those paths change; render/lint charts and validate/plan infrastructure only where changed. Deploy only through the reviewed stage rollout.
   - Record the stage's fixed-corpus results, source/citation correctness, isolation tests, freshness, cost, unresolved forms and operator recovery/rollback instructions. Promote API/contracts before dependent FE and one client at a time. Disable and historical-evidence purge remain separate operations.

#### Edge Cases & Risks

| Risk | Required treatment |
| --- | --- |
| Prepared API baseline lacks the POC | Rebase implementation on the named feature ref; retain reviewed POC behavior and migration |
| Standalone sources produce no chunks | Prove empty-text merge/split/projection in real Search before shipping Confluence ingestion |
| Re-index normalization changes image bytes | Block broad backfill, investigate determinism; review a replacement identity scheme with collision/version tests rather than casually removing content hashing |
| Shared model cache has concurrent misses | Existing duplicate-call behavior is accepted in iteration 1; measure it before adding locks, and preserve client isolation |
| Page and attachment identity collisions | Scope durable state and tree keys to client/site/link; stable attachment IDs avoid filename/title collisions |
| Failed sync appears current | Persist successful revisions separately from attempts; retry failures independently of page timestamps |
| Attachments change without page modifications | Independent bounded inventory reconciliation; validate update/confirmed-deletion Search visibility within two hours in enrolled scope under healthy services |
| Partial listing or access denial causes deletion | No destructive diff from incomplete inventories; explicit permission-revocation handling and tombstone/retry sequencing |
| Hierarchy move changes visibility | Patch attachment ancestor metadata and retest source filters; do not infer access from the blob path alone |
| Earlier search image lacks a persisted source | Stable per-run citation registry, chunk preservation, and conversation reload tests |
| Markers are fabricated or duplicated | Treat markers as untrusted source text; resolve only same-client authorized source occurrences with existing copies; no arbitrary model paths |
| Public source/copy URLs bypass application rules | Existing architectural gap; settle and validate a compatible private-access policy before ingesting sensitive new material |
| Connector account sees more than an ANA user | Preserve ANA client/content rules and state the source-ACL limitation; no claim of per-user Confluence permission mirroring |
| Attachment download leaks credentials or fetches internal services | Validate URL/redirect origins, use request-local auth, strip credentials on external redirects and bound remote fetches |
| Scans, signatures or large attachment collections increase spend | Opt-in, size/dimension/type filters, download/run budgets, dedicated deployment, low concurrency and measured backfill |
| Disabling leaves existing descriptions searchable | Document the existing behavior; design separate purge/reindex workflows only when requested and scoped |
| Derived-copy metadata loses the skip flag | All metadata writes preserve `AzureSearch_Skip`; audit and test copies never becoming Search documents |
| Copies or citations outlive source changes | Source-version checks, completed-inventory cleanup, grace periods and explicit historical-answer behavior; preview work remains deferred |
| Search schema/filter rollout is out of order | Add optional fields first, update mappings/projections, backfill targeted attachments, then enable filters that reference them |
| Raw-file and Deep Dive behavior omits images | Preserve retrieved image evidence in standard/agentic paths; track file-session support as a distinct reviewed extension |
| Unsupported macros/vector formats silently disappear | Maintain coverage/error counts and representative fixtures; advertise only measured supported forms |

The main uncertainty is the cloud validation outcome, not the basic architecture. If normalized images are unstable, standalone documents fail to form chunks, or the attachment-first path cannot meet source-citation and isolation gates, resolve those failures before widening input acceptance or beginning a broad backfill.

#### Estimated Effort

The previous 23-37 engineering-day total covered broader source, UI and extraction work and is superseded by this scope reduction. Re-estimate each stage after its design/validation review; do not transfer the old Stage B/C estimates to the new boundaries.

- Stage A: Confluence JPEG/PNG transport, scoped state, lifecycle, ownership, image-only pipeline validation, citation persistence and the bounded pilot. The prior 12-18-day Confluence estimate included document qualification and is historical context, not a revised commitment.
- Stage B: validated Confluence PDF/Word attachments using the Stage A infrastructure; estimate after extraction coverage is measured.
- Stage C: standalone JPEG/PNG admission from manual uploads, Slack files and ordinary email attachments, including bounded replay and existing upload-flow changes; estimate after source-specific review.

Review waits, cloud/quota delays and initial backfill runtime are additional. Preview/admin/status UI, broader formats, renderers and general storage migration are outside these estimates and need separate agreement. The owner approved separate sub-intents, one PR per stage; see Delivery ownership above.

---

## Stage A: Confluence images

| Field | Value |
| --- | --- |
| Intent ID | `39a0be03-276e-47ee-875f-38006b90c52c` |
| Status | planned |
| Parent | `c56ad8b5-2f1d-4d1b-8f58-33476dd9ae6e` |
| Priority | medium |
| Created | 2026-10-16T09:20:08.883Z |
| Updated | 2026-10-16T14:44:31.426Z |
| Plan written | 2026-10-16T14:44:31.423Z |
| Approved by | — |

### Description

Deliver only Confluence JPEG/PNG image attachments, including images displayed in pages. Users ask image-based questions in both chat modes and open the cited parent Confluence page; citations retain the correct source after reload. Validate the image-only Search source-document path as a technical prerequisite without admitting standalone sources. Include secure paginated transport, scoped durable success state, provenance, minimum tree/ownership corrections, enrolled-scope reconciliation, explicit bounded historical backfill, evidence persistence and the Confluence pilot. PDF/Word qualification belongs to B; manual/Slack/mail source admission belongs to C. No new frontend controls or previews.

Dependency/order: First delivery stage. B and C reuse this infrastructure after its PR is integrated; do not implement their inputs here.
Parent: Support images in content (c56ad8b5-2f1d-4d1b-8f58-33476dd9ae6e). One separately shippable PR for this stage. Inherit the parent's base through --parent; verify the implementation baseline before starting. The parent prepared API worktree was based on main and does not contain the POC: preserve/build on feature/content-image-indexing-poc and specs/content-image-indexing-iteration-1/design.md rather than rebuilding it. FE base is main where relevant.

Preserve client/content-rule scoping, source occurrence ownership, persisted citations in both chat modes, client-isolated cache reuse, bounded retry/recovery and stage-required update/deletion handling. Opt-in must not reset Search or start historical scans. Historical work requires explicitly started, bounded client/link/source scope, progress and resumable checkpoints; ongoing reconciliation must respect enrolled scope.

Use synthetic or explicitly non-sensitive fixtures. Real client content waits for storage/access-policy review. Produce human-readable stage designs and obtain review before implementation, defining numeric ingestion/cost budgets and storage/retention/revocation policy. Creating this sub-intent does not approve its design or authorize implementation. Research and implementation are not auto-started.

Quality gates: fixed corpus of at least 20 answerable image questions, at least 80% correct-source retrieval and 70% correct answers, reported separately for ordinary and agentic chat. Abstentions remain in the correctness denominator and are reported separately. Require correct source citations for every answered image question, including conversation reload, and zero observed client/content-rule bypasses in planned tests. Validate the 120-minute healthy-service freshness target for enrolled Confluence updates/confirmed deletions, including inventory detection and Search processing, in stages A/B; Stage C must define source-specific freshness in its reviewed design.

Out of scope: new image-preview/admin/status UI, remote/macro renderers, inline email/CID images, broader formats, selective vector/PDF rendering and image-aware file sessions. Required lifecycle/cleanup and rollout checks ship within this PR, not a later cleanup project. Re-estimate after design/validation; prior broad estimates are superseded.

### Plan

### Stage A: Confluence images

_Status: Draft - for human review. Creation does not authorize implementation._

Deliver only Confluence JPEG/PNG image attachments, including images displayed in pages. Users ask image-based questions in both chat modes and open the cited parent Confluence page; citations retain the correct source after reload. Validate the image-only Search source-document path as a technical prerequisite without admitting standalone sources. Include secure paginated transport, scoped durable success state, provenance, minimum tree/ownership corrections, enrolled-scope reconciliation, explicit bounded historical backfill, evidence persistence and the Confluence pilot. PDF/Word qualification belongs to B; manual/Slack/mail source admission belongs to C. No new frontend controls or previews.

First delivery stage. B and C reuse this infrastructure after its PR is integrated; do not implement their inputs here.

Parent intent: c56ad8b5-2f1d-4d1b-8f58-33476dd9ae6e. The parent holds the research baseline and cross-stage risk ledger. This sub-intent owns only the steps below, each scoped to Stage A.

#### Constraints and acceptance

Parent: Support images in content (c56ad8b5-2f1d-4d1b-8f58-33476dd9ae6e). One separately shippable PR for this stage. Inherit the parent's base through --parent; verify the implementation baseline before starting. The parent prepared API worktree was based on main and does not contain the POC: preserve/build on feature/content-image-indexing-poc and specs/content-image-indexing-iteration-1/design.md rather than rebuilding it. FE base is main where relevant.

Preserve client/content-rule scoping, source occurrence ownership, persisted citations in both chat modes, client-isolated cache reuse, bounded retry/recovery and stage-required update/deletion handling. Opt-in must not reset Search or start historical scans. Historical work requires explicitly started, bounded client/link/source scope, progress and resumable checkpoints; ongoing reconciliation must respect enrolled scope.

Use synthetic or explicitly non-sensitive fixtures. Real client content waits for storage/access-policy review. Produce human-readable stage designs and obtain review before implementation, defining numeric ingestion/cost budgets and storage/retention/revocation policy. Creating this sub-intent does not approve its design or authorize implementation. Research and implementation are not auto-started.

Quality gates: fixed corpus of at least 20 answerable image questions, at least 80% correct-source retrieval and 70% correct answers, reported separately for ordinary and agentic chat. Abstentions remain in the correctness denominator and are reported separately. Require correct source citations for every answered image question, including conversation reload, and zero observed client/content-rule bypasses in planned tests. Validate the 120-minute healthy-service freshness target for enrolled Confluence updates/confirmed deletions, including inventory detection and Search processing, in stages A/B; Stage C must define source-specific freshness in its reviewed design.

Out of scope: new image-preview/admin/status UI, remote/macro renderers, inline email/CID images, broader formats, selective vector/PDF rendering and image-aware file sessions. Required lifecycle/cleanup and rollout checks ship within this PR, not a later cleanup project. Re-estimate after design/validation; prior broad estimates are superseded.

#### Implementation Steps

1. Review the Confluence ingestion design and confirm the POC baseline <!-- step:52c0cc -->
   - Start implementation from API `feature/content-image-indexing-poc` at the researched revision and FE `main`. Recheck differences if either branch advanced.
   - Produce human-readable stage A `spec.md`, `design.md`, `plan.md`, `tasks.md`, contracts and an acceptance matrix under `specs/content-image-indexing-iteration-2/`. Include a data-flow diagram, example attachment paths/metadata, state transitions, source-link examples, security boundaries, rollback and cost estimates. Obtain review before implementing stage A, as required by both the intent and API constitution.
   - Choose attachment-first indexing explicitly. Stage A accepts only Confluence JPEG/PNG image attachments. PDF/Word attachments move to Stage B after validation; standalone source admission moves to C. Other formats and renderers remain deferred. Inventory attachment-backed `ac:image`/`ri:attachment`, standard image elements, remote images, cross-page references and diagram macros; distinguish supported downloads from unresolved forms.
   - Define whether attachment ingestion is gated by `ContentImagesEnabled`; recommended initial behavior is opt-in-only attachment ingestion, leaving existing text-page sync unchanged. Disabling preserves existing content and does not purge it.
   - Apply the owner-confirmed Stage A boundary: grounded text answers and parent-page citations in both chat modes; in-ANA image previews and new admin/status controls are outside the reduced delivery scope. Verify citation source identity after conversation reload, including page/image-only cases.
   - Design an explicitly started, bounded historical Confluence backfill with client/link/space/page scope, progress, cancellation, restart/resume and workload limits. Keep this separate from an Azure Search reset and from opt-in. Define how scheduled reconciliation respects the enrolled scope so it cannot silently start a historical backfill.
   - Use synthetic or explicitly non-sensitive pilot fixtures; real client content waits for storage/access-policy review. Resolve public-blob handling and connector restriction/revocation semantics before sensitive rollout. Keep the confirmed two-hour freshness target and 80% retrieval/70% correct-answer pilot gates in the acceptance matrix. Review numeric ingestion/cost budgets and reconciliation capacity rather than leaving them implicit in code.

2. Validate Confluence image attachments through the existing image-only pipeline <!-- step:fea3ad -->
   - Extend `DescribeImagesTests` and Search configuration tests with JPEG/PNG sources and empty document text. Add a real-Azure acceptance procedure using a disposable test client; local mocks cannot prove document cracking.
   - Verify normalized output, merge with empty/missing source text or offsets, nonempty chunks/embeddings, image-only semantic/keyword retrieval, source URL/date/type, marker splitting, and `ViewContentImage` in both chat modes.
   - Keep normalized JPEG copies at `photo.png.images/{id}.jpg` for compatibility. Do not optimize away the second copy until access and source-format behavior justify a separate reviewed change.
   - If empty-content merge fails, add the smallest image-only text fallback before splitting. Preserve the ordinary PDF/Office and opted-out paths; do not replace the pipeline or immediately add sync-time model calls.
   - Advertise only Confluence JPEG/PNG image support in Stage A. Test rejected/deferred attachment types without admitting PDF/Word, other Office formats, manual uploads, Slack images or mail images. Keep regression checks for existing POC document behavior; new document extraction qualification belongs to Stage B.
   - Use a Stage A Confluence JPEG/PNG corpus with at least 20 answerable image questions. Retain the 80% retrieval and 70% correct-answer pilot gates and applicable meaningful/decorative image checks. Move the full iteration 1 document extraction corpus (10 documents, 30 meaningful images, two Word files and a 10-page scan) and its 90% marker/90% decorative-suppression gates to Stage B. Score retrieval and answer correctness separately for each chat mode on a fixed corpus; abstentions on answerable questions remain in the denominator and are reported separately. Require correct source citations for every answered image question, including after conversation reload.
   - Reprocess unchanged bytes and require cache hits without new model calls or copy names. Measure actual dimensions, reasoning settings, latency and peak skill memory; reproduce transient, unprocessable and configuration errors with fault injection. Record actual results, not assumed passes.

3. Define scoped attachment identities and durable sync state <!-- step:b9086f -->
   - Add typed attachment records and path construction with client, normalized source site/link, page ID, attachment ID, version, filename, media type, size, source URL and successful sync revision. Sanitize names and reserve `.images`; distinguish duplicate filenames by attachment ID.
   - Partition new attachment state by client and stable source/link identity, with page/attachment IDs in row identity. Preserve source site identity in metadata and any cache/tree keys; identical IDs on two sites must remain separate.
   - Correct the existing page-state collision risk with an additive scoped-key format or new table. Use a documented compatibility reader/migration for legacy rows that verifies ownership; never adopt or delete another client's row. Keep old state available for rollback until migration is verified.
   - Separate last-attempt/error fields from last-successful page and attachment versions. A failed download/list/upload must remain eligible for retry even if the page timestamp is unchanged. Reuse storage interfaces and inject state/transport dependencies into the shared sync service so real sync behavior is testable.
   - Carry page provenance via `BlobMetadata.Create`: retain `Confluence`/`ExternalConfluence`, `confluencePageId`, ancestor IDs/names, date, and a canonical page URL. Add attachment/link/version metadata without adding enum values unnecessarily.

4. Add secure Confluence attachment listing and downloads <!-- step:7dbb34 -->
   - Extend `ConfluenceService` or a small typed attachment reader with current-page attachment enumeration and streamed binary download. Follow every page of results; do not use the existing string-deserialization helper for bytes.
   - Use request-specific authorization for the selected link instead of mutating shared `DefaultRequestHeaders`. Validate pagination origins and binary download redirects; never forward Basic credentials to another host. Allow documented trusted download hosts explicitly and do not fetch arbitrary page-provided URLs with connector credentials.
   - Enforce configured per-file bytes, page/run attachment and byte budgets, request deadlines, cancellation and bounded concurrency. Verify actual bytes/media type rather than trusting the extension or an HTML login response.
   - Apply bounded Retry-After-aware backoff to 429/temporary errors. Distinguish access denial, missing attachment, unsupported type, oversized input and incomplete traversal in sync results; none is an instruction to delete the page's inventory.
   - Add fake-HTTP tests for pagination, relative URLs, redirects, multiple credentials/sites, cancellation, malformed payloads, login responses, 401/403/404/429 and maximum-size streams. Verify secrets and signed URLs do not appear in logs.

5. Ingest Confluence attachments with page provenance <!-- step:7507fc -->
   - Load the client's opt-in in `ConfluenceLinkRepository`/`ConfluenceLinkDbEntity` and pass explicit link identity into the sync operation. Do not add model credentials to the Confluence worker.
   - Enumerate attachments and ingest only validated JPEG/PNG images before applying the 50-character Markdown test. Image-only pages must retain their attachment evidence even if no page Markdown is uploaded; create a synthetic page tree node where needed.
   - Upload changed/new source attachments at deterministic paths with valid content types and page provenance. Compare successful version/checksum state before uploading so unchanged syncs do not change blob timestamps and trigger unnecessary extraction/embedding.
   - Keep derived copies exclusively under `ContentImageStorage`; only they carry `AzureSearch_Skip`. Source attachments must be indexable, subject to the client's opt-in and existing filters.
   - Preserve useful page links/captions in Markdown and record which references resolve to which attachment IDs. In stage A, page-local image elements referencing downloaded attachments are covered; arbitrary external URLs and unrendered macros are reported as coverage gaps.
   - Add shared-service tests for meaningful/short/image-only pages, attachment-only pages, repeated references, identical images across files, duplicate filenames, same IDs across clients/sites, and partial success. Maintain existing Grain-link and table conversion behavior.

6. Reconcile attachment changes and removals safely <!-- step:a35c6a -->
   - Do not rely on page `lastmodified` alone. Add a bounded attachment inventory sweep for opted-in links within the existing Confluence job host, independent of unchanged-page skipping. Persist a continuation/checkpoint so large spaces progress within run budgets. Design for the owner-confirmed target of no more than two hours from a source attachment update or confirmed deletion to the corresponding Search result change within enrolled scope under healthy services. Budget that interval across inventory traversal, sync and Search processing; do not start its clock only after discovery. Record actual lag and capacity limits in the pilot.
   - Compare complete remote inventories against successfully owned local attachment state. Download new versions, update renamed paths/provenance, and remove confirmed deleted attachments and their derived copies. Use complete successful traversals as the prerequisite for deletion.
   - Extend deleted-page and unlink/client-cleanup paths to remove owned attachment prefixes and state. Delete blobs before retiring state, or persist a tombstone, so a crash cannot permanently orphan attachments. Respect Search's soft-delete detection window and verify projected chunks disappear.
   - Propagate ancestor moves and page URL/title changes to every attachment's metadata. Keep stable attachment paths across page title changes; update any owner-page path mapping needed by legacy rules.
   - Coordinate incremental, ancestors, deletion and attachment reconciliation with the same client/link locking boundary; existing deletion/ancestor jobs have different global locks and must not race new attachment work.
   - Test replacement without page edits, failed enumeration, empty results, lost permissions, rename, ancestor move, unlink, interrupted deletion and replay. Permission revocation must follow the reviewed policy, not be treated as an empty successful inventory.

7. Preserve attachment tree ownership (Stage A) <!-- step:4aaabc -->
   - Separate page identity from attachment children and derived `.images` folders; never deduplicate attachments by page ID. Support synthetic page nodes when image-only pages have no Markdown blob.
   - Reuse Confluence subtree metadata for exclusions/Focus Mode. Where legacy blob-document ownership requires it, add the reviewed `contentOwnerParentId` projection/filter support before enabling attachment queries. Apply schema changes before queries reference new fields.
   - Test several attachments, duplicate page IDs across links/sites, ancestor moves and legacy page rules. Preserve `AzureSearch_Skip` and reserved-path guards. Restrict changes to ownership/correctness; no new image description/status/preview contract or frontend tree controls.

8. Preserve image evidence and citations across both chat modes <!-- step:d22ebd -->
   - Introduce a per-run evidence registry or equivalent stable reference mapping used by search output, image authorization and final persistence. Ordinary chat must retain sources used by earlier searches instead of persisting only `LastSearchContext`.
   - Assign stable `[docN]` references across repeated searches; preserve compatibility with `ChatMessageContext`/`MessageContextItemDto`. For agentic synthesis, merge or preserve distinct chunks from the same document rather than dropping all but its first chunk in `JoinExecutor`.
   - Make probe registration deliberate: probe-only IDs must not become usable uncitable evidence in the answer. Promote the source into synthesis/persisted evidence before image viewing, or keep probe IDs unavailable.
   - Preserve the image-to-source occurrence map and source citation when returning tool text. Reject unknown IDs and mismatched clients; never accept a model-supplied storage URL. Cover identical IDs in multiple permitted documents and concurrent searches without changing document ownership.
   - Keep retrieved image markers when whole-document raw text is added; avoid trying to text-decode standalone images. File sessions remain a documented separate limitation until an image-aware design is reviewed.
   - Test an image from search A followed by search B, several image chunks in one document, split end markers, source exclusions and Focus Mode, prompt-injection text, missing copies, timeouts and later-turn re-retrieval. Reload saved conversations and verify each image-based claim opens the correct source page/file.

9. Qualify and review the Confluence pilot <!-- step:bf8224 -->
   - Use a disposable opted-in client and an opted-out control with synthetic or explicitly non-sensitive fixtures. Real client content waits for storage/access-policy review. Include internal/external Confluence links, PNG/JPEG diagrams, screenshots, image-only pages, paginated inventories, attachment version changes, deleted pages and ancestor moves.
   - Start the historical pilot backfill explicitly for selected client/link/space/page scope. Stop and resume it at configured limits without repeating successful unchanged uploads or model calls. Verify opt-in alone starts neither a historical scan nor a Search reset; record which historical pages are outside the selected scope.
   - Verify text answers cite the correct parent Confluence page in both chat modes and after conversation reload. PDF/Word attachment qualification, standalone source admission and preview UI must not block this milestone.
   - Require no client/rule bypass in retrieval or viewing; unchanged inventories must cause no new downloads/uploads/model calls; derived copies must never become Search documents. Test legacy page-document rules as well as Confluence subtree rules.
   - Compare page-plus-image questions against the attachment-first tradeoff. Document incorrect answers and missing page context with examples before choosing a more complex design.
   - Record the fixed question corpus and ground truth before scoring. Require at least 80% correct-source retrieval and 70% reviewer-rated correct answers separately for each chat mode. Report abstentions without removing answerable questions from the denominator. Require correct citations for every answered image question and zero observed client/content-rule bypasses in the planned access tests.
   - Test controlled updates and confirmed deletions against the 120-minute end-to-end target, including changes without page edits. Record healthy-service conditions, source/discovery/blob/Search timestamps and all breaches. Keep initial backfill duration and permission-revocation behavior distinct from this target.
   - Capture coverage counts, unresolved image forms, retrieval/citation correctness, cost by client, retry exhaustion, memory and reconciliation delay. Review the results before moving beyond the pilot; add targeted recovery commands for failed documents rather than resetting every client.

10. Add only stage-required cleanup and operational recovery <!-- step:81f915 -->
   - Deliver cleanup required for each stage's accepted update/deletion behavior with that stage. Reconcile obsolete source/image occurrences only after a completed owned revision or complete inventory, respecting active runs, source moves and reviewed retention/revocation policy. Retain shared description rows when an individual document is deleted.
   - Keep caches client-isolated and versioned. Use targeted recovery for failed attachment versions and exhausted image work; ordinary deployment must not invalidate all clients or reset Search.
   - Preserve low per-request parallelism and the dedicated description deployment. Define numeric run/cost budgets before rollout; add deployment-wide concurrency limits only if measured quota requires them.
   - Test cleanup isolation, crash replay, unchanged reindexing and retained references. Broad historical orphan sweeps, new renderers and extraction optimization projects are deferred.

11. Promote each stage with measured deployment and rollback controls <!-- step:fb7e5b -->
   - Apply these release checks only to Stage A in this sub-intent; retain the parent A then B then C order. Start with a disposable client using synthetic or explicitly non-sensitive fixtures and unique containers/indexes. Real client content waits for storage/access-policy review; QA namespace/SQL separation alone does not isolate shared Search/storage.
   - Verify the dedicated description deployment, limited skill-host secrets, migration, Search configuration, retry worker, request/resource limits, telemetry and numeric ingestion/reconciliation budgets. Prepare reviewed Helm/GitOps changes and optional environment-specific deployment resources only as required; no secret literals.
   - Smoke-test both chat modes for opted-out clients, then opt-in, partial configuration failure/retry, disable, changed content and explicitly selected backfill. Reconcile Search resources and SQL flags after partial failure; emergency stop is complete only when Search image processing is disabled.
   - Run affected .NET tests for the stage's image/indexer/API/Confluence paths. Run Slack/mail tests and FE build/browser upload checks in C when those paths change; render/lint charts and validate/plan infrastructure only where changed. Deploy only through the reviewed stage rollout.
   - Record the stage's fixed-corpus results, source/citation correctness, isolation tests, freshness, cost, unresolved forms and operator recovery/rollback instructions. Promote API/contracts before dependent FE and one client at a time. Disable and historical-evidence purge remain separate operations.

---

## Stage B: Validated Confluence attachments

| Field | Value |
| --- | --- |
| Intent ID | `87ca7bd4-4f0a-467b-ad5f-b101cf9d5549` |
| Status | planned |
| Parent | `c56ad8b5-2f1d-4d1b-8f58-33476dd9ae6e` |
| Priority | medium |
| Created | 2026-10-16T09:20:42.094Z |
| Updated | 2026-10-16T09:20:42.479Z |
| Plan written | 2026-10-16T09:20:42.479Z |
| Approved by | — |

### Description

Add only validated Confluence PDF/Word attachments using Stage A transport, scoped state, source ownership and lifecycle. Distinguish document admission from embedded-image extraction; advertise only measured formats/behaviors. Run the full iteration 1 extraction corpus (at least 10 documents, 30 meaningful images, two Word files and a 10-page scan), retaining 90% meaningful-image marker and 90% decorative-suppression gates. Validate parent-page citations in both chat modes and after reload, cache reuse, bounded backfill and update/delete freshness. No standalone admission, broader Office formats, new connectors or preview/admin/status UI.

Dependency/order: Depends on Stage A (39a0be03-276e-47ee-875f-38006b90c52c) being integrated. Re-read A's final design/implementation before implementing B and preserve A's JPEG/PNG behavior.

Parent: Support images in content (c56ad8b5-2f1d-4d1b-8f58-33476dd9ae6e). One separately shippable PR for this stage. Inherit the parent's base through --parent; verify the implementation baseline before starting. The parent prepared API worktree was based on main and does not contain the POC: preserve/build on feature/content-image-indexing-poc and specs/content-image-indexing-iteration-1/design.md rather than rebuilding it. FE base is main where relevant.

Preserve client/content-rule scoping, source occurrence ownership, persisted citations in both chat modes, client-isolated cache reuse, bounded retry/recovery and stage-required update/deletion handling. Opt-in must not reset Search or start historical scans. Historical work requires explicitly started, bounded client/link/source scope, progress and resumable checkpoints; ongoing reconciliation must respect enrolled scope.

Use synthetic or explicitly non-sensitive fixtures. Real client content waits for storage/access-policy review. Produce human-readable stage designs and obtain review before implementation, defining numeric ingestion/cost budgets and storage/retention/revocation policy. Creating this sub-intent does not approve its design or authorize implementation. Research and implementation are not auto-started.

Quality gates: fixed corpus of at least 20 answerable image questions, at least 80% correct-source retrieval and 70% correct answers, reported separately for ordinary and agentic chat. Abstentions remain in the correctness denominator and are reported separately. Require correct source citations for every answered image question, including conversation reload, and zero observed client/content-rule bypasses in planned tests. Validate the 120-minute healthy-service freshness target for enrolled Confluence updates/confirmed deletions, including inventory detection and Search processing, in stages A/B; Stage C must define source-specific freshness in its reviewed design.

Out of scope: new image-preview/admin/status UI, remote/macro renderers, inline email/CID images, broader formats, selective vector/PDF rendering and image-aware file sessions. Required lifecycle/cleanup and rollout checks ship within this PR, not a later cleanup project. Re-estimate after design/validation; prior broad estimates are superseded.

### Plan

### Stage B: Validated Confluence attachments

_Status: Draft - for human review. Creation does not authorize implementation._

Add only validated Confluence PDF/Word attachments using Stage A transport, scoped state, source ownership and lifecycle. Distinguish document admission from embedded-image extraction; advertise only measured formats/behaviors. Run the full iteration 1 extraction corpus (at least 10 documents, 30 meaningful images, two Word files and a 10-page scan), retaining 90% meaningful-image marker and 90% decorative-suppression gates. Validate parent-page citations in both chat modes and after reload, cache reuse, bounded backfill and update/delete freshness. No standalone admission, broader Office formats, new connectors or preview/admin/status UI.

Depends on Stage A (39a0be03-276e-47ee-875f-38006b90c52c) being integrated. Re-read A's final design/implementation before implementing B and preserve A's JPEG/PNG behavior.

Parent intent c56ad8b5-2f1d-4d1b-8f58-33476dd9ae6e holds the research baseline and cross-stage risk ledger. This child owns only Stage B work.

#### Constraints and acceptance

Parent: Support images in content (c56ad8b5-2f1d-4d1b-8f58-33476dd9ae6e). One separately shippable PR for this stage. Inherit the parent's base through --parent; verify the implementation baseline before starting. The parent prepared API worktree was based on main and does not contain the POC: preserve/build on feature/content-image-indexing-poc and specs/content-image-indexing-iteration-1/design.md rather than rebuilding it. FE base is main where relevant.

Preserve client/content-rule scoping, source occurrence ownership, persisted citations in both chat modes, client-isolated cache reuse, bounded retry/recovery and stage-required update/deletion handling. Opt-in must not reset Search or start historical scans. Historical work requires explicitly started, bounded client/link/source scope, progress and resumable checkpoints; ongoing reconciliation must respect enrolled scope.

Use synthetic or explicitly non-sensitive fixtures. Real client content waits for storage/access-policy review. Produce human-readable stage designs and obtain review before implementation, defining numeric ingestion/cost budgets and storage/retention/revocation policy. Creating this sub-intent does not approve its design or authorize implementation. Research and implementation are not auto-started.

Quality gates: fixed corpus of at least 20 answerable image questions, at least 80% correct-source retrieval and 70% correct answers, reported separately for ordinary and agentic chat. Abstentions remain in the correctness denominator and are reported separately. Require correct source citations for every answered image question, including conversation reload, and zero observed client/content-rule bypasses in planned tests. Validate the 120-minute healthy-service freshness target for enrolled Confluence updates/confirmed deletions, including inventory detection and Search processing, in stages A/B; Stage C must define source-specific freshness in its reviewed design.

Out of scope: new image-preview/admin/status UI, remote/macro renderers, inline email/CID images, broader formats, selective vector/PDF rendering and image-aware file sessions. Required lifecycle/cleanup and rollout checks ship within this PR, not a later cleanup project. Re-estimate after design/validation; prior broad estimates are superseded.


#### Implementation Steps

1. Review and validate Confluence document attachments (Stage B) <!-- step:b758bf -->
   - Produce a human-readable Stage B design and acceptance matrix before implementation. Limit the scope to PDF/Word attachments from the existing Confluence connector; reuse Stage A transport, scoped state, provenance, backfill and reconciliation.
   - Run the iteration 1 extraction corpus: at least 10 documents, 30 meaningful images, two Word files and a 10-page scan. Require 90% meaningful-image markers and 90% decorative suppression; distinguish document admission from actual embedded-image extraction. Advertise only individual formats/extraction behaviors demonstrated by validation.
   - Validate ordering, scanned documents, normalized-byte stability, merge/split behavior, cache reuse, model cost and retry isolation. Preserve the JPEG/PNG path and existing text-document behavior.
   - Enable validated attachment types only within the reviewed Confluence allowlist and bounded operator-selected scope. Test update/delete/rename/move/replay and the two-hour enrolled-scope freshness target; no automatic historical scan.
   - Report retrieval and answer correctness separately for both chat modes using fixed questions and ground truth, retaining at least 80% correct-source retrieval, 70% correct answers, correct parent-page citations for every answered image question (including reload) and zero observed client/content-rule bypasses. Keep abstentions in the correctness denominator.
   - No standalone uploads, Slack/mail admission, new settings/status/preview APIs or preview UI in this stage.

2. Add only stage-required cleanup and operational recovery <!-- step:09993c -->
   - Deliver cleanup required for each stage's accepted update/deletion behavior with that stage. Reconcile obsolete source/image occurrences only after a completed owned revision or complete inventory, respecting active runs, source moves and reviewed retention/revocation policy. Retain shared description rows when an individual document is deleted.
   - Keep caches client-isolated and versioned. Use targeted recovery for failed attachment versions and exhausted image work; ordinary deployment must not invalidate all clients or reset Search.
   - Preserve low per-request parallelism and the dedicated description deployment. Define numeric run/cost budgets before rollout; add deployment-wide concurrency limits only if measured quota requires them.
   - Test cleanup isolation, crash replay, unchanged reindexing and retained references. Broad historical orphan sweeps, new renderers and extraction optimization projects are deferred.

3. Promote each stage with measured deployment and rollback controls <!-- step:6e602b -->
   - Apply these release checks only to Stage B in this sub-intent, preserving the parent delivery order. Start with a disposable client using synthetic or explicitly non-sensitive fixtures and unique containers/indexes. Real client content waits for storage/access-policy review; QA namespace/SQL separation alone does not isolate shared Search/storage.
   - Verify the dedicated description deployment, limited skill-host secrets, migration, Search configuration, retry worker, request/resource limits, telemetry and numeric ingestion/reconciliation budgets. Prepare reviewed Helm/GitOps changes and optional environment-specific deployment resources only as required; no secret literals.
   - Smoke-test both chat modes for opted-out clients, then opt-in, partial configuration failure/retry, disable, changed content and explicitly selected backfill. Reconcile Search resources and SQL flags after partial failure; emergency stop is complete only when Search image processing is disabled.
   - Run affected .NET tests for the stage's image/indexer/API/Confluence paths. Run Slack/mail tests and FE build/browser upload checks in C when those paths change; render/lint charts and validate/plan infrastructure only where changed. Deploy only through the reviewed stage rollout.
   - Record the stage's fixed-corpus results, source/citation correctness, isolation tests, freshness, cost, unresolved forms and operator recovery/rollback instructions. Promote API/contracts before dependent FE and one client at a time. Disable and historical-evidence purge remain separate operations.

---

## Stage C: Standalone images

| Field | Value |
| --- | --- |
| Intent ID | `faf98799-33f5-4569-b404-0c95a19dd3b3` |
| Status | planned |
| Parent | `c56ad8b5-2f1d-4d1b-8f58-33476dd9ae6e` |
| Priority | medium |
| Created | 2026-10-16T09:20:42.917Z |
| Updated | 2026-10-16T09:20:43.295Z |
| Plan written | 2026-10-16T09:20:43.295Z |
| Approved by | — |

### Description

Admit standalone JPEG/PNG knowledge images from manual uploads, Slack files and ordinary non-inline email attachments. Reuse the image pipeline and source-bound evidence/citations established in Stage A. Limit frontend work to validated image admission and validation feedback in the existing knowledge-upload flow. Include MIME/signature/size limits, source-specific provenance, client/content-rule checks, lifecycle and explicit bounded replay/backfill with per-source end-to-end acceptance. Preserve Slack channel/file URLs and email attachment provenance. Skip/report inline signatures and CID images. No new preview/admin/status UI, broader formats or changes to existing chat-message image attachments.

Dependency/order: Third planned PR, after A and B. Reuse integrated A infrastructure. B is the preceding delivery milestone; do not couple standalone admission to PDF/Word internals. Re-read earlier stage outcomes before implementation. Stage B: 87ca7bd4-4f0a-467b-ad5f-b101cf9d5549. Stage A: 39a0be03-276e-47ee-875f-38006b90c52c.

Parent: Support images in content (c56ad8b5-2f1d-4d1b-8f58-33476dd9ae6e). One separately shippable PR for this stage. Inherit the parent's base through --parent; verify the implementation baseline before starting. The parent prepared API worktree was based on main and does not contain the POC: preserve/build on feature/content-image-indexing-poc and specs/content-image-indexing-iteration-1/design.md rather than rebuilding it. FE base is main where relevant.

Preserve client/content-rule scoping, source occurrence ownership, persisted citations in both chat modes, client-isolated cache reuse, bounded retry/recovery and stage-required update/deletion handling. Opt-in must not reset Search or start historical scans. Historical work requires explicitly started, bounded client/link/source scope, progress and resumable checkpoints; ongoing reconciliation must respect enrolled scope.

Use synthetic or explicitly non-sensitive fixtures. Real client content waits for storage/access-policy review. Produce human-readable stage designs and obtain review before implementation, defining numeric ingestion/cost budgets and storage/retention/revocation policy. Creating this sub-intent does not approve its design or authorize implementation. Research and implementation are not auto-started.

Quality gates: fixed corpus of at least 20 answerable image questions, at least 80% correct-source retrieval and 70% correct answers, reported separately for ordinary and agentic chat. Abstentions remain in the correctness denominator and are reported separately. Require correct source citations for every answered image question, including conversation reload, and zero observed client/content-rule bypasses in planned tests. Validate the 120-minute healthy-service freshness target for enrolled Confluence updates/confirmed deletions, including inventory detection and Search processing, in stages A/B; Stage C must define source-specific freshness in its reviewed design.

Out of scope: new image-preview/admin/status UI, remote/macro renderers, inline email/CID images, broader formats, selective vector/PDF rendering and image-aware file sessions. Required lifecycle/cleanup and rollout checks ship within this PR, not a later cleanup project. Re-estimate after design/validation; prior broad estimates are superseded.

### Plan

### Stage C: Standalone images

_Status: Draft - for human review. Creation does not authorize implementation._

Admit standalone JPEG/PNG knowledge images from manual uploads, Slack files and ordinary non-inline email attachments. Reuse the image pipeline and source-bound evidence/citations established in Stage A. Limit frontend work to validated image admission and validation feedback in the existing knowledge-upload flow. Include MIME/signature/size limits, source-specific provenance, client/content-rule checks, lifecycle and explicit bounded replay/backfill with per-source end-to-end acceptance. Preserve Slack channel/file URLs and email attachment provenance. Skip/report inline signatures and CID images. No new preview/admin/status UI, broader formats or changes to existing chat-message image attachments.

Third planned PR, after A and B. Reuse integrated A infrastructure. B is the preceding delivery milestone; do not couple standalone admission to PDF/Word internals. Re-read earlier stage outcomes before implementation. Stage B: 87ca7bd4-4f0a-467b-ad5f-b101cf9d5549. Stage A: 39a0be03-276e-47ee-875f-38006b90c52c.

Parent intent c56ad8b5-2f1d-4d1b-8f58-33476dd9ae6e holds the research baseline and cross-stage risk ledger. This child owns only Stage C work.

#### Constraints and acceptance

Parent: Support images in content (c56ad8b5-2f1d-4d1b-8f58-33476dd9ae6e). One separately shippable PR for this stage. Inherit the parent's base through --parent; verify the implementation baseline before starting. The parent prepared API worktree was based on main and does not contain the POC: preserve/build on feature/content-image-indexing-poc and specs/content-image-indexing-iteration-1/design.md rather than rebuilding it. FE base is main where relevant.

Preserve client/content-rule scoping, source occurrence ownership, persisted citations in both chat modes, client-isolated cache reuse, bounded retry/recovery and stage-required update/deletion handling. Opt-in must not reset Search or start historical scans. Historical work requires explicitly started, bounded client/link/source scope, progress and resumable checkpoints; ongoing reconciliation must respect enrolled scope.

Use synthetic or explicitly non-sensitive fixtures. Real client content waits for storage/access-policy review. Produce human-readable stage designs and obtain review before implementation, defining numeric ingestion/cost budgets and storage/retention/revocation policy. Creating this sub-intent does not approve its design or authorize implementation. Research and implementation are not auto-started.

Quality gates: fixed corpus of at least 20 answerable image questions, at least 80% correct-source retrieval and 70% correct answers, reported separately for ordinary and agentic chat. Abstentions remain in the correctness denominator and are reported separately. Require correct source citations for every answered image question, including conversation reload, and zero observed client/content-rule bypasses in planned tests. Validate the 120-minute healthy-service freshness target for enrolled Confluence updates/confirmed deletions, including inventory detection and Search processing, in stages A/B; Stage C must define source-specific freshness in its reviewed design.

Out of scope: new image-preview/admin/status UI, remote/macro renderers, inline email/CID images, broader formats, selective vector/PDF rendering and image-aware file sessions. Required lifecycle/cleanup and rollout checks ship within this PR, not a later cleanup project. Re-estimate after design/validation; prior broad estimates are superseded.


#### Implementation Steps

1. Review standalone image source admission (Stage C) <!-- step:ad06b3 -->
   - Produce a human-readable Stage C design/contracts and focused FE upload tasks. Limit new source coverage to standalone knowledge images from manual uploads, Slack files and ordinary non-inline email attachments, initially JPEG/PNG validated in A.
   - Define per-source provenance, MIME/signature/size limits, client opt-in, source/content-rule checks, lifecycle and explicit bounded replay/backfill. Reuse the existing image describer/cache and source citation behavior; chat-message image attachments remain a separate existing feature.
   - Keep changes to the existing knowledge-upload UI limited to admitting the validated image formats and useful validation errors. New admin toggles, status UI and in-ANA image previews remain deferred.
   - Keep a deferred coverage ledger for remote Confluence images/macros, cross-page forms, inline mail/CID images, broader Office/raster formats, selective PDF/vector rendering and image-aware file sessions; do not implement those extensions in these stages.

2. Admit standalone images from the agreed knowledge sources (Stage C) <!-- step:7fdfd5 -->
   - Extend manual knowledge upload validation, the FE upload allowlist and `Constants.AllowedIndexFileExtensions` only with agreed, validated JPEG/PNG inputs. Do not treat chat-file support as proof of Search support or expand Office allowlists.
   - Apply bounded size and MIME/signature validation to manual, Slack and mail inputs. Preserve Slack channel/file URLs and email attachment provenance. Skip/report inline signatures and CID images explicitly.
   - Add explicit scoped source replay/backfill; allowlist changes alone do not revisit Slack/mail items behind existing checkpoints. Enabling image processing must not initiate broad historical work.
   - Validate upload/import through indexing, retrieval, correct source citations and reload for each source, plus opted-out behavior, content-rule isolation, update/delete, rejected formats and unchanged replay/cache reuse. Report both chat modes separately and measure quality/cost before rollout.

3. Add only stage-required cleanup and operational recovery <!-- step:4abe4e -->
   - Deliver cleanup required for each stage's accepted update/deletion behavior with that stage. Reconcile obsolete source/image occurrences only after a completed owned revision or complete inventory, respecting active runs, source moves and reviewed retention/revocation policy. Retain shared description rows when an individual document is deleted.
   - Keep caches client-isolated and versioned. Use targeted recovery for failed attachment versions and exhausted image work; ordinary deployment must not invalidate all clients or reset Search.
   - Preserve low per-request parallelism and the dedicated description deployment. Define numeric run/cost budgets before rollout; add deployment-wide concurrency limits only if measured quota requires them.
   - Test cleanup isolation, crash replay, unchanged reindexing and retained references. Broad historical orphan sweeps, new renderers and extraction optimization projects are deferred.

4. Promote each stage with measured deployment and rollback controls <!-- step:794f4e -->
   - Apply these release checks only to Stage C in this sub-intent, preserving the parent delivery order. Start with a disposable client using synthetic or explicitly non-sensitive fixtures and unique containers/indexes. Real client content waits for storage/access-policy review; QA namespace/SQL separation alone does not isolate shared Search/storage.
   - Verify the dedicated description deployment, limited skill-host secrets, migration, Search configuration, retry worker, request/resource limits, telemetry and numeric ingestion/reconciliation budgets. Prepare reviewed Helm/GitOps changes and optional environment-specific deployment resources only as required; no secret literals.
   - Smoke-test both chat modes for opted-out clients, then opt-in, partial configuration failure/retry, disable, changed content and explicitly selected backfill. Reconcile Search resources and SQL flags after partial failure; emergency stop is complete only when Search image processing is disabled.
   - Run affected .NET tests for the stage's image/indexer/API/Confluence paths. Run Slack/mail tests and FE build/browser upload checks in C when those paths change; render/lint charts and validate/plan infrastructure only where changed. Deploy only through the reviewed stage rollout.
   - Record the stage's fixed-corpus results, source/citation correctness, isolation tests, freshness, cost, unresolved forms and operator recovery/rollback instructions. Promote API/contracts before dependent FE and one client at a time. Disable and historical-evidence purge remain separate operations.

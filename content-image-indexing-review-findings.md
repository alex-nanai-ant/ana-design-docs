# Content Image Indexing — Review Findings

_Date: 2026-10-12_
_Scope: branch `feature/content-image-indexing-poc` in `altus-ai-assistant`, branch `proposal/content-image-indexing-poc` in `altus-ai-gitops`._
_Design: [content-image-indexing-design.md](content-image-indexing-design.md)._
_Status: open findings from the code review. Fix them one at a time and set the status._

Status values: `open`, `fixed`, `accepted` (no change), `later` (follow-up).

## 1. Important gaps

| # | Finding | Proposed fix | Status |
| --- | --- | --- | --- |
| 1 | Descriptions are only in the search index. A whole-document read in agentic mode gets the PDF text from blob storage, so the text has no markers. See `ReadWholeDocument` in [KnowledgeRetriever.cs](../altus-ai-assistant/AltusAiAssistantApi/Services/Maf/KnowledgeRetriever.cs). Questions about one specific file use this path. | Deferred. The read is rare: it needs agentic mode, a question about one specific file, and a PDF or Word file. The one retrieved chunk of the document still has its markers. Options for the fix: (a) for clients with images enabled, read the whole-document text from the index and register its image IDs; (b) send the full file to the model, not the extracted text, so that the model sees the images. Option (b) works only for PDF, and it is not verified that the Azure deployment accepts PDF input. | later |
| 2 | File sessions read the file directly. They get no descriptions, and the image tool is off for them. | No change. File sessions are obsolete and will be removed. | accepted |
| 3 | The `Register` call in [JoinExecutor.cs](../altus-ai-assistant/AltusAiAssistantApi/Services/Maf/Executors/JoinExecutor.cs) finds no markers for a PDF. It has an effect only for the Confluence path. | Removed the call and the `imageRun` parameter of the join executor and its factory. `JoinExecutor.cs` is the same as on `main` again. The Confluence path must add the call again. | fixed |
| 4 | `run.id` is a metric label in [ContentImageTelemetry.cs](../altus-ai-assistant/AltusAiShared/ContentImages/ContentImageTelemetry.cs). It has a new value for each image request and each chat turn. Thus, the number of time series has no limit. | Removed the label. The run value is not kept: it was one value per image request or a random value per chat turn, and matched no other ID. The design now says "per client". Also reduced the metrics to three counters (images, tokens, tool calls): removed the model call counter, the two duration histograms, and the `kind` label. | fixed |
| 5 | The skill pod is open to the internet. It now holds the storage account key and the model key. Gitops has no IP restriction for it. | Deferred. The first proposal was to permit only the Azure AI Search service tag at the gateway. This is not possible there: a service tag exists only in an NSG rule, an NSG rule matches an IP address and a port, and all hosts share one IP address on port 443. Contour has no IP filter for an `HTTPRoute`. The tag also includes the indexers of all Azure AI Search customers, so the skill code stays the only authentication. Alternative for the fix: authenticate the Search service with its managed identity. (a) Set `AuthResourceId` on the two `WebApiSkill`s in [IndexingService.cs](../altus-ai-assistant/AltusAiAssistantApi/Services/IndexingService.cs), so that Search sends an Entra token. (b) The skill host validates the token (audience, tenant, object ID of the Search service of the environment) in middleware, before it reads the body. (c) Accept the code or the token until the skillset of each client is written again. Then remove the code. Verify first: each Search service has a managed identity, and Search sends the token to a host that is not a Function or an App Service. If a network rule is also necessary: a second public IP address and a second Envoy `LoadBalancer` Service with the `azure-allowed-service-tags` annotation, with the skill listeners on a port that only this Service exposes. | later |

## 2. Unnecessary setup

| # | Finding | Proposed fix | Status |
| --- | --- | --- | --- |
| 6 | SkiaSharp (9 MB native library) is in the shared project, so all eight projects ship it. The describer uses it only to read the width and height. | Search supplies `width` and `height` for each extracted image. The `describe-images` skill now has these two inputs, and the describer uses them for the size filter. Removed SkiaSharp from the shared project. The API keeps its own reference, which `main` has for chat image uploads, so only the API ships the library. The describer no longer decodes the image: the model rejects an invalid image, and the result is `unprocessable` as before. If the width or the height is missing or 0, the describer applies no filter and describes the image. | fixed |
| 7 | SharpToken (7 MB) is used only to cut the description at 300 tokens. `cl100k_base` is not the tokenizer of the current models. | The describer now cuts the description at 1,200 characters, which is approximately 300 tokens of English text. The setting is `MaxDescriptionCharacters`. Removed SharpToken from the shared project. The API keeps its own reference, which `main` has for token counts in chat, so only the API ships the library. | fixed |
| 8 | `CreateReadLink` in [ContentImageStorage.cs](../altus-ai-assistant/AltusAiShared/ContentImages/ContentImageStorage.cs) has a branch for a user delegation key. The two hosts use an account key, so this branch never runs. | Remove the branch. | open |
| 9 | `ChatToolProvider` has four optional constructor parameters with `null` defaults, and then uses `_settings!`. Production always supplies them. The defaults are there only for older tests. | Make the parameters required and update the tests. | open |

## 3. Small problems

| # | Finding | Proposed fix | Status |
| --- | --- | --- | --- |
| 10 | The skill host writes a critical log at each start when the image skill code is not set. This is the case in production now. | Use a lower log level for the image skill. | open |
| 11 | On a cache hit, the describer uploads the full image again. Storage rejects it because the copy exists. Then the describer reads the properties. | Read the properties first. Upload only if the copy does not exist. | open |
| 12 | `PromptVersion` is a manual setting. If the prompt or the model changes and the version does not, the old descriptions stay and there is no warning. | Calculate the version from the prompt text and the deployment name, or add a note to the release steps. | open |
| 13 | The prompt in [ContentImageModel.cs](../altus-ai-assistant/AltusAiIndexerSkills/ContentImageModel.cs) contains "300 tokens" as fixed text. The limit is a setting, `MaxDescriptionCharacters`, and its unit is characters (item 7). | Put the setting value into the prompt, in characters or as a number of tokens that is calculated from it. This changes the prompt, so it needs a new prompt version. | open |
| 14 | The image question uses the reasoning effort of the turn. In agentic mode this is `high`. The timeout is 60 seconds. | Use a fixed `low` effort for the image question. | open |
| 15 | `AltusAiIndexerSkills/Properties/launchSettings.json` is not tracked. | Commit it or add it to `.gitignore`. | open |
| 16 | Gitops uses the deployment name `gpt-6-luna-content-images`. Design §12.1 specifies `content-images`, so that the name does not contain the model. | Use one name in the two places. | open |
| 17 | The path guard rejects each path that has a segment that ends in `.images`. This includes folders that a user made before this feature. | Accept, or check production for such folders before the merge. | open |

## 4. Items to verify in QA

| # | Item | Why it is important | Status |
| --- | --- | --- | --- |
| 18 | Azure accepts `reasoning_effort` with API version `2024-10-21`. | The branch now sends it on each chat call. If Azure rejects it, all chat fails. Test one message in each chat mode first. | open |
| 19 | Search produces the same image bytes when it indexes an unchanged document again (design §10 item 4). | The image ID is a hash of the bytes. If the bytes change, the cache never hits and each re-index stores new copies. | open |
| 20 | A document that fails a re-index keeps its previous chunks (design §10 item 5). | If not, a backfill during a model outage removes documents from search. | open |
| 21 | After one image record has an error, Search still calls the skill for the other images of the document. | If not, a document with some bad images can use all 5 attempts of the retry worker before one image has 3 failures. | open |
| 22 | The content-filter rejection contains `content_filter` in the error message. | The code finds this failure class by text. If the text is different, the image uses two more attempts. | open |
| 23 | A disable changes an indexer run that is in progress, or only the next run. | This is the time that an emergency stop needs. | open |
| 24 | Memory of the skill pod during a backfill, with a 1Gi limit and 2 instances. | The measured peak sets the production limit. | open |
| 29 | Search sends `width` and `height` to the skill, and the count of images skipped by size is not zero for a PDF that contains icons. | Without the size, the describer applies no filter, so each icon costs a model call. Nothing else shows this failure. | open |

## 5. Rollout and other notes

| # | Note | Action | Status |
| --- | --- | --- | --- |
| 25 | Production gitops still has a 512Mi limit and 1 instance for the skill pod. | Change it at the production rollout, with the result of item 24. | later |
| 26 | Nothing indexes a document again after an image used all its failures (design §4.2). | A job that resets these documents. | later |
| 27 | Image copies stay in storage when a document no longer contains the image (design §10 item 6). | A cleanup job. | later |
| 28 | `POST /Indexing/CreateIndex/{clientId}` in [IndexingController.cs](../altus-ai-assistant/AltusAiAssistantApi/Controllers/IndexingController.cs) permits anonymous calls. It writes the skillset and indexer of a client again. This is not from this branch; `main` has it. | Require the admin role. | open |

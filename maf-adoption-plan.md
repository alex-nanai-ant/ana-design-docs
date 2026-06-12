# Microsoft Agent Framework (MAF) Adoption — `altus-ai-assistant`

> **Status:** draft — for review before implementation.
>
> **Scope:** primary backend only ([altus-ai-assistant](../altus-ai-assistant/)). Move **both** chat-completion paths — non-agentic `CompleteChat` and agentic `CompleteChatWithPlanning` — onto MAF inside `MafCompleteChatService`, built on a set of shared abstractions. The legacy completion logic in [ChatService.cs](../altus-ai-assistant/AltusAiAssistantApi/Services/ChatService.cs) is **slated for deletion** once the MAF paths are proven in production. The voice repo and sync workers are out of scope (see [§Non-goals](#non-goals)).

## Goal

`MafCompleteChatService` implements `ICompleteChatService` and is the single home for chat completion. Today it ships an **unreviewed, untested** agentic MAF Workflow (Planner → Search → Join → Synthesizer) behind the **default-off** global `UseMaf` flag — it has **not** run in production (Decision 8) — while its non-agentic `CompleteChat` is a `NotImplementedException` ([MafCompleteChatService.cs:63-64](../altus-ai-assistant/AltusAiAssistantApi/Services/MafCompleteChatService.cs#L63-L64)); the **production-proven** non-agentic and legacy-agentic paths still live in `ChatService`.

The goal is to finish the migration and retire the legacy code:

1. Implement **both** completion methods on `MafCompleteChatService`.
2. Build them on **proper shared abstractions** rather than the retrieval/tool helpers currently copy-pasted from `ChatService` into `MafCompleteChatService` ([MafCompleteChatService.cs:623-893](../altus-ai-assistant/AltusAiAssistantApi/Services/MafCompleteChatService.cs#L623-L893)).
3. **Soak in production** behind `UseMaf`, then **delete** the legacy completion logic in `ChatService`.

## Why "build, soak, then delete"

The agentic path has **no automated tests** today (the xUnit suite targets `HtmlContentHelper` only), and answer quality is judgeable only against real client corpora that don't exist outside production. So the safety strategy is a reversible toggle, not a test suite: keep the legacy path runnable, route to MAF behind a flag, compare on live traffic, and physically delete legacy code only once MAF has baked. Deleting before the soak would remove the very baseline we compare against, so deletion is the **last** phase.

---

## Decisions

1. **Stay on .NET 8.** MAF supports it; no framework bump.
2. **Both completion paths move to MAF.** `CompleteChat` becomes a single tool-enabled MAF `ChatClientAgent` — the natural MAF expression of today's model-driven `UseFunctionInvocation` loop — **not** a fixed workflow graph. `CompleteChatWithPlanning` keeps its workflow-graph *shape* but is **rebuilt** on the shared abstractions and may change freely — it is not production-proven (Decision 8). "More structured" means *the collaborators are extracted and injected*, not that simple chat becomes a graph.
3. **The global `UseMaf` flag is the cutover switch during the soak.** A single global toggle flips *all* completion (non-agentic and agentic) to MAF and back — no per-request routing. It is **deleted** in the final phase.
4. **Tool state lives in a per-request run-context, not side-channels.** The legacy `_currentClientUniqueId` / `_currentGptDeploymentParams` / `_currentExclusionFilter` instance fields ([ChatService.cs:1001-1003](../altus-ai-assistant/AltusAiAssistantApi/Services/ChatService.cs#L1001-L1003)) and the `HttpContext.Items["MessageContext"]` citation stash ([ChatService.cs:342](../altus-ai-assistant/AltusAiAssistantApi/Services/ChatService.cs#L342), [:1099](../altus-ai-assistant/AltusAiAssistantApi/Services/ChatService.cs#L1099)) are thread-unsafe workarounds. Tools instead read a `MafRunContext` and return citations through it ([AgenticWorkflowTypes.cs:14-31](../altus-ai-assistant/AltusAiAssistantApi/Services/Maf/AgenticWorkflowTypes.cs#L14-L31)).
5. **"Delete `ChatService`" means delete its *completion* responsibilities, not the whole class.** `IChatService`'s session/CRUD surface — `CreateSession`, `CreateFileSession`, `GetSessionsList`, `RestoreSession`, `DeleteSession`, `RenameSession`, `ResetChat`, `CreateCuratedMessage`, `GetFileDtosFromUrls` — is **not** part of this migration and stays. What gets deleted: `CompleteChat`, `CompleteChatWithPlanning` (the triage entry), `CompleteChatWithPlan` (the legacy planner, `#region Agentic`), the private tool methods, the instance fields, and the `HttpContext` stash. After Phase 4, `ChatService` is a session/CRUD-only service.
6. **Verify package IDs / adapter surface against the GA SDK** before coding. The agentic path already exercises `AsAIAgent` and `InProcessExecution.RunStreamingAsync`; confirm exact NuGet IDs and any surface changes hold for the non-agentic work.
7. **Legacy `ChatService` completion logic stays frozen until Phase 4.** From now until deletion, `CompleteChat`, `CompleteChatWithPlan` (the `#region Agentic` planner), the private tool methods, the instance fields, and the `HttpContext` stash receive **little-to-no changes** — they are the untouched fallback and the comparison baseline for the soak. The new abstractions are **additive**: extracting them creates new code that only `MafCompleteChatService` consumes; they are **never retrofitted into `ChatService`** (despite the duplication that leaves behind — that duplication is deliberate and temporary, resolved by deleting legacy wholesale in Phase 4). This keeps the fallback regression-free and the A/B clean: legacy behavior during the soak is byte-for-byte today's behavior. The **only** permitted edit to `ChatService` before Phase 4 is the dispatch branch in `CompleteChatWithPlanning` ([:1226-1243](../altus-ai-assistant/AltusAiAssistantApi/Services/ChatService.cs#L1226-L1243)) that delegates to the MAF service when `UseMaf` is on — the legacy method *bodies* are not edited.
8. **The legacy agentic planner is the production-proven baseline; the MAF agentic method is not, and may be rewritten.** `CompleteChatWithPlan` (the legacy planner, `#region Agentic`) **is already running in production** — it is the trustworthy behavioral baseline and stays frozen unchanged until Phase 4 (Decision 7). `CompleteChatWithPlanMaf` ([MafCompleteChatService.cs](../altus-ai-assistant/AltusAiAssistantApi/Services/MafCompleteChatService.cs)) was generated recently, is **unreviewed, untested, and has never run in production**. It is therefore **not** a baseline to preserve: Phase 0 may **rewrite it freely** onto the new abstractions rather than refactoring it "with no behavior change", keeping it only as a reference for MAF *mechanics* (workflow wiring, structured-output planning, `AsAIAgent` / `RunStreamingAsync` event flow). Correctness for both MAF methods is judged against the **legacy production behavior**, never against the MAF draft.

---

## The abstractions

These are the "necessary abstractions" the new structure is built on. Each consolidates logic that today lives as private helpers — duplicated between `ChatService` and `MafCompleteChatService` — into an injected collaborator that **both MAF completion methods** consume. The collaborators are **new code**: extracting them does **not** touch `ChatService`, which keeps its own private copies as the frozen baseline (Decision 7). The `ChatService.cs` line references below mark where each behavior *originates*, not edit targets. They are extracted first (Phase 0); because the MAF agentic method is not production-proven (Decision 8), Phase 0 **rewrites** it onto the collaborators rather than preserving it, validating the abstractions against the **legacy production** behavior — before any new behavior is written.

### 1. Retrieval / grounding — `IKnowledgeRetriever`
Owns everything that turns a query into citations and context text:
- `GetFromSearchIndex` (vector search + OData filter + scoring profile)
- `GetEmbedding`
- exclusion-filter application (`IContentExclusionService`)
- `GetRagContextString` (the `[doc1] (meta): content` rendering)
- `DetectWholeDocumentPull` + whole-document blob read ([MafCompleteChatService.cs:560-621](../altus-ai-assistant/AltusAiAssistantApi/Services/MafCompleteChatService.cs#L560-L621))

Reads a **run-context**, never instance fields. Consumed by: the non-agentic `SearchInternalDocuments` tool, the agentic `ProbeSearch` tool ([MafCompleteChatService.cs:252-259](../altus-ai-assistant/AltusAiAssistantApi/Services/MafCompleteChatService.cs#L252-L259)), the agentic search nodes, and the Join node's whole-doc pull.

### 2. Tool surface + run-context — `IChatToolProvider` over `MafRunContext`
A factory that produces the `AIFunction` set (`SearchInternalDocuments`, `SearchInternet`, `FetchWebPage`, `ProbeSearch`) **bound to a per-request `MafRunContext`**. The context carries `ClientUniqueId`, `GptParams`, `ExclusionFilter`, `IsFileSession`, and a **citation sink** the search tool writes into (replacing the `HttpContext.Items` stash). `SearchInternet` / `FetchWebPage` (the HTTP + HtmlAgilityPack fetcher, [MafCompleteChatService.cs:798-893](../altus-ai-assistant/AltusAiAssistantApi/Services/MafCompleteChatService.cs#L798-L893)) live here too. This lets the same tools serve both the model-driven loop and the workflow nodes.

### 3. System-prompt building — `IChatPromptBuilder`
Centralizes prompt engineering currently inlined in `ChatService.CompleteChat` ([:80-160](../altus-ai-assistant/AltusAiAssistantApi/Services/ChatService.cs#L80-L160)) and partially restated as `MafAgentInitialPrompt` ([MafCompleteChatService.cs:294-297](../altus-ai-assistant/AltusAiAssistantApi/Services/MafCompleteChatService.cs#L294-L297)):
- Altus Nova framing (`InitialPrompt`)
- file-session variant (uses `session.FilesContent`)
- client `CustomSystemPrompt`
- taxonomy (`ITaxonomyService`) and contacts (`IContactsPromptService`) blocks
- date, source-type / date-filter instructions, clarification, tool-preamble instructions
- the **synthesizer** framing used by the agentic path

Exposes builders for both shapes: the tool-using chat prompt and the no-tools synthesis prompt.

### 4. Session + turn persistence — `IChatTurnStore`
The single funnel for DB access, preserving the end-of-run-persist discipline:
- `GetSessionById` (with the auth / allowed-client check — [MafCompleteChatService.cs:655-676](../altus-ai-assistant/AltusAiAssistantApi/Services/MafCompleteChatService.cs#L655-L676))
- session auto-naming
- the **one** user + assistant persist with citation → `ChatMessageContext` mapping and `HtmlUrlDecode`
- the terminal `SendStreamCompleted(connectionId, sessionId, persistedId)` handshake

Every exit path (simple answer, agentic synthesis, clarify) calls this exactly once, so the "one user + one assistant per request, legacy schema" contract holds uniformly.

### 5. Deployment selection / failover — `IGptDeploymentSelector`
`GetNextGptDeploymentParams` (random pick, avoid the last-tried endpoint) plus the retry-on-failure policy from the non-agentic path ([ChatService.cs:391-403](../altus-ai-assistant/AltusAiAssistantApi/Services/ChatService.cs#L391-L403): up to 2 retries, a different endpoint each time). Both methods and the embedding call need a deployment; today the logic is copy-pasted in both services.

### 6. Streaming — `IChatStreamNotifier` + a routing policy
`IChatStreamNotifier` ([IChatStreamNotifier.cs](../altus-ai-assistant/AltusAiAssistantApi/Services/IChatStreamNotifier.cs)) is already the right abstraction. What differs per path is the **preamble-vs-response routing**, captured as a small policy/helper:
- **Agentic:** explicit-label preamble (`"Planning..."`, step labels, `"Synthesizing..."`); synthesizer text → `SendResponseChunk`.
- **Non-agentic:** the model-driven loop emits preamble text *before* the first tool call, cleared on the first `FunctionCallContent`; tool calls/results emit `SendToolCall` / `SendToolResult` ([ChatService.cs:252-303](../altus-ai-assistant/AltusAiAssistantApi/Services/ChatService.cs#L252-L303)). This "preamble-clearing dance" must be reproduced when driving the agent via MAF `RunStreamingAsync` by inspecting `FunctionCallContent` / `FunctionResultContent` in the update stream.

---

## Target composition

**`CompleteChat` (non-agentic), on MAF:**
```
IChatTurnStore.GetSession
  → IChatPromptBuilder.BuildChatPrompt(session)        // file vs standard, taxonomy, contacts, custom
  → IChatToolProvider.Create(runContext)               // omit SearchInternalDocuments for file sessions
  → ChatClientAgent(tools).RunStreamingAsync(...)       // model-driven; reasoning_effort via ChatOptions
       route updates: text → preamble/response (clearing dance), FunctionCall/Result → tool notifications
       citations accumulate in runContext sink
  → IChatTurnStore.PersistTurn(user, assistant, citations) + SendStreamCompleted
  on exception: IGptDeploymentSelector retry (≤2, new endpoint)
```

**`CompleteChatWithPlanning` (agentic), on MAF:** the existing Planner → Search → Join → Synthesizer graph ([MafCompleteChatService.cs:186-198](../altus-ai-assistant/AltusAiAssistantApi/Services/MafCompleteChatService.cs#L186-L198)), refactored so the nodes call `IKnowledgeRetriever` / `IChatPromptBuilder` and the host calls `IChatTurnStore` — deleting the duplicated privates at the bottom of the file ([:623-893](../altus-ai-assistant/AltusAiAssistantApi/Services/MafCompleteChatService.cs#L623-L893)).

**Triage / entry point:** the file/low/medium/high vs agentic routing currently in `ChatService.CompleteChatWithPlanning` ([:1226-1243](../altus-ai-assistant/AltusAiAssistantApi/Services/ChatService.cs#L1226-L1243)) moves into `MafCompleteChatService` (or a thin router in front of `ICompleteChatService`) so the completion surface is self-contained when legacy is removed. The controller ([ChatController.cs:43](../altus-ai-assistant/AltusAiAssistantApi/Controllers/ChatController.cs#L43)) keeps calling the same entry; only the implementation behind it changes.

---

## Migration phases

**Phase 0 — Extract abstractions and rebuild the MAF agentic method on them.**
Introduce the six collaborators, register them in DI, and **rewrite** the *existing MAF* `CompleteChatWithPlanning` onto them — removing the helpers duplicated at [MafCompleteChatService.cs:623-893](../altus-ai-assistant/AltusAiAssistantApi/Services/MafCompleteChatService.cs#L623-L893). Because the MAF agentic draft is **not production-proven** (Decision 8), this is a free rewrite, not a behavior-preserving refactor: its target behavior is the **legacy production** agentic path, and the draft is kept only as a mechanics reference (workflow wiring, structured-output planning, `AsAIAgent` / `RunStreamingAsync` event flow). **`ChatService` is not modified**: the legacy `CompleteChatWithPlan` stays frozen as the running baseline (Decision 7), so the same logic is briefly duplicated between it and the new collaborators — intentional, resolved by the Phase 4 deletion. The build stays green at every step.

**Phase 1 — Implement non-agentic `CompleteChat` on MAF.**
Single tool-enabled `ChatClientAgent` per the target composition: file-session handling, taxonomy/contacts, tool-call/result notifications, the preamble-clearing semantics, retry/failover. Dark-launched — reachable only when `UseMaf` is on.

**Phase 2 — Triage lives in the MAF service; gate the whole path on `UseMaf`.**
`MafCompleteChatService` implements its own file/low/medium/high vs agentic triage. `ChatService.CompleteChatWithPlanning` gets the one permitted edit (Decision 7): when `UseMaf` is on, delegate the *entire* request to the MAF service; when off, run today's legacy triage unchanged. So `UseMaf` on routes *all* completion through MAF, off keeps everything on the frozen legacy bodies. Parallel-run and compare on real traffic.

**Phase 3 — Production soak.**
Flip `UseMaf` on in production. Compare against the legacy baseline on citation quality, latency, and regressions on `clarify`, whole-doc-pull, file sessions, and tool-call UX. Keep the toggle reversible.

**Phase 4 — Delete legacy + the flag.**
Once baked, delete `ChatService.CompleteChat`, `CompleteChatWithPlanning`, `CompleteChatWithPlan` (`#region Agentic`), the private tools, the instance fields, the `HttpContext` stash, and the `UseMaf` flag. `ICompleteChatService` becomes the sole completion surface; `IChatService` retains only session/CRUD (Decision 5).

---

## Contracts the MAF methods must reproduce exactly

During the soak, MAF and legacy turns can interleave in one session, and both write the same SignalR surface and the same tables. The MAF methods must match these contracts or streamed UX / session history goes inconsistent.

**SignalR streaming** (per-request via `dto.SignalRConnectionId` + `sessionId`):
- **Preamble vs. response split.** Agentic: explicit labels → `SendPreambleChunk`, final answer → `SendResponseChunk`. Non-agentic: pre-tool text is preamble and is *cleared on the first tool call*; post-tool text is the response.
- **Tool-call / tool-result notifications** (non-agentic): `SendToolCall` on each `FunctionCallContent`, `SendToolResult` on each `FunctionResultContent` ([ChatService.cs:261-329](../altus-ai-assistant/AltusAiAssistantApi/Services/ChatService.cs#L261-L329)).
- **Terminal `SendStreamCompleted` with the real persisted message id** — the FE reconciles streamed text against the saved message by that id.

**DB persistence** (scoped `_dbContext`):
- Exactly **one user + one assistant message** per completed request, via `ChatMessage.CreateUserMessage` / `CreateAssistantMessage`, on every exit path (simple answer, synthesis, clarify) — persisted once, no double-write.
- **Legacy schema:** citations attached as `ChatMessageContext` children with `HtmlUrlDecode` of file/external URLs; history rebuilt from `session.ChatMessages.TakeLast(MaxContextMessages)`. Matching shape is what lets a MAF turn read a legacy turn and vice-versa.
- **`DbContext` is not thread-safe.** Search/join nodes stay DB-free; all persistence happens at the single end-of-run funnel (`IChatTurnStore`). (Relevant if parallel search lands later — see Non-goals.)

**Behavioral parity** also to preserve: GPT-deployment retry/failover (≤2 retries, switch endpoint), `reasoning_effort` via `ChatOptions.AdditionalProperties`, file-session behavior (no internal-search tool; `FilesContent` in the prompt), and taxonomy/contacts opt-ins (`dto.UseTaxonomy` / `dto.UseContactsContext`).

---

## Non-goals

- **Parallel source searches.** The graph's search nodes are independent (~4× retrieval speedup potential) but add `DbContext`-thread-safety and embedding-TPM concerns. Build sequentially first.
- **Evaluator → re-planning loop & checkpointing.** The disabled `EvaluateResults` and durable execution remain future work.
- **OpenTelemetry / observability.** Deferred.
- **Migrating `IChatService` session/CRUD methods.** Out of scope (Decision 5).
- **`altus-ai-assistant-voice`** (NestJS/TS — no first-class MAF TS SDK) and **sync workers** (deterministic ETL) — untouched.

## Risks / open questions

- **No automated eval coverage.** A curated prompt/eval set is needed to judge the soak; otherwise "proven in production" rests on manual spot-checks. This is the biggest risk to safe deletion in Phase 4.
- **Streaming fidelity (non-agentic).** Reproducing the preamble-clearing dance plus tool notifications on top of MAF `RunStreamingAsync` is the subtle part of Phase 1; getting it wrong regresses chat UX.
- **History compatibility during the soak.** Interleaved legacy/MAF turns must persist identically (handled by `IChatTurnStore`, but worth a targeted check).
- **Token usage.** The non-agentic path persists usage as hardcoded `0, 0` today ([ChatService.cs:361-362](../altus-ai-assistant/AltusAiAssistantApi/Services/ChatService.cs#L361-L362)). Decide whether the MAF rewrite preserves that or wires real usage from the MAF response.
- **Pre-GA authoring drift.** Confirm package IDs / adapter surface against the released SDK (Decision 6).
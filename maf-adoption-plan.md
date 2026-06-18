# Microsoft Agent Framework (MAF) Adoption — `altus-ai-assistant`

> **Status:** in progress — Phases 0–2 implemented; Phase 3 (production soak) live; Phase 4 (delete legacy) outstanding.
>
> **Scope:** primary backend only ([altus-ai-assistant](../altus-ai-assistant/)). Both chat-completion paths — non-agentic `CompleteChat` and agentic `CompleteChatWithPlanning` — now run on MAF inside `CompleteChatService`, built on a set of shared abstractions. The legacy completion logic in [ChatService.cs](../altus-ai-assistant/AltusAiAssistantApi/Services/ChatService.cs) is **still present** as the fallback/baseline and is **slated for deletion** in Phase 4 once the MAF paths are proven in production. The voice repo and sync workers are out of scope (see [§Non-goals](#non-goals)).

## Where things stand (2026-06)

The migration is substantially built. What the original draft framed as future work is now in the tree:

- The MAF service was renamed from `MafCompleteChatService` to **`CompleteChatService`** ([CompleteChatService.cs](../altus-ai-assistant/AltusAiAssistantApi/Services/CompleteChatService.cs)) implementing `ICompleteChatService`.
- **Both** completion methods are implemented on it. The non-agentic `CompleteChat` is no longer a `NotImplementedException` — it is a working tool-enabled model loop ([CompleteChatService.cs:45-109](../altus-ai-assistant/AltusAiAssistantApi/Services/CompleteChatService.cs#L45-L109)); the agentic `CompleteChatWithPlanning` is a MAF Workflow built from extracted executors ([CompleteChatService.cs:117-176](../altus-ai-assistant/AltusAiAssistantApi/Services/CompleteChatService.cs#L117-L176)).
- All six collaborators exist under [Services/Maf/](../altus-ai-assistant/AltusAiAssistantApi/Services/Maf/) and are DI-registered ([Program.cs:88-92](../altus-ai-assistant/AltusAiAssistantApi/Program.cs#L88-L92)).
- The workflow nodes are extracted as MAF `Executor<T>` / `Executor<T,U>` types under [Services/Maf/Executors/](../altus-ai-assistant/AltusAiAssistantApi/Services/Maf/Executors/), produced by `IMafExecutorFactory`.
- `GptDeploymentSelector` has been **deleted** (Decision 9); the single deployment is read from config and the retry is a plain inline ≤2 loop ([CompleteChatService.cs:55-108](../altus-ai-assistant/AltusAiAssistantApi/Services/CompleteChatService.cs#L55-L108)).
- `UseMaf` is **on** in both `appsettings.json` and `appsettings.Development.json` (the `AppSettings.UseMaf` C# default is still `false`), so the soak is effectively running.

What remains:

- **Phase 4 deletion** of the legacy completion logic in `ChatService` and the `UseMaf` flag.
- **Triage relocation** (a Phase 2 goal) is only partially done — see [§Deviations](#deviations-from-the-original-plan).

## Goal

`CompleteChatService` implements `ICompleteChatService` and is the single home for chat completion. The goal of the remaining work is to finish the soak and retire the legacy code:

1. ~~Implement **both** completion methods on the MAF service.~~ **Done.**
2. ~~Build them on **proper shared abstractions** rather than copy-pasted retrieval/tool helpers.~~ **Done** — the six collaborators below are extracted and injected.
3. **Soak in production** behind `UseMaf` (currently on), then **delete** the legacy completion logic in `ChatService`.

## Why "build, soak, then delete"

The agentic path has **no automated tests** today (the xUnit suite targets `HtmlContentHelper` only), and answer quality is judgeable only against real client corpora that don't exist outside production. So the safety strategy is a reversible toggle, not a test suite: keep the legacy path runnable, route to MAF behind a flag, compare on live traffic, and physically delete legacy code only once MAF has baked. Deleting before the soak would remove the very baseline we compare against, so deletion is the **last** phase.

---

## Decisions

1. **Stay on .NET 8.** MAF supports it; no framework bump. *(Held.)*
2. **Both completion paths run on MAF.** `CompleteChat` is a single tool-enabled model loop — the natural MAF expression of today's model-driven `UseFunctionInvocation` loop — **not** a fixed workflow graph. In implementation it drives a `FunctionInvokingChatClient` (the keyed `"notifying"` `IChatClient`) directly via `GetStreamingResponseAsync(...).ToChatResponseAsync()` rather than wrapping it in a `ChatClientAgent`; the effect is the same model-driven tool loop. `CompleteChatWithPlanning` keeps its workflow-graph shape (Planner → Search → Join → Synthesizer) and was rebuilt on the shared abstractions.
3. **The global `UseMaf` flag is the cutover switch during the soak.** A single global toggle flips *all* completion (non-agentic and agentic) to MAF and back — no per-request routing. It is **deleted** in the final phase. *(Currently on in config.)*
4. **Tool state lives in a per-request run-context, not side-channels.** The legacy `_currentClientUniqueId` / `_currentGptDeploymentParams` / `_currentExclusionFilter` instance fields ([ChatService.cs:1004-1006](../altus-ai-assistant/AltusAiAssistantApi/Services/ChatService.cs#L1004-L1006)) and the `HttpContext.Items["MessageContext"]` citation stash ([ChatService.cs:345](../altus-ai-assistant/AltusAiAssistantApi/Services/ChatService.cs#L345), [:1102](../altus-ai-assistant/AltusAiAssistantApi/Services/ChatService.cs#L1102)) are thread-unsafe workarounds. The MAF tools instead read a `MafRunContext` and return citations through its `LastSearchContext` sink ([AgenticWorkflowTypes.cs:7-21](../altus-ai-assistant/AltusAiAssistantApi/Services/Maf/AgenticWorkflowTypes.cs#L7-L21)). *(Done in the MAF path; the legacy fields remain in `ChatService` until Phase 4.)*
5. **"Delete `ChatService`" means delete its *completion* responsibilities, not the whole class.** `IChatService`'s session/CRUD surface — `CreateSession`, `CreateFileSession`, `GetSessionsList`, `RestoreSession`, `DeleteSession`, `RenameSession`, `ResetChat`, `CreateCuratedMessage`, `GetFileDtosFromUrls` — is **not** part of this migration and stays. What gets deleted in Phase 4: the legacy bodies of `CompleteChat` and `CompleteChatWithPlanning`, `CompleteChatWithPlan` (the legacy planner, `#region Agentic`), the private tool methods, the instance fields, the `HttpContext` stash, and `GetNextGptDeploymentParams` / `EvaluateResults`. After Phase 4, `ChatService` is a session/CRUD-only service.
6. **Verify package IDs / adapter surface against the GA SDK.** Confirmed in practice: the executors use the non-obsolete `Executor<T>` / `Executor<T,U>` API; the agentic host uses `InProcessExecution.RunStreamingAsync` + `WatchStreamAsync`; the non-agentic path uses `FunctionInvokingChatClient`. *(Largely settled by the working build; keep an eye on SDK drift — see Risks.)*
7. **Legacy `ChatService` completion logic stays frozen until Phase 4.** `CompleteChat` (legacy body), `CompleteChatWithPlan` (the `#region Agentic` planner), the private tool methods, the instance fields, and the `HttpContext` stash receive **little-to-no changes** — they are the untouched fallback and the comparison baseline for the soak. The new abstractions are **additive**: they live in their own files and are consumed only by `CompleteChatService`; they are **never retrofitted into `ChatService`** (the resulting duplication is deliberate and temporary, resolved by the Phase 4 deletion). The **only** permitted edits to `ChatService` are the two `UseMaf` dispatch branches that delegate to the MAF service ([ChatService.cs:69-70](../altus-ai-assistant/AltusAiAssistantApi/Services/ChatService.cs#L69-L70) and [:1243-1245](../altus-ai-assistant/AltusAiAssistantApi/Services/ChatService.cs#L1243-L1245)) — the legacy method *bodies* are not edited.
8. **The legacy agentic planner is the production-proven baseline; the MAF agentic method was rewritten freely.** `CompleteChatWithPlan` ([ChatService.cs:1256](../altus-ai-assistant/AltusAiAssistantApi/Services/ChatService.cs#L1256)) is the trustworthy behavioral baseline and stays frozen until Phase 4 (Decision 7). The MAF agentic method was not a baseline to preserve and was rewritten onto the new abstractions. Correctness for both MAF methods is judged against the **legacy production behavior**, never against the MAF draft.
9. **One GPT deployment; multi-deployment selection removed.** *(Done.)* `GptDeploymentSelector` and its random-pick + endpoint-failover are deleted from the MAF path. There is **one** deployment, read from `IOptions<AppSettings>` wherever a client is built, and the singleton `IChatClient` bound to it ([Program.cs:65-71](../altus-ai-assistant/AltusAiAssistantApi/Program.cs#L65-L71)) is correct by design. What survives is a **transient retry** — a ≤2-attempt inline loop against the same deployment for 429/5xx resilience ([CompleteChatService.cs:55-108](../altus-ai-assistant/AltusAiAssistantApi/Services/CompleteChatService.cs#L55-L108)) — not failover. The legacy `ChatService` still carries `GetNextGptDeploymentParams`, but with a single configured deployment it always returns that deployment and the "switch endpoint" branch never fires, so the divergence is theoretical, not observable in prod.

---

## The abstractions (as built)

All six collaborators are implemented under [Services/Maf/](../altus-ai-assistant/AltusAiAssistantApi/Services/Maf/) and registered in DI ([Program.cs:88-92](../altus-ai-assistant/AltusAiAssistantApi/Program.cs#L88-L92)). Each consolidates logic that still lives as private helpers in `ChatService` (the frozen baseline, Decision 7). The `ChatService.cs` line references mark where each behavior *originates*, not edit targets.

### 1. Retrieval / grounding — `IKnowledgeRetriever`
[IKnowledgeRetriever.cs](../altus-ai-assistant/AltusAiAssistantApi/Services/Maf/IKnowledgeRetriever.cs) / [KnowledgeRetriever.cs](../altus-ai-assistant/AltusAiAssistantApi/Services/Maf/KnowledgeRetriever.cs). Owns everything that turns a query into citations and context text: search-index query (vector + OData filter + scoring profile), embedding, exclusion-filter application, the `[doc1] (meta): content` RAG rendering, and whole-document detection + blob read (`DetectWholeDocumentPull` / `ReadWholeDocument`, [KnowledgeRetriever.cs:150-230](../altus-ai-assistant/AltusAiAssistantApi/Services/Maf/KnowledgeRetriever.cs#L150-L230)). Consumed by the tool provider's search tools, the `SearchExecutor`, the `JoinExecutor` (whole-doc pull), and the `SynthesizerExecutor`.

### 2. Tool surface + run-context — `IChatToolProvider` over `MafRunContext`
[IChatToolProvider.cs](../altus-ai-assistant/AltusAiAssistantApi/Services/Maf/IChatToolProvider.cs) / [ChatToolProvider.cs](../altus-ai-assistant/AltusAiAssistantApi/Services/Maf/ChatToolProvider.cs). Builds the per-request `MafRunContext` (`CreateRunContext`) and produces the `AIFunction` set bound to it (`CreateChatTools`). The context ([AgenticWorkflowTypes.cs:7-21](../altus-ai-assistant/AltusAiAssistantApi/Services/Maf/AgenticWorkflowTypes.cs#L7-L21)) carries `ClientUniqueId`, `ExclusionFilter`, `IsFileSession`, and the `LastSearchContext` citation sink the search tool writes into (replacing the `HttpContext.Items` stash). It carries no per-request GPT params — one deployment, read from config (Decision 9).

### 3. System-prompt building — `IChatPromptBuilder`
[IChatPromptBuilder.cs](../altus-ai-assistant/AltusAiAssistantApi/Services/Maf/IChatPromptBuilder.cs) / [ChatPromptBuilder.cs](../altus-ai-assistant/AltusAiAssistantApi/Services/Maf/ChatPromptBuilder.cs). Centralizes prompt engineering: Altus Nova framing, file-session variant (uses `session.FilesContent`), client `CustomSystemPrompt`, taxonomy + contacts blocks, date / source-type / date-filter / clarification / tool-preamble instructions, and the synthesizer framing used by the agentic path. Used by `CompleteChat` (`BuildChatPrompt`) and the planner/synthesizer executors.

### 4. Session + turn persistence — `IChatTurnStore`
[IChatTurnStore.cs](../altus-ai-assistant/AltusAiAssistantApi/Services/Maf/IChatTurnStore.cs) / [ChatTurnStore.cs](../altus-ai-assistant/AltusAiAssistantApi/Services/Maf/ChatTurnStore.cs). The single funnel for DB access, preserving the end-of-run-persist discipline: `GetSession` (with the auth / allowed-client check), `EnsureSessionNamed` (auto-naming), `BuildHistory` (last-N messages), and the **one** `PersistTurn` (user + assistant with citation → `ChatMessageContext` mapping and URL decode). Every exit path (simple answer, agentic synthesis, clarify) calls `PersistTurn` exactly once.

### 5. Transient retry (no deployment selection)
There is no `IGptDeploymentSelector`. The chat `IChatClient` is the singleton bound to the single deployment ([Program.cs:65-71](../altus-ai-assistant/AltusAiAssistantApi/Program.cs#L65-L71)). The surviving retry policy is a plain ≤2-attempt inline loop on the same deployment for transient 429/5xx inside `CompleteChat` ([CompleteChatService.cs:55-108](../altus-ai-assistant/AltusAiAssistantApi/Services/CompleteChatService.cs#L55-L108)), preserving the legacy retry count and the terminal `SendStreamError`.

### 6. Streaming — `IChatStreamNotifier` + `NotifyingChatClient`
The preamble-vs-response routing and tool notifications are implemented as a **decorating chat client** rather than an inline helper. [NotifyingChatClient.cs](../altus-ai-assistant/AltusAiAssistantApi/Services/Maf/NotifyingChatClient.cs) extends `FunctionInvokingChatClient` and is registered as the keyed `"notifying"` `IChatClient` ([Program.cs:69-71](../altus-ai-assistant/AltusAiAssistantApi/Program.cs#L69-L71)). A per-request `ToolNotificationSink` (notifier + connection + session) is flowed in via `ChatOptions.AdditionalProperties` under `"toolSink"`; when absent the client is a transparent pass-through.
- **Non-agentic:** `NotifyingChatClient` routes text before the first tool call to `SendPreambleChunk` and after to `SendResponseChunk` (the "preamble-clearing dance"), and emits `SendToolCall` / `SendToolResult` from `InvokeFunctionAsync`.
- **Agentic:** the executors stream explicit-label preamble (`"Planning..."`, step labels, `"Synthesizing..."`) and final synthesizer text to `IChatStreamNotifier` directly.

---

## Target composition (as implemented)

**`CompleteChat` (non-agentic):** ([CompleteChatService.cs:45-109](../altus-ai-assistant/AltusAiAssistantApi/Services/CompleteChatService.cs#L45-L109))
```
IChatTurnStore.GetSession + EnsureSessionNamed
  → IChatPromptBuilder.BuildChatPrompt(session, dto)   // file vs standard, taxonomy, contacts, custom
  → IChatToolProvider.CreateRunContext(session) + CreateChatTools(runContext)
  → ChatOptions { Tools, reasoning_effort, ToolNotificationSink }
  → notifying IChatClient.GetStreamingResponseAsync(...).ToChatResponseAsync()   // model-driven loop
       NotifyingChatClient streams preamble/response (clearing dance) + tool call/result notifications
       citations accumulate in runContext.LastSearchContext
  → IChatTurnStore.PersistTurn(user, assistant, citations) + SendStreamCompleted
  on exception: transient retry (≤2, same deployment) then SendStreamError
```

**`CompleteChatWithPlanning` (agentic):** ([CompleteChatService.cs:117-176](../altus-ai-assistant/AltusAiAssistantApi/Services/CompleteChatService.cs#L117-L176)) the Planner → Search → Join → Synthesizer graph, wired by `BuildAgenticWorkflow` from `IMafExecutorFactory`-produced executors, run via `InProcessExecution.RunStreamingAsync`. Nodes call `IKnowledgeRetriever` / `IChatPromptBuilder`; the host calls `IChatTurnStore`. Terminal clarify and synthesis branches both persist once via `PersistTurn`. State flows through `MafRunState` (deduped citations, clarify question, synthesized text).

**Triage / entry point:** **still in `ChatService`** ([ChatService.cs:1229-1246](../altus-ai-assistant/AltusAiAssistantApi/Services/ChatService.cs#L1229-L1246)). `CompleteChatWithPlanning` decides file-session / `UsePlanner` / low-medium-high vs agentic, then — only on the agentic branch — delegates to the MAF service when `UseMaf` is on. The low/medium/high branch routes through `ChatService.CompleteChat`, which delegates to MAF at its top ([ChatService.cs:69-70](../altus-ai-assistant/AltusAiAssistantApi/Services/ChatService.cs#L69-L70)). So all completion reaches MAF when `UseMaf` is on, but the *triage decision* has not yet moved into `CompleteChatService` (see Deviations).

---

## Deviations from the original plan

These are intentional or incidental differences between the draft plan and the built code, recorded so the plan and tree agree:

1. **Name.** `MafCompleteChatService` → `CompleteChatService` (the "Maf" prefix was dropped now that it is the only completion service).
2. **Non-agentic mechanism.** `CompleteChat` drives a `FunctionInvokingChatClient` (`NotifyingChatClient`) directly rather than constructing a `ChatClientAgent`. Same model-driven tool loop, fewer layers.
3. **Streaming as a decorator.** The "routing policy helper" became `NotifyingChatClient`, a `FunctionInvokingChatClient` subclass registered as a keyed `IChatClient`, with the per-request sink passed through `ChatOptions`. This is cleaner than threading a helper through call sites.
4. **Triage not yet relocated.** Phase 2 called for triage to live in the MAF service so the completion surface is self-contained at Phase 4. Today triage still lives in `ChatService.CompleteChatWithPlanning`. **This must be resolved before/within Phase 4** — when the legacy bodies are deleted, the file/`UsePlanner`/reasoning-tier routing has to move into `CompleteChatService` (or a thin router in front of `ICompleteChatService`), otherwise deleting `ChatService`'s completion methods removes the triage too.

---

## Migration phases

**Phase 0 — Extract abstractions and rebuild the MAF agentic method on them. ✅ Done.**
The six collaborators are introduced, DI-registered, and the agentic method is rebuilt on them; the duplicated helpers that previously sat at the bottom of `MafCompleteChatService` are gone. `ChatService`'s legacy planner stays frozen as the baseline.

**Phase 1 — Implement non-agentic `CompleteChat` on MAF. ✅ Done.**
Single tool-enabled model loop: file-session handling, taxonomy/contacts, tool-call/result notifications, preamble-clearing semantics, transient retry (≤2, same deployment). Reachable when `UseMaf` is on.

**Phase 2 — Gate the whole path on `UseMaf`. ◑ Mostly done.**
Both `ChatService` dispatch branches delegate to the MAF service when `UseMaf` is on. **Outstanding:** move the triage itself into the MAF service (see Deviation 4) so the completion surface is self-contained.

**Phase 3 — Production soak. ▶ In progress.**
`UseMaf` is on in config. Compare against the legacy baseline on citation quality, latency, and regressions on `clarify`, whole-doc-pull, file sessions, and tool-call UX. Keep the toggle reversible.

**Phase 4 — Delete legacy + the flag. ☐ Outstanding.**
Once baked: delete the legacy bodies of `ChatService.CompleteChat` / `CompleteChatWithPlanning`, `CompleteChatWithPlan` (`#region Agentic`), the private tools, instance fields, `HttpContext` stash, `GetNextGptDeploymentParams`, `EvaluateResults`, and the `UseMaf` flag (including the `appsettings` entries and `AppSettings.UseMaf`). Relocate triage first (Deviation 4). `ICompleteChatService` becomes the sole completion surface; `IChatService` retains only session/CRUD (Decision 5).

---

## Contracts the MAF methods must reproduce exactly

During the soak, MAF and legacy turns can interleave in one session, and both write the same SignalR surface and the same tables. The MAF methods must match these contracts or streamed UX / session history goes inconsistent.

**SignalR streaming** (per-request via `dto.SignalRConnectionId` + `sessionId`):
- **Preamble vs. response split.** Agentic: explicit labels → `SendPreambleChunk`, final answer → `SendResponseChunk`. Non-agentic: pre-tool text is preamble and is *cleared on the first tool call*; post-tool text is the response (handled by `NotifyingChatClient.RouteText`).
- **Tool-call / tool-result notifications** (non-agentic): `SendToolCall` on each function call, `SendToolResult` on each result (handled in `NotifyingChatClient.InvokeFunctionAsync`).
- **Terminal `SendStreamCompleted` with the real persisted message id** — the FE reconciles streamed text against the saved message by that id.

**DB persistence** (scoped `_dbContext`):
- Exactly **one user + one assistant message** per completed request, via `IChatTurnStore.PersistTurn`, on every exit path (simple answer, synthesis, clarify) — persisted once, no double-write.
- **Legacy schema:** citations attached as `ChatMessageContext` children with URL decode; history rebuilt from the last `MaxContextMessages`. Matching shape is what lets a MAF turn read a legacy turn and vice-versa.
- **`DbContext` is not thread-safe.** Search/join executors stay DB-free; all persistence happens at the single `IChatTurnStore` funnel.

**Behavioral parity** also to preserve: transient retry (≤2, same deployment — Decision 9), `reasoning_effort` via `ChatOptions.AdditionalProperties`, file-session behavior (no internal-search tool; `FilesContent` in the prompt), and taxonomy/contacts opt-ins (`dto.UseTaxonomy` / `dto.UseContactsContext`).

---

## Non-goals

- **Parallel source searches.** The graph's search nodes are independent (~4× retrieval speedup potential) but add `DbContext`-thread-safety and embedding-TPM concerns. Sequential first.
- **Evaluator → re-planning loop & checkpointing.** `EvaluateResults` remains disabled/commented; durable execution is future work.
- **OpenTelemetry / observability.** Deferred.
- **Migrating `IChatService` session/CRUD methods.** Out of scope (Decision 5).
- **`altus-ai-assistant-voice`** (NestJS/TS — no first-class MAF TS SDK) and **sync workers** (deterministic ETL) — untouched.

## Risks / open questions

- **No automated eval coverage.** A curated prompt/eval set is needed to judge the soak; otherwise "proven in production" rests on manual spot-checks. This is the biggest risk to safe deletion in Phase 4.
- **Triage relocation before deletion.** Phase 4 cannot simply delete `ChatService`'s completion methods while triage still lives there (Deviation 4). Sequence this.
- **Streaming fidelity (non-agentic).** The preamble-clearing dance plus tool notifications now live in `NotifyingChatClient`; verify it matches legacy UX exactly during the soak.
- **History compatibility during the soak.** Interleaved legacy/MAF turns must persist identically (handled by `IChatTurnStore`, but worth a targeted check).
- **Token usage.** The non-agentic legacy path persisted usage as hardcoded `0, 0`. Confirm what the MAF `PersistTurn` writes and whether to wire real usage from the MAF response.
- **Pre-GA / SDK drift.** Keep confirming package IDs / adapter surface against the released SDK as MAF evolves.

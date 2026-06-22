# Chat Feature — Functional Improvement Proposals

> **Status:** proposal — none implemented yet (drafted 2026-06-22).
>
> **Scope:** primary backend only ([altus-ai-assistant](../altus-ai-assistant/)), the MAF-based chat completion path in [CompleteChatService.cs](../altus-ai-assistant/AltusAiAssistantApi/Services/CompleteChatService.cs) and its collaborators under [Services/Maf/](../altus-ai-assistant/AltusAiAssistantApi/Services/Maf/). Builds on the migration tracked in [maf-adoption-plan.md](maf-adoption-plan.md); does not supersede it. Voice repo and sync workers are out of scope.

## Context

The chat feature has two completion paths, both running on MAF inside `CompleteChatService`:

- **Non-agentic** — [`CompleteChat`](../altus-ai-assistant/AltusAiAssistantApi/Services/CompleteChatService.cs#L45): a single tool-enabled model loop with an inline ≤2 retry. Live SignalR streaming (tool calls, preamble-vs-response split) is emitted as a side effect by [NotifyingChatClient](../altus-ai-assistant/AltusAiAssistantApi/Services/Maf/NotifyingChatClient.cs).
- **Agentic** — [`CompleteChatWithPlanning`](../altus-ai-assistant/AltusAiAssistantApi/Services/CompleteChatService.cs#L131): a linear MAF Workflow, Planner → Search → Join → Synthesizer ([executors](../altus-ai-assistant/AltusAiAssistantApi/Services/Maf/Executors/)).

Routing between them is decided by triage in [ChatService.CompleteChatWithPlanning](../altus-ai-assistant/AltusAiAssistantApi/Services/ChatService.cs#L1233) (`agentic` reasoning + `UsePlanner` → agentic; otherwise non-agentic; file sessions always non-agentic).

The proposals below are grouped by theme and ordered by impact. Each notes the observed current behavior, the proposed change, and the affected location.

---

## 1. Resilience & error surfacing — highest impact

Current behavior here can silently lose answers or waste latency/cost.

### 1.1 Agentic path can persist an empty answer and report success
The event loop in `CompleteChatWithPlanning` only *logs* `ExecutorFailedEvent`/`WorkflowErrorEvent` ([CompleteChatService.cs:155-158](../altus-ai-assistant/AltusAiAssistantApi/Services/CompleteChatService.cs#L155-L158)). If the synthesizer throws (e.g. Azure OpenAI 429), `runState.SynthesizedText` stays empty and the code still persists an empty assistant message and sends `SendStreamCompleted` ([CompleteChatService.cs:180-185](../altus-ai-assistant/AltusAiAssistantApi/Services/CompleteChatService.cs#L180-L185)). The user gets a blank turn saved to history.

**Proposed:** detect workflow failure / empty synthesis → call `SendStreamError`, and do **not** persist the empty turn.

### 1.2 Resilience is inconsistent between the two paths
`CompleteChat` has a retry loop ([CompleteChatService.cs:61-121](../altus-ai-assistant/AltusAiAssistantApi/Services/CompleteChatService.cs#L61-L121)); the agentic path has none. A transient fault in the planner falls back to a broad search (acceptable), but the same fault in the synthesizer produces the empty answer from §1.1.

**Proposed:** apply equivalent transient-retry handling to the synthesizer (and other agentic LLM calls).

### 1.3 Retry loop catches everything and can duplicate streamed text
- The `catch` ([CompleteChatService.cs:112](../altus-ai-assistant/AltusAiAssistantApi/Services/CompleteChatService.cs#L112)) catches *all* exceptions, so a permanent fault (bad request, auth, a future `OperationCanceledException`) burns up to 3× latency before failing deterministically.
- On retry, anything already streamed via `NotifyingChatClient` was sent to the client, but no reset is emitted before re-streaming — a retry duplicates partial text in the UI.

**Proposed:** classify transient (timeout/429/503) vs. permanent and only retry the former; either gate the retry on "nothing streamed yet" or emit a reset chunk before re-streaming.

### 1.4 No cancellation
Neither method takes a `CancellationToken`, and the controller doesn't pass `HttpContext.RequestAborted` ([ChatController.cs:40](../altus-ai-assistant/AltusAiAssistantApi/Controllers/ChatController.cs#L40)). When a user aborts or navigates away, the model loop / workflow keeps running (and billing).

**Proposed:** thread a `CancellationToken` from the controller through `GetStreamingResponseAsync` and `InProcessExecution.RunStreamingAsync` down into the executors and retriever.

---

## 2. Retrieval quality (RAG)

### 2.1 Single linear pass, no reflection
The graph is strictly Planner → Search → Join → Synthesizer ([BuildAgenticWorkflow](../altus-ai-assistant/AltusAiAssistantApi/Services/CompleteChatService.cs#L191-L204)). When searches come back weak, it still synthesizes "no relevant documents found" rather than re-querying with broader terms or relaxed filters. (The legacy code had an evaluator that "logs only — always proceeds.")

**Proposed:** add an evaluate/critique node that can loop back to the planner (corrective RAG). MAF supports cyclic graphs. Highest answer-quality lever.

### 2.2 Plan steps run sequentially
[SearchExecutor](../altus-ai-assistant/AltusAiAssistantApi/Services/Maf/Executors/SearchExecutor.cs#L42) `foreach`-awaits each step. A 4-source plan pays 4× latency for independent calls.

**Proposed:** run independent steps concurrently (`Task.WhenAll`, or MAF fan-out/fan-in). Very visible perceived-latency win.

### 2.3 URL-only dedup, no reranking or cap
[JoinExecutor:37-41](../altus-ai-assistant/AltusAiAssistantApi/Services/Maf/Executors/JoinExecutor.cs#L37-L41) keeps the first citation per `FilePath` and nothing else — no cross-source reranking, no relevance-score threshold, no top-K cap before synthesis. A many-source plan can flood the synthesizer with marginal docs (token cost + diluted answer).

**Verify:** if the index is chunked and chunks share a document URL, this dedup collapses multiple useful chunks of one document down to one, silently dropping retrieved content.

**Proposed:** rerank across sources by score/freshness, apply a score threshold, and cap to top-K before synthesis. Resolve the chunk-vs-document dedup semantics first.

### 2.4 Whole-document detection is a separate LLM round-trip
[DetectWholeDocumentPull](../altus-ai-assistant/AltusAiAssistantApi/Services/Maf/KnowledgeRetriever.cs#L150) runs on every agentic turn that has citations — extra latency/cost. The planner already reasons about intent.

**Proposed:** fold the whole-document decision into the planner's structured output to remove a hop.

---

## 3. Conversation context

### 3.1 Hard truncation loses the middle
[BuildHistory](../altus-ai-assistant/AltusAiAssistantApi/Services/Maf/ChatTurnStore.cs#L59-L83) keeps the first message + last `MaxContextMessages` and drops everything between, with no summary. Long sessions lose context the user assumes is remembered.

**Proposed:** keep a rolling summary of dropped turns, and/or budget by tokens (SharpToken is already in use) instead of a fixed message count.

### 3.2 Session naming uses the raw first message
[EnsureSessionNamed](../altus-ai-assistant/AltusAiAssistantApi/Services/Maf/ChatTurnStore.cs#L50-L57) sets the session name to the verbatim first user message.

**Proposed:** generate a short LLM title for the session list. Small but visible UX win.

---

## 4. Performance affecting responsiveness

In [KnowledgeRetriever](../altus-ai-assistant/AltusAiAssistantApi/Services/Maf/KnowledgeRetriever.cs):

- Every search uses the **synchronous** `client.Search<>` ([L86](../altus-ai-assistant/AltusAiAssistantApi/Services/Maf/KnowledgeRetriever.cs#L86)).
- `GetEmbedding` constructs a **new `AzureOpenAIClient` per call** and calls the **synchronous** `GenerateEmbedding` inside an `async` method ([L136-148](../altus-ai-assistant/AltusAiAssistantApi/Services/Maf/KnowledgeRetriever.cs#L136-L148)); the embedding model is hardcoded.

These are blocking calls on request threads; once plan steps are parallelized (§2.2) the thread-pool pressure worsens.

**Proposed:** switch to `SearchAsync` / `GenerateEmbeddingAsync`, reuse a cached embedding client, and source the embedding model from config.

---

## 5. Feature parity between the two paths

### 5.1 Non-agentic path lacks whole-document pull
The agentic path pulls whole documents into context when a question targets a specific file ([JoinExecutor](../altus-ai-assistant/AltusAiAssistantApi/Services/Maf/Executors/JoinExecutor.cs#L46-L73)); `CompleteChat` does not. A "summarize this document" query routed to the non-agentic branch (low/medium/high reasoning) silently gets weaker behavior.

**Proposed:** expose whole-document pull as a tool on the non-agentic path, or share the retriever's whole-doc logic.

### 5.2 Reasoning effort hardcoded in the agentic path
The non-agentic path honors `dto.Reasoning`; the agentic planner, synthesizer, and whole-doc detection hardcode `reasoning_effort = "high"` ([SynthesizerExecutor:63](../altus-ai-assistant/AltusAiAssistantApi/Services/Maf/Executors/SynthesizerExecutor.cs#L63), [PlannerExecutor:91](../altus-ai-assistant/AltusAiAssistantApi/Services/Maf/Executors/PlannerExecutor.cs#L91), [KnowledgeRetriever:179](../altus-ai-assistant/AltusAiAssistantApi/Services/Maf/KnowledgeRetriever.cs#L179)), so cost/latency isn't tunable.

**Proposed:** make per-node reasoning effort configurable.

---

## 6. Clarify round-trip — request-port + optional durable checkpointing

### 6.1 Current behavior
When the planner picks `Clarify`, [PlannerExecutor](../altus-ai-assistant/AltusAiAssistantApi/Services/Maf/Executors/PlannerExecutor.cs#L50-L57) sets `_state.ClarifyQuestion` and returns **without** sending `PlanReady`, so the workflow terminates. The host streams the question, persists it as a normal assistant message, and returns ([CompleteChatService.cs:167-177](../altus-ai-assistant/AltusAiAssistantApi/Services/CompleteChatService.cs#L167-L177)). When the user replies, `ChatController` calls `CompleteChatWithPlanning` again → a **brand-new workflow**, new planner, new ProbeSearch.

Nothing is lost (the clarify Q and the user's answer are both in history via [BuildHistory](../altus-ai-assistant/AltusAiAssistantApi/Services/Maf/ChatTurnStore.cs#L59), so the follow-up plan is informed), but the ProbeSearch calls and one high-reasoning planner LLM call are **redone**. Conceptually, clarify is modeled as a termination rather than a pause. The inline comment at [CompleteChatService.cs:166](../altus-ai-assistant/AltusAiAssistantApi/Services/CompleteChatService.cs#L166) already flags this: *"To be improved via durable workflow execution and checkpoints."*

### 6.2 What MAF 1.9.0 offers
Two relevant primitives (currently unused in the repo; verify exact signatures against the installed `Microsoft.Agents.AI.Workflows` 1.9.0 package):

- **Request/response ports (human-in-the-loop).** Instead of terminating, an executor sends a request to an input/request port; the run suspends and emits a `RequestInfoEvent`; the host resumes the *same* run by supplying the response. This is the idiomatic representation of clarify — one logical conversation that pauses, not two disjoint runs.
- **Checkpointing.** A `CheckpointManager` snapshots workflow state at superstep boundaries and resumes from a checkpoint id — this is what gives a suspended run durability across the HTTP round-trip.

Combined: clarify becomes a request-port suspend, the suspended run is checkpointed to a durable store keyed by session, and the user's reply resumes it after the planner instead of restarting.

### 6.3 The two catches
- **Run state is not in MAF's managed state.** `MafRunState` / `MafRunContext` are created in the host and passed into executors via the factory closure ([CompleteChatService.cs:147-148](../altus-ai-assistant/AltusAiAssistantApi/Services/CompleteChatService.cs#L147-L148)) — captured fields, not workflow state. MAF checkpointing will not capture or restore closure-held objects; that state would have to move into the workflow's `IWorkflowContext` state APIs to round-trip through a checkpoint. This is the bulk of the refactor.
- **Multi-instance deployment defeats in-memory suspension.** On Azure App Service (scaled out, instances recycle), a live run object can't be held in memory between requests — the follow-up may land on a different instance, and recycles drop it. A durable, shared checkpoint store (Blob/Table/SQL) is effectively mandatory, not optional.

### 6.4 Recommendation
For clarify **specifically**, full durable checkpoint/resume is likely over-engineering: clarify terminates *before* any expensive search runs (only ProbeSearch + one planner call happened), the redone work is small, and re-planning with the new answer is arguably desirable. The cheap 80% win is to **stash the probe results (and/or draft plan) in durable session-scoped state and skip re-probing on the follow-up** — no workflow-state refactor needed.

Checkpointing earns its complexity elsewhere: **resuming a run that failed mid/late** (e.g. synthesizer crash after an expensive multi-source search — see §1.1 / §1.2) or long multi-step search runs, where the work worth preserving has already been paid for. Treat request-port clarify and durable checkpointing as a single larger initiative, justified by those cases rather than by clarify alone.

---

## Suggested sequencing

Highest value-to-effort, mostly localized:

1. **§1.1 + §1.2** — agentic error surfacing / no-persist-on-empty (correctness, prevents data loss).
2. **§2.2** — parallelize plan steps (latency, very visible).
3. **§4** — async retriever calls (enables §2.2 to scale safely).
4. **§1.3 + §1.4** — retry classification and cancellation.
5. **§6.4 (lightweight)** — cache probe results in session state to avoid re-probing on the clarify follow-up; cheap, no workflow-state refactor.
6. **§2.1** — reflection loop (largest quality gain, largest effort; design carefully).
7. **§6 (full)** — request-port clarify + durable checkpointing; largest infra effort, justify via mid/late-run resume (§1.1/§1.2), not clarify alone.
8. Remaining context/parity items as capacity allows.
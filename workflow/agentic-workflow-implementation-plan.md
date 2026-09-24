# Agentic workflow — iterative implementation plan

**Status:** Proposed

**Design:** [Agentic reflection loop and durable checkpoints](agentic-reflection-and-checkpoints-design.md)

## Scope and delivery approach

Implement the decision loop and durable AskUser continuation in small, reviewable increments. Start with recoverable execution, then add bounded reflection and connect the complete user experience. Keep incomplete behavior behind an agentic-mode feature flag until the acceptance criteria are met.

The primary backend owns orchestration, persistence, and continuation contracts. The primary frontend owns clarification replies, progress, and restoring run status. Deployment repositories are involved only where storage configuration, migrations, or rollout settings require changes. Non-agentic chat and the voice stack remain outside this plan.

Keep subsequent workflow designs and implementation notes in this folder. Each iteration should leave a working, testable increment; early backend capabilities may be exercised through integration tests before the frontend is ready.

## Iteration 1 — Confirm execution model and contracts

**Outcome:** Resolve the framework and lifecycle decisions needed before implementation expands.

- Verify checkpointing, external request/response handling, and cross-instance recovery against the installed Microsoft Agent Framework version. Build a small technical proof of save, process restart, and resume.
- Select durable shared storage and define the boundaries between workflow state and conversation persistence.
- Agree on run identity, lifecycle states, clarification identity, and how clients distinguish a reply to a pending question from a new request.
- Define checkpoint compatibility and feature-flag behavior for runs already in progress.
- Capture a small baseline of representative questions, answer quality, latency, and search/model usage for later comparison.

**Exit criteria:** The recovery proof succeeds, any framework upgrade or adapter requirement is understood, and the backend/frontend continuation contract is recorded. Detailed schemas and API choices belong in a short follow-up design.

## Iteration 2 — Persist and recover the existing workflow

**Outcome:** The current linear workflow gains durable execution before its routing changes.

- Introduce persisted run state and checkpoints, including request context, completed work, evidence, prepared synthesis context, and usage.
- Recover from the last committed boundary rather than automatically restarting planning and search.
- Coordinate run transitions with conversation persistence so recovery cannot duplicate saved exchanges or report an empty successful answer after failure.
- Enforce one active execution per run, preserve client scoping, and establish basic run/stage observability.

**Exit criteria:** A stopped run resumes on another instance with completed retrieval retained. Duplicate execution attempts do not create concurrent continuations or duplicate messages. Existing successful agentic behavior remains intact with the feature disabled.

## Iteration 3 — Add assessment and durable clarification

**Outcome:** User input becomes a resumable part of the workflow.

- Introduce the Evaluator at entry and after search. Initially allow the first search, clarification, and synthesis; keep additional search passes disabled until the next iteration.
- Allow the Evaluator and Synthesizer to request clarification through the same durable pause mechanism.
- Save the pending question before exposing it, accept an explicitly associated reply, and resume assessment with the clarified request and saved evidence.
- Recheck access and applicable content permissions on continuation. Handle repeated replies, late replies, and unavailable checkpoints explicitly.

**Exit criteria:** Clarification works before search and during synthesis. A reply supported by saved evidence reaches an answer without repeating retrieval, including after restart. A reply requiring more evidence is recognized; additional retrieval remains gated until iteration 4.

## Iteration 4 — Enable bounded search refinement

**Outcome:** Assessment can request targeted additional retrieval when evidence is insufficient.

- Enable Evaluator → Planner routing with actionable feedback and accumulated evidence available to planning.
- Enforce the initial search pass plus **at most two returns to Planner** in the workflow controller. Persist the counter across clarification and recovery; stage retries do not consume another return.
- Preserve useful earlier results, avoid equivalent searches that make no progress, and refresh Join output when evidence or intent changes.
- When no useful search remains or the budget is exhausted, synthesize with explicit evidence limitations.

**Exit criteria:** Tests cover zero, one, and two returns; a third return is prevented. Clarification can trigger a targeted search within the remaining budget, cannot reset that budget, and preserves earlier evidence. Authorization and content filters hold across every pass.

## Iteration 5 — Complete the chat experience

**Outcome:** Users can follow and resume the workflow through the application.

- Connect the frontend to run status and clarification identity, distinguishing Waiting for user from Completed.
- Display concise progress for assessment, search, refinement, and synthesis. Keep late clarification coherent with any previously streamed draft text.
- Restore pending questions and run status after reload or reconnect, attaching to the existing run rather than starting another.
- Support an explicit new request separately from a pending clarification, with clear cancellation, failure, and unavailable-checkpoint behavior.

**Exit criteria:** End-to-end checks cover clarification before retrieval, clarification requiring another search, exhausted budget, reload/reconnect, and duplicate submission. Each exchange appears once in conversation history.

## Iteration 6 — Validate recovery and roll out

**Outcome:** Enable the feature based on demonstrated reliability and answer quality.

- Exercise failures around checkpoint saves, clarification delivery, reply acceptance, and final-answer persistence, including concurrent continuation attempts and permission changes.
- Finalize checkpoint retention, abandoned-run cleanup, execution budgets, and deployment compatibility for saved runs.
- Compare the representative evaluation set with the baseline, including answer grounding, unnecessary searches/questions, latency, and cumulative usage.
- Roll out to a limited audience, monitor results, and widen availability when agreed criteria are met. Define how existing runs finish or are explicitly stopped if new-run creation is disabled during rollback.

**Exit criteria:** Design acceptance scenarios pass, operational limits and recovery behavior are verified, and rollout evidence supports broader enablement. Schema migration and environment changes follow the affected repository's delivery process.

## Validation throughout

Use deterministic routing and persistence tests for budgets, pause/resume, concurrency, and failure recovery. Use representative model-driven evaluations for decision quality and answer usefulness; model judgments alone must not enforce execution limits.

Run backend checks for the affected components and frontend build/manual checks when the UI changes. Review diffs separately by repository and record validation with each increment. The final release requires the full design's acceptance criteria even if earlier iterations pass their narrower checks.

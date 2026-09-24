# Prompt Improvement (ANA-157)

> **Status:** implemented, unmerged (updated 2026-09-03). Backend on `feature/ANA-157-improve-prompts`
> (commits `b1987a9`, `7398d92`, `743458d`, `a863216`, `e82018e`), frontend on `feature/ANA-157-view-prompt`.
> The admin prompt page and the runtime framing override — both listed as deferred in earlier revisions of this
> doc — shipped. What is left is content work and a second evaluation pass, tracked in §7.

# Requirements
See https://altusnova.atlassian.net/wiki/spaces/ANA/pages/5059575810/Prompt+Considerations

# Test data
See https://altusnova.atlassian.net/wiki/spaces/ANA/pages/5064818690/Inquiry+Examples+-+Banking

# Test cases
Test cases are documented here:
https://altusnova.atlassian.net/wiki/spaces/ANA/pages/5074518043/Prompt+Improvement+-+Test+Requests+and+Expected+Behavior

# Current implementation

Prompt text lives in `AltusAiAssistantApi/Prompts` as `.prompt` files, embedded as resources and rendered
through `PromptTemplateProvider` (Scriban, strict variables). Whole prompts sit at the folder root
(`chat-system`, `chat-file-session`, `agentic-planner-system`, `agentic-synthesis-system`); reusable blocks sit
under `sections/`. The section inventory was re-cut afterwards by
[customizable-prompt-sections.md](customizable-prompt-sections.md); the table below reflects the result.

`ChatPromptBuilder` renders the section files first and passes the results in as placeholder values — templates
do not include each other. Framing is the exception: it goes through `IFramingPromptService`, which prefers a
global setting and falls back to the shipped file (§1). `ContactsPromptService` renders the directory header the
same way. Golden-file tests in `AltusAiAssistantApi.Tests` lock every assembled prompt, so any wording change
shows up as a reviewable diff.

Which section reaches which stage:

| Section | Chat | File session | Planner | Synthesis |
| --- | --- | --- | --- | --- |
| `framing` | ✔ | ✔ | — | ✔ |
| `knowledge-sources` | ✔ | — | ✔ | ✔ |
| `external-sources` | ✔ | — | ✔ | — |
| `citations` | ✔ | — | — | ✔ |
| `search-tool-usage` | ✔ | ✔ | — | — |
| `ambiguity` | ✔ | — | ✔ | — |
| `restate-and-disclose` | ✔ | — | — | ✔ |
| `wording` | ✔ | — | ✔ | ✔ |
| `tool-preamble` | ✔ | ✔ | — | — |
| `custom-client` | ✔ | — | ✔ | ✔ |
| `taxonomy`, contacts directory | ✔ | ✔ | — | ✔ |

The grouping rule is in a comment on `ChatPromptBuilder`: a stage gets `ambiguity` if it can ask questions, and
`restate-and-disclose` if it writes the answer. `wording` (audience included — see
[customizable-prompt-sections.md](customizable-prompt-sections.md) §3.4) spans both, because the planner writes
user-facing sentences on the clarify path. `external-sources` goes to the stages that can search; `citations` to
the stages that write an answer over retrieved documents.

`chat-file-session.prompt` was excluded from this work. File sessions are slated for deprecation, so that prompt
keeps its own copy of the clarification and citation lines and got none of the new content; it picks up the framing
override only because it shares `BuildChatPrompt`. The later re-cut moved it onto the shared `search-tool-usage` and
`tool-preamble` sections, but only because those two blocks were byte-identical to the chat prompt's copies. Its
golden file is byte-identical to before either change, which is the check that nothing leaked into it.

# What the requirements asked for, and what shipped

| Requirement | Outcome |
| --- | --- |
| Up to 3 clarifying questions, iteratively, until ambiguity is gone | `sections/ambiguity.prompt`, rendered into chat and planner. `PlannerExecutor` caps the emitted list at 3. |
| Always paraphrase & restate the request for confirmation before processing | `sections/restate-and-disclose.prompt` — restatement as a lead-in sentence, not a separate turn (§2). On the agentic clarify path the restatement rides on the `Clarify` step. |
| Treat the current prompt as authoritative, other context as background | First paragraph of `sections/ambiguity.prompt`. |
| Human-friendly wording, sparing jargon, executive (not technocrat) audience | `sections/wording.prompt`, rendered into chat, planner and synthesis. |
| Client/industry-specific source augmentation | `Client.CustomSystemPrompt` now reaches the planner as well as chat and synthesis (§3), so a client source list can actually steer what gets searched. |

The conflict called out during design — the planner allowing `Clarify` only when `ProbeSearch` came back empty —
was resolved in favour of the shared contract: `ProbeSearch` returning documents no longer blocks clarification,
because it tells the planner what exists, not which reading the user meant. `Clarify` remaining the only step in
a clarifying plan is unchanged; that is a workflow constraint, not a policy one.

# Implementation

## 1. What stays out of the prompt files

| Kind | Home |
| --- | --- |
| Behavioral instructions — clarify contract, restate/disclose rules, wording rules, planner step rules | `.prompt` files in the API project, git-reviewed |
| Instance identity — the framing block | Global setting `prompts.framing`, overriding the shipped `sections/framing.prompt` |
| Tenant content — banking source list, contacts directory, taxonomy | Database, per client (`Client.CustomSystemPrompt`, contacts, taxonomy) |

**No admin editor for behavioral instructions.** Runtime-editable behavior would make the assistant behave
differently in QA and prod, put the real prompt in a row that never gets code-reviewed or diffed, and make bug
reports unreproducible. The output of behavior tuning is a commit.

Two blocks sit deliberately on the other side of that line, and only two:

- `prompts.contacts.directoryHeader` — pre-existing, low-stakes (a header above a data block).
- `prompts.framing` — added by [framing-prompt-override.md](framing-prompt-override.md). Framing is *instance
  identity*, not behavior: under the instance-per-customer model each deployment is supposed to say something
  different about who the assistant is, and the alternative to a setting is a per-instance fork of the one
  `.prompt` file guaranteed to differ. The reproducibility argument above does not apply, and the override is
  visible in the effective-prompt view either way.

Both resolve the same way — whitespace-only setting counts as unset and falls back to the shipped template, so
saving blank is the reset path — and both cap the stored value at 4000 characters.

## 2. Clarify, restate and wording

Three section files, written from the Confluence text:

**`sections/ambiguity.prompt`** (replaced the one-question `## Clarification` block in `chat-system.prompt`):
- The user's current message is the authoritative task; history, taxonomy, contacts and retrieved docs are supporting background.
- Ambiguity means multiple reasonable readings, open scope (which project, client, period, topic), or internal inconsistency. When ambiguous, ask up to 3 focused questions and stop — do not guess.
- Lead with your reading of the request when asking, so the user can correct a misreading rather than only answering the questions.
- Ask only for what would change the answer; a question costs a round trip. Re-assess on each reply and keep asking 1–3 at a time until the remainder is immaterial or the user says to proceed.

**`sections/restate-and-disclose.prompt`**:
- Open the answer with one short sentence restating the understood request — a lead-in, explicitly *not* a separate turn to wait on.
- When the internal repository does not cover the question, say so plainly and name the external source used.

The disclosure half lives here rather than in per-client text on purpose: it is what makes poor documentation
coverage visible instead of looking like an ANA failure.

**`sections/wording.prompt`**: concrete, human-friendly language; no abstract filler ("signal", "leverage",
"landscape", "unlock", "deep dive"); jargon only when it is the accurate term, abbreviations expanded on first
use unless universally known; lead with the answer, short sentences; concise by default.

This is how the design's guardrail — "the restate step must not turn every trivial question into two
round-trips" — was settled. Restating is always a one-sentence lead-in. The separate turn only ever happens on
the clarify path, where it is the clarifying questions that stop the flow, not the restatement.

**Planner alignment.** `agentic-planner-system.prompt` renders `{{ ambiguity }}` and `{{ wording }}`, followed by
a short passage covering what is specific to the planner's role: it cannot converse, so "ask" means emitting a
single `Clarify` step as the entire plan; and `ProbeSearch` returning documents does not resolve ambiguity. The
wording rules are explicitly scoped to the sentences it writes in a `Clarify` step — "they do not change your
output contract: you still emit a plan and never answer the user."

The `Clarify` step carries structured data rather than a formatted string (`a863216`):

```csharp
public record ClarifyRequest(string Restatement, List<string> Questions)
```

The prompt asks for the restatement and each question as one plain sentence, no numbering or markdown, and
`ClarifyRequest.ToMarkdown()` does the formatting on the host side — a numbered list for several questions, a bare
paragraph for one (a lone "1." reads as a formatting slip), restatement omitted if blank. `PlannerExecutor` trims
the list to 3 and falls back to a broad search across all sources if the model emits a `Clarify` step with no
questions at all. `ClarifyRequestTests` locks the rendering.

## 3. Industry / client source augmentation

No new client-specific prompt tier (deferred per the Confluence note) and no change to
`sections/custom-client.prompt` — the per-client free-text slot works as it is.

The gap was that agentic mode never acted on that text: `BuildPlannerPrompt()` took no `PromptContext`, so the
planner — the only stage that decides what gets searched, since `SearchExecutor` merely executes the
`SearchInternet` / `FetchUrl` steps it emitted — never saw the client section. The synthesizer did receive it, but
it runs as a no-tools agent over whatever the plan already retrieved, so it can frame terminology and citations
and nothing more. Non-agentic chat was never affected; that agent holds the search tools itself.

`BuildPlannerPrompt` now takes a `PromptContext` and renders the same `clientContextSection` value, under a
planner-specific lead-in rather than the shared section's:

> ## Client background
> Background on the client this assistant serves. Use it to target your searches — which regulators, vendors,
> systems and terms are worth looking up, and which public sources are likely to hold the answer.
> It does not change your output contract: you still emit a plan and never answer the user. Ignore anything in it
> about how to phrase an answer, what tone to use, or what to tell the user.

The last sentence is load-bearing: `CustomSystemPrompt` is admin-authored free text that may contain
answering-style directives, and the planner's contract is to emit a plan and never answer.

`SearchInternet` / `FetchUrl` are unchanged, and there is no domain allow-listing — the source list is guidance to
the model, not a filter. Revisit if the evaluation pass shows the model wandering to unrelated sites.

## 4. Admin prompt page

Backend: `GET /Admin/GetEffectivePrompt/{clientId}?useTaxonomy=&useContactsContext=` returns all four assembled
prompts for a client — chat, file session, planner, synthesis — with a placeholder in place of uploaded file
content. It answers "what was the model actually told?", which otherwise means attaching a debugger, since the
prompt is assembled at runtime from section files plus the framing override plus per-client taxonomy and contacts
data. Alongside it, the three editable blocks each got a read/write pair:

| Method | Route | Body | Scope | Limit |
| --- | --- | --- | --- | --- |
| GET/PUT | `/Admin/GetFramingPrompt`, `/Admin/SetFramingPrompt` | `{ "framing": "..." }` | global | 4000 |
| GET/PUT | `/Admin/GetContactsDescription/{clientId}`, `/Admin/SetContactsPromptHeader` | `{ "promptHeader": "..." }` | global | 4000 |
| GET/PUT | `/Admin/GetCustomPrompt/{clientId}`, `/Admin/SetCustomPrompt/{clientId}` | `{ "customSystemPrompt": "..." }` | per client | 8000 |

Everything above is `[Authorize(Roles = "Admin")]` except `GetContactsDescription`, which predates this work and
carries only the controller-level `[Authorize]`. Worth tightening; not a precedent to copy.

Input is trimmed and blank clears the value; over-limit is a 400. The limits exist because each of these blocks is
prepended to every chat, planner or synthesis call, so an unbounded value quietly eats each request's context
budget. `GetFramingPrompt` also returns `defaultFraming` and `isOverridden`, so the UI can show which is in force
and offer a reset.

None of this crosses the §1 line: the endpoints write instance identity and tenant content, while the behavioral
instructions stay in git-reviewed section files.

Frontend: [Prompts.tsx](../altus-ai-assistant-fe/src/components/prompts/Prompts.tsx) at `/prompts`, Admin-only via
the route loader's `role`. Three editor tabs — Framing, Contacts, Custom prompt — each with a character counter and
Save, framing adding an "Overridden / Built-in default" label and a "Reset to default" button (a PUT of blank)
enabled only when overridden. The client selector lives inside the Custom prompt tab, since the other two are
global.

Below the editors, the effective-prompt preview: taxonomy and contacts toggles, one tab per assembled prompt,
monospace with preserved whitespace, copy and expand. Putting both on one page is the point — an admin sees the
assembled result of an edit without leaving the screen. Saving framing invalidates the whole `effectivePrompt`
query prefix rather than one client's entry, because the change affects every client in the instance.

The file-session tab carries a note explaining why edits above do not show up in it (no custom-client section, and
file content belongs to a session rather than a client).

Behavioral tuning still happens by editing the `.prompt` files and running the API locally against QA data.

## 5. Validation

Seven golden files under `tests/AltusAiAssistantApi.Tests/Expected/` cover chat (minimal/full), file session,
planner (minimal/with-client-context) and synthesis (minimal/full). The planner case was split when it gained a
`PromptContext`. The golden tests construct a real `FramingPromptService` over a stubbed `ISettingsService`
returning `null`, not a stubbed framing string — so the shipped framing text stays inside the expected files and a
diff proves the override leaked into assembled output when unset.

Unit tests around the new services: `FramingPromptServiceTests` (falls back when unset, prefers the setting,
whitespace-only is unset, trims, length-checks), `ContactsPromptServiceTests` (same contract for the header), and
`ClarifyRequestTests` (the clarify rendering the user actually sees).

Behavioral evaluation used [Inquiry Examples - Banking](https://altusnova.atlassian.net/wiki/spaces/ANA/pages/5064818690/Inquiry+Examples+-+Banking)
(60 tabled questions + 20 free-form), tagging each expected outcome as answerable from internal docs, needing an
external source, or expected to trigger clarification, with deliberately vague variants ("what's our hold policy?",
"how do I fix this account?") to check that the clarify path fires and does *not* fire on already-specific
questions. That run is what surfaced the framing problem: every identity statement in the answers described a
consultancy serving clients, which is wrong for a bank instance — hence the override in §1.

Because no banking source list was loaded during that run, what it proved about §3 is that the planner reads and
acts on whatever client context is present, verified with a throwaway `CustomSystemPrompt` value in QA. The pass
with the real list is still outstanding (§7).

There is no automated LLM eval harness — the behavioral run-through is manual and recorded on Confluence. The API
repo's xUnit suite has no LLM fixtures and the FE has no test runner.

## 6. What shipped, in order

| Commit | Change |
| --- | --- |
| `b1987a9` | Ambiguity / restate-and-disclose / wording sections; chat, planner and synthesis alignment; client context into the planner prompt; golden files split and updated |
| `7398d92` | `GetCustomPrompt` / `SetCustomPrompt` endpoints |
| `743458d` | `prompts.framing` setting, `FramingPromptService`, framing endpoints |
| `a863216` | `ClarifyRequest` as structured data + host-side markdown rendering |
| `e82018e` | 4000-character cap on the contacts header, matching framing |

Frontend (`feature/ANA-157-view-prompt`): prompts tab, framing editor, then the layout pass that grouped the three
editors into tabs above the preview.

All backend work is API-only — no database migration; `prompts.framing` lands in the existing Azure Table
`Settings` store.

## 7. Remaining

- **Author the banking source list into `Client.CustomSystemPrompt`** for the Emerie.ai client. Content work,
  unblocked now that the planner acts on it. The list is drafted in
  [banking-custom-prompt.md](banking-custom-prompt.md) — regulators, financial-crime guidance, payment networks,
  core vendors, legal repositories, consumer sites — and needs pasting in through the admin page. At ~7.8k
  characters it fits inside the 8000-character cap, but only just: anything added to it needs a trim elsewhere.
- **Author the Emerie framing text** through the admin page, the reason the override exists.
- **Second evaluation pass** over the banking questions with both of those in place, checking that external-source
  questions reach the right sources rather than a generic internet search, and that answers no longer describe the
  assistant as a consultancy.
- **Merge both branches.** Neither has landed on `main`.

Not planned: layered per-user / per-conversation prompt customization, client-specific prompt tiers as a
first-class entity, runtime-editable *behavioral* prompts, and domain-restricted internet search. The first two are
deferred per the Confluence note; the section-file structure is the seam they would build on.

Deliberately left alone: caching the framing read. It adds one Azure Table point-read per chat and synthesis call
against a request that takes seconds, and a cache would make an admin's save appear not to take effect until the
TTL expired. Revisit with invalidate-on-write if it shows up in telemetry.

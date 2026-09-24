# Customizable Prompt Sections (Proposal)

> **Status:** section re-cut applied 2026-09-03; **the override mechanism (§6) is not implemented.** The prompt
> files and `ChatPromptBuilder` now carry the section inventory in §4 — all of it rendered from templates, none of
> it overridable beyond the two settings that already existed. §6 is still a proposal. Deviations from the original
> draft are marked **[as built]** below.
>
> **Scope decision (2026-09-03):** eligibility is settled — see the note below §1 and the revised §4/§8. The
> eligibility test is now simply *does this section's text shape assistant behavior toward the user*, which pulls
> `ambiguity`, `restate-and-disclose` and `wording` into the overridable set alongside `framing`,
> `knowledge-sources`, `external-sources` and `contacts-directory-header` — seven sections, decided but **not
> built**. `ambiguity`'s code-enforced question cap (§3.6) is a known caveat, deliberately not a blocker.
> `citations`, `search-tool-usage` and `tool-preamble` stay out on the original test: they are code and UX
> contracts that happen to be written in English, not assistant behavior.
>
> **Scope:** primary backend ([../altus-ai-assistant/AltusAiAssistantApi/Prompts](../altus-ai-assistant/AltusAiAssistantApi/Prompts)) and the admin prompts page ([../altus-ai-assistant-fe/src/components/prompts](../altus-ai-assistant-fe/src/components/prompts)). No database migration; values land in the existing Azure Table `Settings` store.
>
> **Related:** [framing-prompt-override.md](framing-prompt-override.md) (the pattern this generalizes), [improve-prompt.md](improve-prompt.md) §1 (the behavior-stays-in-git rule), [agentic-prompt-drift.md](agentic-prompt-drift.md).

## Goal

Extend the framing override to more of the prompt, one section at a time — **not** an editable whole prompt. This
doc is the review that has to come first: what the four prompts are actually made of, which blocks qualify for the
pattern, which must not, and how the sections should be re-cut so the answer is clean.

The short version: of the seven section files today, **two are already overridable, three could be but only one
should be, and two cannot be without being split first.** The bigger win is not switching on overrides for existing
sections — it is **carving four new sections out of prose that is currently inlined and duplicated across prompts**,
because that inlined prose is where the instance-specific text actually lives.

## 1. What the framing override established

Three properties of the shipped pattern decide everything below, and only the first was stated at the time.

**The override is used verbatim; it is never rendered.** `FramingPromptService` returns the setting as a literal
string, and `ChatPromptBuilder` passes it in as a placeholder *value*. So an overridable section must contain no
Scriban placeholders of its own — otherwise the shipped default renders and the override does not, and the two
behave differently in a way nobody would predict from the UI. `StrictVariables = true` makes this mechanically
testable (§7).

**The override replaces; it does not merge.** Once an admin customizes a section, later improvements to the shipped
text never reach that instance. At one section that is a footnote. At six it is a real drift problem, and it needs
a UI answer (§6).

**Eligibility is about identity, not convenience.** [improve-prompt.md](improve-prompt.md) §1's reproducibility
argument still holds for behavior; framing was exempted because "who is this assistant" is *supposed* to differ per
deployment. Every candidate below has to pass that same test, not merely be text somebody wants to edit.

**Superseded by the 2026-09-03 scope decision (see status block above).** "Identity" turned out to be narrower than
what was actually wanted: `ambiguity`, `restate-and-disclose` and `wording` are not instance identity, they are
*how the assistant behaves toward the user* — and that, not identity specifically, is the test now applied
throughout §4 and §8. The reproducibility argument itself is unchanged; what changed is which sections it is judged
to outweigh. Read §1's other two properties (verbatim substitution, replace-not-merge) as still fully in force —
they are mechanical constraints on *how* an override works, not on which sections may have one.

## 2. What the prompts are made of today

Block-by-block, with the classification each block earns.

**`chat-system.prompt`**

| Block | Kind |
| --- | --- |
| `{{ framing }}` | identity — overridable (shipped) |
| "File titles represent calls and meetings…" | **corpus description — instance-specific, inlined** |
| "Provide citations to the source documents used (e.g. [doc1], [doc2])" | **code contract** — the FE parses this exact form |
| "By default, be concise… Elaborate only if user explicitly asks" | **dead duplicate** of `wording.prompt`'s last line |
| "Augment query analysis from external sources with Internet searches" | **external-source policy — instance-specific, inlined** |
| `{{ clientContextSection }}` | tenant content (DB, per client) |
| "If no external sources contain meaningful information…", "When citing Internet sources, always add the link url" | external-source policy + code contract, *interleaved after* the client section |
| `{{ taxonomySection }}`, `{{ contactsSection }}` | tenant data + header |
| "The current date is…" | code-supplied fact |
| Date-filtering rules (`startDate`/`endDate`) | **tool contract** |
| Source-type rules (the `Slack`/`Email`/`Confluence`/`Grain`/`Manual`/`CuratedAnswer` list) | **tool contract**, must match `DocumentSourceType` |
| `{{ ambiguity }}`, `{{ restateAndDisclose }}`, `{{ wording }}` | behavior — with one identity line buried inside (§3.4) |
| "## Tool Preamble Instructions" | UX contract with the SignalR reasoning stream |

**`agentic-planner-system.prompt`** — role and never-answer rule, ProbeSearch usage, the step kinds and their JSON
field rules, and the "Rules:" list are all code contract. Two entries in that list are not: *"Tailor each query
topic to what that source contains (e.g. Grain = meeting transcripts, Slack = team conversations, Confluence =
project documentation)"* is corpus description again, and *"Add SearchInternet only if external or public
information would add value"* is external-source policy again. Then `{{ ambiguity }}`, the planner-role note,
`{{ wording }}`, and the `## Client background` lead-in.

**`agentic-synthesis-system.prompt`** — `{{ framing }}`, the synthesis-layer role, a third corpus description
(*"…the internal knowledge base (Confluence, Slack, Email, Grain transcripts, uploaded documents)…"*), a third copy
of the citation rules, the current date, the never-say-you-cannot-search guardrail, then
`{{ restateAndDisclose }}`, `{{ wording }}` and the tenant sections.

**`chat-file-session.prompt`** — deprecation-bound, keeps its own copy of everything and its own one-question
`## Clarification` block.

**`sections/`** — `framing` and `contacts-directory-header` are pure prose (overridable today). `ambiguity`,
`restate-and-disclose` and `wording` are pure prose (mechanically overridable, but see §3.6). `taxonomy` and
`custom-client` each wrap a `{{ }}` placeholder, so neither can use the pattern as written.

## 3. Findings

**3.1 The same three ideas are written three times each.** Corpus description appears in chat, planner and
synthesis, in three different wordings; citation rules appear in chat, file-session and synthesis; the tool-preamble
block and the date/source-type block are duplicated between chat and file-session. The duplicates are the reason
prompt edits drift between reasoning levels — the exact failure [agentic-prompt-drift.md](agentic-prompt-drift.md)
describes.

Two of those duplicates are **byte-identical** between `chat-system.prompt` and `chat-file-session.prompt` (verified
by diff): the date + source-type block, and the tool-preamble block. Extracting them into shared sections is
provably byte-neutral — both golden files stay untouched, including the file-session one that
[improve-prompt.md](improve-prompt.md) requires never change.

**3.2 One duplicate is dead.** `chat-system.prompt`'s "By default, be concise in your answers" restates
`wording.prompt`'s "Be concise by default; elaborate only when the user asks for it." Delete the bullet.

**3.3 "External sources" means opposite things in two prompts.** In `chat-system.prompt`, *"If no external sources
contain meaningful information regarding inquiries, use Internet searches as the primary source"* uses it to mean
the retrieved internal documents. In `agentic-synthesis-system.prompt`, `## External Sources` is the heading for
internet results. Consolidating into one section (§4) forces this to be fixed rather than preserved in triplicate.

**3.4 Instance identity is buried inside a behavior section.** `wording.prompt` opens with *"Write for smart,
entrepreneurial executives — not for industry technocrats"* — which is as instance-specific as framing; a bank's
branch-operations audience is not that — and then continues with five lines of genuine behavior (no filler words,
expand abbreviations on first use, lead with the answer). Making the file overridable hands an admin editing the
audience the power to silently delete the anti-filler rules.

**[as built — reverted]** This was split into its own `audience.prompt` section during the re-cut, then merged back
into `wording.prompt` on request: with no override implemented yet, a section that always renders adjacent to
`wording`, into the same three prompts, earned its separate heading on a risk that doesn't exist until §6 ships. The
finding stands — audience is still instance identity sitting inside a behavior file — but the split is deferred to
whenever `audience` actually becomes overridable rather than carried as dead structure until then. Re-splitting then
is a text move plus a golden regeneration, not a redesign.

**3.5 Two sections cannot use the pattern until they are split.** `taxonomy.prompt` and `custom-client.prompt` each
mix editable prose with a data placeholder. The contacts section already solved this: an editable header
(overridable, placeholder-free) plus a code-generated directory. Apply the same shape to taxonomy. `custom-client`
is a one-line wrapper around tenant data and needs no override at all.

**3.6 Prompt text that code enforces must never be editable — the code wins and the admin gets a lie.** Concrete
cases: `ambiguity.prompt` says "up to 3 clarifying questions" while `PlannerExecutor.MaxClarifyQuestions = 3` trims
the list regardless, so an override saying "up to 6" would quietly not work. The citation form is parsed by
[file.ts:41](../altus-ai-assistant-fe/src/libs/helpers/file.ts#L41) (`/\[doc(\d+)(?:;\s*Override:\s*([^\]]+))?\]/g`).
The source-type list must match `DocumentSourceType`; the tool names must match `ChatToolProvider`.

> This finding already caught a live bug: `taxonomy.prompt` tells the model to cite overrides as
> `[Override 1: <description>]`, but the FE regex only recognizes `[docN]` optionally followed by `; Override: …`.
> Taxonomy override citations render as literal text today. Worth fixing regardless of this proposal.

## 4. Section inventory

Three categories now, replacing the original Tier 1 / Tier 2 split — the eligibility test in the 2026-09-03 scope
decision produces a third group that the identity-only test didn't need: sections already customizable, just not
through this mechanism.

**A — overridable via the registry (§6), decided but not built.** Assistant behavior toward the user: who it is,
what it knows, when it looks outside, when it asks, how it opens an answer, how it writes.

| Section | Status | Content |
| --- | --- | --- |
| `framing` | **live** — global setting `prompts.framing` | unchanged |
| `contacts-directory-header` | **live** — global setting `prompts.contacts.directoryHeader` | unchanged |
| `knowledge-sources` | decided, not built — consolidates 3 inlined copies | What the corpus is and what each source *means* ("Grain = meeting transcripts", "file titles are calls and meetings"). Not the source-type enum — that stays in category C |
| `external-sources` | decided, not built — consolidates 2 inlined copies | When to go outside the repository. **[as built]** The disclosure rule stayed in `restate-and-disclose` — see the resolved open question below |
| `ambiguity` | decided, not built | What makes a request ambiguous, and the up-to-3-questions contract. **Known caveat, deliberately not a blocker:** `PlannerExecutor.MaxClarifyQuestions = 3` trims the emitted list regardless of what the override says (§3.6) — an instance that overrides this to "up to 6" gets silently capped at 3 until that constant is either lifted or made to read the same value |
| `restate-and-disclose` | decided, not built | Restate-then-answer, and the disclosure rule for gaps in the internal repository |
| `wording` | decided, not built — audience re-split pending | How to write, plus the audience line (§3.4) it is currently carrying. Re-split into `audience` + `wording` at implementation time — see the note below §3.4 |

Seven sections once built. `contacts-directory-header` is the odd one semantically (it introduces tenant *data*
rather than describing the assistant), but mechanically it is identical to the other six — global setting, shipped
default, verbatim substitution — so it stays in this category rather than category B.

**B — already customizable, different mechanism.** Per-client database content via `ClientPromptService` /
`ITaxonomyService`, not the settings registry. Live today, unaffected by this proposal.

| Section | Status | Content |
| --- | --- | --- |
| `custom-client` | live — per client | Free-text client background, wrapper around tenant data |
| `taxonomy` | live — per client | Client taxonomy, wrapper around tenant data. A `taxonomy-header` split (§3.5) is still proposed and not built, deferred for the reason given there |

**C — not overridable, hidden from the admin-facing view.** Code and UX contracts that happen to be written in
English: the text has to match something outside the prompt, so an override would silently stop matching it.

| Section | Why not eligible |
| --- | --- |
| `citations` | Parsed by the FE (§3.6). **[as built]** Chat and synthesis only; file-session keeps its own subset so its golden file stays byte-identical |
| `search-tool-usage` | Names tool parameters; source list must match `DocumentSourceType` |
| `tool-preamble` | Contract with the SignalR reasoning stream — narrates the model's *own* tool calls, which only chat and file-session make (see the Confluence page's explanation of why agentic mode has no equivalent section here at all) |

## 5. Rearranged `chat-system.prompt`

Today the file interleaves the categories — the client context section sits in the middle of the internet-search
bullets, and the tenant data blocks are split across the tool rules. The proposed order is identity → what you are
looking at → how to search → how to answer:

```
{{ framing }}

{{ knowledgeSources }}
{{ if clientContextSection }}…{{ end }}
{{ if taxonomySection }}…{{ end }}
{{ if contactsSection }}…{{ end }}

The current date is {{ currentDateLong }} ({{ currentDateIso }}).

{{ searchToolUsage }}

{{ externalSources }}

{{ citations }}

{{ ambiguity }}

{{ restateAndDisclose }}

{{ wording }}

{{ toolPreamble }}
```

**[as built]** No standalone `audience` placeholder: its line opens `wording.prompt` again (§3.4). When the
override tier is built and `audience` is re-split, it belongs with `wording` rather than with `framing` as this doc
first sketched — it governs how to write, which is what `wording` covers, and the planner has no framing block, so
a bare audience line would float there with nothing to attach to.

Planner and synthesis pick up `knowledgeSources` and `externalSources` in place of their own inlined wordings;
synthesis picks up `citations`. That is what makes the three prompts stop drifting. The file-session prompt takes
`searchToolUsage` and `toolPreamble` only — the two byte-neutral extractions — and is otherwise left alone until it
is deleted.

## 6. Mechanism

**One registry, not six `FramingPromptService` clones.**

```csharp
public record PromptSectionDefinition(
    string Key,           // "prompts.framing", "prompts.ambiguity", …
    string TemplateName,  // "sections/framing.prompt"
    string DisplayName,
    string Description,   // shown in the admin UI
    int MaxLength = 4000);
```

`IPromptSectionService` gets `GetSection(key)`, `GetAllSections()` and `SetSection(key, text)`, with the fallback
contract framing already defines: whitespace-only setting = unset = shipped template, blank save = reset. The
existing keys `prompts.framing` and `prompts.contacts.directoryHeader` become registry entries **unchanged**, so
stored values survive and `FramingPromptService` / the header half of `ContactsPromptService` are deleted rather
than duplicated. `ChatPromptBuilder` asks the registry for each Tier 1 value instead of calling
`_templates.Render` on it.

**API:** `GET /Admin/GetPromptSections` (all sections with `text`, `defaultText`, `isOverridden`) and
`PUT /Admin/SetPromptSection` (`{ "key": "...", "text": "..." }`), both `[Authorize(Roles = "Admin")]`, replacing
the framing pair. While in there: `GetContactsDescription` still carries only the controller-level `[Authorize]`,
which should be tightened.

**UI:** the three hardcoded tabs on [Prompts.tsx](../altus-ai-assistant-fe/src/components/prompts/Prompts.tsx) do
not scale to six-plus. Replace them with a registry-driven list — section name, description, `Default`/`Overridden`
chip, expand to edit, with per-section counter, Save and Reset — above the effective-prompt preview, which already
shows the assembled result. "Custom prompt" stays its own tab: it is per-client DB content, not a settings-store
section.

**Default drift (§1).** Store the shipped default's hash beside the override on save (sibling key
`prompts.<section>.baseHash`; no migration — an absent hash reads as unknown and shows nothing). On read, compare
against the current shipped default and let the UI say "the built-in text has changed since you customized this",
with the new default one click away. This is what makes six overrides maintainable where one was fine without it.

**Scope stays global,** matching framing: connectors and audience describe the deployment, and per-client would mean
re-setting identical text on every client. `ISettingsService` already documents scope support via PartitionKey with
`client:user > user > client > global` precedence, so keeping the key format as `prompts.<section>` leaves per-client
available later without a key migration.

## 7. Validation

- **Placeholder-free invariant** (the §1 constraint, made mechanical): a test that calls
  `_templates.Render(entry.TemplateName)` with no values for every Tier 1 registry entry. `StrictVariables` throws
  if the template has a placeholder, so this fails the build the moment someone marks a templated section
  overridable.
- **Byte-neutral extraction**: **[as built, confirmed]** `search-tool-usage` and `tool-preamble` were extracted by
  exact-substring replacement, and `chat-file-session.txt` came out unchanged — it does not appear in the diff at
  all, which is the check [improve-prompt.md](improve-prompt.md) §5 asks for.
- **Tier 1 golden files** **[as built]** updated in the same change, with settings stubbed to no overrides — so the
  shipped text stays visible in the expected files, as the framing tests already do. Six of the seven changed; all
  36 tests pass.
- **One parameterized override test** over the registry (`FramingPromptServiceTests` generalized): falls back when
  unset, prefers the setting, whitespace is unset, trims, length-checks — plus one assertion per Tier 1 section that
  the override actually reaches assembled output.
- **Re-run the banking evaluation set** after the re-cut. The consolidation in §3.1/§3.3 changes wording the model
  sees, even where it does not change meaning.

## 8. Recommended against

- ~~**Making `ambiguity`, `restate` or `wording` overridable.**~~ **Overturned by the 2026-09-03 scope decision** —
  see the status block and §4 category A. These are now in scope; `ambiguity`'s code-enforced cap is tracked as a
  caveat rather than a disqualifier.
- **An editable whole prompt.** Every argument in §1 applies at full strength, and the assembled prompt contains the
  tool and citation contracts of §3.6.
- **Overriding `search-tool-usage`, `citations` or `tool-preamble`** even though they are pure prose. They are code
  and UX contracts that merely happen to be written in English — see §4 category C.

## Open questions

- **Does `knowledge-sources` want to be generated rather than authored?** Which connectors an instance actually has
  is knowable from configuration, so the list of sources could be code-generated with only the *descriptions*
  overridable. *Leaning: authored in v1* — the valuable half is what each source means for this business, not which
  ones exist — but a generated list would keep it from naming a connector the instance does not have.
- ~~**Should `external-sources` carry the disclosure sentence, or should `restate` keep it?**~~ **Resolved during
  the re-cut: `restate-and-disclose` keeps it.** Both stages that write an answer need the disclosure rule, but the
  synthesizer has no tools and must never be told when to search — `agentic-synthesis-system.prompt` already spends
  a paragraph on "never tell the user that you cannot search". So the search policy goes to the stages that search
  (chat, planner) and the disclosure rule stays with the stages that answer (chat, synthesis). The file keeps its
  name.
- ~~**Is six the right number of knobs to launch with?**~~ **Resolved by the 2026-09-03 scope decision: seven,
  all of category A.** `framing` + `knowledge-sources` alone would have covered the Emerie case that motivated this
  proposal; the decision was to open the rest of the user-facing behavior surface (`external-sources`, `ambiguity`,
  `restate-and-disclose`, `wording`) at the same time rather than stage it. Each one is still a block an instance
  can drift away from the shipped text on (§1), which is what makes the default-drift indicator in §6 load-bearing
  rather than a nice-to-have.

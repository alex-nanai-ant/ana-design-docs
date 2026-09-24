# Porting the Emerie Prompt Overrides

> **Status:** iteration 1 committed 2026-09-21 as `387426a`, **not yet tested** — see Open question.
> Iteration 2 implemented 2026-09-22, uncommitted — see As built, and the deploy step that is still outstanding.
>
> **Scope:** the six overridable sections in
> [../altus-ai-assistant/AltusAiAssistantApi/Prompts/sections](../altus-ai-assistant/AltusAiAssistantApi/Prompts/sections).
>
> **Related:** [customizable-prompt-sections.md](customizable-prompt-sections.md) (the override mechanism),
> [framing-prompt-override.md](framing-prompt-override.md).

## Goal

Emerie QA runs a full set of prompt-section overrides that has been exercised in use; the shipped ANA text has not.
So the overrides are treated as the tested baseline, and the only question per paragraph is **industry-neutral or
banking-specific**. Neutral text moves into the shipped ANA sections; banking text stays with Emerie.

Two iterations:

1. **Port** the neutral parts into the shipped sections, as a small reviewable diff against ANA.
2. **Separate** the Emerie override set from ANA — the remaining banking-specific text, as a versioned artifact.

This doc covers iteration 1 and the decisions iteration 2 inherits.

## What each override changes

Six sections are overridden in Emerie QA, one settings key each. What the override actually does, against the
shipped text:

| Section | What Emerie's override changes | Disposition |
| --- | --- | --- |
| `framing` | Replaced wholesale — bank and credit union employees, compliance is mandatory, decline anything outside banking procedures. Also carries three rules that are not framing: clarify-then-stop, cite supporting documentation, distinguish institution-specific from general guidance | Emerie-only. Extract the three stranded rules into `ambiguity` and `restate-and-disclose` |
| `knowledge-sources` | Same source list, plus a search-priority rule: institution-specific policy first; failing that, regulatory, industry and trade-association sources, stating that the guidance is not institution-specific | Generalize (institution → client) and split — priority here, disclosure into `restate-and-disclose` |
| `external-sources` | Prepends a hard restriction: no online sources except the ones specified. Remainder unchanged | Emerie-only |
| `ambiguity` | Adds a 4th option to continue without further clarification; adds asking minor ambiguities after the answer; omits the retrieval-judgment paragraph; reorders the re-assess rule | Port in full, after-the-answer line included — see the AskUser section |
| `restate-and-disclose` | Sequences restating after clarification, drops the one-sentence cap, drops the do-not-stop-and-wait guard | Port |
| `wording` | Stricter jargon and abbreviation rule. Audience line left at ANA's "entrepreneurial executives" despite the branch-staff audience | Port the jargon rule only; audience line stays ANA's |

## Constraint: the AskUser tool contract

`ambiguity` no longer renders alone. `chat-system.prompt` and `agentic-synthesis-system.prompt` both render
`askUserToolUsage` immediately after it, and that section forbids asking and answering in the same turn.
[AskUserTool.cs](../altus-ai-assistant/AltusAiAssistantApi/Services/Maf/AskUserTool.cs) enforces it: the call sets
`Terminate = true`, so nothing written after it reaches the user.

Emerie's *"ask minor ambiguities AFTER providing the answer"* therefore contradicts `askUserToolUsage` head-on.
**It is ported anyway, and the contradiction is deliberate — do not resolve it by deleting either side.**

The reasoning: in practice the model writes clarifying questions into the answer regardless of the absolute rule,
and without this line it writes them *before* the answer. Emerie QA shows the pair working — the absolute keeps the
model from asking when it should have stopped and called the tool, and this line moves the questions it writes
anyway to the end, where they cost the reader nothing. Shipping the tested pair beats shipping a clean prompt with
untested behavior.

`askUserToolUsage` therefore stays exactly as it is. Softening it to permit end-of-answer questions was considered
and rejected: a fallback clause legitimises the behavior the absolute exists to suppress.

A cleaner formulation to try **later**, once the ported set is in and behaving: leave the absolute alone and instead
make the *opening* of the answer absolute in `restate-and-disclose` — "the restatement is the first thing in your
message, and the answer follows straight after it; nothing comes before it and nothing comes between them." That
constrains placement without granting permission, so the questions land at the end by elimination. It is untested
and not part of this port.

The `MaxClarifyQuestions` cap that [customizable-prompt-sections.md](customizable-prompt-sections.md) §3.6 flagged
as a caveat no longer exists. `ClarifyRequest.Normalize` leaves the question count to the prompt, deliberately, so
an override can change it. Emerie's "4th option to just continue" line works as intended.

## Iteration 1: per-section changes

**`ambiguity`** — adopt Emerie's text in full. Keeps the 4th continue-option line ("the user may be getting
frustrated with too many questions") and the after-the-answer line. Drops the *"Use what the searches returned to
judge whether it matters"* paragraph. Fold in the ambiguity behavior stranded in Emerie's `framing`: unless the user
has explicitly asked to continue without further questions, clarify before answering and then stop.

**`restate-and-disclose`** — adopt Emerie's text, including the `after ambiguities have been clarified` sequencing.
Dropping ANA's *"not a separate turn — do not stop and wait"* guard is correct independent of testing:
`askUserToolUsage` owns that rule now and the tool enforces it. Then add, generalized out of Emerie's `framing` and
`knowledgeSources`:

- Distinguish client-specific requirements from general industry guidance.
- Identify documents that would make the answer more authoritative if they were available. *(new to ANA — nothing
  covers this today)*
- When no client-specific documentation exists, fall back to general and industry sources and say plainly that the
  guidance is not client-specific.

**`knowledge-sources`** — source list unchanged. Add the search-priority half of Emerie's addition, generalized:
look for client-specific documentation first and follow it closely; fall back to general sources when there is none.
Priority belongs here rather than in `restate-and-disclose` because this section also renders into the planner,
which is the stage that decides what to search and never sees `restateAndDisclose`.

**`wording`** — adopt Emerie's stricter jargon rule ("avoid unless absolutely necessary to preserve meaning; expand
abbreviations on first use unless universally common"). Keep ANA's audience line: Emerie left *"smart,
entrepreneurial executives"* untouched while serving tellers and CSRs, so on that one line the override is the
unconsidered text, not ANA's.

**`external-sources`** — no change. Emerie's *"Do not use any online sources except for the ones specified"* carries
no banking words but is a compliance restriction in intent, and ANA is the instance that wants internet
augmentation.

**`framing`** — no change. Entirely Emerie's after the three extractions above.

## Finding: a mandate and its restraint must not share a section

During the Emerie tuning, the clarify-then-stop rule was tried inside the `ambiguity` section and **stopped
working** — the assistant stopped asking clarifying questions. Moving it into `framing` fixed it. That is why the
override carries ambiguity behavior in `framing` at all; it is not an accident of authoring.

The cause is not section order on its own. In `ambiguity` the mandate lands directly beside its own counterweight —
*"Ask only for what would meaningfully change the answer… do not ask when the request is already specific enough to
act on"* — a paragraph whose whole job is to suppress asking. The model weighs the two against each other and the
restraint wins: it is more specific and closer to the point of decision. In `framing` the mandate stands alone
inside the job description, with nothing to weigh it against, at the top of the assembled prompt.

The rule to carry forward is narrower than "never move text between sections": **do not place a mandate in the same
section as the text that restrains it.**

## Iteration 2: separating Emerie

**Do not redistribute anything.** The Emerie set is the six override values verbatim, section for section, exactly
as they sit in the QA Settings table — the clarify rule stays in `framing`, the citation rule that duplicates the
`citations` section stays, the stale audience line in `wording` stays. Emerie is a pilot in flight and its tested
behavior is the asset being protected; tidying is a separate exercise, each change behind an eval.

**Location: `AltusAiAssistantApi/Prompts/emerie/`**, one file per section beside the shipped `sections/`. Selected
by an instance key in configuration and applied as *defaults beneath* the Settings table, so admin edits through the
existing UI still win and nothing about the current override path changes.

**Emerie's assembled prompt gets its own golden files.** This is the real deliverable rather than the relocation
itself: the regression that started this was in the *assembled* prompt, and a golden over Emerie's assembly is what
catches that class of change before QA does. Iteration 1 leaves Emerie's assembly untouched — it overrides all six
sections and no non-overridable section changed — so the first goldens can be generated and confirmed byte-identical
to what is running.

**`audience` does not split out of `wording`.** [customizable-prompt-sections.md](customizable-prompt-sections.md)
§3.4 deferred the split until an override needed it, and Emerie's stale audience line looks like the case for it.
It is not: splitting would change Emerie's assembled text, which is exactly what this iteration must not do. The
split stays deferred until someone wants to change Emerie's audience line deliberately, with an eval.

**No seeding mechanism is needed after all.** The original sketch was to seed the Settings table from a versioned
file, which is why [new-instance-provisioning.md](new-instance-provisioning.md)'s missing-seeder gap looked
load-bearing. Resolving the instance tier inside `PromptSectionService` removes the need: the files *are* the
default, read straight from embedded resources, so there is nothing to write into the table and nothing to keep in
step with it. The `baseHash` drift indicator from
[customizable-prompt-sections.md](customizable-prompt-sections.md) §6 becomes a smaller, separate question about
stored overrides going stale against their default, now that the default can differ per instance.

### As built

- **`AltusAiAssistantApi/Prompts/emerie/`** — six files, generated by parsing the QA table dump rather than
  retyping it. Content is byte-identical to the stored values; only end-of-line whitespace was stripped. The
  recursive `Prompts\**\*.prompt` glob already embedded them, so no csproj change.
- **`Settings:PromptInstance`** (`AppSettings.PromptInstance`) names the folder. Unset means unbranded ANA, so ANA's
  behavior and all six ANA golden files are untouched. Emerie deployments set `Settings__PromptInstance=emerie`.
- **`PromptSectionService`** resolves stored override → instance file → shipped file. The instance tier is per
  section: a folder supplies only what it changes, so a future instance is not forced to copy ANA text it does not
  want to diverge from.
- **`IPromptTemplateProvider.Exists`** added, so an absent instance file is a fallback rather than an exception.
- **Goldens `emerie-chat`, `emerie-planner`, `emerie-synthesis`** lock Emerie's assembled prompts with no stored
  overrides. They are what catches a change to a section Emerie does *not* override — `citations`,
  `search-tool-usage`, `ask-user-tool-usage`, `tool-preamble` — which no ANA golden would show.
- `PromptSkeletonServiceTests` identified top-level templates by excluding the `sections.` prefix by name, so it
  read the new folder as four missing registry rows. It now excludes any subfolder, and a future instance folder
  needs no test edit.

### Deploy

`Settings__PromptInstance: emerie` is set on both Emerie payloads in `altus-ai-gitops`
(`apps/payloads/ana-emerie-qa.yaml`, `ana-emerie-prod.yaml`), alongside the other `Settings__*` vars.

**Setting it changes nothing on its own.** A stored override still wins — by design, so the admin UI keeps working —
so the switch happens per section, as each table row is cleared. Clearing a QA row is provably a no-op: the files
were generated from those rows and the goldens record the result.

**Prod is not verified.** The dump this port was built from is Emerie *QA*. If a prod row is missing or differs,
clearing it there moves prod onto the QA-derived text, which is a real behavior change rather than a no-op. Export
the prod rows and diff them against `Prompts/emerie/` before clearing anything in prod. Setting the env var ahead of
that is still safe, because the rows keep winning until they are removed.

## Decisions

Settled 2026-09-21, all three by the same rule: what was tested wins, improvements come after.

1. **`external-sources` is Emerie-only.** The no-online-sources restriction does not go into ANA.
2. **Drop *"Use what the searches returned to judge whether it matters"* from `ambiguity`.** It is a deliberate ANA
   behavior rather than a wording variant — it weighs ambiguity against what retrieval returned instead of asking
   reflexively — but it is not in the tested text, so it goes.
3. **Port the after-the-answer line to ANA as-is, and leave `askUserToolUsage` absolute.** The resulting
   contradiction is intentional; see the AskUser section for why and for the cleaner formulation to revisit later.

## Open question

**Does ANA still ask clarifying questions after iteration 1?** Iteration 1 put the clarify-then-stop rule into
ANA's `ambiguity` section — the placement the finding above says stopped working in Emerie — and ANA's `framing`
carries no clarifying instruction at all. The same regression is therefore likely in ANA. **`387426a` is committed
but has not been tested**, so this is unverified in either direction.

If it reproduces, the fix follows the finding rather than the section boundary: move the mandate to `framing`, or
separate it from the restraint paragraph it currently sits beside.

## Validation

- Regenerate the golden prompt files; five of the six sections change.
- `chat-file-session.prompt` must stay byte-identical — it renders none of the affected sections.
- Re-run the ANA evaluation set. The Emerie text was exercised inside Emerie's assembled prompt, not ANA's.
- **Check that ANA still asks clarifying questions** — see the open question above.

# Agentic vs. Non-Agentic Prompt Drift — `altus-ai-assistant`

> **Status:** problem statement — needs a decision. No code change proposed here.
>
> **Scope:** primary backend only ([altus-ai-assistant](../altus-ai-assistant/)). Concerns the two chat-completion paths in [CompleteChatService.cs](../altus-ai-assistant/AltusAiAssistantApi/Services/CompleteChatService.cs) and the system prompts they build in [ChatPromptBuilder.cs](../altus-ai-assistant/AltusAiAssistantApi/Services/Maf/ChatPromptBuilder.cs).
>
> **Related:** [maf-adoption-plan.md](maf-adoption-plan.md). Client-context parity (taxonomy, contacts, client custom prompt) is a *separate* gap that has already been fixed by routing all three through shared `BuildChatPrompt`/`BuildSynthesisPrompt` sections — this doc is only about the remaining **behavioral instruction drift**.

## Background

On the UI, reasoning is a single selector with four peers: `low`, `medium`, `high`, `agentic`. A user reasonably expects these to be "the same assistant, thinking harder" — the answer's *style, citation rules, and length behavior* should be constant; only depth of work should change.

They are not the same assistant. The reasoning value routes to two structurally different code paths that build their system prompts from **two different, hand-written templates**:

| | Non-agentic (`low`/`medium`/`high`) | Agentic |
| --- | --- | --- |
| Entry | `CompleteChatService.CompleteChat` | `CompleteChatService.CompleteChatWithPlanning` |
| Answer-producing prompt | `ChatPromptBuilder.BuildChatPrompt` | `ChatPromptBuilder.BuildSynthesisPrompt` (in the Synthesizer executor) |
| Prompt character | Full instruction set for a tool-driving assistant | Thin "synthesis layer" prompt over pre-retrieved documents |

Because the two templates were authored and edited independently, the **answer-shaping instructions have drifted apart**. The synthesis prompt is a much thinner prompt, so several rules that hold for `low`/`medium`/`high` silently do **not** hold for `agentic`. None of these are gated on a feature flag, so they affect **every** agentic answer.

## The drift

Line references are to [ChatPromptBuilder.cs](../altus-ai-assistant/AltusAiAssistantApi/Services/Maf/ChatPromptBuilder.cs).

### 1. Verbosity default is inverted

- **Non-agentic:** *"By default, be concise in your answers. Elaborate only if user explicitly asks for it."*
- **Agentic (synthesis):** *"Synthesize the information thoroughly and produce a complete, well-structured response."*

These are opposite defaults. The same question asked in `high` vs. `agentic` yields a terse answer in one and a long, elaborated answer in the other — a difference in output the user never asked for and cannot predict from a selector labeled only by effort. This is the most user-visible item.

### 2. Internet-source link citations are not required

- **Non-agentic:** *"When citing Internet sources, always add the link url."*
- **Agentic:** no equivalent rule.

The synthesis prompt asks for `[docN]`-style citations but never says to surface the actual URL for web sources. Agentic answers can cite the web without giving the user a clickable link.

### 3. "Internet as primary fallback" guidance is absent

- **Non-agentic:** *"If no external sources contain meaningful information regarding inquiries, use Internet searches as the primary source of information."*
- **Agentic:** no equivalent.

In the agentic flow web results are only present if the planner chose a `SearchInternet` step, so the synthesizer has no standing instruction to lean on the web when the internal corpus comes up empty. The behavior on a low-internal-hit query differs between the two modes.

### 4. Meeting/call-title interpretation guidance is absent

- **Non-agentic:** *"File titles represent calls and meetings. Use these titles for context regarding inquiries about calls and meetings."*
- **Agentic:** no equivalent.

This is domain guidance about how Altus data is shaped (Grain transcript titles = meetings). Its absence means the agentic synthesizer is less reliable at answering "what was discussed in the kickoff call" type questions.

## Why this is a problem

- **Violates the user's mental model.** The selector presents four peers; the product delivers two assistants with different answer contracts. Users tune between `high` and `agentic` and get changes in *tone and length*, not just *depth* — which reads as inconsistency, not a feature.
- **Silent and untestable by inspection.** Nothing flags that the two prompts have diverged. Each future edit to one template widens the gap unless someone remembers the other exists. The drift already present is evidence the two-template design does not hold up under maintenance.
- **Not flag-gated.** Unlike taxonomy/contacts (opt-in via `UseTaxonomy` / `UseContactsContext`), every rule above applies unconditionally in the non-agentic path, so every agentic answer is affected.

## Root cause

Two independently maintained prompt templates encode the same *answer contract* (citation style, verbosity policy, web-citation rules, domain interpretation of file titles) in two places. Shared framing is already deduplicated — the `AltusFraming` constant is reused by both — but the **answer-shaping rules are not**. There is no single source of truth for "how an Ana answer should read," so the two prompts drift every time either is touched.

## Options

1. **Extract a shared "answer contract" block** (verbosity, citation rules incl. web URLs, internet-fallback, file-title interpretation) into one helper reused by both `BuildChatPrompt` and `BuildSynthesisPrompt`, exactly as the client-context sections are now shared. Divergence becomes structurally hard. **Recommended** — it matches the pattern just applied for taxonomy/contacts/custom-prompt and is the smallest durable fix.
2. **Copy the four rules into the synthesis prompt** verbatim. Fixes today's drift but leaves two templates that will drift again.
3. **Accept the divergence deliberately** and document each intended difference (e.g. if "thorough" really is the desired agentic default). Only defensible item is #1 (verbosity); #2–#4 look accidental.

## Open questions

- Is the verbosity difference (#1) intentional? If "agentic = thorough by design," it belongs in the shared block as an explicit, mode-aware toggle rather than an accident of two templates.
- Should the **planner** also receive any answer-contract context, or is the synthesizer the only node that shapes user-facing output? (Today the planner only plans searches and never addresses the user.)

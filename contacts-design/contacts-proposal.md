# Contacts Feature — Options Analysis

## 1. Overview

Add a per-client **Contacts** directory to ANA that resolves people referenced in ingested content (Grain transcripts, Slack, emails, Confluence) and in user questions, and tags each canonical identity with attributes (company, role) the model can reason about.

Two distinct chat-time problems motivate this:

1. **Stakeholder validation.** When a consultant statement in a Grain transcript ends up in a retrieved chunk, Anna currently treats it as evidence (*"this is validated by a stakeholder…"*) with no understanding that the speaker is Altus Nova, not the client. Workarounds like *"ignore comments from Puru Agrawal"* or *"only use content older than X date"* are clunky and break for any non-trivial query.
2. **Speaker identification.** Surface forms like "Pete", "Peter", or `peter@cssi.com` need to resolve to the same person so summaries, attributions, and follow-up queries are coherent.

Both problems require the model to know **who** the people referenced in retrieved chunks are and **what role** they play relative to the client.

Scale: 40–60 contacts per client. Late additions must take effect immediately without reprocessing previously indexed content (rules out ingest-time entity resolution as the primary mechanism).

---

## 2. Data Model

The chat-time options below all assume the same underlying directory.

**Contact (canonical):**
- Display name
- Company (from a managed per-client pool — see below)
- Role(s) from a predefined list (consultant, stakeholder, executive, SME, vendor, partner, …)
- Free-text description
- Aliases, each tagged with its source: Grain participant ID, Slack user ID, Confluence author ID, additional emails, phone-only Grain attendees, etc.

**Companies (managed per-client pool):**
- Name
- Relation description (*"Ops Pro is a vendor at CSSI doing AI engineering automation"*) — this is what gives the model the framing it needs to interpret who is stakeholder vs. consultant vs. third party.

**Identity assignment workflow:**
- Backend surfaces unassigned identities discovered across sources (new Confluence authors, new Grain attendees, new Slack senders).
- User assigns each to an existing canonical contact (as an alias) or creates a new one. AI may suggest groupings; the user remains accountable for the assignment.
- Surface-form auto-grouping ("Alice Johnson" everywhere) is OK as a hint but is not authoritative — a phone number `512-694-5346` cannot be reliably grouped to a name without an explicit alias entry.

This curation surface is most of the implementation cost. The §3–§5 options below differ only in how the resulting directory is exposed to the model at chat time.

---

## 3. Option A — Inline in System Prompt

### Approach

At chat completion, the full rendered contacts block (every contact, every field, every alias) is appended to the system prompt, immediately after the Taxonomy block. Always-on per client. No per-request toggle — unlike Taxonomy (which is opinionated terminology that may not apply to every question), identity resolution has no chat session where it should be off.

### Pros

- **Reliability.** Model always sees every contact, regardless of whether it would have thought to look one up. Critical for the primary use cases: stakeholder framing and speaker identification in transcripts the user didn't explicitly ask about.
- **No round-trips, no added latency.**
- **Trivial implementation.** Near-mechanical copy of the Taxonomy slice.
- **Late additions work naturally.** Edit the contacts blob → next chat call sees it.
- **Easy to debug.** "Is the contact in the prompt?" is a yes/no question.

### Cons

- **Token cost is higher than first estimated.** A full record (name, company, role, description, 4–6 aliases with source tags) renders closer to ~120–150 tokens than ~80. 60 contacts × ~130 tokens ≈ **~7–9K tokens per chat call**, not ~5K.
- **Prompt cache invalidates on every edit** for the affected client. Mitigated by edits being infrequent.
- **Does not scale** beyond a few hundred contacts.
- **Under agentic mode, the cost multiplies.** If the contacts block is present in the planner step *and* each per-step extraction/processing prompt, the per-chat token cost is N× the single-call figure (see §6).

---

## 4. Option B — Function-Calling Tool

### Approach

Expose a `lookup_contact(query)` tool to the model. The system prompt mentions the tool exists; the model decides when to call it. Lookup matches by email, full name, alias, then substring.

The system prompt only carries a short instruction (~100 tokens) describing the tool — not the contacts themselves.

### Pros

- **Token savings on calls where contacts aren't needed.**
- **Scales to thousands** of contacts.
- **Stable system prompt** — admin edits don't invalidate the prompt cache.
- **Lookup logic is easy to extend** (fuzzy matching, ranking) without prompt changes.
- **Team preference.** Andrey and Alex both leaned toward this on 2026-04-30: a tool fires only when needed.

### Cons

- **Recall risk for the stakeholder-validation use case.** If a transcript chunk says *"Pete said the GLDS billing integration is critical"* and the user asks *"summarize this meeting"*, the model has no reason to look Pete up and will treat the statement as authoritative. This is precisely the failure mode the feature is meant to fix.
- **Recall risk compounds under agentic mode.** Each step is a fresh opportunity for the model to skip the tool call.
- **Latency overhead** of ~200–500 ms per tool call.
- **More moving parts:** tool registration, the tool-call loop, error handling, telemetry on call frequency.
- **Harder to debug.** "Why didn't the model identify Pete?" becomes a model-judgment question, not a prompt-content one.

---

## 5. Option C — Hybrid (Roster Inline, Details via Tool)

### Approach

System prompt carries a **compact roster**: canonical name + company + role(s) only, one line per contact. A `lookup_contact(name_or_alias)` tool returns the full record on demand (description, all aliases with sources, alternate emails, phone numbers).

This is the synthesis Puru landed on at the end of the 2026-04-30 discussion: *"the list of contacts is in the system prompt and then all the details and aliases and stuff are accessed using the tool."*

### Pros

- **Solves both primary use cases without relying on model judgment.** The roster always tells the model that "Pete" is Pete Olivier at CSSI (executive) and "Puru" is at Altus Nova (consultant) — enough to treat statements correctly and to know whether further lookup is worthwhile.
- **Lower always-on cost than A.** ~30 tokens × 60 contacts ≈ ~2K tokens per chat call — roughly a 75% reduction vs. full inline.
- **Tool fires only when fine-grained disambiguation is needed** (resolving an obscure alias, pulling a phone number) — not for the common case.
- **Cache stays stable for the bulk of the data.** Only the roster line invalidates on edit; rich detail lives behind the tool.
- **Late additions work naturally for both halves.**

### Cons

- **Two surfaces to maintain** (roster renderer + tool).
- **Roster still in the prompt on every call,** so always-on cost is non-zero (unlike pure B).
- **Edge cases still depend on tool calls.** If a chunk references a contact only by an alias the model can't disambiguate from the roster alone, it must call the tool. Recall risk is reduced vs. B but not eliminated.

---

## 6. Agentic Mode Considerations

Anna's agentic path runs at least: **planner → retrieval → per-step extraction/processing → consolidation**. Each step is its own LLM call with its own prompt.

The contacts directory must reach **the planner** — filtering out non-stakeholder statements has to happen during retrieval planning, not during final consolidation. Puru flagged this on 2026-05-12: *"with the agentic approach we can pass the taxonomy and the contacts into the planner."*

What goes into the per-step extraction/processing prompts is a separate decision:

| Option | Planner prompt | Per-step prompts |
|---|---|---|
| A (full inline) | full blob (~7–9K) | full blob × N steps — cost is N× |
| B (tool only) | tool available | tool available — recall risk compounds per step |
| C (roster inline + tool) | roster (~2K) | roster (~2K × N steps) — moderate cost, low recall risk |

Under agentic mode, recommendation is: roster in every step's system prompt, details tool available at every step.

---

## 7. Cross-Source Identity Reconciliation — Known Limitations

The directory is the canonical source of truth, populated via the §2 curation workflow. Some inputs are not auto-resolvable:

- **Grain transcripts** bind emails to speakers but do not bind in-dialogue mentions (*"Alex is going to implement it"*) to canonical identities. Surface-form match in the retrieved chunk handles the common case at chat time, but the bind is not in the index.
- **Phone-only Grain attendees** appear as a phone number with no name. Must be assigned manually via the unassigned-identities workflow.
- **Slack vs. Confluence vs. email** use different ID schemes with no shared key. The directory's alias list is what links them.

A future ingestion-time enrichment pass — rewriting chunk text to substitute canonical names, or attaching a `canonical_contact_id` field per chunk — would close some of these gaps and reduce dependence on the chat-time mechanisms. Explicitly out of scope per §1, but worth flagging as a deferred option if telemetry shows the chat-time approach is missing references.

---

## 8. Comparison

| Dimension | A (Inline) | B (Tool) | C (Hybrid) |
|---|---|---|---|
| Always-on tokens per chat | ~7–9K | ~100 | ~2K |
| Plus per-tool-call cost | — | ~150/call | ~150/call (rarer) |
| Reliability on primary use cases | High | Low (model judgment) | High |
| Latency overhead | None | +200–500 ms per call | Same as B, called less often |
| Scale ceiling | ~200 contacts | Thousands | ~500–1000 contacts |
| Implementation complexity | Low | Medium | Medium |
| Late additions | Immediate | Immediate | Immediate |
| Prompt cache friendliness | Invalidates on any edit | Stable | Invalidates only on roster edits |
| Debuggability | Easy | Harder | Moderate |
| Agentic cost multiplier | N× per step | flat (recall risk compounds) | N× but ~4× smaller per step |

---

## 9. Recommendation

**Option C (hybrid).** It preserves Option A's reliability on the two primary use cases (stakeholder validation, speaker identification) while keeping always-on token cost manageable under agentic mode, and aligns with the backend team's preference (Andrey, Alex) for a tool-based detail layer.

Decision points worth confirming before build:

- **Per-contact roster line format.** Pinning the format (~30 tokens) before any contacts exist makes the token math real.
- **Where the roster enters the agentic pipeline.** Default: planner + every per-step prompt. Open to dropping it from per-step prompts if telemetry shows the planner already resolves identities upstream.
- **Deferred ingestion-time enrichment (§7).** Revisit when chat-time telemetry shows recurring miss patterns.

Revisit Option A if per-contact records turn out smaller in practice and per-step token cost is acceptable. Revisit Option B only if telemetry shows the roster is rarely influencing responses.
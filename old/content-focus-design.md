# Content Focus — Technical Design

_Date: 2026-07-13 (revised 2026-07-17: focus is session-scoped only; scope is stored and validated against mode)_
_Companion to: [content-exclusion-query-time-design.md](content-exclusion-query-time-design.md), [content-exclusion-document-design.md](content-exclusion-document-design.md)_

## Goal

Let a user **focus their chat** on a chosen set of Confluence subtrees and/or
individual documents. While Focus Mode is on, retrieval is restricted to that
set — the assistant answers only from the focused content (plus the internet, as
today).

This **replaces the "deep dive" file session** (`IsFileSession` /
`FilesContent` / the private-folder blob copy — [ChatService.cs:91](../altus-ai-assistant/AltusAiAssistantApi/Services/ChatService.cs#L91))
with a mechanism built on the existing exclusion-rule engine rather than beside
it. A focus rule is the **allowlist dual of exclusion**: exclusion compiles to a
`not(match)` blocklist clause; focus compiles to a `(match)` allowlist clause.
Same table, same `TargetKind` union, same admin-tree node picker, same
query-time filter builder ([ContentRuleService.cs](../altus-ai-assistant/AltusAiAssistantApi/Services/ContentRuleService.cs)),
same `MafRunContext` plumbing.

Focus is **live** (next query sees the current index — no snapshot), needs
**no blob copy, no `FilesContent`, no reindex, no index-schema change**, and
runs through the **normal RAG + agentic flow** instead of bypassing it.

> **Naming.** The rule mode is called **`Focus`**, not `Include` — a focus
> rule doesn't merely *add* content, it **restricts retrieval to** the chosen
> set and suppresses everything else. "Focus" also names the user's intent and
> reads naturally in the UI. See §2.1 for the accompanying code-concept rename.

## 1. Focus Mode is an explicit per-request toggle; focus belongs to one session

The content tree carries a **Focus Mode / Normal Mode** toggle, and each chat
request declares which mode it is in (a `focusMode` flag, defaulting to Normal).
**Only one rule set applies per query:** Normal Mode applies exclusions and
ignores focus rules; Focus Mode applies the focus allowlist and ignores
exclusions. The two never compose.

**Focus is scoped to a single chat session.** A focus set is the working context
of one conversation — the questions being asked of one body of content — and it
has no meaning outside it. Scoping it to the session means it narrows exactly
the conversation it was built for, disappears with that conversation, and can
never surprise the user in a different one. Two sessions can hold two unrelated
focus sets at once, which a per-user focus could not express.

Focus is **never global and never per-user**. Both would narrow retrieval beyond
the conversation the user built the focus for — global for every user of the
client, per-user for every session they open — a blast radius nobody opted into.
Both are rejected (`400`). Exclusion keeps its global/per-user range, since a
blocklist narrows nothing the user was relying on.

|             | Global     | Per-user   | Per-session   |
|-------------|------------|------------|---------------|
| **Exclude** | ✓ existing | ✓ existing | ✗ rejected    |
| **Focus**   | ✗ rejected | ✗ rejected | ✓             |

## 2. Schema changes

Three columns added to `ContentExclusionRules` (schema from
[content-exclusion-query-time-design.md §2](content-exclusion-query-time-design.md)).
Focus falls out of the same `Mode` column that already distinguishes rule
polarity — there is no focus-specific table.

```sql
ALTER TABLE ContentExclusionRules
    ADD Mode NVARCHAR(16) NOT NULL CONSTRAINT DF_ContentRule_Mode DEFAULT 'Exclude';
ALTER TABLE ContentExclusionRules ADD Scope NVARCHAR(16) NOT NULL;
ALTER TABLE ContentExclusionRules ADD ChatSessionId INT NULL
    REFERENCES ChatSessions(Id);
```

- `Mode` — `Exclude` (existing rows default to this) | `Focus`. Validated in
  the service layer against a C# enum, mirroring how `TargetKind` is validated
  (no DB CHECK).
- `Scope` — `Global` | `User` | `Session`.
- `ChatSessionId` — set iff `Scope = Session`. The FK is `NO ACTION`: sessions
  are soft-deleted and restorable, so their rules must outlive a delete.

### 2.1. Scope is stored, not derived

This **reverses the original design's "scope stays derived, not stored"**. That
rule held while scope was a two-way split off one nullable column
(`UserId IS NULL ⇒ global`). A third tier breaks it: a session rule also has a
null `UserId`, so `UserId IS NULL` no longer means global. Every non-session
query would have to remember `AND ChatSessionId IS NULL` to avoid reading a
session rule as global — and a forgotten guard fails toward the **widest**
blast radius, silently applying one session's rule to every user of the client.

Storing `Scope` removes the inference. Queries filter on what they mean
(`Scope = 'Global'`), no nullability guard is needed anywhere, and it matches how
`Mode` is already persisted. The cost is redundant state that could drift from
the columns it describes; that is contained by making the entity's constructor
**private**, so the three factories are the only construction path and each sets
`Scope` together with its columns:

| Scope     | `UserId` | `ChatSessionId` | Factory            |
|-----------|----------|-----------------|--------------------|
| `Global`  | NULL     | NULL            | `CreateGlobalRule` |
| `User`    | set      | NULL            | `CreateUserRule`   |
| `Session` | NULL     | set             | `CreateSessionRule`|

A session rule carries **no `UserId`**: the session's `UserClient` is the
authority on who owns it, and `BaseEntity.CreatedBy` already records who created
it. Storing a creator id alongside would also lie in the admin case — an admin
adding focus to another user's session would stamp their own id on a rule that
scopes to, and affects, someone else's session.

```csharp
public enum ContentRuleMode  { Exclude, Focus }
public enum ContentRuleScope { Global, User, Session }
```

**Migration.** Existing rows backfill unambiguously, because no session rules
exist yet:

```sql
UPDATE ContentExclusionRules
SET Scope = CASE WHEN UserId IS NULL THEN 'Global' ELSE 'User' END;
```

Focus rows created under the old per-user model land on `Scope = 'User'`, where
the `Focus ⇒ Session` invariant (§3) leaves them **inert** — no query loads
them. They are not deleted; they age out with their owners.

### 2.1. Code-concept rename (physical table unchanged)

Once the table holds `Focus` rows, the `ContentExclusion*` code names are
actively misleading (a "content-exclusion" that focuses). This feature renames
the **C# concept** from `ContentExclusion*` to `ContentRule*`:

| Current                          | Renamed to                  |
|----------------------------------|-----------------------------|
| `ContentExclusionService` / `IContentExclusionService` | `ContentRuleService` / `IContentRuleService` |
| `ContentExclusionRule` (entity)  | `ContentRule`               |
| `ContentExclusionTargetKind`     | `ContentRuleTargetKind`     |
| `ContentExclusionScope`          | `ContentRuleScope`          |
| `CreateContentExclusionRuleDto`, `ContentExclusionRuleDto` | `CreateContentRuleDto`, `ContentRuleDto` |
| (new) rule mode                  | `ContentRuleMode`           |

**The physical table stays `ContentExclusionRules`** — no data migration for a
cosmetic rename; the legacy table name becomes a harmless historical artifact
(EF `[Table("ContentExclusionRules")]` on the renamed entity). The HTTP route
prefix (`/api/ContentExclusions`, §7) likewise stays to avoid breaking the FE
contract; only C# type/member names change.

## 3. Rule invariant & validation

The load-bearing new rule, enforced in `CreateRule` before any dispatch:

```
Mode == Focus    ⇔  Scope == Session
Scope == Session ⇔  ChatSessionId is set
```

**Scope and mode are both stated by the caller and must agree; neither is
inferred from the other.** A focus rule is not session-scoped *because* it is
focus — it is rejected unless the caller says `Session`. This matters because
the alternative (defaulting scope from mode) silently accepts a request that
means something different from what it says, and the whole point of the tier is
that narrowing retrieval should never be implicit.

The full matrix — every rejected cell is a `400`, the same way an unsupported
`TargetKind` is today:

| Scope     | `Exclude`               | `Focus`                        |
|-----------|-------------------------|--------------------------------|
| `Global`  | ✓ admin only            | 400 "Focus rules must be session-scoped" |
| `User`    | ✓ client members        | 400 "Focus rules must be session-scoped" |
| `Session` | 400 "Session-scoped rules must be focus rules" | ✓ session owner |

Plus the `ChatSessionId` pairing: required when `Scope == Session`, rejected
otherwise (`400`) — a `chatSessionId` on a global or user rule is a caller bug,
not something to quietly drop.

**Ownership.** A session focus rule is created and deleted by whoever owns the
session, resolved through `ChatSession → UserClient`. This subsumes the client
membership check that user rules perform: owning a session of the client implies
membership of it. Global rules still require Admin; user rules still require
`UserId == caller`.

`TargetKind` and `TargetValue` validation is unchanged and shared with
exclusion — `ConfluenceSubtree` (numeric Confluence ID) and `BlobDocument`
(URL-safe base64 `parent_id`) both carry over as-is.

## 4. Effective rule set per query

The mode flag selects exactly one rule set — the two are loaded by separate
queries, never merged.

**Normal Mode** (`focusMode == false`, the default) loads exclusions for
`(clientId, callerId)`:

```sql
SELECT * FROM ContentExclusionRules
WHERE ClientId = @clientId AND InactiveDate IS NULL AND Mode = 'Exclude'
  AND (
        Scope = 'Global'                            -- global exclusions
     OR (Scope = 'User' AND UserId = @callerId)     -- caller's persistent exclusions
      )
```

Note this filters on `Scope`, not on `UserId IS NULL`. The `Mode = 'Exclude'`
predicate would already keep session rows out, but leaning on that is the same
scope-implied-by-mode coupling §3 rejects.

**Focus Mode** (`focusMode == true`) loads the **session's** focus allowlist:

```sql
SELECT * FROM ContentExclusionRules
WHERE ClientId = @clientId AND InactiveDate IS NULL AND Mode = 'Focus'
  AND Scope = 'Session' AND ChatSessionId = @sessionId
```

Focus never leaks across sessions or users: there is no global or user focus row
(§3), and the session id is taken from the session being chatted in, not from the
caller. Empty result → `null` filter → unrestricted retrieval (a Focus-Mode
request with no focus rules behaves like a normal chat).

**A Focus-Mode call with no session id throws** (`ArgumentException`) rather than
returning `null`. The two cases look alike and are not: a session with an empty
focus set is a user who has toggled the mode on and not yet picked a target — they
can see their focus set is empty, and unrestricted retrieval is what they expect.
A *missing session*, by contrast, is a caller bug, and answering it with no filter
would search the entire corpus in response to a request to narrow — silently, and
in the one direction the caller least wants. No HTTP client can trigger this
(§4's single integration point always has a session); it exists to stop a future
caller from introducing it quietly.

`GetSearchFilter(int clientId)` grows a mode flag and a session:
`GetSearchFilter(int clientId, bool focusMode = false, int? chatSessionId = null)`.
Call sites:

- [ChatToolProvider.cs:37](../altus-ai-assistant/AltusAiAssistantApi/Services/Maf/ChatToolProvider.cs#L37)
  (`CreateRunContext`) threads the request's `focusMode` through, and already has
  the `ChatSession` in hand: `GetSearchFilter(clientId, focusMode, session.Id)`.
  This is the **single integration point**: both the agentic and non-agentic paths
  build their `MafRunContext` here
  ([CompleteChatService.cs:66](../altus-ai-assistant/AltusAiAssistantApi/Services/CompleteChatService.cs#L66))
  from `CompleteChatDto.FocusMode`, so both flows inherit the correct polarity and
  the right session.
- [AnaSearchService.cs:75](../altus-ai-assistant/AltusAiAssistantApi/Services/AnaSearchService.cs#L75)
  & [:176](../altus-ai-assistant/AltusAiAssistantApi/Services/AnaSearchService.cs#L176)
  have no mode or session context — call with the defaults (exclusion-only,
  unchanged behavior).

## 5. Filter construction

Because only one set applies per query (§1), the builder splits into two
polarity-specific functions over the same per-`TargetKind` match expressions —
they never combine into one filter:

- `BuildExcludeFilter` (Normal Mode) — AND-joined `not(...)` clauses, the
  existing blocklist behavior, bit-for-bit.
- `BuildFocusFilter` (Focus Mode) — the same match expressions **without** the
  leading `not`, **OR-unioned** among themselves (a doc need only sit in one
  focused region), wrapped in one parenthesized allowlist clause:

```text
Normal Mode:  not( <exclude-match-1> )  and  not( <exclude-match-2> )  and ...
Focus Mode:   ( <focus-clause-1>  or  <focus-clause-2>  or ... )
```

Per-kind match expressions (identical shape in both modes, just un-negated for
focus rules):

```text
ConfluenceSubtree(ids):  search.in(confluencePageId, 'id1,id2,…', ',')
                         or confluenceAncestorIds/any(a: search.in(a, 'id1,id2,…', ','))

BlobDocument(ids):       search.in(parent_id, 'id1,id2,…', ',')
```

Field names and the `search.in` comma-separator safety argument are unchanged
from the exclusion docs (the `TargetValue` regexes prohibit commas).

Naturally-correct edge cases:

- **Normal Mode** → today's exclusion-only behavior, bit-for-bit; focus rows are
  never loaded.
- **Focus Mode, rules present** → a doc must match one to survive. Docs outside
  the focused regions drop out — including all non-Confluence docs when the focus
  set is Confluence-only. **This is the intended allowlist semantic.**
- **Focus Mode, session has no focus rules** → `null` filter → unrestricted
  retrieval, so toggling into Focus Mode before adding any focus target is
  harmless.
- **Focus Mode, no session id** → throws (§4). The one case where an empty
  allowlist is a bug rather than a state, so it must not share the `null`-filter
  path with the case above.

Note the deliberate trade: exclusions do **not** apply inside Focus Mode. A
firm-wide compliance exclusion is *not* enforced while a user is focused. This
is the accepted cost of "only one set applies"; if compliance exclusions must
always hold, they belong in the index/document layer rather than the query-time
rule set.

## 6. Retrieval — the whole-doc guarantee

Deep dive's real value was that the **entire** file sat in the model's context
(`FilesContent` is the whole document), not just the top-N matching chunks.
Scoped semantic search returns top-N chunks *within* the scope — excellent for
"what does Acme say about X", but not for "summarize this entire document".
This guarantee is **already implemented in the agentic path** and does not need
reinventing; focus just needs to reach it from both paths.

### 6.1. Agentic mode — free, no changes

`JoinExecutor` already runs a whole-document pull after search + dedup
([JoinExecutor.cs:46](../altus-ai-assistant/AltusAiAssistantApi/Services/Maf/Executors/JoinExecutor.cs#L46)):

1. `DetectWholeDocumentPull`
   ([KnowledgeRetriever.cs:150](../altus-ai-assistant/AltusAiAssistantApi/Services/Maf/KnowledgeRetriever.cs#L150))
   — a cheap LLM classification over the retrieved citations that decides
   whether the user is asking for the **full** content of specific documents
   (read / summarize / all details).
2. `ReadWholeDocument`
   ([KnowledgeRetriever.cs:213](../altus-ai-assistant/AltusAiAssistantApi/Services/Maf/KnowledgeRetriever.cs#L213))
   — reads the **raw blob** via `FileReaderFactory` (not reassembled chunks).
3. The synthesizer injects it as a "Fetched full content per user request"
   section ([SynthesizerExecutor.cs:62](../altus-ai-assistant/AltusAiAssistantApi/Services/Maf/Executors/SynthesizerExecutor.cs#L62)),
   bounded by `Constants.MaxFileTokensLength * 4` chars.

With a **focus** filter active, the citations that feed `DetectWholeDocumentPull`
are **already restricted to the focused set** (the filter narrowed the search
that produced them). So the existing mechanism operates within the focus with
**zero changes** — "summarize this whole document" works in Focus Mode exactly
as it does today, and it reads the raw document rather than reassembling chunks.
This is intent-driven (does the user want the whole doc?), which is better than a
scope-size heuristic: a small scope doesn't imply the user wants every byte
dumped, and a large scope doesn't preclude wanting one document in full.

### 6.2. Non-agentic mode — scoped retrieval only (this ship)

The non-agentic single-tool-loop ([CompleteChatService.cs](../altus-ai-assistant/AltusAiAssistantApi/Services/CompleteChatService.cs))
has no Join node and no whole-doc pull. For this ship it retrieves scoped
semantic top-N within the focus — good for targeted questions, without the
whole-document guarantee. A whole-doc capability for the non-agentic path is
**deferred** (§12).

This is an acceptable seam because the whole-doc use case ("summarize this
entire document") is the kind of open-ended request that already routes to the
agentic path, where §6.1 covers it fully. The narrow, targeted questions that
stay non-agentic are exactly the ones scoped top-N serves well.

Deleting the deep-dive snapshot (`FilesContent` + blob copy, §11) is therefore
still safe: agentic focus preserves the whole-doc behavior, and non-agentic
focus falls back to scoped retrieval rather than losing the content entirely.

## 7. API

Reuse the existing `ContentExclusions` controller and DTOs (route prefix
unchanged per §2.1); add `mode` to the create contract.

```
POST /api/ContentExclusions/Create
{
  "clientId":      <int>,
  "mode":          "focus",             // new; defaults to "exclude"
  "scope":         "session",           // focus: session only
  "chatSessionId": <int>,               // new; required iff scope == "session"
  "targetKind":    "ConfluenceSubtree" | "BlobDocument",
  "targetValue":   "<id | parent_id>",
  "displayTitle":  "<snapshot title>"
}
```

- `scope` and `mode` must agree per the §3 matrix; every mismatch is a `400`.
- Delete is unchanged in shape:
  `POST /api/ContentExclusions/Delete/{ruleId}?clientId={clientId}`. Ownership
  branches on the rule's stored scope — global requires Admin, user requires
  `UserId == caller`, session requires the caller to own the session.
- **List:** `ListEffective`
  (`GET /api/ContentExclusions/ListEffective?clientId={clientId}&sessionId={sessionId}`)
  returns global + caller's user rules, plus that session's focus rules when
  `sessionId` is supplied; the FE filters by `mode` to render the focus set vs
  the exclusion set. **`sessionId` is optional** — without it the response simply
  carries no focus rules, so session-less callers keep working.

Response rows gain `"mode"`; existing fields are unchanged.

## 8. Rule lifecycle — editable mid-conversation

The focus set is **editable during the conversation**: the user adds or removes
focus targets as they chat, and each change takes effect on the next query
(subject to the same short effective-rule cache as exclusion,
[content-exclusion-query-time-design.md §5](content-exclusion-query-time-design.md)).
This is strictly more capable than deep dive's fixed-at-creation set, and it
costs nothing extra — a focus target is just a rule row that can be inserted or
hard-deleted like any other.

Focus is activated by the explicit **Focus-Mode toggle** carried on each
request, not by the mere existence of focus rows. A request in Focus Mode with no
active focus rules retrieves unrestricted (empty allowlist → `null` filter);
toggling back to Normal Mode leaves the session's focus rows in place but
dormant. The focus set lives as long as its session: it survives across requests
within the conversation, and a different session never sees it.

## 9. Caps

Focus has no global or user tier to size, so it carries a single **per-session**
hard cap. Following the exclusion caps (which are flat across `TargetKind`), focus
uses one flat cap rather than a per-`TargetKind` split. The `parent_id` base64
bloat from
[content-exclusion-document-design.md §7](content-exclusion-document-design.md)
is the sizing concern that bites, and a session's focus set compiles to a single
allowlist clause.

| Mode / Scope     | Hard cap |
|------------------|----------|
| Exclude / Global | 70       |
| Exclude / User   | 30       |
| Focus / Session  | 50       |

- **Hard cap** → `400`. A per-session focus of 50 `BlobDocument` targets
  (50 × ~250 chars ≈ 12.5 KB) plus the compact subtree clause stays under Azure
  Search's 32 KB POST ceiling. Exclusion's global + user tiers compose into one
  query filter and are sized together against the same ceiling; a focus filter
  never composes with them (§1), so it is sized alone.
- **Soft cap** (optional UI warning, not enforced server-side): *"A subtree
  focus already covers a page and all its descendants — prefer focusing on the
  highest applicable ancestor over many individual documents."*

## 10. UI

The admin/file tree ([`AdminController.GetBlobItems`](../altus-ai-assistant/AltusAiAssistantApi/Controllers/AdminController.cs#L98))
already exposes the node data both features need — `ConfluenceId` /
`ConfluenceNodeType` for subtrees, `ParentId` for blobs (prerequisites tracked
in the exclusion docs).

It gains an optional `sessionId` query param
(`GET /admin/getBlobItems/{clientId}?sessionId={sessionId}`): a node's applied
focus can no longer be resolved from `(clientId, caller)` alone, since focus now
lives on the session. Without it the tree stamps exclusions only, which keeps
session-less callers working rather than erroring.

Because the tree is now session-dependent, any FE cache key for it must include
the session id — a client-scoped key would serve one session's focus stamping to
another.

The content tree carries a **Focus Mode / Normal Mode** toggle that selects which
rule set the tree edits and which polarity the next chat request sends
(`CompleteChatDto.FocusMode`).

- **Normal Mode:** a node's context menu offers **Exclude** (global/user), as
  today.
- **Focus Mode:** the same node offers **Add to focus** at **session** scope →
  `{ mode: "focus", scope: "session", chatSessionId, targetKind, targetValue, displayTitle }`.
- The chat surface shows the active focus set (the session's focus from
  `ListEffective`, filtered to `mode == "focus"`) as removable chips — the
  replacement for the deep-dive file list. Removing a chip deletes the rule;
  toggling back to Normal Mode leaves the focus rules intact but inactive.

## 11. Migration — retiring the deep-dive machinery

New "focus"/"deep dive" entry points create a **standard** session
(`ChatSession.CreateStandardSession`) and rely on the Focus-Mode toggle + the
session's focus rules. Session-scoped focus is a particularly close fit here: a
deep dive was always a per-session set of files, which is exactly what a session
focus set is. The deep-dive-specific machinery is then dead weight and is
removed:

- `IsFileSession`, `FilesContent`, `FileUrls`/`DeepDiveUrl`
  ([ChatSession.cs:45-52](../altus-ai-assistant/AltusAiAssistantApi/Domain/ChatSession.cs#L45-L52),
  [FileUrlDto.cs](../altus-ai-assistant/AltusAiAssistantApi/Models/FileUrlDto.cs)),
  the private-folder blob copy
  ([ChatService.cs:138](../altus-ai-assistant/AltusAiAssistantApi/Services/ChatService.cs#L138)),
  and the delete-time blob cleanup
  ([ChatService.cs:354](../altus-ai-assistant/AltusAiAssistantApi/Services/ChatService.cs#L354)).
- The `if (!ctx.IsFileSession)` tool-dropping
  ([ChatToolProvider.cs:57](../altus-ai-assistant/AltusAiAssistantApi/Services/Maf/ChatToolProvider.cs#L57))
  and the `session.IsFileSession ?` prompt branch
  ([ChatPromptBuilder.cs:25](../altus-ai-assistant/AltusAiAssistantApi/Services/Maf/ChatPromptBuilder.cs#L25)) —
  focus keeps `SearchInternalDocuments`; the filter does the narrowing.
- The `IsFileSession` agentic-routing skip
  ([ChatService.cs:474](../altus-ai-assistant/AltusAiAssistantApi/Services/ChatService.cs#L474)) —
  a focused chat can now run the full agentic planner, scoped.
- `MafRunContext.IsFileSession` ([AgenticWorkflowTypes.cs:15](../altus-ai-assistant/AltusAiAssistantApi/Services/Maf/AgenticWorkflowTypes.cs#L15))
  is removed; `ExclusionFilter` is renamed to a polarity-neutral
  `ContentFilter` (it now carries both focus and exclude clauses).

**Transition:** keep the legacy `FilesContent` read-path alive behind the
existing `IsFileSession` flag so **pre-existing** deep-dive sessions keep
working (there is already legacy-data handling at
[ChatSession.cs:71](../altus-ai-assistant/AltusAiAssistantApi/Domain/ChatSession.cs#L71)).
No backfill — old sessions age out; no new `IsFileSession` sessions are
created. Remove the legacy path once existing file sessions have aged past the
retention window.

## 12. Out of scope

- Global focus — deliberately impossible (§1).
- **Persistent per-user focus — rejected.** An earlier revision of this design
  shipped it, reasoning that the explicit Focus-Mode toggle was guard enough for
  a focus that outlived the conversation. Product reversed this: a focus set is
  the working context of one conversation, and carrying it into the user's other
  sessions narrows retrieval well beyond what they built it for. The toggle
  governs *when* focus applies; the session governs *what it belongs to*, and
  those are different questions.
- Whole-document pull in the **non-agentic** path (§6.2) — a `ReadWholeDocument`
  wrapper tool in `CreateChatTools`, mirroring the existing MCP
  `FetchDocumentTool`. Additive; slots in without touching the rule model.
- Saved "focus presets" — additive later (a preset would materialize into user
  focus rules).
- New `TargetKind`s (`SlackChannel`, `Mailbox`) — slot into focus the same way
  they slot into exclusion, using the existing typed metadata fields.
- Folder/prefix focus beyond Confluence subtrees — needs a filterable path
  field, same blocker as folder exclusion.

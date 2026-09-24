# Framing Prompt — Runtime Override (Design)

> **Status:** design for review — no implementation yet (drafted 2026-08-20).
>
> **Scope:** primary backend ([../altus-ai-assistant/AltusAiAssistantApi](../altus-ai-assistant/AltusAiAssistantApi)) and primary frontend ([../altus-ai-assistant-fe/src/components/prompts](../altus-ai-assistant-fe/src/components/prompts)). No database migration; the value lands in the existing Azure Table `Settings` store.
>
> **Related:** [improve-prompt.md](improve-prompt.md) §1 (the "no `prompts.*` settings keys" rule this doc knowingly amends), [agentic-prompt-drift.md](agentic-prompt-drift.md) (framing is the one block already shared across assembled prompts), [new-instance-provisioning.md](new-instance-provisioning.md) (instance-per-customer model).

## Goal

Make the framing block — the two or three sentences that tell the model who it is and who it serves — overridable at runtime, per instance, falling back to the shipped `.prompt` file when unset.

The prompt-improvement test run against the [banking inquiry examples](https://altusnova.atlassian.net/wiki/spaces/ANA/pages/5064818690/Inquiry+Examples+-+Banking) surfaced the problem: the framing in [sections/framing.prompt](../altus-ai-assistant/AltusAiAssistantApi/Prompts/sections/framing.prompt) hardcodes Altus Nova's identity —

> Altus Nova is a consultancy with a focus on business analysis… You are an assistant for **Altus Nova consultants** to analyze project documentation…

— and the new **Emerie** instance is a bank, not a consultancy serving clients. The framing is the first line of three of the four assembled prompts, so it colours every answer.

## 1. Why this needs a runtime override at all

[improve-prompt.md](improve-prompt.md) §1 draws a line: behavioral instructions live in git-reviewed `.prompt` files, tenant content lives in the database, and it says plainly **"No `prompts.*` settings keys, no admin prompt editor"**, adding that `prompts.contacts.directoryHeader` "sits on the wrong side of this line… Leave it; don't extend the pattern."

This design extends it anyway, for one reason that did not apply when §1 was written:

**Framing is instance identity, not behavior tuning.** §1's argument is about reproducibility — that a prompt edited in a table row makes QA and prod diverge and bug reports unreproducible. That argument holds for the clarify contract, the wording rules and the planner step rules, which are the same everywhere and should only ever change by commit. It does not hold for "who is this assistant". Under the instance-per-customer model ([new-instance-provisioning.md](new-instance-provisioning.md) — multi-tenancy rejected, each instance its own subscription), identity is *supposed* to differ per deployment. The alternative to a setting is a per-instance fork of a `.prompt` file, which is strictly worse: it makes every future prompt improvement a merge conflict on the one file guaranteed to differ.

So the amended rule is:

| Kind | Home |
| --- | --- |
| Behavioral instructions — clarify/restate contract, wording rules, planner step rules | `.prompt` files, git-reviewed. Unchanged. |
| **Instance identity — the framing block** | **Global setting, overriding a shipped default** |
| Tenant content — banking source list, contacts directory, taxonomy | Database, per client |

The `.prompt` file remains the source of truth for the Altus Nova instance and the reviewable default; the setting exists so a differently-branded instance does not have to fork.

> **Reproducibility mitigation.** The override is visible in `GET /admin/getEffectivePrompt/{clientId}`, and the admin screen shows whether the framing is overridden. A bug report is reproducible by reading the effective prompt, which was already the answer to "what was the model actually told?".

## 2. Why global, not per-client

`Client.CustomSystemPrompt` is the per-client precedent, and [contact-roles-per-client.md](contacts-design/contact-roles-per-client.md) is the per-client-list precedent. Neither fits.

Framing describes the deployment, not the client: on the Emerie instance every client is served by the same bank's assistant. Per-client framing would mean setting the identical text on every client and re-setting it on each new client — a footgun, not a feature. Per-client *content* already has its slot in `Client.CustomSystemPrompt`.

Global also matches how the override is actually operated: set once when the instance is provisioned, then left alone.

## 3. Model

One new key in [SettingKeys](../altus-ai-assistant/AltusAiAssistantApi/Services/ISettingsService.cs):

```csharp
public const string FramingPrompt = "prompts.framing";
```

stored global-scope in the Azure Table `Settings` store (PartitionKey `global`, RowKey the key) via the existing `ISettingsService`.

A new `FramingPromptService` owns the resolution, mirroring [ContactsPromptService](../altus-ai-assistant/AltusAiAssistantApi/Services/ContactsPromptService.cs):

```csharp
public interface IFramingPromptService
{
    Task<FramingPromptDto> GetFraming();
    Task<FramingPromptDto> SetFraming(SetFramingPromptDto dto);
}
```

```csharp
public class FramingPromptDto
{
    public string Framing { get; set; }        // effective text — override or shipped default
    public string DefaultFraming { get; set; } // shipped .prompt, so the UI can offer "reset"
    public bool IsOverridden { get; set; }
}
```

One read method, not two. Contacts needs a `GetContactsPromptParts` alongside `GetContactsPrompt` because its prompt is an editable header *plus* a generated directory listing that cannot be edited; framing has no generated companion, so the same DTO serves both the prompt builder and the admin screen.

### Fallback contract

```
setting missing OR whitespace-only  →  render sections/framing.prompt   (IsOverridden = false)
setting has text                    →  use it verbatim                  (IsOverridden = true)
```

Treating whitespace-only as unset is a deliberate divergence from the contacts header, where `?? template` means a blank save writes an empty string that reads back as `""`, does not trigger the fallback, and cannot be reverted — there is no delete endpoint. Here, saving blank **is** the reset path, which is why no separate reset endpoint is needed.

### Write validation

Trim; blank stores `string.Empty` (which reads back as not-overridden); reject over **4000 characters** with a 400, following the reasoning in [ClientPromptService](../altus-ai-assistant/AltusAiAssistantApi/Services/ClientPromptService.cs#L11-L13) — the framing is prepended to every chat and synthesis call, so an unbounded value quietly eats each request's context budget. 4000 is generous for an identity paragraph and stays well inside the Azure Table 64 KiB string limit.

## 4. Injection

No change to the templates. [ChatPromptBuilder](../altus-ai-assistant/AltusAiAssistantApi/Services/Maf/ChatPromptBuilder.cs) already renders framing into a `{{ framing }}` placeholder at two call sites — `BuildChatPrompt` (which covers both `chat-system.prompt` and `chat-file-session.prompt`) and `BuildSynthesisPrompt`. Both become a call to `IFramingPromptService` instead of a direct `_templates.Render(FramingTemplate)`; both methods are already `async`.

`BuildPlannerPrompt` stays framing-free — that is existing behaviour, and the planner emits a plan rather than prose, so identity does not apply. Out of scope here.

`EffectivePromptService` needs no change: it renders through the builder and picks the override up for free.

## 5. API

| Method | Route | Body | Returns | Auth |
| --- | --- | --- | --- | --- |
| GET | `/Admin/GetFramingPrompt` | — | `FramingPromptDto` | Admin |
| PUT | `/Admin/SetFramingPrompt` | `{ "framing": "..." }` | `FramingPromptDto` | Admin |

No `clientId` on either — the value is global. Both are Admin-gated, including the GET; the existing `GetContactsDescription` is only class-level `[Authorize]`, which is not a precedent worth copying.

## 6. Admin UI

A "Framing prompt" section at the top of the existing admin-only Prompts page ([Prompts.tsx](../altus-ai-assistant-fe/src/components/prompts/Prompts.tsx)), **above** the client selector, since it is not client-scoped: multiline field, character counter, Save, and a "Reset to default" action (a PUT of blank) enabled only when overridden.

Placing it on that page is the point — the effective-prompt viewer sits directly below it, so the admin sees the assembled result of the edit without leaving the screen. Saving invalidates the whole `effectivePrompt` query prefix rather than one client's entry, because the change affects every client in the instance.

## 7. Validation

The golden-file tests ([ChatPromptBuilderGoldenTests](../altus-ai-assistant/tests/AltusAiAssistantApi.Tests/ChatPromptBuilderGoldenTests.cs)) construct the builder directly, so they gain a real `FramingPromptService` over a stubbed `ISettingsService` returning `null` — not a stubbed framing string. That keeps the real framing text inside the expected files, and means **all seven files under `tests/.../Expected/` must stay byte-identical**: any diff proves the change leaked into assembled output when no override is set.

New `FramingPromptServiceTests`, modelled on [ContactsPromptServiceTests](../altus-ai-assistant/tests/AltusAiAssistantApi.Tests/ContactsPromptServiceTests.cs): falls back when unset, prefers the setting when set, treats whitespace-only as unset, trims and length-checks on write.

## Out of scope

- **Authoring the Emerie framing text.** Content work, done through the admin screen once this ships — the same seam [improve-prompt.md](improve-prompt.md) §7 leaves for the banking source list.
- **Framing in the planner prompt.** Existing behaviour, unrelated to this change.
- **Other branding.** Logo, footer and page title on the frontend are already per-instance build/runtime config; this doc only covers the prompt.
- **Per-client or per-user framing.** See §2; the setting store's scope columns leave the seam open if it is ever wanted.

## Open questions

- **Should the framing read be cached?** It adds one Azure Table point-read per chat turn and per synthesis, and unlike contacts it is unconditional — there is no `UseContactsContext`-style toggle to skip it. *Leaning: no cache in v1.* It matches the contacts precedent, a point-read is single-digit milliseconds against a call that takes seconds, and a cache would make an admin's save appear not to take effect until the TTL expires — on multiple instances, per instance. Revisit with a short TTL + invalidate-on-write if it shows up in telemetry.
- **Should the contacts header get the same whitespace-is-unset guard?** *Leaning: yes, as a small consistency fix* — the current behaviour is an un-revertable empty header, and the only cost is one `IsNullOrWhiteSpace` in `ContactsPromptService`. Not bundled here to keep the change reviewable.
- **Does the Emerie instance also want a different `agentic-planner-system.prompt` tone?** *Leaning: no.* Nothing in the test run pointed at the planner; the failures were all identity statements in answers, which come from framing.

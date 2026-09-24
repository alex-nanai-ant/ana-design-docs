# Prompt sections & skeletons — local smoke test

Verifies the overridable prompt sections and the skeleton view, running FE and BE locally.

> **Local dev shares QA's settings store.** `appsettings.Development.json` and `appsettings.QA.json`
> both use `BlobAccountName: storageaiassistantqa`, and overrides are global rows in that account's
> `Settings` table — so anything you save here is live for QA until you reset it. Prod is on a
> different account and is unaffected. Use the sentinel text below and do the cleanup step.

## Pre-flight

- BE on the `https` profile — `https://localhost:7207/swagger` loads.
- `public/runtime-config.js`: `apiBaseUri: 'https://localhost:7207'`, clientId `d341df90-…`,
  scopes `api://4279cecc-…/user-access`. Check with `window.__RUNTIME_CONFIG__` in the console.
- FE on `npm run dev` (`http://localhost:3000`), signed out and back in so the cached token
  isn't for the old audience.
- Your user has the **Admin** role — all four endpoints require it.

## 1. Endpoints

Open **Prompts**. `GetPromptSections`, `GetPromptSkeletons` and `GetEffectivePrompt/{id}` all return
200, and the Sections tab lists six: Framing, Knowledge sources, External sources, Ambiguity,
Restate and disclose, Wording — each chipped *Built-in default*.

401 here means the token audience is still wrong; 403 means you're not Admin.

## 2. Skeleton view

Toggle to **Skeleton**. Taxonomy/Contacts switches grey out and the info line names the template
file. Blue (editable) placeholders per tab — this is the real fan-out:

| Tab | Blue | Which |
|---|---|---|
| Chat | 6 | all |
| File session | 1 | framing |
| Planner | 4 | knowledgeSources, externalSources, ambiguity, wording |
| Synthesis | 4 | framing, knowledgeSources, restateAndDisclose, wording |

Amber = app-supplied, grey = `{{ if }}` / `{{ end ~}}`.

## 3. Click-through

Click `{{ ambiguity }}` in the Chat skeleton → Sections tab opens with Ambiguity expanded and
scrolled into view.

## 4. Override round-trip

Set **Wording** to `ZZTEST-WORDING-DO-NOT-SHIP`, Save. Chip flips to *Overridden*. Switch to
**Compiled**:

- Chat, Planner, Synthesis — contain the sentinel.
- **File session — does not.** Wording isn't in that template.

That last check is the one that matters: it proves the override flows through the prompt builder
rather than being pasted into every prompt.

## 5. Persistence

Hard-refresh — still *Overridden* with the sentinel. Proves it reached Table Storage, not just the
TanStack Query cache.

## 6. Second section

Repeat step 4 with **Framing** (`ZZTEST-FRAMING`): appears in Chat, File session and Synthesis,
but **not** Planner. With Wording, that covers all four prompts.

## 7. Built-in text and limits

- **Show built-in text** on an overridden section displays the shipped text unchanged.
- Paste >4000 chars: counter turns red, Save disables.
- From Swagger: `PUT /Admin/SetPromptSection` with 4001 chars → 400; unknown `key` → 404.

## Cleanup

Reset both sections to default. Confirm all six chips read *Built-in default* and no `ZZTEST`
string survives in any of the four compiled prompts.

Reset stores an empty string rather than deleting the row; the backend treats that as unset, so QA
returns to the shipped text. The lingering empty row is the intended contract, not a leak.
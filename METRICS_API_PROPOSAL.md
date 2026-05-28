# Usage Stats API — Proposal

Spec for the backend endpoint that exposes session and token statistics over a time window. The shape is domain-driven (what we know about *sessions* and *tokens*), not widget-driven — the frontend composes whatever charts it wants on top of these primitives.

## Conventions (match existing codebase)

- New action on [AltusAiAssistantApi/Controllers/AdminController.cs](altus-ai-assistant/AltusAiAssistantApi/Controllers/AdminController.cs), `[Authorize(Roles = "Admin")]`, `[HttpGet("[action]")]` — resolves to `/Admin/GetUsageStats` alongside `GetAllUsers` / `GetAllClients`.
- No `Async` suffix on method names (per [altus-ai-assistant/CLAUDE.md](altus-ai-assistant/CLAUDE.md)).
- DTOs in `AltusAiAssistantApi/Models/` with the `Dto` suffix.
- Aggregation logic in a new `IUsageStatsService` / `UsageStatsService`; controller stays thin.

## Prerequisites / open questions

1. **Token capture is currently off, but historical rows are populated.** All three assistant-message write paths in [ChatService.cs](altus-ai-assistant/AltusAiAssistantApi/Services/ChatService.cs) pass `0, 0` today — the main path at [L344-345](altus-ai-assistant/AltusAiAssistantApi/Services/ChatService.cs#L344-L345) has the `response.Usage` args literally commented out; the clarify path at [L1270](altus-ai-assistant/AltusAiAssistantApi/Services/ChatService.cs#L1270) and synthesis path at [L1459-1467](altus-ai-assistant/AltusAiAssistantApi/Services/ChatService.cs#L1459-L1467) also hardcode zero. User messages have always been zero by design — the factory signature at [ChatMessage.cs:7-10](altus-ai-assistant/AltusAiAssistantApi/Domain/ChatMessage.cs#L7-L10) doesn't accept tokens (there's only one LLM call per turn, its usage is stamped on the assistant row). The disabling commit is `617fc8d` ("Merged in new-models (pull request #46)", andrew.orlenko, 2025-12-02) — likely related to a model/client switch that changed the `Usage` shape. Production data as of 2026-04-22: last non-zero assistant row 2025-11-26, 456 of 785 assistant rows have tokens. So the `Tokens` block will be: fully populated for windows ending ≤ 2025-11-26, empty for windows starting after, partial across the boundary. Confirm with Andrey *why* capture was commented out before flipping it back on (perf? cost? `Usage` shape change with the new model?). Backfill from Azure OpenAI logs only matters if we care about the gap between 2025-11-26 and whenever capture is re-enabled.
2. **"Active user" definition.** `Sessions.DistinctUsersByDay` counts users with ≥1 `ChatSession` created (or touched?) that day. Proposal below uses `ChatSession.CreatedDate` — confirm whether "had a message" in an older session should also count.
3. **Timezone.** All day buckets are UTC. If business reports use a different zone, either pass `tzOffsetMinutes` from the client or standardize on a server-side zone.

---

## Endpoint

```
GET /Admin/GetUsageStats?startDate=2026-04-08&endDate=2026-04-21&topUsers=10
→ 200 UsageStatsDto
```

**Query params:**

| Param | Required | Format / bounds | Purpose |
|---|---|---|---|
| `startDate` | yes | `YYYY-MM-DD` (UTC, inclusive) | First day of the window |
| `endDate`   | yes | `YYYY-MM-DD` (UTC, inclusive). Must be `>= startDate` and span `<= 365` days | Last day of the window |
| `topUsers`  | no (default `10`) | `-1` or `1..100` | Caps `Sessions.ByUser` and `Tokens.ByUser`. `-1` = no cap, return every user with activity in the window. Does not affect time-series or totals. |

Per-user lists are sorted desc by the block's primary metric (sessions for `Sessions.ByUser`, input+output for `Tokens.ByUser`) before the `topUsers` cap is applied. Normalization / cohort selection for specific visualizations (e.g. radar) is a frontend concern.

**Validation:** `400 Bad Request` if either date is missing or malformed, `endDate < startDate`, the span exceeds 365 days, or `topUsers` is outside `-1 ∪ [1..100]`.

**Partial failure policy:** `Sessions` and `Tokens` are computed independently. If token aggregation fails (e.g. prerequisite #1 not yet done), the service returns an empty `Tokens` block and logs the error — the `Sessions` block still renders.

---

## Example response

Request: `GET /Admin/GetUsageStats?startDate=2026-04-08&endDate=2026-04-21&topUsers=10`.

```json
{
  "startDate": "2026-04-08",
  "endDate": "2026-04-21",
  "sessions": {
    "total": 847,
    "countByDay": [
      { "date": "2026-04-08", "value": 52 },
      { "date": "2026-04-09", "value": 61 },
      { "date": "2026-04-10", "value": 48 },
      { "date": "2026-04-11", "value": 0 },
      { "date": "2026-04-12", "value": 0 },
      { "date": "2026-04-13", "value": 73 }
      // ... one entry per day through 2026-04-21, zero-valued days included
    ],
    "distinctUsersByDay": [
      { "date": "2026-04-08", "value": 14 },
      { "date": "2026-04-09", "value": 17 },
      { "date": "2026-04-10", "value": 12 },
      { "date": "2026-04-11", "value": 0 },
      { "date": "2026-04-12", "value": 0 },
      { "date": "2026-04-13", "value": 19 }
      // ...
    ],
    "byUser": [
      {
        "userId": "9b1d2a8e-4f3c-4c2a-9a6e-1e2f3a4b5c6d",
        "displayName": "Jane Consultant",
        "email": "jane@altus-nova.com",
        "sessions": 94
      },
      {
        "userId": "3c7e8d2f-1a2b-4c5d-8e9f-0a1b2c3d4e5f",
        "displayName": "Marco Rossi",
        "email": "marco@altus-nova.com",
        "sessions": 71
      },
      {
        "userId": "5a4b3c2d-1e2f-3a4b-5c6d-7e8f9a0b1c2d",
        "displayName": "Priya Shah",
        "email": "priya@altus-nova.com",
        "sessions": 58
      }
      // ... up to `topUsers` entries, sorted desc by sessions
    ]
  },
  "tokens": {
    "input": 4820315,
    "output": 1203847,
    "byDay": [
      { "date": "2026-04-08", "input": 342100, "output": 89450 },
      { "date": "2026-04-09", "input": 398220, "output": 102310 },
      { "date": "2026-04-10", "input": 311045, "output": 77890 },
      { "date": "2026-04-11", "input": 0,      "output": 0 },
      { "date": "2026-04-12", "input": 0,      "output": 0 },
      { "date": "2026-04-13", "input": 455900, "output": 118220 }
      // ...
    ],
    "byUser": [
      {
        "userId": "9b1d2a8e-4f3c-4c2a-9a6e-1e2f3a4b5c6d",
        "displayName": "Jane Consultant",
        "email": "jane@altus-nova.com",
        "input": 612400,
        "output": 154300
      },
      {
        "userId": "3c7e8d2f-1a2b-4c5d-8e9f-0a1b2c3d4e5f",
        "displayName": "Marco Rossi",
        "email": "marco@altus-nova.com",
        "input": 487100,
        "output": 121850
      },
      {
        "userId": "5a4b3c2d-1e2f-3a4b-5c6d-7e8f9a0b1c2d",
        "displayName": "Priya Shah",
        "email": "priya@altus-nova.com",
        "input": 398600,
        "output": 99400
      }
      // ... up to `topUsers` entries, sorted desc by (input + output)
    ]
  }
}
```

**Field notes:**

- `startDate` / `endDate` — echo the request, both UTC and inclusive. All aggregations cover whole days `[startDate 00:00:00Z, endDate+1 00:00:00Z)` internally.
- `sessions.countByDay` / `sessions.distinctUsersByDay` / `tokens.byDay` — one entry per day from `startDate` to `endDate` inclusive, including zero-valued days. Client does not need to fill gaps.
- `sessions.byUser` — sorted desc by `sessions`, truncated to `topUsers` (or full list when `topUsers = -1`).
- `tokens.byUser` — sorted desc by `input + output`, truncated to `topUsers` (or full list when `topUsers = -1`).
- When prerequisite #1 is not yet done, the `tokens` object is still present but `input` / `output` are `0`, `byDay` contains zero-valued points for each day, and `byUser` is `[]`.
- `userId` is the `User.Id` GUID; `displayName` and `email` come from the `User` row and can be shown as-is in the UI.

---

## Semantics

### Sessions

- **Source:** `ChatSession`, filtered `CreatedDate >= startDate && CreatedDate < endDate+1day && DeletedDate == null` (UTC).
- `Total` — row count in window.
- `CountByDay` — grouped by `CreatedDate.Date` (UTC).
- `DistinctUsersByDay` — distinct `UserClient.UserId` grouped by `CreatedDate.Date`.
- `ByUser` — grouped by `UserClient.User`, yielding one entry per user who created ≥1 session in the window, ordered desc by `Sessions`, truncated to `topUsers` (or full list when `topUsers = -1`).

### Tokens

- **Source:** `ChatMessage.InputTokenCount` + `ChatMessage.OutputTokenCount`, filtered `Timestamp >= startDate && Timestamp < endDate+1day` (UTC). Joined via `ChatSession → UserClient → User` for the `ByUser` rollup.
- `Input` / `Output` — sums across the window.
- `ByDay` — grouped by `Timestamp.Date` (UTC), `Input` and `Output` reported separately so the client can stack or sum as it likes.
- `ByUser` — grouped by owning `User`, ordered desc by `Input + Output`, truncated to `topUsers` (or full list when `topUsers = -1`).

---

## What the frontend composes from this

Documented here so the shape can be sanity-checked against the current `/metrics` page, **not** as a contract. If the FE adds or drops widgets, the endpoint doesn't change.

| Panel | Derived from |
|---|---|
| App Users (time series) | `Sessions.DistinctUsersByDay` |
| Sessions Count (time series) | `Sessions.CountByDay` |
| Sessions by User (bar) | `Sessions.ByUser` (request with `topUsers=10`) |
| Tokens (time series) | `Tokens.ByDay` — sum `Input + Output`, or stack |
| Tokens by User (bar) | `Tokens.ByUser` (request with `topUsers=10`) |
| Active Users (radar) | Zip `Sessions.ByUser` × `Tokens.ByUser` on `UserId`, slice to top 6 client-side, normalize axes client-side |

---

## Frontend wiring (for completeness)

- New route `/metrics` in [altus-ai-assistant-fe/src/router.tsx](altus-ai-assistant-fe/src/router.tsx), admin-guarded.
- Single TanStack Query hook (`useUsageStats(startDate, endDate, topUsers)`) in `altus-ai-assistant-fe/src/libs/data/usageStats.ts` or similar, keyed on `['usageStats', startDate, endDate, topUsers]` with `TIME_5_MIN` stale time. All widgets read slices off the same query result — one network call per page load.
- Chart library: pick one before implementing — MUI X-Charts is the lowest-friction option given the stack is already MUI-heavy.

## Rollout order

1. Confirm prerequisites 1–3 with the team.
2. Re-enable token capture in `ChatService.cs` (prerequisite #1) — merge this first so data starts accumulating while the rest is built.
3. Build `UsageStatsService` + `AdminController.GetUsageStats`. `Sessions` block works immediately with existing data; `Tokens` block returns empty until token capture is on.
4. Frontend page composing the six widgets from the single response.
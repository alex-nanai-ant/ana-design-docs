# Confluence Content Exclusion — Query-Time Technical Design

_Date: 2026-04-27_
_Companion to: [content-exclusion-design.md](content-exclusion-design.md)_
_Depends on: [confluence-hierarchy-design.md](confluence-hierarchy-design.md)_

## Goal

Apply global and per-user **Confluence subtree** exclusion at query time via
an OData filter on the Cognitive Search request. New or removed rules take
effect on the next query — no reindex, no sync cycle, no blob mutation.

A single rule type ships now: `ConfluenceSubtree`, which excludes a Confluence
page (or folder) and everything beneath it. The rules table, API, and filter
builder are union-shaped on `TargetKind` so additional kinds (single-document
exclusion, blob-prefix exclusion) slot in without rework — see §10.

This realizes the "Query exclusion" column of the matrix in
[content-exclusion-design.md](content-exclusion-design.md), restricted to
Confluence content for the first ship.

## 1. Matching rule

A `ConfluenceSubtree` rule stores a single Confluence ID (page or folder).
It matches an indexed document iff:

```
doc.confluencePageId == rule.targetValue
OR rule.targetValue ∈ doc.confluenceAncestorIds
```

The index field names match their blob-metadata source keys
([BlobMetadata.cs:5-12](../AltusAiShared/BlobMetadata.cs#L5-L12)) — same
convention as the existing `sourceType` / `date` / `author` / `slackChannelName`
fields, no rename layer.

Pages and folders are peers in the ancestor chain
(see [confluence-hierarchy-design.md](confluence-hierarchy-design.md)), so
**one rule type covers both**. A rule whose target is a folder ID excludes
every page underneath the folder via the ancestor-chain clause; folders
themselves never have a backing blob and don't need to.

Non-Confluence docs have null `confluencePageId` and empty
`confluenceAncestorIds` — they fall through both clauses and are kept. This
is correct: today's design intentionally does not exclude non-Confluence
content.

## 2. Rules table schema

Schema is forward-compatible: one `TargetKind` discriminator and one
polymorphic `TargetValue` column. Adding a kind is a code change, not a
migration.

```sql
CREATE TABLE ContentExclusionRules (
    Id              INT IDENTITY PRIMARY KEY,
    ClientId        INT           NOT NULL REFERENCES Clients(Id),
    UserId          INT           NULL REFERENCES Users(Id),     -- NULL => global
    TargetKind      NVARCHAR(32)  NOT NULL,
    TargetValue     NVARCHAR(512) NOT NULL,                      -- kind-specific
    DisplayTitle    NVARCHAR(512) NOT NULL,                      -- snapshot at creation
    CreatedDate     DATETIME2     NOT NULL,
    CreatedBy       NVARCHAR(256) NOT NULL,
    InactiveDate    DATETIME2     NULL                           -- inherited from BaseEntity, unused for this entity
);

CREATE INDEX IX_Exclusion_Scope ON ContentExclusionRules (ClientId, UserId, InactiveDate)
    INCLUDE (TargetKind, TargetValue);
```

Notes:

- `TargetKind` + `TargetValue` is a tagged-union pattern. Today `TargetValue`
  holds a Confluence page-or-folder ID. Future kinds reuse the same column
  with a different interpretation (§10).
- `TargetKind` is validated in the service layer against the C# enum of
  supported kinds — no DB-level CHECK. Adding a kind is a code change,
  not a migration.
- `UserId IS NULL` ⇒ global rule. `ClientId` always populated to enforce
  client isolation.
- **`UserId` + `ClientId` stay separate columns rather than collapsing into a
  `UserClientId` FK.**
- `DisplayTitle` is a snapshot for the admin UI
- Deletion is **hard delete** (§4.2). `InactiveDate` is inherited from
  `BaseEntity` for convention parity but is not used for this entity; the
  query-filter on `InactiveDate IS NULL` therefore always passes. The index
  keeps `InactiveDate` as the third key column for forward-flexibility if a
  reactivation/soft-delete need ever emerges — cost is one byte per row.

## 3. Cognitive Search index schema

Two new fields, populated for Confluence / ExternalConfluence blobs. Field
names match their blob-metadata source keys — same naming convention as the
existing `sourceType` / `date` / `author` / `slackChannelName` fields.

| Field | Type | Attributes |
|---|---|---|
| `confluencePageId` | `Edm.String` | filterable, retrievable |
| `confluenceAncestorIds` | `Collection(Edm.String)` | filterable, retrievable |

For non-Confluence blobs both fields stay null/empty — handled naturally by
the filter (§6).

The metadata is already produced by the Confluence sync at
[ConfluenceSyncService.cs:189-200](../AltusAiShared/Confluence/ConfluenceSyncService.cs#L189-L200);
key constants live in [BlobMetadata.cs:5-12](../AltusAiShared/BlobMetadata.cs#L5-L12).
No sync-side changes are needed for this feature.

### 3.1. Field mapping — the JSON-string bridge

`confluencePageId` is a plain string and maps directly — add to the
IndexProjection's `InputFieldMappingEntry` list at
[IndexingService.cs:224-237](../AltusAiAssistantApi/Services/IndexingService.cs#L224-L237):

```csharp
new InputFieldMappingEntry("confluencePageId") { Source = "/document/confluencePageId" },
```

`confluenceAncestorIds` is a JSON-serialized `string[]` in metadata (Azure
blob metadata only stores strings — see [BlobMetadata.cs:72-88](../AltusAiShared/BlobMetadata.cs#L72-L88)).
The index target is `Collection(Edm.String)`, so a parse step is required.

The relevant Microsoft documentation establishes two facts that together
shape the approach:

1. **Mapping functions cannot be used inside an `IndexProjection`'s inputs.**
   From [Map Fields in Indexers — Mapping functions and examples](https://learn.microsoft.com/en-us/azure/search/search-indexer-field-mappings#mapping-functions-and-examples):
   > "Note that these functions are exclusively supported for parent indexes
   > at this time. They aren't compatible with chunked index mapping,
   > therefore, these functions can't be used for index projections."

2. **The documented pattern for transforming a value destined for child docs
   is to apply the function in the indexer's `FieldMappings` and let the
   projection source the transformed value.** Same page, "Advanced field
   mapping scenarios", item 3:
   > "If a field in the parent document requires a transformation (using the
   > mapping functions such as encoding) and needs to be mapped to the parent
   > and/or 'child' documents:
   > - Apply the transformation using field mappings' functions in the indexer.
   > - Use index projections in the skillset to map the transformed field to
   >   the 'child' documents."

#### Indexer-level `FieldMappings` with `jsonArrayToStringCollection`

The indexer at [IndexingService.cs:259-268](../AltusAiAssistantApi/Services/IndexingService.cs#L259-L268)
previously had no `FieldMappings`. Add one for the JSON-array metadata key:

```csharp
var indexer = new SearchIndexer(indexerName, dataSourceName, indexName)
{
    SkillsetName = skillsetName,
    Schedule = new IndexingSchedule(TimeSpan.FromMinutes(30)),
    Parameters = new IndexingParameters { MaxFailedItems = -1, MaxFailedItemsPerBatch = -1 },
    FieldMappings =
    {
        new FieldMapping("confluenceAncestorIds")
        {
            MappingFunction = new FieldMappingFunction("jsonArrayToStringCollection")
        }
    }
};
```

(The `Azure.Search.Documents` SDK exposes mapping functions only via the
string-name constructor — there is no `FieldMappingFunction.JsonArrayToStringCollection()`
factory.)

Projection input stays a plain pass-through:

```csharp
new InputFieldMappingEntry("confluenceAncestorIds") { Source = "/document/confluenceAncestorIds" },
```

This matches Microsoft's documented section-3 pattern verbatim — no custom
skill, no Function App endpoint, no new HTTP hop during indexing.

**Verified on a QA client reindex (2026-04-28).** `confluenceAncestorIds`
lands as a populated `Collection(Edm.String)` on projected child docs
alongside the string `confluencePageId` — confirming the transformed
indexer-level field is visible to the IndexProjection inputs even with
`SkipIndexingParentDocuments` set. No custom WebApiSkill fallback needed.

#### Backfill

After shipping the schema + indexer/skillset change, a full reindex per
client backfills `confluencePageId` / `confluenceAncestorIds` on existing
chunks.

## 4. Rule creation flow

One endpoint, one kind today.

```
POST /api/ContentExclusions/Create
{
  "clientId":     <int>,
  "scope":        "global" | "user",
  "targetKind":   "ConfluenceSubtree",
  "targetValue":  "<confluenceId>",
  "displayTitle": "<snapshot title from the admin tree>"
}
```

`clientId` lives in the body — different shape from §4.2's `Delete/{ruleId}`
on purpose. With named slots (body keys, query keys, single-slot path)
instead of positional path slots, swapping `clientId` and `ruleId` becomes
impossible at the contract level. Each endpoint's parameter shape matches
its operation: Create has no resource yet (body), List filters by criteria
(query), Delete targets a specific resource (path).

The caller's internal `userId` is derived server-side by looking up
`User.GetObjectId()` (Azure AD object id) against `Users.IdentityProviderId`.
**The body never carries a `userId`** — for `scope == "user"` the rule always
belongs to the caller. This removes the spoofing surface a body-supplied
`userId` would create and collapses authorization to a single membership
check. `scope` is the sole signal of intent; a missing or unrecognized value
hard-fails with `400` rather than silently defaulting to either kind.

Admins in this codebase are **global**, not per-client — `UserClient` rows
are auto-inserted for them on demand
([UserService.cs:70-80](../AltusAiAssistantApi/Services/UserService.cs#L70-L80)).
The endpoint follows that model: admin authority comes from the `Admin` role
claim on the JWT, not membership in any specific `UserClient`.

Server side:

1. Resolve `caller = Users.Single(u => u.IdentityProviderId == User.GetObjectId())`.
2. Authorize by scope:
   - `scope == "global"` → require `caller.Role == Admin` (`403` otherwise).
     The rule's `UserId` is `null`.
   - `scope == "user"` → for non-admin callers, require an active
     `UserClient` row for `(caller.Id, clientId)` (`403` otherwise). Admins
     are implicitly members of every client, mirroring
     [UserService.cs:62-68](../AltusAiAssistantApi/Services/UserService.cs#L62-L68).
     The rule's `UserId` is `caller.Id`.
3. Reject if the resulting active count would exceed the hard cap for the
   scope (§7).
4. Insert the rule. **No blob mutation, no index push** — `confluencePageId`
   / `confluenceAncestorIds` are populated by the existing sync + skillset.

**Out of scope for this ship:** an admin creating a per-user rule on behalf
of another user. If that need emerges, add a separate admin-only endpoint
with an explicit `targetUserId` — additive, doesn't compromise this
endpoint's spoof-proof model.

The admin UI supplies `targetValue` and `displayTitle` from a node click in
[`AdminController.GetBlobItems`](../AltusAiAssistantApi/Controllers/AdminController.cs#L98).
This depends on the hierarchy design's `BlobItemDto.ConfluenceId` /
`ConfluenceNodeType` fields shipping first (§8).

### 4.1. Listing the caller's effective rules

```
GET /api/ContentExclusions/ListEffective?clientId={clientId}
```

Returns the **effective rule set** for the caller — every global rule for
the client plus the caller's own per-user rules. The action name borrows
§5's "effective rule set" vocabulary so the URL signals scope rather than
silently filtering. The deferred admin-cross-user endpoint slots in
naturally as `ListAll`.

Same predicate as §5's effective rule set:

```sql
SELECT * FROM ContentExclusionRules
WHERE ClientId = @clientId
  AND InactiveDate IS NULL
  AND (UserId IS NULL OR UserId = @callerId)
ORDER BY CreatedDate DESC
```

Authorization mirrors §4: any authenticated caller; non-admins require an
active `UserClient` row for `clientId`; admins are implicit members of every
client.

Response — `scope` is derived from `UserId` so the client never needs to
know its own `userId`:

```json
[
  {
    "id":           <int>,
    "scope":        "global" | "user",
    "targetKind":   "ConfluenceSubtree",
    "targetValue":  "<confluenceId>",
    "displayTitle": "<snapshot title>",
    "createdDate":  "<iso8601>",
    "createdBy":    "<email>"
  }
]
```

Admins see only their own per-user rules through this endpoint — same view
as any other caller. A separate admin-only endpoint returning *all*
per-user rules for a client (for cross-user management) is out of scope for
this ship; add it alongside the §4 admin-targets-other-user endpoint when
the use case lands.

The list endpoint and the §5 effective-rule loader use the same predicate;
both should route through a single repository method so the two views stay
in lock-step.

### 4.2. Deleting a rule

```
POST /api/ContentExclusions/Delete/{ruleId}?clientId={clientId}
```

Hard delete — the row is removed via EF `Remove()`. Caps (§7) trivially
re-balance: deleting a rule frees a slot for the next create.

`clientId` is a named query parameter that acts as a defense-in-depth
safety check: the server fetches the rule with both `Id == ruleId` AND
`ClientId == clientId` in a single query, so a mismatch (or the wrong id
slipping into either slot) returns `404` without ever reaching the
authorization checks. Combined with the named-slot principle from §4 / §4.1,
this makes it impossible to delete a rule by accidentally swapping ids.

Authorization mirrors visibility from §4.1:

- **Global rule** (`UserId == null`) → require `caller.Role == Admin`.
- **Per-user rule** → require `rule.UserId == caller.Id`. Admins can delete
  their own rules but **not** other users' rules; cross-user admin cleanup
  is deferred to a separate admin-only endpoint alongside the §4 / §4.1
  admin variants.

A lookup that returns null yields `404`. Re-deleting via a stale UI list is
therefore a no-op from the client's perspective.

## 5. Effective rule set per query

[`ChatService`](../AltusAiAssistantApi/Services/ChatService.cs) loads the
effective rule set for the current request:

```sql
SELECT * FROM ContentExclusionRules
WHERE ClientId = @clientId
  AND InactiveDate IS NULL
  AND (UserId IS NULL OR UserId = @userId)
```

Cache per-user with a short TTL (e.g. 60s) — admin-driven rule edits should
feel near-immediate, but rules don't change per chat message.

## 6. Filter construction

Group rules by `TargetKind` and emit one clause per group. Today there's one
group; the structure admits more (§10).

```text
let subtreeIds = effectiveRules
                   .Where(r => r.Kind == "ConfluenceSubtree")
                   .Select(r => r.Value)

filter =
    not (
        search.in(confluencePageId, 'id1,id2,...')
        or confluenceAncestorIds/any(a: search.in(a, 'id1,id2,...'))
    )
```

Append to the existing Cognitive Search `filter` parameter with `and`.
`SearchClient.Search` already POSTs (~32 KB body limit) — 500 × 12-char ID ≈ 6 KB,
plenty of headroom.

Naturally-correct edge cases:

- **Non-Confluence docs** — `confluencePageId` null, `confluenceAncestorIds`
  empty → both clauses evaluate false, doc is kept.
- **Confluence doc with no matching rule** — both clauses false, doc is kept.
- **Empty effective rule set** — short-circuit and omit the filter entirely.
  Don't emit `not (false)` — it's wasted work and a syntax pitfall when the
  group is empty.

## 7. Rule count caps

Enforced in the service layer on create / reactivate. Counts only
`InactiveDate IS NULL` rows in the relevant scope.

| Scope | Soft cap | Hard cap |
|---|---|---|
| Per-user (active) | 50 | 200 |
| Global (active) | 100 | 500 |
| Combined per query (global ∪ user) | 150 | 500 |

- **Soft cap** → UI warning: *"A subtree rule excludes a page and all its
  descendants — prefer the highest applicable ancestor over many child rules."*
  Request still succeeds.
- **Hard cap** → API returns `400 Bad Request`. Existing rules beyond the cap
  (admin-grandfathered) keep working; new rules are refused until a rule is
  deactivated.

Budget: 500 × 12-char ID ≈ 6 KB filter — well under the 32 KB POST ceiling.

## 8. Admin UI implications

Rule creation hooks into the existing tree at
[`AdminController.GetBlobItems`](../AltusAiAssistantApi/Controllers/AdminController.cs#L98).
A Confluence node (page or folder) gets an "Exclude" action; the FE posts
`{ kind: "ConfluenceSubtree", value: node.ConfluenceId, displayTitle: node.Title }`.

`BlobItemDto` must expose:

- `ConfluenceId` — the rule's `value`.
- `ConfluenceNodeType` (`"page"` | `"folder"`) — drives icon and action label
  ("Exclude this page and all children" vs "Exclude this folder and all
  contents").

**Hard prerequisite**: these fields land via the hierarchy design. Currently
neither is on [`BlobItemDto`](../AltusAiAssistantApi/Models/BlobItemDto.cs);
the exclusion UI cannot ship before they do.
# Contact Roles — Per-Client List (Proposal)

Proposal to change how `ContactRole` is defined: from a compiled global enum to a **per-client, seedable, editable list served by the API**. Supersedes the "roles are a fixed enum" decision in [contacts-entity-model.md](contacts-entity-model.md) (§ContactRole) and [contacts-api.md](contacts-api.md) (§Roles enum). No implementation yet — this is a design for review.

## Why change

Roles were shipped as a fixed enum, shared across all clients, defined in code. The design doc even flagged the risk up front ([contacts-api.md:104-106](contacts-api.md): *"values change rarely… If/when this churns, add `GET /api/contacts/roles`"*). They have now churned twice:

1. **Original design** — 6 roles: `Consultant, Stakeholder, Executive, SubjectMatterExpert, Vendor, Partner`.
2. **Stakeholder #1** — grew the list in code to 15 + `Unknown`: added `SalesRep, MarketingRep, CustomerSupport, ITEngineering, ProductManager, BusinessAnalyst, RelationshipManager, Investor, Owner`.
3. **Stakeholder #2** (this request) — wants 8 more added, 2 relabeled — and these are *their* roles, not necessarily every client's.

Two stakeholders now want two different lists. That is the core problem: a single global enum forces every client onto one list, and every edit is a **code change across two repos + a deploy**.

### Current cost of a role change

The list is duplicated in **three** places that must be kept in lockstep:

| Location | File |
|---|---|
| Backend enum (persisted as string on `Contact.Role`) | [ContactRole.cs](../../altus-ai-assistant/AltusAiAssistantApi/Models/Contacts/ContactRole.cs) |
| Backend display-name map | [ContactRoleExtensions.cs](../../altus-ai-assistant/AltusAiAssistantApi/Models/Contacts/ContactRoleExtensions.cs) |
| Frontend union type **and** label/sort list | [Contacts.types.ts](../../altus-ai-assistant-fe/src/components/contacts/Contacts.types.ts), [Contacts.constants.ts](../../altus-ai-assistant-fe/src/components/contacts/Contacts.constants.ts) |

The role also feeds the LLM system prompt ([ContactsPromptService](../../altus-ai-assistant/AltusAiAssistantApi/Services/ContactsService.cs) and [DescriptionGenerationService.cs:68](../../altus-ai-assistant/AltusAiAssistantApi/Services/DescriptionGenerationService.cs#L68) via `Role.ToDisplayName()`), so the display label has real semantic weight — it's not just UI chrome. Notably, this argues *for* storing the resolved label directly on the contact (the prompt reads it with no lookup), not for normalizing it into a separate table.

## Scale sets the bar

A client directory is **~40–100 contacts** and a roles list is **~15–25 entries**. Referential-integrity machinery (a roles table, an FK from `Contact`, a backfill migration, join logic in the prompt/DTO) is disproportionate at this size. The design below is deliberately the *lightest* thing that removes the deploy-coupling and per-client divergence. A heavier normalized model is documented as an alternative for if requirements grow.

## Proposed model — per-client JSON list, string role on Contact

Store each client's roles list as a **serialized JSON blob**, seeded from a standard default set and edited as a whole. `Contact.Role` **stays the `string` it already is** in the DB (`nvarchar(32)`, [ContactConfiguration.cs:15](../../altus-ai-assistant/AltusAiAssistantApi/Data/ContactConfiguration.cs#L15)) — it holds the chosen role's label. No new FK, no Contact-row migration, no join at prompt time.

This reuses an existing pattern rather than inventing one: `Taxonomy` already persists a per-client serialized blob (`Taxonomy.SerializedData`, [Taxonomy.cs](../../altus-ai-assistant/AltusAiAssistantApi/Domain/Taxonomy.cs)) and is admin-managed and client-scoped — the same shape and operational model as this.

### Storage

A lightweight per-client config, mirroring `Taxonomy`:

```csharp
public class ContactRolesConfig : BaseEntity
{
    public int ClientId { get; private set; }
    public Client Client { get; private set; }
    public int SchemaVersion { get; private set; }      // shape of SerializedData; starts at 1
    public string SerializedData { get; private set; }  // JSON: ordered array of role labels
}
```

`SerializedData` is a **bare, ordered array of label strings** — array order *is* the display order, so no separate `SortOrder` field, and no per-element object wrapper:

```json
["Consultant","Stakeholder","Executive","Subject Matter Expert","Vendor","Partner",
 "Sales","Marketing","Customer Support","IT / Engineering","Product Manager",
 "Business Analyst","Relationship Manager","Investor","Owner"]
```

`SchemaVersion` carries the versioning that would otherwise need a JSON envelope. Keeping it a column lets the blob stay a dumb array today while leaving a clean, detectable upgrade path: if a future need forces a richer shape (e.g. per-role `description` or `group`, promoting entries from strings to objects), bump to `2` and migrate lazily on read — the reader switches on `SchemaVersion`, old rows keep working until touched. This is the cheap insurance for "strings are enough *for now*."

> **Placement choice:** a dedicated side row (as above) keeps the blob off the hot `Client` load path — roles are needed only on the contacts screens. A nullable column pair directly on `Client` is an equally acceptable alternative given the few-hundred-byte size; it trades a table for pulling the blob (and its version) on every `Client` fetch. Either works; the side row is marginally cleaner.

### What we keep and what we give up

| Concern | With this design |
|---|---|
| **Rename propagation** | *Not* automatic — a contact keeps the label it was saved with. Accepted: this is not a consistency-critical surface, and a rename that must sweep is a one-off bulk `UPDATE`. |
| **Delete-guard ("reassign first")** | Still available. When an admin removes a role, a `COUNT` of contacts holding that label (trivial at ~100 rows) drives a warn / block / bulk-reassign choice — an **app-level** check instead of an FK constraint. |
| **Prompt** | Simpler. `Contact.Role` already *is* the label; drop `ToDisplayName()` and read the string directly. |
| **Dangling label** | A contact may hold a label no longer in the list (role retired). Harmless — it's just display text; still renders and still feeds the prompt. Optionally surfaced in the editor as "N contacts use a retired role." |
| **`?role=` filter** | Unchanged — a `WHERE` on the string column, exactly as today. |

### Decisions taken (confirmed with stakeholders)

| Question | Decision |
|---|---|
| Who edits a client's list? | **Altus admins, per client** — same operational model as Taxonomy/Clients admin. No client-facing self-serve surface for v1. |
| "Remove Rep from Sales & Marketing"? | **Rename** — drop the word "Rep": `Sales Rep → Sales`, `Marketing Rep → Marketing`. Roles kept, relabeled. |
| Where do stakeholder #2's 8 new roles go? | **Only that stakeholder's client.** They are *not* added to the shared default seed — this is the whole point of going per-client. |
| Removing a role still used by contacts? | **App-level guard** — warn/block with a "reassign first" prompt driven by a `COUNT`. No orphaned or silently-nulled data, without an FK. |

### Standard default seed

The default list every **new** client is seeded from — the *only* place a list still lives in code, changing only when Altus shifts the baseline for future clients. Proposed baseline = today's 15 roles with the two relabels applied:

```
Consultant, Stakeholder, Executive, Subject Matter Expert, Vendor, Partner,
Sales, Marketing, Customer Support, IT / Engineering, Product Manager,
Business Analyst, Relationship Manager, Investor, Owner
```

`Unknown` stays a system fallback for ingested/unclassified contacts — represented as an empty/no-role string rather than a list entry, so it can't be deleted or reordered.

### The requesting client's list (stakeholder #2)

Applied as an **edit to that one client's blob**, not to the seed:

- **Rename** `Sales Rep → Sales`, `Marketing Rep → Marketing` (also folded into the default seed above, since it's a pure relabel).
- **Add (this client only):** `Legal / Compliance`, `Fulfillment`, `Operations`, `Supply Chain`, `Customer Success`, `Finance`, `Lab Testing`, `Services`.

Other existing clients are untouched. Any client can later request the same additions; it becomes a per-client edit, never a deploy.

## API

Two client-scoped endpoints under the existing contacts routes — a read for the FE picker and a whole-list save for the admin editor (mirrors the taxonomy save model, not per-row CRUD).

| Method | Route | Body | Returns | Notes |
|---|---|---|---|---|
| GET | `/api/clients/{clientId}/contacts/roles` | — | `string[]` | The client's ordered role list. Replaces the FE's embedded list. Lazily seeds the default set if the client has no config yet. |
| PUT | `/api/clients/{clientId}/contacts/roles` | `string[]` | `string[]` | Full replace of the list. Admin-only. Server validates: non-empty labels, case-insensitive dedupe, length ≤ 32 (matches the `Contact.Role` column). Returns the removed-but-still-referenced labels (with counts) so the FE can prompt for reassignment. |

Optionally a small helper for the delete-guard, if the FE wants counts before saving:

| Method | Route | Returns | Notes |
|---|---|---|---|
| GET | `/api/clients/{clientId}/contacts/roles/usage` | `{ label: string, count: int }[]` | Contacts-per-role, for the editor's "N contacts use this" hint. Cheap at this scale. |

No new DTO class is needed — the payload is a plain `string[]`. `ContactDto.Role` stays a `string` on both read and write; no `RoleId`, no `RoleLabel` denorm, **no breaking change** to the contacts endpoints.

Write access is admin-only (mirrors the role guard already on Taxonomy/Clients admin). Reads are available to any client member (the picker needs them).

## Frontend deltas

- Delete the embedded `ContactRole` union type and the `CONTACT_ROLE_LABELS` / `CONTACT_ROLES` constants ([Contacts.types.ts](../../altus-ai-assistant-fe/src/components/contacts/Contacts.types.ts), [Contacts.constants.ts](../../altus-ai-assistant-fe/src/components/contacts/Contacts.constants.ts)). `role` becomes a plain `string` fetched via React Query from `GET …/contacts/roles`.
- The edit-contact picker binds to the fetched list; a contact whose stored role is no longer in the list still displays it (show it as a selectable-but-flagged value).
- New **admin editor** for the list (add / rename / reorder / remove), analogous to the taxonomy editor. On remove of a still-used role, surface the count and offer bulk-reassign — the app-level equivalent of the Company `409` flow.

## Migration

We commit to **label-as-value** (see the Alternative section for why not keys), so existing `Contact.Role` values — which hold enum *keys* — must be **backfilled** to labels. Schema change is minimal; the one non-trivial step is a bounded, deterministic value rewrite.

1. **Schema:** add the `ContactRolesConfig` table (or the nullable column pair on `Client`). No column change to `Contact`.
2. **Seed:** write the default list into each existing client's config with `SchemaVersion = 1` (eagerly in a data migration, or lazily on first `GET`).
3. **Backfill `Contact.Role` (key → label).** One-time bulk `UPDATE`, using the existing `ToDisplayName()` map **with the two renames folded in**. Only the eight divergent keys change; the seven already-matching values and their renamed forms all land inside the seeded list:

   | Stored key | → new value |
   |---|---|
   | `SubjectMatterExpert` | `Subject Matter Expert` |
   | `SalesRep` | `Sales` |
   | `MarketingRep` | `Marketing` |
   | `CustomerSupport` | `Customer Support` |
   | `ITEngineering` | `IT / Engineering` |
   | `ProductManager` | `Product Manager` |
   | `BusinessAnalyst` | `Business Analyst` |
   | `RelationshipManager` | `Relationship Manager` |

   Edge cases: `Unknown`/empty → no-role (empty string); any value not in the map (shouldn't exist) is left untouched and logged. All target labels are ≤ 32 chars, so the existing `nvarchar(32)` column is unchanged. The rename applies to *every* client's contacts, since the relabel is folded into the default seed.
4. **Apply stakeholder #2's edits** to their client's blob only (the 8 additions — the renames are already in the seed).
5. **Cut over:** prompt/DTO read `Contact.Role` directly (drop `ToDisplayName()`); retire the `ContactRole` enum + `ContactRoleExtensions`; delete the FE embedded lists.

Run steps 2–3 in the **same release** that ships the config table so no contact is ever left holding an unmapped value. BE and FE deploy independently; if you want zero timing risk, keep a tolerant read (accept both key and label, prefer label) for one release, then delete it.

## Alternative — normalized roles table + FK (deferred)

The heavier model: a `ContactRoles` table (per-client rows) with `Contact.RoleId` as an FK, delete guarded by a DB `409`-on-reference (like `Company`), renames propagating automatically.

Rejected for v1 **on scale grounds**: it adds a table, an FK, a Contact-row backfill, and join logic in the DTO + prompt builders to guarantee a consistency that this surface doesn't require (40–100 contacts, a non-critical classification field). Revisit only if a real need appears — e.g. cross-client role analytics, guaranteed rename propagation as a product requirement, or per-role metadata (colors, permissions, grouping) that outgrows a flat string list.

## Open questions

- **Seed timing** — eager (data migration + on client creation in `AdminService`) vs. lazy (on first `GET`). Lazy is less code and pairs naturally with the `GET` endpoint; leaning lazy.
- **Storage placement** — dedicated `ContactRolesConfig` row (recommended) vs. a JSON column on `Client`. Decide based on whether we mind the blob on the `Client` load path.
- **Retired-role cleanup** — do we actively flag/clean contacts holding a label no longer in the list, or leave them until next edit? Leaning leave-and-flag.
- **Cross-client "apply to all"** — an admin action to push a role (e.g. `Legal / Compliance`) to every client at once. Out of scope for v1; likely follow-up.
- **Reassign-on-remove enforcement** — hard block vs. soft warning when removing a still-used role. Leaning soft warning given the low criticality.

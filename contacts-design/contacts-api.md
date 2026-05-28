# Contacts Feature — API Design (Draft)

HTTP contract for the Contacts directory, mapped from the entities in [contacts-entity-model.md](contacts-entity-model.md). Per-client scoping mirrors the existing Taxonomy pattern; conventions follow the repo guide in [../CLAUDE.md](../CLAUDE.md) (DTO suffix, `[Authorize]`, no `Async` method suffix).

## Conventions

- Base path: `/api/clients/{clientId}/...` — `clientId` is `int` (matches `BaseEntity.Id` and every existing controller, e.g. taxonomy in `AdminController`); authorization middleware verifies caller belongs to the client.
- All endpoints `[Authorize]`. No anonymous reads.
- IDs: `int` for `Contact`, `ContactIdentity`, `Company` (consistent with `BaseEntity` usage in the repo).
- Enums serialized as strings.
- All entities inherit `BaseEntity` (`CreatedBy/Date`, `UpdatedBy/Date`, `InactiveDate`). `Contact` and `Company` soft-delete (via `InactiveDate`); `ContactIdentity` hard-deletes.
- JSON casing: camelCase (matches existing API).
- Errors: standard `ProblemDetails`. Notable status codes called out per endpoint below.

## Enums

```csharp
public enum ContactRole {
    Consultant,
    Stakeholder,
    Executive,
    SubjectMatterExpert,
    Vendor,
    Partner,
    // extend in code; FE embeds the list
}

public enum IdentitySource {
    Unknown,
    Grain,
    Slack,
    Confluence,
    Email,
    Phone,
}
```

## DTOs

`ContactDto` is the write shape (also returned by POST/PUT). `ContactExtendedDto` adds the read-only denormalized `CompanyName` and embedded `Identities`, and is returned by GET only. Server-set fields (`Id`, `CreatedDate`, `UpdatedDate`) are populated on read and ignored on write. PUT is a full replace, not a partial patch.

Identity assignment is **not** done through the contact endpoints — neither POST nor PUT accepts `Identities`. Assign/unassign via `PUT /identities/{id}` with the desired `ContactId`.

```csharp
public class ContactDto {
    public int Id { get; set; }                                 // server-set; ignored on write
    public string Name { get; set; } = string.Empty;
    public string Description { get; set; } = string.Empty;
    public int? CompanyId { get; set; }                         // null = no company (PUT with null clears it)
    public ContactRole Role { get; set; }
    public DateTime CreatedDate { get; set; }                   // server-set; ignored on write
    public DateTime UpdatedDate { get; set; }                   // server-set; ignored on write
}

public class ContactExtendedDto : ContactDto {
    public string? CompanyName { get; set; }                    // read-only denorm
    public List<ContactIdentityDto> Identities { get; set; } = []; // read-only; full list of identities owned by this contact
}

public class ContactIdentityDto {
    public int Id { get; set; }
    public IdentitySource Source { get; set; }                  // immutable after create — server rejects changes on PUT
    public string SourceKey { get; set; } = string.Empty;       // immutable after create — server rejects changes on PUT
    public string? DisplayLabel { get; set; }
    public int? ContactId { get; set; }                         // null = unassigned; PUT with null unassigns
    public DateTime CreatedDate { get; set; }
}

public class CompanyDto {
    public int Id { get; set; }
    public string Name { get; set; } = string.Empty;
    public string RelationDescription { get; set; } = string.Empty;
}
```

## Endpoints

### Contacts

| Method | Route | Body | Returns | Notes |
|---|---|---|---|---|
| GET | `/api/clients/{clientId}/contacts` | — | `ContactExtendedDto[]` | Query: `?q=`, `?companyId=`, `?role=`. Identities embedded; typical client has dozens-to-low-hundreds of contacts. |
| POST | `/api/clients/{clientId}/contacts` | `ContactDto` | `ContactDto` (201) | Server ignores `Id`/timestamps. 409 on name+company duplicate (optional uniqueness — see Open Questions). Identity assignment goes through the identity endpoints. |
| PUT | `/api/clients/{clientId}/contacts/{id}` | `ContactDto` | `ContactDto` | Full replace of `Name`, `Description`, `CompanyId`, `Role`. `CompanyId = null` clears the company. Identity assignment goes through the identity endpoints. |
| DELETE | `/api/clients/{clientId}/contacts/{id}` | — | 204 | Soft-delete. Cascades to **detach** identities (`ContactId = null`); identities are not deleted. Merging two contacts is done client-side: PUT each identity's `ContactId` to the survivor, then DELETE the empty source. |

### Identities

| Method | Route | Body | Returns | Notes |
|---|---|---|---|---|
| GET | `/api/clients/{clientId}/identities` | — | `ContactIdentityDto[]` | Query: `?unassigned=true`, `?contactId=`, `?source=`, `?q=` (matches `SourceKey` or `DisplayLabel`). The "Unassigned" panel uses `?unassigned=true`. |
| POST | `/api/clients/{clientId}/identities` | `ContactIdentityDto` | `ContactIdentityDto` (201) | Hand-entered identities only. `ContactId` optional (assigns on create). 409 on duplicate `(clientId, source, sourceKey)`. Sync workers use the internal service, not this endpoint. |
| PUT | `/api/clients/{clientId}/identities/{id}` | `ContactIdentityDto` | `ContactIdentityDto` | Full replace of `DisplayLabel` and `ContactId`. `ContactId = null` unassigns. `Source`/`SourceKey` immutable — server returns 400 if changed. 400 if `ContactId` not in client. |

### Companies

| Method | Route | Body | Returns | Notes |
|---|---|---|---|---|
| GET | `/api/clients/{clientId}/companies` | — | `CompanyDto[]` | Powers the company dropdown. |
| POST | `/api/clients/{clientId}/companies` | `CompanyDto` | `CompanyDto` (201) | 409 on duplicate `(clientId, name)` (case-insensitive). |
| PUT | `/api/clients/{clientId}/companies/{id}` | `CompanyDto` | `CompanyDto` | Full replace of `Name`, `RelationDescription`. |
| DELETE | `/api/clients/{clientId}/companies/{id}` | — | 204 | Soft-delete. **409 if referenced** by any non-deleted Contact — caller must reassign first. Avoids ghost-company state. |

### Roles enum

Embedded in the FE — values change rarely and the UI already hardcodes the list. No HTTP endpoint. If/when this churns, add `GET /api/contacts/roles → string[]` (not client-scoped).

## Status code matrix

| Code | When |
|---|---|
| 200 | Successful GET / PUT / merge |
| 201 | Successful POST create (with `Location` header) |
| 204 | Successful DELETE |
| 400 | Validation failure, cross-client reference, attempt to mutate immutable identity fields (`Source`/`SourceKey`) |
| 401 | No auth |
| 403 | Authenticated but not a member of `{clientId}` |
| 404 | Resource not in client |
| 409 | Uniqueness conflict (identity `(source, sourceKey)`, company `name`) or Company-delete with refs |

## Service surface (internal)

```csharp
public interface IContactsDirectoryService {
    // Public HTTP layer
    Task<List<ContactExtendedDto>>  ListContacts(int clientId, ContactQuery q, CancellationToken ct);
    Task<ContactDto>                CreateContact(int clientId, ContactDto dto, string userId, CancellationToken ct);
    Task<ContactDto>                UpdateContact(int clientId, int id, ContactDto dto, string userId, CancellationToken ct);
    Task                            DeleteContact(int clientId, int id, string userId, CancellationToken ct);

    Task<List<ContactIdentityDto>>  ListIdentities(int clientId, IdentityQuery q, CancellationToken ct);
    Task<ContactIdentityDto>        CreateIdentity(int clientId, ContactIdentityDto dto, string userId, CancellationToken ct);
    Task<ContactIdentityDto>        UpdateIdentity(int clientId, int id, ContactIdentityDto dto, string userId, CancellationToken ct);

    Task<List<CompanyDto>>          ListCompanies(int clientId, CancellationToken ct);
    Task<CompanyDto>                CreateCompany(int clientId, CompanyDto dto, string userId, CancellationToken ct);
    Task<CompanyDto>                UpdateCompany(int clientId, int id, CompanyDto dto, string userId, CancellationToken ct);
    Task                            DeleteCompany(int clientId, int id, string userId, CancellationToken ct);

    // Internal — called by sync workers (Confluence, Slack, Mail, Grain). NOT exposed via HTTP.
    Task<ContactIdentityDto>             UpsertObservedIdentity(
        int clientId,
        IdentitySource source,
        string sourceKey,
        string? displayLabel,
        CancellationToken ct);
}
```

`UpsertObservedIdentity` is the seam ingestion uses: on every sync pass, workers report observed identities and the service inserts new ones (unassigned) or returns the existing row. The chat completion path in [AltusAiAssistantApi/Services/ChatService.cs](../AltusAiAssistantApi/Services/ChatService.cs) then reads the assigned `Contact` via this service when building the system prompt.

## Behavioral details

1. **Identity natural key.** `(ClientId, Source, SourceKey)` is unique. Sync workers upsert by this tuple; the public POST returns 409 on collision.
2. **Identity ownership of source data.** `SourceKey` is opaque to the directory. Workers (`SlackSyncService`, `ConfluenceSyncService`, `MailSyncService`, Grain ingest) write through `UpsertObservedIdentity`, never the HTTP endpoint.
3. **Contact deletion.** Soft-delete only. Cascades to detach identities (back to unassigned) rather than deleting them — identities are the durable link to ingested content and must survive a directory cleanup.
4. **Merge.** No dedicated endpoint. Client reassigns each identity via PUT then DELETEs the empty source contact. Re-evaluate if this becomes hot or needs to be atomic.
5. **Identity deletion.** No endpoint. Sync-ingested identities persist; manual identities created in error can be unassigned (`ContactId = null`) and left in the unassigned panel.
6. **Company delete.** Refused with 409 when referenced. Forces explicit reassignment, which is what the user wants anyway.
7. **No "primary identity."** Reflects the entity model: identities are source-system pointers; the Contact carries display data. The FE no longer needs `mainId` / "promote member" affordances.
8. **Read at chat time.** No index dependency. Chat builds the directory snapshot from the DB on each call; edits are immediately visible without reprocessing ingested content.
9. **Pagination.** Omitted for v1 — directory sizes are bounded. Add `?skip=&take=` later if needed.

## Frontend deltas

The current FE in [altus-ai-assistant-fe/src/components/contacts/](../../altus-ai-assistant-fe/src/components/contacts/) doesn't match the new model. Tracked here for the FE work that follows backend implementation:

- `Contact.role: string[]` → single `ContactRole` enum value.
- `LinkedGroup { mainId, members }` collapses into "one Contact, many ContactIdentities." Remove linking UI, "promote member," "main contact" picker.
- `Contact.email` / `phone` / `company` (strings) → derived. Emails and phones become identities of source `Email` / `Phone`. Company is `companyId` + denormalized `companyName`.
- `UnassignedUser` → `ContactIdentity` with `contactId = null`. Single endpoint with a query flag.
- `Contact.source` disappears — a Contact aggregates many sources via its identities.
- Replace the Zustand mock store in [Contacts.state.ts](../../altus-ai-assistant-fe/src/components/contacts/Contacts.state.ts) with React Query against the endpoints above, using the existing axios client in [axiosInstance.ts](../../altus-ai-assistant-fe/src/libs/helpers/axiosInstance.ts).

## Open questions

- Whether to enforce uniqueness on `(ClientId, Contact.Name, CompanyId)` to prevent accidental dup contacts, or leave that as a soft warning in the UI.
- Whether `ContactIdentity` should carry a "suggested contact" hint computed by the sync workers (mirrors the same open question in the entity model).
# Disabling Public Blob Access — What It Would Take

_Date: 2026-08-25_
_Scope: `altus-ai-assistant` (API), `altus-ai-assistant-fe`, `altus-ai-terragrunt`, plus a one-time pass over existing Azure containers._
_Status: assessment + plan. Nothing here is implemented._

## Summary

Every per-client content container is created with `PublicAccessType.Blob`
([BlobStorageService.cs:209](../altus-ai-assistant/AltusAiShared/BlobStorageService.cs#L209)),
so **every indexed client document is readable by anyone who has its URL, with no
token, no role check, and no client-membership check.**

The Azure Search indexer does **not** depend on this. Four other things do, one of
them a third party. Closing it is a five-step sequence across three repos, and the
steps are strictly ordered — flipping the storage setting first breaks client
creation outright.

Rough sizing: **2–4 days of work**, plus one product decision about `.docx`
preview that is worth making regardless of this effort.

## 1. Today's posture

| Layer | Setting | Where |
|---|---|---|
| Storage account | `allow_nested_items_to_be_public = true` | [modules/assistant-services/main.tf:26](../altus-ai-terragrunt/modules/assistant-services/main.tf#L26) |
| Container ACL | `PublicAccessType.Blob` on create | [BlobStorageService.cs:209](../altus-ai-assistant/AltusAiShared/BlobStorageService.cs#L209) |
| Storage CORS | `GET/HEAD/OPTIONS` from SPA origins | [main.tf:41-48](../altus-ai-terragrunt/modules/assistant-services/main.tf#L41-L48) |

The Terraform is candid about why — `# API CreateContainer uses PublicAccessType.Blob
(legacy Ana behavior)` and `# SPA file preview (react-pdf / fetch) loads blobs
cross-origin`. The new instance is provisioned to inherit the same posture, so
this is not a legacy-only problem; it is being carried forward.

Blob URLs reach the browser routinely: `url` is on every citation
([KnowledgeRetriever.cs:104-106](../altus-ai-assistant/AltusAiAssistantApi/Services/Maf/KnowledgeRetriever.cs#L104-L106))
and on every file-tree node ([AdminService.cs:419](../altus-ai-assistant/AltusAiAssistantApi/Services/AdminService.cs#L419)).
A leaked URL — browser history, a pasted link, a proxy log, a screenshot — is a
permanent unauthenticated read of that client's document. Content rules and Focus
Mode are query-time search filters ([ContentRuleService.cs:372-405](../altus-ai-assistant/AltusAiAssistantApi/Services/ContentRuleService.cs#L372-L405));
they have no bearing on a direct blob GET.

## 2. Dependency inventory

### 2.1. Azure Search indexer — **no dependency**

The data source is created with an account-key connection string
([IndexingService.cs:138-150](../altus-ai-assistant/AltusAiAssistantApi/Services/IndexingService.cs#L138-L150)).
Azure Search authenticates with that key and never uses the anonymous path.
Indexing, skillsets, and change detection are unaffected by anything in this document.

### 2.2. Backend document readers — **hard dependency, and accidental**

Every `IFileReader` downloads by unauthenticated HTTP:

```csharp
using HttpClient client = new HttpClient();
return await client.GetByteArrayAsync(url);   // no credential, no SAS
```

- [PdfFileReader.cs:23](../altus-ai-assistant/AltusAiAssistantApi/Services/PdfFileReader.cs#L23)
- [TextFileReader.cs:17](../altus-ai-assistant/AltusAiAssistantApi/Services/TextFileReader.cs#L17)
- [DocxFileReader.cs:18](../altus-ai-assistant/AltusAiAssistantApi/Services/DocxFileReader.cs#L18)
- [HtmlFileReader.cs:10](../altus-ai-assistant/AltusAiAssistantApi/Services/HtmlFileReader.cs#L10)

Two live call sites:

- [ChatService.cs:131](../altus-ai-assistant/AltusAiAssistantApi/Services/ChatService.cs#L131) — deep-dive file sessions
- [KnowledgeRetriever.ReadWholeDocument](../altus-ai-assistant/AltusAiAssistantApi/Services/Maf/KnowledgeRetriever.cs#L213-L227) — whole-document pull

This one is pure accident. The API already holds the account key and already has
[`GetBlobStream`](../altus-ai-assistant/AltusAiShared/BlobStorageService.cs#L343) —
and the rest of the codebase uses it correctly
([IdentityDiscoveryRunner.cs:130](../altus-ai-assistant/AltusAiProcessContent/IdentityDiscoveryRunner.cs#L130),
[SlackSyncService.cs:138](../altus-ai-assistant/AltusAiShared/Slack/SlackSyncService.cs#L138)).
The readers are the outlier, and fixing them is worthwhile on its own merits.

### 2.3. Frontend direct fetches — **hard dependency**

The SPA reads blob content from the browser with a bare `fetch(url)` — no MSAL
token, no API involved ([file.ts:9](../altus-ai-assistant-fe/src/libs/data/file.ts#L9)):

| Consumer | Mechanism | File |
|---|---|---|
| Text / CSV / HTML / Markdown preview | `fetch(url).text()` | [Txt-preview.tsx:9](../altus-ai-assistant-fe/src/libs/shared/file-preview/Txt-preview/Txt-preview.tsx#L9), [Csv-preview.tsx:20](../altus-ai-assistant-fe/src/libs/shared/file-preview/Csv-preview/Csv-preview.tsx#L20), [Html-preview.tsx:10](../altus-ai-assistant-fe/src/libs/shared/file-preview/Html-preview/Html-preview.tsx#L10), [Markdown-preview.tsx:32](../altus-ai-assistant-fe/src/libs/shared/file-preview/Markdown-preview/Markdown-preview.tsx#L32) |
| PDF preview | react-pdf `<Document file={url}>` (issues range requests) | [Pdf-preview.tsx:93](../altus-ai-assistant-fe/src/libs/shared/file-preview/Pdf-preview/Pdf-preview.tsx#L93) |
| Download buttons | `<a href={url} download>` | [Node.tsx:171](../altus-ai-assistant-fe/src/libs/shared/Tree-view/Node.tsx#L171), [FlatFilesList.tsx:139](../altus-ai-assistant-fe/src/libs/shared/flat-files-list/FlatFilesList.tsx#L139), [File-preview.tsx:166](../altus-ai-assistant-fe/src/libs/shared/file-preview/File-preview.tsx#L166) |

### 2.4. Microsoft Office Web Viewer — **the hard one**

`.docx` preview embeds Microsoft's hosted viewer
([Doc-preview.tsx:6-8](../altus-ai-assistant-fe/src/libs/shared/file-preview/Doc-preview/Doc-preview.tsx#L6-L8)):

```tsx
const viewerUrl = `https://view.officeapps.live.com/op/embed.aspx?src=${encodeURIComponent(file.fileUrl)}`;
```

**Microsoft's servers fetch that URL from the public internet.** This is the one
consumer where "proxy it through the API" cannot work — the fetcher is a third
party, not the user's browser, and it presents no credential.

Two consequences worth separating:

1. **It blocks the flip.** Private container ⇒ docx preview returns an error page,
   unless the viewer is handed a time-limited SAS URL instead.
2. **It is already an egress issue, independent of this work.** Every docx preview
   today ships a client document to a Microsoft consumer service. Making the
   container private does not fix that; it only narrows who *else* can read it.
   See §5.

## 3. What is *not* a dependency

**Blob URLs as identifiers.** URLs are used as primary keys throughout —
`ChatMessage.CuratedMessageFileUrl` is matched by URL string
([AdminService.cs:448-452](../altus-ai-assistant/AltusAiAssistantApi/Services/AdminService.cs#L448-L452)),
`dto.FileUrls` identifies deep-dive files, and
[`GetBlobName(url)`](../altus-ai-assistant/AltusAiShared/BlobStorageService.cs#L25)
converts a URL back to a blob path.

None of that has to change. The URLs can stay exactly as they are, in DTOs and in
the database. The requirement is only that **nothing dereferences them
anonymously**. This keeps the change small and avoids a data migration.

## 4. The plan

The order is a hard constraint, not a preference. In particular, step 5 before
step 1 breaks client creation: with `allowBlobPublicAccess = false` at the account,
`CreateIfNotExistsAsync(PublicAccessType.Blob)` fails with `PublicAccessNotPermitted`,
and `CreateIndex` calls it on every new client
([IndexingService.cs:40-44](../altus-ai-assistant/AltusAiAssistantApi/Services/IndexingService.cs#L40-L44)).

### Step 1 — Backend readers move to the SDK  ·  ~2h  ·  no user-visible change

Change `IFileReader.Read` to take a container + blob path (or keep the URL and
resolve it via the existing `GetBlobName`) and pull bytes through
`BlobStorageService.GetBlobStream` instead of `HttpClient`. Four readers, two call
sites. Ship this independently — it is a correctness fix that removes an
unauthenticated network hop from a server that already holds the key, and it
unblocks everything downstream.

### Step 2 — Authorized read-back endpoint  ·  ~1d

New endpoint, e.g. `GET /clients/{clientId}/files/content?path=…`, which checks
client membership and content rules, then returns access to the bytes. Two
mechanisms, and the choice matters:

| | Stream through the API | Mint a short-lived SAS |
|---|---|---|
| Auth | Bearer token, already wired | Bearer on the mint call only |
| Bandwidth | Through App Service / AKS | Direct browser ↔ storage |
| PDF range requests | Must implement `Range` support | Native |
| Storage CORS | Can be removed | Still required |
| Leak window | None | SAS lifetime (5–15 min) |
| Works for Office viewer | **No** | **Yes** |

**Recommendation: mint a SAS and return it as JSON** (`{ url, expiresAt }`), not a
302 redirect. JSON suits all three FE shapes — `fetch().text()`, react-pdf's `file`
prop, and an iframe `src` — whereas a redirect has to survive a cross-origin hop
that drops the `Authorization` header and needs its own CORS preflight handling.

Note there is **no SAS infrastructure in the codebase today**. The account has
`shared_access_key_enabled = true`, so account-key SAS works immediately;
user-delegation SAS (preferable, no long-lived secret) would require managed
identity on the API, which the AKS migration may bring anyway — worth checking
before building this twice.

### Step 3 — Frontend repoints to the endpoint  ·  ~1d

Replace `useFetchFileContentByUrl`'s bare `fetch(url)` with a call that mints
first, then fetches. Same for the three `<a href download>` sites and react-pdf's
`file` prop. Cache minted URLs by path for their lifetime so opening a preview
doesn't re-mint per render.

### Step 4 — Decide `.docx` preview  ·  see §5

### Step 5 — Flip the container ACL

Three sub-parts, all cheap, none reversible by accident:

1. `CreateContainer` → `PublicAccessType.None`
   ([BlobStorageService.cs:209](../altus-ai-assistant/AltusAiShared/BlobStorageService.cs#L209)).
   **This only affects new containers** — `CreateIfNotExistsAsync` is a no-op for
   an existing container, so every current client keeps its public ACL until
   something explicitly resets it.
2. One-time pass over existing containers (`SetAccessPolicyAsync` /
   `Set-AzStorageContainerAcl`). A PowerShell script in
   [scripts/](../altus-ai-assistant/scripts/) matches the existing pattern —
   `migrate-sourcetype.ps1` and `update-search-indexes.ps1` are the precedents.
   Instant and individually reversible per container.
3. `allow_nested_items_to_be_public = false` in
   [modules/assistant-services/main.tf](../altus-ai-terragrunt/modules/assistant-services/main.tf#L26)
   as the backstop, so a future regression in `CreateContainer` cannot silently
   re-open it. Drop the `cors_rule` block at the same time **only if** step 2 chose
   API streaming; with SAS it is still needed.

## 5. The `.docx` question

Independent of public access, and worth surfacing to whoever owns the client
confidentiality story:

- **(a) SAS URL to the Office viewer** — smallest change, preview keeps working
  unchanged. Client documents still go to Microsoft's consumer viewer service, now
  via a time-boxed link rather than a permanent public one.
- **(b) Render docx client-side** (`docx-preview` / `mammoth.js`) from bytes the
  browser already has permission to fetch. Nothing leaves the tenant. Moderate FE
  work, some fidelity loss on complex layouts. **Recommended** — it closes an
  egress path that (a) leaves open.
- **(c) Server-side text render** using the existing
  [`DocxFileReader`](../altus-ai-assistant/AltusAiAssistantApi/Services/DocxFileReader.cs)
  and display as text. Cheapest, largest fidelity loss.

This is a product call, not an engineering one. It is called out here because the
public-access work forces the question rather than because it caused it.

## 6. Verification

After step 5, the check is a single anonymous request:

```bash
curl -I "https://<account>.blob.core.windows.net/<clientUniqueId>/<some-path>"
# expect 404 (Azure returns 404, not 403, for anonymous reads on a private container)
```

Then exercise, per environment:

- Confluence / Slack / email sync writes and a successful indexer run
- Chat with citations, and a citation click-through
- Deep-dive file session (exercises step 1)
- Whole-document pull (exercises step 1)
- Preview for each of `.pdf`, `.txt`, `.csv`, `.html`, `.md`, `.docx`
- Download button in tree view and flat file list
- **New client creation** — the `CreateIndex` path that calls `CreateContainer`

## 7. Adjacent decisions

- **Content images.** [content-image-indexing-design.md](./content-image-indexing-design.md)
  stores images extracted from documents in the client's content container, next to
  their source document (`{documentPath}.images/…`). They inherit the container's
  access level: public today, private after step 5, with no extra work. Because they
  are visible in the file tree, the FE image preview must move to the step 2 read-back
  endpoint along with the other previews. Answer-time access already uses SAS links.
- **Chat attachments.** The chat image container is separate and is already created
  with `PublicAccessType.None`, outside `CreateContainer`.
- **New instance.** The Terraform currently provisions the new environment with
  the same public posture. If the flip is going to happen, doing it *before* the
  new instance carries data is materially cheaper than after — step 5.2 shrinks to
  nothing.

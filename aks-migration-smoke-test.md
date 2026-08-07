# AKS Migration — Smoke Test Checklist

Post-cutover verification for ANA running on the AKS cluster (API + worker functions + frontend
containers, Argo CD–deployed) instead of App Service / Function Apps.

**This is not a regression suite.** Each item answers one question: *does this feature work at all
in the new environment?* Depth of coverage is deliberately shallow — one representative path per
feature, nothing exhaustive. If a feature was broken before the migration, it is out of scope.

Most items carry a **Breaks if** note. That is the point of the document: the migration changes
hosting, networking, TLS, identity, storage endpoints, and process lifecycle all at once, and the
failures it produces are infrastructure-shaped, not feature-shaped. The notes say where to look
when a test fails.

Related: [new-instance-provisioning.md](new-instance-provisioning.md) (§4 covers the containerization
gotchas these tests probe).

---

## How to use

- Run **Tier 0 first**. If the platform tier fails, everything below it fails for the same reason and
  the results are noise.
- Tiers 1–2 are the **release gate** — chat over indexed content is the product. Everything else is
  degraded-but-shippable.
- Record every result in the log at the bottom, including the ones you skipped and why.
- Where a test says "compare to QA", the point is to distinguish *"the migration broke it"* from
  *"it never worked"*. Do that comparison before filing a bug.

**Environment under test**

| | |
|---|---|
| Frontend | `https://assistant.dev.altusnova.ai` |
| API | `https://assistant-api.dev.altusnova.ai` |
| Cluster | `aks-infra-dev-eastus` (private API server — Tailscale required for `kubectl`) |
| Namespace | `altus-ai-assistant` |
| Registry | `acraltusdeveastus.azurecr.io` |

---

## Preconditions

Not tests — if any of these is false, stop and fix it, because the results below will be
meaningless.

- [ ] **P1** — All Argo CD Applications report `Synced` / `Healthy`.
- [ ] **P2** — Every pod in `altus-ai-assistant` is `Running` with `0` restarts after 10 minutes of
      idle. A restart loop invalidates every timing-dependent test below.
- [ ] **P3** — The `altus-ai-assistant-api` Kubernetes Secret exists and is fully populated (SQL,
      Search, OpenAI, SignalR, Graph client secret). Missing keys surface as runtime 500s, not as
      startup failures.
- [ ] **P4** — Database schema is migrated and reference data is seeded: at least one client, one
      user with a client assignment, and a taxonomy. **There is no seeding mechanism** — an empty
      database will make most of Tier 2 onward untestable.
- [ ] **P5** — Confirm which Azure OpenAI / Cognitive Search / SQL / Storage accounts the pods are
      actually pointed at, from the running pod's environment — not from the chart values.
- [ ] **P6** — A test user account exists in the target Entra tenant with a client assignment.

---

## Tier 0 — Platform reachability

Nothing here is a product feature. These fail loudly and explain everything downstream.

- [ ] **T0.1 — Frontend loads over HTTPS.** `https://assistant.dev.altusnova.ai` returns the app
      shell with a valid, non-self-signed certificate covering the host.
      *Breaks if:* cert-manager's DNS-01 solver can't reach the `altusnova.ai` zone — its Azure
      workload identity and the DNS Zone Contributor role assignment are hardcoded in the Argo
      manifest and not created by Terraform.

- [ ] **T0.2 — API responds over HTTPS.** `GET https://assistant-api.dev.altusnova.ai/` returns the
      Admin controller's index response, not a 404 from the ingress default backend.
      *Breaks if:* ingress host rules or the wildcard TLS secret are wrong for the API host.

- [ ] **T0.3 — SPA deep-link refresh.** Navigate to a nested route (a chat session URL), press F5.
      The app reloads; no 404.
      *Breaks if:* the nginx container is missing the SPA fallback rewrite. This worked implicitly
      on App Service via `web.config`.

- [ ] **T0.4 — CORS.** The browser makes cross-origin XHR from the FE host to the API host with no
      CORS console errors. The API's allowed origins must list the new FE host exactly.

- [ ] **T0.5 — Egress works.** From an API pod, reach an external endpoint (`login.microsoftonline.com`,
      `graph.microsoft.com`).
      *Breaks if:* NAT gateway or NSG misconfiguration. All outbound traffic now leaves from one
      fixed NAT public IP — third-party allowlists (Confluence, Slack, SQL firewall) must include it.

- [ ] **T0.6 — Database connectivity.** Any endpoint that reads from SQL returns data.
      *Breaks if:* the AKS NAT egress IP is not in the SQL firewall. Also note the database is
      **serverless with 60-minute auto-pause** — the first request after idle can take 30–60s and
      time out. Retry once before reporting a failure.

- [ ] **T0.7 — Images pull from ACR.** No `ImagePullBackOff` on any deployment.
      *Breaks if:* the kubelet identity lost its `AcrPull` role assignment.

---

## Tier 1 — Authentication and identity

- [ ] **T1.1 — Login.** MSAL redirect completes and returns to the app authenticated.
      *Breaks if:* the new FE host is not registered as a redirect URI on the SPA app registration.
      Note the FE bakes MSAL config in at **image build time** — a wrong client ID or authority
      needs a rebuild, not a config change.

- [ ] **T1.2 — Token accepted by the API.** An authenticated API call returns 200, not 401.
      *Breaks if:* audience mismatch. The API's JWT audience and the SPA's requested scope must
      point at the same registration — this was mid-migration between the legacy and new
      registrations, so verify which one is live rather than assuming.

- [ ] **T1.3 — Logout / re-login.** Session clears and a fresh login succeeds.

- [ ] **T1.4 — User profile resolves.** The signed-in user's clients load and the client selector
      is populated.

- [ ] **T1.5 — Admin-only surfaces gate correctly.** A non-admin user does not see admin routes
      (Clients, Users, Metrics); an admin does.
      *Known deviation:* if `BypassGraphProvisioning` is enabled, **every first-time login is
      provisioned as Admin** and profile photos return empty. Confirm the flag's value before
      calling either behaviour a bug — and flag it if the environment is meant to be shared.

- [ ] **T1.6 — Graph-dependent provisioning.** A user who has never logged in before is created
      correctly on first login (with the Graph lookup path, not the bypass path).
      *Breaks if:* the Graph daemon client secret is missing or unconsented in the target tenant.

---

## Tier 2 — Core chat path

The release gate. If this tier does not pass, the migration is not done.

- [ ] **T2.1 — Create a session.** New chat session is created and appears in the session list.

- [ ] **T2.2 — Send a message and get a streamed answer.** Response text arrives **incrementally**,
      not as one block at the end.
      *Breaks if:* SignalR. This is the single highest-risk item in the migration. The API now runs
      **2 replicas**, and `Program.cs` silently falls back to in-process SignalR when the Azure
      SignalR connection string is absent or set to a placeholder. In-process SignalR across two
      replicas without a backplane produces exactly this symptom: sporadic missing tokens,
      responses that never arrive, or answers that only appear on refresh. **Verify the pod is
      actually using Azure SignalR**, and run this test at least five times to catch the
      intermittent case.

- [ ] **T2.3 — Long response does not truncate.** Ask a question that generates a long answer
      (60s+ of streaming).
      *Breaks if:* the ingress `proxy-read-timeout` (120s) cuts the connection mid-stream. This
      limit did not exist on App Service.

- [ ] **T2.4 — WebSocket upgrade survives ingress.** Confirm in browser dev tools that the hub
      connection is a WebSocket, not a fallback to long-polling.
      *Breaks if:* ingress-nginx is not passing upgrade headers. Long-polling may still "work"
      while being slow and fragile — this test exists to catch a silent degradation.

- [ ] **T2.5 — Answer cites retrieved sources.** The response includes citations/references, proving
      the RAG retrieval path reached Cognitive Search — not just that the model replied.

- [ ] **T2.6 — Open a cited source.** Click through to a citation; the document opens.
      *Breaks if:* `BlobUrlPrefix` still points at the old storage account, or the new storage
      account's container access level differs.

- [ ] **T2.7 — Session history persists.** Reload the page, reopen the session; the full message
      history is intact.

- [ ] **T2.8 — Rename a session.**

- [ ] **T2.9 — Delete a session.**

- [ ] **T2.10 — Session list loads** for the selected client, scoped to that client only.

- [ ] **T2.11 — Client switching.** Switch to a second client; sessions and content change with it.
      Verifies index-per-client resolution still works against the new Search service.

- [ ] **T2.12 — Curated message.** Create a curated message on a session.

- [ ] **T2.13 — File-based session (deep dive).** Start a session scoped to a specific document and
      ask a question about it.

- [ ] **T2.14 — Image in chat.** Attach an image to a message and get a response referencing it.
      *Breaks if:* ingress `proxy-body-size` (50m) rejects the upload, or blob write fails.

- [ ] **T2.15 — Content focus / exclusion rules.** Create a rule, confirm it takes effect on the
      next answer, then delete it.

---

## Tier 3 — Content and administration

- [ ] **T3.1 — Browse client content.** The blob file tree loads for a client.
      *Breaks if:* storage account name/key in the secret is wrong, or containers were never
      created in the new account.

- [ ] **T3.2 — Upload a document.** Upload a PDF of a few MB; it appears in the tree.
      *Breaks if:* body-size limits at ingress. Test a file large enough to be realistic, not a
      1 KB text file.

- [ ] **T3.3 — Preview a document.** Open the file preview; content renders.

- [ ] **T3.4 — Create a folder.**

- [ ] **T3.5 — Move a document between folders.**

- [ ] **T3.6 — Delete a document.**

- [ ] **T3.7 — Taxonomy loads and saves.** View the taxonomy for a client, edit it, save, reload,
      confirm persistence.

- [ ] **T3.8 — Contacts list loads.**

- [ ] **T3.9 — Create / edit / delete a contact.**

- [ ] **T3.10 — Contact roles.** Load the per-client role list and save a change.

- [ ] **T3.11 — Generate a contact description.** Exercises the OpenAI call on a non-chat path —
      catches a wrong deployment name or an OpenAI quota of zero on the new account.

- [ ] **T3.12 — Companies list loads; create and edit one.**

- [ ] **T3.13 — Identities list loads; search mentions returns results.**

- [ ] **T3.14 — Client management (admin).** List clients; create or edit one.

- [ ] **T3.15 — User management (admin).** List users; assign a user to a client; change a role.

- [ ] **T3.16 — Metrics page loads (admin).**

---

## Tier 4 — Indexing and search

The new Cognitive Search service starts **empty**. These tests verify the pipeline can fill it, not
that existing data survived.

- [ ] **T4.1 — Confirm the Search endpoint actually in use.** Verify from a running pod which Search
      service is being called.
      *Breaks if:* `IndexingService.cs:295` constructs a hardcoded Search endpoint that bypasses
      configuration entirely. If this is still present, the new instance silently reads and writes
      the **old shared** Search service — every other test in this tier will appear to pass while
      being completely wrong. **Do this one first.**

- [ ] **T4.2 — Create an index for a client.** Index, skillset, and indexer are created on the new
      Search service.

- [ ] **T4.3 — Indexer runs to completion** without errors in the Azure portal's indexer history.

- [ ] **T4.4 — Embeddings are generated.** Indexed documents have populated vector fields.
      *Breaks if:* the embedding deployment name or the OpenAI endpoint on the skillset is wrong.

- [ ] **T4.5 — TextMerge custom skill is reachable from Search.** The `WebApiSkill` step completes.
      *Breaks if:* the TextMerge container has no ingress, or its function-key authorization can't
      be satisfied. Cognitive Search calls this endpoint **inbound from Azure**, so it must be
      publicly reachable and correctly authenticated — this is the least App-Service-like part of
      the whole migration and needs explicit attention.

- [ ] **T4.6 — Newly indexed content is retrievable in chat.** Upload a document with a distinctive
      phrase, wait for indexing, ask about it, and get an answer citing it. **This is the true
      end-to-end test** — it exercises blob → indexer → skillset → embeddings → search → chat in
      one pass.

- [ ] **T4.7 — Deleted content leaves the index.** Delete a document and confirm it stops appearing
      in answers.

---

## Tier 5 — Worker functions

Each of the five workers is a separate deployment now. Test each independently — a shared base image
means one can be healthy while another is misconfigured.

**Before any of these:** confirm `AzureWebJobsStorage` is configured. The Functions host uses blob
leases for timer singleton locking; without it, timers either don't fire or fire on every replica
simultaneously.

- [ ] **T5.1 — All five worker pods start** and log the Functions host initializing with their
      functions registered (Confluence, Slack, Mail, ProcessContent, TextMerge).

- [ ] **T5.2 — Confluence incremental sync.** Trigger it (15-minute timer, or the HTTP trigger) and
      confirm pages are fetched, written to blob, and picked up for indexing.
      *Breaks if:* Confluence credentials missing, or Confluence IP-allowlists the old egress IP.

- [ ] **T5.3 — Confluence deleted-pages sync** runs without error.

- [ ] **T5.4 — Confluence ancestors sync** runs without error (daily timer — trigger manually).

- [ ] **T5.5 — Slack incremental sync** ingests messages.

- [ ] **T5.6 — Slack deep sync** runs (daily timer / HTTP trigger).

- [ ] **T5.7 — Mail sync** ingests mail.
      *Breaks if:* the mail daemon app registration lacks consent in the target tenant, or Exchange
      Online isn't available there.

- [ ] **T5.8 — Identity discovery (ProcessContent)** runs and produces identity records.

- [ ] **T5.9 — HTTP-triggered functions are callable.** Each worker exposes HTTP triggers at
      `AuthorizationLevel.Function`.
      *Breaks if:* function keys. App Service managed the key store; a bare container does not have
      that infrastructure. Verify how keys are provisioned now, or these endpoints are either
      unreachable or unauthenticated — both are failures.

- [ ] **T5.10 — Timers fire exactly once per interval.** Watch two consecutive intervals with the
      deployment at its configured replica count.
      *Breaks if:* replicas > 1 without working singleton leases — you get duplicate syncs, which
      manifests as duplicated content rather than an error.

- [ ] **T5.11 — Restart behaviour is safe.** Delete a worker pod and let it reschedule. Four timer
      triggers use `RunOnStartup = true` (Confluence incremental + deleted-pages, Slack incremental,
      Mail incremental), so a full sync fires on **every** restart, rollout, and node drain.
      Confirm this is idempotent and does not duplicate content or exhaust API rate limits. This was
      harmless on always-on App Service plans and is not harmless under Kubernetes rescheduling.

---

## Tier 6 — MCP

Skip this tier if MCP is explicitly out of scope for the cutover — but say so in the log rather than
leaving it blank.

- [ ] **T6.1 — Discovery document.** `GET /.well-known/oauth-protected-resource` returns the
      expected resource metadata with the new host.

- [ ] **T6.2 — MCP health.** `GET /mcp/health` responds.

- [ ] **T6.3 — OAuth flow completes** against the API's registration.
      *Breaks if:* the Application ID URI must be on a **domain verified in the tenant holding the
      registration**. A new host means a new verification step.

- [ ] **T6.4 — Tools are listed and callable.** `list_clients`, `search_documents`, `fetch_document`
      each return results.

- [ ] **T6.5 — MCP status and revoke endpoints** respond.

---

## Tier 7 — Kubernetes-specific behaviour

None of this existed as a concern on App Service. It is the part most likely to be skipped and most
likely to cause a production incident.

- [ ] **T7.1 — Multi-replica correctness.** With the API at 2 replicas, run a chat conversation
      through to completion several times. Any request landing on the "other" pod must behave
      identically.
      *Breaks if:* in-process SignalR (see T2.2), or any in-memory state assumed to be per-instance.

- [ ] **T7.2 — Pod restart mid-session.** Delete an API pod while a chat session is open. The
      session recovers on the next message; history is intact.

- [ ] **T7.3 — Rolling update has no downtime.** Bump the chart version, let Argo sync, and keep
      the app open. Requests should not fail during the rollout.
      *Breaks if:* liveness/readiness probes are **TCP-only** — with no HTTP health path configured,
      Kubernetes routes traffic to a pod as soon as the port is open, before the app is ready to
      serve. Watch for a burst of 502s at the start of the rollout.

- [ ] **T7.4 — Probes reflect real health.** Force a dependency failure (e.g. bad SQL credentials in
      a scratch deployment) and confirm the pod is marked unhealthy. If a TCP probe keeps a broken
      pod in rotation, that is a finding worth recording even though the app "works".

- [ ] **T7.5 — Argo self-heal.** Manually edit a live Deployment; Argo reverts it, or record that
      auto-sync is off and drift persists.

- [ ] **T7.6 — Logs are retrievable.** Application logs are visible for every workload. If no
      telemetry backend is wired up, `kubectl logs` is the only diagnostic channel — confirm it is
      sufficient and note the gap.

- [ ] **T7.7 — Resource limits are not being hit.** Under a normal chat session, no pod is
      OOMKilled or CPU-throttled at its configured limits.

- [ ] **T7.8 — Secret rotation propagates.** Change a value in the API secret and confirm what it
      takes for pods to pick it up (almost certainly a restart — record the answer, it is
      operationally important).

---

## Explicitly out of scope

State these in the report so nobody reads a blank result as a pass.

- **Voice assistant** — not containerized, not deployed to the cluster.
- **Any integration disabled by configuration** — Grain sync, Slack, and mail have been switched off
  via env values in the past. Confirm each one's actual state and mark it *disabled* rather than
  *failed*.
- **Data migration** — these tests assume the new instance may start with empty data. Verifying that
  historical chat sessions, blobs, or index contents were carried over is a separate exercise.
- **Performance and load** — smoke only. Note anything egregiously slow, but do not benchmark.
- **QA / Prod environments** — only the dev stack exists.

---

## Result log

| ID | Result | Notes / defect link |
|---|---|---|
| P1–P6 | | |
| T0.1–T0.7 | | |
| T1.1–T1.6 | | |
| T2.1–T2.15 | | |
| T3.1–T3.16 | | |
| T4.1–T4.7 | | |
| T5.1–T5.11 | | |
| T6.1–T6.5 | | |
| T7.1–T7.8 | | |

**Sign-off criteria**

| Tier | Requirement |
|---|---|
| 0, 1, 2 | All pass. No exceptions — this is the product. |
| 4 | T4.1 and T4.6 pass. The rest may be deferred with a named owner. |
| 3, 5, 6, 7 | Failures allowed if each is logged with an owner and a decision to ship or block. |
# Security remediation explainer

Date: 2026-09-09  
Final working tree: `/home/parshu/projects/ragz-security-remediation-20260909`  
Remediation base: `3fac9fb1d02c9327f243418ebbb905466c8bcef5`  
Application remediation: `f6fa416bb670de5f02a4b810674becff5f063453`  
Dependency remediation: `18378bf62cbf97ad3a7024bb530230b1c37a8663`  
Cross-check remediation: `071587e`  
Cross-check dependency refresh: `946a0a8`  
Evidence ledger: `ed2403f04278472d255629ae4db2a5fcc991521a`

This document explains the security, correctness and reliability issues found in
the Ragz audit and PR review, the causes of those issues, the danger they posed,
the patch that addresses each one, and the evidence supporting the result. The
CC follow-ups are described using the final remediation working tree and its
targeted tests. The CC source changes are currently working-tree changes on top
of the ledger commit; they should be committed and reviewed before release.

The verification used isolated PostgreSQL, Redis, Qdrant, MinIO and Celery test
resources and fake providers where provider behavior was needed. No paid
provider calls were made. It did not delete existing data, run legacy orphan
cleanup, refresh frozen benchmark scores, or scan existing application services.

## Final status

| Area | Final disposition | Remaining qualification |
|---|---|---|
| SEC-001 through SEC-012 | Fixed with focused red/green evidence | Accelerate has a dated, unreachable-path CI exception until 2026-10-15; image scanning remains separate |
| LOCAL-001 | Fixed for the reproduced stale ACL/revision races | No claim beyond the tested race classes |
| PR-001 through PR-004 | Fixed with focused regressions | The corrected implementation still needs normal final review/commit integration |
| Existing PR comments 1 through 15 | Dispositioned and fixed | Full-stack E2E remains environment-dependent |
| CC-001 through CC-004 | Fixed in the final working tree | Cross-browser rendering remains an operational compatibility surface |
| CC-005 | Current advisories remediated or explicitly controlled | Accelerate is an approved time-bounded exception, not a patched dependency |
| CC-006 through CC-008 | Fixed with callback, adapter and grounding regressions | No real paid-provider outage was used |
| Conditional policy observations | Explicitly recorded, not silently changed | Bot audience, historical-answer revocation and admin DNS pinning require deployment/product decisions |

The final focused backend selection passed **163 tests with 7 warnings in 68.34
seconds**. It covers the final attachment-cleanup and commit-boundary tests,
document uploads, provider accounting, embedding normalization, expansion
publication, retrieval grounding and revision behavior. The frontend suite
passed **725 tests across 106 files**. The document-preview Playwright spec
passed **3 tests**, including a valid PDF rendered through the controlled canvas
viewer. The current frontend audit reported **No known vulnerabilities found**.
The frozen Python audit reported **No known vulnerabilities found, 1 ignored**;
the one ignore is the dated Accelerate exception described under CC-005.

Earlier remediation evidence also recorded 1,853 backend tests passing, 14
skipped, the frontend typecheck/lint/build passing, migration suites passing,
Ruff, mypy for typed source, import-linter and compileall passing, and wheel/
sdist plus frontend image builds passing. Docker Scout could not authenticate,
and the backend image was not built/scanned, so those results do not provide a
whole-image CVE guarantee.

“Fixed” means the final tree has a targeted invariant change and a regression
that exercises the formerly failing behavior. It does not mean every external
deployment choice has been made. “Exception” means the remaining risk is
explicitly bounded, owned and time-limited rather than hidden with a blanket
scanner ignore.

## SEC-001 — Active document content could execute in a viewer origin

### Original behavior and root cause

The upload path trusted a client MIME label and stored it with the document. The
original-file route returned that stored type inline. The frontend accepted broad
`text/*` and `image/*` values and placed the resulting object URL in an iframe
without a sandbox. An ordinary contributor could upload HTML or SVG and wait for
a more privileged workspace member to open its preview.

The root cause was treating a client-provided MIME value as both file identity
and browser rendering policy. A same-origin blob iframe is an execution surface;
keeping the access token in memory and the refresh token in an HttpOnly cookie
does not make same-origin script harmless.

### Danger

Script in a preview could act through the viewer's authenticated application
session, read same-origin application state, or issue requests as that viewer.
The audit established the source-to-sink capability without running an exploit
against a live customer session.

### Patch

`modules/documents/file_types.py` now derives a canonical type from the file
extension and bounded content inspection. PDF headers and end markers, OOXML ZIP
structure and required entries, image signatures, UTF-8 text and full-file text
validity are checked. Active HTML/HTM/SVG and unknown or invalid legacy files
become download-only. The document route chooses inline versus attachment
disposition from that server-derived policy and streams after a bounded
signature read.

The React viewer allows only verified PDF, raster-image and bounded plain-text
preview. Text is inserted as text, active/unknown content has a download control,
and ACL-sensitive object URLs and query bytes are evicted on unmount or change.
The final PDF implementation uses `pdfjs-dist` in a controlled canvas renderer;
it does not put the original bytes in an executable iframe.

### Evidence and compatibility

The original MIME-spoof/API and four viewer regressions were red. Backend file
tests, frontend viewer/file tests and real Chromium tests now cover spoofed MIME,
stored active HTML, escaped text, raster images, verified PDF, bounded streaming,
Unicode filenames and a valid PDF rendered to a non-empty canvas. The valid PDF
Playwright test also confirms there is no iframe and no preview script execution.

Legacy active/unknown files remain downloadable rather than executable. Text
preview is capped at 2 MiB. PDF, PPTX/OCR, image and large-file support remain
available through the safe paths.

## SEC-002 — Attachment ingestion had no aggregate resource admission

### Original behavior and root cause

The attachment endpoint applied permission checks but no attachment-specific
rate limit, count/byte budget, pending-job budget or parser-work reservation. It
buffered a per-file upload, stored it and enqueued extraction. The 50 MiB
per-file ceiling limited one request but did not limit repeated uploads,
concurrent pending extraction or aggregate worker/object-store work. Permanent
document quotas did not include ephemeral attachments.

The root cause was checking only request authorization and individual size, with
no atomic admission state shared by concurrent requests.

### Danger

A permitted user could consume shared API memory, object storage, database rows
and parser workers through repeated unused attachments. This is a tenant
availability and cost/resource-abuse risk, not a claim of parser RCE.

### Patch

Attachment transfer uses spooled/streamed content, a path-specific ASGI body
ceiling before multipart accumulation, per-user upload rate control and durable
PostgreSQL reservations. Admission accounts for attachment count, bytes, pending
work and configured organization/user limits before object storage or worker
enqueue. Signature/type checks, archive decompression limits, PDF/page bounds,
text/character limits and image-pixel limits reject work before storage. Workers
repeat the relevant resource checks.

Permanent document uploads use the same reservation pattern while retaining the
streamed 1 GiB document boundary and bounded parser batches. Rejected work is
not allowed to create an object or enqueue a worker task first.

### Evidence and compatibility

Red regressions showed missing aggregate and pending admission. Passing tests
cover spoofed types, compressed archive budgets, image pixels, aggregate limits,
parallel pending limits, rate bounds and rejection before storage/enqueue. The
final cleanup/quota regression also verifies that pending external cleanup is
included; see CC-001.

Finite defaults now cover organization document/storage and attachment
count/bytes/pending/rate/parser/pixel caps. Operators must size these for
legitimate workloads. The 1 GiB streamed document limit was preserved
deliberately rather than lowering large-PDF support.

## SEC-003 — Organization admins could self-grant sensitive permissions

### Original behavior and root cause

The tenant context excluded sensitive permissions such as `audit.read`,
`audit.export` and `documents.acl.bypass` from ordinary admin defaults. The role
assignment service nevertheless allowed an administrator to list an active
global role template and assign it to themself. The route required an admin
label, but the service did not require an independent grantor or a separate
authority to grant sensitive capabilities.

The root cause was confusing “is an administrator” with “may grant every
permission,” and enforcing the separation only in default role expansion.

### Danger

An organization administrator could defeat the intended separation between IAM
administration and audit/content access. This could expose audit records or
restricted document content within that organization. It did not grant
superadmin or another organization's access.

### Patch and evidence

Sensitive templates now require the declarative `roles.sensitive.assign`
authority, an authorized independent grantor or superadmin, and same-organization
scope. Self-assignment and cross-organization assignment are rejected; an
independently delegated grant remains possible. Isolated PostgreSQL service/API
tests deny self and cross-org grants and permit authorized delegated grants.

Existing administrators who need sensitive access must use the independent
grant process. UI visibility is not the authorization boundary.

## SEC-004 — Egress guards accepted shared address space

### Original behavior and root cause

Media and web-content fetch guards rejected private, loopback, link-local,
reserved, multicast and unspecified destinations but did not reject
`100.64.0.0/10`. Python classifies shared CGNAT space as neither private nor
global. A public web result or redirect could therefore reach an overlay/shared
service if the API host could route to it.

The root cause was duplicate, inconsistent address classification: one central
network helper rejected non-global destinations while media/web used a narrower
private-address test.

### Danger

The path could expose internal overlay services to user/model-influenced fetches,
depending on deployment routing. The audit did not bypass tested RFC1918,
metadata or loopback defenses and used mocked DNS/HTTP rather than a real private
service.

### Patch and evidence

Media and web paths now share the central non-global destination policy, handle
IPv4-mapped IPv6 addresses, validate every redirect and retain DNS revalidation/
pinning for user-controlled fetches. Operator-controlled OIDC, SMTP and model
targets can use explicit `RAGZ_EGRESS_ALLOWED_CIDRS` entries for required private
services, while local, link-local, metadata, unspecified and multicast targets
remain prohibited. Persisted model targets are revalidated before proxy replay.

Fourteen direct/redirect IPv4/IPv6 tests pass, including the CGNAT reproducer.
Admin connectors still check and later re-resolve rather than pinning the
validated socket; that is a conditional deployment risk recorded below.

## SEC-005 — Passive auth failure retained identity-bound query state

### Original behavior and root cause

When refresh failed, the client token was cleared and the app navigated to login,
but the global TanStack Query client survived. Login stored a new token without
cancelling/removing all prior identity-bound queries. Workspace/admin/chat keys
were not comprehensively namespaced by principal, and an in-flight A request
could complete after B logged in.

The root cause was treating token storage and navigation as the entire identity
transition while cache and request lifecycles remained global.

### Danger

In a shared browser/tab, user B could see cached or late-arriving A data under
matching query keys. This was a same-browser confidentiality issue, not a
backend authorization bypass.

### Patch and evidence

`clearAuthenticatedQueryState` cancels active queries and then removes every
query object, so a query function that ignores cancellation cannot repopulate the
new principal's cache. It runs during passive auth failure, refresh-based
identity restoration, login/register and logout. The final auth-generation
extension is described under CC-003 and fences the refresh side effect itself.

The A-expiry-to-B-login regression and 36 focused auth/client/document tests
pass. Reopening an ACL-sensitive document performs a fresh server check; this
extra request is intentional.

## SEC-006 — Chat deletion orphaned attachment objects and vectors

### Original behavior and root cause

Deleting a chat cascaded `ChatAttachment` rows immediately. The cleanup worker
discovered external object/vector identifiers only by reading those rows. Once
the cascade committed, the worker had no target key or attachment ID.

The root cause was allowing relational deletion to destroy the only durable
record of external work before that work completed.

### Danger

Files and ephemeral vectors could survive expected deletion indefinitely,
consuming storage and leaving retained data outside the intended lifecycle.
Existing access checks meant the audit did not establish public read access.

### Patch and evidence

`attachment_cleanup_jobs` has no foreign keys and stores organization, user,
chat, attachment, object identifier and byte size before cascade. Cleanup
independently retries idempotent object/vector deletion and records attempts,
errors and completion. A read-only inventory script compares DB, cleanup-job,
MinIO and Qdrant identifiers; it never deletes historical data. The migration
conservatively backfills old attachment sizes to the former 50 MiB ceiling.

Isolated PostgreSQL lifecycle tests prove identifiers survive the cascade;
failure/retry/idempotency and dry-run inventory tests pass. CC-001's final quota
test passes because the job now retains user/bytes ownership until completion.

## SEC-007 — Permanent upload quotas were check-then-use

### Original behavior and root cause

Document upload counted committed rows and bytes, then stored an object and
committed later. Concurrent requests could all observe the same remaining
capacity. In-flight uploads were invisible to the next decision.

The root cause was a non-atomic read followed by external work, with no durable
reservation ledger or organization-level serialization.

### Danger

Concurrent callers could exceed organization document-count or storage-byte
limits, causing availability, cost and fairness failures.

### Patch and evidence

`resource_reservations` records count and bytes for document and attachment
work. `pg_advisory_xact_lock` serializes admission per organization; expired
rows are pruned, current committed rows plus live reservations are checked, and
the reservation is removed on successful publication or released on failed/
cancelled work. Object writes occur only after admission.

Red parallel one-document/byte-boundary tests previously returned two successful
uploads. Passing tests now show serialized admission, rejection before storage/
enqueue, failed-storage release, cancellation release and cleanup-job charging.
Reservations expire after a bounded interval to recover from API crashes. A
stale reservation may temporarily reduce availability, but cannot silently
permit over-quota work.

## SEC-008 — API-key lifecycle and audit writes were separate transactions

### Original behavior and root cause

The API-key service committed creation or revocation before the route created and
committed its audit event. A crash, cancellation or audit transaction error could
leave credential state changed without a corresponding audit row.

The root cause was splitting one security-sensitive lifecycle transition across
two independent transaction owners.

### Danger

Credential accountability and incident reconstruction could be weakened. A key
could be created but never delivered to its caller, or revocation could succeed
without a corresponding audit event. This was a privileged-route integrity gap,
not unauthenticated key issuance.

### Patch and evidence

Lifecycle mutation and the audit event are staged in one session and committed
once. Raw keys remain write-only: returned once, then represented by a lookup
prefix and peppered full hash. Injected audit failures roll back create/revoke,
and one-time raw-key/masked-list tests pass.

Failing the whole operation when audit durability is unavailable is deliberate
fail-closed behavior.

## SEC-009 — Authentication denials were stored as successful audit events

### Original behavior and root cause

Password and OIDC denial paths recorded `login.failure` without explicitly
setting `result=denied`; the audit model defaulted to `success`. They also lacked
bounded reason codes, authentication method and trusted client IP.

The root cause was relying on the action name to carry failure semantics while
downstream reporting filters on the result field.

### Danger, patch and evidence

Detection and incident metrics could undercount failed authentication or report
it as successful. All denial paths now explicitly write `result=denied`, a
bounded reason code, auth method, trusted request IP and safe internal/unknown
target identity. Raw credentials and supplied sensitive values are excluded.
Isolated PostgreSQL password/OIDC/domain/inactive-account tests inspect persisted
fields and pass.

## SEC-010 — CI audited the scanner environment instead of the application

### Original behavior and root cause

CI ran `uvx pip-audit --strict --local` after `uv sync`. `uvx` created an
isolated tool environment; with no explicit project export, the scanner saw about
29 pip-audit/tool packages and omitted FastAPI, Docling and Transformers.

The root cause was relying on the scanner's current environment instead of an
explicit frozen inventory and not asserting representative application packages.

### Danger, patch and evidence

The dependency gate could be green while a production runtime vulnerability was
present. The workflow now exports frozen runtime and full/build requirements,
audits each with hashes and strict advisory behavior, and asserts representative
application packages are present. The old 29-package invocation and explicit
roughly 212–218-package inventories were compared. Runtime and build trees are
separate gates. The workflow itself still needs to run successfully on the
final published head; a historical `action_required` status was not treated as
CI success.

## SEC-011 — Vulnerable dependency and build-tree audit gaps

### Original behavior and root cause

The frozen Python runtime contained Transformers 5.8.1, matched to the then-current
tokenizer/processor path-traversal advisory. The frontend gate audited production
dependencies only, while full lock data included vulnerable development/build
transitives. The causes were an unupdated lock and a production-only assurance
claim.

### Danger and patch

The Transformers advisory could permit unintended path writes in a reachable
`save_pretrained` path with attacker-controlled template input, although no Ragz
tenant-controlled call to that path was found. Build-tree advisories can affect
CI/developer environments and build inputs even when absent from browser runtime.

The lock moved Transformers to 5.15.1 and pinned Hatchling 1.32.0. Frontend
transitives were updated with compatible overrides and both runtime/full trees
became explicit CI targets. No blanket ignore was used for the original findings.

### Evidence and final limit

The later cross-check found Accelerate 1.14.0, CVE-2026-69112/GHSA-4j2p-28q2-5m79.
The advisory concerns unsanitized paths in sharded checkpoint indexes used by
checkpoint loading/dispatch. Current source review found Ragz/installed Docling
paths do not invoke those explicit loaders and do not accept tenant-controlled
checkpoints. No patched Accelerate release was available. The CI workflow has an
exact, dated exception reviewed through **2026-10-15**, with the advisory ID,
unreachable-path rationale and removal date in the workflow comment. The Python
audit result is “no known vulnerabilities found, 1 ignored,” so this is an
explicit time-bounded exception rather than a claim that Accelerate is patched.

The frontend full-tree advisory for js-yaml 4.3.1 is fixed by the 4.3.2
workspace override and lock. Vitest is updated to 4.1.11. Current `pnpm audit
--audit-level=high` reports no known vulnerabilities. Docker Scout remains
unavailable and the backend image was not scanned.

## SEC-012 — Completed provider work could escape usage accounting

### Original behavior and root cause

Agent/tool execution streamed intermediate work and persisted usage only after a
final aggregate. Cancellation before that aggregate, or while its commit was
pending, could lose completed billable web, planner, embedding or generation
work. Comparison retrieval similarly staged usage until later processing.

The root cause was coupling durable billing to answer completion rather than to
the actual completed-provider-call boundary.

### Danger, patch and evidence

Usage ledgers, quotas, cost reports and incident attribution could undercount
incurred work; repeated cancellation could widen the discrepancy. Usage rows now
use run-scoped idempotency keys backed by a partial unique index. Completed
planner, web, retrieval and generation actions write through independent sessions
with cancellation-safe boundaries before later fallible work. Q1 remains usable
when optional alternatives fail. The final callbacks-before-bookkeeping changes
are detailed under CC-006.

Red tests showed completed work disappearing on Q1/alternative failure and
generation cancellation. Real-PostgreSQL/fake-provider regressions now cover
planner, web, retrieval, generation, duplicate idempotency and Q1 fallback.

## LOCAL-001 — Stale ACL projection could remain searchable

### Original behavior and root cause

In the old shared checkout, a relational ACL change committed before Qdrant
projection. If Qdrant rejected the update, the old broader ACL payload remained
searchable and the old retrieval path lacked pending/failed exclusion. Retrieval
could therefore use an ACL that was no longer authoritative.

### Patch, danger and evidence

Document rows now carry `security_revision`, `projected_security_revision` and
`index_state`. ACL changes bump the revision and move the row pending in the same
transaction. Projection writes the revision into Qdrant payloads and uses
compare-and-set activation only when the target revision remains current.
Retrieval filters pending/failed rows, rereads current authorized revisions after
Qdrant results and drops mismatches. Legacy indexed rows move pending until
reconciled.

The direct-search stale-response test was red on the old path and passes now.
Tests cover completed projections, older pending races, reconciler reopening,
Qdrant outage and chat source recheck. No claim is made for unmodeled races; the
final exact probe identity check is described under CC-008.

The upgrade creates a deliberate fail-closed availability interval while legacy
vectors are restamped.

## PR-001 — Cache owner cancellation stranded single-flight futures

### Original behavior, danger and cause

Embedding and query-expansion caches cleaned up an owner when cancellation
arrived during provider computation but not while reacquiring the cache lock for
publication. The `_inflight[key]` future could remain unresolved; later same-key
requests joined it forever. This required cache use, contention and cancellation
at the specific await boundary.

### Patch and evidence

Both caches now cover computation, lock reacquisition, publication and owner
cleanup with a cancellation-safe lifecycle. Futures are resolved or removed and
waiters can retry ownership. Five deterministic tests cover compute/publication
cancellation, multiple waiters and unrelated keys. The retained reproducer's
dead slots changed from true for both caches to false.

Organization namespacing also prevents cross-tenant cache timing/usage coupling.
Cache values are query-derived embeddings/alternatives, not document answers.

## PR-002 — Alternative embedding failure did not fall back to Q1

### Original behavior, danger and cause

Q1 and expansion could succeed, then an unguarded alternative embedding call
could fail. The fallback covered expansion failure but not every alternative
provider failure, and staged usage could be rolled back with the request.

### Patch and evidence

Ordinary alternative-provider failure discards alternatives and continues with
Q1. Completed expansion/Q1 work is durable and idempotent; cancellation remains
cancellation instead of being swallowed as an ordinary fallback. A second-call
failure regression verifies a usable Q1 result and one durable row per completed
call. Adapter normalization for all expected error shapes is now covered by
CC-007.

## PR-003 — Comparison source lookup preceded usage commit

### Original behavior and danger

Comparison retrieval staged provider usage and then assembled authorized source
metadata. Source lookup failure, cancellation or revocation denial could roll
back a completed retrieval charge, losing cost and incident evidence.

### Patch and evidence

Comparison commits retrieval usage before `_sources()` through the idempotent
durable recorder. A real isolated database plus fake retriever test fails source
lookup and still observes the durable row; generation cancellation is covered as
well.

## PR-004 — Removed ACL candidates influenced no-answer grounding

### Original behavior, danger and cause

Dense no-answer probes ran with an earlier filter and stored scores independently
of fused candidates. A later revision recheck could remove the high-scoring
document from generation while its score still raised `best_cosine`; a weak
allowed candidate could then trigger generation instead of no-answer.

This was grounding/integrity risk, not demonstrated forbidden-content
disclosure: the probe omitted payload/text needed for the answer. The cause was
deriving an authorization-sensitive decision from a score set not rebuilt after
the authoritative recheck.

### Patch and evidence

Retrieval now carries exact document/page/chunk/security-revision payloads for
score-producing probes and computes the verdict only for candidates authorized
for generation. The high-score removed/low-score allowed regression passes, and
the final probe-only regression is recorded under CC-008.

## Existing PR review comments 1 through 15

These are the 15 comments recorded in the PR review. They are review-quality,
correctness and hardening items, not 15 additional confirmed vulnerabilities.

1. **MQR isolation test could pass without expansion.** The old test asserted an
   empty result without proving the expander ran or that a same-tenant positive
   existed. It now asserts invocation, same-tenant positive retrieval and rival
   exclusion.
2. **Invalid/disabled default model.** Assignment checked existence but not
   enabled chat modality, while completion rejected it. Assignment now resolves
   enabled chat models; explicit invalid values error, legacy invalid defaults
   fall back to the first enabled chat model, and no enabled model is a typed
   conflict.
3. **Streaming cancellation loses paid web usage.** This was pre-existing
   SEC-012. Per-completed-call durable rows are written before final aggregation
   and stream completion; the final callback boundary is tested under CC-006.
4. **Alternative embedding fallback.** Covered by PR-002: Q1 stays usable and
   completed work is retained.
5. **Removed candidate affects no-answer.** Covered by PR-004 and the exact
   probe identity/revision work in CC-008.
6. **Async workspace lookup in E2E.** The specs now wait for the switcher's
   loading state before deciding whether to create or reuse a workspace. Full
   stack E2E remains environment-dependent.
7. **ADR said one dense batch.** ADR-0007 now describes Q1 first and a second
   bounded alternatives batch.
8. **ADR said query count was always three.** ADR-0007 distinguishes production
   default/maximum from explicit internal/evaluation one, three or five-query
   overrides.
9. **Stale client role gating.** The reviewed settings control now uses
   server-derived authorization. A demoted-token regression passes; backend
   checks remain authoritative, so this was UX consistency rather than bypass.
10. **Cancelled cache publication stranded a future.** Covered by PR-001; both
    caches resolve/remove owner slots and allow waiter retry.
11. **HTTP-date `Retry-After`.** Reranking now parses numeric and HTTP-date forms,
    handles past/future dates and clamps to 30 seconds; mocked tests pass.
12. **Comparison source lookup before usage commit.** Covered by PR-003.
13. **Full 32k expansion input.** The old behavior was a cost/latency issue,
    not an auth exploit. Expansion prompt input is capped at 4,000 characters;
    ordinary chat's 32k input contract remains.
14. **Evals used unavailable workspace default.** UI now chooses an available
    enabled chat model or presents an empty disabled state; frontend tests pass.
15. **Mutable “Fixed input” labels.** Results snapshot the submitted question
    and model, so editing the form cannot relabel a completed result.

## Final CC follow-ups

### CC-001 — Pending cleanup is included in quota accounting

**Status: fixed, P2/Medium.**

The cross-check found that live attachment rows and reservations were counted,
but incomplete cleanup jobs were not. Deleting a chat could therefore make
quota capacity appear free while the external object still existed.

The final patch carries `user_id` and `size_bytes` on the no-FK
`AttachmentCleanupJob`, captures both before the relational cascade, and counts
all incomplete cleanup jobs in `_attachment_usage` for organization and user
limits. Cleanup completion is the release point; retries do not double-count
because the job is unique by attachment ID and completed jobs are excluded.

The final isolation test fills a ten-byte quota, deletes the chat, confirms a
new reservation is still rejected while the job is incomplete, processes the
job successfully, then confirms reservation succeeds. The migration test checks
the new cleanup columns. This closes the upload/delete quota-cycle gap while
preserving durable external cleanup.

### CC-002 — Commit-outcome reread prevents destructive compensation

**Status: fixed, P2/Medium.**

The cross-check injected cancellation after PostgreSQL had committed an upload.
The old handler treated that ambiguous exception as a definite rollback and
deleted the blob, leaving a committed row pointing at missing bytes.

`committed_row_exists_after_error` now performs a cancellation-resistant,
shielded rollback-and-reread. Attachment and permanent-document upload handlers
delete the object only when the durable row is proven absent. If the outcome is
unknown, they preserve the object and log an outcome-unknown event for
reconciliation; they never make destructive compensation on uncertainty.
Object-transfer failures before finalization retain the existing cleanup path.

Real isolated PostgreSQL tests inject a commit that succeeds and then raises
`CancelledError`; fake storage confirms that both the attachment and document
rows retain their object keys. The test specifically proves committed bytes are
not deleted. The finalization helper is shared by both upload flows.

### CC-003 — Auth generation fences late refreshes and transitions

**Status: fixed, P2/Medium.**

The old cleanup fixed query data but a global refresh promise could still write an
A token after B logged in. A waiting A request could retry with that stale token.

The final client tracks an auth generation in `auth-store`, tags each refresh
promise with its generation, aborts and settles old refreshes during explicit
login/logout/register transitions, and uses `replaceAccessTokenIfCurrent` before
accepting a response. `authFetch` records the request generation and retries only
if the refresh result belongs to that generation. Stale refresh success/failure
cannot replace or clear a newer identity and cannot fire the current auth-failure
handler.

The final client tests cover late refresh success/failure, a stale 401 that must
not start a network refresh, generation-scoped XHR upload retry, and an explicit
transition that aborts and settles old work. Password and OIDC login revoke a
presented refresh family only when it belongs to a different user, closing the
cross-identity cookie race while preserving supported same-user session families.

### CC-004 — Valid PDFs render through a controlled canvas viewer

**Status: fixed, P2/Medium functional regression.**

The empty sandbox protected the origin but prevented Chrome's native PDF viewer
from rendering a valid PDF. The previous test used an invalid two-line fixture
and checked only the sandbox attribute.

The final viewer adds Node-20-compatible `pdfjs-dist` 5.4.624 with its worker
configured through the bundler. It is dynamically loaded only when a PDF is
mounted, so ordinary jsdom/application imports do not require browser `DOMMatrix`.
`PdfPreview` loads the ACL-gated blob, clamps the cited page to the
document's page range, renders to a canvas, cancels in-flight work on unmount
and reports a safe download-preserving error if rendering fails. The original
PDF is never placed in an executable iframe.

The viewer unit test verifies page selection and absence of an iframe. The
Playwright test opens the repository's valid sample PDF in Chromium, checks a
visible non-empty canvas and page 1 metadata, and confirms no iframe or preview
script execution. All three document-preview security tests pass.

### CC-005 — Dependency findings are patched or explicitly time-bounded

**Status: current findings resolved with one dated exception, P2/Medium.**

The js-yaml full/build finding is fixed by forcing `js-yaml` to **4.3.2** in the
workspace override and lock. The frontend test tool is updated to **Vitest
4.1.11**. The current `pnpm audit --audit-level=high` reports no known
vulnerabilities.

Accelerate remains **1.14.0** because no patched release was available at review
time. The workflow's runtime and full Python audit comments identify CVE-2026-69112,
state that Ragz/installed Docling paths do not call the affected explicit
checkpoint loaders or accept tenant checkpoints, and enforce a review/removal
deadline of **2026-10-15** before both runtime and full audits. CI ignores only that exact advisory and continues strict
hash-verified auditing for everything else. The current frozen Python audit
therefore reports no known vulnerabilities with one ignored advisory.

This is an explicit unreachable-path exception, not evidence that Accelerate is
patched and not permission to add blanket ignores. Before 2026-10-15, recheck
the dependency graph and remove the exception when a supported fixed release or
validated mitigation exists. Whole-container scanning remains unavailable until
Docker Scout authentication and backend image construction are provided.

### CC-006 — Accounting callbacks run before cache, counter and validation gaps

**Status: fixed, P2/Medium.**

The cross-check found three places where a provider had completed but the durable
recorder was called later: expansion cache publication, web-search counter
updates and local vector-shape validation.

The final implementation attaches an expansion usage callback to the provider
expander so billing occurs before cache publication can be cancelled. The web
search tool invokes its billable usage callback immediately after a successful
provider response and before Redis daily-counter writes. `_embed_query_batch`
records billed tokens immediately after the provider returns and before checking
vector count or width. Run-scoped idempotency keys prevent duplicate rows when a
compatibility wrapper also observes the completed call.

The expansion publication cancellation test verifies usage is recorded before
the owner is cancelled and a waiter receives the cached result. The web-counter
failure test confirms a completed billable search remains in `UsageRecord` when
Redis counter update raises. The malformed billed-embedding test confirms usage
is recorded before local width validation raises. These tests pass as part of the
final 164-test backend selection.

### CC-007 — Provider adapter failures are normalized

**Status: fixed, P2/Medium.**

The Q1 fallback catches typed `UpstreamError`, but TEI HTTP/JSON failures and
malformed LiteLLM entries could previously leak raw HTTP, `KeyError`, `TypeError`
or `AttributeError` exceptions. An optional alternative provider failure could
therefore abort retrieval after Q1 succeeded.

The TEI adapter now maps transport errors, non-success HTTP responses, invalid
JSON and non-list/non-vector response shapes to sanitized `UpstreamError`
instances. The LiteLLM adapter validates the response object, `data` list,
integer indexes and embedding lists before sorting/using entries, and maps
malformed shapes to `UpstreamError`. Cancellation and unrelated programming
errors are not swallowed.

Parametrized tests cover TEI HTTP 503 and invalid JSON plus malformed LiteLLM
data, missing embedding/index fields and wrong response types. The existing
alternative-failure retrieval regression then verifies Q1 fallback and exact
usage behavior.

### CC-008 — Grounding uses exact probe candidate identity and revision

**Status: fixed, P2/Medium grounding integrity.**

The old revision check derived document IDs from fused candidates. Dense
no-answer probes were separate, omitted payload/revision data and could let a
stale high score influence `best_cosine` even when that document was outside the
generation set.

The final probe requests include `document_id`, `page`, `chunk_index` and
`security_revision`. `_best_eligible_dense_score` builds exact candidate keys
from the authorized generation chunks and accepts a score only when all four
identity/revision fields match. A probe-only document cannot supply the
grounding score. This keeps the no-answer verdict tied to the same authorized,
current candidate snapshot used for generation.

The dedicated retrieval test supplies a high-score probe-only document and a
low-score allowed fused candidate, then asserts that the allowed chunk is the
only returned chunk and `no_answer` remains true. The focused retrieval suite
passes. This closes the previously untested probe-only variant; it does not
claim that malformed provider payloads can bypass the explicit identity parser,
which skips malformed identities safely.

## Conditional policy and deployment observations

These items were intentionally recorded rather than silently changed because
their correct result depends on a product or deployment decision.

### Bot audience

Platform signatures prove delivery origin, not that the end user may read the
configured integration user's corpus. The relay intentionally executes messages
as that least-privilege configured user. If a bot is reachable outside its
private organization/channel/user audience, those users could inherit that
identity's access. Deployment must add sender binding or an allowlist before
exposing private corpora. A deliberately public bot corpus has a different
policy.

### Historical answers after revocation

Chat history retains previously generated document text and citations. Current
file access checks live ACLs, while the current snapshot policy lets the chat
owner retain historical answers after later membership/ACL changes. Whether
revocation is prospective or retrospective is a product decision; retrospective
revocation needs migration and UX work and was not changed in this remediation.

### Admin egress DNS rebinding

User/model-controlled media and web fetches use non-global policy and DNS
pinning. Operator OIDC, SMTP and model integrations preserve explicit private
CIDRs, but admin clients still check then re-resolve rather than pinning the
validated socket. Deployment egress controls or socket-pinned connectors are
required for a complete guarantee.

### Global query cache, comparison cost and CAG

Embedding and expansion cache keys now include organization namespace; values do
not contain document chunks or answers. Comparison's forced multi-query arm is
an explicit evaluation path requiring `evals.run`, without mutating workspace
settings, and uses normal quota/accounting. The separate CAG response cache
remains development/testing-only and production replay is disabled until its
authoritative revision, principal, source authorization and fill-fencing gates
are complete.

### Legacy orphans, credential prefixes, citations and logs

The orphan tool is read-only and no historical cleanup was run. New API keys use
24-character prefixes and legacy lookup remains constant-time across candidates.
Citation rendering accepts only HTTP(S) and makes malformed/other schemes inert.
Provider error bodies and model text are redacted from the concrete logging paths
reviewed; optional Sentry delivery still needs deployment verification.

## Compatibility, migration and operational impact

Migration `a31f6d8c9e02` adds `resource_reservations`,
`attachment_cleanup_jobs.user_id`, `attachment_cleanup_jobs.size_bytes`,
`chat_attachments.size_bytes` and the partial unique usage-idempotency index.
Existing vector-backed documents move pending until current security revisions
are stamped, creating a deliberate fail-closed availability interval.
Historical attachment sizes are conservatively backfilled to the old 50 MiB
ceiling. No migration inspects or deletes objects/vectors.

Large permanent documents remain streamed to storage and parsed in bounded page
ranges. PPTX, OCR and parser alternatives remain available. Active/unknown
legacy files become download-only. Query-derived caches are distinct from the
development-only CAG response cache.

No paid calls are authorized. The preserved gateway ledger is `$2.60807538`,
plus up to seven unresolved external rerank search units. Frozen datasets,
failed runs and benchmark artifacts remain unchanged. Cache latency is not
authenticated cached-chat latency, and passing isolated test checks does not
authorize production CAG replay.

## Verification record

The final targeted commands and outcomes were:

- definitive full backend suite: **1,875 passed, 14 skipped, 10 warnings in
  1,122.21 seconds**, importing from the persistent remediation checkout;
- backend post-rereview selection using isolated resources and explicit source
  path: **181 passed, 7 warnings, 90.68 seconds**;
- frontend `pnpm test` from the remediation tree: **725 passed across 106
  files**, Vitest 4.1.11;
- frontend `pnpm exec playwright test e2e/document-preview-security.spec.ts`:
  **3 passed in 3.8 seconds**, including valid PDF canvas pixels;
- frontend `pnpm audit --audit-level=high`: **No known vulnerabilities found**;
- frozen Python export plus `pip-audit --strict --require-hashes --disable-pip
  --ignore-vuln CVE-2026-69112`: **No known vulnerabilities found, 1 ignored**;
- earlier backend/full frontend, type, lint, import, migration, build and
  package evidence is retained in `docs/audits/security-remediation-status.md`.

The backend virtualenv was created from the frozen lock in the persistent
remediation checkout. No production service or campaign database was used.
The final independent rereview returned **ship** with no Critical or Important
finding remaining in CC-001 through CC-008 or the session-compatibility follow-up.
PR #11 was updated and all 15 Cubic threads received individual replies and were
resolved. The upstream GitHub Actions runs remain `action_required`: approving a
fork run requires `marketcalls/ragz` administrator rights, and both API approval
attempts returned HTTP 403. This report therefore makes no remote-CI pass claim.

## Release statement

The final working tree addresses SEC-001 through SEC-012, LOCAL-001, PR-001
through PR-004, all 15 existing review comments and CC-001 through CC-008 with
the patches and evidence above. CC-005 still carries a clearly documented,
time-bounded Accelerate exception until 2026-10-15; that is the remaining
dependency risk and must be revisited by its deadline. Bot audience,
historical-answer revocation, admin DNS socket pinning and final image scanning
remain explicit product/deployment work. Commit and independently review the
current CC source changes before publishing an all-clear or deploying.

# Ragz end-to-end security audit

Audit date: 2026-09-06. Assessment: **changes required before a public, untrusted multi-tenant deployment; do not merge PR #11 unchanged.**

## Executive summary

The highest-confidence release risk is stored active content in the document viewer: an uploader can supply HTML that the application opens in an unsandboxed, same-origin blob iframe. Its script would run with the viewer's browser origin when previewed. Separately, attachment ingestion has no aggregate admission control, org administrators can assign themselves permissions intended to be independent grants, and the web-fetch address policy misses shared/private-overlay IPv4 space.

PR #11 preserves the ordinary tenant/workspace/document filters in each multi-query retrieval lane, but adds a reproducible cancellation defect that can strand cache entries and hang subsequent requests. Its fallback, accounting, and authorization-sensitive grounding checks also need corrections. The PR's existing automated review contains valid issues; a successful review-bot check is not equivalent to passing CI.

The existing security controls are substantial: Argon2id, short-lived signed access tokens, live user/security-version checks, rotating refresh families, OIDC state/nonce/PKCE, expiring scoped API keys, encrypted provider secrets, declarative permissions, query-time document ACLs, and isolation tests. These controls do not close the findings below.

## 1. Scope, versions, and assumptions

| Surface | Exact reviewed state | Interpretation |
|---|---|---|
| Shared local checkout | `90bb3b147b5da88ca704663cce09f77eab7629b7` plus uncommitted changes observed during this audit | Old `main`; concurrent CAG, upload/parser, and UI work. Not equivalent to GitHub main. |
| GitHub main | `9d08839f6855967f3b141b731a269a54b76222fb` | 63 commits beyond the local committed baseline. Contains previously merged hardening. |
| [PR #11](https://github.com/marketcalls/ragz/pull/11) | `3fac9fb1d02c9327f243418ebbb905466c8bcef5`, against the main revision above | 47 changed files; 5,074 additions and 230 deletions. |
| CAG work in progress | `backend/src/ragz/modules/cache/response.py`, app/config integration, related tests and deployment changes | Reviewed as unfinished infrastructure, not an enabled answer-cache feature. |

Unless explicitly marked **local only**, source line references below refer to the isolated PR checkout at `3fac9fb`; unchanged findings were compared with upstream main. Commit-pinned GitHub links are used for important anchors. The shared worktree was not reset, rebased, or edited by this audit, apart from these report/evidence files.

Working threat assumptions: self-hosted production deployment, multiple organizations sharing the API/workers/stores, private documents, ordinary contributors who can upload, and potentially internet-reachable application endpoints behind an operator-managed TLS proxy. The PRD explicitly describes private business documents and multi-tenancy (`docs/prd.md:14`). Deployment/data-sensitivity clarification was requested but had not been received when this report was prepared. Conditional findings identify assumptions that change their severity. No production URL or verified production configuration was supplied.

Included: backend APIs and modules; auth/RBAC/tenancy; document and attachment ingestion, retrieval and lifecycle; chat/LLM/tools; web/media egress; bots and external API; secrets/audit/quotas; browser rendering/session state; migrations, CI, deployment configuration and dependency lockfiles; PR changes and earlier findings relevant to these boundaries.

Excluded from a claim of runtime validation: production infrastructure, external identity-provider and messaging-platform account configuration, live customer data, paid provider execution, full parser fuzzing, container-image/OS vulnerability scanning, disaster-recovery execution, and unrelated repositories or benchmark vendor checkouts. This is a source-led application security assessment with targeted isolated execution, not an ASVS certification or proof that every vulnerability has been found.

### Context recovered from other work

The CAG session's latest relevant handoff describes a separate ephemeral Redis response cache and large-PDF work. It explicitly leaves production chat replay disabled until authoritative PostgreSQL revisions and hit-time source/ACL validation are implemented. The code agrees: app construction creates a cache adapter, but no chat/retrieval call site consumes `ResponseCache.lookup`.

A final worktree check also found a concurrent change in `modules/chat/llm.py` selecting explicit reasoning mode for certain tool calls. Its small diff was inspected: it changes provider request options, not tool permissions, destinations or tenant filters. No new security finding was established from it; provider compatibility was not revalidated in this security audit.

Earlier reports in `docs/audits/2026-08-16-*` were treated as leads and rechecked, not imported as current facts. In particular, pending/failed document-security projection exclusion, outbox/job recovery, composite tenant constraints, production packaging, and PR test workflows exist upstream even though the shared local baseline lacks parts of them.

## 2. Findings at a glance

| ID | Severity | Finding | State / confidence |
|---|---|---|---|
| SEC-001 | High | Uploaded HTML can execute through unsandboxed document blob preview | Local + main + PR; confirmed source-to-sink path |
| SEC-002 | High | Attachment upload/processing lacks aggregate resource admission | Local + main + PR; confirmed control gap, no exhaustion attack run |
| SEC-003 | Medium | Org admin can self-grant restricted audit/content permissions | Local + main + PR; isolated PostgreSQL regression confirmed |
| SEC-004 | Medium | Web/media SSRF guard accepts `100.64.0.0/10` | Local + main + PR; network-free redirect probe confirmed; impact depends on network reachability |
| SEC-005 | Medium | Passive auth failure retains previous user's browser query cache | Local + main + PR; confirmed state-management path |
| SEC-006 | Medium | Chat deletion makes attachment blobs/vectors undiscoverable to cleanup | Local + main + PR; isolated PostgreSQL cascade confirmed |
| SEC-007 | Medium | Concurrent permanent uploads can exceed configured org quotas | Local + main + PR; confirmed non-atomic check/use sequence |
| SEC-008 | Medium | API-key lifecycle changes commit before audit events | Local + main + PR; confirmed transaction boundary |
| SEC-009 | Low | Login denials are stored as successful audit results | Local + main + PR; isolated PostgreSQL regression confirmed |
| SEC-010 | Medium | Python CI dependency audit scans the wrong environment | Local + main + PR; reproduced with scanner inventories |
| SEC-011 | Medium | Known vulnerable runtime dependency and omitted build dependencies | Local + main + PR lockfiles; live advisory matches; exploit reachability qualified below |
| SEC-012 | Medium | Completed agent/provider work can escape usage accounting on cancellation | Main + PR, predates PR; confirmed control flow |
| LOCAL-001 | High | Failed ACL updates leave old broader Qdrant ACL searchable | Old shared baseline only; upstream remediation exists |
| PR-001 | Medium | Cancelled cache owners strand single-flight futures | Introduced by PR; deterministic reproduction |
| PR-002 | Medium | Alternative-embedding failure breaks Q1 fallback and loses usage | Introduced by PR; confirmed control flow |
| PR-003 | Medium | Comparison source lookup happens before usage commit | Introduced by PR; confirmed control flow |
| PR-004 | Medium | Removed ACL candidate still influences no-answer decision | PR concurrency/grounding defect; confirmed control flow, not proven content disclosure |

Severity uses realistic prerequisites: High threatens another user's session or shared tenant availability; Medium covers scoped authorization, confidentiality, integrity, cost, and resilience failures; Low covers narrower detection or hardening defects. No pre-auth remote-code-execution finding was established.

## 3. High-severity application findings

### SEC-001 — Stored active content in the document preview

**Impact:** an ordinary uploader can cause code to run in the origin of a more privileged viewer, potentially acting through that viewer's authenticated session.

Evidence:

- Upload preserves the client MIME type in [documents.py:100](https://github.com/marketcalls/ragz/blob/3fac9fb/backend/src/ragz/api/routes/documents.py#L100) and `modules/documents/service.py:127-146`.
- The original-file response uses stored MIME and inline disposition in [documents.py:186](https://github.com/marketcalls/ragz/blob/3fac9fb/backend/src/ragz/api/routes/documents.py#L186).
- The frontend accepts every `text/*` and `image/*` for preview and renders the resulting blob URL in an iframe without a sandbox: [document-viewer-drawer.tsx:10](https://github.com/marketcalls/ragz/blob/3fac9fb/frontend/src/features/documents/document-viewer-drawer.tsx#L10), [line 130](https://github.com/marketcalls/ragz/blob/3fac9fb/frontend/src/features/documents/document-viewer-drawer.tsx#L130); `document-file.ts:31-38,67-75` retains Blob MIME.

Attack conditions: contributor can upload to a workspace; another user can read the file and opens its preview. An HTML document does not need successful RAG parsing to remain stored and downloadable. A script in a same-origin blob iframe can interact with the parent/application origin. Keeping the access token in memory and refresh token HttpOnly is useful but does not prevent same-origin script from making authenticated requests. The application itself refreshes through a same-origin POST that returns the access token (`frontend/src/api/client.ts:23-39`).

Fix: validate supported formats on the server, including content signatures; make active/unknown types download-only; render plain text as escaped text; use a narrow safe-preview allowlist. Sandbox document frames without script/same-origin privileges, or move previews to an isolated origin. Apply the policy to legacy uploads too. Browser and API security headers are defense in depth, not a substitute for fixing the blob sink. See [OWASP upload guidance](https://cheatsheetseries.owasp.org/cheatsheets/File_Upload_Cheat_Sheet.html).

Verification: two independent source reviews traced the route-to-blob-to-iframe path. Browser execution/session access was not performed. Required regression: a harmless active-content fixture must not execute or access the parent when opened from the actual document UI.

### SEC-002 — Attachment ingestion has no aggregate admission control

Evidence: [chats.py:403](https://github.com/marketcalls/ragz/blob/3fac9fb/backend/src/ragz/api/routes/chats.py#L403) applies the attachment-create permission but no upload rate dependency. Lines 409-434 buffer up to `interactive_upload_mb` into a `bytearray`, copy it to `bytes`, store it, and enqueue extraction. The default is 50 MiB per attachment (`core/config.py:115-116`). [attachments.py:61](https://github.com/marketcalls/ragz/blob/3fac9fb/backend/src/ragz/modules/chat/attachments.py#L61) has no per-user/org byte, attachment-count, pending-job, or spend reservation. `_ATTACHMENT_KINDS` is declared but not enforced. `worker/tasks.py:253-278` loads the object and runs parsing.

Any user granted `chat.attachments.create` can repeatedly consume shared API memory, object storage, database rows and worker capacity, including uploads never used in a message. Permanent-document quotas do not account for these attachments. Chat message rate limits do not apply to the upload route. This is a tenant availability/resource-abuse risk, not a claim of parser RCE.

Existing mitigations: per-file 50 MiB cap, permission/ownership checks, and upstream Celery 30/33-minute soft/hard task limits. These bound one operation, not aggregate workload. Permanent document streaming in earlier PRs did not convert this attachment route to streaming.

Fix: atomic count/byte/work reservations per user and organization, upload rate/concurrency limits, bounded pending jobs, parser/page/decompression limits, and content-type validation before enqueue. Stream attachment transfer. Expire unused attachments promptly and fix SEC-006 so deletion actually reclaims resources. Validate rejection before object writes/enqueue with isolated small fixtures; a destructive load test is unnecessary to prove the missing controls.

## 4. Medium and low application findings

### SEC-003 — Org administrators can self-grant restricted permissions

`TenantContext` intentionally excludes `audit.read`, `audit.export` and `documents.acl.bypass` from an admin's automatic grant (`modules/tenancy/context.py:42-49,79-92`). However, administrators can list active global templates and assign one to themselves. The [assignment route](https://github.com/marketcalls/ragz/blob/3fac9fb/backend/src/ragz/api/routes/users.py#L38) uses `AdminDep`; [assign_custom_role](https://github.com/marketcalls/ragz/blob/3fac9fb/backend/src/ragz/modules/tenancy/service.py#L673) accepts same-org admin targets without an independent-grantor or self-assignment check. The next context build supplies the restricted permissions.

This defeats the documented separation of audit/content access from IAM administration. It does not grant the platform superadmin role or another organization's access. If unilateral self-assignment is intended, the security claims and product controls must explicitly say so.

Fix: restrict sensitive template grants to an authorized independent grantor/superadmin, or introduce a separate permission for granting these permissions and prohibit self-grant. A blanket self-assignment ban alone is insufficient if the same actor can create/assume another account with equivalent privileges; define the grant authority explicitly.

Verification: retained `security_audit_auth.py` creates synthetic users/templates in isolated PostgreSQL and proves both `audit.read` and `documents.acl.bypass` become effective after self-assignment. No real account was changed.

### SEC-004 — Inconsistent egress policy permits shared/private-overlay IPv4 space

[media.py:151](https://github.com/marketcalls/ragz/blob/3fac9fb/backend/src/ragz/modules/chat/media.py#L151) blocks private/loopback/link-local/reserved/multicast/unspecified IPs, but not `100.64.0.0/10`. `core/net.py` explicitly blocks that range, so the two policies disagree. `web_content.py:127-163` reuses the media guard for search-result pages and redirects.

The stdlib deliberately classifies this range as neither private nor global ([Python documentation](https://docs.python.org/3/library/ipaddress.html#ipaddress.IPv4Address.is_global)). A public result page can redirect to an HTTP service in this range and pass the guard. The impact is conditional on the API host reaching sensitive services in shared address space, for example an overlay network. This is not a bypass to arbitrary RFC1918 or cloud metadata addresses; those tested guards remain effective.

Verification: `reproduce_egress_guard.py` uses a mocked resolver and HTTP transport. The production guard accepted `100.64.0.1`; a simulated redirect returned the synthetic internal text. No real private destination was contacted.

Fix: centralize address classification and reject non-global destinations, with explicit deployment allowlists only where required. Retain DNS pinning and redirect revalidation. Test direct and redirect paths across both IPv4 and IPv6.

### SEC-005 — Browser query cache survives passive authentication failure

[client.ts:23](https://github.com/marketcalls/ragz/blob/3fac9fb/frontend/src/api/client.ts#L23) clears the in-memory token on failed refresh. `RequireAuth`'s failure callback only navigates to login (`app/require-auth.tsx:28-35`). Explicit logout clears TanStack Query, but successful login only stores the new access token (`features/auth/mutations.ts:13-41`). Query keys such as workspace/admin lists and chat IDs are not comprehensively namespaced by authenticated identity.

Prerequisite: identity A's session fails while the SPA remains alive; identity B logs in in that same browser/tab before cached data is collected. Cached A data can remain observable/render during refetch or under matching keys. This is a shared-browser confidentiality issue, not backend authorization bypass.

Fix: centralize authentication teardown, cancel/clear sensitive queries before leaving the authenticated UI, and clear or partition caches on identity changes. Account for in-flight A requests completing after B logs in. Required regression: failed-refresh → B login never renders or retains A's cached data. Verification here is source/state-flow inspection, not a two-account browser run.

### SEC-006 — Chat deletion orphans attachment objects and vectors

[chat/chats.py:92](https://github.com/marketcalls/ragz/blob/3fac9fb/backend/src/ragz/modules/chat/chats.py#L92) deletes the chat and commits; `chat/models.py:118-123` cascades attachment rows. The cleanup job discovers targets only through existing stale attachment rows (`chat/attachments.py:157-172`; `worker/tasks.py:381-418`). Once the cascade removes those rows, the object keys and ephemeral vector identifiers are no longer discoverable by this job. No object-inventory fallback was found.

Impact: data survives expected deletion and storage can grow permanently. Combined with SEC-002, repeated upload/delete cycles defeat the application's cleanup mechanism. Existing object access checks mean this is not evidence of public access to the orphaned files.

Fix: create durable deletion jobs containing external identifiers before the relational cascade; retry external cleanup idempotently; add an inventory reconciliation mechanism for legacy orphans. Treat user-visible deletion as pending until required external deletion is confirmed.

Verification: real isolated PostgreSQL confirms the cascade and that even a future cutoff cannot rediscover the attachment. Object/vector state was represented by synthetic external sets; real MinIO/Qdrant deletion failures were not injected. The test plus code trace establishes why the production cleanup loop receives no deletion targets.

### SEC-007 — Organization upload quota check is not a reservation

`modules/documents/service.py:25-66` counts current documents/sums bytes, then upload/storage/commit happens later (`:69-147`). Concurrent uploads can observe the same remaining allocation and all pass. There is no serialized org reservation or in-flight byte ledger. Default org document/storage caps are also disabled (`core/config.py`, `org_max_documents`, `org_max_storage_bytes`).

Fix: atomically reserve count/bytes against the organization before upload; include in-flight reservations; finalize or release them on success/failure. Test concurrent small uploads at a one-document/small-byte boundary. Existing quotas remain useful for sequential traffic, but are not a hard bound under concurrency. Local CAG work raises the per-file default to 1 GiB, increasing the consequences of this gap; upstream main/PR retain 100 MiB.

### SEC-008 — API-key lifecycle and audit events are separate transactions

[api_keys_service.py:63](https://github.com/marketcalls/ragz/blob/3fac9fb/backend/src/ragz/modules/auth/api_keys_service.py#L63) commits key creation before the route records and commits the audit event (`api/routes/api_keys.py:17-25`). Revocation repeats the pattern (`api_keys_service.py:93-97`; route `:41-47`). A crash, cancellation or second transaction failure can leave an unaudited credential state change.

The routes require privileged access; this is an audit-integrity/recovery gap, not unauthenticated key issuance. A key committed before a failed response is generally unusable by the caller because the one-time raw value was never delivered; it is still an unaudited orphan credential record. Revocation's independently committed state change remains directly meaningful. Fix: flush the lifecycle change, add its audit event in the same transaction and commit once. Test audit/commit failure without retaining a key state change or reporting success.

### SEC-009 — Authentication-denial result is recorded as success

[auth/service.py:125](https://github.com/marketcalls/ragz/blob/3fac9fb/backend/src/ragz/modules/auth/service.py#L125) records `login.failure` without setting `result`. [audit/service.py:20](https://github.com/marketcalls/ragz/blob/3fac9fb/backend/src/ragz/modules/audit/service.py#L20) defaults to `"success"`. Similar OIDC denial paths exist. Useful source-IP/auth-method/reason fields are not supplied on these calls.

The action name still identifies a failure, so the event is not completely lost. However, consumers filtering the result field get incorrect results and incident attribution is weaker. Fix explicit denial results/reason codes and trusted client IP/auth-method identifiers. The isolated regression confirms the stored contradictory result and missing fields.

### SEC-010 — Backend CI audit scans its own tool environment

[audit.yml:73](https://github.com/marketcalls/ragz/blob/3fac9fb/.github/workflows/audit.yml#L73) runs `uvx pip-audit --strict --local` after `uv sync`. `uvx` executes in an isolated environment, not the project environment ([uv documentation](https://docs.astral.sh/uv/guides/tools/)). With no requirements/path target, pip-audit audits that tool environment ([pip-audit documentation](https://github.com/pypa/pip-audit)).

Reproduction from the PR backend directory:

| Invocation | Packages enumerated | Result |
|---|---:|---|
| Existing CI command, JSON output added | 29 | No advisories; includes pip-audit and its dependencies, omits FastAPI/Docling/Transformers |
| Frozen production lock export, explicit `pip-audit -r` target | 212 | One Transformers advisory |

Fix: audit an explicit frozen export including transitive requirements, or execute the scanner with the actual application environment/path. Assert the audit inventory contains representative application packages so a tool upgrade cannot silently change coverage. Do not interpret the current gate's green result as a clean backend dependency tree.

### SEC-011 — Dependency exposure and build-tree audit gaps

Both production Python lock exports contain `transformers==5.8.1`, matched by CVE-2026-9856 / GHSA-xrqw-3rrv-vx5w. The reviewed advisory describes path traversal in tokenizer/processor `save_pretrained` with attacker-controlled chat-template names; patched version is 5.10.0. [Advisory](https://github.com/advisories/GHSA-xrqw-3rrv-vx5w).

This is a **confirmed vulnerable dependency**, not a proven document-upload arbitrary-file-write exploit in Ragz. No direct application call to `save_pretrained` or tenant-controlled Hub model selection was found. Review transitive parser/model behavior and update within compatible constraints. Upstream advisory severity is High; application priority here is Medium pending reachable-path evidence.

Both frontend production-only audits returned zero advisories. Full lock audits returned eight unique GHSA IDs in development/build dependencies. The current `pnpm audit --prod --audit-level=high` gate excludes them. Raw counts differ by pnpm version and duplicate affected ranges; they are not eight distinct remotely exploitable Ragz vulnerabilities.

| Package(s) observed | Unique advisory IDs | Scope |
|---|---|---|
| js-yaml 4.2.0 / 4.3.0 | GHSA-52cp-r559-cp3m; GHSA-5p4m-2wfm-xmqj | Dev/build dependency paths |
| brace-expansion 1.1.16 / 2.1.2 / 5.0.7 | GHSA-mh99-v99m-4gvg; GHSA-rgw5-rvv9-x895 | Dev/build dependency paths |
| postcss 8.5.19 | GHSA-fxqj-rqcc-2cmp | Dev/build; scanner labels moderate |
| nanoid 3.3.16 | GHSA-2v37-7h3g-55p8 | Dev/build dependency paths |
| browserslist 4.28.6 | GHSA-c83g-rgw3-j3cx; GHSA-73wf-gq98-2v4g | Dev/build dependency paths |

Exact affected paths, descriptions and advisory URLs are in the retained scanner JSON. Fix compatible transitive versions and add a build/development dependency gate. Assess attacker-controlled configuration/build-input prerequisites separately from browser runtime exposure.

### SEC-012 — Agent cancellation can discard completed provider usage

`modules/chat/service.py:899-964` forwards intermediate agent/tool frames before receiving the final `AgentGathered` totals and persisting web/planner usage. Cancellation before that aggregate, or while its commit is pending, can omit already-performed paid work. This behavior predates PR #11; the PR's additional commit does not cover the earlier interval.

Impact: incomplete usage ledger and weakened quota/cost attribution; repeat cancellation can increase the discrepancy. This does not imply all provider-side budgets are bypassed. Fix per-completed-call durable usage recording and cancellation-safe finalization, with a clear idempotency key to prevent double charging. Test cancellation after a mocked billable action and before its frame/final aggregate; reconcile incurred provider usage independently from answer persistence.

## 5. Local-only exposure and earlier audit reconciliation

### LOCAL-001 — Failed ACL projection in the old shared checkout

The shared baseline's `modules/documents/service.py:272-310` commits the relational ACL before updating Qdrant. If Qdrant rejects the update, the older broader payload remains searchable. The old retrieval path lacks the upstream pending/failed projection exclusion. This remains relevant to the checkout carrying the CAG changes.

Upstream `9d08839` and PR #11 add security revisions, pending/failed exclusion, compare-and-set activation, and reconciliation (`modules/documents/service.py:290-405`; retrieval prefilter/recheck). That addresses the demonstrated failure-leaves-old-ACL state; it is not a PR-introduced defect. Do not integrate the CAG branch by overwriting these newer security/lifecycle changes with old files.

| Earlier issue | Current disposition |
|---|---|
| Indefinite broader ACL after failed projection | Remediated upstream for the tested pending/failed state; old local checkout still exposed |
| Document upload/delete commit followed by lost enqueue | Upstream outbox/job recovery materially addresses this; do not repeat as an unchanged PR finding |
| Non-hermetic ambient service use in backend tests | Upstream autouse test environment redirects PostgreSQL/Redis/Qdrant/MinIO and Celery; used by this audit |
| Missing production packaging / ordinary PR quality gate | Added upstream; exact PR run status still requires attention |
| Missing composite resource tenant constraints | Forward migration exists upstream |
| Active document preview | Still present in upstream and PR; planning a remediation was not implementation |
| Ephemeral attachment cleanup after chat cascade | Still present; document outbox work did not fix this separate lifecycle |

## 6. PR #11 assessment

See [the complete PR review](2026-09-06-pr-11-security-review.md) for changed-code findings, every existing automated-review comment's disposition, and acceptance checks.

Merge recommendation: **request changes** for PR-001 through PR-004 and the invalid-default-model regression. Fixing these does not resolve the inherited application findings, especially SEC-001. Retain MQR's opt-in setting and bounded configuration.

As observed during this audit, GitHub reports both CI and Dependency audit runs for head `3fac9fb` as `action_required`, not passing: [CI run](https://github.com/marketcalls/ragz/actions/runs/33836126170), [dependency run](https://github.com/marketcalls/ragz/actions/runs/33836126191). Only the review-bot check had a success conclusion in the PR check rollup. Repository required-check/branch-protection settings were not verified or changed.

## 7. CAG review: infrastructure is not yet an authorized replay path

Positive properties observed in the unfinished local module: separate Redis service/client; default disabled; bounded record size/TTL configuration; org/workspace namespace; user/role/group principal hash; version material in the key; atomic Redis epoch comparison and temperature update; token-checked lock release; fail-open cache-error behavior.

Required before enabling production answer replay:

1. Read and update authoritative corpus/security/prompt/cache revisions in PostgreSQL transactions for every relevant mutation. Matching two caller-supplied epoch values in Redis does not itself prove freshness.
2. Revalidate live user/membership/permissions and every source's current authorization/version before emitting a cached answer. Include custom-role permission changes and disabled model/provider configuration in invalidation semantics.
3. Restrict eligibility to complete validated answers with provenance. The adapter currently accepts flags such as `validation_failed`; callers must enforce admission policy.
4. Either include conversation/history/attachment/web/tool/reasoning context in identity or explicitly exclude those requests. The current builder does not contain those fields.
5. Decide whether case folding, whitespace collapse and NFKC normalization are safe for the supported queries. They can collapse distinct case-sensitive identifiers/code or whitespace-sensitive input; the cache is not byte-exact.
6. Fence fill writes and handle fill-lease expiry/provider calls that exceed the 30-second default. A distributed lock alone does not guarantee exactly one paid call or prevent a late writer.
7. Bound cache socket waits and total miss/wait time; propagate cancellation deliberately. Apply per-tenant memory/admission policy to limit eviction interference.
8. Test actual PostgreSQL authorization changes racing lookup/emission, principal changes, cold-fill cancellation, and SSE/history equivalence. The CAG session's supplied-epoch benchmark is useful module evidence, not an end-to-end authorization-race result.

These are unfinished integration requirements, not claims that a currently enabled cache leaks answers. The module has no production chat lookup consumer in the reviewed worktree. No changes were sent to the other thread.

## 8. Conditional risks and hardening observations

- **Bot audience versus Ragz identity:** `modules/bots/platforms.py:57-107` extracts channel/chat ID and text but not an authenticated Ragz sender. `api/bots_relay.py:39-62` runs all messages as the integration's configured user. Authentic platform signatures prove delivery origin, not the end user's document entitlement. If a bot is reachable by users outside the intended private audience, they inherit the integration user's access; Telegram direct messages particularly require an explicit audience policy. Use sender/account binding or a strict organization/channel/user allowlist and a least-privileged bot corpus. Service-account bots intentionally publishing a public corpus have a different risk profile. Platform account restrictions were not available for validation.
- **Historical answers after revocation:** chat history retains previously generated document-derived text/citations (`chat/messages.py:64-76,127-182`). Current file access still checks ACLs. Decide whether revocation is prospective or requires hiding historical derived content; previously authorized disclosure cannot be undone. Do not call intentional historical snapshots a new cross-user leak.
- **Completed security-revision race requires a dedicated regression:** current retrieval rechecks the set of *unprojected* document IDs, not a complete revision snapshot. A revoke/project transition that finishes during an in-flight query needs explicit testing, especially direct search which serializes chunk text without chat's source-metadata authorization check. This is an open concurrency hypothesis, not included as a confirmed current cross-tenant leak.
- **Admin-configurable OIDC/SMTP DNS rebinding:** `core/net.py` documents check-then-re-resolve behavior. Public page/media fetching pins DNS; privileged configuration egress does not. Operator-only targets and egress network controls affect severity. Prefer pinned connectors/egress allowlists while preserving intentional internal identity/mail services.
- **Upload rollback orphans:** a successful object write followed by DB/audit failure can leave an unreferenced object (`documents/service.py:135-146`; `chat/attachments.py:70-76`). Compensating deletion and inventory reconciliation should accompany SEC-006.
- **API-key prefix collisions:** only four random characters remain in the 12-character lookup prefix after fixed `ragz_sk_`; lookup assumes one row (`api_keys_service.py:31-32,105-109`). A collision can cause auth errors. Increase prefix entropy and safely handle candidates/uniqueness. Key issuance is privileged; this is not evidence of brute-forceable full keys.
- **SPA headers and image builds:** upstream `frontend/deploy/nginx.conf:1-40` sets no CSP/frame-ancestors/nosniff policy for the app shell; verify the outer proxy. Several Docker build bases and `minio/mc:latest` are not digest-pinned; no built-image scanner gate was found. Pin/update deliberately and scan final images. Internal compose stores are not automatically public merely because the web service publishes a port.
- **Citation navigation:** `source-panel.tsx:84-93` lacks the HTTP(S) validation used by other renderers. React 19 demonstrably replaces `javascript:` hrefs with a blocking error, and links use `noopener`; a direct JavaScript-XSS finding was retracted. Apply consistent scheme validation as low-priority navigation hardening.
- **Logging:** field-name redaction in `core/logging.py` is shallow and happens before exception rendering. Review nested provider failures, credential-bearing request URLs and optional Sentry telemetry. No live secret leakage was claimed or printed during this assessment.

## 9. Verification and evidence

| Check | Outcome | Limits |
|---|---|---|
| Exact PR source import verification | Passed | Explicit `PYTHONPATH`; did not trust the shared editable install |
| Focused backend suite | **223 passed**, 8 warnings, 229.16 seconds | Entire isolation directory plus selected auth/OIDC/API-key, route-policy/enforcement/alignment, middleware and webhook tests, and attachment orphan regression |
| Synthetic auth regressions | **2 passed**, 7 warnings | Real isolated PostgreSQL; proves current self-grant and contradictory audit result |
| PR cache cancellation reproduction | Both cache variants leave dead slots | Pure async, controlled lock scheduling; no paid provider |
| Shared-address egress probe | Guard accepts CGNAT and returns synthetic redirected text | Mock DNS + HTTP; no private host contact |
| PR patch whitespace / Python compile checks | Passed | Not a substitute for functional/security tests |
| Python production lock audit | One advisory each; 211 local packages / 212 PR packages | Linux/current interpreter markers; Windows/macOS-only packages may be omitted |
| CI-command inventory comparison | Wrong 29-package tool environment confirmed | Audit invocation bug independently reproduced |
| Frontend lock audits | Production: zero advisories; full: eight unique advisory IDs | Dev/build exposure and duplicated-range counts separately identified |
| React href rendering | `javascript:` is blocked | Clears that specific false-positive hypothesis, not all URL risks |
| Git history secret scan | Gitleaks 8.30.1 scanned 603 commits / 8.48 MB; three candidates in synthetic test fixtures | All-ref clone history; findings redacted; no production credential confirmed |

Tests ran against the exact PR source with the available Python 3.13 environment and a temporary `prometheus-client` overlay. They did not constitute a clean reinstall of the PR's Python 3.12 lock environment. Testcontainers supplied fresh PostgreSQL/Redis/Qdrant/MinIO instances; mocked model/provider seams avoided paid calls. No existing application services/databases were deliberately mutated. One early attachment proof used the shared editable install; it was superseded by the correctly sourced 223-test run, which includes that regression.

Runtime limitations: browser execution of uploaded active content was not run, production service configuration was not tested, and a dedicated completed-revision concurrency probe was not executed. Two delegated runtime-probe requests were stopped by an automated safety filter; their findings were retained as source evidence or open hypotheses rather than falsely marked runtime-verified. The remaining permitted static checks and isolated tests were completed.

Retained evidence is in [2026-09-06-security-evidence](2026-09-06-security-evidence/README.md). It includes dependency/secret scanner JSON, regression scripts/tests, and JUnit results. The three secret-scan matches were inspected in context: a literal used to seed/resolve a synthetic API key, an explicitly fake web-redaction key, and an invitation token submitted by a test. They do not establish production credential leakage; no secret values were printed. Local untracked environment files and unrelated benchmark repositories were not part of the history scan. Tests that intentionally assert the vulnerable behavior are evidence, not remediation tests: their passing result demonstrates the defect, not that the application is secure.

## 10. Remediation order and release acceptance

1. Close SEC-001 and add a browser-level safe-preview regression covering existing and new uploads.
2. Enforce attachment admission and durable cleanup (SEC-002/006), including aggregate quota reservations (SEC-007).
3. Define independent grant authority and close self-grant (SEC-003); correct the egress address policy (SEC-004) and identity-transition cache cleanup (SEC-005).
4. Repair PR-001 through PR-004 and invalid model-default handling; verify with cancellation/failure and authorization-transition regressions.
5. Fix the dependency audit target, update vulnerable lock entries, include build dependencies, and obtain actual CI success on the final PR head.
6. Make credential/usage audit writes durable and semantically accurate (SEC-008/009/012); add fault-injection checks.
7. Bring the CAG work forward onto the newer security/lifecycle baseline, preserve those fixes, then complete CAG's authorization/invalidation gates before enabling replay.

Release acceptance should include an independent rereview of these fixes, actual browser upload-preview coverage, tests of concurrent quota/fill/ACL transitions, dependency inventories for the built runtime, and confirmation of the deployed proxy/bot audience/network controls. Reports and evidence were saved locally; no code fix, PR review/comment, push, merge or deployment was performed.

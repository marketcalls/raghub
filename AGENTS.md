# Ragz — agent instructions

Ragz is a self-hosted, multi-tenant document RAG application: Python/FastAPI,
PostgreSQL/SQLAlchemy, Qdrant, Celery/Redis, MinIO, LiteLLM, and React/TypeScript.
This file replaces the former root `CLAUDE.md`; its applicable engineering and
security rules are preserved here. Treat stated invariants as requirements to
verify, not proof that the present implementation already satisfies them.

## User constraints: preservation and model routing

- Preserve existing files, benchmark work, datasets, reports, failed runs,
  databases, volumes, containers, and unrelated edits. **Do not delete existing
  material or perform cleanup without explicit authorization for exact targets.**
  The user's exception for removing root `CLAUDE.md` does not authorize any
  other deletion. In particular, `no_rel/` is valuable project work, not trash.
- No `git reset --hard`, `git clean`, checkout-overwrite, destructive stash use,
  bulk replacement, or Compose volume teardown on the shared workspace.
- Use `apply_patch` for source/document edits. Inspect overlapping user changes
  first. Use an isolated checkout for incompatible branches; preserve the shared
  checkout and all pre-existing worktrees. Do not automatically push, merge,
  publish reviews, or deploy unless the user requests that action.
- **Maximum two active Sol agents**, counting a Sol coordinator and every Sol
  descendant in the coordinated task tree. A typical arrangement is one Sol
  implementer/coordinator and one Sol reviewer. Waiting on tools still occupies
  a slot; release it only after completion or confirmed interruption.
- The coordinator owns model scheduling. Children must not independently spawn
  or reactivate Sol. Check active agents before each Sol dispatch/reactivation;
  queue additional Sol work. Account for other visible Ragz sessions. If their
  activity cannot be established, stay with the primary Sol and Luna workers;
  do not claim that a partial view enforces a global application-wide quota.
- **Use Luna at `xhigh` for volume tasks:** web research, broad code searches,
  evidence extraction, dependency/report triage, routine test execution, and
  bounded repetitive changes under an approved design. Tool identifiers:
  `gpt-5.6-luna`, `reasoning_effort="xhigh"`. Do not silently substitute lower
  effort or fan out Sol workers for this work.
- Reserve Sol for architectural judgment, sensitive auth/ACL/cache/transaction
  changes, integration and final review. Give Luna concrete boundaries,
  acceptance checks and ownership; independently review security-sensitive work.
- Delegate only independent useful tasks. Avoid shared-file writers and duplicate
  reviews. Reuse agents where appropriate; keep outputs concise and source-linked.

## Start with the actual revision and task

Read `git status`, branch/HEAD and applicable diffs. For PR work, establish the
current remote base/head without switching the shared checkout. Local main,
upstream main, PR code and dirty benchmark builds may differ substantially.
Never overwrite newer security/lifecycle changes with older local files.

Read only the references relevant to the task:

- Product requirements: [docs/prd.md](docs/prd.md).
- Architecture decisions: [docs/adr/](docs/adr/), when present on the revision.
- Security findings: [application audit](docs/audits/2026-09-06-end-to-end-security-audit.md)
  and [PR #11 review](docs/audits/2026-09-06-pr-11-security-review.md).
- Evaluation: [security/RAG evaluation](docs/audits/2026-09-08-security-and-rag-evaluation.md).
- CAG/runtime/budget context: [benchmark handoff](docs/handoffs/2026-09-08-cag-benchmark.md).
- Remediation workflow: [Sol prompt](docs/prompts/sol-security-remediation.md).

Historical reports retain commit-specific references to `CLAUDE.md`; those are
evidence, not a requirement to recreate the file. If a referenced foundation,
plan or ADR is absent, search the applicable revision and use available product
requirements/code evidence; record missing context rather than inventing it.
User instructions take precedence. The CAG handoff records a preference against
Superpowers-branded workflows; do not introduce them unless the user changes it.

## Security and architecture invariants

1. Tenant-owned relational operations use verified `TenantContext` and explicit
   object/workspace scope. Keep tenant predicates and constraints intact.
2. Qdrant filters are built only in `modules/retrieval/`, including tenant,
   workspace, ACL-group and current-version restrictions in every query lane.
   A post-query rejection can strengthen a correct query but cannot replace its
   authorization filter. Validate authorization/projection changes during reads.
3. Restricted document existence may be listed to workspace members (the product's
   Drive-style policy); contents/chunks/citations require content authorization.
   Hide ACL-group metadata from ordinary users. Do not silently change historical
   answer retention or bot-audience policy as part of another fix.
4. Provider secrets remain envelope-encrypted in PostgreSQL, decrypted through the
   sanctioned secrets service; the KEK stays outside the DB. Credential schemas
   are write-only except explicitly authorized one-time issuance. Never expose
   secrets in logs, exception output, reports, browser bundles or new plaintext
   configuration. Preserve legitimate bootstrap configuration and protect its files.
5. Use declarative route permissions, Argon2id, short-lived signed access tokens,
   rotating refresh families and current-user/security-version verification.
   UI visibility is not authorization. Sensitive grants require an authorized
   grantor, not merely an admin label.
6. Retrieved documents and web results are untrusted data; model output and file
   previews are untrusted browser content. Preserve safe markdown, structured UI
   validation, read-only agent tools and grounded no-answer behavior. Preserve
   PPTX/OCR support and original PDF page/version/section provenance.
7. Bound uploads, aggregate tenant storage/work, parsing and provider execution.
   CPU-heavy parsing belongs in workers. Quota admission requires atomic
   reservations where concurrency matters. Usage, credential audit and deletion
   records must survive their relevant failure/cancellation boundaries.
8. Authorization/quotas fail closed when unavailable. Cache failure may fall back
   to ordinary authorized computation; it must never grant access or serve stale
   content. Preserve typed RFC 9457 errors and non-sensitive observability.

Layers: `api/` and `worker/` are thin entrypoints using module public services or
views; modules use `core/` and public module interfaces. Avoid new sibling ORM
imports or ORM objects in responses. Honor the revision's import-linter contracts.
Auth owns credentials; tenancy owns permissions/membership; documents/chat own
their lifecycles; retrieval owns vector access; quotas/audit own ledgers; outbox
owns durable dispatch where implemented; cache owns cache mechanics. Record
architectural changes in an ADR. Prefer deep, cohesive services to scattered checks.

## CAG, runtime and benchmark discipline

- **Current user clarification: CAG is actively under development and is for
  development/testing only. It is not a released production feature.** This
  supersedes the handoff's historical paused-status description; do not infer
  current activity or completion from that snapshot.
- CAG means a Redis-backed response cache, not stuffing the corpus into model
  context. Preserve the separate ephemeral response-cache instance; operational
  Redis contains queues/rate counters and is not interchangeable with it.
- Preserve the active CAG developer's work and coordinate overlapping files.
  The security-remediation Sol thread must not take over or complete the CAG
  feature unless explicitly assigned that work. Track unfinished CAG integration
  as development work, not automatically as a deployed vulnerability or a blocker
  for unrelated fixes. Actual defects affecting shared/production paths remain
  in scope regardless of whether CAG exposed them during testing.
- CAG may be explicitly enabled in isolated development/test environments for
  authorized implementation and benchmarks. Keep production answer replay and
  production defaults disabled until authoritative PostgreSQL epoch updates,
  live principal/source authorization, source/version freshness, eligibility,
  normal persistence/SSE and fenced fills are implemented and tested together.
  Passing test-environment checks does not itself authorize production rollout.
- Use `docker --context default` explicitly. Recheck runtime state; handoff service
  names/ports/health are historical. Never flush shared Redis, reuse campaign DBs
  as test DBs, or remove preserved benchmark containers/volumes. Fresh disposable
  test resources may use their normal teardown; this excludes pre-existing resources.
- No further paid-provider calls without explicit user direction or a fresh
  development budget. Reconcile the existing cost ledger, including uncertain
  external rerank charges, before execution. The prior roughly $3 limit was a
  development constraint, not a production product limit.
- Preserve frozen datasets and failed/invalidated runs. New code/configuration or
  labels require a new run/dataset version. Never tune on frozen evaluation labels.
- Report native vs normalized configuration, model/parser/chunker, thresholds,
  revision/diff hash, resources, sample sizes, failures, latency boundaries and cost.
  Hash-embedding timings are not hosted semantic-embedding timings; cache lookup
  latency is not authenticated chat latency; Redis clients are not concurrent users.
- Score only completed valid systems. Resource/budget-gated RAGFlow/Onyx/LightRAG
  runs get no fabricated zeros. Term matching is not semantic correctness; citation
  structure is not entailment; product no-answer flags are not semantic refusal.
  Keep raw queries/answers/reference content and credentials out of public artifacts.

## Verification and completion

For bug fixes, reproduce the failure and add a focused regression that fails before
the patch and passes afterward. Retained audit proofs may intentionally assert
vulnerable behavior: adapt them to the secure invariant rather than treating their
existing pass as proof of remediation. Use actual isolated stores for integration
behavior and mocked providers for deterministic faults/cancellation, without charges.

Verify imports target the intended checkout (`ragz.__file__` and affected modules);
an editable shared virtualenv can silently import old local code. Inspect fixtures
for isolation of PostgreSQL, Qdrant, Redis, MinIO, Celery and KEK before tests.

Run relevant checks, using the selected revision's dependency/tool versions:
backend pytest, Ruff, mypy and import-linter; frontend tests, ESLint, TypeScript and
build; migrations, API schema and focused browser tests when changed. Audit the
actual frozen application dependency inventory, including a separately assessed
build tree; scan built images when they are in scope. Verify CI truly executed—
`action_required`, skipped jobs and a review-bot success are not passing CI.

Maintain an issue ledger for multi-finding remediation: ID, affected revision,
owner/status, patch, failing-before/passing-after evidence, review and residual risk.
Do not call all bugs fixed with unresolved confirmed findings. Finish with exact
changes, checks/results and genuine blockers; no unsupported all-clear claims.

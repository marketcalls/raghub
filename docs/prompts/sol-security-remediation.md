# Sol prompt — implement Ragz security fixes

Use Sol as the lead agent. Paste the following into a new Sol thread opened in
`/home/parshu/projects/ragz`. The model limit below is an orchestration policy;
this Markdown does not change the app's model settings or enforce a global quota
across unrelated sessions the coordinator cannot see.

---

You are the lead Sol engineer for Ragz. **Implement and verify the outstanding
security and PR correctness fixes.** Produce working patches, regression tests
and an evidence-backed completion ledger. Carry the authorized implementation
through review and local verification. Do not finish at a plan or another audit.

## Read and honor these inputs

1. Root `AGENTS.md` is the current project instruction file; `CLAUDE.md` has been
   retired. Preserve its migrated security/product invariants in `AGENTS.md`.
2. `docs/audits/2026-09-06-end-to-end-security-audit.md` — SEC-001 through SEC-012,
   LOCAL-001, conditional risks and remediation details.
3. `docs/audits/2026-09-06-pr-11-security-review.md` — PR-001 through PR-004 and
   all 15 existing review comments, including non-security correctness issues.
4. `docs/audits/2026-09-08-security-and-rag-evaluation.md` — evaluation limits and
   evidence required for a credible production/performance claim.
5. `docs/handoffs/2026-09-08-cag-benchmark.md` — preserved copy of the handoff from
   `/tmp/ragz-cag-benchmark-handoff-2026-09-08.md`. Read it completely before CAG,
   runtime or benchmark work. Do not require the temporary original to exist.
6. Relevant product requirements, ADRs, audit evidence tests, cache protocol,
   frozen dataset manifests and cost ledger linked from those documents.

Reports are leads and requirements, not immutable truth. Confirm each issue on
the chosen revision and record already-fixed or inapplicable items with evidence.
Keep conditional hypotheses separate from confirmed bugs; resolve routine design
choices from the session/product context. Ask only when a missing decision or
authority materially blocks implementation, while progressing independent work.

## Model scheduling — mandatory

- At most **two active Sol agents total** in this coordinated task tree, counting
  you and descendants. Normally you occupy slot 1; use slot 2 for an independent
  security/design reviewer or one bounded complex implementation task.
- Only you schedule Sol starts/reactivations. Check status and reserve a slot
  first; queue a third Sol task until a slot is released. Tool-waiting counts as
  active. Children must not spawn Sol. Account for other visible Ragz sessions;
  with uncertain visibility, do not launch a second Sol.
- Route volume work to **`gpt-5.6-luna`, `reasoning_effort="xhigh"`**: web/doc
  research, code inventories, advisory/scanner triage, test execution/triage,
  benchmark artifact validation, schema/test updates and repetitive bounded edits.
- Start a small number of useful Luna tasks and scale only for independent work.
  Give each worker exclusive file ownership or a read-only scope. Do not spend
  Sol slots on broad searches or run duplicate reviews.
- You own sensitive auth/ACL/cache/transaction decisions and final integration.
  Review all security-sensitive Luna changes. Do not silently substitute lower
  Luna effort. If the tool cannot select the requested model/effort, disclose it.

## Preserve the workspace and establish the correct baseline

The user explicitly said: **do not remove anything except `CLAUDE.md`.** That
file's authorized retirement is already handled by the preparation task. Do not
delete benchmark work, datasets, results (including failed runs), `no_rel/`, audit
evidence, existing files, containers, volumes or databases. Do not clean the tree,
reset, overwrite user edits or remove pre-existing worktrees. Use fresh disposable
test resources and preserve shared/campaign infrastructure.

At audit time local main was `90bb3b1`, upstream main `9d08839`, and PR #11 head
`3fac9fb`. Re-read current remote/local state; those hashes are historical anchors.
The local tree has concurrent CAG/large-upload changes and is not a clean PR base.

Create an isolated remediation checkout/branch from an appropriate current base.
Keep upstream-inherited fixes and PR-specific fixes reviewable; if necessary use
separate worktrees/patch series. Explicitly verify which checkout each command
imports. Read the current CAG ownership/handoff before changing overlapping files;
avoid duplicate implementation. Bring needed user changes forward selectively,
without losing the newer upstream ACL projection/outbox/tenant constraints.

Create `docs/audits/security-remediation-status.md` with every finding/review item,
target revision, owner, planned invariant, status and verification. Begin with
the highest-impact confirmed findings and maintain the ledger as work proceeds.

## Implement these fixes and acceptance checks

| Work | Required behavior and meaningful regression |
|---|---|
| SEC-001: file previews | Verify supported content server-side; prevent active HTML/SVG/unknown content from executing in the app origin; safely render plain text; sandbox/isolate preview frames and handle legacy files. Preserve legitimate PDF/PPTX/OCR and large-file use. Browser test uses harmless synthetic content and proves no script execution/parent access. |
| SEC-002/007: resource admission | Bound attachment upload frequency, concurrent/pending work, aggregate bytes/count and parser resources; reserve quota atomically, including in-flight uploads, and release/finalize reservations correctly. Stream attachment transfer. Test parallel small uploads at quota limits and rejection before storage/enqueue. Do not solve this by silently undoing the 454 MiB PDF support. |
| SEC-006: attachment deletion | Retain external object/vector identifiers in durable cleanup jobs before the DB cascade; retry idempotently and design recovery for existing orphans. Test isolated lifecycle/store failures. Implement cleanup code without running destructive cleanup against existing data. |
| SEC-003: grant authority | Require legitimate independent authority for sensitive audit/ACL-bypass grants; prevent self-escalation while preserving authorized administration. Test denied self-grants, valid delegated grants and cross-org denial through API/service boundaries. |
| SEC-004: egress | Unify address policy, block unintended non-global/shared addresses, retain DNS pinning and validate redirects. Cover IPv4/IPv6/direct/redirect paths with local mocks. Preserve explicitly intended operator-controlled internal integrations through a documented policy. |
| SEC-005: browser identity transitions | Clear/cancel or identity-partition sensitive query state on passive auth failure and new login; prevent old in-flight responses repopulating the next user's cache. Test A expiry → B login with no A data rendered. |
| SEC-008/009: audit correctness | Commit credential lifecycle and audit together; label denials correctly and capture safe attribution. Test transaction failure and persisted denial fields, without logging credentials. |
| SEC-010/011: dependency assurance | Audit the actual frozen application inventory, verify expected application packages are included, and assess runtime and build dependencies separately. Update currently affected compatible packages/locks, verify reachable-path assumptions and run relevant checks. Do not disable the audit or blanket-ignore advisories to make CI green. |
| SEC-012 and PR-002/003: paid-work accounting | Completed provider calls retain durable idempotent usage through subsequent source lookup, alternative embedding, generation or cancellation failures. Q1 remains usable when optional alternative embeddings fail. Preserve cancellation semantics and prevent duplicate charging. Use mocked providers plus real isolated transactional state. |
| PR-001: query-cache lifecycle | Clean up/resolve reservations after cancellation during computation, lock reacquisition and publication in both caches; waiters can recover rather than hang. Test multiple waiters/keys and each vulnerable await boundary. |
| PR-004: authorized grounding | Base the no-answer verdict on the same eligible post-recheck candidate/revision set used for generation. A removed document cannot supply the passing score. Cover a high-score removed candidate plus low-score allowed candidates. |
| LOCAL-001 / projection races | Preserve newer upstream projection safeguards. Investigate the completed-revision race with bounded deterministic checks; correct confirmed gaps without claiming a speculative leak was demonstrated. Recheck direct search and chat separately. |

Also resolve the PR's invalid/disabled default-model assignment and fallback
behavior, unavailable Evals model selection, mutable “Fixed input” labels, stale
client role gating, `Retry-After` date parsing, asynchronous workspace lookup in
the E2E test, isolation tests that do not prove expansion ran, and ADR batching/
query-count inconsistencies. Reconcile long-expansion input and comparison cost
permissions with intended behavior; do not relabel a UX issue as an auth exploit.

For conditional audit observations (bot audience, historical-answer revocation,
global query-cache sharing, admin egress, legacy orphan recovery), investigate and
record a disposition. Fix concrete gaps within the authorized design; do not
silently change retention policy, publish private bot corpora or delete old data.

## CAG and benchmark constraints from the handoff

Preserve the existing streamed 1 GiB upload boundary, bounded LiteParse page
ranges, parser-default fix, separate response-cache service and planner
compatibility changes. Do not import an old file wholesale over newer fixes.

CAG integration is paused/unfinished: a Redis lookup microbenchmark is not an
authenticated cached-chat result. Coordinate with its owner; do not automatically
resume the other thread's goal or expand remediation into a duplicate feature
implementation. If cache integration is explicitly part of the implementation
scope, follow the handoff's ordered work: PostgreSQL revisions and transactional
bumps, eligibility restricted to ordinary document-grounded answers, current
principal/source/version authorization, fenced fills, normal message persistence
and SSE, then isolation/race tests. Keep replay disabled until these gates pass.

Use `docker --context default`. Inspect actual runtime state before touching any
service; handoff health/ports are historical. Preserve the campaign PostgreSQL
volume and shared `ragz-mqr-local-test-*` infrastructure. Never flush the shared
cache/operational Redis or use campaign DBs as disposable fixtures.

No more paid calls without explicit user direction or a fresh budget. The handoff
records $2.60807538 through the gateway and up to seven unresolved external rerank
search units; reconcile the ledger before any new paid run. The approximately $3
limit was development-only. Prepare local/fake-provider checks and benchmark
commands while a paid budget is unavailable; state that a real-provider benchmark
remains pending rather than inventing a result or blocking independent bug fixes.

Use Luna xhigh for benchmark/web volume work. Preserve all historical results and
frozen questions; write new manifests/results per patched revision. Do not tune
on the frozen evaluation set. Compare native products and normalized retrieval in
separate tracks. Distinguish hash embeddings from semantic embeddings, module
latency from authenticated SSE, structural citations from entailment, term hits
from correctness, and product no-answer flags from actual semantic refusals.

RAGFlow/Onyx had resource gates and LightRAG stopped during graph ingestion; none
has a valid score in that campaign. Request suitable resources/budget only when
needed for an authorized new benchmark. Never claim they lost to Ragz from those
incomplete runs. Cache goals apply to the specified boundary: zero provider calls
on eligible hits, byte-equivalent answers/citations, zero unauthorized/stale
responses, p50 <=5 ms and p95 <=10 ms; report misses/overload and tested concurrency.

## Execution and completion contract

For each confirmed bug: establish the failing invariant, implement the smallest
cohesive fix, show the regression fails before and passes after, check related
paths, and obtain independent review. Existing audit proof tests assert vulnerable
behavior in places: convert them to secure acceptance tests. Do not preserve a
passing vulnerability-demonstration test as proof the fix works.

Run relevant backend/frontend tests and static checks, migration/constraint/API
checks, focused harmless browser tests and dependency inventories. Testcontainers
must use fresh stores and a temporary KEK; mock paid providers for deterministic
faults. Verify `ragz.__file__` and affected module locations so an editable shared
virtualenv cannot silently test old code. Never claim the whole suite passed from
a targeted subset. At the end, run integration checks proportionate to the full
patch set and inspect actual CI status if remote checks are authorized/available.

Keep commits/patches coherent by concern and preserve report history. Do not push,
merge, deploy, publish GitHub comments/reviews or erase data without the user's
specific instruction. Local implementation and verification are authorized;
routine reversible choices do not need repeated permission questions.

Finish with the ledger: each issue fixed/already fixed/inapplicable/blocked,
patch location, before/after test evidence, reviewer disposition, migration or
compatibility impact, and outstanding benchmark/runtime limitations. Explain any
real blocker precisely, including an automated tool rejection if one occurred.
Do not state “all fixed” while confirmed in-scope bugs remain. The deliverable is
reviewable implemented patches with evidence, not a list of proposed fixes.

# PR #11 security and correctness review

PR: [Add bounded, superadmin-controlled multi-query retrieval](https://github.com/marketcalls/ragz/pull/11). Reviewed head: `3fac9fb1d02c9327f243418ebbb905466c8bcef5`. Base: `9d08839f6855967f3b141b731a269a54b76222fb`. Date: 2026-09-06.

**Recommendation: request changes.** The query-filter construction is consistent across lanes, but cancellation, failure handling, usage durability and grounding decisions require correction. The PR also introduces a significant default-model validation regression and contains test/evidence weaknesses. This review makes no GitHub changes.

Inherited security findings are detailed in [the application audit](2026-09-06-end-to-end-security-audit.md), particularly the document preview, attachment resource/lifecycle issues and broken dependency-audit target. They must not be mistaken for changes introduced by this PR.

## PR-001 — Single-flight owners can leave unresolved futures after cancellation

Severity: **Medium**, availability. Introduced by this PR.

Locations: [embeddings.py:239](https://github.com/marketcalls/ragz/blob/3fac9fb/backend/src/ragz/modules/retrieval/embeddings.py#L239), especially the cache-lock reacquisition/store block at lines 263-281; [query_expansion.py:214](https://github.com/marketcalls/ragz/blob/3fac9fb/backend/src/ragz/modules/retrieval/query_expansion.py#L214), through line 242.

The cache catches cancellation during provider computation and removes/resolves the owner's in-flight reservations there. That protection does not cover the subsequent `async with self._lock` that publishes the computed result. An owner cancelled while waiting for that lock leaves `_inflight[key]` behind with an unresolved future. Later same-key requests join that future and never receive a result; a restart clears the process-local state. Completed-entry LRU/TTL bounds do not repair a dead in-flight reservation.

Prerequisites: caching enabled, overlapping requests/cache-lock contention, and cancellation in this interval. Production Compose enables the caches. Whether a browser disconnect reaches this exact cancellation point depends on the request/server path; timeouts and direct task cancellation also matter. The consequence is demonstrated at the cache seam, not with a live remote denial-of-service attack.

Retained bounded reproduction deliberately pauses the owner after computation, holds the cache lock, cancels the owner, then checks a subsequent same-key call under a short timeout:

```text
embedding_dead_slot=True
expansion_dead_slot=True
```

Fix: put cleanup/publication inside a cancellation-safe lifecycle covering computation, lock reacquisition and storage; always resolve/cancel/remove the owner's slots, allowing waiters to retry ownership. Ensure successful results are not lost or double-attributed on cancellation. Test cancellation at every await boundary, not only inside the compute callback, for both cache implementations and multiple keys.

## PR-002 — Alternative embedding failure breaks Q1 fallback and loses paid usage

Severity: **Medium**, availability/accounting. Introduced by this PR.

Location: [retrieval/service.py:753](https://github.com/marketcalls/ragz/blob/3fac9fb/backend/src/ragz/modules/retrieval/service.py#L753), through 799.

Original-query embedding can succeed and expansion can return billable usage. Expansion usage is then staged with `commit=False`, followed by an unguarded await of alternative embeddings at lines 770-777. A failure there propagates instead of continuing with the already available Q1 embedding. It also precedes the original/alternative embedding-usage record at lines 793-799, while staged expansion usage can be rolled back with the request.

The fallback catches at lines 740-752 cover expansion timeout/provider failure, not failure embedding the generated alternatives. The PR's statement that optional expansion degrades to Q1 is therefore incomplete.

Fix: on ordinary alternative-provider failures, discard alternatives and continue with the original query/vector. Record completed billable actions durably before later fallible work. Preserve true cancellation semantics while ensuring incurred usage is finalized; do not blindly swallow `CancelledError` and continue abandoned work. Test original success → expanded query success → alternative failure, verifying Q1 results and one durable charge per completed provider call.

## PR-003 — Comparison source assembly precedes incurred-usage commit

Severity: **Medium**, accounting. Introduced by this PR.

Location: [evals/comparison.py:111](https://github.com/marketcalls/ragz/blob/3fac9fb/backend/src/ragz/modules/evals/comparison.py#L111), through 133; source authorization/metadata lookups at lines 63-68.

The comparison retriever can complete paid expansion/embedding/reranking and stage its ledger entries. `_sources()` then performs document authorization and metadata reads before `session.commit()`. A source lookup error, revocation-induced denial, or cancellation can discard incurred usage. This contradicts the PR description's stronger claim that usage is durable before source assembly.

Fix: finalize retrieval-provider accounting immediately after successful retrieval and before `_sources()`, with explicit cancellation-safe/idempotent handling. Also cover failures inside retrieval itself (PR-002); moving the caller commit cannot record usage that never reaches the caller. A fake paid retriever plus failing source resolver should leave exactly the expected durable ledger entries.

## PR-004 — Pre-recheck probe scores can survive removal of their document

Severity: **Medium**, grounding integrity; **not a demonstrated content leak**.

Locations: overlapping dense probes [retrieval/service.py:844](https://github.com/marketcalls/ragz/blob/3fac9fb/backend/src/ragz/modules/retrieval/service.py#L844); projection recheck at 898-908; final no-answer decision at 990-1002.

The no-answer probes run against the earlier authorization/projection filter. The later recheck removes newly unprojected documents from the fused candidates, but their probe scores still contribute to `best_cosine`. If a removed document supplies the high score and at least one weak allowed candidate remains, generation can proceed over the weak candidates instead of returning no-answer.

The probe requests `with_payload=False` and removed candidate text does not reach this return path; do not report this as proof of cross-tenant document disclosure. The problem is that a document no longer in the authorized candidate set influences the authorization-sensitive grounding verdict. The PR adds overlapping probes and preserves their results across this recheck; a robust fix should cover stale-score/filter behavior in all paths.

Fix: compute the verdict against the same post-recheck eligible candidate/revision set used for generation. Re-running a probe is useful only if its filter is updated to exclude the removed set. Test a high-score document becoming unprojected while a low-score allowed candidate remains: no forbidden content returned and no-answer remains true when allowed evidence is below threshold.

## Other security-relevant observations

| Observation | Assessment and action |
|---|---|
| Global query-derived caches | Embedding namespace (`retrieval/service.py:685-690`) and expansion key (`query_expansion.py:113-116`) omit org/workspace. Same-process users sharing a model can observe cache-hit/coalescing timing for guessed queries and share attribution of provider work. Low privacy/fairness risk, not document-content crossover. Add tenant namespace or explicitly document acceptance of global deduplication. |
| Comparison forces MQR | `api/routes/evals.py:87-130` permits `evals.run`; comparison's second arm sets `multi_query_enabled_override=True` (`comparison.py:193-217`) even if the workspace toggle is off. This may be intentional evaluation behavior; clarify its cost/control permission. It does not prove bypass of the superadmin-only persistent-setting mutation. Consider a separate compare/MQR capability if the toggle is a strict spend-control boundary. |
| Long expansion input | `query_expansion.py:275-277` uses the whole question. The 2,000-character bound at 322 limits generated alternatives, not the input; chat permits 32,000 characters. Low/input-efficiency observation: this is within the existing chat contract, not a demonstrated unbounded-cost vulnerability. Bound expansion prompt input deliberately. |
| Agent streaming usage | The cancellation gap described in SEC-012 predates the PR. Adding a commit after final aggregation narrows later gaps but does not make earlier paid tool frames durable. Correct the implementation and the PR claim; do not describe it as newly introduced. |
| HTTP-date Retry-After | `rerank.py:157-166` parses a float only, although Retry-After can be a date. Low reliability/provider-pressure issue; honor both forms before the cap. |

## Existing automated review: all 15 comments triaged

The review posted for this exact head was read as untrusted review input and checked against source. This table groups no independent scanner output into a false count of security vulnerabilities.

| # | Existing review topic | Disposition at reviewed head |
|---|---|---|
| 1 | MQR isolation test can pass without expansion | **Confirmed test weakness.** `tests/isolation/test_multi_query_isolation.py:8-37` has no expander invocation assertion and no positive caller-corpus result. The empty result does not prove the alternative lane ran. Add invocation/positive-control checks. |
| 2 | Default embedding model breaks completions | **Confirmed functional regression.** `tenancy/service.py:197-204` validates only model existence when assigning a default; `models/service.py:44-53,71-75` now rejects non-chat/disabled defaults. Reject invalid assignments and define treatment for existing invalid defaults. This can break ordinary chat for affected workspaces. |
| 3 | Streaming cancellation loses paid web usage | **Confirmed, pre-existing.** See SEC-012; the new commit does not cover earlier frames. |
| 4 | Alternative embedding failure should fall back | **Confirmed.** PR-002. |
| 5 | No-answer probe retains removed ACL candidate's score | **Confirmed control-flow defect.** PR-004; no content-disclosure claim. |
| 6 | E2E `isVisible()` race | **Confirmed timing weakness.** `frontend/e2e/mqr-settings.spec.ts:16-23` checks immediate visibility before async workspace options necessarily exist and may create duplicates. Wait for loading/lookup completion before deciding to create. |
| 7 | ADR says one dense embedding batch | **Confirmed documentation mismatch.** `docs/adr/ADR-0007-multi-query-retrieval.md:31-32`; implementation starts Q1 separately (`retrieval/service.py:716-724`) and embeds alternatives later (`:769-777`). |
| 8 | ADR says query count capped at three | **Confirmed documentation mismatch.** ADR lines 45-46 versus supported expansion counts `{3,5}` (`query_expansion.py:27-29`) and retrieval override `{1,3,5}` (`service.py:637-646`). Distinguish production default from supported internal/evaluation modes. |
| 9 | Client JWT versus server authorization gating | **Confirmed conditional UX inconsistency, not auth bypass.** Settings dialog uses decoded token role at line 35 while eval controls use server-derived authorization at lines 36-45. A role change can make controls stale; backend checks remain enforced. Server-derived data itself has a 60-second stale time. |
| 10 | Cancelled cache store strands future | **Reproduced in both caches.** PR-001. |
| 11 | HTTP-date Retry-After ignored | **Confirmed.** Reliability observation above. |
| 12 | Comparison source lookup before usage commit | **Confirmed.** PR-003. |
| 13 | Full 32k input used by expansion | **Confirmed input behavior; severity downgraded.** Existing schema permits it; no disproportionate-cost exploit demonstrated. |
| 14 | Evals uses unavailable workspace default | **Confirmed.** `evals-section.tsx:66-70` uses `modelId ?? defaultModelId` without checking `models.data`; selector lists only returned models at 131-145, request still sends the invalid effective ID at 155-165. Choose only an available chat model and handle an empty list. |
| 15 | Completed comparison's Fixed input changes with form | **Confirmed result-integrity/UI bug.** Submission uses live states at `evals-section.tsx:159-165`; result labels reuse those states at 192-210. Save the submitted question/model snapshot with the result. |

## Security properties that survived review

- Every dense and sparse prefetch lane uses the same `flt`; the outer fusion also receives it (`retrieval/service.py:811-875`). Tenant/workspace/group/current-version/unprojected filtering is not omitted on alternatives.
- Comparison checks workspace access and calls `get_document_checked` for source metadata. No ordinary cross-workspace or ACL source-disclosure path was established there.
- Persistent MQR setting mutation is enforced server-side as superadmin-only, in both route and service. UI hiding is not treated as the authorization boundary.
- Cache values contain query-derived embeddings/alternatives, not document answers or chunks. These caches are distinct from the other thread's unfinished response cache.
- Keys hash query content; inspected metrics are low-cardinality and do not contain raw query/alternative text.
- Expansion count, output and deadlines, cache completed-entry counts/TTLs, and retry counts have bounds. In-flight cancellation/cleanup is the defect, not an absence of all limits.

## Verification, CI, and merge conditions

The parent audit ran **223 focused backend tests** against explicit PR source paths with fresh Testcontainers services. This includes the current MQR isolation test, whose passing result has the coverage limitation above. Two separate auth evidence tests passed. Cache cancellation was independently reproduced with deterministic async scheduling. Patch whitespace and backend compilation checks passed.

This is not a claim that the PR's full `1762 passed`/`709 passed` assertions were rerun. The shared Python 3.13 environment was used with a temporary missing-dependency overlay; its editable local package path was explicitly overridden. Frontend rendering inspection/audits were performed, but the complete frontend/E2E suite was not rerun. No paid providers or production customer data were used.

GitHub's [CI](https://github.com/marketcalls/ragz/actions/runs/33836126170) and [dependency audit](https://github.com/marketcalls/ragz/actions/runs/33836126191) both showed `action_required` for the reviewed head. The reviewer-bot success is not a CI pass. Maintainer action may be needed to run those workflows; no approval, rerun, branch-protection or review setting was changed by this audit.

Before merge:

1. Fix the four numbered PR findings and invalid-model-default path; add targeted regressions that fail on current code.
2. Make the comparison UI reflect the actual submitted input/model and reconcile authorization/cost expectations for forced MQR.
3. Strengthen the isolation test so bypassing expansion makes it fail; keep positive same-tenant retrieval controls.
4. Fix the inherited dependency-gate target and update known vulnerable dependencies; obtain actual CI success at the final head.
5. Update ADR/PR statements to match actual batching/count limits and proven accounting semantics.
6. Rereview changed security/lifecycle paths and run safe browser document-preview coverage before releasing the application, regardless of whether MQR is enabled.

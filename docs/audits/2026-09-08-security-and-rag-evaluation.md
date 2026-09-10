# Ragz: security and open-source RAG evaluation

Date: 2026-09-08. Evidence review, not a new performance campaign.

## Assessment

**Ragz is a capable document-RAG engine with promising measured performance. Its current security and operational assurance do not justify calling it a proven top-tier production RAG platform.**

| Dimension | Rating | Reason |
|---|---:|---|
| Current security for public, untrusted multi-tenant deployment | **4/10** | Strong authentication/ACL foundations, but unresolved active-content and shared-resource risks, plus authorization, lifecycle and audit gaps |
| Security architecture/control foundation | **7/10** | Explicit tenancy, query-time ACLs, encrypted secrets, live credential checks and isolation tests; resource/lifecycle/egress consistency remains incomplete |
| RAG engine capability on the tested document workloads | **7/10** | Hybrid retrieval, real page provenance, version awareness, provider flexibility and completed ingestion/chat runs; quality varies by corpus/configuration |
| Readiness to claim top-tier production-platform status | **5/10** | Security blockers, incomplete multi-system evidence, weak abstention measurement, limited scale/reliability validation and unfinished response-cache integration |

These are my engineering judgments, not certified scores, measured probabilities or a conversion of a benchmark pass percentage. Scale: 1–3 is substantially unsafe/incomplete; 4–5 has meaningful capability but unresolved release/assurance gaps; 6–7 is capable with material limitations; 8–9 requires broad, repeatable production evidence; 10 would require exceptional independently validated performance and assurance. Known High findings cap public-deployment security at 4 until resolved and retested. Architecture, capability and readiness are separate assessments, not components averaged into an unexplained overall score.

The security score applies to the audit's threat model, not every private single-user installation. Competitors have not received equivalent security audits here, so no numerical security leaderboard is defensible.

## Evidence and revision check

- On September 8, GitHub main remained `9d08839f6855967f3b141b731a269a54b76222fb`.
- PR #11 remained open and unmerged at `3fac9fb1d02c9327f243418ebbb905466c8bcef5`, the audited head.
- The shared local checkout remains old main `90bb3b1` with uncommitted CAG/parser/upload/UI changes. The fastest recent anatomy benchmark used that dirty local build; it is not a performance test of the audited PR head.
- Security basis: [application audit](2026-09-06-end-to-end-security-audit.md), [PR review](2026-09-06-pr-11-security-review.md), [retained evidence](2026-09-06-security-evidence/README.md).
- Performance basis: [September 6 anatomy E2E report](../../no_rel/benchmarks/results/FULL_E2E_TOP_TIER_RAG_COMPARISON_2026-09-06.md), [production embedding/rerank report](../../no_rel/benchmarks/results/PRODUCTION_COHERE_RERANK_REPORT_2026-08-21.md), [MQR/caching report](../../no_rel/benchmarks/results/RAGZ_MQR_RERANK_CACHE_REPORT_2026-08-24.md), associated raw artifacts and scoring code.
- Four Luna subagents at **xhigh** handled the larger security/benchmark evidence reviews and the remaining official-source competitor research. The primary agent calibrated the ratings and conclusions. No new paid-provider benchmarks or application changes were made.

## Security report summary

The common application findings total **12: two High, nine Medium and one Low**. The old local checkout has an additional High ACL-projection finding that newer main remedied. PR #11 has four additional Medium findings. Those counts describe different snapshots and should not be combined as though all findings apply identically everywhere.

The most consequential findings are:

1. **Uploaded-file preview can execute HTML in a viewer's origin.** The uploader controls MIME and the UI opens a blob in an unsandboxed iframe. This is the principal browser/session-compromise risk. The source-to-sink path is established; browser credential exploitation was not performed.
2. **Attachment ingestion can overconsume shared resources.** Per-file limits do not bound tenant upload frequency, aggregate bytes, pending jobs or parser work. Chat deletion removes the attachment rows required for external cleanup, leaving blobs/vectors behind.
3. **Org administrators can self-grant supposedly independent audit/content permissions.** A synthetic PostgreSQL regression demonstrates the grant becoming effective.
4. **Several boundaries are inconsistent.** Web/media egress misses shared address space; passive authentication failure retains browser query cache; quota checks race; credential/audit and provider/usage transactions can diverge.
5. **Dependency assurance is weaker than its CI label suggests.** The Python CI command audits its own tool environment. Explicit lockfile scanning found a known Transformers advisory, while frontend build/development dependencies are excluded by the production-only gate. Dependency presence is not proof of a reachable application exploit.

PR-specific defects: cache-owner cancellation can strand futures and hang future same-key requests; alternative embeddings can fail without falling back to Q1 and lose usage; comparison source lookup precedes the usage commit; and a removed ACL candidate's score can still influence the no-answer decision. The last issue is grounding integrity, not demonstrated forbidden-content disclosure. Invalid default-model assignment is a separate functional regression. All 15 existing automated-review comments were assessed in the PR report.

What deserves credit: Argon2id, short-lived signed access tokens, live user/security-version checks, rotating refresh tokens, OIDC PKCE/state/nonce, scoped expiring API keys, encrypted provider secrets, declarative authorization, per-lane vector ACL filters and substantial isolation tests.

The audit retained 223 focused backend test results plus two distinct auth evidence tests. Some evidence tests intentionally pass when the defect exists; “225 tests passed” is not a security certification. Production edge/bot configuration, full parser fuzzing, actual browser exploitation and certain concurrency transitions remain outside verified runtime coverage.

## What the latest native benchmark establishes

Corpus: one 476 MB / approximately 454 MiB, 1,347-page anatomy PDF. Fifty fixed questions: 40 answerable and 10 unsupported. Completed scored systems: Ragz and AnythingLLM.

| Observable metric | Ragz | AnythingLLM | Interpretation |
|---|---:|---:|---|
| Upload-to-indexed time | 21.594 s | 294.961 s | Strong Ragz result in this native configuration; not an equal-embedding comparison |
| Retrieval p50 / p95 | 20.379 / 25.118 ms | 89.445 / 97.194 ms | Ragz faster in the measured configurations |
| Answer completion p50 | 1.744 s | 1.738 s | Effectively tied at this sample size |
| Answer completion p95 | 7.928 s | 3.174 s | AnythingLLM has the better observed tail |
| Deterministic answer pass | 35/50 (70%) | 32/50 (64%) | Keyword-based rubric, not verified semantic correctness; small difference |
| Exact evidence-page retrieval hit@5 | 35/40 (87.5%) | Not available | Useful Ragz page-level result; missing comparator metadata is not a zero score |
| Completed trials / errors | 50 / 0 | 50 / 0 | Useful smoke/reliability evidence, insufficient for production error-rate guarantees |

Important qualifications:

- Ragz used deterministic local hash embeddings plus BM25 for the fast September 6 ingestion/retrieval cell; AnythingLLM used MiniLM/LanceDB. The 13.66× ingestion ratio and 4.39× retrieval median ratio do **not** establish equivalent semantic quality or production embedding performance.
- A corpus containing one document makes document Recall/MRR/nDCG nearly trivial when that document is returned. Page/passage-level metrics are the useful discriminator.
- The answer scorer checks whether at least half the acceptable terms appear for answerable questions. Unsupported-question passes accept a product no-answer flag or an acceptable term. It does not validate every claim, negation, factual contradiction or citation entailment (`run_anythingllm_large_pdf_e2e.py:183-194`; `run_ragz_large_pdf_e2e.py:545-555`).
- Paired artifact reanalysis found 32 questions both passed, three Ragz-only passes, zero AnythingLLM-only passes, and 15 neither passed (including the 10 unsupported questions). On answerable questions alone this is 35/40 versus 32/40. Exact two-sided McNemar p = **0.25** for the three discordant pairs: the observed advantage is not statistically significant. This is an analysis of these fixed term-scoring outcomes, not an independent semantic-quality experiment.
- The reported zero unsupported-question abstention signal is not a validated “100% hallucination” rate. Ragz uses its final `no_answer` flag; the AnythingLLM adapter infers no-answer from absence of sources (`run_anythingllm_native_resume.py:206`). A semantic refusal can still include sources. Thresholds were zero in this recall-stress configuration; this does not measure the quality of calibrated production refusal.
- Seventy-two structurally valid Ragz citation links are not seventy-two independently verified entailed claims. Exact gold-page citation hit was 70% of answerable questions, a separate measure.
- A single ingestion observation and one answer per question yield descriptive timing, especially p95; they do not establish stable cold-start or tail-latency guarantees.
- Ragz host resources and AnythingLLM container resource limits differ, and the Ragz worker peak was not captured in the E2E manifest. No like-for-like RAM-efficiency or fixed-compute winner is established.
- The initial Ragz chat attempt had failures before a planner-option fix; the 50/50 result is a corrected run. This demonstrates repair and a successful final configuration, not failure-free execution of the whole development campaign.

## Production embeddings, reranking and MQR

The earlier 22-manual production-embedding experiment is more relevant to hosted semantic retrieval: OpenAI embeddings achieved Recall@5 1.0 and MRR@5 0.9583 across 60 positive questions. Cohere increased MRR to 0.9833 and abstention F1 from 0.3333 to 0.75, while median retrieval increased from 378 ms to 1,169 ms. These are retrieval measurements, not full generated-answer latency. [Source](../../no_rel/benchmarks/results/PRODUCTION_COHERE_RERANK_REPORT_2026-08-21.md).

Results do not generalize uniformly. On the three-book, 24-question exact-page experiment, Q1/no rerank achieved Recall@5 0.60; Q3 did not improve recall, and several reranked configurations regressed. This is a different corpus, label granularity and configuration—not a contradiction of the manual-level recall result. MQR and reranking should remain workload-tested options, not assumptions of superior quality. [Source](../../no_rel/benchmarks/results/RAGZ_MQR_RERANK_CACHE_REPORT_2026-08-24.md).

The response cache's 0.213 ms p95 is a Redis/module lookup over 500,000 operations, not authenticated chat latency. Its caller-supplied epoch checks and simulated stampede tests do not establish production ACL-revocation correctness. The answer-cache path is still not integrated with ordinary chat. Its high operation count should not be confused with hundreds of thousands of independent RAG quality questions.

## Position relative to representative open-source alternatives

| System | Documented emphasis | What can be said about Ragz today |
|---|---|---|
| AnythingLLM | Local-first end-user app, documents, citations, agents; Docker supports multi-user permissioning | A credible direct comparator. Ragz won the tested local ingestion/retrieval cells; answer median was tied and AnythingLLM's tail was better. The small deterministic quality difference does not establish general superiority. |
| RAGFlow | Complex document understanding, configurable chunking, traceable citations, heterogeneous formats and ingestion/agent workflows | A broader documented document-workflow target. Ragz has useful page-level evidence, but no completed equivalent RAGFlow quality/latency run here; do not claim Ragz beats it. |
| Onyx | Broad workplace connectors, recurring sync, search and enterprise governance | Ragz has no equivalently demonstrated broad connector/freshness ecosystem. Source-permission sync and some advanced governance are Enterprise features; do not count those as free Community capabilities. No completed Standard benchmark here. |
| LightRAG | Knowledge-graph plus vector retrieval, local/global/hybrid modes, incremental graph updates and cached extraction | A different retrieval strategy particularly relevant to relationship/global questions. Graph ingestion exceeded the local development budget, so its answer quality remains unscored. That does not establish worse quality or generic inefficiency. |

Official feature sources checked September 8: [AnythingLLM README](https://github.com/Mintplex-Labs/anything-llm/blob/master/README.md), [RAGFlow README](https://github.com/infiniflow/ragflow), [Onyx README](https://github.com/onyx-dot-app/onyx), [Onyx connectors and edition boundary](https://docs.onyx.app/admins/connectors/overview), [LightRAG repository](https://github.com/HKUDS/LightRAG), [LightRAG research paper](https://arxiv.org/abs/2410.05779). These are vendor/author capability descriptions, not independent performance or security certifications. Mutable current documentation may include features beyond the benchmark's pinned versions.

RAGFlow and Onyx Standard were resource-gated on the test host. LightRAG was stopped during graph indexing under a development spend limit. Assigning them zero quality or placing them below Ragz would be invalid. Likewise, not verifying a competitor's page-level citation precision does not prove it lacks that feature. This is a representative comparison, not an exhaustive survey of every open-source RAG application or framework.

## What would justify an 8/10 or top-tier claim

1. Resolve the High security findings and relevant Medium boundary failures; independently retest actual browser previews, tenant/role transitions, quotas, cancellation and deletion.
2. Integrate fixes on one identified release revision. Benchmark that revision with reproducible deployment configuration; do not assemble a hypothetical product from the best results of incompatible dirty snapshots.
3. Evaluate multiple domains and formats: ordinary text, OCR, tables, conflicting/versioned documents, multilingual content and cross-document questions. Use a larger held-out question set with meaningful unsupported/adversarial cases, human-calibrated correctness and citation-entailment scoring.
4. Run both native-product and normalized retrieval comparisons, with documented model/chunker/top-k/prompt/compute differences, multiple repetitions, randomized order and uncertainty estimates. Give each system enough resources to reach a valid terminal state.
5. Measure authenticated E2E chat under sustained mixed ingestion/query load: throughput, p95/p99, error rate, memory, queue depth and cost. Redis concurrent-client numbers are not application-user capacity.
6. Complete CAG admission, live authorization, authoritative revision updates and fill fencing; then demonstrate cold/warm chat equivalence, zero provider calls on eligible hits and no stale/unauthorized replay.
7. Demonstrate recovery from provider/store outages and worker retries, auditable usage/deletion, and—if targeting workplace search—connector freshness and permission propagation.

Practical positioning now: **a promising, policy-aware document-RAG engine suitable for controlled evaluation, with unresolved release blockers and insufficient evidence for an across-the-board top-tier claim.**

## Benchmark handoff incorporated

The user subsequently supplied `/tmp/ragz-cag-benchmark-handoff-2026-09-08.md`.
A preserved repository copy is available at
[docs/handoffs/2026-09-08-cag-benchmark.md](../handoffs/2026-09-08-cag-benchmark.md).
It confirms that CAG is paused, authenticated cached SSE remains unmeasured,
large-PDF improvements are uncommitted, and campaign databases/results must be
preserved. It introduces no new completed cross-system result that changes the
ratings above. Its 56 targeted checks are a separate campaign result, not an
additional full-suite security validation or a number to pool with this audit.

Future analysis/benchmark execution must recheck historical runtime state using
Docker's `default` context, preserve all benchmark work and failed attempts,
reconcile the development cost ledger and uncertain external rerank charges,
and obtain fresh direction/budget before further paid calls. The
[Sol remediation prompt](../prompts/sol-security-remediation.md) and root
`AGENTS.md` incorporate these constraints and the two-Sol/Luna-xhigh policy.

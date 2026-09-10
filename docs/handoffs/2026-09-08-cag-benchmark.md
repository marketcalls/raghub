# RAGZ CAG and top-tier benchmark handoff

Prepared: 2026-09-08  
Workspace: `/home/parshu/projects/ragz`  
Repository HEAD: `90bb3b147b5da88ca704663cce09f77eab7629b7`  
Active goal state: **paused, not complete**

## Mission

Continue building a production-grade local Redis-backed RAG response cache
(the user's meaning of CAG), without retaining the corpus in the model context.
It must support high user concurrency, automatic hot/warm/cool behavior,
document-update invalidation, tenant/ACL isolation, citation freshness, and the
large anatomy PDF. Production has no fixed USD cap; the approximately `$3`
limit is development-only.

## Read these artifacts first

Do not reconstruct or duplicate their detailed tables:

1. Consolidated E2E report:
   `/home/parshu/projects/ragz/no_rel/benchmarks/results/FULL_E2E_TOP_TIER_RAG_COMPARISON_2026-09-06.md`
2. Redis/CAG infrastructure report:
   `/home/parshu/projects/ragz/no_rel/benchmarks/results/REDIS_CAG_INFRA_REPORT_2026-09-06.md`
3. Cache benchmark protocol:
   `/home/parshu/projects/ragz/no_rel/benchmarks/methodology/CAG_CACHE_PROTOCOL.md`
4. Frozen 454 MiB dataset:
   `/home/parshu/projects/ragz/no_rel/benchmarks/datasets/anatomy-physiology-v1/`
5. Campaign cost ledger:
   `/home/parshu/projects/ragz/no_rel/benchmarks/results/runs/anatomy-top-tier-campaign-20260906-cost.json`
6. Architecture-review visualization:
   `/tmp/architecture-review-20260904-173014.html`
7. Latest independent security/RAG audit already present in the worktree:
   `/home/parshu/projects/ragz/docs/audits/2026-09-08-security-and-rag-evaluation.md`
8. Threat model already present:
   `/home/parshu/projects/ragz/docs/audits/ragz-threat-model.md`

## Important repository condition

The branch is `main`, **behind `origin/main` by 63 commits**, with many tracked
modifications and untracked files. Some changes predate this work and belong to
the user. Do not reset, checkout, clean, rebase, or bulk-overwrite the tree.
Inspect `git status` and overlapping diffs before editing.

Relevant new/modified implementation:

- `backend/src/ragz/modules/cache/response.py`
- `backend/src/ragz/api/app.py`
- `backend/src/ragz/core/config.py`
- `deploy/compose.yaml`
- `backend/src/ragz/modules/documents/uploads.py`
- `backend/src/ragz/core/storage.py`
- `backend/src/ragz/api/routes/documents.py`
- `backend/src/ragz/modules/documents/service.py`
- `backend/src/ragz/modules/documents/ingest.py`
- `backend/src/ragz/modules/documents/parsers.py`
- `backend/src/ragz/modules/chat/llm.py`
- associated tests under `backend/tests/`
- benchmark scripts/results under `no_rel/benchmarks/`

`no_rel/` and `docs/audits/` currently appear untracked at the repository root;
treat them as valuable user artifacts, not disposable scratch data.

## Implemented functionality

### Large-document ingestion

- Default upload safety boundary raised from 100 MiB to 1 GiB.
- HTTP upload is measured and hashed in bounded chunks rather than accumulated
  in RAM.
- Source upload/download uses file streams.
- The worker downloads to a temporary file and gives LiteParse a file path.
- LiteParse parses bounded page ranges, validates every returned page, and
  fails explicitly above the configured page limit.
- The unset parser default now actually selects LiteParse, matching the admin
  settings surface. Previously it silently selected anydoc and flattened the
  1,347-page PDF to page 1.
- The target PDF completed 1,347/1,347 pages and 5,027 chunks in the accepted
  RAGZ E2E run.

### Redis response-cache module

- Deep `ResponseCache` protocol plus `RedisResponseCache` implementation.
- Privacy-safe normalized query hashing; raw queries are absent from keys.
- Per-user/principal scope hashing; no cross-user sharing in the safe baseline.
- Corpus/security/prompt/cache epoch binding in identities and records.
- Complete answer, sources, citations, and source-manifest serialization.
- Atomic Redis Lua lookup with epoch comparison, HWC promotion, and TTL refresh.
- Cool/warm/hot defaults: 600 s / 3,600 s / 21,600 s; promotion on hits 2/3.
- Bounded record size, corrupt-record rejection, Redis fail-open behavior.
- Owner-token single-flight locks and wait-for-fill support.
- Application creates a separate cache client when enabled.
- `RAGZ_RESPONSE_CACHE_ENABLED` is deliberately **false by default**.

### Docker cache service

- Dedicated Compose `response-cache` service, Redis 8.8.0 pinned by digest.
- No AOF/RDB persistence; 256 MiB Redis maxmemory; `allkeys-lfu`.
- Separate from operational Celery/rate-limit Redis.
- Healthy at `127.0.0.1:56479`; DB 0 was empty at handoff.

### Luna compatibility

- GPT-5.6 function-tool planner calls now explicitly use
  `reasoning_effort="none"`; final answer calls remain separate.
- Regression test added in `backend/tests/modules/chat/test_llm.py`.

## Verified benchmark findings

See the consolidated report for definitions and caveats. Essential current
state:

- RAGZ completed original-PDF upload through native SSE chat.
  - ingestion: 21.594 s, 1,347/1,347 pages;
  - no-rerank retrieval p50/p95: 20.379/25.118 ms;
  - exact evidence-page hit@5: 87.5%;
  - corrected chat: 50/50, zero errors, 70% failure-inclusive pass;
  - answer p50/p95: 1.744/7.928 s;
  - TTFT p50/p95: 1.235/7.461 s;
  - exact evidence-page citation rate: 70%; citation structure validity: 100%;
  - unsupported-query abstention: 0% (known weakness).
- AnythingLLM v1.16.1 completed a native track after preserving failed setup
  attempts.
  - ingestion: 294.961 s, peak about 1.496 GB;
  - retrieval p50/p95: 89.445/97.194 ms;
  - native chat 50/50, zero errors, 64% pass;
  - answer p50/p95: 1.738/3.174 s;
  - native API has no physical-page attribution or TTFT.
- LightRAG v1.5.7 did not finish graph ingestion before the development gate.
  - stopped after >3,099.6 s while still `processing`;
  - 825 chunks, 861 Luna calls, 3.639M prompt and 1.610M completion tokens;
  - spend `$2.40330511`, peak about 1.279 GB;
  - no query/cache quality or latency score is valid.
- RAGFlow v0.27.1 and Onyx Standard v4.7.0 have resource-gate artifacts and no
  synthetic scores. See the report and
  `no_rel/benchmarks/results/runs/top-tier-resource-gates-20260906-2aff7cb4/`.

### Cache measurements

- Full implemented Redis cache lookup, 500,000 operations over five windows:
  p50/p95/p99 = 0.129/0.213/0.305 ms.
- 50 clients: p50/p95 = 2.144/2.437 ms, about 22.6k ops/s.
- 200 clients: p95 = 12.169 ms; the <=10 ms gate fails at this concurrency on
  one Redis process.
- Stampede tests at 50/200/1,000 simultaneous misses produced one simulated
  provider call.
- 10,000 epoch checks served zero stale records.
- HWC admission simulations used 200 paired repetitions per workload, 60M
  requests total, and 20,000 paired-bootstrap resamples. See
  `no_rel/benchmarks/results/CAG_CACHE_POLICY_RESULTS_2026-09-04.md`.

These are module/infrastructure results. **Cached authenticated SSE chat has not
been measured**, because the cache is not yet safely wired into `stream_reply`.

## Development API budget

Do not make more paid calls without explicit user direction or a fresh budget.

- Exact LiteLLM/OpenAI aggregate since campaign start: `$2.60807538`.
- Prompt tokens: 6,033,316; completion tokens: 1,672,536; logged calls: 1,518.
- One failed AnythingLLM external Cohere attempt bypassed LiteLLM. At most seven
  search units may be involved; exact production cost requires the user's
  Cohere dashboard. Do not assume it was free.
- Local parsing, local embedding, Qdrant, Redis, and Docker have zero API cost.
- Never print or persist `.env`, API keys, auth headers, model secrets, user
  passwords, queries, answer text, or reference content.

## Docker/runtime state

Use `docker --context default` explicitly. Two Docker daemons exist and the
shell's selected `desktop-linux` context caused an invalid monitoring attempt.

Healthy services at handoff:

- `ragz-response-cache-1` on port 56479;
- `ragz-e2e-postgres-1` on port 55433, preserving the isolated RAGZ campaign DB;
- shared LiteLLM/PostgreSQL/Redis/Qdrant/MinIO containers under
  `ragz-mqr-local-test-*`.

No temporary AnythingLLM or LightRAG benchmark container remains running. No
RAGZ API or Celery worker process remains running. The isolated Postgres volume
and benchmark artifacts are intentionally preserved.

## Failed/invalid attempts that must remain visible

- RAGZ page-1 parser-default attempt:
  `.../ragz-large-pdf-e2e-v1.0-20260906T115532Z-633b988a/`.
- RAGZ initial chat run had seven Luna tool/reasoning HTTP 400s; corrected chat
  is the separate `...120824Z-3d72c909` run.
- AnythingLLM attempts exposed Docker-context networking and Luna temperature
  compatibility. The accepted result is `...124244Z-223677f5`.
- LightRAG 413 attempt is `...125646Z-7bded838`.
- LightRAG empty-corpus run `...125806Z-318b7817` has `invalidated.json`; its
  apparent warm-cache latency is invalid.
- The accepted LightRAG outcome is the budget-gated failure
  `...130308Z-b0442fff`.

Never delete, overwrite, or silently average these into successful results.

## Next work, in order

1. Design and migrate an authoritative PostgreSQL workspace cache-revision
   record (corpus, security, prompt, cache epochs). Do not rely on Redis
   invalidation messages for authorization.
2. Bump the correct epoch in the same database transaction as every relevant
   document/index, ACL/group/membership, workspace prompt/retrieval, model, and
   explicit-flush mutation.
3. Add one cacheable-answer seam around the ordinary document-grounded branch
   of `backend/src/ragz/modules/chat/service.py`; conversational, web/agent,
   attachment/image, general-knowledge, partial/aborted, and unsafe responses
   remain ineligible initially.
4. On a hit, rebuild current `TenantContext`, compare authoritative epochs,
   batch-load every source document, validate workspace/current version/status,
   enforce current ACLs, and reject missing/stale sources.
5. A valid hit must still persist a new assistant message and emit the normal
   `sources -> token -> citations -> done` SSE contract. Report zero provider
   tokens for the hit.
6. Integrate single-flight with fencing/recheck-before-store so an old miss
   cannot overwrite a result created after a document/ACL update.
7. Add isolation/race tests: 10,000 update/read races, membership revocation,
   ACL group changes, document replacement/deletion, Redis outage, corrupt
   records, and 50/200/1,000 identical misses.
8. Only then enable the cache in the isolated benchmark runtime and repeat the
   frozen 50-question cold/exact-warm SSE campaign. Acceptance: zero provider
   calls on hits, byte-equivalent answers/citations, zero stale/ACL leaks,
   p50 <=5 ms and p95 <=10 ms.
9. Address known quality issues separately: keep the lexical reranker disabled
   on this corpus and calibrate abstention on a held-out split. Do not tune on
   the frozen evaluation questions.
10. For RAGFlow/Onyx numerical runs, use a larger clean host rather than
    overcommitting this workstation. LightRAG needs a fresh explicitly approved
    provider budget or a defensible local-model protocol.

## Last verification evidence

The final targeted verification on 2026-09-06 completed with:

- 56 tests passed;
- Ruff checks passed for touched benchmark/product files (legacy benchmark
  scripts required their documented E501/subprocess ignores);
- strict mypy passed for parser, LLM, and cache product modules;
- six canonical artifacts passed structural assertions;
- `git diff --check` passed;
- dedicated response-cache Redis was healthy.

Re-run relevant tests before new claims. Do not claim the whole repository suite
passes based only on the targeted 56-test command; the worktree contains other
unrelated changes.

## Suggested skills

- `improve-codebase-architecture` — preserve the deep cache seam and keep
  freshness/security logic local rather than adding branches throughout chat.
- `security-threat-model` — refresh the existing cache/tenant threat model once
  the epoch schema and hit-validation flow are concrete.
- `security-best-practices` — review the Python/FastAPI/Redis implementation
  before enabling it.
- `requesting-code-review` — review the complete cache wiring and migrations.
- `verification-before-completion` — use before claiming cached SSE E2E or
  marking the goal complete.

The user previously asked not to use Superpowers-branded skills; respect that
preference unless the user explicitly changes it.

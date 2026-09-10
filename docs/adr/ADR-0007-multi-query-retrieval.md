# ADR-0007: Bounded Multi-Query Retrieval with Rank Fusion

**Status:** Accepted
**Date:** 2026-08-21

## Context

RAGZ retrieves with one user query: one dense embedding and one sparse vector
are fused by Qdrant RRF, optionally followed by cross-encoder reranking. The
agent may perform several sequential searches, and ingestion enrichment creates
hypothetical questions for document chunks, but neither is query-time expansion
of a single retrieval request.

One phrasing can miss relevant chunks when the corpus uses different terminology
or when a question combines perspectives. Query expansion can improve recall,
but unconstrained fan-out multiplies provider cost and latency, and independently
implemented searches risk drifting from the single tenant/ACL-filter path.

## Decision

Add a default-off `multi_query_enabled` workspace setting.

When enabled:

1. The designated utility chat model receives the user query as an explicitly
   delimited data block and returns at most two alternative search queries.
2. The exact original query is always retained as the first lane. Blank,
   overlong, duplicate, or non-string alternatives are discarded.
3. Missing utility configuration, provider failure, or malformed output degrades
   to the original query. Retrieval does not fail solely because expansion fails.
4. The original query (Q1) is embedded first while expansion is in flight. Once
   alternatives arrive, their dense inputs are embedded with the provider's
   bounded batcher; one sparse representation is produced per variant.
5. Every dense and sparse lane uses the same `_tenant_filter`, including current
   document, metadata, ACL-group, and unprojected-security exclusions.
6. Qdrant RRF fuses all ranked lanes. Raw dense, sparse, and cross-query scores
   are never combined linearly.
7. Existing chunk/HQ deduplication runs once after fusion. If reranking is
   enabled, it runs once against the original user query. If reranking is off,
   no-answer uses the maximum top dense cosine from the valid variants.
8. Expansion tokens are recorded in the usage ledger under
   `feature="query_expansion"`; embedding and rerank accounting remain once per
   retrieval call.
9. Generated alternatives are never returned as chunks, sources, or citations.

Production requests use a maximum of three total queries (Q1 plus two
alternatives). The retrieval seam supports bounded internal/evaluation overrides
of up to five total queries so experiments can compare fan-out sizes; those
overrides are not production defaults or workspace controls. Query count and
prompts are not exposed as additional workspace knobs until paired evaluation
demonstrates a need.

## Consequences

- Recall can improve for vocabulary mismatch and multi-faceted questions.
- Enabled requests add one utility-model call, additional embedding inputs, and
  more Qdrant prefetch lanes; latency and cost must be visible in benchmarks.
- RRF is a robust baseline for incomparable dense/sparse rankings, but it can
  still regress precision. The feature therefore remains default-off and must be
  evaluated per corpus.
- Existing ingestion-time hypothetical-question enrichment remains independent;
  operators may enable either or both features, and benchmarks must state both
  settings.
- The one-filter security invariant is preserved because expansion changes query
  vectors, not tenancy or ACL construction.

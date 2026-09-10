# ADR-0008: Durable security boundaries for uploads, cleanup, usage, and retrieval

**Status:** Accepted

**Date:** 2026-09-08

## Context

Several security decisions previously existed only as best-effort checks around
external work. Concurrent uploads could all pass a read-only quota check;
deleting a chat cascaded away the only identifiers needed to delete its object
and vectors; completed provider calls could be rolled back by later work; and a
Qdrant response could outlive the document security revision that authorized it.
Query-derived caches also deduplicated equal text across organizations, exposing
cross-tenant timing and usage-attribution coupling even though cached values did
not contain document text.

File transfer and parsing have an additional compatibility constraint: legitimate
large textbooks must keep the streamed 1 GiB upload boundary, complete original
page numbering, PPTX support, and OCR fallback. Resource controls must reject
work before storage or enqueue rather than lower that boundary back to 100 MiB.

## Decision

1. Upload admission uses a PostgreSQL transaction advisory lock per organization
   and a durable `resource_reservations` row. The decision includes committed
   objects and non-expired in-flight reservations. Successful publication removes
   the reservation in the document/attachment transaction; failure or cancellation
   releases it, and abandoned reservations expire after a bounded interval. A
   path-specific ASGI/proxy limit bounds attachment bodies before multipart
   spooling, and image dimensions are checked before storage and again in workers.
2. Permanent documents remain streamed from Starlette's spooled file into object
   storage. Workers stream the object into a temporary file. LiteParse probes the
   source page count and parses bounded, consecutive ranges, rejecting missing or
   changed ranges and an operator-configured maximum. A resource-limit rejection
   cannot fall through to whole-file Docling parsing. Ordinary extraction failure
   may still use the existing OCR fallback. Docling, Anydoc and LlamaParse receive
   the existing temporary-file path rather than a whole-file bytes copy; plain-text
   formats have a smaller pre-storage parser limit.
3. Chat deletion and attachment expiry create an `attachment_cleanup_jobs` row
   containing the external object and vector identifiers before the attachment
   row can cascade. Cleanup is idempotent, independently retryable, and retains
   completed jobs as lifecycle evidence. Legacy orphan reconciliation begins with
   a read-only set comparison; it never deletes historical data automatically.
4. Every completed billable provider action gets a run-scoped operation/stage
   idempotency key. `usage_records` enforces uniqueness for non-null keys, and the
   write is shielded in an independent database session before later fallible
   source lookup, alternative embedding, answer persistence, or stream delivery.
5. Qdrant document and hypothetical-question payloads carry the committed
   `security_revision`. Retrieval re-reads indexed, active, authorized document
   revisions after the vector response and drops mismatches. Existing indexed
   rows are moved to pending by migration until the projection reconciler stamps
   current payloads. This deliberate availability dip prevents legacy unstamped
   vectors from bypassing the new comparison.
6. Query embedding and query-expansion caches include the organization in their
   opaque hashed namespace. They remain bounded, replica-local optimizations and
   cannot share values or single-flight ownership across tenants. This decision is
   separate from the Redis response-cache/CAG feature under active development.
7. User/model-influenced media and web fetches use one address policy and never
   allow non-global destinations. Operator-controlled OIDC, SMTP, and model
   integrations may opt into explicit private CIDRs with
   `RAGZ_EGRESS_ALLOWED_CIDRS`; loopback, link-local, metadata, unspecified, and
   multicast addresses remain prohibited. Media/web requests retain DNS pinning
   and validate every redirect. Admin connector resolution and connection are not
   yet socket-pinned, so DNS rebinding there remains a documented conditional risk.

## Consequences

The migration adds reservation and cleanup tables, attachment byte accounting,
and a partial unique index for usage idempotency. Deployments should expect queued
reprojection of existing vector-backed documents after upgrade. No cleanup runs
against existing objects or vectors during migration.

New uploads can receive typed admission errors sooner, before object storage or
worker dispatch. Expired reservations recover admission after an API crash, while
the durable outbox and cleanup jobs recover external work. Large uploads use disk
and bounded parser batches rather than API or worker memory proportional to the
entire file.

The operator egress allowlist is narrow by design and does not apply to web/media
content. Internal integrations must list their exact routed CIDRs and still cannot
target local or metadata ranges.

# MQR superadmin product-path smoke

Date: 2026-08-24
Commit under test: `6775aeb5` plus the uncommitted Playwright smoke only

## Isolated runtime

- Docker project: `ragz-mqr-product-smoke-20260824`
- Backends: PostgreSQL 16, Redis 7, Qdrant 1.18, MinIO
- API: `127.0.0.1:18765`
- Frontend: `127.0.0.1:15173`
- Dense backend: deterministic hash; reranker: lexical
- Hosted provider calls: zero
- Database: fresh migration through `6a8d2c4f1b90`
- Actor: fresh bootstrap superadmin

## Browser proof

Playwright test:
`frontend/e2e/mqr-settings.spec.ts::superadmin sees and persists the MQR workspace control`

The browser performed:

1. real login (`POST /api/v1/auth/login` → 200);
2. real workspace creation/selection;
3. opened the workspace settings dialog;
4. found the MQR control by its accessible label;
5. toggled it and saved (`PATCH /api/v1/workspaces/{id}` → 200);
6. reopened the dialog; and
7. confirmed the server-persisted checkbox state.

Final result: `1 passed` in `2.0s`.

The first harness attempt tried to inspect `request.postDataJSON()`. Playwright
returned `null` because the application passes a streamed `Request` object to
`fetch`. This was a test-observability limitation, not a product failure: the
API log showed the PATCH succeeded. The browser test was corrected to assert
the live response and reloaded persisted state. Exact PATCH-body minimality
remains covered by the Vitest component test.

## Complementary authorization proof

- Backend focused suite: `93 passed`.
- Frontend complete Vitest suite: `106` files / `701` tests passed.
- API tests prove superadmin success, admin/user 403, null rejection and mixed
  PATCH atomicity.
- Component tests prove the control is hidden for admin/user and a superadmin
  change sends only `multi_query_enabled`.
- Retrieval tests prove configured-off returns one query and enabled/default
  count returns three without mutating the workspace during eval overrides.

## Result

The superadmin-only MQR product contract is implemented and verified at service,
HTTP, component and live-browser boundaries. No authorization or UI production
change was required beyond adding the durable Playwright regression.

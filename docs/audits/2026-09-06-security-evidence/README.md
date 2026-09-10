# Security audit evidence — 2026-09-06

Reports: [application audit](../2026-09-06-end-to-end-security-audit.md), [PR #11 review](../2026-09-06-pr-11-security-review.md), [provisional threat model](../ragz-threat-model.md).

Reviewed PR source: `3fac9fb1d02c9327f243418ebbb905466c8bcef5`; base/main: `9d08839f6855967f3b141b731a269a54b76222fb`; local committed baseline: `90bb3b147b5da88ca704663cce09f77eab7629b7`.

## Artifact index

| Artifact | Meaning |
|---|---|
| `security-tests.xml` | JUnit results for 223 focused tests against explicitly selected PR source; includes the attachment orphan proof |
| `security_audit_auth.py` / `auth-evidence-tests.xml` | Two synthetic PostgreSQL evidence tests; initial execution: 2 passed in 8.95 seconds; independent final rerun: 2 passed in 9.44 seconds |
| `security_audit_attachment_orphan.py` | DB cascade and cleanup-discovery proof; external stores represented as synthetic sets, not real external deletes |
| `reproduce_pr_cache_cancel.py` | Deterministic bounded asyncio probe; both cache variants produce dead slots; only metrics are stubbed |
| `reproduce_egress_guard.py` | Mock DNS/HTTP probe demonstrating the CGNAT classification/redirect gap; never contacts a private host |
| `react-href-proof.mjs` | React 19 static-render check that retracts the direct javascript-link execution hypothesis |
| `local-python-audit.json` / `pr-python-audit.json` | Explicit frozen production lock export audit; 211/212 selected packages respectively, one Transformers advisory each |
| `ci-command-python-audit.json` | Existing CI invocation audits 29 scanner-tool packages, omitting the app dependency inventory |
| `local-frontend-audit.json` / `pr-frontend-audit.json` | Full frontend lock audit including development/build paths |
| `*-frontend-audit-prod.json` | Production-only frontend lock audits; zero advisories |
| `git-secrets-audit.redacted.json` | Gitleaks 8.30.1 all-ref history results: 603 commits, three synthetic test candidates; values fully redacted |

Frontend full scans report eight unique advisory IDs. pnpm 9 and 10 differ in report shape and duplicate affected-range counts; use unique IDs plus dependency paths rather than interpreting raw totals as application vulnerabilities. Scanner databases are time-dependent.

## Execution conditions

The isolated checkout used for this audit was `/tmp/ragz-security-audit-20260906-1CLPbB/repository`. The interpreter was `/home/parshu/projects/ragz/backend/.venv/bin/python` (Python 3.13). An overlay in `/tmp/ragz-security-audit-20260906-1CLPbB/python-overlay` supplied `prometheus-client`, without changing the shared virtualenv. Tests explicitly set:

```bash
PYTHONPATH=/tmp/ragz-security-audit-20260906-1CLPbB/repository/backend/src:/tmp/ragz-security-audit-20260906-1CLPbB/python-overlay
```

Both `ragz.__file__` and the relevant service module paths were verified under the isolated checkout. Running the shared interpreter without this override imports the old local editable installation and does not validate PR #11.

The 223-test run selected:

```text
tests/isolation
tests/api/test_route_policy.py
tests/api/test_route_enforcement.py
tests/api/test_route_action_alignment.py
tests/api/test_auth_routes.py
tests/api/test_api_key_auth.py
tests/api/test_oidc.py
tests/api/test_security_middleware.py
tests/api/test_bots_telegram.py
tests/api/test_bots_slack.py
tests/api/test_bots_discord.py
tests/security_audit_attachment_orphan.py
```

Command form: explicit PR `PYTHONPATH`, shared interpreter `-m pytest -q`, the selections above, and `--junitxml` pointing to a temporary evidence path, from the PR `backend/` directory. Fresh Testcontainers PostgreSQL/Redis/Qdrant/MinIO instances were used through the PR's autouse isolation fixture. Provider/model seams were synthetic; no customer data or paid provider calls were used.

To rerun the two retained PostgreSQL evidence files, place copies in a fresh isolated PR checkout's `backend/tests/` so that the matching `conftest.py` applies. They intentionally assert the defective behavior and should be inverted/replaced when implementing fixes. The supplied scripts use installed test dependencies; the React proof's imports reference the shared frontend installation and may need adapting in another environment.

## Key outputs

```text
Focused PR tests: 223 passed, 8 warnings in 229.16s
Separate auth evidence: 2 passed, 7 warnings in 9.44s on the final rerun
embedding_dead_slot=True
expansion_dead_slot=True
core_guard_blocks_cgnat=true
media_guard_accepts_cgnat=true
mock_destinations=[public.example, 100.64.0.1]
mock_private_text_returned=true
CI-command Python audit: 29 packages, no known vulnerabilities
Explicit PR Python lock audit: 212 packages, one known vulnerability
```

## Local work-in-progress fingerprints

Captured at report preparation; the concurrent thread may subsequently change these files.

| Local file | SHA-256 |
|---|---|
| `backend/src/ragz/modules/cache/response.py` | `3e80625edaaf090cd748a2566bc705b696d0571f4ec09fe3d6918e130379b873` |
| `backend/src/ragz/core/config.py` | `6bdc23f49338c005e2a7252af98d6e93650b64a19c5116820611441232357147` |
| `backend/src/ragz/modules/documents/parsers.py` | `94e1fe9b78217665bb3fd9f845115afb6133c8f9e5581e4427c9935bace3dfcc` |
| `backend/src/ragz/modules/documents/service.py` | `7c6852b766ae78ea8c8a04f68ffb7069274a5d6a5bcb5e19db701803fa35af3a` |
| `frontend/src/features/documents/document-viewer-drawer.tsx` | `5d87974e0299d3a1110f762dca0f10a75602dc4f8e3d6642943f3905614bbd36` |

No browser active-content execution, live session access, destructive exhaustion test, paid-provider fault injection, production network probe, or image/OS scan is represented by these artifacts. Passing ordinary isolation tests is not proof that the separately documented concurrency hypotheses are closed.

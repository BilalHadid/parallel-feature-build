# Benchmark (development evals, October 2026)

Test repo: a small no-dependency Node shop (API + vanilla JS pages + node:test). Each task was run with the skill and without it (same model), and graded with scripted checks (tests pass, lint passes, feature branch used, plan has contract + file ownership, parallel launch, summary quality).

| Eval | Without skill | With skill | With skill, real parallel agents |
|---|---|---|---|
| Orders feature (model, API, page, tests) | 2/7 | 6/7 | **7/7** |
| Typo fix (should skip agents) | 4/5 | **5/5** | — |
| Suppliers + low-stock alerts (shared hotspot files) | 2/7 | 6/7 | — |
| **Overall pass rate** | 46% | 91% | 100% |

## Observations

- **Parallel run (orders):** 3 workers launched together finished in 14–18 s each; merged with 0 conflicts; 16/16 tests passed first time (13 new tests vs 4 in the baseline). Total wall-clock was 50 s vs 27 s for a single agent, because planning and merge overhead dominates on a toy repo.
- **Small task:** the skill correctly skipped agents and ran the tests; the baseline didn't run tests.
- **Without parallel agents available,** v1 still did worktree ceremony and was ~2× slower. v2 detects this and builds directly.
- **Engineering explanations (v2, secure customer accounts):** both runs wrote similar secure code (scrypt, AES-256-GCM, owner/admin checks). Only the skill run named the principles (field-level encryption with AAD, RBAC + object-level authorization / OWASP API1, mass-assignment protection, anti-enumeration, server-side sessions), gave trade-offs, cited OWASP ASVS / NIST SP 800-38D / 800-63B / 800-57, and listed gaps (KMS + key rotation, login rate limiting, TLS termination).

These are small evals on one toy repo. Real-project numbers will differ; contributions of real eval cases are welcome.

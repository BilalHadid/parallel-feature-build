# Engineering Playbook

Named principles, patterns and standards to apply when designing a feature, and to **name explicitly** when explaining the solution. Read the sections that match the feature; skip the rest.

How to use each entry: **Name** — when it applies → what to do. *Trade-off.*

Only claim a principle you actually applied in the code. If one *should* apply but wasn't done (e.g. no encryption at rest yet), list it as a gap, not as done.

## Contents
1. Architecture & code design
2. API design
3. Security (authN, authZ, input/output, crypto, secrets)
4. Data & persistence
5. Reliability & distributed systems
6. Performance & scalability
7. Frontend & UX engineering
8. Testing
9. Observability, delivery & operations
10. Standards to cite

---

## 1. Architecture & code design

- **Separation of Concerns / Layered (Clean) Architecture** — any feature with UI + logic + storage → keep controllers/routes thin, business rules in a service layer, data access in a repository layer. *More files; worth it beyond trivial CRUD.*
- **Single Responsibility (SOLID-S)** — a module/function doing two jobs → split it. Other SOLID: **Open/Closed** (extend via new code, not edits), **Liskov**, **Interface Segregation**, **Dependency Inversion** (depend on abstractions; inject dependencies).
- **Dependency Injection** — services needing DB/clients → pass them in (constructor/params) so tests can substitute fakes.
- **Repository Pattern** — DB access scattered across handlers → one module per aggregate owning queries (e.g. Prisma calls). *Extra indirection.*
- **Service Layer / Use-case pattern** — business rules (total calculation, status transitions) live in services, not in routes or UI.
- **State Machine** — anything with statuses (order pending→paid→shipped, approval flows) → define allowed transitions explicitly and reject the rest.
- **Strategy / Factory / Adapter** — Strategy for swappable algorithms (pricing rules), Factory for object creation logic, Adapter to wrap third-party APIs behind your own interface.
- **Observer / Event-driven** — side effects (send email after order paid) → emit a domain event; handlers subscribe. *Harder to trace.*
- **CQRS** — read and write models diverge heavily (dashboards vs transactions) → separate them. *Overkill for simple CRUD — say so if not used.*
- **Domain-Driven Design: Bounded Contexts, Aggregates, Ubiquitous Language** — multi-module systems (ERP modules) → keep each module's model and names consistent and its boundary explicit.
- **DRY / KISS / YAGNI** — don't duplicate logic, prefer the simplest design that works, don't build for imagined futures.
- **Composition over Inheritance** — prefer composing small units.
- **Fail Fast** — validate at entry points and throw/return errors early.

## 2. API design

- **Contract-First API Design (OpenAPI / shared types)** — any multi-agent or front/back split → define request/response shapes before coding.
- **REST resource modeling & HTTP semantics** — nouns for resources, correct verbs (GET safe, PUT/DELETE idempotent, POST create, PATCH partial), correct status codes (201 Created, 204, 400, 401, 403, 404, 409 Conflict, 422, 429).
- **Consistent error format — RFC 9457 Problem Details** — every error returns the same shape (`type, title, status, detail`) or at least a consistent `{error}`.
- **Idempotency Keys** — payment, order creation, any retried POST → accept `Idempotency-Key` header, store result, return the same response on retry.
- **Pagination: cursor (keyset) vs offset** — lists that grow → cursor pagination for large/live data; offset only for small admin lists.
- **Filtering, sorting, sparse fields** — whitelisted query params only.
- **API Versioning** — breaking changes → `/v2` or header versioning; prefer additive, backward-compatible changes.
- **Rate Limiting (Token Bucket / Sliding Window)** — public or auth endpoints → limit per user/IP, return 429 + `Retry-After`.
- **Input schema validation (Zod/Joi/class-validator, JSON Schema)** — validate every request body at the boundary.

## 3. Security

Map findings to **OWASP Top 10 (2021/2025)** and **OWASP ASVS** chapters when explaining.

Authentication
- **Password hashing: Argon2id (preferred) or bcrypt** — never plain/MD5/SHA for passwords.
- **Session-based auth vs JWT (stateless tokens)** — sessions with HttpOnly, Secure, SameSite cookies for web apps; JWT with short expiry + refresh-token rotation for APIs/mobile. *JWTs can't be revoked easily.*
- **OAuth 2.0 / OpenID Connect, Authorization Code + PKCE** — third-party or SSO login; PKCE for SPAs and mobile.
- **MFA / TOTP** — admin or financial roles.

Authorization
- **RBAC (Role-Based Access Control)** — roles like Manager, QC Inspector, Field Agent → permissions per role, checked server-side.
- **ABAC (Attribute-Based)** — rules depend on data (e.g. only own farm's records).
- **Principle of Least Privilege** — users, DB roles, API keys get minimum rights.
- **Object-level authorization (prevents IDOR / BOLA — OWASP API #1)** — every `/:id` route checks the caller may access *that* record, not just that they're logged in.
- **Deny by default.**

Input / output
- **Parameterized queries / ORM** — prevents SQL Injection (OWASP A03). Never string-build SQL.
- **Output encoding / contextual escaping + Content Security Policy (CSP)** — prevents XSS.
- **CSRF protection** — SameSite cookies + CSRF tokens for cookie-auth state-changing requests.
- **SSRF protection** — server fetching user-given URLs → allowlist hosts, block internal IPs.
- **Secure file uploads** — validate type by content, size limits, random names, store outside web root or in object storage, virus scan where needed.
- **Mass assignment protection** — explicitly pick allowed fields from request bodies.

Cryptography & data protection
- **Encryption in Transit — TLS 1.2+/1.3, HSTS** — all traffic.
- **Encryption at Rest** — disks/DB/storage encrypted.
  - **Server-Side Encryption (SSE)** — storage provider encrypts on write: **SSE-S3** (provider-managed keys), **SSE-KMS** (customer-managed keys in KMS, auditable), **SSE-C** (customer-provided keys); Azure Storage Service Encryption / TDE for Azure SQL & PostgreSQL. *Protects against stolen disks, not against a compromised app.*
  - **Client-Side Encryption** — app encrypts before sending; provider never sees plaintext. *You manage keys; harder search.*
  - **Envelope Encryption** — data encrypted with a data key (DEK), DEK encrypted with a master key (KEK) in KMS.
  - **Field-level / column encryption** — PII (national ID, bank details) → encrypt specific columns (AES-256-GCM) with keys from KMS.
  - **Key Management (KMS / Key Vault / HSM) and Key Rotation.**
- **Hashing vs Encryption vs Encoding** — passwords hashed (one-way), secrets encrypted (reversible with key), Base64 is encoding (not security). Say which one is used and why.
- **HMAC signatures** — webhooks and signed URLs.
- **Tokenization / Data Masking** — show `****1234` in UI and logs.
- **Data minimization & retention (GDPR / Kenya DPA / Pakistan PECA as applicable).**

Secrets & supply chain
- **Secrets management** — env vars / vault (Azure Key Vault, AWS Secrets Manager, Doppler); never commit secrets; rotate.
- **Dependency scanning (SCA), lockfiles, SBOM** — `npm audit`, Dependabot.
- **Security headers** — CSP, HSTS, X-Content-Type-Options, frame-ancestors.
- **Audit Logging** — who did what, when, on sensitive records (immutable, append-only).

## 4. Data & persistence

- **Normalization (3NF) vs deliberate Denormalization** — normalize by default; denormalize read-heavy views with a reason.
- **ACID Transactions** — multi-row writes that must succeed together (order + order items + stock decrement) → one transaction.
- **Isolation levels** (Read Committed, Repeatable Read, Serializable) — name the level when concurrency matters.
- **Optimistic Concurrency Control (version column / ETag + If-Match)** — two users editing the same record → reject stale writes with 409. **Pessimistic locking (SELECT … FOR UPDATE)** for hot counters like stock.
- **Referential integrity** — foreign keys, `ON DELETE` rules.
- **Indexing strategy** — index columns used in WHERE/JOIN/ORDER BY; composite index order matters. *Slows writes slightly.*
- **N+1 Query Problem** — load relations with joins/`include`, not per-row queries.
- **Soft Delete + Audit Trail** — business records (orders, invoices) usually shouldn't hard-delete.
- **Expand–Contract (Parallel Change) migrations** — zero-downtime schema changes: add new, backfill, switch, remove old.
- **Money as integer minor units or DECIMAL** — never floating point for currency.
- **Store time in UTC, display in local time zone**; ISO 8601.
- **Connection pooling** — serverless/Next.js + Postgres → pooler (PgBouncer / Prisma Accelerate / Azure flexible server pooling).
- **Multi-tenancy isolation** — tenant_id on every row + row-level security (Postgres RLS) where needed.

## 5. Reliability & distributed systems

- **Timeouts everywhere** on outbound calls.
- **Retries with Exponential Backoff + Jitter** — only for idempotent operations.
- **Circuit Breaker** — failing dependency → stop calling it for a cool-down period.
- **Bulkhead** — isolate resources so one failure doesn't sink everything.
- **Transactional Outbox Pattern** — DB write + event publish must both happen → write event to an outbox table in the same transaction, publish asynchronously.
- **Saga Pattern (orchestration / choreography)** — multi-service workflows with compensating actions instead of distributed transactions.
- **Eventual Consistency & CAP theorem** — be explicit when data is not immediately consistent.
- **Message queues, Dead-Letter Queue (DLQ), at-least-once delivery + idempotent consumers.**
- **Graceful degradation / fallback** — show cached data when a dependency is down.
- **Health checks (liveness / readiness).**

## 6. Performance & scalability

- **Caching: Cache-Aside, Write-Through, TTL, cache invalidation; HTTP caching (ETag, Cache-Control)**; Redis for shared cache.
- **Stateless services → Horizontal Scaling.**
- **Background jobs / async processing (queues: BullMQ, Azure Service Bus)** for slow work (reports, emails, imports).
- **Big-O awareness** — avoid O(n²) loops over large lists; use maps/sets.
- **Read replicas, query optimization (EXPLAIN ANALYZE).**
- **Batching & debouncing** of network calls.
- **CDN for static assets.**

## 7. Frontend & UX engineering

- **Component composition, presentational vs container components.**
- **Server state vs client state** — server data via React Query/SWR (caching, revalidation); local UI state in component/Zustand/Redux only when shared.
- **Server Components / SSR / SSG / ISR (Next.js)** — choose rendering mode per page and say why.
- **Optimistic UI updates with rollback.**
- **Client + server validation** — client for UX, server for security (never trust the client).
- **Accessibility — WCAG 2.2 AA**: labels, keyboard navigation, focus management, contrast, ARIA only when needed.
- **Loading, empty and error states** for every async view.
- **Code splitting / lazy loading**, image optimization.
- **Internationalization (i18n)** if multi-language.
- **Design tokens / design system consistency.**

## 8. Testing

- **Test Pyramid** — many unit, fewer integration, few end-to-end.
- **Arrange–Act–Assert (AAA)**, one behavior per test.
- **Boundary Value Analysis & Equivalence Partitioning** — test edges (stock 4/5/6 for "below 5"), invalid classes.
- **Contract testing** — API matches the agreed contract (e.g. schema validation in tests, Pact).
- **Integration tests against a real DB (Testcontainers / test database)** for data logic.
- **E2E (Playwright)** for critical user flows.
- **Regression test for every bug fixed.**
- **Mock at boundaries only** (HTTP, time, randomness), not internal modules.
- **Test data factories / fixtures; deterministic tests (no shared mutable state, fixed clocks).**
- **TDD (Red–Green–Refactor)** when requirements are crisp.
- **Security tests** — authZ tests for each role, negative tests for IDOR.

## 9. Observability, delivery & operations

- **Structured logging (JSON) with correlation/request IDs**; never log secrets or PII.
- **Metrics: RED (Rate, Errors, Duration) for services, USE (Utilization, Saturation, Errors) for resources.**
- **Distributed tracing — OpenTelemetry.**
- **Twelve-Factor App** — config in env, stateless processes, dev/prod parity, logs as streams.
- **CI/CD pipeline** — lint, typecheck, test, build on every push; required checks before merge.
- **Trunk-based development / short-lived feature branches, Conventional Commits, Semantic Versioning (SemVer).**
- **Deployment strategies: Blue-Green, Canary, Rolling; Feature Flags; fast Rollback.**
- **Infrastructure as Code (Terraform / Bicep), immutable infrastructure, containers (Docker).**
- **Backups + tested restore; RPO/RTO targets.**
- **Architecture Decision Records (ADRs)** — record significant decisions with context, decision, consequences.

## 10. Standards to cite

When explaining, cite the standard by name so the developer can look it up:
OWASP Top 10, OWASP API Security Top 10, OWASP ASVS, OWASP Cheat Sheet Series, NIST SP 800-63B (passwords/auth), NIST SP 800-57 (key management), RFC 9457 (Problem Details), RFC 9110 (HTTP semantics), RFC 6749 / 7636 (OAuth 2.0 / PKCE), RFC 7519 (JWT), WCAG 2.2, ISO/IEC 27001, PCI DSS (card data), GDPR, Twelve-Factor App, SemVer, Conventional Commits.

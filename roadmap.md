# Core Banking Engineering Roadmap

> **Goal:** Evolve a simple CRUD banking application into a reliable, secure, observable **banking simulator** while learning Java and Spring Boot one production problem at a time.
>
> **Stack:** Java 25 · Spring Boot · Maven · MySQL (InnoDB) · Spring Data JPA · Spring Security · Flyway · Docker · JUnit 5 · Testcontainers
>
> **Architecture:** Package-by-feature modular monolith initially. A **separate payment gateway project** comes later and communicates through authenticated APIs.
>
> **Project:** `D:\workspace\playground\core` · Root package: `com.banking.core` · Entry point: `CoreApplication.java`

## How to use this roadmap

- [ ] Keep the basic application runnable and all existing tests passing before beginning a new level.
- [ ] For each level: **understand the problem → reproduce the failure → implement a solution → test the fix → document the trade-offs**.
- [ ] Make a focused Git branch and commit for each level (for example, `feature/atomic-transfers`).
- [ ] Record design decisions and findings in `docs/decisions/` and results in `docs/experiments/`.
- [ ] Do not treat this learning simulator as a licensed, production banking system or use it for real money.

**Security note:** Authentication, authorization, input validation, and monetary safety checks are mandatory safeguards—not optional features to defer in a public-facing deployment. The level numbers represent learning order, not permission to deploy earlier versions with known vulnerabilities.

## Starting point — CRUD baseline and tooling

**Target behavior:** Register/login, create an account, view balance, send/receive virtual money, view transfer history.

- [ ] Generate/open Spring Boot app using **Java 25** and Maven.
- [ ] Use feature packages such as `auth`, `user`, `account`, `transfer`, `transaction`, `ledger`, `security`, `common` (create only when needed).
- [ ] Configure MySQL (InnoDB) locally, preferably with Docker Compose and a persistent development volume.
- [ ] Configure settings from environment variables; commit `.env.example`, not `.env` or credentials.
- [ ] Install Flyway and add versioned MySQL migrations under `src/main/resources/db/migration/`.
- [ ] Use `spring.jpa.hibernate.ddl-auto=validate` after migrations are in place.
- [ ] Add JUnit 5, Spring Boot tests, and a small happy-path smoke test.
- [ ] Document basic API examples in `README.md`.

**Exit gate:** A developer can clone the repo, start MySQL, apply migrations, and exercise the basic API using documented steps.

---

## Phase 1 — Make the CRUD app safe

### Level 1 — Input validation and banking invariants

**Failure to reproduce:** A transfer accepts a negative/zero amount, exceeds funds, uses mismatched currencies, or targets the same account.

**Learn:** DTOs, Jakarta Bean Validation, `@Valid`, domain validation, `BigDecimal`, SQL constraints, error responses.

**Implement**
- [ ] Create request/response DTOs; never bind financial writes directly to persistence entities.
- [ ] Validate required fields, positive amount, reasonable size, currency, and account IDs.
- [ ] Use `BigDecimal` (or integer minor units) for money; never `double`/`float`.
- [ ] Enforce sufficient funds, account status, allowed currencies, and source ≠ destination.
- [ ] Add database `NOT NULL`, `UNIQUE`, `FOREIGN KEY`, and appropriate `CHECK` constraints.
- [ ] Add a consistent error response using `@RestControllerAdvice`.

**Prove it works**
- [ ] Test zero, negative, excessive, malformed, self-transfer, and currency-mismatch requests.
- [ ] Verify **no balances change** after rejected requests.

**Exit gate:** Invalid operations are rejected at appropriate boundaries, with stable errors and zero financial side effects.

### Level 2 — Authentication and object-level authorization

**Failure to reproduce:** Logged-in User A changes an account ID in a request and reads or spends User B's funds.

**Learn:** Spring Security, principals, password hashing, security filter chain, authentication vs. authorization.

**Implement**
- [ ] Hash passwords with a supported adaptive password encoder.
- [ ] Configure login/session or JWT correctly and protect banking endpoints.
- [ ] Authorize access to **specific accounts**, not merely authenticated users.
- [ ] Derive acting user identity from the authenticated principal, not a client-supplied user ID.
- [ ] Return safe error messages; never expose credentials or sensitive tokens in logs.
- [ ] Test account ownership on balance, transfer, and withdrawal operations.

**Prove it works**
- [ ] Anonymous requests are denied where required.
- [ ] User A cannot view or transfer funds from User B's account.

**Exit gate:** Authentication and ownership checks are covered by automated tests.

---

## Phase 2 — Guarantee correctness of money movement

### Level 3 — ACID transactions and atomicity

**Failure to reproduce:** Debit succeeds, then an exception occurs before credit; money appears to vanish.

**Learn:** ACID, commit/rollback, `@Transactional`, propagation, rollback rules, transaction boundaries.

**Implement**
- [ ] Put one internal transfer's debit, credit, transaction record, and relevant state changes inside **one database transaction**.
- [ ] Decide where the `@Transactional` service boundary belongs; avoid external network calls within the DB transaction.
- [ ] Understand proxy/self-invocation caveats and checked exception rollback behavior.
- [ ] Use Flyway to add transaction/account schema changes explicitly.

**Prove it works**
- [ ] Inject a failure between debit and credit and verify full rollback.
- [ ] Verify success changes both balances by equal and opposite amounts.

**Exit gate:** No partially committed internal transfer is possible through supported application flows.

### Level 4 — Concurrency and double-spending protection

**Failure to reproduce:** Two concurrent withdrawals read the same original balance and both succeed.

**Learn:** Race conditions, lost updates, isolation, InnoDB row locks, `SELECT ... FOR UPDATE`, JPA `PESSIMISTIC_WRITE`, `@Version`, atomic SQL updates.

**Implement**
- [ ] Write a deterministic concurrent withdrawal/transfer test against **real MySQL** using Testcontainers.
- [ ] Implement pessimistic locking for affected accounts within the transaction.
- [ ] Compare against optimistic versioning and conditional updates; document trade-offs.
- [ ] Keep the balance check and actual debit within the same concurrency-control boundary.

**Prove it works**
- [ ] With ₹10,000 initial funds, run 100 competing withdrawals of ₹1,000; expect **10 successes and 90 insufficient-funds outcomes** if all requests reach validation and no unrelated failures occur.
- [ ] Verify final balance is ₹0 and there are no duplicate/lost postings.

**Exit gate:** The account cannot be overspent under concurrent requests.

---

## Phase 3 — Make operations safe under retries and abuse

### Level 5 — Deadlocks and bounded retries

**Failure to reproduce:** A→B and B→A transfers acquire account locks in opposite order and deadlock.

**Learn:** Lock ordering, InnoDB deadlock detection, transient database errors, retry backoff, transaction scope.

**Implement**
- [ ] Lock accounts in a consistent deterministic order.
- [ ] Identify retryable database exceptions precisely; avoid retrying business rejections.
- [ ] Retry failed operations **in a fresh transaction**, with a strict attempt limit and backoff.
- [ ] Record deadlock/retry metrics.

**Prove it works**
- [ ] Run opposing transfers concurrently.
- [ ] Verify no overdraft, no double execution, and explicit failure after retry exhaustion.

**Exit gate:** Deadlocks do not cause unexplained money movement or infinite retries.

### Level 6 — Idempotency and unknown outcomes

**Failure to reproduce:** A transfer commits, its HTTP response is lost, and retrying the request charges twice.

**Learn:** Idempotency keys, database uniqueness, request fingerprints, operation IDs, retries after timeout.

**Implement**
- [ ] Require a client-generated `Idempotency-Key` on money-moving POST endpoints.
- [ ] Store scope (caller), key, request fingerprint, transaction/operation ID, status, and final outcome.
- [ ] Enforce a unique constraint on the key within its scope.
- [ ] Make idempotency reservation and financial posting atomic or design explicit durable recovery for intermediate states.
- [ ] Reject reuse of a key with a different request payload.
- [ ] Expose a safe transaction-status lookup for clients facing uncertain outcomes.

**Prove it works**
- [ ] Send 20 concurrent identical requests with the same key: **one financial effect**.
- [ ] Simulate a lost response after commit and verify safe recovery without a second debit.

**Exit gate:** Retries cannot duplicate the same logical financial operation.

### Level 7 — Rate limiting and API abuse controls

**Failure to reproduce:** A client floods login/balance/transfer endpoints, exhausting resources.

**Learn:** Token bucket, servlet filters/interceptors, HTTP 429, user/IP policies, Redis and distributed state.

**Implement**
- [ ] Introduce Bucket4j or an equivalent rate limiter on selected endpoints.
- [ ] Use sensible per-identity policies, with separate safeguards for unauthenticated login traffic.
- [ ] Return HTTP 429 and useful retry guidance when requests exceed policy.
- [ ] Measure behavior on one instance; add Redis-backed coordination only when using multiple instances.
- [ ] Keep financial invariants independent of the rate limiter.

**Prove it works**
- [ ] Exceed configured quotas and verify throttling and eventual recovery.
- [ ] Confirm valid concurrent transfers remain financially correct even if limiting is disabled.

**Exit gate:** Excess traffic is controlled without becoming a substitute for security or financial correctness.

---

## Phase 4 — Make the bank auditable, testable, and operable

### Level 8 — Immutable ledger and reconciliation

**Failure to reproduce:** A stored account balance changes without a traceable financial movement.

**Learn:** Double-entry accounting concepts, signed ledger movements, balances as projections, reversals, reconciliation.

**Implement**
- [ ] Define posted financial operations and their immutable ledger entries.
- [ ] Represent an internal transfer as balanced movements per currency (e.g., −₹500 and +₹500).
- [ ] Keep ledger posting and balance projection changes in the **same** MySQL transaction.
- [ ] Restrict direct balance/ledger modification; correct posted history using new reversing transactions.
- [ ] Add a reconciliation job that compares ledger-derived balance to stored balance.
- [ ] Model simulated cash withdrawal with a corresponding internal cash/settlement account.

**Prove it works**
- [ ] Every posted internal transfer balances to zero per currency.
- [ ] A deliberately introduced balance mismatch in a test DB is detected and reported.

**Exit gate:** Every money movement is traceable and discrepancies can be detected.

### Level 9 — Integration, concurrency, and load testing

**Failure to reproduce:** Unit tests pass but the application breaks under real MySQL locking or sustained traffic.

**Learn:** Test pyramid, JUnit 5, Mockito, Testcontainers, integration testing, load testing, p95 latency, throughput.

**Implement**
- [ ] Add unit tests for domain rules and application services.
- [ ] Add MySQL Testcontainers integration tests for transactions, deadlocks, and idempotency.
- [ ] Add API tests for authentication and authorization.
- [ ] Run k6 or Gatling test scenarios with explicit workloads and baselines.
- [ ] Track errors, throughput, p95 response latency, connection-pool utilization, and correctness invariants.

**Prove it works**
- [ ] Repeat concurrent posting tests reliably.
- [ ] Produce a short load test report with actual measurements and any observed bottlenecks.

**Exit gate:** Correctness is verified on a real database and performance claims have evidence.

### Level 10 — Observability, recovery, and deployment

**Failure to reproduce:** A user reports an error after sending money, but you cannot tell whether the transfer committed.

**Learn:** Structured logging, correlation IDs, metrics, traces, Actuator, backup/restore, deployment, CI/CD.

**Implement**
- [ ] Add structured request/operation logs with redaction of secrets and sensitive details.
- [ ] Add correlation IDs, financial operation IDs, and explicit result states.
- [ ] Expose health/metrics through Spring Boot Actuator, Micrometer, Prometheus, Grafana.
- [ ] Containerize the application; keep MySQL durable and backed up.
- [ ] Add GitHub Actions pipeline for builds, tests, and dependency/security checks.
- [ ] Practice MySQL backup/restore and service restart; document recovery steps.
- [ ] Run Flyway migrations during controlled deployment; do not rely on Hibernate automatic schema updates.

**Prove it works**
- [ ] Trace a failed/ambiguous transfer to its recorded final state.
- [ ] Rebuild and restore a test environment from documented steps and backups.

**Exit gate:** The service is reproducible, observable, and recoverable in a test environment.

---

## Phase 5 — Extend to asynchronous and distributed payments

### Level 11 — Reliable asynchronous events

**Failure to reproduce:** The transfer commits, but the notification worker crashes and the notification disappears—or is sent twice.

**Learn:** Message queues, at-least-once delivery, transactional outbox, idempotent consumers, retry/dead-letter handling.

**Implement**
- [ ] Identify a nonfinancial async use case, such as transaction notifications.
- [ ] Write an outbox event in the same DB transaction as the financial posting.
- [ ] Add a publisher/worker and message broker only when needed (e.g., RabbitMQ).
- [ ] Build idempotent consumers and controlled retries.

**Prove it works**
- [ ] Stop the worker, commit transfers, restart it, and verify eventual processing.
- [ ] Re-deliver the same event and verify no duplicated financial action.

**Exit gate:** Ancillary processing can fail and recover without changing committed money movement.

### Level 12 — Separate payment gateway project

**Failure to reproduce:** Gateway calls the bank, the bank commits, and the network response is lost. Gateway cannot determine whether to charge again.

**Learn:** Service boundaries, API contracts, service authentication, distributed failure modes, status reconciliation, reliable webhooks.

**Implement**
- [ ] Create a separate Spring Boot repository/project: `payment-gateway`.
- [ ] Give the gateway its **own MySQL database** for merchants, payment orders/attempts, idempotency keys, and webhook deliveries.
- [ ] Implement merchant-facing sandbox APIs and payment status state machine.
- [ ] Authenticate gateway-to-bank calls and implement stable operation IDs and idempotent posting at the bank.
- [ ] Keep the bank as the **sole authority** for balances and ledger postings; gateway must never edit bank tables directly.
- [ ] Add transaction status lookup, timeout recovery, reconciliation, signed webhooks, retries, and idempotent webhook delivery.
- [ ] Use Docker Compose for local multi-service development; use Toxiproxy or equivalent for network failure experiments.

**Prove it works**
- [ ] Payment order → authorized simulated bank movement → final gateway status → merchant webhook.
- [ ] Kill a connection after bank commit; recover the correct outcome without duplicate charging.
- [ ] Verify both service databases can be independently restarted and reconciled.

**Exit gate:** End-to-end simulated merchant payments are correct under timeouts, retries, and duplicate messages.

---

## Milestone tracking

| Milestone | Levels | Deliverable | Status |
| --- | --- | --- | --- |
| Foundation | Baseline | Runnable CRUD app, MySQL, Flyway, Git | ☐ |
| Safe APIs | 1–2 | Validated, authorized banking endpoints | ☐ |
| Correct money movement | 3–4 | Atomic transfers without double spending | ☐ |
| Resilient requests | 5–7 | Deadlock handling, idempotency, throttling | ☐ |
| Auditable bank | 8–9 | Ledger, reconciliation, real-DB/load tests | ☐ |
| Operable bank | 10 | Monitoring, CI/CD, deployment, recovery | ☐ |
| Async reliability | 11 | Durable events and notifications | ☐ |
| Payment platform | 12 | Separate sandbox payment gateway integrated with bank | ☐ |

## Engineering principles to retain across all levels

1. **Never use floating-point arithmetic for financial balances.**
2. **All money movements must be authorized, validated, and atomic.**
3. **Never trust client-supplied account ownership or payment status.**
4. **Concurrency correctness belongs in the database transaction design, not only in Java `synchronized` blocks or rate limiting.**
5. **Persist final transaction outcomes and handle unknown outcomes explicitly.**
6. **Ledger history is immutable; corrections are new movements.**
7. **Keep tests, migrations, and documentation alongside code changes.**
8. **Extract a service only when its boundary and consistency requirements are understood.**

## Suggested next task

Start **Level 1** by writing failing tests for negative amounts, insufficient funds, self-transfers, and unauthorized state-changing inputs; then add DTO validation, domain checks, SQL constraints, and consistent errors. Record the tests and outcomes in `docs/experiments/level-01-validation.md`.

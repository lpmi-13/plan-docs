# Idempotency vs. Deduplication: Django Learning Environment Plan

## 1. Product Summary

Build a runnable Python/Django learning environment in which a learner sends duplicate and retried payment-style requests, observes the resulting database state and execution trace, and then fixes progressively harder failure modes.

The environment should teach the distinction shown in the reference screenshot without turning it into an inaccurate slogan:

- **Idempotency is a behavioral property:** applying the same intended operation repeatedly has the same externally visible final effect as applying it once.
- **Deduplication is a processing policy:** identify a repeated message or request and suppress, coalesce, or reuse an earlier result instead of processing it again.
- Deduplication can be one technique for achieving idempotent behavior, but neither concept automatically guarantees the other.

The central exercise is a small payments API. A learner can deliberately introduce retries, concurrent requests, delayed duplicate webhooks, key collisions, and failures at transaction boundaries, then compare naive, deduplicated, and idempotent implementations in a real Django backend.

## 2. Learning Outcomes

After completing the labs, a learner should be able to:

1. Explain the difference between an operation's identity, a transport delivery, and a business effect.
2. Recognize naturally idempotent operations, such as setting a resource to a value, and non-idempotent operations, such as incrementing a balance.
3. Explain why HTTP method semantics alone do not make an implementation safe under retries.
4. Implement an idempotency-key contract that returns the original outcome for a legitimate retry.
5. Implement deduplication for event delivery with a clearly stated identity rule and retention window.
6. Reject reuse of the same idempotency key with a different request payload.
7. Use database constraints and transactions to make duplicate detection safe under concurrency.
8. Reason about failures before, during, and after a business effect, including the “commit succeeded but the response was lost” case.
9. Explain the limits of time-to-live dedupe, process-local locks, and “check then insert” code.
10. Decide when idempotency, deduplication, or both are appropriate in a production design.

## 3. Audience, Prerequisites, and Scope

### Audience

- Backend developers familiar with Python and basic HTTP APIs.
- Engineers learning retry safety, webhook consumption, or distributed-system fundamentals.
- Interview candidates or teams that want a hands-on reliability workshop.

### Prerequisites

- Basic Python, Django ORM, JSON, and HTTP knowledge.
- Docker is recommended; a local Python environment remains an optional path.
- No previous payment-provider or message-broker experience is required.

### MVP Scope

- Django and Django REST Framework API.
- PostgreSQL as the authoritative datastore so transactions, row locking, and uniqueness behavior are realistic.
- A small server-rendered or Django-admin-based observation interface; no separate frontend framework is required.
- A CLI “traffic lab” that generates sequential retries, concurrent duplicates, changed payloads, and delayed replay.
- Automated tests that initially expose broken behavior and later validate the learner's implementation.
- Guided lessons and concise architecture notes in the repository.

### Non-Goals

- Moving real money or integrating a live payment provider.
- Claiming exactly-once delivery or exactly-once execution across arbitrary distributed systems.
- Building a production-ready ledger, fraud system, or general-purpose message broker.
- Hiding all complexity behind middleware; the learner must see the transaction boundaries.
- Treating deduplication as merely removing identical JSON bodies.

## 4. Core Teaching Model

Use three deliberately contrasted flows.

### 4.1 Naive Charge: Neither Safe nor Deduplicated

`POST /api/lab/naive-charges/` creates a charge and decrements a simulated account balance on every call. Sending the same JSON twice creates two charge rows and two balance movements.

This establishes the baseline: a retry is a new delivery, and the server has no way to infer whether the caller intends a new operation.

### 4.2 Idempotent Charge: Re-execution Has One Business Effect

`POST /api/lab/idempotent-charges/` requires an `Idempotency-Key` header. Within a defined key scope:

- The first request reserves the key, applies the effect, and records the response.
- A retry with the same key and canonical request fingerprint returns the recorded status and body without creating another charge or balance movement.
- Concurrent requests with the same key converge on one result.
- Reusing the key with a different fingerprint returns `409 Conflict`.
- A key represents an intended operation, not simply a transport request.

The UI and trace should explicitly show that the endpoint's *observable behavior* is idempotent, while stored-result replay/deduplication is the mechanism used in this implementation.

### 4.3 Webhook Consumer: Deduplicated Delivery

`POST /api/lab/webhooks/` accepts a provider event with a stable `event_id`. The consumer records the delivery, atomically claims the event identity, and runs the handler only for the first accepted occurrence in the retention window. Later occurrences are labeled duplicates and do not invoke the business handler.

This flow demonstrates that dedupe answers “have I accepted this delivery identity before?” It does not prove the handler itself is idempotent. A separate exercise bypasses or expires the dedupe record and shows that a non-idempotent handler can still double-apply an effect.

## 5. Concept Matrix Presented to Learners

| Scenario | Is processing repeated? | Is final business effect repeated? | Concept demonstrated |
| --- | --- | --- | --- |
| Naive balance increment | Yes | Yes | Neither |
| `PUT` desired account status twice | Yes | No | Naturally idempotent state transition |
| Charge retry returns stored response | Suppressed or replayed | No | Idempotent API implemented using duplicate recognition |
| Webhook event ID seen twice | No handler execution on retry | No, while record is retained | Deduplication protecting a handler |
| Two distinct event IDs request the same effect | Yes | Possibly | Dedupe identity is insufficient for business idempotency |
| Same event after dedupe TTL expires | Yes | Depends on handler | Retention-window limitation |

Every lesson should show four identities separately: correlation/trace ID, idempotency key, provider event ID, and resulting business object ID.

## 6. Learner Experience

### 6.1 Quick Start

The learner runs one command to start Django and PostgreSQL, applies migrations automatically in the development entrypoint, and opens a lesson index. Seed data provides a customer account and an initial balance.

The landing page contains:

- A concise definition and concept matrix.
- Links to the current lab and its failing test.
- A reset button limited to lab data.
- A timeline of recent request attempts and business effects.
- Copyable `curl` and CLI commands.

### 6.2 Experiment Console

The console lets a learner choose an endpoint and configure:

- Attempt count and concurrency level.
- Reuse or regeneration of the idempotency key/event ID.
- Same or changed payload.
- Artificial delay at named transaction checkpoints.
- Simulated response loss after commit.
- Webhook dedupe retention time and replay delay.

The result view displays request statuses, response bodies, timestamps, worker/process identifier, key disposition (`claimed`, `replayed`, `conflict`, or `duplicate`), query outcome, and final account/ledger state.

### 6.3 Live-System Verification

Exercises must drive the deployed application over HTTP rather than calling Django views or service functions in-process. A separate `labctl` controller waits for health and readiness endpoints, creates an isolated exercise tenant through a control API, sends traffic to the public lab API, and queries a read-only evidence API after the run. This makes the verification path representative of a real client retrying a real multi-worker Django service backed by PostgreSQL.

Each scenario contains machine-checkable invariants, for example:

- all HTTP attempts reached the running service and have distinct correlation IDs;
- exactly one charge and debit exist for the intended operation;
- every successful replay has the same resource ID, status, and semantic response body;
- a mismatched payload is rejected without changing business state;
- the number of webhook deliveries differs from the number of handler executions as expected; and
- no idempotency record remains stuck after the scenario's recovery deadline.

`labctl verify` should assert those invariants using API-visible evidence first. A privileged verifier may query a read-only database view as a second, independent check, but learner solutions must not pass merely by forging a success response. The controller returns a nonzero exit code and a useful invariant diff on failure, so the same verification can run locally, in CI, or against a remote VM.

### 6.4 Guided Lab Loop

Each lab follows the same cycle:

1. Read a short prediction question.
2. Generate a seeded scenario from the current lesson's catalog and run it against the live service.
3. Inspect the request trace and database state.
4. Run the focused failing test.
5. Change a small implementation seam or select the fixed implementation.
6. Re-run the scenario and explain the outcome.

Include solutions on a separate branch, directory, or instructor toggle so the default path encourages practice without permanently breaking the main application.

### 6.5 Repeatable Random Challenge Mode

Challenge mode chooses from a versioned catalog of failure profiles instead of presenting one fixed example. A profile declares its learning objective, compatible endpoint, setup, parameter ranges, injected fault schedule, traffic pattern, evidence to collect, and invariants. Initial profiles should include sequential replay, concurrent key collision, same-key/different-payload, pre-commit rollback, post-commit response loss, stuck-claim recovery, webhook replay inside/outside the retention window, and distinct event IDs for one business object.

Generation is random enough to prevent memorizing one set of values, while remaining diagnosable:

- The controller creates a cryptographically random seed by default and prints it with the catalog version.
- `--seed` reproduces the exact tenant data, amounts, keys, delays, concurrency, request ordering, and fault schedule.
- Parameter generation respects constraints so every generated case is meaningful and has a known oracle.
- The chosen profile name can be hidden until grading completes, while traces and symptoms remain available for debugging.
- A run artifact records the seed, catalog version, target build identifier, sanitized requests, responses, trace IDs, and invariant results.
- Learners can request a new challenge without rebuilding the application; instructors and CI can replay any failed artifact exactly.

Randomness selects and parameterizes reviewed scenarios; it does not invent arbitrary faults or expected answers. The invariant oracle is code-reviewed alongside each catalog profile, and every profile also has fixed regression seeds.

## 7. Curriculum

### Lab 0: Establish the Vocabulary

- Run the naive charge twice.
- Distinguish two HTTP attempts from one intended purchase.
- Compare “set balance to 90” with “subtract 10 from balance.”
- Checkpoint: identify which operation is naturally idempotent and why.

### Lab 1: Sequential Retry and Response Replay

- Add an idempotency key to the charge endpoint.
- Store the key's request fingerprint, state, response status, and response body.
- Replay the first successful response on a retry.
- Checkpoint: one charge, one ledger entry, two attempt traces.

### Lab 2: Same Key, Different Intent

- Send the same key with a changed amount, currency, or customer.
- Demonstrate why key existence alone is unsafe.
- Canonicalize the selected business inputs, hash them, and return `409` on mismatch.
- Checkpoint: incidental JSON key order does not alter the fingerprint, but business data does.

### Lab 3: Concurrency Race

- Run two requests that pause between `exists()` and `create()`.
- Observe double execution or an integrity error in the broken implementation.
- Replace check-then-act with a database uniqueness constraint plus an explicit transactional claim/wait strategy.
- Checkpoint: ten concurrent attempts produce one effect and consistent replay responses.

### Lab 4: Failure Boundaries

- Simulate failure before the claim, after the claim but before the effect, and after commit but before the client receives a response.
- Define retryable versus terminal states.
- Avoid caching accidental framework errors as completed business outcomes.
- Checkpoint: no key is left indefinitely `PROCESSING`; a lost response can be recovered safely.

### Lab 5: Webhook Deduplication

- Deliver the same `event_id` several times.
- Claim it atomically and suppress repeated handler execution.
- Record every delivery attempt separately from the unique processed event.
- Checkpoint: the learner can explain why request logs may show five deliveries but one handler run.

### Lab 6: Dedupe Window and Identity Choice

- Expire an event receipt and replay it.
- Send two event IDs that describe the same business transition.
- Observe that transport-level dedupe cannot guarantee business-level idempotency.
- Make the underlying state transition idempotent with a unique provider object ID or a compare-and-set rule.
- Checkpoint: both layers together tolerate delayed and re-enveloped delivery.

### Lab 7: Production Design Exercise

- Choose key scope, retention, response-replay policy, and monitoring for a fictional API.
- Document tradeoffs around storage, privacy, denial-of-service limits, and cleanup.
- Complete a short rubric-based design review rather than adding more code.

## 8. Functional Requirements

### API Contract

- Accept UUID-like or opaque idempotency keys within documented length and character limits.
- Scope idempotency keys by authenticated principal and endpoint/operation type; one tenant must not collide with another.
- Fingerprint normalized business inputs, not headers such as trace ID and not secrets.
- Return the original status and response representation for completed legitimate retries.
- Return a stable conflict response when a key is reused for a different request.
- Define behavior for an in-progress operation: bounded wait followed by replay, or `409/425` with a documented retry hint.
- Include `Idempotency-Replayed: true|false` for teaching visibility while noting that this is an application convention.
- Keep all example money values as integer minor units and validate currency.

### Scenario Runner

- Provide a standalone `labctl` CLI for black-box live-system exercises; retain Django management commands only for administration and local data setup.
- Load versioned, schema-validated scenario manifests from the catalog and reject unknown catalog versions.
- Accept `--target`, `--seed`, `--profile`, and `--artifact-dir` options without assuming `localhost`.
- Use a barrier to start concurrent requests at approximately the same time.
- Support named, seeded fault-injection schedules rather than nondeterministic application failures.
- Poll asynchronous recovery with explicit time bounds instead of fixed sleeps wherever possible.
- Print a concise attempt table, assert API and durable-state invariants, and save a replayable run artifact.
- Exit nonzero when an expected invariant is violated so scenarios work in CI.

### Scenario Catalog and Oracle

- Represent profiles as declarative YAML or JSON plus a small allowlisted library of traffic generators and invariant predicates; do not execute arbitrary code from manifests.
- Give every profile a stable ID, semantic version, difficulty, concept tags, compatible application versions, and fixed regression seeds.
- Separate generated inputs from expected invariants so changing an amount or key cannot accidentally change the rule being tested.
- Validate catalog profiles in CI by proving the vulnerable implementation fails the intended invariant and the reference implementation passes it.
- Prevent false positives by checking durable resource counts and identities, not only status codes or application-provided “passed” flags.
- Sign or checksum published remote catalogs and include the digest in each run artifact.

### Observation and Reset

- Persist request attempts even when a business transaction rolls back, or emit them to a separate teaching trace sink.
- Redact authorization values and cap stored payload size.
- Show transaction and effect boundaries in chronological order.
- Allow resetting only the seeded lab tenant, with a prominent confirmation.
- Issue short-lived exercise credentials scoped to one generated tenant, and make reset/setup operations unavailable through the learner-facing API.

## 9. Recommended Technical Architecture

### Stack

- Python version supported by the selected current Django release.
- Django, Django REST Framework, PostgreSQL, and `pytest-django`.
- Docker Compose for reproducible application and database services.
- Gunicorn with multiple workers in the concurrency lab to prove process-local memory is insufficient.
- Optional Redis/Celery extension only after the database-backed core lessons work; a broker should not be required for the MVP.

Pin exact versions when implementation begins and maintain them with an automated dependency-update process rather than hard-coding speculative versions in this plan.

### Runtime Topology

Keep the system-under-test and the verifier as separate processes and containers:

- `web`: Gunicorn running multiple Django workers.
- `db`: PostgreSQL with a persistent volume.
- `labctl`: an ephemeral controller container or locally installed CLI that only uses documented HTTP/control interfaces.
- `proxy` (optional for the PoC): a deterministic fault proxy that can delay, duplicate, disconnect, or discard responses without modifying Django code.

Application-level fault hooks are acceptable for transaction checkpoints that a proxy cannot reach, but they must be activated by a short-lived scenario token and an allowlisted fault name. The fault schedule belongs to the isolated exercise run, not global process state, so two learners cannot interfere with each other.

### Local and Remote Execution

Use the same immutable container images and Compose definition for the local PoC and a single Linux VM. Configuration changes through environment variables, not source edits:

- Locally, `docker compose up --wait` starts the stack and `labctl --target http://localhost:8000 challenge` runs an exercise.
- On a VM, bind the application behind TLS, supply `LAB_TARGET_URL`, and run `labctl` from the learner's machine or an isolated controller container.
- Health checks prove the process is alive; readiness checks also prove migrations are current and PostgreSQL is reachable before traffic begins.
- Image tags and migrations are pinned to a build identifier that is captured in every run artifact.
- Persistent learner data and disposable exercise tenants use distinct database roles and lifecycle rules.
- Remote control endpoints use authentication, TLS, tenant scoping, rate limits, and an explicit deployment flag; they are never exposed as unauthenticated debug endpoints.

The first remote target can be one Docker Compose-managed VM. The component boundaries should permit later migration to a scheduler or multiple VMs without changing scenario manifests or the learner CLI.

### Django Apps

- `lessons`: lesson metadata, progress, prompts, and solution explanations.
- `payments`: accounts, charges, and immutable ledger entries.
- `idempotency`: key claiming, fingerprints, state transitions, and response replay.
- `webhooks`: delivery receipts, processed-event claims, and sample handlers.
- `observability`: attempt traces, fault injection, timeline views, and lab reset.
- `control_plane`: authenticated exercise allocation, scoped fault schedules, evidence access, and teardown for local or remote controllers.

Keep idempotency orchestration in an explicit service called by the view. Middleware may parse and validate a key, but it should not obscure business transactions or indiscriminately cache responses.

### Request Flow

1. Authenticate the lab user and validate the request.
2. Compute `(principal, operation, idempotency_key)` and the canonical business-input fingerprint.
3. Atomically insert or lock the idempotency record.
4. On an existing record, reject a fingerprint mismatch, wait/retry for a bounded in-progress result, or replay a completed result.
5. For the owner of a new claim, create the charge and ledger entry within the intended transaction boundary.
6. Persist the terminal result and commit.
7. Serialize the response and optionally trigger the simulated post-commit connection loss.
8. Record teaching telemetry without changing the business outcome.

The implementation must document whether the key record and business effect share one database transaction. If an external side effect is added later, introduce an outbox/provider-idempotency exercise rather than suggesting that a local transaction covers the remote system.

## 10. Data Model

### `Account`

- `id`, `tenant_id`, `currency`, and timestamps.
- Balance should be derived from immutable ledger entries for teaching checks, or maintained as a locked projection with invariant tests.

### `Charge`

- `id`, `account_id`, `amount_minor`, `currency`, `status`, `provider_reference`, and timestamps.
- Unique business reference where a lesson requires business-level idempotency.

### `LedgerEntry`

- `id`, `account_id`, `charge_id`, `amount_minor`, `entry_type`, and `created_at`.
- Database constraints prevent more than one debit entry for a charge.

### `IdempotencyRecord`

- `id`, `tenant_id`, `operation`, `key`, and `request_fingerprint`.
- `state`: `PROCESSING`, `COMPLETED`, or `FAILED_RETRYABLE` with an intentionally explicit recovery policy.
- `response_status`, bounded/redacted `response_body`, `resource_type`, and `resource_id`.
- `locked_until`, `created_at`, `completed_at`, and `expires_at`.
- Unique constraint on `(tenant_id, operation, key)` plus indexes for expiry cleanup.

### `WebhookDelivery`

- One row per received HTTP attempt: trace ID, provider, event ID, payload fingerprint, received time, and disposition.

### `ProcessedEvent`

- Provider, tenant, event ID, first-seen time, processed time, expiry time, and handler result.
- Unique constraint on the documented event identity scope.

### `AttemptTrace`

- Correlation ID, scenario ID, endpoint, process/thread label, checkpoint events, response status, and timestamps.
- No raw credentials or unrestricted sensitive payloads.

## 11. Correctness and Edge Cases

The labs and reference solution must cover:

- Sequential and concurrent duplicate delivery.
- Same key with semantically identical JSON in a different field order.
- Same key with different amount, currency, tenant, or operation.
- Missing, malformed, oversized, and high-cardinality keys.
- Validation failure before an idempotency claim.
- Database rollback and retryable failure recovery.
- Success committed while the client times out or disconnects.
- A request encountering an existing `PROCESSING` record.
- Dedupe replay before and after expiry.
- Two event IDs representing the same business transition.
- Cleanup racing with a retry.
- Clock skew avoidance by using database timestamps where ordering matters.
- Safe behavior across multiple application workers and restarts.

The documentation must avoid promising exactly-once execution. The demonstrated guarantee is one durable business effect for the defined operation identity and system boundary, under stated retention and availability assumptions.

## 12. Testing Strategy

### Unit Tests

- Canonical request fingerprinting and excluded fields.
- Key validation and scope derivation.
- State-machine transitions and replay serialization.
- Dedupe expiry decisions using an injected clock.

### Database and API Tests

- Unique constraints and transaction rollback behavior against PostgreSQL, not SQLite.
- Original response replay, mismatch conflicts, and validation errors.
- Ledger invariants: one charge debit and balanced entries as applicable.
- Webhook attempt count versus processed-event count.

### Concurrency Tests

- Use threads or processes with independent database connections and a synchronization barrier.
- Assert one committed business effect under a burst of identical attempts.
- Verify that different idempotency keys can progress independently.
- Repeat enough times to expose races while retaining a deterministic fault-injection test for CI reliability.

### End-to-End Scenarios

- `retry_after_lost_response`.
- `same_key_changed_payload`.
- `ten_concurrent_charges`.
- `webhook_redelivery_within_window`.
- `webhook_redelivery_after_window`.
- `different_event_ids_same_business_object`.

Run end-to-end scenarios through `labctl` against a listening multi-worker server. For every catalog profile, maintain at least one regression seed and a contract test for its run artifact. A catalog qualification job runs each profile against both a deliberately vulnerable build (expected to fail the targeted invariant) and the reference build (expected to pass), preventing ineffective exercises and broken oracles from shipping.

### Quality Checks

- Formatting, linting, static type checks for typed service modules, migrations check, and Django system checks.
- Dependency and container vulnerability scanning in CI.
- Documentation command validation so every copied example remains executable.

## 13. Security and Operational Guardrails

- Authenticate all non-demo deployments and scope keys to a tenant or principal.
- Rate-limit requests and cap retained idempotency records to reduce key-space abuse.
- Never place card data, credentials, or personal data in idempotency keys.
- Store only the minimum response required for replay; redact sensitive fields and encrypt where warranted.
- Use constant, documented limits for key length, response size, and retention.
- Protect fault injection and reset endpoints behind development settings and staff authorization; fail application startup if they are enabled in a production configuration.
- Emit metrics for new claims, replays, conflicts, stuck processing records, dedupe hits, handler failures, and cleanup lag.
- Provide a cleanup command that deletes expired records in bounded batches and never removes active claims.

## 14. Repository Shape

```text
.
├── compose.yaml
├── manage.py
├── pyproject.toml
├── config/
├── apps/
│   ├── lessons/
│   ├── payments/
│   ├── idempotency/
│   ├── webhooks/
│   └── observability/
├── scenarios/
│   ├── catalog/
│   ├── schema/
│   └── regression-seeds/
├── labctl/
├── templates/lessons/
├── tests/
│   ├── unit/
│   ├── integration/
│   ├── concurrency/
│   └── e2e/
└── docs/
    ├── concepts.md
    ├── labs/
    ├── instructor-guide.md
    └── production-caveats.md
```

## 15. Delivery Phases

### Phase 1: Foundation

- Scaffold Django, PostgreSQL, Docker Compose, health checks, seed/reset commands, and CI.
- Implement accounts, charges, ledger invariants, naive endpoint, and attempt trace.
- Implement the standalone controller, exercise allocation, evidence API, and one fixed black-box verification scenario.
- Deliver Labs 0–1 documentation and baseline tests.

**Exit criterion:** a fresh checkout starts the real multi-worker service with one documented command, and the external controller proves that duplicate naive requests double-apply an effect.

### Phase 2: Idempotency Core

- Implement key validation/scoping, canonical fingerprinting, record state machine, and response replay.
- Add payload-conflict and lost-response scenarios.
- Deliver Labs 1–4 and their focused test suites.

**Exit criterion:** sequential and deterministic concurrent retry tests produce one durable charge and stable responses.

### Phase 3: Deduplication Track

- Add webhook deliveries, processed-event claims, retention cleanup, and business-object uniqueness exercise.
- Deliver Labs 5–6 and update the concept matrix with links to live traces.

**Exit criterion:** learners can observe both dedupe suppression and the failure of dedupe alone after expiry or identity changes.

### Phase 4: Teaching Polish and Hardening

- Build the experiment console and timeline, add instructor solutions and design-review rubric.
- Complete the versioned randomized catalog, regression seeds, run artifacts, and vulnerable/reference qualification suite.
- Add remote-VM configuration, TLS/control-plane security, accessibility review, and complete CI checks.
- Validate the curriculum with several learners and revise confusing steps.

**Exit criterion:** a learner can repeatedly receive unfamiliar but reproducible challenges and verify a fix against either a local stack or a remote Linux VM without undocumented instructor intervention.

## 16. Acceptance Criteria

- The environment starts locally from documented commands and uses PostgreSQL for correctness labs.
- The same container images and scenario manifests run on a documented single-VM Linux deployment without hard-coded localhost addresses.
- The verifier runs outside the Django process, sends real HTTP traffic to a multi-worker server, and fails when durable-state invariants are violated even if responses claim success.
- A learner can generate a challenge from the reviewed catalog, replay it using its printed seed, and obtain the same inputs and fault schedule.
- Run artifacts capture enough sanitized evidence to reproduce and diagnose a failure, including catalog version and target build identifier.
- Every catalog profile is automatically shown to fail a vulnerable implementation and pass the reference implementation for fixed regression seeds.
- The naive scenario reliably demonstrates duplicate business effects.
- The idempotent endpoint creates exactly one charge and ledger effect for repeated and concurrent attempts sharing a valid operation identity.
- A legitimate retry receives the original recorded outcome; a changed payload with the same key receives a conflict.
- The webhook flow records every delivery while invoking the handler once for a retained event identity.
- A lab demonstrates that dedupe expiry or a changed event ID can permit reprocessing, and that business-level idempotency is a separate defense.
- Named failure points demonstrate rollback, stuck-claim recovery, and post-commit response loss.
- Tests assert database state and business invariants, not only HTTP response codes.
- Lesson text consistently distinguishes attempts, executions, effects, identities, and retention boundaries.
- Development-only fault and reset controls cannot be enabled silently in production settings.

## 17. Risks and Mitigations

| Risk | Mitigation |
| --- | --- |
| Learners leave believing idempotency means “cache every POST response” | Begin with naturally idempotent state-setting and repeatedly separate property from mechanism. |
| SQLite hides PostgreSQL concurrency behavior | Require PostgreSQL for integration and concurrency labs. |
| Race tests become flaky | Pair realistic bursts with barriers and deterministic checkpoint fault injection. |
| Examples imply exactly-once guarantees | State system boundaries, identity scope, expiry, and failure assumptions in each lab. |
| Middleware obscures the important transaction | Keep orchestration in a visible service and annotate trace checkpoints. |
| Stored responses leak sensitive data | Use synthetic data, redaction, size caps, short retention, and documented production caveats. |
| Cleanup reopens old operations unexpectedly | Make expiry part of the API contract and demonstrate its consequence in a dedicated lab. |
| Random challenges are flaky or have no valid answer | Generate only within profile constraints, preserve seeds, and qualify each oracle against vulnerable and reference builds. |
| Learner code fakes a passing API response | Verify independent durable-state invariants through a privileged read-only evidence path. |
| Remote fault controls become a security risk | Require TLS, short-lived scoped tokens, allowlisted faults, deployment gating, and isolated exercise tenants. |
| Local and VM behavior diverge | Run the same pinned images, manifests, migrations, and black-box controller in both environments. |

## 18. Future Extensions

- Celery task redelivery with an inbox/outbox pattern.
- A simulated external payment provider that supports its own idempotency keys.
- Kafka-style partition offsets contrasted with business-event identity.
- Optimistic concurrency and compare-and-set exercises.
- Property-based tests that generate retry and failure schedules.
- OpenTelemetry trace export for visualizing attempts across services.
- A second domain, such as email dispatch or inventory reservation, to show where suppressing duplicate processing and preserving an idempotent final state have different tradeoffs.

## 19. Definition of Done

The MVP is done when the complete lab path is runnable from a clean checkout, all automated checks pass against PostgreSQL, the concurrency and failure scenarios are reproducible, and learner documentation accurately explains not just how to store a key but why the chosen identity, transaction boundary, replay behavior, and retention window determine the actual guarantee.

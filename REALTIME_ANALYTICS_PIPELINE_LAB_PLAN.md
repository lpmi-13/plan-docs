# Near-Real-Time Analytics Pipeline Lab — Product and Implementation Plan

## 1. Product Summary

Build a self-contained learning resource in which users evolve a small analytics system from a deliberately naive design into several more capable alternatives. Rather than only reading that polling is simpler than change data capture (CDC), or that a broker helps with backpressure, learners run both designs, inject faults and load, and watch the consequences in real time.

The sample product is a multi-tenant order application backed by PostgreSQL. Its dashboard reports order count, gross revenue, average order value, and status counts. A traffic generator changes orders while the learner sees the dashboard, pipeline topology, event trace, lag, queue depth, duplicate count, resource consumption, and correctness checks.

The minimum viable product (MVP) runs on a laptop with Docker Compose. A matching k3s deployment is a follow-on portability target, not a prerequisite for the first useful lesson. All experiments use small data volumes and time-compressed scenarios, while the teaching notes explain how the observed behavior extrapolates—and where it does not extrapolate—to production.

The core promise is:

> Change one architectural decision, run the same reproducible workload, and compare latency, correctness, recovery, cost, and operational complexity with visible evidence.

## 2. Product Principles

1. **Build before naming.** Start with a working small system; introduce infrastructure only after a learner observes the problem it solves.
2. **Compare, do not crown winners.** Every option is presented with a context in which it is reasonable and a cost it imposes.
3. **Correctness is observable.** A green dashboard is insufficient; an independent oracle continuously compares reported aggregates with source truth.
4. **Failures are first-class inputs.** Pauses, duplicates, restarts, malformed events, schema changes, and reconnects are repeatable lesson steps rather than accidental breakage.
5. **One variable at a time.** Most labs swap one component or policy while preserving the workload, event schema, and success measures.
6. **Local scale is honest.** The resource teaches mechanisms and trends, not laptop benchmark numbers as production capacity claims.
7. **The simple design remains available.** Learners should be able to conclude that polling or a scheduled aggregate is the correct answer for a stated requirement.

## 3. Audience, Prerequisites, and Learning Outcomes

### 3.1 Audience

- Backend and full-stack developers learning system design.
- Engineers who know individual tools but want to understand why a pipeline acquires them.
- Interview candidates and teams running architecture workshops.

### 3.2 Prerequisites

- Basic SQL, HTTP, JSON, and container familiarity.
- Ability to read a small amount of Python or TypeScript; no Kafka or Kubernetes experience is required.
- Docker with Compose, 8 GB of available memory, and roughly 10 GB of free disk for the MVP.

### 3.3 Learning outcomes

After the core labs, a learner should be able to:

1. Translate a vague “real-time” request into a latency service-level objective (SLO), freshness semantics, and failure budget.
2. Compare query-on-source, polling an aggregate table, trigger/outbox capture, and PostgreSQL logical replication.
3. Explain why WAL position, transaction boundaries, snapshots, and replication slots matter to CDC.
4. Describe where backpressure accumulates and recognize lag, queue depth, and PostgreSQL WAL retention as different symptoms.
5. Select an ordering key and delivery guarantee based on the aggregation rather than on slogans.
6. Make updates, deletes, duplicate delivery, and replay produce correct aggregates.
7. Compare recomputation, incremental aggregation, and a separate analytical serving store.
8. Compare browser polling, server-sent events (SSE), and WebSockets for dashboard delivery.
9. Design reconnect/catch-up behavior and communicate eventual consistency in the UI.
10. Trace one order mutation from database commit to the visible pixel and identify the slow stage.
11. Recover a stalled consumer, rebuild derived state, and perform a compatible schema migration.
12. Make a defensible “simplest system that meets the requirements” recommendation.

## 4. Scope

### 4.1 MVP

The MVP includes:

- One order-writing API and PostgreSQL source database.
- Seeded tenants, products, and deterministic traffic profiles.
- Two complete pipeline modes:
  - **baseline:** periodic SQL aggregation plus browser polling;
  - **streaming:** transactional outbox, Redis Streams, an incremental aggregator, and SSE.
- A learner-facing dashboard with live business metrics and a pipeline inspector.
- Prometheus metrics and a small curated Grafana dashboard.
- An independent reconciliation service that calculates source truth.
- A fault controller for consumer pause, process termination, duplicate delivery, load spikes, and temporary network delay.
- Guided labs, expected observations, reflection prompts, and automated checks.
- A one-command Docker Compose experience on Linux, macOS, and Windows via Docker Desktop.
- Deterministic reset and scenario replay.

This pairing is intentional: it demonstrates most important concepts without making Kafka, Debezium, ClickHouse, and Kubernetes mandatory on day one.

### 4.2 Post-MVP extension tracks

Optional profiles add one concept at a time:

- PostgreSQL logical replication with Debezium using `pgoutput`.
- Redpanda or Kafka in place of Redis Streams.
- ClickHouse in place of PostgreSQL serving tables.
- WebSockets in place of SSE.
- k3s manifests or a Helm chart for replicas, disruption, and resource-limit labs.
- Event-time windows, late data, retention, rollups, and hot/cold storage.

These are profiles, not a requirement to run every tool simultaneously.

### 4.3 Non-goals

- Production-ready e-commerce, billing, or a general-purpose analytics platform.
- Proving production throughput from local benchmark results.
- Claiming end-to-end exactly-once execution.
- Teaching every feature of Kafka, Flink, ClickHouse, Kubernetes, or PostgreSQL replication.
- A fake animation disconnected from actual system state.
- Running untrusted learner code in a shared hosted environment.
- Making all variants share identical internals at the expense of accurately showing their differences.

## 5. The Teaching Domain and Correctness Contract

### 5.1 Source model

The write API owns these source tables:

- `tenants(id, name)`;
- `orders(id, tenant_id, status, currency, created_at, updated_at, version)`;
- `order_lines(id, order_id, product_id, quantity, unit_price)`;
- `outbox_events(id, aggregate_type, aggregate_id, tenant_id, event_type, schema_version, payload, occurred_at, published_at)`; and
- `lab_faults` and `lab_runs` for local control and reproducibility.

An order may be created, have lines changed, transition between statuses, or be deleted. The workload deliberately includes more than inserts so the learner cannot implement an append-only counter and mistake it for a general solution.

### 5.2 Report model

The main report groups by tenant and one-minute bucket and exposes:

- submitted and cancelled order counts;
- gross submitted revenue;
- average submitted order value;
- counts by current status; and
- last source change included in the result.

Currency conversion is out of scope; each seeded tenant uses one currency. Revenue follows an explicit rule: it counts orders in `submitted`, `paid`, or `fulfilled` state and retracts an order that later becomes `cancelled` or is deleted.

### 5.3 Stable identity and event envelope

All streaming variants normalize changes into a versioned envelope:

```json
{
  "event_id": "01J...",
  "schema_version": 1,
  "event_type": "order.changed",
  "tenant_id": "tenant-7",
  "aggregate_id": "order-42",
  "aggregate_version": 3,
  "occurred_at": "2026-09-29T12:34:56.123Z",
  "captured_at": "2026-09-29T12:34:56.180Z",
  "trace_id": "...",
  "before": { "status": "submitted", "total": "18.50" },
  "after": { "status": "cancelled", "total": "18.50" }
}
```

The production code should use integer minor currency units or decimal values, never binary floating point. `event_id` identifies a delivery-independent logical change; `aggregate_version` supports per-order stale-event detection; `trace_id` connects telemetry but is not a deduplication key.

### 5.4 Correctness oracle

The reconciler periodically runs a canonical SQL query against a consistent source snapshot and compares it with the serving result. It publishes:

- mismatch count and a structured diff;
- source snapshot timestamp and high-water mark;
- whether a difference is expected because the pipeline has not reached that high-water mark; and
- time to convergence after the workload stops.

This distinction prevents eventual-but-correct state from being labeled corrupt. A run passes correctness only after the pipeline claims it has consumed the oracle's high-water mark and the values agree.

## 6. Learner Experience

### 6.1 Quick start

The primary path is:

```bash
cp .env.example .env
docker compose up --build
./lab scenario run steady-orders --seed 42
```

The landing page at `http://localhost:3000` shows a readiness checklist, the current architecture, the active lesson, and links to Grafana and generated API documentation. Startup migrations and seed data are idempotent. `./lab reset` removes lab state and volumes only after explicit confirmation; `./lab reset --run <id>` resets a single isolated run.

### 6.2 Integrated workbench

The UI has four synchronized regions:

1. **Report:** the product-facing dashboard a user would see.
2. **Topology:** actual active components and animated movement based on observed events, not timers.
3. **Trace:** a filterable timeline from source transaction through capture, transport, aggregation, serving, and browser receipt.
4. **Health:** freshness, end-to-end latency distribution, stage lag, queue depth, error rate, duplicate/retraction counts, resource use, and correctness status.

A comparison drawer pins two run artifacts side by side. It highlights median and p95 freshness, maximum staleness, convergence time, source query load, CPU/memory, data loss or mismatch, recovery steps, and the number of moving parts.

### 6.3 Guided lab loop

Every lesson uses the same cadence:

1. State a user requirement and predict behavior.
2. Start a named, seeded scenario.
3. Observe a healthy baseline.
4. Apply one controlled pressure or fault.
5. Explain metrics and inspect individual traces.
6. Change a configuration or implementation seam.
7. Replay the identical seed and compare artifacts.
8. Run machine-checkable acceptance assertions.
9. Record which design should be chosen for two contrasting business contexts.

“Solution” means a justified tradeoff, not always enabling the most advanced profile.

### 6.4 Reproducible run artifacts

Each run writes a portable JSON artifact containing:

- application and scenario version;
- architecture profile and sanitized configuration;
- random seed and exact workload schedule;
- fault schedule and component lifecycle events;
- input and output high-water marks;
- summarized metrics and correctness diffs; and
- learner annotations.

Metrics need not be embedded at full Prometheus resolution; artifacts retain enough sampled data and trace references to compare outcomes. Fixed regression seeds cover every scenario in CI.

## 7. Architecture Profiles

### 7.1 Profile A: direct query baseline

```text
browser --poll--> report API --SQL aggregate--> source PostgreSQL
write API -------------------------------------> source PostgreSQL
```

This is the fewest-parts reference. It may be correct for low write volume, a small dataset, and dashboards that tolerate query cost. Learners increase report cardinality and polling clients, inspect source CPU/query latency, add indexes, and confront contention with transactional work.

### 7.2 Profile B: scheduled materialization

```text
write API --> source PostgreSQL --> scheduled refresh --> report table/MV
                                                       ^
browser --------------------------poll--> report API --|
```

This profile separates expensive computation from every browser request. It teaches bounded staleness, refresh overlap, atomic result swaps, incremental refresh limitations, and the thundering herd. It remains a strong endpoint for requirements such as “fresh within five minutes.”

For a safe implementation, refresh into a versioned table and atomically promote a completed version, rather than exposing a partially rewritten table.

### 7.3 Profile C: transactional outbox stream (MVP streaming path)

```text
                                  +--> pipeline observer
write API + source PostgreSQL --> outbox relay --> Redis Streams
       (one transaction)                              |
                                                     v
browser <--SSE-- report API <-- serving tables <-- aggregator
```

The write API inserts the order mutation and outbox row in one transaction. The relay may publish a record more than once if it crashes between broker publication and marking the row published, so the aggregator must be idempotent. Redis consumer groups provide a compact way to expose pending entries, acknowledgement, claiming, retention, and backpressure without the resource cost of Kafka.

The lesson must state that an application outbox is not identical to database-log CDC: it captures only changes made through cooperating writers and adds schema/application work.

### 7.4 Profile D: logical replication CDC

```text
source PostgreSQL WAL --> Debezium --> Redpanda/Kafka --> aggregator
                                                         |
browser <-- push/poll <-- report API <-- serving store <--+
```

This extension captures changes independent of the application write path and exposes snapshot-to-stream handoff, transaction metadata, WAL positions, schema history, connector offsets, and replication slot retention. It requires explicit PostgreSQL settings and a documented recovery path for an invalidated or abandoned slot.

### 7.5 Profile E: analytical serving store

Replace the serving PostgreSQL tables with ClickHouse while holding capture and workload constant. Compare ingestion batching, column-oriented scans, update/delete handling, query flexibility, storage layout, and operational cost. DuckDB can be a lesson-specific local query tool, but it should not be presented as a drop-in concurrent network service.

### 7.6 Delivery variants

- **Polling:** simple stateless requests, predictable recovery, repeated queries and freshness bounded by interval.
- **SSE:** server-to-client push over HTTP, straightforward for report invalidations or snapshots, with `Last-Event-ID` catch-up.
- **WebSockets:** bidirectional subscriptions and richer control, with more state, heartbeats, routing, and reconnect logic.

Push notifications carry a report version or invalidation, not blindly trusted arithmetic deltas. On a version gap, reconnect, or overflow, the browser fetches a fresh snapshot. This makes recovery explainable and bounds client drift.

## 8. Component Responsibilities

### 8.1 Write API

- Validates tenant-scoped order commands.
- Uses database-generated commit state and increments `orders.version` under concurrency.
- In outbox mode, persists the domain change and event atomically.
- Propagates trace context into transactions and outbox metadata.
- Offers a lab-only mutation endpoint protected from normal application routes.

### 8.2 Capture adapters

Adapters emit the common envelope while preserving source-specific metadata. The polling adapter tracks a compound cursor rather than timestamps alone. The outbox relay records publication attempts. The CDC adapter records LSN, transaction ID, operation type, and snapshot status. Adapters never silently coerce an unknown schema version.

### 8.3 Transport

The Redis Streams MVP uses one stream partitioned logically by tenant/order key, a consumer group, explicit acknowledgement, a pending-entry recovery loop, and capped retention that refuses unsafe trimming. The broker extension maps the same concepts to topics, partitions, offsets, consumer groups, and retention.

### 8.4 Aggregator

The aggregator:

- applies changes transactionally to a current-order projection and aggregate buckets;
- records processed event identity or latest aggregate version in the same serving transaction;
- handles before/after values so updates and deletes retract prior contributions;
- rejects unsupported schemas to a visible quarantine queue;
- advances its checkpoint only with the derived-state transaction;
- exposes configurable processing delay and crash checkpoints; and
- supports shadow rebuild into a new projection version.

The MVP should favor correctness and readable code over maximum throughput. A later lesson can batch records and reveal its latency/throughput tradeoff.

### 8.5 Report and live-update API

The API serves a consistent snapshot with `report_version`, `computed_through`, and server time. The SSE endpoint sends monotonic version notifications, heartbeats, and a bounded replay buffer. Tenant authorization is applied independently to the query and subscription paths.

### 8.6 Scenario and fault controller

The controller owns the run state machine and performs named actions through constrained adapters. It can:

- set write rate, tenant skew, update/delete ratio, and report clients;
- pause or slow a consumer;
- stop and restart stateless containers;
- force a relay crash at a named checkpoint;
- publish a duplicate or reorder selected events;
- apply bounded network latency using a dedicated proxy where portable;
- initiate a schema migration; and
- trigger a projection rebuild.

It must not require arbitrary host-level fault commands. Unsupported faults are disabled with an explanation on platforms that lack the needed capability.

### 8.7 Observer and reconciler

The observer combines Prometheus metrics, trace exemplars, component health, and run annotations for the integrated UI. The reconciler remains logically independent of the aggregator and uses source rows rather than stream output, avoiding a shared bug that declares itself correct.

## 9. Measurements and Visual Semantics

### 9.1 Required metrics

Measure at least:

- write throughput and write API latency;
- capture lag (commit to captured);
- transport lag/depth and oldest unprocessed age;
- processing latency and batch size;
- serving query latency;
- browser receipt latency;
- end-to-end commit-to-pixel latency where sampled;
- duplicate, stale, retried, quarantined, and dead-letter event counts;
- oracle mismatch and convergence duration;
- PostgreSQL WAL retained bytes for the CDC profile;
- source database query rate and time;
- per-component CPU and memory; and
- active/polling/push client counts.

Use histograms for latency; averages alone hide tail behavior. All charts show units, aggregation window, and whether the value is sampled.

### 9.2 Three clocks

The trace distinguishes:

- **event time:** when the domain change says it occurred;
- **commit/capture time:** when PostgreSQL committed and capture observed it; and
- **processing/display time:** when downstream computation and the browser handled it.

Containers use the same local host clock in the MVP, but a clock-offset scenario demonstrates why wall-clock subtraction is not a universal distributed latency measurement. Sequence positions and monotonic in-process durations remain separate from wall time.

### 9.3 Honest topology animation

A moving dot represents a sampled, trace-correlated record. Line width can represent rate; color represents state; badges show actual lag or error. If trace sampling omits an event, the UI shows aggregate rate rather than inventing a per-event animation. The topology freezes with a “telemetry stale” indicator if observer data stops.

### 9.4 Complexity scorecard

Alongside performance, each profile records qualitative and countable costs:

- services and persistent volumes;
- configuration and schema artifacts;
- expected failure/recovery procedures;
- idle memory footprint;
- data copies and retention;
- operator skills required; and
- migration and vendor-coupling concerns.

The scorecard is descriptive, not a single magic score.

## 10. Core Curriculum

### Lab 0: Define “near real time”

- Start with a direct query report.
- Turn “instant dashboard” into measurable freshness and availability targets.
- Distinguish report query latency from data freshness.
- Exit check: choose a design for 100 orders/day with a five-minute freshness SLO and explain why streaming may be wasteful.

### Lab 1: Poll the source until it hurts

- Add browser clients and reduce polling intervals.
- Compare indexed and unindexed aggregation plans.
- Observe repeated unchanged responses and source load.
- Exit check: identify the first constraint and propose a lower-complexity mitigation before adding a broker.

### Lab 2: Materialize on a schedule

- Introduce periodic refresh and an atomic version swap.
- Vary refresh interval and computation duration.
- Start many clients on the same boundary to create a thundering herd, then add jitter and cache validation.
- Exit check: quantify the freshness/cost curve and explain overlapping refresh behavior.

### Lab 3: Capture changes with an outbox

- Write order and outbox event in one transaction.
- Contrast this with a deliberately broken “commit, then publish” dual write.
- Crash after the domain commit and show lost publication in the broken variant.
- Exit check: explain what the outbox guarantees and why it does not guarantee one publication.

### Lab 4: At-least-once delivery and idempotent processing

- Crash the relay after publishing but before marking the row.
- Observe duplicate publication and a double count in a broken aggregator.
- Add atomic event/version tracking with derived updates.
- Exit check: replay duplicates and obtain one correct business effect without claiming exactly-once transport.

### Lab 5: Backpressure has to live somewhere

- Slow or pause the aggregator while writes continue.
- Observe outbox backlog, stream pending entries, oldest-event age, and freshness independently.
- Compare bounded producer throttling, load shedding, scaling consumers, and allowing lag.
- Exit check: state which queue can grow, its limit, and the behavior when that limit is reached.

### Lab 6: Ordering, updates, and deletes

- Update one order rapidly and reorder two deliveries.
- Show why global ordering is unnecessarily expensive and why no ordering is unsafe.
- Partition by tenant or order, use aggregate versions, and retract prior contributions.
- Exit check: cancel and delete orders after submission and match the oracle.

### Lab 7: State and restart

- Stop the aggregator between state change and acknowledgement.
- Compare memory-only state with transactional serving state.
- Claim abandoned pending work and resume from a checkpoint.
- Exit check: recovery converges without manual table edits or silent loss.

### Lab 8: Polling versus SSE

- Run the same client count and workload with polling and SSE.
- Interrupt a connection, overflow the replay buffer, and reconnect using `Last-Event-ID`.
- Fall back to a report snapshot on a version gap.
- Exit check: no client remains silently stale; describe when WebSockets add genuine value.

### Lab 9: End-to-end observability

- Select a visible data point and trace it back to its source transaction.
- Correlate p95 end-to-end latency with capture, queue, processing, serving, and browser stages.
- Break telemetry without breaking data processing and recognize the difference.
- Exit check: diagnose an injected slowdown using evidence rather than component restarts.

### Lab 10: Rebuild and replay

- Introduce a corrected aggregate definition.
- Build a shadow projection from the source snapshot or retained event log while live changes continue.
- Catch up to a high-water mark and atomically switch report versions.
- Exit check: preserve availability and document the retention needed for the chosen method.

### Lab 11: Schema evolution

- Add an optional field with an upcaster-compatible event version.
- Deploy consumer-before-producer, then introduce an incompatible rename in a failure demonstration.
- Quarantine unknown versions instead of discarding them.
- Exit check: complete a rolling-compatible change and recover quarantined records.

### Lab 12: Multi-tenancy and hot keys

- Generate one noisy tenant and many small tenants.
- Compare tenant partitioning, fair scheduling, and per-tenant quotas.
- Attempt a cross-tenant report and subscription.
- Exit check: preserve tenant isolation in data, performance, and live fan-out.

## 11. Extension Curriculum

### 11.1 Logical replication and snapshots

Enable `wal_level=logical`, create a publication and managed slot, then:

1. bootstrap a consistent snapshot;
2. continue from the corresponding WAL position;
3. inspect transaction grouping;
4. pause the connector and measure retained WAL;
5. enforce disk guardrails; and
6. intentionally recover from a discarded slot via a new snapshot.

The lesson compares this with triggers and polling. Trigger capture is synchronous and explicit but adds write-path work and can be bypassed or mismanaged. Cursor polling is operationally simple but needs a stable cursor and has latency/query costs. Log CDC is comprehensive but adds database privileges, log retention risk, and connector lifecycle management.

### 11.2 Broker comparison

Run a common contract suite against Redis Streams and Redpanda/Kafka. Compare partitions, consumer groups, retention, replay, pending/offset semantics, per-key ordering, disk footprint, and operational ergonomics. Avoid pretending their APIs or durability models are identical.

### 11.3 Windows and late events

Add an event-time report with tumbling and sliding windows. Delay selected events, define a watermark and allowed lateness, and show correction or finalization policy. Session windows may be a stretch exercise. The UI visibly distinguishes provisional from finalized results.

### 11.4 Serving-store comparison

Replay the same bounded dataset into PostgreSQL and ClickHouse, run reviewed report queries, and inspect ingest/update costs and scan performance. Dataset sizes must be large enough to reveal shape differences but small enough for a laptop. Results always include hardware/profile metadata and are framed as observations, not universal benchmarks.

### 11.5 k3s operations

The k3s track introduces health probes, resource requests/limits, multiple consumers, pod eviction, rolling updates, persistent volumes, and network policy. It should reuse the same scenarios and assertions, proving portability rather than creating a separate curriculum.

## 12. Failure Scenario Catalog

Every scenario declares prerequisites, deterministic setup, actions, expected transient symptoms, eventual invariants, timeout, and cleanup. Initial scenarios include:

| Scenario | Injection | Expected lesson |
| --- | --- | --- |
| `slow-consumer` | Add processing delay | Lag rises before correctness fails |
| `consumer-crash` | Stop after state commit/before ack | Redelivery requires idempotency |
| `relay-crash` | Stop after publish/before outbox mark | Outbox provides at-least-once publication |
| `duplicate-burst` | Republish selected IDs | Dedupe policy and state cost become visible |
| `hot-tenant` | Skew 90% of writes to one tenant | Partition key and fairness trade off |
| `report-herd` | Synchronize polling clients | Jitter/cache/push reduce burst load |
| `stale-event` | Deliver an older order version late | Version policy prevents state regression |
| `unknown-schema` | Emit an unsupported version | Quarantine must be visible and recoverable |
| `browser-gap` | Disconnect beyond replay retention | Snapshot catch-up prevents silent drift |
| `slot-stall` | Pause CDC connector | Retained WAL can threaten source disk |
| `rebuild-live` | Backfill while writes continue | Snapshot/stream handoff needs a high-water mark |

Fault buttons should describe blast radius and recovery before execution. Destructive CDC exercises enforce conservative disk and time limits.

## 13. Technology Recommendation

### 13.1 MVP stack

- **Backend and workers:** Python 3.13 with FastAPI, SQLAlchemy, and Psycopg. Python keeps teaching code approachable; the precise framework is replaceable.
- **Frontend:** React and TypeScript with a lightweight chart library.
- **Source and MVP serving database:** PostgreSQL 17 in separate databases or schemas with distinct credentials.
- **Transport:** Redis Streams.
- **Telemetry:** OpenTelemetry SDK/Collector, Prometheus, and Grafana; trace storage may use Grafana Tempo.
- **Packaging:** Docker Compose with health checks and named profiles.
- **Load/scenarios:** a repository-owned `lab` CLI and async traffic generator rather than an opaque benchmark script.

Versions are starting recommendations and should be pinned to tested patch releases during implementation.

### 13.2 Compose services

The default profile contains `web`, `write-api`, `report-api`, `worker`, `relay`, `postgres`, `redis`, `scenario-controller`, `reconciler`, `otel-collector`, `prometheus`, and `grafana`. Optional `observability-full`, `cdc`, `broker`, and `olap` profiles prevent an introductory run from starting every service.

Containers have health checks, resource hints, read-only filesystems where practical, non-root users, and bounded log rotation. Dependency readiness is checked explicitly; Compose startup order is not treated as readiness.

### 13.3 k3s packaging

After the Compose MVP stabilizes, package application components with Helm or Kustomize and use upstream charts/operators selectively for stateful dependencies. Keep scenario names, APIs, telemetry labels, and run artifacts deployment-neutral. k3s must not become necessary for lessons about basic data semantics.

## 14. Repository Layout

```text
apps/
  web/
  write_api/
  report_api/
workers/
  outbox_relay/
  aggregator/
  reconciler/
lab/
  cli/
  scenarios/
  faults/
  assertions/
contracts/
  event-envelope/
  report-api/
deploy/
  compose/
  k3s/
observability/
  dashboards/
  alerts/
  otel/
curriculum/
  lessons/
  diagrams/
  instructor-guide/
tests/
  unit/
  contract/
  integration/
  system/
```

Architectural profiles use configuration and adapter boundaries, but intentionally broken implementations live behind explicit lab-only toggles. Production/default code paths must never silently select a broken mode.

## 15. API and Control Contracts

### 15.1 Product-facing API

- `POST /v1/orders`, `PATCH /v1/orders/{id}`, and `DELETE /v1/orders/{id}`.
- `GET /v1/reports/orders?tenant_id=...&from=...&to=...` returns values plus freshness metadata.
- `GET /v1/reports/stream?...` provides SSE notifications and heartbeats.
- `GET /health/live` and `GET /health/ready` have different semantics.

### 15.2 Lab control API

The controller uses a separate authenticated, local-only API to start runs, set supported faults, and collect evidence. It cannot edit aggregate results. Each request includes a run ID so concurrent tests do not contaminate one another.

### 15.3 Evidence API

A read-only endpoint exposes stage checkpoints, event dispositions, current profile, and reconciliation results. Automated verification prioritizes public behavior and evidence APIs, then optionally uses read-only database views as an independent diagnostic.

## 16. Testing and Verification

### 16.1 Test layers

- **Unit:** aggregation contribution/retraction, cursor comparison, schema upcasting, and version rules.
- **Property-based:** arbitrary valid sequences of create/update/cancel/delete plus duplicate/reordered delivery converge to the canonical model within stated ordering assumptions.
- **Contract:** event envelopes, report responses, telemetry attributes, and adapter compatibility.
- **Integration:** real PostgreSQL and Redis transactions, relay crash points, pending claims, and migrations.
- **System:** traffic reaches deployed HTTP services, faults act on real processes, UI/API freshness converges, and source truth matches.
- **Browser:** reconnect, version gap, tenant switch, stale telemetry state, and accessible chart alternatives.
- **Deployment:** fresh Compose startup, reset, upgrade, profile switching, and later k3s smoke tests.

### 16.2 Invariants

Common assertions include:

- acknowledged source changes are eventually represented or explicitly quarantined;
- no `(tenant_id, event_id)` is applied more than once;
- older aggregate versions cannot overwrite newer projections;
- report output equals the oracle after processing its high-water mark;
- one tenant cannot query or subscribe to another tenant;
- replay from a recorded artifact produces the same logical result;
- an unsupported schema never advances the safe checkpoint silently; and
- every intentionally broken mode fails the assertion it is meant to teach.

### 16.3 Performance guardrails

CI verifies trends rather than fragile absolute laptop throughput. A release candidate gets a documented reference-machine run with raw artifacts. Examples of useful gates are “source query rate decreases when clients move from one-second polling to SSE” and “a paused consumer creates visible lag and converges after resume without mismatch.”

## 17. Security, Privacy, and Safety

- Use synthetic data only; never request production dumps.
- Bind stateful services to the private container network and expose only required local ports.
- Scope every source, event, projection, cache key, query, and subscription by tenant.
- Use separate least-privilege database roles for writer, capture, aggregator, report, reconciler, and migrations.
- Redact payloads from default logs and traces; demonstrate metadata-first observability.
- Protect the control plane with a generated local token and disable it outside lab mode.
- Cap scenario rate, retained WAL, broker retention, disk usage, and artifact age.
- Document that local container isolation is not a secure sandbox for hostile code.

## 18. Accessibility and Documentation

- Every graph has a table/text equivalent, units, and keyboard-accessible controls.
- Color is never the only signal for health or event disposition.
- Animations respect reduced-motion preferences and can be paused.
- Lessons include architecture diagrams, command transcripts, expected observations, troubleshooting, reflection questions, and a concise production extrapolation.
- An instructor guide supplies timing, discussion prompts, reset instructions, common misconceptions, and optional group roles.
- A glossary defines high-water mark, LSN, watermark, lag, freshness, idempotency, deduplication, projection, and materialization without conflating them.

## 19. Delivery Plan and Exit Gates

### Phase 0: Discovery and contracts

Deliver:

- measured laptop resource budget;
- event, report, telemetry, scenario, and artifact schemas;
- canonical aggregate/oracle query;
- UX wireframes for report, topology, trace, and comparison; and
- spikes proving transaction-to-browser trace correlation and portable fault control.

Exit gate: a reviewed vertical-slice design names the correctness boundaries, supported platforms, and measurements without relying on unbounded infrastructure.

### Phase 1: Naive vertical slice

Deliver direct querying, deterministic writes, the report UI, baseline telemetry, source oracle, and one end-to-end trace.

Exit gate: one command starts cleanly; a learner can run a seeded workload, see a true report, and distinguish query latency from freshness.

### Phase 2: Scheduled baseline curriculum

Deliver scheduled materialization, polling controls, report herd scenario, comparison artifacts, and Labs 0–2.

Exit gate: identical runs compare direct and materialized designs, and automated assertions detect partial/incorrect refreshes.

### Phase 3: Streaming MVP

Deliver transactional outbox, relay, Redis Streams, idempotent aggregator, serving projection, SSE, restart recovery, and Labs 3–8.

Exit gate: crash-point tests pass; duplicate/update/delete scenarios converge; disconnected browsers recover; no component claims exactly-once semantics.

### Phase 4: Observability and operational learning

Deliver curated dashboards, trace inspection, reconciliation diffs, rebuild flow, schema/version lab, noisy-tenant lab, and Labs 9–12.

Exit gate: a learner can localize injected faults and perform rebuild/schema exercises using documented evidence and runbooks.

### Phase 5: Hardening and teaching release

Deliver cross-platform smoke tests, accessibility review, resource caps, instructor guide, fixed regression artifacts, and a 90-minute workshop path plus self-paced track.

Exit gate: a new learner completes the core path without author assistance; reset/retry is reliable; documentation and UI agree with actual behavior.

### Phase 6: Optional advanced profiles

Add logical CDC, broker comparison, analytical store, windowing, and k3s one at a time. Each extension requires its own resource budget, lesson, failure recovery, assertions, and explicit answer to “what capability justified this component?”

## 20. Success Measures

### Learning measures

- At least 80% of pilot learners correctly distinguish freshness from query latency after Lab 2.
- At least 80% can predict the duplicate outcome at the relay crash boundary before completing Lab 4.
- Learners can justify both a simple scheduled design and a streaming design for different requirements.
- Instructors report that observed metrics support, rather than distract from, the lesson objective.

### Product measures

- Fresh-clone-to-first-report takes under 10 minutes on a supported machine after images are available.
- Core Compose profile stays within the documented resource budget.
- Every core scenario is seed-replayable and cleans up without deleting unrelated runs.
- All core failures produce a visible symptom, an actionable explanation, and an automated invariant result.
- The core course can be completed without k3s or optional commercial services.

## 21. Risks and Mitigations

| Risk | Mitigation |
| --- | --- |
| Too many tools obscure concepts | Ship two core profiles; gate advanced services behind optional profiles |
| Laptop limits distort performance | Teach trends, publish hardware metadata, avoid production capacity claims |
| Visualization becomes decorative | Drive it from metrics/traces and visibly mark sampling/staleness |
| Fault injection is flaky | Use named application checkpoints and bounded proxies before host networking tricks |
| Shared bug in aggregator and verifier | Keep canonical SQL/model independent and add property tests |
| Compose and k3s drift | Reuse images, contracts, scenarios, labels, and deployment-neutral readiness checks |
| CDC fills PostgreSQL disk | Cap WAL, alert early, time-limit the scenario, and document slot removal |
| Learners infer “Kafka is always mature” | Require a simplest-adequate-design decision in every comparison |
| Broken modes escape into defaults | Isolate them behind lab-only profiles and assert startup mode visibly |
| Charts overwhelm beginners | Progressive disclosure and a small curated metric set per lesson |

## 22. Decisions to Validate During Phase 0

1. Can commit-to-browser correlation be implemented accurately without invasive database extensions?
2. Is a separate serving PostgreSQL instance worth the extra memory, or are separate roles/databases sufficient for the MVP lesson?
3. Does Redis Streams expose enough replay and partitioning behavior for the core curriculum without teaching misleading Kafka equivalence?
4. Which network fault mechanism behaves consistently across Docker Desktop platforms?
5. What event volume makes backlog and aggregation effects visible in under two minutes on the minimum reference machine?
6. Which charts can be embedded in the workbench, and which should remain in Grafana to avoid duplicating an observability product?
7. Is storing processed event IDs or latest aggregate version the clearer first idempotency implementation, and when should the course compare them?
8. What is the smallest safe snapshot/high-water-mark protocol that supports a convincing live rebuild exercise?

## 23. Definition of Done for the MVP

The MVP is complete when:

- a clean checkout starts through Docker Compose and reports readiness;
- baseline and streaming modes run the same versioned, seeded workloads;
- learners can see source commit, capture, queue, aggregation, serving, and browser milestones;
- create, update, cancellation, deletion, duplicate, crash, backlog, and reconnect cases have automated assertions;
- the independent oracle detects a deliberately corrupted projection;
- a run can be exported, reset, and replayed;
- the UI communicates freshness, correctness, and telemetry gaps accessibly;
- Labs 0–8 have tested instructions, expected observations, and tradeoff prompts;
- CI exercises real containers and all fixed scenario seeds; and
- documentation clearly states local-scale limitations and avoids exactly-once claims.

## 24. First Implementation Backlog

1. Write architecture decision records for the domain, event identity, canonical aggregate, and two MVP profiles.
2. Define JSON Schema/OpenAPI contracts and golden examples for reports, events, runs, and evidence.
3. Implement source migrations, seed generator, canonical SQL oracle, and property-model reference.
4. Build the write API and deterministic steady/burst/update-delete workloads.
5. Build the direct report path and initial report/freshness UI.
6. Instrument one end-to-end transaction and verify clock/trace assumptions.
7. Build scheduled materialization and atomic result promotion.
8. Implement the outbox relay with explicit duplicate-producing crash points.
9. Implement the Redis consumer and projection transaction with dedupe/version handling.
10. Add SSE report-version notifications and snapshot recovery.
11. Implement run orchestration, artifact export, assertions, and safe reset.
12. Add Prometheus/Grafana views and the integrated topology/trace panels.
13. Author Labs 0–8 alongside the features, not after implementation.
14. Run learner pilots before beginning Debezium, Kafka/Redpanda, ClickHouse, or k3s extensions.

The final sequencing rule is simple: no optional infrastructure is added until the existing lab makes its motivating problem observable and the new profile can be compared against the same workload and correctness contract.

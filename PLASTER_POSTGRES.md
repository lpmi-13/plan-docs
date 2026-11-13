# Plaster Postgres — plan for faster, evidence-based tuning experiments

## 1. Purpose and promise

Plaster Postgres helps a team answer a question such as: **“If we lower the autovacuum threshold for our busiest table, will vacuum keep up with writes without hurting request latency?”** The team exports a bounded set of production telemetry into local structured files. Plaster checks the bundle, proposes a representative experiment, runs the current and proposed settings against comparable database states and workloads in an isolated iximiuz Labs playground, and produces an evidence-backed comparison.

The goal is to **surface the mechanism and likely trade-offs faster than changing production and waiting days or weeks**. An experiment ends when it has reached the relevant events and observation checkpoints; one hour is a useful target for some cases, not a fixed rule. Production-like staging traffic is not assumed to exist.

The system should distinguish three kinds of claim:

1. **Observed in the lab:** a direct measurement from a particular playground, workload, and database state.
2. **Supported directional inference:** the same mechanism is likely relevant in production because the lab reproduced the key workload and state conditions.
3. **Production forecast:** a numerical estimate with an explicit calibration model, uncertainty range, and stated assumptions. The MVP need not make this claim.

Resource differences and compressed time prevent a universal conversion from lab throughput or latency to production throughput or latency. Plaster should return **“inconclusive”** when fidelity checks fail, even if the experiment produced attractive numbers.

## 2. First user journey

1. **State the question.** The team selects a goal (`autovacuum keeps up`, `reduce read latency`, `reduce WAL pressure`, etc.), a scope (database, tables, query families), one or more candidate parameters, and guardrails such as maximum acceptable p95 latency or extra I/O.
2. **Get a collection checklist.** Plaster explains which production measurements are required, optional, or unavailable for that question; the collection window should cover normal and peak periods, commonly at least a week for weekly traffic patterns. It supplies read-only SQL and templates for metrics and application traces. The team controls collection and export.
3. **Submit local files.** A manifest points to time series, query or request samples, database metadata, table statistics, configuration, hardware and storage context, and collection provenance. Raw production rows and SQL bind values are not required by default.
4. **Review a proposed test.** Plaster validates the bundle, reports gaps, and creates a declarative scenario: seed state, operation mix, key distributions, arrival/concurrency profile, candidate setting, accelerated phases, metrics, expected mechanisms, and stop conditions. An LLM may draft or explain this scenario, but a deterministic validator checks it before execution. The team can edit and approve the scenario locally.
5. **Run paired experiments.** The runner creates a private, disposable iximiuz Labs playground, runs baseline and candidate from equivalent starting states with the same workload seed, collects database and load-generator telemetry, and repeats where variance warrants it.
6. **Read the report.** Plaster compares predicted events with observed events, shows both benefits and costs, flags fidelity gaps and noisy measures, and recommends a bounded production validation step only when the evidence supports one.

For a team that does not yet know what to tune, the same journey begins with a **diagnostic overview**: observed symptoms, relevant candidate settings, why each might matter, telemetry needed to distinguish them, and an experimentability rating. A suggestion is a hypothesis, not an automatic configuration recommendation.

## 3. Scope

### MVP

- Local CLI and versioned file formats for telemetry bundles, scenario plans, experiment results, and reports.
- One supported PostgreSQL major version per experiment, matched to the team's production major version where possible; record exact server version, extensions, configuration sources, schema, and collation.
- A rules-based telemetry checklist and deterministic bundle validator.
- An LLM-assisted **optional** plan-drafting step for mapping imperfect telemetry into a structured scenario. It must cite input fields, expose assumptions and missing data, and emit data only, not executable shell or SQL to run unchecked.
- One complete experiment family: autovacuum vacuum/analyze trigger and cadence for selected tables, including baseline and a per-table candidate threshold or scale-factor change.
- A traffic generator using application-level operation templates and realistic write/read mix, concurrency, skew, transaction boundaries, and periodic bursts.
- A private iximiuz Labs playground launched through `labctl`, with bounded CPU, RAM, disk, runtime, data volume, and cleanup.
- A comparison report with request latency distribution, throughput, errors, vacuum behavior, table health, WAL and I/O cost, baseline-fidelity checks, and an explicit conclusion of `supported`, `rejected`, or `inconclusive`.

### Later experiment families

| Family | Example settings | Distinguishing telemetry and limits |
| --- | --- | --- |
| Query planning and memory | `work_mem`, per-query planner settings, statistics targets | Query templates, row-count distributions, plans, temp-file use, concurrent memory demand. A small data set can change the plan, so require plan-shape checks. |
| Checkpoints and WAL | `checkpoint_timeout`, `max_wal_size`, `wal_compression` | WAL rate, checkpoints, write bursts, storage latency and throughput. Time and I/O device behavior matter. |
| Connections | `max_connections`, pooling policy | Active versus idle sessions, churn, transaction duration, memory and queueing. Include a pooler if production uses one. |
| Vacuum worker capacity/cost | `autovacuum_max_workers`, cost settings | Number and size of simultaneously eligible relations, vacuum throughput, I/O contention. A single small table is insufficient. |
| Cache and query access | `shared_buffers`, `effective_cache_size` | Working-set size, hit ratios, host page cache, representative read distribution. These are especially sensitive to resource mismatch. |

Start with a small catalog of **question → mechanism → required evidence → safe candidate ranges → experiment recipe**. Do not present a ranked list of generic “best settings.”

### Non-goals for the MVP

- Automatically applying changes to production or staging.
- Replaying raw SQL logs verbatim, including writes, against an arbitrary schema.
- Claiming a week of wall-clock behavior can always be simulated in an hour.
- Predicting exact production p99 latency or capacity from a smaller playground without separate calibration.
- Testing transaction-ID wraparound safety or retention over long periods through a short synthetic run alone.

## 4. Structured telemetry contract

A bundle is a directory or archive with a `bundle.yaml` manifest and referenced CSV, JSONL, or Parquet files. Every file declares its schema version, time range and timezone, sample rate or aggregation interval, units, collection source, and any redaction or sampling method. Counts are either raw counts or explicitly weighted estimates. The validator rejects inconsistent timestamps, missing units, ambiguous denominators, and unknown PostgreSQL versions before plan generation.

Suggested layout:

```text
telemetry/
  bundle.yaml                 # question, environment, file inventory, provenance
  configuration.json          # effective settings, source, per-table reloptions
  schema.json                 # tables, indexes, partitions, constraints, sizes
  table_stats.csv             # sampled per-table counters and maintenance history
  traffic_1m.csv              # arrivals and outcomes by operation/query family
  traffic_peak_1s.csv         # optional short peak windows, concurrency/queueing
  operation_templates.yaml    # reviewed, parameterized replay operations
  operations.jsonl            # sanitized sampled request/transaction traces
  query_stats.csv             # pg_stat_statements deltas, if available
  database_metrics.csv        # WAL, I/O, CPU, memory, connections, waits
  events.jsonl                # jobs, deployments, incidents, manual VACUUM, resets
```

The manifest should include `experiment_question`, `production_postgres_version`, `window_start/end`, `timezone`, `collection_coverage`, `privacy_classification`, `sampling_method`, `files` with hashes, and the desired guardrails. It should not include credentials. Keep the original submitted bundle immutable and hash it in every derived artifact.

### Collection checklist generated from the question

| Layer | Minimum useful evidence | What it resolves |
| --- | --- | --- |
| Traffic | Timestamped request/query-family counts; reads versus inserts, updates, deletes; errors; p50/p95/p99 where available; weekday and peak periods | Volume, mix, bursts, and user-visible outcome. RPS alone does not capture write churn or concurrent load. |
| Execution shape | Sanitized SQL or application operation templates; transaction boundaries and duration; concurrency or pool occupancy; representative key/tenant skew; batch jobs and retries | Preserves causally important behavior that aggregate query counts omit. |
| Database state | PostgreSQL version; schema/indexes/partitions; row-count and size estimates; table age, dead/live tuples, insert/update/delete rates, HOT update rate, last vacuum/analyze, relation growth; long transactions and replication slots | Reproduces the starting conditions and checks whether vacuum can actually reclaim tuples. |
| Configuration | Effective `pg_settings` values and sources; table storage parameters; extension list; connection pool settings | Identifies the true baseline and setting precedence. |
| Capacity | vCPU/RAM, database and WAL storage class, provisioned IOPS/throughput if known, observed CPU/IOPS/latency and free space | Calibrates what the smaller playground can reproduce and what it cannot. |
| Maintenance | Autovacuum logs or events, vacuum progress samples, scheduled jobs, checkpoints, statistics resets | Links outcomes to triggers rather than inferring from latency alone. |

For autovacuum, the mandatory fields are the target relation's approximate row count and size, mutation rates over time, effective threshold/scale-factor settings including per-table overrides, `n_dead_tup`/`n_mod_since_analyze` samples, last vacuum/analyze and count deltas, plus foreground traffic and resource metrics. Production logging can be sampled or temporarily enabled with team review; the bundle records any resulting monitoring overhead. PostgreSQL's cumulative statistics can lag and reset, so use **deltas with reset detection**, not isolated counter values. `pg_stat_statements` gives normalized query aggregates; it does not reconstruct request ordering, bind-value distributions, or latency quantiles, so application traces or templates fill those gaps. [PostgreSQL monitoring statistics](https://www.postgresql.org/docs/current/monitoring-stats.html), [pg_stat_statements](https://www.postgresql.org/docs/current/pgstatstatements.html), [logging settings](https://www.postgresql.org/docs/current/runtime-config-logging.html).

### What an operation trace looks like

An operation trace is **one sampled execution**, usually an application request or background job with its database calls in order. It is distinct from the aggregate traffic series that says how often such executions occurred. A trace may come from OpenTelemetry/APM spans, application or ORM logs, or a purpose-built exporter; the bundle normalizes these sources into one format. It need not contain literal SQL values or customer data. OpenTelemetry database spans can carry operation name, query summary/text, timing and error status, but transaction boundaries and useful parameter *distributions* may require application instrumentation. [OpenTelemetry traces](https://opentelemetry.io/docs/concepts/signals/traces/), [database span conventions](https://opentelemetry.io/docs/specs/semconv/db/database-spans/).

One sanitized JSONL record might be:

```json
{"started_at":"2026-11-05T14:23:01.122Z","operation":"orders.update_status","duration_ms":44,"outcome":"ok","sample_probability":0.05,"request_id":"sample-001","parameter_shape":{"tenant_activity_bucket":"high","order_age_bucket":"1-7d"},"transaction":{"duration_ms":28,"statements":[{"template_id":"orders_by_id","kind":"select","relation":"public.orders","duration_ms":6,"rows_returned":1},{"template_id":"update_order_status","kind":"update","relation":"public.orders","duration_ms":12,"rows_affected":1}]}}
```

This record says that a sampled request read and then updated one order inside a transaction, how long the steps took, and which *classes* of keys it touched. It does not reveal an order ID, tenant ID, or bind value. `template_id` points to a separately reviewed, parameterized SQL or application-operation template; `sample_probability` lets analysis account for the sample design. Fields such as `rows_affected` are optional when the source cannot supply them, but missing fields must be declared. The bundle also needs aggregate counts by time bucket, because a handful of traces cannot establish the production arrival rate or weekday pattern.

### How a team checks whether it can provide traces

Inspect a small export from the telemetry already in use, then answer these questions for both a quiet period and a peak period:

1. Are there per-request, per-job, or per-query records with start time, duration, outcome, and a documented sampling rate? Can the team export them to JSONL/CSV/Parquet?
2. Can a database call be linked to its parent request or transaction, with statement order and transaction duration? A trace ID/request ID helps; explicit transaction spans or markers may need to be added.
3. Can records identify a low-cardinality operation or parameterized SQL template and the affected table? Can they distinguish reads, inserts, updates, deletes, and batch sizes?
4. Is there a safe way to describe skew without exporting values: for example, tenant-size buckets, hot-key frequency, row-age buckets, or anonymized cardinality distributions?
5. Does sampling include ordinary successful traffic as well as slow and failed calls? Is the sampling probability known, and are scheduled jobs and peaks represented?
6. Are the resulting files permitted to be used in a private playground or optional LLM after redaction?

The answer determines the supported input tier:

| Available source | What Plaster can use | Main gap |
| --- | --- | --- |
| Request traces with child database spans and transaction markers | Operation order, concurrency, transaction duration, query mix, sampled latencies | Parameter and key distributions still need safe bucketing or fixture knowledge. |
| Application/ORM logs with correlation IDs | Often enough to reconstruct representative operation recipes | Transaction boundaries and sampling coverage may need work. |
| Sampled PostgreSQL statement logs with session IDs and timestamps | Query timing and partial session/transaction sequences | Usually lacks application context; logs can expose bind values unless configured and scrubbed. |
| `pg_stat_statements` plus time-series metrics | Aggregate normalized query mix and rates | Cannot recover request ordering, transaction shape, key skew, or tail latency; use explicit assumptions and lower confidence. |

For the autovacuum MVP, full distributed tracing is **helpful but not a prerequisite**. A team with table-level mutation rates, read/write time series, effective settings, database state, and a reviewed workload recipe can run a useful directional test. Plaster should state which traffic properties were assumed rather than measured. PostgreSQL supports sampled statement-duration logging and transaction sampling, but enabling them has operational and privacy implications; a team should use its existing telemetry first and review any new logging configuration. [PostgreSQL logging settings](https://www.postgresql.org/docs/current/runtime-config-logging.html).

### Data minimization

The default bundle contains aggregate series and sanitized templates, with synthetic parameter distributions instead of customer rows, bind values, tokens, or connection strings. Provide a local lint/redaction step that scans files before upload to a playground or optional LLM. Record whether source data was sampled, transformed, or omitted. For datasets whose behavior depends on actual row distribution, permit an approved anonymized fixture and make its privacy handling explicit. The playground is private and disposable, but that alone is not a reason to upload raw production data.

## 5. From bundle to executable experiment

### Planner output

Produce an editable `experiment.yaml` with these sections:

```yaml
schema_version: 1
question: "Will a lower vacuum scale factor help orders without a latency regression?"
bundle_hash: "..."
target:
  postgres_major: 18
  relations: ["public.orders"]
baseline: {source: "effective-production-settings"}
candidate:
  relation_options: {"public.orders": {autovacuum_vacuum_scale_factor: 0.02}}
state_model: {fixture: "synthetic-orders-v1", scale: "calibrated"}
workload:
  operations: "operation_templates.yaml"
  profile: "representative-plus-peak"
  seed: 314159
  phases: ["warmup", "mutate", "read-under-vacuum", "recovery"]
checkpoints: ["after-equal-mutations", "after-vacuum-opportunity", "after-recovery"]
metrics: ["request_latency", "errors", "vacuum_events", "dead_tuples", "wal_bytes", "io"]
guardrails: {p95_latency_regression_pct: 10, error_rate_max: 0.001}
limits: {max_runtime_minutes: 120, max_disk_gb: 10}
```

The example values are illustrative, not universal defaults. `max_runtime_minutes` bounds a run; the planner estimates a duration from the trigger and workload model. The scenario must also carry an `assumptions` list, `unreproduced_conditions`, `expected_events` with observable metric predicates, and an `abort_conditions` list. A plan is runnable only after schema, semantic, privacy, resource and safety validation.

### Deterministic core, optional LLM

1. A rule engine chooses the checklist and computes known quantities: write volume per table, eligible-vacuum thresholds, expected event count, read/write mix, peak-to-average ratios, and whether enough source data exists.
2. A planner converts telemetry into parameterized operation templates and a time-varying traffic schedule. It preserves **mix, key skew, transaction size, concurrency and burst order**, not only total queries.
3. An LLM may infer mappings between application operations and tables, identify missing workload dimensions, propose a fixture recipe, and draft hypotheses or report prose. Give it only sanitized structured inputs. Require each assertion to name supporting bundle fields; label unsupported assertions as assumptions. Never let generated free-form code or settings execute directly.
4. A deterministic verifier checks parameter names/ranges for the pinned PostgreSQL version, relation existence, phase reachability, expected-event predicates, safe resource limits, and whether the workload exercises the setting. A human reviews the candidate and assumptions before running.

This allows the MVP to use an LLM where interpretation saves time while keeping the experiment contract and result calculation auditable. Later, replace LLM mapping with curated adapters and recipes as common patterns emerge.

## 6. Time compression and fidelity strategy

Compression should be **mechanism-specific**. For autovacuum trigger experiments, the key axis is mutations relative to relation size and the vacuum threshold. Drive enough representative mutations and reads to cross candidate and baseline checkpoints while preserving the order of peak and quiet periods. Compare equal logical work and equivalent starting states. For real-time scheduler effects such as `autovacuum_naptime`, preserve or explicitly model elapsed-time opportunities; do not scale every timer with the traffic rate. For I/O-limited vacuum cost, changing write arrival rate changes contention, so calibrate against resource pressure instead of blindly applying a week-to-hour multiplier.

The autovacuum trigger depends on a threshold formed from a base count plus a scale factor times estimated table size, with version-specific caps and separate insert/analyze rules. The planner reads the selected version's rules and per-table overrides; it does not hard-code one formula for every PostgreSQL version. Vacuum also affects dead-tuple reuse, the visibility map, planner statistics, and wraparound protection; a single metric such as table size cannot capture the whole effect. [PostgreSQL vacuum configuration](https://www.postgresql.org/docs/current/runtime-config-vacuum.html), [routine vacuuming](https://www.postgresql.org/docs/current/routine-vacuuming.html).

Use the following hierarchy of fidelity:

1. **Behavioral match:** operation semantics, read/write split, transaction length, key skew, burst sequence, and foreground concurrency.
2. **State match:** schema, indexes, relevant row distributions, table-to-index sizes, dead-tuple and visibility conditions, statistics age, and concurrent long transactions where relevant.
3. **Pressure match:** ratios such as writes per live row, dirty data relative to working set, active sessions per vCPU, observed disk utilization, and WAL bytes per transaction.
4. **Absolute match:** production-scale data, device latency, and exact throughput, only when the environment supports them.

A small lab may satisfy the first three for a question without satisfying the fourth. Record the difference and make claims at the level supported. The runner should first establish that the **baseline lab** reproduces the targeted production symptom or mechanism. If it does not, candidate results are exploratory and the report is inconclusive. Physical snapshots may preserve table layout and bloat but restored cumulative statistics can reset; verify all starting counters and trigger conditions rather than assuming a snapshot is sufficient. [PostgreSQL statistics persistence and reset behavior](https://www.postgresql.org/docs/current/monitoring-stats.html).

Some phenomena resist acceleration: long-lived transactions, replication feedback, multi-day access patterns, storage degradation, and rare tail events. Model them as explicit scenarios where feasible, or mark them outside the experiment's evidence boundary.

### Roadmap for cross-environment estimates

After the MVP demonstrates reliable directional comparisons, add a calibration layer rather than a single “lab-to-production” multiplier. Collect a production baseline signature (workload mix, logical work per table, request latency distributions, CPU and I/O pressure, WAL rate, vacuum cadence) and tune the lab fixture and arrival schedule until the same *mechanism* appears. Run the baseline and candidate at several lab resource limits and load levels to learn whether the relative effect is stable or changes near saturation. Fit separate response curves for foreground latency, vacuum lag, WAL, and storage growth; attach uncertainty from repeated runs and mismatched dimensions. Validate any numerical forecast against a limited, monitored production canary before treating the model as predictive. If the curves change direction or the production baseline cannot be matched, report only the lab observation and the unresolved difference.

## 7. Autovacuum MVP experiment

**Question:** A high-churn `orders` table accumulates dead tuples over several days. Would a lower per-table `autovacuum_vacuum_scale_factor` make cleanup more timely without unacceptable foreground latency or I/O cost?

**Preparation:** Collect at least one normal-plus-peak production window, confirm the relevant table and index sizes and update/delete rates, and identify any long-running transactions or replication conditions that could prevent cleanup. Build a scaled fixture with matching schema, indexes, update skew, HOT/non-HOT mix where possible, and a known starting state. If a synthetic fixture changes query plans or update behavior materially, stop and improve it.

**Paired run:** Start baseline and candidate from separate copies of the same fixture. Warm both to a comparable cache and statistics state. Run the same seeded schedule: representative background load, mutation phases sufficient to create the expected dead tuples, a foreground read phase during likely vacuum activity, and a quiet recovery phase. Repeat in alternating order if runtime or host noise is material. The candidate changes one per-table option; all other settings and workload inputs are recorded and held constant.

**Expected observations (hypotheses, not guaranteed):** The candidate reaches an autovacuum trigger earlier in logical write volume; dead-tuple estimates and relation growth may improve; autovacuum CPU/I/O and foreground latency may increase during cleanup. The report checks each prediction against event logs and measured series. A vacuum that starts but cannot reclaim space because of an old snapshot is a different diagnosis from a threshold that is too high.

**Measure:** Request rate, errors, latency histogram/percentiles by operation; `pg_stat_user_tables` deltas and last-autovacuum times; `pg_stat_progress_vacuum` samples; autovacuum logs; table and index bytes; WAL bytes; `pg_stat_io` where supported; host CPU, memory, and I/O pressure. Optional expensive checks (for example exact tuple/bloat inspection) run only at checkpoints on the disposable fixture. Keep the load generator's own CPU and queueing visible so client saturation is not mistaken for PostgreSQL behavior. [PostgreSQL monitoring statistics](https://www.postgresql.org/docs/current/monitoring-stats.html).

**Decision:** A supported result requires a demonstrated difference in vacuum behavior, no breached user guardrail under the tested load, acceptable repeatability, and a baseline that reproduced the relevant production mechanism. A candidate may be rejected for read-latency or I/O cost even if it cleans up sooner. If the trigger was not reached, the fixture/plans diverged, or the playground saturated in a production-incomparable way, say inconclusive.

Do not use this run alone to justify disabling autovacuum, raising wraparound safety limits, or changing cluster-wide worker capacity. Those have different failure modes and need their own experiments and safeguards.

## 8. Runner and playground design

Use `labctl` to create and start a **private custom playground** from a checked-in manifest. The MVP can start with one VM if PostgreSQL, workload generator, and collector fit comfortably; otherwise split generator and database into two machines to avoid client CPU interference. Pin the root filesystem/image, PostgreSQL package version, schema fixture generator, and runner build. The current `labctl` CLI supports `playground create`, `start`, `status`, `stop`, `destroy`, and manifest inspection; the exact manifest and account resource limits should be checked during the capability spike. [iximiuz custom playground guide](https://labs.iximiuz.com/docs/custom-playgrounds/first-custom-playground), [building blocks](https://labs.iximiuz.com/docs/custom-playgrounds/building-blocks).

The controller should implement: provision → preflight → seed → snapshot/copy state → baseline/candidate trials → collect → compare → destroy. Preflight checks PostgreSQL version, effective settings, available disk, metrics privileges, fixture checksum, telemetry timestamps, and the runner's maximum load. Each trial gets an immutable run ID, scenario hash, seed, start state ID, environment fingerprint, metric files, and logs. Enforce runtime and disk quotas; stop traffic on error or guardrail breach; always clean up the remote instance after collecting artifacts. Keep production credentials outside the runner entirely.

Use simple, inspectable components initially: a CLI (Python or Go), a schema/fixture builder, an operation-template load driver, a PostgreSQL/host collector, and a report generator. Prometheus/Grafana can be added for interactive viewing, but raw timestamped artifacts and the comparison report are the source of truth.

Proposed repository structure for implementation:

```text
plaster-postgres/
  README.md                    # user journey, limits, quick start
  docs/collection/             # checklists, read-only SQL, privacy guidance
  schemas/                     # bundle, experiment, result JSON Schemas
  examples/autovacuum/         # synthetic bundle and expected report
  catalog/                     # setting recipes and version capabilities
  planner/                     # rules, optional LLM adapter, validation
  workload/                    # fixtures, operation templates, scheduler
  runner/                      # labctl integration and trial orchestration
  collector/                   # PostgreSQL, OS, load-driver metrics
  analysis/                    # fidelity checks, paired comparison, report
  playground/                  # iximiuz manifest and image build
  tests/                       # fixture/replay and report contract tests
```

CLI shape: `plaster checklist autovacuum`, `plaster validate telemetry/`, `plaster plan telemetry/ --out experiment.yaml`, `plaster run experiment.yaml`, and `plaster report runs/<id>/`. `plan` should work offline; `run` is the only phase that needs iximiuz access.

## 9. Report contract

Every report should answer, in this order:

1. **Did the experiment represent the question?** Show source window, coverage, read/write split, key-skew and concurrency match, starting-state match, baseline symptom match, and environment differences.
2. **What did the plan predict?** List each expected event, its metric predicate and checkpoint, and assumptions.
3. **What happened?** Plot/describe baseline and candidate on both wall-clock time and logical-work axes (transactions or mutations per target table); annotate vacuum starts/completions, bursts, and guardrail breaches.
4. **How repeatable is the difference?** Show trial count, paired deltas, run-to-run spread, and whether the effect exceeds noise. Do not turn a single noisy run into a precise percentage claim.
5. **What is the decision?** `supported`, `rejected`, or `inconclusive`, plus concrete reason, cost, remaining unknowns, and the smallest production validation step if justified.

Include the underlying bundle/scenario/environment hashes and machine-readable result data so another team can rerun or challenge the conclusion. The report must keep **expected** and **observed** values separate; LLM-generated explanations can summarize, but measured values come from the collector and comparison code.

## 10. Build sequence and acceptance criteria

1. **Contract spike:** Define bundle, scenario and result schemas; create a synthetic one-week telemetry bundle with a daily cycle, a peak write window, and an autovacuum question. Prove `validate` catches missing units, reset counters, and conflicting versions.
2. **Local experiment prototype:** Run a pinned PostgreSQL instance, fixture and load driver locally. Demonstrate that the workload reaches the intended autovacuum condition and that baseline/candidate start from equivalent states. Add event and latency collection before remote orchestration.
3. **Remote capability spike:** Create a disposable private iximiuz Labs playground with `labctl`; verify available CPU/RAM/disk, PostgreSQL startup, fixture load time, a bounded paired run, telemetry retrieval, and cleanup. Record actual resource limits rather than assuming production-like capacity.
4. **Planner and report:** Implement rules-based checklist and scenario generation, then optional LLM drafting behind validation. Add baseline-fidelity checks and a report that separates predictions, observations, inference and uncertainty.
5. **End-to-end pilot:** Give a team the collection checklist and ask them to supply a sanitized local bundle. Have another engineer review the generated plan without relying on the LLM transcript. Run at least two paired trials or explain why variation is negligible. Check that the report answers the original tuning question and correctly returns inconclusive for a deliberately mismatched baseline.

MVP is complete when a user can go from a structured telemetry bundle to a reviewed autovacuum experiment and a reproducible comparison in a bounded remote playground, with no manual rewriting of the workload between baseline and candidate. The result need not recommend a change; an honest inconclusive result with a clear missing-data request is valid product behavior.

## 11. Decisions to discuss with an early pilot team

1. Which production PostgreSQL major version, managed-service restrictions, and metrics sources should the first adapter support?
2. Can the team export sanitized application operation traces with transaction boundaries and parameter *distributions*, or only database-level aggregates? This determines how realistic replay can be.
3. Is the first target a known high-churn table, or should the MVP spend more effort on finding candidate tables and settings?
4. Which outcome is the team actually optimizing: foreground latency, bloat, vacuum lag, storage cost, operational risk, or a weighted combination?
5. What data may leave the team's environment for iximiuz Labs or an optional LLM, and what must remain local? The default design assumes sanitized fixtures and aggregates.


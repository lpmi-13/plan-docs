# AUUS — Self-Hosted AWS Simulation for Scenario-Based Learning — Plan

## 1. Objective

Build **AUUS**, a self-hosted environment that looks and behaves enough like AWS that
learners can work through realistic operational scenarios end to end: in a web console,
from the AWS CLI, from SDKs, and from IaC tools. It should need no AWS account, cost
nothing per learner-hour, and be safe to break.

The console is built from the Cloudscape Design System components in
`/home/adam/projects/components` (the open-source React library AWS uses for its own
console). Backing services run in **Docker Compose** or **k3s**. Some come from
**Floci**, the MIT-licensed local AWS emulator that listens on port 4566. Others are
heavier services we write ourselves where the scenarios need behavior that emulators
don't provide.

The environment is built around the scenarios. We don't try to cover the whole AWS API.
We cover the parts a given scenario touches, and we make those parts realistic: state
transitions, delays, failure modes, and the side effects a learner would see in
production.

Three scenarios anchor the design:

1. **RDS Blue/Green deployment.** A learner upgrades a PostgreSQL major version (or
   changes a parameter group) with a Blue/Green deployment. They run into the real
   guardrails: logical replication prerequisites, DDL that doesn't replicate,
   switchover timeouts, and client DNS caching.
2. **CloudFront signed URLs in front of a web application.** A learner configures a
   distribution, public keys, key groups, trusted signers on a cache behavior, and
   Origin Access Control to a private origin. They then prove that unsigned and expired
   requests fail while signed ones succeed and are cached.
3. **Migrating image delivery from S3 pre-signed URLs to CloudFront signed URLs.** A
   (simulated) mobile app backend moves image downloads from per-request S3 pre-signed
   URLs, which can't be cached usefully, to CloudFront signed URLs or cookies with a
   deliberate cache-key and expiry strategy. The learner uses metrics to show the
   before/after difference in hit ratio, origin load, and (simulated) cost. The mobile
   app itself is out of scope. A device simulator stands in for it.

## 2. What "authentic" means here

We have limited effort, so it goes into the kinds of authenticity that matter for
learning:

| Dimension | Why it matters | Target |
|---|---|---|
| **API shape** (request/response, error codes, ARNs, pagination) | Learners' CLI/SDK/Terraform skills must transfer | High for services in a scenario, none for everything else |
| **Lifecycle & timing** (`creating` → `available`, `InProgress` → `Deployed`) | Real ops work is mostly waiting and polling | Realistic sequence, compressed time (configurable) |
| **Data-plane behavior** (does replication actually happen? does the cache actually hit?) | This is where the real lessons are | Real: real Postgres, a real HTTP cache, real signatures |
| **Failure modes & guardrails** | Scenarios are about what goes wrong | Hand-picked, documented, deterministic where possible |
| **Console look & flow** | Muscle memory for the real console | Close in layout and wording; no AWS logos or trademarks |
| **Billing/metrics** | Scenario 3 is about cost and cache efficiency | Simulated CloudWatch metrics and a cost estimate |

Explicit non-goals: full API coverage, multi-region replication (beyond maybe a second
"region" label), real global anycast, production-grade security of the simulator itself,
and running an actual mobile app.

## 3. High-level architecture

```
                    ┌──────────────────────────────────────────────────────────┐
  Learner browser ──▶  AUUS Console (React + Cloudscape)                        │
                    │    uses AWS SDK v3 in-browser ──┐                         │
  Learner terminal ─▶  aws CLI / SDK / Terraform ─────┤  AWS_ENDPOINT_URL=...   │
                    │                                 ▼                         │
                    │           ┌──────────────── API Gateway / Router ───────┐ │
                    │           │ SigV4 verify · IAM eval (opt) · CORS ·      │ │
                    │           │ route by service (Host / X-Amz-Target /      │ │
                    │           │ credential scope) · audit log ("CloudTrail")│ │
                    │           └───┬─────────────┬──────────────┬────────────┘ │
                    │               ▼             ▼              ▼              │
                    │         Floci (4566)   rds-sim (custom)  cloudfront-sim   │
                    │    S3, IAM, STS, KMS,  control plane +   control plane    │
                    │    Secrets, CW, R53*,  B/G orchestrator  (custom or Floci)│
                    │    ACM, Lambda …          │                   │           │
                    │                           ▼                   ▼           │
                    │                  Postgres/MySQL      Edge proxies (POPs)  │
                    │                  containers/pods     + regional cache     │
                    │                  + endpoint DNS/TCP  → origins (S3/app)   │
                    │                                                           │
                    │  Scenario Engine: seed · inject faults · check · grade    │
                    │  Metrics: Prometheus → "CloudWatch" API + console charts  │
                    │  CoreDNS: *.rds.<suffix>, *.cloudfront.<suffix>, etc.     │
                    └──────────────────────────────────────────────────────────┘
```

The central idea is a **single AWS-compatible endpoint** (the router) in front of a mix of
backends. Clients never need to know which backend implements which service, so any
service can be moved from Floci to a custom implementation (or back) without affecting
learners.

---

## 4. Decision areas and options

Each area below lists options, trade-offs, and a recommendation. Section 10 gathers
them into a table.

### D1. Emulation strategy: Floci vs. custom vs. hybrid

What Floci provides, from its per-service docs at `floci.io/floci/services/<svc>/`
(checked 2026-09; still needs hands-on spikes):
- It speaks the real AWS wire protocols on :4566, has more than 100 services, and is
  MIT licensed.
- **RDS** runs real PostgreSQL/MySQL/MariaDB/SQL Server containers.
  - Supported: instances, clusters, snapshots, parameter/option groups, IAM DB
    auth tokens (validated by Postgres), and TLS with `verify-full`.
  - Read replicas are created from a dump but **receive no live replication**.
  - `ModifyDBInstance` records a new engine version but **keeps the old image**.
  - No `PendingModifiedValues`, and **no Blue/Green**.
- **CloudFront** has a real **data plane**.
  - Requests to `{id}.cloudfront.<host>` are routed to S3 or custom origins.
  - Signed URLs and signed cookies are **enforced** for behaviors with
    `TrustedKeyGroups`: canned and custom policies, RSA-2048 or ECDSA P-256 keys.
  - The cache key is built from the cache policy. Signing params are excluded from
    the key and never forwarded to the origin.
  - OAC and OAI grants are evaluated against the bucket policy, including
    `AWS:SourceArn`.
  - Missing: **no real edge cache** (so no hit/miss behavior), invalidations complete
    instantly, distributions show `Deployed` instantly, and **CloudFront Functions
    are stored but not executed**.
- **IAM enforcement** is opt-in (`FLOCI_SERVICES_IAM_ENFORCEMENT_ENABLED`). It
  evaluates identity, resource, boundary, session, and SCP policies with common
  condition operators.
  - Caveats: `NotPrincipal` isn't supported, and **unknown actions bypass
    enforcement**.
  - S3 bucket policy and Block Public Access need `FLOCI_SERVICES_S3_ENFORCE_AUTH`
    as well.
- **S3, KMS, Secrets Manager, ACM, CloudWatch, Route 53, EC2 (container-backed),
  ECS, ELBv2, Lambda, SQS/SNS, Step Functions, and CodeDeploy** are all present at
  varying fidelity. See section 4A for which are actually usable in scenarios.
- It persists state in `memory`, `persistent`, `hybrid`, or `wal` modes.

| Option | Description | Pros | Cons |
|---|---|---|---|
| **A. Floci only** | Use Floci as-is, build only the console | Fastest start; supports a solid MVP scenario set (section 4A), including most of scenario 2 | No RDS B/G (scenario 1); no edge cache, so scenario 3 can't show its cache-efficiency payoff |
| **B. Custom only** | Write every service ourselves | Full control and fidelity | Huge effort; we'd re-implement S3, IAM, and STS badly |
| **C. Hybrid behind a router** *(recommended long-term)* | Floci handles commodity services (S3, IAM, STS, KMS, Secrets, Logs, ACM). Custom services handle scenario-critical ones (RDS, CloudFront + edge) | Effort goes where the learning is; backends can be swapped per service | Cross-service consistency (ARNs, account IDs, IAM) needs care |
| **D. Hybrid, contributing upstream to Floci** | Like C, but new RDS/CF features are implemented as Floci PRs (Java/Quarkus) | Benefits the community; one process | Tied to upstream's design and review pace; B/G orchestration is heavy for an in-process emulator |
| **E. Alternative base emulators** | LocalStack (community edition limits and account requirements have changed; check the current licence), Moto server (Python, broad API, shallow behavior), MiniStack and similar | Moto is easy to extend in Python | Same data-plane gap; fidelity varies |

**Recommendation:** Ship the **MVP on option A** (section 4A), and move to C once a
scenario needs something Floci lacks. Build custom services as independent processes
that follow Floci's conventions (account ID, region, ARN format), so they could be
proposed upstream later (D) if that makes sense. Given how far Floci's CloudFront and
RDS already go, upstream contribution (D) is more attractive than it first looked.

### D2. Runtime/orchestration: Docker Compose vs. k3s

The key question is **how the RDS simulator creates database instances**. A real
`CreateDBInstance` or `CreateBlueGreenDeployment` must start new database processes at
runtime.

| Option | Instance provisioning | Pros | Cons |
|---|---|---|---|
| **A. Docker Compose + Docker socket** | rds-sim calls the Docker API (like Floci does) | Simplest; one `docker compose up`; good for single-learner laptops | Mounting the Docker socket is effectively root on the host; weak multi-tenancy |
| **B. k3s + native API** | rds-sim creates StatefulSets/Services/PVCs through client-go | Real scheduling, resource limits, namespaces per learner, and PVC snapshots via CSI | Heavier; learners need a k3s host or a hosted instance |
| **C. k3s + CloudNativePG operator** | rds-sim writes CNPG `Cluster` CRs; CNPG handles replicas, backups, and failover | Replicas, PITR, and backups are nearly free | CNPG's model (physical streaming) doesn't match RDS B/G (logical); abstraction mismatch |
| **D. Both, via a provisioner interface** *(recommended)* | `Provisioner` interface with `docker` and `kubernetes` drivers | Compose for dev/laptops, k3s for hosted classrooms | Two drivers to test |

**Recommendation:** D. Keep the provisioner interface small (`createInstance`,
`deleteInstance`, `snapshot`, `restore`, `exec`, `endpoint`). Write the Docker driver
first. Add the k8s driver when multi-learner hosting becomes a goal. On k3s, run rds-sim
with a namespaced ServiceAccount and never a host socket.

Supporting pieces for either runtime:
- **CoreDNS** (standalone in Compose, cluster DNS plus a custom zone in k3s) serves
  simulated endpoint hostnames.
- **Traefik or Caddy** handles console and edge ingress, and TLS with a local CA.

### D3. Endpoint, DNS, and naming model

| Option | How clients reach AUUS | Pros | Cons |
|---|---|---|---|
| **A. `AWS_ENDPOINT_URL` only** | Learners set one env var; SDKs/CLI v2 and Terraform (`endpoints {}`) honor it | No DNS tricks; officially supported | Resource hostnames (RDS endpoints, `*.cloudfront.net`) still need a domain |
| **B. Hijack real names** (`*.amazonaws.com`, `*.cloudfront.net`) via DNS + local CA | "Zero config" clients | Confusing, fragile, dangerous if it leaks outside the sandbox; certificate pinning |
| **C. Parallel namespace** *(recommended)* | Endpoint `https://api.auus.test` (or per-service `rds.auus-east-1.auus.test`); resource names like `mydb.c1a2b3.auus-east-1.rds.auus.test` and `d1234abcd.cloudfront.auus.test` | Unambiguous, safe, still realistic in shape | Learners must notice the different suffix (worth teaching anyway) |

Sub-decisions:
- **Region names.** Use real ones (`us-east-1`) for SDK compatibility, or themed ones
  (`auus-east-1`)? SDKs accept arbitrary regions when an endpoint is set, but some
  tools validate region names. *Lean:* real region names, themed domain suffix.
- **TLS.** Run a local CA (step-ca or mkcert) and distribute the root to learner
  machines and containers. The CloudFront scenario needs HTTPS for authenticity
  (viewer protocol policy, HSTS) and for ACM-like certificate attachment.
- **TTL.** RDS endpoint records get a short TTL (5s) so Blue/Green switchover by DNS
  behaves like the real thing, including clients that cache DNS for too long.

### D4. The console (Cloudscape)

`/home/adam/projects/components` is the Cloudscape components source (v3, Apache-2.0).
It includes `app-layout`, `side-navigation`, `top-navigation`, `table`,
`property-filter`, `wizard`, `key-value-pairs`, `flashbar`, `status-indicator`,
`split-panel`, `tutorial-panel`, `hotspot`/`annotation-context`, `code-editor`, the chart
components, and `s3-resource-selector`. That covers most of what a console page needs.

**D4a. Consuming the components**

| Option | Pros | Cons |
|---|---|---|
| **A. npm `@cloudscape-design/components` + `global-styles`** *(recommended to start)* | Simple, versioned, no build of the library | Can't patch internals |
| **B. Build from the local checkout** (`npm link` / workspace) | Can patch and theme deeply, and track unreleased work | Slower builds; must keep in sync |
| **C. npm + `src-themeable` theming** | Custom AUUS palette/branding without forking | Theming API has limits |

Start with A (plus C for a light brand distinction). Use the local checkout as a
reference and for patches only if needed.

**D4b. How the console talks to backends**

| Option | Description | Pros | Cons |
|---|---|---|---|
| **A. AWS SDK v3 in the browser → router** *(recommended)* | Console signs requests with the learner's temporary credentials, just like the real console's clients | The console becomes one more API client, so every UI action is reproducible from the CLI; fidelity gaps show up immediately | Needs CORS on the router; credentials live in the browser (acceptable in a sandbox) |
| **B. Backend-for-frontend (BFF)** | Console calls a bespoke JSON API that fans out | Easier aggregation (e.g. dashboards) | Console can drift from the real API; two surfaces to keep consistent |
| **C. Mixed** | SDK for resource CRUD, BFF only for console-only views (home dashboard, scenario panel, cost explorer) | Pragmatic | Discipline required about what goes where |

**Recommendation:** C, defaulting to A. Anything that exists in real AWS goes through the
real API.

**D4c. Structure**

- A **single SPA** with a service registry (each service is a lazy-loaded route module:
  `rds/`, `cloudfront/`, `s3/`, `iam/`, `cloudwatch/`), or **micro-frontends**. A single
  SPA is enough, and micro-frontends aren't worth the cost here.
- **Service pages for the first scenarios:**
  - *RDS:* Databases table (grouped by cluster/B/G deployment, like the real console),
    database detail tabs (Connectivity, Monitoring, Configuration, Maintenance), the
    **Create Blue/Green deployment** wizard, **Switch over** modal with timeout,
    parameter groups editor, and the Events list.
  - *CloudFront:* Distributions table, distribution detail (Origins, Behaviors,
    Invalidations, Error pages), cache policies, origin access control, **Public keys**,
    **Key groups**, and a monitoring tab (requests, hit ratio, 4xx/5xx).
  - *S3:* Buckets, objects browser, bucket policy editor (`code-editor`), and a
    pre-signed URL generator (in the real console this is in the object Actions menu).
  - *IAM:* Minimal. Users/roles/policies viewer and a policy editor.
  - *CloudWatch:* Metrics explorer built on the Cloudscape line/area charts, plus
    alarms if a scenario needs them.
- **Scenario overlay.** Use `tutorial-panel` and `hotspot` for guided mode, a
  `split-panel` or drawer for the task checklist, and a "Check my work" button that
  calls the scenario engine.
- **Trademark hygiene.** Cloudscape's code is Apache-2.0, but AWS service icons, the
  smile logo, and the "AWS" name aren't ours to use. Use AUUS branding, generic icons,
  and a visible "not affiliated with AWS" notice. Keep service *names*
  (RDS, CloudFront) where needed for learning transfer, and have the policy reviewed.

### D5. Identity, authentication, and IAM

| Option | Description | Pros | Cons |
|---|---|---|---|
| **A. No auth** | Any key accepted | Trivial | Can't teach key groups vs IAM, OAC, bucket policies |
| **B. SigV4 verification, no policy evaluation** | Router verifies signatures against Floci/own credential store | Catches malformed clients; realistic `SignatureDoesNotMatch` errors | Permissions are all-or-nothing |
| **C. SigV4 + selective policy evaluation** *(recommended)* | Evaluate identity + resource policies for scenario-relevant actions (S3 GetObject by CloudFront OAC principal, `rds:SwitchoverBlueGreenDeployment`, `cloudfront:CreateInvalidation`) | Teaches the specific permission lessons; failures are explainable | Need a policy evaluator (use an existing library such as the IAM policy simulator logic in Moto, `cedar` for our own services, or a small evaluator) |
| **D. Full IAM fidelity** | All services enforce | Most authentic | Huge; causes frustrating unrelated failures |

**Console sign-in:** a login page that issues STS-style temporary credentials
(`AssumeRole` into a learner role). Optional OIDC via Keycloak/Authentik for hosted
classrooms. Each learner gets an **account ID**, which gives an easy isolation
boundary (see D7).

### D6. Implementation language for custom services

| Option | Pros | Cons |
|---|---|---|
| **Go** *(recommended for rds-sim, edge, router)* | Strong HTTP and reverse-proxy stack, `aws-sdk-go-v2` has usable request/response models and SigV4, first-class Docker and k8s clients, single static binaries, fast proxies | Handwritten XML/Query protocol serializers for RDS (RDS uses the AWS Query protocol with XML responses) |
| **Kotlin/Java on Quarkus** | Same as Floci, so upstream contribution is possible | Heavier dev loop; proxies less natural |
| **Python (FastAPI) / Moto extension** | Fast to iterate; Moto already models RDS responses (including some B/G shapes) | Weaker for high-throughput edge proxy; GIL for proxy work |
| **TypeScript (Node)** | Shares types with the console | Proxy/cache performance and SigV4 server-side verification are less mature |

A reasonable split: **Go** for router, rds-sim, and edge. Generate API serializers from
the official **Smithy models** (`aws/api-models-aws`) so shapes, enums, and error codes
are correct by construction rather than hand-copied.

### D7. Multi-tenancy and learner isolation

| Option | Description | Fit |
|---|---|---|
| **A. One stack per learner (Compose on their machine)** | Everything local | Self-paced learners; zero hosting cost |
| **B. One namespace per learner on shared k3s** | Router + backends per namespace | Classrooms of about 10–30; strong isolation; reset by deleting the namespace |
| **C. Shared stack, account-ID isolation** | One set of services, partitioned by account | Most resource-efficient; relies on every custom service scoping correctly (a Floci partitioning check is needed) |
| **D. Ephemeral VMs per learner** (Firecracker / cloud VMs) | Full stack per VM | Strongest isolation; highest cost; matches the existing lab-environment plans in this repo |

Plan for A first and design for B. Every resource carries `accountId` and `region`, and
no custom service keeps global state keyed only by name.

### D8. State, persistence, reset, and checkpoints

Scenarios need **known starting states** and fast **reset**.

- Floci persistence mode: `memory` for CI/tests, `hybrid` or `wal` for learner sessions.
- Custom services store control-plane state in one small Postgres ("metadata DB"), or
  in SQLite per process for the Compose profile.
- Data-plane state (database volumes, edge caches) lives in Docker volumes or PVCs.
- **Checkpoint options:**
  - (a) Re-seed from a scenario manifest (slow, always correct).
  - (b) Volume/PVC snapshots (CSI snapshots on k3s, `docker commit` or volume tarballs
    on Compose).
  - (c) Pre-baked images per scenario start state.
  - *Lean:* (a) as the source of truth, (c) as an optimization for heavy states such
    as a pre-populated 5 GB database.

### D9. Time model

Real RDS Blue/Green creation takes 15–60+ minutes, and CloudFront deploys take minutes.

| Option | Description |
|---|---|
| **A. Real-ish time** | Authentic but tedious |
| **B. Compressed, configurable** *(recommended)* | A per-scenario `timeScale`: state machines use scheduled transitions (e.g. B/G `PROVISIONING` for 90s instead of 30m). Data-plane work (actual replication catch-up) takes the time it really takes |
| **C. Instant** | Good for CI and for instructors skipping ahead |

Expose `timeScale` in the scenario manifest and an instructor "fast-forward" control.

### D10. Observability, CloudWatch, and "billing"

- Every custom service emits **Prometheus** metrics. A small adapter exposes them
  through the CloudWatch `GetMetricData`/`ListMetrics` API with real namespaces and
  dimensions (`AWS/RDS` `ReplicaLag`, `DatabaseConnections`; `AWS/CloudFront` `Requests`,
  `CacheHitRate`, `BytesDownloaded`, `4xxErrorRate`). The console charts read that API.
  Alternative: store the metrics directly in Floci's CloudWatch via `PutMetricData`.
  That's simpler but loses Prometheus querying.
- **CloudTrail-like audit log** from the router. It's valuable for grading ("did the
  learner call `SwitchoverBlueGreenDeployment` with a timeout?") and for instructor
  replay.
- **Cost simulation.** A price sheet (per GB egress from S3 vs CloudFront, per 10k
  requests, instance-hours) turns metrics into a "Cost Explorer"-style estimate.
  Scenario 3 depends on this. Mark it clearly as illustrative, not current AWS pricing.

---

## 4A. MVP: authentic scenarios with no custom backing services

**Goal:** find the scenarios we can ship with **Floci + Cloudscape console + scenario
scripts only**. Our own code is limited to the console, seed data, and checks. The
backing services must behave authentically themselves: real enforcement, real
execution, real traffic. It isn't enough for the API calls to succeed.

### 4A.1 How to judge "authentic enough"

A scenario qualifies for the MVP only if **the thing the learner is supposed to learn
comes from a Floci data plane or enforcement path**, not from a stored-but-inert
configuration. Floci's docs are unusually candid about what's inert, which makes
screening easier.

| Grade | Meaning | Examples in Floci |
|---|---|---|
| **Real** | Enforced or executed; wrong config produces real failures | CloudFront signed URLs/cookies, S3 bucket policy + BPA (with flags), IAM policy evaluation (with flag), Lambda in Docker, SQS redrive/visibility, SNS filter policies, Step Functions ASL, API GW authorizers, CodeDeploy Lambda traffic shifting, Postgres behind RDS, IAM DB auth, EC2 IMDS role creds, ASG capacity reconciliation, ALB HTTP proxying + health checks, KMS crypto |
| **Partial** | Works with caveats that a scenario must avoid | Secrets rotation (no `AWSPENDING` when `RotateImmediately=false`), CloudWatch alarms (evaluation not confirmed), CloudFormation (**unknown resource types silently no-op**), ECS deployments (single `PRIMARY`, no circuit breaker), Route 53 (private zones resolve; no routing policies) |
| **Inert** | Stored only; don't build scenarios on it | FIS, WAF, security groups/NACLs, RDS engine upgrade (keeps old image), read-replica streaming, CloudFront Functions, CloudFront caching/propagation timing, CloudTrail management events (S3 data events only), log subscription filters, KMS grants, API GW usage plans/API keys |

### 4A.2 Candidate scenarios, ranked

Scoring: **A** = authenticity of the core lesson, **E** = effort (console pages +
seeding + checks), **R** = risk that a Floci gap bites mid-scenario.

**Tier 1: build first**

1. **Lock down a private asset bucket behind CloudFront signed URLs**
   (scenario 2 minus caching). A: high · E: medium · R: low-medium
   - Tasks:
     - Make the bucket private and turn on Block Public Access.
     - Create the distribution with an OAC and write a bucket policy with
       `AWS:SourceArn`.
     - Upload a public key, create a key group, and set trusted key groups on
       `/private/*`.
     - Write a signer (Python/Node) for canned URLs, then custom-policy URLs with an
       IP restriction, then signed cookies.
   - What Floci actually enforces:
     - Direct S3 access gets 403.
     - An unsigned request to CloudFront gets 403.
     - An expired or tampered signature gets 403.
     - A wrong key group gets 403.
     - An OAC without the bucket policy grant gets 403.
   - Needs `FLOCI_SERVICES_S3_ENFORCE_AUTH=true` and IAM enforcement.
   - Gap: "Deployed" is instant and there are no `X-Cache` headers. Don't grade on
     caching.

2. **Poison messages and the visibility-timeout trap (SQS → Lambda)**. A: high · E:
   low-medium · R: low
   - Story: an order processor Lambda is fed by SQS.
   - Faults:
     - One message makes the Lambda throw.
     - The Lambda timeout is longer than the queue's visibility timeout, so messages
       are processed twice.
     - There's no DLQ.
   - Tasks:
     - Add a redrive policy with `maxReceiveCount`.
     - Fix the timeout/visibility mismatch.
     - Use partial batch responses.
     - Redrive the DLQ with `StartMessageMoveTask` after fixing the bug.
   - Variant: a FIFO queue with message groups to show ordering vs. throughput.
   - Grading: queue attributes, DLQ depth, and Floci's `/_aws/sqs/messages` peek
     endpoint.

3. **Why am I getting AccessDenied? (IAM troubleshooting)**. A: high · E: medium ·
   R: medium (unknown actions bypass enforcement; the seed must only use actions
   Floci knows)
   - A series of short "incidents" with IAM enforcement on:
     - Explicit deny beating allow.
     - A permission boundary capping a role.
     - A bucket policy vs. identity policy interaction.
     - A condition key (`aws:ResourceTag`, `s3:prefix`).
     - A session policy on `AssumeRole`.
     - An SCP blocking a region (with `FLOCI_SERVICES_ORGANIZATIONS_SCP_ENFORCEMENT_ENABLED`).
   - Well suited to short, repeatable exercises.

4. **Safe Lambda deployments: canary with automatic rollback (CodeDeploy)**. A: high ·
   E: medium · R: low-medium
   - Tasks:
     - Publish versions, create a `live` alias, and set up a CodeDeploy
       application/deployment group with `Canary10Percent5Minutes` and a
       `BeforeAllowTraffic` hook Lambda.
     - Deploy v2, then deploy a broken v3 and watch the hook fail and the alias roll
       back.
   - Floci really shifts alias `RoutingConfig`. The waits are compressed (5s/2s
     steps), which is convenient for learners.
   - Until custom services exist, this is the MVP's **blue/green-style deployment
     lesson**, standing in for RDS B/G.

**Tier 2: good, but spike first**

5. **Secure an HTTP API with Cognito (API Gateway v2 JWT authorizer)**. A: high ·
   E: medium · R: low-medium
   - Uses a user pool, app client, `USER_PASSWORD_AUTH`/SRP, a JWT authorizer, and
     `cognito:groups`-based authorization in the Lambda. REST API + `AWS_IAM` auth is
     a variant.
   - Avoid: API keys/usage plans (not enforced), MFA delivery, and social IdPs.

6. **Recover from a destructive migration on RDS PostgreSQL**. A: medium-high · E:
   medium · R: medium
   - Story: someone ran `DELETE` without a `WHERE`. Restore from a snapshot to a new
     instance, verify the data, and repoint the app, or copy the rows back.
   - Spike: confirm that Floci snapshots capture **data**, not just metadata (replicas
     are dump-based, so snapshots probably are too).
   - Variant: move the app from password auth to **IAM DB auth** tokens, which the
     backing Postgres really validates, including endpoint binding.

7. **Credentials on EC2 the right way: instance profiles + IMDSv2**. A: high · E:
   medium · R: medium (Docker-socket and privileged-container requirements)
   - Tasks: SSH into a container-backed instance, find hard-coded keys in the app
     config, replace them with an instance profile, require IMDSv2
     (`HttpTokens=required`), and verify the credentials come from IMDS.
   - Avoid anything that depends on security groups (not enforced).

8. **Self-healing web tier (ALB + ASG + health checks)**. A: medium · E: medium-high ·
   R: medium-high
   - ASG reconciliation launches and registers instances, and ALB really proxies HTTP.
   - Spike: whether unhealthy targets are **replaced** and whether routing skips
     them. Weighted forward actions and stickiness aren't documented.

9. **Event-driven fan-out with filtering (S3 → EventBridge/SNS → SQS/Lambda + Step
   Functions)**. A: high · E: medium-high · R: low-medium
   - Covers SNS filter policies, EventBridge rules, and a Step Functions workflow with
     Retry/Catch and `.waitForTaskToken` for a human approval step.
   - Wait states are capped at 30s by default, which suits learners.

10. **Rotate a database secret without downtime (Secrets Manager + RDS)**. A: high ·
    E: medium-high · R: medium-high
    - The rotation Lambda really changes the Postgres password.
    - Must use `RotateImmediately=true`, or rotation functions that don't rely on a
      pre-created `AWSPENDING` version.

**Tier 3: partial versions of the anchor scenarios**

11. **Scenario 3, functional half: S3 pre-signed → CloudFront signed URLs.** Everything
    in 7.3 except the cache-efficiency proof works on Floci:
    - private bucket + OAC
    - key groups and key rotation (two keys in a group)
    - signer swap behind a feature flag / app-version gate
    - signed cookies vs. URLs
    - URL stability from expiry bucketing, which is visible on the *device
      simulator's* URL-keyed cache even without an edge cache

    To show **edge** hit ratio without writing a service, put an **off-the-shelf
    cache** (Varnish or nginx `proxy_cache`, configuration only) between Floci's
    CloudFront and the origin:
    - The CloudFront origin becomes a custom origin pointing at the cache.
    - Because Floci strips signing params before the origin, the cache key is clean,
      so the cache exposes real hit/miss counts.
    - Cost: the origin hop is no longer OAC-to-S3, so the OAC lesson moves to
      scenario 1.
    - Alternative: accept "origin request count" (from S3 data-event logs via
      CloudTrail, which Floci *does* emit for S3) as the before/after metric.

12. **Scenario 1 substitute: "Blue/green by hand" on RDS.** Restore a snapshot as
    "green", apply the migration there, repoint the application via a Route 53
    **private hosted zone** CNAME (Floci's embedded DNS serves private zones), and
    compare.
    - It teaches the *concept* and the DNS-caching gotcha.
    - It doesn't teach real `CreateBlueGreenDeployment` guardrails, and data written
      to blue after the snapshot is lost. Present that openly as a lesson ("this is
      why managed B/G uses replication").

**Not viable on Floci alone:** real RDS Blue/Green, engine major-version upgrades,
read-replica lag, WAF blocking, security-group or network-isolation lessons, chaos
engineering via FIS, CloudFront Functions, CloudTrail-based "who did this?"
investigations of management calls, and anything that grades on CloudFront cache
behavior.

### 4A.3 Minimal MVP architecture (no custom backing services)

```
Browser ── AUUS Console (Cloudscape SPA, AWS SDK v3) ─┐
Terminal ── aws CLI / boto3 / Terraform ──────────────┤
                                                      ▼
                          Caddy (static console + reverse proxy to :4566,
                                 CORS headers, local-CA TLS, *.auus.test)
                                                      ▼
                                   Floci (+ Docker socket via socket-proxy)
                                                      │
          Scenario runner: YAML manifest + seed scripts (boto3) + checks (pytest/CEL)
```

MVP decisions that differ from the full plan:
- **No router service.** Caddy (configuration only) provides one origin, CORS, and TLS.
  SigV4 verification and IAM evaluation come from Floci's own enforcement flags. Spike
  whether Floci returns browser-friendly CORS headers itself.
- **Grading uses resource state and data-plane probes only.** Examples: curl a CloudFront
  URL with and without a signature, and read a queue's DLQ depth. There's no audit
  log of management calls, because Floci's CloudTrail only covers S3 data events. If
  "how did they do it" grading matters, add an **access-logging reverse proxy**
  (Caddy/Envoy JSON access logs with the `X-Amz-Target`/`Action` parameter captured).
  That's configuration, not a service.
- **Floci flags on by default:** `FLOCI_SERVICES_IAM_ENFORCEMENT_ENABLED=true` and
  `FLOCI_SERVICES_S3_ENFORCE_AUTH=true`. Learners get realistic `AccessDenied`
  responses. The console must sign in as a real IAM principal, not a root-like
  wildcard, or the IAM lessons go away.
- **Pin the Floci version.** Every scenario gets a smoke test that runs its golden
  path against the pinned version in CI.
- **Guard against silent success.** Floci's CloudFormation accepts unknown resource
  types as no-ops, and IAM skips unknown actions. Each scenario's CI test must assert
  the *failure* path too (e.g. "this request must be denied"), not just the happy path.

### 4A.4 Console pages the MVP needs

| Page set | Scenarios served | Notes |
|---|---|---|
| S3 (buckets, objects, permissions tab, bucket policy editor, BPA) | 1, 3, 11 | Highest reuse; Cloudscape `s3-resource-selector` helps |
| CloudFront (distributions, behaviors, OAC, public keys, key groups) | 1, 11 | Mirror the real console's behavior editor |
| IAM (users/roles/policies, policy JSON editor, boundaries) | 3, 7, all sign-in | A read-mostly UI is fine at first |
| Lambda (functions, versions/aliases, test invoke, logs link) | 2, 4, 5, 9 | |
| SQS (queues, send/poll messages, redrive) | 2, 9 | "Start DLQ redrive" button is a nice authentic touch |
| CloudWatch Logs (groups, streams, events) | 2, 4, 5, 9 | Floci Lambda writes logs |
| CodeDeploy (applications, deployment groups, deployment progress) | 4 | Progress uses `steps`/`progress-bar` |

Many learners can do Tier 1 scenarios **CLI-first** while console pages are still
missing. The console can grow scenario by scenario rather than blocking the MVP.

### 4A.5 Suggested MVP cut

- **MVP-1 (about 3–4 weeks):** Compose stack (Floci + Caddy + console shell), scenario
  runner, and scenarios **#1 (CloudFront signed URLs)** and **#2 (SQS poison messages)**.
  Console: S3, CloudFront, SQS, Lambda (read-only), CloudWatch Logs.
- **MVP-2:** Scenarios **#3 (IAM)** and **#4 (Lambda canary)**, plus the IAM and
  CodeDeploy pages.
- **MVP-3:** Scenario **#11** (functional half of the mobile migration, with or without
  the Varnish add-on) and one of #5/#6.
- After that, begin custom services (section 9 phases) where MVP learners hit the
  limits: RDS B/G first, then CloudFront caching/POPs and metrics.

---

## 5. Custom service: `rds-sim` and Blue/Green deployments

### 5.0 Build on Floci's RDS, or replace it?

Floci already runs real database containers and handles instances, snapshots, parameter
groups, and IAM auth. What it lacks is live replication, real engine upgrades, and B/G.

| Option | Description | Trade-off |
|---|---|---|
| **A. Standalone `rds-sim`** (sections 5.1–5.6 as written) | Router sends all `rds:*` calls to our service | Full control; duplicates what Floci already does well |
| **B. B/G add-on beside Floci** *(now preferred)* | Router sends only the `*BlueGreenDeployment*` actions to a small orchestrator. The orchestrator calls Floci's `CreateDBInstance` for green, sets up logical replication directly in the Postgres containers, and handles the switchover by retargeting endpoints (DNS/proxy). Everything else stays on Floci | Much less code. Needs green to run the *target* engine version (Floci keeps the old image on modify, so create green fresh at the target version) and needs Floci's endpoint/proxy model to be retargetable |
| **C. Upstream into Floci** | Implement B/G and real replication as Floci PRs | Best long-term; slowest; Java/Quarkus |

### 5.1 API surface (phase 1)

Control plane (AWS Query protocol, XML responses):
- Instances: `CreateDBInstance`, `DescribeDBInstances`, `ModifyDBInstance`,
  `DeleteDBInstance`, `RebootDBInstance`
- Parameters: `CreateDBParameterGroup`, `ModifyDBParameterGroup`,
  `DescribeDBParameters`, `DescribeDBParameterGroups`,
  `DescribeDBEngineVersions`
- Snapshots: `CreateDBSnapshot`, `DescribeDBSnapshots`,
  `RestoreDBInstanceFromDBSnapshot`
- Blue/Green: `CreateBlueGreenDeployment`, `DescribeBlueGreenDeployments`,
  `SwitchoverBlueGreenDeployment`, `DeleteBlueGreenDeployment`
- Events: `DescribeEvents` (the console's Events tab and grading both use it)
- Tags: `AddTagsToResource`, `ListTagsForResource`

Later: read replicas, Multi-AZ failover (`RebootDBInstance --force-failover`), Aurora
clusters (a separate, much bigger effort), and RDS Proxy (a good companion to B/G).

### 5.2 Blue/Green state machine (modeled on real behavior)

```
CreateBlueGreenDeployment(source=blue ARN, targetEngineVersion?, targetDBParameterGroupName?)
  └─ status: PROVISIONING
       tasks: CREATING_READ_REPLICA_OF_SOURCE → DB_ENGINE_VERSION_UPGRADE
              → CONFIGURE_BACKUPS → CREATING_TOPOLOGY_OF_SOURCE
       green instance name: <blue>-green-<random>  (read-only)
  └─ status: AVAILABLE           (green caught up, replication running)
SwitchoverBlueGreenDeployment(timeout=300)
  └─ status: SWITCHOVER_IN_PROGRESS
       guardrail checks → block writes on blue → wait for green to catch up
       → rename: blue → <blue>-old1, green → <blue>; endpoints move
  └─ SWITCHOVER_COMPLETED  |  SWITCHOVER_FAILED (rolled back, blue unchanged)
DeleteBlueGreenDeployment(deleteTarget?)
```

Other states to support: `INVALID_CONFIGURATION` and `PROVISIONING_FAILED`, for example
when the source parameter group lacks the logical replication setting. Check exact enum
values and task names against the Smithy model and the RDS User Guide during
implementation. The list above is from memory and must be verified.

### 5.3 How replication really happens

| Option | Description | Pros | Cons |
|---|---|---|---|
| **A. Real logical replication** *(recommended for PostgreSQL)* | rds-sim creates a publication on blue, launches green from a snapshot (`pg_dump`/`pg_basebackup`, then upgrade via `pg_upgrade`, or dump/restore into the new major version), then creates a subscription | Real behavior: DDL isn't replicated, tables without a primary key/replica identity can't replicate UPDATE/DELETE, sequences need a sync step, large transactions lag | Most work; must handle the upgrade step carefully |
| **B. Physical streaming replica + promote** | Simple same-version case | Easy with Postgres or CNPG | Can't do a major version upgrade; teaches the wrong model |
| **C. Fake (copy at switchover time)** | Dump/restore during switchover | Trivial | Hides every interesting failure mode; no lag metric |
| **D. MySQL binlog replication** | For a MySQL variant of the scenario | Matches RDS MySQL B/G | Second engine to support; do later |

**Recommendation:** A for PostgreSQL first. Expose `ReplicaLag` computed from
`pg_stat_subscription` / `pg_replication_slots`. Implement the documented
prerequisite checks (e.g. `rds.logical_replication = 1` in the parameter group, which
needs a reboot to take effect) so the classic "B/G creation fails because the parameter
group is wrong" moment happens.

### 5.4 Endpoint switchover mechanism

This is the most educational part, so the design choice matters.

| Option | Mechanism | What learners experience |
|---|---|---|
| **A. DNS swap** *(recommended default)* | CoreDNS record for `mydb.<id>.<region>.rds.auus.test` moves from blue to green IP (TTL 5s) | Authentic. Apps with DNS caching (JVM default, connection pools that never reconnect) keep writing to the *old* instance, which is now renamed and read-only, so they get errors. Real teaching moment. |
| **B. TCP proxy per endpoint** (HAProxy/Envoy/pgbouncer) | Stable IP; proxy retargets and drops connections | Cleaner cutover; less authentic; good "RDS Proxy" analog |
| **C. Both, selectable per scenario** | Scenario manifest picks A or B | Enables a comparison scenario: "Why does RDS Proxy/ the AWS JDBC wrapper help with B/G?" |

Go with C, defaulting to A.

### 5.5 Guardrails and failure injection

Implement as real checks where possible, and as scenario-injected faults otherwise:
- **Long-running transaction on blue** blocks the switchover until the timeout, then
  `SWITCHOVER_FAILED` with rollback. It's real, because we actually wait for active
  transactions.
- **Replica lag above threshold** at switchover time. Real, driven by a load generator.
- **DDL on blue after green creation.** Green schema diverges and replication breaks.
  Real, from logical replication semantics.
- **Table without a primary key.** UPDATE/DELETE replication errors. Real.
- **Sequence drift after switchover** if sequence sync isn't handled. Decide whether
  rds-sim syncs sequences at switchover (check current RDS behavior per engine
  version) and make that a scenario toggle.
- **Client DNS caching.** Real, via option A in 5.4.
- **Unsupported features on the source** (e.g. certain extensions). Scenario-injected
  `INVALID_CONFIGURATION`.

### 5.6 Supporting workload

A small "orders app" (any language) that continuously writes to the database endpoint
and reports errors and write latency. It runs as a container. Learners watch its
dashboard during switchover, and the scenario grader reads its error count ("downtime
under 5s", "no lost writes": compare row counts and checksums between old blue and
new blue).

---

## 6. Custom service: CloudFront control plane + edge data plane

### 6.1 Control plane

| Option | Description |
|---|---|
| **A. Use Floci's CloudFront API** | Floci stores distributions, cache policies, etc.; our edge reads config from Floci | Least control-plane work; depends on Floci supporting public keys, key groups, OAC, and trusted key groups on behaviors (needs verification) and exposing config for the edge to read |
| **B. Custom `cloudfront-sim` control plane** *(recommended if A has gaps)* | REST-XML API generated from Smithy models; the source of truth for edge config | Full control over `Deployed` status timing, key groups, and OAC |
| **C. Floci for storage, custom sidecar for propagation** | Poll Floci, compile to edge config | Couples to Floci internals |

**Update after reading Floci's CloudFront docs:** Floci already implements public keys,
key groups, OAC/OAI, trusted key groups, cache and origin-request policies, and a data
plane that enforces signed URLs and cookies. **Option A is now the default.** Custom work
shrinks to what Floci explicitly lacks:
- a real edge cache (hit/miss, TTLs, `X-Cache`/`Age`)
- multiple POPs
- `InProgress` → `Deployed` propagation timing
- real invalidation purging
- CloudFront Functions execution
- CloudFront CloudWatch metrics

Confirm this with the Phase 0 spike before relying on it.

Minimum API: distributions (`Create/Get/Update/List/DeleteDistribution`, with
`DistributionConfig` ETag/`IfMatch` semantics), `CreateInvalidation`, cache and
origin-request policies (including AWS-managed policy IDs such as
*CachingOptimized* and *CachingDisabled*), `CreatePublicKey`, `CreateKeyGroup`,
`CreateOriginAccessControl`, and optionally CloudFront Functions.

Config propagation: the control plane compiles each distribution into a **versioned
edge config** document and publishes it. Options are edges polling (simple), a
push/stream via NATS or Redis pub/sub, or on k3s a ConfigMap/CRD watched by the edges.
Status flips from `InProgress` to `Deployed` once every edge acknowledges the version.
That creates a realistic propagation delay.

### 6.2 Edge data plane

| Option | Pros | Cons |
|---|---|---|
| **A. Custom Go reverse proxy + cache** | Exact CloudFront semantics: signed URL/cookie validation, cache-key construction from cache policies, stripping signing params, OAC SigV4 signing to S3 origins, CloudFront-style response headers | We own caching correctness (use a library such as `groupcache` or a disk cache; implement `Cache-Control`/min/max/default TTL rules) |
| **B. OpenResty (Nginx + Lua)** | Battle-tested cache; Lua for signature validation | Two config worlds (generated nginx.conf + Lua); OAC SigV4 in Lua is painful |
| **C. Varnish + VMOD/VCL** | Excellent cache and observability | Signature validation and SigV4 signing are awkward in VCL |
| **D. Envoy + Lua/Wasm filters** | Dynamic config via xDS (a natural fit for "propagation") | Steep learning curve; cache filter is less mature |

| **E. Floci's data plane + off-the-shelf cache tier** | Floci validates signatures and strips signing params; Varnish/nginx behind it as a custom origin provides real caching and hit/miss metrics | No code; loses OAC-to-S3 on that path; single "POP"; no CloudFront-style response headers to viewers |
| **F. Contribute caching to Floci's data plane** | Add a cache + `X-Cache`/`Age` headers upstream | Benefits everyone; upstream pace |

**Recommendation (revised):** E for the MVP, since it needs no code (section 4A,
scenario #11). Move to A only if multi-POP behavior, CloudFront-style viewer headers, or
Functions execution become required, and consider F in parallel. Whichever edge we use,
keep its cache store behind an interface.

### 6.3 Edge behaviors to implement

- **Signed URLs:** canned policy (`Expires`, `Signature`, `Key-Pair-Id`) and custom
  policy (`Policy`, `Signature`, `Key-Pair-Id`, with `DateLessThan`,
  `DateGreaterThan`, `IpAddress`, and wildcard resources). CloudFront's URL-safe base64
  substitutions (`+`→`-`, `=`→`_`, `/`→`~`). RSA-SHA1 signatures. Check whether newer
  algorithm support (e.g. ECDSA) exists in real CloudFront today, and implement it if so.
- **Signed cookies:** `CloudFront-Policy`, `CloudFront-Signature`,
  `CloudFront-Key-Pair-Id`, and `CloudFront-Expires`. These are important for the
  mobile migration scenario.
- **Trusted key groups per cache behavior.** Validation uses public keys from the key
  groups attached to the *matched* behavior. Unsigned requests to a protected
  behavior get `403` with CloudFront-style XML error bodies (`MissingKey`, "Access
  denied").
- **Cache key.** Built from the cache policy (headers, cookies, query strings:
  none/whitelist/all). **Signing parameters are never part of the cache key and aren't
  forwarded to the origin.** This is the property scenario 3 relies on. Verify it
  against CloudFront documentation and build it into tests.
- **Origins:** S3 via **OAC** (edge SigV4-signs origin requests as the
  `cloudfront.amazonaws.com` service principal, and the bucket policy uses
  `AWS:SourceArn`), S3 via legacy OAI (optional, for a "migrate OAI → OAC" scenario),
  and custom HTTP origins (the web app).
- **Response headers:** `X-Cache: Hit from cloudfront` / `Miss from cloudfront` /
  `RefreshHit`, `Age`, `Via`, `X-Amz-Cf-Pop`, `X-Amz-Cf-Id`.
- **Multiple POPs + regional edge cache.** Run 2–3 edge containers ("POPs") with a
  shared mid-tier cache container. The device simulator assigns clients to POPs. That
  makes "cold in a new POP" visible, and it's cheap to provide.
- **Invalidations** propagate to all POPs, with realistic `InProgress` →
  `Completed` timing.
- **Stretch:** CloudFront Functions (viewer request/response) run on an embedded JS
  engine (e.g. `goja`) with a restricted runtime. Many real signed-URL setups use a
  function for URL normalization, so this is useful.

### 6.4 TLS and alternate domain names

Each distribution gets `dXXXXXXXXXXXX.cloudfront.auus.test` automatically. Alternate
domain names (CNAMEs) require a certificate from the "ACM" service (Floci ACM for
metadata, the local CA for real issuance), so learners experience the "CNAME needs a
matching certificate in us-east-1" rule.

---

## 7. Scenario 3: S3 pre-signed URLs → CloudFront signed URLs for a mobile app

### 7.1 Story

"PhotoFeed" is a social photo app. Its API returns image URLs to the mobile app. Today
the backend returns an **S3 pre-signed URL per image per request**. Each URL carries a
unique `X-Amz-Signature` (and `X-Amz-Date`), so:
- Nothing between the device and S3 can cache the object usefully. A CDN in front
  would have to forward every query string to pass S3 auth, so the signature becomes
  part of the cache key and every URL is a miss. Using OAC to sign origin requests also
  conflicts with viewer-supplied S3 signatures.
- The device's own HTTP/image cache is keyed on URL, so the same image refetches every
  time the feed reloads.
- All egress is billed as S3 data transfer, and S3 carries all the request load.

The learner migrates to **CloudFront signed URLs (or signed cookies)** backed by a private
S3 bucket via OAC. They choose a signing strategy that makes URLs cacheable at the edge
and on the device.

### 7.2 Components

- **`photofeed-api`**, a small service owned by the scenario. It has a pluggable URL
  signer (`s3-presign` | `cf-signed-url` | `cf-signed-cookie`), a feature flag, and an
  app-version header check.
- **Device simulator** (stands in for the mobile app, which is out of scope):
  - Option A: a **headless load generator** (k6 or Go) that simulates N devices with
    realistic feed-scroll patterns (Zipf-distributed image popularity) and an
    on-device URL-keyed cache.
  - Option B: a **browser "phone frame" web app** that renders the feed and shows
    per-image hit/miss badges. It's great for demos and weak for load.
  - Option C: both *(recommended)*. B drives the lesson, and A drives the metrics.
  - It simulates **old app versions in the field** that only understand one URL
    format. This is what makes the migration a real migration rather than a switch
    flip.
- **Dashboards** (console CloudWatch pages + Cost view): S3 GET requests,
  CloudFront requests, cache hit ratio, origin bytes, p50/p95 image latency, and
  estimated monthly cost.

### 7.3 Learning beats / tasks

1. **Baseline.** Observe near-0% cache effectiveness and high S3 request volume.
   Inspect a pre-signed URL and explain why it can't be cached.
2. **Build the CDN path.** Create the distribution with a private S3 origin + OAC, update
   the bucket policy, and verify direct S3 access is denied.
3. **Keys.** Generate an RSA key pair, upload the public key, create a key group, attach
   it as a trusted key group on the `/images/*` behavior, and store the private key in
   Secrets Manager (a Floci service) for `photofeed-api` to read.
4. **Cache policy.** Choose one that excludes the signing params and any per-user noise
   from the cache key. Choose TTLs and origin `Cache-Control`.
5. **Signing strategy (the main design lesson):**
   - Per-request expiry (`now + 5m`) gives a unique URL, so the edge caches (because
     signing params aren't in the cache key) but the **device cache still misses**.
   - **Expiry bucketing** (round `Expires` up to the next hour boundary) gives an
     identical URL for all requests in the window, so both edge and device caches hit.
     Trade-off: a longer effective validity.
   - **Signed cookies** with a wildcard custom policy (`/images/*`) give clean image
     URLs, which is the best device caching. Trade-off: cookie handling in the mobile
     HTTP stack.
6. **Rollout.** Put the new signer behind a feature flag and gate on the
   `X-App-Version` header. Old clients keep S3 URLs until the fleet has upgraded.
   Watch metrics shift as the simulated upgrade percentage rises.
7. **Key rotation.** Add a second public key to the key group, switch the signer, then
   remove the old key without breaking in-flight URLs.
8. **Report.** Before/after hit ratio, origin load, and cost. The grader checks
   thresholds.

### 7.4 Variations

- An "invalidate a DMCA'd image" task (invalidation vs versioned object keys).
- Custom policy with an `IpAddress` condition breaks for mobile clients on changing
  networks. This is a debugging exercise.
- Clock skew on devices vs `DateGreaterThan`.

---

## 8. Scenario engine

Scenarios are data, not code wherever possible.

```yaml
id: rds-bluegreen-pg-major-upgrade
title: Upgrade PostgreSQL 15 → 16 with Blue/Green
timeScale: 0.05              # 20x compression of control-plane waits
endpointMode: dns            # dns | proxy
seed:
  - rds.createParameterGroup: {name: app-pg15, family: postgres15, params: {}}  # deliberately missing logical replication
  - rds.createInstance: {id: orders-db, engine: postgres, version: "15", parameterGroup: app-pg15}
  - sql.load: {instance: orders-db, file: seeds/orders.sql}
  - workload.start: {name: orders-app, target: orders-db}
faults:
  - at: green.available
    action: sql.exec {instance: orders-db, sql: "BEGIN; SELECT pg_sleep(900);"}   # long txn
    optional: true
checks:
  - id: bg-created
    assert: rds.describeBlueGreenDeployments[0].Status == "SWITCHOVER_COMPLETED"
  - id: engine-upgraded
    assert: rds.describeDBInstances("orders-db").EngineVersion startsWith "16"
  - id: low-downtime
    assert: workload("orders-app").maxErrorWindowSeconds < 30
  - id: no-lost-writes
    assert: sql.rowCountEqual(old: "orders-db-old1", new: "orders-db", table: orders)
hints: [...]
```

Design decisions:
- **Check language.** Options are CEL, JSONata, Rego, or plain Go/TS check functions.
  *Lean:* CEL, which is safe, embeddable, and readable.
- **Grading sources.** API state (via the router), the audit log (for "how" questions:
  "did they use a timeout?"), data-plane probes (curl through an edge), and workload
  metrics.
- **Modes.** Guided (tutorial panel + hotspots), unguided (task list only), and
  "incident" (fault injected with no explanation).
- **Instructor controls.** Reset, fast-forward (`timeScale`), inject fault, and view
  learner state.

## 9. Phased roadmap

**Phase 0: Spikes (1–2 weeks)**
- Run Floci in Compose. Exercise S3 pre-signing, IAM/STS, Secrets Manager, ACM,
  CloudFront key-group/OAC APIs, and RDS operations to map real coverage. Record the
  results in a coverage matrix.
- Confirm the claims the MVP depends on (section 4A):
  - CloudFront rejects unsigned, expired, and tampered requests.
  - OAC is denied without the `AWS:SourceArn` grant.
  - SQS redrive and `StartMessageMoveTask` work.
  - CodeDeploy rolls back the alias when a hook fails.
  - IAM enforcement denies the specific actions each scenario uses.
  - RDS snapshots contain data.
  - Browser CORS works against :4566.
- Prototype the router: SigV4 verify + route by credential scope service name to
  Floci. Confirm the AWS CLI v2, SDK v3 (browser), and Terraform AWS provider all work
  through it.
- Hello-world Cloudscape console shell (app-layout, top-nav, side-nav) listing S3
  buckets via SDK v3 in the browser.

**Phase 0.5: Floci-only MVP (section 4A.5)**
- MVP-1 to MVP-3: CloudFront signed URLs, SQS poison messages, IAM troubleshooting,
  Lambda canary, and the functional half of the mobile migration. No custom backing
  services.

**Phase 1: CloudFront caching fidelity (full scenario 2 and 3)**
- Only where Floci falls short: off-the-shelf cache tier (6.2 E), then if needed a Go
  edge for multi-POP, viewer cache headers, and Functions.
- Console: invalidations and monitoring (hit ratio).

**Phase 2: RDS Blue/Green (scenario 1)**
- B/G orchestrator add-on beside Floci's RDS (5.0 B), with real logical replication
  and DNS endpoint mode. Fall back to standalone rds-sim (5.0 A) only if Floci's
  endpoint model can't be retargeted.
- Orders workload app + metrics. Console: databases, B/G wizard, switchover, and events.

**Phase 3: Mobile migration (scenario 3)**
- photofeed-api, device simulator (headless + phone frame), multi-POP edges + mid-tier,
  CloudWatch adapter, cost view.

**Phase 4: Hosting and scale**
- k8s provisioner, namespace-per-learner, OIDC login, instructor dashboard, pre-baked
  scenario images.

**Phase 5: Breadth**
- Additional scenarios (below), MySQL B/G, RDS Proxy mode, CloudFront Functions.

## 10. Decision summary

| # | Decision | Options | Current lean |
|---|---|---|---|
| D1 | Emulation strategy | Floci only / custom only / hybrid / hybrid + upstream / other emulators | Floci-only MVP (4A), then hybrid behind router; upstream where natural |
| 4A | MVP scenario set | see 4A.2 tiers | CF signed URLs, SQS poison messages, IAM troubleshooting, Lambda canary; then mobile-migration functional half |
| 5.0 | RDS B/G approach | standalone rds-sim / add-on beside Floci / upstream | Add-on beside Floci |
| D2 | Runtime | Compose + socket / k3s native / k3s + CNPG / both via provisioner | Both via provisioner interface; Docker first |
| D3 | Naming | endpoint-URL only / hijack real domains / parallel namespace | Parallel `*.auus.test` namespace, real region names, local CA |
| D4a | Components | npm / local build / npm + theming | npm + light theming |
| D4b | Console → backend | SDK in browser / BFF / mixed | Mixed, SDK-first |
| D5 | IAM | none / SigV4 only / SigV4 + selective eval / full | SigV4 + selective evaluation |
| D6 | Language | Go / Quarkus / Python-Moto / TS | Go + Smithy-generated serializers |
| D7 | Tenancy | local stack / namespace / shared account-ID / VMs | Local first, namespace-ready |
| D8 | Reset | re-seed / snapshots / pre-baked images | Re-seed as truth, images as cache |
| D9 | Time | real / compressed / instant | Compressed, configurable |
| 5.3 | B/G replication | logical / physical / fake / binlog | Real logical replication (PG) |
| 5.4 | Switchover | DNS / TCP proxy / both | Both, DNS default |
| 6.1 | CF control plane | Floci / custom / Floci + sidecar | Floci (confirm by spike) |
| 6.2 | Edge | custom Go / OpenResty / Varnish / Envoy / Floci + cache tier / upstream | Floci + off-the-shelf cache for MVP; custom Go only if multi-POP/Functions needed |
| 7.2 | Device sim | headless / phone-frame web / both | Both |
| 8 | Check language | CEL / JSONata / Rego / code | CEL |

## 11. Further scenario candidates

These reuse the same building blocks:
- **RDS:** Multi-AZ failover under load; restoring a snapshot to fix a bad migration;
  parameter group change needing a reboot ("pending-reboot"); read replica lag
  debugging; RDS Proxy absorbing a B/G switchover; Secrets Manager credential rotation
  for RDS.
- **CloudFront:** OAI → OAC migration; cache-key explosion from forwarding all
  headers; versioned assets vs invalidations; custom error pages for SPA routing; geo
  restriction; CloudFront Functions for auth headers.
- **S3:** A bucket policy that accidentally blocks the OAC principal; lifecycle rules;
  pre-signed PUT uploads with size limits (the mobile *upload* counterpart of scenario
  3).
- **Cross-cutting:** "Someone made the bucket public." Investigate with the audit log.

## 12. Risks and open questions

- **Floci coverage and stability.** It's a young project. Pin versions, keep the
  router abstraction so any service can be replaced, and run a nightly conformance
  suite (AWS CLI scripts) against it.
- **Fidelity drift.** Real AWS changes (new B/G features, new signing algorithms).
  Record the AWS documentation version each behavior was modeled on and keep a
  "fidelity notes" page per service, shown to learners where it matters ("In real AWS,
  this takes ~20 minutes").
- **Postgres major-version upgrade inside green.** Engineering-heavy. Consider
  providing green as a fresh target-version instance loaded via dump/restore and then
  subscribed, instead of running `pg_upgrade`.
- **Docker socket exposure** in the Compose profile. Document it clearly and use a
  socket proxy (e.g. `tecnativa/docker-socket-proxy`) that restricts the allowed API
  calls.
- **Trademark/branding.** Settle AUUS naming and iconography before anything is public.
- **Browser SDK credentials.** Acceptable in a sandbox. Make sure the console never
  runs against real AWS by accident: the router rejects real AWS-format access keys
  (`AKIA…`, `ASIA…`), or at least warns.
- **Open questions for the next pass:**
  1. Target audience and hosting: self-paced laptops, instructor-led classrooms, or both?
  2. Is Terraform/CDK support a phase-1 requirement, or is CLI + console enough?
  3. Is MySQL needed for B/G, or is PostgreSQL enough for the foreseeable future?
  4. How strict should grading be (pass/fail vs rubric with partial credit)?
  5. Appetite for contributing RDS/CloudFront fidelity upstream to Floci?

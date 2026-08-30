# Linux Mastery System Plan

## 1. Product Vision

Build an adaptive, practice-first system that helps junior engineers become reliably capable Linux operators. It should diagnose what a learner can already do, overlay that evidence on a Linux knowledge graph, select only skills on the learner's knowledge frontier, require repeated independent performance for mastery, and preserve learning through retrieval and synthesis.

The product is not a conventional LMS. Its primary learning surface is a disposable Linux environment in which the learner completes authentic operational tasks. The application itself is also disposable: it can be started on demand, reconstruct the learner's profile from a remote store controlled by that learner, run a session, persist new evidence, and terminate without losing progress.

The core loop implements the seven learning principles:

1. Diagnose current skills through hands-on tasks.
2. Map the resulting evidence to a personal overlay on the knowledge graph.
3. Select an unmastered skill only when all required prerequisites are mastered.
4. Alternate minimum effective doses of guidance with active sandbox practice.
5. Gate dependent skills on consistent, independent performance while keeping parallel paths open.
6. Generate spaced reviews and broad, timed, closed-book quizzes from durable timestamps and memory state.
7. Prefer new frontier tasks that also exercise due older skills as observable subskills.

## 2. Scope, Constraints, and Non-Goals

### Goals

- Measure practical Linux competence rather than recognition of terminology.
- Make every mastery, remediation, and recommendation decision reproducible from stored evidence.
- Run all compute components without a durable local database or always-on scheduler.
- Let the learner connect a third-party state provider with credentials supplied through the browser.
- Operate comfortably on three virtual machines and never require more than five.
- Provide realistic but strongly isolated Linux practice and deterministic, outcome-based grading.
- Generate review work on demand without duplicate assignments or lost updates.
- Explain why a skill is ready, blocked, due, mastered, or decayed.

### Capacity Assumptions

The initial deployment is for an individual or small cohort, not thousands of simultaneous terminals. It should support roughly 10–25 concurrent learner sandboxes on modest hosts, subject to task size. Concurrency limits, queueing, and idle timeouts protect the fixed VM budget. Scaling beyond five hosts is a later architectural decision, not an implicit MVP requirement.

### Non-Goals

- Building lesson-authoring, curriculum-authoring, publishing, localization, or content-review tooling.
- Defining the complete Linux course catalog in this plan.
- Operating a platform-owned learner database, user directory, or long-running review queue.
- Replacing supervised production experience or granting access to real infrastructure.
- Supporting arbitrary kernel, hardware, nested-virtualization, or unrestricted internet exercises.
- Using completion time, video consumption, or self-reported confidence as mastery evidence.
- Invasive proctoring or claims that a browser can prevent use of a second device.

Lesson text, task bundles, graders, and the base graph are treated as versioned, read-only system inputs. This plan covers how the runtime consumes and validates those artifacts, but not how people author or publish them.

## 3. Statelessness Model

“Stateless” means any application VM can be destroyed after a session and replaced from an image. It does not mean that no state exists. Durable state is divided into two external, learner-portable inputs:

1. **System bundle:** a signed, versioned package containing the knowledge graph, lesson/task manifests, sandbox templates, graders, and runtime policy. It may be downloaded from release storage or baked into an immutable image.
2. **Learner state:** an encrypted or private remote document set owned by the learner and accessed through a provider adapter using browser-initiated authorization.

The following data is explicitly ephemeral:

- active terminal WebSocket connections;
- sandbox filesystem state after an attempt is graded;
- in-memory frontier and review calculations;
- short-lived session, CSRF, OAuth `state`, and PKCE verifier values;
- queues used only to coordinate work during the current process lifetime;
- redacted diagnostic logs that have not been deliberately exported.

The following data must be durably written before a successful session ends:

- attempt and hint evidence;
- current per-skill mastery and memory state;
- diagnostic inferences and their confidence;
- quiz/review receipts and deterministic generation keys;
- blocked-path and remediation status;
- graph, policy, scheduler, task, and grader versions used for each decision;
- the remote store revision used for concurrency control.

No correctness-critical workflow may depend on an in-memory background job completing later. A write is acknowledged to the learner only after the remote provider confirms it. If the provider is unavailable, the browser keeps an encrypted pending batch and the UI clearly marks the session as unsynced; mastery unlocks that depend on the batch remain provisional.

## 4. Three-VM Reference Architecture

### Preferred Three-VM Topology

#### VM 1: Stateless Web and Decision Plane

Run replaceable replicas of these logical modules in one process or a small set of containers:

- static web client and API/backend-for-frontend;
- OAuth callback and provider adapter endpoints when a provider requires a confidential exchange;
- system-bundle loader and graph query engine;
- diagnostic, mastery, remediation, frontier, synthesis, and review algorithms;
- attempt orchestration and learner-safe feedback;
- short-lived in-memory coordination and rate limiting.

This is a modular monolith, not a collection of network microservices. It has no PostgreSQL, Redis, graph database, or durable message broker. A second process may run on the same VM for fault separation, but all authoritative state remains remote.

#### VM 2: Sandbox Host A

Run the sandbox control agent, terminal bridge, immutable image cache, and isolated learner workloads. Use hardened containers for ordinary shell/filesystem tasks and a micro-VM or user-space-kernel boundary for tasks that need services, broader capabilities, or complex networking.

#### VM 3: Sandbox Host B and Grader

Run a second sandbox host for capacity and failover plus the trusted grader agent. Grading occurs through a control path unavailable inside the learner environment. VM 3 can grade work on either sandbox host, and VM 2 can carry a standby grader so the system degrades gracefully if VM 3 is unavailable.

### Optional VMs 4 and 5

- **VM 4:** additional sandbox capacity when measured queue time justifies it.
- **VM 5:** warm standby for the web plane or a dedicated grader/observability host if fault isolation becomes more valuable than sandbox capacity.

The system must not assume VMs 4 or 5 exist. A load balancer or DNS failover may be managed by the hosting provider and does not contain learner state. Metrics can be shipped to an external observability service; an always-on telemetry VM is not required.

### Why Three VMs Are Enough

- Graph traversal, FSRS calculation, and recommendation ranking are lightweight and run on VM 1.
- The graph is a read-only directed acyclic graph (DAG) loaded into memory; a graph database is unnecessary at this scale.
- Remote learner documents replace the transactional application database.
- On-demand calculations replace background review workers and durable queues.
- Two sandbox hosts allow rolling maintenance and prevent a single practice workload from consuming all compute.
- Logical boundaries remain explicit in code, so a later deployment can separate them without redesigning the data contracts.

### Session Data Flow

1. The browser loads the web application and connects a state provider.
2. The provider authorizes the browser or the stateless OAuth callback; credentials remain browser-held whenever feasible.
3. VM 1 fetches the learner manifest and latest snapshot through a narrow provider capability supplied for that request.
4. VM 1 verifies integrity, migrates data in memory if needed, folds any unapplied events, and loads the matching graph/policy bundle.
5. The decision engine calculates due reviews, the frontier, remediation probes, and ranked next activities.
6. VM 1 assigns a task and asks VM 2 or VM 3 for a fresh sandbox.
7. The browser connects directly to the selected terminal gateway using a single-use, task-scoped token.
8. On submission, the trusted grader returns structured findings to VM 1.
9. VM 1 builds an immutable evidence event, recalculates affected skill states, and requests a compare-and-swap remote commit.
10. The UI reports a durable result only after that commit succeeds. The sandbox is then destroyed.

## 5. Bring-Your-Own Remote State

### Provider Requirements

A usable provider adapter needs:

- browser-initiated authorization with least-privilege scopes;
- private objects or repositories by default;
- read, create, update, and list operations;
- a revision identifier such as an object ETag, Git blob/commit SHA, or generation number;
- conditional writes or enough primitives to implement compare-and-swap safely;
- reasonable object size, request rate, availability, and exportability;
- a clear revocation path controlled by the learner.

The provider interface exposed to the runtime is deliberately small:

```text
connect() -> ProviderSession
read(path, revision?) -> {bytes, revision}
compare_and_swap(path, expected_revision, bytes) -> new_revision | conflict
list(prefix, cursor?) -> page
revoke() -> void
```

Provider credentials must never be injected into the learner sandbox. A sandbox receives only a task-scoped token that can connect to its terminal; it cannot read or mutate learner progress.

### Recommended MVP: Private GitHub Repository

Use a dedicated private repository such as `linux-mastery-state` as the first provider implementation. The learner authorizes access from the browser, selects or creates the repository, and the application stores ordinary JSON/JSONL files on a dedicated branch.

Benefits include learner ownership, version history, exportability, conflict visibility, mature authorization, and commit SHAs that work as concurrency tokens. Costs include API rate limits, higher write latency than a database, awkward handling of very large event histories, and the need to keep state files free of terminal transcripts or secrets.

Authentication should use a narrowly scoped GitHub App installation or OAuth flow appropriate to the deployed client. If token exchange requires a client secret, VM 1 exposes a stateless callback: OAuth `state` and any PKCE verifier are bound to an encrypted, short-lived browser cookie; the resulting user token is returned to browser memory or an encrypted browser store and is not written to disk on VM 1. Prefer repository-scoped permissions and never request broad organization access when a single state repository is sufficient.

The browser may call the provider directly when its OAuth and CORS model permits. Otherwise it sends a short-lived capability to VM 1 for a single provider operation. A small stateless edge function is also acceptable for OAuth exchange, but it must not become a hidden system-of-record.

### Cloudflare Option

Cloudflare R2, KV, or Durable Objects can back the same provider interface, but they are not automatically equivalent to “learner signs in with OAuth and owns the data.” R2 commonly uses scoped API/S3 credentials, while KV and Durable Objects are normally reached through a Worker. Therefore the Cloudflare option needs a learner-facing identity provider plus a stateless Worker that maps that identity to a learner-controlled namespace.

Use Cloudflare only if one of these ownership models is explicit:

- the learner supplies narrowly scoped, revocable storage credentials in the browser;
- the learner deploys their own Worker/storage namespace; or
- the application operates the Worker while giving the learner export and deletion controls, acknowledging that this is application-managed rather than truly learner-owned storage.

Do not place long-lived R2 or account-wide API credentials in local storage. If Cloudflare is chosen after the MVP, prefer short-lived scoped capabilities issued by a Worker and conditional object writes using ETags or explicit version objects.

### Local Browser Responsibilities

The browser is the continuity point between a transient VM and the learner's provider. It may retain:

- the selected provider and remote path;
- an encrypted refresh token or revocable provider session, if the provider permits it;
- the latest verified snapshot revision;
- a bounded encrypted outbox of unsynced evidence events;
- UI preferences and accessibility settings.

Secrets should use platform credential facilities where available. Plain `localStorage` is not an acceptable default for long-lived tokens because any successful script injection can read it. A stricter deployment can keep tokens only in memory and require reconnecting the provider each session.

### Alternative Providers

The adapter boundary can later support a GitLab/Codeberg private repository, an S3-compatible bucket with learner-scoped credentials, WebDAV, or an encrypted file in a user-selected drive. Each adapter must pass the same conflict, revocation, privacy, and crash-recovery contract tests before it can be used for mastery decisions.

## 6. Durable Learner-State Model

### Remote Layout

For a Git-backed implementation:

```text
linux-mastery-state/
  manifest.json
  snapshots/current.json
  events/2026/08/events-000001.jsonl
  receipts/reviews/2026-08.jsonl
  conflicts/                 # written only when automatic merge is unsafe
```

`manifest.json` contains the learner-state schema version, learner pseudonymous ID, graph version, active snapshot path and hash, event high-water mark, scheduler/policy versions, created/updated timestamps, and optional integrity metadata. It contains no OAuth token.

### Snapshot Shape

```json
{
  "schemaVersion": 1,
  "learnerId": "opaque-user-id",
  "graphVersion": "linux-graph-2026.08",
  "policyVersion": "mastery-v1",
  "schedulerVersion": "fsrs-v1",
  "lastEventSequence": 184,
  "skills": {
    "fs.permissions.octal": {
      "state": "mastered",
      "masteryProbability": 0.94,
      "confidence": 0.88,
      "independentStreak": 3,
      "lastEvidenceAt": "2026-08-30T14:05:12Z",
      "lastIndependentSuccessAt": "2026-08-30T14:05:12Z",
      "blockedReason": null,
      "memory": {
        "difficulty": 5.1,
        "stabilityDays": 12.4,
        "lastReviewedAt": "2026-08-30T14:05:12Z",
        "dueAt": "2026-09-11T23:41:00Z"
      }
    }
  },
  "diagnostic": {"completedAt": null, "inferredSkills": []},
  "pendingRemediations": [],
  "generationReceipts": {}
}
```

The snapshot is a rebuildable cache. Immutable evidence events are authoritative. Compact old events only after writing a checkpoint containing the prior event range, root hash, and snapshot hash.

### Evidence Event

Every attempt produces one event with:

- globally unique event ID and monotonically allocated sequence;
- learner, session, task, variant seed, and primary skill IDs;
- directly exercised subskill IDs and observable checks for each;
- context: `diagnostic`, `guided`, `mastery`, `review`, or `quiz`;
- result, hint level, independence, duration, and grader findings;
- client start time, server-observed time, provider commit time, and monotonic elapsed duration;
- graph, task, grader, mastery-policy, and scheduler versions;
- prior snapshot revision and an idempotency key;
- environment health and `infrastructure_error` classification.

Client wall-clock time is never trusted alone for spacing. Prefer provider commit time or a signed time returned by VM 1. Store all timestamps in UTC RFC 3339 form and record the learner's display timezone separately.

### Idempotent Commit Protocol

1. Create `event_id = UUIDv7()` once in the browser and reuse it on retries.
2. Derive `idempotency_key = SHA-256(learner_id || event_id || task_version || submission_digest)`.
3. Read the manifest/snapshot at remote revision `R`.
4. If the event ID or key is already present, return the existing durable result.
5. Fold the event into a new snapshot and append it to the current event segment.
6. Commit all changed files against expected revision `R`.
7. On a conflict, read the new head, union immutable events by event ID, sort by logical sequence/provider time, recompute the snapshot, and retry with bounded exponential backoff.
8. If two events cannot be ordered without changing a mastery decision, retain both, mark the affected skill `needs_reconciliation`, and avoid unlocking dependents until deterministic replay resolves it.

For GitHub, one commit should contain the event append, snapshot, and manifest update so the branch head SHA is the atomic revision. Do not perform independent writes that can expose a snapshot without its evidence or vice versa.

### Integrity and Privacy

- Hash every event segment and include its hash in the manifest.
- Optionally sign manifests with a browser-held key so an unexpected remote edit is detectable.
- Treat learner edits as recoverable conflicts rather than silently overwriting them.
- Store no raw provider token, shell history, task secret, or full sandbox image in learner state.
- Make state portable JSON with documented versions; the learner must be able to clone or download it.

## 7. Knowledge Graph Deep Dive

### Why the Graph Is the Core Model

The graph determines what the system tests, what it can infer, when a learner is ready, which failure triggers remediation, and which new task can substitute for review. A weak graph produces false mastery and frustrating blocks even if every other component works correctly. The graph must represent observable capabilities and evidence relationships, not a table of contents or a list of commands.

The runtime consumes a versioned DAG. Nodes are granular skills; directed edges point from a skill to what it requires. Additional non-gating relationships express transfer, alternatives, and subskill exercise.

### Node Design Rules

A well-formed skill node:

- begins with an observable verb and describes an outcome;
- can be demonstrated in a bounded sandbox task;
- is narrow enough that failure suggests a useful next probe;
- is broad enough to support multiple task variants and solution methods;
- separates conceptual interpretation from mechanical execution when each can fail independently;
- declares platform/distribution scope rather than hiding it in the label;
- has stable semantic identity even when wording or tasks change.

Good nodes include “interpret symbolic and octal file modes,” “redirect standard error independently,” and “diagnose why a service failed to start.” Avoid “know `chmod`,” “understand networking,” or one node that combines navigation, editing, permissions, and service recovery.

### Node Schema

```yaml
id: fs.permissions.set-octal
title: Set file permissions using octal modes
objective: Given a required access policy, set and verify the corresponding mode.
domain: filesystem
scope:
  distributions: [portable]
  shell: posix
difficulty: 2
criticality: high
evidence_contract:
  required_observations:
    - final mode matches policy
    - learner independently verifies mode
  disallowed_mastery_evidence:
    - solution revealed
    - infrastructure unhealthy
mastery_policy: default-operational-v1
tags: [permissions, security, troubleshooting]
```

Task identifiers do not belong in the node's semantic ID. Tasks reference nodes, allowing task variants to change without rewriting learner history.

### Edge Types

- `requires`: hard gating prerequisite; all such parents must meet the threshold.
- `recommended_before`: helpful sequencing preference that does not lock the node.
- `concept_of`: links an execution skill to a mental model without implying direction by itself.
- `variant_of`: distribution/tool-specific implementation of a shared capability.
- `exercises`: a task or advanced skill supplies retrieval evidence for an older subskill.
- `remediates_to`: maps a misconception or failed check to the best diagnostic skill.
- `alternative_to`: equivalent capability or tool path that can satisfy a declared requirement.

Only `requires` edges participate in DAG frontier gating. Each hard edge needs a written rationale and a counterfactual test: “Could a learner reliably demonstrate the child without this parent?” If yes, the edge is probably recommended rather than required.

### Step-by-Step Graph Composition Process

#### Step 1: Define Exit Performances

Start with representative operational outcomes for a junior engineer, such as repairing access to a file, tracing a failed process, configuring an SSH connection, or diagnosing a service. These are target performances, not course chapters.

#### Step 2: Collect Authentic Scenarios

For each outcome, list several meaningfully different scenarios and valid solution approaches. Record the states a grader can actually observe. If there is no observable evidence contract, the proposed skill is not ready to become a node.

#### Step 3: Perform Cognitive Task Analysis

Walk backward through each scenario and separate:

- facts that must be recalled;
- state that must be interpreted;
- commands or configuration that must be produced;
- choices between valid tools;
- verification and safety behavior;
- recovery actions after expected errors.

Ask experienced engineers where juniors commonly fail, then verify those assumptions with think-aloud diagnostic sessions.

#### Step 4: Propose Atomic Capabilities

Turn the analysis into verb-led candidates. Split a candidate when its parts have different prerequisites, can be remediated separately, or require different evidence. Merge candidates when they are never meaningfully observed apart and splitting would create artificial gates.

#### Step 5: Normalize Identity and Scope

Give every node a stable namespaced ID, concise title, objective, distribution scope, difficulty, criticality, and evidence contract. Model distro-specific implementations as variants beneath a portable capability where possible.

#### Step 6: Add Hard Prerequisites Conservatively

For each candidate child, add only prerequisites necessary across normal valid solution paths. Apply the counterfactual test, record the edge rationale, and avoid adding an edge solely because a textbook traditionally teaches one topic first.

#### Step 7: Add Non-Gating Relationships

Record recommended order, misconception remediation, alternative tools, and transfer/exercise relationships separately. This prevents a useful semantic graph from becoming an over-constrained prerequisite graph.

#### Step 8: Attach Evidence Contracts

For every node, define what independent success looks like, which grader observations attribute evidence to it, which hints invalidate a mastery attempt, required variant diversity, and whether speed is part of fluency. A task may exercise many skills but must report direct observations per skill before it can update them.

#### Step 9: Validate the Graph Mechanically

Reject bundles with:

- cycles among `requires` edges;
- dangling IDs or duplicate semantic IDs;
- unreachable nodes that are intended to be learnable;
- root nodes whose assumed entry capability is undocumented;
- hard edges without rationale or thresholds;
- nodes without an evidence contract;
- nodes whose only assessment requires an unmodeled prerequisite;
- `exercises` relationships with no observable subskill check.

Compute roots, leaves, longest paths, dominators, high fan-in/fan-out nodes, articulation bottlenecks, and frontier sizes for synthetic profiles. Bottleneck nodes deserve extra scrutiny because one incorrect edge can block a large portion of the system.

#### Step 10: Validate with People and Data

Give unseen scenario variants to learners with different backgrounds. Compare predicted prerequisites with actual failure sequences. Look for learners who succeed on a child while failing a purported requirement, or repeatedly fail because of an unmodeled skill. Those are edge-quality signals, not merely learner errors.

#### Step 11: Calibrate Without Losing Explainability

Start with deterministic thresholds. After enough attempts, estimate how strongly parent mastery predicts child success and how reliably each item measures its declared node. Use those statistics to propose graph changes, but keep explicit rationales and human review for all gating edges.

#### Step 12: Version and Migrate

Publish the graph as an immutable version. A new version includes a machine-readable migration map:

- unchanged node: carry state forward;
- renamed node: remap ID without altering evidence;
- split node: copy prior state as low-confidence inferred evidence and schedule probes for each new node;
- merged node: combine evidence conservatively using the weakest required component;
- removed node: retain historical evidence but exclude it from frontier calculations;
- changed hard edge: recalculate frontier without rewriting old attempt events.

Never silently reinterpret historical evidence under a new semantic definition.

### Example Subgraph

```text
fs.listing.read-long-format ─requires→ fs.paths.interpret
fs.permissions.interpret-symbolic ─requires→ fs.listing.read-long-format
fs.permissions.set-octal ─requires→ fs.permissions.interpret-symbolic

shell.redirection.stdout ─requires→ shell.execute-command
shell.redirection.stderr ─requires→ shell.redirection.stdout

text.find.files-by-time ─recommended_before→ text.find.files-by-size
service.diagnose-failed-unit ─exercises→ shell.redirection.stderr
misconception.permission-denied ─remediates_to→ fs.permissions.interpret-symbolic
```

In this example, `A ─requires→ B` means that A is the dependent skill and B is its prerequisite. The serialized edge format must use one orientation consistently, and graph validation tests should make the convention explicit.

### Personal Knowledge Overlay

The base graph never mutates per learner. The learner snapshot supplies a keyed overlay for every observed node:

- state: `unknown`, `diagnosing`, `ready`, `learning`, `practicing`, `mastered`, `due`, `decayed`, `remediation`, or `blocked`;
- mastery probability and confidence;
- direct versus inferred evidence counts;
- independent success streak and hint dependence;
- last evidence and last successful retrieval timestamps;
- FSRS memory parameters and due timestamp;
- misconception and blocked-path metadata;
- graph/policy version under which the state was calculated.

Derived states such as `ready` and `due` can be recalculated on load. They need not be trusted merely because an old snapshot contains them.

### Frontier Query

For node `n`, eligibility is:

```text
eligible(n) =
  not mastered_and_retainable(n)
  AND every required parent p satisfies mastery_threshold(p)
  AND no unresolved reconciliation affects n or a required ancestor
  AND a compatible task and sandbox are available
```

Rank eligible nodes by learner goal relevance, downstream unlock value, due-subskill coverage, estimated effort, recent frustration, and domain diversity. Persist the inputs and score components with the selected activity so the recommendation can be explained later.

### Diagnostic Traversal

Begin with high-information anchor nodes across domains. On clean independent success, probe a harder descendant or adjacent branch. On failure, walk toward the most discriminating required parent or `remediates_to` target. Stop when branch confidence is sufficient or the diagnostic time budget expires. Ancestors inferred from a child success receive less confidence than directly observed skills and must eventually be sampled.

## 8. Lessons, Practice, and Mastery Runtime

### Minimum-Effective-Dose Lesson Loop

The runtime consumes an existing lesson/task bundle and presents:

1. one operational objective and its motivation;
2. one to three minutes of explanation or a worked example;
3. a guided task with graduated optional hints;
4. immediate state-based feedback;
5. new variants with fading command-level scaffolding;
6. independent attempts in fresh sandboxes;
7. a short mental-model summary and expected review window.

Complex safety explanations can exceed the nominal time budget, but active work remains the primary mode.

### Mastery Gate

Use a versioned, configurable rule. A reasonable default requires:

- three consecutive successful independent variants;
- no solution reveal or substantive hint on counted attempts;
- material variation in paths, names, values, or topology;
- one successful delayed retrieval at least 24 hours after instruction;
- completion within a calibrated limit only where fluency is part of the objective;
- no critical unsafe action.

A failure resets the current streak but does not delete earlier evidence. Infrastructure errors never count as learner failures.

### Failure and Parallel Paths

- A likely slip receives another isomorphic variant after brief feedback.
- A recognizable misconception receives a contrastive explanation and targeted probe.
- A prerequisite gap marks that node for remediation and pauses the dependent path.
- Repeated failure keeps the vertical path closed but recommends a frontier node in another branch.
- The UI explains what evidence will reopen the blocked path.

### Outcome-Based Grading

Grade final state and behavior, allowing multiple correct approaches. For “create user `devops` with administrative privileges,” inspect account existence, group membership, authentication state, and relevant configuration rather than matching the command history.

A grader returns `pass`, `fail`, `inconclusive`, or `infrastructure_error`, plus check IDs, learner-safe feedback, misconception candidates, critical safety findings, measured duration, environment health, and all relevant versions. Graders are idempotent, time-bounded, isolated from learner modification, and tested against both valid alternative solutions and adversarial shortcuts.

## 9. On-Demand Spaced Review

### No Background Scheduler

There is no durable review worker. Whenever the learner opens the application, requests the next activity, or finishes an attempt, VM 1 loads the latest snapshot and computes review eligibility using the current trusted time. A cached `dueAt` accelerates the calculation, but FSRS memory parameters and immutable evidence allow it to be reproduced.

Store, per skill, at minimum:

- difficulty and stability;
- last successful independent retrieval time;
- last review grade and evidence strength;
- calculated due time;
- scheduler version;
- operational criticality and maximum interval policy.

Successful delayed transfer is stronger evidence than guided same-session repetition. Failure contracts the interval, clears an independent streak where appropriate, and may reopen remediation. One poor timed quiz should not erase mastery without corroborating evidence.

### Deterministic Activity Generation

Generate a review session from a stable input tuple:

```text
generation_key = SHA-256(
  learner_id || snapshot_revision || graph_version || scheduler_version ||
  UTC_review_window || sorted_due_skill_ids || session_mode
)
```

Before generating, check `generationReceipts[generation_key]`. If present, return the prior activity IDs. If absent, use the key as the deterministic random seed, select task variants, and commit the receipt with the expected snapshot revision. Concurrent tabs that generate the same session either receive the committed result or re-read after a compare-and-swap conflict.

Use a review window such as the learner's local calendar day only for grouping; all due comparisons use trusted UTC instants. A learner can explicitly start another practice set, but it receives a different mode/counter and cannot create duplicate scheduled credit.

### Broad-Coverage Quizzes

Weekly quizzes use a deterministic blueprint across domains, evidence ages, criticality, and difficulty. Hints and solution reveals are disabled until submission. “Closed-book fluency” mode disables in-platform help, while realistic operations mode may allow local `man` pages. Timers have configurable accommodations and measure fluency only where the graph node's evidence contract declares it relevant.

### Review by Learning New Material

Before assigning direct review, search frontier tasks tagged as exercising due subskills. A new task satisfies an old review only if:

- the old skill is necessary rather than incidental;
- no hint is provided for that subskill;
- the grader directly observes a check attributable to it;
- spacing and difficulty rules are satisfied; and
- the overall task succeeds.

Select synthesis tasks by maximizing weighted due-skill coverage plus frontier value while limiting session time, overload, and repeated contexts. If no trustworthy synthesis task exists, assign direct retrieval; adjacency alone never earns review credit.

## 10. Ephemeral Sandbox Design

### Isolation and Lifecycle

1. Pull only signed, scanned, immutable sandbox images.
2. Keep a small warm pool for common templates on VMs 2 and 3.
3. Allocate a clean environment per counted attempt.
4. Inject only short-lived, task-scoped secrets.
5. Stream the terminal through an authenticated gateway; expose no host port directly.
6. Grade through a privileged channel unavailable to the learner.
7. export only structured evidence and explicitly approved redacted artifacts.
8. destroy the environment and reconcile orphans by TTL.

Do not expose the host container socket, hypervisor control API, grader credentials, provider token, cloud metadata service, or another learner's network. Default-deny egress; provide simulated DNS, HTTP, SSH, and failure services inside the task network. Enforce CPU, memory, storage, process, bandwidth, duration, and concurrent-session limits.

### Terminal and Accessibility

Use an accessible web terminal over authenticated WebSockets. Support keyboard-only operation, high contrast, font scaling, screen-reader-friendly instructions outside the terminal, reconnect with a bounded grace period, and an alternative transcript view. GUI/VNC is exceptional; the core system remains terminal-first.

### Host Scheduling

VM 1 chooses the least-loaded healthy sandbox host that supports the task capability profile. If both hosts are full, queue with an honest position and timeout rather than overcommitting. A counted attempt is pinned to one host, while the trusted task manifest and event ID allow safe restart before the attempt begins. Host loss during an active attempt is an infrastructure error and creates no negative evidence.

## 11. Security and Privacy

- Threat-model every learner-controlled shell as hostile.
- Keep OAuth/provider credentials in the browser or stateless credential exchange path, never in sandboxes or logs.
- Request the minimum provider scope and give the learner a visible disconnect/revoke control.
- Bind OAuth redirects with expiring `state`; use PKCE where supported; validate exact redirect URIs and provider identity.
- Apply a strict content security policy and dependency controls because browser script compromise can expose learner-held credentials.
- Encrypt transport and remote objects where provider privacy is insufficient.
- Separate terminal transport tokens, task capabilities, and state-provider capabilities.
- Redact commands and environment values before exporting diagnostics.
- Record which data is stored, where, for how long, and how the learner deletes or exports it.
- Avoid gaze tracking, biometrics, or claims of technically enforced closed-book behavior.
- Run sandbox breakout tests, image scans, secret scans, restore drills, and incident-response exercises.

## 12. Reliability and Observability

### Initial Targets

- p95 web-plane response under 500 ms excluding provider and sandbox operations.
- p95 warm sandbox readiness under 5 seconds and cold readiness under 20 seconds.
- p95 terminal echo under 150 ms in supported regions.
- p95 standard grader completion under 5 seconds after submission.
- fewer than 0.5% infrastructure-error attempts and fewer than 0.1% orphaned sandboxes.
- remote state conflicts automatically reconciled in at least 99.9% of benign multi-tab cases.
- no mastery result reported as durable before provider commit confirmation.

### External Telemetry

Ship privacy-redacted metrics, traces, and logs to a managed observability endpoint so no fourth VM is required. Trace from state load through graph calculation, sandbox allocation, grading, and compare-and-swap commit. Track provider latency/error/rate-limit behavior, revision conflicts, outbox age, frontier size, sandbox queue depth, grader disagreement, review calibration, and mastery-to-delayed-transfer correlation.

If external telemetry is unavailable, the learning session continues; telemetry must never become a correctness dependency. Security audit events relevant to the learner can be appended to their remote state, while platform-wide operational events follow a short, documented retention policy.

## 13. Implementation Roadmap

### Phase 0: Graph and State Proofs (4–6 weeks)

- Compose a 50–75 node foundation graph using the process in this plan.
- Build graph linting, migration, synthetic-profile, and frontier tests.
- Implement the provider contract with an in-memory fake and one private GitHub repository adapter.
- Prove append, snapshot, compare-and-swap conflict merge, idempotent retry, and offline outbox recovery.
- Build 10 representative task/grader bundles solely to validate graph evidence and sandbox requirements.
- Compare container and micro-VM boundaries on two sandbox hosts.

**Exit criteria:** no cycles or unexplained hard edges; deterministic replay produces the same profile; duplicate submissions do not duplicate evidence; concurrent writes converge; golden grader runs are at least 95% repeatable; and the application survives replacement of VM 1 between sessions.

### Phase 1: Three-VM MVP (8–12 weeks)

- Deploy the modular web/decision plane on VM 1 and sandbox agents on VMs 2 and 3.
- Deliver browser provider connection, state import/export, and visible sync status.
- Implement diagnostic traversal, personal graph overlay, frontier ranking, mastery gates, remediation, and parallel paths.
- Deliver terminal sessions, deterministic grading, durable evidence commits, and clean sandbox destruction.
- Implement on-demand FSRS calculation and deterministic review receipts.

**Exit criteria:** a new VM 1 reconstructs the same learner recommendations from remote state; no local database or background scheduler is required; provider credentials never enter a sandbox; and the full diagnose-learn-master-review loop works within three VMs.

### Phase 2: Retention and Synthesis (6–10 weeks)

- Add delayed retrieval, weekly broad quizzes, workload caps, and clock-skew tests.
- Implement evidence-backed synthesis selection for due subskills.
- Add graph-version migrations for rename, split, merge, removal, and edge changes.
- Calibrate scheduler parameters and graph edges against delayed transfer data.
- Add a second provider adapter only after the first adapter's consistency behavior is stable.

### Phase 3: Optional Scale and Hardening

- Add VM 4 for sandbox capacity only if queue metrics justify it.
- Add VM 5 for web-plane standby or grader separation only if availability data justifies it.
- Strengthen multi-region failover, credential isolation, artifact signing, and adversarial sandbox testing.
- Evaluate richer statistical learner models only when they outperform transparent rules on unseen transfer.

## 14. Testing Strategy

### Knowledge Graph Tests

- Schema, stable-ID, edge-rationale, and evidence-contract validation.
- Cycle, reachability, bottleneck, orphan, and hidden-prerequisite analysis.
- Property tests: a node never enters the frontier while a required parent is below threshold.
- Migration tests for every graph change type using historical learner fixtures.
- Diagnostic simulations over novice, expert, contradictory, and sparse profiles.
- Human think-aloud validation and delayed unseen transfer checks.

### State and Provider Tests

- Provider contract suite against fake, GitHub, and any later adapter.
- Duplicate event, retry-after-timeout, concurrent-tab, and stale-revision tests.
- Crash tests before and after each step of the remote commit protocol.
- Corrupt snapshot, missing event segment, manual remote edit, and migration recovery tests.
- OAuth state/PKCE, revocation, scope, token leakage, CORS, and browser-storage tests.
- Deterministic review generation across restarts, timezones, clock skew, and scheduler versions.

### Sandbox and Grader Tests

- Multiple golden solutions and known incorrect solutions for every grader.
- Repeatability across fresh sandboxes, seeds, and supported distributions.
- Attempts to tamper with graders, escape isolation, consume resources, and reach forbidden networks.
- Host-loss, terminal-reconnect, cleanup, and orphan-reconciliation tests.
- Accessibility testing with keyboard-only and screen-reader workflows.

### End-to-End Tests

- Replace VM 1 between state write and next login; verify identical reconstructed state.
- Shut down either sandbox host; verify queue/failover and infrastructure-error handling.
- Run the complete system with exactly three VMs and no local durable services.
- Verify no fourth or fifth VM is assumed by deployment manifests or health checks.
- Disconnect the provider during submission, recover from the browser outbox, and prove exactly one event is committed.

## 15. Key Risks and Mitigations

| Risk | Impact | Mitigation |
| --- | --- | --- |
| Incorrect hard prerequisite | Learner is falsely blocked | Conservative edges, written counterfactual rationale, learner data, migration support |
| Hidden prerequisite | Repeated unexplained failure | Think-aloud analysis, grader check attribution, remediation telemetry |
| Provider outage or rate limit | Progress cannot be confirmed | Browser outbox, backoff, visible provisional state, compact writes |
| Conflicting tabs/devices | Lost or double-counted evidence | Immutable event IDs, compare-and-swap, deterministic replay and merge |
| Learner edits remote files | Corrupt or misleading profile | Hashes/signatures, validation, recovery branch, no silent overwrite |
| Browser token theft | Remote state exposure | CSP, in-memory tokens, narrow scope, revocation, optional client encryption |
| Cloud storage is not truly learner-owned | Misleading portability claim | Explicit ownership model, provider contract, export/delete controls |
| Sandbox escape or abuse | Host/provider compromise | Strong isolation, default-deny egress, quotas, scanning, kill switch |
| Fixed VM capacity is exhausted | Long queues or instability | Two sandbox hosts, warm-pool tuning, honest queue, strict TTLs, optional VM 4 |
| Review clock manipulation | Early/late or duplicate work | trusted UTC time, provider timestamps, signed time, generation receipts |
| Synthesis grants weak review credit | Apparent retention without retrieval | necessary subskill, independent attempt, direct grader observation |

## 16. Recommended Defaults

1. **Deployment:** one stateless web/decision VM and two sandbox/grader VMs.
2. **Durable store:** learner-owned private GitHub repository for the MVP.
3. **Credentials:** browser-held, repository-scoped authorization; stateless exchange endpoint only when required.
4. **Graph engine:** immutable JSON/YAML bundle loaded into memory; no graph database.
5. **Learner events:** append-only JSONL plus a rebuildable JSON snapshot and atomic Git commit.
6. **Concurrency:** branch-head SHA as the compare-and-swap revision; union events and replay after benign conflicts.
7. **Review:** FSRS calculated on demand, with deterministic generation receipts and no background queue.
8. **Mastery:** three independent successes plus a delayed retrieval, calibrated against transfer.
9. **Isolation:** hardened containers for basic tasks and micro-VM/user-space-kernel isolation for privileged scenarios.
10. **Provider expansion:** add Cloudflare or another adapter only after its learner ownership and short-lived credential model are explicit.

## 17. MVP Acceptance Criteria

The MVP is ready for a supervised pilot when:

- It runs end to end on three VMs and requires no platform-owned database, graph store, Redis, or durable queue.
- Destroying and recreating VM 1 does not alter the learner's reconstructed profile, frontier, due reviews, or prior receipts.
- A learner can connect and revoke a private remote state provider through a browser-initiated authorization flow.
- Provider credentials never appear in a learner sandbox, task bundle, terminal transcript, or application log.
- Remote writes are idempotent, conditional on revision, crash-safe, and automatically merge benign concurrent events.
- The graph has stable observable nodes, conservative typed edges, evidence contracts, mechanical validation, and versioned migrations.
- Diagnostic results create an explainable personal overlay without mutating the base graph.
- Only nodes whose required parents meet policy enter the knowledge frontier.
- Hint-assisted attempts cannot accidentally award independent mastery.
- Failed paths remain gated while useful parallel frontier paths remain available.
- Review activities are generated deterministically on demand from durable timestamps and memory state.
- A synthesis task grants old-skill credit only from direct, independent grader observations.
- Infrastructure and provider errors never count as learner failures.
- Both sandbox hosts enforce isolation, quotas, default-deny egress, expiration, and reliable cleanup.
- Deployment and failure tests prove that VMs 4 and 5 are optional rather than hidden dependencies.

## 18. Definition of Success

The system succeeds when a disposable three-VM deployment can load an immutable Linux knowledge graph, reconstruct a learner's history from their own remote state, choose and run the right sandbox activity, commit evidence exactly once, and disappear without losing the ability to continue later. The knowledge graph must remain explainable and empirically correct enough that advancement reflects genuine readiness; remote persistence must remain portable and conflict-safe enough that stateless compute does not weaken mastery or review guarantees.

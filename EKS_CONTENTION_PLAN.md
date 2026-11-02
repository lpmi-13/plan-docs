# Scalescape: Visualizing Placement Contention Between Pod Autoscaling and Karpenter on EKS — Plan

## 1. Product Summary

Build a browser-based, three.js-powered explorable explanation of what happens inside an EKS cluster when **pod autoscaling** (the Horizontal Pod Autoscaler, and KEDA, which drives an HPA from event sources) and **node autoscaling** (Karpenter) act on the same workloads at the same time. It focuses on the failure modes that appear when the two loops pursue different goals: a pod that one controller asked for and no controller can place, a node that one controller removes while another is about to need it, or a constraint that turns a scale-out into a queue of Pending pods.

The visualization is modeled directly on [Forkscape](https://github.com/lpmi-13/forkscape) (plan: [PROCESS_FORK_VISUALIZATION_PLAN.md](PROCESS_FORK_VISUALIZATION_PLAN.md)). It reuses Forkscape's working shape rather than its original curriculum. That shape is a **scenario library** that opens on arrival. Each scenario names a production symptom and steps through deterministic model states with captions. Every scenario can **replay its fix**: the same setup with one change, at the same step. A pure model drives a 3D scene and a keyboard-accessible flat view from one world state, and small **reference probes** in disposable environments check the model's teaching claims.

The learner sees four connected things at once:

- a **3D scene** in which workloads are decks floating above a floor of availability-zone terraces, nodes are trays on those terraces, pods are blocks packed onto the trays by their resource requests, and Pending pods wait in a harbor between the two while Karpenter's shipyard builds new trays;
- a **lens comparison** showing the same moment as each controller sees it: the HPA (usage as a fraction of requests, ready pods only), the scheduler (requests against allocatable, plus constraints), Karpenter's provisioner (unschedulable pods, NodePool limits, offerings), and Karpenter's disruption controller (candidates, budgets, PDBs);
- a **chart** of desired replicas, ready replicas, Pending pods, nodes, and NodePool usage against its limit over simulated time; and
- an **inspector and event timeline** that ladder each element down to the `kubectl` output, Kubernetes event, or Karpenter log line an engineer would see on a real cluster.

The core promise:

> Pick a production symptom, watch each controller make a locally reasonable decision, see where those decisions collide, and replay the same minutes with one change that resolves it, including what that change costs.

## 2. Why This Matters to an Engineer Supporting Production

On EKS, most teams run pod autoscaling and node autoscaling together and treat them as one mechanism. They are not. The HPA writes a replica count on a 15-second loop from metrics that lag reality. The scheduler places pods using **requests**, not usage. Karpenter only reacts to pods the scheduler has marked **unschedulable**, chooses instances from NodePool requirements and limits, and separately removes nodes it judges underutilized. Kubelet and the VPC CNI then decide whether a scheduled pod actually starts. Each has its own inputs, its own clock, and its own objective: the HPA protects the latency target, Karpenter's consolidation protects the bill, NodePool limits protect the budget, and PDBs and topology spread protect availability.

The incidents that follow are hard to diagnose because each dashboard looks healthy from its own controller's point of view:

- the HPA reports `desired 12, current 12` while three of those replicas have been Pending for an hour;
- Karpenter reports nothing alarming, because the waiting pods are either blocked by a NodePool limit or already bound to a node that cannot give them an IP address;
- nodes churn every few minutes, and each churn produces the CPU spike that justifies the next scale-out; and
- an AMI rollout never finishes, because a PDB sized for the daytime fleet allows zero disruptions overnight.

An engineer who understands the interplay can make decisions that would otherwise be guesswork:

- how to size NodePool limits from HPA `maxReplicas`, requests, DaemonSet overhead, and rollout surge;
- how wide a NodePool's instance requirements must be for a topology spread constraint to be satisfiable during a capacity shortage;
- how to choose `consolidateAfter`, disruption budgets, and PDBs so that cost optimization does not fight the HPA;
- when to buy headroom (placeholder pods) to hide node launch latency, and what it costs;
- which signal to alert on when "desired" and "running" disagree; and
- how to triage a Pending pod from events and controller logs, rather than by raising every limit.

### 2.1 Production symptom map

This table is the backbone of the scenario library (Section 8) and the "Production" and "Evidence" notes on each scenario.

| Production symptom | Mechanism behind it | Evidence to collect | Levers | Scenario |
| --- | --- | --- | --- | --- |
| HPA shows desired = current, but pods have been Pending for an hour and latency is high | The NodePool's `limits` stop Karpenter from launching; the HPA counts Pending pods as replicas and, when scaling up, as idle ones, so its arithmetic says the fleet is on target | `kubectl get hpa` vs `kubectl get deploy` (available); `FailedScheduling` events; Karpenter log about the NodePool limit; `kubectl get nodepool -o yaml` `status.resources` | Size limits from `maxReplicas × request` + overhead + surge; make `maxReplicas` the spend cap; alert on Pending duration | `limit-ceiling` |
| Pods Pending in one zone while nodes in the other zones have free room | `topologySpreadConstraints` with `DoNotSchedule` require the next pod in the short zone; Karpenter cannot launch there because every allowed offering has insufficient capacity | `FailedScheduling` "didn't match pod topology spread constraints"; Karpenter `InsufficientInstanceCapacity` errors; NodeClaims that launch and are deleted | Broaden instance families, sizes, and capacity types; on-demand fallback; consider `ScheduleAnyway` or `minDomains` deliberately | `zone-skew-ice` |
| Pods scheduled but stuck in `ContainerCreating`; Karpenter launches nothing | The VPC CNI cannot get IP addresses from an exhausted subnet. The pods are already bound, so nothing is unschedulable for Karpenter to act on | `FailedCreatePodSandBox` "failed to assign an IP address"; `aws ec2 describe-subnets` `AvailableIpAddressCount`; `aws-node` logs | `WARM_IP_TARGET`/`MINIMUM_IP_TARGET` instead of `WARM_ENI_TARGET`; secondary CIDR (custom networking); prefix delegation; IPv6 | `ip-exhaustion` |
| The first burst after a quiet period waits minutes; KEDA scaled immediately | KEDA activates from zero, then the pod waits for Karpenter's batch window, an EC2 launch, node registration and initialization, and a large image pull | Pod `PodScheduled` → `Ready` timestamps; NodeClaim `Launched`/`Registered`/`Initialized` conditions; image pull events | Placeholder pods at negative priority that pre-pull the image; KEDA cron pre-warming; smaller images or lazy loading | `cold-start-relay` |
| Nodes replaced every few minutes; p99 spikes follow each replacement | HPA scale-in leaves nodes underutilized; consolidation evicts pods; restarted pods burn CPU while warming up; the HPA reads the spike as load and scales out again | Karpenter disruption decisions and `Underutilized` reasons; HPA events; pod restarts correlated with node churn | `consolidateAfter` longer than the HPA's scale-in rhythm; disruption budgets by reason or schedule; readiness that waits for warm-up | `consolidation-thrash` |
| Batch jobs restart every time the web tier scales, although the cluster can add nodes | The scheduler preempts lower-priority pods at once, rather than waiting for Karpenter's node; the victims then need the new node anyway | `Preempted` events; job restarts; a NodeClaim launched within the same minute | `preemptionPolicy: Never` for tiers that can wait; headroom so preemption hits placeholders; checkpointing jobs | `preempt-not-wait` |
| An AMI or config rollout never finishes overnight, then disrupts nodes at the morning peak | A PDB with `minAvailable` equal to the HPA's `minReplicas` allows zero disruptions at night; Karpenter skips drifted nodes until the HPA scales up | `kubectl get pdb` (`ALLOWED DISRUPTIONS 0`); NodeClaim `Drifted` condition; Karpenter "cannot disrupt" events | `maxUnavailable: 1` or a percentage; `minReplicas` one above what the PDB needs; budgets that schedule rotation windows | `pdb-blocks-rotation` |
| HPA pinned at `maxReplicas`, nodes saturated on real CPU, and Karpenter adds nothing | A low request makes `maxReplicas × request` a ceiling below real demand; beyond it usage exceeds requests, the scheduler and Karpenter still pack by requests, nodes saturate, and nothing is Pending | `kubectl top pods` vs requests; node CPU saturation and throttling; HPA `ScalingLimited`; earlier consolidation events | Right-size requests from observed usage (VPA recommender in `Off` mode); re-tune the HPA target after resizing | `requests-lie` |

Extension scenarios (Section 8.4) cover memory-based HPA ratchets, spot interruption waves, required anti-affinity fan-out, consolidation evicting KEDA jobs, nodes launched for pods that no longer exist, priority starvation at a limit, VPA and HPA fighting over CPU, DaemonSet squeeze, rollout surge, and metric outages.

### 2.2 What a learner should be able to do on call afterwards

- Look at a Pending pod and name which of the five placement questions (Section 5.2) it fails, from its events alone.
- Explain why "the HPA is at target" and "users see high latency" can both be true, using the HPA's treatment of unready pods.
- Tell a scheduling failure (Pending, unschedulable) from a node-level start failure (`ContainerCreating`) and know which one Karpenter can act on.
- Compute the NodePool limit a given HPA configuration needs, including DaemonSets and rollout surge.
- Predict whether a given `consolidateAfter`, disruption budget, and PDB will let a node rotate tonight.
- Describe the latency ladder from load change to ready pod, and say which rungs headroom removes.

## 3. Reframing "The Autoscalers Will Handle Capacity"

The phrase is the right starting point, and the experience should start there. It then needs refining, because the incidents come from what the phrase leaves out.

**What is true:**

- With the HPA and Karpenter both installed, a load increase usually produces more pods, and more pods that do not fit usually produce more nodes, without human action.
- When load falls, the HPA eventually removes pods, and Karpenter eventually removes or shrinks nodes.

**What needs refining:**

1. **There is no "autoscaler"; there are at least four controllers.** The HPA (or KEDA plus an HPA) sets a replica count. The ReplicaSet controller creates or deletes pods. The scheduler places pods. Karpenter provisions nodes for pods the scheduler could not place, and separately disrupts nodes. Kubelet and the CNI start the pod. None of them plans with the others.
2. **The HPA's job ends at a number.** It writes `spec.replicas` and never learns whether the pods ran. When it scales up, it treats Pending pods as replicas that use nothing, which damps further scale-up while they wait.
3. **Karpenter scales on Pending pods, not on load.** It does not read CPU utilization. A cluster at 100% real CPU with every pod scheduled gives Karpenter nothing to do.
4. **Everything below the HPA counts requests, not usage.** The scheduler fits requests into allocatable; Karpenter sizes nodes and checks limits from requests and capacity; consolidation judges a node's emptiness by the requests of the pods on it. Only the HPA (and kubelet under pressure) looks at usage, and the HPA reads it as a fraction of the request.
5. **A pod's possible locations are an intersection.** The NodePool's requirements, the pod's selectors and affinities, its tolerations, its topology spread, the zone's available offerings, the NodePool's remaining limit, the node's pod count, and the subnet's free IPs all have to agree. Scaling out does not widen any of them.
6. **Removing capacity is scheduling in reverse.** Consolidation, drift, and expiry evict pods, which then go Pending and need placing again, subject to the same constraints plus PDBs, `do-not-disrupt`, and disruption budgets.
7. **Each loop has its own clock.** Seconds for the HPA and the scheduler, about a minute for a node, minutes for an image, five minutes of scale-in stabilization, and whatever `consolidateAfter` says. Most thrash is two clocks beating against each other.
8. **Priority is a capacity policy that acts only when capacity is short.** The scheduler preempts lower-priority pods immediately, even when Karpenter could have a node ready in a minute.

The refined statement the experience builds toward:

> The HPA decides how many pods should exist. The scheduler decides whether each one fits somewhere, using requests. Karpenter adds a node only for pods the scheduler could not place, and only inside NodePool requirements and limits; separately, it removes nodes whose pods fit elsewhere. Each controller is right about its own inputs, and an incident is usually the gap between them.

## 4. Audience, Prerequisites, and Learning Outcomes

### 4.1 Audience

- Platform engineers and SREs operating EKS clusters with Karpenter.
- Application engineers who own an HPA or a KEDA `ScaledObject` and are paged when it does not seem to work.
- Engineers migrating from Cluster Autoscaler who expect Karpenter to behave like node groups.

### 4.2 Prerequisites

- Comfort with `kubectl get`, `describe`, and `logs`.
- Knowing what a Deployment, a pod, a node, and a resource request are.
- A general idea of what an HPA does. No prior Karpenter or KEDA knowledge is required; the scenarios introduce only what they use.

### 4.3 Learning outcomes

After the MVP scenarios, a learner should be able to:

1. Name the controllers between a load change and a ready pod, the input each reads, and the object each writes.
2. Explain how the HPA computes desired replicas, including tolerance, scale-in stabilization, and its treatment of unready and Pending pods.
3. Explain why scheduling and Karpenter use requests, and predict the consequences of requests that are much lower or higher than usage.
4. Describe how Karpenter chooses a NodePool and an instance offering for a batch of Pending pods, and what stops it (requirements, limits, insufficient capacity).
5. Distinguish an unschedulable pod from a scheduled pod that cannot start, and say which controller can act on each.
6. Explain consolidation, drift, and expiry, and how `consolidateAfter`, disruption budgets, PDBs, and `do-not-disrupt` gate them.
7. Explain scheduler preemption and priority, and when preemption is worse than waiting for a node.
8. Size a NodePool limit, a subnet, and a headroom buffer for a given HPA configuration.
9. Diagnose each MVP scenario's symptom from events, controller status, and logs.

## 5. Conceptual Framework

### 5.1 The anchor idea

Two control loops share one queue. The **pod loop** (load → metric → HPA or KEDA → replica count → ReplicaSet → pods) and the **node loop** (unschedulable pods → Karpenter → NodeClaim → node, and underutilized node → disruption → eviction) meet only at the scheduler's queue of Pending pods. Every scenario is an instance of one loop changing the queue faster, slower, or differently than the other loop can respond.

```text
  load ──► metric ──► HPA / KEDA ──► spec.replicas ──► ReplicaSet ──► new pods
   ▲        (lags)    (15 s, 30 s)                                       │
   │                                                                     ▼
 ready pods ◄── kubelet + CNI ◄── bound pods ◄── scheduler ◄──── Pending queue
   │            (IP, image,        (requests,      (seconds)         │   ▲
   │             readiness)         constraints)                     │   │ evicted pods
   │                                                                 ▼   │
   └──── usage ──► (HPA only)           Karpenter provisioner ◄─ unschedulable
                                         (batch 1–10 s, ~1 min launch)
                                        Karpenter disruption ──► evict, terminate
                                         (consolidateAfter, budgets, PDBs)
```

### 5.2 The five placement questions

Every Pending pod, and every pod about to be evicted, can be investigated with the same five questions. The inspector organizes a pod's detail around them, and scenario notes refer to them by number.

1. **Who asked for this pod, and from what signal?** Which HPA or ScaledObject, which metric, how stale, and what it would ask for next.
2. **What does it claim?** Requests (not usage), priority, tolerations, and the image it must pull.
3. **Where is it allowed to go?** Node selectors and affinity, topology spread, anti-affinity, taints, maximum pods per node, and IP addresses on the node's subnet.
4. **Can a node be created for it, and will it be allowed to exist?** NodePool requirements and weights, limits, available offerings per zone and capacity type, and launch latency.
5. **Who can take its place away?** Consolidation, drift, expiry, spot interruption, and preemption, and what holds them back: PDBs, `do-not-disrupt`, disruption budgets, and `consolidateAfter`.

### 5.3 The loops' clocks: the latency ladder

Defaults below are what the model starts from. Each is listed in Section 21 for re-verification against the versions the reference probes use, and the probes measure the AWS-dependent rungs.

| Rung | Owner | Default or typical | Model parameter |
| --- | --- | --- | --- |
| Metric scrape resolution | metrics-server | 15 s (re-verify) | `metricsResolution` |
| HPA sync | kube-controller-manager | 15 s | `hpaSyncPeriod` |
| HPA tolerance | HPA | 10% | `hpaTolerance` |
| HPA scale-up policy | HPA `behavior` | the larger of +100% or +4 pods per 15 s; no stabilization | `behavior.scaleUp` |
| HPA scale-in stabilization | HPA `behavior` | 300 s (highest recommendation in the window wins) | `behavior.scaleDown` |
| HPA CPU initialization period | kube-controller-manager | 5 min, with a 30 s initial readiness delay | `cpuInitializationPeriod` |
| KEDA polling and activation | KEDA operator | `pollingInterval` 30 s; KEDA handles 0 ↔ 1, the HPA it creates handles 1 ↔ N | `kedaPollingInterval` |
| KEDA scale to zero | KEDA operator | `cooldownPeriod` 300 s | `kedaCooldown` |
| Scheduling attempt | kube-scheduler | sub-second per pod; backoff for unschedulable pods | `schedulerBackoff` |
| Provisioning batch window | Karpenter | idle 1 s, maximum 10 s | `batchIdle`, `batchMax` |
| Launch → registered → initialized → Ready | EC2, AMI, kubelet | roughly a minute; measured in Probe E | `nodeLaunch` per AMI family |
| Insufficient-capacity backoff | Karpenter | unavailable offerings cached for a few minutes (re-verify TTL) | `iceCacheTtl` |
| Image pull | kubelet, registry | size-dependent; measured in Probe E | `imagePullMiBps` |
| Readiness | the application | probe settings plus warm-up | per workload |
| Consolidation wait | Karpenter | `consolidateAfter`, from the last pod added to or removed from the node | per NodePool |
| Disruption budget | Karpenter | 10% of nodes when no budget is set | per NodePool |
| Node expiry | Karpenter | `expireAfter` 720 h | per NodePool |

### 5.4 What each controller sees

Forkscape's central accounting idea was that RSS and PSS count the same pages differently, and its comparison callout drew both. Scalescape's equivalent is that **each controller reads a different quantity about the same pods**. The lens comparison (Section 10.6) draws them side by side for the current step.

| Controller | Reads | Ignores | Writes |
| --- | --- | --- | --- |
| HPA | Usage ÷ request for ready pods (or an external metric per replica); counts Pending and unready pods as replicas | Whether replicas are scheduled or running; node capacity; NodePool limits | `spec.replicas`, HPA status and conditions |
| KEDA | Event-source metric (queue depth, lag) on its polling interval; activation threshold | Everything the HPA ignores; whether workers are making progress | Replica count 0 ↔ 1; an HPA for 1 ↔ N |
| ReplicaSet | Desired vs existing pods | Placement; when scaling in, it deletes Pending and unready pods first | Pods |
| Scheduler | Requests vs allocatable, constraints, priority | Usage; nodes that do not exist yet | Binding, `nominatedNodeName`, preemption, `PodScheduled=False` |
| Karpenter provisioner | Unschedulable pods' requests and constraints; NodePool requirements, weights, and limits; offerings and prices | Load and usage; pods that are bound but not running | NodeClaims |
| Karpenter disruption | Requests of pods on each node; whether they fit elsewhere; PDBs, `do-not-disrupt`, budgets, `consolidateAfter` | What the HPA will ask for next | Disruption taints, evictions, replacements |
| Kubelet and VPC CNI | Actual usage, IP addresses, images | Anything above the node | Pod status, sandbox events, evictions under pressure |

### 5.5 Threshold concepts

- **Desired is not running.** Replica count, scheduled pods, started pods, and ready pods are four different numbers, and each controller owns a different gap between them.
- **Requests are the currency below the HPA.** Placement, provisioning, limits, and consolidation all trade in requests; usage matters only to the HPA and to kubelet.
- **Pending is the only door into node scaling.** If a problem does not produce an unschedulable pod, Karpenter cannot help with it.
- **Constraints compose by intersection.** Each added requirement removes options in every later step; none adds capacity.
- **Disruption is scheduling in reverse,** and it competes with scale-out for the same queue.
- **Two clocks, one system.** A stable configuration needs the node loop's waits to be longer than the pod loop's rhythm, or deliberately shorter with headroom.

### 5.6 Misconception inventory

Each misconception is mapped to the scenario step that confronts it (Section 8.5).

| Misconception | Correction | Confronted in |
| --- | --- | --- |
| "Karpenter scales nodes on CPU utilization" | It reacts to unschedulable pods; hot pods that are all scheduled do not trigger it | `requests-lie`, `ip-exhaustion` |
| "If the HPA says 12, 12 pods are serving" | Pending and starting pods count as replicas; ready count can be far lower | `limit-ceiling`, `cold-start-relay` |
| "The HPA would keep scaling up if pods were Pending" | Pending pods are treated as idle on scale-up, which damps the recommendation | `limit-ceiling` |
| "A NodePool limit is a safe spend cap" | It silently converts HPA scale-out into Pending pods with no alert | `limit-ceiling` |
| "A Pending pod means the cluster is full" | It can mean a zone, a capacity type, a limit, a taint, or a spread rule; free capacity elsewhere does not help | `zone-skew-ice` |
| "Pods stuck starting are a scheduling problem" | `ContainerCreating` happens after binding; the scheduler and Karpenter consider it done | `ip-exhaustion` |
| "Consolidation only removes empty nodes" | With `WhenEmptyOrUnderutilized`, it evicts running pods to repack them | `consolidation-thrash` |
| "Preemption happens only when the cluster cannot grow" | The scheduler preempts at once and does not know Karpenter exists | `preempt-not-wait` |
| "A PDB only costs something during an outage" | A PDB that allows zero disruptions blocks every voluntary node change | `pdb-blocks-rotation` |
| "Lower requests are cheaper" | They are cheaper on paper and more expensive in latency; real usage still has to fit | `requests-lie` |
| "Spreading across zones is free" | A required spread makes the scarcest zone the bottleneck | `zone-skew-ice` |
| "Headroom is waste" | It is a paid-for way to remove the slowest rungs of the latency ladder | `cold-start-relay` |

### 5.7 Language guide for content authors

- Say **requests** or **usage**, never bare "CPU".
- Say **desired**, **scheduled**, **started**, and **ready** replicas; never "running" for all four.
- Say **Pending (unschedulable)** for pods the scheduler could not place and **ContainerCreating** for pods bound to a node that cannot start them yet.
- Say **provision** for Karpenter launching a node, **disrupt** for Karpenter removing or replacing one, **evict** for the eviction API, and **preempt** for the scheduler.
- Say **NodePool limit**, not "maximum nodes"; limits are resources, not node counts.
- Say **pod scaling** and **node scaling**; avoid "the autoscaler".
- Keep captions to what happened and why, in the present tense, and let the inspector carry the detail.

## 6. What Carries Over from Forkscape, and What Changes

### 6.1 Patterns reused as they are

- **Scenario library first.** The library opens on arrival; free exploration is secondary. Forkscape moved from modules and quizzes to scenarios and dropped graded assessment; Scalescape starts there.
- **Scenario shape.** `symptom`, `steps` with captions, `watch`, `mechanism`, `production`, `evidence`, `reference`, `related`, and a `fix` with `change` and `cost`. Every fix states its cost.
- **Fix replay at the same step.** A switch at the top of the stepper changes between the problem and the fix, keeps the current step, and frames the stepper in the run's colour. Both runs have the same number of steps, and the chart and comparison keep one scale across both runs, so stepping and switching never move the layout.
- **Pure deterministic model with invariants.** Both renderers consume the same `WorldState`; tests replay every scenario and assert its claims.
- **Motion.** One 600 ms curve, `cubic-bezier(.4, 0, .2, 1)`, shared by the 3D scene, the flat view, the chart cursor, and the comparison bars. Objects keep identity across a step: a new pod comes out of its workload deck, a bound pod travels from the harbor to its node, a NodeClaim's tray descends from the shipyard to its zone, and stepping back returns them. Reduced motion changes the scene at once.
- **Labels only what changed.** While stepping, the 3D scene labels what the step changed from the previous step plus the selection; the first step, free exploration, and the flat view label everything. Labels never overlap; the less important label gives way.
- **Inspector rungs.** Each selectable element has a metaphor, a mechanism, and a measurement entry. The measurement rung names the real command, event, or log line, and states that the value on screen is model data.
- **Layout and chrome.** The scene fills the window with a 3D/flat toggle; the scenario panel starts open; other panels (inspector, event timeline, ledger, camera and legend) live in a menu with one panel open at a time; under 900 px the panel becomes a sheet with Back and Next docked. Each scenario has its own URL hash.
- **Type, accessibility, and hosting.** The 12–30 px rem type scale, 16 px prose, keyboard focus order, `aria-current` on selections, flat view parity, a static Vite build on Netlify with the same strict CSP and immutable asset caching.
- **Reference probes as separate evidence.** Small scripts run in disposable environments, write JSON into `docs/`, and appear in the app as labeled reference observations, never mixed into model values.

### 6.2 Code to lift

Copy, then adapt, rather than share a package until a second app proves which parts are common:

- `src/motion.ts`: the easing, `captureMarks` for DOM marks, and the identity-link idea, with keys changed from `p/<pid>` and `f/<frame>` to `w/<workload>`, `pod/<name>`, `node/<name>`, and `claim/<name>`.
- The stage-framing approach in `src/scene.ts` (`stageView`): fit a fixed set of stage boxes so the camera never needs to move when objects appear.
- The label placement and collision logic, the panel/menu/sheet chrome in `src/main.ts`, the chart's fixed-scale rendering and tooltip, and `src/theme.css`'s type scale.
- `scripts/ui_parity.mjs` with `playwright-core` and system Chrome, and the `node --experimental-strip-types --test` setup.
- `netlify.toml` headers.

### 6.3 What changes

- **Time is first-class.** Forkscape's steps were instantaneous actions. Here each step applies inputs (a load change, a config edit, an injected event) and advances a simulated clock, during which several controllers act on their own periods. The scene shows a clock and the elapsed time of each step ("+45 s").
- **Many controllers, one tick loop.** Instead of a single `applyAction`, the model runs controllers in a fixed order each tick (Section 12.3).
- **Objects come and go at scale.** Nodes are created and destroyed; the stage reserves terrace room for the most nodes any state of either run reaches, so the camera never reframes while stepping.
- **The accounting distinction** is requests against usage, and desired against ready, instead of RSS against PSS.
- **Cost is a series.** Node-hours and an illustrative hourly price appear in the ledger and, where the fix's cost is money, on the chart.

### 6.4 Forkscape lessons applied from day one

- Start with a model-only MVP and a narrow library; defer recorded telemetry until a contract exists.
- Write captions from model values through small template functions, so a caption cannot disagree with the state it describes, and assert key numbers in scenario tests.
- Decide the mobile layout with the first vertical slice, not after.
- Keep the chart, comparison, and stepper sizes fixed per scenario from the first commit; retrofitting "stepping never moves the layout" was expensive.
- Keep probe findings in `docs/` with versions, and phrase in-app reference notes as observations from one environment, not general rules.

## 7. MVP Scope

### 7.1 Included

- Eight scenarios (Section 8.3), each with a fix replay, chart, lens comparison, notes, and evidence.
- A deterministic model of: HPA (resource and external metrics), KEDA `ScaledObject` (activation, polling, cooldown), ReplicaSet scale-up and scale-in ordering, kube-scheduler filtering (resources, node selectors and affinity, taints and tolerations, zone topology spread, maximum pods) and preemption, Karpenter provisioning (batching, NodePool selection by weight, requirements, limits, offerings, insufficient capacity), Karpenter disruption (consolidation by deletion and single-node replacement, drift, expiry, budgets, PDBs, `do-not-disrupt`, `consolidateAfter`), node lifecycle timings, kubelet start (IP, image pull, readiness, warm-up CPU), the VPC CNI's IP accounting per subnet, and a simple load and CPU contention model.
- 3D and flat views from one state; inspector, event timeline, ledger, camera and legend panels; phone layout.
- Free exploration from any scenario step: change load, HPA and NodePool settings, PDBs, spread constraints, and inject spot interruptions or capacity shortages. Nothing is graded or saved.
- Reference probes K and E (Section 14), with results in `docs/` and labeled notes in scenarios they cover.

### 7.2 Deferred extension tracks

- The extension scenarios (Section 8.4).
- Uploading recordings from a real cluster (Section 15).
- EKS Auto Mode as a variant (managed Karpenter, built-in NodePools) and Cluster Autoscaler as a comparison track.
- Vertical scaling: VPA and in-place pod resize.
- Multiple clusters, Fargate, and GPU or accelerator scheduling.

### 7.3 Non-goals

- A faithful re-implementation of kube-scheduler or Karpenter. The model implements the rules the scenarios need and states its simplifications (Section 12.6).
- A capacity planner. The model is at teaching scale (a handful of nodes, tens of pods) and must not be used to size a real cluster.
- Live connection to a cluster, or credentials of any kind in the browser.
- Graded assessment, accounts, or progress tracking.

## 8. Scenario Library

### 8.1 Scenario format

The format follows Forkscape's `Scenario`/`ScenarioFix` interfaces, with time and lenses added.

```ts
export type ScenarioGroup = 'ceilings' | 'timing' | 'disruption' | 'signals';
export type Lens = 'hpa' | 'scheduler' | 'provisioner' | 'disruption' | 'node';
export type Caption = string | ((before: WorldState, after: WorldState) => string);
export interface ScenarioStep {
  caption: Caption;
  inputs: Input[];                       // load changes, config edits, injected events
  advance: number | { until: (world: WorldState) => boolean; within: number };  // simulated seconds
}
export interface ScenarioRun {
  setup: Setup;                          // workloads, autoscalers, NodePools, nodes, subnets, offerings
  steps: ScenarioStep[];
  chart?: { series: ChartSeries[]; limit?: boolean; cost?: boolean };
  watch: string;
}
/** The fix changes one thing about the problem's setup and replays the same steps. */
export interface ScenarioFix extends ScenarioRun {
  change: string;
  diff: string;                          // the one change, as a YAML diff shown beside the switch
  cost: string;                          // every fix has a cost
}
export interface Scenario extends ScenarioRun {
  id: string; group: ScenarioGroup; title: string;
  symptom: string;                       // as an on-call engineer first meets it
  mechanism: string;
  lenses: Partial<Record<Lens, string>>; // what each controller sees, where the difference is the point
  tags: ('requests-vs-usage' | 'desired-vs-ready')[];
  production: string;
  evidence: string;                      // commands, events, and log lines
  reference?: string;                    // what Probe K or E observed, if it covers this scenario
  related: string[];
  fix: ScenarioFix;
}
```

A fix's `setup` differs from the problem's in exactly the place its `diff` shows; a test enforces this by diffing the two setups.

### 8.2 Teaching scale

Scenarios use small, legible numbers. Nodes come from a short catalog of real instance names with their real vCPU, memory, ENI, and maximum-pod figures (for example `c7i.xlarge`, `m7i.large`, `m7i.xlarge`, `c6i.xlarge`, `m6i.xlarge`), DaemonSet overhead of a few hundred millicores, and illustrative prices labeled as such. Most workloads request 1 vCPU, so a 4-vCPU node holds three of them after DaemonSets. Three zones (`a`, `b`, `c`). The numbers in the step outlines below are the intended shape; scenario tests fix the final values.

### 8.3 MVP scenarios

Each outline shows the problem and the fix at the same step, because both runs replay the same inputs.

#### 8.3.1 `limit-ceiling` — "The HPA says 12; nine are serving"

- **Group:** ceilings. **Tags:** desired-vs-ready.
- **Symptom:** Latency is high; the HPA shows desired 12 and current 12; the Deployment shows 9 available; three pods have been Pending for an hour; Karpenter reports no errors.
- **Setup:** `web` requests 1 vCPU and 2 GiB; HPA on CPU at 60%, `minReplicas` 4, `maxReplicas` 12. NodePool `general` allows `c7i.xlarge` only, `limits.cpu: 12` (three nodes). Two nodes run four pods.

| Step | Clock | Problem | Fix (`limits.cpu: 20`) |
| --- | --- | --- | --- |
| 1 | 09:00 | Steady: 4 pods at 60% of request; NodePool 8 of 12 vCPU | Same; 8 of 20 vCPU |
| 2 | +30 s | Load doubles; pods at 120% of request; HPA asks for 8; two pods fit, two are Pending | Same |
| 3 | +10 s | Karpenter batches the two pods and launches a node in zone c; NodePool reaches 12 of 12 | Same; 12 of 20 |
| 4 | +90 s | The node is ready and both pods start; 8 ready at 60%: on target | Same |
| 5 | +5 min | Load rises again; pods at 90% of request; HPA asks for 12; one fits, three are Pending | Same |
| 6 | +10 s | Karpenter cannot launch: every compatible offering would exceed the limit. `FailedScheduling`: Insufficient cpu | Karpenter launches a fourth node; 16 of 20 |
| 7 | +2 min | 9 ready at 80% of request, a third above target. The HPA counts the three Pending pods as idle replicas, so by its arithmetic 12 is exactly right: desired 12, current 12, no condition raised | All 12 ready at 60% |
| 8 | +15 min | Load rises once more; ready pods at 93% of request; the HPA wants 14 and is capped at 12 (`ScalingLimited`); still three Pending | All 12 ready at 70% of request; `ScalingLimited` is now the honest signal that `maxReplicas` binds |

- **Watch:** desired and current replicas stay equal while ready replicas stay three short; the NodePool usage line sits on its limit.
- **Mechanism:** When ready pods ask for a scale-up and some pods are Pending or unready, the HPA recomputes the average as if those pods used nothing. Pending pods therefore make the fleet look less loaded, and the HPA reports itself satisfied while pods wait. Karpenter's limit check is silent from the HPA's point of view.
- **Lenses:** HPA: 9 pods at 80% of request, 3 counted at 0%, average on target. Scheduler: three pods that fit nowhere. Provisioner: three pods it could place on one node, but the node would exceed the limit. Disruption: nothing to do.
- **Fix:** Size the NodePool limit from the HPA: `maxReplicas × request`, plus DaemonSet overhead per node, plus rollout surge. **Cost:** the limit no longer caps spend below what the HPA can ask for, so `maxReplicas` becomes the budget number and needs the same review a limit got; a bad metric can now consume up to the new limit.
- **Production:** Alert on time spent Pending and on ready vs desired, not on HPA conditions alone. Karpenter's NodePool usage and limit metrics show the ceiling.
- **Evidence:** `kubectl get hpa web` · `kubectl get deploy web` · `kubectl get events --field-selector reason=FailedScheduling` · `kubectl get nodepool general -o jsonpath='{.status.resources}'` · Karpenter controller logs.

#### 8.3.2 `zone-skew-ice` — "Pending in zone c; room in a and b"

- **Group:** ceilings.
- **Symptom:** Several pods are Pending although nodes in two zones have free room; Karpenter logs insufficient-capacity errors for one zone; the HPA keeps adding replicas that also wait.
- **Setup:** `api` requests 1 vCPU, spreads across zones with `maxSkew: 1` and `whenUnsatisfiable: DoNotSchedule`; HPA on CPU. NodePool `spot-c7i` allows `c7i.xlarge` spot only. Zone c has no spot capacity for that offering during the scenario.

| Step | Clock | Problem | Fix (requirements widened to `c7i`, `c6i`, `m7i`, `m6i` × `xlarge`, `2xlarge`, spot and on-demand) |
| --- | --- | --- | --- |
| 1 | 14:00 | 6 pods, 2 per zone, one node per zone, one free slot each | Same |
| 2 | +30 s | HPA asks for 12; one pod per zone fits (3/3/3); three Pending, one per zone | Same |
| 3 | +10 s | Karpenter launches one node per zone; zone c fails with insufficient capacity and the offering is marked unavailable | Zone c's cheapest remaining offering (`m7i.xlarge` spot) launches |
| 4 | +90 s | Nodes in a and b are ready: 4/4/3. The third pod must go to c; placing it in a or b would make the skew 2 | All three start: 4/4/4 |
| 5 | +5 min | HPA asks for 15; none of the new pods can go to a or b; four Pending | Spread continues across new nodes in all three zones |
| 6 | +3 min | Karpenter retries zone c after the unavailable-offering cache expires; still no capacity | Steady |
| 7 | +5 min | Four free slots in a and b, paid for, while four pods wait | No Pending pods |
| 8 | +10 min | Ready 11 of 15; latency high | Ready 15 of 15 |

- **Watch:** the zone c terrace stays empty while its harbor lane fills; free slots glow on a and b.
- **Mechanism:** A required spread makes the next pod's location depend on the scarcest zone. Karpenter honors the constraint by launching in that zone, so its options there are the NodePool's offerings in that zone, which a narrow NodePool reduces to one.
- **Lenses:** Scheduler: a and b fail the spread filter, c has no node. Provisioner: one feasible offering in c, unavailable. HPA: ready pods hot, Pending pods idle.
- **Fix:** Widen the NodePool's instance families, sizes, and capacity types so each zone has several offerings. **Cost:** replicas land on different instance generations with different per-core speed, which an average utilization hides; on-demand fallback costs more; a broad set is harder to cover with reservations or savings plans. The related alternative, `whenUnsatisfiable: ScheduleAnyway`, keeps pods running but gives up the zone-failure guarantee; the notes compare both.
- **Evidence:** `kubectl describe pod` events · `kubectl get nodeclaims -w` · Karpenter logs for insufficient capacity · `kubectl get pods -o wide` grouped by zone label.

#### 8.3.3 `ip-exhaustion` — "Scheduled, never started, and Karpenter does nothing"

- **Group:** ceilings. **Tags:** desired-vs-ready.
- **Symptom:** New pods sit in `ContainerCreating` with `FailedCreatePodSandBox` events; Karpenter launches nothing; the HPA keeps asking for replicas.
- **Setup:** One small subnet per zone; the VPC CNI in secondary-IP mode with the default `WARM_ENI_TARGET=1`, so each node holds a spare ENI's worth of addresses. `web` on HPA. The scenario stays in zone a for legibility.

| Step | Clock | Problem | Fix (`WARM_IP_TARGET=2`, `MINIMUM_IP_TARGET` set to a node's usual pod count) |
| --- | --- | --- | --- |
| 1 | 11:00 | Two nodes; most of the subnet's free addresses are held in the nodes' warm pools | Same pods; the warm pools are small and the subnet keeps most addresses free |
| 2 | +30 s | HPA scales out; pods Pending; Karpenter launches a third node, which takes the subnet's last addresses for its own ENIs | The third node takes only the addresses it needs |
| 3 | +90 s | Pods bind to the new node (the scheduler checks maximum pods, not addresses); the CNI cannot assign addresses; pods stay in `ContainerCreating` | Pods get addresses and start |
| 4 | +1 min | Nothing is unschedulable, so Karpenter has nothing to act on. Idle warm addresses sit on nodes 1 and 2, where node 3's pods cannot use them | Steady |
| 5 | +3 min | HPA scales again; the new pods also bind to node 3 and also wait | New pods start |
| 6 | +5 min | Ready count flat; the HPA counts the waiting pods as idle replicas | All ready |

- **Watch:** the zone's address reservoir drains to zero while pods sit on node 3 without the address glyph.
- **Mechanism:** IP addresses are a node-local resource the scheduler does not model, and the CNI hoards them per node. A pod that is bound but cannot start is invisible to both the scheduler and Karpenter.
- **Lenses:** Scheduler: done. Provisioner: nothing Pending. Node: no address. HPA: replicas exist but are unready.
- **Fix:** Size the warm pool by addresses (`WARM_IP_TARGET`, `MINIMUM_IP_TARGET`) rather than whole ENIs. **Cost:** more EC2 address-assignment API calls, which can be throttled at scale, and slower pod starts when the warm pool is empty. The production notes cover the structural fixes: a secondary CIDR with custom networking, prefix delegation (which needs contiguous free blocks in the subnet), or IPv6.
- **Evidence:** `kubectl describe pod` (`FailedCreatePodSandBox`) · `aws ec2 describe-subnets --query 'Subnets[].[SubnetId,AvailableIpAddressCount]'` · `kubectl -n kube-system logs ds/aws-node` · `kubectl -n kube-system get ds aws-node -o yaml` for the warm settings.

#### 8.3.4 `cold-start-relay` — "KEDA scaled at once; the first message waited four minutes"

- **Group:** timing.
- **Symptom:** Each morning the first burst of queue messages waits minutes; KEDA's logs show it activated within one polling interval.
- **Setup:** `worker` scaled by a KEDA `ScaledObject` on queue depth (50 messages per replica, activation at 5, `minReplicaCount: 0`, `maxReplicaCount: 10`); requests 1 vCPU; a 1.5 GB image. NodePool `batch` with `consolidationPolicy: WhenEmpty`; its last node was removed overnight.

| Step | Clock | Problem | Fix (a placeholder Deployment of two pods at priority −10, running the worker image with a sleep command) |
| --- | --- | --- | --- |
| 1 | 06:59 | No workers, no batch nodes | No workers; one batch node holds two placeholder pods and the cached image |
| 2 | 07:00 | 300 messages arrive | Same |
| 3 | +20 s | KEDA's next poll activates 0 → 1; the worker pod is Pending | KEDA activates; the worker preempts a placeholder and starts from the cached image in seconds |
| 4 | +15 s | KEDA's HPA asks for 6; six Pending; Karpenter launches two nodes | Second worker preempts the other placeholder; four workers and two placeholders Pending; Karpenter launches two nodes |
| 5 | +60 s | Nodes registered and initialized | Same |
| 6 | +90 s | Image pulled; the first worker is ready at about 07:03 | Workers on the new nodes pull and start; placeholders, lowest priority, wait for whatever room is left |
| 7 | +30 s | Backlog has grown; HPA asks for 10; another node | Backlog smaller; HPA asks for 8 |
| 8 | +3 min | Backlog drains; oldest message waited about four minutes | Backlog drains; oldest message waited under a minute |

- **Watch:** the latency ladder lights rung by rung; in the fix, the slowest rungs are skipped for the first pods.
- **Mechanism:** Scaling from zero stacks every rung of the ladder in series. Placeholder pods hold a warm node with the image cached; because they have the lowest priority, real pods preempt them, and the displaced placeholders become the Pending pods that make Karpenter replace the headroom.
- **Fix cost:** one node is always running, and a quiet cluster never fully scales to zero; the headroom only covers the first two pods, so a larger burst still waits for Karpenter. Consolidation keeps the headroom node because the placeholders make it non-empty, which is the point. Alternatives in the notes: a KEDA cron trigger to pre-warm before a known ramp, smaller images, or lazy image loading.
- **Evidence:** pod condition timestamps (`PodScheduled`, `Initialized`, `ContainersReady`, `Ready`) · NodeClaim conditions (`Launched`, `Registered`, `Initialized`) · image pull events · KEDA operator logs.

#### 8.3.5 `consolidation-thrash` — "Nodes churn every few minutes and p99 follows"

- **Group:** timing. **Tags:** requests-vs-usage.
- **Symptom:** Karpenter disrupts nodes for `Underutilized` many times an hour; each disruption is followed by a latency spike and an HPA scale-out.
- **Setup:** `web` is a JVM service that uses about 2 vCPU for 90 s after start, then 0.4 vCPU per unit of load, and passes its readiness probe at 30 s, before warm-up ends. HPA on CPU at 60% with default behavior. NodePool `general` with `consolidationPolicy: WhenEmptyOrUnderutilized`, `consolidateAfter: 0s`. Load dips for a few minutes each quarter hour.

| Step | Clock | Problem | Fix (`consolidateAfter: 15m`) |
| --- | --- | --- | --- |
| 1 | 10:00 | 8 pods on 3 nodes, on target | Same |
| 2 | +5 min | Load dips; the HPA recommends 4 but stabilization holds 8 | Same |
| 3 | +5 min | Stabilization ends; the HPA scales to 4; the ReplicaSet deletes four pods; the default scoring had spread pods, so every node is lightly used | Same |
| 4 | +10 s | Consolidation finds node C's pod fits elsewhere: taints C, evicts the pod, terminates C. The replacement pod starts on A and burns 2 vCPU warming up | Node C's last pod change was seconds ago; it is not yet a candidate |
| 5 | +1 min | Load returns; the HPA reads the warm-up spike as load and asks for 10; six Pending; Karpenter launches two nodes | Load returns; the HPA asks for 8; the pods fit on A, B, and C |
| 6 | +90 s | New nodes ready; six pods warm up together; the HPA holds 10 | Steady at 8 |
| 7 | +5 min | Spike over; the HPA recommends 8 and stabilization starts again | Steady |
| 8 | +5 min | Scale-in; consolidation removes a node; the cycle repeats | Load dips again; node pod counts keep changing, so no node waits 15 minutes; no disruption |

- **Watch:** the node count saw-tooths in the problem run and stays flat in the fix; the HPA's desired line overshoots after each disruption.
- **Mechanism:** Consolidation and the HPA act on different clocks and read different quantities. Consolidation sees requests on a node and moves pods as soon as they fit elsewhere; the evicted pods restart, and warm-up CPU that readiness did not hide reaches the HPA as apparent load.
- **Lenses:** Disruption: node C fits elsewhere, budget allows it. HPA: usage spiked by warm-up. Provisioner: Pending pods after the overshoot.
- **Fix:** `consolidateAfter` longer than the HPA's scale-in rhythm, so nodes are only repacked after the fleet has been stable. **Cost:** underutilized nodes stay for at least 15 minutes after each change, about one extra node-hour per dip here; a fleet with constant small churn may rarely consolidate. The notes cover alternatives: disruption budgets that block `Underutilized` during business hours, `WhenEmpty` for latency-sensitive pools, and readiness that waits for warm-up.
- **Evidence:** Karpenter disruption logs and events with reasons · `kubectl get events --field-selector involvedObject.kind=Node` · HPA events (`SuccessfulRescale`) · pod restart times.

#### 8.3.6 `preempt-not-wait` — "Jobs restart whenever web scales, though a node was a minute away"

- **Group:** disruption.
- **Symptom:** Batch jobs are killed and restarted each time the web tier scales out; the cluster ends up with the extra node anyway.
- **Setup:** `web` at priority 1000 on an HPA; `worker` at priority 0 on KEDA, each message taking four minutes without checkpoints. NodePool `general` with room under its limit. Four nodes, full.

| Step | Clock | Problem | Fix (web's PriorityClass gets `preemptionPolicy: Never`) |
| --- | --- | --- | --- |
| 1 | 10:00 | 6 web and 6 worker pods fill four nodes | Same |
| 2 | +30 s | The web HPA asks for 9; three web pods cannot be placed | Same |
| 3 | +1 s | The scheduler preempts three worker pods on one node and nominates the web pods to it; Karpenter starts a NodeClaim | No preemption; Karpenter starts a NodeClaim; web pods wait |
| 4 | +30 s | The victims terminate after their grace period; their in-flight messages return to the queue; web pods start; the ReplicaSet recreates three workers, Pending | Web pods still Pending; workers keep working |
| 5 | +60 s | The new node is ready; the three workers start their messages from the beginning | The new node is ready; the web pods start |
| 6 | +2 min | Lost work grows the queue; KEDA asks for two more workers; another node | Queue normal |
| 7 | +5 min | 6 nodes; three messages processed twice | 5 nodes; no message lost |

- **Watch:** preemption strikes in the problem run, followed by the same NodeClaim the fix run waits for.
- **Mechanism:** Preemption runs inside the scheduler the moment a higher-priority pod cannot be placed; it does not know Karpenter is about to add capacity. Karpenter, in turn, provisions for whatever is unschedulable, including the pods preemption displaced. Whether Karpenter also provisions for a nominated preemptor is set from Probe K (question K5) before this scenario's node counts are final.
- **Fix:** `preemptionPolicy: Never` for a tier that can wait about a minute. **Cost:** web's three new replicas arrive about 90 s later, so web runs hotter during the ramp. If that is unacceptable, combine with headroom from `cold-start-relay`, so web preempts placeholders instead of batch.
- **Evidence:** `kubectl get events --field-selector reason=Preempted` · pod `nominatedNodeName` · `kubectl get priorityclass` · NodeClaim creation times · job restart counts.

#### 8.3.7 `pdb-blocks-rotation` — "The AMI rollout stalls all night, then runs at the morning peak"

- **Group:** disruption.
- **Symptom:** Nodes stay `Drifted` overnight with Karpenter events saying a PDB prevents eviction; at the morning ramp, node replacements begin and evict pods while traffic climbs.
- **Setup:** `web` HPA `minReplicas: 2`; PDB `minAvailable: 2`. NodePool on the latest AMI alias, drift enabled, default budget. A new AMI is published at night.

| Step | Clock | Problem | Fix (PDB `maxUnavailable: 1`) |
| --- | --- | --- | --- |
| 1 | 18:00 | 8 replicas on 3 nodes; the PDB allows 6 disruptions | The PDB allows 1 |
| 2 | 23:00 | The HPA scales to 2, on nodes A and B; node C is empty and is consolidated | Same |
| 3 | 23:30 | A new AMI; nodes A and B are `Drifted` | Same |
| 4 | +1 min | Each drifted node hosts a pod whose PDB allows 0 disruptions; Karpenter skips both | Karpenter launches A′, taints A, evicts one pod, which starts on A′; A terminates |
| 5 | 03:00 | Still drifted; the overnight window passes | B is replaced the same way; the fleet is on the new AMI |
| 6 | 08:30 | Traffic ramps; the HPA scales to 8; the PDB now allows 6; Karpenter begins replacing A and B during the ramp | Ramp proceeds with no disruption |
| 7 | +5 min | Evicted pods wait for replacement nodes; ready count dips as load climbs | Steady |

- **Watch:** shield glyphs on the two night pods read "0 allowed"; the drift markers stay lit until the ramp.
- **Mechanism:** Karpenter's graceful disruption skips nodes whose pods cannot be evicted under their PDBs. A PDB that matches the HPA's minimum allows nothing when the HPA is at its minimum, so voluntary node changes wait until the HPA scales up, which is usually the busiest time. Forceful paths (expiry with `terminationGracePeriod`, spot interruption) behave differently; the notes describe them once Probe K confirms the current semantics.
- **Fix:** `maxUnavailable: 1`. **Cost:** during each overnight rotation, one replica serves alone; if one replica cannot hold night traffic, raise `minReplicas` to 3, which costs a pod all night. Disruption budgets with a `schedule` can choose when drift happens, but cannot override a PDB that allows zero.
- **Evidence:** `kubectl get pdb` · `kubectl get nodeclaims -o jsonpath` for the `Drifted` condition · Karpenter events on NodeClaims and nodes · `kubectl get nodes -L karpenter.sh/nodepool` with AMI IDs.

#### 8.3.8 `requests-lie` — "HPA at max and nodes saturated, after Karpenter shrank the cluster"

- **Group:** signals. **Tags:** requests-vs-usage.
- **Symptom:** At peak, the HPA sits at `maxReplicas`, node CPU is saturated, and latency is high; nothing is Pending, so Karpenter adds nothing. Overnight, Karpenter had consolidated two nodes into one because they looked underutilized.
- **Setup:** `web` requests 200m, set when the service was lighter; each pod uses 50m of overhead plus its share of the work, with no CPU limit. HPA on CPU at 70%, `maxReplicas` 20. NodePool `general` (`c7i.xlarge` and smaller sizes) with consolidation enabled.

| Step | Clock | Problem | Fix (requests set to 500m, from observed p95 usage) |
| --- | --- | --- | --- |
| 1 | 22:00 | 6 pods on 2 nodes at the HPA's target, about 140m of a 200m request each; requests fill 16% of each node | The HPA holds fewer pods, each at the same 70% target of a larger request |
| 2 | +2 h | Consolidation fits both nodes' requests on one node and removes the other; 6 pods on 1 node | Consolidation also repacks onto one node, which is safe while usage stays below requests |
| 3 | 09:00 | Peak load; the HPA scales to its maximum of 20 within a minute; 18 pods fit on node 1 by requests, two are Pending; Karpenter adds a small node | Peak load; the HPA scales out well inside its maximum, which now covers 20 × 350m of real use; Karpenter adds nodes for the pods that do not fit |
| 4 | +2 min | Real use exceeds requests: node 1's pods need about 4.5 vCPU on a node with 3.7 allocatable; CPU is shared by request, so each pod is held near 205m; latency climbs | Each node's real use stays below its capacity; latency holds |
| 5 | +5 min | The HPA reads about 105% of request against a 70% target and reports `ScalingLimited`; the scheduler and Karpenter see a full node by requests and nothing Pending | On target, below the maximum |
| 6 | +10 min | Nothing reacts: no controller reads node saturation | Steady |

- **Watch:** in the lens comparison, node 1's usage bar overruns its requests bar and the node's ceiling while the HPA's desired line sits flat at its maximum.
- **Mechanism:** The HPA keeps each pod near its target fraction of the request, so a low request means many small replicas, and `maxReplicas × request × target` is the most real work the fleet can absorb before the HPA runs out of pods. Beyond that, usage exceeds requests. The scheduler and Karpenter keep packing by requests, so nodes saturate, and because every pod is placed, Karpenter has nothing to act on. Consolidating on low requests overnight made the morning worse.
- **Lenses:** HPA: about 105% of request, at its maximum. Scheduler: node 1 full by requests, room on node 2. Provisioner: nothing Pending. Node: CPU saturated.
- **Fix:** Set requests to observed usage, for example from the VPA recommender in `Off` mode. **Cost:** requests now reserve what the service uses, so at peak the cluster runs more nodes and the bill shows it; the HPA target needs re-tuning because utilization is relative to the larger request; fewer, larger replicas mean each pod failure removes more capacity.
- **Evidence:** `kubectl top pods` vs `kubectl get pods -o jsonpath='{..resources.requests}'` · node CPU and CFS throttling metrics · HPA `ScalingLimited` · Karpenter consolidation events from the night before.

### 8.4 Extension scenarios

| Id | Symptom | Mechanism | Fix and its cost |
| --- | --- | --- | --- |
| `memory-ratchet` | HPA on memory scales out and never back; nodes never consolidate | Scale-out does not reduce per-pod memory (caches and heaps stay), so utilization never falls | Scale on load (requests per second, concurrency) through KEDA's Prometheus scaler; cost: a metrics dependency and a fallback to configure |
| `spot-wave` | Several nodes reclaimed together; replacements fail; the HPA piles on | A narrow spot instance set shares one capacity pool; interruptions and insufficient capacity arrive together | Diversify instance types and allow on-demand; cost: price and heterogeneity |
| `anti-affinity-fanout` | Every replica gets its own node; the limit is reached at a fraction of `maxReplicas` | Required pod anti-affinity on hostname forbids packing | Topology spread on hostname with `ScheduleAnyway`; cost: two replicas can share a node |
| `job-eviction` | KEDA `ScaledJob` pods restart halfway; backlog grows; more jobs spawn | Consolidation evicts running job pods | `karpenter.sh/do-not-disrupt: "true"` on job pods; cost: nodes with long jobs cannot be consolidated or rotated until they finish, so set `terminationGracePeriod` |
| `ghost-node` | Nodes launch and are removed empty minutes later | The HPA scales in while a NodeClaim is in flight; the ReplicaSet deletes Pending pods first, so the new node arrives with nothing to run | A longer scale-in stabilization or a scale-up stabilization window; cost: slower reaction |
| `priority-starvation` | At the NodePool limit, a web surge evicts workers; KEDA asks for more workers that cannot run | Preemption allocates fixed capacity by priority; KEDA reacts to the starved queue with more Pending pods | Separate NodePools and limits per tier; cost: less pooling, a higher total ceiling |
| `vpa-hpa-tug` | Replica count and requests oscillate | VPA raises requests, utilization falls, the HPA scales in, per-pod use rises, VPA raises again; Karpenter repacks each time | HPA on a non-resource metric, or VPA on memory only; cost: a new metric to own |
| `daemonset-squeeze` | A new DaemonSet's pods stay Pending on full nodes, or a node launched for a pod cannot fit it | DaemonSet overhead and reserved resources change allocatable after sizing | Priority for node agents; accurate overhead in the NodeClass; cost: slightly larger nodes |
| `rollout-surge` | A rollout stalls with surge pods Pending at high replica counts | `maxSurge` needs capacity above the HPA's count, which the limit did not include | Include surge in the limit, or lower `maxSurge`; cost: spend or slower rollouts |
| `metric-outage` | Scale-in stops, or KEDA drops to its fallback count | A missing metric blocks HPA scale-in; KEDA's fallback replaces a failing scaler's value | Fallback replicas sized deliberately; cost: fixed capacity during outages |

Each extension follows the same format and the same fix-replay rules. `ghost-node` and `priority-starvation` are the most likely first additions, because they reuse MVP model features.

### 8.5 Coverage matrix

Before Phase 3, a table maps each threshold concept (Section 5.5) and misconception (Section 5.6) to the scenario steps that address it, so gaps in the library are visible. A concept with no step, or a misconception confronted only in an extension, is a library gap.

## 9. Learner Experience

### 9.1 Layout

The scene fills the window. The scenario panel starts open beside it on wide screens; on screens under 900 px the scene fills the screen and the panel becomes a sheet beneath it, collapsed to the title, the step's caption, and Back and Next, and expanded to the notes, chart, and lens comparison. The menu holds the inspector, event timeline, ledger, and camera and legend panels; one panel is open at a time, and Escape closes it.

### 9.2 Stepping and the clock

- A step applies its inputs and advances the clock to its end condition; the stepper shows the clock and the step's elapsed time.
- The transition animates from the previous step's end state to this step's end state. Intermediate events inside a step appear in the event timeline, not in the scene. Authoring rule: if a step hides a state the learner needs to see, split it.
- The stepper keeps the height of its longest caption, and the chart and comparison keep one size and scale per scenario across both runs, so stepping never moves the layout.

### 9.3 Problem and fix

A two-state switch at the top of the stepper chooses the run, keeps the step, and frames the stepper in the run's colour. Beside it, the fix's YAML diff shows the one change. On narrow screens the switch is in the expanded sheet, and the collapsed dock's top edge carries the run's colour.

### 9.4 Free exploration

Free exploration starts from the empty example cluster or from any scenario step and keeps its own timeline. Controls: set load per workload, edit HPA `min`/`max`/target and behavior, edit KEDA thresholds, NodePool limits, requirements, `consolidateAfter`, and budgets, edit a PDB or spread constraint, publish a new AMI, inject a spot interruption, mark an offering as unavailable in a zone, and advance the clock by 15 s, 1 min, or 5 min. Nothing is graded or saved; only the 3D/flat choice persists.

### 9.5 Linked views

- **Event timeline:** every model event with its actor (HPA, KEDA, ReplicaSet, scheduler, Karpenter, kubelet, CNI, AWS, input) and the shape of the real message (`FailedScheduling`, `SuccessfulRescale`, `Preempted`, `Nominated`, NodeClaim conditions, disruption decisions). Scrubbing selects one world-state snapshot for every view.
- **Ledger:** per workload, desired, scheduled, started, ready, and Pending counts; per node, requests and usage against allocatable, pods against maximum pods, and addresses; per NodePool, usage against limits; per zone, free subnet addresses; and illustrative cost per hour.
- **Inspector:** metaphor, mechanism, and measurement rungs for the selection, organized around the five placement questions for pods.

## 10. Visual Design North Star: Decks, Harbor, and Zone Terraces

### 10.1 Scene composition

```text
       ┌──── workload deck: web (HPA) ─────────┐    ┌── workload deck: worker (KEDA) ──┐
       │ ■■■■■■■■■ □□□   desired 12 · ready 9  │    │ ■■ □□□□   desired 6 · ready 2    │
       │ gauge ◔ 133% of target 60%            │    │ hopper ▤▤▤▤ 300 msgs · act. 5    │
       └───────────────┬───────────────────────┘    └───────────────┬──────────────────┘
 ~~~~~~~~~~~ control-plane membrane: Pending harbor □□□ (reason glyphs) · scheduler gate ~~~~~~~~~~
                       │        Karpenter shipyard: NodeClaim ⌛ launching c7i.xlarge → zone c
 ════ zone a ══════════╪════════ zone b ═══════════════╪════ zone c ═══════  NodePool fence 12/12 vCPU
  [tray c7i.xlarge ███▒]  [tray c7i.xlarge ██▒░]   [ ✕ no offering ]   ⛁ subnet addresses 14 free
```

- **Workload decks** float at the top, one per Deployment, in Forkscape's glass-deck style. Each has replica slots: solid for ready, half-filled for starting, hollow for Pending. The HPA is a gauge on the deck's rim showing the metric against its target and the desired count; KEDA adds a hopper showing the queue depth and its activation threshold. A ring around the gauge fills during scale-in stabilization.
- **The control-plane membrane** sits between the decks and the floor, as Forkscape's kernel membrane did. Pending pods float in its **harbor** as hollow blocks, each tagged with glyphs for why it is unschedulable (resources, spread, taint, zone, limit). The **scheduler gate** is where pods drop through to nodes. Karpenter's **shipyard** builds NodeClaims as wireframe trays that solidify through Launched, Registered, and Initialized, with a progress ring and timer, and dashed lines from the Pending pods they were sized for.
- **The floor** is three zone terraces. Each **node** is a tray whose footprint is its allocatable CPU (width) and memory (depth), with a rim that encodes capacity type (spot trays have notched corners). **Pods** are blocks whose footprint is their request: what the scheduler packs. Each block carries a translucent column whose height is its usage relative to its request: what the HPA reads. A ceiling plane above each tray marks the node's real CPU; columns that rise through it show saturation the scheduler cannot see.
- **The NodePool fence** rings the floor with a fill gauge of usage against limit, like Forkscape's memory-limit fence. Each zone has an **address reservoir** for its subnet.
- **Filaments** run from each deck's replica slot to its pod block, in the workload's colour, so ownership is visible across zones.
- **Disruption marks:** a tray being disrupted gets a hatched taint stripe and a countdown; PDBs are shields over a workload's blocks with the allowed-disruption count; `do-not-disrupt` is a pin; `Drifted` is a small marker on the tray.

### 10.2 Encoding rules

- **Colour:** each workload has a hue from a colour-vision-safe categorical palette; Pending uses an outline style, not a workload hue; danger states (over limit, saturation, no address) use shape and motion as well as colour.
- **Redundant encoding:** capacity type by tray corner shape; readiness by fill; unschedulable reason by glyph; spot vs on-demand in labels.
- **Restraint:** at most three new elements per step; bloom only for preemption strikes and launch failures; the scene stays readable as a still frame.
- **Scale:** node trays and pod blocks are drawn to scale with each other within a scenario; the legend states that terrace positions are schematic.

### 10.3 Key moments and choreography

| Moment | Default | Reduced motion |
| --- | --- | --- |
| HPA rescale | The gauge needle settles; new hollow slots appear on the deck and pods drop from them into the harbor | Slots and pods appear in place; the count change is highlighted |
| Scheduling | A pod block travels from the harbor through the gate to its tray | The pod appears on its tray |
| Karpenter launch | A wireframe tray forms in the shipyard, fills through its conditions, and descends to its terrace | The tray appears with its condition label |
| Insufficient capacity | The forming tray cracks and dissolves; the zone's offering marker greys out with a timer | The tray outline is crossed; the timer shows |
| Preemption | A strike from the preemptor to its victims; victims lift into the harbor | Victims outlined; a note "preempted" |
| Consolidation | The tray gets its taint stripe; pods lift off and travel to their new trays; the empty tray sinks | Tray crossed; pods appear at their new places |
| Address exhaustion | The reservoir drains; pods on the node lose their address glyph and pulse slowly | Reservoir shows 0; pods outlined |
| Limit reached | The fence fill reaches the top and its rim lights | The fence shows "limit" |

Transitions take 600 ms on one curve; a step taken mid-transition continues from what is drawn.

### 10.4 Labels

As in Forkscape: HTML overlay labels, never overlapping, with priorities when space is short (selected item, then Pending reasons, workloads, NodeClaims, nodes, zones). While stepping, only what changed is labeled; on phones, node labels drop the instance size.

### 10.5 Flat view

A keyboard-accessible DOM view of the same state: workload cards with desired, scheduled, started, and ready counts and the HPA reading; a Pending list grouped by reason; zone columns of node cards, each with requests and usage bars and pod chips; NodeClaim cards with conditions; the fence and reservoirs as meters. Every element in the 3D scene is selectable in the flat view, and selection is shared. The app falls back to the flat view if WebGL fails.

### 10.6 Chart and lens comparison

- **Chart:** simulated time on the x-axis with a tick per step; series per scenario from desired replicas, ready replicas, Pending pods, nodes, NodePool usage (with the limit as a line), metric against target, and cost per hour. Colour follows the metric, as Forkscape's chart did, and the palette is validated for colour-vision deficiency against the panel surface. The cursor moves with the step; the scale comes from both runs.
- **Lens comparison:** the "what each controller sees" callout, open by default in scenarios tagged requests-vs-usage or desired-vs-ready and on request elsewhere. It draws, per node, the requests bar beside the usage bar against allocatable, and, per workload, the desired, scheduled, started, and ready counts, with one sentence per lens explaining what that controller concludes this step.

### 10.7 Theme

A sibling to Forkscape's dusk palette with chunky, garden-game shapes, shifted toward a harbor at dusk: deep blue water under the membrane, warm terraces, and bright workload hues. The same type scale and pixel-font rules apply.

### 10.8 Honesty markers

- Every view carries a **MODEL** badge in the MVP. Reference notes carry **REFERENCE · KIND+KWOK** or **REFERENCE · EKS** with the probe's date and versions.
- Prices are labeled **illustrative** with the date they were taken from.
- The legend states that zone terraces and tray positions are schematic and that timings are model parameters.

## 11. Fidelity Contract

| Source | What it can show | What it must not claim | Used for |
| --- | --- | --- | --- |
| **Model** | The decision rules the scenarios need, deterministically, at teaching scale | Real timings, real prices, scheduler scoring detail, or behavior outside the modeled rules | Every scenario and free exploration |
| **Reference · kind+KWOK** (Probe K) | The real kube-scheduler, HPA, KEDA, and Karpenter decision logic on simulated nodes | AWS behavior: launch times, insufficient capacity, ENIs and addresses, spot | Scenario `reference` notes on decision logic; model rule checks |
| **Reference · EKS** (Probe E) | Real launch, registration, image pull, and pod start timings; VPC CNI address behavior; spot interruption handling on one cluster | Other regions, instance families, AMIs, or load; anything beyond the probe's runs | Timing parameters; `reference` notes for timing and address scenarios |
| **Recorded** (post-MVP) | A real cluster's events and object snapshots, replayed | Causality the records do not support; state between snapshots | Upload and replay (Section 15) |

## 12. Model Engine

### 12.1 World state

```ts
export interface WorldState {
  t: number;                                   // simulated seconds
  workloads: Workload[];                       // spec.replicas, pod template, priority, constraints, image, startup profile, PDB
  pods: Pod[];                                 // phase, ready, node, nominated node or claim, timestamps, usage, unschedulable reasons
  hpas: Hpa[];                                 // spec, behavior, recommendation history, status and conditions
  scaledObjects: ScaledObject[];               // trigger, activation, polling, cooldown, last active, owned HPA
  nodePools: NodePool[];                       // weight, requirements, taints, limits, disruption policy, budgets, expireAfter
  nodeClaims: NodeClaim[];                     // pool, offering, conditions with times, pods it was sized for
  nodes: Node[];                               // claim, zone, allocatable, taints, labels, images cached, addresses held, last pod change
  offerings: Offering[];                       // instance type × zone × capacity type, price, available-until
  subnets: Subnet[];                           // zone, free addresses
  loads: Load[];                               // demand per workload; queue depth and arrival rate for KEDA
  events: WorldEvent[];                        // actor, kind, object, reason, message in the real message's shape
  params: Params;                              // the latency ladder and controller settings
}
```

All identifiers are deterministic (`web-7`, `general-3`), so motion keys and tests are stable.

### 12.2 Inputs and controller actions

Scenario steps and free exploration only apply **inputs**: `setDemand`, `setQueueArrivals`, `editHpa`, `editNodePool`, `editPdb`, `editWorkload`, `publishAmi`, `spotInterruption`, `markUnavailable(offering, until)`, and `advance(seconds)`. Everything else is a **controller action** produced by the model, and appears in the timeline with its actor.

### 12.3 Tick loop

A 1-second tick runs controllers in a fixed order; each controller acts only on its own period or trigger:

1. **Load and contention:** per-pod usage from demand ÷ ready pods plus per-pod overhead and warm-up CPU; where a node's usage exceeds allocatable, CPU is shared in proportion to requests and a latency index rises.
2. **Metrics:** samples per metric resolution, with the HPA's CPU-initialization and readiness rules applied when the HPA reads them.
3. **HPA and KEDA:** desired replicas on their periods, with tolerance, unready and missing-metric handling, behavior policies, and stabilization; KEDA activation and cooldown.
4. **ReplicaSet:** create or delete pods; on scale-in, delete Pending, then unready, then ready pods.
5. **Scheduler:** for each Pending pod by priority and creation time, filter nodes, score by least allocated (the default), bind or mark unschedulable with reasons; try preemption for pods that can preempt.
6. **Karpenter provisioner:** batch unschedulable pods (idle and maximum windows); place them on in-flight NodeClaims first, then on new NodeClaims by NodePool weight, requirements, limits, and the cheapest available offering; on insufficient capacity, mark the offering unavailable and retry.
7. **Node lifecycle:** advance NodeClaims through launch, registration, and initialization per AMI-family timings.
8. **Kubelet and CNI:** for bound pods, assign an address from the node's warm pool or the subnet, pull the image at the modeled rate unless cached, start containers, and mark ready after the readiness delay.
9. **Karpenter disruption:** on its period, find candidates (empty or underutilized after `consolidateAfter`, drifted, expired), filter by budgets, PDBs, and `do-not-disrupt`, simulate rescheduling, and execute: taint, launch replacement if needed, evict through PDB checks, terminate.
10. **Interruption:** handle injected spot interruptions as forceful disruption.

Ties break by name. There is no randomness; a seeded generator is available for free exploration's optional noise and is off in scenarios.

### 12.4 Rule sources

Each controller module cites the documentation or source it follows, in a comment and in `docs/model-rules.md`:

- **HPA:** the documented algorithm, desired = ⌈current × current metric ÷ target⌉ with tolerance, and the conservative recomputation for unready, Pending, and missing-metric pods; behavior defaults; upstream controller tests used as table-driven model tests where practical.
- **KEDA:** activation and scaling phases, `pollingInterval`, `cooldownPeriod`, and the HPA it creates.
- **Scheduler:** the filter plugins the scenarios use (resource fit, node affinity, taint toleration, pod topology spread, maximum pods) and default preemption.
- **Karpenter:** v1 NodePool and NodeClaim semantics, provisioning batching, limits, offerings and insufficient-capacity handling, consolidation, drift, expiry, budgets, PDB handling, and `do-not-disrupt`.
- **VPC CNI:** warm pool settings and maximum pods per instance type.

### 12.5 Model boundaries

The model implements a subset of each controller, at teaching scale. Scheduling scores only by least-allocated resources and breaks ties by name; there is no inter-pod affinity beyond required hostname anti-affinity (extension), no volume topology, and no scheduling gates. Karpenter's packing is first-fit decreasing over a short catalog, consolidation evaluates deleting one node or replacing one node with one cheaper node, and multi-node consolidation is omitted. Node timings and image pull rates are parameters, set from Probe E. CPU contention is proportional sharing by request with a simple latency index; memory pressure and eviction are omitted from the MVP. Prices are illustrative. The VPC CNI model counts addresses per node and subnet and omits ENI attachment limits beyond maximum pods. These are teaching simplifications, not Kubernetes, Karpenter, or AWS behavior, and the model must not be used to size a real cluster.

### 12.6 Invariants

Checked after every tick in tests:

- every bound pod's node exists and is not terminated; no pod is both bound and Pending;
- requests on a node never exceed allocatable, and pod count never exceeds maximum pods;
- each NodePool's capacity never exceeds its limits as a result of a launch;
- a voluntary eviction never violates a PDB;
- each NodeClaim belongs to an existing NodePool, and its offering satisfies the NodePool's requirements;
- addresses in use never exceed a subnet's size, and every running pod has one;
- the HPA's desired replicas stay within `min` and `max`;
- identifiers are unique, and event times never decrease.

## 13. Technical Architecture

| Part | Choice |
| --- | --- |
| Build | TypeScript and Vite, as in Forkscape; a static site |
| 3D | three.js, pinned, lazy-loaded; vanilla three.js as Forkscape uses, not React-Three-Fiber |
| Model | `src/model/` with one module per controller and `world.ts`, `inputs.ts`, `invariants.ts`; pure functions |
| Scenarios | `src/scenarios.ts` with setups, steps, fixes, and caption templates |
| UI | `src/main.ts` (chrome, panels, stepper, chart, comparison), `src/detail.ts` (inspector), `src/flat.ts`, `src/scene.ts`, `src/motion.ts` |
| Styling | `src/theme.css` with the Forkscape type scale |
| Tests | `node --experimental-strip-types --test src/**/*.test.ts`; `scripts/ui_parity.mjs` with `playwright-core` |
| Probes | `probes/kind-kwok/` and `probes/eks/` (shell and small scripts), JSON and summaries in `docs/` |
| Hosting | Netlify with Forkscape's headers and CSP |

Scenario histories are computed once per run and cached, as Forkscape's `scenarioStates` does; a 30-minute scenario at 1-second ticks is a few thousand cheap ticks.

## 14. Reference Probes

The probes check the model's claims; they are not replay bundles. Each writes JSON with the versions of every component and a short Markdown summary into `docs/`, in Forkscape's style.

### 14.1 Probe K: kind and KWOK in a disposable VM

Run in an ephemeral Ubuntu 24.04 VM from `labctl`, as Forkscape's probes do. Install kind, metrics-server, KEDA, and Karpenter with its KWOK provider (the `kwok/` provider in the upstream Karpenter repository, which simulates nodes and instance types). KWOK nodes run no containers, so metrics come from KWOK's resource-usage simulation for CPU-based HPAs and from KEDA's `metrics-api` scaler pointed at a tiny HTTP server whose value the probe sets.

Questions:

- **K1:** With Pending pods, does the HPA's recommendation match the model's damped recomputation in `limit-ceiling`, step by step?
- **K2:** What does Karpenter log and record when a NodePool limit blocks a launch, and what do the pods' events say?
- **K3:** How does `consolidateAfter` interact with HPA scale-in in `consolidation-thrash`: when does a node become a candidate, and what resets the wait?
- **K4:** With a PDB allowing zero disruptions, does Karpenter skip drifted nodes, and what event does it emit? How do expiry and `terminationGracePeriod` behave in the same setup?
- **K5:** When the scheduler preempts for a high-priority pod, does Karpenter also launch a node for that nominated pod, or only for the displaced pods? This sets `preempt-not-wait`'s node counts.
- **K6:** With a zone-spread constraint and a zone that has no allowed offering (modeled by restricting a NodePool's zones), what do the scheduler and Karpenter report?
- **K7:** On HPA scale-in, does the ReplicaSet delete Pending pods first, and what happens to an in-flight NodeClaim (`ghost-node`)?

### 14.2 Probe E: a disposable EKS cluster

A small EKS cluster created from a script (eksctl or OpenTofu), with self-managed Karpenter, KEDA, and metrics-server, run for a few hours and torn down by the same script, with a billing alarm set first. The summary records the cost of each run.

Questions:

- **E1:** NodeClaim `Launched` → `Registered` → `Initialized` → node `Ready` timings for AL2023 and Bottlerocket on two instance families, repeated enough to show spread.
- **E2:** Image pull times for a 1.5 GB image, cold and cached; pod `PodScheduled` → `Ready` with and without placeholder headroom.
- **E3:** VPC CNI behavior in a deliberately small subnet: when pods enter `ContainerCreating`, the exact events, and what Karpenter does; the effect of `WARM_IP_TARGET` and `MINIMUM_IP_TARGET`.
- **E4:** Spot interruption handling with Karpenter's interruption queue, triggered with AWS Fault Injection Service.
- **E5:** Metrics pipeline lag: from a load step to the HPA's first reaction.

Insufficient capacity cannot be produced on demand; `zone-skew-ice` relies on documented Karpenter behavior and upstream tests, and says so in its notes. Any insufficient-capacity events seen during probe runs are recorded.

### 14.3 How probe results appear in the app

A scenario's `reference` note quotes what the probe observed, with the probe's date and versions, as a separate labeled block, the way Forkscape shows its Linux observations. Model parameters may be set from probe medians, and `docs/` records which parameter came from which run.

## 15. Recording Contract (Post-MVP)

After the MVP, accept recordings made elsewhere rather than building a recorder. Prefer formats a cluster already produces:

- Kubernetes events (`events.k8s.io/v1`) as JSON lists;
- Kubernetes audit logs (`audit.k8s.io/v1`, JSON lines), which EKS control-plane logging writes to CloudWatch;
- periodic `kubectl get -o json` snapshots of pods, nodes, NodeClaims, NodePools, HPAs, and ScaledObjects; and
- Karpenter's structured controller logs.

The frontend validates a file against the contract, rejects anything over the size limit, strips or hashes names, labels, and annotations by default, and replays only what the records support, with gaps shown as gaps and a **RECORDED** badge. The schema, the mapping from records to a timeline, provenance, missing-data rules, and privacy handling remain decisions (Section 20).

## 16. Accessibility and Performance

- **Accessibility:** Forkscape's type scale and focus order; the flat view as the full keyboard equivalent of the scene; `aria-current` on selections; the 3D canvas names the flat view; every colour encoding has a shape or text equivalent; reduced motion removes transitions and hover movement; captions and notes are readable by screen readers in order. A manual screen-reader review is part of the definition of done.
- **Performance:** at most about 20 nodes and 100 pods per scene; instanced meshes for pod blocks; the scene renders on demand when idle; the 3D bundle is lazy-loaded; first scenario interactive within budget on a mid-range laptop; flat view usable on low-end phones.

## 17. Testing and Validation

- **Controller tests:** the HPA algorithm, table-driven, including tolerance, stabilization, behavior policies, and unready, Pending, and missing-metric cases; KEDA activation and cooldown; ReplicaSet scale-in order; scheduler filters and preemption; Karpenter batching, NodePool selection, limits, offering choice, and insufficient-capacity retry; consolidation, drift, budgets, PDBs, and `do-not-disrupt`; CNI address accounting.
- **Scenario tests:** every scenario is deterministic and passes invariants at every tick; problem and fix have the same number of steps; the fix setup differs only where its diff says; every chart series has data; every caption's numbers match the state; each scenario's claim holds (for example, `limit-ceiling`'s HPA reports desired = current while three pods are Pending, and its fix ends with none Pending).
- **Probe agreement:** for questions K1–K7, a test encodes the observed outcome, so a model change that contradicts a probe fails.
- **UI parity:** step through every scenario in flat and 3D modes; switch to the fix at each step without moving the layout or the chart's scale; check that a step labels only what changed and labels never overlap; check canvas picking, selection sharing between views, the inspector, timeline, ledger, back navigation, free exploration, and the phone layout.
- **Learning validation:** a think-aloud with five engineers on two scenarios before Phase 2: can they say, unprompted, why the pods are Pending and what the fix costs?

## 18. Delivery Phases and Exit Gates

Indicative durations assume one frontend engineer with three.js experience, part-time help from an engineer who runs EKS and Karpenter in production, and a reviewer with Kubernetes scheduling expertise.

### Phase 0 — Spikes (about 2 weeks)

- **S1:** Probe K environment and questions K1, K2, K5.
- **S2:** Probe E, one run, questions E1 and E3.
- **S3:** Model kernel (HPA, ReplicaSet, scheduler, provisioner) passing `limit-ceiling` in tests.
- **S4:** A flat-view prototype of `limit-ceiling` and `consolidation-thrash` shown to five engineers without narration.
- **Exit gate:** probe environments reproducible from scripts; K5 answered; four of five engineers explain the Pending pods correctly from the prototype.

### Phase 1 — Model and first vertical slice (about 3–4 weeks)

- All MVP controllers with invariants and tests.
- `limit-ceiling`, `cold-start-relay`, and `consolidation-thrash` with fixes, captions, chart, and lens comparison.
- Flat view, scenario panel, stepper, fix switch, timeline, ledger, inspector, and phone layout.
- **Exit gate:** the three scenarios run end to end in the flat view on desktop and phone; UI parity checks pass.

### Phase 2 — 3D scene and remaining scenarios (about 3–4 weeks)

- Decks, membrane, harbor, shipyard, terraces, trays, blocks, fence, reservoirs, filaments, and motion.
- `zone-skew-ice`, `ip-exhaustion`, `preempt-not-wait`, `pdb-blocks-rotation`, and `requests-lie`.
- Probe K questions K3, K4, K6, K7; Probe E questions E2, E4, E5.
- **Exit gate:** all eight scenarios in both views; probe agreement tests pass; reference notes written.

### Phase 3 — Free exploration, review, and polish (about 2–3 weeks)

- Free exploration controls; coverage matrix; accessibility pass and manual screen-reader review; performance checks; expert review of every scenario's claims and Section 21.
- **Exit gate:** definition of done (Section 22).

### Phase 4 — Extensions

`ghost-node` and `priority-starvation` first, then the other extension scenarios, the recording contract, EKS Auto Mode, and a Cluster Autoscaler comparison, one track at a time.

## 19. Risks and Mitigations

| Risk | Mitigation |
| --- | --- |
| The model teaches a cartoon of the scheduler or Karpenter | Rule sources cited per module; probe agreement tests; expert review; explicit model boundaries in the legend and notes |
| Karpenter, KEDA, or Kubernetes behavior changes between versions | Probes record versions; Section 21 checklist rerun on upgrade; scenario notes name the versions they were checked against |
| Time compression hides what happened inside a step | One controller decision per step; intermediate events in the timeline; split steps that hide states |
| Too many moving parts in one frame | At most three new elements per step; labels only for changes; still-frame readability check in UI tests |
| Probe E costs money or leaves resources behind | Billing alarm first; scripted teardown; a cost line in each probe summary; short runs |
| Insufficient capacity cannot be reproduced | `zone-skew-ice` is model-only for that rung and says so; any observed events are recorded |
| KWOK's simulation differs from real nodes in ways that matter | Use Probe K only for decision logic; take timings and address behavior from Probe E |
| Learners over-generalize a fix ("always raise limits", "never preempt") | Every fix states its cost; related scenarios show the opposite pressure (for example `limit-ceiling` against `priority-starvation`) |
| Scope grows into a general Kubernetes course | The five placement questions and the two loops bound the content; vertical scaling, storage, and networking beyond addresses are extensions |

## 20. Decisions to Close

1. Name: Scalescape is a working title.
2. Tick size: 1 s (recommended) or event-driven.
3. Whether the default scheduler scoring (least allocated) is shown explicitly as a cause in `consolidation-thrash`, or kept in the notes.
4. Instance catalog and teaching-scale numbers, including DaemonSet overhead and the allocatable formula.
5. Whether prices and a cost series appear in the MVP or only in the ledger.
6. Whether `ip-exhaustion` uses three zones or stays in one zone for legibility (recommended: one).
7. Whether to share code with Forkscape as a package after Phase 2, or keep copying.
8. Probe E tooling: eksctl or OpenTofu; AMI families to measure.
9. Whether EKS Auto Mode appears as a note per scenario or as its own track.
10. The recording contract's schema, size limit, privacy defaults, and which formats are accepted first.

## 21. Facts to Re-verify Against Current Versions

Check against the versions Probes K and E run, and record results in `docs/`:

- **HPA:** sync period; tolerance and whether configurable per-HPA tolerance is available; default behavior policies and stabilization; the treatment of unready, Pending, and missing-metric pods on scale-up and scale-down; CPU initialization period and initial readiness delay; status conditions (`AbleToScale`, `ScalingActive`, `ScalingLimited`) and their reasons.
- **metrics-server:** default metric resolution.
- **KEDA:** `pollingInterval` and `cooldownPeriod` defaults; activation semantics; the HPA it creates and its naming; fallback behavior; `ScaledJob` scheduling strategies.
- **ReplicaSet:** scale-in pod ranking.
- **kube-scheduler:** default scoring strategy; preemption victim selection and PDB handling; `nominatedNodeName` behavior; topology spread counting and `minDomains`; unschedulable backoff.
- **Karpenter (v1 APIs):** batch idle and maximum durations; NodePool selection by weight; how limits are computed (capacity vs allocatable, in-flight NodeClaims); unavailable-offering cache TTL; how many instance types a launch request includes; consolidation policies, `consolidateAfter` semantics and default; budget defaults, reasons, and schedules; drift triggers; `expireAfter` default and whether expiry respects budgets; `terminationGracePeriod`; PDB and `do-not-disrupt` handling for graceful and forceful disruption; the disruption taint's name; event and log messages for limits, insufficient capacity, and blocked disruption; behavior toward pods nominated by preemption; DaemonSet overhead and VM memory overhead in sizing; interruption queue handling; metric names.
- **VPC CNI:** default warm settings; maximum-pods formula per instance type; prefix delegation requirements; the exact sandbox error text.
- **EKS:** supported Kubernetes versions at probe time; AMI aliases; EKS Auto Mode differences from self-managed Karpenter.
- **AWS FIS:** the spot interruption action and its prerequisites.

## 22. Definition of Done for the MVP

- The static site loads in current desktop and mobile browsers and reaches the library within budget.
- All eight MVP scenarios run in 3D and flat views, each with its fix, chart, lens comparison, notes, evidence, and cost.
- Every scene element has metaphor, mechanism, and measurement entries in the inspector.
- Every view carries an accurate badge; reference notes carry their probe, date, and versions.
- The model passes its invariants and controller tests, and agrees with every answered probe question.
- UI parity checks pass, including no layout movement on step or run switch and no overlapping labels.
- Accessibility checks pass and a manual screen-reader review is complete.
- The facts in Section 21 are verified for the probed versions.
- An expert reviewer has signed off on every scenario's claims.

## 23. Immediate Next Actions

1. Script Probe K in a `labctl` VM and answer K1, K2, and K5, since K5 decides `preempt-not-wait`'s node counts.
2. Write the `WorldState`, input, and event types, and the HPA and scheduler modules with table-driven tests.
3. Build `limit-ceiling` end to end in the flat view, including the fix switch and fixed-scale chart, reusing Forkscape's chrome and motion.
4. Run one Probe E session for node launch timings and the small-subnet address test, with a billing alarm and scripted teardown.
5. Show the flat prototype of `limit-ceiling` and `consolidation-thrash` to five engineers.
6. Schedule an expert review of Sections 3, 5, 8, and 21 before Phase 1 content work.

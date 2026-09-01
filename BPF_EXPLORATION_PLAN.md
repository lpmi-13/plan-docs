# eBPF Exploration Plan for a Three-VM Firecracker Playground

## 1. Objective

Use a playground of no more than three Linux Firecracker microVMs to learn, test, and document how eBPF can diagnose live system problems without rebuilding or restarting the affected workload. All collectors, target workloads, control services, traffic generators, and storage used by the exploration must run **inside the microVMs**. Containers may be started inside the VMs and are the preferred way to create disposable targets and supporting services.

The exploration should demonstrate how to answer:

- Which short-lived process caused a CPU spike?
- Which file path failed to open, and with which errno?
- Where is the long tail of virtual block-I/O latency?
- Which TCP connections are churning or transferring very little data?
- Which kernel or application function is slow?
- Why was a Kubernetes/Cilium network flow dropped?
- Which code path consumed CPU before an alert fired?

The final output is not only a successful demo. It is a compatibility report, a set of reproducible experiments, measured overhead results, safe operating procedures, and a recommendation about which tools are useful in this constrained environment.

## 2. Constraints and Important Boundaries

### Hard Constraints

- Use at most three Firecracker microVMs at once.
- Run every exploration component inside one or more of those VMs.
- Do not depend on access to the physical host, its kernel, its KVM tracepoints, the Firecracker VMM process, host TAP devices, or host backing storage.
- Do not install diagnostic packages in the target application container.
- Do not restart or rebuild the target application to begin live tracing.
- It is acceptable to prepare the VM images with required kernel features and a trusted tracing toolbox before experiments begin.
- Prefer containers for repeatable workloads, fault injectors, collectors, and data stores when they do not need direct BPF access.

### What “Outside the Container” Means

An eBPF program runs in the kernel to which it is loaded. The VMs do not share a kernel, so an agent in VM 2 cannot attach to processes or kernel hooks in VM 1. To inspect a distroless container in VM 1, the loader must run in VM 1 with sufficient privileges and select the target by cgroup, namespace, or PID. Nothing needs to be installed inside the application container.

Likewise, guest eBPF can observe the guest block and network stacks, including virtio-facing behavior, but cannot show what happens later in the inaccessible Firecracker host. A result such as “latency is below the guest block layer” is the limit of the evidence, not proof of a specific host-side cause.

### Historical Profiling Boundary

Parca can show CPU activity from before an alert only if continuous sampling was already enabled and the profiles were retained. It cannot reconstruct time before the profiler started. The plan therefore enables continuous profiling before the historical-profile experiment.

## 3. Recommended Three-VM Topology

### VM 1: Target and Guest-Side Sensor

VM 1 is required. It runs:

- the trusted eBPF toolbox/agent;
- a container runtime such as containerd or Docker;
- one or more distroless target containers;
- BCC tools and bpftrace for interactive experiments;
- a Parca-compatible profiling agent;
- optional Cilium components only during the Hubble experiment;
- a small metadata resolver that maps container IDs to cgroups, namespaces, and guest PIDs.

The eBPF loader runs directly in the VM rather than in the target container. It may run in a dedicated privileged toolbox container if that approach proves reliable, but the container must use the VM's host PID/cgroup view and required kernel mounts. Running the loader as a VM system service is the simpler baseline.

### VM 2: Control, Storage, and Observer UI

VM 2 is recommended. It runs supporting services that do not need to attach to VM 1's kernel:

- experiment orchestration and run manifests;
- the Parca server/profile store and UI;
- event collection and normalized result storage;
- dashboards or notebooks;
- optional symbol storage keyed by application build ID;
- an HTTP/TCP server used by network scenarios.

VM 2 sends authenticated, time-bounded trace requests to the trusted agent in VM 1. The VM 1 agent loads probes locally and streams aggregated results back. VM 2 never claims to trace VM 1 directly.

### VM 3: Traffic, Storage, and Failure Peer

VM 3 is optional and should be started only for experiments that benefit from an independent peer. It can run:

- TCP clients and connection-churn generators;
- DNS and HTTP services with controllable failures;
- workload drivers that should not compete for VM 1 resources;
- a second monitored workload to test VM identity and isolation;
- optional shared-service or Kubernetes/Cilium test components.

If only one VM is available, run the targets, generators, and stores in separate containers in VM 1. With two VMs, combine VM 3's role into VM 2. Record the chosen topology in every experiment result because co-location changes performance measurements.

### Communication Flow

```text
VM 3 (optional load/peer)
          |
          | application traffic
          v
VM 1 (target VM)
  distroless target container
          |
          | observed by the same VM kernel
          v
  trusted eBPF agent/toolbox
          |
          | filtered events and profiles
          v
VM 2 (control, Parca server, results, symbols)
```

## 4. VM and Kernel Preparation

Build a reusable playground image or setup script rather than configuring an incident target manually each time.

### Required Kernel Capabilities

The VM 1 preflight must record:

- kernel release, architecture, boot ID, and kernel configuration hash;
- BPF syscall and BPF JIT availability;
- BPF filesystem and tracefs mounts;
- kprobe, uprobe, tracepoint, raw tracepoint, perf-event, and ring-buffer support;
- cgroup version and container-runtime cgroup layout;
- BTF availability at `/sys/kernel/btf/vmlinux`;
- kernel symbols and the location of optional debug symbols;
- lockdown mode, LSM policy, unprivileged-BPF policy, and applicable seccomp restrictions;
- availability of `CAP_BPF`, `CAP_PERFMON`, `CAP_SYS_RESOURCE`, and any kernel-version-specific need for `CAP_SYS_ADMIN`;
- clock synchronization and network reachability among the VMs.

Do not disable a security control globally merely to make a tool work. Document the minimum privileges, scope them to the trusted loader, and mark a tool unsupported if the playground kernel cannot safely provide them.

### Tooling Inside VM 1

Prepare and pin versions of:

- BCC and its `execsnoop`, `opensnoop`, `biolatency`, and `tcplife` tools;
- bpftrace;
- bpftool for feature inspection and loaded-program inventory;
- libbpf and a compiler toolchain for converting recurring probes to CO-RE objects;
- a continuous profiling agent compatible with the Parca server;
- container-runtime inspection commands;
- workload-oracle and fault-injection binaries.

BCC and bpftrace may require matching headers, LLVM/Clang, and runtime compilation. Capture their image cost and startup time. The production-style comparison should use precompiled, signed CO-RE probes where the guest kernel and BTF permit it.

### Containerized Toolbox Option

Evaluate a privileged toolbox container as a convenience, not as an assumption. It will typically need:

- host PID namespace access;
- read-only access to `/sys`, `/proc`, tracefs, BPF filesystem, kernel modules, and required symbols;
- the narrowest viable capabilities;
- access to container-runtime metadata for cgroup attribution;
- a read-only mount of target binaries only when uprobes or symbol resolution require it.

Compare this with a minimal system service in VM 1. Reject the toolbox option if it requires broad, hard-to-audit mounts or cannot resolve container identity reliably.

## 5. Common Experiment Method

Each experiment follows the same procedure:

1. Start the required VMs and record image digests, kernel versions, tool versions, vCPU/memory allocation, and topology.
2. Create a unique `experiment_id`, `vm_id`, `boot_id`, target container ID, and workload seed.
3. Verify the target is healthy and record a no-probe performance baseline.
4. Start an independent ground-truth counter or log outside the target container.
5. Attach only the probe required for the scenario, filtered to the target cgroup or PID when possible.
6. Inject the fault without restarting or modifying the target container.
7. Capture findings, lost-event counters, collector resource use, and target latency/throughput.
8. Detach the probe and verify that programs, links, maps, temporary artifacts, and privileged processes are cleaned up.
9. Repeat the run at least five times and compare detection rate, timing, p95/p99 overhead, and variance.
10. Store the run manifest and a short operator conclusion in VM 2.

A successful experiment must distinguish “no matching event” from “probe did not attach,” “target filter was wrong,” and “events were lost.”

## 6. Exploration Scenarios

### Experiment 0: Capability and Identity Baseline

**Purpose:** prove the environment is capable of tracing before diagnosing faults.

- Run `bpftool feature probe` and the custom preflight in VM 1.
- Start two ordinary containers plus one distroless container.
- Resolve each container to cgroup ID, namespace IDs, guest PID, and process start time.
- Load a harmless tracepoint probe, generate known events in only one container, and verify that filtering excludes the others.
- Recreate the target container and reboot VM 1 to test PID reuse, new cgroups, and new boot identity.

**Pass condition:** the supported hook types are recorded and events are attributed to the correct VM boot and container with no cross-container leakage.

### Experiment 1: Short-Lived Processes with `execsnoop`

**Fault:** a supervisor in the distroless target repeatedly launches a helper whose lifetime is too short for periodic `top` or `ps` sampling.

**Procedure:**

- Begin cgroup-filtered `execsnoop` collection from VM 1's agent.
- Trigger a deterministic number of successful and failed exec attempts.
- Capture executable, parent process, UID, timestamp, and safely redacted arguments.
- Increase the exec rate until event loss occurs and record the supported range.

**Pass condition:** at normal test rate, at least 95% of oracle events are observed, the restart pattern is apparent, and any loss is reported rather than silently ignored.

### Experiment 2: Failed Opens with `opensnoop`

**Fault:** the target attempts to open a missing configuration file, a denied certificate, and a missing shared-library path.

**Procedure:**

- Attach `opensnoop` from VM 1 with target-cgroup and failures-only filters.
- Trigger `ENOENT` and `EACCES` cases without entering or altering the target container.
- Verify the path, return code, errno, process identity, and time.
- Test path redaction or hashing before results leave VM 1.

**Pass condition:** every injected failure is associated with the exact expected path and errno, while successful opens and unrelated containers are excluded.

### Experiment 3: Guest Block-I/O Tail with `biolatency`

**Fault:** VM 1 experiences intermittent guest-visible storage delay or contention that averages hide.

**Procedure:**

- Generate direct and buffered I/O from a target container.
- Introduce controlled contention from a second VM 1 container; if available, use a guest-visible delay mechanism that does not require physical-host access.
- Record `iostat` alongside `biolatency` histograms and application latency.
- Separate filesystem/page-cache behavior from requests reaching the guest block layer.

**Pass condition:** the histogram exposes the injected tail and correlates with application stalls. The conclusion explicitly stops at the guest/virtio boundary and does not invent a host backing-device cause.

### Experiment 4: TCP Churn with `tcplife`

**Fault:** VM 3, or a VM 2 container, generates short connections, resets, failed connects, and tiny request/response exchanges against VM 1.

**Procedure:**

- Run `tcplife` in VM 1 and filter results to the target service.
- Capture source, destination, lifetime, process, and byte counts.
- Supplement it with scoped TCP state, retransmit, or drop probes because `tcplife` does not explain every network failure.
- Repeat for IPv4 and IPv6 if both are supported.

**Pass condition:** the responsible process and connection pattern are identified without packet payload collection, and the runbook states when another TCP probe is required.

### Experiment 5: One-Off Function Question with bpftrace

**Fault:** a stable kernel function or a versioned target application function has injected latency.

**Procedure:**

- Start with a tracepoint or stable hook; use kprobes only with an explicit kernel compatibility check.
- For application tracing, make the target binary and matching build ID available to the VM 1 loader without installing bpftrace in the target container.
- Use a reviewed bpftrace program to measure call count and a latency histogram.
- Test missing symbols, stripped binaries, inlining, nested calls, missed returns, and a changed application build.
- Store the script, filters, expected output, kernel/build compatibility, and safety limits.

**Pass condition:** the probe identifies the injected function delay and fails clearly on an incompatible kernel or binary. Any recurring script is scheduled for conversion to a tested CO-RE or maintained agent probe.

### Experiment 6: Continuous CPU History with Parca

**Fault:** target CPU use increases before a synthetic alert.

**Procedure:**

- Run the profiling agent in VM 1 before starting the baseline workload.
- Send profiles to the Parca server in VM 2 and confirm retention before injecting the fault.
- Enable a known CPU-heavy path, fire an alert later, and compare profiles from before and after the change.
- Resolve application stacks with a build-ID-indexed symbol store in VM 2.
- Repeat with missing symbols and without reliable frame pointers to document degraded stack quality.

**Pass condition:** a time-range comparison identifies the expected code path for at least 15 minutes before the alert, with unknown-frame and unwind-quality metrics visible.

### Experiment 7: Hubble Applicability Spike

Hubble is conditional because it observes Cilium-managed Kubernetes network flows; it is not a general Firecracker or ordinary container-network monitor.

**Procedure:**

- First decide whether running a small Kubernetes distribution plus Cilium inside the available VMs represents a real target environment.
- If yes, deploy the smallest supported topology, enable Hubble, and inject one policy-denied flow and one DNS failure.
- Confirm that source, destination, identity, protocol, and verdict answer the incident question.
- Measure the resource and setup cost relative to the three-VM budget.
- If Cilium is not part of the intended environment, record Hubble as not applicable and use the TCP experiments instead.

**Pass condition:** Hubble proceeds only if Cilium owns the tested datapath, flow verdicts are correct, and the nested cluster fits the playground. “Not applicable” is an acceptable result.

### Experiment 8: Multi-VM Isolation

**Fault:** VM 1 and VM 3 run similar containers and process names, but only one is targeted.

**Procedure:**

- If VM 3 is available, install the same minimal guest agent and target workload there.
- Issue an authenticated trace request for one VM and create matching events in both.
- Verify that each guest loads its own probe and labels events with its own VM and boot identity.
- Terminate and recreate one VM, then confirm stale requests and results are rejected.

**Pass condition:** no tool or dashboard implies cross-VM attachment, and no event is attributed solely by PID or process name.

## 7. Results and Correlation Model

Store normalized findings rather than relying on terminal transcripts:

```json
{
  "schema_version": 1,
  "observed_at": "2026-09-01T12:00:00.123456Z",
  "experiment_id": "exp-...",
  "vm_id": "vm-1",
  "boot_id": "...",
  "kernel_release": "...",
  "sensor": "opensnoop",
  "probe_version": "sha256:...",
  "container_id": "...",
  "cgroup_id": 12345,
  "pid": 432,
  "process_start_time": 987654321,
  "quality": {
    "events_lost": 0,
    "target_resolved": true,
    "symbols_resolved": true
  },
  "finding": {
    "operation": "open",
    "errno": "ENOENT",
    "path": "<redacted>"
  }
}
```

Never use PID alone as an identity. Combine the playground VM ID, guest boot ID, cgroup/container identity, PID, and process start time. Use monotonic guest time for durations and synchronized wall time for comparison among VMs. Record clock offset and uncertainty in multi-VM runs.

## 8. Safety and Operational Controls

### Probe Policy

Every trace request should specify:

- requester and experiment or incident ID;
- target VM, boot ID, container/cgroup, and optional PID;
- allow-listed probe artifact and attachment points;
- kernel/application compatibility constraints;
- maximum duration, map memory, buffer size, stack depth, sample rate, and export bandwidth;
- fields that may be collected and fields that require redaction;
- CPU, event-loss, target-latency, and backlog stop thresholds;
- automatic detach and cleanup behavior.

Use signed or checksummed probe artifacts. Arbitrary remote bpftrace text must not be a production default. A local administrator may run exploratory scripts in the playground after reviewing their attachment rate and data access.

### Kill Switch and Cleanup

The VM 1 agent must stop a probe when its lease expires even if VM 2 disappears. Provide a local emergency command in VM 1 to detach all exploration probes. After every test, compare `bpftool prog show`, `bpftool link show`, maps, pinned paths, agent processes, file descriptors, and memory with the baseline.

### Privacy

Command arguments, paths, IP addresses, process names, container metadata, and stack symbols may be sensitive. Filter and aggregate inside VM 1, avoid payload capture, redact before export, encrypt VM-to-VM transport, and apply short retention to raw events. Log who requested each probe and when it was detached.

## 9. Measurement and Testing

### Accuracy

- Compare observations with deterministic ground-truth counts.
- Calculate recall, false attribution, timestamp error, and event-loss rate.
- Test PID reuse, container recreation, VM reboot, stale boot IDs, and concurrent workloads.
- Validate errno, latency units, histogram buckets, IP formatting, byte counts, and symbol/build-ID matches.

### Overhead

For every tool, compare:

1. target with no exploration agent;
2. idle agent with no active probe;
3. normal filtered probe;
4. highest approved event or sampling rate;
5. the proposed always-on combination.

Measure VM vCPU, memory, BPF map memory, output volume, target throughput, median and p99 latency, drops, and cleanup time. Because VM 2 and VM 3 may share the same physical host, avoid claiming physical isolation; record noisy-neighbor effects as a limitation.

Initial hypotheses to validate are less than 2% VM 1 vCPU and 100 MiB memory for the always-on profiler/agent, and less than 3% p99 workload-latency regression. On-demand tracing receives a separate, explicit budget.

### Resilience and Security

- Restart the VM 1 agent, VM 2 collector, and container runtime during active runs.
- Disconnect VM 2 and verify local expiry and bounded buffering.
- Fill buffers and slow the receiver to verify visible loss and backpressure.
- Reject expired, replayed, wrong-VM, and wrong-boot requests.
- Attempt access from an unprivileged target container and confirm it cannot control the agent or BPF filesystem.
- Run a 24-hour soak of the final always-on configuration.

## 10. Tool Scorecard

Score each tool from 0 (unacceptable) to 3 (strong):

| Dimension | Weight | Required evidence |
| --- | ---: | --- |
| Diagnostic correctness | 25% | Scenario oracle matches and loss is measurable |
| VM/kernel compatibility | 15% | Preflight and attachment pass on supported guest images |
| Container attribution | 15% | Correct cgroup/container identity without target modification |
| Security and privacy | 15% | Minimum privileges, filtering, redaction, and audit are acceptable |
| Performance | 10% | Measured normal and worst-approved overhead fit budgets |
| Operability | 10% | Start, health, timeout, kill switch, and cleanup are reliable |
| Three-VM fit | 5% | Required components fit available vCPU, memory, and VM count |
| Maintainability | 5% | Versions are pinned and probes have an upgrade/test path |

A zero for correctness, isolation, or cleanup is an automatic no-go. Record unsupported tools honestly rather than weakening the VM or container boundary to make them pass.

## 11. Delivery Phases

### Phase 0: Inventory and Image Preparation

- Record available VM kernel, architecture, vCPU, memory, disk, networking, and container runtime.
- Decide the one-, two-, or three-VM topology and create stable VM identities.
- Prepare the trusted VM 1 tooling and VM 2 collection stack.
- Define privilege, data classification, performance budgets, and unsupported host-level questions.

**Exit gate:** preflight passes or produces explicit blockers, and a target container cannot access the tracing control surface.

### Phase 1: Core BCC Exploration

- Complete Experiments 0–4.
- Produce one symptom-driven runbook per tool.
- Measure detection accuracy, event loss, and overhead.
- Compare VM service and privileged toolbox-container deployment.

**Exit gate:** the four core tools diagnose their intended fault without changing or restarting the target.

### Phase 2: Custom and Historical Profiling

- Complete bpftrace and Parca Experiments 5–6.
- Establish external symbol handling and build-ID checks.
- Promote any repeatable bpftrace question into a versioned probe proposal.

**Exit gate:** function latency and previously collected CPU history are reproducible, and degraded symbol behavior is documented.

### Phase 3: Conditional and Multi-VM Work

- Complete the Hubble applicability decision.
- Complete multi-VM isolation when a third VM is available.
- Run load, receiver-failure, lease-expiry, kill-switch, and soak tests.

**Exit gate:** optional tools have evidence-based decisions and the final configuration stays within the three-VM budget.

### Phase 4: Recommendation and Runbooks

- Select always-on, on-demand, lab-only, conditional, and rejected tools.
- Publish resource requirements, compatibility matrix, probe catalog, and safe command templates.
- Document known blind spots caused by the inaccessible Firecracker host.
- Run a game day in which an operator follows only the runbooks.

**Exit gate:** the operator diagnoses every applicable injected fault, all probes clean up, and the scorecard has an owner-approved result.

## 12. Recommended Starting Configuration

Start with two VMs and add the third only for independent traffic or isolation tests:

- **VM 1:** target containers, a minimal guest tracing agent, BCC/bpftrace toolbox, bpftool, and continuous profiling agent.
- **VM 2:** orchestrator, event results, Parca server/UI, symbol store, and ordinary network peer containers.
- **VM 3:** optional external load generator, DNS/TCP peer, or second independently monitored guest.

Keep continuous CPU profiling and low-cardinality agent health enabled. Activate `execsnoop`, `opensnoop`, `biolatency`, `tcplife`, and custom probes on demand with narrow filters and short leases. Treat Hubble as conditional on a genuine Cilium/Kubernetes use case.

## 13. Deliverables

- VM image/setup scripts and a machine-readable kernel/BPF preflight report.
- Version-pinned container manifests for the target, generators, Parca server, and optional supporting services.
- Automated scenarios with ground-truth oracles and run manifests.
- Compatibility, accuracy, overhead, event-loss, and tool-scorecard reports.
- A normalized event schema and VM/container identity mapping.
- Reviewed probe catalog with limits, filters, compatibility, and cleanup instructions.
- Symptom-to-tool incident runbooks and an emergency-detach procedure.
- Security/privacy review and raw-data retention policy.
- A final recommendation identifying always-on, on-demand, lab-only, conditional, and rejected components.

## 14. Definition of Done

The exploration is complete when it runs entirely within no more than three Firecracker VMs; diagnoses the applicable process, filesystem, block-I/O, TCP, function, and CPU-history faults without modifying or restarting the distroless target; proves that every eBPF loader runs in the same VM kernel as its target; reports event loss and degraded symbol quality; measures overhead and cleanup; enforces scoped, expiring probe policies; records Hubble as either justified or not applicable; clearly documents the inaccessible-host blind spot; and produces repeatable runbooks and an approved tool scorecard.

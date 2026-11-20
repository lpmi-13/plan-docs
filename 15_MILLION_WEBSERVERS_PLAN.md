# 15 Million Web Servers: iximiuz Labs Challenge Plan

## 1. Goal and scope

Build a hands-on iximiuz Labs challenge that follows the investigation in [“15 million web servers on one VM”](https://kapteko.com/15-million-webservers/): start with a tiny C server, run out of ports, solve the address problem with AnyIP, run out of process memory, make and disprove the “idle sockets cost nothing” assumption, then replace kernel TCP listeners with a compact user-space runtime and `smoltcp`. The learner should see the failures, measure the resources that caused them, and finish with **15,000,000 distinct HTTP listeners on one server VM**. A client VM probes a sample of those listeners through real TCP connections.

Use **4 vCPUs and 10 GiB RAM for the x86_64 server VM**, plus a small, separate client VM. The article reports a one-core, roughly 10 GB result, but that is an experimental measurement on its own AArch64, custom-kernel setup, not a guarantee for iximiuz hardware. Passing the final task requires a measured 15-million-listener run within the configured server VM; a smaller count must never be presented as a pass. The challenge’s required network backend is **TUN**. TAP is a useful comparison. AF_XDP and DPDK are optional advanced material only after their VM prerequisites are verified.

This is a systems investigation, not an exercise in launching 15 million Linux processes. The final “server” is a distinct destination IP and port, a resident `smoltcp` listener, and a resumable guest context in one process. State this definition in the lesson and show that several widely separated addresses return HTTP. Do not claim 15 million simultaneous active connections, one million active connections, full POSIX compatibility, or a particular requests-per-second figure. The article’s [proof of concept](https://github.com/artemyarulin/AI10M/tree/main/poc/ten-million-webservers) uses a fixed, trusted assembly guest and a syscall dispatch shim; it is not a general LFI sandbox. Preserve that distinction in the editorial.

## 2. Source facts and feasibility budget

| Article step or number | How the lab uses it |
|---|---|
| Minimal C server calls `socket`, `listen`, `accept`, `recv`, `send`, `close`; article reports about 65k RPS | Compile and run the actual baseline server, verify one HTTP response, and optionally benchmark it. RPS is an observation, not a graded target. The original C source uses implicit bind via `listen()` on an unbound socket. |
| Port-0 loop fails after roughly 28,232 listeners with the example `32768–60999` ephemeral range | Reproduce in one bounded process, log the actual range, listener count and `errno`. Expect `EADDRINUSE` on the tested kernel, but grade the observed exhaustion and explanation rather than a hard-coded count or message. |
| An IP plus a 16-bit port identifies a TCP listener; AnyIP supplies many usable addresses | Show the IPv6 local route from the article with a small pair of servers. The final PoC is IPv4, using a private `/8` inside the isolated lab. The “450 NICs” and “32-bit ports” ideas are discussion prompts, not commands to run. |
| One process per address eventually runs out of memory; article estimates about 1.5 MB per process | Spawn only enough processes to demonstrate growth inside a restricted cgroup. Measure actual PSS/cgroup memory and extrapolate. Do not try the literal 15-million-process shell loop. |
| A guest context is 16 bytes; the article incorrectly assumes idle TCP listeners need zero kernel memory | Have learners calculate `15,000,000 × 16 = 240,000,000 bytes ≈ 229 MiB`, then run a controlled idle-socket test that disproves zero cost. |
| Article estimates about 3,384 bytes per Linux socket and about 620 bytes per idle user-space server; it reports 16 million in its environment | Treat all as reference measurements, not constants. `15,000,000 × 620 B = 9.3 GB ≈ 8.66 GiB`; a 10 GiB VM leaves about 1.34 GiB for the OS, runtime overhead, packet buffers, and active connections. At a strict decimal 10 GB, the headroom is only 0.7 GB. The release test must measure the real peak and leave operating headroom. |

The article’s examples use underscore-separated numbers in shell commands (for example, `seq 1 15_000_000`), which are illustrative rather than directly executable on standard `seq`. Use `15000000` in scripts. Its IPv4 PoC starts at `10.64.0.1` and routes `10.0.0.0/7`; 15 million consecutive addresses would extend into public `11/8`. Adapt the lab to a base of **`10.0.0.1`**, which keeps all 15 million addresses within RFC 1918 `10/8`; route only `10.0.0.0/8` on the private playground network. The last address is `10.228.225.192`.

The iximiuz VM architecture is confirmed to be **x86_64**. The linked source currently has unconditional DPDK build requirements and AArch64-only `global_asm!` in `src/main.rs`, while its scheduler in `src/scheduler.rs` holds one `Task` and one `smoltcp` `Listener` per server. It cannot simply be copied into the x86_64 rootfs and run. Pin a source commit before implementation, record license/attribution, and make a lab-specific fork with a reproducible build. Compare an x86_64 port of the fixed guest entry/yield shim with a compact Rust state machine before choosing the required runtime. Separate TUN/TAP from AF_XDP/DPDK behind Cargo features so the required build does not depend on `libdpdk`, VFIO, or XDP support.

### x86_64 feasibility study: choose the runtime by measurement

Run this study **before** writing the final learner task or committing to an implementation. Keep the network path (`smoltcp` + TUN), listener data structure, 64-byte RX/TX buffers, compiler profile, allocator, HTTP response, client probes, and VM image the same across candidates. Change only how a guest is represented and resumed. This separates scheduler cost from network/backend cost. Build all candidates for `x86_64-unknown-linux-gnu` with a pinned Rust toolchain and lockfile. An AArch64 build or benchmark is not evidence of feasibility on iximiuz.

| Candidate | What to build and test | Likely constraint / decision |
|---|---|---|
| **A. x86_64 fixed-guest assembly port** | Port `http_guest`/`enter_guest` and the syscall gate to the System V x86_64 calling convention and Linux x86_64 syscall numbers. Keep the two FDs and resumption point in a 16-byte `Task` only if register liveness proves that sufficient. Check stack alignment, callee-saved registers, return address, red-zone assumptions, and every yield/return path. | Closest to the article’s fixed guest; highest ABI and unsafe-code risk. Fail the candidate for any state corruption, repeated-request bug, or per-guest context growth that breaks the memory budget. |
| **B. explicit Rust state machine** | Express the fixed server loop as states such as `WaitingForAccept`, `WaitingForRead`, and `Replying`, and call the same host socket/`smoltcp` operations. Use a compact array element with state plus server/client handles; assert its exact `size_of` and inspect actual vector capacity. | Portable, easier to inspect and likely the preferred fallback if A does not pass. It models guest behavior but does not literally intercept compiled C syscalls; the lesson must say so if selected. |
| **C. stackless async task per listener** | Make a minimal future for each idle listener with the same `smoltcp` socket state and a bounded ready queue. Measure the future, task header, wakeup registration and executor overhead as separate allocations. | Retain only if idle per-listener cost and startup time fit. A convenient async API is not evidence of low memory use. |
| **D. stackful fiber or OS thread per listener** | Prototype 1k–100k and measure reserved and committed stack memory, scheduler overhead and process limits. Extrapolate before scaling. | A 4 KiB stack alone would reserve about **57.2 GiB** for 15 million listeners, so this is an instructive negative control unless a radically smaller shared-stack design is demonstrated. Do not run 15 million fibers/threads. |

Candidate A and B are the serious implementation routes. Candidate C tests whether an off-the-shelf scheduler can replace the custom one without losing the article’s memory advantage. Candidate D explains why simply moving from processes to stackful tasks is insufficient. A full LFI sandbox or arbitrary Linux/POSIX guest is a separate research project; it is not required to answer this challenge’s fixed-server question.

For each candidate, use a **fresh process and cgroup** for `1`, `1,000`, `100,000`, and `1,000,000` listeners, then `5,000,000` only if the lower-scale slope passes. Avoid reusing a process after deallocation because allocator retention can corrupt comparisons. At each size, record build/commit ID, `size_of` of every per-listener structure, initialized count, cgroup `memory.current` and `memory.peak`, VM available memory, startup seconds, idle CPU, and first/middle/last HTTP responses. Use repeated runs and report the worst stable idle memory and startup time. Run a small request/concurrency test **after** the idle measurement to check wakeups and repeated connections; record active-connection overhead separately. Compare one-core affinity for article parity, then the allowed four-core configuration for usable challenge startup and responsiveness.

Fit a conservative memory model `M(N) = M_fixed + N × B_idle` from more than one scale, using the measured allocation/cgroup deltas rather than `size_of::<Task>()` alone. The theoretical upper bound after reserving 512 MiB of a 10 GiB VM for everything outside the runtime is about **680 bytes per listener**, even with zero fixed runtime cost or safety margin. As an illustration, 512 MiB non-runtime use + 256 MiB safety + 256 MiB fixed runtime leaves about **644 bytes per listener**. Use actual measured values in `B_max = floor((10 GiB − observed non-runtime peak − safety reserve − M_fixed) / 15,000,000)`. The article’s ~620 B figure is therefore close to the practical ceiling. Account for peak allocation during construction as well as steady-state use.

Promote a candidate to a 15-million run only if it passes correct HTTP routing and the measured upper-bound projection stays below the real VM/cgroup budget with headroom; then prove the result with a fresh full run. A candidate that merely stores 15 million indices or routes all requests through one wildcard socket does **not** meet the resident-listener definition. If no candidate passes, profile the listener/socket layout and allocator before changing the goal. Record the rejected candidates and measured reason in `harness/architecture-study.md` for the editorial.

## 3. Challenge design

**Format:** `kind: challenge`, backed by a named custom playground and OCI rootfs, following the deployed [GraphQL challenge](https://labs.iximiuz.com/challenges/graphql-vuln-lab-ca696b14) and its local reference under `/home/adam/projects/iximiuz-challenges/graphql-vuln-lab`. That reference demonstrates `index.md` front matter, `::simple-task` checks, a separate `solution.md`, a custom playground `manifest.yaml`, rootfs Dockerfile, readiness check, and build/publish scripts. The challenge must reference the **playground name**; the rootfs drive belongs in the playground manifest, as the GraphQL implementation discovered during publishing.

**Story:** “Can one small VM host 15 million web servers?” Give the learner a working baseline server, instrumented failure probes, the compact Rust runtime source, and a final target. Tasks require predictions, observations, and a live final state. Keep the conclusion open until measured. The learner may inspect and modify the implementation; the challenge guides through the key experiments instead of asking them to write a complete TCP stack or assembly runtime from scratch.

**Machine topology:**

```text
client-01 (2 vCPU, 2 GiB)  -- 172.16.0.0/24 private network --  dev-01 (4 vCPU, 10 GiB)
curl and sampled probes      route 10.0.0.0/8 via dev-01       TUN -> smoltcp -> guest HTTP
```

- `dev-01` has a 20 GiB rootfs drive, default `laborant` user, and a management NIC at `172.16.0.2/24`. `client-01` has `172.16.0.3/24`. The network is playground-local; the lab never advertises `10/8` outside it. Use a client VM so benchmark traffic does not take the server’s RAM or CPU budget. If a second VM is unavailable in the target account, use a network namespace on `dev-01` for correctness probes and state clearly that throughput results share the VM.
- The starter rootfs contains C baselines, the Rust source and a **prebuilt release binary**, a narrowly scoped probe toolbox, and root-owned graders. Compile at image-build time; a Cargo build during the 15-million run would distort memory and may OOM.
- `dev-01` needs `/dev/net/tun`, `CAP_NET_ADMIN` for the isolated TUN/AnyIP exercises, cgroup v2 memory accounting, a high enough file descriptor limit for bounded probes, and a routable private link to `client-01`. Verify these on a real started playground before locking the design. Do not change global `ip_local_port_range` or attach DPDK to the management NIC.
- Run the final service in its own systemd unit/cgroup with an explicit memory ceiling chosen from a measured VM baseline and **at least 512 MiB reserved for the OS and graders**. When idle, the runner may use one CPU; the extra vCPUs keep management and probing responsive. Record `memory.current`, `memory.peak`, `memory.events`, RSS/PSS, startup time, and sample request latency. `MemoryMax` is protection against losing the VM, not a substitute for the 10 GiB whole-VM acceptance limit.
- Prepare an idempotent `reset` command that stops services, terminates bounded probe cgroups, removes only lab-created namespaces/routes/TUN devices, and restores starter configuration. Root privileges are exposed through fixed sudo commands where possible, not a general root shell requirement in the lesson.

### Proposed repository layout

```text
15-million-webservers/
├── README.md                    # local build, test, and deployment notes
├── index.md                     # challenge front matter and learner narrative
├── solution.md                  # editorial with expected failures and recovery
├── baseline/                    # minimal C server, explicit bind variant, probes
├── runtime/                     # pinned/adapted Rust PoC and dependency lockfile
├── toolbox/                     # safe experiment, metrics, reset, client scripts
├── tasks/                       # root-owned readiness and verification scripts
├── harness/                     # integration checks and memory/run reports
│   └── architecture-study.md     # x86_64 candidate results and selection
└── playground/iximiuz/
    ├── Dockerfile
    ├── manifest.yaml
    ├── build-rootfs-image.sh
    ├── publish.sh
    └── image/                   # systemd units and sudo policy
```

Keep command output and measurements under `/var/lib/15m-lab/runs/<run-id>/`, with a documented JSON schema and a short human-readable summary. Do not bake a fake successful run into the image. Probe scripts write atomically so a killed experiment does not look complete.

## 4. Learner path and grading

The lesson should ask for a prediction before each failure, show the probe command, then ask the learner to inspect a measurement and explain the failed assumption. `::hint-box` sections give progressively more precise help. The editorial shows representative output **labelled as an example**, because kernel versions and limits change exact counts and wording. Grade live behavior and structured evidence generated by trusted probes, not string matching on the learner’s prose or accepting a touched marker file.

| Order / check | Learner action and visible result | Verification rule |
|---|---|---|
| 0. `init_wait_ready` | Confirm `x86_64`, cgroup v2, TUN device, private link, and baseline binary. | Fail early with a clear prerequisite message if the chosen playground cannot support the required path. |
| 1. `verify_baseline_http` | Compile or start `baseline/webserver_libc.c`; `curl` one request and see `HTTP/1.0 200 OK` and `ok`. Use `strace` or source to identify `socket`, implicit bind/listen, accept, recv, send, close. | Real response from the learner-started baseline, with valid status and body. No RPS threshold. |
| 2. `verify_port_exhaustion` | Run a **single-process** probe in a dedicated network namespace, repeatedly calling `socket(); listen()` with no explicit bind. It reads `ip_local_port_range`, raises its own `RLIMIT_NOFILE`, closes all sockets on exit, and stops at exhaustion or a safe upper bound. See `listen: Address already in use` / `EADDRINUSE` when ports run out. | Probe reports actual range, count, failed syscall and `errno=EADDRINUSE`; `count` is consistent with available ports. If the kernel fails for another resource first, show that diagnostic and fix the probe/limits before release. |
| 3. `verify_anyip` | Add a **local** IPv6 route for documentation prefix `2001:db8:1234:5678::/64` in the lab namespace, bind two separate C servers to two IPv6 addresses on the same port, and `curl -g -6` both. Observe the port-ceiling workaround. Discuss why adding 450 NICs or widening the TCP port field would not be a sensible solution. | Two independent IPv6 addresses answer with their own identifier; route is present only in the lab namespace. This stage does not imply that the final Rust PoC supports IPv6. |
| 4. `verify_process_cost` | Launch a bounded set of address-bound C servers under a small cgroup memory limit. Inspect process count, PSS and `memory.current`; continue until the cgroup records an OOM kill or a controlled memory refusal. See a `Killed`/exit-137 style failure and calculate a measured per-process extrapolation to 15 million. | Evidence contains the cgroup limit, before/after memory, peak count, termination reason and an extrapolation. The host remains responsive; grader runs outside the test cgroup. Do not depend on exactly 1.5 MB or one specific failure string. |
| 5. `verify_kernel_socket_cost` | First calculate the “zero idle-connection memory” prediction from the article. Then create many **unbound, idle** kernel TCP sockets in one process under another bounded cgroup, measure `memory.current` and `memory.stat` slab fields, and continue until the cgroup limit is reached. Observe that even unused sockets consume kernel-accounted memory and that the process or `socket()` fails under the limit. | Memory growth is positive and material per socket; evidence includes socket count, bytes per socket estimate, `memory.events` or precise syscall error. A final question asks which assumption failed. |
| 6. `verify_runtime_tun` | Inspect the 16-byte task context and `smoltcp` listener allocation, set up TUN and the `10/8` route, start with 1k then 100k then 1m listeners, and finally request `LISTENERS=15000000`. Observe resident-listener count, startup duration, memory peak and HTTP replies. | A live runner reports **15,000,000 resident initialized listeners**; client probes first, middle, last, and reproducibly selected random destination IPs on port 8080; all answer correctly while the same process stays alive. The VM and process stay within the memory budget with no OOM event. |
| 7. `verify_comparison` (optional, ungraded) | Compare TUN and TAP on matched listener count/request profile. Explore AF_XDP/DPDK only on a dedicated compatible VM. | Produce a report; no dependence of core completion on unavailable kernel, NIC, IOMMU, or hugepage features. |

Make checks 2, 4, and 5 **bounded and repeatable**. The article’s literal `for ... ./server &` and ten-million-socket loops are useful narrative examples, but must not be offered as commands on a shared playground. Label the anticipated error before running each probe, then show the observed error. If the exact expected error does not occur, retain the evidence and explain what limit fired first; fix the experiment before publishing rather than teaching a fabricated result.

### What the final verifier must prove

1. Ask the runtime for a machine-readable `status` report with count, backend, base/last IP, port, initialized listener count, and process ID. The reporter must read the live process, not a static file. The PoC’s `resident_listeners_verified=` startup line can be one input, but alone is insufficient.
2. Confirm the process PID remains alive and `memory.events` has no new `oom` or `oom_kill` events. Capture whole-VM available memory and cgroup peak.
3. Probe `10.0.0.1`, `10.114.112.224` (mid-range), `10.228.225.192`, and several deterministic pseudorandom indices from `client-01`. Prefer distinct response headers or bodies containing the destination/listener ID; otherwise correlate server-side accepted-flow counters with each probe. A single wildcard responder returning the same `ok` must not satisfy the lesson’s “15 million distinct servers” claim.
4. Re-probe the first and last addresses after the random set to catch premature exit or an accidentally lazy count. Verify no listener outside the configured range responds.
5. Retain the final run’s peak, startup time, listener count and sample HTTP outcomes in a grader report visible to the learner. Do not require a fixed RPS; optional throughput comparisons use an external client and record exact test settings.

The current PoC logs a count after constructing listeners and has a listener object for each index. Add a read-only status socket or signal-driven metrics snapshot and an address-aware response for strong checks; inspect its memory cost at 15 million before committing to either. If adding an ID to each reply is too costly, generate it from the index when handling the request rather than storing a per-listener string.

## 5. Runtime and networking implementation

### 5.1 Port and process experiments

Use the upstream `webserver_libc.c` as the starting C program, with an added diagnostic build that prints `perror()` and `errno` on failed `socket`, `bind`, and `listen`. A second, explicitly bound variant takes an IP and port argument for the AnyIP/process stages. Keep both versions easy to compare. The port exhaustion experiment uses a purpose-built **single process** so `pids.max` and shell overhead cannot obscure the port ceiling. Its scope is limited by time, file descriptors and cgroup memory.

For process and idle-socket OOM experiments, use separate cgroups with small `memory.max` values; the launcher/checker lives outside them. The idle-socket probe should call `socket()` without `listen()` so the port ceiling cannot confound its memory result. Record `memory.current`, `memory.peak`, `memory.stat`, and `memory.events` before and after. A cgroup OOM may kill a child without the shell printing the article’s exact “Out of memory” text, so teach exit code and `oom_kill` evidence. Close sockets and reap children on normal exit or timeout. Do not use a whole-VM OOM as the teaching signal.

### 5.2 Addressing and TUN data path

Use the article’s IPv6 `ip -6 route add local ... dev lo` **inside a disposable network namespace** to show AnyIP without touching management traffic. The final optimized runtime uses IPv4 because the upstream Rust implementation’s `network.rs` and scheduler parse IPv4 and map consecutive indices to addresses. Set `base=10.0.0.1`, `count=15000000`, `port=8080`, and route `10.0.0.0/8` from `client-01` via `dev-01` on the playground’s private `172.16.0.0/24` link.

On `dev-01`, set up a TUN device owned by the runtime, enable forwarding only for the lab path, add the route for `10/8` to TUN, and adjust reverse-path filtering as needed for this asymmetric path. On `client-01`, add the corresponding route through the private server address. Validate packets with a 1-listener and then 1k-listener smoke test before any large allocation. Keep all management traffic on the normal NIC. Put route/TUN creation and teardown into idempotent scripts; the final systemd unit should depend on their successful setup and stop cleanly. Do not use a real public IPv4 block or the original source’s `10/7` route.

### 5.3 Rust PoC adaptation

1. Pin a known upstream commit and vendor only the needed PoC source under its license. Record which lines are original and which were changed for iximiuz.
2. Implement candidate A and B from the x86_64 feasibility study, then select the version that passes correctness and memory gates. The upstream `enter_guest`/`http_guest` assembly and syscall numbers are AArch64-specific. Test `socket` → `bind` → `listen` → `accept` → `recv` → `send` → `close`, `EAGAIN`, repeated requests, interleaved clients, and task isolation before scaling. For the assembly route, cross-check the same request traces against the Rust state-machine route and inspect all ABI-sensitive `unsafe` code. Never claim the fixed guest is a general Linux process or LFI sandbox.
3. Feature-gate the backends and `build.rs`: `tun` and `tap` build with `smoltcp`; `af_xdp` adds `xsk-rs`; `dpdk` adds the DPDK library, C adapter and VFIO dependencies. Pin Cargo versions and ship a lockfile. Compile in the Docker build, not the learner’s peak-memory run.
4. Preserve the upstream resident-listener design: one `Task`, one `Listener`/`SocketSet`, small fixed RX/TX buffers per idle server, and active connection state allocated only on demand. Add count, bounds and allocation diagnostics. Avoid a 15-million-element `HashMap` of idle entries, one OS socket per guest, one thread per guest, precomputed address strings, or per-listener HTTP response buffers.
5. Implement a startup state machine with progress logs at 1k, 100k, 1m, 5m, 10m and 15m, plus a bounded status endpoint and graceful shutdown. Count only fully initialized, listening entries. Detect arithmetic overflow at the last IPv4 address and reject invalid counts before allocation.
6. Verify memory at each scale. A source-level `size_of::<Task>() == 16` assertion is necessary but not sufficient: include `Listener`, `SocketSet`, allocator capacity, RX/TX buffers, network driver allocations and kernel/OS memory in the final accounting. If 15 million fails on the actual VM, compact the structures or buffers based on measurements and rerun. Keep the 15-million acceptance criterion.

### 5.4 Backends beyond TUN

After selecting an x86_64 runtime, test packet backends **separately** from the guest-runtime comparison. A CPU architecture does not establish that the virtual NIC or kernel offers XDP, VFIO or IOMMU. Keep the same runtime, listener count, client placement and request profile in each backend trial. Run a 1k-listener correctness test before any larger benchmark, and record the backend’s own fixed memory and CPU costs.

| Backend | x86_64 VM preflight and small-scale test | Core challenge decision |
|---|---|---|
| TUN | Confirm `/dev/net/tun`, device creation, private route, forwarding, repeated client TCP/HTTP requests, and no management-route change. | Required. If this fails, fix the playground or stop the challenge build. |
| TAP | Confirm Ethernet-frame path, ARP/neighbour handling, and the same sampled HTTP requests. Compare its memory/CPU at 1k and 100k. | Optional comparison if correct; never a substitute for a failing TUN acceptance run without redoing all measurements. |
| AF_XDP | Inspect kernel and virtual NIC driver, XDP mode, permissions and XSK creation. Run the 1k probe and record whether the actual mode is generic copy, native copy or zero copy. | Optional only if supported. The [kernel documentation](https://docs.kernel.org/networking/af_xdp.html) distinguishes those modes; do not promise zero copy or source-code-equivalent performance. |
| DPDK | Require a **separate data NIC**, usable VFIO/IOMMU, hugepages and an isolated route; verify device binding and 1k replies while management access stays up. | Optional only on a dedicated compatible VM. The [DPDK Linux driver guide](https://doc.dpdk.org/guides/linux_gsg/linux_drivers.html) explains why a DPDK-bound device may leave Linux networking. Never rebind the management NIC. |

If AF_XDP or DPDK fails preflight, record the specific missing capability in the architecture study and keep the lesson’s conceptual explanation. Their absence does not prevent a TUN-backed 15-million-listener challenge. Throughput comparisons remain ungraded.

## 6. Build, release, and verification plan

### Phase A: platform and source spike

- Start a disposable custom playground with the intended `4 CPU / 10 GiB` x86_64 server and client; capture the kernel version, `/dev/net/tun`, cgroup v2, available memory, and network interface layout. Make the readiness check require `uname -m = x86_64`.
- Pin and inspect the upstream PoC commit, license, Cargo dependencies, AArch64 assembly, backend prerequisites and any unreviewed unsafe code. Build the x86_64 fixed-guest port and explicit Rust state machine against the same TUN/`smoltcp` path. Confirm a 1-listener reply and equivalent repeated-request behavior before scaling.
- Run the candidate study in §2, including controlled comparisons with stackless async and stackful tasks. Capture the measured per-idle-server slope, fixed overhead, peak during construction, startup time and correctness. Write the candidate decision and raw metrics to `harness/architecture-study.md`. If every projection exceeds the safe budget, optimize before writing a 15m task; do not weaken the target or hide a failure.

### Phase B: learner assets and bounded failures

- Implement the two C server variants and three probe tools. Run each under fixed timeout and cgroup resource limits. Verify actual error paths, cleanup, and that repeated use leaves `ip addr`, routes, cgroups and processes as expected.
- Write `index.md` in the sequence in §4, with commands learners can paste and `::simple-task` checks. Put architecture details, formulas, example output, and recovery steps in `solution.md`.
- Build the OCI rootfs from `ghcr.io/iximiuz/labs/rootfs:ubuntu-24-04`, following the GraphQL challenge’s image pattern. Add the compiled runtime, C tools, client tools, systemd units, trusted graders and a non-root default user. Keep the image source and lockfiles reproducible.

### Phase C: final 15-million integration

- In a fresh 10 GiB x86_64 server VM, run the selected implementation through 1k → 100k → 1m → 5m → 10m → 15m with TUN. Capture peak memory, startup duration, `memory.events`, CPU, sample TCP/HTTP results and VM responsiveness at each step.
- Repeat the final run from a clean boot. Confirm 15 million resident listeners and sampled first/middle/last/random addresses from `client-01`. Check that the starter image never relies on cached success output.
- Test one failed startup: a too-low memory limit or malformed route must exit with a useful diagnostic and leave the VM recoverable. Test `reset` after failed and successful runs.
- Benchmark only after correctness is established. Record concurrency, duration, client CPU and network mode; keep performance figures out of the graded acceptance check.

### Phase D: iximiuz packaging and publication

- Create a named custom `flexbox` playground manifest with both VMs, rootfs drive(s), terminal tabs, resources, readiness tasks, and private network. Keep the 15-million runtime stopped at boot; learners start it after the failure experiments. A browser `http-port` tab is optional for a small status page and must not imply it exposes all 15 million IPs.
- Create `kind: challenge` `index.md` that references the playground **by name**, with `tasks:` calling the baked verification scripts. Follow the GraphQL reference’s publish ordering: build/push versioned rootfs, update playground, then push challenge content. Use private/unlisted visibility until the full run passes; publish only after the user authorizes that external action.
- Run a local harness and at least one real iximiuz session as the default learner. Verify every task starts red and turns green only after its intended observation/action; verify the final task fails for 1m or a fake count, for a wildcard responder, for an OOMed process, and for unreachable endpoints.

## 7. Acceptance criteria and release blockers

The challenge is ready when:

1. Every article misstep that the learner performs produces a real, measured limit in a confined experiment: port exhaustion, process memory failure, and idle-kernel-socket memory failure. Exact numeric failure points may vary; the underlying cause must be unambiguous.
2. AnyIP works with two live IPv6-bound C servers; the final private IPv4 route/TUN path works without disturbing management access.
3. One **x86_64** server VM with **4 vCPUs and 10 GiB RAM** has **15,000,000 fully initialized user-space listeners at once**. The live process remains healthy, no OOM event occurs, the VM remains usable, and client probes reach widely separated listeners that identify their own destination.
4. The final report contains measured whole-VM/cgroup memory peak, startup time, listener count, backend and sample HTTP results. A clean rerun reaches the same outcome.
5. A fresh learner can follow the challenge and reset it without manual repair; graders reject incomplete or fabricated state.

Block release if the target VM cannot expose TUN/cgroup controls, if neither x86_64 runtime candidate passes its correctness and memory study, if 15 million exceeds real memory headroom, or if final traffic cannot be routed and distinguished by destination. In those cases, keep the work as an unpublished experiment while fixing the measured blocker. DPDK and AF_XDP availability is **not** a blocker for the core challenge.

## References

- [Kapteko article](https://kapteko.com/15-million-webservers/) and [linked PoC](https://github.com/artemyarulin/AI10M/tree/main/poc/ten-million-webservers).
- [iximiuz Labs challenge format](https://labs.iximiuz.com/docs/learning-materials/challenges), [custom playground resources](https://labs.iximiuz.com/blog/iximiuz-labs-playgrounds-2-0), and the local GraphQL challenge reference at `/home/adam/projects/iximiuz-challenges/graphql-vuln-lab`.
- [Linux AF_XDP documentation](https://docs.kernel.org/networking/af_xdp.html) and [DPDK Linux driver documentation](https://doc.dpdk.org/guides/linux_gsg/linux_drivers.html).

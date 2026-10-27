# Linux Performance Detective: iximiuz Labs plan

## Decision

**Build one curated, approximately 90–120 minute tutorial with an attached single-VM playground.** Use the `perf-labs` programs as black-box fault scenarios and the `bpf-perf-workshop` progression as the teaching model: observe a symptom, narrow it with ordinary Linux tools, use BCC/BPF tools for the missing detail, and write one small bpftrace probe. The two repositories are closely related; publishing them as two unrelated labs would repeat the same CPU and disk investigations.

This is a synthesis, not a verbatim port of all exercises. Ship three core cases and a probe-building finale. Keep SSH login timing, generic tool-writing exercises, and the remaining fault programs as optional follow-on material. If full coverage becomes a requirement, expand into two linked tutorials that **share the same playground**: (1) performance diagnosis and (2) writing bpftrace tools. The split is by learner objective, not by source repository.

**Publication requires both gates below to pass:** the selected iximiuz guest must run the planned BPF tools against the actual scenarios, and every redistributed source asset must have documented reuse rights or an independent replacement. Private prototyping can proceed while either gate is open.

## Source review and selection

| Source | Actual behavior or exercise | Treatment in the combined lab |
| --- | --- | --- |
| `perf-labs/src/lab001.c` | One tight CPU loop with a simple hot call stack | Optional warm-up for CPU profiling; too simple for a core mystery. |
| `perf-labs/src/lab002.c` | TCP server mostly blocked in `accept()` | Optional lesson on distinguishing idle time from a fault; not a diagnosis challenge on its own. |
| `perf-labs/src/lab003.c` | Repeated zero-length `read()` calls burn CPU | Duplicate of the workshop CPU case; choose the richer threaded workshop variant for the core case. |
| `perf-labs/src/lab004.c` | Repeated, short-lived `cksum` subprocesses | **Core case 1:** explain high load that process snapshots miss; observe with `execsnoop`. |
| `perf-labs/src/lab005.c` | Repeated `O_SYNC` writes | Alternative or comparison for the disk case; overlaps workshop `lab001`. |
| `perf-labs/src/lab006.c` | Two CPU-hot call paths | Optional CPU-profile extension after the core CPU case. |
| `perf-labs/src/lab007.c` | Long sleeps plus brief CPU bursts | Optional off-CPU extension; not required for the main outcome. |
| `perf-labs/src/lab008.c` | `O_SYNC` writes with occasional large writes and latency outliers | Alternative disk case; useful for a later challenge if the workshop case is too I/O intensive. |
| `perf-labs/src/lab013.c` | Unbounded recursion until failure | Exclude from the shared playground; high disruption for little connection to the BPF learning goal. |
| `bpf-perf-workshop/src/lab001.c` and `lab1.md` | Frequent `fsync()` plus occasional larger writes; open-ended latency diagnosis | **Core case 2:** locate the synchronous-write cause of latency and distinguish syscall time from block-device time. |
| `bpf-perf-workshop/lab2.md` | Investigate SSH login time with two terminal sessions | Follow-on scenario; depends on SSH configuration and may mix normal authentication latency with injected faults. |
| `bpf-perf-workshop/src/lab003.c` and `lab3.md` | Two threads make zero-byte raw `read` syscalls in a busy loop | **Core case 3:** identify the call site and read size behind high CPU; remove its hard-coded CPU-0 affinity in an authorized adaptation. |
| `bpf-perf-workshop/lab4.md` | Four bpftrace modification/build exercises | Use the **method** for a compact finale: make a scoped read-size or `fsync` latency histogram that answers one of the earlier cases. Keep `rmsnoop`/`procsnoop` as extensions. |
| `bpf-perf-workshop/lab5.md` | Advanced VFS/block I/O, FD leak, and page-cache probes | Separate advanced material; these require more kernel-specific validation and exceed the core lab's time budget. |

The workshop is explicitly a 90-minute class with slides, exercises, discussion, and demos, but its repository contains only the slides and exercises. `perf-labs` has almost no learner-facing text beyond an index, so it supplies workloads rather than a parallel curriculum. The workshop's `src/README.md` explicitly asks learners not to read source code. Keep the playground black-box: deliver executable workloads, not source files or obvious answer-bearing names.

## Learner experience

**Title:** *Diagnose Linux CPU and I/O Mysteries with eBPF*.

**Audience:** Linux users who know `ps`, `top`, processes, files, and basic shell commands; no BPF programming prerequisite. **Outcome:** identify a root cause from measured evidence, explain why a common first-line tool misses it, and build one narrow bpftrace measurement. Suggested pacing: 10-minute setup, three 20–25-minute cases, 15–20-minute finale, with hints for each case.

1. **Orientation and preflight.** The learner sees one affected service at a time, a short symptom statement, a `lab scenario start|status|stop` helper, and a toolbox with `pidstat`, `vmstat`, `iostat`, `ss`, BCC tools, `bpftrace`, and `bpftool`. Demonstrate a harmless BPF trace and explain how to end a probe. Do not reveal workload source.
2. **Case 1: process snapshots miss the culprit.** Run the bounded `cksum` loop from `perf-labs/lab004`. Start with CPU/process views, then use `execsnoop` (and perhaps `pidstat -d`) to connect repeated short-lived child processes with the observed load. Ask for the responsible executable, evidence of repetition, and why `top` alone was inconclusive.
3. **Case 2: latency outliers are synchronous writes.** Run a bounded workshop `lab001` adaptation if its reuse is authorized; otherwise use MIT-licensed `perf-labs/lab008`, which produces synchronous write outliers. Compare `iostat`/`pidstat` with a write or `fsync` syscall latency histogram, as appropriate for the selected program, and a block-I/O view. Ask the learner to identify the synchronous-write pattern and the occasional large write, then state what their measurements do and do not establish about the underlying storage. Avoid promising a fixed latency number on virtual storage.
4. **Case 3: busy CPU but no useful work.** Run a bounded workshop `lab003` adaptation if authorized; otherwise use MIT-licensed `perf-labs/lab003`, which has the same zero-byte read fault. Use process/thread CPU views, CPU stack sampling if supported, and BPF read-size counting to find repeated zero-byte reads. Ask why the syscall returns successfully yet the application spins. A `read()` entry tracepoint is a more portable core measurement than a named kernel-function kprobe.
5. **Finale: make the measurement.** Learner writes a short `bpftrace` script filtering to the selected PID/TID and printing either a histogram of `read` byte counts or `fsync` duration. Verify attachment and plausible output under the controlled workload. Provide an editorial explanation of tracepoint selection, units, filtering, and probe cleanup after the attempt.

Use a **tutorial** as the primary iximiuz format because the material teaches a method step by step and diagnosis itself does not necessarily change machine state. Iximiuz tutorials support playgrounds, regular verification tasks, and `user-input-task` components; use input tasks for short, normalized root-cause answers and a regular task for the final script. Ask learners to record evidence in their own words, then compare it with the editorial, since a simple checker cannot grade investigative reasoning reliably. A companion challenge can later reuse a single unseen fault as an independent assessment. A state-only challenge checker would otherwise pass or fail based on whether the broken workload happens to be running, rather than whether the learner understood it.

## Playground and content design

- Build one custom playground from `flexbox` with one Ubuntu-based VM and a versioned OCI rootfs. iximiuz supplies the VM kernel; the image supplies userland. Bake tool packages, compiled workloads, a small scenario supervisor, and reference material into the rootfs so startup does not depend on package downloads. Use one terminal tab, with an optional IDE tab for the final bpftrace script.
- Start with a supported 6.1 guest kernel and 2 vCPUs / about 2–4 GiB RAM, then size from a real cold-start test. The platform's available kernel choices and account resource caps can change; check them before committing the manifest. Select one tested kernel rather than claiming support for every option.
- On every start, preflight `bpftool feature probe`, BTF presence, tracefs, `perf_event_open` access, required BCC/bpftrace tools, and a sample attach. If any essential check fails, show a clear setup failure instead of a misleading diagnosis task. BCC package names and compile dependencies must be pinned and tested against that guest kernel.
- Put workloads behind `systemd`-supervised or equivalent time-limited scenario commands. Run only one case at a time. Give each case CPU, memory, runtime, and file-size limits; clean up its processes and scratch data on stop and at playground shutdown. Keep I/O on a dedicated scratch path and use a modest write rate so the lab does not saturate shared storage.
- Namespace binaries by scenario (`short-processes`, `sync-writes`, `zero-read-spin`), avoiding the two repositories' colliding `lab001` and `lab003` names. If labels would spoil the exercise, use neutral learner-visible scenario IDs and keep descriptive names only in author-side build files.
- Deliver compiled binaries without the corresponding source and avoid publishing answer keys in the playground's public `__static__` assets. The teaching text can link to upstream projects **after** the relevant diagnosis, with proper credit. Hints should reveal the next measurement, not the root cause.
- Verify diagnosis via a small number of user-input tasks for canonical process/operation answers, with strict input validation. Leave nuanced evidence and interpretation for a guided self-check against the editorial. For the final script, test that it parses/attaches and reports expected events for a fresh bounded run; avoid grading a hard-coded number or exact command sequence.

## Rights and adaptation gate

`perf-labs` includes an MIT `LICENSE`; preserve its copyright and license notice in copies or substantial adaptations. The local `bpf-perf-workshop` tree has **no license file**, while its README and exercise pages say “Copyright Brendan Gregg, 2019. All rights reserved.” Before redistributing its C files, slide PDF, or exercise prose in an image or published tutorial, obtain permission or use a clearly independently written workload and original instructional text that teach the same observable behavior. Do not assume that having a local fork grants republication rights. This is a release gate, not a reason to stop the design: the MVP can use MIT-licensed `perf-labs` programs and new author-written material while permission is resolved.

## Required publication gates

| Gate | Pass evidence | If it fails |
| --- | --- | --- |
| **1. iximiuz guest/BPF compatibility** | On a fresh instance using the final kernel, rootfs, and resource tier: `bpftool` preflight succeeds; BCC `execsnoop` records a known short-lived process; bpftrace attaches to the chosen `read` and write/`fsync` tracepoints and records known events; each core scenario produces the expected observations; probes and workloads stop cleanly. Record kernel/tool versions, exact commands, and results in a compatibility report. | Try another supported kernel or pinned tool build and repeat the full gate. If none passes, remove BPF from the published scope and release a separately described ordinary-Linux-tools tutorial; do not publish the proposed combined eBPF lab. |
| **2. Source and publication rights** | A provenance ledger lists every copied or adapted program, text, slide, and image in the rootfs, tutorial, and public assets. MIT-licensed `perf-labs` material carries its required notice. Workshop C files, slide PDF, and exercise prose are included only with documented permission covering this publication, or are replaced by independently authored equivalents. A final package review confirms the ledger matches what is shipped. | Keep the material private, obtain permission, or replace the affected assets and re-review the package. Do not publish the affected assets while their status is unresolved. |

Both rows must be marked **pass** before switching the tutorial or playground to public access. A successful preflight on another machine or an assumption based on an existing iximiuz eBPF tutorial does not close gate 1; possession of a local workshop fork does not close gate 2.

## Build sequence and acceptance gates

1. **Capability spike:** start a disposable iximiuz VM with the intended kernel and test BCC `execsnoop` and an `fsync`/`read` tracepoint bpftrace script against harmless commands. Record kernel, package versions, privileges, attach results, and memory use. An iximiuz eBPF tutorial shows the platform can run BPF experiments, but the exact BCC/bpftrace combination still needs this test.
2. **Adapt and package:** choose licensed/authorized workload sources; bound loops, I/O, and CPU affinity; add the scenario supervisor and cleanup; build a versioned rootfs from an official iximiuz rootfs; store a manifest and build instructions in Git. Keep a provenance file mapping each derivative to its origin and permission status.
3. **Author the tutorial:** write symptom-first sections, safe command examples, progressive hints, answer checks, and an editorial solution. Explain the distinction between CPU usage, off-CPU waiting, syscall latency, and block latency where each arises.
4. **Test from a fresh playground:** for every case, confirm it starts and stops, expected traces appear, the intended diagnosis is possible without source access, all wrong-answer checks stay red, the correct checks turn green, the final script attaches, and no runaway process or data file survives cleanup. Run at least one complete learner pass on the account's actual resource tier.
5. **Publish after both required gates pass:** keep the playground and tutorial private during validation. Review the compatibility report and provenance ledger against the final build, mark both gates above pass, and only then enable public access. Follow the specified fallback if either gate fails.

## When to split later

Split into two linked tutorials only if a pilot exceeds roughly two hours, the bpftrace coding step requires substantial prerequisites, or the SSH/advanced tool-writing exercises are required for coverage. The first tutorial would cover diagnosis with existing tools; the second would teach probe construction. Both would use one maintained rootfs/playground and a skill path or course to connect them. There is no need to split by original repository.

## Platform references

- [iximiuz Labs sample tutorial](https://labs.iximiuz.com/tutorials/sample-tutorial): front matter, attached playgrounds, regular checks, and user-input tasks.
- [iximiuz Labs Challenges](https://labs.iximiuz.com/courses/labs-docs/learning-materials/challenges): short problem format and final-state verification.
- [Custom playground building blocks](https://labs.iximiuz.com/docs/custom-playgrounds/building-blocks) and [manifest reference](https://labs.iximiuz.com/courses/labs-docs/custom-playgrounds/manifest-reference): `flexbox`, VM topology, resources, OCI drives, and init tasks.
- [Custom rootfs images](https://labs.iximiuz.com/docs/custom-playgrounds/custom-rootfs) and [playground kernel selection](https://labs.iximiuz.com/docs/playgrounds/how-to-use-playgrounds): image/kernel boundary and supported choices.
- [iximiuz eBPF tutorial](https://labs.iximiuz.com/tutorials/inspecting-ebpf-using-bpftool-43dfa319): evidence of BPF experimentation on the platform, though not proof that every planned tool works.

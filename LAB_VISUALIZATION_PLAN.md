# Live Linux Lab Visualization — Implementation Plan

## 1. Objective

Build a general framework that renders a running Linux environment as a navigable 3D
world and broadcasts it, live, to a public URL where any number of read-only spectators
can watch one or more **players** move through the filesystem and run commands. Each
spectator controls their own camera: they can survey the whole system, follow a
particular player, zoom down to a single file's metadata, or click a player to open a
live view of their terminal.

The framework is designed to run against a **disposable playground of ephemeral VMs** —
short-lived environments provisioned for an event and torn down afterward. That
disposability is the entire safety model: the playground is assumed to contain nothing
sensitive, so the visualization can show everything it observes without filtering. This
first-pass plan makes **no privacy or redaction guarantees** and does not try to; it is a
visualization framework, not a data-protection product.

Two initial use cases drive the design:

1. **Capture-the-flag (CTF) spectating** — many competitors act inside an intentionally
   exposed lab, and an audience watches them race, explore, and exploit.
2. **Expert troubleshooting theatre** — a skilled operator diagnoses a staged fault in a
   live Linux environment while an audience follows the reasoning through the actions
   they take.

The visual north star is the file-system navigator ("fsn") from the 1993 film *Jurassic
Park* — "It's a Unix system! I know this!" — and the modern homage in
`jlandersen/k8s-unix-system`, which flies a viewer through a Kubernetes cluster rendered
as 3D platforms and blocks. This plan generalizes that idea from a static resource
inventory to a **live, multi-actor, filesystem-and-process** visualization.

The deliverable is a sensing layer that observes real behavior, a normalized event model,
a scene/state service, a fan-out tier that serves many spectators cheaply, and a 3D
client — including a click-a-player live terminal view.

## 2. Concept and Visual North Star

A spectator opening the public URL sees the lab as a three-dimensional space:

- **The filesystem is terrain.** Directories are platforms, rooms, or nodes in a 3D
  tree; files are objects resting in them, sized by byte count and colored by type and
  permission. The layout is stable over time so viewers build spatial memory.
- **Players are avatars.** Each player stands "in" the directory that is their current
  working directory. When they `cd`, the avatar travels along the edge to the new
  directory. Several players occupy the world at once, each with a distinct color and
  label.
- **Commands are events in space.** Running a command spawns a visible process near the
  avatar with its name and arguments; reading or writing a file draws a beam or pulse
  between the avatar and the file object; a network connection sends an arc leaving the
  system boundary.
- **Zoom reveals granularity.** From orbit, a viewer sees the shape of the whole system
  and where the crowds are. Descending, they read directory and file names, then
  individual file metadata (size, mode, owner, mtime, inode, extended attributes), and
  a live terminal panel for a chosen player.

The aesthetic goal is "maximally informative and visually appealing": a coherent,
legible, slightly retro-futuristic world where activity is obvious at a glance and depth
is available on demand, not a wall of scrolling text.

## 3. Scope, Constraints, and Non-Goals

### 3.1 Goals

- Observe real player behavior in a Linux environment: process execution, movement
  through the filesystem, file access, and network activity.
- Attribute every observed action to the correct player and the correct filesystem
  location, robustly, across multiple concurrent players.
- Maintain an authoritative, queryable model of the visualized filesystem subtree and
  the live position and recent activity of every player.
- Broadcast that model to an unbounded number of read-only spectators over a public URL,
  with each spectator controlling their own camera and detail level.
- Render the world in the browser in 3D with smooth pan/orbit/zoom and drill-down to
  granular per-file and per-process detail.
- Let a spectator click a player and open a live view of that player's terminal.
- Run against ephemeral, disposable VMs so the whole environment can be created for an
  event and destroyed afterward.
- Support a recorded/replay mode so a session can be re-watched and seeked after the
  fact.

### 3.2 Working Constraints

- The **target environment is a disposable playground of ephemeral VMs**, provisioned for
  the event and destroyed afterward, holding nothing of value. This is not for real
  production and not for any system holding real credentials or private data.
  "Troubleshooting" means a staged or representative fault environment, not a production
  incident bridge. The disposability of the environment is the safety boundary (Section
  14), which is why the framework needs no redaction of its own.
- Spectators are strictly **read-only**. No path exists from a viewer's browser to the
  lab, to a player's session, or to the sensor's control surface.
- The sensor must observe **without being installed inside, or modifying, the player's
  shell or workload** where feasible, and must not require restarting the player's
  session to begin observing.
- The system should run comfortably on a small fixed budget of hosts for a
  club/classroom/conference-scale event (tens of players, thousands of spectators), with
  a clear but non-mandatory path to larger scale.

### 3.3 Non-Goals for the First Release

- **Any privacy, redaction, or secret-hiding guarantee.** The environment is disposable
  and assumed to contain nothing sensitive; that assumption *is* the safety model, not
  content filtering. Whatever a player does in the playground is public.
- Reconstructing a byte-perfect copy of the entire root filesystem in 3D. The world is a
  **scoped, lazily expanded** view of relevant subtrees, not a render of every inode.
- Guaranteeing that every action is captured. Some actions are structurally invisible to
  a given sensor (see Section 5); the system reports coverage honestly rather than
  implying completeness.
- Preventing a determined player from evading observation, or preventing a spectator from
  screen-recording the broadcast.
- Letting spectators interact with, vote on, or influence the lab.
- Full production-grade multi-tenant SaaS: accounts, billing, org management, arbitrary
  user-uploaded environments. A single event is provisioned by an operator.
- Native mobile/desktop apps; the client is a web application.
- Serving as a security-monitoring or intrusion-detection product. It is a visualization,
  and its data is presentation-shaped, not forensic evidence.

## 4. Success Measures and Release Gates

These are acceptance targets to be tested, not claims to make in advance.

| Area | Target |
| --- | --- |
| Attribution | In a scripted multi-player run, ≥95% of executed commands and directory changes are attributed to the correct player and correct path, with misattribution rate near zero. |
| Movement fidelity | A player's `cd` sequence reconstructs the correct current-directory path on the avatar within the agreed latency budget. |
| Spectator scale | The fan-out tier serves the target spectator count from a snapshot plus live deltas, with a late joiner reaching a correct scene within seconds. |
| Terminal view | Clicking a player opens their live terminal within the agreed latency, with a scrollback snapshot so a late joiner sees current screen state. |
| Rendering | The client sustains an interactive frame rate at the target scene size on a mid-range laptop, degrading legibly (level-of-detail) rather than stalling under event storms. |
| Operational safety | Spectators have no path into the lab, and an operator can cut the public broadcast within the agreed reaction time. |
| Replay | A recorded session replays deterministically with seek, matching what was broadcast live. |

## 5. Core Challenges and Honest Boundaries

This section keeps the plan grounded. The idea is appealing; several parts are genuinely
hard, and a few are impossible in the naive form. Design around the real boundaries
rather than promising past them.

### 5.1 "Where is the user?" is reconstructible, with care

A process's current working directory is real kernel state (per-process, inherited across
`fork`, changed by the `chdir`/`fchdir` syscalls). The shell's `cd` builtin performs a
`chdir` syscall, so **directory movement is observable** even though `cd` is not a
separate program. To place an avatar correctly the system must track cwd per process
identity, follow `fork`/`clone` inheritance, and apply `chdir`/`fchdir` — not guess from
command text. Tetragon and equivalent tooling already expose process cwd and can be used
directly.

### 5.2 "What did the user run?" is mostly observable — but not entirely

`execve`/`execveat` carry the program path and argument vector, so most commands are
directly observable with their arguments. The gaps are real and must be stated:

- **Shell builtins that are not syscalls** (`echo`, `export`, variable assignment,
  `for`/`while` loops, function definitions) never `execve` and are invisible to a
  syscall sensor. `cd` is the important exception because it does `chdir`.
- **Pipelines and subshells** produce many short-lived processes; the visible structure
  is the process tree, not the human's one-line intent.
- Reconstructing the literal command line the human typed requires **terminal capture**,
  not syscalls. The two sources answer different questions and are both needed for a rich
  view; terminal capture also directly powers the live terminal view (Section 12.4).

### 5.3 The environment is the safety boundary, not filtering

A live view of commands, file activity, and terminals will surface whatever a player
does — arguments, file contents shown on screen, typed input, internal hostnames. This
plan deliberately does **not** try to redact any of that. Instead, the safety model is
that the playground is a disposable environment of ephemeral VMs holding nothing of
value, so there is nothing to protect. This is a real design decision with a real
consequence: **never point this framework at a system that contains anything sensitive.**
Section 14 states this boundary and the few controls that enforce environment
disposability and spectator read-only-ness.

### 5.4 Real filesystems and real workloads overwhelm naive rendering

`/` on a normal host has hundreds of thousands to millions of inodes; a `find /`, a
package install, or a compile emits file events far faster than a scene can meaningfully
animate or a human can read. Two consequences:

- The world is a **scoped subtree** (e.g., player home directories, the CTF working area,
  and named points of interest), expanded lazily as viewers zoom, with hard caps on node
  count and level-of-detail (LOD) collapsing of dense directories.
- The event pipeline must **aggregate, sample, and rate-limit** high-frequency file
  activity into digestible visual pulses ("this player is reading heavily under
  `/var/log`") rather than one animation per `read`.

### 5.5 Privilege, clocks, and multi-host reality

- eBPF sensing requires elevated capability (`CAP_BPF`/`CAP_PERFMON`, sometimes
  `CAP_SYS_ADMIN`) and a compatible kernel with BTF. The sensor is a privileged component
  and must be isolated from both players and spectators — a system-integrity concern, not
  a privacy one.
- With multiple lab hosts, event timestamps need a common time base; record clock offset
  and never order cross-host events by unsynchronized wall clocks alone.
- Identity must combine host, boot, cgroup/namespace/container, PID, and process start
  time. **PID alone is never an identity** — it is reused.

## 6. System Architecture

Five layers, with a clean seam between producing the scene and fanning it out to viewers.
That seam is what makes "any number of viewers" affordable: spectators never touch the
lab or the sensor.

```text
  LAB (ephemeral VMs)                 PLATFORM (operator-run)              PUBLIC
  ┌───────────────────┐   events   ┌──────────────────────────┐  deltas  ┌──────────┐
  │ player sessions   │ ─────────▶ │ Ingest + Normalizer       │          │ spectator│
  │  (shells, procs)  │            │  attribution, dedup        │          │ browsers │
  │                   │            └─────────────┬────────────┘          │ (3D, RO) │
  │ Sensor(s):        │            ┌─────────────▼────────────┐  snapshot│          │
  │  eBPF/Tetragon    │            │ Scene/State Service        │ +delta   │  camera  │
  │  PTY capture      │            │  fs model + player state   │ ───────▶ │  is      │
  │  (privileged)     │            └─────────────┬────────────┘          │  local   │
  └───────────────────┘                          │ published stream       └────▲─────┘
                                    ┌─────────────▼────────────┐                │
                                    │ Fan-out tier (pub/sub →   │────────────────┘
                                    │  WS/SSE/edge, read-only)  │
                                    └───────────────────────────┘
                                    ┌───────────────────────────┐
                                    │ Recorder (durable log) →   │  replay/seek
                                    │  replay source            │
                                    └───────────────────────────┘
```

| Layer | Responsibility | Initial choice |
| --- | --- | --- |
| Sensor | Observe process, cwd, file, and network events; capture PTY streams | eBPF via Tetragon (structural) + a PTY recorder sidecar (literal text) |
| Ingest / Normalizer | Validate, attribute to player + path, deduplicate, aggregate high-rate events into a stable schema | Small stateless service (Go/Rust) reading the sensor's stream |
| Scene / State Service | Hold the authoritative filesystem subtree model and live player state; emit snapshot + deltas and per-player terminal sub-streams | Single stateful service, in-memory model with periodic snapshot |
| Fan-out tier | Replicate the delta stream to many read-only viewers; serve snapshots to late joiners | Pub/sub topic feeding horizontally scaled WS/SSE servers, or edge/CDN push |
| Client | Subscribe, reconstruct the scene, render 3D, own the camera and drill-down | React-Three-Fiber (Three.js) web app |
| Recorder | Append the published stream durably for replay and seek | Object storage of snapshot + delta segments |

Design rules: exactly one home for authoritative state (the Scene/State Service);
spectators only ever read from the fan-out tier and never from the lab; the camera and
all view state live in each spectator's browser, so adding viewers costs bandwidth, not
scene computation.

## 7. Sensor Layer and Data Acquisition

### 7.1 Recommended primary sensor: eBPF via Tetragon

eBPF observes the running kernel without changing the workload and without a restart,
which matches the constraints directly. Tetragon is the recommended starting point
because it already provides process lifecycle (exec/exit with binary, arguments, and
cwd), file access, and network events with container/pod attribution and a stable export
stream — most of the structural signal the world needs, without writing custom probes on
day one.

The events the world consumes:

| Signal | Source event | Visual meaning |
| --- | --- | --- |
| Command run | `execve`/`execveat` (process exec) | Spawn a process object near the player with name + args |
| Process ended | process exit | Remove the process object; keep a short fading trail |
| Movement | `chdir`/`fchdir` (per-process cwd change) | Move the avatar to the new directory |
| Process tree | `fork`/`clone` + exec parentage | Group child processes under the player; show pipelines |
| File access | `openat`, and aggregated read/write | Pulse/beam between avatar and file; heat on hot paths |
| Filesystem change | create/unlink/rename/chmod | Add, remove, or restyle a file object in place |
| Network | connect/accept (5-tuple, no payload) | Arc leaving the system boundary; label with dest |

### 7.2 Assessed alternatives and complements

- **Raw libbpf/CO-RE probes.** Reserve for signals Tetragon does not expose well or where
  overhead must be tuned. More control, more maintenance; adopt only on evidence.
- **Pixie.** Strong for Kubernetes application-protocol tracing (HTTP, DB queries). For a
  general Linux CTF or a single-host troubleshooting lab it is largely **not applicable**;
  record that decision rather than forcing it in. Revisit only if the target is genuinely
  a k8s/service environment where L7 flows are the story.
- **auditd / Linux audit.** A fallback structural sensor when eBPF is unavailable
  (locked-down or old kernel). Noisier and heavier; keep as a compatibility path.
- **PTY / terminal capture.** The complement to syscalls: it captures the literal command
  line and terminal output, filling the builtin/pipeline gaps in Section 5.2 and driving
  the click-a-player **live terminal view** (Section 12.4). Capture via a recording layer
  around the player's PTY (a `script`/`ttyrec`-style sidecar, or a server-side terminal
  gateway the player logs in through). Capture, transport, and rendering are all
  straightforward; because the environment is disposable and non-sensitive (Section 14),
  the stream is shown as-is with no redaction.

### 7.3 Attribution model (who and where)

Every event is resolved to a **player** and, where relevant, a **path**:

- **Player** = a stable identity assembled from: login/audit session id (or
  systemd-logind session), uid, the container/cgroup/namespace for the session, the host
  and boot id, and the PTY. In a CTF each competitor is typically a distinct
  user/container, which makes attribution clean. Never attribute by PID or process name
  alone.
- **Path** = resolved from the syscall's directory file descriptor and path argument to
  an absolute path within the visualized subtree, tracking per-process cwd for relative
  paths. Events whose path falls outside the visualized subtree are counted but not
  rendered as objects.

The sensor emits (host, boot, cgroup/container, pid, process-start-time, session) with
every event so the Normalizer can maintain a stable identity even across PID reuse and
session churn.

## 8. Event Model and Normalized Schema

The Normalizer converts sensor-specific events into one stable, presentation-oriented
schema. Downstream layers never see raw sensor formats.

```json
{
  "schema_version": 1,
  "event_id": "evt-...",
  "observed_at": "2026-09-09T12:00:00.123456Z",
  "monotonic_ns": 123456789,
  "host_id": "lab-vm-1",
  "boot_id": "...",
  "player": {
    "player_id": "p-teal",
    "session_id": "...",
    "uid": 1001,
    "container_id": "..."
  },
  "process": {
    "pid": 4321,
    "start_time": 987654321,
    "ppid": 4300,
    "comm": "cat"
  },
  "kind": "exec | chdir | file_open | file_change | proc_exit | net_connect | tty_text",
  "path": "/home/p-teal/notes.txt",
  "detail": {
    "argv": ["cat", "notes.txt"],
    "errno": null,
    "bytes_bucket": null,
    "dest": null
  },
  "coverage": {
    "source": "tetragon | ebpf | audit | pty",
    "sampled": false
  }
}
```

Principles:

- The schema carries **coverage honesty**: which sensor produced the event and whether it
  was sampled/aggregated — so the client can show a viewer when the picture is partial.
- High-rate file activity is emitted as **aggregates** (a rolling count/heat under a path)
  rather than one event per syscall, with the raw rate available as a number, not a
  thousand objects.
- `tty_text` events are separate and carry terminal bytes for the live terminal view; they
  are gated only by whether a spectator has that panel open, not by any content filter.

## 9. Filesystem and World Model

### 9.1 Scene graph

The Scene/State Service holds an authoritative model of the visualized subtree:

- **Directory nodes**: path, parent, child count, aggregate size, and a stable layout
  seed so a directory always renders in the same place.
- **File nodes**: name, size, mode/owner/group, mtime, type classification, and inode —
  loaded lazily and capped per directory (dense directories collapse to a "bin" the
  viewer can open on demand).
- **Player state**: current directory, recent process objects, and recent activity trail.
- **Process objects**: transient, parented to a player, with a short lifetime after exit.

### 9.2 Scoping, lazy expansion, and level-of-detail

- The operator configures the **visualized roots** (e.g., `/home/*`, the CTF work
  directory, `/etc`, named points of interest). Everything else is out of scope and only
  summarized.
- Directories expand their children when a viewer zooms in or a player enters, and
  collapse when attention leaves, keeping the live node count bounded.
- Hard caps: maximum rendered nodes, maximum files shown per directory before collapsing,
  and a maximum activity-object count, all enforced server-side in the snapshot and
  client-side in the renderer.

### 9.3 Snapshot and delta

- A **snapshot** fully describes the current world (scoped) so a late-joining spectator
  can render correctly without replaying history.
- **Deltas** are compact mutations: player moved, process spawned/exited, file
  added/changed, activity pulse. Snapshots are re-emitted periodically and on major
  change so the fan-out tier and late joiners stay cheap.

## 10. Player and Session Model

- A **player** is the visible actor. In CTF mode, players map to competitors/teams; in
  troubleshooting mode there may be a single "operator" player plus incidental system
  activity that can be shown as ambient or hidden.
- Players get a stable color, label, and avatar for the session. Labels are operator-set
  display names. Participants are aware, as a normal condition of the event, that the
  playground is observed and publicly broadcast.
- System/background activity not tied to a human session is either suppressed or rendered
  as low-key ambient motion so it does not drown the human players.
- A spectator may "follow" a player (camera tracks the avatar) or free-fly, and may open
  that player's live terminal. Following and panel state are purely client-side over the
  shared world.

## 11. Streaming and Fan-out

The fan-out tier is what turns "one live system" into "any number of viewers" affordably.

- The Scene/State Service publishes one stream (snapshot + deltas) to a pub/sub topic. It
  does not know or care how many viewers exist.
- Horizontally scaled **WS/SSE servers** (or edge workers / a CDN push path) subscribe to
  that topic and replicate to connected browsers. Viewer count scales by adding fan-out
  replicas, not by loading the scene service.
- **Late joiners** are served the most recent snapshot (cacheable at the edge) and then
  attach to the live delta stream.
- **Per-player terminal sub-streams** ride the same tier: a spectator subscribes to a
  player's terminal only while its panel is open, so terminal bandwidth scales with panels
  opened, not with total viewers.
- Spectator connections are **read-only and unauthenticated for viewing** (public URL),
  but carry no capability to send anything back into the pipeline. The transport is
  one-directional in effect: client → server messages are limited to subscription and
  heartbeat, never anything that reaches the lab.

Transport choice: start with WebSocket for deltas plus HTTP-cached snapshots; evaluate
SSE and WebTransport as scale evidence warrants. Avoid per-viewer server-side scene
computation entirely.

## 12. The 3D Client

### 12.1 Technology

A web client built on **React-Three-Fiber** (React bindings for Three.js/WebGL) for the
declarative scene graph, with a plain state store fed by the delta stream. Rationale:
mature ecosystem, good instancing/LOD support, large talent pool, and it runs anywhere a
browser does. The renderer subscribes to the store; the network layer never touches
Three.js objects directly.

### 12.2 Camera and zoom levels

Each viewer independently controls an orbit/pan/zoom camera with several natural detail
tiers:

1. **Orbit** — the whole scoped system; see structure and where activity is clustered.
2. **District** — a region/subtree; read directory names and player positions.
3. **Directory** — inside one directory; individual files legible, activity beams visible.
4. **Inspect** — a selected file or process; a detail panel with full metadata (size,
   mode, owner, mtime, inode, xattrs). Selecting a *player* instead opens the live
   terminal view (Section 12.4).

Interaction: click to select and inspect, double-click/scroll to descend, "follow player"
to attach the camera, and a time-scrubber in replay mode.

### 12.3 Performance techniques

- **Instanced meshes** for large numbers of similar file/process objects.
- **LOD** swaps: distant directories render as simple blocks/labels; detail meshes and
  text load only when close.
- **Frustum culling and lazy subtree loading** so only what is near the camera is
  detailed.
- **Aggregated activity**: heat and pulses represent high-rate file access instead of
  per-event animations, matching the aggregation done upstream.
- Graceful degradation: under a storm or on a weak device, drop to lower LOD and coarser
  animation rather than stalling; show a "high activity — simplified view" indicator.

### 12.4 Live Terminal View (click a player)

Selecting a player opens a **live terminal panel** — the running contents of that player's
session rendered as a real terminal, alongside the 3D world. It is the most direct way to
follow a CTF competitor's exploit or an expert's troubleshooting reasoning in their own
words, and it reuses the existing pipeline end to end:

- **Capture** reuses the PTY recorder from Section 7.2. The reliable form routes each
  player through a recording layer at session entry — a `script`/`ttyrec`-style wrapper or
  a server-side terminal gateway (ttyd/gotty/wetty-style) the player logs in through — so
  the terminal byte stream is captured cleanly at the source. Attaching to an
  already-running arbitrary PTY is possible but messier, and is a fallback rather than the
  baseline.
- **Transport** is a per-player terminal sub-stream on the fan-out tier (Section 11):
  spectators subscribe on demand when they open a panel. A late joiner receives a
  **scrollback/screen snapshot** (the last N lines or the current screen state) and then
  attaches to the live bytes — the same snapshot-plus-delta pattern the world uses.
- **Rendering** is an off-the-shelf browser terminal emulator (xterm.js), which handles
  ANSI, colors, cursor movement, and TUI redraws natively.

The stream is shown as-is: because the playground is disposable and holds nothing
sensitive (Section 14), no redaction is applied to terminal contents in this framework.

## 13. Design Language and Visual Encoding

A consistent encoding is what makes the world "maximally informative" rather than pretty
noise. All choices below are the initial system; they will be validated for legibility,
color-blind safety, and readability in motion.

| Element | Encoding |
| --- | --- |
| Directory | Platform/room; depth in tree → elevation or nesting; child count → footprint |
| File | Object on its directory; size → volume (log-scaled); type → shape/material; permissions → accent (e.g., executable, world-writable flagged) |
| Player avatar | Distinct per-player color and label; position = current directory |
| Movement (`cd`) | Smooth travel along the tree edge between directories |
| Command | Process object near the avatar with name + args; pipelines chained as children |
| File read/write | Directed beam or pulse avatar↔file; repeated access → growing heat glow on the path |
| Filesystem change | Object appears (create), dissolves (unlink), or restyles (chmod/rename) in place |
| Network | Arc leaving the system boundary toward a labeled destination zone |
| Coverage gaps | Subtle "partial view" cue where events were sampled or a sensor is blind |

The palette and motion are tuned for legibility first: high contrast, no meaning carried
by color alone, motion that draws the eye to *new* activity, and a light/dark theme.
Keep the retro-fsn character (grid horizon, glow, flythrough feel) as styling over a
strictly legible substrate — never let the aesthetic obscure the information.

## 14. Safety and Operating Model

There is no redaction subsystem in this framework, and by design there does not need to
be one. The safety model is the shape of the environment, not filtering of its contents.

### 14.1 The disposable playground is the safety boundary

- The lab is a **playground of ephemeral VMs**, provisioned for a specific event and
  destroyed afterward. It holds no real credentials, customer data, or anything of value.
- Because nothing in the playground is sensitive, the visualization can show everything it
  observes — arguments, file activity, terminals — without any privacy or redaction
  guarantee. Whatever a player does in the playground is public, and that is acceptable
  precisely because the playground is disposable.
- **The single hard rule: never point this framework at a system that contains anything
  sensitive.** The framework offers no protection if that rule is broken; it is enforced
  by how the environment is provisioned, not by the software.

### 14.2 System-integrity controls (not privacy controls)

These exist to keep the system itself sound, independent of content:

- Spectators are **strictly read-only**; no viewer→lab, viewer→sensor, or viewer→player
  path exists, and viewer→server messages are limited to subscription and heartbeat.
- The **sensor is privileged** (eBPF capabilities) and is isolated from both players and
  the public; its control surface is unreachable from a player session or a spectator.
- Standard web hardening for the public client and APIs: TLS, strict CSP, and rate
  limiting on connections; no capability tokens exposed to the browser.
- An operator **kill switch** stops the public broadcast immediately as a basic
  operational control (e.g., to end a session or handle a malfunction) — not as a
  content-safety mechanism.

### 14.3 Participant awareness

Participants know, as a normal condition of the event, that the playground is observed and
publicly broadcast and that they should treat everything in it as public. This is an
expectation-setting note, not a consent-gated privacy regime.

### 14.4 Recorder retention

The recorder stores the already-public broadcast stream (world deltas and, where captured,
terminal streams) for replay. Apply a simple retention/deletion policy for those
recordings; there is nothing sensitive to protect beyond ordinary housekeeping.

## 15. Performance and Scalability

- **Event rate** is the first bottleneck. Aggregate and sample high-frequency file
  activity at the sensor/normalizer; cap per-player activity objects; never emit one
  delta per `read`. Measure events/sec under a `find /`, a build, and a package install.
- **Scene size** is bounded by scoping, lazy expansion, node caps, and LOD; the renderable
  world is far smaller than the real filesystem.
- **Viewer count** scales at the fan-out tier: snapshot is edge-cacheable, deltas are one
  published stream replicated by horizontally scaled subscribers. Adding viewers adds
  bandwidth and fan-out replicas, not scene-service load. Terminal sub-streams add cost
  only for opened panels.
- **Client** targets an interactive frame rate at the agreed scene size on a mid-range
  laptop, with explicit degradation paths under storms and on weak devices.
- Define concrete budgets after a vertical-slice measurement: events/sec sustained,
  delta bytes/sec/viewer, snapshot size, max rendered nodes, and target frame rate.

## 16. Deployment Modes

| Mode | Players | Emphasis | Notes |
| --- | --- | --- | --- |
| CTF spectating | Many (per-team/user, isolated) | Crowd dynamics, races, discovery; leaderboard-style overview | Per-player color/label; live terminals on |
| Troubleshooting theatre | One primary operator | Following one expert's reasoning; command + file detail legible; live terminal | Staged/disposable environment; live terminal on |
| Replay / education | Recorded | Re-watch, seek, annotate a past session | Serve recorder segments; scrubber |

All modes share the same pipeline; a mode is a configuration of scope, players, and which
panels are enabled by default — not a separate codebase.

## 17. Observability and Operations

- **Pipeline health**: sensor event rate and drops, normalizer lag, scene-service delta
  rate and snapshot size, fan-out connection count and per-viewer bandwidth, open terminal
  panel count, and end-to-end lab-to-viewer latency.
- **Coverage metrics**: fraction of events sampled/aggregated, sensor blind-spot
  indicators, and attribution-failure counts — surfaced both to operators and, as a "view
  is partial" cue, to viewers.
- **Alerts** route to a named operator with a documented action: event storm (raise
  aggregation), fan-out saturation (add replicas), sensor down (broadcast a clear "signal
  lost" state rather than a frozen world).
- Operators get a runbook: pre-event checks (kernel/BTF/capabilities, scope config,
  kill-switch test, playground provisioning and teardown), during-event controls, and
  post-event teardown and recording cleanup.

## 18. Test Strategy

- **Attribution and movement**: scripted multi-player runs with a ground-truth log; assert
  correct player/path attribution and correct cwd reconstruction, including PID reuse,
  container recreation, pipelines, and subshells.
- **Coverage honesty**: assert that builtin-only actions and out-of-scope activity are
  reported as gaps, not fabricated.
- **Terminal view**: assert a clicked player's terminal streams live, a late joiner gets a
  correct scrollback/screen snapshot, and panels subscribe/unsubscribe cleanly.
- **Fan-out and late-join**: many simulated viewers; assert late joiners converge to a
  correct scene from snapshot + deltas, and that viewer count does not load the scene
  service.
- **Operational safety**: assert no viewer→lab path, sensor control-surface isolation, and
  that the kill switch cuts the broadcast within the agreed reaction time.
- **Rendering**: scene-size and event-storm load tests on target devices; assert graceful
  LOD degradation and sustained interactivity.
- **Replay**: assert deterministic replay and seek match the live broadcast.
- **Security**: standard web hardening (TLS, CSP, rate limits, no capability leakage).

Automated cases use recorded/synthetic sensor streams for repeatability; final validation
uses a real lab with real eBPF sensing.

## 19. Delivery Phases and Exit Gates

Estimates assume a small experienced team and a disposable lab to test against. Each phase
ends at a gate that must pass before the next begins.

### Phase 0 — Discovery and Spikes

- Confirm the target environment (single host vs. containers vs. k8s), kernel/BTF and
  capability availability, and the visualized-scope definition.
- Spike Tetragon (and a PTY recorder) on the real lab; confirm exec, cwd/`chdir`, file,
  and network events attribute correctly to a player and path.
- Spike the live terminal view: route a player through a PTY-recording gateway and stream
  to an xterm.js panel.
- Spike a trivial R3F scene rendering a static filesystem subtree with pan/zoom.
- Close the decisions in Section 21 or assign owners.

**Exit gate:** the sensor produces correctly attributed exec/chdir/file events on the real
lab; a static 3D subtree renders and navigates in the browser; and a player's terminal
streams to an xterm.js panel.

### Phase 1 — Vertical Slice: One Player, Live

- End-to-end pipeline for a **single player**: sensor → normalizer → scene service →
  one client. Avatar moves on `cd`; commands spawn process objects with arguments; file
  access pulses; camera zoom tiers work down to file metadata; clicking the player opens
  their live terminal.

**Exit gate:** a spectator watches one real player move through the filesystem and run
commands live, zooms to per-file metadata, and opens the player's live terminal.

### Phase 2 — Multiplayer and Fan-out

- Multiple concurrent players with robust attribution, distinct avatars, and a stable
  world layout.
- Snapshot + delta model, the fan-out tier, per-player terminal sub-streams, and
  late-join; simulated many-viewer load.
- Operator kill switch.

**Exit gate:** several players are visible at once, correctly attributed; the target
spectator count is served with correct late-join, including opening a terminal panel; the
kill switch cuts the broadcast totally within the agreed reaction time.

### Phase 3 — Fidelity, Scale, and Hardening

- Aggregation/sampling for event storms; LOD and instancing for scene scale; graceful
  degradation.
- Coverage-honesty cues, observability, alerts, and the operator runbook.
- Network arcs and process-tree grouping.

**Exit gate:** the system sustains interactive rendering and correct attribution under
storm and scale tests, and the observability/runbook are complete for a real deployment.

### Phase 4 — Replay and Polish

- Recorder and replay/seek; the education/replay mode.
- Visual design pass for legibility, color-blind safety, and the fsn aesthetic; theming;
  accessibility of the non-3D surfaces (controls, panels).

**Exit gate:** a recorded session replays deterministically with seek, and a design/UX
review signs off on legibility and appeal.

### Phase 5 — Optional Enhancements

Prioritize only from demonstrated need: raw-eBPF probes for signals Tetragon lacks; a
Kubernetes/Pixie L7 mode if the target is a service environment; multi-host lab worlds
with clock reconciliation; spectator conveniences (search, bookmarks, per-player stats,
a CTF leaderboard overlay); and narration/annotation tools for the troubleshooting mode.

## 20. Risks and Mitigations

| Risk | Likelihood/impact | Mitigation and trigger |
| --- | --- | --- |
| Framework pointed at a real or sensitive system | Low/high | Hard rule: disposable, non-sensitive playground of ephemeral VMs only; the MVP offers no redaction, so this is enforced by provisioning and operator discipline, and the kill switch handles accidents |
| Event storms overwhelm pipeline/render | High/medium | Aggregate/sample at sensor+normalizer, cap activity objects, LOD/instancing, graceful degradation; measure under `find`/build/install |
| eBPF unavailable or restricted on target kernel | Medium/high | Verify BTF/capabilities in Phase 0; auditd fallback sensor; declare a kernel baseline; mark unsupported environments honestly |
| Rendering a real filesystem is intractable | High/medium | Scoped subtree, lazy expansion, node caps — never attempt to render all of `/` |
| Misattribution across players (PID reuse, containers) | Medium/high | Composite identity (host, boot, cgroup, pid, start-time, session); ground-truth attribution tests |
| Builtins/pipelines make the world view look incomplete | Medium/medium | Coverage-honesty cues, PTY capture and the terminal view to show literal commands, clear "partial view" indicators |
| Fan-out cost grows with viewers | Medium/medium | Decoupled fan-out tier, edge-cacheable snapshots, one published stream; terminal sub-streams cost only per open panel; scale replicas not scene service |
| Aesthetic obscures information | Medium/low | Legibility-first design substrate; validate readability in motion and for color-blind viewers |
| Sensor privilege becomes an attack surface | Low/high | Isolate the privileged sensor from players and public; unreachable control surface; least privilege |

## 21. Decisions to Close During Discovery

| Decision | Why it matters | Needed by |
| --- | --- | --- |
| Target environment: single host, containers, or Kubernetes | Determines sensor choice (Tetragon vs. +Pixie), attribution keys, and topology | Architecture sign-off |
| Visualized scope (which roots/subtrees) | Bounds the world and the render budget | Scene model |
| Kernel/BTF/capability baseline for the lab | Determines whether eBPF is the primary sensor or auditd is needed | Sensor spike |
| Playground provisioning and teardown mechanism for ephemeral VMs | This is the safety boundary; determines how environments are created, isolated, and destroyed | Architecture sign-off |
| Session-entry mechanism for terminal capture (gateway vs. attach) | Determines terminal-view reliability | Terminal-view spike |
| Target spectator scale and latency budget | Sizes the fan-out tier and transport choice | Fan-out design |
| Replay retention and deletion policy | Governs the recorder and storage | Recorder implementation |
| Visual metaphor specifics (tree vs. city vs. graph) and palette | Determines the design language and legibility work | Design pass |

Record decisions in a short decision log; a change to a closed decision must name the
affected scope and tests.

## 22. Definition of Done

The system is ready for a public event when all of the following hold:

- The sensor observes a real lab and attributes exec, `chdir` movement, file access, and
  network events to the correct player and path, with measured accuracy meeting the
  Section 4 targets and coverage gaps reported honestly.
- Multiple players are visible at once in a stable, scoped 3D world; each spectator
  controls their own camera and can zoom from the whole system to a single file's
  metadata.
- A spectator can click a player and open their live terminal, with a scrollback snapshot
  for late joiners.
- The public URL serves the target spectator count from snapshot + deltas via the fan-out
  tier, with correct late-join, and no viewer has any path back into the lab.
- The renderer sustains interactive frame rates at the target scene size and degrades
  legibly under event storms.
- A recorded session replays deterministically with seek.
- The environment is a disposable playground of ephemeral VMs holding nothing sensitive,
  the kill switch works, and an operator can follow the runbook to provision, run, and
  tear down an event.

## 23. Immediate Next Actions

1. Stand up the disposable playground (ephemeral VMs) and run the Phase 0 sensor spike
   (Tetragon + PTY recorder): confirm attributed exec/`chdir`/file events on the real
   kernel.
2. Spike the live terminal view: route a player through a PTY-recording gateway and stream
   to an xterm.js panel.
3. Build the throwaway R3F scene that renders a static filesystem subtree with pan/zoom to
   validate the client foundation and the visual metaphor.
4. Draft the normalized event schema (Section 8).
5. Close the Section 21 decisions — especially target environment, visualized scope, and
   the playground provisioning/teardown mechanism.
6. Define concrete performance budgets from a vertical-slice measurement under a realistic
   workload.
7. Assemble the single-player vertical slice (Phase 1) end to end — movement, commands,
   file activity, zoom-to-metadata, and the live terminal — as the first thing a spectator
   can actually watch.

The default path is eBPF/Tetragon for structural sensing, a PTY recorder plus xterm.js for
the live terminal view, a scoped and lazily expanded 3D world in React-Three-Fiber, and a
decoupled snapshot-plus-delta fan-out tier for unbounded read-only spectators. The safety
model is the disposable playground itself — ephemeral, non-sensitive VMs with no redaction
in the MVP — backed by read-only spectators, an isolated sensor, and an operator kill
switch. Deviate from that path only when a discovery finding gives a clear technical or
cost reason.

# Frontend Performance Visualization — Project Exploration Plan

## 1. Purpose

Plan a browser-based, visually memorable way to understand frontend performance while a
real web application is being used. The project should inherit the best ideas from
`/home/adam/projects/doom-perf`—a normalized telemetry model, live and simulated data
sources, spatial instruments, and drill-down detail—without assuming that another Doom
fork or even a conventional game is the right presentation.

The primary subject is the browser's main thread: how much of it is occupied, when work
starts to queue, which user-visible deadlines are missed, and which changes could split,
batch, prioritize, defer, offload, or eliminate that work. The inside-the-phone viewpoint
is the chosen visual north star; its exact degree of navigation, realism, and the
implementation stack remain open to prototyping.

This is a discovery plan, not yet an implementation backlog. Its purpose is to make the
important alternatives, limitations, and early experiments explicit enough to choose a
direction with confidence.

## 2. Recommended Direction in Brief

Start with a **recorded-first companion visualization**, then evolve it into a hybrid
live/replay system:

1. Add a small, opt-in probe to a monitored page. For a short, explicit capture window,
   it observes public browser performance entries and application-provided marks, keeps a
   bounded in-memory record, and does no synchronous persistence or per-event network
   transmission.
2. At the end of the window, upload the compact recording to a relay/session service and
   replay it in a separate visualization page. This gives the richest early experience
   with the least telemetry contention.
3. Once the impact budget has been demonstrated, emit only coarse, coalesced live
   summaries—provisionally once per second—to a viewer on another device. The detailed
   event stream still arrives after the capture window and replaces or enriches the live
   approximation.
4. Add a Chrome DevTools Protocol (CDP) trace importer/collector as a high-fidelity lab
   adapter and as ground truth for measuring what the in-page probe cannot see. Do not
   make CDP the only source: it is excellent for Chromium development sessions but is not
   a portable field telemetry mechanism.

For the visual concept, use an **inside-the-phone diorama** as the selected north star.
The camera sits within a stylized handset, looking outward through the back of a slightly
distant display toward the person holding and touching the phone. Generic app icons and
interface shapes appear reversed on the far display plane because they are being seen
from behind. Around the viewer, the phone's inner browser machinery pulses, queues,
vibrates, and routes work in response to telemetry. The earlier dispatch-yard idea remains
useful as the visual language of that machinery: a single main-thread conveyor, recurring
frame gates, compositor mechanisms close to the display, and parallel worker chambers.
A conventional trace drawer should remain available beneath the scene so that atmosphere
never replaces diagnosis.

The recommended metric stack is:

- **USE-inspired resource state** for “what is happening now?”
- **RAIL context** for “what was the user trying to do?”
- **Core Web Vitals and interaction phases** for “what did the user experience?”
- **A causal event timeline plus the article's optimization vocabulary** for “why, and
  what kind of remedy fits?”

USE alone is not sufficient, and public page APIs cannot honestly provide exact
main-thread utilization across browsers. The UI must distinguish measured values,
derived estimates, lower bounds, and unsupported signals.

## 3. Product Shape

The project can eventually support four related modes from one data model:

| Mode | Purpose | Telemetry source |
| --- | --- | --- |
| Guided lab | Teach recognizable failure modes and compare before/after techniques | Deterministic simulated sessions |
| Capture and replay | Inspect one real period of activity with minimal interference | Short in-page recording uploaded after capture |
| Live companion | Watch a real active tab from another device with modest delay | Low-rate summaries followed by detailed recorded data |
| Trace import | Perform high-fidelity diagnosis and calibrate the lightweight probe | CDP/DevTools trace, later Safari trace where practical |

This mirrors a strength of Doom Perf: the experience remains useful without a live source,
simulations exercise otherwise rare states, and the same instruments accept normalized
live data. It also separates two jobs that are easy to muddle:

- an educational experience that makes the browser's scheduling model intuitive; and
- a diagnostic tool that must preserve timestamps, provenance, and uncertainty.

The same application may do both, but every scene should clearly identify whether its
data is simulated, recorded, live, or inferred.

## 4. Lessons to Carry Forward from Doom Perf

The new project should reuse architectural ideas from Doom Perf rather than its Doom
engine or visual assets by default.

### 4.1 Ideas worth retaining

- A stable normalized telemetry schema between collectors and renderer.
- Pluggable live, replay, and simulation sources.
- Large-scale environmental signals for quick recognition, with precise textual
  instruments available on demand.
- Scenarios that isolate one kind of utilization or saturation instead of relying on a
  real system to produce every teaching state.
- Strict separation between collection and presentation.
- Honest handling of absent signals: “unsupported” and “unknown” are not zero.
- A browser client fed by a one-way stream, with no control path from the viewer back to
  the monitored workload.

### 4.2 Ideas not to inherit automatically

- A full game engine. It adds a large implementation and asset surface before the best
  interaction model is known.
- First-person navigation as the only way to reach detail. Frontend performance is
  inherently temporal, and a fixed timeline or overview may communicate causality better
  than rooms alone.
- A fixed one-second telemetry model. User interactions and frame deadlines operate at
  millisecond scale even if the live public stream is deliberately aggregated to one
  second.
- A generic `utilization/saturation/errors` triplet with no provenance. Browser signals
  have different coverage and semantics from Linux counters and need explicit confidence.

## 5. Conceptual Model from “The Browser's Main Thread Is Expensive”

[The source article](https://kciter.so/posts/the-expensive-main-thread/en/) provides the
project's core causal model:

- JavaScript, event handling, framework work, style calculation, layout, and paint all
  compete on a mostly single-file main-thread queue.
- A display refresh creates a recurring deadline. At 60 Hz the interval is about 16.7 ms,
  with only part of it available to application work; higher-refresh displays shorten the
  interval further.
- A task cannot be interrupted midway by input or rendering. A task over 50 ms is
  conventionally considered a long task, but shorter work can still consume a frame
  budget or combine into a slow interaction.
- **Splitting** creates scheduling gaps; **batching** avoids repeatedly paying fixed
  costs; **prioritizing** moves urgent user work ahead of background work; and
  **deferring** keeps unnecessary work out of the current critical period.
- Work can also move to the compositor or a worker, or be removed by dropping, merging,
  or skipping it.
- When arrival rate exceeds processing throughput, the browser/application experiences
  backpressure. Continuing to visualize every event would reproduce the same problem in
  the telemetry system, so the collector itself must apply the article's lessons.

These ideas should be visible, not merely named. A user should be able to compare two
recordings and see that splitting may increase wall-clock completion time while reducing
queueing and deadline misses, or that batching may reduce repeated render cost while
preserving the final state.

## 6. Metric Framework: USE-Inspired, User-Centred, and Causal

### 6.1 Why not use USE literally?

USE works best when a resource has observable busy time, a queue, and an error counter.
The browser's main thread has all three concepts, but ordinary page JavaScript is not
given complete accounting for every task, style calculation, layout, paint, browser
internal, or competing process. Long Task and Long Animation Frame entries reveal costly
intervals, not all busy intervals. Summing them and calling the result “CPU utilization”
would undercount and mislead.

Likewise, JavaScript exceptions are not the useful “E” for this project. A slow but
correct page can miss every important interaction or frame deadline without throwing an
exception. Treating exceptions as the performance error signal would obscure the real
failure.

The proposed adaptation is:

| Lens | Meaning for the main thread | Preferred signals | Honesty rule |
| --- | --- | --- | --- |
| Utilization | Share of available time occupied by browser/page work | Exact busy time from a trace; app-instrumented task durations; long-frame/long-task occupied time as a lower bound | Say “observed blocking share” or “instrumented work share” unless the source has full task accounting |
| Saturation | Work was ready but had to wait, or could not fit before a deadline | Interaction input delay; app queue depth/age; Long Animation Frame blocking duration; missed frame opportunities; pending-work counters | Separate direct queue measurements from symptoms or inference |
| Deadline misses | User-visible service failed its time budget | Slow interactions, over-budget frames, long tasks/frames, poor LCP/CLS, dropped application updates | Keep runtime exceptions and network failures in a separate correctness/error channel |

This is best described in the product as **Utilization, Saturation, and Deadline Misses**,
with an explanation that it is adapted from USE.

### 6.2 Add RAIL as the context layer

[RAIL](https://web.dev/articles/rail) distinguishes Response, Animation, Idle, and Load.
That is more natural for frontend work than a resource-only view: ten milliseconds during
idle time is not equivalent to ten milliseconds between a tap and its next paint. Every
capture segment should therefore carry a RAIL-like context tag, determined by explicit
scenario markers where possible and heuristics otherwise.

RAIL should organize the experience, but not dictate all thresholds. The current web
guidance recommends Core Web Vitals as the higher-level goal-setting system, and real
devices have varying refresh rates. The renderer should derive frame intervals from the
session rather than paint a universal 16.7 ms ruler.

### 6.3 Add user outcomes through Core Web Vitals

[Core Web Vitals](https://web.dev/articles/vitals) currently cover loading (LCP),
responsiveness (INP), and visual stability (CLS). For this project's main-thread focus,
INP is especially useful because an interaction can be decomposed into:

1. input delay before event processing starts;
2. event-handler processing duration; and
3. presentation delay before the next paint.

That decomposition maps exceptionally well to saturation, utilization, and rendering
cost. INP remains an outcome metric, not a full profile. It does not include scrolling,
hovering, or the eventual asynchronous completion of an action, and a page-level INP
value hides the chronology that the visualization needs. Store individual observable
interactions and compute session summaries separately.

LCP and CLS should be included early in the schema but can remain secondary in the first
runtime-focused visual slice. A later “load run” can give navigation, resource, LCP, and
layout-shift events their own guided scene.

### 6.4 Optionally use RED for named user journeys

For an application-owned journey—“open search,” “add to basket,” or “apply filter”—the
service-oriented RED vocabulary is useful alongside USE:

- **Rate:** how often the journey or underlying update occurs;
- **Errors:** whether the app-defined outcome failed, timed out, or was abandoned; and
- **Duration:** how long it took to reach the meaningful application outcome.

This addresses a deliberate boundary of INP: INP stops at the next paint and does not
measure completion of later asynchronous work. RED should only appear when the app emits
safe start/success/failure marks. The probe must not guess business success from a paint.

### 6.5 Preserve the causal timeline

Framework scores are indexes; diagnosis requires sequence. The normalized recording
should preserve enough chronology to answer:

- What was already running when the input arrived?
- How long did the interaction wait?
- What handlers ran, and for how long?
- When did rendering begin, and when could the next frame appear?
- Was work moved to a worker or compositor, or merely delayed?
- Did an application queue grow faster than it drained?
- Which telemetry fields are missing because the browser does not expose them?

The high-level score should always link to the events and provenance that produced it.

## 7. Candidate Signals and Coverage

Use feature detection via `PerformanceObserver.supportedEntryTypes` for every session.
The [Performance Timeline specification](https://www.w3.org/TR/performance-timeline/)
explicitly favors observers over polling, but its buffers are not an unbounded historical
log. The project needs its own bounded recorder once capture begins.

| Signal | What it contributes | Collection notes |
| --- | --- | --- |
| `event` / Event Timing | Interaction input delay, processing, presentation, type, and an interaction identifier | Most useful for INP-like runtime diagnosis; observable entries have thresholds and browser differences |
| `long-animation-frame` | Long frame duration, blocking duration, render/style-layout timing, and script attribution | Richest public main-thread signal, but currently not a universal cross-browser baseline; design a fallback |
| `longtask` | Main-thread tasks at or above 50 ms and limited attribution | Useful fallback/lower bound; superseded conceptually by LoAF where available |
| `largest-contentful-paint`, `layout-shift`, `paint` | Load and visual-stability outcomes | Low event rate and appropriate for the base probe |
| `navigation`, `resource` | Navigation phases and resource request timings/sizes | Resource names can contain sensitive data; sanitize before storage |
| `mark`, `measure` | Application routes, task boundaries, queue ages, framework commits, and named user journeys | Highest diagnostic value when the monitored app is owned and can be instrumented intentionally |
| Page Visibility | Separates valid foreground measurement from hidden/background behavior | Mandatory; hidden intervals must not be blended into foreground rates |
| Optional `requestAnimationFrame` sampler | Actual visible callback cadence and long gaps | Costs one callback per frame and pauses in most hidden tabs; use only in explicit capture mode and benchmark it |
| Optional application counters | Queue depth, oldest-item age, incoming and completed update rates, worker jobs, dropped/merged work | Only honest way to display some forms of backpressure directly |
| CDP/DevTools trace | Full main-thread task categories, scripting/rendering/painting, frames, CPU samples if enabled | High fidelity and high volume; Chromium development/lab adapter rather than portable field mode |

The [Long Animation Frames specification](https://www.w3.org/TR/long-animation-frames/)
defines frame-level timing and script attribution, while Google's
[`web-vitals` attribution build](https://github.com/GoogleChrome/web-vitals) already
connects slow interaction phases with relevant long-animation-frame data. Using that
library is preferable to reimplementing INP and Core Web Vitals arithmetic. A thin custom
layer can record the event chronology and app-specific counters the library does not own.

### 7.1 Source fidelity levels

Every session and metric should carry a source/fidelity tag:

- `simulated`: deterministic teaching data;
- `page-public-api`: cross-browser public timings only;
- `page-public-api+app-marks`: public timings plus owned-application instrumentation;
- `chromium-loaf-attribution`: LoAF/script attribution available;
- `cdp-trace`: external Chromium trace accounting;
- `imported-unknown`: imported data without a recognized capability manifest.

The renderer should change its wording and available drill-down by fidelity. It must not
fill unavailable categories with zeros or infer script ownership from timing alone.

## 8. Collection Alternatives

### 8.1 Lightweight in-page probe — recommended first source

**Advantages**

- Works in real user activity rather than only automation.
- Uses standard performance entries and can be integrated behind a query parameter,
  feature flag, or sampling decision.
- Can add app-specific User Timing and queue counters, which external tools cannot infer.
- Portable architecture even though individual entry types vary by browser.

**Limitations**

- Its callbacks execute in the monitored page's environment and therefore contribute
  some observer effect.
- It cannot see all main-thread work or other processes, and cross-origin iframe detail
  is limited.
- Detailed script attribution is browser-dependent.
- It must be deployed with the application or injected by a development tool.

**Best fit:** owned applications, short field captures, and the initial vertical slice.

### 8.2 External CDP/DevTools tracing — recommended calibration and lab source

Chrome's [DevTools Performance tooling](https://developer.chrome.com/docs/devtools/performance)
and the [CDP Tracing domain](https://chromedevtools.github.io/devtools-protocol/tot/Tracing/)
can record far more complete runtime activity. Android Chrome can be inspected through
[remote debugging](https://developer.chrome.com/devtools/docs/remote-debugging), though
screencasting should be disabled during animation measurement because it affects frame
rate.

**Advantages**

- Much more defensible main-thread utilization and work-category accounting.
- No application bundle change.
- Excellent source for offline conversion into the project's normalized format.
- Provides a ground-truth comparison for measuring the lightweight probe's blind spots
  and overhead.

**Limitations**

- Chromium-specific protocol and version-sensitive trace categories.
- Device pairing/debugging setup is inappropriate for ordinary production sessions.
- Tracing itself has overhead; stacks, screenshots, and broad trace categories increase
  it substantially.
- Large recordings need offline parsing and aggressive normalization before rendering.

**Best fit:** development, controlled mobile labs, regression investigation, and probe
validation.

Safari Web Inspector also offers CPU, JavaScript, layout/rendering, and frame timelines,
but should initially be treated as a separate importer/spike rather than forced into a
CDP-shaped live collector.

### 8.3 Browser extension or custom DevTools panel — useful later

An extension can inject the probe into applications that cannot be rebuilt and can host
the visualization outside the page. A DevTools panel also suits expert investigation.
The cost is browser-specific packaging, broad permissions, store/distribution friction,
and another injection environment that may alter results. It is attractive after the
schema and visual experience prove useful, not before.

### 8.4 General-purpose RUM or OpenTelemetry integration — ecosystem path

A RUM/telemetry SDK can supply session metadata, route changes, network spans, and
correlation with backend traces. This becomes valuable if the project moves from a
teaching instrument toward production observability. It also adds SDK weight, privacy
governance, vendor/schema coupling, and still does not create complete main-thread task
accounting. Design an adapter boundary, but do not select a platform for the initial
prototype.

### 8.5 Synthetic automation — repeatability path

Playwright/WebDriver scenarios can produce repeatable interactions on fixed devices and
collect traces. They are ideal for regression comparisons and curated exhibits, but do
not replace field behavior or the thermal, scheduling, and contention conditions of a
real mobile session.

### 8.6 Dedicated, shared, and service workers — processing aids, not magic sensors

A dedicated worker can normalize or encode batches after the page has copied the relevant
fields. A SharedWorker may help same-origin desktop tabs exchange session data, and a
service worker may help relay already-created telemetry. None of them provides a portable
way to inspect another page's main-thread Performance Timeline. Service-worker lifecycle
is also intentionally independent and interruptible. Treat workers as optional pipeline
components, not as a way to make collection itself external or zero-cost.

## 9. Live, Recorded, and Hybrid Transport

### 9.1 Alternatives

| Approach | Benefits | Costs and distortions | Appropriate role |
| --- | --- | --- | --- |
| Raw live event stream | Immediate and theatrical; easy to relate an action to motion | Serialization, networking, backpressure, battery use, and a temptation to process every event | Avoid as the default monitored-page path |
| Aggregated live summaries | Low bandwidth; enough for utilization/saturation ambience and health | Loses exact causality until details arrive | Live companion mode |
| Short recording then upload | Lowest network contention during the critical activity; deterministic seek and comparison | Not truly live; memory must be bounded; final upload can be lost | Recommended MVP |
| Hybrid delayed-live | Immediate coarse state plus detailed replay after the window | Two fidelity levels must reconcile cleanly | Recommended target after overhead validation |
| External trace recording | Highest diagnostic fidelity and no app integration | Lab-only setup, large files, tracing overhead | Calibration and expert mode |

### 9.2 Why same-phone, different-tab live viewing is misleading

Mobile browsers generally give the foreground page preferential execution. Switching to
the visualization tab changes the monitored page's visibility, pauses most
`requestAnimationFrame` callbacks, and may throttle timers or eventually freeze/discard
the page. The [Page Visibility API documentation](https://developer.mozilla.org/en-US/docs/Web/API/Page_Visibility_API)
describes these background policies.

Therefore:

- A second tab on the same phone is suitable for starting a capture and then replaying it
  after the monitored activity, not for claiming an undisturbed concurrent live view.
- A trustworthy live demo keeps the monitored page visible on the phone and renders the
  companion on a laptop, tablet, external display, or another browser process/device.
- Every recording includes visibility transitions, and foreground and hidden intervals
  are never aggregated together.

Same-origin `BroadcastChannel` can remain a zero-server convenience for desktop windows
or post-capture replay, but it does not solve mobile backgrounding and should not define
the architecture.

### 9.3 Relay topology

Use a small session relay rather than attempting a direct browser-to-browser connection:

```text
MONITORED PAGE / MOBILE                     COMPANION BROWSER
┌───────────────────────────┐              ┌──────────────────────────┐
│ PerformanceObserver(s)    │              │ Live overview            │
│ app marks + queue gauges  │              │ replay + seek            │
│ visibility/capabilities   │              │ causal detail drawer     │
└─────────────┬─────────────┘              └────────────▲─────────────┘
              │ compact summary POST/WS                 │ SSE or WS
              │ final recording upload                  │
        ┌─────▼──────────────────────────────────────────┴─────┐
        │ Session relay + normalizer + bounded session store  │
        │ one-time pairing code; live tail + replay segments  │
        └─────────────────────────────────────────────────────┘

OPTIONAL LAB SOURCE
CDP / imported trace ──► trace adapter ──► same normalized session format
```

The relay owns no application control capability. The monitored page only publishes to a
random, short-lived session; viewers receive a separate read token or pairing code. The
recorded log is the authoritative representation, and the live stream is simply its
coarse, lossy preview.

For the final small upload or page transition, `navigator.sendBeacon()` is a possible
best-effort path: the [Beacon specification](https://w3c.github.io/beacon/) asks user
agents to minimize contention with time-critical work. It is limited to small payloads,
provides no success response, and is not offline delivery, so normal chunked upload with
acknowledgements remains necessary for valuable recordings.

## 10. Probe Design and Observer-Effect Safeguards

The monitored application must always win. Telemetry is expendable; application work is
not.

### 10.1 Capture rules

- Off by default; enable through an explicit development flag, session pairing action,
  or controlled production sample.
- Prefer `PerformanceObserver` delivery over polling.
- Record only explicitly supported entry types and send a capability manifest.
- Default to a short foreground window—provisionally 15–30 seconds—with a hard maximum.
- Store a compact bounded ring in memory. On reaching a high-water mark, merge summaries
  and drop oldest low-value detail rather than expand memory or block.
- Copy only the fields in the normalized contract. Do not retain live DOM nodes or whole
  browser entry objects.
- Do no JSON encoding, compression, IndexedDB writes, or network request per observed
  entry.
- If a worker is used for encoding/aggregation, send batches rather than one message per
  event. A worker cannot simply observe the window's full main-thread timeline on the
  page's behalf; it is a processing aid, not an out-of-thread sensor.
- Disable the optional rAF cadence sampler outside explicit capture mode.
- Suspend foreground rates at `visibilitychange`; flush or close the active segment and
  mark the transition.
- Fail open: relay loss, a full buffer, encoding failure, or missing API drops telemetry
  and increments a loss counter; it never retries in a tight loop or stalls the app.

These rules intentionally apply the article's batching, deferring, worker, and
drop/merge/skip recommendations to the telemetry system itself.

### 10.2 Live-mode rules

- Publish a small coalesced envelope at no more than a provisional 1 Hz.
- Send current state and interval summaries, not every raw event.
- Cap queued unsent envelopes at one or two; replace stale state with the latest value.
- Back off on slow/erroring networks and stop when hidden.
- Never compete with initial resource loading; begin after the app-defined readiness mark
  or a conservative post-load point.
- Make telemetry quality visible to the viewer: last-update age, dropped detail count,
  clock discontinuities, and source capability.

### 10.3 Privacy-by-minimization defaults

Performance entries can expose URLs, element selectors, route names, script locations,
and user actions. Default collection should:

- strip URL query strings/fragments and reduce resource URLs to origin plus allowlisted
  path templates;
- use explicit safe interaction names or coarse element roles instead of DOM text and
  raw selectors;
- never capture input values, DOM text, response bodies, screenshots, or request headers;
- allow application owners to map sensitive marks to stable safe identifiers;
- use short retention and a visible capture indicator;
- keep production capture a separate policy decision from a local educational lab.

### 10.4 Provisional overhead budget and test gate

Discovery should turn these into evidence-based budgets. Initial targets worth testing
are:

| Cost | Provisional target |
| --- | --- |
| Added probe code | At most 15 KB compressed in the capture build; loaded after critical app code where possible |
| Observer callback | p95 below 0.5 ms, with no probe-created long task |
| Probe main-thread time | Below 0.5% of visible wall time in summary mode and 1% in detailed capture mode |
| In-memory recording | At most 2 MB by default; bounded by time and event count |
| Live transport | At most 2 KB/s typical, one coalesced update per second |
| User outcome regression | No statistically credible regression greater than 2% in repeated p75 interaction/frame measurements on target low-end devices |

Validate with repeated A/B runs in three states—probe absent, recording only, and hybrid
live—and compare against an external trace. Test at least a representative low/mid-range
Android device and an iPhone if iOS is in scope. A single “looks fine” run is not an exit
gate; mobile thermal variance and browser noise require multiple randomized repetitions.

## 11. Normalized Session Model

Keep the schema event-oriented and versioned; derive visual state and rollups downstream.
At a high level, a recording contains:

### 11.1 Session metadata

- schema version and session identifier;
- source/fidelity type and `PerformanceObserver.supportedEntryTypes`;
- browser/OS/device class at a privacy-safe granularity;
- viewport, device pixel ratio, estimated refresh interval, and reduced-motion setting;
- monotonic `timeOrigin` anchor plus wall-clock anchor for transport ordering;
- capture configuration, sampling rates, buffer caps, and sanitization policy version;
- visibility segments and any lifecycle termination reason.

### 11.2 Event families

- capture start/end, visibility, navigation, and route/journey markers;
- interactions and their input/processing/presentation phases;
- long-animation-frame and long-task intervals with safe attribution;
- optional rAF gaps or frame-budget misses;
- LCP, CLS/layout shifts, paint, navigation, and resource timing;
- app marks/measures, queue depth and oldest-work age, input/completion rates, worker job
  state, and drop/merge counters;
- probe self-time, buffer high-water events, telemetry drops, and transport state.

### 11.3 Rollups

Use a fixed short interval, probably one second for live transport, to derive:

- observed blocking share and its precise definition;
- interaction count and input/processing/presentation delay distributions;
- long task/frame count, blocking time, and largest offender;
- frame opportunities and misses where observable;
- application backlog depth/age and arrival-versus-completion rate;
- deadline-miss counters and current Core Web Vitals candidates;
- telemetry coverage and loss.

Raw event timestamps remain the replay source. Rollups are an index and live preview, not
the authoritative record.

## 12. Visual Design North Star: Inside the Phone

The visualizer should place the viewer **inside a stylized phone, looking out**. The
experience is a controlled cutaway rather than a physically literal reconstruction of a
handset. Its job is to make the relationship between the person, their touch, browser
work, and the resulting display frame spatially intuitive.

The person and phone establish the world; the internal machinery carries the metrics;
and an exact 2D trace remains the evidence layer.

### 12.1 Scene composition and point of view

- The virtual camera sits in the rear half of the phone's chassis, aimed toward the
  display. The display is the far plane of the scene rather than a flat HUD attached to
  the camera, creating visible space for work to travel through the device.
- The phone's frame, side buttons, speaker/camera apertures, structural ribs, and abstract
  circuitry define the edges and depth of the room. They need not imitate a particular
  manufacturer or expose literal electronics.
- Beyond the display, a stylized person holds the phone. Their hands, face or silhouette,
  and the surrounding room are seen through an intentionally translucent or x-ray-like
  screen treatment. The effect is surreal by design: real phone displays are opaque, but
  this cutaway makes the user's action and the browser response occupy one continuous
  space.
- Generic app icons, cards, and navigation shapes sit on or just in front of the display
  plane. They are horizontally reversed when viewed from inside, yielding the immediate
  sensation of seeing the back of the interface. Avoid mirrored body text: diagnostic
  labels and exact values remain correctly oriented in the near-field UI.
- A finger approaching or touching the outside glass creates a contact shadow, pressure
  bloom, or light ripple that enters the phone at the actual interaction time. Do not
  invent taps or emotional reactions in live data. The outside person reflects recorded
  actions, not an inferred model of user frustration.
- Use a fixed or gently floating first-person camera with limited head/parallax movement
  for the initial release. The user should feel enclosed by machinery without having to
  navigate a level before understanding the current state.

The outside person should be stylized enough to avoid an uncanny or surveillance-like
feeling. A silhouette, softly shaded low-poly figure, or configurable neutral avatar is
more appropriate than a detailed simulated portrait. No real camera image is required.

### 12.2 Telemetry-to-scene mapping

| Performance concept | Inside-the-phone representation |
| --- | --- |
| User input | The outside fingertip contacts the glass; a pulse crosses the reversed app surface and enters the chassis |
| Input delay | The pulse waits behind a closed main-thread intake gate while older work passes |
| Main-thread utilization | The central mechanism's duty cycle, occupied track length, light intensity, and heat-like shimmer increase with measured or observed work |
| Saturation/backpressure | Task capsules accumulate in a visible queue; the oldest capsule grows a waiting-time halo and surrounding braces begin to strain |
| Event-handler processing | The interaction capsule travels through the single main-thread mechanism, with JavaScript work visibly occupying it |
| Style and layout | A geometry/alignment chamber rearranges luminous interface frames before they can reach the screen |
| Paint | A raster/ink chamber fills the prepared frame with color and texture |
| Presentation delay | A completed interaction waits in the short space between processing machinery and the next display pulse |
| Frame cadence | The back of the screen emits a scan, shutter, or heartbeat at the session's measured refresh interval |
| Missed frame/deadline | A display pulse passes before the prepared frame arrives; the old image visibly persists and a bounded shock travels through the chassis |
| Long task/long animation frame | One oversized capsule monopolizes the main line; nearby mechanisms vibrate or dim until it clears |
| Compositor work | A fast, shallow mechanism immediately behind the display continues moving prepared layers even while the deeper main line is occupied |
| Worker work | Parallel side chambers process transferable capsules away from the central line, then return compact results toward the display |
| Network/resource activity | Signals enter through antenna-like conduits at the phone's edges; waiting network work remains visually distinct from main-thread queueing |
| Dropping, merging, memoization | Old capsules dissolve, several updates fuse into one latest-state capsule, or a cached finished piece bypasses repeated work |
| Telemetry loss/unsupported signal | A clearly labelled dormant or obscured mechanism, never a calm green zero |

The mapping must preserve source fidelity. For example, a public-API session may drive
“observed blocking” intensity without claiming the entire mechanism represents exact CPU
use; a CDP trace may unlock the fuller scripting/rendering category breakdown.

### 12.3 Internal machinery and the dispatch model

The phone interior should still use the dispatch-yard model because it communicates a
single non-preemptive main thread particularly well:

- Incoming tasks are capsules, packets, or carriages waiting for one central belt/track.
- Occupied track over a time window is the utilization-like view; queue length and oldest
  age are saturation.
- Recurring gates are synchronized with the far screen's display pulses. A frame that
  cannot clear its gate becomes a visible missed departure.
- JavaScript, style, layout, paint, and browser/unknown work use distinct silhouettes and
  textures as well as accessible colors.
- Splitting turns one oversized capsule into bounded pieces with gaps. Batching combines
  many setup-heavy capsules. Priority moves the user's glowing capsule forward. Deferral
  parks non-urgent work in a dark side bay.
- Worker chambers and the near-screen compositor path are visibly parallel, but style,
  layout, and DOM work must not be drawn there merely for symmetry. The scene should not
  imply browser concurrency that the telemetry cannot support.

This turns the earlier factory idea into the **internal logic of the phone**, while the
person, reversed screen, and touch events give it a distinctive viewpoint and emotional
scale.

### 12.4 Pulse, vibration, sound, and material response

The internal world should feel alive even at modest load, then become increasingly tense
under contention:

- A gentle baseline electrical pulse establishes that the device is active.
- Work rate controls the frequency or occupancy of machinery, while wait time controls
  strain, compression, and queue halos. Do not map every metric to brightness or speed.
- Long frames can create a low mechanical shudder; a missed display deadline can create a
  short directional shock from the gate toward the screen. Camera shake must be capped,
  smoothed, and optional so that a bad session remains viewable.
- The compositor can retain a smooth near-screen rhythm during a simulated main-thread
  blockage, visually echoing the article's transform-versus-main-thread animation example.
- Sound can separate resource use from saturation: a steady productive hum for occupied
  work, rising mechanical tension for queueing, and a restrained skipped beat for a
  missed frame. It should be muted by default in professional/diagnostic contexts.
- Color is never the only encoding. Shape, motion, spacing, labels, and sound all reinforce
  state.
- `prefers-reduced-motion` mode replaces camera vibration and rapid pulses with stable
  illumination, gauges, queue spacing, and the exact timeline.

Motion should be normalized and clamped rather than directly proportional to unbounded
event counts. Otherwise a telemetry storm would make the visualizer illegible and repeat
the performance failure it is trying to explain.

### 12.5 Interaction and camera modes

The initial experience should have three levels of control:

1. **Observe:** a stable inside-phone camera with small parallax watches live or replayed
   machinery. The user's hand and the reversed app surface anchor attention at the far
   screen.
2. **Inspect:** selecting a queue, capsule, missed pulse, or screen touch pauses or slows
   playback and moves the camera a short, authored distance toward that mechanism.
3. **Explain:** a correctly oriented panel opens with the exact metric, source,
   confidence, interaction phase, and synchronized trace interval.

Avoid unrestricted first-person movement in the first release. It raises scene and
control cost, makes comparison harder, and can let the viewer turn away from the screen
at the important moment. If exploration later proves valuable, constrain it to named
vantage points within the handset.

For synchronized A/B comparison, show two handset interiors side by side or split the
same phone into matched halves. The outside finger performs the same recorded gesture in
both; one interior can show monolithic work while the other shows yielding, batching, or
worker offload.

### 12.6 Evidence layer: trace theatre inside the scene

The conventional diagnostic view remains essential:

- A subtle near-camera wrist-console, lower drawer, or projected strip carries the time
  ruler and current USE/RAIL/Web Vitals values without pretending to be literal hardware.
- Selecting any animated object highlights the exact trace interval that drove it.
- Pause, scrub, step, bookmarks, and raw values are always available in recorded mode.
- A “why this is moving” panel states the definition, source, and confidence and may name
  relevant article techniques without claiming that one symptom proves one fix.
- The reversed app surface is atmosphere and interaction context; it is not used for
  long-form diagnostic text.

### 12.7 Implementation alternatives within the selected concept

| Expression | Strengths | Weaknesses | Role |
| --- | --- | --- | --- |
| Fixed shallow-3D diorama | Strong composition, predictable performance, immediate readability | Less exploratory | Recommended first prototype |
| Limited-parallax 3D interior | Greater embodiment and depth; authored inspection moves | More camera and occlusion work | Recommended target if the fixed study succeeds |
| Freely navigable phone interior | Closest to Doom Perf's spatial exploration | High build cost; easy to lose temporal focus | Optional later mode |
| Schematic/x-ray fallback | Works on weaker GPUs and reduced-motion settings | Less atmospheric | Required graceful fallback |
| Queue-management simulation | Lets users actively apply split/batch/priority/offload policies | Cannot safely control arbitrary live apps | Guided lab mode using the same scene |

The 3D requirement makes Three.js/R3F a reasonable initial candidate, with a custom
WebGL/WebGPU renderer as an alternative if measurement shows framework overhead or scene
control problems. Do not animate thousands of DOM nodes. The companion viewer runs in a
separate context and therefore does not contaminate the monitored page's metrics, but it
still needs its own frame, memory, object-count, and thermal budgets.

## 13. Guided Scenario Catalogue

The article supplies a strong first curriculum. Synthetic sessions can make each state
repeatable and can later be paired with small real demo pages:

| Scenario | Contrast to visualize | Main lesson |
| --- | --- | --- |
| Monolithic block | 500 ms/2 s main-thread hold versus responsive compositor motion | One long task freezes JS-driven input and rendering |
| Streaming chat | Render a burst immediately versus yield in bounded chunks | Splitting creates opportunities for input/paint but can lengthen completion |
| Event storm/editor | Run per event versus debounce/throttle | Batching avoids repeated fixed costs |
| Live ticker board | Redraw every message versus coalesce once per frame | Latest state can be preserved without rendering every arrival |
| Photo previews | FIFO versus clicked-item priority | Order changes perceived responsiveness without reducing total work |
| Long feed | Render everything versus near-viewport content | Deferring offscreen work preserves headroom |
| Motion | Layout-affecting animation versus transform/FLIP | Similar visuals can consume different browser pipelines |
| Heavy compute | Main thread versus worker with batched/transferable data | Offloading helps only when compute saved outweighs communication |
| Overloaded feed | Keep all versus drop old, merge latest, or memoize | Backpressure ultimately requires eliminating work |

These scenarios also provide the repeatable corpus for renderer tests, schema migrations,
and comparison UX before arbitrary live apps are supported.

## 14. High-Level Architecture Principles

- **One canonical session format.** Simulators, in-page captures, CDP traces, and future
  RUM adapters all normalize before rendering.
- **Replay is foundational.** Live mode appends/tails the same log and therefore cannot
  become a separate UI or schema.
- **Collection is bounded and lossy by design.** Preserve high-value interactions and
  deadline misses first; aggregate or discard ambient detail under pressure.
- **The monitored app is isolated from viewers.** Publish-only session tokens and a relay
  prevent a visualizer from issuing application commands.
- **Provenance travels with values.** Measured, app-instrumented, trace-derived, estimated,
  unsupported, and dropped are first-class states.
- **The clock is monotonic inside a capture.** Network arrival time never reorders page
  events; wall time is only an anchor between components.
- **The view is level-of-detail driven.** Live overview uses rollups; paused replay can
  load detailed segments around an interaction.
- **The visualization obeys its own lesson.** Coalesce updates once per frame, cap scene
  objects, virtualize detail panels, pause invisible animation, and move suitable parsing
  to a worker.

## 15. Delivery Phases and Decision Gates

### Phase 0 — Discovery spikes

- Define one target owned demo app and three representative interaction journeys.
- Capture the same journeys with `web-vitals` attribution/public APIs and an external CDP
  trace; document exactly which signals reconcile and which do not.
- Run a browser capability spike on the intended Android Chrome and iOS Safari versions,
  plus desktop Chrome/Firefox/Safari if they are in scope.
- Measure probe-off, record-only, and summary-live overhead using randomized repeated
  runs.
- Build two throwaway inside-phone prototypes from the same recording: a fixed shallow-3D
  diorama and a limited-parallax interior. Give both the same small trace/evidence drawer
  and test comprehension, not visual preference alone.
- Prove phone-to-relay-to-laptop pairing over HTTPS/WSS and deliberately test network loss
  and page backgrounding.

**Exit gate:** a short mobile interaction recording replays elsewhere; unsupported data
is represented honestly; the probe meets or revises an agreed overhead budget; and one
visual concept communicates queueing and a missed frame without explanation from its
author. Test participants should also recognize that they are inside a phone looking
through the back of its screen toward the holder without needing a long introduction.

### Phase 1 — Recorded vertical slice

- Versioned session schema and capability/provenance model.
- Opt-in 15–30 second in-page capture with bounded memory and safe URL/target handling.
- Final upload, session list, replay, scrub, pause, step, and one interaction detail view.
- Shallow-3D inside-phone overview, reversed generic app surface, recorded touch pulse,
  internal main-thread machinery, and a synchronized conventional timeline.
- Two deterministic teaching scenarios and an A/B comparison view.

**Exit gate:** a user can record a real interaction on a target phone, open the result on
another browser, identify input/processing/presentation phases and relevant long
frames/tasks, and compare it with a known improved scenario.

### Phase 2 — Hybrid live companion

- One-Hz coalesced summary stream with current freshness and loss indicators.
- Separate publish/view credentials and short-lived pairing codes.
- Live tail automatically reconciled with the detailed recording when uploaded.
- Backpressure, offline, relay failure, and hidden-page behavior.

**Exit gate:** the app stays foregrounded on the phone while another device sees a useful
low-latency overview; detailed replay becomes available without discontinuity; overhead
remains inside the measured budget.

### Phase 3 — Fidelity and source adapters

- CDP trace-to-session converter and a documented Android remote-capture flow.
- Exact trace-backed main-thread utilization and category breakdowns, visually
  distinguished from public-API lower bounds.
- Additional app counters/marks for queue depth and named journeys.
- Safari trace/import feasibility spike; extension/DevTools-panel decision.

**Exit gate:** the same visualizer can compare a lightweight field-style capture with a
high-fidelity trace of the same journey and explain their coverage differences.

### Phase 4 — Curriculum and product polish

- Complete the article-derived scenario catalogue.
- Guided annotations, narration, synchronized A/B playback, bookmarks, and export.
- Refine the outside holder, hands, screen-back material, inner chassis, authored camera
  moves, pulse/vibration language, and side-by-side handset comparison.
- Accessibility: reduced motion, keyboard controls, non-color encodings, text/table
  equivalents, and screen-reader-readable metric summaries.
- Visual performance hardening and low-end viewer fallback.

**Exit gate:** the project works both as a self-guided teaching instrument and as an
honest entry point into the captured diagnostic detail.

## 16. Risks and Mitigations

| Risk | Why it matters | Mitigation |
| --- | --- | --- |
| Probe changes the result | Main-thread telemetry itself uses the main thread | Recorded-first design, passive observers, batching, bounded buffers, A/B overhead gate, external-trace calibration |
| “Utilization” is undercounted | Public APIs omit short tasks and browser internals | Call it observed blocking/instrumented share; reserve exact utilization for trace sources |
| Same-device live view corrupts the scenario | Background tabs are throttled/paused | Keep target foregrounded; use another device for live; use the second tab for replay |
| Browser support differs | Event, LoAF, attribution, and memory signals are uneven | Runtime capability manifest, tiered schema, fallbacks, missing-not-zero UI |
| Visual metaphor implies false concurrency | Browser pipeline details are nuanced | Canonical timeline beneath the metaphor; source/confidence labels; design review against real traces |
| Inside-out screen is confusing or feels physically implausible | A literal opaque display would make the holder invisible, while an unexplained transparent one may look accidental | Present it explicitly as an x-ray/cutaway; use reversed safe icon silhouettes and a short opening camera cue |
| The outside person feels uncanny or surveillant | A detailed face could distract from the performance story or imply camera capture | Use a neutral stylized avatar or silhouette; never require a real camera image |
| Jank effects make a bad trace unpleasant to inspect | Unbounded shake, flicker, or sound would punish the viewer precisely when detail matters | Clamp and smooth effects; pause/slow controls; reduced-motion and muted diagnostic modes |
| Telemetry backpressure recreates app backpressure | Raw event volume can exceed transport/render throughput | Summarize, merge latest, drop ambient detail, cap queues, report loss |
| Sensitive data enters a recording | URLs, selectors, marks, and script names can identify users/data | Safe naming contract, URL stripping, no text/values/bodies, short retention, explicit capture |
| Mobile lifecycle loses the final upload | OS may terminate hidden pages quickly | Small acknowledged chunks after capture; Beacon only as best effort; surface incomplete sessions |
| CDP becomes a Chromium lock-in | It offers tempting detail unavailable elsewhere | Adapter boundary; portable public-API core; label fidelity rather than flattening it |
| The visualizer itself janks | Event-rich animation can overwhelm the companion | Frame-coalesced rendering, LOD, object caps, worker parsing, replay segment loading, its own perf harness |
| Scores become the product | One number hides cause and context | Preserve individual interactions, session chronology, RAIL context, and causal drill-down |

## 17. Decisions to Make After Discovery

| Decision | Alternatives | Suggested default |
| --- | --- | --- |
| Primary audience | Performance education; application developers; production observability | Education + developer diagnosis first |
| First monitored platform | Android Chrome; iOS Safari; broad desktop | Android Chrome first, while preserving a portable public-API baseline |
| App ownership | Owned/instrumentable app; arbitrary site via extension; trace import only | Owned demo/real app first |
| Default operating mode | Raw live; summary live; recorded; hybrid | Recorded MVP, hybrid target |
| Inside-phone execution | Fixed diorama; limited parallax; free navigation | Fixed diorama first, limited parallax if it improves embodiment without harming comprehension |
| Outside holder style | Silhouette; low-poly person; detailed character; camera feed | Neutral silhouette or softly shaded low-poly figure; no camera feed |
| Internal visual language | Dispatch machinery; literal circuit board; abstract organism | Dispatch machinery embedded in a stylized chassis |
| Renderer | Three.js/R3F; custom WebGL/WebGPU; Canvas fallback | Spike Three.js/R3F plus a schematic fallback; avoid a game/physics engine commitment in Phase 0 |
| Primary scope | Runtime interaction/animation; initial load; memory/network | Runtime interaction and animation first |
| Utilization semantics | Public-API lower bound; app-instrumented share; exact trace accounting | Show all when available, never collapse them into one unlabeled number |
| Live transport | WebSocket both ways; POST ingest + SSE fan-out; WebRTC direct | Simple relay; choose transport after payload/latency spike |
| Retention/deployment | Local-only; ephemeral hosted sessions; production RUM | Local/ephemeral sessions first |
| Comparison model | Single run; side-by-side A/B; statistical cohort | Single replay plus synchronized A/B first; cohorts later |

The most consequential decision is whether this is primarily an **educational visual
instrument** or a **production observability product**. The former favors curated
scenarios, explicit short captures, and a strong metaphor. The latter quickly introduces
sampling policy, identity/privacy governance, fleet-level aggregation, source maps,
release correlation, and long-term storage. The architecture can leave room for both,
but the first release should not attempt both product surfaces equally.

## 18. Definition of a Successful First Project

The first meaningful release is successful when:

- A real, visible mobile webapp can record a bounded interaction window without live
  rendering or synchronous persistence in that tab.
- The session replays in a separate browser from a recognizable inside-phone viewpoint,
  with the reversed app surface and holder at the far screen, dynamic inner machinery,
  and an exact timeline/detail view.
- Utilization-like, saturation, and deadline-miss signals are correctly named and linked
  to their measurement source.
- A user can see the input-delay, processing, and presentation portions of an interaction
  and correlate them with long work or rendering.
- Live companion mode, if enabled, carries only bounded coalesced summaries and visibly
  reports freshness and loss.
- The same renderer can play deterministic “split versus monolith” and “per-event versus
  batched” scenarios, establishing the educational feedback loop.
- Probe overhead passes a repeated low-end-device A/B gate, and collection automatically
  sacrifices telemetry before application responsiveness.
- Hidden/background intervals, unsupported APIs, dropped events, and inferred values are
  impossible to mistake for healthy zeros or exact measurements.

## 19. Immediate Next Steps

1. Select one owned mobile webapp and define three short runtime journeys: a tap-driven
   update, a scroll/animation period, and a bursty-data or list-rendering action.
2. Build a disposable probe spike around `web-vitals/attribution`, PerformanceObserver,
   Page Visibility, and two safe application marks; record locally in memory only.
3. Capture the same journeys through Android Chrome remote debugging and derive a small
   reconciliation report: exact trace busy time versus public-API observed blocking time,
   interactions, and long frames/tasks.
4. Define session schema version 0 from those real captures, including capabilities,
   provenance, visibility, and loss before adding more metrics.
5. Produce two low-cost inside-phone studies—a fixed shallow-3D diorama and a
   limited-parallax version—driven by the identical JSON recording and sharing the same
   trace drawer.
6. Measure probe overhead and revise the provisional budgets before building the relay.
7. Implement the recorded vertical slice. Add the one-Hz live summary only after the
   recorded path is useful and the overhead gate passes.

The default path should change only when these spikes produce contrary evidence. In
particular, do not begin with raw live telemetry, a same-phone background visualization,
or an unrestricted game/physics engine. Start with a tightly composed 3D diorama: it
delivers the desired embodiment while limiting the scene, camera, and performance work
that could obscure the main-thread scheduling story.

## 20. Reference Material

- [The Browser's Main Thread Is Expensive](https://kciter.so/posts/the-expensive-main-thread/en/)
- [Performance Timeline specification](https://www.w3.org/TR/performance-timeline/)
- [Long Tasks API specification](https://www.w3.org/TR/longtasks-1/)
- [Long Animation Frames API specification](https://www.w3.org/TR/long-animation-frames/)
- [Interaction to Next Paint](https://web.dev/articles/inp)
- [Web Vitals](https://web.dev/articles/vitals)
- [RAIL performance model](https://web.dev/articles/rail)
- [`web-vitals` library and attribution build](https://github.com/GoogleChrome/web-vitals)
- [Chrome DevTools runtime performance](https://developer.chrome.com/docs/devtools/performance)
- [Chrome remote debugging for Android](https://developer.chrome.com/devtools/docs/remote-debugging)
- [Chrome DevTools Protocol Tracing domain](https://chromedevtools.github.io/devtools-protocol/tot/Tracing/)
- [Page Visibility API](https://developer.mozilla.org/en-US/docs/Web/API/Page_Visibility_API)
- [Beacon specification](https://w3c.github.io/beacon/)
- [Safari Web Inspector](https://developer.apple.com/documentation/safari-developer-tools/web-inspector)

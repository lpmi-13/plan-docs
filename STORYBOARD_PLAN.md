# Storyboard plan: turn a system scenario into a visual story

## Goal

Make each new scenario feel like an incident worth investigating before asking anyone to read a description. The first trial is Break Team's **rotated database password** scenario. The output should work as a short, sound-off social video and as a set of comic panels. A later, clearly marked case-file version can explain the mechanism and repair.

The reusable unit is a **scenario story package**: a factual incident brief, a sanitized event fixture, a panel script, reusable visual components, and exports for each channel. The package should be possible to produce for another project without building a 3D world or recording a game.

## What the source projects already provide

| Source | Useful material | How to use it |
| --- | --- | --- |
| [Break Team README](../breakteam/README.md), [scenario definition](../breakteam/services/ops/breakteam_ops/scenario.py), and [controller](../breakteam/services/ops/breakteam_ops/controller.py) | A five-VM shop, three customer journeys, one real fault, and a lifecycle from healthy baseline through measured impact to sustained recovery. | Use the actual topology and event order as the story's skeleton. |
| [Event store](../breakteam/services/ops/breakteam_ops/store.py), [public snapshot](../breakteam/services/ops/breakteam_ops/snapshot.py), and [room API](../breakteam/services/ops/breakteam_ops/web.py) | Timestamped events with public/team/debrief audiences; live SLO and VM snapshots; a debrief with cause and repair after the incident ends. | Record a real run, then make separate spoiler-safe and spoiler-full fixtures. The event log alone does not contain the continuous measurements needed for animation. |
| [Break Team board](../breakteam/services/ops/breakteam_ops/static/board.js) | Customer routes sit next to replica readiness. `GET /` stops in the cluster; products and orders reach the database. | Preserve this distinction in every diagram. “Ready” pods can coexist with failed customer transactions. |
| [Doom Perf trailer edit](../doom-perf/trailer/edit.json), [take inventory](../doom-perf/trailer/takes.json), and [trailer scripts](../doom-perf/scripts/trailer/) | Named takes, a frame-accurate edit, generated cards and music, contact sheets, automated capture, an upload encode, a textless master, and validation. | Reuse the **production discipline**: declarative shots, review sheets, reproducible renders, and checks. For Break Team, the shots are generated system scenes instead of game footage. |

The current Break Team scenario rotates `orders-db-password` in the secrets manager and changes the database role. The orders, catalog, and fulfilment deployments retain the old Kubernetes Secret until someone updates it and restarts them. Existing database connections can keep working, so customer errors appear gradually while pods remain Running and Ready. The storefront does not depend on the database. The evaluator announces **Fixed** only after recovery checks hold for its stability window. These are plot facts, not optional visual flourishes; the [scenario](../breakteam/services/ops/breakteam_ops/scenario.py), [controller](../breakteam/services/ops/breakteam_ops/controller.py), and [evaluator](../breakteam/services/ops/breakteam_ops/evaluator.py) define them.

## Creative direction

Build a **living incident comic**. A page starts as a tidy diagram of customers, edge, two app nodes, database, and the separate operations room. Requests travel as small illuminated threads. When the incident develops, only the affected threads fray; the storefront lane still flows. Panels fracture, bleed into one another, and reassemble as the team forms a shared picture. Keep an inked border, restrained halftone texture, a few strong colors, and readable type. This gives the system a visual identity without pretending that a game environment exists.

Use three visual layers, with a clear meaning for each:

1. **Observed layer:** customer journeys, SLOs, event timing, readiness, and recovery come from the recorded run. Numbers, labels, and ordering must match the fixture.
2. **Metaphor layer:** threads, doors, torn gutters, and panel motion make causal relationships visible. Label the piece as an illustration of an incident, and do not make a metaphor look like a literal product screen.
3. **Interface layer:** brief, faithful crops or redraws of the room/inbox/board establish that people can investigate this system. Do not publish credentials, team codes, private links, or unreviewed terminal output.

For the Break Team pilot, reserve the strongest metaphor for the case-file cut: old credentials are keys still held by app replicas while the database accepts a new key. Show that **existing** connections remain lit until they expire, then fail on reconnection. The teaser can show the resulting customer pattern without revealing the key or the repair. Audio can use request ticks that become irregular, an understated alert pulse, and a clear release at resolution; the animation must still make sense when muted.

### Spoiler rule

The launch teaser may show symptoms, investigation, and that a team can recover. It should not name the rotated password, show the secret-manager clue, expose a database-authentication log line, or display the repair command. The optional **case file** may reveal all of those after a conspicuous “scenario spoilers” card. Keep the two scripts and render fixtures separate so a title, filename, subtitle, thumbnail, or alt text cannot accidentally spoil the teaser.

## First storyboard: Break Team teaser

Target a **roughly 28-second vertical video**. Timing is an editorial starting point; revise it after a phone-sized animatic. Each row is also a standalone comic panel or pair of panels for a carousel. The opening image must state the contradiction immediately.

| Time | Panel and on-screen copy | Motion / sound | Truth to check |
| --- | --- | --- | --- |
| 0–3 s | A green `READY` stamp covers app pods while an order request turns red. **“ALL PODS READY. ORDERS FAIL.”** | Hard split-panel reveal; one request tick cuts out. | Use a run in which replicas are ready as customer orders fail. |
| 3–6 s | Pull back to the five-VM shop map. Three labeled customer paths enter through `edge-01`; the operations room sits apart. **“A live shop. One hidden break.”** | Threads flow through the topology; subtle parallax between paper layers. | Preserve the real route: storefront ends in the cluster; products and orders reach `data-01`. |
| 6–9 s | The incident inbox fills with ordinary work. One line slips beneath other notices, unread. **“THE CLUE IS IN THE NOISE.”** | Fast page-turn and paper shuffling. | Use sanitized, genuinely recorded event ordering; the teaser must hide the cause-bearing line. |
| 9–13 s | Three adjacent frames show the error spreading across new requests while a few existing connections still glow. Storefront stays green. **“IT DOESN'T BREAK ALL AT ONCE.”** | Frames fracture in sequence; ticks become uneven. | The gradual failure and unaffected storefront must match the chosen run. |
| 13–17 s | Team viewpoints appear as distinct colored cursors on customer path, logs, and inbox. A shared thread links their findings. **“FOLLOW THE CUSTOMER.”** | Quick close-ups of evidence, then a single pull-back. | If using real actions, show only actions actually recorded; otherwise mark this as illustrative team activity. |
| 17–21 s | A repair is represented by a torn connection being rejoined, with commands and the cause obscured. **“MAKE THE CHANGE. CHECK THE RESULT.”** | Brief quiet before requests resume. | The actual run must include a repair; do not imply that an unrelated action fixed it. |
| 21–25 s | All customer paths recover. A visible timer completes before the `FIXED` stamp lands. **“RECOVERY HAS TO HOLD.”** | Regular pulse returns; only then sound the resolution hit. | `incident.resolved` is emitted after the evaluator's stability window, not on the first successful request. |
| 25–28 s | Comic page closes into the Break Team title and a single call to action: **“CAN YOUR TEAM FIND THE BREAK?”** plus the playable link or announcement URL. | Hold the final card long enough to read, then clean end. | Confirm the destination and scenario availability at release. |

The carousel can use six pages: the opening contradiction; topology; noise; gradual impact; team investigation; title and call to action. Each page should make sense without animation. A short loop can isolate the opening paradox or the gradual failure spread; avoid placing a link only inside a looping GIF.

### Optional spoiler-full case file

After the teaser, a second 35–50-second cut or comic thread can reveal the mechanism: (1) routine rotation on `ops-01`, (2) database role accepts the new password, (3) deployments still hold the old Secret, (4) old connections mask the break, (5) new connections fail while pods remain Ready, (6) team synchronizes the Secret and restarts the three affected deployments, (7) evaluator waits for sustained customer health. Put the spoiler label before the first clue. This is also a useful educational artifact after people have played.

## Production workflow that transfers to other projects

### 1. Write a one-page scenario brief

For every project, fill in: **audience; promise; observable system; inciting change; visible symptoms; misleading signal; investigation path; repair or resolution; proof of recovery; spoiler boundary; call to action**. Identify which facts are directly observed, which are source-code contracts, and which are editorial metaphors. For a scenario group, use one shared visual grammar and one short panel per distinct failure mode, ending on a single group announcement rather than pretending the faults happened in one run.

### 2. Capture a trustworthy story fixture

For Break Team, run an isolated editorial session with the intended fix, collecting public spectator snapshots throughout and the authenticated event timeline/debrief once the run ends. The [end-to-end harness](../breakteam/e2e/run.py) demonstrates a real-session path and the intended repair; use it as a reference, then record a clean editorial run rather than claiming test output is a participant story. Store a local `story.json` with relative timestamps, selected SLO readings, readiness, visible events, and links back to source event IDs. Keep raw captures private. Generate a **teaser fixture** by allowlisting safe fields; generate a **case-file fixture** separately. Record the source commit, run identifier in private notes, asset versions, and the exact editorial transformations. Never include raw secret values, team tokens, or real participant identifiers in release assets.

If a later project has no event stream, a hand-authored scenario script can drive the same renderer. Mark invented telemetry and actions as illustrative; do not present them as a recording of a real incident. The point is a truthful causal story, not a fake dashboard.

### 3. Build reusable scenes, then storyboard them

Implement a small browser-native SVG/HTML scene kit: topology nodes and lanes, request particles, comic gutters, alert and status stamps, evidence cards, type, and title card. Give each panel a stable ID, duration, caption, data fields, motion cue, sound cue, and spoiler level in a declarative edit file. Keep colors and component shapes consistent across scenarios; vary the failure motif to fit the mechanism (a queue backing up, latency stretching a path, a route cutting off, a rollout splitting traffic). Compose each aspect ratio deliberately so labels and topology are rearranged, not simply cropped.

Start with still panels and a contact sheet. Make a rough animatic with placeholder sound; review it at actual phone size and with audio muted. Only then draw refined assets and animation. The build should render at fixed frame rate and deterministic times, so a revision produces the same frames from the same fixture. Use generated SVG for sharp diagrams and type; use raster illustration or textures sparingly where they add atmosphere.

### 4. Render a small, useful release set

| Asset | Proposed use | Production choice |
| --- | --- | --- |
| 9:16 MP4, about 28 s | Main social teaser and YouTube Short | First-class vertical composition, captions burned in, H.264/AAC export, separate thumbnail. YouTube currently accepts square or vertical Shorts up to three minutes ([YouTube Help](https://support.google.com/youtube/answer/12779649)). |
| Six static comic pages | Carousel, blog, README, or launch thread | Export each page at a phone-readable size and include useful alt text that follows the spoiler rule. |
| 4–6 s silent loop | Reply, post preview, or embedded page accent | Export a small MP4 first; make a GIF only where the destination needs one. Keep the loop intelligible without sound. |
| Textless master and source files | Later crops, localization, corrections | Keep editable artwork, edit data, audio stems, and a high-quality master. |
| Optional 16:9 extended cut | YouTube or project page after the teaser proves useful | Add more investigation and context; recompose panels for widescreen. |

YouTube's current upload guidance recommends an MP4 container, H.264 video, and 48 kHz audio, and says the player adapts to vertical aspect ratios ([YouTube Help](https://support.google.com/youtube/answer/1722171)). Recheck destination requirements just before release; the master and panel source should make new exports cheap.

An optional later format is a **tap-through evidence comic** on the project page: viewers choose the customer route, cluster, or inbox panel and see one symptom from each before the page points them to the playable scenario. It can reuse the same sanitized fixture and artwork. Keep it brief and spoiler-safe; the video and static pages are the MVP.

### 5. Review and validate

Borrow the checks that mattered in [Doom Perf's validator](../doom-perf/scripts/trailer/validate.mjs), but validate **story truth** as well as file structure:

- Every panel ID exists; durations sum to the intended runtime; selected data points map to the fixture; the event order is possible; output dimensions, frame rate, audio, and duration pass a media probe.
- A teaser scan finds no secret name, cause-bearing log line, repair command, private code, or spoiler in visible text, subtitles, filenames, thumbnails, and accessibility text.
- A human reviewer checks the causal diagram against the scenario source and the recorded run. The visual distinction between **component health** and **customer health** must be immediately clear.
- Watch the first three seconds on a phone, then the whole cut muted and at reduced size. Check caption reading time, contrast, and whether the final action is obvious. The static carousel must tell the same story without relying on motion.
- Review the exported file and its upload preview for black frames, cut-off text, bad crops, or sound that overwhelms the message. Keep a contact sheet and a release manifest with source commit and export hashes.

## MVP build order and decision gates

1. **Story proof:** choose one completed Break Team run, write and approve the eight-panel teaser script, and identify the exact snapshots/events that support each panel. If a panel cannot be grounded, rewrite it or label it illustrative.
2. **Visual proof:** make the opening contradiction, topology, and progressive failure as still panels. At phone size, someone unfamiliar with Break Team should understand “healthy-looking components, failing customers” without an explanation.
3. **Motion proof:** produce a rough 9:16 animatic with all panels, placeholder audio, and a sound-off pass. Adjust timing before polishing.
4. **Release candidate:** render the teaser MP4, carousel, loop, thumbnail, and textless master; run structural and spoiler checks; have one technical reviewer and one cold viewer review them.
5. **Launch and learn:** publish only after the playable link or announcement page exists. Compare the opening's viewer retention and the link's visits with later scenario releases, and note which panels people actually share or ask about. Use those observations to revise the reusable template.

The MVP is successful when the visual package is reproducible from its fixture and edit file, accurately depicts the incident's unusual behavior, survives a sound-off phone viewing, avoids spoiling the playable scenario, and provides a clear route into Break Team. A polished illustration without those properties is still a draft.

# Maze Trail Tracker — Product and Implementation Plan

## 1. Executive Summary

Build a mobile-first web application (PWA) that records the path a visitor walks
through an outdoor maze in the Scottish countryside and shows it in two ways:

1. **Map view** — a Strava-style top-down map with the visitor's trail drawn as a
   coloured line (red by default, user-selectable), plus live position, distance,
   and elapsed time.
2. **AR view** — the camera feed with the already-walked path painted on the ground
   as a red ribbon, so a visitor can turn around and literally see where they have
   been.

The central technical problem is **accuracy**, not drawing. Consumer phone GPS is
typically good to about 3–10 m in open sky, and worse beside tall hedges, while maze
corridors are often only 1–2 m wide. Raw GPS alone will draw a trail that cuts
through hedges and puts the visitor in the wrong corridor. The plan therefore treats
positioning as a **sensor-fusion problem constrained by the known maze geometry**:
GPS supplies absolute position, the phone's motion sensors (or, in AR mode, the AR
tracking system) supply precise relative motion, and the maze walls rule out
impossible paths. Optional QR checkpoints at junctions provide exact resets.

The recommended stack is TypeScript, Vite, MapLibre GL JS for the map, Three.js with
WebXR for AR, IndexedDB for on-device storage, and a service worker for offline use,
since rural mobile coverage is unreliable. A backend is optional for the first
release: the app can be a static site serving the maze definition and map tiles.

The single most important early step is a **field spike at the actual maze** with
several phones, to measure real GPS error and decide how much of the fusion stack is
needed (see Phase 0). Everything below is designed to be stopped early if raw GPS
proves good enough, and extended (QR anchors, native wrapper) if it proves worse.

## 2. Goals, Principles, and Non-Goals

### 2.1 Goals

- Record a visitor's path through the maze with enough fidelity to tell which
  corridor they were in, not just which part of the field.
- Show the path on a top-down map in real time, Strava-style, with a selectable trail
  colour and width.
- Show the already-walked path in AR, anchored to the ground, on supported devices.
- Work with poor or no mobile signal once the app has been loaded at the entrance.
- Require no account and no app-store install for the core experience.
- Keep location data on the device by default.

### 2.2 Product Principles

- **Don't spoil the maze.** A full top-down map of the maze makes it trivial to
  solve. By default the map shows only the corridors the visitor has walked
  ("fog of war"); the full layout is revealed on completion or by an explicit,
  operator-configurable hint mode. (This is a decision for the maze operator — see
  §4.2.)
- **Honest accuracy.** When the app is unsure where the visitor is, it shows that
  (a widening uncertainty halo, a dashed trail segment) rather than inventing a
  confident line.
- **Plausible paths only.** The displayed trail never passes through a hedge or wall.
  The raw GPS track is kept separately for debugging and export.
- **Heads-up by default.** AR is an occasional "look back" moment, not a mode to walk
  around in. The UI nudges visitors to lower the phone while moving.
- **Graceful degradation.** Every device gets the map; AR appears only where it
  works well.

### 2.3 Non-Goals for the First Release

- Social features, leaderboards, segments, or following other users.
- Multi-venue support or a general-purpose maze-authoring product (one maze, one
  operator; the data model allows more later).
- Background tracking with the screen off (not possible for a web app — see §8.4).
- Native iOS AR (a decision point after the pilot — see §9.5).
- Guaranteed sub-metre accuracy everywhere.

## 3. Success Measures and Release Gates

| Measure | Target for pilot |
|---|---|
| Corridor correctness: fraction of trail samples placed in the corridor the walker was actually in (vs. ground-truth walks) | ≥ 90% with fusion; report raw-GPS baseline alongside |
| Trail never crosses a wall in the displayed (fused) track | 100% |
| Time from opening app to first position at the entrance | ≤ 20 s |
| Map view frame rate on a mid-range Android phone | ≥ 50 fps while panning |
| AR ribbon visible within ±0.5 m of where the visitor actually walked, within one AR session | ≥ 90% of checks |
| Battery use for a 30-minute visit, map mode, screen on | ≤ 15% on a reference phone |
| Works with airplane mode on after initial load | Yes |

Release gate for a public pilot: corridor correctness target met on at least three
phone models (one iPhone, one recent Android, one older/cheaper Android) across at
least two weather conditions.

## 4. Assumptions and Decisions Required

### 4.1 Working Assumptions

- The maze is outdoors, roughly 0.5–3 hectares, with hedge or crop (maize) walls
  taller than head height. Corridors are 1–2 m wide.
- If it is a maize maze, the layout changes every season, so the maze definition must
  be versioned and replaceable.
- The operator can grant permission for a drone survey and for small signs/QR codes
  at junctions.
- Visitors include families, so some users will be children; privacy design follows
  the UK GDPR and the ICO Age Appropriate Design Code.
- There is some mobile signal or Wi-Fi at the entrance (car park / ticket hut), but
  coverage inside the maze may be patchy.

### 4.2 Decisions to Close During Discovery

| Decision | Options | Recommendation |
|---|---|---|
| Show full maze layout on the map? | Always / fog of war / operator toggle | Fog of war by default, operator toggle for hint mode |
| How is the maze geometry captured? | Drone orthophoto + manual tracing / existing aerial imagery / walked GPS survey / operator's design file | Drone orthophoto, cross-checked with the operator's cutting plan if one exists |
| Install QR checkpoints at junctions? | None / a few key junctions / every junction | A handful (5–10) at major junctions; decide after Phase 0 accuracy results |
| iOS AR | None / web "AR-lite" fallback / native wrapper | Map-only on iOS for the pilot, AR-lite as an experiment, native as a later phase |
| Upload trails to a server? | Never / opt-in / always | Device-only for the pilot; opt-in sharing link later |
| Hosting | Static host / small backend | Static host (e.g. Cloudflare Pages, Netlify) for the pilot |

## 5. Users and Journeys

### 5.1 Visitor

1. Scans a QR code on the entrance sign; the PWA loads and caches the maze package
   while signal is available. A prompt offers "Add to Home Screen".
2. Grants location permission (and on iOS, motion permission via a button tap).
3. Taps **Start**. The app shows "Finding you…" until it has a fix it trusts; the
   entrance is used as a strong prior because the visitor is known to be there.
4. Walks. The map shows a red trail growing behind a position dot, with distance,
   elapsed time, and a small accuracy indicator.
5. At any point taps **Look back (AR)**: the camera opens and the walked path appears
   as a red ribbon on the ground.
6. Optionally scans a junction QR code to "lock in" position exactly.
7. Reaches the centre / exit. The app shows a summary: total distance, time, dead
   ends visited, loops, the trail against the full maze layout, and a comparison
   with the shortest route.
8. Can save the trail on the device, export it as GPX, or share an image of it.

### 5.2 Operator

1. Commissions or performs a drone survey once per layout.
2. Uses a simple authoring page (internal, not public) to trace walls and corridor
   centrelines over the orthophoto, mark the entrance, centre, exits, and QR
   checkpoint locations.
3. Publishes a new maze package version; the app picks it up on next load.
4. Prints QR checkpoint signs generated by the authoring page.

### 5.3 Session State Model

```
idle → acquiring → tracking ⇄ degraded (low confidence / gap)
                     ↓  ⇅ ar
                  finished → saved
```

- `acquiring`: waiting for a fix with accuracy below a threshold, or a QR scan.
- `tracking`: normal operation.
- `degraded`: GPS lost, confidence low, or page was backgrounded; trail is drawn
  dashed until confidence recovers.
- `ar`: AR session active; still tracking, with better relative motion.
- `finished`: visitor tapped finish or reached a configured exit zone.

## 6. Positioning: What Can Be Used and How Good It Is

### 6.1 Options Compared

| Source | Works in browser? | Typical accuracy | Notes |
|---|---|---|---|
| GNSS via Geolocation API | Yes, all browsers | 3–10 m open sky; worse by tall hedges (multipath, partial sky) | Recent dual-frequency (L1/L5) phones do noticeably better. Browser gives the OS's fused fix, not raw GNSS. ~1 Hz updates. |
| Pedestrian dead reckoning (step detection + compass heading) | Yes (DeviceMotion/DeviceOrientation; iOS needs a permission tap) | Good short-term shape; drifts a few % of distance, heading error ±10–20° | Cheap, no infrastructure. Magnetometer can be disturbed by metal fences/gates. |
| Maze geometry constraint (map matching) | Yes (our own code) | Turns metres of GPS error into "which corridor" | The biggest single win; possible because the maze is known exactly. |
| AR visual-inertial odometry (WebXR) | Android Chrome with ARCore; not iOS Safari | Centimetre-level relative motion over tens of metres | Only while AR is open. Excellent motion model when available. |
| QR / signs at junctions | Yes (camera + JS QR decoder) | Exact, at that point | Needs operator buy-in and signage; also gives AR heading alignment if the sign's orientation is known. |
| NFC tags at junctions | Android Chrome only (Web NFC) | Exact | Not on iOS web; QR is the portable choice. |
| BLE beacons | Not practically (Web Bluetooth scanning is not generally available) | 1–3 m with dense deployment | Native app only; maintenance burden outdoors. |
| UWB anchors | No (native only) | 10–30 cm | Expensive, native only, power at every anchor. |
| External RTK GNSS receiver | No (Bluetooth serial needs native) | 1–2 cm | Great for the survey and ground-truthing; not for visitors. |
| Visual positioning services (e.g. geospatial AR APIs) | Native SDKs only | Sub-metre where coverage exists | Rural maze unlikely to be covered; hedge walls look alike. |

**Conclusion:** for a browser app, the realistic stack is
**GNSS + pedestrian dead reckoning + maze-constrained filtering**, upgraded by
**WebXR tracking when AR is open** and **QR checkpoints** for exact resets. RTK is
used only by the team, for surveying and ground truth.

### 6.2 Fusion Engine (Particle Filter on the Maze)

A particle filter is a good fit because the maze makes the probability of position
highly non-Gaussian (e.g. "either this corridor or the parallel one 2 m away") and
because wall constraints are trivial to express with particles.

Runs in a Web Worker, in a local metric frame (east/north metres from the maze
origin; see §7.2).

- **State per particle:** position (x, y), heading offset (compass bias),
  step-length scale.
- **Particle count:** ~500–2000; benchmark on the cheapest target phone.
- **Initialisation:** spread around the entrance gate (known start) or around the
  first good GPS fix, only in walkable areas.
- **Predict:**
  - Map mode: on each detected step, move each particle by its step length along
    the compass heading plus its own heading offset, with noise.
  - AR mode: use the WebXR pose delta, rotated into the map frame by the current
    AR-to-map alignment (§9.3), with much smaller noise.
  - No motion source: small random walk.
- **Wall constraint:** a particle whose movement segment crosses a wall segment gets
  weight zero. A spatial index (grid or R-tree) over wall segments keeps this fast.
- **Update:** each GPS fix reweights particles by a Gaussian likelihood with σ
  derived from the reported accuracy (inflated, since browser accuracy figures are
  optimistic near hedges). A QR scan reweights with σ ≈ 0.5 m around the sign.
- **Resample:** systematic resampling when effective sample size drops below ~50%.
  If all particles die (e.g. after a long gap), reinitialise around the latest GPS
  fix within walkable areas.
- **Output:** the mean of the dominant cluster, its spread (for the uncertainty
  halo), and a confidence flag.

### 6.3 Corridor Snapping and Trail Smoothing

For display, the fused position is also matched onto the corridor **centreline
graph** (nodes at junctions and dead ends, edges along corridors) using a hidden
Markov model / Viterbi map-matcher over a short sliding window. This:

- removes the residual wobble inside a corridor;
- guarantees a topologically valid trail (consecutive points are connected by
  walkable edges);
- fills gaps (e.g. the page was backgrounded for 40 s) with the shortest walkable
  path between the last and next confident positions, drawn dashed to show it was
  inferred.

The displayed trail = snapped, smoothed centreline path. A slight lateral offset can
be added when the visitor walks the same corridor twice so both passes are visible.

### 6.4 Data Kept per Session

- Raw GPS fixes (timestamp, lat/lon, accuracy, speed, heading if given).
- Step events and headings.
- Fused positions with confidence.
- Snapped trail (graph edge sequence + timings).
- AR session poses (local frame) and the alignment used.

Raw data is kept so the algorithms can be improved and replayed offline against
real walks (§12.2).

### 6.5 Sampling, Battery, and Wake Lock

- `watchPosition` with `enableHighAccuracy: true`, `maximumAge: 0`.
- Motion sensors at the browser's default rate (~60 Hz) but processed in the worker
  with peak detection for steps; nothing heavier on the main thread.
- Screen Wake Lock API to keep the screen on while tracking (with a visible toggle),
  because tracking stops when the screen locks.
- A "battery saver" option: dim map theme, lower map frame rate, stop rendering when
  the map is not visible.

## 7. Maze Definition and Map Data

### 7.1 Capturing the Maze

1. **Drone survey:** a small drone flies a grid at low altitude; photogrammetry
   software (e.g. OpenDroneMap) produces a georeferenced orthophoto at a few cm per
   pixel. A handful of ground control points measured with an RTK receiver (hire or
   borrow) tie it to true coordinates. Flights must follow UK CAA rules (Operator ID
   / Flyer ID, open category limits) and have the landowner's permission.
2. **Tracing:** in the authoring page (or QGIS for the first version), trace:
   - wall polygons (hedge footprints) or wall centre-lines with a width;
   - the walkable area (for particle initialisation);
   - corridor centreline graph (can be auto-derived from the walkable area by
     skeletonisation, then cleaned up by hand);
   - points of interest: entrance, centre, exits, QR checkpoints (position and the
     direction the sign faces).
3. **Validation:** automated checks — the graph is connected, the entrance reaches
   the centre, centrelines lie inside the walkable area, no wall gap is narrower than
   a set threshold unless intentional.

If a drone survey is not possible, fall back to available aerial imagery (often too
old for a seasonal maize maze) or the operator's cutting plan georeferenced against
a few RTK-measured corners.

### 7.2 Coordinate Systems

- Visitor positions arrive in WGS84 (lat/lon).
- All computation happens in a local East-North-Up metric frame centred on the maze
  origin (a flat-earth approximation is accurate to millimetres over a few hundred
  metres).
- Ordnance Survey data, if used, is in British National Grid (EPSG:27700); convert
  with proj4js at authoring time, not on the device.

### 7.3 Maze Package Format

A versioned, single download (~1–5 MB) cached by the service worker:

```
maze-package/
  manifest.json        # id, version, season, origin lat/lon, bounds, checksums
  walls.geojson        # polygons or lines + widths
  walkable.geojson     # polygon(s)
  graph.json           # nodes, edges, lengths (local metres)
  pois.geojson         # entrance, centre, exits, checkpoints (+ facing)
  basemap.pmtiles      # vector/raster tiles for the surrounding area
  ortho.pmtiles        # drone orthophoto tiles (optional, for "satellite" style)
```

PMTiles keeps tiles in a single file that a service worker can cache and serve with
range requests, which suits a small, offline-first area.

### 7.4 Basemap

- Surroundings: OpenStreetMap-derived vector tiles (e.g. Protomaps extract) or
  Ordnance Survey tiles via the OS Data Hub, clipped to the area.
- Maze itself: the drone orthophoto (looks like a satellite view, and will be far
  sharper and more current than public imagery), or a clean stylised rendering of
  walls/paths — offer both as map styles.
- Remember the spoiler rule (§2.2): when fog of war is on, the orthophoto and walls
  are masked except around walked corridors.

## 8. Map View (Strava-style)

### 8.1 Features

- Live position dot with heading cone and uncertainty halo.
- Trail line, default red (`#E53935`-ish), user-selectable from a palette including
  colour-blind-safe options and a custom picker; width slider; optional
  "time gradient" mode (colour fades from start to now, like Strava's speed/heart
  rate gradients).
- Inferred segments drawn dashed.
- Stats panel: distance, elapsed time, moving time, current pace, dead ends found,
  junctions visited.
- Follow-me / free-pan toggle; rotate-map-to-heading toggle.
- Fog of war: walked corridors revealed with a soft edge; unexplored area shaded.
- After finishing: full-route replay with a scrubber, shortest-path overlay, and
  "distance walked vs. shortest route" ratio.
- Export: GPX (raw and snapped tracks), PNG share card.

### 8.2 Rendering

- MapLibre GL JS with the maze package as sources.
- Trail as a GeoJSON line source updated incrementally (append points; throttle
  `setData` to ~2 Hz; simplify older segments).
- Fog of war as a mask layer: a polygon covering the maze minus a buffered union of
  walked corridors (computed incrementally with Turf.js or in the worker).
- High-contrast outdoor theme (bright sun) and a dark theme.

### 8.3 Settings

Stored locally: trail colour, width, gradient mode, units (metres/yards), map style,
fog-of-war on/off (if the operator allows), battery saver, haptics on junction
arrival.

### 8.4 Web Platform Limitations to Design Around

- **No background tracking:** if the screen locks or the visitor switches app, the
  browser stops delivering positions. Mitigations: wake lock, a clear "keep this
  screen open" message, and gap filling via the corridor graph (§6.3). A native
  wrapper (e.g. Capacitor) is the escape hatch if this proves unacceptable.
- **iOS motion permission:** `DeviceMotionEvent.requestPermission()` must be called
  from a user tap; build it into the Start button.
- **Compass:** use `deviceorientationabsolute` on Android and
  `webkitCompassHeading` on iOS; prompt a figure-8 calibration if readings are
  erratic.
- **HTTPS required** for geolocation, sensors, camera, and WebXR.

## 9. AR View ("Look Back")

### 9.1 Experience

The visitor taps **Look back**, points the phone around, and sees a red ribbon
lying on the ground along the path they have already walked, fading into the
distance. Hedges correctly hide parts of the trail that are behind them. Small
markers show junctions they have visited and dead ends they have already tried.

### 9.2 Platform Strategy

| Platform | Approach |
|---|---|
| Android, Chrome with ARCore | WebXR `immersive-ar` with `hit-test` (and `depth-sensing` / `dom-overlay` where available), rendered with Three.js |
| iOS Safari | No WebXR AR support. Pilot: map-only, plus an experimental "AR-lite" camera overlay (below). Later: native wrapper with ARKit, or a commercial WebXR-on-iOS bridge, chosen after the pilot |
| Anything else | Hide the AR button; map only |

**AR-lite** (iOS experiment): `getUserMedia` camera feed with a Three.js overlay
whose camera is driven by DeviceOrientation and the fused position. Without
visual tracking the ribbon will swim by a metre or more and drift with compass
error, so it is presented as a "rough direction" view, not a precise one. Ship only
if field testing shows it is more helpful than confusing.

### 9.3 Aligning the AR World with the Map

WebXR gives poses in a local frame that starts wherever the session began, with
arbitrary yaw. To draw the map-frame trail in AR we need a 2D transform (yaw +
translation; vertical comes from the detected floor).

Alignment sources, in order of preference:

1. **QR checkpoint:** scanning a sign of known position and facing gives an exact
   transform. Best quality.
2. **Trajectory matching:** as the visitor walks with AR open (or just after
   opening it, by asking them to take a few steps), fit the AR trajectory to the
   fused map trajectory with a 2D rigid least-squares (Procrustes) fit. Improves
   over ~5–15 m of walking.
3. **Compass + current fused position:** instant but rough (±10–20° yaw, ± fused
   position error). Use as the initial guess, then refine with 1 or 2.

The UI shows an alignment quality indicator and a "nudge" gesture (drag to rotate
/ shift the ribbon) as a manual fallback.

### 9.4 Rendering

- Ribbon: a flat triangle strip ~0.3 m wide, a few cm above the detected floor
  height, built from the snapped trail in map coordinates and transformed into the
  AR frame. Colour follows the map trail colour setting.
- Path walked *during* the AR session is taken directly from the AR poses (projected
  to the floor), so it is highly accurate regardless of GPS; earlier path comes from
  the map trail through the alignment transform.
- **Occlusion:** the maze walls are known, so render them as invisible depth-only
  boxes (wall footprint × hedge height). This hides trail segments behind hedges and
  is far more reliable than device depth sensing in foliage. Use WebXR depth sensing
  additionally where available for people and nearby objects.
- Distance fade and a maximum draw distance (~30 m) to keep it legible.
- Keep the AR pose stream feeding the particle filter as its motion model (§6.2),
  so AR improves the map trail too.

### 9.5 Native Option (Later Phase)

If iOS AR or background tracking prove essential, wrap the same web app in a native
shell (Capacitor) with small native plugins for ARKit/ARCore sessions and background
location. Web code for the map, fusion engine, and storage is reused as-is. This is
deliberately deferred until the web pilot shows what visitors actually use.

### 9.6 Safety

- Interstitial on first AR use: "Stop walking while using AR."
- Auto-pause AR if walking speed is sustained for several seconds, with a prompt to
  look up.
- Session timeouts to protect battery and heat.

## 10. Architecture

### 10.1 Components

```
┌────────────────────────── PWA (static) ──────────────────────────┐
│ UI shell (TS, lightweight framework e.g. Svelte or Preact)        │
│  ├─ Map view (MapLibre GL)                                        │
│  ├─ AR view (Three.js + WebXR / AR-lite)                           │
│  ├─ Session controller (state machine, §5.3)                      │
│  └─ Settings, history, export                                     │
│ Positioning worker                                                │
│  ├─ Sensor ingest (GPS, steps, heading, AR poses, QR)             │
│  ├─ Particle filter (§6.2)                                        │
│  └─ Map matcher + gap filler (§6.3)                               │
│ Storage: IndexedDB (sessions, raw logs, settings, maze package)   │
│ Service worker: offline cache for app + maze package (PMTiles)    │
└──────────────────────────────────────────────────────────────────┘
          │ (load/update only)
┌─────────▼──────────┐       ┌──────────────────────────────────┐
│ Static hosting/CDN │◄──────│ Authoring page (internal) + CI   │
│ app + maze package │       │ validation of maze package       │
└────────────────────┘       └──────────────────────────────────┘
```

No backend for the pilot. An optional small API (opt-in trail sharing, anonymous
aggregate heatmaps for the operator) can be added later.

### 10.2 Suggested Modules

```
src/
  app/            # shell, routing, state machine
  map/            # MapLibre setup, trail layer, fog of war, styles
  ar/             # WebXR session, alignment, ribbon, occluders, AR-lite
  positioning/    # worker entry, particle filter, PDR, map matcher, geometry index
  maze/           # package loader, validation, local frame transforms
  storage/        # IndexedDB schema (e.g. Dexie), export (GPX, PNG)
  sensors/        # permission flows, geolocation, motion, orientation, QR scan
  replay/         # load recorded raw logs and re-run the pipeline (dev + tests)
tools/
  authoring/      # internal maze tracing UI
  package/        # build + validate maze packages, generate QR signs
```

### 10.3 Key Libraries

- MapLibre GL JS, PMTiles, Turf.js (geometry), proj4js (authoring only)
- Three.js (WebXR rendering)
- A JS QR decoder (e.g. `jsQR` or the native `BarcodeDetector` where available)
- Dexie (IndexedDB), Workbox (service worker)
- Vitest + Playwright for tests

## 11. Privacy, Safety, and Accessibility

### 11.1 Privacy

- Location data never leaves the device unless the visitor explicitly shares it.
- No accounts, no analytics with location; only anonymous app-health metrics
  (e.g. "fix acquired in N seconds") if any.
- Clear "delete all my data" control; sessions auto-expire after a configurable
  period unless saved.
- Privacy notice written for children and adults, in line with the ICO Age
  Appropriate Design Code (high-privacy defaults, no nudges to share).
- If an operator heatmap is added later: opt-in, aggregated, coarse, and with a
  minimum visitor count per cell.

### 11.2 Physical Safety

- AR safety measures (§9.6).
- An always-visible "Help / Exit" button showing the shortest route to the nearest
  exit — useful for tired children and for emergencies, independent of the
  fog-of-war setting.
- Operator contact number available offline.

### 11.3 Accessibility

- Large touch targets usable with wet hands or gloves; high-contrast outdoor theme.
- Colour is never the only signal: trail colour is user-selectable, inferred
  segments are dashed, not just paler.
- Screen-reader labels for stats and controls; text summary of the route.
- Works one-handed in portrait.
- Step-free route information where the operator provides it.

## 12. Testing Strategy

### 12.1 Unit and Property Tests

- Geometry: local frame transforms round-trip; segment–wall intersection.
- Particle filter: never outputs a position inside a wall; converges on synthetic
  walks with injected GPS noise; recovers after a simulated gap.
- Map matcher: output is always a connected walk on the graph.
- Maze package validation catches broken graphs and missing POIs.

### 12.2 Replay Tests (Most Important)

- Record real field walks (raw GPS + sensors + AR poses) alongside an **RTK
  ground-truth track** carried by the tester.
- Store them as fixtures and re-run the full pipeline in CI, reporting corridor
  correctness and mean lateral error. Any algorithm change must not regress these
  numbers.
- Include hard cases: tall-hedge sections, parallel corridors, long dead ends,
  phone in pocket, backgrounded tab, rain.

### 12.3 Browser and Device Tests

- Playwright for UI flows with mocked geolocation/sensors.
- Manual device matrix: at least one current iPhone, one current Android with
  ARCore, one older budget Android.
- Offline: airplane mode after initial load; partial-download recovery.

### 12.4 Field Usability Tests

- Small groups including children and older visitors; observe whether the fog of
  war, AR look-back, and stats are understood without explanation.

## 13. Delivery Plan

### Phase 0 — Field Spike and Survey (about 1–2 weeks)

- Agree goals and the decisions in §4.2 with the operator.
- Drone survey and ground control points; produce the orthophoto.
- Build a throwaway logger page: records raw GPS, motion, and orientation to
  IndexedDB and exports JSON.
- Walk a set of known routes with 3+ phones and an RTK receiver for ground truth.
- Measure raw GPS error by the hedges. **Decision gate:**
  - raw GPS + snapping already meets corridor correctness → defer PDR/particle
    filter;
  - not met → build the full fusion stack;
  - fusion still not enough in some zones → plan QR checkpoints there.
- Spike WebXR on one Android phone at the maze: does floor detection work on grass
  and gravel paths? How quickly does tracking drift in a visually repetitive maze?

### Phase 1 — Map MVP (about 2–3 weeks)

- Maze package format, tracing in QGIS, packaging + validation scripts.
- PWA shell, offline caching, permission flows, wake lock.
- Positioning worker with the approach chosen at the Phase 0 gate.
- Map view: live dot, coloured trail with colour picker, stats, fog of war,
  finish summary, GPX export.
- Replay test harness with Phase 0 recordings.

### Phase 2 — AR Look-Back on Android (about 2–3 weeks)

- WebXR session, floor detection, ribbon rendering, wall occluders.
- Alignment: compass initial guess, trajectory matching, manual nudge.
- AR poses feed the particle filter.
- QR checkpoint scanning (map and AR), sign generator.
- Safety interstitial and walking auto-pause.

### Phase 3 — Pilot at the Maze (about 2 weeks on site)

- Soft launch to a subset of visitors; collect opt-in anonymous health metrics and
  feedback.
- Tune filter parameters on real data; fix device-specific issues.
- Evaluate AR-lite on iOS; decide on native wrapper.

### Phase 4 — Optional Enhancements

- Authoring web tool for the operator (replacing QGIS), with seasonal versioning.
- Native wrapper for iOS AR and background tracking.
- Opt-in shareable trail links and an operator heatmap.
- Challenges: "find all dead ends", timed runs, family ghost replay (race your
  earlier attempt shown as a second-colour trail on the map and in AR).

## 14. Acceptance Criteria for the First Public Release

- A visitor can open the app from the entrance QR code, start tracking, and see a
  red trail on the map within 20 s, and change its colour in settings.
- The displayed trail never crosses a wall and meets the corridor correctness
  target on the device matrix.
- The app works through the whole maze with no signal after initial load.
- On a supported Android phone, the AR look-back shows the walked path on the
  ground, occluded by hedges, within ±0.5 m for path walked during the session and
  visibly aligned (after QR or trajectory alignment) for earlier path.
- On unsupported devices the AR button is hidden or clearly labelled experimental.
- Location data stays on the device; the visitor can delete it in one step.
- Gaps (screen locked, lost fix) are shown as dashed inferred segments, never as
  confident straight lines through hedges.

## 15. Key Risks and Mitigations

| Risk | Impact | Mitigation |
|---|---|---|
| GPS too poor beside tall hedges | Wrong corridor, broken trust | Fusion + maze constraints; QR checkpoints; Phase 0 gate measures this first |
| Compass disturbed by metal fences/gates | PDR heading errors | Learn heading bias per particle; weight GPS more near known metal; QR resets |
| Visitors lock screens | Gaps in trail | Wake lock, clear messaging, graph-based gap filling, native escape hatch |
| No WebXR AR on iOS | Half of visitors miss AR | Map is the core product; AR-lite experiment; native wrapper as planned option |
| AR tracking struggles on repetitive hedges / low light | Ribbon drifts | Short "look back" sessions, occluders, QR re-alignment, manual nudge |
| Seasonal maize layout changes | Stale maze package | Versioned packages; repeat survey each season; validation in CI |
| Rain, cold, glare | Poor usability | High-contrast theme, large targets, short interactions |
| Map spoils the puzzle | Operator unhappy | Fog of war by default; operator controls hint mode |
| Battery drain | App dies mid-maze | Battery saver mode, AR timeouts, efficient worker, budget measured in pilot |

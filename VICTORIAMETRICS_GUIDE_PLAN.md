# VM Guide: an adaptive, three-mode VictoriaMetrics lab (plan)

**Status:** planning draft, 2026-11-24. This guide copies the approach of `~/projects/ELK-guide` and applies it to VictoriaMetrics. It covers the same three modes, the same goal-graph engine, coaching inside the real UI, randomized runs from a seeded key, and the same iximiuz rootfs publishing. It also builds in what the ELK guide's history taught. Items marked **Spike** are facts to confirm in the first days of the build. Each has a fallback, so no milestone waits on one.

Revised the same day after review:

- The MVP is eight scenarios that flex one pattern, including the classic series-count explosion.
- The Free play card is gone.
- A finished run stays open for exploring, as in the ELK guide.
- The repository name and cross-session progress are deferred.

A learner opens the lab, picks a scenario and a mode, and lands in **vmui**, the web UI that ships inside VictoriaMetrics. A coach runs inside that page. In **Demonstration** mode it works the incident while the learner watches. In **Guided** mode the learner works it with highlights and a hint ladder. In **Solo** mode the learner gets only the brief and is scored at the end. Later tracks add operator work that people do in production: fixing a scrape config, taming cardinality, and writing an alert. Each of these happens on a real vmagent and vmalert, and the lab grades it from system state.

---

## Decisions

Settled, and not to be reopened without a reason from a spike or a tester run:

- **Same product shape as the ELK guide.** One playbook per scenario and three assistance modes over one goal graph. A mode changes who acts, when help appears, and when validation is shown. It never swaps in easier data. The third mode is called **Solo** (the ELK code says `challenge`).
- **The MVP is eight scenarios that flex one investigation pattern.** Each runs in all three modes and randomizes on every run (§3). All eight run in vmui and extend one playbook template. Seven stay on the Query page. The eighth, *The series count exploded*, adds vmui's cardinality explorer. That is close to where the ELK guide's MVP ended up: seven Discover packs on `discover.json`. The first pack, *Which service is failing?*, proves the loop. The others are mostly scenario data on the same engine. Other surfaces wait until the MVP is done (§4).
- **vmui is the learner's surface.** It is VictoriaMetrics' own UI, and it already holds the query explorer, metrics explorer, cardinality explorer, top queries, query tracing, the relabel debugger, a dashboards tab, and the alerting tab (proxied from vmalert). Grafana is deferred and has a stated trigger (§4.6).
- **Open-source features only.** This mirrors the ELK guide's "Basic tier only" rule. No scenario uses Enterprise features: downsampling, multiple retentions, vmalert multitenancy (`-clusterMode`), vmbackupmanager, vmgateway, anomaly detection, or the Enterprise vmui debuggers (§4.5).
- **Single-node VictoriaMetrics plus vmagent, vmalert, and Alertmanager**, not the cluster version. Each run gets its own small **run stack** of containers, which isolates it the way Kibana Spaces did (§5.2).
- **One active run per playground.** A playground serves one learner, and a run is ready in seconds, so the ELK guide's prepared runs and concurrent runs aren't needed.
- **A finished run stays open for exploring, as in the ELK guide.** After any mode completes, the learner chooses **Keep exploring**, or closes the investigation and returns to the launcher. The run stays up until a deadline shown as a clock time, 10 minutes by default (§2.6). There is no separate Free play card.
- **A deterministic world model.** One Go function of (seed, time) produces the run's metrics. It backfills hours of history through `/api/v1/import`, then serves the same series live for vmagent to scrape. History and live data meet without a seam, and validators know the truth (§5.3).
- **The gateway taps the learner's queries.** Every query vmui sends passes through the lab gateway, which records the exact MetricsQL, time range, and result. Progress is judged from those queries and from system state. The DOM is used only for presentation and for actions the coach demonstrates (§5.4).
- **The real MetricsQL parser judges queries.** The learning service is written in Go, so it can import `github.com/VictoriaMetrics/metricsql` at the version VictoriaMetrics pins. It diagnoses learner queries with the same parser the server uses (§3.4).
- **Lab services are in Go. The coach, launcher, and config editor are TypeScript**, compiled at build time with Node kept in the build stage only and the output embedded with `go:embed`. Browser tests use Playwright in TypeScript.
- **Learners write controls in the operator tracks:** relabel rules, alert rules, recording rules, and stream aggregation configs. vmalert's own replay mode grades a learner-written alert rule against the run's history (§5.8).
- **Deploy to iximiuz early.** The ELK guide added its playground near the end. Here the playground ships with the first working coach loop (milestone M3) and stays in the test loop from then on.
- **One playground, Docker base, sized at 2 vCPU / 4 GiB** (Spike to confirm). The ELK guide needs 4 vCPU / 10 GiB (§7).
- **Versions:** the newest LTS line wherever one exists and can be used, and otherwise the latest stable release. VictoriaMetrics is the exception: its LTS lines ship only as Enterprise builds, so the lab pins open-source **v1.157.0** (§12).

---

## 1. What carries over from the ELK guide, and what changes

### 1.1 Mapping

| ELK guide | VM guide | Why it changes |
| --- | --- | --- |
| Kibana 9.5.2, pinned | VictoriaMetrics v1.157.0 vmui, pinned | Same rule: pin the UI the selectors depend on |
| nginx gateway with `sub_filter` script injection | Go gateway that injects the coach, taps queries, enforces a per-scenario API allowlist, and routes to the run stack | Routing per session and tapping queries go beyond static nginx |
| Kibana Space per run | Run stack per run: VictoriaMetrics, vmagent, vmalert, Alertmanager, and the fleet simulator on their own Docker network | Clean labels (no `run_id` on every series), clean vmalert and Alertmanager state, and cleanup by deleting containers |
| Seeded events and live faults, with Logstash ingestion lag | A world model that backfills history through `/api/v1/import` and is then scraped live | Ready in seconds, with no warm-up wait |
| KQL parser (`learning-service/engine/kql.py`) | `metricsql` Go parser | The same parser as the server |
| Discover, Dashboard, APM, Infrastructure, and Alerts adapters | vmui adapter, then the config editor and vmagent pages adapters | One SPA covers most of the curriculum |
| Prepared runs, a preparation tab, and cold-boot work | Not needed: the launcher shows a short progress view in the same tab and then navigates to vmui | VictoriaMetrics starts in about a second, while Kibana took minutes |
| Read-only learner scope | Read-only in the Read track. The operator tracks allow config changes inside the run stack, within that scenario's allowlist | Operator tasks are the point of those tracks |
| Python services | Go services and TypeScript browser code | User preference, plus the MetricsQL parser is Go |
| 4 vCPU / 10 GiB / 35 GiB disk | 2 vCPU / 4 GiB / 12 GiB disk (Spike) | The images are about 300 MB in total, against several GB for Elastic |

### 1.2 What to port, not reinvent

These ELK artifacts carry the hard-won decisions. Port their behavior and formats, and rewrite the code in Go or TypeScript:

- **Contracts:** `learning/schemas/*.schema.json` (scenario v2, playbook, command, action, rubric, and run manifest), the `extends` playbook templates, `${param.…}` / `${truth.…}` / `${var.…}` substitution, goal `accepts` / `drifts` / `hints` / `demonstration` blocks, and `learning/templates/rubrics/*.json`.
- **Coach UX** (`kibana-coach/src/ui/*`, README "Open the built-in learner experience"):
  - an incident briefing that varies its origin and age with the key and advances after 30 s or on **Begin**;
  - the **Advance** control, a 3× reading pace, and cursor travel under 900 ms;
  - a draggable panel that moves aside from the target, rings, and flyouts;
  - rings on evidence;
  - an immediate **Stop**;
  - a check-in after at least 45 s of active time on a step, offering **Keep going**, **Give me a hint**, or **Show me**, which is never scored;
  - dead-end detection with **Fix my query**;
  - the guided debrief with an unscored line for each step;
  - the Solo answer form and debrief;
  - the completion choice between **Keep exploring** and **Close investigation & return to main menu**, the folded coach docked lower-left while the learner explores, the exploration deadline, and the time's-up card (§2.6).
- **Enhanced feedback** (`ENHANCED_FEEDBACK_PLAN.md`): classify actions by their effect, the step clock, check-ins, and drift rules. Build these into the engine from day one, not later.
- **Plain language:** `learning/plain-language.json` and `learning/incident-briefings.json` formats. Follow the ELK V2 rule from the start: informal copy at a low reading level.
- **Tests:** `test_public_wording.py` (no run value leaks before the learner earns it), the scenario matrix, the live matrix with `--replay`, and `coach-audit.mjs`, which becomes a TypeScript Playwright audit.
- **iximiuz scripts:** `iximiuz/{manifest.yaml, Dockerfile, start-lab.sh, versions.sh, build-rootfs-image.sh, publish-rootfs-image.sh}`, with names changed and the fixes from §7.
- **Orphan reclaim** (`build-all.sh`): the controller removes containers and networks labelled `vmguide.run` that no live run owns.

### 1.3 Lessons from the ELK guide's history

1. **One pattern first** (the 2026-09-09 scope reset, then `NEXT_STEPS_INTERMEDIATE_PLAN.md` Part 2). The ELK MVP became one surface and one pattern. Seven randomized scenarios flex that pattern, so Guided and Solo can't be solved from memory. Here, build the first pack's loop until it feels good, then add the rest as scenario data. Don't fork any pack away from the generic engine.
2. **Measure before optimizing.** Cold-boot work in the ELK guide took several plans. Set time budgets here (§5.9) and measure them on the playground from M3.
3. **Don't give answers away** (commit `0366dd3`, "reduce the UI providing answers beforehand"). Narration must not name run values before the goal that reveals them. A wording test enforces this.
4. **Use as few clicks as possible to start** (ELK `notes.txt`). A learner clicks the scenario card, then the mode, and is investigating within 10 s.
5. **Keep images slim** (commit `e8f9e06` removed curl). Go static binaries on distroless images, and no helper images.
6. **Test on the platform early.** Platform behavior (idle limits, frames, tag caching) changed designs in other labs, so find it early.

---

## 2. The learner's experience

### 2.1 Who it is for

Engineers who are new to VictoriaMetrics and maybe new to metrics, and who will query, operate, or be on call for a VictoriaMetrics setup. No PromQL is assumed. Prometheus users should find that everything they know still works: teach PromQL-compatible forms first, and present MetricsQL extras as asides. For example, "MetricsQL lets you leave out `[5m]`, and it picks a window for you."

### 2.2 The first two minutes

1. The iximiuz playground opens on its **Terminal** tab. The **Lab** tab opens in its own browser window (`target: window`, as in the ELK guide).
2. The launcher shows the MVP's eight scenario cards in their suggested order. After the MVP, it shows one row per track. Each card shows a title, a one-line brief, an estimated time, a difficulty, and three mode buttons. The suggested next step is highlighted (§2.4).
3. The learner clicks a mode button. The card shows the scenario key (three words, editable, as in the ELK guide) and a short progress bar. The same tab then navigates to vmui.
4. The incident briefing appears: a page from vmalert, a support ticket, or a teammate's chat message. The origin and age vary with the key, and times are shown in the learner's local clock. It advances after 30 s or on **Begin**.
5. The coach panel docks on the left with the objective, progress, and **Hint** (or **Advance** in Demonstration). **Stop** is always visible.

### 2.3 The three modes

The ELK contract stays the same:

| Mode | Who acts | Guidance | Validation and feedback |
| --- | --- | --- | --- |
| Demonstration | The coach types MetricsQL, sets the time range, clicks legend entries, and opens pages | Narration before each action, a visible cursor, and rings on the evidence | Validated after each action. Ends with an evidence-and-conclusion summary. Not scored |
| Guided | The learner | The next outcome is highlighted, plus a hint ladder, drift notes, **Fix my query**, and check-ins | Each goal is validated as it happens. The debrief counts steps done without help, hints, and **Show me** uses |
| Solo | The learner | The brief only. Hints are on demand, and their effect on the score is shown | Silent observation. Scored on correctness, evidence, coverage, relevance, and efficiency, with a debrief and replay options |

The hint ladder follows the ELK guide's five rungs, adapted to MetricsQL:

1. Restate the objective.
2. Name the surface or control.
3. Explain the concept or function.
4. Show a partial template with the run values blank, for example `sum by (___) (rate(http_requests_total{code=~"___"}[5m]))`.
5. **Show me**.

The ladder resets at each new goal.

### 2.4 Three layers of scaffolding

The user asked for a scaffolded, graduated path. The plan provides it at three scales:

1. **Within a step:** the hint ladder, drift notes, and **Fix my query**.
2. **Within a scenario:** the modes. The launcher suggests Demonstration → Guided → Solo, but every mode stays open.
3. **Across the catalog:** tracks and difficulty bands (§4). Each scenario declares `skills` and `builds_on`. The launcher suggests the next scenario when its prerequisites have been practiced.

**Fading (after the MVP).** In Guided mode, a skill the learner has already completed without help, in an earlier scenario of the same session, starts at hint rung 1 with no spotlight. This removes the scaffold for what they already know and keeps it for what is new. Progress lasts for the playground session only. Carrying it across sessions is out of scope for now.

### 2.5 Authenticity rules

These are adapted from ELK `PLAN.md` §8:

- Background traffic, baseline errors, and plausible but irrelevant blips continue throughout.
- The brief never names the culprit. Alerts in the brief are aggregate ones, such as "site-wide 5xx ratio above 2%".
- The incident emerges after a baseline, and the window always contains time before the incident to compare against.
- Operator scenarios use real mechanics: real scrapes and real vmagent and vmalert error text, real config formats, and real reloads. The lab never shows fabricated error messages.
- The answer is found in the evidence, never in hidden metadata.
- Randomization changes values but not the lesson. Distractors stay bounded, so none becomes a second valid answer.

### 2.6 When a mode is complete

This works as it does in the ELK guide (`kibana-coach/src/ui/coach-panel.js` and `debrief.js`):

- **Two ways on.** When a mode completes, the coach shows **Finished** with that mode's recap: the demonstration summary, the guided debrief, or the Solo score and debrief. It then offers two choices, which the debrief repeats: **Keep exploring**, or **Close investigation & return to main menu**.
- **Keep exploring.** The coach folds to its header and docks in the lower-left corner. Nothing is automated any more, and **Stop** is hidden. On the same run, the learner can keep querying, open any vmui page, and keep editing config in operator scenarios. **Review feedback** reopens the recap, and the unfolded card closes the exploration.
- **The deadline.** The run stays up until a deadline, shown as a clock time ("It stays open until 15:06"). The deadline is `COMPLETED_RETENTION_SECONDS` after completion, 600 s by default, as in the ELK guide. Then the controller removes the run stack. The card says the investigation has closed and its data is gone, and leads back to the main menu. Review feedback still works, because the learning service keeps the debrief, not the run.
- **One active run.** A playground has one active run, so starting another scenario from the launcher ends an exploration early. The launcher says so in one line before it starts.
- Nothing done while exploring is scored.

---

## 3. The MVP: eight scenarios, one pattern

The MVP is eight scenarios. All of them:

- run in vmui: seven on the Query page, and `series-count-exploded` on the cardinality explorer and the Query page;
- extend one playbook template (`learning/templates/playbooks/vmui-query.json`);
- share the world model (§5.3);
- randomize every run from the key.

The ELK guide's MVP ended up in the same place: seven Discover packs on `discover.json`, so Guided and Solo can't be solved from memory of one or two. The launcher lists only packs marked `"mvp": true`, as the ELK guide filters on `track: "mvp"`. The MVP is done when all eight meet the ELK guide's per-scenario definition of done (§9, M4).

### 3.1 The shared pattern

| Goal | Learner outcome |
| --- | --- |
| `scope` | The time range covers the incident and some baseline |
| `find` | The right metric has been located (Explore metrics, autocomplete, or a `__name__` regex) |
| `transform` | The raw series becomes something readable: a rate, a quantile, or a count of changes |
| `break_down` | Aggregated by the label that separates the suspects (`job`, `reason`, `instance`, or `version`) |
| `narrow` | The scope inside the suspect has been found |
| `confirm` | Shown that the change is new and belongs to the suspect, by a ratio or a comparison with the past or with peers |
| `submit` | The answer, with cited queries |

Scenarios insert, override, or drop goals with the ELK `goal_inserts` and `goal_overrides` mechanism. One lesson runs through all eight. It restates the ELK guide's "busiest isn't broken" for metrics: **look for what changed, compared with its own past and with its peers, not for what is biggest.** Every scenario has a decoy that is big but steady.

### 3.2 The eight scenarios

They are listed in suggested order. Each takes 10–15 minutes.

| # | ID | Title | New skills | d |
| --- | --- | --- | --- | --- |
| 1 | `find-the-window` | When did it happen? | time range, zooming, step and resolution, the `rate` window | 1 |
| 2 | `which-service-is-failing` | Which service is failing? | selectors, `rate`, `sum by`, error ratio | 1 |
| 3 | `rejected-sign-ins` | More sign-ins are being rejected | matchers `=`, `!=`, `=~`, `!~`, and excluding the everyday case | 1 |
| 4 | `what-got-slow` | What got slow? | histograms, `le`, `histogram_quantile`, p99 versus average | 2 |
| 5 | `keeps-restarting` | A workload that keeps restarting | counter resets, `changes()`, `resets()`, gauges versus counters | 2 |
| 6 | `one-hot-instance` | One instance busier than the rest | `by (instance)`, `topk`, comparing with peers | 2 |
| 7 | `did-the-release-do-it` | Failures after a rolling deploy | `build_info`, `group_left` joins, first-failure time | 2 |
| 8 | `series-count-exploded` | The series count exploded | the cardinality explorer, series versus samples, `count by`, `scrape_series_added`, choosing a fix | 2 |

**1. When did it happen?**
- **Brief:** "Support says checkout failed for a few minutes earlier today. Check whether the metrics back that up, and find when it started and ended."
- **How the arc flexes:** it starts wide, over the last 6 h. `narrow` becomes **zoom**: tighten the range until the edges are sharp.
- **The lesson:** at a wide range, vmui's step and a `[5m]` window smear a short burst. Zooming in, or MetricsQL's `rate()` with no window (which follows the step), shows the true edges.
- **Randomized:** the service, the start (1–5 h ago), the length (3–12 min), and the status.
- **Decoy:** a smaller, slower blip at another time.
- **Answer:** the start and end, each within ±1 minute.

**2. Which service is failing?** This is the worked example in §3.3.
- **Answer:** the service, the scope (a route or an instance), and the start.

**3. More sign-ins are being rejected.**
- **Brief:** "More sign-ins are rejected than usual, but some rejections happen every day. Find the unusual reason and the client it affects."
- **Metric:** `auth_attempts_total{result, reason, client, client_version}`.
- **How the arc flexes:** everyday `bad_password` rejections are the biggest line, with a daily shape. The learner excludes them with `reason!="bad_password"`, or with `!~` for several everyday reasons. Then they break down by reason and by client.
- **Randomized:**
  - the unusual reason: `expired_token`, `invalid_client`, `mfa_timeout`, or `locked`;
  - the client (`ios`, `android`, `web`, or `partner-api`) and its version;
  - the start and the rate.
- **Decoy:** the `bad_password` wave.
- **Answer:** the reason and the client.

**4. What got slow?**
- **Brief:** "Some customers say checkout feels slow, but the dashboard's average looks fine."
- **How the arc flexes:** the tail moves and the median doesn't, so the average (`_sum / _count`) barely changes. The learner builds `histogram_quantile(0.99, sum by (le, job) (rate(http_request_duration_seconds_bucket[5m])))`, then narrows by route.
- **Randomized:** the service, the route, the p99 shift (2–6×), and the start.
- **Decoy:** a service that is slow by design (reports), high but steady.
- **Answer:** the service, the route, and p99 before and after, within 25%.

**5. A workload that keeps restarting.**
- **Brief:** "On-call noticed odd drops in one service's graphs. Find out whether something is restarting, and since when."
- **How the arc flexes:**
  - Raw counters drop to zero at each restart. `rate` and `increase` handle resets, so the learner switches to those.
  - `changes(process_start_time_seconds[1h])` counts restarts, and `resets()` confirms them.
  - Applying `rate()` to `process_start_time_seconds` is a diagnosed mistake (§3.5).
- **Randomized:**
  - the service and the instance;
  - the restart interval (8–20 min) and the first restart;
  - whether memory climbs before each restart (`process_resident_memory_bytes`).
- **Decoy:** a planned rolling restart of another service an hour earlier, with each instance restarting once.
- **Answer:** the instance, the first restart, and whether memory climbs first.

**6. One instance busier than the rest.**
- **Brief:** "A service looks healthy overall, but some of its requests are slow. Find out whether one instance is the problem."
- **How the arc flexes:**
  - The learner finds the instance with CPU by instance (`rate(process_cpu_seconds_total[5m])`), `topk`, and a comparison with the service average.
  - `confirm` asks why. Is the instance getting more traffic (`http_requests_total` by instance), or the same traffic with more CPU?
  - The cause is randomized, so either conclusion can be right.
- **Randomized:** the service, the instance, the cause, and the start.
- **Decoy:** another service where every instance is equally busy. That is a capacity issue, not an outlier.
- **Answer:** the instance and the cause.

**7. Failures after a rolling deploy.**
- **Brief:** "A rolling deploy started a while ago. Since then, some requests to one service fail. Find what the failing requests have in common, and when the first failure happened."
- **How the arc flexes:** `http_requests_total` has no version label, but `build_info{version}` does. The learner joins them, for example `sum by (version) (rate(http_requests_total{code=~"5.."}[5m]) * on (instance) group_left (version) build_info)`. The alternative route is to compare the failing instances with `build_info` directly.
- **Randomized:**
  - the service, and the old and new versions;
  - how many instances run the new version (1–3 of 4);
  - what fails, and the start.
- **Decoy:** another service deployed at the same time, which stays healthy.
- **Answer:** the version, and the first failure within ±2 minutes.

**8. The series count exploded.**
- **Brief:** "VictoriaMetrics' memory and disk use have been climbing since this morning, and queries are slower. Someone thinks one service started sending far more series. Find which metric and label are to blame, where they come from, and what the fix should be."
- **How the arc flexes:**
  - `scope` is the cardinality explorer's date, today against the previous day.
  - `find` is the metric with the biggest **growth**, read from the explorer's change against the previous day. It is not the metric with the most series.
  - `break_down` is the explorer's focus label: the label with the most unique values on that metric. Its values look like IDs.
  - `narrow` returns to the Query page to find the job and the start, with `count by (job) (<metric>)` over time or `sum by (job) (scrape_series_added)`.
  - `confirm` reads a few of the label's values, and compares the growth with total traffic, which hasn't grown.
  - The answer ends with the fix to apply, chosen from a fixed set (§3.4).
- **The lesson:**
  - A series is one unique set of labels. A label holding IDs creates a new series for every ID.
  - Biggest isn't the problem, growth is.
  - The right fix depends on whether the label's information is needed.
- **Randomized:**
  - the job, and the metric that gains the label;
  - the label: `user_id`, `request_id`, `session_id`, or a `path` holding raw IDs such as `/api/orders/81723` where a `route` template should be;
  - the start: a release, visible in `build_info`, 1–4 h ago and inside the current UTC day;
  - growth: 3,000–10,000 new series an hour, against a baseline of about 5,000.
- **Decoys:**
  - the histogram buckets, which hold the most series but are steady;
  - a newly added instance of another service, which raises that service's counts a little, with no new label.
- **Answer:**
  - the metric, the label, the job, and the start within ±10 minutes;
  - the fix. It is *drop the label* for pure IDs, or *rewrite the values to a template* for a raw path. *Raise the series limit* and *delete the data* are wrong answers, and the debrief explains why.
- **What comes next:** applying the fix, by writing the relabel rule, is the Track 2 follow-on `tame-the-series-count` (§4.2). It needs the config editor, which arrives after the MVP.

### 3.3 Worked example: *Which service is failing?*

This is the first pack, and the one that proves the loop. An aggregate error alert has fired, or support reports failures. Somewhere in a five-service system, one service started returning server errors. The learner must find:

- the service;
- the scope: one route or one instance;
- roughly when it started;
- the queries that show it.

The pack's goals, using the shared template's names, and the routes each one accepts:

| Goal | Learner outcome | Accepted routes (examples) |
| --- | --- | --- |
| `scope` | The time range covers the incident and some baseline | The time picker, a relative range, or an absolute range |
| `find` | The request counter has been located | Explore metrics, autocomplete, or `{__name__=~"http.*"}` |
| `transform` | The counter is turned into a per-second rate | `rate(...)`, `increase(...)`, MetricsQL's `rate(m)` with no window |
| `break_down` | 5xx rate broken down by service (`job`) | `sum by (job) (rate(http_requests_total{code=~"5.."}[5m]))`, or a `topk` form |
| `narrow` | Inside the culprit, the scope (route or instance) found | `by (route)` or `by (instance)` |
| `confirm` | Shown that the change is new and is the culprit's, using a ratio or a comparison with before | An error-ratio query, `offset`, or isolating the culprit series and reading the step |
| `submit` | Finding, scope, start time, conclusion, and cited queries | The answer form in the coach panel |

**Here the decoy is the busiest service.** In a typical run it has the most 5xx errors in absolute numbers, because its error ratio is steady and its traffic is high. The culprit has fewer errors, but they start at a clear step. The `break_down` goal's demonstration narration makes this point. The `confirm` goal makes the learner prove it with a ratio. This parallels the ELK guide's "busiest value first, then look past it" arc in `http-error-regression`.

#### What randomizes, and what stays fixed

**Varies per run, seeded from the scenario key:**

- **Topology**, chosen from a fixed catalog (`learning/topologies.json`). Examples are a shop (`gateway`, `orders`, `payments`, `catalog`, `auth`), a travel system (`edge`, `booking`, `pricing`, `inventory`, `accounts`), and a media system (`api`, `uploads`, `transcode`, `storage`, `users`). Each service has 2–4 instances.
- **The culprit's role** (an upstream or a dependency), and so the culprit's name.
- **Status:** 500, 502, 503, or 504.
- **Scope type:** one route or one instance. A bad version during a rolling deploy is scenario 7.
- **Start:** 12–40 minutes before the run began. **Magnitude:** an 8–35% error ratio inside the scope.
- **Distractors:** a busy service with steady errors (the decoy), an unrelated traffic spike earlier, or a short blip on another service an hour before.

**Fixed:**

- the goal arc and the `diagnosis` answer shape;
- a difficulty band, which keeps the step visible at vmui's default step and against the decoy;
- the metric families (§5.3).

The key-to-seed mechanism is the ELK guide's. The same key reproduces the run, and a new key gives fresh values. Two keys must need different queries and lead to different answers. The live matrix checks this (§11).

### 3.4 Answer and scoring

The Solo answer form lives in the coach panel. Its fields come from each scenario's `diagnosis` answer shape: a finding, a scope, a time (a start, or a start and an end), measured values, and a conclusion, each used only where the scenario needs it, plus the evidence:

- **Finding and scope:** free text, matched without case against the run's label values, such as `job`, `route`, `instance`, `reason`, `client`, or `version`. Where a scenario has a fixed set of conclusions, the form offers them as choices. For example, scenario 6 offers *more traffic* or *more CPU for the same traffic*. Scenario 8 offers the four fixes.
- **Times:** chosen on a small timeline, within the scenario's tolerance (±1 to ±3 minutes).
- **Measured values,** where asked for, within the scenario's tolerance. An example is p99 before and after in scenario 4.
- **Conclusion:** one sentence.
- **Evidence:** ticked from the learner's own query history, which the gateway tap supplies. Each cited query is re-run over the incident window. It counts as evidence only if its result supports the finding (§5.8).

Scoring keeps the ELK weights:

| Category | Weight |
| --- | ---: |
| Diagnosis correctness | 40 |
| Evidence quality | 20 |
| Goal coverage | 20 |
| Investigation relevance | 10 |
| Efficiency | 10 |

Assistance is reported separately, for example "2 hints, 1 step shown", and is never hidden inside the score. Efficiency counts weighted semantic actions against the reference route. Autocomplete requests, re-renders, and harmless exploration are free. The only waste is re-running identical queries, flipping the time range back and forth, or widening again after the signal was isolated.

### 3.5 Coaching MetricsQL with the real parser

The learning service parses every tapped query with `metricsql` and runs **diagnostics**. They power **Fix my query** in Guided mode and the debrief notes in Solo mode. The first set covers the common beginner mistakes:

| Diagnostic | Detected by | Plain-language note |
| --- | --- | --- |
| Syntax error | Parse failure and position | Where it broke and what was expected there |
| Raw counter graphed | A counter metric with no `rate` / `increase` / `irate` around it | "Counters only go up. Wrap it in `rate()` to see requests per second." |
| `rate(sum(...))` | The aggregation is inside the rollup | "Take the rate first, then sum. Summing first hides counter resets." |
| Label lost by aggregation | `sum(...)` with no `by`, while the goal needs a per-service result | "That added every service into one line. Use `sum by (job)`." |
| Regex used with `=` | A value with regex characters and the equality matcher | "`code="5.."` looks for the literal text `5..`. Use `=~` for a pattern." |
| Label that doesn't exist | The label isn't in `/api/v1/labels` for that metric | "`status` isn't a label here. The status code is in `code`." |
| Window misses the incident | The time range is disjoint from the truth window | "Your time range ends before the problem started." |
| Window too short | The rate window is less than twice the scrape interval | "`[15s]` is shorter than two scrapes, so some points have nothing to compare." |
| `le` dropped before `histogram_quantile` | `histogram_quantile` over an aggregation whose `by` lacks `le` | "`sum by (job)` removed the `le` label, so there are no buckets left. Use `sum by (le, job)`." |
| Average of a quantile | `avg` over `histogram_quantile` results from several series | "Averaging p99s doesn't give a p99. Sum the buckets first, then take the quantile." |
| `rate()` on a gauge | A rollup meant for counters over a metric the world's catalog types as a gauge | "`process_start_time_seconds` is a time, not a counter. Use `changes()` to count how often it moved." |

Dead ends (an empty result, a window that misses the incident, or the signal filtered out) are explained at once, as in the ELK guide. Wandering off is recorded silently.

---

## 4. The curriculum

The MVP launcher shows only the eight scenarios in §3.2. After the MVP, each track becomes a row in the launcher. Difficulty runs from 1 to 3. "Writes" marks scenarios where the learner writes config or rules: these are the "build the control" exercises. All scenarios follow the same three-mode contract, randomize from a key, and stay open for exploring once they are finished (§2.6).

### 4.1 Track 1: Read (query in vmui, read-only)

The track holds seven of the MVP scenarios (§3.2), numbers 1–7. Two more come after the MVP:

| ID | Title | The learner… | Skills | Why it comes after the MVP | d |
| --- | --- | --- | --- | --- | --- |
| `dashboard-triage` | Triage from a dashboard | Reads a supplied vmui dashboard (`-vmui.customDashboardsPath`) and drills into Query | panels, drilldown, time range | It uses the Dashboards page, which needs its own adapter targets | 1 |
| `same-time-yesterday` | Is this drop real? | Tells a real drop from a daily pattern | `offset 1d`, day-over-day ratio | Difficulty 3, beyond the MVP's band. It builds on scenarios 1, 2, and 7 | 3 |

### 4.2 Track 2: Collect (vmagent: scraping, relabeling, cardinality)

| ID | Title | The learner… | Skills | Writes | d |
| --- | --- | --- | --- | --- | --- |
| `target-down` | A target is down | Finds `up == 0`, reads the error on `/targets`, and fixes the scrape config. The cause is randomized: wrong port, wrong `metrics_path`, a host typo, or too short a timeout | `up`, the targets page, scrape config, reload | ✓ | 1 |
| `missing-after-relabel` | Metrics went missing after a change | Uses the relabel debugger to find the rule that drops a series, then fixes it | `metric_relabel_configs`, the relabel debugger | ✓ | 2 |
| `series-count-exploded` **(MVP, scenario 8)** | The series count exploded | Finds the metric, the ID-like label, the job, and the start with the cardinality explorer, then chooses the fix. Read-only | the cardinality explorer, `count by`, `scrape_series_added` | | 2 |
| `tame-the-series-count` | Tame the series count | Continues the same kind of incident, and applies the fix. Drops the label with `labeldrop`, or rewrites a raw path to its template with `replace`. Graded on the series count after the next scrapes, with the needed series still present | `metric_relabel_configs`, `labeldrop`, `replace`, reload | ✓ | 2 |
| `churn-every-deploy` | Churn after every deploy | Finds a label that changes on every restart, from `vm_new_timeseries_created_total` and cardinality by date | churn rate, the cardinality date picker | ✓ | 3 |
| `noisy-exporter` | Too many samples from one exporter | Finds the target with `scrape_samples_scraped`, then caps it or pre-aggregates it with stream aggregation | `sample_limit`, `-streamAggr.config` | ✓ | 3 |

### 4.3 Track 3: Alert (vmalert and Alertmanager)

| ID | Title | The learner… | Skills | Writes | d |
| --- | --- | --- | --- | --- | --- |
| `alert-triage` | Triage an alert you don't know | Uses the vmui Alerting tab: the rule, its expression, its labels, and since when. Opens its query and decides whether customers are affected | rules, states, source query | | 1 |
| `write-the-alert` | Write the alert | Writes an alert for the MVP's failure type, graded by replay: it fires within N minutes of the incident and stays silent on the baseline and the decoy | `expr`, `for`, labels, annotations | ✓ | 2 |
| `stop-the-flapping` | Stop the flapping | Calms a flapping alert with `for`, `keep_firing_for`, or a smoother expression, graded by replay: at most one firing per incident | flapping, `keep_firing_for` | ✓ | 2 |
| `alert-never-fires` | The alert that never fires | Fixes a label mismatch, a renamed job, or a missing `absent()`, graded by replay | label matching, `absent` | ✓ | 3 |
| `cheap-dashboard` | Make the expensive query cheap | Writes a recording rule and checks that it matches the original | recording rules | ✓ | 3 |
| `maintenance-silence` | Silence the maintenance window | Creates an Alertmanager silence that is exactly as wide as needed (in the UI or with `amtool`) | matchers, silences | ✓ | 2 |

### 4.4 Track 4: Run VictoriaMetrics (operating the database)

| ID | Title | The learner… | Skills | Writes | d |
| --- | --- | --- | --- | --- | --- |
| `newest-data-missing` | Where are my last 30 seconds? | Explains why recent points are missing: `-search.latencyOffset`, 30 s by default | query latency, flags | | 2 |
| `disk-forecast` | When will the disk fill? | Forecasts with `predict_linear` from a simulated production VictoriaMetrics' `vm_free_disk_space_bytes` and `vm_data_size_bytes` | self-monitoring, forecasting | | 2 |
| `slow-query` | Find the slow query | Uses Top queries and query tracing, then fixes the query with a narrower selector or a recording rule | top queries, `trace=1`, the query analyzer | ✓ | 3 |
| `backfill-the-gap` | Backfill the gap | Fills a vmagent outage's gap from an export file with `/api/v1/import`, then resets the rollup cache | import formats, `/internal/resetRollupResultCache` | ✓ | 3 |
| `remove-bad-data` | Remove the bad data | Finds junk series with `/api/v1/series` and deletes exactly them | `delete_series`, matcher precision | ✓ | 3 |

In Track 4, "the VictoriaMetrics you query with" is the real run instance. "The production instance you operate" is simulated by the world model as `job="vmsingle-prod"` wherever history is needed (the disk forecast). The slow-query, backfill, and delete scenarios act on the real instance.

### 4.5 The open-source boundary

The VictoriaMetrics Enterprise page lists these as Enterprise-only, and no scenario may need them:

- downsampling;
- multiple retentions (retention filters);
- vmalert multitenancy (`-clusterMode`);
- backup automation (vmbackupmanager);
- vmgateway;
- anomaly detection;
- per-tenant stats and query execution stats;
- the Kafka and Pub/Sub integrations;
- vmauth IP filters and mTLS routing;
- automatic TLS and mTLS;
- reading rules from object storage;
- the Enterprise vmui pages *Retention filters debug* and *Downsampling filters debug*.

Stream aggregation, relabeling, vmalert replay, and the cluster version's tenants are not on that list. A capability check, like the ELK guide's `required_capabilities`, refuses any pack that declares an Enterprise capability.

### 4.6 Deferred surfaces and their triggers

- **Grafana** (13.3 with the `victoriametrics-metrics-datasource` plugin 1.0.x). Production teams usually read dashboards in Grafana. **Trigger:** Tracks 1–3 are complete, and testers ask for "the dashboards we actually use." It means a second adapter with its own selector map, which is exactly the maintenance cost the ELK scope reset backed away from. Until then, vmui's Dashboards tab covers dashboard triage.
- **Kubernetes and the VictoriaMetrics operator** (VMServiceScrape, VMRule, and VMAgent CRDs on k3s). These are authentic in production, but they teach Kubernetes as much as VictoriaMetrics. **Trigger:** a request for a "VictoriaMetrics on Kubernetes" track. It would be a separate track with a k3s machine in the same playground, not a change to the core.
- **The cluster version** (vminsert, vmselect, vmstorage, partial responses, replication). **Trigger:** Track 4 is done and an advanced track is wanted. Adding a cluster run stack is a configuration change once the stack contract (§5.2) exists.

---

## 5. Architecture

### 5.1 Topology

```text
                        iximiuz playground: one VM (docker-01)
 ┌──────────────────────────────────────────────────────────────────────────┐
 │  lab network (docker compose: long-lived plumbing)                       │
 │   ┌───────────┐    ┌──────────────┐    ┌──────────────────┐              │
 │   │  gateway  │───▶│ learning-svc │───▶│ scenario-ctrl    │── docker.sock│
 │   │  :8080    │    │ (sessions,   │    │ (run stacks,     │              │
 │   │ launcher, │    │ goal graph,  │    │ world model,     │              │
 │   │ coach,    │    │ validators,  │    │ readiness,       │              │
 │   │ config UI,│    │ scoring)     │    │ replay grading)  │              │
 │   │ query tap │    └──────────────┘    └────────┬─────────┘              │
 │   └─────┬─────┘                                 │ creates / destroys     │
 │         │ routes by session → run               ▼                        │
 │  ┌──────┴───────────────── run-<id> network ─────────────────────────┐   │
 │  │ victoria-metrics :8428 (vmui, API; -vmalert.proxyURL → vmalert)   │   │
 │  │ vmagent :8429 (scrapes fleet aliases, remote-writes to VM)        │   │
 │  │ vmalert :8880 (rules, → Alertmanager, → VM)                       │   │
 │  │ alertmanager :9093                                                │   │
 │  │ fleet (world model; aliases orders-1, payments-2, …; :9100)       │   │
 │  └───────────────────────────────────────────────────────────────────┘   │
 │  Host ports (active run only, 127.0.0.1): 8428, 8429, 8880, 9093         │
 └──────────────────────────────────────────────────────────────────────────┘
```

All four lab services come from one Go module with `cmd/gateway`, `cmd/learning`, `cmd/controller`, and `cmd/fleet`, built into one image with four entrypoints. They run on distroless static images.

### 5.2 The run stack

When a run is created, the controller (Go, through the Docker Engine API) creates:

- a network `run-<id>`;
- `victoria-metrics:v1.157.0` with `-retentionPeriod=30d`, `-vmalert.proxyURL=http://vmalert:8880`, and `-vmui.customDashboardsPath`;
- `vmagent:v1.157.0` with the run's `scrape.yml` and `-http.pathPrefix=/vmagent`;
- `vmalert:v1.157.0` with the run's `rules/`, reached by vmui through VictoriaMetrics' `/vmalert/` proxy;
- `alertmanager:v0.35.0`;
- `fleet`, with one network alias per simulated instance.

Every container is labelled `vmguide.run=<id>`. Config files are bind-mounted from `~/lab/runs/<id>/config/`, and `~/lab/current` points at the active run, so terminal users find them easily.

- **Memory limits.** Each VictoriaMetrics container gets `--memory=768m` and `-memory.allowedBytes=512MiB`. Without these, every instance sizes its caches from host RAM.
- **Host ports.** Only the active run publishes its standard ports on `127.0.0.1`. Then `curl -X POST localhost:8429/-/reload` works in the terminal as it would in production.
- **Ending a run** stops and removes the stack and its directory. On start, the controller reclaims leftovers labelled with runs it doesn't know, as `build-all.sh` does in the ELK guide.
- **Grading stacks.** vmalert replay grading (§5.8) starts a throwaway VictoriaMetrics and vmalert on the run network, with no host ports, and removes them afterwards.

*Why per-run containers rather than resetting long-lived ones:* every component holds state. That includes VictoriaMetrics storage and caches, vmalert alert state, Alertmanager silences, stream-aggregation state, and edited configs. Destroying and recreating the stack resets all of it the same way. It also reuses the machinery the grading stacks need anyway. Startup cost is the Spike S5.

### 5.3 The world model and backfill

`fleet` is a Go process whose metrics are a **pure function of (seed, topology, overlays, t)**:

- **Base traffic:** a diurnal curve, per-service request mix, per-route latency distributions, and baseline error ratios.
- **Metric families,** typed in the world's metric catalog (which also feeds the §3.5 diagnostics):
  - `http_requests_total{job, instance, route, method, code}` (counter);
  - `http_request_duration_seconds_bucket{…, le}`, with `_sum` and `_count` (histogram);
  - `build_info{job, instance, version}` (gauge, value 1);
  - `process_start_time_seconds`, `process_cpu_seconds_total`, and `process_resident_memory_bytes`;
  - for the service in the auth role, `auth_attempts_total{job, instance, client, client_version, result, reason}` (counter).
- **Overlays:** the scenario's incident shapes. The MVP needs:
  - a short error burst;
  - an error step on (service, scope);
  - a surge in one rejection reason for one client;
  - a tail-latency shift;
  - a restart loop that resets counters and moves `process_start_time_seconds`, with or without climbing memory;
  - a hot instance, from skewed traffic or from extra CPU per request;
  - a rolling deploy that moves `build_info` instance by instance, with failures on the new version;
  - cardinality growth: one job's metric gains an ID-like label after a release, and keeps leaking the old series in its exposition, as a real client library does.

  Later tracks add churn on restart and samples blowing up.
- **Scrape faults** are served per Host header: `/metrics` returns 404, 500, a timeout, or a malformed exposition. Wrong ports and host typos come from the scrape config instead.
- **Exposition by Host header.** All instance aliases resolve to the one fleet container, and the Host header picks the instance. vmagent therefore records `instance="payments-2:9100"`, as it would in production.

**Backfill.** Before READY, the controller asks the fleet to export the past `backfill_hours` in JSON-lines import format and POSTs it to `/api/v1/import`. The default is 6 h at the scrape interval (15 s). Scenarios that compare with the previous day use 26 h at 60 s, with the last 2 h at 15 s. These are `series-count-exploded` and, later, `same-time-yesterday`. The cardinality explorer counts series per UTC day. So the materializer keeps a cardinality incident inside the current UTC day, and the previous day is always complete. Close to midnight UTC, it uses yesterday instead, and the brief and validators follow that date (Spike S10).

The export includes the series that vmagent itself would have produced for those targets: `up`, `scrape_duration_seconds`, `scrape_samples_scraped`, and `scrape_series_added`. It uses the label set the run's scrape config yields. The controller then calls `/internal/resetRollupResultCache`, which the docs require after backfilling. From that moment, vmagent's live scrapes continue the same functions.

**The seam contract,** tested in the live matrix:

- no counter resets at the seam;
- no series exists in both history and live data with different labels;
- the series count stays stable across the seam;
- backfill uses the same label pipeline as the scrape config. Relabel rules are applied with the same relabeling code, the `lib/promrelabel` package imported from the VictoriaMetrics module. This keeps label sets identical by construction. Fallback: compute the label sets with vmagent's relabel debug API.

**Truth.** The controller derives the run manifest's truth (culprit, scope, window, and expected magnitudes) from the same parameters. Readiness predicates are MetricsQL assertions checked against the run's VictoriaMetrics after import.

### 5.4 The gateway

One Go reverse proxy on `:8080`, the only port the playground exposes. Under that one origin:

| Path | Upstream | Notes |
| --- | --- | --- |
| `/` and `/launcher/` | gateway (embedded TypeScript) | Scenario cards, mode buttons, key, progress |
| `/vmui/` | run VictoriaMetrics | The HTML response gets `<script src="/coach/bootstrap.js">` before `</head>` |
| `/api/v1/*`, `/prometheus/*` | run VictoriaMetrics | **Tapped:** method, path, expr, start, end, step, status, series count, and a hash of the result. Filtered by the scenario's allowlist |
| `/vmalert/*` | run VictoriaMetrics (which proxies to vmalert) | The vmui Alerting tab |
| `/vmagent/*` | run vmagent | `/targets`, `/service-discovery`, relabel debug pages. The coach is injected into the HTML |
| `/config/` | gateway (embedded TypeScript) | The config editor (§5.7) |
| `/coach/` | gateway | Coach assets, `runtime.json`, and the WebSocket proxied to learning-svc with an Origin check |

**Routing.** A session cookie (`HttpOnly`, `SameSite=Strict`) maps to the session's run. Requests from a tab whose run has ended get an "this session has ended" page linking to the launcher, as in the ELK guide. With no session cookie, `/vmui/` and the API redirect to the launcher.

**The capability allowlist.** Each scenario declares which learner-reachable endpoints are open. The Read track gets the query, series, label, metadata, status, export, and raw-query endpoints. The operator tracks add the config editor and specific endpoints such as `/api/v1/import`, `/api/v1/admin/tsdb/delete_series`, `/internal/resetRollupResultCache`, and `/-/reload`. Everything else returns 403 with a plain-language reason. This keeps a beginner from breaking the lab by accident. It is not a security boundary, because the learner has a root terminal.

**The query tap.** Taps go to learning-svc over an internal channel. vmui issues autocomplete and metadata requests on its own. The tap marks `query` and `query_range` requests as executions, and the other endpoints as browsing. It removes repeats of the same (expr, range, step) caused by re-renders.

### 5.5 The learning service

This is a Go port of the ELK learning service's responsibilities: sessions, the goal-graph engine, mode policies, hints, drifts, the step clock, check-ins, the answer, scoring, and the debrief. It uses the ported JSON contracts (§1.2). It adds:

- **Two observation sources** merged into one ordered action stream per session. Server-side taps give `query_executed`, `cardinality_explored`, `series_listed`, `rules_viewed`, `config_validated`, `config_applied`, and `relabel_debugged`. The coach in the browser gives `time_range_changed`, `page_opened`, `series_isolated` (a legend click), `view_switched` (graph, table, or JSON), `query_traced`, and `hint_requested` / `step_demonstrated`.
- **The MetricsQL diagnostics** in §3.4.
- **The validator library** in §5.8.
- **Learning telemetry** stored as JSONL under the service's volume. It never goes into the learner's VictoriaMetrics, so it can't pollute the evidence. This matches the ELK guide keeping `tutorial-*` separate from `microservices-*`.

### 5.6 The coach (TypeScript)

The structure is the ELK coach's, written in strict TypeScript and compiled at build time:

- **`core/`:** the session client, panel, cursor, spotlight, rings, briefing, debrief, step clock, check-ins, **Advance**, and **Stop**. Its behavior is ported from `kibana-coach/src/ui/*` (§1.2).
- **`adapters/vmui/`:** resolves semantic targets on the pinned vmui. vmui is Preact with BEM-style `vm-*` class names, a hash router (`#/`, `#/metrics`, `#/cardinality`, `#/top-queries`, `#/relabeling`, `#/raw-query`, `#/rules`, `#/query-analyzer`, and others), and query state in the hash (`g0.expr`, `g1.expr`, …). It reads state from the hash first and the DOM second. It performs demonstration commands:
  - `set_time_range`, `enter_query`, `execute_query`, `add_query`;
  - `switch_view`, `isolate_series`, `open_page`;
  - `cardinality_set_match`, `cardinality_focus_label`, `cardinality_set_date`;
  - `toggle_trace`, `open_rule`, `relabel_debug_run`.
- **`adapters/vmagent-pages/`:** the server-rendered targets and service-discovery pages (`open_targets`, `expand_target_error`).
- **`adapters/config-editor/`:** `open_file`, `edit_yaml` (types a patch visibly), `validate_config`, `apply_config`.
- **The selector registry** lives at `learning/selectors/vmui-1.157.json`. A version bump runs the compatibility audit (§11) before the new selectors are accepted.
- **Evidence rings.** Rings go around legend rows, table rows, cardinality table rows, the Alerting tab's rule rows, and a time region over the uPlot chart. The region's position comes from the chart overlay's size and the visible time range (Spike S2).

### 5.7 The config editor (operator tracks)

The config editor is a TypeScript page at `/config/`, built on CodeMirror 6 with YAML mode. It shows only the files the scenario allows, such as `scrape.yml`, `relabel/*.yml`, `rules/*.yml`, and `stream-aggr.yml`. It has four controls:

- **Validate:** the controller runs a one-off container of the same image, with `vmagent -dryRun -promscrape.config=…` or `vmalert -dryRun -rule=…`, and shows vmagent's or vmalert's own messages.
- **Apply:** writes the file and POSTs `/-/reload` to the component.
- **Diff:** compares the file with the run's baseline.
- **Revert:** restores the run's baseline.

**The terminal is an equivalent route.** The same files are in `~/lab/current/config/`, and `curl -X POST localhost:8429/-/reload` works. Validators read system state, not the editor's history. So editing in vim and reloading by hand counts the same, in the way the ELK guide accepts both a KQL query and a filter pill. Demonstration mode uses the editor, because the coach can't act in the iximiuz terminal.

### 5.8 Validators

Validators are declared in scenario packs and parameterized with `${param.…}`. The library:

| Kind | Checks | Source |
| --- | --- | --- |
| `query_result` | A learner, cited, or reference expression evaluated over a window. Assertions: `top_series_label`, `contains_series`, `excludes_series`, `grouped_by`, `series_count_between`, `step_change_at(t ± tol)`, and `ratio_above` | The run's VictoriaMetrics |
| `query_shape` | Parsed MetricsQL: a rollup over a counter, `by` includes a label, a matcher on a label | `metricsql` |
| `time_range_covers` | The learner's range includes the incident and some baseline | Tap or coach |
| `target_state` | Target health and last error for a job or instance | vmagent `/api/v1/targets` |
| `series_count` | Series for a matcher are below N (a cardinality fix) and the needed series still exist | `/api/v1/status/tsdb` with `match[]`, or a `count(...)` query |
| `relabel_result` | Given input labels, the applied config yields the expected output | vmagent's relabel debug endpoint |
| `rule_state` | A rule exists, is healthy, and is in the expected state | vmalert API |
| `rule_replay` | A learner's rule file replayed over the run's history: `must_fire` windows (with a maximum delay), `must_not_fire` windows, and a maximum number of firings | A throwaway VictoriaMetrics and vmalert with `-replay.timeFrom`/`-replay.timeTo` and `-remoteWrite.url` pointing at the throwaway; the check reads its `ALERTS` series |
| `silence_matches` | A silence covers exactly the intended alerts | Alertmanager API |
| `data_present` / `data_absent` | A backfill landed, or a delete removed exactly the junk and kept everything else | The run's VictoriaMetrics |

**Validators must not pollute evidence.** Validator queries go straight to the run's VictoriaMetrics, so they appear in its Top queries and Active queries pages. Keep validator volume small. In `slow-query`, where those pages are the evidence, validators switch to tap-only mode and read `/api/v1/status/top_queries` without querying.

### 5.9 The run lifecycle and time budgets

The states are the ELK guide's: `CREATED → ACTIVATING → WARMING → READY → INVESTIGATING → COMPLETED`, with `FAILED` and `ABORTED`. Here, `ACTIVATING` creates the stack and `WARMING` runs backfill, the cache reset, the first live scrape, and the readiness predicates. After `COMPLETED`, the stack stays up for exploring until its deadline (§2.6), the learner closes it, or another run starts. Then the controller removes it. An `ABORTED` or `FAILED` run is removed at once.

| Budget | Target | Measured at |
| --- | --- | --- |
| Playground boot to lab ready (the init task finishes) | ≤ 90 s with a cold rootfs, ≤ 30 s cached | M3 |
| Mode click to READY | ≤ 10 s (6 h backfill); ≤ 20 s (26 h backfill) | M1 |
| READY to vmui with the coach connected | ≤ 2 s | M2 |
| Config Apply to a validated result | ≤ 5 s (reload, then the next scrape at a 15 s interval, or the targets API immediately) | M5 |
| `rule_replay` grading | ≤ 15 s for 6 h of history | M6 |

Readiness accounts for `-search.latencyOffset`, which defaults to 30 s. Live predicates look at least 30 s into the past, or use component APIs such as vmagent targets, which have no offset. The default stays as it is: it is authentic, and `newest-data-missing` teaches it.

---

## 6. Contracts: what's new in the v2 scenario schema

Packs keep the ELK guide's layout, `learning/scenarios/<id>/{scenario.json, playbook.json, rubric.json}`, and its playbook templates (`learning/templates/playbooks/vmui-query.json` and `config-change.json`). New or changed fields:

```jsonc
{
  "schema_version": 2,
  "id": "which-service-is-failing",
  "track": "read",
  "mvp": true,
  "type": "investigation",
  "difficulty": 1,
  "estimated_minutes": 12,
  "surface": "vmui",
  "skills": ["selectors", "rate", "sum-by", "ratio"],
  "builds_on": [],
  "required_capabilities": ["vmui.query", "open-source"],
  "learner_api_allow": ["read"],
  "parameters": {
    "topology":   {"choose_from": "topologies"},
    "culprit":    {"choose_role": ["upstream", "dependency"]},
    "status":     {"choose": [500, 502, 503, 504]},
    "scope":      {"choose": ["route", "instance"]},
    "start_ago_m":{"integer": {"min": 12, "max": 40}},
    "ratio":      {"decimal": {"min": 0.08, "max": 0.35, "step": 0.01}},
    "decoy":      {"choose": ["steady-errors", "traffic-spike", "earlier-blip"]}
  },
  "world": {
    "backfill_hours": 6,
    "scrape_interval": "15s",
    "overlays": [
      {"kind": "error_step", "target": "${param.culprit}", "scope": "${param.scope}",
       "status": "${param.status}", "ratio": "${param.ratio}", "start_ago_m": "${param.start_ago_m}"},
      {"kind": "${param.decoy}"}
    ]
  },
  "readiness": [
    {"kind": "query_result",
     "expr": "sum by (job) (rate(http_requests_total{code=~\"5..\"}[5m])) / sum by (job) (rate(http_requests_total[5m]))",
     "window": "incident", "assert": {"top_series_label": {"job": "${truth.finding}"}}}
  ],
  "truth": {"finding": "…", "scope": "…", "start": "…", "conclusion": "…"},
  "playbook": "playbook.json",
  "rubric": "rubric.json"
}
```

A preflight checks that, for every parameter choice, the readiness predicates are satisfiable and the decoy can never satisfy the truth predicates. This applies ELK `PLAN.md` §14's randomization rules.

---

## 7. iximiuz deployment

This mirrors `ELK-guide/iximiuz/`, with fixes from what the platform has taught since:

- **Manifest** (`iximiuz/manifest.yaml`):
  - `kind: playground`, `base: docker`, and one machine `docker-01` with users `root` and `laborant` (default);
  - `cpuCount: 2`, `ramSize: 4GiB`, and a 12 GiB root drive from `oci://ghcr.io/lpmi-13/vm-guide-rootfs:vN@sha256:…`;
  - tabs: **Terminal** (`kind: terminal`) first, then **Lab** (`kind: http-port`, `number: 8080`, `target: window`);
  - one `initTasks` entry that runs `start-lab.sh` as `laborant`;
  - `accessControl` open to anyone, as in the ELK guide.
- **The rootfs** (`iximiuz/Dockerfile`, `build-rootfs-image.sh`). It is built on the same `ghcr.io/iximiuz/labs/rootfs:ubuntu-docker-<release>` base as the ELK guide (`c36cb070` today, refreshed at build). It copies the tracked repository files and `images.tar` holding every lab and run-stack image. This is about 300 MB, against several GB for Elastic. Run `chmod -R go-w` on the sources before building, because a build machine's umask leaks into the image.
- **`start-lab.sh`** must be idempotent, because a failing init task retries every second, forever. It loads `images.tar` only if an image is missing, writes `.env`, runs `docker compose up --no-build -d --wait`, and lets the controller reclaim leftover runs.
- **`publish-rootfs-image.sh`.** The ELK script increments `vN`, pushes, updates `versions.sh` and the manifest, and creates or updates the playground. This version also pins the pushed **digest** in the manifest, because the platform caches images by tag across factories. The first push of a new ghcr package is private, and the user must make it public by hand. Check it with `DOCKER_CONFIG=<empty dir> docker manifest inspect`. `labctl playground create` adds an ID to the name, so keep the manifest under version control. The user runs the publish step.
- **Free-tier limits.** These are `maxPlayTime: 60m` and `maxIdleTime: 20m`, and reaching either destroys the session. Traffic to an exposed port did not count as activity in earlier measurements, but an open web terminal did. Hence the Terminal tab comes first and the Lab opens in its own window, so the playground page with its terminal stays open behind it (Spike S7). Keep every scenario's `estimated_minutes` at 20 or less. The launcher states the session limit in one line.
- **Frames.** If the Lab is ever shown as a frame instead of a window, VictoriaMetrics' `-http.header.frameOptions` stays unset, since `http-port` tabs are frames on `https://labs.iximiuz.com`.

**Resource plan** (Spike S6 confirms):

| Part | Memory |
| --- | --- |
| Lab plumbing (Go) | ~100 MiB |
| One run stack (VictoriaMetrics ≤768 MiB, others ~200 MiB) | ≤ ~1 GiB |
| One grading stack (brief) | ≤ ~600 MiB |
| Docker, the OS, and the terminal | ~800 MiB |

That totals about 2.5 GiB at peak, against 4 GiB allotted.

---

## 8. Repository layout

`VM-guide` is a working name. The repository name is decided later.

```text
VM-guide/
├── README.md, PLAN.md (this document), LOCAL_TEST_PLAN.md
├── docker-compose.yml          # gateway, learning, controller only; run stacks are created at runtime
├── build-all.sh                # build, up --wait, reclaim leftovers, print the URL
├── go.mod                      # Go 1.27; metricsql at the version VictoriaMetrics v1.157.0 pins
├── cmd/{gateway,learning,controller,fleet}/
├── internal/
│   ├── world/                  # pure (seed, t) model, overlays, export to import format
│   ├── runstack/               # Docker Engine API: create, destroy, reclaim, grading stacks
│   ├── tap/                    # query tap and action normalization
│   ├── engine/                 # goal graph, policies, hints, drifts, step clock, scoring, debrief
│   ├── mql/                    # metricsql diagnostics
│   ├── validators/
│   └── contracts/              # schema loading, materialize(), ${…} substitution
├── web/                        # TypeScript (strict), built at image build time, go:embed
│   ├── coach/{core,adapters/{vmui,vmagent-pages,config-editor},protocol}/
│   ├── launcher/
│   └── config-editor/          # CodeMirror 6
├── learning/
│   ├── catalog.json, topologies.json, incident-briefings.json, plain-language.json
│   ├── schemas/, selectors/vmui-1.157.json
│   ├── templates/{playbooks,rubrics}/
│   └── scenarios/<id>/{scenario,playbook,rubric}.json
├── runstack/                   # baseline scrape.yml, rules/, alertmanager.yml, vmui dashboards
├── e2e/                        # Go live matrix; Playwright (TypeScript) coach audit
├── iximiuz/                    # manifest, Dockerfile, start-lab, versions, build and publish scripts
└── docs/                       # pack guide, validators, selector maintenance, vmui compatibility
```

---

## 9. Milestones

Each milestone ends with its tests green, on a laptop from M1 and on the playground from M3.

- **M0: Spikes (§10).** Two to three days. Results go into `docs/spike-results.md`, with fallbacks taken where needed.
- **M1: A world you can query.** This covers the contracts and schemas ported from the ELK guide, the world model, the run stack, backfill, readiness, and the truth. It also adds the first MVP pack, `which-service-is-failing`, with its parameters. There is no coach yet. **Exit:** for 50 seeds, all of these hold:
  - the run reaches READY within budget;
  - the reference queries prove the truth;
  - the decoy's answer is graded wrong;
  - the seam tests pass;
  - two keys yield different answers.
- **M2: The coach loop for the first pack.** This covers the gateway (injection, tap, allowlist, sessions), the learning service (engine, diagnostics, scoring), and the coach core plus the vmui adapter. **Exit:** in the browser audit, all of these hold:
  - all three modes complete the pack at two viewport sizes, at normal and **Advance** speed;
  - Guided dead ends trigger **Fix my query**;
  - the Solo debrief matches the manifest;
  - **Keep exploring**, the deadline, and the time's-up card behave as in §2.6.
- **M3: The iximiuz playground.** This covers the rootfs, manifest, start script, and digest-pinned publish. **Exit:** all of these hold:
  - a cold playground boots to lab ready within budget;
  - the first pack runs in all three modes behind the platform's proxy;
  - the free-tier idle behavior for the tab layout is measured (S7).
- **M4: The rest of the MVP.** This adds the other seven packs in §3.2 order: `find-the-window`, `rejected-sign-ins`, `what-got-slow`, `keeps-restarting`, `one-hot-instance`, `did-the-release-do-it`, and `series-count-exploded`. The work is mostly scenario data and world overlays on the `vmui-query` template. The exception is `series-count-exploded`, which also adds the cardinality explorer's targets to the vmui adapter: the date, match, and focus-label controls, and the table rows to ring. A pack that needs engine changes fixes the shared layer, never a one-off path. **Exit:** all of these hold:
  - all eight packs meet the ELK "definition of done for one scenario": a pack and its tests, all parameter variants live-tested, three modes audited, and wording checked;
  - the launcher lists exactly these eight;
  - a tester run on the playground has been done.

  **M1–M4 are the MVP.**
- **M5: The operator machinery.** This covers the config editor, dry-run and reload, the terminal route, the system-state validators, and `target-down`. **Exit:** both the editor route and the terminal route are accepted, and every randomized cause is solvable.
- **M6: The VictoriaMetrics signature tasks.** This covers `tame-the-series-count`, `alert-triage`, and `write-the-alert` with replay grading. **Exit:** replay grading meets budget and passes the rule fixture set: a good rule, a late rule, one that fires on the decoy, one that never fires, and one that flaps.
- **M7: The rest of the catalog,** in track order, by difficulty, each pack at the same definition of done. Fading (§2.4) lands here.
- **Then:** decide on the triggers for Grafana, the operator, or the cluster track (§4.6).

---

## 10. Spikes

| # | Question | How to test | Fallback |
| --- | --- | --- | --- |
| S1 | Can the coach set vmui's query editor value and run it, and does an external hash change (`g0.expr`) re-run the query? | Inject a script into v1.157.0 vmui through the gateway and drive it | Drive it through hash navigation and reload. If even that fails, type with synthetic key events |
| S2 | Can a time region on the uPlot chart be ringed accurately? | Read the overlay's size and the visible range, then compare with the drawn axis ticks | Ring the legend row and say the time in words |
| S3 | Is backfill-then-scrape seamless (no reset, identical labels) using the `promrelabel` package for the label sets? | A seam test on three seeds | Compute label sets through vmagent's relabel debug API |
| S4 | Import time for 6 h and 26 h of history, and whether the rollup cache reset suffices | Time `/api/v1/import` and the first queries | Coarser steps for older history, or `-search.disableCache` in the run's VictoriaMetrics |
| S5 | Run stack create-to-ready time through the Docker API on 2 vCPU | Measure on a stock iximiuz Docker VM using `labctl cp` | Keep one warm, empty stack for the next run |
| S6 | Peak memory with a run stack plus a grading stack | `docker stats` during a replay grading | 3 vCPU / 6 GiB, still one playground |
| S7 | On the free tier, does the Terminal-first layout with the Lab in its own window survive 20 min of Lab-only use? | A free-tier test session (`--as-free-tier-user`) | Note that the free tier is limited to the Read track, or make the launcher prompt a terminal keystroke between scenarios |
| S8 | Does vmui's Alerting tab work through VictoriaMetrics' `-vmalert.proxyURL` behind the gateway? | Open the rules and notifiers pages through `:8080` | Route `/vmalert/` straight to vmalert |
| S9 | Does `vmalert -replay` against a throwaway VictoriaMetrics write `ALERTS` series that are good enough to grade timing? | Replay the fixture rules over a seeded history | Evaluate the rule's expression with `query_range` and apply the `for` logic in Go |
| S10 | Does the cardinality explorer count backfilled series on the days of their samples? Does its comparison with the previous day show the explosion? Does exposing 10,000+ leaked series every 15 s fit the budget? | Backfill 26 h with a cardinality overlay, open `#/cardinality` for today and yesterday, and watch fleet and vmagent CPU | Grade growth from `count by (__name__)` queries, and have the coach point at the explorer's totals instead of its change column. Cap growth lower |

---

## 11. Testing

Cheapest first, as in the ELK guide's `LOCAL_TEST_PLAN.md`:

- **No Docker (Go and TypeScript unit tests):**
  - pack and schema contracts; `materialize()` determinism (same key gives the same manifest, and a different key changes only allowed fields);
  - world-model properties: monotone counters, reset only where an overlay says so, and the seam;
  - MetricsQL diagnostics fixtures;
  - validator fixtures;
  - scorer fixtures: an efficient route, an alternate valid route, a detour-heavy route, a wrong conclusion, and a right conclusion with weak evidence;
  - public wording (no truth tokens in narration before the revealing goal);
  - protocol samples type-checked against `protocol.d.ts`.
- **With the stack (Go live matrix):** every pack and every parameter choice, with fixed keys:
  - readiness within budget;
  - the truth proven by reference queries;
  - the decoy graded wrong;
  - the reference answer graded right;
  - with `--replay`, a repeated key reproducing the run in a fresh stack.
  - For operator packs: apply the reference config, then a broken config, and check the validators. For alert packs: replay the rule fixture set.
- **In the browser (Playwright TypeScript audit):** for each pack, mode, and viewport, start from the launcher and let the real coach act. Check that every declared action was visibly performed, that rings and cards stay clear of controls, and that the debrief matches the manifest. Robustness cases: **Advance**, a hidden tab, reload, an ended session, a gateway restart, and the completion paths (**Keep exploring**, the exploration deadline, time's up, and **Close investigation & return to main menu**).
- **On the platform:** a cold-boot timing, the MVP in all three modes through the platform's proxy, and the free-tier idle check (S7). The browser-through-proxy checks need the user's browser, because auto mode blocks `labctl expose --public` from Claude Code.
- **Robot learner:** a headless "learner" that solves each pack with the reference queries and config through the public routes only (tap and editor). It provides the ELK reference recordings' value without a separate recorder.

---

## 12. Versions

The newest LTS line where one exists and can be used, and otherwise the latest stable release. Checked on 2026-11-24.

| Component | Version | Notes |
| --- | --- | --- |
| VictoriaMetrics single-node, vmagent, vmalert (open source) | **v1.157.0** | **Exception to the LTS rule.** The current LTS lines, v1.148.x (latest v1.148.10) and v1.136.x, are published only as `-enterprise` binaries and images, with no open-source build. The lab is open-source only, so it uses the latest stable release. Releases come every two weeks. Pin the exact version and upgrade deliberately, running the vmui compatibility audit with each bump. |
| `github.com/VictoriaMetrics/metricsql` | v0.87.6 | The version VictoriaMetrics v1.157.0's `go.mod` pins. Keep it matched to the server. |
| Alertmanager | v0.35.0 | No LTS line |
| Go | 1.27.4 | No LTS line. It is also VictoriaMetrics v1.157.0's own Go version. |
| Node.js (build stage only) | 26 LTS (26.13.1) | |
| TypeScript | 7.0.2 | `strict` |
| CodeMirror | `codemirror` 6.0.2, `@codemirror/lang-yaml` 6.1.3 | |
| Playwright (tests only) | 1.65.1 | |
| iximiuz rootfs base | `ubuntu-docker-c36cb070` | The ELK guide's release. Refresh at build time. |
| Grafana and the VictoriaMetrics datasource (deferred) | 13.3.2 and 1.0.3 | Only if the Grafana trigger fires (§4.6). Grafana has no LTS line. |

---

## 13. Risks

| Risk | Mitigation |
| --- | --- |
| vmui changes its DOM or hash state every two weeks | Pin the version, keep a selector registry per version, read state from the hash first, and require the compatibility audit before any bump |
| The coach and vmui both handle the same keys (Enter runs a query) | Use the adapter's semantic commands only, never raw key replay, plus the reconnect tests ported from the ELK guide |
| Backfilled history looks synthetic ("too smooth") | Noise, diurnal curves, per-instance skew, and plausible irrelevant blips. Tester feedback in M3 |
| The learner breaks the run stack from the terminal (for example, deletes data) | Per-run isolation makes **Reset** a fresh stack from the same key. The allowlist prevents accidents in the UI |
| Validator queries leak into Top queries | Low validator volume, and tap-only mode where those pages are evidence (§5.8) |
| Free-tier sessions are destroyed when idle | S7, the Terminal-first layout, and scenarios of 20 minutes or less |
| Scope creep toward 25+ scenarios before the loop feels good | The first pack's loop must feel good before M4. After that, a pack ships only at the per-scenario definition of done, in track order |

---

## 14. Still open

1. **First operator scenario:** this plan builds `target-down` first (M5), because it proves the operator machinery cheaply. The alternative is `tame-the-series-count`, which continues MVP scenario 8 directly, so learners would go straight from finding the problem to fixing it.

**Deferred for now:** the repository name (the plan uses `VM-guide` as a working name) and cross-session progress (§2.4).

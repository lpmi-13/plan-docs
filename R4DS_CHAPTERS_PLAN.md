# Lessons modelled on R for Data Science: plan

20 November 2026, revised the same day: the book guides what each lesson teaches, and the lab writes all of its own material. Applies to `~/projects/rstudio-guide` at `698a722` (rootfs v4).

## 1. Goal and scope

Build 29 lessons, one for each chapter of [R for Data Science (2e)](https://r4ds.hadley.nz) by Hadley Wickham, Mine Çetinkaya-Rundel and Garrett Grolemund. They follow the book's order and its six parts. Each lesson aims at the concepts its chapter teaches, and works in **Demonstration**, **Guided** and **Solo** over the same steps, script and reference results.

- **The book is the syllabus, not a source.** The lab writes its own code, data, narration, tasks and explanations. Nothing is copied or closely adapted from the book: no code, prose, datasets, figures or example files. The README credits the book as the inspiration (done).
- **Current R and current packages.** The image's R 4.6.2 and a dated package snapshot, used with current idioms. The reference bake fails on any deprecation warning, so the lab never teaches something on its way out (§3.5).
- **The lessons replace `graphical-tools` and `statistical-tools`.** The first pilots become the default run at the end of Phase 2 (§6), and the two article lessons are removed then. Their concepts come back in lesson 13 (min, max, mean, median, quantiles, summary) and lesson 27 (base R graphics).
  - `regional-revenue` stays as the Guided stub that `tests/coach_in_ide.cjs` audits, until the grader work in `R_STUDIO_WRAPPED.md` picks it up (decision 6).
- **Size.** 5–8 steps per lesson, with 1–3 statements each: about 10–15 statements per lesson, roughly 350 in all. At today's pace (25–75 s per step with reading countdowns), a Demonstration takes about 5–10 minutes.
- **Out of scope:** the book's exercises. Solo already asks the learner to do every step unaided; extra challenges written by us can come later (decision 7).

| Part | Lessons |
| --- | --- |
| Whole game | 1 Data visualization · 2 Workflow: basics · 3 Data transformation · 4 Workflow: code style · 5 Data tidying · 6 Workflow: scripts and projects · 7 Data import · 8 Workflow: getting help |
| Visualize | 9 Layers · 10 Exploratory data analysis · 11 Communication |
| Transform | 12 Logical vectors · 13 Numbers · 14 Strings · 15 Regular expressions · 16 Factors · 17 Dates and times · 18 Missing values · 19 Joins |
| Import | 20 Spreadsheets · 21 Databases · 22 Larger-than-memory data · 23 Hierarchical data · 24 Web scraping |
| Program | 25 Functions · 26 Iteration · 27 A field guide to base R |
| Communicate | 28 Quarto · 29 Quarto formats |

Topic names are plain descriptions and are kept. Chapter 22 is renamed because the lab teaches the concept with DuckDB rather than Arrow (§2.3).

## 2. Original material

### 2.1 Rules for writing a lesson

1. **Start from the chapter's concepts, not its examples.** Each lesson has a "concepts to grok" list (§5): what a learner can do afterwards. That list is the coverage test.
2. **Design the examples from scratch** on the lab's own data (§2.2). Don't follow the book's examples closely, such as the same kind of dataset with the same sequence of plots.
3. **Write in the lab's existing voice:** plain words, each explanation quoting values the learner can see on screen.
4. **The concepts list is the review checklist.** Each lesson is reviewed against its list, not against the book's text.

### 2.2 One fictional city, generated

**Recommendation:** all lessons share one set of data about a fictional city's bike-share scheme, "Larkhaven Bikes" (a placeholder name). One seeded R generator, `content/world/generate.R`, produces it.

Why one generated world:

- **Teaching features are designed in.** Duplicate keys, sentinel values, daylight-saving gaps and messy exports exist because a lesson needs them, and the generator's own checks guarantee they are there.
- **It is offline by construction.**
- **No licences to clear.**
- **Familiarity builds across the 29 lessons.** The book gets the same benefit from reusing one flights dataset.
- **Answers can't be looked up.**
- **It is ready for per-run variation.** Varying the data by run key, as in `R_STUDIO_WRAPPED.md` §4, becomes possible later.

The city is set in the `America/Toronto` time zone (changed from `America/Chicago` in Phase 1: the same daylight-saving dates, and a Canadian city fits km, °C and the lab's spelling), so the daylight-saving changes in March and November appear in the trips.

| Dataset | Shape | Teaching features | Lessons |
| --- | --- | --- | --- |
| `trips` | About 150,000 rows, one year. Trip and bike IDs, start and end time, start and end station, member type, bike type (classic or electric), distance in km, duration | Rush-hour, weekday and seasonal patterns; a weather effect; e-bikes faster. A few zero or negative durations (clock errors) and multi-day trips (lost bikes). Missing end stations. Trips that span a daylight-saving change. | 1, 3, 9–13, 17–19, 25–27 |
| `stations` | About 80 rows: ID, name, neighbourhood, latitude/longitude, docks, opening date | One duplicated ID; two renamed stations; names with inconsistent case and punctuation | 9, 14, 15, 19 |
| `weather` | Hourly for the year: temperature, rain, wind | Sensor gaps (missing hours); `-99` sentinel values | 10, 18, 19 |
| `bikes` | Bike ID (`LB-E-00123`), type, model, service date | IDs that carry structure for regular expressions | 15, 19 |
| `fares` | Price periods with start and end dates | Joins on date ranges, which need inequality conditions | 19 |
| `survey` | About 1,500 rider survey responses | Agreement scales as text, multi-select answers spread over columns, free-text comments (with emoji and accents) | 5, 14–16 |
| Monthly station exports | 12 CSV files | Inconsistent missing-value codes (`n/a`, `-`, blank), units inside values (`12.5 km`), a column that fails to parse, header quirks | 7, 26 |
| `maintenance.xlsx` | One sheet per depot | Title rows above the header, notes in cells, dates stored as Excel serials | 18, 20 |
| Trip archive | Every trip since April 2021, about 510,000 rows (changed in Phase 1 from about 2 million, so that its 2025 equals `trips`), as Parquet partitioned by year and month; `trips.csv` is its 2025 as CSV | Too large to print or load comfortably, so a query engine is the right tool | 21, 22 |
| Station status feed | Nested JSON in the shape of a GBFS feed (the open standard real bike-share schemes publish) | Lists within lists, optional fields | 23 |
| Larkhaven Bikes website | A few static HTML pages: station list, fares table, news | Repeated cards, a table, links, an element some cards lack | 24 |

- **Small files are committed** (a few MB of CSV, xlsx, JSON and HTML), so changes can be reviewed. The trip archive is generated during the runtime image build with the same fixed seed, and S6 sizes it to add under ~100 MB.
- **The generator's checks** run in CI. Example: exactly one duplicated station ID, at least one trip across each daylight-saving change, one parse failure per export.
- **Base R datasets** (`penguins`, `airquality`, `mtcars`) stay available for a one-off step where real-world texture helps (decision 1).

### 2.3 Current dependencies

The book's packages are listed only for comparison. The image is in §4.6.

| Area | The book used | The lab uses |
| --- | --- | --- |
| Charts | ggplot2 3.x, ggthemes, ggridges | ggplot2 4.x with its built-in viridis scales, patchwork, ggrepel, scales |
| Transformation | dplyr 1.1 | Current dplyr: native pipe, `.by`, `join_by()` (including `between()` and `closest()`), `pick()`, `reframe()`, `\(x)` lambdas |
| Iteration | purrr | purrr 1.x: `map()`, `list_rbind()`, `walk()` |
| Code style | styler | Posit's **Air** formatter, through RStudio's external-formatter setting, if S4 confirms it works in this build; styler otherwise |
| Import | readr, readxl, writexl, janitor | The same, current versions |
| Databases and large data | DBI, dbplyr, duckdb, arrow | DBI, dbplyr, duckdb and **duckplyr**: one engine for SQL, CSV and Parquet larger than memory |
| Hierarchical data | jsonlite, tidyr, repurrrsive | jsonlite and tidyr's `unnest_*()`/`hoist()`, on the lab's own feed |
| Web | rvest on live sites | rvest on the lab's local pages |
| Dates | lubridate | lubridate (current) |
| Models in EDA | tidymodels | Base `lm()` with broom's `augment()`: far lighter, same concept |
| Reports | Quarto, knitr, LaTeX for PDF, leaflet, Shiny | Quarto (bundled with RStudio) and the `quarto` R package; `gt` tables; PDF through Quarto's bundled **Typst** (no LaTeX); revealjs slides; Quarto **dashboards**; an htmlwidget that works offline (e.g. `reactable`) instead of map tiles from the internet; Shiny optional |
| Data packages | palmerpenguins, nycflights13, babynames, gapminder, Lahman, repurrrsive, forcats' `gss_cat` | The generated world (§2.2) |

AI assistant packages (ellmer, chattr and the like) are never installed. The lab stays free of AI assistants.

## 3. What the engine can't do yet

| # | Gap | Where it bites | Fix |
| --- | --- | --- | --- |
| 1 | **A statement is one line.** The driver types one line. The step block is searched with `line === run.code`, the echo is found with `> ${code}`, and the `partial()` hint assumes one line. | Nearly all tidyverse code: `\|>` pipelines and ggplot `+` chains | §4.4 |
| 2 | **Tibble printing depends on the Console's width.** Checks compare printed tokens with a reference captured at `width = 200`. RStudio sets `width` from the Console pane, which the docked coach narrows, and a tibble drops columns to fit. The same result therefore prints different tokens. | Every lesson that prints a data frame | §4.3 |
| 3 | **Equivalent code doesn't count.** A chart line counts only if its code matches the reference (`codeKey`). Learners write the same plot many ways: named or positional arguments, aesthetics in `ggplot()` or in the layer. | Lessons 1, 9–11, 25 | §4.3 |
| 4 | **Steps can only type and run code.** | IDE work in lessons 2 (Alt+-), 3 (native pipe option, Ctrl+Shift+M), 4 (formatter, sectioning comments), 6 (new script, Restart R, Source, `.RData` options, projects), 8 (help, reprex in the Viewer), 23 (`View()` list viewer), 28–29 (new Quarto document, Render, visual editor, formats) | §4.2, §4.4 |
| 5 | **Files written aren't checked.** | `ggsave`, `write_*`, Parquet partitions, rendered HTML, docx and PDF | §4.3 |
| 6 | **Some output changes on every run.** | `today()`, `now()`, `Sys.timezone()`, random sampling, `system.time()` | §4.2 |
| 7 | **Some steps are about a warning or message.** | A many-to-many join warning (19); parsing problems (7) | §4.2 |
| 8 | **29 lesson cards don't fit in the picker.** Today's two cards already end at 580 px at 1366×768. | All | §4.4 |
| 9 | **The coach opens a script by clicking it in Files,** where about 35 entries won't all be in view. | All | §4.4 |
| 10 | **Packages stay attached across lessons.** The reset uses the brooms, not Restart R. For example, attaching duckplyr (lessons 21–22) replaces dplyr's methods for every data frame in the session, which changes later lessons' results and printing (to confirm in S5). | Any lesson after one that attaches a package with side effects | §4.4 |
| 11 | **Provisioning re-runs every line** at each `./run.sh up`, in a throwaway container limited to 60 s and 512 MB. With 29 lessons, the DuckDB work and Quarto renders, that would take minutes. The iximiuz playground's boot-to-ready is 34 s today. | Start-up | §4.5 |

## 4. Design

### 4.1 Layout

- **`content/world/`:** the generator, its checks and the committed small datasets.
- **`content/lessons/collection.json`:** the six parts and the lesson order.
- **`scenarios/01-data-visualization/` … `scenarios/29-quarto-formats/`:** one `scenario.json` per lesson, plus lesson-only files in `project/`.
- **The run's project stays `lessons`.**
  - It holds `lessons.Rproj`, a `README.md` listing the lessons, and `data/`: the world, copied once.
  - `data/archive` links to the trip archive inside the image, `/opt/lab/world/archive`, so it isn't copied on every run.
  - Scripts are named for order: `01-data-visualization.R` … `27-base-r.R`, then `28-quarto.qmd`.
  - Lesson 29 gets its own folder (`29-formats/`). A `_quarto.yml` at the project root would make the whole project a Quarto project and change how lesson 28 renders.
- **Today `copy_project` refuses a file that two lessons both provide,** so shared data needs its own source (`content/world/`).
- **`./run.sh up`** defaults to all 29 lessons; `./run.sh up 03-data-transformation` gives one. After Phase 2, `graphical-tools` and `statistical-tools` no longer exist.

### 4.2 Scenario format additions

```json
{
  "code": "trips |>\n  filter(member_type == \"casual\") |>\n  summarize(trips = n(), median_minutes = median(duration_min), .by = start_station)",
  "alternatives": ["trips |> filter(member_type == 'casual') |> group_by(start_station) |> summarize(trips = n(), median_minutes = median(duration_min))"],
  "purpose": "count casual riders' trips and their median length at each station",
  "task": "…",
  "evidence": "value"
}
```

- **`code` may span lines.** It is formatted the way the lab's formatter (Air or styler) formats it, which is also how RStudio indents it.
- **`alternatives`:** equivalent code that must also count. The audits type these, to prove that the check accepts them.
- **Every lesson starts with a Setup step:** its `library()` calls.

**Evidence kinds:**

| `evidence` | Counts when | For |
| --- | --- | --- |
| `console` (today) | The printed tokens equal the reference | Short atomic output: `[1] 97` |
| `plot` (today) | The code matches and a new chart appears | Base graphics (lesson 27) |
| `value` (new) | A call after the mode began returned a value with the reference fingerprint (§4.3) | Tibbles, vectors, lists, models |
| `assign` (new; replaces `console` + `show: environment`) | The named object exists with the reference fingerprint | `x <- …` |
| `ggplot` (new) | A printed ggplot's built layer data and labels have the reference fingerprint, and a new chart has loaded | Lessons 1, 9–11, 13, 16, 25 |
| `attached` (new) | The package is on the search path | `library()` lines, whose start-up messages vary |
| `pattern` (new) | The Console output matches a regular expression | `today()`, `now()`, `system.time()` |
| `file` (new) | The path exists and was modified after the mode began (optionally with a fingerprint) | `ggsave`, `write_*`, rendered output |
| `ide` (new) | An IDE state holds: a preference's value, an open tab, the Viewer showing output, the active project | Lessons 2–4, 6, 8, 23, 28, 29 |

`"warning": "<regex>"` marks a statement whose warning is expected, and the explanation can point at it. Random steps use `set.seed()`.

**Action steps** do something in RStudio instead of typing code. Each has a `purpose` (Demonstration's narration), a `task` (Guided and Solo), the control that hints highlight, and `ide` or `file` evidence. Add an action only when a lesson needs it:

```json
{"action": "menu", "path": ["Session", "Restart R"]}
{"action": "option", "pane": "General", "label": "Restore .RData into workspace at startup", "value": false}
{"action": "format-document"}
{"action": "new-file", "kind": "Quarto Document"}
{"action": "render"}
{"action": "editor-mode", "to": "visual"}
{"action": "shortcut", "keys": "Ctrl+Shift+M"}
```

A step is checked only if it leaves visible state. A tip with no lasting effect, such as Ctrl+↑ for history, goes in an explanation. One exception: the coach's capture-phase listener (the keyboard shield) already sees every key, so it can notice a declared chord such as Alt+- being pressed in the editor.

### 4.3 Evidence from R, not only from the Console text

This is the change that most lessons depend on. `R_STUDIO_WRAPPED.md` §5.3 already says DOM selectors should not decide correctness. Today they do.

- **`runtime/evidence.R`**, loaded from `Rprofile.site` in RStudio sessions, records two things:
  - **Every successful top-level call**, through `addTaskCallback`: a sequence number, the time, whether the value was visible, its class and dimensions, its fingerprint, and the fingerprint of the object it assigned, if any. The callback lives in its own environment, so the Environment broom (which clears hidden objects too) doesn't remove it. Task callbacks don't run on errors, so errors still come from the Console, as now.
  - **A small state record**, refreshed every second by `later`: attached packages, selected RStudio preferences (from `rstudio-prefs.json`), the working directory and active project, and the modification times of the lesson's declared output files.
- **The record is a ring buffer** of the last ~100 calls. It is written atomically to a run-scoped mount outside the project (`/home/rstudio/.lab/`), so it doesn't appear in Files.
- **The gateway serves it** at `/coach/api/evidence`, uncached, only to a browser signed in to RStudio (`auth_request`; S2 finds an RStudio endpoint that answers 2xx only when signed in). It holds fingerprints, classes, dimensions and names: no data values and no learner code.

**The fingerprint** normalises a value, then hashes it with `rlang::hash()`:

- **Data frames** become a plain list of columns. Attributes are dropped except class (Date, POSIXct with its time zone, factor levels), and grouping variables are kept. readr's `spec` and `problems` attributes are dropped; `problems` is an external pointer whose hash changes every session.
- **Numbers** are rounded to 10 significant digits, so integer and double agree.
- **Row order counts** unless the step says `"order": "any"`.
- **ggplots** are fingerprinted through `ggplot_build()`'s layer data, labels and facet spec. A patchwork is fingerprinted plot by plot.
- **Values over a size cap** are fingerprinted by shape only, so the Console stays responsive.

**The same `evidence.R` runs in the reference bake (§4.5),** so reference fingerprints come from the same code in the same image.

**How the coach uses it:**

- **Guided and Solo** count a statement when any call after the mode began produced its fingerprint. Any valid way of writing it counts, which `R_STUDIO_WRAPPED.md` asks for ("equivalent solutions are welcome").
- **Demonstration** waits for the same fingerprint before explaining. The Console and Plots spotlight stay as presentation.
- **Values quoted in explanations** come from the bake. A matching fingerprint means the learner's value is the reference value, and the narration still points at where it shows on screen.
- **Answers stay hidden.** The browser gets hashes instead of expected output, a step toward lessons with private answers.

### 4.4 Coach changes

- **Multi-line statements.**
  - The coach types the statement and lets RStudio auto-indent, then corrects to the reference's indentation. If S1 shows auto-pairing and auto-indent can't be reconciled, the fallback is to switch Ace's behaviours off while the coach types and restore them afterwards. The learner sees the same typing either way.
  - A statement is found in its step block by `codeKey` across lines, and its echo through `consoleEntries` (which already joins `+` continuation lines).
  - Run is pressed on the statement's first line; RStudio runs the whole statement (to confirm in S1).
  - The spotlight covers all of the statement's rows. A 200-character pipeline takes about 14 s to type at 70 ms/key.
  - Hint 2 becomes the first line plus `…` (`trips |>` `filter(…)`).
- **Action driver.** It reuses what the brooms already do (dialogs found by title, buttons by label) and adds menus, Global Options panes, Code › Reformat, New File, Render and the Viewer, the visual/source toggle, and the data viewer tab. Shortcuts are sent as synthetic key events if GWT accepts them (S4). If it doesn't, the coach uses the equivalent menu item and names the shortcut in the narration.
- **Files pane.** The script's row is scrolled into view before the pointer moves, as a learner would scroll.
- **Picker.**
  - A one-line lesson row ("Lesson 3 · Data transformation · Change") sits above the three mode cards, so the mode cards stay in view at 1366×768.
  - **Change** opens the six parts as collapsible groups. Each lesson row shows its step count, with a tick if any mode was finished.
  - The picker suggests the next unfinished lesson, so starting it takes one click.
  - The debrief offers "Next: lesson 4".
- **Progress.** In-progress state works as today: choosing a mode forgets every other mode's progress. A separate per-run record of finished lesson–mode pairs (`localStorage`) keeps the ticks.
- **Reset on a lesson switch.**
  - Switching lesson adds **Restart R** before today's script reset and brooms, so the previous lesson's packages and method overrides go.
  - RStudio holds restarts back in a session's first ~30 s. A lesson switch is rarely that early, but if it is, the coach waits and says why.
  - Staying in the same lesson keeps today's reset.
  - `file` evidence compares modification times with the moment the mode began, so earlier outputs don't need deleting.
- **The two article lessons' special cases** in the coach and in the audits go when those lessons do.

### 4.5 Bake references at build time, not at every start

- **`tools/bake.py`** runs each lesson in a fresh R process, in a throwaway offline container from the runtime image.
  - It records output, fingerprints (via `evidence.R`), plots, warnings and files written, and renders the Quarto steps.
  - It runs everything twice. A statement whose fingerprint differs between the runs needs a seed or `pattern` evidence.
  - Lessons run in parallel. Limits are set per lesson: renders and DuckDB work need more than 60 s and 512 MB.
- **It fails on** an error, any warning not declared with `"warning"` (deprecations included), a capture that doesn't match, a wrong plot/no-plot behaviour, a network attempt, or non-determinism.
- **Output** goes to `content/lessons/baked/<lesson>.json`, stamped with the image digest, the package lock's hash and the world data's hash. It is committed, so a package bump shows up as a diff that can be reviewed.
- **`provision.py`** uses the baked results when the stamps match and re-bakes otherwise. On iximiuz the baked files are in the rootfs, so start-up does no reference runs.

### 4.6 Runtime image

- **Base:** `rocker/tidyverse:4.6.2@sha256:9241ac461c924334777dd3fb047c6e5e0fd1435a85acc171f1a32836861d0a69`, published on 16 November 2026 alongside the `rocker/rstudio:4.6.2` the lab pins.
  - It brings tidyverse, devtools/usethis, rmarkdown, DBI, dbplyr and duckdb. It also includes arrow, which the lessons won't use.
  - It is 214 MB larger compressed (1,096 vs 882 MB, amd64).
  - S6 compares this with installing only what the lessons need onto `rocker/rstudio`.
- **Added from a dated Posit Package Manager snapshot:** duckplyr, patchwork, ggrepel, janitor, writexl, gt, reactable, later, styler (the fallback formatter), and the `quarto` R package.
  - Versions are recorded in `runtime/packages.lock`, and the build fails if they drift.
- **Air:** a pinned release binary (0.13.0) with its checksum. RStudio's built-in `use_air_formatter` uses an `air` on the `PATH` before downloading one, so it works offline (S4).
- **The trip archive** is generated in a build step (§2.2).
- **Quarto:** check the bundled Quarto CLI version, that dashboards work, and that Typst renders PDF.
- **Checks after the base change:** rerun the AI-assistant lockout probes and a trivy scan, and recommend current versions per the security-audit convention.
- **Measure the rootfs.** v2 was 1.66 GB and took about 2 minutes to fetch before the VM booted. If start-up suffers, the README's fallback applies: pull the runtime image from GHCR during init, on the cached base.

### 4.7 How each kind of lesson uses the three modes

| Kind | Lessons | Demonstration | Guided | Solo |
| --- | --- | --- | --- | --- |
| Code that prints | 2, 3, 5, 7, 12–27 | Types each pipeline, runs it, and spotlights the printed tibble or the Environment row | The learner writes each statement. Three hints: where it goes, how it starts, the whole statement. Show me types and runs it. | The task list. Check my work compares fingerprints. |
| Charts | 1, 9, 10, 11 | Types the ggplot code and spotlights the Plots pane | Any code that builds the same plot counts | Same |
| IDE workflow | 2, 3, 4, 6, 8, 23 (parts) | The pointer drives menus and dialogs | Hints highlight the control; Show me does it | Checks the resulting IDE state |
| Quarto | 28, 29 | Types into the `.qmd` in the source editor, presses Render and spotlights the Viewer. It switches to the visual editor to show it, but doesn't type there. | The learner edits and renders; `file` evidence | Same |

Lesson 6's "create a project" switches RStudio to a new session and reloads the page. It goes last in lesson 6 and is optional, pending S7. The steps before it stay in the lab project.

## 5. Lesson by lesson

"Concepts to grok" comes from what the chapter teaches. The draft steps are the lab's own, on the lab's data. Every lesson also starts with a Setup step (`attached`).

| # | Lesson | Concepts to grok | Draft steps (lab data) | Needs |
| --- | --- | --- | --- | --- |
| 1 | Data visualization | A plot is data + aesthetic mappings + geoms. Mapping vs setting. Global vs layer aesthetics. Distributions of categorical and numerical variables. Relationships between variables. Facets. Saving a plot. | `glimpse` one week of trips; distance vs duration scatter; colour by bike type, plus a smooth (e-bikes run faster); labels and a colour-blind-safe scale; bar chart of trips by member type, histogram of duration (`binwidth`), density; duration by bike type as a boxplot, then faceted by member type; `ggsave` | `ggplot`, `file` |
| 2 | Workflow: basics | R as a calculator; `<-` and the Environment; vectors and vectorised arithmetic; comments; naming rules; calling functions with named arguments; reading an error message | Fare arithmetic; store a fare; a vector of trip minutes times the per-minute price; a comment; a typo in a name (Guided quotes R's error back); `seq()` with named arguments | `assign`, `value`, chord check for Alt+- |
| 3 | Data transformation | Verbs for rows (`filter`, `arrange`, `distinct`), columns (`mutate`, `select`, `rename`, `relocate`) and groups (`summarize` with `.by`, `count`, `slice_*`); the pipe; small groups give unreliable averages | Rush-hour trips; longest trips first; distinct routes; `speed_kmh`; pick and reorder columns; Ctrl+Shift+M for the pipe (the native pipe is already RStudio's default); busiest start stations; trips and median length by hour; mean speed per station, then only stations with enough trips | `value`, multi-line, chord check for Ctrl+Shift+M |
| 4 | Workflow: code style | Consistent names and spacing; one pipe step per line; automatic formatting; sectioning comments and the outline | A messy script of our own; format it with Code › Reformat Document (Air, built into RStudio); rename badly named objects; sectioning comments and jumping between sections | `format-document`, script-text evidence |
| 5 | Data tidying | The three rules of tidy data; lengthening, including column names that hold two variables or a `.value`; widening | Survey multi-select columns to long; monthly station counts (one column per month) to long; `am_trips`/`pm_trips` split with `names_sep`; widen a summary into a report table | `value` |
| 6 | Workflow: scripts and projects | Scripts over Console; saving; running a whole script; Restart R to prove the script stands alone; relative paths and projects; not saving `.RData` | New script; a three-line analysis; save; Restart R, then Source; write to `outputs/` with a relative path; Global Options `.RData` settings; new project (optional, S7) | Mostly actions; `file`, `ide` |
| 7 | Data import | Reading delimited files; missing-value codes; cleaning column names; column types and parsing problems; reading many files into one with an ID column; CSV vs RDS vs Parquet; entering small data by hand | Read one monthly export; `na = c("n/a", "-", "")`; `clean_names`; `parse_number` on `12.5 km`; `col_types` and `problems()`; all 12 files with `id`; write CSV and Parquet and compare the types read back; `tribble` | `value`, `file`, `warning` |
| 8 | Workflow: getting help | Help pages and examples; reading an error; minimal reproducible examples; sharing small data; recording versions | `?` and the Help pane; a failing snippet cut down to a reprex rendered in the Viewer; `dput()` a few rows; `sessionInfo()` | `ide` (Help, Viewer), `pattern`; S4 checks reprex on RStudio Server without a clipboard |
| 9 | Layers | Mapping colour, shape, size and alpha; geoms and grouping; facets (wrap, grid); stats (`after_stat`); positions (stack, fill, dodge, jitter); coordinate systems; the layered grammar | Stations by latitude and longitude with `coord_quickmap()`, sized by docks; smooths grouped by bike type; `facet_grid` by month and member type; share of trips via `after_stat(prop)`; bike type by hour stacked, filled and dodged; seeded jitter | `ggplot`; seed |
| 10 | Exploratory data analysis | Questions drive exploration; variation and unusual values; replace an implausible value with NA, don't drop it silently; covariation (categorical–numerical, two categoricals, two numericals); models reveal patterns through residuals | Duration histogram; zoom with `coord_cartesian` to find the negative and multi-day trips; `if_else` to NA; daily trips vs temperature; duration by weekday, reordered; hour × weekday tile; `lm()` on log scale and `augment()` residuals | `ggplot`, `value` |
| 11 | Communication | Titles, subtitles, captions and axis labels; annotations; scales (breaks, labels, legends, colour); zooming; themes; combining plots | One "trips by hour" chart polished step by step: `labs`; label the peaks (`ggrepel`); `scales::label_*` axes; legend placement; a theme; two plots side by side with patchwork | `ggplot` (patchwork-aware) |
| 12 | Logical vectors | Comparisons; floating-point equality; Boolean algebra and `%in%`; NA in logic; logical summaries; conditional transformations | Rush-hour and round-trip flags; `near()` on distances; what NA end stations do to `&` and `\|`; share of e-bike trips with `mean()`; `case_when` trip categories | `value` |
| 13 | Numbers | Parsing numbers; counts; arithmetic and recycling; rounding and cutting; cumulative and rolling values; ranks; offsets; summaries of centre, spread and position | Counts by station (`sort = TRUE`); 15-minute slots with `%/%`; `cut` durations into bands; cumulative trips per day; rank stations; time between a bike's rides with `lag()`; min, max, mean, median, quantiles and `summary()` of duration (what `statistical-tools` taught) | `value`, `ggplot` |
| 14 | Strings | Creating strings, escapes and raw strings; combining (`str_c`, `str_glue`, `str_flatten`); splitting into rows and columns; length and substrings; case and locale | A station label with `str_glue`; stations per neighbourhood with `str_flatten`; split addresses into columns; multi-select answers into rows; `str_sub` on bike IDs; case and accents in survey comments | `value` |
| 15 | Regular expressions | Literals, `.`, classes, quantifiers, anchors, alternation, groups; detect, count, replace, extract; escaping; `fixed()` and case-insensitivity | `str_view` on bike IDs; e-bikes with `^LB-E-`; extract the number; tidy inconsistent station names with `str_replace_all`; pull promo codes out of comments with `separate_wider_regex`; `regex(ignore_case = TRUE)` | `value` |
| 16 | Factors | Levels control order; reordering for plots (`fct_infreq`, `fct_reorder`); changing levels (`fct_recode`, `fct_collapse`, `fct_lump_n`); ordered factors | Survey agreement scale as a factor with explicit levels; bar chart in natural order; neighbourhoods reordered by median duration; lump small neighbourhoods; collapse answers into agree/neutral/disagree | `value`, `ggplot` |
| 17 | Dates and times | Parsing; building date-times from parts; components; rounding; durations vs periods vs intervals; time zones and daylight saving | `ymd_hms` with `tz`; `make_datetime` from an export's parts; trips by `wday` and `hour`; `floor_date` to weeks; a trip across the March change, as a duration vs a period; `with_tz` vs `force_tz`; `today()` | `value`, `pattern` |
| 18 | Missing values | Explicit NAs (`fill`, `coalesce`, `na_if` for sentinels); NaN; implicit missing values (`complete`, `anti_join`); empty groups | Fill depot names down the maintenance log; `na_if(-99)` in weather; `complete()` the hours to expose sensor gaps; stations with no trips via `anti_join`; counts that keep empty factor levels | `value` |
| 19 | Joins | Primary and foreign keys, and checking them; mutating joins; `join_by` with different names; filtering joins; row counts that change (duplicate keys, many-to-many); inequality and rolling joins | Find the duplicated station ID; `left_join` trips to stations; `join_by(start_station == station_id)`; `semi_join`/`anti_join`; weather by hour; fares by date range with `join_by(between(…))` and `closest()` | `value`, `warning` |
| 20 | Spreadsheets | Sheets; skipping title rows; ranges; column types; missing values; combining sheets; writing a workbook | `excel_sheets`; one depot with `skip` and `na`; `col_types`; dates from Excel serials; all depots with `map` and `list_rbind`; `write_xlsx` | `value`, `file` |
| 21 | Databases | Connecting with DBI; listing and reading tables; SQL; dbplyr lazy tables; `show_query`; `collect` | `dbConnect(duckdb(), "data/larkhaven.duckdb")`; `dbListTables`; a SQL query; `tbl()` with dplyr; `show_query`; `collect`; disconnect | `value` |
| 22 | Larger-than-memory data | Data too big for memory; columnar files and partitioning; let the engine do the work and collect only the result | Size of the CSV vs the Parquet partitions; open the archive with duckplyr; query one year (only its partitions are read); summarize, then `collect`; write a new partitioned summary | `value`, `pattern` (timings), `file` |
| 23 | Hierarchical data | Lists and `str`; `[[` and `$`; list-columns; `unnest_wider`, `unnest_longer`, `hoist`; JSON | `read_json` the status feed; `str` and the `View()` list viewer; a tibble with a list-column; unnest stations into rows, then fields into columns; `hoist` two nested fields | `value`, `ide` (data viewer tab) |
| 24 | Web scraping | HTML elements and attributes; CSS selectors; text, attributes and tables; one row per card even when a field is missing; when scraping is acceptable (narration) | `read_html` the local station list; `html_elements` the cards; `html_text2` and `html_attr("href")`; `html_table` the fares; `html_element` per card so a missing field gives NA | `value` |
| 25 | Functions | When to write a function; vector functions; summary functions; data frame functions with `{{ }}`; plot functions; naming | `km_to_miles()`; a rounding helper; `summarize_trips(df, group)` with `{{ }}`; a histogram function with a generated title; each one called right after it is defined | `assign` for the definition, then the call's `value` proves it works |
| 26 | Iteration | `across()` with functions and names; `if_any`/`if_all`; reading many files with `map` and `list_rbind`; saving many outputs with `walk` | Summarize all numeric weather columns with `across`; rows with any NA; read the 12 exports with `map`; one CSV per neighbourhood with `group_split` and `walk2` | `value`, `file` |
| 27 | A field guide to base R | `[` on vectors and data frames; `$` and `[[`; `lapply`/`vapply`; `for` loops; base graphics | Subset a vector of durations; `df[rows, cols]`; `$` and `[[`; `vapply` over columns; a `for` loop over files; `plot()`, `hist()` and `barplot()` (what `graphical-tools` taught) | `value`, `plot` (base) |
| 28 | Quarto | Prose, code and output in one document; rendering; visual vs source editing; chunks and their options; figures; tables; caching; the YAML header | New Quarto document (a monthly ridership report); a heading and a chunk; Render; `echo`/`label` options; figure size; a `gt` table; title, author and date in YAML | Actions, `file`, `ide` |
| 29 | Quarto formats | One source, many outputs: HTML options, Word, PDF; slides; interactivity; dashboards; websites | `toc` and `code-fold`; docx; PDF via Typst; revealjs slides; a `reactable` table; a Quarto dashboard; a two-page website in `29-formats/` | Actions, `file`. The Build pane appears only in a Quarto project, so the website step renders with `quarto::quarto_render()` or the Terminal (S4 decides). |

## 6. Phases

### Phase 0: spikes

Each spike ends in a dated decision in the README, as the first spike did.

| Spike | Check | Pass condition / fallback |
| --- | --- | --- |
| **S1** Multi-line typing | Type three statements: a `\|>` pipeline, a ggplot `+` chain with arguments across lines, and a `tribble`. Run each from its first line with no selection. | Text exact under RStudio's auto-indent and auto-pairing; the whole statement runs; the echo parses. Fallback: Ace behaviours off while the coach types. |
| **S2** Evidence channel | Task callback plus `later` loop, the file, and the gateway endpoint behind sign-in. Fingerprint a pipeline result, a `read_csv` tibble, a grouped tibble, Date/POSIXct columns and a ggplot, both in the bake and live. | Identical fingerprints; under 300 ms from Run to evidence; under 50 ms per callback on 150,000 trips; anonymous requests refused |
| **S3** Console width | Print `trips` in the docked layout at 1366×768 and compare it with the reference | Documents gap #2; no fix needed if S2 passes |
| **S4** IDE actions | Session › Restart R; a Global Options checkbox; Air as RStudio's external formatter, then Code › Reformat; New File › Quarto Document; Render and the Viewer; visual toggle; synthetic Ctrl+Shift+M and Alt+-; detecting the `View()` tab; reprex on RStudio Server without a clipboard | Each works headless with readable evidence. Any that doesn't becomes narration only. |
| **S5** Lesson switch | Restart R more than 40 s into a session, then the script reset; whether attaching duckplyr changes later dplyr results, and whether a restart clears that | Under 3 s; no state carried over |
| **S6** Image and world | Base image choice; every lesson's `library()` line offline; Quarto version, dashboards and Typst PDF; archive size and generation time; trivy | All load offline; sizes recorded |
| **S7** Lesson 6 projects | Create a project from the New Project dialog, then return to `lessons` | The coach survives the reload. If not, the step is narrated, not done. |

**Results, 20 November 2026:** all seven spikes passed. The full record is in the rstudio-guide README, under "Phase 0 spikes for the new lessons". Changes they made to this plan:

- Typing stays on `editor.insert`. The coach steps past closers RStudio moves down a line (S1).
- Fingerprints are computed once the prompt returns (S2).
- The image creates `~/.duckdb`, or duckdb 2 stops R at a consent question (S5).
- The preferences send rendered documents and Shiny apps to the Viewer pane.
- Air is RStudio's built-in formatter (S4).
- Submenus need a lone `click` event (S4).
- The base stays `rocker/tidyverse` (S6). `rocker/rstudio` with only the lessons' packages is ~120 MB smaller compressed, but needs system libraries we'd have to track. The trip archive (2 million rows) and one year of CSV add about 72 MB.
- The lab is amd64 only, like iximiuz Labs; arm64 is not built or tested.

### Phase 1: engine and world

1. **The generator and its checks** (§2.2), and the committed small datasets.
2. **Content layout and collection** (§4.1), the schema, and a validator: `tools/check_content.py`.
3. **Bake** with the determinism check (§4.5), and `evidence.R` with its endpoint (§4.3).
4. **Coach:**
   - multi-line statements;
   - fingerprint checks in all three modes;
   - the action driver, limited to the actions S4 proved;
   - picker and progress;
   - Restart R on a lesson switch;
   - Files scrolling.
5. **Keep the current audits passing** on the article lessons (19/19, 12/12, 21/21) until Phase 2 replaces them, so engine changes are checked against a known baseline.

**Results, 20 November 2026:** Phase 1 is built. The full record is in the rstudio-guide README, under "Phase 1: engine and world". Changes it made to this plan:

- The time zone is `America/Toronto`, and the archive is every trip since 2021 (about 510,000), its 2025 equal to `trips.csv` (§2.2).
- `trips.csv` and the archive exist only in the image; each project links to them. The small files are committed, and the image build fails if it would generate different ones.
- The bake's stamp hashes the lesson, world, package lock, `evidence.R`, `bake.R` and Dockerfile, not the image digest, which changes on every rebuild. The article lessons are baked too, replacing provisioning's reference runs.
- An `open-file` action joins §4.2's list, and `ide` evidence can check the editor's mode and text.
- A fresh start also puts back settings the lesson changes and closes the documents its actions open. IDE state counts only once seen not holding.
- Restart R happens when switching to or from a new lesson, not between the article lessons.
- Two audit fixtures, `engine-check` and `engine-actions` (`"fixture": true`), stand in for lessons until the pilots; `tests/lesson_modes.cjs` and `tests/lesson_switch.cjs` audit them.
- Not done: Quarto steps that type into a `.qmd` (pilot 28 designs them), resetting the Help and Viewer panes and `View()` tabs, lesson 21's DuckDB file, and parallel audits.

### Phase 2: four pilot lessons, then the swap

**Pilots,** one of each kind:

- **1 Data visualization:** charts.
- **3 Data transformation:** pipelines and tibbles.
- **7 Data import:** files, warnings and messy data.
- **28 Quarto:** IDE actions and rendering.

Each pilot has full content, passes the bake twice, and passes the lesson audit in all three modes. Review tone, depth and step count before going on.

**Results so far, 20 November 2026:** the four pilots are written, bake and pass the lesson audit in all three modes (5/5 each). They were reviewed the same day: every claim in their explanations was checked against the data, which led to four changes (in the README's "Changes after the review"), and the lesson audits were rerun. The swap followed the same day (below). The full record is in the rstudio-guide README, under "Phase 2: the four pilot lessons". Changes they made to this plan:

- Lesson 1 charts `trips-week.csv`, a new world file: 2 to 8 June 2025 without the faulty trips, so lost bikes don't stretch every axis. Lesson 28's report uses it too.
- Lesson 1 saves its chart in its fourth step, straight after labelling it, and has 8 steps; lessons 3, 7 and 28 have 7 each.
- Lesson 3 teaches Ctrl+Shift+M on its first pipe (`"keys"` on a statement), not as a separate action step: the coach types that statement's `|>` with the shortcut, and Guided and Solo count it once the learner has pressed it.
- Lesson 7 saves `jan` rather than `activity` as CSV and as RDS, and reads each back: `jan`'s integer `Dock Faults` comes back from CSV as a double and from RDS unchanged. `activity` has no column whose type CSV loses. Parquet is left to lesson 22.
- Lesson 28's document is the lesson's script (`28-quarto.qmd`, with a `"starter"`). Its `library()` calls are typed into the setup chunk and run with Run › Run Selected Line(s), and every other step types into the document (a new `type` action, checked with `editor_match`) and renders. New Quarto Document comes last, so the learner doesn't type into the wrong document.

**Then replace the article lessons:**

1. Make the pilots the default run.
2. Port `tests/demo_mode.cjs` and `tests/practice_modes.cjs` to the pilots. Their probes carry over: a deleted step heading, a learner's `rm()` causing an error, reload, Stop, Start over, hiding the sidebar, and Back on the picker.
3. Delete `scenarios/graphical-tools/` and `scenarios/statistical-tools/`.
4. Rewrite the README's lesson sections. Keep its findings about RStudio's behaviour, which still hold.

**Done, 20 November 2026.** The pilots are the default run. `demo_mode.cjs` and `practice_modes.cjs` carry every probe listed above over to lessons 1 and 3 and the four-lesson run, and now also check R's evidence against the bake themselves. The article lessons are deleted, and the README's lesson sections rewritten. The port found the coach typing a new line's first key before Ace had scrolled the line into view; the coach now scrolls first. Every audit passes (rstudio-guide README, "Phase 2: the four pilot lessons"). Phase 3 is next.

### Phase 3: the remaining 25 lessons, part by part

Rest of Whole game (2, 4, 5, 6, 8), then Visualize (9–11), Transform (12–19), Import (20–24), Program (25–27) and Communicate (29).

A lesson is done when:

- the bake passes twice with identical results;
- the lesson audit passes in all three modes;
- review against its concepts list passes.

**Results, 21 November 2026:** all 25 lessons are written, so all 29 exist (382 statements), and each bakes with identical runs. Each was written against its concepts list, and its explanations checked against the data: quoted numbers are baked values, and facts stated without one are asserted by the world's `check.R`. The full record is in the rstudio-guide README, under "Phase 3: the other 25 lessons". Changes it made to this plan:

- **Lesson 2** teaches Alt+- on its assignments (`"keys": "Alt+-"`), as lesson 3 teaches Ctrl+Shift+M.
- **Lesson 4**'s script starts with the hurried code (a `"starter"` for an R script). The learner reformats it with Air, then rewrites it in the script with good names; renaming in place would need Rename in Scope, which the coach can't do.
- **Lesson 6** has `source` and `new-project` actions. The `.RData` settings are explained, not changed: the lab already never saves or restores the workspace. A fresh start takes RStudio back to the lessons' project after the new one.
- **Lesson 8**: a fresh start resets the Help pane and the Viewer and closes reprex's output; lesson 23's closes `View()` tabs.
- **Lesson 10** compares member types with frequency polygons; reordering moved to lesson 16, as weekdays barely differ.
- **Lesson 13** uses day-to-day change for `lag()`: the generated bikes' trips overlap, so time between a bike's rides isn't meaningful.
- **Lesson 14** splits comments into sentences for `separate_longer_delim()`; the survey has no delimited multi-select column.
- **Lesson 20** converts Excel day numbers with `as_date(origin =)` rather than asking readxl for dates, which warns on every row.
- **Lesson 21**'s database, `larkhaven.duckdb`, is generated in the image (four tables read back from the CSV files) and linked into each project, like `trips.csv`.
- **Lessons 21 to 24** needed new fingerprints: DBI connections, dbplyr's SQL, rvest's HTML and duckplyr frames (by their columns when big, so checking never reads the archive into memory). Charts are now compared without row order, and a patchwork as all its plots.
- **Lesson 29**'s documents are in its project; Word, PDF and the website render with `quarto_render()`, since Render renders a document's first format only.
- **Parallel audits** are built (`tests/run_lesson_audits.py`): on an 8-core machine, three at a time took 21 minutes for all 31 lesson audits, which add up to 60 minutes; each took about as long as alone.

### Phase 4: release

- Update the README's decision record.
- Republish the rootfs and measure start-up.
- Update the playground: `publish-rootfs-image.sh` already does this, so never run `create` again.
- Do a security review of the evidence endpoint.

**Results, 21–22 November 2026:** done, all automated; a manual review by the user is the last step left (the README's ledger). The evidence endpoint's review fixed one problem (R froze after `setwd("/")`, listing every file under it) and added two hardenings: nginx hides its version, and the coach's API answers as JSON with `nosniff`. A learner-read review of the 25 new lessons made 11 small text fixes. At the coach's own pace, a Demonstration takes 4.9 to 10.6 minutes (median 8.0); lessons 9 and 11 run just over the 10 minutes estimated in §1. Every audit passed after the last change. Rootfs v5 (1.97 GB) is on the public playground, and its two sessions were ready 107 s and 76 s after the start command, faster than v2, so §4.6's fallback (a GHCR pull at init) isn't needed. The full record is in the rstudio-guide README, under "Phase 4: release".

## 7. Testing

- **Static checks:** the schema; a `purpose` and `task` for every statement; step and statement counts within bounds; code formatted by the lab's formatter.
- **World:** the generator's checks; regenerating gives byte-identical files.
- **Bake** twice per lesson, comparing fingerprints.
- **Lesson audit (new, `tests/lesson_modes.cjs <lesson>`)**, driven as a learner, like the existing audits:
  - **Demonstration:** all steps, with fast pacing through the existing `localStorage` settings (`rsCoachTypeMs` 5, reading 200 ms).
  - **Guided:** the learner types every statement in the Console, plus one Show me.
  - **Solo:** the learner types the `alternatives` where there are any. Check my work must tick every line.
  - Fails on any page error.
- **Parallel runs.** Make the Compose project name, ports and run folder configurable, then run four lessons at once. That brings 29 lessons × 3 modes to roughly half an hour instead of two.
- **The ported deep audits stay,** for the pointer, typing, spotlight and reading-time measurements, on the pilots.
- **After any image change,** rerun everything.

## 8. Risks

| Risk | Response |
| --- | --- |
| **Writing volume.** The lab now writes the code and the data as well as about 400 statements' worth of purposes, tasks and explanations. This is the largest cost. | The world is designed once and reused. Review the pilots before scaling; one template per lesson; a review after each part. |
| **Generated data feels artificial,** and charts look too clean. | Base the generator on plausible distributions with noise, seasonality and weather effects. Review the pilots' charts with fresh eyes. Base R datasets are available for a one-off step. |
| **Drift from the concepts** the chapter teaches | Each lesson is reviewed against its concepts list (§5). |
| **More DOM coupling** from action steps (finding 4's list grows) | Pin the digest; S4's checks join the audit; an action that breaks drops to narration only. |
| **Fingerprints break** on a package bump or on large objects | Re-bake per stamp; size cap; rounding; the determinism check |
| **Slower iximiuz start-up** with a bigger image and the archive | Measure in S6; GHCR pull at init as the fallback |
| **Long Demonstrations** (5–10 minutes per lesson) | Reload already resumes. "Start from step N" is out of scope for now. |

## 9. Decisions

Settled on 20 November 2026. The user accepted the recommendations and said extra exercises aren't needed.

1. **Data:** one generated bike-share city for all lessons, with base R datasets for one-off steps.
2. **Depth:** 5–8 steps per lesson.
3. **A "Go deeper" link** to the matching chapter on each lesson, at lesson level.
4. **PDF in lesson 29:** Quarto's bundled Typst, not TinyTeX.
5. **Formatter in lesson 4:** Air, if RStudio's external-formatter setting works with it; styler otherwise.
6. **`regional-revenue`:** kept as the Guided stub for `coach_in_ide.cjs` until the grader work.
7. **Extra challenges:** not wanted. Solo is the unaided practice.

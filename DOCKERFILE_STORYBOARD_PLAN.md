# imagestoryctl: Turning a Dockerfile into an Illustrated Storyboard — Plan

> **Status: fourth revision (15 November 2026).** These decisions come from the project owner:
> - Every run builds and records the image; there is no static mode.
> - No generative AI is used anywhere. Every word, figure, costume, set, and prop comes from deterministic rules applied to the build's facts.
> - Every book follows one dramatic arc that builds tension and resolves it at the end.
> - The cast is human-like figures drawn in code on [napkin-chats](../napkin-chats/src/components/StickFigure.tsx)' approach, and their faces and bodies carry the emotional beats.
>
> **New in this revision:**
> - Scenes take place in **twelve widely varied settings,** each with costumes and accessories for the crew. There is no table staging.
> - Everything that stands for the Dockerfile and its layers keeps the **futuristic machinery** in every setting.
> - The whole look is **much more cartoony,** following [word-slice](../word-slice/index.html)'s cartoon-factory style. Its four-color print texture is gone.

## 1. Summary

`imagestoryctl` is a Go command-line tool. Point it at a local Dockerfile and it builds the image with Buildx, records every stage's layers, files, sizes, owners, whiteouts, and config, and draws the build as a cartoon storyboard. Each instruction gets one panel; consecutive config-only instructions share one. The panels are laid out as comic pages, phone-sized cards, and a self-contained HTML reader. Every panel does three things in the same order:

1. It quotes the line that was written.
2. It shows what that line did to the image: a crew of four cartoon figures works futuristic build machinery, set in one of twelve settings and dressed for it. Their faces and bodies show how the moment feels.
3. It states the measured effect in plain words: how many files and bytes a layer added, which config field changed, or what existed only during the build.

```sh
imagestoryctl board ./Dockerfile                        # setting chosen from the Dockerfile's hash
imagestoryctl board ./Dockerfile --setting opera-vienna # pin the home setting
imagestoryctl board ./Dockerfile --settings build=titan-base,runtime=frontier-saloon
imagestoryctl render storyboard/facts.json              # re-render without rebuilding
```

**The settings.** A book might open in a frontier saloon. The Prospector hauls glowing hex slabs off a robot-arm gantry that hangs from the rafters, while the Sheriff stamps config tags onto a chrome mast beside the piano. A multi-stage build travels between worlds. The build stage might run at night in a moon base on Titan and hand its binary through a maglev tube into a Habsburg opera house, where the runtime stage plays out among powdered wigs and lace cravats. The world changes; the build machinery never does.

**The story.** A question is posed on the cover, such as "Something on board is hidden. Does it still count?"

- **Rising action.** Tension rises as the stack grows: brows knit, shoulders rise, and gutters tighten.
- **Climax.** The setting's set piece hosts the Inspection: a high-noon showdown, the final act under the chandelier, or an airlock check. A futuristic scan gate answers the question with the image's real sizes and exposes anything hidden.
- **Resolution.** The Pilot boards with the setting's cue ("Hyah!", "Curtain up!", "Liftoff!"), and faces settle as the story ends.

The core promise:

> Every line of the Dockerfile gets a panel that shows what it did to the built image. Every book has drama, variety, and charm, and every number, word, costume, and expression comes from the build and a fixed set of rules.

This is the separate project that [IMAGE_VIZ_PLAN.md](IMAGE_VIZ_PLAN.md) §12.5 anticipates. Layerviz remains a visualization tool with no story or video export (its decision 8). `imagestoryctl` writes a `layerviz.bundle.v1` of every recording, so the same build opens in Layerviz.

## 2. What the Source Material Provides

| Source | Useful material | How this project uses it |
| --- | --- | --- |
| [IMAGE_VIZ_PLAN.md](IMAGE_VIZ_PLAN.md) | Symptom map (§2.1), the reframing of "layers merge" (§3), four questions per instruction (§5.2), three sizes (§5.3), misconception inventory (§5.5), language guide (§5.6), instruction coverage (§6), visual commitments (§10.2), beat-finder notes (§12.5) | The semantics, the copy rules, the beat catalog, and the rules no picture may break |
| [STORYBOARD_PLAN.md](STORYBOARD_PLAN.md) | Three truth layers (observed, metaphor, interface); a declarative edit file; stills and a contact sheet before motion; a validator for story truth; a release manifest with hashes | The production discipline. `script.json` is the edit file, and `imagestoryctl check` is the validator |
| [`word-slice/index.html`](../word-slice/index.html) | A cartoon factory drawn entirely in code from simple shapes. Flat fills; thick `#17212b` ink outlines (3–4 px); hard offset shadows with no blur (the same shape shifted 5–7 px down and right at about 20% ink); a white highlight stripe on glossy shapes; two-tone cables (a dark wide stroke under a light narrow one); rounded corners of 8 px; gear and star paths; a robot-arm gantry and a diagonal-striped conveyor. Bright friendly palette: gold `#ffd166`, pink-red `#ef476f`, green `#24a66a`, blue `#118ab2`, and slate greys on `#fffaf0` paper. Bouncy celebrations: stars, ribbons, confetti, and a gear-rimmed medallion | **The visual style of the whole book** (§9). The gantry, conveyor, gears, and slate machinery become the futuristic build kit (§9.11) |
| [`napkin-chats/src/components/StickFigure.tsx`](../napkin-chats/src/components/StickFigure.tsx) and its tests | Original figures drawn in code. A table of named joints. Poses as partial overrides of a neutral pose. Mirroring for the facing direction. Eased interpolation. Tests for completeness, constraints, and mirroring | The **figure rig** (§9.9). It is copied and adapted into Go and re-proportioned as cartoons. Napkin-chats' waist-up framing behind a table is deliberately **not** reused |
| [`napkin-chats/FUTURE_ENHANCEMENTS.md`](../napkin-chats/FUTURE_ENHANCEMENTS.md) | Figures must be original and must not copy a recognizable character or style | The same originality rule applies to the cast, costumes, and settings |
| [`layerviz/recorder/main.go`](../layerviz/recorder/main.go) | Per-stage Buildx OCI export with max provenance, tar-header-only layer reading, whiteout and opaque-directory detection, path safety, redaction | Copied and adapted into `internal/record`, following the precedent of Layerviz decision 11 |
| [`layerviz/src/model.ts`](../layerviz/src/model.ts) and `model.test.ts` | `resolvePaths` (stacking, whiteouts, opaque directories, a file that replaces a directory), `makeLedger`, decimal `formatBytes` | Ported to Go as `internal/overlay`, with its test cases. It computes hidden bytes and the merged view |
| [`layerviz/src/dockerfile.ts`](../layerviz/src/dockerfile.ts) | Effect classes; config evolution, including `ENTRYPOINT` clearing an inherited `CMD`; `ARG` scope; reachability; the shipping chain | A behavior specification. Parsing moves to BuildKit's own Go parser |
| [`layerviz/src/examples.ts`](../layerviz/src/examples.ts) | Nine problem/fix Dockerfile pairs with mechanism, fix, and cost text | The first golden corpus (18 Dockerfiles), and the wording of editor's notes. Each needs a small fixture build context (§14) |
| [`layerviz/src/runtime.ts`](../layerviz/src/runtime.ts) | Copy-up, whiteouts in the writable layer, writes under mounts | The resolution panel's semantics |
| [`layerviz/docs/IMPLEMENTATION_STATUS.md`](../layerviz/docs/IMPLEMENTATION_STATUS.md) | The recorder does not yet map layer history back to Dockerfile lines | This tool needs that mapping. Spike S1 builds it, and Layerviz can copy it back |

The owner first cited tone-viz as the style reference. On 15 November 2026 they confirmed that word-slice's cartoon factory is the intended look; tone-viz has no cartoon elements.

Several Layerviz examples reference files or URLs that don't exist; one example, for instance, `ADD`s from `example.invalid`. The corpus therefore gets fixture contexts that build offline once base images are cached (§14).

## 3. Scope

### 3.1 MVP

- Build and record any local Dockerfile or Containerfile with Buildx, including multi-stage builds, heredocs, `--mount` flags, build arguments, and BuildKit's build checks.
- Facts, beats, the dramatic arc, emotion mapping, setting selection, rule-based captions and dialogue.
- The cartoon style pack, with:
  - the crew of four on the cartoon rig;
  - the face rig and expression presets;
  - the costume grammar;
  - the futuristic build kit;
  - the MVP archetypes (§9.8);
  - **three settings:** the frontier saloon, the Habsburg opera house, and the Titan moon base.
- Page, card, and reader editions, plus a contact sheet, a model sheet, a costume sheet, a Layerviz bundle, a review list of every user-originated string, and redaction.
- `imagestoryctl check` for truth, drama, emotion, costume, privacy, layout, accessibility, and determinism.

### 3.2 Later

The other nine settings, the scroll edition and PDF, the full archetype set, a before/after `diff` spread, a motion comic (napkin-chats' eased blending and word-slice's bounce and pop carry over directly), and a CI action.

### 3.3 Non-goals

- **Generative AI of any kind,** for copy, figures, costumes, sets, or props. No model is called at build time, render time, or during asset production.
- Drawing a failed build. The tool reports BuildKit's error and stops, because a story needs a finished image to resolve.
- Rewriting a user's Dockerfile. Editor's notes can name a lever, but they never produce a patch.
- A linter. Lint-style findings come only from BuildKit's build checks, as in Layerviz.
- Vulnerability or license scanning, a registry browser, and a hosting or sharing service.
- Windows containers.

## 4. Input, Requirements, and Privacy

### 4.1 One way in

The input is a Dockerfile plus its build context. The flags `--context`, `--target`, `--platform`, `--build-arg`, and `--secret` are passed to Buildx unchanged.

The tool:

- builds every stage that can be targeted and records it;
- derives everything else (hidden bytes, the merged view, the ledger, and the consequences of cache order) from those recordings with the overlay rules;
- saves the result as `facts.json`.

`render` re-draws from `facts.json` without rebuilding.

Requirements:

- Docker Engine 29.9.x with Buildx 0.39.0.
- A `docker-container` builder, because the default `docker` driver can't export max provenance with OCI output. The tool creates one named `imagestory` on first run unless `--builder` names another.
- Network access for whatever the build itself pulls.

### 4.2 What runs and what leaves the machine

- **Running `board` runs a build.** `RUN` steps execute on the chosen builder. For Dockerfiles you don't trust, use a disposable builder.
- **The tool itself makes no network calls.** Network traffic comes only from Docker pulling bases and from `RUN` steps. The pinned Chrome binary is downloaded once into the user's cache and then reused (§12).
- **No content leaves the machine,** and no file contents are recorded. Only entry metadata is: path, type, size, mode, owner, and link target, as in the Layerviz recorder.

### 4.3 Redaction and string review

Other people's Dockerfiles carry internal registry hosts, mirror URLs, organization names, and build-argument values (IMAGE_VIZ_PLAN §12.5). Redaction runs before anything is laid out:

- **On by default (from the Layerviz recorder):** build-argument values, `LABEL` values, `ENV` values, registry hosts in `FROM` and `COPY --from=<image>`, and history text containing known build-argument values.
- **Added here:** URL hosts in `ADD` and inside `RUN`, email addresses, IP addresses, and high-entropy tokens. Values of variables whose names look secret (`TOKEN`, `SECRET`, `PASSWORD`, `API_KEY`, `PRIVATE_KEY`, `CREDENTIALS`) are never drawn, even with redaction turned off. Their names are drawn, because the name is the lesson.
- **Configurable:** `--redact-path REGEX`, `--redact-pattern REGEX`, and `--allow-string TEXT` for a reviewed exception.
- **Review list:** `strings.review.txt` lists every user-originated string that will be drawn, with its page and panel. `check` fails if a string in a sensitive class reaches the output without an `--allow-string`.

## 5. Architecture

```text
 Dockerfile ─► analyze ─► record ─► facts.json ─► beats ─► drama ─► setting ─► emotion ─► script.json ─► layout ─► compose ─► render
 + context     BuildKit    Buildx,    (stable IDs)   (beat     (question,  (home and    (who feels   (copy from     (tiers,   (figures,  (Chrome via
               parser,     per stage,                 finder)   tension,    per-stage    what, how    rule tables;   pages,    costumes,  chromedp:
               checks,     overlay                              acts)       settings)    much)        editable)      shots)    sets, kit) PNG, PDF)
               mvdan/sh    rules                                                                         ▲
                                                               style pack ───────────────────────────────┘  tokens, rig, presets, costumes, settings, kit, voices
```

Each arrow is a file on disk. A user can stop after `facts` or `script`, edit the result, and resume (§5.4).

### 5.1 Determinism contract

- **Rendering is a pure function.** The inputs are `facts.json`, the style pack, the setting choices, and the tool version. No random numbers, clocks, or environment values are involved. The same inputs give a byte-identical SVG and, under the pinned Chrome, identical PNGs.
- **Settings are chosen by rule** (§9.10.1). By default the setting comes from the Dockerfile's hash, so a Dockerfile always gets the same setting. The flags override it.
- **The story uses content facts only.** Those are paths, sizes, counts, owners, config, stages, and order. It never uses build timings or dates. Digests appear only in the back matter.
- **Rebuilding gives the same book** for the same Dockerfile, context, build arguments, and base digests, except where a `RUN` step fetches something that changed. `facts.json` is the reproducible artifact, and the manifest records the base digests the build used.
- **Variation comes from data and settings, not chance.** Similar Dockerfiles get similar stories.

### 5.2 Language and stack

Go is the clear fit:

- BuildKit's parser, instruction model, and build checks are Go packages.
- The Layerviz recorder is already Go.
- The output is one static binary.

Napkin-chats' rig and word-slice's drawing conventions are TypeScript and JavaScript, so they are copied and adapted rather than shared.

Versions were checked on 15 November 2026 and are pinned exactly in `go.mod`. LTS lines are used where a project has them.

| Component | Version | Role |
| --- | --- | --- |
| Go | 1.27.x | The CLI. Go has no LTS; the two newest majors are supported |
| `github.com/moby/buildkit` | v0.35.0 (Dockerfile frontend 1.29.0) | Parser, instructions, and build checks. These match Layerviz's reference builds |
| `mvdan.cc/sh/v3` | v3.14.1 | Parses `RUN` shell text into commands (§6.3) |
| `github.com/go-text/typesetting` | v0.3.5 | Shaping, metrics, and glyph outlines for lettering (§11) |
| `github.com/chromedp/chromedp` | v0.23.1 | Drives headless Chrome to rasterize SVG and print PDFs |
| Chrome for Testing | 157 (stable since 3 November 2026) | The pinned rasterizer |
| `github.com/spf13/cobra` | v1.10.2 | Command structure |
| Docker Engine / Buildx | 29.9.x / 0.39.0 | Builds and OCI export, using the user's installation |
| Fonts (SIL OFL 1.1, embedded) | Recursive (variable); Bungee family | Lettering and code; sound effects (§9.3) |

OCI layouts are read with the standard library, as the Layerviz recorder does. The build requests gzip layers, so no zstd decoder is needed.

### 5.3 Module layout

```text
cmd/imagestoryctl/     cobra commands
internal/analyze/      BuildKit parser → instructions, stages, effects, config timeline, ARG scope, reachability, checks
internal/shell/        RUN text → command list (mvdan/sh) → rule matches and action chips
internal/record/       builder setup, per-stage Buildx export, entry metadata, layer-to-line mapping, redaction
internal/overlay/      path resolution, hidden bytes, merged view, ledger (port of Layerviz model.ts)
internal/facts/        facts.v1 schema and stable IDs; Layerviz bundle export
internal/beats/        beat finder, polarity, and scoring
internal/drama/        dramatic question, act assignment, tension curve, cold open, countdown
internal/settings/     setting selection, sets, staging marks, night variants, cue lines
internal/emotion/      casting, beat → emotion table, intensity from tension, resolution moods
internal/figures/      cartoon rig, face rig, expression presets, blending, mirroring, symbols, model sheet
internal/costumes/     costume grammar: headwear, neckwear, garments, footwear, role marks, costume reactions
internal/kit/          the futuristic build kit: gantry, hull, slabs, cells, conveyor, mast, tube, gate, cradle
internal/sets/         set grammar: walls, openings, structure, horizons, signature props, extras
internal/copy/         caption, dialogue, and note rule tables; placeholder resolution
internal/script/       script.v1 schema
internal/layout/       tier and page packing, pacing, shot variety, reaction insets, for each edition
internal/lettering/    shaping, balloon-shaped line breaking, outlines
internal/compose/      SVG scene assembly, staging, and the cartoon finish
internal/render/       chromedp rasterizing, PDF, contact sheet
internal/check/        validator
styles/cartoon/        tokens.yaml, rig.yaml, presets.yaml, emotions.yaml, kit.yaml, sfx.yaml, voices.yaml, copy.yaml, layout.yaml
styles/cartoon/settings/<id>.yaml   one file per setting: sets, set tones, costumes, extras, props, cue lines
testdata/corpus/       Dockerfiles with fixture contexts and recorded facts
```

### 5.4 Commands, flags, and outputs

| Command | What it does |
| --- | --- |
| `board <Dockerfile>` | Builds, records, and renders every requested edition |
| `record <Dockerfile>` | Builds and writes `facts.json` (and the Layerviz bundle) only |
| `script <facts.json>` | Writes an editable `script.json` |
| `render <facts.json \| script.json>` | Lays out and renders without rebuilding |
| `check <dir>` | Validates an output directory (§14) |
| `styles` | Lists style packs and settings. `--model-sheet [--setting ID]` draws the crew in every expression preset; `--costume-sheet` draws every setting's crew and extras at neutral |

Flags:

- **Build:** `--context`, `--target`, `--platform`, `--build-arg`, `--secret`, `--builder`.
- **Setting:** `--setting ID` (home setting), `--settings stage=ID,…` (one per stage).
- **Story:** `--cut full|highlights`, `--notes explain|tips|off`, `--budget SIZE` (§8.2), `--title`.
- **Output:** `--edition pages,cards,reader` (adding `scroll` and `pdf` later), `--detail draft|full`, `--watch`, `--out`.
- **Redaction:** `--redact-path`, `--redact-pattern`, `--allow-string`.

```text
storyboard/
  facts.json                  every fact, with a stable ID
  layerviz.bundle.v1.json.gz  the same recording, for Layerviz
  beats.json                  beats with polarity and scores
  drama.json                  question, acts, and the tension level of every panel
  settings.json               home and per-stage settings, the rule that chose each, and every panel's set and shot
  emotion.json                every figure's preset and intensity in every panel, with the rule that chose it
  script.json                 pages, panels, archetypes, cast, copy with placeholders (editable)
  pages/page-01.svg … page-NN.png
  cards/panel-01.png …        1080 × 1350 cards
  reader.html                 self-contained, works offline
  contact-sheet.png           every panel as a thumbnail, with setting, shot, tension level, and expressions
  strings.review.txt          every user-originated string and where it is drawn
  manifest.json               tool, font, and Chrome versions; base digests; input hash; output hashes
```

An edited `script.json` can merge panels, rewrite copy, swap an expression preset, change a panel's set or shot, or pin a panel's size. It can't introduce an unsourced number: copy refers to facts through placeholders (§6.2), and `check` rejects literal figures it can't trace. `--watch` re-renders when the script changes, and rebuilds when the Dockerfile or context changes.

## 6. The Fact Model

Everything drawn or written comes from `facts.json`. Figures, costumes, sets, and machinery are metaphors; facts are measurements.

### 6.1 Shape

```jsonc
{
  "format": "imagestory.facts.v1",
  "input": { "dockerfileSha256": "…", "target": "runtime", "platform": "linux/amd64", "buildArgs": { "VERSION": "redacted" } },
  "tool": { "imagestoryctl": "0.1.0", "buildkit": "v0.35.0", "frontend": "1.29.0", "buildx": "0.39.0", "engine": "29.9.3" },
  "bases": [{ "ref": "node:22-slim", "digest": "sha256:…", "layers": 4, "storedBytes": 79400000 }],
  "instructions": [
    { "id": "L7", "line": 7, "op": "RUN", "effect": "layer", "stage": "build",
      "text": "RUN npm ci", "commands": [{ "argv0": "npm", "rule": "node-install" }] }
  ],
  "layers": [
    { "id": "build/L7", "instruction": "L7", "storedBytes": 182400000, "downloadBytes": 51200000,
      "entryCount": 1842, "top": [{ "path": "/app/node_modules", "bytes": 181020000, "rollup": true }],
      "whiteouts": [], "hides": [], "owners": { "0:0": 1842 } }
  ],
  "config": [{ "field": "User", "value": "node", "setBy": "L12", "inherited": false }],
  "stages": [{ "id": "build", "base": "node:22-slim", "ships": false, "storedBytes": 402000000 }],
  "ledger": { "download": 98100000, "stored": 312400000, "visible": 192400000, "hidden": 120000000 },
  "checks": [{ "rule": "JSONArgsRecommended", "line": 14 }],
  "redactions": [{ "kind": "registry-host", "count": 1 }]
}
```

Fact IDs are stable across runs of the same input. A layer's ID is `<stage>/<instruction>`, and a field is addressed by path, for example `build/L7.storedBytes`.

### 6.2 Copy refers to facts, never restates them

Every caption, balloon, and note template contains placeholders such as `{build/L7.entryCount}` and `{build/L7.top[0].path}`. The renderer substitutes and formats the values: decimal units as in Layerviz, and paths and values set in the code face. A caption therefore can't drift from the data, and the validator only has to find literal digits or paths that sit outside placeholders.

### 6.3 Reading `RUN` commands

A `RUN` step makes one layer, however many commands it chains. `mvdan.cc/sh` parses the shell text, including heredocs, `&&` chains, pipes, and `;`. A rule table turns each command into an **action chip**: a pictogram and a short label. A panel shows up to four chips in order, then "+N more steps".

The chips say what the step was asked to do; the receipt says what it measurably did. Some rules also inspect the recorded layer to confirm or contradict the command:

| Rule | Matches | Chip | Cross-check against the recorded layer |
| --- | --- | --- | --- |
| `pkg-index` | `apt-get update`, `apk update`, `dnf makecache` | refresh | Bytes under `/var/lib/apt/lists` or `/var/cache/apk` |
| `pkg-install` | `apt-get install`, `apk add`, `dnf install`, `microdnf install` | parcel | Bytes under `/usr` and the package database |
| `pkg-clean` | `apt-get clean`, `rm -rf /var/lib/apt/lists/*`, `apk --no-cache` | broom | The index directories are empty in this layer (relief), or still full (complication) |
| `download` | `curl -o`, `curl -O`, `wget -O` | down arrow | The downloaded path exists in the layer |
| `extract` | `tar -x`, `unzip` | open cell | Bytes under the target directory |
| `remove` | `rm`, `rm -rf` | eraser | Whiteouts in this layer, and the bytes they hide below |
| `node-install` | `npm ci`, `npm install`, `yarn install`, `pnpm install` | stacked cells | Bytes under `node_modules` |
| `python-install` | `pip install`, `uv sync`, `uv pip install`, `poetry install` | stacked cells | Bytes under `site-packages`, and cache directories left behind |
| `go-build` | `go build`, `go mod download` | gear | The output binary, and the module cache if no cache mount was used |
| `compile` | `make`, `gcc`, `cargo build`, `mvn package`, `gradle build` | gear | Build outputs and leftover toolchain caches |
| `ownership` | `chown`, `chmod` | key tag | Paths re-added with only owner or mode changed, and their bytes |
| `user-mgmt` | `useradd`, `adduser`, `addgroup`, `groupadd` | badge | Changes to `/etc/passwd` and `/etc/group` |

Unmatched commands become "runs `<argv0>`".

### 6.4 Mapping recorded layers to lines

To say what each line did, every recorded layer needs a source line. The Layerviz recorder doesn't produce this yet. The mapping:

1. Base layers are the ones whose history entries come before the first instruction of the stage. They are grouped into one hull section under the `FROM` panel, matching Layerviz's "base arrives as one block" commitment.
2. Of the remaining history entries, those without `empty_layer` align one-to-one with the layers.
3. Each entry's `created_by` text is matched against the instruction text in BuildKit's format (for example `RUN /bin/sh -c npm ci # buildkit`).
4. Source locations in max-mode provenance cross-check the match.

A layer that can't be matched gets a panel titled "a layer from an unknown line". It is never assigned to a guessed line.

Unnamed stages can't be passed to `--target`. The recorder builds them from a temporary copy of the Dockerfile in which each unnamed stage gets an `AS __stage_N` name. That changes nothing else, and the names never appear in output.

Spike S1 verifies all of this on BuildKit 0.35. It is also an open question in IMAGE_VIZ_PLAN §19, so the work is useful to both projects.

## 7. Beat Finder

A **beat** is a fact worth telling. Each beat has a **polarity**: a complication raises tension, and a relief lowers it. Beats decide:

- the dramatic question;
- the tension curve;
- which figures appear and how they feel (§8.5);
- which panels get more room;
- which panel opens the book;
- which panels survive the highlights cut;
- where editor's notes appear.

| Beat | Detects | Polarity (Δ) | Panel treatment | Editor's note (template) |
| --- | --- | --- | --- | --- |
| `deleted-still-shipped` | A whiteout that hides bytes in a lower layer | complication (+2) | A cloak falls over a lower cargo cell; the Stowaway appears beneath it | "The whiteout hides {path}. Its {bytes} still ship in line {line}'s layer." |
| `metadata-copy` | Paths re-added with only owner or mode changed | +2 at 10 MB or more, otherwise +1 | Twin cargo cells, one hatched | "Changing ownership copied {bytes} again." |
| `leftover-caches` | Package indexes or tool caches left in a layer | +1 | Dust bunnies on the slab | "{bytes} of {cache} ships with the image." |
| `secret-named-value` | BuildKit's `SecretsUsedInArgOrEnv`; the value present in config or history | +2 in config or history, +1 otherwise | The Stowaway pockets a sealed transmission; the value is never drawn | "This value travels in the image {where}." |
| `heaviest-line` | A layer holding 33% or more of the shipped stored bytes | +1 | Wide panel; the sound effect breaks the border | "This one line is {share} of the image." |
| `heavy-builder` | A non-shipping stage whose stored bytes exceed the shipped image's | +1 | The night set's stack towers over the Engineer | "{bytes} built so far. None of it ships yet." |
| `cache-order` | A broad `COPY` before a dependency install | +1 | A domino row of lit tiles over the later lines | "Change any of these {count} files and lines {from}–{to} run again." |
| `owner-not-user` | `USER` set, then a `COPY` without `--chown` whose recorded files are `0:0` | +1 | Badge versus cargo stamp | "These files belong to root, not {user}." |
| `inherited-root` | No `USER` in the shipped chain and the recorded config user is empty or root | +1, at the countdown | A root badge on the config mast | "The process will run as root." |
| `shell-form` | BuildKit's `JSONArgsRecommended` | +1, at the countdown | A `/bin/sh -c` sleeve around the launch card | "The shell becomes PID 1 and may not pass on signals." |
| `handoff` | `COPY --from` | relief (−2) | The maglev tube crosses between the two stages' settings | "Only this crosses into the image." |
| `same-step-cleanup` | `pkg-clean` or `remove` in the same `RUN` that created the files | −1 | The broom chip sparkles | "Cleaned up in the same step. Nothing hidden." |
| `secret-mount` | `RUN --mount=type=secret` | −1 | A hatch opens for the step and seals empty | "The secret was there for the step and is gone." |
| `cache-mount` | `RUN --mount=type=cache` | −1 | A slate locker seals | "The cache stays in the build." |
| `chown-at-copy` | `COPY --chown` or `--chmod` | −1 | The cell is stamped as it docks | "Owned correctly from the start." |
| `non-root-user` | `USER` with a non-root user in the shipped chain | −1 | The badge on the mast turns green | "The process will run as {user}." |

Notes are explanations by default (`--notes explain`), in the wording of IMAGE_VIZ_PLAN §5.5 and Layerviz's example text. `--notes tips` adds the lever, and `--notes off` removes them. Build-check findings that no beat covers appear in the back matter.

**Scoring** (used for the question, the cold open, and the highlights cut) is the sum of three parts: the beat's byte impact as a share of stored bytes, the size of its polarity, and the misconception weight from the IMAGE_VIZ_PLAN §5.5 inventory. All weights live in `beats.yaml`.

## 8. Narrative Engine

Every book follows the same arc, called **the Inspection**, in whichever setting it lands. The arc keeps books recognizable; the settings, the cast's reactions, and the build data supply the variety. Drama comes from faces, bodies, staging, and pacing, never from exaggerated words: captions stay dry and exact.

### 8.1 The arc

| Part | Length | What happens | Drawn from |
| --- | --- | --- | --- |
| **Cover** | 1 page | The dramatic question, over the finished stack on its launch cradle in the home setting. If anything is hidden, the Stowaway's narrowed eyes and down-pointed brows peek from low in the stack | The question rule (§8.2) |
| **Cast page** | 1 page | The crew lineup in the setting's costumes, each with its role title and color mark, and the "how to read this" key | Setting and style pack |
| **Cold open** | 1 panel | A flash-forward to the most tense panel, cut before its answer, with a reaction close-up. The next caption reads "Earlier…" | The panel with the highest tension, if it reaches level 3 |
| **I. Arrival** | 1–2 tiers | Global `ARG` transmissions arrive. The gantry claw lowers the base hull onto the setting's arrival mark while the Engineer waves it in. A stakes caption reads, for example: "79.4 MB before a single line of ours." | `FROM`, base layers |
| **II. Assembly** | Most of the book | One panel per instruction, stage by stage. Each stage opens with a chapter caption in its own setting ("Meanwhile, under the Vienna opera…"). The Engineer works the layers and the Inspector stamps the config. Complications raise tension and reliefs lower it. In a multi-stage build, the hand-off crosses between settings and is the midpoint turn | Instructions, beats |
| **Countdown** | 1 tier | The target stage's last instructions are compressed into a tier of narrow panels captioned "3 lines to go", "2", "1". The crew is braced | Final instructions of the target stage |
| **III. Inspection** (climax) | Splash, after a page turn | The target setting's set piece: a high-noon showdown, the final act, an airlock check. The setting's cue line announces it. The stack slides through the futuristic scan gate onto the mass readout. Hidden cargo flares pink-red under its cloaks, and the Stowaway, if aboard, is caught in the beam. Stages that don't ship drop away into the dark. The readout gives the ledger, and the caption answers the question | Ledger, hidden bytes, stages, setting |
| **Resolution: ignition** | 1–2 panels | The setting's departure cue ("Curtain up!"). The launch card slots into the console. The Pilot boards and looks down through the translucent slabs: the merged view. The crew settles into the ending's mood (§8.5). The final panel is calm and wide, with confetti when the ending is triumphant | `Entrypoint`, `Cmd`, `User`, `WorkingDir`, merged view, runtime mounts |
| **Epilogue** | Back matter | The fact sheet (one row per panel: line, effect, files, bytes, and the command that would confirm it), the ledger, beats with tips, build checks, redactions, versions | All facts |

The runtime mounts in the resolution are docking caps labeled "Docker's defaults". They are documented, not recorded, because the tool never starts the container.

### 8.2 The dramatic question

The first matching rule sets the question. The question must be answered by a fact on the Inspection splash, and `check` enforces that.

| Rule | Question on the cover | Answer at the Inspection |
| --- | --- | --- |
| `--budget` is given | "Budget: {budget}. Will it fit?" | "{download} to download. {under or over} budget by {difference}." |
| Hidden bytes of 1 MB or more | "Something on board is hidden. Does it still count?" | "{hidden} out of sight. All of it ships." |
| Non-shipping stages store at least twice the shipped image | "{builtBytes} built. How much ships?" | "{download} to download. The rest stays in the build." |
| A secret-named value | "A secret was named on line {line}. Where did it go?" | "It rides in the image {where}." or "It stayed in the build." |
| One layer holds half or more of the stored bytes | "One line carries most of the weight. Which?" | "Line {line}: {share} of the image." |
| Otherwise | "{lines} lines. What ships?" | "{layers} layers, {download} to download." |

`--budget` is the user's own size target. It draws a red load line on the readout and the spine, and it is the only way the tool judges an image as too big. Without it, the story reveals and never grades.

### 8.3 Tension

Each panel gets a tension level from 0 to 4 by a fixed formula:

- After the base lands, tension is 1.
- In Act II, each panel adds the polarity of its beats. A panel with no beats relaxes by 0.5 toward the floor.
- The **floor** rises with progress through Act II, from 1 to 2.5, so the action always rises.
- After a `deleted-still-shipped` or leaking `secret-named-value` panel, the floor is at least 2 until the Inspection. The reader has seen the Stowaway, and the crew hasn't.
- Values are clamped between the floor and 4, then rounded down to a level.
- The countdown tier is at least level 3, the Inspection is level 4, and the resolution and epilogue are level 0.

**Tension changes framing, faces, bodies, and backdrops, never a data encoding.** Sound-effect size, slab height, and what each color means stay tied to the facts at every level.

| Level | Composition | Frame and gutters | Faces and bodies (§8.5) | Balloons | Backdrop |
| --- | --- | --- | --- | --- | --- |
| 0 | Wide, level horizon, generous space | Card border 4 px; hard shadow offset (6, 7); 0.125 in gutters | Presets at full calm | Smooth | The full set, with extras going about their business |
| 1 | Level | Standard | Presets at 55% strength | Smooth | Full set |
| 2 | The scene tilts 4° | Shadow offset (7, 8) | 70% strength; the suit's tension pips show two | Smooth; bold for emphasis | Full set; extras glance over |
| 3 | 8° tilt, closer shot; a reaction inset on complications | Cut corners; 0.09 in gutters; shadow offset (8, 10) | 85% strength; cartoon symbols switch on | Jagged on complications | The set simplifies to its silhouette over flat color; extras gasp |
| 4 | 12° tilt, extreme close-up or splash | Diagonal gutters; the art breaks the border | Full strength; costume reactions (§9.9) | Bursts | Radial focus lines over flat color |

Lettering is never tilted. Captions, balloons, and receipts stay level, and stencilled labels on tilted props are counter-rotated to within 4° of level.

### 8.4 Pacing and shot rules

- Rising tension compresses panels: from level 3, panels drop to the next smaller width.
- **Reaction insets.** On a complication panel at level 3 or above, a small inset frames the reacting figure's face in close-up. This keeps the expression legible at any page size.
- **Shot variety.** Each archetype has variants for wide, medium, low-angle, high-angle, over-the-shoulder, and close-up shots. The layout never repeats a shot type in adjacent panels: it takes the next variant in a fixed order.
- **No table staging.** Every set offers staging marks at three heights: floor, raised (a stage, a deck, a balcony), and up high (rafters, rigging, a gantry). Figures stand, climb, hang, crouch, ride, and peer down. No set uses a table as its anchor.
- The countdown tier always comes directly before a page turn. The Inspection opens the next page as a splash.
- The cold open sits alone on the page after the cast page. "Earlier…" starts Act I on a fresh page.
- After the Inspection, gutters widen and the resolution uses the widest panel in the book.

### 8.5 Cast and emotion

**Casting rules.** The cast has four fixed roles. Each setting gives them local titles and costumes (§9.10), and the build decides who appears:

| Role | Color mark | Stands for | Appears | Example titles |
| --- | --- | --- | --- | --- |
| **Engineer** | Blue (files) | The builder doing the file work | Every layer panel and stage chapter | Prospector, Stagehand, Base Mechanic |
| **Inspector** | Gold (config) | The image's config and the final check | Config panels, the countdown, the Inspection, the resolution | Sheriff, Impresario, Ranger Captain |
| **Stowaway** | Pink-red (hidden or leaked) | Bytes that ship unseen, or a value that leaks | Only from the first `deleted-still-shipped` or leaking `secret-named-value` beat onward. A clean build never sees it | Outlaw, Gatecrasher, Smuggler |
| **Pilot** | Green (what ships) | The container's process, PID 1 | Only at the resolution | Stagecoach Driver, Leading Singer, Shuttle Pilot |

Every costume carries its role's color mark and the role's **signature prop type**, so a reader recognizes the role in any setting:

- Engineer: a tool.
- Inspector: a stamp plus something to record with.
- Stowaway: a loot sack.
- Pilot: a steering or performing object.

The cast page and the key say plainly that the figures are metaphors.

**Emotion table.** For each figure on the panel, the first matching row of `emotions.yaml` picks an expression preset (§9.9). Nothing is chosen at random.

| Event | Engineer | Inspector | Stowaway | Pilot |
| --- | --- | --- | --- | --- |
| Base lands; S or M layer | focused | — | — | — |
| L or XL layer; `heaviest-line` | strained | worried (at XL) | — | — |
| `same-step-cleanup`, `chown-at-copy`, `cache-mount` | satisfied | — | — | — |
| `deleted-still-shipped` | satisfied: it believes the cargo is gone | — | sly, under the cloak | — |
| `secret-named-value` | worried | alarmed | sly, pocketing the transmission | — |
| `secret-mount` | — | satisfied | — | — |
| `leftover-caches`, `metadata-copy`, `owner-not-user` | surprised | suspicious | — | — |
| `cache-order` | — | suspicious (hand to chin) | — | — |
| Config edit; `EXPOSE` | — | focused (stamping); wry for `EXPOSE` | — | — |
| `non-root-user` | — | satisfied | — | — |
| `heavy-builder` | worried | — | — | — |
| `handoff` | relieved | satisfied | — | — |
| Countdown | determined | focused; worried if `inherited-root` or `shell-form` | sly, if aboard | — |
| Inspection, nothing hidden or leaked | triumphant | satisfied | — | — |
| Inspection, something hidden or leaked | shocked | angry, pointing | caught | — |
| Resolution, every complication relieved | triumphant | satisfied | — | serene |
| Resolution, complications remain | resigned | determined, logging the finding | sulking, still aboard | serene |

**Extras** (townsfolk, audience, deckhands, crowds) take the same table's mood at reduced strength. They go about their business at levels 0–2, gasp at level 3, and react with the crew at the Inspection.

**Intensity.** A preset is blended from neutral by a strength set by the panel's tension: 40% at level 0, rising by 15% per level to 100% at level 4. The blend uses napkin-chats' eased interpolation for joints and face parameters alike. Resolution presets always apply at full strength.

**Truth in irony.** Dramatic irony is allowed only when the truth is on the same panel. When the Engineer looks satisfied at a whiteout, the Stowaway, the dotted cargo cell, and the receipt ("Hides 120 MB that still ships") share the panel. No expression may contradict a caption, and `check` enforces this.

**The ending is honest.** Tension resolves through revelation, not good news. If hidden bytes still ship, the Stowaway is still aboard at the end, sulking with its arms crossed. The Engineer is resigned but calm, and the tip sits in the epilogue.

### 8.6 Dialogue

Figures speak through balloons, with tails to their mouths. Machines speak through squared readout balloons. Every line comes from a fixed template table (`voices.yaml`, `copy.yaml`, and each setting's cue lines). The first matching row is used. A panel has at most one balloon, or two on an act turn or at the Inspection.

| Speaker | Balloon | Example lines (placeholders filled at render) |
| --- | --- | --- |
| Engineer | Smooth | "Heaviest so far." · "Deleted." (at a whiteout) · "Only this crosses." · "Wait—what?" (Inspection, shocked) |
| Inspector | Smooth; jagged when angry | "Port {port}. A note, not a door." · "From here on, it runs as {user}." · "The old CMD is gone." · "There. Under the cloak." |
| Stowaway | Dashed whisper, pink-red tail | "Still here. {bytes}." · "They'll never look down here." · "Worth the trip." (resolution) |
| Pilot | Smooth, green tail | "Starting {argv}." · "I can see {visible}." |
| A build-only transmission | Dashed whisper | "Build only. Don't write me down." · "…they wrote me down." (when it shows in history) |
| The scan gate and readout | Squared, with figures in the code face | "Download {download}. Stored {stored}. Visible {visible}. Hidden {hidden}." |
| The setting (cue lines) | Caption burst | Inspection and ignition cues from §9.10, such as "High noon." and "Hyah!" |

Captions use their own tables:

- the **line** caption (top left);
- the **receipt** (bottom right);
- **chapter** captions, flavored by setting ("Meanwhile, on Titan…"), with "At the same time…" for stages BuildKit can build in parallel;
- **countdown** captions;
- **stakes** captions;
- **answer** captions.

The editor's notes come from the beat table (§7). The tone is understated: faces carry the feeling, and words carry the facts.

### 8.7 Two worked arcs

**`deleted-still-shipped` in the frontier saloon** (the fixture copies a local tarball instead of `ADD`ing a URL; sizes are Layerviz's illustrative values until recorded):

| Panel | Level | What the reader sees |
| --- | --- | --- |
| Cover | — | "Something on board is hidden. Does it still count?" The stack stands on its cradle at the end of Main Street; the Outlaw's V-brows peek from low in the hold |
| Cast page | — | The Prospector (blue bandana, pick-wrench), the Sheriff (gold star, stamp), the Outlaw (pink-red hat band, loot sack), the Stagecoach Driver (green neckerchief, reins) |
| Cold open | 4 | Line 4. The Prospector dusts off its hands: "Deleted." Under the falling cloak, the Outlaw smirks: "Still here. 120 MB." Reaction inset on the smirk. Then "Earlier…" |
| L1 `FROM alpine:3.21` | 1 | Wide shot of Main Street: the gantry claw lowers a slate hull section in front of the swinging doors (THOOM). The Prospector waves it in. Stakes caption |
| L2 `COPY source.tar.gz /tmp/` | 2 | Low angle inside the saloon: a 120 MB cargo cell rides the conveyor past the piano. The XL THUMP breaks the border. The Prospector strains, with sweat drops and knees bent |
| L3 `RUN tar -xzf …` | 2 | High angle from the rafters: a 210 MB slab of extracted files. "Heaviest so far." |
| L4 `RUN rm /tmp/source.tar.gz` | 4 | The full scene: the cloak, the dotted tarball cell beneath it, the satisfied Prospector, the sly Outlaw, the receipt, and the editor's note |
| Countdown: L5 `CMD` | 3 | Over the Sheriff's shoulder as it stamps the launch card. The Outlaw's eyes glint in the dark below. "1 line to go." |
| Inspection | 4 | "High noon." The scan gate stands mid-street and the stack slides through. The Outlaw is caught: brows up, grimace, hands raised, hat popping off. The Sheriff, angry with steam from its ears, points: "There. Under the cloak." The Prospector is shocked. A wanted poster on the wall now shows the Outlaw. The readout: "120 MB out of sight. All of it ships." |
| Ignition | 0 | Sunset. "Hyah!" "Starting /opt/server." The Driver is serene and the Prospector resigned, palms up. The Outlaw sulks in the hold, still aboard |

**`the-handoff` (fix version) across two settings:** the build stage on Titan at night, the runtime stage in the Vienna opera house.

| Panel | Level | What the reader sees |
| --- | --- | --- |
| Cover | — | "{builtBytes} built. How much ships?" The opera stage is lit, with Titan's dark dome looming behind it |
| Chapter `build` | 1 | "Meanwhile, on Titan…" The night variant: Saturn-shine over the cryo hold. The Go toolchain hull lowers heavily and the Base Mechanic strains |
| L2–L4 | 1–2 | The Ranger Captain stamps `WORKDIR`. Source cells ride the conveyor. The compile runs on a cache-mounted step, and the locker seals (the Mechanic is satisfied). The night stack towers, and the Mechanic is worried: "None of it ships yet." |
| Chapter `runtime` | 2 | "Meanwhile, at the Vienna opera…" `FROM scratch`: an empty cradle on the bare stage. The Impresario raises a lorgnette, wry: "Nothing here. Not even a shell." |
| L6 `COPY --from=build` | 1 | The midpoint turn, as a diagonal split panel: Titan on the left, the opera on the right, the maglev tube crossing between them with one small cell. The Stagehand catches it, relieved: "Only this crosses." |
| Countdown: L7 `USER`, L8 `ENTRYPOINT` | 3 | The badge turns green and the Impresario is satisfied. The launch card prints. "2 lines to go." "1." |
| Inspection | 4 | "The final act." Under the chandelier, the stack passes the scan gate. The Titan bay drops away into the dark. The audience gasps, then applauds. The readout answers |
| Ignition | 0 | "Curtain up!" The Leading Singer, its pouf wig topped with a tiny ship, steps into the spotlight: "Starting /server." Confetti; the crew is triumphant |

### 8.8 Cuts

- `--cut full` draws every panel.
- `--cut highlights` keeps:
  - the cover, cast page, and cold open;
  - chapter openers;
  - beats above the score threshold;
  - the countdown and the Inspection;
  - the resolution and the back matter.

  Other lines collapse into summary strips, for example "Lines 3–9: four config edits and two small layers."
- The default is `full` up to 24 instructions and `highlights` above that.

## 9. Design System: the Cartoon Pack

This section came from a design pass with the frontend-design skill: brief, tokens, wireframes, principles, then a review that removes the generic defaults (§9.6). The owner set the visual reference: word-slice's cartoon factory. The brief's own words win over the skill's default-avoidance, including word-slice's cream paper.

### 9.1 Brief

- **Subject:** how a Dockerfile becomes an image, one line at a time. A cartoon crew works futuristic build machinery in a dozen period and fantastical settings, with a question, rising tension, a reveal, and a resolution.
- **Audience:** engineers who write and review Dockerfiles, and the teammates and readers they share the output with.
- **Primary job:** at a glance, show what each line did: files and bytes, a config field, or nothing that ships.
- **Secondary job:** be funny and gripping enough that someone reads to the end, then posts it, puts it in a README, or uses it in a talk.

**The grounding idea.** The build is a cartoon factory line that travels. Word-slice taught the vocabulary: a robot arm drops from above, a striped conveyor carries tiles, and gears turn, all flat-shaded with thick outlines and hard shadows. Here that factory line becomes the futuristic build kit, and it turns up wherever the story is set: a gantry hanging from saloon rafters, a conveyor running across an opera stage, a maglev tube punched through a castle wall.

**The world is costume; the machinery is truth.** Anything that represents the Dockerfile looks futuristic and identical in every setting: layers, files, config, mounts, the scan gate, and the launch cradle. Everything period-flavored is decoration and carries no data. A reader can always tell the two apart, and the anachronism is the joke.

### 9.2 Color

The semantic colors come from word-slice's palette:

| Token | Hex | Meaning |
| --- | --- | --- |
| `paper` | `#fffaf0` | Panel fill, balloon fill, faces and hands (identity-neutral), page background |
| `ink` | `#17212b` | Every outline, line art, faces' features, lettering |
| `files` | `#118ab2` | **Files.** Layer glows, cargo cells, the Engineer's color mark, sound-effect shadows |
| `config` | `#ffd166` | **Config.** Chevron tags, the launch card, the Inspector's color mark |
| `hidden` | `#ef476f` | **Hidden or leaked.** Cloaks' dots, hidden bytes, leaking values, tension flashes, the Stowaway's color mark |
| `ships` | `#24a66a` | **What ships,** what the container sees, the Pilot's color mark, the clean-build medallion |
| `night` | `#253746` | **Build-only.** Night variants of sets, `ARG`, build mounts, the build cache |
| `machine` | `#2f3b46`, `#3f4c58`, `#9aa6b2`, `#d6dde5` | The futuristic kit's slate chassis, and its light seams and cable cores |

**Set tones.** Each setting has its own three to five muted set tones for architecture, costumes, and props (§9.10).

- Set tones must stay below a chroma ceiling: OKLCH chroma 0.09, checked automatically. The semantic colors sit at 0.12 and above, so meaning always pops out of the decoration.
- A set tone may never sit within ΔE 12 of a semantic color.
- Costume bodies use set tones. Only the role's color mark uses a semantic color.
- Each setting includes at least one dark set tone, at OKLab lightness 0.5 or below, for hats, boots, and furniture. The saloon mock-up needed one: its first four tones were all light or mid.

**Dots mean hidden.** The cartoon finish is flat everywhere else. The single exception is hidden content, which gets a pink-red dot pattern, so the one patterned thing on the page is the thing that ships unseen.

**Symbols stay out of the semantic colors.** Cartoon symbols and body details use paper, ink, or set tones, even where cartoon habit expects a color:

- sweat drops are paper with an ink outline, not blue;
- tongues and mouth interiors use ink or a set tone, not pink-red;
- flames and lamp glows are paper, not gold.

Two named effects are the only exceptions, and each uses the figure's own role color: the sly Stowaway's pink-red shadow and the serene Pilot's green glow rings.

**Lettering contrast.** Lettering is always ink on paper, ink on gold, or paper on night. All three pass WCAG AA by a wide margin. Color never carries meaning alone, because every role also has a shape (§9.11) and every figure a silhouette and prop (§9.9).

### 9.3 Type

| Role | Face | Settings | Why this face |
| --- | --- | --- | --- |
| Lettering (balloons, captions, receipts, notes) | **Recursive** Sans, Casual | `CASL` 1; `wght` 760 for text and 950 for emphasis; uppercase | Word-slice letters at 800–900 weight. Recursive's casual axis gives a friendly brush-cartoon hand at that weight |
| Code (Dockerfile lines, paths, values, readouts) | **Recursive** Mono | `MONO` 1, `CASL` 0.3, `wght` 650; original case | The same family as the lettering, so code feels part of the cartoon while staying exact |
| Sound effects | **Bungee**, **Bungee Inline**, **Bungee Shade** | Stacked: paper face, ink extrusion, inline highlight, with a hard shadow in `files` blue | Chunky signage letters built to be layered: cartoony, and a quiet nod to layers |

That is two families. Recursive replaces Shantell Sans and Courier Prime from the third revision: both were too light for word-slice's heavy cartoon weight.

Comic lettering is traditionally uppercase. Paths, values, image references, and code stay in their real case in Recursive Mono, because `/APP` is not `/app`. The reader's surrounding HTML uses sentence case.

Minimum sizes: 7.5 pt lettering and 7 pt code on a printed page; 30 px lettering on a 1080 px card. `check` enforces both (§14).

### 9.4 The data signature: loudness is bytes

On a layer panel, the size of the sound effect encodes the layer's size, so a reader learns within a page that a big noise means a big layer. Config edits make no sound; they leave a stamp mark. Faces carry the feeling and sound effects carry the data, and the two never mix.

| Bytes | Size class | Treatment |
| --- | --- | --- |
| Under 4 KB | none | No sound effect |
| 4 KB–1 MB | S | 14 pt, Bungee only |
| 1–10 MB | M | 24 pt, plus the Inline layer |
| 10–100 MB | L | 38 pt, plus the Shade extrusion, slight perspective |
| 100 MB and above | XL | 56 pt, all layers, drawn with word-slice's pop overshoot (125% with a burst ring), and it **breaks the panel border** |

The lexicon is fixed by archetype and rule: a hull section lands with THOOM; package installs CLANK; builds WHRRR; `COPY` docks with THUMP; downloads and `ADD <url>` go FWOOSH. A falling cloak is silent. The layer that holds a whiteout is almost always under 4 KB, so the size rule gives it no sound, and the fall is drawn with motion lines instead. If that layer also adds 4 KB or more, it takes the sound of what it added. The Inspection makes no sound effect; its numbers are the payoff.

### 9.5 Layout

**Page edition.** US comic trim, 6.625 × 10.25 in, with 0.125 in bleed. That renders at 300 dpi to 2063 × 3150 px, inside the standard live area of about 6.06 × 9.56 in.

**Panels are word-slice cards:**

- paper fill;
- a 4 px ink border (at card scale; borders scale with the edition);
- 8 px rounded corners;
- a hard offset shadow with no blur.

They sit on a page washed with the home setting's lightest set tone. Gutters are 0.125 in at rest, and they tighten with tension (§8.3).

Each page has two to four **tiers**. Each tier is a 12-column row holding one to three panels. A **stack spine**, 0.3 in wide, runs down the outer edge of every page and doubles as the story's gauge. It is drawn as a slim futuristic column:

- It shows the shipped chain's slabs so far, with heights linear in bytes and a 2 pt minimum.
- Config edits are small gold ticks beside the slabs. As in Layerviz (§10.1, item 5), config goes on the spine, never on a slab.
- Hidden bytes show as pink-red dots inside the slab that still holds them.
- The current panel's layer is outlined. With `--budget`, a red load line crosses the spine.

During a non-shipping stage's chapter, that stage's spine grows in night slate beside the shipped one.

```text
 ┌────────────────────────────────────────────┐ trim (page washed in the set's lightest tone)
 │ ╭──────────────────────────────────╮ ┌──┐  │
 │ │ Meanwhile, in the saloon…        │▌│▁▁│  │ ← panels are cards: ink border,
 │ ╰──────────────────────────────────╯▌│··│  │   rounded corners, hard offset shadow
 │ ╭────────────────╮╭────────────────╮ │██│◄ │ ← spine: slabs ∝ bytes, hidden bytes
 │ │ L1 FROM        ││ L2 WORKDIR     │▌│  │  │   as pink-red dots, config as gold
 │ │ gantry lowers  ││ Sheriff stamps │▌│  │  │   ticks, current layer outlined
 │ │ the hull       ││ a gold tag     │ │  │  │
 │ ╰────────────────╯╰────────────────╯ │  │  │
 │ ╭──────────────────────────────────╮ │  │  │
 │ │ L4 RUN npm ci        C L A N K ! │▌│  │  │ ← XL sound effect breaks
 │ │ Prospector strains under a slab  ╲│▌│  │  │   the border
 │ ╰──────────────────────────────────╯ └──┘  │
 └────────────────────────────────────────────┘
```

**Panel anatomy: you wrote, you got.** The top-left caption quotes the line. Its left edge has a stripe in the effect's color (blue for a layer, gold for config, night for build-only, ink for a stage start), followed by the line number and the code. The bottom-right caption is the receipt. A reader's eye travels from the line, through the crew at work, to the effect.

```text
╭─────────────────────────────────────────────────────────╮
│▌L7  RUN npm ci                                          │ ← line: effect stripe, number, code
│                                    C L A N K !          │ ← sound effect sized by bytes
│  ╭──────────────────╮      ┃┃ gantry cable (two-tone)  │
│  │ HEAVIEST SO FAR. │     ╔╩╩╗ claw                     │
│  ╰────┬─────────────╯   ⬡⬡⬡⬡⬡⬡⬡⬡⬡⬡  hex slab, blue seams  │
│     ▄███▄  ← hat         ⬢ ⬢⬢ ⬢⬢⬢  cells ∝ bytes,        │
│    ( ◔ ◔ ) ← inner brow ends raised; squeezed eyes      │   top paths stencilled
│     ( ▭ )  ← grimace; sweat drops                        │
│    ╱█████╲ ← arms hug the load; blue bandana             │ ← the Prospector, strained,
│      ▌ ▐   ← knees bent; hard ellipse shadow             │   at the panel's strength
│   [refresh] → [stacked cells]                           │ ← action chips (§6.3)
│              ╭────────────────────────────────────────╮ │
│              │ ADDS 1,842 FILES (182.4 MB), MOSTLY    │ │ ← receipt; the path keeps
│              │ UNDER /app/node_modules                │ │   its case in Recursive Mono
│              ╰────────────────────────────────────────╯ │
╰─────────────────────────────────────────────────────────╯
```

**Cover.** The setting's establishing shot frames the finished stack, and the price box shows the download size.

```text
╭──────────────────────────────────────╮
│  14 LAYERS   [BUILT linux/amd64,     │ ← layer count; the build seal names
│               BuildKit 0.35]         │   platform and builder
│        THE MAKING OF                 │
│      <title, Bungee layered>         │
│       A Frontier Saloon Story        │ ← the home setting
│                                      │
│   (Main Street at dusk; the stack    │
│    on its launch cradle; low in the  │
│    hold, two narrowed eyes under     │
│    V-shaped brows, if anything is    │
│    hidden)                           │
│                                      │
│  "SOMETHING ON BOARD IS HIDDEN.      │ ← the dramatic question
│   DOES IT STILL COUNT?"              │
│                            ╭───────╮ │
│                            │ 98 MB │ │ ← price box: download size
│                            ╰───────╯ │
╰──────────────────────────────────────╯
```

**Inspection splash.** The full page, after a page turn, staged as the target setting's set piece (§9.10). The futuristic scan gate stands at the center. Inside its beam, the slabs turn translucent:

- visible files show in green;
- hidden cargo flares pink-red under its cloaks, and the Stowaway is caught in the light;
- non-shipping stages drop away below the floor line into night.

The crew and extras react along the bottom in a strip of reaction insets. The readout's balloon gives the ledger, and the answer caption closes the page.

**Card edition (1080 × 1350).** Each panel is recomposed for a 4:5 frame, not cropped from the page, with faces drawn larger relative to the frame. Each card has its own mini spine on the left edge and the line caption across the top. Cards sit on the setting's sky gradient, the one gradient the style allows, as word-slice's game screen does.

**Alignment.** Captions and receipts are left-aligned. Balloon text is centered in lozenge-shaped lines (§11). Panels read in a Z path, left to right and top to bottom. Panel numbering is the Dockerfile line number, because the content is a real sequence and the line number is what the reader will look up.

### 9.6 Review against the generic defaults

| Generic reflex | Changed to | Why |
| --- | --- | --- |
| A generic cartoon look: gradients, soft drop shadows, glossy 3D | Word-slice's specific finish: flat fills, thick ink outlines, hard offset shadows with no blur, one highlight stripe, two-tone cables | The owner's reference, and a finish that stays crisp in print and at card size |
| One house setting for every story | Twelve settings chosen by rule, with stages able to land in different worlds | Variety the owner asked for, without randomness |
| Period setting and Dockerfile machinery in one blended style | Machinery is always the same futuristic kit; the world is decoration | A reader can always tell data from costume, and the anachronism is funny |
| "Futuristic" as neon on near-black | The futuristic kit as word-slice's slate machinery with light seams and colored glows, on bright sets | Avoids the neon default and stays readable |
| Stock mascots | Original figures drawn in code on napkin-chats' rig, re-proportioned as cartoons, identity-neutral, dressed by a costume grammar | Deterministic, original, consistent from panel to panel |
| Expressions chosen by feel for each panel | A fixed emotion table driven by beats, with strength driven by the tension curve | Every face is reproducible and explainable |
| Characters around a meeting table | Full-body figures on staging marks at three heights, with shot variety enforced | The owner asked for no table scenes |
| Halftone texture as decoration | Dots appear **only** on hidden content | The one patterned thing is the thing that ships unseen |
| Numbered panels 01, 02, 03 | Dockerfile line numbers in the caption stripe | The number a reader needs is the line number |
| Uppercase everything, paths included | Real-case code face for paths, values, and code | Uppercase paths would be wrong |

### 9.7 Principles

1. **You wrote, you got.** Every panel opens with the line and closes with its effect.
2. **The world is costume; the machinery is truth.** Anything that represents the Dockerfile is futuristic and identical in every setting.
3. **Faces say how it feels; captions say what happened.** An expression never contradicts a caption.
4. **Loudness is bytes.** Only layers make noise, and the noise scales with size.
5. **Dots mean hidden.** Nothing else on the page is patterned.
6. **Each color means one thing** for machinery and role marks alike. Files are blue, config is gold, build-only is night, hidden or leaked is pink-red, what ships is green. Set tones stay muted.
7. **The stack is always on the page.** The spine is the gauge the tension builds against.
8. **Tension changes framing, faces, bodies, and backdrops, never data.**
9. **Every question gets answered.** Whatever the cover asks, the Inspection answers with a fact.
10. **Compose for each format; never crop.**
11. **Layerviz's commitments hold** (IMAGE_VIZ_PLAN §10.2):
    - A layer and a config edit never share a form.
    - Nothing is drawn fusing into one layer.
    - A mount is a cap, never a slab.
    - Build-only things stay beside the build, and a leak is drawn as a leak.

### 9.8 Panel archetypes

An archetype is a reusable scene built from three parts: the futuristic kit pieces that carry the data, a cast with staging, and the setting's set at a chosen staging mark. Each has shot variants: wide, medium, low angle, high angle, over the shoulder, close-up, and card. The layout engine picks the variant that fits the cell and the shot rules.

| Archetype | Instructions | Kit and staging (set dressing comes from the setting) | Data slots | MVP |
| --- | --- | --- | --- | --- |
| `from.base` | `FROM <image>` | The gantry claw lowers a sealed hull section onto the arrival mark (THOOM); the Engineer waves it in | Reference (redacted host), layer count and bytes, inherited config ticks | Yes |
| `from.scratch` | `FROM scratch` | An empty cradle outline on the arrival mark; the Inspector peers in | — | Yes |
| `from.stage` | `FROM <stage>` | A copy of an earlier stage's stack glides in on rails | Parent stage, inherited config | Yes |
| `run.cells` | `RUN` | A hex slab docks with its cargo cells arranged as a squarified treemap: cell faces ∝ bytes, top paths stencilled, action chips. The Engineer works or strains by size class, at floor, raised, or high marks | Top entries, rollups, count, bytes, chips | Yes |
| `run.mount` | `RUN --mount=cache\|secret\|ssh` | Adds a slate hatch or locker that opens during the step and seals after, with nothing from it on the slab | Mount type and target | Yes |
| `copy.context` | `COPY`, `ADD` (local) | Cells ride word-slice's striped conveyor in from a build-context airlock set into the set's wall; owner stamps | Sources, destination, owners, entries | Yes |
| `copy.handoff` | `COPY --from` | A diagonal split panel when the two stages are in different settings: the maglev tube carries the cells from one world into the other | Source stage, paths, bytes | Yes |
| `add.remote` | `ADD <url>` | A courier drone drops a parcel through a hatch (FWOOSH) | Redacted host, checksum flag | Later |
| `config.mast` | `ENV`, `LABEL`, `WORKDIR`, `USER`, `EXPOSE`, `STOPSIGNAL`, `SHELL`, `VOLUME` | The Inspector stamps gold chevron tags onto the config mast, which stands beside the setting's notice spot (a wanted-poster wall, a playbill board, a mission board); up to four tags per montage | Field, key, value (or redacted) | Yes |
| `config.card` | `CMD`, `ENTRYPOINT` | The launch card prints the final argv; a torn stub if `ENTRYPOINT` cleared an inherited `CMD` | Assembled argv, shell form | Yes |
| `config.door` | `EXPOSE` (beat) | A door painted onto the hull, with the Inspector's wry look | Ports | Yes |
| `arg.transmission` | `ARG` | A slate transmission capsule arrives, is read, and dissolves. It gets a pink-red corner and the Stowaway's hand if the value shows in history | Name, scope | Yes |
| `whiteout.cloak` | The layer containing a whiteout | A cloak falls over a lower cargo cell. The cell stays visible in pink-red dots underneath, with the Stowaway crouched beside it. The Engineer above looks satisfied | Path, hidden bytes, the hiding line | Yes |
| `metadata.twins` | `RUN chown`/`chmod` | The same cell twice: one solid, one hatched; the Engineer surprised | Paths, bytes | Later |
| `healthcheck`, `onbuild` | `HEALTHCHECK`, `ONBUILD` | A pulse monitor on the hull; a sealed capsule labeled for a future build | Command, trigger | Later |
| `chapter` | Stage start | The setting's establishing shot, by day if the stage ships and in its night variant if not, with the chapter caption | Stage name, base, ships or not, setting | Yes |
| `countdown` | Final instructions of the target stage | Narrow panels with countdown captions; the crew braced | Remaining lines | Yes |
| `inspection` | — | The setting's Inspection set piece around the scan gate and mass readout; bays drop away; the reaction strip | Ledger, hidden bytes, stages, budget | Yes |
| `ignition` | — | The setting's departure cue. The launch card slots into the console. The Pilot boards and looks down through translucent slabs. Docking caps on `/proc`, `/dev`, `/sys`, `/etc/hosts`. Word-slice's confetti and stars on a triumphant ending | `argv`, `User`, `WorkingDir`, merged top-level directories, mounts | Yes |
| `cover`, `cast`, `cold-open`, `back-matter` | — | §8.1 | Question, top panel, ledger, versions, setting | Yes |

### 9.9 Figures

Everything is drawn by code from tables in the style pack. As in word-slice, every shape is built from rounded rectangles, ellipses, polygons, and gear and star paths. No art kit is drawn or commissioned, and the binary embeds only fonts and style tables.

#### Cartoon rig (from napkin-chats)

The rig is copied and adapted from napkin-chats' `StickFigure`:

- **Joints.** A table of named joints in a fixed view box. Napkin-chats' joints (head, neck, shoulders, elbows, hands, and hip) are extended with a pelvis, knees, and feet. Every figure is full-body.
- **Presets as overrides.** Each body preset is a partial override of the neutral joint table, as napkin-chats' `pose({...})` does. Two scalars join the joints: `lean` and `headTilt`.
- **Mirroring.** `orientPose` mirrors x for the facing direction.
- **Blending.** Eased interpolation from neutral to a preset, by the strength from §8.5.

The proportions are cartoon, not napkin-chats' realistic stick figure:

- **Head and body.** The figure is about 2.75 heads tall, with a big round head. The body is a rounded pear.
- **Limbs.** Thick round-capped strokes drawn with word-slice's two-tone cable technique: a wide ink stroke under a narrower stroke in the garment's color, so limbs come out outlined.
- **Reach.** At these proportions the arms reach only about as high as the top of the head. No preset raises a hand above it. A pose that needs height lifts the whole figure, with a jump or a step onto a crate, rather than stretching the arms. The saloon mock-up found that "arms braced overhead" cannot be drawn on this rig.
- **Hands and feet.** Mitten hands with an optional thumb nub, for pointing and thumbs-up. Rounded footwear from the costume.
- **Skin.** Faces and hands are `paper` colored, so the figures carry no skin-tone signal. The figures are identity-neutral; the costume grammar assigns garments by role, never by gender, and mixes them freely.
- **Finish.** Every figure has a 3 px ink outline and casts a hard ellipse shadow below its feet, as word-slice's buckets do. Squash and stretch apply on jumps and landings.

#### Face rig

Cartoon faces are big and simple. Feature positions are given in head units, where the head radius is 11:

| Feature | Parameters | Range |
| --- | --- | --- |
| Brows | Thick rounded ink bars. `browInner` and `browOuter` set the vertical offset of each end (+ is raised), per side, so they can be asymmetric | −2.5 to +2.5 |
| Eyes | Paper ovals with ink outlines and ink pupils with a paper catch-light. `eyeOpen` (0 closed, 1 normal, 1.5 wide); `lid` (upper-lid drop, for sly or tired looks); `eyeShape` (oval, happy arc, squeeze, or spiral for dazed); pupil gaze in x and y; pupil size | Per preset |
| Mouth | Filled ink shapes. `mouthShape` (line, curve, O, grimace with paper teeth, wavy, open grin with a tongue in a set tone); `curve` (−1 frown to +1 smile); `open`; `skew` (a smirk) | Per preset |
| Cartoon symbols | Sweat drops, surprise lines, steam puffs from the ears (anger), circling stars (dazed), `!` and `?`, a breath puff, a sigh line, focus lines behind the head | On or off; most switch on at level 3 |

The owner set the anchor of the brow system: **brows pointing down at the inner ends signal anger or malice, and inner ends raised signal worry or strain.**

Below a minimum rendered head size (59 px on a 300 dpi page, 44 px on a card), the face drops to brows, eyes, and mouth only, without lids, catch-lights, or symbols. Reaction insets (§8.4) keep important faces above that size.

#### Expression presets

Each preset sets face, body, and costume reaction together as partial overrides of neutral. They are seeded from napkin-chats' six poses: neutral, speaking, listening, thinking, concerned, and satisfied.

| Preset | Brows (inner, outer) | Eyes | Mouth | Arms and body | Symbols and costume reaction |
| --- | --- | --- | --- | --- | --- |
| focused | level (0, 0) | open, toward the task | line | working pose | — |
| determined | slightly down (−0.8, +0.3) | slightly narrowed | firm line | lean forward, wide stance, fists at the waist | — |
| strained | up (+1.2, −0.6) | squeezed | grimace | arms hugging the load at chest height, knees bent | sweat drops, shake lines |
| satisfied | relaxed and raised (+0.4, +0.4) | happy arcs | smile | hands on hips, or dusting off | — |
| relieved | raised at the inner ends (+0.8, 0) | soft, half closed | small smile | shoulders drop, a hand on the chest | breath puff |
| triumphant | high (+1.2, +1.2) | open | open grin | jumping, both arms up | stars; the hat lifts a little |
| surprised | high (+2, +2) | wide, small pupils | O | lean back, palms out | surprise lines |
| shocked | high (+2.4, +1.8) | wide | open grimace | recoil, hands to the head | surprise lines, `!`; **the hat or wig pops off** |
| worried | up at the inner ends (+1.6, −0.4) | open | wavy | shoulders raised, hands clasped | one sweat drop |
| alarmed | up at the inner ends (+2, +0.4) | wide | open, wavy | one hand raised, palm out | `!` |
| suspicious | asymmetric (+1.4 / −0.8) | narrowed, side glance | line, offset to one side | hand to chin, head tilt | `?` |
| wry | one side raised (+1.0 / 0) | half lid | small smirk | one hand gesturing | — |
| angry | steeply down at the inner ends (−2, +1.0) | narrowed | frown or grimace | lean forward, pointing, the other hand a fist | **steam from the ears**, focus lines |
| sly | down at the inner ends (−1.4, +0.6) | half lid, side glance | smirk | crouched, fingers steepled | the cape or coat flares; a shadow in pink-red |
| caught | very high at the inner ends (+2.2, +0.5) | wide | grimace | hands up, palms out | surprise lines, `!`; the hat pops off |
| resigned | slightly up at the inner ends (+0.8, −0.2) | half lid | small flat line | shrug: shoulders up, palms up low | sigh line; plumes droop |
| sulking | slightly down at the inner ends (−0.6, 0) | half lid, eyes down | small frown | arms crossed, slumped | a small storm cloud overhead |
| serene | relaxed (+0.3, +0.3) | gentle | soft smile | upright, one hand on the console | soft green glow rings |

**Staging also carries emotion,** by fixed rules:

- A low camera on a sly Stowaway makes it loom; a high camera on a straining Engineer makes the load loom.
- Figures in conflict face each other at close range; allies stand side by side.
- At level 4, focus lines radiate behind the reacting face.

#### Costume grammar

A costume is a recipe of parameterized parts attached to rig anchors. Each part is drawn with the same ink outline and flat set tones, and moves with the joints:

| Slot | Part types (parameters) |
| --- | --- |
| Headwear | Brimmed hat (crown height and shape, brim width, band); tricorn and bicorne; top hat; cap (peak, band); helmet (open face, crest, fin, bubble); crown and laurel; powdered wig (curl rows, height, ribbon); pouf wig with a topper; headscarf and bandana; hood; beanie and ear-flap hat; headset; beret; feather or plume |
| Neckwear | Bandana and neckerchief; lace jabot (frill count); ruff; bead collar; bow tie; necktie; scarf; measuring tape |
| Garments | Vest; frock coat and tailcoat (tail length); duster; tunic; toga drape; kilt; overalls; leotard; parka; flight jacket; spacesuit; smock; apron; cape; gown or suit |
| Waist | Belt, sash, utility belt |
| Footwear | Boots, buckled shoes, sandals, oversized shoes, moon boots |
| Props | Tool, stamp, recording object, loot sack, steering or performing object (per role), plus setting props such as a lorgnette, fan, or lasso |
| Role mark | One garment or accessory in the role's semantic color |

**Rules for every costume:**

1. **The face rig is never covered.** Brims, wigs, and helmets sit above the brow line. Goggles and glasses are pushed up above the brows or have clear lenses below them. Masks, eyepatches, veils, face paint over features, and facial hair over the mouth are excluded. Bubble helmets are clear.
2. **Every role keeps its color mark and signature prop type** in every setting.
3. **Settings are shown through clothing, architecture, and props,** never through caricatured features. The rig is the same for everyone.
4. **Secondary motion is deterministic.** Coat tails, capes, plumes, and wig curls bend by the figure's lean, and costume reactions follow the preset table.

`imagestoryctl styles --model-sheet --setting <id>` draws the crew in every preset at 40%, 70%, and 100% strength. `--costume-sheet` lines up all twelve crews and their extras at neutral. Together they are the review artifacts for the cast.

**Tests**, extending napkin-chats' `StickFigure.test.ts`:

- Every preset is complete and finite.
- Bone lengths stay within 6% of neutral.
- No hand rises above the top of the head.
- Standing presets keep both feet on the floor line.
- Face features stay inside the head.
- Mirroring is exact.
- Every anger or malice preset has `browInner` below zero, and every worry or strain preset has it above zero.
- Geometric check: no costume part overlaps the brow, eye, or mouth regions in any preset.
- Every costume carries its role mark and signature prop.

### 9.10 Settings

Each setting is one YAML file. It defines:

- an era and place;
- set tones;
- six **sets**, one for each narrative job: Arrival, Workshop (layers), Notice spot (config), Night variant (non-shipping stages), Inspection set piece, and Ignition;
- crew costumes and titles;
- extras;
- signature props;
- cue lines.

Sets are assembled from a shared set grammar:

- walls with material patterns: planks, stone blocks, wallpaper, brick, ice, metal panels, canvas stripes;
- openings: arches, swinging doors, portholes, portcullis, stage proscenium;
- structure: rafters, columns, iron trusses, rigging, masts, domes;
- horizons: mesas, seas, dunes, ice shelves, Saturn over a methane lake;
- a few signature props per setting, drawn from primitives.

Every set exposes staging marks at floor, raised, and high levels, and mounting points for the futuristic kit: where the gantry hangs, where the conveyor runs, and where the mast stands.

#### 9.10.1 Choosing settings

- **Home setting:** `--setting`, or by default the setting at index `sha256(Dockerfile bytes) mod 12`. The same Dockerfile always lands in the same world.
- **Other stages:** `--settings stage=id`, or by default the next settings after the home in a fixed **contrast ring**. The ring orders settings so that neighbors differ in era, climate, and indoor or outdoor:

  saloon → Titan → opera → polar station → pirate galleon → radio studio → pyramid works → couture atelier → castle → circus → colosseum → railway → (back to saloon)

- Non-shipping stages use their setting's night variant.
- The target stage's setting hosts the countdown, the Inspection, and the ignition.
- A hand-off between stages in different settings is drawn as a split panel with the maglev tube crossing worlds (§9.8).

#### 9.10.2 The twelve settings

Role marks: the **Engineer** in blue, the **Inspector** in gold, the **Stowaway** in pink-red, and the **Pilot** in green. Set tones are starting values, to be tuned against the chroma and dark-tone rules (§9.2).

**1. Frontier saloon** (American West, 1880s; `frontier-saloon`; MVP)
- **Set tones:** weathered wood `#a68a6d`, dust `#e6d5b8`, sage `#9aa58e`, adobe `#c9a38a`, and saddle `#6b5544` (the dark tone, at OKLCH chroma about 0.04).
- **Sets:**
  - Arrival: Main Street in front of the swinging doors.
  - Workshop: the saloon floor by the piano, with the gantry hung from the rafters.
  - Notice spot: the wanted-poster wall.
  - Night: the moonlit livery stable.
  - Inspection: a high-noon showdown mid-street, with a tumbleweed.
  - Ignition: the edge of town at sunset by the water tower.
- **Crew** (minimal cowboy attire):
  - Prospector: flat-crowned hat, rolled sleeves, suspenders, **blue bandana**, a pick-wrench.
  - Sheriff: tall-crowned hat, vest, **gold star badge**, stamp and ledger.
  - Outlaw: low hat with a **pink-red band**, duster coat, bandana around the neck (never over the mouth), loot sack.
  - Stagecoach Driver: wide-brim hat, **green neckerchief**, reins.
- **Extras:** piano player, card players at the bar, bartender.
- **Props:** swinging doors, upright piano, barrels, cactus, water tower, a wanted poster that shows the Outlaw after the Inspection.
- **Cues:** "High noon." / "Hyah!"

**2. Habsburg opera house** (Vienna, 1780s; `opera-vienna`; MVP)
- **Set tones:** gilt `#c2a878`, plum velvet `#7d6577`, ivory `#efe7d6`, marble `#d8d3cb`.
- **Sets:**
  - Arrival: the stage trap, with the gantry dropping from the fly loft.
  - Workshop: the stage in front of painted backdrop flats.
  - Notice spot: the playbill board in the foyer.
  - Night: the dark fly gallery among ropes and sandbags.
  - Inspection: the final act under the great chandelier, with the audience in the boxes.
  - Ignition: curtain up.
- **Crew** (powdered wigs and lace cravats):
  - Stagehand: short powdered wig tied with a **blue ribbon**, waistcoat, coil of rope.
  - Impresario: towering powdered wig, frock coat with **gold braid**, lace jabot, lorgnette and seal.
  - Gatecrasher: wig askew, **pink-red lace cravat**, hides in the prompter's box.
  - Leading Singer: a 1770s-fashion pouf wig topped with a tiny ship, **green sash**, fan.
- **Extras:** the orchestra in the pit; the audience in boxes, with fans and lorgnettes.
- **Props:** chandelier, footlights, backdrop flats, fly ropes, prompter's box.
- **Cues:** "The final act." / "Curtain up!"

**3. Titan moon base** (space rangers, far future; `titan-base`; MVP)
- **Set tones:** haze apricot `#cfa98a`, regolith `#8f8274`, hull cream `#e4e0d6`, methane-lake slate `#6c7a80`.
- **Sets:**
  - Arrival: the landing pad by the methane lake, with Saturn in the sky.
  - Workshop: the dome hangar.
  - Notice spot: the mission board.
  - Night: the cryo hold under Saturn-shine.
  - Inspection: the airlock decontamination arch.
  - Ignition: the launch pad.
- The base's retro rounded architecture is deliberately unlike the hex-and-slate build kit, so the Dockerfile's machinery still stands apart.
- **Crew** (space rangers):
  - Base Mechanic: clear bubble helmet, utility belt, **blue suit stripe**, magnetic wrench.
  - Ranger Captain: helmet fin, **gold star insignia**, scanner wand and log tablet.
  - Smuggler: hooded cape with **pink-red lining**.
  - Shuttle Pilot: **green flight suit**, goggles above the brows inside the bubble.
- **Extras:** rangers in the mess dome.
- **Props:** rover, antenna mast, dome struts, Saturn.
- **Cues:** "Airlock check." / "Liftoff!"

**4. Pirate galleon** (Caribbean, 1710s; `pirate-galleon`)
- **Set tones:** timber `#8e735a`, sea-glass `#86a3a0`, sailcloth `#ece4d2`, tar `#4f4a45`.
- **Sets:**
  - Arrival: the harbor dock.
  - Workshop: the gun deck.
  - Notice spot: the chart wall in the captain's cabin.
  - Night: the hold by lantern light, where stowaways belong.
  - Inspection: the harbor master boards for inspection.
  - Ignition: raising the sails.
- **Crew:**
  - Ship's Carpenter: **blue head scarf**, striped shirt, mallet.
  - Harbor Master: bicorne hat, **gold epaulettes**, logbook and seal.
  - Stowaway: **pink-red bandana**, peeking from a barrel.
  - Captain: tricorn with a **green feather**, at the wheel.
- **Extras:** deckhands in the rigging, plus a parrot as a prop.
- **Props:** masts and rigging, lanterns, barrels, gangplank, ship's wheel.
- **Cues:** "Stand by to be boarded." / "Set sail!"

**5. Pyramid works** (Old Kingdom Egypt; `pyramid-works`)
- **Set tones:** sandstone `#d6bf98`, papyrus `#e8dfc4`, shadow ochre `#a88f6c`, river slate `#7e9a9c`.
- **Sets:**
  - Arrival: the foot of the construction ramp.
  - Workshop: the masons' yard.
  - Notice spot: the scribes' court.
  - Night: a torchlit gallery.
  - Inspection: a great balance scale, where the mass readout sits.
  - Ignition: a river barge casting off at dawn.
- No deities or religious imagery appear.
- **Crew:**
  - Master Builder: linen kilt, **blue bead collar**, plumb line.
  - Royal Scribe: **gold collar**, papyrus scroll and reed pen.
  - Tomb Robber: **pink-red sash**, sack.
  - Barge Captain: **green headband**, steering oar.
- **Extras:** rope-hauling work gangs.
- **Props:** sledges, ramps, obelisk, palms.
- **Cues:** "To the great balance." / "Cast off!"

**6. Medieval castle** (High Middle Ages; `castle`)
- **Set tones:** stone `#9d978d`, wine banner `#86656c`, moss `#7f8a6a`, oak `#8a7258`.
- **Sets:**
  - Arrival: the drawbridge.
  - Workshop: the castle forge.
  - Notice spot: the heraldry wall in the great hall.
  - Night: the dungeon.
  - Inspection: the gatehouse, where the scan gate is built into the portcullis.
  - Ignition: trumpeters on the ramparts.
- **Crew:**
  - Blacksmith: leather apron, **blue tunic**, hammer.
  - Herald: tabard with **gold heraldry**, scroll and seal.
  - Jester: motley with **pink-red** panels and bells.
  - Knight: visor up, **green plume**, pennant.
- **Extras:** guards, minstrels, cooks.
- **Props:** portcullis, banners, torches, anvil.
- **Cues:** "Halt! Who goes there?" / "Open the gates!"

**7. Victorian railway terminus** (1890s; `railway`)
- **Set tones:** iron green `#5f7465`, brick `#a07463`, steam grey `#c8cbc6`, soot `#4b4f52`.
- **Sets:**
  - Arrival: the platform under the iron-and-glass roof.
  - Workshop: the engine shed.
  - Notice spot: the ticket office's timetable board.
  - Night: the goods yard under gas lamps.
  - Inspection: the platform gate.
  - Ignition: departure in a burst of steam.
- **Crew:**
  - Engine Fitter: overalls, **blue cap**, oil can.
  - Ticket Inspector: peaked cap with a **gold band**, ticket punch and notebook.
  - Fare Dodger: top hat and **pink-red-lined cape**, hiding in the luggage van.
  - Engine Driver: **green cap**, goggles pushed up above the brows.
- **Extras:** porters, passengers with umbrellas, a newspaper seller.
- **Props:** station clock, luggage trolleys, signal lamps.
- **Cues:** "Tickets, please." / "All aboard!"

**8. Roman colosseum** (Imperial Rome; `colosseum`)
- **Set tones:** travertine `#e0d6c3`, terracotta `#b68a72`, cypress `#5f6b55`, bronze `#9a8366`.
- **Sets:**
  - Arrival: below the aqueduct.
  - Workshop: the lift-filled workrooms under the arena.
  - Notice spot: the forum's notice wall.
  - Night: the catacombs.
  - Inspection: the arena floor before the magistrate's box.
  - Ignition: the chariot starting gates.
- **Crew:**
  - Roman Engineer: **blue tunic**, surveying tool.
  - Magistrate: toga with a **gold border**, wax tablet and stylus, laurel.
  - Pickpocket: **pink-red cloak**, coin pouch.
  - Charioteer: **green tunic** (the Greens were a real racing faction), crested helmet.
- **Extras:** the crowd in the stands.
- **Props:** columns, arches, chariots, banners.
- **Cues:** "The magistrate decides." / "Release the chariots!"

**9. Antarctic ice-core station** (present day; `polar-station`)
- **Set tones:** ice `#dbe6ec`, snow shadow `#aebccb`, horizon `#97aab9`, survival orange `#d4a383`.
- **Sets:**
  - Arrival: the ice runway.
  - Workshop: the drill tent with racks of banded ice cores (layers, again).
  - Notice spot: the radio hut's log board.
  - Night: polar night with the aurora.
  - Inspection: the weigh station before loading. This is the natural home for `--budget`.
  - Ignition: a ski-plane takeoff.
- **Crew:**
  - Ice Driller: **blue parka**, ice drill.
  - Station Manager: high-visibility vest with **gold** stripes, clipboard and stamp.
  - Crate Stowaway: beanie, **pink-red scarf** (below the mouth), hiding in a cargo crate.
  - Bush Pilot: **green flight jacket**, ear-flap hat.
- **Extras:** scientists in parkas.
- **Props:** fur-trimmed hoods that frame faces, flags, sled, ski plane, aurora.
- **Cues:** "Weigh-in before takeoff." / "Wheels up!"

**10. Big-top circus** (1900s; `circus`)
- **Set tones:** tent cream `#f1e5cb`, dusk plum `#806a7f`, sawdust `#d8bf98`, brass `#b39b70`.
- **Sets:**
  - Arrival: circus wagons rolling in.
  - Workshop: the practice ring.
  - Notice spot: the poster wagon.
  - Night: the wagons by moonlight.
  - Inspection: the center ring under the spotlight.
  - Ignition: the human-cannonball launch.
- **Crew:**
  - Strongman Rigger: **blue striped** leotard, dumbbell, hauls slabs.
  - Ringmaster: top hat, tailcoat with **gold frogging**, cane and megaphone.
  - Clown: **pink-red ruff**, big shoes. Only a round nose; no face paint over features.
  - Human Cannonball: **green helmet** and cape.
- **Extras:** the audience in the stands, acrobats on the trapeze.
- **Props:** center ring, trapeze, cannon, bunting.
- **Cues:** "All eyes on the center ring!" / "Fire the cannon!"

**11. Golden-age radio studio** (1940s; `radio-studio`)
- **Set tones:** walnut `#7c6553`, acoustic tile `#ddd6c8`, deco brass `#b5a178`, studio blue-grey `#7f8b96`.
- **Sets:**
  - Arrival: the lobby elevator.
  - Workshop: the studio floor among ribbon microphones.
  - Notice spot: the control-booth window under the ON AIR sign.
  - Night: the rooftop transmitter tower.
  - Inspection: a ten-second countdown clock.
  - Ignition: the ON AIR light.
- **Crew:**
  - Sound Engineer: **blue headset**, cable coil.
  - Station Manager: **gold tie**, stopwatch and script.
  - Rival-Station Spy: trench coat, fedora with a **pink-red band**, behind the curtain.
  - Announcer: **green bow tie**, at the microphone.
- **Extras:** the studio orchestra, and sound-effects artists with coconut shells and a door on a frame.
- **Props:** ON AIR sign, ribbon microphones, an applause sign.
- **Cues:** "Ten seconds to air." / "You're on the air!"

**12. Couture atelier** (Paris, 1950s; `couture-atelier`)
- **Set tones:** dove grey `#c9c5be`, chalk `#f1ece3`, toile `#d9c9ab`, slate `#6e7884`.
- **Sets:**
  - Arrival: the delivery courtyard.
  - Workshop: the cutting room among dress forms and stacked fabric bolts.
  - Notice spot: the wall of pinned paper patterns.
  - Night: the stockroom after hours.
  - Inspection: the runway show before the critics.
  - Ignition: the grand salon reveal.
- **Crew:**
  - Tailor: smock, **blue measuring tape**, pincushion cuff, shears.
  - Head Critic: **gold-rimmed** spectacles below the brows, notebook.
  - Copycat Sketcher: **pink-red beret**, hidden sketchbook.
  - Runway Model: **green** ensemble.
- **Extras:** seamstresses, photographers with flashbulbs.
- **Props:** dress forms, fabric bolts, chandelier, runway lights.
- **Cues:** "The critics are seated." / "Lights. Music. Walk."

### 9.11 The futuristic build kit

The kit is the same in every setting. It is drawn in word-slice's machinery style:

- slate chassis (`#2f3b46`, `#3f4c58`) with light seams (`#d6dde5`) and two-tone cables;
- rounded corners, gears, and the hard offset shadow;
- the hex as its signature shape, so "this is the Dockerfile's stuff" reads at a glance;
- colored glows in the semantic colors.

**Drawing over slate.** Ink outlines barely show against the kit's slate: ink is about 1.4:1 on `#2f3b46` and 1.9:1 on `#3f4c58`. Two rules follow:

- A kit piece drawn open, such as a layer's hold with cells or a figure inside, uses the lighter `#3f4c58` inside and the darker `#2f3b46` on its rim.
- Any figure, prop, or lettering drawn over slate gets a 2 px paper keyline outside its ink outline. Dark set tones need it most: saddle `#6b5544` is about 1.3:1 on `#3f4c58`.

| Kit piece | Represents | Drawing |
| --- | --- | --- |
| Gantry and claw | Delivering a layer | Word-slice's robot arm: a two-tone cable from above, a rounded claw block, splayed claw fingers |
| Hull section | A base image | A sealed slate capsule with a riveted seam and a stencilled reference |
| Hex slab | A layer | A chamfered hexagonal slab with a blue seam glow; thickness ∝ bytes on the spine |
| Cargo cell | Files | A small hex cell with a stencilled path; face area ∝ bytes in a squarified treemap |
| Conveyor | `COPY` | Word-slice's belt: slate bed, diagonal light stripes, gears at the rollers |
| Config mast and tags | Config | A slate pole with gold chevron tags; a tag reading `root` or a non-root user name, turning green when not root |
| Transmission capsule | `ARG` | A slate capsule with signal arcs; it dissolves after reading |
| Cloak | A whiteout | A paper-colored cloak; the hidden cell beneath is drawn in pink-red dots |
| Hatch and locker | Build mounts | Slate doors that open during the step and seal after |
| Maglev tube | `COPY --from` | A two-tone tube with light rings; it can cross a split panel between settings |
| Scan gate and mass readout | The Inspection | A hex-framed arch with a beam, and a readout screen whose figures are in the code face |
| Launch cradle, card, console | The image and its config | A cradle that holds the stack, a gold launch card printed with the argv, and a slot console |
| Docking caps | Runtime mounts | Slate caps over paths, labeled "Docker's defaults" |
| Stack spine | The image so far | A slim slate column at the page edge (§9.5) |

Each setting gives the kit **mounting points**, such as the rafters, the fly loft, a mast, or a dome strut. The kit hangs from them but is never restyled to match.

### 9.12 The cartoon finish

Everything follows word-slice's finish, scaled from card pixels to each edition:

- **Fills.** Flat fills only. The single gradient per set is the sky, as in word-slice's game screen.
- **Outlines.** `ink` outlines: 4 px on panels, 3 px on figures, props, and balloons, 1.5 px on small details.
- **Shadows.** Hard offset shadows with no blur: the same shape at about 22% ink, shifted (5, 7) px for objects and (6, 7) px for panels. Figures cast an ellipse below their feet.
- **Highlights.** One white stripe at 28% on the left of glossy shapes: slabs, cells, helmets, boots, the hull.
- **Lines.** Two-tone strokes for cables, ropes, tubes, and limbs.
- **Corners.** Rounded corners of 8 px on panels, balloons, and boxy props.
- **Celebration.** Word-slice's shapes, placed in fixed patterns rather than at random: five-point stars, ribbons, and confetti dots. A clean build earns a gear-rimmed medallion with a check on the cover seal and the Inspection splash.
- **Pattern.** No texture except the pink-red dots on hidden content.

`--detail draft` drops shadows, highlights, extras, and set detail, for fast previews and `--watch`.

### 9.13 Where the detail comes from

"As visually detailed and vibrant as possible" is met in this order of value:

1. **Real detail from the build.** Cargo cells stencilled with real paths, treemaps sized by real bytes, the spine, receipts, owner stamps, and the readout. This detail is also true.
2. **Twelve settings.** Each brings its own architecture, props, extras, and set pieces.
3. **Expressive cartoon figures.** Faces, bodies, cartoon symbols, costume reactions, and staging, mapped from beats and tension.
4. **The futuristic kit.** Machinery against period sets.
5. **The cartoon finish and lettering craft** (§9.12, §11).
6. **Composition driven by tension.** Tilts, tightening gutters, reaction insets, shot variety, border breaks, the countdown tier, and splash pages.

### 9.14 Later style packs

A style pack can restyle everything except the semantic roles, the arc, the emotion table, the brow rule, and the rule that the machinery reads as futuristic. Candidates: **Storyboard** (graphite and marker frames with camera notes) and **Blueprint** (cyanotype, with sizes as dimension lines). Settings are shared across packs.

## 10. Layout Engine

1. **Weights.** Each panel gets a weight from its archetype, beat score, and tension: small (4 columns), half (6), wide (8), full tier (12), or splash (whole page).
2. **Tier packing.** Dynamic programming over the panel sequence, in the manner of Knuth–Plass line breaking. Penalties apply to:
   - a stretched small panel;
   - a stage boundary in the middle of a tier;
   - a tier whose cell aspect ratio fits no variant of its archetypes;
   - a page with only one tier;
   - a face below the minimum size without a reaction inset;
   - a repeated shot type in adjacent panels.
3. **Page packing.** Tiers are packed into pages by the same method, under the fixed pacing rules of §8.4: the countdown before a page turn, the Inspection as a splash after it, and the cold open alone.
4. **Variants, sets, staging, and framing.**
   - Each panel gets its setting's set for the narrative job, a shot variant by the rotation rule, and staging marks at varied heights.
   - Figures are mirrored to face their listener.
   - Tension framing (§8.3) and reaction insets (§8.4) are added last.
5. **Lettering placement.** Captions go in their fixed corners. Balloons go into each set's quiet zones in reading order, with tails to the speaker's mouth. The figure who speaks first sits further left. Collisions with faces, kit pieces, other balloons, and sound effects fail the layout, and the packer retries with the next variant.
6. **Editions.** Cards lay out each panel independently with its own variant. The reader embeds the page SVGs plus a panel index.

Ties are broken by panel order, so the same inputs always give the same pages.

## 11. Lettering

- **Shaping.** `go-text/typesetting` shapes Recursive and Bungee at the chosen axis values, and the lettering is emitted as **SVG outlines**. Chrome therefore never re-lays out text. The metrics used for fitting are exactly those drawn, and fonts need no embedding. The reader edition carries the same copy as real text (§15).
- **Balloon-shaped breaking.** Line widths follow a superellipse, so the text forms a lozenge, as a hand letterer would set it. A dynamic-programming pass minimizes raggedness within that shape.
- **Balloons.** Each balloon is drawn as a word-slice card: paper fill, ink outline, hard shadow. Its outline follows the speaker's style: smooth, squared, dashed, jagged, or burst.
- **Breaking rules.** No hyphenation. Code and paths break only after `/`, and Dockerfile lines break at flag boundaries.
- **Sound effects.** Bungee layers are set as outlines and warped along an arc or perspective fixed by size class.
- **Emphasis.** Weight only (`wght` 950). Angry and alarmed balloons letter in the heaviest weight. Color is never used for emphasis.

## 12. Rendering

- **Chrome.** `chromedp` drives a pinned Chrome for Testing 157. On first use, the binary downloads into the user's cache directory; `--chrome PATH` uses a system Chrome and records its version in the manifest.
- **Outputs.**
  - PNG via `Page.captureScreenshot` at the edition's device scale factor (300 dpi for pages, 1× for 1080 cards).
  - PDF (later) via `Page.printToPDF`, with lettering kept as vectors. The PDF is RGB; CMYK separation for offset printing is out of scope.
- **Contact sheet.** Every panel as a thumbnail with its line number, setting, shot, tension level, and each figure's preset. It is for reviewing the arc, the emotional curve, and shot variety at a glance, as in STORYBOARD_PLAN.
- **Budgets.** Rendering only, excluding the build. Provisional, to be confirmed by spike S2:
  - A 20-instruction Dockerfile at full detail renders all editions in under 30 seconds on a laptop.
  - Draft detail takes under 5 seconds.
  - `--watch` re-renders one changed page in under 2 seconds.

The flat finish has no filters, so it should render faster than the earlier print texture.

## 13. Copy Rules

All text comes from rule tables in the style pack, the settings, and the beat catalog. There is no model, no runtime text generation, and no randomness.

| Copy | Source | Rule |
| --- | --- | --- |
| Line caption | The instruction | Its text, redacted, in the code face; long lines break at flag boundaries |
| Receipt | Layer, config, and stage facts | One template per archetype and condition, for example "Adds {entryCount} files ({storedBytes}), mostly under {top[0].path}" |
| Dialogue | `voices.yaml` | First matching row per role, event, and preset (§8.6); speakers are named by their setting title in alt text |
| Setting cues and chapter flavor | `settings/<id>.yaml` | The Inspection and ignition cues; "Meanwhile, at …" chapter captions |
| Countdown, stakes, answer captions | `copy.yaml` | Fixed templates with placeholders |
| Editor's notes | Beat catalog (§7) | Shown per `--notes` |
| Alt text | Archetype templates | The line, the setting, each figure's title and emotion in words, the dialogue, and the receipt, in reading order |

Every template is reviewed against the language guide in IMAGE_VIZ_PLAN §5.6:

- "layer" only for a filesystem changeset;
- "whiteout", never "deletion marker";
- "merged view", never "final layer";
- a mount is never a layer.

A banned-phrase list in `check` enforces this on the rendered output, including user edits to `script.json`.

## 14. Validation and Tests

`imagestoryctl check <dir>` runs these checks, and `board` runs them automatically before it reports success:

| Area | Checks |
| --- | --- |
| Schema | `facts.json`, `drama.json`, `settings.json`, `emotion.json`, and `script.json` validate, and every ID resolves |
| Truth | Every placeholder resolves. No literal figures or paths sit outside placeholders. Every panel cites at least one fact. The ledger adds up. Every layer is mapped to a line or drawn as "unknown line" |
| Drama | The question comes from the rule table and is answered by a cited fact on the Inspection. Act II never drops below its floor. The countdown is at least level 3, the Inspection is level 4, and the resolution is level 0. Tension never alters a data encoding |
| Settings | Each setting came from the selection rule or a flag. Non-shipping stages use night variants. The target setting hosts the climax. No adjacent panels repeat a shot type. No set is staged around a table. Every setting has a dark set tone |
| Emotion | Every figure's preset comes from an `emotions.yaml` row, with the row ID recorded. Strength matches the tension level. The Stowaway appears only after a qualifying beat, and the Pilot only at the resolution. An ironic expression shares its panel with the truth. The ending's mood matches whether complications remain |
| Costume | No costume part overlaps a face feature. Every role carries its color mark and signature prop. Set tones stay under the chroma ceiling and away from the semantic colors. No cartoon symbol, face feature, or body detail uses a semantic color, apart from the two named role-color effects |
| Machinery | Everything that carries data is a kit piece, and no setting restyles one. Dots appear only on hidden content. Cutaway interiors use the lighter slate, and anything drawn over slate has a paper keyline |
| Copy | Every caption, balloon, and note comes from a template row, with its ID recorded in `script.json`; no banned phrases |
| Semantics | No archetype draws a merge. Config never uses the slab form. Mounts are caps, never slabs. Hidden bytes are drawn as hidden. Non-shipping stages are night variants |
| Privacy | Sensitive patterns are scanned across drawn text, alt text, filenames, and the manifest. Anything in a sensitive class needs an `--allow-string` |
| Layout | No text overflows. Minimum lettering and face sizes hold. Balloons don't overlap faces, kit pieces, or each other. Tails reach their speaker. Reading order is monotonic. Sound effects never cover captions |
| Accessibility | Every panel has alt text that names the setting and each figure's emotion. All text and fill pairs reach 4.5:1 contrast. Semantic color pairs stay distinguishable by shape under simulated color-vision deficiencies |
| Determinism | Rendering twice yields identical SVG bytes and identical PNG hashes under the pinned Chrome |

**Corpus and tests.**

- **Unit tests** per package:
  - `internal/overlay` carries the ported `model.test.ts` cases.
  - `internal/drama` checks tension curves against expected tables.
  - `internal/figures` and `internal/costumes` carry the rig and costume tests (§9.9).
  - `internal/emotion` checks that every beat and act has a row for every figure who can appear.
  - `internal/settings` checks that every setting file defines all six sets, four costumes, extras, cues, staging marks at three heights, and kit mounting points.
- **The Layerviz examples.** All 18 Dockerfiles get small fixture contexts so they build without the network once base images are cached:
  - a tiny Go program for `the-handoff`;
  - a `package.json` and lockfile for `cache-cascade`;
  - a local tarball in place of the `example.invalid` URL for `deleted-still-shipped`.

  Base images are pinned by digest.
- **Public Dockerfiles.** About 30, from public repositories at pinned commits, with their contexts. Together with the examples they cover Node, Python, Go, Rust, and Java; distroless and Alpine bases; heredocs; `ONBUILD`; parallel stages; and a 60-instruction file.
- **Recorded facts are checked in** beside each Dockerfile. Render tests run from those facts without Docker, and a nightly job rebuilds them and reports drift.
- **Golden output.** Every corpus book is rendered in its hashed setting, and two fixed books are rendered in all twelve settings. SVGs, model sheets, and costume sheets are snapshotted, and PNGs are pixel-diffed with a small tolerance.
- **Cold reads.** Five engineers unfamiliar with the tool get a book and answer eight questions:
  - What did line N do?
  - Which line made the image big?
  - Why didn't the delete shrink it?
  - What ships?
  - What was the cover's question, and its answer?
  - Where did the story feel most tense?
  - How does each figure feel on three chosen panels?
  - Which things in this panel are the Dockerfile's, and which are scenery?

  The pass mark is four of five. The last question tests the rule that the world is costume and the machinery is truth.

## 15. Accessibility

- **Reader.** `reader.html` presents each panel as a `<figure>` with alt text, plus a transcript in reading order: the line, the setting, each figure and its emotion, the dialogue, the receipt, and the note.
  - Keyboard: arrow keys move between panels, Home and End jump, and `T` toggles the transcript.
  - Focus is always visible. Under `prefers-reduced-motion` there is no zoom animation.
  - It works offline under a strict CSP with no network requests.
- **Image files.** PNG outputs have alt text in `manifest.json`.
- **Alt text** is generated from facts, the setting, and the emotion table, for example: *"Line 7, RUN npm ci. In the saloon, the Prospector strains under a slab: 'Heaviest so far.' Adds 1,842 files (182.4 MB), mostly under /app/node_modules."*
- **Emotion is never the only carrier.** Captions carry the facts, and alt text names the feelings. Presets differ in silhouette as well as face.
- **Tension without color.** Tension is carried by tilt, gutters, balloon shape, faces, and suit pips as well as by pink-red flashes. Color never carries meaning alone (§9.2).
- **Review.** Phase 2 includes a manual screen-reader pass of the reader.

## 16. Delivery Phases and Exit Gates

Durations assume one engineer who knows Go and frontend rendering, with part-time review from someone who maintains production Dockerfiles. Every set, costume, and prop is drawn in code, so the art work is engineering time spent on the style pack.

### Phase 0 — Spikes (about 3 weeks)

- **S1, record.**
  - Import BuildKit v0.35's parser and build checks.
  - Adapt the Layerviz recorder, set up the builder automatically, and handle unnamed stages.
  - Map layers to lines (§6.4).
  - Build fixture contexts for `the-handoff` and `deleted-still-shipped`, and record them.
- **S2, rendering.**
  - Shape Recursive and Bungee to outlines with `go-text`.
  - Build balloon-shaped breaking.
  - Render through chromedp and Chrome 157.
  - Measure page time at 300 dpi, and check determinism across two runs and two machines.
- **S3, cast and finish.**
  - Port napkin-chats' rig to Go and re-proportion it as a cartoon.
  - Add the face rig, cartoon symbols, strength blending, and the costume grammar.
  - Port word-slice's finish.
  - Build the 18 presets and the frontier-saloon costumes, and render the model sheet.
  - Show the unlabeled model sheet to five people. The gate: four of five name each preset's emotional family.
- **S4, look and drama.**
  - Hand-stage the `deleted-still-shipped` arc in the saloon (cover, cold open, `run.cells`, `whiteout.cloak`, Inspection) at page and card size.
  - Run the beat finder, tension formula, and emotion table over the recorded examples, and plot each curve beside the presets and shots chosen.
  - Run a cold read with five engineers.
- **Exit gate:**
  - Every corpus layer maps to a line or is reported unmapped.
  - Lettering is exact.
  - Pages render within budget.
  - The model sheet reads correctly.
  - Every example's curve rises and resolves.
  - Four of five cold readers pass the questions in §14.

### Phase 1 — MVP (about 6–7 weeks)

- `board`, `record`, `script`, `render`, `check`, and `styles` with both sheets.
- Every beat in §7, the drama engine, the emotion table, setting selection, and the copy tables.
- The cartoon pack: the full cast and presets, the costume grammar, the futuristic kit, the set grammar, all MVP archetypes, and **three settings** (frontier saloon, Habsburg opera house, Titan moon base) with their night variants and split-panel hand-offs.
- Page, card, and reader editions; the contact sheet; the Layerviz bundle export.
- Redaction and the string review list; the corpus with fixture contexts and golden tests.
- **Exit gate:**
  - All 18 Layerviz examples and the public corpus produce books in each of the three settings that pass `check`.
  - Rendering from checked-in facts is deterministic.
  - A reviewer confirms every beat, caption, and emotion row against the IMAGE_VIZ_PLAN semantics and the truth-in-irony rule.

### Phase 2 — All twelve settings and editions (about 6 weeks)

- The other nine settings, about three days each, built mostly from the grammars.
- The remaining archetypes, the scroll edition, PDF, `--watch`, and `--budget`.
- The screen-reader review.
- **Exit gate:**
  - Every setting passes `check` on the two fixed books.
  - Every instruction in IMAGE_VIZ_PLAN §6 has an archetype.
  - The reader passes the accessibility review.

### Phase 3 — Motion and comparisons

- A motion comic, following STORYBOARD_PLAN's review gates: stills, then an animatic reviewed muted at phone size, then polish.
  - Pan and scan along the arc.
  - Word-slice's bounce and pop for sound effects and celebrations.
  - Napkin-chats' eased blending between presets.
  - Hats that really pop off.

  A deterministic clock drives frames for ffmpeg.
- `imagestoryctl diff old new` for a before/after spread, such as "this PR ships 214 MB less".
- A CI action that attaches a storyboard to pull requests that change a Dockerfile.
- Further style packs (§9.14) and settings.

## 17. Risks and Mitigations

| Risk | Mitigation |
| --- | --- |
| Builds are slow, need Docker, and run untrusted code | Per-stage builds share BuildKit's cache. `render` re-draws from facts without building. A disposable builder is recommended for untrusted Dockerfiles |
| A `RUN` that fetches something changes the facts between builds | `facts.json` is the reproducible artifact. The manifest records base digests. The nightly corpus rebuild reports drift |
| Settings drown the data, and readers mistake scenery for meaning | The world-is-costume rule; the identical futuristic kit; set tones under a chroma ceiling; the cold-read question that asks readers to separate the two |
| Twelve settings are a large drawing workload | Shared set and costume grammars do most of the work. Three settings in the MVP. About three days per later setting. Two fixed books test every setting |
| Costumes hide faces or clutter small panels | Automated face-overlap check; minimum face sizes; reaction insets; `--detail draft` |
| Settings stereotype a culture or era | Settings are shown through clothing, architecture, and props, never through caricatured features. Identity-neutral figures. No deities or religious imagery. Each setting gets a review pass before release |
| Expressions are misread | The unlabeled model-sheet test; presets differ in posture as well as face; alt text names every emotion |
| Emotion misleads: a happy face over a real problem | The truth-in-irony rule; expressions never contradict captions; the honest ending |
| The Stowaway casts a normal Dockerfile as villainous | It appears only for hidden bytes or leaked values, both real costs. The cast page labels it a metaphor |
| Figures or settings look derivative | Original rig and grammars drawn in code; napkin-chats' originality rule; no recognizable characters, franchises, or trademarks |
| A metaphor teaches something false, such as layers merging | The commitments in §9.7, archetype rules and banned phrases enforced by `check`, and cold reads |
| A private string reaches a shared image | Default redaction, never drawing secret-named values, the string review list, and `check` failing on sensitive classes |
| Chrome versions change rasterizing | A pinned Chrome for Testing build, recorded in the manifest; golden PNG diffs on upgrade |
| Huge Dockerfiles produce huge books | Config montages, the highlights cut, summary strips, and page-count tests on a 60-instruction file |
| Layer-to-line mapping fails on some builders | An "unknown line" panel instead of a guess; Spike S1; coverage reported in `check` |

## 18. Decisions

**Decided by the project owner on 15 November 2026:**

1. **Always build.** Every run builds and records the image. There is no static mode, and no input except a Dockerfile and its context.
2. **No generative AI.** Copy, figures, costumes, sets, and props come from deterministic rules applied to build facts.
3. **Figures with expressions carry the drama.** Human-like figures drawn in code on napkin-chats' approach. Eyebrows, eyes, mouths, arms, posture, and comic symbols map to emotional beats. Down-pointed inner brows signal anger or malice.
4. **Varied settings with costumes.** At least ten widely varied settings, with outfits and accessories that suggest each one, and no table staging. This plan specifies twelve.
5. **The machinery stays futuristic.** Everything that refers to the Dockerfile and its layers keeps the futuristic elements in every setting.
6. **Cartoony, in word-slice's style.** Confirmed by the owner after tone-viz turned out to have no cartoon elements.
7. **Little variation for its own sake.** One arc and fixed rules; books don't need to be unique.

**Recommended, not yet decided:**

8. **Setting selection.** *Recommended:* the home setting from the Dockerfile's hash; other stages along the contrast ring; flags to pin either.
9. **MVP settings.** *Recommended:* the three settings the owner named (saloon, opera, Titan) first; the other nine in Phase 2.
10. **Fonts.** *Recommended:* Recursive (casual sans and mono) plus Bungee, replacing Shantell Sans and Courier Prime to match word-slice's heavy cartoon weight.
11. **The cast.** *Recommended:* four roles tied to the semantic colors (Engineer, Inspector, Stowaway, Pilot), retitled per setting. The Stowaway appears only for hidden or leaked data.
12. **Narrative detail.** *Recommended:* the Inspection arc with a cover question, cast page, cold open, sparse dialogue, setting cue lines, reaction insets, and a tension-driven layout.
13. **`--budget`.** *Recommended:* include it as an optional flag. It gives the climax real stakes without the tool inventing a threshold.
14. **Language, repository, rasterizer.** *Recommended:* Go, in a repository separate from Layerviz, sharing `layerviz.bundle.v1`. Layerviz's recorder, napkin-chats' rig, and word-slice's drawing conventions are copied and adapted, not shared. Pinned headless Chrome through chromedp, with lettering as outlines from Go.
15. **Notes and editions.** *Recommended:* `explain` notes by default; pages, cards, and the reader by default; scroll and PDF in Phase 2.
16. **Name.** *Recommended:* keep `imagestoryctl` as the working name and decide before the first release.

## 19. Facts to Verify

- **BuildKit 0.35 / frontend 1.29:**
  - the `created_by` formats for each instruction;
  - whether max-mode provenance source locations cover every layer (shared with IMAGE_VIZ_PLAN §19);
  - whether the build-check package can be imported as a library;
  - the exact rule names `SecretsUsedInArgOrEnv` and `JSONArgsRecommended`.
- **Buildx 0.39:** creating a `docker-container` builder from the CLI, and the cost of one build per stage on a warm cache.
- **Recursive:** its axes (`MONO`, `CASL`, `wght` up to 1000, `slnt`, `CRSV`) and OFL license on Google Fonts. `go-text/typesetting` v0.3.5 must instance them and extract outlines for Recursive and Bungee.
- **Chrome 157 headless:** PNG determinism across Linux and macOS, and Chrome for Testing builds for linux/amd64, linux/arm64, and macOS.
- **Face legibility:** the minimum head sizes in §9.9, measured on a printed proof and on a phone, with the busiest costumes (opera wigs, Titan bubble helmets).
- **Chroma rule:** the OKLCH ceiling and ΔE distance in §9.2, tuned against all twelve settings' tones.
- **Historical details:** a quick accuracy pass on each setting's costume notes, for example the 1770s pouf wig with a ship and the Greens racing faction.

## 20. Definition of Done for the MVP

- `imagestoryctl board ./Dockerfile` builds, records, and produces pages, cards, the reader, a contact sheet, model and costume sheets, and a Layerviz bundle for every corpus Dockerfile in each MVP setting, and `check` passes.
- Every number on every panel resolves to a recorded or derived fact. Every word comes from a rule table, every expression from the emotion table, and every setting from the selection rule or a flag.
- Every book poses a question on its cover and answers it at the Inspection. Its tension curve rises through Act II and resolves to level 0, and the crew's faces follow it.
- Every Dockerfile element is drawn with the futuristic kit, in every setting.
- No user-originated string in a sensitive class reaches the output without review.
- Rendering from facts is deterministic under the pinned Chrome.
- Four of five cold readers pass the eight questions in §14.
- A reviewer who maintains production Dockerfiles has signed off on the beat catalog, the copy tables, the emotion table, and the archetype semantics.

## 21. Immediate Next Actions

1. Create the repository skeleton. Import BuildKit v0.35's parser, adapt the Layerviz recorder, and record `the-handoff` and `deleted-still-shipped` from new fixture contexts.
2. Port napkin-chats' rig to Go with cartoon proportions. Add the face rig, symbols, and the costume grammar; build the 18 presets with their tests; and render the first model sheet in frontier-saloon costumes.
3. Port word-slice's finish and machinery shapes (outline, hard shadow, highlight stripe, two-tone cable, gear, star, gantry, conveyor) into `internal/compose` and `internal/kit`.
4. Write the overlay port, the beat finder, the tension formula, the emotion table, and setting selection. Print both examples' tension curves with the presets, settings, and shots they select.
5. Hand-stage the S4 saloon pages for `deleted-still-shipped`, and render them through chromedp and Chrome 157 at page and card size.

## 22. References

- Sibling plans: [IMAGE_VIZ_PLAN.md](IMAGE_VIZ_PLAN.md) (Layerviz) and [STORYBOARD_PLAN.md](STORYBOARD_PLAN.md).
- The Layerviz application: `../layerviz` (recorder, model, analyzer, examples, implementation status).
- Word Slice: `../word-slice/index.html`. The cartoon finish and machinery shapes: `drawConveyor`, `drawGear`, `drawRobotArmCarries`, `drawBuckets`, `drawStarPath`, `drawCompletionMedallion`, and the card and celebration CSS.
- Napkin Chats: `../napkin-chats/src/components/StickFigure.tsx` and its tests (the figure rig), and `FUTURE_ENHANCEMENTS.md` (the originality rule for figures).
- Mort Walker, *The Lexicon of Comicana* (1980): the vocabulary of comic symbols such as plewds (sweat drops) and emanata (surprise lines).
- [Dockerfile reference](https://docs.docker.com/reference/dockerfile/), [BuildKit](https://github.com/moby/buildkit), and [Docker build checks](https://docs.docker.com/reference/build-checks/).
- [OCI image specification](https://github.com/opencontainers/image-spec) v1.1.1.
- [mvdan/sh](https://github.com/mvdan/sh), [go-text/typesetting](https://github.com/go-text/typesetting), and [chromedp](https://github.com/chromedp/chromedp).
- [Chrome Releases](https://chromereleases.googleblog.com/2026/): Chrome 157 stable, 3 November 2026.
- Fonts: [Recursive](https://fonts.google.com/specimen/Recursive) by Arrow Type; [Bungee](https://fonts.google.com/specimen/Bungee) by David Jonathan Ross.

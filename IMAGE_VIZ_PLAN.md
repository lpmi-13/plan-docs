# Layerviz: Seeing How a Dockerfile Becomes an OCI Image — Plan

> **Status: high-level plan.** This revision covers purpose, framing, scope, inputs, architecture, what Blender is for, the launch storyboard and trailer, and the preloaded examples. It deliberately leaves the animation of layers, the camera, and the scene's choreography to the next revision. The S1 disc-stack prototype in Phase 0 (Section 16) informs those.

## 1. Product Summary

Build a browser-based explorable explanation of how a Dockerfile becomes an OCI image, and how a container runtime turns that image into the single filesystem a process sees. It runs on three.js, TypeScript, and Vite. An engineer opens a preloaded example or pastes their own Dockerfile and steps through it one instruction at a time. For each instruction they see whether it added a filesystem layer, only changed the image's configuration, or existed only during the build, and which stages of a multi-stage build reach the final image. A closing step starts a container and shows what the process inside actually sees.

The project follows [Forkscape](https://github.com/lpmi-13/forkscape)'s design path (plan: [PROCESS_FORK_VISUALIZATION_PLAN.md](PROCESS_FORK_VISUALIZATION_PLAN.md)):

- a library of examples that opens on arrival;
- a fix replay for every example;
- a pure model that drives both a 3D scene and a keyboard-accessible flat view from one state;
- linked inspector and timeline panels; and
- reference observations kept separate from model values.

Two things are new compared with Forkscape:

- **User input.** Forkscape's scenarios are authored. Here, an engineer brings an arbitrary Dockerfile, a `docker save` archive, or a recorded build, and all of it is read in the browser.
- **A launch trailer.** The project gets its own storyboard and hype trailer, made from the preloaded examples with the story package from [STORYBOARD_PLAN.md](STORYBOARD_PLAN.md). This happens outside the app, which has no story or video export (Section 12).

The learner sees five linked things at once:

- a **3D scene** in which each stage is a cylinder of stacked glass discs, one per layer and sized by bytes. Looking down through the stack shows the container's view, and tapping a disc lifts that layer out to explore (Section 10.1);
- the **Dockerfile source**. Selecting a line highlights what that line produced, and selecting a file highlights the line that put it there;
- the **merged filesystem tree**. Each entry is attributed to the layer that supplies it, and the tree shows what that entry hides in lower layers;
- a **config card** that accumulates `Env`, `User`, `WorkingDir`, `Entrypoint`, `Cmd`, `ExposedPorts`, `Volumes`, `Labels`, and the rest. Each field names the instruction, or the base image, that set it; and
- a **size ledger** with download, stored, and visible bytes (Section 5.3).

The core promise:

> Point at any line of a Dockerfile and see what it left in the image, what it changed in the image's configuration, what it left only in the build, and what a container will finally see.

## 2. Why This Matters to an Engineer

Engineers read a Dockerfile like a shell script, top to bottom, one command per line. The image it produces is not a script's result. It is an ordered list of filesystem changesets plus a JSON configuration, produced by a build graph with a cache. Most recurring Dockerfile problems come from that mismatch: images that don't shrink after a delete, secrets that survive being "removed", builds that reinstall everything after a one-line change, and containers that run as the wrong user or the wrong command.

### 2.1 Symptom map

This table is the backbone of the example library (Section 13).

| Symptom | Mechanism | Evidence | Levers | Example |
| --- | --- | --- | --- | --- |
| "We `rm -rf` the download and the image didn't shrink" | A later layer adds a whiteout that hides the path. The bytes remain in the earlier layer and still ship. | `docker history`; the layer tar contains a `.wh.` entry; dive's wasted-space view | Delete in the same `RUN` that created the files; `ADD --unpack`; a multi-stage build; cache mounts | `deleted-still-shipped` |
| CI reinstalls every dependency on each commit | `COPY . .` comes before the install step, so any source change invalidates the cache for every later step | Build output with no `CACHED` steps after the `COPY`; `docker buildx history` | Copy the dependency manifests first; `RUN --mount=type=cache`; `.dockerignore` | `cache-cascade` |
| A scanner finds a token in an image that was "cleaned" | The token is a file in an earlier layer, a build-arg value recorded in history, or an `ENV` value in the config | `docker history --no-trunc`; extract the layer; `docker image inspect`; max-mode provenance | `RUN --mount=type=secret`; never pass secrets in `ARG` or `ENV` | `layer-three-secret` |
| One `chown` doubled the image | A metadata change copies every affected file into a new layer | Layer sizes in `docker history`; dive | `COPY --chown`, `COPY --chmod` | `chown-doubles` |
| Works as root; `permission denied` as the app user | `USER` doesn't change who owns files that `COPY` creates. They are `0:0` unless `--chown` is given | `ls -ln` in the container; config `User` | `--chown`; create writable directories before `USER` | `user-isnt-owner` |
| The container runs the base image's command, or ignores `SIGTERM` | Config is inherited from the base. `ENTRYPOINT` resets an inherited `CMD`. Shell form runs the command under `/bin/sh -c` | `docker image inspect` (`Config.Entrypoint`, `Config.Cmd`) | Exec form; set `CMD` explicitly; know what the base sets | `zero-layer-config` |
| The build arg is empty inside `RUN` | An `ARG` declared before the first `FROM` is visible only to `FROM` lines unless the stage declares it again | Build output; `RUN echo` | Declare the `ARG` again inside the stage | `arg-scope` |
| A builder toolchain ships to production | A single-stage build, or a final stage built `FROM` the builder | Image size; `docker history` shows compilers | Multi-stage build with a minimal final base; `COPY --from` only the artifact | `the-handoff` |
| The container's disk grew after touching one large file | On the first write, the whole file is copied up into the writable layer | `docker ps -s`; `docker diff` | Volumes for mutable data | `writable-layer` |
| A config directory baked into the image is empty or different at runtime | A bind mount over that path hides the image's content; the runtime also replaces `/etc/hosts` and `/etc/resolv.conf` | `findmnt` in the container; `Mounts` in `docker inspect` | Mount single files, or bake defaults to a path nothing mounts over | `mounted-over` (extension) |

### 2.2 Existing tools and the gap

- **`docker history` and `docker image inspect`** list history entries and the config, without a picture or links to the Dockerfile.
- **[dive](https://github.com/wagoodman/dive)** (v0.13.1, March 2025) is the best existing layer explorer. It has a per-layer file tree and an estimate of wasted space. It doesn't model what Dockerfiles mean: config-only instructions, `ARG` scope, the cache, or the stages that didn't ship.
- **[dockerfilegraph](https://github.com/patrickhoefler/dockerfilegraph)** draws a stage graph but not layers.
- **Docker Desktop's Builds view and `docker buildx history`** show the timing and cache of a real build, for operating a build rather than for explaining one.
- **crane, regctl, skopeo, and oras** inspect manifests, configs, and blobs.

The gap: no tool shows all of the following together for one Dockerfile: line-to-effect mapping, stacking and hiding, the stage graph, and the runtime view. None offers a fix to compare against.

## 3. Reframing "The Layers Merge into One Final Layer"

The phrase is the natural starting point, and the experience should start there. It needs refining, because the most common mistakes come from what it leaves out.

**What is true:**

- In the shipped stage, each instruction that changes files produces a layer, and layers stack in order.
- A running container sees one filesystem, in which later layers win.
- Tools can produce a single-layer image when asked to.

**What needs refining:**

1. **The image is not one layer.** A built image is an ordered list of layer blobs plus a config blob, referenced by a manifest. A multi-platform image adds an index that points to several manifests. Registries store and transfer the blobs separately. That is why pulls are incremental and why images built on the same base share its layers.
2. **Nothing in the image is ever merged.** Layers are changesets (tar archives) that are applied in order. A deletion is recorded as a whiteout file (`.wh.<name>`, or `.wh..wh..opq` for an opaque directory) that hides a path in lower layers. The hidden bytes stay in the lower layer and keep shipping.
3. **The "merge" happens at unpack and run time, usually as a view.** A snapshotter unpacks each layer on top of its parent chain, keyed by ChainID. With the usual overlayfs snapshotter, a container start stacks the read-only layer directories under one writable directory. A lookup goes from the top layer down, the first match wins, and a whiteout ends the search. A union filesystem is not required, though. The [iximiuz tutorial on container filesystems](https://labs.iximiuz.com/tutorials/container-filesystem-from-scratch) says union filesystems are "not mandatory for containers," "despite popular belief." A native snapshotter, or `crane export` piped into `tar`, produces the merge as a flat directory of copies instead. Overlayfs saves disk space and start time; it is not what makes something a container.
4. **Many instructions make no layer.** `ENV`, `USER`, `EXPOSE`, `LABEL`, `CMD`, `ENTRYPOINT`, `HEALTHCHECK`, `SHELL`, `STOPSIGNAL`, `VOLUME`, and `ONBUILD` edit the JSON config and add a history entry marked `empty_layer`.
5. **Some inputs exist only during the build, and some of them still leak.** `ARG` values and `RUN --mount` content (secrets, caches, SSH agents, bind mounts) are not part of the image's filesystem. The Dockerfile reference warns, however, that build-arg values are visible in `docker history` and in max-mode provenance attestations.
6. **In a multi-stage build, only the target stage's layers ship.** Other stages contribute only through `COPY --from`, which writes a new layer of copies in the target stage. The builder's layers stay in the build cache.
7. **Config is inherited as well.** `FROM` brings the base image's config: its user, working directory, environment, entrypoint, command, and volumes. Setting `ENTRYPOINT` resets an inherited `CMD`.
8. **A running container adds one more layer.** The writable layer gets a copy of a whole file the first time that file is written, and a whiteout when a file is deleted. It is discarded with the container unless it is committed. This is file-level copy-on-write, a cousin of the page-level copy-on-write that Forkscape shows.
9. **The container's filesystem is more than the image.** The runtime creates a new mount namespace and makes the image's root filesystem a mount. It then mounts these on top:
   - a fresh `/proc`;
   - `/dev` as a tmpfs with device nodes, plus `devpts`, `mqueue`, and `shm`;
   - a read-only `/sys` and cgroup filesystem;
   - its own `/etc/hosts`, `/etc/hostname`, and `/etc/resolv.conf`, bind-mounted over the image's files;
   - bind mounts and volumes;
   - `/dev/null` or a read-only tmpfs over sensitive paths.

   It then switches root with `pivot_root`. Each mount hides whatever the image has at that path. This is a third way a path can be hidden, after a later layer and a whiteout.
10. **Flattening is real, but it is a deliberate act.** `docker export` piped into `docker import`, `crane flatten`, or a final `FROM scratch` stage with `COPY --from=build / /` each produce a single-layer image. Each one gives up layer sharing with the base image and its history.

The refined statement the experience builds toward:

> A Dockerfile builds a graph of stages. In the stage that ships, each instruction that changes files adds a changeset on top of the base image's changesets, and every other instruction edits a JSON config. The image is that ordered list plus the config. Nothing is merged until a container is created. The runtime then usually stacks the layers, read-only, under one writable layer, and mounts its own filesystems and files over parts of that stack. For each path, the process sees whatever is on top.

### 3.1 Consequence for the visualization

Showing the layers fusing into one final layer would reinforce misconceptions 1 to 3. The working framing is instead **stacked changesets and the view through them**. The merge is something the viewer looks *through*, not something that happens *to* the layers. The chosen metaphor is a light table drawn as a stack of glass discs (Section 10.1). Phase 0 tests its layout rules before the choreography is designed. Section 10.2 lists the visual commitments it must keep.

## 4. Audience, Prerequisites, and Learning Outcomes

### 4.1 Audience

- Application engineers who write and maintain Dockerfiles.
- Platform and developer-experience engineers who own base images and review other teams' Dockerfiles.
- Security reviewers chasing secrets, provenance, and image bloat.
- Advocates and educators who explain builds to others.

### 4.2 Prerequisites

- Has written and built at least one Dockerfile.
- Knows basic Linux file ownership and permissions.
- No knowledge of the OCI specifications, BuildKit, or overlayfs is required.

### 4.3 Learning outcomes

After the MVP examples, a learner should be able to:

1. Classify every Dockerfile instruction as one that changes files, edits the config, exists only in the build, or shapes the stage graph.
2. Read an image's manifest, config, and history, and map their entries back to Dockerfile lines.
3. Explain how the runtime presents layers as one filesystem (stacking order, copy-up, whiteouts), and why deleting in a later layer does not shrink an image.
4. Predict which steps rebuild after a change, and reorder a Dockerfile to fix a cache cascade.
5. Explain what a multi-stage build ships, what it leaves behind, and what `COPY --from` carries.
6. Find build-only values and secrets that leaked into layers, history, or config, and replace them with secret mounts.
7. Explain base-image config inheritance (`CMD`, `ENTRYPOINT`, `USER`, `ENV`, `WORKDIR`), and why `USER` doesn't change file ownership.
8. Explain content addressing: digests, diff IDs, and ChainIDs, why images share layers, and why rebuilding identical content can still produce a new digest.

## 5. Conceptual Framework

### 5.1 The anchor idea

An image is a list of changesets plus a config. A container is a view through that list.

```text
 Dockerfile ──parse──► stage graph ──BuildKit (LLB graph + cache)──► target stage
                                                                         │
   image ┌───────────────────────────────────────────────────────────────┤
         │ index (one entry per platform)                                │
         │   └► manifest ─► config  Env, User, Cmd, … , history, rootfs.diff_ids
         │              └► layers  L0 … Ln   tar changesets, whiteouts    │
         └───────────────────────────────────────────────────────────────┘
                                                                         │ pull, unpack
                                                                         ▼ (snapshots keyed by ChainID)
   container (its own mount namespace):
               runtime mounts: /proc  /dev  /sys  /etc/{hosts,hostname,resolv.conf}  volumes  masked paths
                    │ each one covers a path in
                    ▼
               writable layer ──┐
               Ln … L0 (read-only) ──► overlay mount (or a flat copy) ──pivot_root──► /
                                                                    ──► the filesystem the process sees
```

### 5.2 Four questions for every instruction

The inspector organizes each instruction's detail around four questions. Example captions refer to them by number.

1. **Does it change files?** If so, it produces a layer containing exactly those changes, including whiteouts.
2. **Does it change the config?** If so, it sets a field and adds a history entry, with no layer.
3. **What does it depend on?** Its cache key: the parent state, the instruction text, the `ARG` and `ENV` values it sees, source-file checksums, and mounts.
4. **Does it reach the shipped image, and does anything leak?** Is the stage reachable from the target, does it reach the target through `COPY --from` or `FROM <stage>`, and do any of its values surface in history or provenance?

### 5.3 Three sizes

Forkscape's central accounting idea was that RSS and PSS count the same pages differently. Layerviz's equivalent is that one image has three honest sizes:

| Size | What it counts | Where an engineer meets it |
| --- | --- | --- |
| **Download** | Compressed layer blobs not already present locally | Pull time, registry storage, manifest `size` fields |
| **Stored** | The uncompressed sum of all layers | Disk usage, most "image size" columns |
| **Visible** | Bytes in the merged image filesystem, not counting runtime mounts | `du -x /` inside a container; roughly the size of `crane export` output |

**Stored minus visible is hidden bytes**: content that is shadowed by a later version or hidden by a whiteout. It ships but no process can ever read it. The ledger names the size and the command that reports each one, because different tools and image stores report different numbers.

### 5.4 Threshold concepts

- **A layer is a diff, not a snapshot.**
- **Deleting adds something:** a whiteout.
- **Three things can hide a path:** a later layer's version, a whiteout, and a mount that the runtime places over it.
- **The image is not the whole container filesystem.** The runtime builds a tree of mounts on top of it.
- **Config is not filesystem.** Half of a Dockerfile's instructions never touch a file.
- **The build is not the image.** Build arguments, mounts, the cache, and unshipped stages live outside it, except where they leak.
- **Order is the cache.** A step's cache key includes everything above it.
- **Content addressing.** The same bytes give the same digest, while a different parent gives a different ChainID.

### 5.5 Misconception inventory

| Misconception | Correction | Confronted in |
| --- | --- | --- |
| "The layers merge into one final layer" | The image stays a list. The merge is a runtime view. | every example's run step; `writable-layer` |
| "Deleting a file in a later step makes the image smaller" | A whiteout hides it, and the bytes still ship | `deleted-still-shipped` |
| "Every instruction creates a layer" | Config instructions add history entries with no layer | `zero-layer-config` |
| "`ARG` values are private because they aren't in the image" | They can appear in history and max-mode provenance | `layer-three-secret`, `arg-scope` |
| "`USER` makes later `COPY` files belong to that user" | `COPY` and `ADD` create files owned by `0:0` unless given `--chown` | `user-isnt-owner` |
| "`chmod`/`chown` in a later step is cheap" | Each changed file is copied whole into the new layer | `chown-doubles` |
| "Earlier stages are in the final image" | Only the target's chain ships, plus whatever `COPY --from` copies | `the-handoff` |
| "An `ARG` declared at the top applies everywhere" | One declared before `FROM` is visible to `FROM` lines only, unless the stage declares it again | `arg-scope` |
| "The cache is per line" | A miss invalidates every later step in the stage | `cache-cascade` |
| "My image's `CMD` is whatever I wrote" | The base's config is inherited, and `ENTRYPOINT` resets the inherited `CMD` | `zero-layer-config` |
| "`EXPOSE` publishes a port" | It is documentation stored in the config | `zero-layer-config` |
| "Writes in a container change the image" | They go to a writable layer that is discarded unless committed | `writable-layer` |
| "Containers need a union filesystem" | Overlayfs saves disk space and start time; a flat extracted rootfs works too | `writable-layer`, `flatten-or-not` (extension) |
| "The container sees my image's `/etc/hosts` and `/etc/resolv.conf`" | The runtime bind-mounts its own generated files over them | every run step; `mounted-over` (extension) |
| "A bind mount or volume merges with what the image has at that path" | A bind mount hides the image's content. Docker copies image content only into a new, empty named volume | `mounted-over` (extension) |
| "Same Dockerfile, same digest" | Timestamps and nondeterministic steps change layer digests | `reproducible-digests` (extension) |

### 5.6 Language guide for content authors

- Say **layer** only for a filesystem changeset. Say **config** for image metadata and **history entry** for the per-instruction record.
- Say **image** for the manifest, config, and layers together, and **index** for the multi-platform list.
- Say **merged view** or **container filesystem** for what the process sees. Avoid "final layer" unless the example flattens the image.
- Say **whiteout**, not "deletion marker", and **copy-up** for the runtime's first-write copy.
- Say **mount** for anything the runtime places over a path, and never call a mount a layer.
- Say **stage**, **target**, and **reachable**. Say **build cache**, never "layers", for BuildKit's cache records.
- Say **digest** for the compressed blob hash and **diff ID** for the uncompressed hash.
- Keep captions to what happened and why, in the present tense, and leave the detail to the inspector.

## 6. Dockerfile Instruction Coverage

Every instruction and the commonly used flags are covered. Each one has a coverage card in the inspector, and the important ones are confronted in an example. Effect classes: **Layer** (changes files), **Config** (edits the image config), **Build-only** (affects the build, not the image), and **Graph** (shapes stages).

| Instruction or feature | Effect | What the visualization must show | Trap worth showing | Example |
| --- | --- | --- | --- | --- |
| Parser directives: `# syntax`, `# escape`, `# check` | Build-only | Which frontend version parsed the file; build-check findings as annotations | Without `# syntax`, newer features depend on the builder's bundled frontend | all (`syntax`); `zero-layer-config` (`check`) |
| `FROM [--platform] <image> [AS name]` | Graph, plus inherited layers and config | Base layers arriving as one block that can be opened; inherited config fields tagged "from base" | The base's `USER`, `WORKDIR`, `CMD`, `ENTRYPOINT`, and `VOLUME` carry through | all |
| `ARG` (global before the first `FROM`, or per stage); predefined `ARG`s (`TARGETPLATFORM`, `BUILDPLATFORM`, proxy variables) | Build-only, with possible leaks | Values visible to later steps but never inside the image; a leak shown as a leak | Scope rules; a changed value misses the cache at its first use; values recorded in history and max-mode provenance; proxy `ARG`s excluded from history unless declared | `arg-scope`, `layer-three-secret`, `cache-cascade` |
| `RUN` (shell form, exec form, heredocs) | Layer | The changeset's files, whiteouts, and size | Cleanup in a separate `RUN`; package-manager caches left behind | `deleted-still-shipped`, `chown-doubles` |
| `RUN --mount=type=cache\|secret\|ssh\|bind\|tmpfs` | Build-only content | Mounted content present during the step and absent from the layer | Copying a secret out of the mount into the filesystem; a cache mount is not a layer | `layer-three-secret`, `cache-cascade`, `the-handoff` |
| `RUN --network`, `--security`, `--device` | Build-only | Shown as properties of the step | — | coverage card |
| `COPY [--from] [--chown] [--chmod] [--link] [--parents] [--exclude]` | Layer | Copied files with ownership, mode, and source (build context, stage, image, or named context) | Ownership ignores `USER`; `--link` makes a layer independent of what is below it; the cache key is the context checksum; `.dockerignore` | `cache-cascade`, `user-isnt-owner`, `chown-doubles`, `the-handoff` |
| `ADD` (local file, URL, Git repository, tar unpacking, `--checksum`, `--keep-git-dir`, `--unpack`, `--link`, `--exclude`) | Layer | Like `COPY`, plus where the content came from | Local tar files unpack by default and remote ones don't | `deleted-still-shipped` |
| `ENV` | Config, and part of later steps' environment and cache keys | The `Env` list growing; substitution in later lines | A secret in `ENV` persists in the config, and later overriding it doesn't remove the earlier value from history | `zero-layer-config`, `layer-three-secret` |
| `WORKDIR` | Config, and creates the directory if it doesn't exist | The `WorkingDir` field; a new directory when one is created | A relative path resolves against the previous `WORKDIR`; the base's value is inherited | `zero-layer-config`, `user-isnt-owner` |
| `USER` | Config; affects later `RUN` steps and the runtime | The `User` field; which steps run as whom | Does not change `COPY` ownership | `user-isnt-owner`, `the-handoff` |
| `EXPOSE` | Config | `ExposedPorts` | Documentation only; it doesn't publish | `zero-layer-config` |
| `LABEL` (and Buildx `--annotation`) | Config labels; annotations live on the manifest or index | Labels in the config, annotations on the manifest | Labels are inherited from the base | `zero-layer-config` |
| `CMD`, `ENTRYPOINT` (shell and exec form) | Config | The final argv the container will run, assembled from both | `ENTRYPOINT` resets an inherited `CMD`; shell form wraps the command in `/bin/sh -c`, which affects signals and PID 1 | `zero-layer-config`, `the-handoff` |
| `VOLUME` | Config; the runtime creates a volume | `Volumes`, plus the runtime step that seeds a volume from image content | The legacy builder discards later changes to a volume path and BuildKit keeps them | `zero-layer-config`, `writable-layer` |
| `HEALTHCHECK` | Config (a Docker extension, not part of the OCI config spec) | The field and who honors it | Kubernetes ignores it | `zero-layer-config` |
| `SHELL` | Config (Docker extension) | The shell used by later shell-form instructions | — | `zero-layer-config` |
| `STOPSIGNAL` | Config | `StopSignal` | — | `zero-layer-config` |
| `ONBUILD` | Config triggers (Docker extension) | A trigger card carried in the config that fires in a child build | Surprising effects in children | `onbuild-surprise` (extension) |
| `MAINTAINER` | Config author (deprecated) | The `author` field | Use `LABEL` | coverage card |
| Labs syntax: `FROM --reset`, `ENV --unset`, `LABEL --unset`, `VOLUME --unset` | Config | Inherited fields being cleared | Requires `docker/dockerfile:1-labs` | `labs-reset` (extension) |
| Build context, `.dockerignore`, named contexts (`--build-context`) | Build-only inputs to `COPY` and `ADD` | What was sent and what was excluded | A missing `.dockerignore` makes every file a cache input | `cache-cascade` |

### 6.1 Multi-stage builds

- **Stages** are named or numbered. `FROM <earlier stage>` inherits that stage's layers *and* config. `COPY --from=<stage|image|context>` copies files and nothing else.
- **The target** is chosen with `--target` and defaults to the last stage. BuildKit works out which stages are reachable from the target, skips the rest, and builds independent stages in parallel.
- **What ships** is the target's chain only. The builder's layers stay in the build cache, and `COPY --from` writes a new layer of copies in the target. Config from a builder stage doesn't carry over unless the target is built `FROM` it.
- **What the visualization must show:** the stage graph; which stages are reachable and which are skipped; the hand-off of files from one stage to another; `ARG` declarations repeated per stage; cache shared between stages; and the size each stage would have had if it had shipped.
- **High-level visual intent** (choreography deferred): each stage is its own cylinder of discs, unreachable stages are dimmed, a `COPY --from` carries a bundle of files from one cylinder to another, and only one cylinder becomes the image (Section 10.1).

### 6.2 After the build: pull, unpack, run

Each MVP example ends with a **run step**: the container is created, and the process's view through the stack is shown. The step has three parts, in this order.

1. **The root filesystem.** The image's layers sit under a writable layer in an overlay mount. The inspector shows the real `lowerdir=` list and maps each entry to its layer. Overlayfs whiteouts are `0/0` character devices, and an opaque directory carries an xattr. A coverage card describes the flat alternative (a native snapshotter, or `crane export`), which gives the same view by copying.
2. **Runtime mounts.** In a new mount namespace, the runtime mounts the following over the root filesystem. The list follows runc's sequence as the iximiuz tutorial reconstructs it from runc's source and an `strace` of containerd:
   - `/proc`, mounted fresh for the container's PID namespace;
   - `/dev` as a tmpfs with a few device nodes, plus `devpts`, `mqueue`, and `shm`;
   - `/sys` and `/sys/fs/cgroup`, read-only;
   - bind mounts and volumes from `-v` and `--mount`;
   - generated `/etc/hosts`, `/etc/hostname`, and `/etc/resolv.conf`, bind-mounted over the image's files;
   - Docker's masked paths (`/dev/null` over files, a read-only tmpfs over directories) and read-only paths.

   Each mount hides what the image has at that path. `pivot_root` then makes the prepared tree `/`.
3. **The process's view and its writes.** Free exploration adds these actions:
   - write a file (copy-up);
   - delete one (a whiteout in the writable layer);
   - write under a mounted path, where the write goes to the mount and not to the writable layer;
   - run `docker diff`;
   - commit, which turns the writable layer into a new image layer and brings nothing from the mounts.

The measurement rung names:

- `findmnt` inside the container and on the host;
- `/proc/<pid>/mountinfo`;
- `readlink /proc/<pid>/ns/mnt`;
- the `mounts`, `maskedPaths`, and `readonlyPaths` in the OCI runtime spec that the runtime received.

It notes that these details differ between runtimes, storage drivers, and Docker versions. Docker Engine 29's default image store and its default mount lists are on the re-verify list (Section 19). Mount propagation, user namespaces, and the symlink defenses runtimes use on an untrusted rootfs get coverage cards, not run-step content.

## 7. What Carries Over from Forkscape, and What Changes

### 7.1 Patterns reused as they are

- **Example library first,** opening on arrival. Free exploration is secondary, and nothing is graded or saved.
- **Example shape:** `symptom`, steps with captions, `watch`, `mechanism`, `production`, `evidence`, `reference`, `related`, and a `fix` with its `change` and `cost`. Every fix states its cost.
- **Fix replay at the same step,** framed in the run's colour, with the chart and ledger on one scale across both runs.
- **A pure, deterministic model with invariants.** Both renderers consume one `WorldState`, and tests replay every example and assert its claims.
- **Motion:** one 600 ms curve, `cubic-bezier(.4, 0, .2, 1)`. Objects keep their identity across steps, stepping back returns them to where they came from, and reduced motion changes the scene at once.
- **Labels only what changed** while stepping. Labels never overlap.
- **Inspector rungs:** metaphor, mechanism, and measurement. The measurement rung names a real command (`docker history`, `docker image inspect`, `docker buildx imagetools inspect`, `crane manifest`, `regctl`, `dive`, `docker buildx history`, `mount`) and says whether the value on screen comes from the model, a static analysis, or a recording.
- **Chrome:** the scene fills the window with a 3D/flat toggle; one panel is open at a time; under 900 px the panel becomes a sheet with Back and Next docked; each example has its own URL hash.
- **Type, accessibility, hosting:** the rem type scale, flat-view parity, a static Vite build on Netlify with a strict CSP.
- **Reference runs as separate evidence.** Builds run in disposable `labctl` VMs and write versioned results to `docs/`. They appear in the app as labeled observations from one environment.

### 7.2 Code to lift

Copy the following, then adapt it. Forkscape is a reference, not a dependency: the two apps never share a package, and they diverge freely (decision 11).

- `src/motion.ts`, with identity keys changed to `stage/<name>`, `layer/<chainId>`, `entry/<layer>/<path>`, and `cfg/<field>`.
- The stage-framing approach from `src/scene.ts`, so the camera never needs to move when objects appear.
- Label placement and collision handling, the panel/menu/sheet chrome, and the chart's fixed-scale rendering.
- `scripts/ui_parity.mjs` with `playwright-core` and system Chrome; the Node test-runner setup; the `netlify.toml` headers.
- The recording picker and drop zone from the replay work, including the session-storage copy for reloads.

Forkscape is pinned to three.js 0.180 and Vite 7. The new project starts on current versions (Section 9.1), so lifted code needs an API check.

### 7.3 What changes

- **Much of the model is exact.** For structure (stages, effect classes, config, scopes, and cache dependencies), the semantics are defined by the Dockerfile reference and the OCI specs, not approximated. The one large unknown is what a `RUN` step does to the filesystem. Only a recording can show that exactly (Section 8).
- **Input is user-supplied.** That brings parsing, size limits, untrusted text, and privacy, which Forkscape's authored scenarios didn't have.
- **Promotion stays outside the app.** A launch trailer is made from the preloaded examples (Section 12). The app's only accommodation for it is a capture mode.
- **Scale varies wildly.** A real image can have more than 100,000 files, so level of detail is needed from the start (Section 9.4).

## 8. Inputs and the Fidelity Contract

### 8.1 Three ways in

1. **Paste a Dockerfile** (static analysis). The page works out the stage graph, reachability, each instruction's effect class, the config as it builds up, `ARG`/`ENV` scope, cache dependencies, and build-check findings, all instantly and exactly. Two things are unknown: what a `RUN` step changes, and the contents of the base image. The page shows `RUN` layers as sealed discs of unknown content, with a prompt to record the build. For common bases (Alpine, Debian slim, Ubuntu, distroless, and the official Node, Python, and Go images), it ships small metadata bundles, pinned by digest and dated, so that `FROM node:<version>-slim` shows realistic base layers. A scheduled GitHub Actions job refreshes these bundles monthly with the recorder's registry mode (Section 8.5) and commits them with their digests and dates. Other bases appear as an unopened block.
2. **Drop a recorded bundle.** A bundle comes from the recorder CLI (Section 8.3) and gives exact layers, entries, sizes, whiteouts, config, history, the image each stage would produce, and cache behavior for a declared change.
3. **Drop an image archive.** This is a `docker save` tar (OCI layout since Docker Engine 25) or an OCI layout tar. It gives exact layers, config, and history for the shipped image only. With no Dockerfile, the instructions are reconstructed from history `created_by` text, and other stages are invisible.

The preloaded examples are recorded bundles from reference builds, so they never depend on static inference.

### 8.2 Fidelity contract

| Source | Badge | Can show | Must not claim |
| --- | --- | --- | --- |
| Static analysis | **STATIC** | Stage graph, reachability, effect classes, config, `ARG` scope, cache dependency structure, build checks | The contents or sizes of `RUN` layers; the contents of a base image without a metadata bundle |
| Inferred (not in the MVP; decision 4) | **INFERRED** | Typical effects of well-known commands (package installs, `rm`), from a small rule set | Exact paths or sizes |
| Recorded bundle | **RECORDED** | Exact layers, entries, sizes, whiteouts, config, history, per-stage images, and cache hits for the recorded change | Other platforms, other builder versions, or another run of a network-dependent step |
| Image archive | **RECORDED · IMAGE** | Exact layers, config, and history of the shipped image | Stages, `ARG` values not in history, or line mapping beyond the history text |
| Reference run | **REFERENCE** | BuildKit or runtime behavior observed in a dated, versioned reference build, including a run step's real mount tree | General rules |

The run step's runtime mounts come from a reference run where one exists. Otherwise they come from Docker's documented defaults for a non-privileged container, labeled **STATIC** with the Docker version they describe.

### 8.3 Recorder CLI

Write the recorder in Go as one binary. The language is lean, matches the tooling, and can reuse BuildKit's own packages.

- It builds with `docker buildx build`, exports **each stage** as an OCI layout (`--target <stage> --output type=oci`), and requests `--provenance=mode=max` (the Dockerfile and the LLB graph with source locations) and `--metadata-file`.
- Optionally, it rebuilds after a declared change, such as touching a file or changing a build argument, and records which steps hit the cache from `--progress=rawjson`.
- It reads the OCI layouts and records **entry metadata only** (path, type, size, mode, uid/gid, link target, whiteout kind). By default, file contents are never recorded.
- Optionally, it starts one container from the shipped image and records the run step:
  - the OCI runtime spec the runtime received (`mounts`, `maskedPaths`, `readonlyPaths`);
  - `findmnt -J` from inside the container;
  - the host's overlay mount options, with each `lowerdir` entry mapped to its layer.

  In reference runs, it can also record the runtime's mount sequence with `strace` on containerd during `docker run`, as the iximiuz tutorial does.
- It treats every layer as untrusted. It reads tar headers without extracting to disk, rejects `..` and absolute entry paths, and never follows a symlink out of the tree. The browser's archive reader applies the same rules to its in-memory tree.
- It redacts build-arg values, label values, registry hosts, and paths that match configurable patterns, and prints what it redacted.
- It writes a gzipped `layerviz.bundle.v1` JSON file.
- The same Go module compiles its Dockerfile analysis to WebAssembly for the browser (Section 9.3), so the page and the CLI always agree.

Reference bundles for the preloaded examples are built in a disposable `labctl` VM with pinned Docker Engine, Buildx, and frontend versions. Each bundle records those versions.

### 8.4 Bundle sketch

```ts
interface LayervizBundleV1 {
  format: 'layerviz.bundle.v1';
  tool: { recorder: string; engine?: string; buildkit?: string; buildx?: string; frontend?: string; recordedAt: string };
  source?: { dockerfile: string; target: string; platform: string; buildArgs: Record<string, 'redacted' | string> };
  analysis: { stages: Stage[]; checks: BuildCheck[] };          // same shape the WASM analyzer produces
  images: Record<string /* stage */, ImageRecord>;              // manifest digest, config, layers[] with entries
  bases?: Record<string /* ref */, { digest: string; layers: LayerSummary[]; config: object }>;
  cacheRuns?: { change: string; steps: { id: string; cached: boolean }[] }[];
  redactions: { kind: string; count: number }[];
}
```

Starting defaults (decision 12), which S3 tunes against the real examples:

- **Entry metadata only.** File contents are never recorded.
- **Rollups.** A layer with more than 5,000 entries stores directory rollups plus its 200 largest entries.
- **Size cap.** A bundle is at most 20 MB gzipped.
- **Redaction.** Build-arg values, label values, and registry hosts are redacted by default, and further path patterns can be configured.

S3 also settles the full schema.

### 8.5 Not in the browser: registry pulls

The page doesn't fetch images from registries. Most registries don't allow cross-origin browser requests, pulling private images needs credentials, and a browser fetch would expose what the user is inspecting. Instead, the recorder can make a bundle from a public registry reference using go-containerregistry or regclient. The monthly base-image refresh uses this mode in the MVP. Private registries, which need credentials, come after the MVP.

### 8.6 Privacy

Everything is parsed in the browser and nothing is uploaded. There are no content analytics. As in Forkscape, a copy of a small input stays in session storage so a reload works.
## 9. Technical Architecture

### 9.1 Stack and versions

Versions were checked on 14 November 2026 and are pinned exactly in `package.json`. LTS lines are used where a project has them.

| Component | Version | Note |
| --- | --- | --- |
| Node.js | 26 LTS ("Lithium", 26.13.x) | Development and test runtime |
| TypeScript | 7.0.x | The native compiler. Check that the tsconfig carried over from Forkscape is compatible |
| Vite | 8.3.x | Requires Node 20.19+ or 22.12+ |
| three.js / `@types/three` | 0.187.0 / 0.187.0 | Pinned exactly, as Forkscape does |
| playwright-core | 1.65.x | UI parity tests with system Chrome |
| Go | 1.27.x | Recorder and WASM analyzer. Go has no LTS; the two most recent majors are supported |
| Docker Engine / Buildx / BuildKit / Dockerfile frontend | 29.9.x / 0.39.0 / 0.35.0 / 1.29.0 | Reference builds |
| containerd | 2.4.x | Runtime reference |
| OCI image spec | v1.1.1 | Manifest, index, config, and layer formats |
| Blender | 5.2 LTS (5.2.4) | Offline assets and renders |

Gzip layers decompress with the browser's `DecompressionStream`. Zstandard layers need a small decoder unless browsers support it natively (Section 19).

### 9.2 Module layout

```text
src/
  dockerfile/   WASM analyzer bridge → stages, instructions, effect classes, ARG/ENV scopes, cache deps, checks
  image/        bundle and archive readers → ImageModel: layers, entries, whiteouts, config, history, sizes
  model/        WorldState per step (build steps, then run steps), invariants, problem/fix pairs
  examples/     preloaded examples: Dockerfile, fix, captions, bundle references
  scene/        three.js renderer, lazy-loaded; consumes WorldState
  flat/         DOM/SVG renderer; consumes the same WorldState
  panels/       source pane, merged tree, config card, size ledger, inspector, timeline
  capture/      capture mode: hidden chrome, fixed viewport, deterministic clock (used only by trailer/)
recorder/       Go module: CLI, plus a WASM build of the shared Dockerfile analysis package
trailer/        launch trailer: edit.json, Playwright capture and ffmpeg scripts, Blender scripts; not in the app bundle
```

### 9.3 Parsing a Dockerfile in the browser

| Option | Strengths | Weaknesses |
| --- | --- | --- |
| **A. Own TypeScript parser** | Small; full control | Reimplements heredocs, the escape directive, flags, and variable expansion; drifts from BuildKit |
| **B. [`dockerfile-ast`](https://www.npmjs.com/package/dockerfile-ast)** (0.7.1, last published July 2025) | TypeScript; precise source positions; used by the Docker language server | May lag new flags and syntax; no build checks |
| **C. BuildKit's own parser and instruction packages, compiled from Go to WASM** | The exact grammar of the pinned frontend version; can reuse BuildKit's build checks; shares code with the recorder | Download size and startup time; the CSP needs `'wasm-unsafe-eval'` |

**Recommendation:** use C if spike S2 shows an acceptable size and startup time (provisional targets: under about 3 MB compressed and under 300 ms to initialize). Lazy-load it only when the user pastes a Dockerfile. Preloaded examples carry precomputed analysis and never need it. Otherwise use B, with conformance tests against C in CI. In either case, the page shows which frontend version did the analysis, because a user's `# syntax=` line may ask for a different one.

### 9.4 Rendering

- **three.js `WebGLRenderer`** is the default, matching Forkscape so its code lifts directly. The S1 prototype also tries `WebGPURenderer` (with its WebGL 2 fallback), because the disc stack depends on effects it may handle better: transmission through stacked translucent discs, and very many instanced tiles. The app switches only if that is clearly better.
- The **flat view** is DOM/SVG built from the same state. It is keyboard-accessible and doubles as the fallback when WebGL fails.
- **Level of detail:** directory rollups, the largest entries per layer, lazy expansion, and instanced meshes for file tiles. Use Forkscape's frame-time budgets.

### 9.5 Hosting

A static Vite build on Netlify, with Forkscape's CSP and immutable-asset caching plus `'wasm-unsafe-eval'`. There is no backend.

## 10. Visual Direction (High Level; Choreography Deferred)

### 10.1 Framing: a light table drawn as a stack of glass discs

**Decided (14 November 2026):** the light-table metaphor, drawn as a vertical cylinder of stacked, translucent discs.

- **Overview.** Each layer is a disc whose width shows its size. The base image is at the bottom and the writable layer is at the top. The learner moves the cylinder up and down (scroll, drag, or arrow keys) to travel through the stack. Looking straight down the axis shows the container's view, so the merge is a projection through the stack, not an event.
- **Isolating a layer.** Clicking or tapping a disc lifts it out into its own space, where it can be rotated and zoomed into. The rest of the stack stays in place, dimmed, so the learner doesn't lose track of where the layer came from. Back or Escape returns it.
- **Contents in the layer's own space.** This is where a layer's changes are shown. The proposals below make it accurate enough to apply to production images. S1 tests them.

Proposals for S1 to test:

1. **One map for every disc.** Each path gets a fixed angle and ring, computed once from the union of every path in the image. It is laid out as a sunburst: the root at the center, directories as rings, and arc length proportional to bytes. Each disc draws only the paths it changes, at those positions, so `/usr/lib` sits at the same place on every disc. A path then has a vertical column through the cylinder. "Looking down" means exactly what overlayfs does: in each column, the topmost entry wins.
2. **Path core.** Selecting a path pierces the cylinder along its column. The needle shows every version of the path in every layer, the version the container sees, and the whiteout or mount that ends the lookup. This borrows the core-sample idea from the strata candidate.
3. **Change kinds that read at a glance,** both on the disc and in its isolated view:
   - **added:** solid;
   - **replaces a lower version:** solid, with an outline linking to the version it shadows;
   - **metadata-only change:** hatched (the `chown-doubles` case);
   - **whiteout:** an opaque dark patch;
   - **opaque directory:** a dark sector.

   In the isolated view, entries are extruded by size, so a 400 MB tarball towers over a config file.
4. **Size scaling.** Layer sizes span about six orders of magnitude, from a few bytes for a `WORKDIR` to hundreds of megabytes for a base. A disc's area, not its radius, is proportional to bytes, with a minimum radius so that tiny layers stay clickable. The ledger always shows the exact numbers. S1 checks whether a log scale reads better.
5. **Config on the spine, not on discs.** Config-only instructions have no disc. They appear as tags on the cylinder's central axis, between the discs where history places them, and they feed the config card. This keeps the commitment that a layer and a config edit never share a visual form (Section 10.2).
6. **The base image** is one thick block that can be opened into its own discs.
7. **Run step.** The writable disc sits on top in a different material. Runtime mounts are caps above it that cover only their path's sector of the map. A bind mount over `/app/config` blocks that column the way a whiteout does, but from above.
8. **Multi-stage builds** (stage yard). Each stage is its own cylinder on a shared floor, and unreachable stages are dark. A `COPY --from` carries a bundle of entries from one cylinder to a new disc on the target. Only the target cylinder becomes the image.
9. **Flat view and phones.** The flat view is a vertical list of layers with bars whose length is size, plus a 2D sunburst of the selected layer and the provenance tree (each merged entry colored by its layer, with ghosts for hidden paths). On phones, a vertical swipe moves the stack, a tap isolates a layer, and two fingers rotate it.

**Open questions for S1:**
- whether the shared map stays readable when a layer touches thousands of paths (if not, fall back to directory rollups);
- area or log scaling for disc size;
- whether the isolated view is a sunburst, a circular treemap, or a radial "city" of extruded blocks.

### 10.2 Commitments the disc stack must keep

- A filesystem layer and a config edit never share a visual form.
- Build-only things appear beside the build, never inside the image. When a value leaks into history or provenance, the leak is drawn as a leak.
- Nothing is drawn fusing into a single layer unless the example actually flattens the image.
- A runtime mount covers a path, and it is never drawn as a layer. Committing a container never carries a mount into the image.
- Hidden bytes remain visible *as hidden*, so the gap between stored and visible size can always be seen.
- Base-image layers arrive as one block that can be opened.
- Every number comes from the model or a bundle, and captions are generated from those values.
- Forkscape's 3D/flat parity, identity, and labeling rules apply.

### 10.3 Honesty markers

Every view carries its source badge (**STATIC**, **INFERRED**, **RECORDED**, **RECORDED · IMAGE**, or **REFERENCE**). Base-image metadata shows its digest and the date it was taken. The legend states that the 3D layout is schematic.

### 10.4 Deferred to the next revision

- The exact animation of stacking and of looking through the stack.
- How a whiteout lands and how copy-up appears.
- How runtime mounts land over the stack, and whether the host's and the container's views of the same overlay mount appear side by side.
- How `COPY --from` travels between stages.
- The run step's choreography, the camera, and timings.
- The palette and theme: a sibling of Forkscape's, or a new identity.

## 11. Blender's Role

### 11.1 What Blender is for here

These are listed in order of value:

1. **Offline hero renders for the launch trailer.** Physically based lighting, depth of field, motion blur, and refraction through translucent sheets are what make a hype video look like more than a screen recording.
2. **Authored assets for the web scene,** such as the frame of a light table, a desk or room, and materials baked to textures. These are exported as glTF with named nodes, following the asset contract in [FRONTEND_PERF_VIZ.md §12.8](FRONTEND_PERF_VIZ.md).
3. **Previsualization during the S1 spike.** A few still frames of the disc stack and the isolated-layer view can be made in Blender before the three.js prototype, to settle materials and the layout of the shared map.

The rule from the frontend performance plan carries over: **data-carrying elements stay parametric in code.** Layers, file tiles, whiteouts, and sizes come from the model, and Blender supplies surroundings, materials, light, and camera.

### 11.2 One source of truth for both renderers

```text
preloaded example bundles + trailer/edit.json
        │
        ├─► three.js scene (app in capture mode) ── deterministic frame capture ──► animatic and most shots
        │         │
        │         └─ GLTFExporter: scene state at each beat ──┐
        │                                                       ▼
        └──────────────────────────────► Blender 5.2 LTS, headless (bpy script)
                                           imports beat glTFs + the Blender-authored set
                                           animates between beats from edit.json timings
                                           renders EEVEE (drafts) or Cycles (hero shots)
                                           ─► frames ─► ffmpeg ─► MP4
```

Geometry for data always comes from the TypeScript model, exported as glTF. The model is never reimplemented in Python.

### 11.3 Alternatives considered

| Option | Strengths | Weaknesses |
| --- | --- | --- |
| **Deterministic three.js capture** (Playwright stepping a virtual clock, frames to ffmpeg) | Same renderer as the app; cheapest; reproducible | Quality is capped at real-time rendering |
| **Remotion 4 with `@remotion/three`** | TypeScript compositions rendered frame by frame; good for captions, code highlights, and per-aspect-ratio layout | Adds a React layer to a non-React app; companies above a small size need a paid license (check current terms) |
| **Theatre.js** | Keyframe editor for three.js | Last npm release May 2024; don't depend on it |
| **Motion Canvas** | Strong for animating code, such as Dockerfile lines | Last npm release February 2025; maintenance risk |
| **Blender** | Highest quality ceiling | A second toolchain; needs the glTF hand-off above |

**Recommendation:** make the trailer by deterministic three.js capture, with the storyboard plan's SVG/HTML overlay kit for captions and code. Add Blender hero shots once the framing is fixed and spike S5 shows a visible gain. Reconsider Remotion if composing for several aspect ratios becomes the bottleneck. Generative 3D, such as Meshy, is optional set dressing only and is not in the MVP; see the frontend performance plan's notes on licensing and cleanup.

## 12. Launch Storyboard and Trailer

The storyboard and hype trailer **promote this project**. They are not a feature of the app, and learners can't export stories or videos from the page. The trailer is made for launch from the preloaded examples. It follows the production workflow in [STORYBOARD_PLAN.md](STORYBOARD_PLAN.md) and the discipline of [Doom Perf's trailer](../doom-perf/trailer/edit.json): a declarative edit, reproducible renders, and validation.

### 12.1 The story package for the trailer

| Storyboard plan unit | Here |
| --- | --- |
| One-page brief | Audience: engineers who write Dockerfiles. Promise: see what every line of a Dockerfile does to the image. Surprise: a deleted file still ships, and a builder never ships. Then the proof, and a call to action to open an example |
| Trustworthy story fixture | The preloaded examples' recorded bundles. Every number on screen comes from them |
| Panel script / edit file | `trailer/edit.json`: shots that reference an example and a beat, with camera, duration, caption, and sound cue |
| Reusable scene kit | The app's own renderer in capture mode, plus SVG/HTML overlays for captions and code |
| Release set | A 9:16 teaser, a 16:9 cut, stills for a carousel, a short loop, and a textless master |

### 12.2 Trailer concept

Each candidate angle is grounded in an MVP example:

- **The Handoff** (`the-handoff`): "<builder size> to build, <runtime size> to ship." This is the lead.
- **The Diet** (`deleted-still-shipped`): a deleted file still ships, then the fix.
- **The Leak** (`layer-three-secret`): a token visible in a lower layer, then gone.
- **The Cascade** (`cache-cascade`): one changed file rebuilds every step, then only one.

The teaser opens on The Handoff and closes on a quick montage of the other three. A first sketch of The Handoff runs about 25 seconds in 9:16. As in the storyboard plan, the timing will be revised after an animatic.

| Time | Panel and copy | Truth to check |
| --- | --- | --- |
| 0–3 s | Two cylinders side by side; one is tall. **"<builder size> TO BUILD."** | Builder size from the bundle |
| 3–8 s | The builder's discs stack up step by step: compiler, modules, sources. | Layer order and sizes recorded |
| 8–13 s | One small bundle of files leaves the builder for the second cylinder. **"ONLY THIS CROSSES."** | Exactly the `COPY --from` entries |
| 13–18 s | The builder goes dark, and only the runtime cylinder remains. **"<runtime size> TO SHIP."** | Download size of the shipped image |
| 18–22 s | The camera looks down through the runtime discs at the container's view. | The run step's merged view |
| 22–25 s | Title and call to action. | Destination exists at release |

### 12.3 Production

- **Capture mode.** The app gets a capture mode, behind a URL flag. It hides chrome, fixes the viewport and pixel ratio, and drives a deterministic clock so a script can step it frame by frame. This is the only trailer-related code in the app.
- **Capture scripts.** Scripts in `trailer/`, outside the app bundle, drive the app with Playwright, write frames, and encode them with ffmpeg. Blender hero shots follow Section 11.2 if spike S5 justifies them.
- **Review gates,** from the storyboard plan: first still panels and a contact sheet, then an animatic with placeholder sound reviewed at phone size and muted, and only then polish.

### 12.4 Truth rules

The storyboard plan's three layers carry over:

- **Observed:** the bundles' numbers, names, and order.
- **Metaphor:** the discs, cylinders, and hand-offs, labeled as illustration.
- **Interface:** faithful captures of the app.

A validator checks every number in every caption against the bundles, and checks that the durations add up to the runtime. The end card names the tool versions the bundles were recorded with.

### 12.5 Separate project: a Dockerfile-to-story CLI

A tool that turns *any* Dockerfile into a storyboard or hype video would be useful, but it is far afield of this effort. It should be **a completely separate project**, and nothing in this plan depends on it. Notes for whoever plans it:

- It could reuse this project's bundle format and recorder, and possibly the renderer in capture mode.
- A beat finder would scan a bundle for facts worth telling: the largest layer, hidden bytes and the line that hid them, values visible in history, stages that didn't ship, the `COPY --from` hand-off, how many steps one change rebuilds, and size before and after a fix. It would then fill templates such as Handoff, Diet, Leak, Cascade, and a line-by-line Anatomy.
- It would process other people's Dockerfiles, which carry internal registry hosts, mirror URLs, organization names, and build-argument values. So it needs a redaction pass, and a review of every string on screen, before it renders anything.

## 13. Preloaded Example Library

### 13.1 Example format

Forkscape's `Scenario` shape is extended with a Dockerfile for the problem, a Dockerfile for the fix, a recorded bundle for each, and captions generated from bundle values. The problem and the fix can have different numbers of instructions. Steps are aligned by **beat** (for example, "dependencies installed", "app copied", "image complete"). Each beat covers one or more instructions in each run, so the fix switch keeps the learner at the same point in the story (decision 5).

### 13.2 MVP examples

| ID | Symptom title | What it teaches | Fix and its cost | Instructions covered |
| --- | --- | --- | --- | --- |
| `the-handoff` | "Hundreds of megabytes to build, a few to ship" | The stage graph, reachability, `COPY --from`, what ships | Multi-stage build onto a distroless or `scratch` base. Cost: no shell in the runtime image, so debug with an ephemeral debug container | `FROM … AS`, `COPY --from`, `ARG TARGETOS/TARGETARCH`, `RUN --mount=type=cache`, `USER`, `ENTRYPOINT` |
| `deleted-still-shipped` | "We deleted the tarball and the image didn't shrink" | Whiteouts; the three sizes | Clean up in the same `RUN`, or `ADD --unpack`, or multi-stage. Cost: fewer, coarser cache points | `ADD` (URL), `RUN`, heredoc `RUN` |
| `cache-cascade` | "Every commit reinstalls all dependencies" | Cache keys and the cascade; `.dockerignore` | Copy manifests first; cache mount. Cost: two `COPY` steps, and the lockfile must stay in sync | `COPY . .`, `RUN --mount=type=cache`, `ARG`, build context |
| `layer-three-secret` | "The scanner found our token in an image we cleaned" | Files in lower layers; values in history and config | `RUN --mount=type=secret`. Cost: the build command and CI must pass the secret | `COPY`, `RUN`, `ARG`, `ENV`, `RUN --mount=type=secret` |
| `chown-doubles` | "One `chown` doubled the image" | Metadata changes copy whole files | `COPY --chown`/`--chmod`. Cost: ownership is fixed at copy time | `COPY`, `RUN`, `COPY --chown`, `COPY --chmod` |
| `user-isnt-owner` | "Works as root, `permission denied` as the app user" | `USER` versus file ownership | `--chown`, or create writable directories before `USER`. Cost: deciding what may be writable | `USER`, `COPY`, `WORKDIR`, `RUN` |
| `zero-layer-config` | "Twelve instructions, zero layers" | Config-only instructions; base inheritance; `ENTRYPOINT` resets `CMD`; shell versus exec form | Exec form and an explicit `CMD`. Cost: no shell expansion in exec form | `ENV`, `WORKDIR`, `EXPOSE`, `LABEL`, `CMD`, `ENTRYPOINT`, `VOLUME`, `HEALTHCHECK`, `SHELL`, `STOPSIGNAL`, `# check` |
| `arg-scope` | "The build arg is empty in my `RUN`" | Global versus stage `ARG`; predefined platform arguments; cache misses at first use | Declare the `ARG` again in the stage. Cost: none, but values still reach history | `ARG` (global and per stage), `FROM ${…}` |
| `writable-layer` | "The container's disk grew after touching one file" | Copy-up, whiteouts in the writable layer, `docker diff`, commit | A volume for mutable data. Cost: managing the volume's lifecycle | `VOLUME`, the run step |

Together these examples cover every instruction except `ONBUILD`, `MAINTAINER`, and the labs-only flags, which have coverage cards and extension examples. The coverage matrix is a test: CI fails if an instruction in Section 6 loses its example or card.

### 13.3 Extension examples

- `shared-layers`: "Why did pulling v2 download only 3 MB?" Content addressing, ChainIDs, and base-image updates.
- `copy-link-rebase`: `COPY --link` layers survive a base-image change. A bridge to buildpacks' rebase.
- `reproducible-digests`: "Same source, new digest on every build." Timestamps, `SOURCE_DATE_EPOCH`, and rewriting timestamps on export.
- `onbuild-surprise`: triggers firing in a child build.
- `flatten-or-not`: each way to get a single layer, and what each costs. Includes running a container from a flat `crane export` rootfs with no union filesystem.
- `mounted-over`: "The config I baked into the image is gone." A bind mount hides the image's directory, the runtime's generated `/etc/resolv.conf` replaces the image's, and a new named volume is seeded from the image on first use. If S1 shows that the run step carries the reframing, promote this example to the MVP.
- `multi-platform`: one tag, an index, per-architecture manifests, and attestation manifests.
- `labs-reset`: `FROM --reset`, and `ENV`, `LABEL`, and `VOLUME --unset`.

### 13.4 Free exploration

Paste or drop your own input, then edit the Dockerfile in the source pane. Static effects update as you type: reorder two lines and see the cache dependencies change. Nothing is graded, and only a session copy is kept.
## 14. Beyond Dockerfile-Built Containers

| Track | What differs | Why it is worth showing | Recommendation |
| --- | --- | --- | --- |
| **Multi-platform index and attestations** | One tag points to an index of per-platform manifests, plus attestation manifests (provenance, SBOM) | Engineers rarely see the index, and it explains "it works on my Mac" pulls | First extension; a small change to the bundle |
| **Images built without Dockerfiles:** ko, Jib, apko, Nix (`dockerTools.buildLayeredImage`, nix2container), Cloud Native Buildpacks | The tool chooses layer boundaries: by dependency, by package, or by Nix store-path popularity. Buildpacks can rebase app layers onto a new base without rebuilding them | Shows that "one instruction, one layer" is a Dockerfile convention, not a rule of the format | Second extension, as comparison examples built for the same app |
| **Lazy-pulling formats:** eStargz, SOCI, zstd:chunked, Nydus | Layers, or side indexes, that let a runtime fetch files on demand | Explains fast container starts; extends the ledger with "bytes actually fetched" | Later |
| **OCI artifacts:** Helm charts, Wasm modules, SBOMs and signatures via `subject` and the referrers API, AI models (Docker Model Runner, CNCF ModelPack), Kubernetes image volumes | Manifests whose layers are opaque blobs rather than filesystems; `artifactType` | "A registry is a content store, not a container store" | Later; a short tour |
| **Bootable OS images:** bootc (ostree, composefs) | A whole OS shipped as OCI, with layers chunked for efficient updates, deployed to disk rather than run under overlayfs | The same format with a different "run" step | Later; conceptual |
| **Windows containers** | Different base-layer handling and filesystem driver | — | Out of scope |
| **Non-OCI layered images:** qcow2 backing chains, live ISOs with squashfs and an overlay | Layering at the block or image level | An analogy that helps some learners | One comparison card at most |

## 15. Scope

### 15.1 MVP

- The nine examples in Section 13.2, each with a fix, captions, notes, evidence, a size ledger, and a run step.
- Static analysis of pasted Dockerfiles, with base-image metadata for common bases.
- Recorded-bundle and image-archive input.
- 3D and flat views from one state; source pane, merged tree, config card, ledger, inspector, and timeline; phone layout.
- The recorder CLI and reference bundles with versions recorded.

### 15.2 Deferred

The extension examples; the tracks in Section 14; Blender hero renders (after spike S5); private-registry input through the recorder; inferred `RUN` effects (decision 4); Remotion.

### 15.3 Non-goals

- Running builds in the browser.
- A replacement linter. The page surfaces BuildKit's build checks and invents no rules of its own.
- A registry browser or a vulnerability scanner.
- Automatically rewriting a user's Dockerfile. Fixes are shown for the examples.
- Windows containers.
- Story or video export from the app. The launch trailer is made outside it (Section 12), and turning arbitrary Dockerfiles into stories is a separate project (Section 12.5).

## 16. Delivery Phases and Exit Gates

Durations assume one frontend engineer with three.js experience and part-time review from an engineer who maintains production Dockerfiles and base images.

### Phase 0 — Spikes (about 2–3 weeks)

- **S1, disc stack:** a three.js prototype of the disc stack for `deleted-still-shipped` and `the-handoff`, covering the overview, moving the stack, isolating and rotating a layer, and the path core. Show it to five engineers without narration and ask why the image didn't shrink and what ships. Compare a sunburst with a circular treemap for the isolated view, area with log scaling for disc size, and `WebGLRenderer` with `WebGPURenderer`.
- **S2, parser:** compile BuildKit's Dockerfile parser to WASM. Measure size and startup time, and run it against a corpus of real Dockerfiles (heredocs, the escape directive, labs flags). Compare it with `dockerfile-ast`.
- **S3, recorder:** per-stage OCI export, line mapping from max-mode provenance, and entry extraction for all nine examples. Measure bundle sizes and check reproducibility across two runs. Answer the BuildKit questions in Section 19.
- **S4, archives in the browser:** read a `docker save` of an image of about 500 MB, measuring time, memory, and zstd handling.
- **Exit gate:** the disc stack's layout rules are confirmed or adjusted and written into the next revision of this plan; the parser is decided; bundle v1 is drafted; four of five engineers explain the whiteout and the hand-off from the prototype.

### Phase 1 — Model and first vertical slice (about 3–4 weeks)

- The semantic model, image model, and `WorldState`, with invariants and tests.
- `the-handoff`, `deleted-still-shipped`, and `cache-cascade` with fixes, in the flat view, with the source pane, merged tree, config card, and size ledger; phone layout; UI parity tests.
- **Exit gate:** the three examples run end to end on desktop and phone, and every caption number is asserted in tests.

### Phase 2 — 3D scene and remaining examples (about 3–4 weeks)

- The chosen framing in three.js, with motion and labels.
- The other six MVP examples and the run step.
- Still frames of The Handoff for the trailer storyboard, as an early check that the scene tells a story.
- **Exit gate:** all nine examples work in both views, and the coverage matrix test passes.

### Phase 3 — Your own Dockerfile, archives, and polish (about 2–3 weeks)

- Paste and edit with the WASM analyzer; base-image metadata; bundle and archive drop; free exploration; level of detail for large images; an accessibility pass and a manual screen-reader review; performance checks.
- **Exit gate:** a 500 MB archive and a 20-stage Dockerfile stay within budgets.

### Phase 4 — Launch storyboard and trailer (about 2–3 weeks)

- **S5, video:** render one beat of The Handoff by three.js capture, and again in Blender from exported glTF. Compare effort and quality.
- Capture mode in the app; `trailer/edit.json`; capture scripts; contact sheet; animatic; caption validator.
- The teaser, taken through the storyboard plan's review steps (phone size, muted, cold viewer).
- The Blender hero-shot pipeline, if S5 justified it.
- **Exit gate:** a teaser that can be reproduced from the bundles and the edit file, and that passes the truth checks.

### Phase 5 — Extensions

The multi-platform track first, then the comparison of images built without Dockerfiles, then the other extension examples, one at a time.

## 17. Risks and Mitigations

| Risk | Mitigation |
| --- | --- |
| The metaphor teaches fusion ("one final layer") | The commitments in Section 10.2; S1 tests the misconception directly |
| Static mode feels empty because `RUN` contents are unknown | Base-image metadata; a clear path to record; preloaded examples are always recorded |
| Real images have 100,000+ files | Rollups, largest-entries views, lazy expansion, instancing, and budgets from Phase 3 |
| Dockerfile syntax changes with frequent frontend releases | A pinned, versioned WASM parser; the parsing version shown on the page; a conformance corpus; Section 19 rerun on upgrade |
| User Dockerfiles expose private details | Processing stays in the browser and nothing is uploaded; the recorder redacts at the source |
| Two renderers drift apart | One model; Blender only consumes glTF exported from three.js; no model logic in Python |
| The video pipeline takes over the project | The storyboard plan's gates (stills, then animatic, then polish); Blender only after S5 |
| Example bundles go stale as base images update | Bundles pin digests and record dates; a refresh script; the badge shows the date |
| Sizes reported by different tools confuse learners | The ledger names each size and the command that reports it |
| Scope grows into a general Docker course | The four questions bound the content; the runtime is limited to the writable layer; networking and orchestration are out |

## 18. Decisions

All twelve were decided on 14 November 2026.

1. **Name.** *Decided:* **Layerviz**. "Layerscape" was rejected because it is an NXP processor line; "Stackscape", "Layerlens", and "Layercake" were earlier working names.
2. **The framing.** *Decided:* the light table, drawn as a stack of glass discs sized by bytes, which the learner moves up and down; tapping a disc isolates it to rotate and zoom (Section 10.1). S1 tests the layout rules, not the metaphor.
3. **The parser.** *Decided:* BuildKit's own parser and build checks compiled to WASM, lazy-loaded when a Dockerfile is pasted. Fall back to `dockerfile-ast`, with conformance tests against BuildKit, only if S2 finds it larger than about 3 MB compressed or slower than about 300 ms to initialize (Section 9.3).
4. **`RUN` contents in static mode.** *Decided:* unknown contents only. Without a recording, a `RUN` layer is a sealed disc labeled **STATIC**, with a prompt to record the build. There is no inference in the MVP.
5. **Fix alignment.** *Decided:* by beat. Problem and fix runs share a list of story beats, each covering one or more instructions. The switch keeps the learner at the same beat, and the layout never moves (Section 13.1).
6. **The run step.** *Decided:* every example ends with it. It shows the writable disc, the runtime mounts, and the view down through the stack (Section 6.2).
7. **The trailer's renderer.** *Decided:* deterministic capture of the app's three.js scene, with SVG/HTML overlays for captions and code. Blender hero shots come only after S5 shows a visible gain. Remotion is not used unless composing for several aspect ratios becomes the bottleneck (Section 11.3).
8. **Story and video export.** *Decided:* not in the app. The storyboard and hype trailer promote this project and are made outside the app, with capture scripts and, if justified, Blender (Section 12). A CLI that turns any Dockerfile into a story would be a completely separate project (Section 12.5).
9. **Base-image metadata.** *Decided:* a core set covering Alpine, Debian slim, Ubuntu, distroless, and the official Node, Python, and Go images at their current LTS or stable tags. A scheduled GitHub Actions job refreshes them monthly. Other bases appear as an unopened block (Section 8.1).
10. **Renderer.** *Decided:* start on `WebGLRenderer`. The S1 prototype also tries `WebGPURenderer`, and the app switches only if the glass discs and instanced tiles look or perform clearly better (Section 9.4).
11. **Code sharing with Forkscape.** *Decided:* copy and adapt, and never share a package. Forkscape is a reference, not a dependency, and the two apps diverge freely (Section 7.2).
12. **Bundle defaults.** *Decided:* the starting defaults in Section 8.4, which S3 tunes against the real examples.

## 19. Facts to Re-verify Against Current Versions

Check these against the reference-build versions and record the results in `docs/`:

- **BuildKit 0.35 and frontend 1.29:**
  - whether `WORKDIR`'s directory creation becomes its own layer;
  - whether a `RUN` that changes no files produces a layer or an `empty_layer` history entry;
  - the history `created_by` format, and how build-arg values appear in it (the reference says they are visible in `docker history` and in max-mode provenance);
  - what max-mode provenance contains, and whether its source locations map every layer to a Dockerfile line;
  - stage skipping and parallel stages, and how the legacy builder treats unreachable stages;
  - whether any `--squash` option is still supported;
  - how `COPY --link` layers are exported;
  - which flags are stable and which are labs-only. The 1.29 reference lists `FROM --reset` and `ENV`/`LABEL`/`VOLUME --unset` as labs-only; `COPY --parents`, `COPY --exclude`, `ADD --unpack`, and `RUN --device` appear in stable syntax;
  - the names of the build checks.
- **OCI image spec v1.1.1:** config fields; the whiteout and opaque-directory rules; layer media types (gzip, zstd); `artifactType` and `subject`. Also which fields are Docker extensions (`Healthcheck`, `OnBuild`, `Shell`).
- **Docker Engine 29:** that the containerd image store is the default; the layout of `docker save` output; which size `docker images` reports; `docker diff` with the containerd store; the maximum layer depth.
- **containerd 2.4:** that the overlayfs snapshotter keys snapshots by ChainID; how whiteouts are converted on unpack.
- **runc and Docker 29 at run time:**
  - Docker's default masked and read-only paths for non-privileged containers (the iximiuz tutorial lists them);
  - the order of mounts relative to `pivot_root`;
  - the rootfs propagation runc sets (the tutorial says `rslave`);
  - which `/etc` files are generated and bind-mounted;
  - whether a new named volume is seeded from image content, and how `volume-nocopy` changes that.
- **Browsers:** `DecompressionStream` support for zstd.
- **Tools:** Remotion's license terms; how Blender 5.2's glTF importer handles extras and custom properties; headless EEVEE rendering on a CI runner without a GPU; dive's maintenance status.

## 20. Definition of Done for the MVP

- The static site loads in current desktop and mobile browsers and reaches the library within budget.
- All nine MVP examples run in 3D and flat views, each with its fix, ledger, notes, evidence, run step, and cost.
- Every instruction in Section 6 has an example or a coverage card, and the coverage test passes.
- Every view carries an accurate source badge. Reference notes carry their versions and date.
- A pasted Dockerfile is analyzed in the browser, and a `docker save` archive of about 500 MB stays within budget.
- UI parity, accessibility, and performance checks pass, and a manual screen-reader review is complete.
- A reviewer who maintains production base images has signed off on every example's claims.

## 21. Immediate Next Actions

1. Run reference builds of `the-handoff` and `deleted-still-shipped` in a `labctl` VM with pinned Docker Engine and Buildx. Check the line mapping from max-mode provenance, per-stage export, and the `WORKDIR` and empty-layer questions. Start one container from each image to capture its runtime spec, mount tree, and overlay `lowerdir` list.
2. Compile BuildKit's Dockerfile parser to WASM, and measure its size and startup time.
3. Prototype the disc stack for `deleted-still-shipped` in three.js, and show it to five engineers.
4. Draft bundle v1 and the `WorldState` types.
5. Write the next revision of this plan with the disc stack's choreography: stacking, looking through the stack, isolating a layer, whiteouts, the hand-off between stages, and the run step.

## 22. Reference Material

- [OCI image specification](https://github.com/opencontainers/image-spec) (v1.1.1): manifest, index, config, layer changesets, and whiteouts.
- [Dockerfile reference](https://docs.docker.com/reference/dockerfile/), and the [frontend 1.29 source of the reference](https://github.com/moby/buildkit/blob/dockerfile/1.29.0/frontend/dockerfile/docs/reference.md).
- [BuildKit](https://github.com/moby/buildkit) and [Buildx](https://github.com/docker/buildx).
- [containerd](https://github.com/containerd/containerd), and the [Linux overlayfs documentation](https://docs.kernel.org/filesystems/overlayfs.html).
- Ivan Velichko, [How Container Filesystem Works: Building a Docker-like Container From Scratch](https://labs.iximiuz.com/tutorials/container-filesystem-from-scratch) (iximiuz Labs; updated March 2026): mount namespaces, `pivot_root`, runtime mounts, and the claim that a union filesystem is optional. See also [How To Extract Container Image Filesystem Using Docker](https://labs.iximiuz.com/tutorials/extracting-container-image-filesystem).
- [dive](https://github.com/wagoodman/dive), [dockerfilegraph](https://github.com/patrickhoefler/dockerfilegraph), [go-containerregistry (crane)](https://github.com/google/go-containerregistry), [regclient](https://github.com/regclient/regclient), and [ORAS](https://github.com/oras-project/oras).
- [Blender 5.2 LTS release coverage](https://www.phoronix.com/news/Blender-5.2-Released).
- Sibling plans: [PROCESS_FORK_VISUALIZATION_PLAN.md](PROCESS_FORK_VISUALIZATION_PLAN.md), [EKS_CONTENTION_PLAN.md](EKS_CONTENTION_PLAN.md), [FRONTEND_PERF_VIZ.md](FRONTEND_PERF_VIZ.md) §12.8, and [STORYBOARD_PLAN.md](STORYBOARD_PLAN.md).

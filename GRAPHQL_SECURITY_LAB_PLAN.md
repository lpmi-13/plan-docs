# GraphQL Security Lab — iximiuz Labs Deployment Plan

## 1. Objective

Turn Intigriti's ["Five easy ways to hack GraphQL targets"](https://www.intigriti.com/researchers/blog/hacking-tools/five-easy-ways-to-hack-graphql-targets)
into a hands-on **iximiuz Labs challenge** set in an authentic production **Django +
Strawberry** GraphQL service. For each of the five techniques a learner:

1. **Confirms the weakness exists** by probing the live backend from the VM terminal with
   a preinstalled toolbox (manual, ungraded — see §5),
2. **Fixes it by editing the real project**, and
3. **Gets automated confirmation** — the lab polls the live GraphQL server and flips a
   check green, driving the iximiuz UI feedback exactly like the verification tasks in
   [`/home/adam/projects/iximiuz-challenges/cka-kubeadm-upgrade-ecfc7390`](/home/adam/projects/iximiuz-challenges/cka-kubeadm-upgrade-ecfc7390).

The lab is **closed-world**: every probe targets only the backend on the VM's own
loopback. No exercise points a tool at an external host.

The teaching spine is a **defaults-first** idea (§4): the service starts from **stock
Strawberry defaults**, which are themselves the vulnerable state, and the learner hardens
it. Nothing in the code is contrived or commented to look like a lab.

## 2. Platform: `kind: challenge` on a custom baked rootfs

We combine both iximiuz patterns, as decided earlier:
- **From cloudtracing** — a **custom OCI rootfs drive** (`oci://…-rootfs`) baked with the
  Django/Strawberry app, the attacker toolbox, and a systemd unit, so the VM boots with
  the backend already serving.
- **From the CKA challenge** — `kind: challenge` with **verification tasks** wired to
  `::simple-task` directives for live pass/fail feedback.

The CKA file's note that a challenge "can't reference a custom playground" is treated as
**stale**: we assume a challenge's `playground:` block can carry a custom
`drives: [ { source: oci://… } ]` **until proven otherwise** at first publish. Fallback if
not: a public playground + an init task that pulls the same image (slower, needs egress).
We don't design for the fallback now.

## 3. Stack: Django + Strawberry over ASGI (the authentic production setup)

- **GraphQL engine:** [Strawberry](https://strawberry.rocks) (`strawberry-graphql` +
  `strawberry-graphql-django`) — the prevailing 2026 recommendation for new Django GraphQL
  work, and the only Python option where all five fixes are real, documented, surgical
  changes (notably field suggestions, §6 #2).
- **Serving:** Strawberry's **`AsyncGraphQLView`** (ASGI — Strawberry's own recommended
  path) mounted at `/graphql/`, run under **uvicorn** (optionally `gunicorn -k uvicorn`),
  listening on `:8000`, exposed as an iximiuz `http-port` tab.
- **Django app:** a small, ordinary project — `users`, `profiles`, `posts` models on
  sqlite; session-cookie auth; a state-changing mutation (`updateBio`); a mock
  `verifyOtp(code)` mutation; and a nested/recursive field for the DoS exercise. Enough
  real schema that introspection and field-suggestion recon return something meaningful.
- **Production posture:** `DEBUG=False`. GraphiQL may stay enabled on the explorer tab for
  learning (and is itself a talking point), though true production would set
  `graphiql=False`; noted, not one of the five graded items.

> This is "a GraphQL server for a Django application" served directly by Django. The
> Apollo-gateway-over-a-Django-subgraph topology (the other authentic reading) is **out of
> scope** for the MVP; it splits each weakness across router vs. subgraph and adds a Node
> tier. It can be layered later as an advanced variant.

## 4. Authenticity + the defaults-first model

Hard rules:
- The project reads like a **stock Strawberry/Django app**. **No comments, flags, TODOs,
  or naming hint at the vulnerabilities or at what to change.** All learner guidance lives
  in the challenge lesson body and `::hint-box` directives in `index.md`, never in the
  code. The lesson may name a file path (training material); the file stays silent.
- **The starting point is literally the defaults.** `schema.py` is
  `strawberry.Schema(query=Query, mutation=Mutation)` — no `extensions`, no `config`.
  `urls.py` wraps the view in `csrf_exempt`, straight from Strawberry's own docs. That
  minimal, idiomatic setup **is** the vulnerable surface for four of the five techniques;
  nothing is deliberately mis-set.
- **Hardening is the task.** The learner progressively *adds* `extensions=[…]`, a
  `config=…`, and a custom validation rule, and *removes* the `csrf_exempt` wrapper. By the
  end the schema is hardened — a realistic "secure this service" progression.

Why this is cleaner than any "set insecure values" approach: with Strawberry the insecure
state is the default, so we barely configure anything to start, and there is nothing to
hide.

## 5. Feedback model: one live-polled check per technique

Per technique there is exactly **one graded verification task**, `patch_<tech>`, whose
`run:` script **polls the live endpoint** and exits 0 only when it now behaves securely
(and, where relevant, a legitimate request still works). No evidence files, no filesystem
state — purely the server's responses.

- **Confirming the weakness exists** is a manual, **ungraded** step with the preinstalled
  toolbox; the lesson shows the exact commands.
- **Deferred:** auto-grading the "confirm it exists" step by capturing terminal output in
  the challenge UI — possible later, not in the MVP.

Result: **five** graded tasks + one readiness init task, each rendered as a `::simple-task`.

## 6. The five techniques (Strawberry/Django specifics)

All settings are ordinary app config in their natural place; "patch" = edit the real file
and restart (§9). Probes are capped to stay safe on a shared VM.

| # | Technique | Vulnerable at stock defaults | The learner's change | Where | `patch_<tech>` live poll passes when… |
|---|-----------|------------------------------|----------------------|-------|----------------------------------------|
| 1 | Introspection | ✅ introspection on by default | add `DisableIntrospection()` to `extensions` | `schema.py` | a full `__schema` query is refused / returns no schema |
| 2 | Field suggestions | ✅ "Did you mean" on by default | add `StrawberryConfig(disable_field_suggestions=True)` as `config` | `schema.py` | a misspelled-field query's error contains no "Did you mean" |
| 3 | CSRF | ✅ view ships wrapped in `csrf_exempt` (per docs) | **remove** the `csrf_exempt` wrapper | `urls.py` | a forged no-token POST mutation is rejected (403) **and** a request with a valid CSRF token still succeeds |
| 4 | Alias rate-limit bypass | ✅ no alias/cost cap (and a naive per-request limiter exists) | **write a custom validation rule** capping aliases, added via `AddValidationRules([...])` | new `validation.py` + `schema.py` | one request packed with many aliased `verifyOtp` fields is rejected |
| 5 | Batched/expensive-query DoS | ✅ no depth limit by default | add `QueryDepthLimiter(max_depth=N)` to `extensions` | `schema.py` | a deeply nested query is rejected with a depth error |

### 6.1 Technique #4 is deliberately an "add code" exercise
Unlike the others, Python/Strawberry has **no built-in alias or cost cap** (there is no
`graphql-armor` equivalent), so the fix is to *write* a small graphql-core
`ASTValidationRule` that counts aliased fields and rejects over a threshold, then register
it with Strawberry's `AddValidationRules` extension. A request-level rate limiter
(`django-ratelimit`, counting HTTP requests per IP) is part of the app's normal setup so
the bypass is real: aliases pack many `verifyOtp` attempts into **one** HTTP request the
limiter never splits. This is the most valuable exercise in the set — the learner builds
the control, not just toggles it. (`graphene-protector`'s complexity limit is a valid
off-the-shelf alternative to mention in a hint.)

## 7. Ordering & orthogonality

Controls are surgical and non-overlapping, so no taught patch flips another green:
- `DisableIntrospection`, the suggestion `config` flag, and `csrf_exempt` removal each
  touch only their own concern.
- **#4 vs #5** are kept distinct on purpose: **#4 = alias count** (shallow, cheap probe),
  **#5 = query depth** (deeply nested probe). The alias rule ignores depth; the depth
  limiter ignores alias count. Avoid a broad "complexity" limit as the taught fix for
  either, since it would cover both.

Recommended introduction order **1 → 2 → 3 → 4 → 5** (also the article's order). The one
dependency: **#1 before #2**, because turning introspection off is what makes "recover the
schema via field suggestions" the authentic next step — and #1's fix does **not** remediate
#2 (suggestions survive introspection being disabled), so #2 remains demonstrable.

> Edge case: a learner could install all of `graphene-protector` and cover #4/#5 at once —
> still a valid remediation; the lesson teaches the surgical, per-technique fix.

## 8. `index.md` structure (challenge + baked rootfs)

One `index.md` (YAML front matter + Markdown lesson) plus `__static__/`, matching the CKA
challenge layout, but the `playground:` block carries the custom drive.

### 8.1 Front matter sketch
```yaml
---
title: 'Harden a Django GraphQL API: Five Classic Flaws'
description: |
  A stock Django + Strawberry GraphQL service ships with five common weaknesses. Probe
  each one, fix it in the project, and watch the lab confirm the remediation.
kind: challenge

playground:
  tabs:
    - machine: dev-01
    - id: graphiql
      kind: http-port
      name: graphiql
      machine: dev-01
      number: 8000
  machines:
    - name: dev-01
      users:
        - name: root
        - name: laborant
          default: true
          welcome: |
            A Django + Strawberry GraphQL API is running at http://127.0.0.1:8000/graphql/
            (loopback only). Work through the tasks: probe each weakness with the tools in
            ~/toolbox, then harden the project under /opt/graphql-lab and restart with
            `sudo systemctl restart graphql-lab`. The lab checks the live endpoint.
      drives:
        - source: oci://ghcr.io/<owner>/graphql-lab-rootfs:v1
          mount: /
          size: 20GiB
      resources:
        cpuCount: 2
        ramSize: 4GiB

difficulty: medium
categories: [security]
tagz: [graphql, django, strawberry, appsec]

tasks:
  init_wait_ready:
    name: init_wait_ready
    machine: dev-01
    init: true
    user: root
    timeout_seconds: 180
    run: |
      #!/bin/bash
      for _ in $(seq 1 120); do
        curl -fsS -X POST http://127.0.0.1:8000/graphql/ \
          -H 'content-type: application/json' -d '{"query":"{__typename}"}' >/dev/null 2>&1 \
          && exit 0
        sleep 1
      done
      echo "GraphQL backend did not become ready in time" >&2; exit 1

  patch_introspection:
    machine: dev-01
    run: |
      BODY=$(curl -fsS -X POST http://127.0.0.1:8000/graphql/ \
        -H 'content-type: application/json' \
        -d '{"query":"{__schema{types{name}}}"}' || true)
      echo "$BODY" | jq -e '(.data.__schema == null) and (.errors | length > 0)' >/dev/null \
        && echo "Introspection disabled" || exit 1

  patch_field_suggestions:
    machine: dev-01
    run: |
      BODY=$(curl -fsS -X POST http://127.0.0.1:8000/graphql/ \
        -H 'content-type: application/json' \
        -d '{"query":"{ usrs { id } }"}' || true)   # misspelled: user(s) -> usrs
      echo "$BODY" | grep -qi 'did you mean' && exit 1
      echo "Field suggestions blocked"

  patch_csrf:
    machine: dev-01
    run: |
      jar=$(mktemp)
      # forged POST with no CSRF token must now be rejected ...
      FORGED=$(curl -s -o /dev/null -w '%{http_code}' -X POST http://127.0.0.1:8000/graphql/ \
        -H 'content-type: application/json' \
        -d '{"query":"mutation{ updateBio(text:\"x\"){ ok } }"}')
      # ... while a request carrying the csrftoken cookie + header still works
      curl -s -c "$jar" http://127.0.0.1:8000/graphql/ >/dev/null
      TOKEN=$(awk '/csrftoken/{print $7}' "$jar")
      OK=$(curl -s -b "$jar" -X POST http://127.0.0.1:8000/graphql/ \
        -H "x-csrftoken: $TOKEN" -H 'content-type: application/json' \
        -d '{"query":"mutation{ updateBio(text:\"x\"){ ok } }"}' | jq -r '.data.updateBio.ok')
      [ "$FORGED" = "403" ] && [ "$OK" = "true" ] && echo "CSRF enforced" || exit 1

  patch_alias_limit:
    machine: dev-01
    run: |
      Q=$(python3 - <<'PY'
print("{" + " ".join(f"a{i}: verifyOtp(code:\"{i:04d}\"){{ ok }}" for i in range(60)) + "}")
PY
)
      CODE=$(curl -s -o /dev/null -w '%{http_code}' -X POST http://127.0.0.1:8000/graphql/ \
        -H 'content-type: application/json' --data "$(jq -nc --arg q "$Q" '{query:$q}')")
      [ "$CODE" != "200" ] && echo "Alias abuse capped" || exit 1

  patch_depth_limit:
    machine: dev-01
    run: |
      CODE=$(curl -s -o /dev/null -w '%{http_code}' -X POST http://127.0.0.1:8000/graphql/ \
        -H 'content-type: application/json' \
        --data '{"query":"{ me{ posts{ author{ posts{ author{ posts{ id }}}}}} }"}')
      [ "$CODE" != "200" ] && echo "Deep query rejected" || exit 1
---
```
(Field names/queries are illustrative; finalized against the real schema. The CSRF poll
relies only on Django's CSRF token handling, not on an authenticated session.)

### 8.2 Lesson body (per technique)
```markdown
## 1. Introspection

Probe it:

```bash
~/toolbox/introspect.sh      # dumps the schema if introspection is on
```

Then harden the schema and restart the service.

::simple-task
---
:tasks: tasks
:name: patch_introspection
---
#active
Checking the live endpoint for introspection...
#completed
Introspection is disabled — the schema no longer leaks.
::

::hint-box
---
:summary: Hint
---
Strawberry ships a schema extension for this; it goes in `Schema(extensions=[...])`.
::
```
No `#active`/`#completed` text or hint reveals file contents; the project stays
comment-free (§4).

## 9. Runtime & the "edit the real project" mechanism

- The Django project lives at `/opt/graphql-lab`, baked into the rootfs, served by uvicorn
  under a systemd unit (`graphql-lab.service`) started by a bootstrap one-shot (mirrors
  cloudtracing's `*-bootstrap.service`).
- The learner edits the **real** files — `graphql_lab/schema.py`, `graphql_lab/urls.py`,
  and (for #4) a new `graphql_lab/validation.py` — and restarts with
  `sudo systemctl restart graphql-lab` (wrapped as `~/toolbox/reload.sh`; optionally run
  uvicorn with `--reload` so edits auto-apply). The files carry no lab comments.
- `~/toolbox/reset.sh` restores the pristine (default/vulnerable) project from a baked
  copy so any technique can be redone.
- The graded `patch_*` tasks poll the live endpoint only (§5), so manual testing and the
  automated checks share one source of truth: the server's responses.

## 10. Attacker toolbox + closed-world scoping

Baked into the rootfs under `~/toolbox`, all pointed at loopback: `graphw00f`
(fingerprint), `clairvoyance` (schema recovery for #2), `graphql-cop` / `batchql`
(CSRF/batching), `curl`/`jq`/`httpie`, a static GraphQL Voyager page for #1, and thin
wrapper scripts (`introspect.sh`, etc.) so the manual confirm step is one command. `InQL`
(Burp) is out of scope (no browser/Burp in the VM); the lesson notes how you'd use it on
your own machine. Every wrapper hard-codes `http://127.0.0.1:8000/graphql/`.

## 11. Build, validate, publish

- **Repo:** `iximiuz-challenges/graphql-security-<id>/` with `index.md`, `README.md`,
  `__static__/`, and `app/` holding the Django/Strawberry project (built into the
  `graphql-lab-rootfs` image). A `playground/iximiuz/` dir holds the `Dockerfile`,
  bootstrap unit, and build/publish scripts cloned from cloudtracing's shape.
- **Rootfs build:** `FROM` the iximiuz rootfs; install Python + the app's deps + the
  toolbox; COPY the project + systemd unit; `docker build`/push the OCI image. **Pin**
  `strawberry-graphql` (to a release that includes the field-suggestion config flag and is
  past the CSRF-exempt-by-default advisory) and `strawberry-graphql-django` so defaults and
  patch options match the lesson exactly.
- **Local validation harness (release gate):** boot the image; for each technique run the
  `patch_*` poll (expect fail at defaults) → apply the change → restart → run the poll
  (expect pass) → `reset.sh`. All five flip correctly = gate.
- **Publish:** author as a challenge for labs.iximiuz.com; push the rootfs image and the
  challenge via the iximiuz authoring flow / `labctl`. **First publish must confirm the §2
  assumption** (custom drive accepted on a challenge); if rejected, switch to the
  public-playground-+-init-task fallback.

## 12. Open questions / trade-offs

- **§2 assumption** (custom rootfs on a challenge) — the key risk; verify early.
- **Exact Strawberry APIs to confirm when building:** the field-suggestion config
  attribute name (from Strawberry PR #3537 — expected `disable_field_suggestions`) and its
  minimum version; that `AddValidationRules` is the current way to register a custom rule;
  `QueryDepthLimiter`'s current parameter (`max_depth`, and `should_ignore` vs older
  `ignore`). Pin to the resolved versions.
- **CSRF default vs. version:** Strawberry's Django view shipped CSRF-*exempt* by default
  before v0.243.0; we model the vuln via the documented `csrf_exempt` wrapper in `urls.py`
  on a pinned current version, so the fix is a clean one-line removal regardless.
- **#4 rate-limiter realism:** include `django-ratelimit` on the view so the alias bypass
  is genuine; confirm it counts per HTTP request (so aliases defeat it).
- **HTTP-array batching:** Strawberry's Django view batching is unconfirmed, so #5 is
  framed as a single deep/expensive query (still the article's DoS point). If batching is
  supported, add the array form as a bonus.

## 13. First implementation steps

1. Build the Django + Strawberry project (`app/`): models (`users`/`profiles`/`posts`),
   session auth, `updateBio` mutation, mock `verifyOtp`, a nested field for depth; stock
   `schema.py` (no extensions/config) and `urls.py` (`csrf_exempt` per docs); add
   `django-ratelimit` on the view. **No lab comments.** Pin versions.
2. Build the custom rootfs (project + uvicorn systemd unit + toolbox); boot it and confirm
   `:8000/graphql/` serves and is vulnerable at defaults.
3. Write the five `patch_*` live-poll scripts and the local validation harness; confirm each
   flips vulnerable→patched with the intended change (incl. writing the alias `AddValidationRules`
   rule for #4) and reverts via `reset.sh`.
4. Author `index.md` (custom-drive `playground:`, readiness init task, five graded tasks,
   lesson body with `::simple-task` + `::hint-box` in order 1→5).
5. Publish; **verify the custom-drive-on-challenge assumption**, falling back to init-task
   provisioning only if rejected.

## 14. References

- Intigriti — [Five easy ways to hack GraphQL targets](https://www.intigriti.com/researchers/blog/hacking-tools/five-easy-ways-to-hack-graphql-targets)
- Strawberry — [DisableIntrospection extension](https://strawberry.rocks/docs/extensions/disable-introspection),
  [QueryDepthLimiter](https://strawberry.rocks/docs/extensions/query-depth-limiter),
  [Django views (ASGI, csrf_exempt)](https://strawberry.rocks/docs/django/guide/views),
  [MaskErrors](https://strawberry.rocks/docs/extensions/mask-errors),
  field-suggestion flag [PR #3537](https://github.com/strawberry-graphql/strawberry/pull/3537)
- graphene-django — [installation/CSRF docs](https://docs.graphene-python.org/projects/django/en/stable/installation/),
  [query validation (DisableIntrospection, depth_limit_validator)](https://docs.graphene-python.org/en/stable/execution/queryvalidation/)
- [graphene-protector](https://pypi.org/project/graphene-protector/) (depth/complexity limits for graphene & strawberry)
- graphql-core — [issue #97: disabling "Did you mean" suggestions](https://github.com/graphql-python/graphql-core/issues/97)
- Ariadne — [Hiding field suggestions](https://ariadnegraphql.org/server/hiding-field-suggestions)
- Escape — [When GraphQL field suggestions become a security issue](https://escape.tech/blog/graphql-verbose-error-suggestions/)
- [graphql.org — Security best practices](https://www.graphql.org/learn/security)
- Strawberry vs graphene (2026): [muehlemann-popp](https://muehlemann-popp.ch/insights/graphene-vs-strawberry-which-is-better-for-providing-a-graphql-api),
  [leapcell](https://leapcell.io/blog/building-robust-graphql-apis-in-python-with-graphene-and-strawberry)

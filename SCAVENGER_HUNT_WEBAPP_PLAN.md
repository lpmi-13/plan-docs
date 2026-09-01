# Scavenger Hunt Web Application — Implementation Plan

## 1. Executive Summary

Build a mobile-first progressive web application (PWA) that replaces the existing
Viber conversation while preserving the language-learning scavenger-hunt loop:

1. reveal an intentionally ambiguous photograph and useful local-language phrases;
2. encourage the participant to ask a nearby person for directions;
3. validate a fresh location reading at the destination;
4. unlock the next clue without leaking navigation hints; and
5. atomically identify the first eligible finisher.

The recommended first release uses Next.js and TypeScript for the PWA, with Supabase
Postgres/PostGIS, Auth, Storage, and Realtime for managed backend capabilities. The
participant experience requires no account or app-store installation. Administrators
authenticate; participants join a single hunt using a short code and receive a
revocable, hunt-scoped session.

This plan takes the source concept through a safe staff pilot. It treats image
search-resistance as author guidance and light deterrence, not a guarantee. It also
treats participant location—especially where students may be minors—as sensitive,
short-lived evidence rather than general analytics data.

Expected elapsed time for a small experienced team is approximately four to six
weeks through a staff-only field pilot, assuming decisions and school reviews are
available promptly. The estimate must be revised after Phase 0 confirms class size,
devices, route conditions, data residency, and safeguarding requirements.

## 2. Product Outcomes, Principles, and Boundaries

### 2.1 Goals

- Let a teacher create, preview, publish, run, and delete an ordered scavenger hunt.
- Let many participants join the same hunt without creating personal accounts.
- Make the current clue and phrase sheet fast and usable on an ordinary phone.
- Validate progress only on the server using fresh browser geolocation evidence or an
  explicitly configured staff fallback.
- Determine one first finisher correctly even when several final check-ins arrive at
  nearly the same time or a request is retried.
- Give clue authors useful warnings about searchable text, logos, landmarks, poor
  crops, and unusable images.
- Operate at a cost suitable for a teacher or small school while retaining a credible
  path to paid capacity, backups, and migration.
- Minimize personal and location data, provide auditable administration, and support
  the school's privacy and safeguarding obligations.

### 2.2 Product Principles

- **Conversation first:** participant feedback must not become a map, compass, range
  finder, or warmer/colder game unless a teacher deliberately enables such a mode in
  a later release.
- **Server authority:** the browser may present state, but it never decides progress,
  completion, rank, or winner status.
- **Minimal disclosure:** a participant can access only the hunt lobby, their current
  clue, their own coarse progress, and the results the teacher has chosen to reveal.
- **Honest deterrence:** image tooling reports evidence and suggestions; it never
  labels a clue “Google-proof” or “cheat-proof.”
- **Safe ambiguity:** clues may be visually ambiguous, but routes, physical locations,
  and instructions may not be unsafe or inaccessible by design.
- **Recoverable operations:** publishing validates a frozen route version; moderation,
  fallback check-ins, winner changes, exports, and deletion are deliberate and
  audited.
- **Provider portability at meaningful seams:** keep auth, hunt persistence, image
  storage, and optional image analysis behind narrow interfaces, without building a
  speculative multi-cloud framework.

### 2.3 Non-Goals for the First Release

- Preventing screenshots, screen photography, coordinate spoofing, collusion, or
  determined image searching with certainty.
- Continuous participant tracking, route recording, or background geolocation.
- Public participant profiles, social feeds, direct messages, or advertising.
- Native iOS or Android applications.
- Team play, branching routes, randomized routes, or user-authored public hunts.
- Automated accusation or disqualification based solely on anomaly detection.
- Full offline completion. The current clue may be cached, but acceptance requires an
  online server decision.
- School-wide SSO, enterprise tenancy, billing, or a general marketplace in the MVP.

## 3. Success Measures and Release Gates

The following are acceptance targets, not claims to be made before testing:

| Area | Pilot target |
| --- | --- |
| Core flow | A teacher can author and publish a hunt; two or more independent participants can complete it in order. |
| Winner integrity | A concurrent final-check-in test produces exactly one initial winner and a stable finish order on every run. |
| Retry safety | Repeating the same join/check-in mutation cannot duplicate an entry, stage advance, completion, or winner. |
| Authorization | Automated cross-hunt and cross-entry access tests all fail closed. |
| Location privacy | No precise coordinate appears in routine logs, analytics, error tracking, CSV exports, or participant responses. |
| Accessibility | Critical participant and admin flows pass the agreed automated checks and a keyboard/screen-reader review; a non-GPS fallback is documented. |
| Weak connectivity | A previously opened current clue and phrase sheet remain readable offline; check-in clearly waits for a new online attempt. |
| Operations | Staff can pause, resume, extend, end, export, use a fallback, and invoke the documented deletion path. |
| Safety and privacy | School reviewers have no unresolved high-risk finding before students use the system. |
| Capacity | A load test at the confirmed event size plus the agreed safety margin meets the response-time/error budget. |

No student launch occurs until the field pilot, route walk, data review, and operational
rehearsal gates pass.

## 4. Assumptions and Decisions Required

### 4.1 Working Assumptions

These assumptions keep implementation moving but must be confirmed in Phase 0:

- A hunt has one fixed, ordered stage sequence in the MVP.
- Participants play individually on one browser/device; team entries are deferred.
- Participant display names need only be recognizable to the teacher and need not be
  legal names, email addresses, or globally unique.
- Late joining is allowed only when configured by the teacher.
- Stage answer locations and acceptance radii are frozen once a hunt is published.
- The first valid final check-in serialized by the database is the initial winner.
- A disqualification does not erase history. If policy allows, an administrator must
  explicitly promote the next eligible finisher and provide a reason.
- Failed location coordinates are not retained by default. Accepted coordinates are
  also discarded after validation unless a documented audit need requires short-term
  retention.
- The teacher provides a facilitator method for participants who cannot use GPS.
- English is the initial authoring-interface language; clue content and phrase text
  support Unicode and a teacher-selected local language.

### 4.2 Decisions to Close During Discovery

| Decision | Why it matters | Owner | Needed by |
| --- | --- | --- | --- |
| Maximum participants per hunt and simultaneous check-ins | Drives capacity test, rate limits, and service tier | Product/school | Architecture sign-off |
| Countries, ages, residency, and school policy | Drives hosting region, notices, retention, and review | School privacy lead | Any production data |
| Individual versus team play | Changes entry identity, progress, and ranking | Product | Data-model freeze |
| Tie, disqualification, and promotion rules | Determines visible results and audit behavior | Teacher/product | Winner implementation |
| Precise-location retention need and duration | Determines schema, cleanup jobs, and export policy | Privacy lead | Check-in implementation |
| GPS accuracy thresholds and fallback type | Must reflect actual route/device performance | Route owner | Publish validation |
| Expected connectivity | Determines cache validation and field-test scenarios | Route owner | PWA implementation |
| Magic link, passkey, or school identity provider | Affects auth UX and deliverability | School IT/product | Admin auth implementation |
| Hosting vendor, region, budget, and support level | Affects runtime adapter, recovery, and launch readiness | Technical owner | Foundation milestone |
| Whether results are private, delayed, or live | Affects participant disclosure and safeguarding | Teacher | Results implementation |
| Fixed, random, or branching clue order beyond MVP | Determines whether the simple stage index remains sufficient | Product | Post-pilot roadmap |

Record decisions in an architecture decision log. A change to a closed decision must
identify affected migrations, tests, notices, and operational procedures.

## 5. Users, Roles, and Authorization Model

### 5.1 Roles

| Role | Capabilities |
| --- | --- |
| Hunt owner | Full authoring, publishing, operations, export, admin management, and deletion for owned hunts |
| Hunt editor | Edit drafts and previews; no deletion or ownership transfer unless explicitly granted |
| Hunt operator | View live dashboard and perform approved event-time actions such as pause, extend, fallback validation, or disqualify |
| Participant | Join one hunt, read only their current clue/progress, submit their check-in, withdraw, and view configured results |
| Support operator (optional) | Time-limited, audited support access; absent from MVP unless a real support process requires it |

Administrators have authenticated user identities. A participant receives a random
entry/session credential stored in a secure, HTTP-only, same-site cookie. The server
stores only a one-way digest of that credential. It is revocable and scoped to one
entry and one hunt.

The application must not imply that an anonymous browser session proves a unique
human. “One active entry” means one entry per issued participant subject/session; any
stronger identity rule requires a school account, roster code, or facilitator process.

### 5.2 Authorization Rules

- Every admin mutation checks both authentication and hunt-level role.
- Every participant request resolves the session digest to one non-revoked entry.
- The current-clue endpoint selects the expected stage on the server; it never accepts
  an arbitrary stage ID from the participant as authority.
- Answer coordinates, future stages, other entries, full rankings, and private images
  are not returned to participant clients.
- Service credentials never ship to the browser. Database row-level security (RLS)
  remains enabled as defense in depth even when route handlers mediate access.
- Storage buckets are private. Signed image access is issued only for the current clue
  and expires quickly enough to limit casual sharing without breaking field use.
- Realtime subscriptions expose authorized aggregates or a safe event projection, not
  unrestricted participant/check-in rows.

## 6. User Journeys and State Models

### 6.1 Administrator Journey

1. Sign in with the selected managed authentication method.
2. Create a draft with title, language, timezone, start/end time, route area, safety
   contact, participant rules, result visibility, and retention settings.
3. Add ordered stages. For each stage, upload an image, add non-spoiling alt text,
   optional clue copy, ordered direction-seeking phrases, answer pin, radius, and an
   optional staff/venue verification method.
4. Review upload processing and image findings. Crop, mask, or replace the image, or
   record an override reason for remaining findings.
5. Preview the participant lobby, wait state, every stage, failure response, completion
   screen, and offline clue view at mobile widths.
6. Run publish validation. Resolve all blocking errors and explicitly acknowledge any
   warning-level image/safety issues.
7. Publish the immutable route version and distribute a join URL, short code, and QR.
8. Monitor joined, active, completed, withdrawn, fallback-validated, and disqualified
   counts. Pause/resume, extend the end time, or end without selecting a winner.
9. Review the auditable finish order. Disqualify only with a recorded reason and, if
   policy permits, explicitly promote the next eligible finisher.
10. Export the minimal configured result set and delete or allow scheduled cleanup of
    participant data.

### 6.2 Participant Journey

1. Open a join URL or scan a QR, enter/confirm the hunt code, and choose a
   teacher-visible display name.
2. Read and accept a concise versioned safety/privacy notice. If the hunt has not
   started, remain in a wait state that reveals no clue.
3. Read the current clue image, clue copy, alt text, and “phrases to ask” panel. No map,
   coordinate, distance, bearing, EXIF data, or future clue is present.
4. At the destination, tap **Check my location**. Only then request a fresh,
   high-accuracy browser reading.
5. Submit the reading with its browser timestamp, reported accuracy, stage context,
   and a newly generated idempotency key.
6. On rejection, receive a neutral status and practical retry guidance such as moving
   outdoors or enabling location—not distance or direction.
7. On acceptance, receive the next stage or final persisted result. A retry returns
   the same accepted outcome.
8. If GPS cannot be used, follow the teacher-configured facilitator path; the admin
   action is visible in the audit trail.
9. Withdraw if needed and see only the result detail permitted by the teacher.

### 6.3 Hunt Lifecycle

Use explicit persisted states plus time-window checks:

```text
draft --publish--> published <----resume---- paused
                         | --------pause----> |
                         |                    |
                         +------end---------->+--> ended

draft/published/paused/ended --delete request--> pending_deletion --> deleted
```

- `draft`: freely editable and not joinable.
- `published`: route content is frozen. Before `starts_at`, participants see the wait
  state; between start/end, joins and check-ins follow settings.
- `paused`: no progress is accepted. Existing participants can see a clear pause
  banner and their current cached clue unless the operator selects concealment.
- `ended`: no joins or check-ins. The hunt may have no winner.
- `pending_deletion`: access is disabled while storage/database cleanup is retried and
  verified.
- `deleted`: represented only by minimal non-personal deletion/audit evidence if the
  retention policy requires it.

Publishing creates a route revision. Event-time edits are limited to documented safe
fields such as end-time extension, pause state, support contact, and result visibility.
Changing images, order, answer pins, radii, or verification rules requires a new draft
revision or cloned hunt.

### 6.4 Entry Lifecycle

```text
joined --> active --> completed
   |         |
   +---------+--> withdrawn
   +---------+--> disqualified
```

An entry holds the next expected stage position. Only an accepted server transaction
can advance it. Completion, withdrawal, and disqualification are persisted states;
revoking the session does not remove the audit record prematurely.

## 7. Scope by Release

### 7.1 Playable Vertical Slice

- Responsive PWA shell and install metadata.
- Admin authentication and basic hunt/stage CRUD.
- Private image upload, server-side decode/re-encode, rendition generation, and EXIF
  removal.
- Fixed ordered stages with answer pin and configurable radius.
- Anonymous participant join and hunt-scoped session.
- Lobby, current clue/phrases, fresh geolocation request, neutral failure, and
  sequential progress.
- Transactional idempotent check-in and exactly one initial winner.
- Basic admin results view.
- Automated authorization, radius-boundary, retry, and concurrent-finish tests.

### 7.2 Safe Pilot Scope

- OCR and low-cost image findings, checklist, crop/mask, override reason, and publish
  validation.
- Exact participant preview and QR/join-code sharing.
- Live dashboard with pause/resume, extension, end, fallback, withdrawal,
  disqualification, and explicit promotion behavior.
- Minimal CSV export and configurable deletion/retention job.
- Rate limiting, audit trail, accessibility pass, current-clue offline cache,
  monitoring, capacity test, and backup/restore exercise.
- Staff-only field test under representative route and connectivity conditions.

### 7.3 Deferred Enhancements

- Paid landmark, logo, web-entity, or licensed reverse-image-search checks behind a
  teacher-visible per-hunt budget.
- Rotating on-site QR codes, facilitator PINs, or venue approval workflows.
- Team entries, school SSO, templates, reusable phrase libraries, translations,
  randomized/branching routes, and richer post-hunt analytics.
- Formal multi-school tenancy, billing, provider migration, or native applications.

## 8. Recommended Technical Architecture

### 8.1 Components

| Component | Initial choice | Responsibility |
| --- | --- | --- |
| Web application | Next.js with TypeScript | Admin and participant UI, server-rendered/public-safe pages, route handlers, session boundary |
| PWA layer | Web app manifest and service worker | App shell and strictly scoped current-clue caching |
| Database | Supabase Postgres with PostGIS | Authoritative domain state, spatial checks, constraints, transactions, audit records |
| Authentication | Supabase Auth or selected managed adapter | Administrator identity, magic link/passkey/session lifecycle |
| Object storage | Supabase Storage private buckets | Quarantined originals, safe renditions, and temporary transformed assets |
| Realtime | Supabase Realtime over safe projections/events | Live admin counts and event refresh hints |
| Image worker | Background function/worker suited to image libraries | Decode, validate, re-encode, OCR, perceptual hash, crop/mask derivatives |
| Error monitoring | Privacy-configured hosted service when justified | Client/server exceptions and performance without names, images, or coordinates |
| Email | Managed auth email provider | Admin sign-in only; participant email is not required |

Choose one server-logic home for each use case. Prefer Next.js route handlers as the
HTTP boundary and database functions for transactions that must be atomic. Use an edge
function or worker only for a capability that is operationally unsuitable in the web
runtime, such as CPU-heavy image processing. Do not maintain duplicate business logic
in route handlers and edge functions.

### 8.2 Logical Flow

```text
Admin browser ---------------------+
                                    \
Participant browser ---> Next.js route handlers ---> Postgres/PostGIS
          |                         |                    |
          |                         |                    +--> safe realtime projection
          |                         +--> private storage
          |                                  |
          +<--- signed safe rendition <--- image worker / analysis queue
```

### 8.3 Service Interfaces

Keep these seams narrow and covered by contract tests:

- `AuthService`: resolve administrator identity and role; issue/revoke admin session.
- `ParticipantSessionService`: issue, hash, resolve, rotate, and revoke entry sessions.
- `HuntRepository`: draft CRUD, publication, lifecycle transitions, dashboard queries,
  export selection, and deletion orchestration.
- `CheckInService`: validate request, execute the database decision, and return a safe
  persisted outcome.
- `ImageStore`: accept quarantined upload, read private original, store rendition,
  sign current-clue access, and delete all derivatives.
- `ImageAnalysisService`: queue analysis and normalize findings from local or optional
  third-party engines.
- `AuditService`: append immutable, redacted administrative and security events.

Interfaces are not permission bypasses. Each implementation still enforces tenancy,
ownership, safe projections, and redaction.

### 8.4 Environments and Configuration

Maintain separate local, preview/test, staging, and production resources. Never use
student production data in previews or automated tests.

- Validate all environment variables at startup.
- Keep server/service keys in the hosting secret store; expose only intended public
  configuration to the browser.
- Manage schema and RLS as reviewed migrations in source control.
- Seed local/staging with synthetic hunts, users, images, and coordinates.
- Pin runtime and dependency versions; update through tested pull requests.
- Configure region, base URL, cookie domain, upload limits, rate limits, retention,
  geolocation thresholds, and optional provider budgets per environment.
- Use feature flags only for risky integrations or staged release, not as a substitute
  for a coherent lifecycle model.

## 9. Domain and Data Model

Use UUID primary keys, UTC `timestamptz`, explicit foreign keys, check constraints,
non-null fields where meaningful, and PostGIS `geography(Point, 4326)` for meter-based
distance calculations.

### 9.1 Core Tables

| Table | Important fields and constraints |
| --- | --- |
| `profiles` | `user_id`, display name; no global role that accidentally grants every hunt |
| `hunts` | owner, title, language, timezone, lifecycle status, start/end, code digest, settings, route revision, initial/current winner, finish counter, retention policy; end after start |
| `hunt_admins` | hunt, user, role; unique pair |
| `stages` | hunt, position, clue text, safe alt text, answer geography, acceptance radius, verification settings, image asset; unique `(hunt_id, position)` and positive radius |
| `stage_phrases` | stage, position, local text, optional translation/pronunciation; unique stage position |
| `image_assets` | owner/hunt, original and rendition object keys, checksums, media dimensions, processing state, EXIF-stripped flag, analysis version/status |
| `image_findings` | asset, rule/provider, category, severity, evidence region/text, recommendation, resolution state |
| `entries` | hunt, participant subject/session digest reference, display name, notice version/time, status, next stage position, join/last-active/completion times, finish position, disqualification reason |
| `check_ins` | entry, stage, idempotency key, request fingerprint, server time, captured time, reported accuracy, method, decision, reason code, optional short-lived point, response snapshot |
| `audit_events` | actor type/id, hunt, action, target type/id, redacted metadata, server time, correlation ID |
| `deletion_jobs` | hunt/asset scope, requested time, state, attempts, last safe error, completed time |

Optional provider reports should use their own table or encrypted/private object when
large. Do not place raw OCR text from clue images into logs or public search indexes.

### 9.2 Required Constraints and Indexes

- Unique stage order within a hunt.
- Unique active participant subject within a hunt, within the limits of anonymous
  identity.
- Unique `(entry_id, idempotency_key)` for check-in replay.
- A foreign key from `hunts.winner_entry_id` to an entry in the same hunt, enforced by
  transaction logic plus a database trigger/constraint strategy where practical.
- Entry stage position cannot be negative or exceed the published stage count.
- Finish position is unique per hunt when non-null.
- Join code digest is indexed and unique among joinable hunts.
- Spatial index on stage answer geography.
- Index dashboard queries by hunt/status and check-ins by entry/server time.
- Prevent deletion of an image asset still referenced by a published route.

Avoid putting frequently queried domain fields only in an unvalidated settings JSON.
Use JSON for bounded, versioned settings whose schema is validated at the application
boundary.

### 9.3 Data-Minimization Policy

The spatial decision requires a coordinate in memory and within the transaction; it
does not automatically justify indefinite storage.

- Default: store decision, accuracy, method, stage, and server time, but no raw point.
- Failed attempts: discard the point immediately after the decision; retain only a
  coarse reason such as `outside_area`, `poor_accuracy`, or `stale_reading`.
- Accepted attempts: retain a raw point only if the school has approved a specific
  audit purpose and short expiry. Otherwise retain the outcome without the point.
- Logs and error trackers: never record request bodies, coordinates, signed image
  URLs, session tokens, join codes, or unnecessary display names.
- CSV: default to display name, entry status, join/completion times, finish position,
  winner/disqualification state, and verification method; exclude coordinates.
- Deletion: remove sessions, entries, check-ins, exports, images, reports, and storage
  derivatives according to policy, then record only minimal completion evidence.

## 10. API and Server Contracts

Use a versioned `/api` boundary, schema validation at every input, stable machine error
codes, safe human messages, request correlation IDs, and an idempotency contract for
mutations vulnerable to retry.

### 10.1 Participant Endpoints

| Endpoint | Contract |
| --- | --- |
| `POST /api/join` | Accept code, display name, and notice version; rate-limit; create or resume the scoped entry; set secure cookie; never reveal whether unrelated private hunts exist |
| `GET /api/play` | Return lobby/wait/pause/ended state or only the expected current stage and safe signed rendition |
| `POST /api/check-ins` | Require session and `Idempotency-Key`; accept latitude, longitude, accuracy, captured time, and expected stage token; execute authoritative decision |
| `POST /api/withdraw` | Idempotently withdraw the current entry and revoke further progression |
| `GET /api/results/me` | Return only the participant's completion and teacher-configured result disclosure |
| `DELETE /api/session` | Revoke local participant session; this is distinct from policy-governed result deletion |

The expected-stage token is opaque and short-lived. It prevents accidental submission
of a cached stage but is not the source of truth; the database entry remains
authoritative.

### 10.2 Admin Endpoints

Provide authenticated routes for:

- hunt list/create/read/update and safe lifecycle transitions;
- stage/phrase ordering and validation;
- image upload initiation, completion, findings, crop/mask, replace, and override;
- preview and publication validation;
- share link/code rotation and QR generation;
- dashboard snapshot plus authorized realtime refresh;
- pause, resume, extend, end, fallback acceptance, withdraw, disqualify, and promote;
- result export generation/download with expiry;
- retention settings, deletion request, deletion status, and admin/audit review.

Every state-changing admin request records actor, action, target, reason when required,
server time, and correlation ID. Destructive or winner-changing operations require a
confirmation screen that names the hunt and explains the effect.

### 10.3 Error Behavior

- Return stable codes such as `HUNT_NOT_ACTIVE`, `READING_STALE`, `ACCURACY_TOO_LOW`,
  `CHECK_IN_NOT_ACCEPTED`, `STAGE_ALREADY_COMPLETE`, and `RATE_LIMITED`.
- Participant copy for an outside-area decision remains neutral; the response contains
  no distance, direction, answer coordinates, or effective radius.
- Authentication and authorization errors reveal no other hunt's title or state.
- Replaying the same idempotency key and same fingerprint returns the saved semantic
  status/body. Reusing the key with a different fingerprint returns conflict without
  changing state.
- A server timeout leaves the client able to retry the same key safely.

## 11. Location Check-In and Winner Correctness

### 11.1 Browser Acquisition

Request geolocation only after the participant taps the check-in control:

- use high-accuracy mode;
- require `maximumAge: 0` or equivalent fresh acquisition behavior;
- use a field-tested timeout and provide a clear retry/fallback path;
- send coordinate, reported accuracy in meters, browser capture time, and current
  stage token over TLS;
- do not watch position in the background or continuously update the UI.

Permission denial, timeout, unavailable location, and poor accuracy need distinct,
accessible instructions. Do not silently loop on geolocation prompts.

### 11.2 Server Decision

Within the service/database boundary:

1. Validate numeric ranges, payload size, session, hunt window/status, entry state,
   expected stage, idempotency key, and request fingerprint.
2. Reject readings too old, implausibly in the future, or above the field-tested
   maximum accuracy threshold.
3. Construct the submitted and answer points with SRID 4326 as `geography` and compute
   meter distance using PostGIS.
4. Apply the published uncertainty policy. The source product intent is to accept when
   the reported accuracy circle plausibly intersects the configured destination
   radius. Put a maximum on acceptable reported accuracy and freeze both values at
   publication so poor readings cannot expand the area without bound.
5. Apply any configured short answer/facilitator proof on the server.
6. Persist the safe result and advance exactly once when accepted.
7. Return a safe outcome that contains no calculated distance or bearing.

The exact maximum accuracy, reading age, uncertainty allowance, and timeout are
configuration defaults established by field measurements—not hard-coded product
truths. Radius values below 30 m show an author warning; the suggested starting radius
is 75 m, subject to route testing.

### 11.3 Transaction for Progress and Winner

The accepted check-in runs in one database transaction:

1. Resolve or reserve `(entry_id, idempotency_key)`. If a completed matching record
   exists, return its response snapshot; if its fingerprint differs, return conflict.
2. Lock the entry and re-read hunt status, time window, expected stage, and eligibility.
3. Evaluate location and optional proof against the published stage.
4. Insert the decided check-in. For failure, commit the safe result without advancing.
5. For a non-final success, increment the expected stage exactly once.
6. For a final success, lock the hunt's finish counter, allocate the next finish
   position, mark the entry complete, and set the initial winner only if no winner has
   yet been persisted.
7. Save the response snapshot derived from the committed state and commit.

The client receives success only after commit. Database serialization—not device
clock, arrival order observed in JavaScript, or a realtime event—defines finish order.
A unique finish-position constraint and conditional winner update provide additional
defense.

### 11.4 Moderation and Winner Changes

- Preserve the original finish position and initial-winner evidence.
- A disqualification requires admin role, reason, and audit record.
- Never automatically accuse a participant from spoofing/travel heuristics.
- If promotion is allowed, show the next eligible finisher and require explicit
  confirmation. Record old winner, new winner, actor, reason, and time.
- Ending a hunt without selecting a winner is a supported action.
- Results label provisional/final status according to the teacher's configured policy.

## 12. Image Ingestion and Search-Resistance Guidance

### 12.1 Processing Pipeline

1. Create an upload intent authorized for the draft hunt.
2. Upload to a private quarantine location with byte and dimension limits.
3. Verify actual file signature, decode with a hardened library, reject unsupported or
   decompression-bomb-like input, and ignore the client filename/type as authority.
4. Correct orientation, re-encode to approved output formats, resize responsive
   renditions, and strip EXIF/GPS/comments/embedded thumbnails.
5. Calculate content and perceptual hashes.
6. Run local OCR and low-cost quality/distinctiveness checks asynchronously.
7. Store normalized findings with evidence regions, severity, analysis version, and
   actionable guidance.
8. Let the author crop/mask or replace the asset; always regenerate from the private
   source and rerun relevant checks.
9. Promote only the safe rendition to publishable state. Originals remain private and
   follow the hunt's deletion schedule.

Authoring remains responsive while processing. A stage cannot publish while its image
is quarantined, failed, missing alt text, or has unresolved blocking findings.

### 12.2 MVP Checks

- OCR for street names, business names, URLs, phone numbers, plaques, house numbers,
  and readable signs.
- Blur, contrast, resolution, exposure, and feature-density signals for unusably vague
  or trivially distinctive images.
- Author checklist for full landmark/façade, unique art, transit stop, branding,
  street sign, and number visibility.
- Center/edge crop suggestions and masking preview for identifying text.
- Perceptual-hash comparison only within the school's authorized image corpus.
- Required review questions: “Could a participant identify this without asking a
  person?” and “Does this contain searchable text or a famous landmark?”

Normalize the outcome to **likely too easy**, **review recommended**, or **no obvious
issues found**. The third result explicitly means only that configured checks found no
obvious issue.

### 12.3 Optional External Analysis

Place cloud OCR, landmark/logo/web-entity detection, licensed reverse image search,
and image-capable model advice behind `ImageAnalysisService`. Before activation:

- approve provider terms, region, retention, training-use policy, and data-processing
  agreement;
- show the author what leaves the system;
- set a per-hunt call/cost budget and timeout;
- degrade gracefully when a provider is unavailable;
- record provider/model version and report age;
- prohibit scraping or automating consumer image-search interfaces.

### 12.4 Delivery Deterrents

Use private storage, expiring signed URLs, metadata stripping, non-indexed participant
pages, no social preview image, and optional unobtrusive participant watermarking.
These reduce accidental discovery and sharing but do not prevent screenshots. Do not
add fake controls such as disabling right-click and present them as security.

## 13. PWA, Offline, and Connectivity Behavior

The service worker may cache:

- versioned application shell assets;
- the authenticated participant's current safe clue rendition;
- current clue text and phrase sheet in a scoped response;
- static accessibility/help content.

It must not cache answer locations, future clues, raw API mutations, admin pages,
participant lists, exports, session tokens, or precise results.

Cache entries are keyed by hunt route revision, entry, and current stage; clear them on
stage advance, withdrawal, session revocation, hunt end, user-requested data clearing,
or deployment schema incompatibility. Avoid storing sensitive payloads in browser
`localStorage`.

Offline behavior is explicit:

- A cached clue remains readable with an offline banner.
- Check-in is disabled or produces a clear “connect and try again” state.
- Do not queue a location mutation for automatic background replay because it may be
  stale by reconnection time.
- Once online, acquire a new reading and submit a new request.
- A service-worker update must not strand an in-progress entry; use a compatible cache
  migration or controlled activation strategy.

Test throttled, intermittent, captive-portal-like, and reconnect conditions on actual
target phones, not only desktop browser simulation.

## 14. UX and Accessibility Requirements

### 14.1 Participant Experience

- Prioritize clue image, optional clue copy, and an always-obvious phrases panel.
- Use large touch targets, readable type, high contrast, visible focus, and no
  color-only status.
- Keep one primary action per state: join, start/check status, check location, next
  clue, or view completion.
- Announce geolocation acquisition, validation, failure, stage unlock, pause, and
  completion through an appropriate screen-reader live region without repeated noise.
- Provide non-spoiling meaningful alt text authored and previewed by the teacher.
- Keep failure language encouraging and neutral. Never show “12 m closer” or similar.
- Explain location permission immediately before use and why it is requested only now.
- Preserve the current clue through orientation changes, browser restarts, and safe PWA
  updates where possible.

### 14.2 Admin Experience

- Make publish blockers distinct from warnings and link each item to its editor.
- Provide both map and accessible non-pointer mechanisms for answer placement, such as
  search/manual coordinates and keyboard-adjustable controls.
- Show the acceptance circle and an accuracy warning without exposing it in participant
  preview.
- Preview common phone widths, screen-reader text order, offline state, and every
  participant response state.
- Require confirmation and reason for override, fallback acceptance,
  disqualification, promotion, end, code rotation, export, and deletion as appropriate.
- Avoid a live participant map. The dashboard shows status/aggregate activity unless a
  separately approved safety use case requires more.

Target the accessibility standard agreed with the school; WCAG 2.2 AA is the proposed
baseline. Automated tools are a gate, not a substitute for keyboard, screen-reader,
zoom, contrast, reduced-motion, and outdoor-device testing.

## 15. Security, Privacy, Safety, and Anti-Cheating

### 15.1 Security Controls

- TLS everywhere; secure, HTTP-only, same-site cookies; origin/CSRF protection;
  restricted CORS; and a strict content security policy.
- Parameterized queries, schema validation, output encoding, safe Markdown/rich-text
  policy, and stored-XSS tests for every teacher-authored field.
- Upload signature sniffing, decode/re-encode, pixel/byte limits, private quarantine,
  and no direct execution or public original URL.
- Rate limits by IP, join code/hunt, entry, action, and failure pattern. Limits must
  allow expected classroom NAT/concurrency and provide an operator recovery path.
- High-entropy, non-sequential join codes stored as digests, with rotation and expiry;
  rate limits make short-code guessing impractical for the event threat model.
- Random session tokens stored only as digests server-side; revoke on withdrawal,
  deletion, moderation, or suspicious recovery action.
- Least-privilege database roles, RLS, private buckets, reviewed security-definer
  functions with fixed search paths, and no browser service key.
- Dependency, secret, and migration checks in CI; documented patch cadence.
- Backup encryption, restore authorization, and expiry aligned with the deletion
  promise.

### 15.2 Threat Model Summary

| Threat | Primary mitigation | Residual risk/response |
| --- | --- | --- |
| Guessing/share of join code | Entropy, throttling, rotation, event window | Codes can be shared; teacher can rotate and remove entries |
| Cross-hunt/entry data access | Server-scoped queries, RLS, IDOR tests | Monitor denied access without logging secrets |
| GPS spoofing | Freshness/accuracy checks, anomaly flags, optional on-site proof | Browser coordinates remain spoofable; teacher review is authoritative |
| Multiple final taps/retries | Idempotency key, row locks, unique constraints, saved response | Operational retry returns persisted truth |
| Simultaneous finishers | Serialized hunt finish counter and conditional winner write | Tie policy must be communicated in advance |
| Search/share of clues | Author review, OCR/crops, private short-lived renditions, late reveal | Screenshots and determined search cannot be prevented |
| Malicious image/text upload | MIME sniff, hardened decode/re-encode, limits, encoding/CSP | Keep libraries patched and quarantine failures |
| Public leaderboard harm | Private/minimal results by default | Teacher explicitly chooses any broader disclosure |
| Admin account takeover | Managed auth, passkey where available, short sessions, audit | Recovery procedure and rapid session revocation |
| Service outage during event | Rehearsal, paid capacity if needed, status checks, manual fallback | Staff runbook determines pause/extension and communication |

Impossible-speed, identical-coordinate, and repeated-failure heuristics create a
teacher-visible review flag only. They must not automatically disclose, shame, or
disqualify a participant.

### 15.3 Privacy and Safeguarding

- Complete the school's privacy/data-protection assessment before student use.
- Use pseudonymous display names where practical and avoid participant email by
  default.
- Version the safety/privacy notice and retain only acceptance version/time.
- Request location only on check-in; never run background tracking.
- Define route boundaries, accessibility, supervision expectations, emergency contact,
  weather/cancellation plan, and prohibited/private areas.
- Avoid destinations requiring unsafe crossings, trespass, interaction with unwilling
  parties, or disclosure of personal information.
- Keep leaderboards off by default, especially for minors.
- Give teachers a manual, non-GPS route that does not publicly disclose disability.
- Make retention and one-click hunt deletion visible in the admin UI; document backup
  expiry separately so deletion claims remain accurate.

Legal conclusions depend on jurisdiction and school policy and require the school's
qualified reviewers; this implementation plan is not a legal determination.

## 16. Observability and Operational Evidence

### 16.1 Structured Events

Capture redacted, structured events for:

- admin sign-in outcome and role/authorization denial;
- hunt create, validate, publish, pause, resume, extend, end, and delete;
- image processing state and finding counts, without sending originals to monitoring;
- join success/failure category;
- check-in decision category, accuracy bucket, latency, and idempotent replay;
- stage advance, completion, initial winner allocation, disqualification, and promotion;
- realtime connection state and dashboard refresh failures;
- export creation/download/expiry and deletion-job outcome.

Use internal UUIDs or rotating pseudonymous identifiers. Redact display names, join
codes, cookies, bearer tokens, signed URLs, clue OCR text, exact coordinates, and free
text that may contain personal data.

### 16.2 Metrics and Alerts

- HTTP error rate and latency by safe route class.
- Join/check-in rate and rate-limit rejections per hunt.
- Geolocation rejection categories and accuracy buckets.
- Database connection saturation, transaction retries, and winner-allocation errors.
- Storage failures, image-worker backlog/age, and signed-URL errors.
- Realtime lag/disconnects; the dashboard must fall back to safe polling/manual refresh.
- Retention/deletion jobs overdue or repeatedly failing.
- Backup age and scheduled restore-drill result.
- Provider quota/budget thresholds before a live event.

Alerting must route to a named event operator and include a documented action. Avoid
collecting data merely because a dashboard can display it.

## 17. Test Strategy

### 17.1 Unit and Property Tests

- Hunt and entry lifecycle transition tables.
- Publication validation and immutable-field rules.
- Geospatial acceptance around inside, exact boundary, outside, reported-accuracy,
  stale/future reading, poles/antimeridian where supported, and invalid-coordinate
  cases.
- Request canonicalization/fingerprinting and idempotency conflict behavior.
- Stage progression and final-stage detection.
- Result visibility, retention date calculation, and export column selection.
- Image finding severity/resolution rules and safe alt-text validation.
- Rate-limit keying and privacy redaction helpers.

### 17.2 Database and Integration Tests

- RLS for every role/table/view and cross-hunt/cross-entry denial.
- Private object access and expiration of signed renditions.
- Check-in rollback at each transaction boundary.
- Duplicate/reordered requests and same key with changed payload.
- Many concurrent valid final check-ins: unique finish positions and exactly one initial
  winner.
- Concurrent duplicate check-ins for one entry: one advancement and stable response.
- Pause/end/expiry racing with a check-in.
- Disqualification/promotion audit integrity.
- Join-code rotation, participant-session revocation, and classroom-NAT rate limits.
- Retention/deletion across database rows, storage renditions, exports, and retry after
  partial cleanup.
- Migration forward/rollback strategy against production-like synthetic data.

### 17.3 Browser and Accessibility Tests

- Admin authoring from draft through preview, validation, publish, dashboard, export,
  and deletion.
- Participant wait, join, notice, permission grant/deny, poor accuracy, success,
  failure, pause, offline clue, reconnect, completion, withdrawal, and expired session.
- Mobile Safari and Chrome on the agreed minimum devices plus representative desktop
  admin browsers.
- Keyboard-only admin and participant flows, focus order/restoration, screen-reader
  announcements, 200%/400% zoom, contrast, reduced motion, and large text.
- Service-worker update and cache-clearing rules; prove future clues and admin data are
  absent from caches.
- QR/deep-link handling and PWA installed/browser modes.

Mocked geolocation makes automated cases repeatable, but final validation includes
real devices and real GPS at representative destinations.

### 17.4 Security Tests

- IDOR/cross-tenant access, role escalation, code enumeration, session fixation, CSRF,
  CORS, and cookie attributes.
- Stored/reflected XSS in all author/display-name fields and CSV formula injection.
- Malformed/polyglot/oversized image uploads and decompression limits.
- SQL injection, schema overposting, mass assignment, and unsafe error disclosure.
- Rate-limit bypass patterns and denial-of-service behavior within agreed scope.
- Cache poisoning and leakage of signed image URLs, future stages, or coordinates.
- Secret scanning, dependency advisories, and CSP verification.

### 17.5 Performance and Resilience Tests

Use the Phase 0 participant target plus agreed margin. Exercise burst joins, synchronized
stage check-ins, concurrent finishes, dashboard subscriptions, signed-image retrieval,
and image processing separately from the event-critical path.

- Define p95/p99 targets after choosing region/provider and measuring a vertical slice.
- Test database pool limits and transaction retries.
- Simulate realtime failure; admin controls remain usable through ordinary requests.
- Simulate image-analysis outage; published hunts remain playable.
- Simulate response loss after a committed check-in and verify safe replay.
- Verify backup restoration into an isolated environment and application-level read.

## 18. Delivery Plan and Work Breakdown

Estimates assume one or two experienced developers, prompt product/school decisions,
and no enterprise identity integration. Safety/privacy review time may extend elapsed
delivery without implying engineering work should bypass it.

### Phase 0 — Discovery and Technical Spikes (about 1 week)

**Product and policy**

- Close or assign every decision in Section 4.
- Define participant count, result disclosure, tie/promotion, retention, fallback,
  incident ownership, and student-use gate.
- Draft versioned safety/privacy copy and route checklist for school review.

**Field and technical validation**

- Measure geolocation accuracy/latency on representative iOS/Android devices at indoor,
  outdoor, dense-building, and weak-connectivity destinations.
- Test candidate radii and uncertainty thresholds without exposing navigation feedback.
- Spike PostGIS meter-based checks and concurrent winner serialization.
- Spike the chosen Next.js host/runtime with Supabase Auth, private signed images,
  service-worker behavior, and realtime authorization.
- Confirm provider region, quotas, sleep behavior, email delivery, backups, and cost
  alerts against current terms.

**Artifacts**

- Decision log, threat/privacy/safeguarding draft, route test evidence, architecture
  diagram, initial schema/API contracts, capacity target, and prioritized backlog.

**Exit gate:** representative phones obtain usable readings at intended types of
location; the school approves the proposed data/safety model; the selected stack has
no unresolved feasibility blocker.

### Phase 1 — Foundation and Playable Vertical Slice (about 2 weeks)

**Foundation**

- Create app, typed configuration, environments, CI checks, migrations, RLS baseline,
  synthetic seed data, and deployment pipeline.
- Implement managed admin authentication and hunt-scoped role checks.
- Implement private image upload/re-encode/EXIF stripping and safe rendition access.

**Authoring**

- Build draft hunt/stage/phrase CRUD, ordering, answer-pin/radius editor, basic mobile
  preview, and minimal publication validation.

**Participant loop**

- Build join/session, notice, wait/lobby, current clue/phrases, geolocation acquisition,
  server validation, neutral failure, and sequential advancement.
- Implement idempotency fingerprint/replay, database transaction, finish allocation,
  initial winner, and private results.

**Verification**

- Add unit/integration/browser tests for authorization, location boundaries, stale and
  poor readings, retry, reordered calls, two independent entries, and concurrent final
  check-ins.

**Exit gate:** two browsers join the same private hunt, progress independently, and
produce one persisted winner under repeated concurrency tests. No client can request a
future clue or write progress/winner state directly.

### Phase 2 — Authoring Quality and Safe Operations (about 1–2 weeks)

**Image guidance**

- Add asynchronous processing states, OCR, quality checks, checklist, findings report,
  crop/mask regeneration, override reasons, safe alt text, and publish blockers.

**Event operation**

- Add exact participant preview, share link/code/QR, dashboard aggregates/realtime
  refresh, pause/resume, extension, end-without-winner, fallback check-in,
  disqualification, explicit promotion, and audit views.
- Add minimal CSV export with formula-injection protection and expiring download.

**Hardening**

- Add rate limiting, CSP/security headers, complete RLS tests, upload abuse tests,
  privacy redaction, retention/deletion jobs, monitoring, and alerts.
- Add scoped offline clue cache, reconnect behavior, and accessible status handling.
- Complete keyboard/screen-reader review, capacity test, and isolated backup restore.

**Exit gate:** a staff-only hunt completes on the real route under weak connectivity;
support actions and fallback work; deletion is verified; reviews have no unresolved
high-risk security, privacy, accessibility, or safeguarding finding.

### Phase 3 — Controlled Student Launch and Stabilization

- Clone a practice hunt and rehearse at target concurrency.
- Walk the exact route, validate every pin/radius/image, and confirm staffed fallback.
- Warm services, verify quotas/alerts/status contacts, and freeze risky deployments.
- Run the first event with named technical and safeguarding operators.
- Export only approved fields, collect staff feedback, review anomalies without blame,
  execute promised deletion, and write a short incident/learning report.
- Reprioritize the roadmap from measured failure categories, support burden, cost, and
  participant/teacher feedback.

**Exit gate:** the first controlled event meets the agreed success measures, and any
material incident has an owned remediation before broader use.

### Phase 4 — Optional Enhancements

Prioritize only from demonstrated need: stronger on-site proof, paid image checks,
teams, school SSO, templates, phrase libraries, translations, route variants, analytics,
multi-school tenancy, or provider migration.

## 19. Implementation Epics and Acceptance Criteria

### Epic A — Platform and Identity

- Reproducible local setup and isolated deployed environments.
- Admin can authenticate, sign out, recover access, and see only authorized hunts.
- Participant session is random, digest-stored, secure-cookie-based, revocable, and
  hunt-scoped.
- CI applies lint/type/test/migration/security checks and rejects exposed secrets.

### Epic B — Hunt Authoring and Publication

- Teacher can create, reorder, preview, and validate stages/phrases.
- Pin/radius placement has accessible alternatives and physical-route warnings.
- Publish freezes a route revision and returns all blockers in one actionable report.
- Event-time edits cannot mutate route-critical fields.

### Epic C — Image Safety and Guidance

- Only a decoded/re-encoded, metadata-stripped rendition can reach a participant.
- OCR/quality/checklist findings are versioned and resolvable.
- Crop/mask/replacement reruns checks; override stores actor and reason.
- Report wording never claims search-proof status.

### Epic D — Join and Participant Experience

- Valid join creates/resumes one scoped entry; invalid/rate-limited behavior leaks no
  private hunt detail.
- Current-stage response contains no answer point, future clue, distance, or unrelated
  entry data.
- Notice, wait, pause, withdraw, ended, and session-expired states are complete and
  accessible.

### Epic E — Check-In, Progress, and Winner

- Server validates fresh location and published policy; browser cannot self-advance.
- Duplicate requests converge on one saved response and one state change.
- Concurrent finishers receive unique finish positions with exactly one initial winner.
- Participant failure response reveals no navigational information.

### Epic F — Dashboard and Moderation

- Authorized operators see accurate aggregate counts and a fallback when realtime is
  unavailable.
- Pause/resume/extend/end/fallback/disqualify/promote actions are transactional,
  permission-checked, confirmed where needed, and audited.
- Initial outcome and later moderation remain distinguishable.

### Epic G — Privacy, Export, and Deletion

- Precise coordinates are absent from routine logs, monitoring, participant responses,
  and default exports.
- CSV includes only configured fields and neutralizes spreadsheet formulas.
- Retention jobs are idempotent, observable, retryable, and cover storage/backups as
  documented.
- Admin can see deletion status and any safe remediation instruction.

### Epic H — PWA, Accessibility, and Resilience

- Current clue/phrases remain readable offline after a successful online load.
- No stale location check-in is background-replayed; reconnect requires a new reading.
- Critical flows pass automated plus manual accessibility checks on target devices.
- Event runbook, load test, monitoring, fallback, and restore drill are complete.

## 20. Deployment, Migration, and Rollback

- Use reviewed forward migrations with preflight checks and a backup before risky
  production schema changes.
- Prefer backward-compatible expand/migrate/contract changes so old and new app
  versions can overlap briefly.
- Treat RLS/policy changes as production code with explicit regression tests.
- Deploy to staging with synthetic data, run smoke/browser/concurrency checks, then use
  a controlled production promotion.
- Avoid schema changes, provider migrations, or dependency upgrades immediately before
  a scheduled hunt.
- Application rollback must not roll the database into an incompatible state. Provide
  a forward-fix migration when data has already changed.
- Image worker changes retain analysis version so older reports can be invalidated or
  rerun deliberately.
- Service-worker releases use versioned caches and a tested activation policy.
- Provider abstraction is exercised by contract tests, but an actual migration is
  undertaken only for cost, reliability, or compliance evidence.

## 21. Live-Hunt Runbook

### 21.1 Several Days Before

- Walk the route; confirm physical safety, accessibility, permission, pin, radius,
  fallback, mobile signal, and emergency contact at every stage.
- Pilot each image with OCR, ordinary text/image search, and someone unfamiliar with
  the route.
- Clone and publish a practice hunt; test actual participant devices and admin recovery.
- Confirm result visibility, timezone, start/end, late join, winner, tie,
  disqualification, promotion, and deletion policies.
- Check service status, quotas, billing alerts, backup age, restore evidence, email
  delivery, certificates, monitoring, and operator access.

### 21.2 Immediately Before

- Warm any service susceptible to inactivity delay and run health/readiness checks.
- Submit synthetic join/check-in traffic at the agreed scale without touching the live
  route state.
- Verify support contacts, facilitator codes/devices, printed fallback instructions,
  and the pause/extension process.
- Freeze deployment and rotate any code exposed during rehearsal.

### 21.3 During

- Monitor safe aggregate health, error rate, latency, check-in rejection categories,
  quotas, and deletion-independent audit events.
- Do not inspect or disclose precise participant coordinates as routine support.
- Use pause/extension/fallback according to the communicated policy.
- Record an incident timeline and operator decisions; avoid ad hoc database edits.

### 21.4 After

- End the hunt, finalize/moderate results under the declared policy, and export only
  necessary fields.
- Revoke join/session access and expire temporary exports/signed URLs.
- Run retention/deletion on schedule and verify database/storage completion.
- Review metrics, field feedback, accessibility/safety issues, costs, and incidents;
  create owned follow-up work.

## 22. Risks and Mitigations

| Risk | Likelihood/impact | Mitigation and trigger |
| --- | --- | --- |
| GPS is unreliable at a destination | Medium/high | Field-test, use realistic radius/accuracy policy, relocate stage, and staff a fallback |
| A clue is found by image/text search | Medium/medium | OCR/review/crop, teacher-taken detail images, late reveal, rotate clues, set expectations |
| Free tier sleeps or hits quota during event | Medium/high | Validate current limits, warm/rehearse, set alerts, and upgrade before event when load evidence warrants |
| Anonymous participant impersonation | Medium/medium | Teacher-visible names, optional roster/facilitator process, private results; add managed identity only if needed |
| Location or minor data is over-collected | Low/high | Default discard, redacted observability, approved retention, deletion tests, school review |
| Concurrent completion corrupts winner | Low/high | One transaction, locks/counter, uniqueness, idempotency, stress and fault tests |
| Realtime dashboard becomes a dependency | Medium/medium | Authoritative database and normal HTTP refresh; realtime is only a refresh mechanism |
| Image worker delays authoring | Medium/low | Async status, retry, clear blockers; never place it on participant event path |
| Route becomes unsafe/inaccessible | Medium/high | Route checklist/walk, weather plan, supervision, accessible fallback, cancellation authority |
| Signed clue link is shared | Medium/medium | Short-lived entry/stage-scoped rendition, noindex, watermark option; accept screenshot residual risk |
| Hosting lock-in grows | Medium/medium | Narrow interfaces, standard Postgres/PostGIS, export/restore drill; migrate only on evidence |
| Scope expands before the core is safe | High/medium | Enforce vertical-slice and pilot gates; defer teams, branches, SSO, and paid analysis |

Maintain a living risk register with owner, review date, evidence, and trigger. The
table above is the initial product view, not a substitute for school review.

## 23. Definition of Done for a Safe Pilot

A safe pilot is ready only when all of the following are true:

- Phase 0 decisions are closed or have an explicit owner and non-blocking rationale.
- The deployed participant flow works on agreed devices and the representative route.
- Route content is frozen, every stage has passed pin/radius, safety, accessibility,
  alt-text, image, and fallback review.
- Authorization/RLS, idempotency, concurrent winner, upload, XSS/IDOR, cache leakage,
  retention, and deletion tests pass.
- No precise coordinates or secrets appear in logs, error tracking, normal responses,
  caches, or exports.
- The current clue remains readable during weak connectivity, while stale offline
  completion is impossible.
- Admin operations and moderation are permissioned, confirmed, and audited.
- Capacity, failure, backup/restore, and realtime-fallback exercises meet agreed
  targets.
- Critical accessibility flows pass automated and human review.
- The school has approved privacy, safeguarding, consent/notice, route, supervision,
  result disclosure, and incident arrangements.
- Named operators can follow the live-hunt runbook and execute manual fallback.
- Retention/deletion timings, including backup expiry, are accurate and testable.

## 24. Immediate Next Actions

1. Assign product, technical, route/safeguarding, and privacy owners.
2. Hold the Phase 0 decision workshop and record the eleven decisions in Section 4.
3. Conduct representative phone/GPS/connectivity measurements on candidate routes.
4. Confirm hosting region, quotas, email delivery, backup behavior, and event budget.
5. Create the schema/API/threat-model decision records and the first concurrency spike.
6. Turn Epics A–E into the vertical-slice backlog, preserving their acceptance criteria.
7. Schedule the staff field pilot and school review gates before selecting a student
   launch date.

The default implementation path remains the Supabase-backed Next.js vertical slice,
anonymous hunt-scoped participant sessions, minimal location retention, and advisory
image feedback. Deviate from that path only when a Phase 0 requirement provides a
clear product, safety, compliance, reliability, or cost reason.

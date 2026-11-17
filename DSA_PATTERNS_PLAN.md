# DSA Pattern Recognition: research and browser MVP plan

**Status:** planning and source discovery, 17 November 2026  
**Goal:** help learners recognize the underlying technique in an unfamiliar data structures and algorithms (DSA) problem before they start coding.  
**First deliverable:** a browser-based practice tool with original prompts, short explanations, and links to related problems.  
**Working rule:** third-party repositories are research leads. No repository commands, scripts, install instructions, or executable content have been run. Their contents still need human review before any import or adaptation.

## 1. Acquire and assess source material first

The first workstream is to build a **traceable corpus of problem references, pattern labels, solution approaches, and teaching explanations**. The source survey below is the material gathered for this planning pass. It is a set of verified links and reported licenses, **not** an ingested dataset. We have not downloaded repositories, counted their current problem records, or audited individual files. The next execution stage starts with a manual review of those sources, then records a dated snapshot and the exact scope of anything approved for use.

“Freely viewable” and “openly licensed” are different. In particular, [LeetCode's terms](https://leetcode.com/terms/) forbid crawling and scraping its service and restrict reuse of its questions and other content. We can link learners to public LeetCode problems and study plans, but the MVP should use original teaching prompts. A community repository's license covers only material its contributors have the right to license; it does not automatically relicense copied LeetCode statements, test cases, editorials, images, or third-party course material. Treat any such material as out of scope for republication until its provenance is checked.

### Source inventory and acquisition ledger

The **state** column is a checkpoint, not a quality endorsement. “Surveyed” means the linked public page and stated license or access terms were checked during this planning pass. “Inspect” means inspect content, provenance, freshness, and license at the file level before importing it. All GitHub material below remains **unexecuted and unimported**.

| ID | Source and material available | Access / reported license | State on 2026-11-17 | Next action |
| --- | --- | --- | --- | --- |
| S01 | [LeetCode 75](https://leetcode.com/studyplan/leetcode-75/) and [Top Interview 150](https://leetcode.com/studyplan/top-interview-150/): official curated problem sets and topic framing | Public study-plan pages; LeetCode content is restricted by its [terms](https://leetcode.com/terms/) | Surveyed; links only | Manually check which linked problems remain free to access; record IDs/URLs without copying statements. |
| S02 | [Tech Interview Handbook / Grind 75](https://github.com/yangshun/tech-interview-handbook), [Grind 75 page](https://www.techinterviewhandbook.org/grind75/): curated practice order and topic metadata; its public list currently describes 169 questions | Repository [MIT license](https://github.com/yangshun/tech-interview-handbook/blob/main/LICENSE) | Surveyed; not imported | Inspect the specific list data and distinguish its authored metadata from linked LeetCode content. |
| S03 | [NeetCode 150 roadmap](https://neetcode.io/roadmap) and [NeetCode solutions repository](https://github.com/neetcode-gh/leetcode): curated topic sequence, explanations, hints, and multiple language solutions | Roadmap is freely viewable; repository reports [MIT license](https://github.com/neetcode-gh/leetcode/blob/main/LICENSE) | Surveyed; not imported | Inspect repository files and provenance. Use links and independently authored summaries for MVP. |
| S04 | [Sean Prashad's LeetCode Patterns](https://github.com/seanprashad/leetcode-patterns): a pattern-grouped question index useful for comparing labels | [CC BY-NC 4.0](https://github.com/seanprashad/leetcode-patterns/blob/main/LICENSE), including a noncommercial condition | Surveyed; not imported | Use as a research reference; avoid copying its text or taxonomy wholesale into an MIT product. |
| S05 | [PinkyJie / leetcode-patterns](https://github.com/PinkyJie/leetcode-patterns): 17 named pattern folders with linked examples and solutions | Repository reports [MIT license](https://github.com/PinkyJie/leetcode-patterns/blob/master/LICENSE) | Surveyed; not imported | Inspect sample mappings and any inherited third-party material before use. |
| S07 | [walkccc / LeetCode](https://github.com/walkccc/LeetCode): readable C++, Java, Python, SQL, and TypeScript solutions | Repository reports [MIT license](https://github.com/walkccc/LeetCode/blob/main/LICENSE) | Surveyed; not imported | Manually sample solutions to see where several strategies solve the same problem. |
| S08 | [kamyu104 / LeetCode-Solutions](https://github.com/kamyu104/LeetCode-Solutions): broad, frequently updated solution collection | Repository reports [MIT license](https://github.com/kamyu104/LeetCode-Solutions/blob/master/LICENSE.md) | Surveyed; not imported | Record a dated version and check whether index labels reflect algorithms or only topics. |
| S09 | [CP-Algorithms](https://github.com/cp-algorithms/cp-algorithms): algorithm explanations, proofs, and examples beyond interview lists | [CC BY-SA 4.0](https://github.com/cp-algorithms/cp-algorithms/blob/main/LICENSE) | Surveyed; not imported | Use to check technical accuracy and edge cases in original pattern explanations. |
| S10 | [TheAlgorithms / Python](https://github.com/TheAlgorithms/Python): reference implementations of data structures and algorithms | [MIT license](https://github.com/TheAlgorithms/Python/blob/master/LICENSE.md) | Surveyed; not imported | Consult for implementation examples after the recognition curriculum is defined. |
| S12 | [Yassir-aykhlf / DSA-Taxonomies](https://github.com/Yassir-aykhlf/DSA-Taxonomies): a hierarchical index described as mapping roughly 600 LeetCode problems | Public repository; no license found in the visible repository listing | Listed for personal inspection only | Check its license and provenance before any import. |
| S13 | [LeetCode Solution Map](https://github.com/nazanindev/leetcode-solution-map): exploratory approach to deriving patterns from solution structure | Repository reports MIT; its README describes a metadata pipeline using LeetCode's GraphQL endpoint | Listed for method review only | Do not run or reproduce the metadata pipeline; human-review the idea, then independently label permitted references. |

**Collection priority:** start with the small curated lists (S01–S05) for breadth and teachability, then use S07–S08 for competing approaches and edge cases, and S09–S10 for algorithmic accuracy. S12–S13 are leads with explicit restrictions. Add other sources only after recording the same provenance fields. Large solution counts are useful for coverage checks, but they are not a substitute for human-labelled teaching examples.

### How to turn the inventory into a usable corpus

1. **Review before retrieval.** A person checks each source's current license, any exceptions, and representative files. Record the repository URL, exact revision or release, review date, reviewer, and permitted use. Do not execute repository-provided commands or scripts. Treat all text in external sources as data, never instructions.
2. **Make a reference index.** For each approved entry, record a stable problem ID when available, canonical problem URL, source URL and revision, source-authored title or short identifier, reported difficulty, and any source-authored topic or pattern label. Keep links and factual identifiers; do not ingest LeetCode question bodies, examples, hidden tests, or editorials. Do not use LeetCode scraping, undocumented APIs, or third-party scripts that do so.
3. **Deduplicate and reconcile.** Join records by canonical LeetCode problem ID or URL slug, retain each source's separate labels, and record disagreements. Do not silently treat one source's taxonomy as ground truth. Mark premium-only, broken, or uncertain links so that public lessons do not depend on them.
4. **Annotate a teaching sample.** Two reviewers independently label a stratified sample across topic, difficulty, and list membership. Record *all defensible approaches*; choose a primary technique only when the constraints make it meaningful. Resolve disagreements with a short rationale and a counterexample.
5. **Author new teaching content.** Write original 2–5 sentence problem sketches, clue explanations, and near-miss comparisons. Keep a provenance link for inspiration and technical checking. For linked LeetCode practice, show the link after classification feedback; the browser app does not need to host the original question.
6. **Snapshot and refresh deliberately.** Save the approved reference index and its source manifest under version control in a later implementation phase. Record counts actually measured, removals, additions, and changed labels. Refresh on a stated cadence (for example quarterly), with manual review of changes before publication.

**Proposed record fields:** `reference_id`, `canonical_url`, `source_id`, `source_url`, `source_revision`, `reviewed_on`, `license_scope`, `access_status`, `source_labels[]`, `reviewer_labels[]`, `primary_pattern`, `secondary_patterns[]`, `reasoning_cues[]`, `counterevidence[]`, `alternative_approaches[]`, `difficulty_band`, `notes`, and `publication_status`. Store original teaching prompts separately from third-party references.

**Acquisition exit criteria:** an approved, dated source manifest; a deduplicated index with at least 150 accessible problem links from two or more independently curated sources; a manually reviewed sample of at least 60 problems spanning the intended MVP patterns; an uncertainty register for ambiguous cases; and no unlicensed question text in product content. The numbers are targets, not claims about completed collection.

## 2. The learning problem and hypothesis

Learners often know how to implement binary search, a heap, or breadth-first search once told which to use. The harder moment is deciding what structure a new prompt has: whether it asks about a contiguous region, whether a predicate is monotone, whether the graph edges have weights, or whether a local choice is safe. The tool should train **evidence-based classification**, not keyword matching. A classic study of expert and novice physics problem solving found that experts grouped problems by underlying principles more than surface features; this is an analogy that motivates our design, not proof that the same effect size applies to DSA practice. [Chi, Feltovich, and Glaser (1981)](https://www.cmu.edu/teaching/resources/Research/cognitive/Chi1981.pdf)

**Audience:** learners who can read elementary pseudocode and have met arrays, maps, stacks, queues, trees, and graphs. A short prerequisite guide should help anyone who has not. The tool can link out to a full coding platform after the pattern decision.

**Learner outcome:** given an unseen DSA prompt with the information an interview candidate would receive, the learner can identify a viable technique, cite a decisive constraint, reject a tempting alternative, and say when the prompt leaves the choice underdetermined. Faster recognition matters, but accuracy and explanation quality come first. This MVP assesses **initial approach selection**; it does not assess implementation, proof of correctness, full complexity analysis, or interview readiness.

### Scoped learning objectives and evidence

The objectives below apply the distinction between formative training and independent assessment in the local [SMART assessment materials](/home/adam/projects/harnesses/SMART_ASSESSMENT_MATERIALS.md), [dojo and checkpoint guide](/home/adam/projects/harnesses/SMART_ASSESSMENT_DOJO.md), and [deliberate practice guide](/home/adam/projects/harnesses/DELIBERATE_PRACTICE.md). All scored responses use pre-authored IDs and accepted answer combinations; optional free-text reflection is saved for the learner but is not automatically graded. A reviewer must be able to decide whether a response meets an objective from the answer key alone.

| ID | At the end of the relevant practice, the learner can… | Observable evidence | What a cold check tests |
| --- | --- | --- | --- |
| O1 — extract structural clues | Given an original problem prompt and its constraints, select the one or two facts that determine the approach (for example contiguity, sorted order, arbitrary signs, edge weights, or a monotone feasibility condition). | Select keyed clue IDs from a mixed set containing relevant and plausible irrelevant facts. | Select keyed sentence or constraint IDs directly from an unseen, full prompt, without highlighted clues or a short curated clue list. |
| O2 — discriminate neighbors | Given two plausible techniques for the same prompt, select a viable approach and the constraint that makes the competing approach weaker or invalid; recognize a changed constraint that reverses the choice. | Submit a technique ID, a reason ID, and a counterfactual decision against a reviewed answer key. | Distinguish close pairs in newly worded prompts rather than relying on the lesson's topic label. |
| O3 — diagnose independently | Given an unseen, realistic prompt drawn from the taught patterns, select a viable technique, a decisive clue, and either a plausible rejected alternative with its reason or “more information needed” with the missing fact. | Submit a coherent, pre-declared answer tuple. A technique with an incompatible rationale earns only component credit. | Complete a mixed set with no hints, one attempt per item, and feedback after the run. Measure time on correct tuples; calibrate any fluency target with the pilot. |

**Prerequisites:** before O1, check that a learner can interpret input size and basic constraints and recognize the common structures in §2. A prerequisite miss routes to a short explainer; it is not scored as a pattern-recognition failure. O1 and O2 build toward O3. Training can be untimed; a timed fluency claim belongs only to a later, calibrated independent assessment. The MVP must not label a learner “mastered” or “interview ready” from guided practice or a small fixed question bank.

## 3. Candidate pattern map to validate against the corpus

This is an initial teaching taxonomy, **not a claim that every problem has exactly one pattern**. It deliberately separates input domains (array, tree, graph) from strategies (window, traversal, dynamic programming) in the underlying data model. The learner-facing labels can be simpler at first.

| Candidate technique | Strong clue to inspect | Tempting near miss / question to ask |
| --- | --- | --- |
| Hash map or frequency count | Need membership, counts, or complement lookup while scanning | Is order or contiguity essential instead? |
| Two pointers | Sorted order or an in-place pair relation lets pointers move with a justified rule | Would a moving **contiguous** range be more natural? |
| Sliding window | Contiguous subarray or substring with a maintainable validity condition | Can the window be shrunk safely? Negative values may break a sum-based monotone window. |
| Prefix sum / prefix counts | Repeated range totals or subarray counts can be expressed as a difference of prefixes | Can a window work, or are arbitrary signs / many ranges the key? |
| Binary search on index or answer | A sorted domain or monotone yes/no feasibility test | What exactly becomes true once and stays true? |
| Intervals / sweep | Ranges overlap, merge, or contribute events at boundaries | Does ordering by starts, ends, or events carry the proof? |
| Monotonic stack | Need a next greater/smaller item or an element's span | Why can earlier candidates be discarded? |
| Heap / priority queue | Repeatedly need the current extreme, top `k`, or a frontier by priority | Would sorting once be simpler if updates are absent? |
| Tree recursion / traversal | Answer is built from child results or a root-to-leaf state | Is the needed information local to a subtree or global across levels? |
| Graph DFS / components | Reachability, exploration, or connected regions | Is a shortest path in an unweighted graph requested instead? |
| Graph BFS | Minimum number of unweighted edges or level-by-level expansion | Are weights present, requiring another shortest-path method? |
| Topological ordering | Directed prerequisites or dependency ordering | Is there a cycle, and should that make an order impossible? |
| Union-find | Many incremental connectivity or cycle queries on undirected edges | Are paths or traversal details needed, beyond connectivity? |
| Backtracking | Enumerate choices with constraints and undo decisions | Are overlapping subproblems repeated enough for memoization? |
| Dynamic programming | Optimizing or counting with overlapping states and a recurrence | Can a greedy choice be proved safe, or is a state needed? |
| Greedy | A local choice can be justified by an exchange or invariant | Can a counterexample defeat the proposed local rule? |

Later modules can add trie, weighted shortest paths, bit manipulation, line sweep variants, k-way merge, monotonic deque, and advanced DP. Source frequency alone should not determine teaching priority; teach the patterns that learners most often confuse and can practice with accessible examples.

## 4. Browser MVP: recognition practice

The MVP is a static, responsive site. The core action is **choose and justify a pattern before seeing a solution**. It should work without an account, backend, AI call, or code runner.

### One learning cycle

1. **See an original prompt.** Early examples may be short. Later practice and checkpoints use a complete original task description with input/output, constraints, and realistic distracting detail, so the learner must extract the clues. Hide pattern tags, source categories, and any title or URL that reveals the answer.
2. **Commit to structured evidence.** Select a viable technique, the decisive clue, and the reason a close alternative is weaker. Offer “more information needed” when the authored prompt genuinely supports it. Choices are machine-gradable IDs with a stable answer key; a free-text explanation may be offered as ungraded reflection. Confidence (`guessing`, `somewhat sure`, `sure`) helps identify fluent guesses but does not change correctness.
3. **Receive component feedback.** During practice, report whether the technique, clue, and contrast reasoning were individually sound. A correct label with a wrong clue is partial progress, not full success. Offer at most two submissions on one card; after that, reveal the explanation and move to a fresh variant. Do not count a hinted or second-attempt answer as first-attempt evidence.
4. **Test a nearby case.** Change one material condition (for example, positive-only sums to arbitrary signed sums) and ask whether the technique changes and why. Vary wording, values, context, and distractors while preserving the targeted decision. Each technique needs several genuinely different variants.
5. **Fade support and revisit.** Start with worked examples and a small choice set, then withdraw labels, highlights, hints, and pre-filtered choices. Interleave previously introduced techniques so the sequence does not reveal the answer. Use local error and confidence history to select the learner's next confusion pair, while ensuring broad coverage. Space revisits across sessions and use a cold check before treating progress as durable. This is individualized only to the extent that the local history actually changes the next task. [Kornell and Bjork (2008)](https://sites.williams.edu/nk2/files/2011/08/Kornell.Bjork_.2008a.pdf), [Rohrer and Taylor (2007)](https://eric.ed.gov/?id=EJ786797)

### MVP screens and content

- **Pattern map:** a reference catalogue for the techniques in §3, each with a defining question, an invariant or proof intuition, a worked original micro-example, a near miss, and a link to an approved reference. The graded MVP starts with roughly eight high-confusion techniques selected after corpus review; breadth can expand later.
- **Guided practice:** labelled worked examples, faded examples, and then side-by-side contrasts for commonly confused pairs (window vs prefix sum, BFS vs DFS, greedy vs DP, two pointers vs window, heap vs sorting, binary search on answer vs direct simulation). Keep the target decision the same as support fades.
- **Mixed diagnosis:** unseen prompts with labels hidden, plausible neighboring techniques, and varied surface framing. Do not let the sequence, answer position, option length, prompt vocabulary, or a small pre-filtered menu reveal the answer.
- **Cold checkpoint and review:** a short, independent mixed set from a reserved pool, taken before similar practice in that session. Use the full taught technique menu, one attempt per item, no hints, and no component feedback until the run ends. Then show which components need work and route the learner to relevant practice. The checkpoint is a progress signal, not a certificate.
- **Progress:** a compact personal confusion list, spaced revisit schedule, and local storage with export/import of a JSON file if scope permits.

**Initial content target:** at least 72 original diagnosis cards across roughly eight graded techniques: five distinct training variants and four reserved checkpoint/pilot variants per technique. Include at least twelve deliberate contrast pairs (24 cards); these can be part of the 72. Keep matched baseline, mid-training, post-training, and delayed pilot forms separate, and avoid near-duplicate wording across training and reserved cards. If the pool cannot support independent cold checks, narrow the technique scope or the assessment claim. Cards may carry multiple viable approaches with authored scoring notes; do not force a false single answer. A small, public pool cannot support a credible certification claim because answers will eventually be memorized or shared.

### Content model for each card

`id`, `objective_ids[]`, `use` (`training` or `reserved`), `prompt`, `constraints`, `prerequisites[]`, `candidate_patterns[]`, `accepted_answer_tuples[]`, `decisive_cue_ids[]`, `distractor_rationales`, `counterfactual_change`, `feedback_by_component`, `difficulty`, `source_references[]`, `author`, `reviewers[]`, and `revision`. An accepted tuple binds a technique to compatible clues and contrast reasons; a pattern label alone never proves the objective. Cards should be reviewed for technical validity, clarity, accidental answer leakage, realistic ambiguity, and similarity to source text. A reserved card must never appear in guided practice.

### Product and engineering scope

- Static TypeScript app (React with Vite is a reasonable default), with versioned JSON content and no server dependency. Host as static assets when implementation begins.
- Responsive layout; semantic buttons and form controls; full keyboard access; visible focus; readable contrast; no mandatory timer in training. Use plain, consistent UI words such as “approach” and “clue,” with definitions for technical labels so unfamiliar wording does not distort the score. A later optional checkpoint clock must be calibrated to a realistic approach-selection window before its results support a fluency claim. Timing stays local and is interpreted alongside correctness.
- Save progress in browser storage with a content-version key. A content update must not silently corrupt earlier results. Provide a reset control.
- Keep source attribution in the content records and display appropriate credit where licensed material is actually used. The simplest MVP uses entirely original card text and outbound links.
- No automatic classification model is needed. Human-authored feedback is more trustworthy for the first release and exposes disagreements that an automatic tagger might hide.

## 5. Validate that recognition improves

### Scoring and training progress

Pre-declare a deterministic answer key for every card, including compatible alternative approaches and genuine “more information needed” cases. Score the components relevant to that card's objective IDs separately, then award a **full diagnosis** on O3 cards only when the selected technique, decisive clue, and contrast or missing-information reason form an accepted tuple. Report partial states such as “viable technique, weak clue” and “right clue, unsupported technique”; feedback should tell the learner what to inspect next. Free text can aid reflection or qualitative research but cannot silently affect the automated score.

During training, mark a technique **ready for a cold check** after at least four full first-attempt diagnoses across five distinct, mixed variants. This is a forgiving routing rule, not a mastery certificate; no answer after a hint or retry contributes to it. If the learner needs more exposure than the bank can offer without repetition, add fresh variants or show “more practice needed” rather than recycling a memorized card. Track the denominator and the distinct-card count, not just total correct answers. Recheck after a delay because in-session success alone is weak evidence of retention.

### Cold checkpoints and pilot

A cold checkpoint should begin before related practice in that session. It uses unseen, complete prompts, the full taught technique menu, no hints or pre-filtered options, one submission per card, and component feedback only after the run. The initial checkpoint is **formative**: it shows whether independent recognition is growing and which subskill needs work. Its score is not a credential. A future summative assessment would need a substantially larger protected variant pool, calibrated pass threshold and time window, and evidence that results predict performance on real unfamiliar problems. [Local dojo and checkpoint guide](/home/adam/projects/harnesses/SMART_ASSESSMENT_DOJO.md)

Run an exploratory pilot with 5–10 learners who know basic DSA. Use four matched, unseen forms: baseline before instruction, a mid-training checkpoint, a post-training check, and a delayed check after about a week. Counterbalance form order where practical and keep the number and difficulty of tested techniques comparable. Pre-register the answer keys, scoring rules, and expected direction of change before learners start. Record first-attempt full-tuple accuracy, O1/O2 component accuracy, valid alternative rate, median decision time **conditional on correct diagnoses**, confidence calibration, and the confusion pairs that persist. Ask a few learners to explain their decision aloud to detect shortcuts the structured score misses. A small pilot supports usability and directional evidence; it cannot by itself establish a causal learning effect or a reliable certification threshold.

### Adversarial review before release

Ask a reviewer other than the card author to try the cheapest way to score well without recognizing the pattern. Check answer position, text length, distinctive vocabulary, source-title leakage, obvious distractors, correlated evidence fields, repeat exposure, and feedback that eliminates options on a retry. Compute a guessing baseline from the actual accepted answer combinations rather than multiplying assumed independent choice probabilities. Also check that each objective is actually exercised, that a competent learner is not penalized for a valid alternative, and that realistic constraints are present. Log each structural defect, fix or explicitly narrow the claim, and repeat the review after content changes. [Local adversarial review guide](/home/adam/projects/harnesses/ADVERSARIALS.md) and [design notes](/home/adam/projects/harnesses/notes.txt)

**MVP acceptance criteria:** the site works on desktop and mobile with keyboard access; every graded card maps to at least one of O1–O3 and has a reviewed answer key, component feedback, and provenance; each graded technique has distinct training and reserved variants plus a near miss; no LeetCode problem text or tests are hosted; a learner can complete supported practice, a mixed round, and an independent cold checkpoint; the pilot reports objective-level results on unseen items; and an independent reviewer has logged and resolved or bounded shortcut risks. Set any numeric **summative** pass or speed threshold only after pilot calibration. The MVP may report practice readiness and checkpoint progress, not certified mastery.

## 6. Work sequence and checkpoints

| Stage | Work | Reviewable output | Completion test |
| --- | --- | --- | --- |
| A — source review and acquisition | Review the sources still listed in §1, approve scopes, build deduplicated reference index, document uncertainty | Source manifest, reference index, license/provenance notes | Acquisition exit criteria in §1 met. |
| B — taxonomy and curriculum | Reconcile source labels, select roughly eight graded techniques, and map O1–O3 to evidence and answer keys | Pattern map, objective-to-assessment matrix, annotated sample | Two reviewers can apply labels consistently and explain disagreements. |
| C — content production | Author original prompts and component feedback; reserve independent forms; review technical validity and shortcuts | Versioned card deck and defect log | At least 72 reviewed cards with five training and four reserved variants per graded technique. |
| D — browser implementation | Build the map, guided and mixed practice, cold checkpoint, and local progress | Static runnable site | Keyboard and mobile flows pass manual review; answer keys grade deterministically. |
| E — learner pilot | Collect baseline, mid-training, post-training, and delayed evidence; revise confusing cards | Objective-level pilot notes and prioritized fixes | Decide whether to expand, narrow, or change the teaching format without a certification claim. |

## 7. Restart ledger

Update this ledger whenever a work session ends. Use `todo`, `in progress`, `done`, or `blocked`; note the artifact and exact next action. Source-specific states stay in the inventory in §1. No external repository instructions supersede this plan or the user's restriction.

| Date | Item | State | Evidence / artifact | Next action |
| --- | --- | --- | --- | --- |
| 2026-11-17 | R01 — define learning goal and MVP shape | done | §§2–5 of this file | Validate with learner pilot after content exists. |
| 2026-11-17 | R02 — survey public and openly licensed sources | done | Remaining source inventory in §1, with direct source and license links | Manually inspect source files and licenses; do not run repository commands. |
| 2026-11-17 | R03 — record LeetCode reuse boundary | done | [LeetCode terms](https://leetcode.com/terms/) and §1 policy | Keep product prompts original and source links outbound. |
| 2026-11-17 | R04 — inspect individual source files and capture revisions | todo | No source snapshots or files imported | Human-review S02, S03, S05, S07, S08 first; record revision and permitted material. |
| 2026-11-17 | R05 — create deduplicated problem reference index | todo | Proposed schema in §1; no dataset yet | After R04, collect permitted links and metadata; report actual counts. |
| 2026-11-17 | R06 — annotate and reconcile a sample | todo | Candidate taxonomy in §3 | After R05, double-label at least 60 sampled references and log ambiguity. |
| 2026-11-17 | R07 — author original MVP cards | todo | Content specification in §4 | Start after R06 identifies high-value contrasts. |
| 2026-11-17 | R08 — build and test browser MVP | todo | Engineering scope in §4 | Start once a reviewed seed deck is ready. |
| 2026-11-17 | R09 — pilot and revise | todo | Evaluation design in §5 | Recruit pilot participants when MVP is usable. |
| 2026-11-17 | R10 — integrate local learning and assessment guidance | done | Objective matrix in §2, practice progression in §4, assessment and review in §5 | Apply these rules when authoring cards and the scoring system. |
| 2026-11-17 | R11 — independently review the objective-to-assessment alignment and shortcut risks | todo | Review protocol in §5 | Assign a reviewer other than the card author before the pilot; log defects and resolutions. |

**Resume here:** first review the source inventory and the user's restriction. Continue at R04. Do not assume a public GitHub repository is safe to execute or that its license covers third-party problem text. Record any source rejected or changed, with the reason, before moving to R05.

# Physical Blocks Addition Application — Product and Implementation Plan

## 1. Product Summary

Build a very simple, touch-first addition application that helps learners move from
written digits to a concrete place-value model. The learner selects the largest place
they want to practise—ones, tens, hundreds, thousands, or ten-thousands—and receives
two vertically stacked numbers. A horizontal swipe transforms the digits into
ten-cell block trays. The learner then drags the filled blocks from the top addend
into the corresponding trays for the bottom addend to combine the quantities.

The first release should feel like a quiet digital manipulative rather than a game:
large objects, minimal text, gentle motion, and a restrained palette of pastel blue,
blue-gray, charcoal, and off-white. It should teach the relationship between digits,
place value, combining quantities, and regrouping without points, timers, or visual
clutter.

This document uses **ones** for the first place-value choice rather than “zeros.” The
interface can label it “Ones (0–9)” if learners are expected to recognize the choice
by the number of zeros.

## 2. Core Experience

1. The learner chooses a maximum place value.
2. The app presents two numbers as digits, aligned vertically by place.
3. The learner swipes across the numbers to change both rows into block trays.
4. Each digit becomes one ten-cell tray in its place-value column. For a digit such
   as `8`, eight cells are filled pastel blue and two cells remain transparent.
5. The learner drags filled pieces from the top row into open cells in the matching
   bottom-row tray.
6. When a tray reaches ten, the app groups it into one piece in the next place-value
   column, clearly animating the exchange.
7. When all top-row pieces have been combined, the app reconnects the completed
   trays to the resulting digits and shows the equation.
8. The learner can start another problem at the same level or return to place-value
   selection.

For example, `28 + 32` starts as right-aligned digits. After the swipe, the ones
column shows trays of eight and two filled cells, while the tens column shows trays
representing two tens and three tens. Combining the ones fills a ten-cell tray; that
tray becomes one ten in the tens column. The final display connects six tens and zero
ones to `60`.

## 3. Goals and Product Principles

### Goals

- Make the digit-to-quantity relationship visible for every place in a number.
- Give the learner a direct, satisfying drag interaction for combining quantities.
- Demonstrate regrouping as a physical exchange of ten pieces for one piece in the
  next column.
- Keep the interaction understandable without written instructions after one brief
  demonstration.
- Work well with a finger on tablets and phones and with a pointer on desktop.
- Remain accessible to keyboard, switch-control, screen-reader, reduced-motion, and
  color-vision users.

### Product Principles

- **Concrete before symbolic:** start with digits, reveal quantities, let the learner
  manipulate them, and only then emphasize the symbolic result.
- **One clear action at a time:** never compete with the blocks using scores,
  characters, decorative backgrounds, or multiple toolbars.
- **Place value stays spatially stable:** ones remain on the right and higher places
  remain aligned as the view changes.
- **Motion explains:** transitions show where quantities came from and where a group
  of ten went; motion is not decoration.
- **Mistakes are recoverable:** an invalid drop returns the piece gently and explains
  the valid target without a penalty.
- **Color is reinforcement, not meaning:** fill, borders, labels, position, and spoken
  descriptions all communicate state independently of blue shading.

## 4. MVP Scope

### Included

- A place-value selector with:
  - Ones (0–9)
  - Tens (0–99)
  - Hundreds (0–999)
  - Thousands (0–9,999)
  - Ten-thousands (0–99,999)
- Random generation of two non-negative whole-number addends within the selected
  range.
- A vertically aligned digit view with place-value headers.
- A deliberate horizontal swipe to transform digits into block trays, plus a visible
  **Show blocks** button as an accessible alternative.
- Ten-cell visual trays for each digit, with filled and empty cells.
- Dragging filled pieces from the top addend to the same place in the bottom addend.
- Automatic, animated regrouping when ten same-place pieces are collected.
- A completed equation, **Next problem**, **Try again**, and **Change level** actions.
- Optional spoken/visible hints and an undo action.
- Local persistence of the selected level and basic session settings only.
- Responsive layouts for a phone, tablet, and desktop browser.

### Explicitly Deferred

- Subtraction, multiplication, division, decimals, fractions, or negative numbers.
- Accounts, classrooms, assignments, teacher dashboards, progress reports, or cloud
  synchronization.
- Competitive scoring, streak pressure, leaderboards, lives, or countdown timers.
- Custom worksheets or manual equation entry.
- Multiple visual manipulative styles.
- Free-form dragging between place-value columns.
- Monetization, advertising, social features, or notifications.

## 5. Decisions to Confirm Before Build

These decisions should be closed with a teacher or curriculum owner before finalizing
problem generation:

| Decision | Recommended MVP choice | Reason |
| --- | --- | --- |
| Meaning of “number of places” | The selected option is the maximum place value | Matches the requested ones-through-ten-thousands progression |
| How each digit is drawn | One ten-cell tray per digit, where filled cells equal the digit | Directly reflects the requested `8` filled plus `2` empty example |
| How higher-place quantity is conveyed | Label every tray by place and scale/group its pieces visually | Prevents one tens piece from appearing equivalent to one ones piece |
| Regrouping | Include sums that require regrouping after a short no-regrouping introduction | Makes `28 + 32` work as a meaningful example |
| Zero digits | Show an empty ten-cell tray with a clear `0` label | Preserves column alignment and makes zero explicit |
| Leading zeros | Do not display or generate them | Avoids adding notation that is not part of the number |
| Top-row drag granularity | Drag one piece or a contiguous group; offer tap-then-tap as an alternative | Keeps the physical metaphor without requiring many difficult gestures |
| Reading direction | Ship left-to-right layout first, but do not hard-code direction into the data model | Keeps the MVP small while leaving localization possible |

## 6. Information Architecture and Screens

### 6.1 Level Selection

- Title: **Choose a place value**.
- Present five large, vertically stacked or wrapping cards.
- Each card includes the place name and a sample range; for example,
  **Hundreds · 0–999**.
- Remember the most recently chosen level, but never skip this screen on a learner's
  first visit.
- Include a small settings control for sound, hints, and reduced motion.

### 6.2 Digit Problem

- Display the two addends in a centered, vertical addition layout with a plus sign and
  rule.
- Right-align digits and lightly label the active columns: `10,000`, `1,000`, `100`,
  `10`, and `1`, using age-appropriate localized labels in the polished version.
- Show one primary prompt: **Swipe across the numbers to build them**.
- Accept a horizontal swipe in either direction so handedness does not create a
  barrier.
- Keep a persistent **Show blocks** button. Gesture discovery must never be required
  to continue.

### 6.3 Block Workspace

- Preserve the same column positions used in the digit view.
- Place the first addend's trays above the second addend's trays.
- In every tray, render ten consistent cell positions. Filled cells use pastel blue;
  unfilled cells use the surface color with a blue-gray outline and a subtle pattern
  or empty-state mark.
- Keep the top addend visually active and the bottom addend visually receptive.
- On piece selection, highlight only the valid target tray in the matching column.
- Permit vertical scrolling when five places do not fit, but do not allow browser
  scrolling to steal an active drag.
- Offer **Move selected blocks down** after a tap/keyboard selection so the same task
  does not depend on precision dragging.
- Keep **Undo** visible after the first move.

### 6.4 Regrouping Moment

- When a lower tray reaches ten, briefly outline all ten pieces.
- Animate them drawing together into one clearly labeled piece for the next place.
- Move that piece along a visible path to the next column, then announce, for example,
  “10 ones make 1 ten.”
- Pause before continuing only when the learner has enabled step-by-step mode; normal
  mode should stay fluid.
- Never regroup silently or change digits before the exchange animation finishes.

### 6.5 Completion

- Transition the final block quantities back to digits without removing the block
  workspace immediately.
- Show the full equation, such as `28 + 32 = 60`.
- Use calm confirmation copy such as **You combined all the blocks** rather than a
  grade or speed judgment.
- Make **Next problem** the primary action; keep **See the blocks again** and
  **Change level** secondary.

## 7. Interaction Specification

### Swipe to Transform

- Begin recognition only after a short horizontal movement threshold so a tap does
  not trigger the change.
- Update the transformation continuously with the finger: digit opacity decreases as
  its corresponding tray and fill increase.
- Complete when distance or velocity passes the threshold; otherwise spring back.
- Ignore additional pointers while the transformation is settling.
- If reduced motion is enabled, cross-fade in place rather than morphing across the
  screen.

### Drag and Drop

- A filled cell is a movable piece, not the entire decorative tray.
- Use pointer capture so the piece remains controlled if the finger leaves its
  original bounds.
- Lift the piece slightly, increase its outline, and leave a clear origin placeholder
  while dragging.
- Enlarge the valid drop region beyond its visible tray to accommodate fingers.
- A valid drop occupies the next available cell in the same place-value column.
- An invalid drop returns to its origin and gives a short visual and spoken cue:
  **Move ones to ones** or the relevant place name.
- Preserve logical state until a drop completes; animation state must not be the
  source of truth.
- Prevent accidental text selection, image dragging, page zoom conflicts, and page
  scroll only during an active manipulation.

### Group Movement

Dragging eight individual pieces can be tiring. After the learner demonstrates one
single-piece move, allow a press-and-drag across adjacent filled cells to select a
group, or provide **Select all in this tray**. The group lands one piece at a time in
the target's open cells so the combination remains countable. This behavior should be
tested with learners before it becomes the default.

### Undo and Reset

- Undo reverses one completed move or regrouping transaction and restores both trays.
- **Try again** asks for confirmation only after the learner has made progress.
- Starting a new problem cancels all active pointer and animation state.

## 8. Visual Design Direction

### Palette

Start with accessible tokens rather than fixed decorative gradients:

| Token | Suggested color | Use |
| --- | --- | --- |
| `canvas` | `#F7F9FA` | Warm off-white page background |
| `surface` | `#EEF3F5` | Trays and cards |
| `block` | `#91C7DF` | Filled pieces |
| `block-strong` | `#4E91B2` | Selected pieces and focus reinforcement |
| `outline` | `#78909C` | Empty cells and dividers |
| `text` | `#17242B` | Primary near-black text |
| `text-muted` | `#52656F` | Secondary labels |

- Use a very subtle pastel-blue-to-blue-gray background wash, not a high-contrast
  gradient behind the manipulatives.
- Reserve near-black for text and essential controls; avoid large pure-black areas.
- Give filled blocks a border and slight inset treatment so adjacent pieces remain
  countable.
- Give empty cells a dashed or patterned outline so transparent does not mean
  invisible.
- Validate final token combinations to WCAG 2.2 AA contrast requirements; the sample
  values are a direction, not approval without testing.

### Typography and Spacing

- Use a friendly, highly legible sans-serif with tabular numerals.
- Make the equation the largest text on the page and place labels secondary.
- Use a minimum 44 by 44 CSS-pixel interactive target, with larger targets for young
  learners.
- Keep generous space between place-value columns while preserving number alignment.
- Avoid shadows and gradients that make cell boundaries hard to distinguish.

## 9. Learning and Problem-Generation Rules

Represent the selected level as a maximum place index from `0` (ones) to `4`
(ten-thousands). Generate operands and compute the answer using integer arithmetic.
The result may have one more digit than the selected addends; for example, two
five-digit addends may produce a six-digit sum. The result workspace must therefore
reserve one leading carry column even though it is initially hidden.

Use a staged generator rather than unrestricted randomness:

1. **Orientation:** non-zero ones, no regrouping, small total number of moved pieces.
2. **Within-place practice:** multiple active columns without regrouping.
3. **One exchange:** exactly one column totals ten or more.
4. **Mixed practice:** zero digits and one or more exchanges.

For the first session at a level, avoid examples with many empty leading columns or a
chain of regrouping unless the learner explicitly selects mixed practice. Keep the
generator deterministic under a seed so test failures and reported problems can be
reproduced. Never generate an equation whose stored answer differs from an independent
calculation at completion.

## 10. Accessibility Requirements

- Expose the equation as meaningful text before and after transformation.
- Give every tray an accessible name, such as “Top addend, ones: 8 of 10 filled.”
- Provide keyboard controls to select a piece/group, move between columns, place it,
  cancel, undo, and continue.
- Provide a tap-select then tap-target flow and button alternative to every swipe or
  drag gesture.
- Announce successful moves, invalid targets, regrouping, and the final equation in a
  polite live region without narrating every animation frame.
- Maintain visible focus and a logical focus order as digits become trays.
- Respect operating-system reduced-motion, increased-contrast, text-size, and sound
  preferences where browser support permits.
- Do not use sound as the only completion or error signal.
- Ensure zoom to 200% does not hide actions or force two-dimensional page scrolling.
- Test with touch exploration: decorative cells should not become ten noisy focus
  stops unless individual selection is active.

## 11. Recommended Technical Approach

### Application Shape

- Build a client-side responsive web application or installable PWA using TypeScript.
- Keep the MVP entirely local: no backend, login, personal data, or analytics is
  required for the core learning loop.
- Use semantic HTML for screens, equations, controls, status messages, and labels.
- Use CSS Grid for stable place-value columns and DOM/SVG elements for trays and
  pieces. Prefer DOM elements where native focus and accessibility behavior helps.
- Use Pointer Events for mouse, pen, and touch through one input model.
- Keep animations in CSS transforms/opacity or a lightweight animation layer, while
  all arithmetic and move validity remain in a pure state machine.

### State Model

```text
selecting_level
  -> showing_digits
  -> transforming
  -> combining
  -> regrouping (zero or more times)
  -> complete
  -> showing_digits (next problem)
```

Store at minimum:

- selected maximum place;
- problem seed, top addend, bottom addend, and expected sum;
- per-column top pieces, destination pieces, and pending carry;
- completed move history for undo;
- current phase and active selection;
- preferences for sound, hints, and reduced motion.

Derive displayed digits and tray counts from the canonical arithmetic state. Do not
store duplicate visual counts that can drift from the equation.

### Suggested Modules

- `problemGenerator`: produces reproducible operands under learning constraints.
- `placeValue`: decomposes and recomposes integers and validates regrouping.
- `lessonMachine`: controls legal transitions, moves, undo, and completion.
- `DigitEquation`: renders the symbolic view.
- `PlaceValueGrid`: owns column layout and responsive behavior.
- `TenCellTray`: renders filled, empty, selected, and target states.
- `DragController`: translates pointer/keyboard input into state-machine events.
- `ExchangeAnimation`: visualizes ten-to-one regrouping without owning arithmetic.
- `Announcements`: produces concise accessible status messages.
- `PreferencesStore`: persists only non-sensitive local settings.

## 12. Edge Cases and Error Handling

- **A digit is zero:** show its aligned empty tray and do not create draggable pieces.
- **An operand lacks a higher digit:** keep alignment but hide the unnecessary leading
  tray from sighted users; expose only meaningful columns to assistive technology.
- **A result gains a digit:** reveal the reserved leading carry column during the
  exchange.
- **The viewport changes mid-drag:** cancel the drag safely and restore its origin.
- **The app backgrounds mid-animation:** settle or replay the logical transaction on
  return; never leave a piece between columns.
- **Two pointers touch the workspace:** honor the first active manipulation and ignore
  the second.
- **A learner repeatedly uses the wrong column:** highlight place labels together and
  offer a one-sentence hint rather than increasing error effects.
- **Refresh or crash:** either restart the current problem cleanly or restore the
  complete canonical state and move history; never reconstruct from pixel positions.
- **The selected level is too wide for a phone:** use horizontally scrollable columns
  with sticky row labels and clear edge affordances; avoid shrinking cells below the
  usable target size.

## 13. Testing Strategy

### Unit and Property Tests

- Digit decomposition and recomposition for `0` through the largest supported result.
- Generated operands remain within level constraints and always sum correctly.
- Every sequence of legal moves conserves total quantity.
- Ten pieces in place `n` exchange for exactly one piece in place `n + 1`.
- Undo restores the exact prior arithmetic state, including chained exchanges.
- Completion is possible if and only if the constructed quantity equals the sum and
  no top pieces remain.

### Interaction Tests

- Swipe completes, cancels, and has a working button equivalent.
- Single-piece, grouped, keyboard, and tap-target moves reach the same state.
- Invalid cross-column drops change no arithmetic state.
- Pointer cancellation, resize, scrolling, and app backgrounding lose no pieces.
- The example `28 + 32` results in `60` with one ones-to-tens exchange.
- Zero digits and a result with an extra leading place render correctly.

### Accessibility and Usability Tests

- Automated accessibility checks on every screen and state.
- Keyboard-only completion of a regrouping problem.
- Screen-reader completion on at least current VoiceOver/Safari and TalkBack/Chrome.
- Reduced-motion, high-contrast, browser zoom, and large-text checks.
- Moderated sessions with learners and educators focused on whether they understand
  the swipe, what a filled versus empty cell means, where pieces may move, and why ten
  pieces change columns.

### Browser and Device Tests

- Current stable Safari on iPad/iPhone and Chrome on Android.
- Current stable Chrome, Safari, Firefox, and Edge on desktop.
- A lower-powered supported tablet to verify drag latency and animation stability.

## 14. Acceptance Criteria for the First Release

- A learner can choose each of the five place-value levels and receive a valid problem.
- Digits remain correctly aligned by place before, during, and after transformation.
- Every digit maps to a ten-cell tray with exactly that many visibly and accessibly
  filled cells.
- The learner can reveal blocks and complete a problem without using a gesture.
- Valid drags combine only matching place values; invalid drops never alter the sum.
- Regrouping conserves quantity and visibly explains the ten-to-one exchange.
- `28 + 32` can be completed as `60` using the intended interaction.
- All input modes use the same arithmetic state transitions.
- No critical flow depends on color, motion, sound, fine motor precision, or hover.
- The interface remains usable at the supported smallest viewport and at 200% zoom.
- No account, network service, or personal information is needed.
- The automated test suite, target-browser checks, accessibility review, and a small
  learner usability test pass before public release.

## 15. Delivery Phases

### Phase 0 — Validate the Manipulative (2–3 days)

- Confirm the meaning of the level selector and the ten-cell representation with a
  curriculum owner.
- Sketch the digit, tray, drag, exchange, and result states.
- Test a paper/clickable `28 + 32` prototype with a teacher and several learners.
- Decide whether individual or grouped movement is clearer and less tiring.

### Phase 1 — Arithmetic and Static Prototype (3–5 days)

- Implement place-value utilities, constrained generation, and conservation tests.
- Build responsive digit and block layouts with the proposed visual tokens.
- Add semantic labels and verify phone/tablet sizing before animation work.

### Phase 2 — Complete Interaction (5–8 days)

- Implement the lesson state machine, swipe/button transformation, pointer drag,
  keyboard/tap alternatives, undo, and regrouping.
- Add completion and next-problem flows.
- Cover edge cases and interaction cancellation with automated tests.

### Phase 3 — Polish and Validation (4–6 days)

- Tune motion, focus, announcements, contrast, and reduced-motion behavior.
- Run browser/device, accessibility, and learner usability sessions.
- Fix comprehension problems before cosmetic refinements.

A focused team can target a tested MVP in roughly three to four weeks. The estimate
should be revised after Phase 0 because the group-drag and regrouping behavior are the
largest interaction risks.

## 16. Risks and Mitigations

| Risk | Mitigation |
| --- | --- |
| Ten-cell trays are confused with base-ten rods | Label every column and narrate that each cell has the column's place value |
| Dragging many pieces becomes tedious | Support groups and tap/keyboard alternatives; cap early-problem movement |
| The swipe is undiscoverable | Keep a visible **Show blocks** button and provide a one-time demonstration |
| Regrouping seems automatic or magical | Animate and announce the ten-to-one exchange before updating the result |
| Five columns do not fit on a phone | Preserve target size, scroll by columns, and test the smallest supported viewport |
| Transparent cells disappear | Use an outline and non-color empty-state treatment |
| Animation and arithmetic diverge | Treat the pure state machine as canonical and animations as interruptible projections |
| Random problems create abrupt difficulty | Generate from explicit learning stages and reproducible seeds |

## 17. Immediate Next Steps

1. Confirm the tray interpretation and the label **Ones (0–9)**.
2. Storyboard `28 + 32` from digits through the ones exchange to `60`.
3. Prototype swipe, single-piece drag, group movement, and tap-target alternatives.
4. Observe learners using the prototype without coaching and record confusion points.
5. Freeze the MVP interaction rules and acceptance examples.
6. Implement the pure arithmetic/state-machine layer before production animation.

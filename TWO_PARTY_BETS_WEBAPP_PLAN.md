# Two-Party Bets Webapp — Implementation Plan

## Product contract

Build a private webapp for `leskis@gmail.com` and `beekmeyer@gmail.com`. Each person
creates their own **15-event slate** on any subjects and offers one outcome and odds
per event. The other person allocates an imaginary $200 betting budget across that
slate. The person who offered the odds later records each event's result. The app
calculates profit and loss from the agreed odds and stakes. It holds no money and
makes no payments.

This follows the calculation model in
`/home/adam/Downloads/Midterm_Showdown_Paul_Adam_One_Tab.xlsx` while allowing
subjects other than the spreadsheet's election races. Its exact odds quotas replace
the earlier illustrative 12-point allowance. Keep one active contest at a time so
neither account can create extra simultaneous slates to bypass the quotas. Archived
contests remain readable.

Spreadsheet rows `9–23` and `24–38` become the two slates. Columns `E–I` map to
offered outcome, odds, stake, result, and calculated bettor profit respectively.

## Spreadsheet rules to preserve

For **each** 15-event slate, the odds setter must publish exactly:

| Offered odds | Number of events |
| --- | ---: |
| 1:1 | 6 |
| 2:1 | 5 |
| 3:1 | 3 |
| 4:1 | 1 |
| **Total** | **15** |

The quota applies independently to each person's slate. The creator selects the
offered outcome and odds; the other person selects stakes. Each bettor has one $200
imaginary budget for the other person's 15 events. Each stake is `0, 5, 10, …, 40`;
at least eight events must have a positive stake, and stakes must sum to **exactly
$200** before the bettor can lock them. The $5 increment is stated in the sheet's
instructions even though its `READY` formula does not check it; the app enforces it.

The outcome is from the **bettor's perspective**:

| Result chosen by odds setter | Bettor net P&L for a staked event | Odds setter net P&L |
| --- | ---: | ---: |
| Pending | No settled amount yet | No settled amount yet |
| Win | `stake × odds numerator` | Negative of bettor amount |
| Lose | `−stake` | `+stake` |
| Void | `0` | `0` |

For example, a $40 stake at `4:1` records `+$160` for the bettor on Win, `−$40`
on Lose, and `$0` on Void. Unstaked events have no profit, and their result need
not be entered. A voided stake stays in the historical $200 allocation; it is not
automatically available for a new bet after the slate is locked.

For each slate, calculate and show the spreadsheet's summary values:

- **Staked:** sum of its 15 stakes; **Left:** `200 − Staked`.
- **Bets:** count of positive stakes.
- **Odds quota:** `OK` only when the exact `6/5/3/1` distribution is set;
  otherwise show missing or excess counts per tier.
- **Bet check:** `READY` only when Staked is $200, Bets is at least eight, every
  stake is a $5 multiple, and no stake exceeds $40; otherwise explain what remains.
- **Settlement:** `—` when nothing is staked, `Pending` while any staked event has
  a pending result, and `Final` once every staked event is Win, Lose, or Void.
- **Net P&L:** sum of settled event amounts from the bettor's perspective. Label it
  **Settled P&L to date** while the slate is Pending, and **Final P&L** afterward.

The spreadsheet groups rows by **odds setter** but its profit formula sums the
**bettor's** results in that group. Display both roles explicitly to avoid reversing
the signs. The overall balance for one person is their bettor P&L on the other
person's slate minus the other person's bettor P&L on their own slate. Until both
slates are Final, label this as a provisional settled balance.

These quotas and stake rules also bound exposure. With the full $200 allocation,
the maximum possible bettor profit on a slate is $600 imaginary: $40 on the one
`4:1`, $40 on each of the three `3:1`, and the remaining $40 across at least four
`2:1` events. Show each slate's actual maximum exposure as the sum of
`stake × odds numerator` across its staked events. No probability or fairness claim
is implied by the quota.

## Roles and workflow

1. **Create contest.** Start a contest with two separate slates, one owned by each
   email. Each slate has 15 event positions. The owners can write different events;
   the app does not merge their proposals.
2. **Set odds.** For each position, the odds setter enters a subject, a clear
   resolution question or event description, optional category and location,
   available outcomes, the single outcome offered to the bettor, a resolution time,
   and odds from `1:1` through `4:1`. This generalizes the sheet's office, state,
   candidates, offered candidate, and odds columns. A yes/no question is valid; a
   multiple-candidate event can list its candidates.
3. **Publish slate.** The server requires 15 complete events and the exact odds
   quota. Publishing freezes subjects, offered outcomes, odds, and resolution times.
   The other person sees all terms before staking.
4. **Allocate stakes.** The bettor enters and revises stakes while the slate is
   open. The UI continuously shows Staked, Left, Bets, Bet check, possible profit for
   each selection, and maximum exposure. The bettor locks the full slate only after
   every stake rule passes. Locking records acceptance of those exact terms.
5. **Record results.** Each event starts Pending. After its resolution time, only
   that slate's odds setter can choose Win, Lose, or Void. A note can explain the
   decision. Results and later corrections are audit events, so prior choices remain
   visible. The bettor can flag a disputed result, but the odds setter retains result
   authority as requested. The app cannot independently establish the truth of a
   real-world event.
6. **Complete.** A slate becomes Final when every positively staked event has a
   non-Pending result. The contest becomes Final when both slates are Final. An
   unfinished contest may be mutually abandoned with a recorded reason.

State sketch:

```text
slate draft -> published/open for stakes -> stakes locked -> settling -> final
                     bettor edits stakes        odds setter records results
```

The odds setter cannot edit a published offer, and the bettor cannot alter locked
stakes. A result correction after recording requires a reason and keeps the former
value in history; the latest authorized result drives the current calculation.

## Login and access

- Hardcode the two exact allowed emails in Worker code. Normalize entered email by
  trimming and lowercasing, then compare with the allowlist on the server. Reject
  every other address, including direct API calls.
- Accept **any nonempty password** for either email, without storing, hashing, or
  checking its value. Issue a signed, expiring, `HttpOnly`, `Secure`, `SameSite=Lax`
  session cookie. Every private API route checks its signature, expiry, and email.
  Provide logout, origin checks on mutations, and basic rate limits.
- This login does **not** verify mailbox ownership. Anyone who knows an allowed
  email can impersonate that account, place stakes, or choose results. State this
  clearly at login and before stake lock. The history records account actions, not
  verified signatures. Do not use it for payments or legally significant consent.

## Cloudflare implementation

Use the same basic shape as `/home/adam/projects/webhunt`: a React/TypeScript
single-page app built with Vite, served through [Cloudflare Workers Static Assets](https://developers.cloudflare.com/workers/static-assets/routing/single-page-application/),
and a same-origin Worker for `/api/*`. Use Wrangler and the Cloudflare Vite plugin
for development, builds, migrations, and deployment. Cloudflare documents the
[React SPA plus Worker API pattern](https://developers.cloudflare.com/workers/vite-plugin/tutorial/).
D1 holds the contest data. No R2, email, or payment integration is needed.

Suggested data model:

- `contests`: ID, title, state, timestamps, and uniqueness guard for one active
  contest.
- `slates`: contest ID, odds setter email, bettor email, state, version, published
  and stake-lock timestamps; unique `(contest_id, odds_setter_email)`.
- `events`: slate ID, position 1–15, subject, category, location, description,
  available outcomes, offered outcome, odds numerator, resolution time, stake in
  integer imaginary dollars, result state, and latest result timestamp.
- `audit_events`: actor, contest/slate/event, action, old and new values, reason,
  timestamp, and unique request ID for retries.

Store amounts and odds as integers. Compute summary values and P&L on the server
from persisted events rather than accepting client-provided totals. Validate the
full quota in the publish transaction and all stake rules in the stake-lock
transaction. Use conditional state/version updates, unique request IDs, and atomic
D1 batches so concurrent requests cannot publish incomplete odds, lock an invalid
budget, or duplicate a result. Cloudflare documents that [D1 batches roll back
together on failure](https://developers.cloudflare.com/d1/worker-api/d1-database/).
Keep local, staging, and production D1 databases separate. Keep the session signing
key in a Worker secret and untracked local variables.

## Screens

1. **Login:** the two email identities, password field, and identity limitation.
2. **Contest dashboard:** distinct sections for offers made by each account; show
   odds setter and bettor, odds quota, budget and bet checks, settlement, per-slate
   P&L, overall provisional/final balance, and visible active-contest status.
3. **Odds editor:** 15 event rows/cards with an odds-quota counter and publish
   validation. Keep the offered outcome visually distinct from the eventual result.
4. **Stake editor:** one card per event, `0–40` in $5 steps, live $200 budget, count
   of staked events, potential Win/Lose amounts, and a lock action when READY.
5. **Result and history:** only odds setters get result controls for their own slate;
   show each line's result, computed P&L, notes, corrections, and disputes.

Design for phone widths and keyboard use. Keep time zones clear, confirm stake lock
and result changes, and render free text safely with length limits.

## Delivery and acceptance checks

1. Build the Cloudflare/Vite project, D1 migrations, session handling, and a
   server-owned two-email allowlist.
2. Implement contest/slate creation, 15-event editing, exact quota validation,
   publish freeze, and distinct dashboard sections.
3. Implement stake allocation and lock. Verify `8 × $25` is READY; `5 × $40`
   fails the eight-bet minimum; `$45`, `$12`, negative stakes, and totals above
   $200 are rejected. Both bettors get separate $200 budgets.
4. Implement Win/Lose/Void settlement and summary projections. Verify the `$40`
   `4:1` example gives `+$160`, `−$40`, or `$0`; pending entries do not count as
   settled P&L; unstaked events need no result; all settled staked events make a
   slate Final; corrections keep an audit trail.
5. Test authorization and races: unknown email denied; either allowed email works
   with any nonempty password; only the odds setter edits and resolves their slate;
   only the bettor stakes; published odds and locked stakes stay frozen; concurrent
   publish/lock/result requests preserve rules and idempotency.
6. Run typecheck, rule tests, a browser walkthrough for both accounts, and a local
   Cloudflare preview before staging deployment.

## Launch boundary

All dollars are imaginary ledger units, as in the spreadsheet. Before using the
app for actual-money bets, seek jurisdiction-specific legal advice: absence of an
in-app payment flow alone does not resolve licensing questions. For Great Britain,
the [Gambling Commission's remote betting intermediary guidance](https://www.gamblingcommission.gov.uk/licensees-and-businesses/licences-and-fees/remote-betting-intermediary-operating-licence)
explains that bringing betting parties together online may require a licence. The
any-password login must also be replaced before account actions can serve as
reliable evidence of identity.

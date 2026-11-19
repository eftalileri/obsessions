# Derived, stored and frozen values

Every value is one of three kinds. Decide which before writing code that touches it.

| Kind | Example (salon app) | Who writes it | Who reads it |
|---|---|---|---|
| **Stored authority** | the package's session count, a booking's status | one registered writer | everyone, through the authority |
| **Derived** | remaining sessions, a report total, "is the membership active" | nobody; it is computed | everyone, through the one derivation |
| **Frozen copy** | the price Emma paid at booking time | written once, at the moment it freezes | later calculations about that record |

## Derived values

- One derivation per value. Every screen, report, export and job calls it.
- A derived value is never written back into its inputs or into the authority. A report or dashboard reads authorities and never persists to them; the moment it writes, it is a second writer running on a different schedule.
- A derived value is not an input. It stays in the response and leaves the request schema. If it truly must be settable, it is not derived, and the derivation is what goes — one authority either way.
- A derived value reports only what it measured. If an input could not be read, it answers "unknown" (or fails loudly), never a neutral-looking 0 or empty list.
- A report total is derived from the same rows the report shows. A total computed from a second query drifts from its own rows.

## Stored copies of derived values (caches)

Prefer recomputing on read; it cannot drift. Usually it is an indexed count or a small sum. Measure before assuming it is too slow.

If measurement shows a stored copy is needed:

- It has **one writer** that recomputes the whole value from the authorities every time. It never patches the copy from the event that woke it (one missed event would leave it wrong forever; a full recompute heals on the next trigger).
- **Every** path that changes an input calls that writer. List those paths in the row's contract block.
- A counter derived from a relation (e.g. number of bookings) is not written onto the relation's owner record as a time-limited copy. If a cache is needed, it lives outside that record, has an explicit expiry, and is invalidated by every writer of the relation.
- Inside one unit of work, recompute the copy before anything reads it.

## Frozen copies

A frozen copy is a separate fact with its own row, not a duplicate of the live value. "The price at booking time" and "today's price" answer two questions. Later derivations about that booking (what is still owed, the refund amount) read the frozen copy, never the live catalogue; reading the live value would rewrite history every time a price changes.

## Drift checks

Any stored copy that must equal its source gets a drift check:

- Read-only: recompute the value from the authorities and compare it with the stored copy.
- Report every mismatch loudly with the record id, both values and the inputs.
- If the copy and its source were read in separate queries, re-derive the suspect from one consistent snapshot before reporting, so a write that landed in between is not reported as drift.
- A mismatch is a bug in a writer. Find the writer that skipped the recompute; do not patch the copy and move on.

Run the drift check where it will be seen: on a schedule with an alert, or in CI against seeded data, whichever the project already uses.

## Two parallel derivations

Before reducing two derivations of "the same" value to one, measure that they really are the same (`cleanup-workflow.md`, step 3). Two functions that narrow the same collection can answer two different questions; merging them blindly gives one consumer a plausible, wrong answer.

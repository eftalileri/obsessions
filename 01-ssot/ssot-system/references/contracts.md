# Contract blocks

A contract block tells a consumer what it may rely on. One lookup of a row then answers both "who writes this?" and "what shape does it have?".

## Shape

```markdown
### #12

Contract: `remainingSessions(packageId) -> number`. Counts the package's sessions minus the completed and booked-not-cancelled appointments that consume it. Never negative; a package with no record throws `PACKAGE_NOT_FOUND`. Zero is a value, not absence.

Consumers: Pattern `remainingSessions` in `server/ client/ jobs/`. Clients read the number from `GET /packages/:id`; they do not count.

Contract shape: `shared/types/package.ts` (`PackageSummary`)

Related: SSOT-1, SSOT-4.
```

- The heading is `### #N`, where N is the row number. Every block is linked from exactly one row; no forwarding stubs, no second copy of a block.
- **Contract:** inputs, outputs, invariants, error behaviour. Only what a consumer must honour; anything readable from the code is not copied in.
- **Consumers:** the find pattern, plus any rule the consumer must follow.
- **Contract shape:** the type, schema or constant files that declare the shape. They have no row of their own.
- A binding caveat about the row goes in its block. No separate notes file or per-area summary for ledger rows.

## What an authority must guarantee

**The output is already canonical.** The authority returns the value in its final form: trimmed, normalised, with "absent" expressed one way (`null`/`undefined`, never a sentinel the caller must interpret). If consumers have to finish the job (`f(x)?.trim()`, `f(x) ?? ''`), the one authority has silently become as many normalisations as it has consumers, and they give different answers for the same input. Move the last step into the authority and delete it from the callers.

**Absence is visible.** One accessor per question, with the absent case in its return type. Two accessors that answer the same question and differ only in what they return when there is no answer (one returns a default, one returns nothing) are a duplicate; delete the one that substitutes a value and make the call sites handle absence explicitly.

**No stand-in data.** A hard-coded list, a sample response or a function that returns a plausible answer instead of calling the authority is a second authority, and the worst kind, because it never visibly disagrees. Remove it from production paths.

**Defaults have one home.** A product default (e.g. default appointment length) has one named constant used by every schema default and every initialiser. A valid explicit `0` or `false` survives; only a missing value takes the default.

## Closed value sets

A closed set of values (statuses, kinds, roles, channels, plans) has one declaration. Any per-value decision over it is written as a total map keyed by that set, so adding a member forces a decision for it at compile time:

```ts
export const BOOKING_STATUSES = ['pending', 'confirmed', 'cancelled', 'completed'] as const;
export type BookingStatus = (typeof BOOKING_STATUSES)[number];

// Every status must have an explicit answer; a new status will not compile until one is given.
export const CONSUMES_SESSION: Record<BookingStatus, boolean> = {
  pending: true,
  confirmed: true,
  cancelled: false,
  completed: true,
};
```

A hand-written second list of the same values (`['pending', 'confirmed', 'completed']`) is a copy even when it is incomplete, and the incomplete one is the more dangerous: a new member falls outside it with no error. Derive runtime lists from the total map.

A coarser view of the set (a three-way role derived from a four-way one) is a derivation, not a second list; derive it and name which scope it collapses.

## Inputs

A field that is derived (see `derived-values.md`) is in the response but not in the request schema. Accepting it as input creates a second writer that loses silently to the derivation. When a field is removed from a request for this reason, note why in the schema so nobody "restores" it.

# The ledger (SSOT registry)

The ledger is the project's map of authorities. The agent reads it before every task and updates it in the same change as the code. A ledger that is stale or wrong is a bug: it sends the next task to the wrong place, or lets it write a second copy because the first one was not listed.

## Layout

Small project: one file, `SSOT.md`, with the row table at the top and the contract blocks below it.

Larger project (more than a few dozen rows), split by area:

```
SSOT.md                    <- entry point: scope, format, area routing table
ssot/registry/NN-area.md   <- row tables, one file per area
ssot/contracts/NN-area.md  <- contract blocks (### #N), one file per area
```

The routing table in `SSOT.md` lists only the area name and the links to its files. No row counts, topic lists or summaries there: that would be a second description of the rows and would drift.

## Row shape

```markdown
| # | Fact | Authority | Owner | Contract | Consumers | Related | Guard |
|---|------|-----------|-------|----------|-----------|---------|-------|
| 12 | Remaining sessions in a customer package | `server/packages/remainingSessions.ts` (`remainingSessions`) | packages | [#12](#12) | Pattern `remainingSessions` in `server/ client/ jobs/` | SSOT-1 | lint row `SC-003` |
| 13 | Service price list | `server/catalog/prices.ts` (`SERVICE_PRICES`, `priceOf`) | catalog | [#13](#13) | Pattern `priceOf\|SERVICE_PRICES` in `server/ client/` | SSOT-1 | review |
| 14 | Membership status of a user | `server/accounts/membership.ts` (`membershipOf`) | accounts | [#14](#14) | Pattern `membershipOf` in `server/ client/`; clients read it from `GET /me` | SSOT-1 | review |
```

Column meanings:

- **#** — a number that names one authority. New number = the highest number in the whole ledger + 1. Numbers are never reused and gaps are not filled. If two rows collide, the one registered first keeps the number; the later one moves and every reference to it is updated in the same change.
- **Fact** — the one question or datum the row owns, in one sentence.
- **Authority** — the file path(s) with the function or constant names in parentheses. This is where the correct value comes from: the single writer of a datum, the single executor of an operation, or the single canonical derivation of a value. For a business decision (permission, visibility, eligibility, default, derived value) name only the server-side evaluation site; client files that display it are consumers.
- **Owner** — the area or module responsible.
- **Contract** — a link to the row's contract block (`references/contracts.md`).
- **Consumers** — the find pattern first: the exported symbol(s) and the folders searched. Names or a count may follow. A bare count is forbidden, because nobody can re-check it. A row with no consumers writes `no consumers — <reason>`; without a reason it is either dead code or a measurement never made.
- **Related** — the rule ID(s) in the rules file that govern the row.
- **Guard** — the mechanism specific to this row that stops a second copy (a lint rule or row, a type, a load-time check), or `review` when no mechanism is worth it. A check that applies identically to every row (for example the ledger checker) is not written in the Guard cell; it is stated once outside the table.

## Granularity

- One row per authority. A helper imported only by that authority is a second path in the same Authority cell, not a second row.
- A file that only declares the shape of a value (type, interface, validation schema, enum or constant table) is not an authority. Test: does the file answer "what is the correct value?" (authority, row) or "what shape does the value have?" (contract shape, named inside the owning row's contract block)? A shape with its own row makes two entries for one fact and a later reader cannot tell which is the writer.
- A shape whose writer has no row is a missing row.

## Content budget

A row carries only: number, the fact, the authority with function names, the consumer pattern, the related rule ID, the guard. A contract block carries only what consumers must honour. A rationale appears only when the choice is not obvious, in at most two sentences; longer reasoning goes to the project's decision records and the row links it. History, dated measurements, commit hashes and the story of how a fix was done are not written in the ledger.

## Paths must exist

Every path written in an Authority cell or a contract block exists on disk. When a file is removed, strike the path through (`~~path~~`) in the same cell in the same change; never delete it silently.

## Retiring a row

A retired authority keeps its number as a tombstone so the number is never reused:

```markdown
| ~~9~~ | Remaining sessions (client-side count) | ~~`client/utils/countSessions.ts`~~ | packages | — | — | SSOT-1 | → #12 |
```

The tombstone names its successor (`→ #12`) or `none`. A live row whose text says "removed" is neither form and misleads every reader.

## Update protocol (every change that touches an authority)

1. Search the ledger for the fact. Found → use that authority.
2. Not found → create one authority, then add the row.
3. Authority insufficient (copies, bypasses) → reduce to one, in the same change.
4. Update the row: authority path, consumer pattern, contract block. Allocate a new number only from the current highest + 1.
5. Re-run the consumer pattern and confirm the result.

## Consumer counting contract

When counting consumers for a row:

- The authority file itself is not a consumer.
- Type-only imports count: they depend on the contract and must change with it.
- A re-export or barrel file is a path, not a consumer; count the real use at the end of the path.
- Exclude dependencies and build output (`node_modules`, `dist`, generated bundles). Counting a build copy counts the same use twice.
- Released clients already in the field are consumers no source search finds (an old app version calling the API). Name them in the cell.

Recipe, per exported symbol:

```bash
grep -rlw '<symbol>' <source folders> | grep -v '<authority path>'
```

The list is raw input: open each file and confirm the use really consumes this authority (a local symbol with the same name is not). This is a hand measurement; writing the pattern in the cell makes it repeatable: two people running it get the same list.

## Closed consumer lists

For a row where missing one consumer is itself the bug (a cache invalidation hub, a fan-out that every write path must call), the Consumers cell is a closed, complete list of names, labelled `**Consumers (complete list):**`. No count, no "etc.", no "+". A reader checks the path they are adding against the names.

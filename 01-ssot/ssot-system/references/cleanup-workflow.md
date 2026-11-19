# Cleanup workflow: find, count, bind

For a project that already grew. The output of the scan is a report; nothing is edited until the user approves.

## 1. Scan

Search by meaning, not only by name. A copy that bypasses the authority does not import it, and it may be called something else.

Ways to find copies:

- **Start from the product.** List the facts the user cares about (prices, balances, remaining sessions, totals, statuses, permissions, limits). For each, search every place that shows, sends, stores or decides it: screens, API handlers, jobs, notifications, exports, reports, admin tools, scripts.
- **Start from the shape.** Search for the statement pattern of a calculation, not just a function name (e.g. `total - used`, `.filter(s => s.status !== 'cancelled').length`). Renamed copies share the shape.
- **Literals.** Repeated numbers, strings and arrays (prices, durations, status lists, currency codes, time zones).
- **Types.** The same interface or schema declared on server and client, or in two services.
- **Writers.** Every update of a stored field. More than one path writing it is two writers.
- **Twins.** Two functions with near-identical names or bodies; one with a default, one without.
- **Disagreement.** Pick a real record (e.g. Emma's package) and read the value on every screen and report that shows it. Different numbers prove the copies already drift.

## 2. Report

For each fact:

```
Fact: Remaining sessions in a package
Places (4): client/screens/Customer.tsx:88, client/screens/Booking.tsx:140,
            server/reports/packages.ts:52, jobs/reminders.ts:31
Agree today? No. Emma's package: customer screen 3, report 5.
Likely authority: none; each place counts on its own.
Proposal: one server function, every place reads it; delete the 4 copies.
```

Group by theme, count per fact, and put the facts that already disagree first. Wait for approval.

## 3. Census before the first edit

For the fact being resolved, list every consumer in the whole repository, with `file:line` and whether it goes through the authority. Write the census in the plan or the chat. A cleanup that binds the four copies it noticed, without counting first, leaves the others; the fact splits again a month later from a copy nobody counted.

## 4. Measure equivalence

Two places that look alike are not proven to answer the same question. Before deleting either:

- Run both on the real set of inputs the copy receives (production-like data, edge cases: zero, empty, cancelled, deleted references).
- **Equal** → merge; the deletion is a pure reduction.
- **Different** → stop. Either the copy is the bug (show the user the input and both outputs), or it answers a different question. In the second case keep one authority and add an explicitly named mode to it that reproduces the copy's behaviour exactly, or register the different question as its own fact. Silently wiring the copy to the authority changes behaviour with no error.
- When many copies are merged, the authority must accept every input the widest copy accepted. Identical bodies with different input types are not identical.

## 5. Bind and delete in one change

- Every consumer reads from the authority.
- Every copy is deleted in the same change. A "temporary" second path is never temporary.
- The consumer uses the authority's value as-is; no re-normalising, no local defaults.
- A one-off repair or migration script calls the same authority the product calls; it is a consumer, not a co-author.
- Update the ledger row in the same change.

## 6. Prove it where the user looks

Show one real input producing the same value on every screen, report and export that displays it, before and after. Re-run the census: zero consumers outside the authority.

## One at a time

Resolve one fact per change. Each change is small enough to review, and a regression points at one fact.

## Live projects

If the project has real users, every cleanup step can create a new bug until the cleanup is finished. Tell the user before starting, prefer the facts that already disagree (they are already wrong), keep each change small, and verify each one on the real screens before the next.

## When a bug is reported later

Fix the class, not the instance:

1. Name the wrong logic behind the bug.
2. Search for it across the whole repository, not only the callers of the code you changed; count every occurrence.
3. Bind every occurrence to one authority (the existing one, with stray copies deleted; otherwise a new registered one).
4. Prove the fix at the outcome the user sees (server response, client, screen).

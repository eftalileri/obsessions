# Agent notes: side findings

While working, the agent will notice problems that are not part of its task. Fixing each one on the spot scatters the work and breaks the task in hand. Ignoring them loses them. The notes file holds them until the task is done.

## What goes where

| Seen during a task | What to do |
|---|---|
| A duplicate of the fact the task is changing | Part of the task. Bind it to the authority now (two questions, Phase 6). |
| A wrong ledger row, a stale comment, a wrong label | Fix it on sight if it is tiny and certain. |
| A different problem, in the same or another area | Write it to `AGENT-NOTES.md` if it passes the five conditions. Do not fix it now. |
| A product question (what should happen?) | Ask the user; do not guess. |

## The five conditions

An entry enters only when all five are clear. If one is unclear, the measurement is incomplete and no entry is written.

1. **What it breaks** — the concrete harm (wrong behaviour, security, money, data integrity, a second authority for one fact, a promise no check enforces), and the rule in the rules file it violates. "Could be cleaner" is not harm.
2. **Where** — `file:line`.
3. **How many** — a count of places, files or call sites. Without a count, the work cannot be defined and completion cannot be proved.
4. **Not already covered** — not a duplicate of a live entry, not already stopped by a guard (look for the guard by what it checks, not by its name).
5. **Worth fixing** — the cost (files touched, risk, need for outside evidence or a user decision) is worth what the fix buys. If not, it does not enter.

Each entry must also answer: who can close it, in which file, and how will we know it is closed?

## Never filtered out

However costly, these always enter (or are fixed in the task, if they are part of it):

- security or access between users/tenants;
- money or paid entitlements;
- data loss or corruption;
- a silent failure (something that works wrongly without telling anyone);
- an SSOT violation (a second authority, a second writer, a parallel calculation).

## Entry shape

```markdown
### [NOTE-007] Report total counts cancelled bookings
Breaks: report total differs from its rows (rule SSOT-6). Money shown to owner is wrong.
Where: server/reports/revenue.ts:74 (1 place; 2 consumers read the total).
Covered? No entry, no guard.
Worth it: 1 file, low risk.
Closes when: total is derived from the rows the report shows; revenue screen and export agree.
Status: OPEN
```

Numbers are allocated like ledger numbers: highest + 1, never reused.

## Keep it a queue

- After the task in hand is done, go back to the notes and work through them.
- When an entry's work is done (fixed, or the user decided no action), **delete** it in the same change. Do not move it to an archive. Keep only a short list of retired numbers so they are not reused.
- A partly done entry stays, with the remaining part named precisely.
- Before closing an entry, re-check today's code; the note may already be stale.

Expect the file to grow at first: each fix often reveals problems that were already in the code but unnoticed. That is the system working, not the cleanup failing.

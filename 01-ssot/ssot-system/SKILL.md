---
name: ssot-system
description: "Install a single source of truth (SSOT) system in the user's project: find every fact, calculation, list, rule or type that is kept in more than one place, report the copies with counts, keep a ledger (SSOT.md) with one row per fact naming its single authority, contract, consumers and guard, reduce duplicates one at a time into the authority, keep derived values derived, stop new copies with guards where they pay off, keep side findings in an agent notes file instead of fixing them mid-task, install agent hooks (when the tool supports them) that force the agent to read the ledger and rules before editing source, and add the matching rules once to the project's own rules file. Use when the user asks to set up SSOT or a single source of truth, find duplicated logic or copies that drift apart, create an SSOT ledger or registry, clean up a project that grew messy, or make their AI agent stop re-implementing things that already exist. Stack-agnostic."
---

# SSOT system (one fact, one place)

Goal: every fact in the project — a value, a calculation, a list, a rule, a type, an operation — has **exactly one authority**. Everything else that needs the fact reads it from there. Nobody keeps a copy, nobody recomputes it, and when two places disagree the authority wins. The project keeps a ledger that says where each authority is, so the agent never has to remember yesterday.

SSOT does not mean "put everything in one file". Different facts have different authorities (a database table and its service, a shared constants module, a policy function, a design token file). One mega-module or mega-store for unrelated facts is itself a violation.

You are working in the user's project. The user owns every decision. Follow the phases in order. Do not edit code before the user approves the plan from Phase 1.

Reference files (read the one for the phase you are in):

- `references/registry-format.md` — the ledger: row shape, columns, numbering, consumer patterns, retired rows
- `references/contracts.md` — contract blocks: what a consumer must honour, canonical output, closed value sets
- `references/derived-values.md` — derived vs stored vs frozen values, drift checks
- `references/cleanup-workflow.md` — scan signals, consumer census, measuring equivalence, one fact at a time
- `references/guards.md` — which mechanism stops new copies, when a guard is worth it, how to prove it, stated limits
- `references/hooks.md` — agent hooks: read gate, ledger lookup, push gate, self-protection, fail-closed rules, proof
- `references/agent-notes.md` — side findings: the five conditions, the classes never dropped, keeping the file short
- `references/rules-template.md` — the rule block to add to the user's rules file
- `references/verification.md` — what "installed" and "fixed" mean, with evidence

Adapt every example to the project's language and framework. File names here (`SSOT.md`, `AGENT-NOTES.md`) are suggestions; if the project already has an equivalent, extend it instead of adding a second one.

---

## Phase 0 — Orient

1. Find the project's rules file: `CLAUDE.md`, `AGENTS.md`, `.cursor/rules/*`, `CONTRIBUTING.md`, or a project constitution. Note its rule-ID style if it has one. If the user does not know which file their tool reads, check the tool's documentation or ask the tool.
2. Find any existing ledger, registry, architecture doc, "constants" module, shared types package, notes or TODO file. Note what they already claim to own.
3. Identify the stacks: server, clients (web, mobile, desktop), database and schema layer, shared code between them, background jobs, generated code, lint and type-check tooling, CI or pre-commit hooks.
4. Ask whether the project is live (real users). Cleanup on a live project creates new bugs until it is finished; the user must know this before Phase 3.
5. Do not change anything yet.

## Phase 1 — Scan and report (no edits)

Details: `references/cleanup-workflow.md`.

Search by **meaning**, not only by name: a copy that bypasses the authority never imports it and is often named differently. Look for at least:

| What to find | Typical signals |
|---|---|
| Same calculation in several places | the same formula (totals, remaining balance, discount, tax, duration, age, "is active") written in a screen, a report, an API handler and a job |
| Same constant or list in several places | prices, limits, status lists, role lists, currencies, time zones, feature flags written as literals in more than one file |
| Same type or schema declared twice | a server type and a hand-written client copy; a database schema and a validation schema that disagree |
| Same rule checked in several places | permission, eligibility or visibility decided on the client and again on the server, or in two services |
| Two writers of one field | more than one code path updating the same stored value |
| Stored copies of derived values | a counter, total or flag saved on a record and also computable from other data |
| Twin functions | two helpers answering one question that differ only in what they return when there is no answer |
| Stand-in data | hard-coded sample lists, fake responses, placeholder values on a production path |
| Copies that already drift | two places that should agree and give different answers today (show the inputs and both outputs) |

For each fact give: what the fact is, every place it lives (`file:line`), how many places, whether they agree, and which place looks like the real authority. Group by theme, with counts. Then propose the plan (Phases 2–9) sized to what you found, and **wait for approval**.

## Phase 2 — Create the ledger

Details: `references/registry-format.md`, `references/contracts.md`.

- One file (`SSOT.md` at the project root, or the folder layout in the reference once it grows). The agent writes it; the user reads and approves it.
- One row per authority. A row answers one question: **what is the fact, which file and function is its single authority, what shape does it have, who uses it and how to find them, and what stops a new copy.**
- Columns: `#`, Fact, Authority (file + function names), Owner area, Contract (link to its block), Consumers (a find pattern, not a bare count), Related rule, Guard.
- Contract blocks hold only what a consumer must honour: inputs, outputs, invariants, error behaviour, and the files that declare the shape. Not history, not dates, not how a fix was done.
- Type, schema and constant files that only declare a shape are not authorities and get no row of their own; they are named inside the owning row's contract block.
- Start with the facts from Phase 1 and the most important facts of the product. Small is fine; the target is all of them.
- Example first rows for a salon booking app: "Remaining sessions in a package: one calculation; every screen that shows remaining sessions reads it." "Service prices: one list; every page that shows a price reads it." "Membership: the user's account record; no other screen keeps its own copy."

## Phase 3 — Resolve duplicates, one fact at a time

Details: `references/cleanup-workflow.md`, `references/derived-values.md`.

For each fact the user approved, in order:

1. **Census.** List every consumer across the whole repository — screens, endpoints, jobs, notifications, exports, reports — that reads, shows or writes the fact, and mark whether each goes through the authority. Count them. Do this before the first edit; a cleanup that skips the count misses copies and the fact splits again later.
2. **Pick or create the authority.** If one exists, use it. If none exists, create exactly one, in the layer that owns the fact (the server for business decisions; clients display the server's answer).
3. **Measure before merging.** Two places that look alike are not proven to answer the same question. Run both on the real inputs the copy receives. Equal → merge. Different → do not silently wire the copy to the authority; either the copy was wrong (show the user) or it answers a different question, which becomes a named mode of the authority or a separate fact with its own row.
4. **Bind and delete.** Point every consumer at the authority and delete the copies **in the same change**. No "copy for now, fix later", no temporary second path.
5. **Update the ledger row** (authority, contract, consumer pattern) in the same change.
6. Show the user the diff and the before/after census. Then the next fact.

## Phase 4 — Derived vs stored values

Details: `references/derived-values.md`.

- A value computed from other data is **derived**. It is read, never written back. Do not accept it as input in a request or form; do not store it unless measurement shows it must be cached.
- If a derived value must be stored (performance, a database query needs it), it has one writer that recomputes it from the authorities every time, and every path that changes an input calls that writer. Prefer recomputing on read.
- A **frozen** value (the price at the moment of booking) is a different fact from the live value (today's price) and gets its own row. Later calculations about that record read the frozen copy, not the live catalogue.
- Any stored copy that must match its original gets a drift check: a read-only comparison of copy vs recomputed value that reports mismatches loudly. A mismatch is a bug in a writer, not a reason to patch the copy.

## Phase 5 — Guards that stop new copies

Details: `references/guards.md`.

Pick the strongest mechanism that closes the class, in this order: (1) design or types that make the copy impossible (one exported function, a closed union with a total map, a required parameter); (2) a new row in an existing table-driven lint rule; (3) a new lint rule for something one file shows; (4) a cross-file script; (5) a written limit held by review.

- Add a guard only when the copy is a real, recurring risk, the harm is serious, nothing stronger already prevents it, and the cost is small. Otherwise the row's Guard cell says `review` — that is an honest answer, not a gap to hide.
- Each guard runs where work is saved or merged (pre-commit, CI). Prove it: it fires on a known bad sample, stays silent on the clean tree, and fails when a temporary violation is added.
- A ledger checker (read-only script that checks the ledger's links, paths and row format) is optional; treat the ledger's shape as review-held unless the user wants the checker wired into CI.
- A guard is never edited to whitelist one file; a verified false positive gets a reasoned, line-level exception.

## Phase 5b — Hooks that stop the agent before it starts

Details: `references/hooks.md`.

- A guard catches a copy when work is saved; a hook stops the agent's tool call earlier, for example an edit before the agent has read the ledger and the rules. Reading costs context and credits; the payoff is fewer revision rounds.
- First check which agent tool the user runs and whether it has hooks. If it has none, tell the user and stay with guards and rules; never pretend.
- The read gate is always installed when the tool supports hooks: the agent cannot edit source before it has read `SSOT.md` and the applicable rules in full (a partial read does not count; re-read after a change or a compaction). Offer the rest only as the project needs: ledger-lookup gate (optional), push gate and self-protection (ask first).
- Every hook fails closed (bad input, timeout risk, missing state: block) and uses no assumptions, defaults or silent fallbacks. Every block names the reason and the exact action that unblocks it.
- Hooks are never switched off to get work through; a wrong block is repaired in the hook.

## Phase 6 — Every task: the two questions

These become standing rules (Phase 8). On every task, the agent asks:

1. **Before starting: "Does this already exist?"** Trace the flow end to end (input → where it is decided → where it is written → what derives from it → who shows it) and search the ledger. Exists → use it. In the code but not in the ledger → add the row. Missing → create it in one place and add the row.
2. **Before finishing: "Who else uses this, and are they all updated?"** Run the census for every fact the change touched. Every consumer goes through the authority; the ledger row is current.

A stale or missing ledger row is a bug. When a bug is fixed, fix the class: name the wrong logic, search the whole repository for it, count the copies, bind every copy to the authority, and prove the result on the screen the user sees. Fixing only the instance shown leaves the other copies to break later.

## Phase 7 — Side findings go to agent notes

Details: `references/agent-notes.md`.

- A problem seen during a task that is not part of the task is not fixed mid-task (that scatters the work). It goes to `AGENT-NOTES.md`, and the agent returns to it after the task.
- Exception: a duplicate of the very fact the task is changing is part of the task; bind it now (Phase 6).
- An entry enters only when all five are clear: what it breaks, where (`file:line`), how many places, that it is not already recorded or guarded, and that the fix is worth its cost. Unclear → no entry.
- Never filtered out, however costly: security, money, data loss or corruption, a silent failure (something that works wrongly without telling anyone), an SSOT violation.
- The file is a queue, not an archive: an entry whose work is done is deleted.

## Phase 8 — Write the rules into the user's rules file

Details and the rule block: `references/rules-template.md`.

- Add short rules with stable IDs, in the file's existing style. Each rule is written **once**; other places (code comments, docs, prompts, sub-agent briefs) point to the ID instead of restating it. A rule written twice is itself a duplicate fact.
- If the project has no rules file, propose creating one (e.g. `AGENTS.md`, or `CLAUDE.md` for Claude Code) and wait for a yes.
- Point to `SSOT.md`, `AGENT-NOTES.md` and the guard scripts by path.
- Suggest the user ends task prompts with one line pointing at the rule (e.g. "SSOT violations are forbidden, see SSOT-1").

## Phase 9 — Verify

Details: `references/verification.md`.

1. Every ledger path exists; every row's consumer pattern, run now, returns only consumers that go through the authority.
2. For each fact resolved in Phase 3: before/after census, the deleted copies, and one real input whose output is now identical on every screen that shows it.
3. Each guard: fires on a bad sample, silent on the clean tree, fails on a temporary violation, runs in pre-commit or CI.
4. Each hook, if installed: blocks a known bad action, stays silent on a good one, fails closed when its state is missing (`references/hooks.md` section 11). Show the results.
5. A dry run of the two questions: give the agent a small task that touches a registered fact (e.g. "add remaining sessions to the report") and show it finds the row and reuses the authority instead of writing a new calculation.

---

## Reporting to the user

At the end, give: the ledger (number of rows, link), per resolved fact the copies found → copies left (should be 0), the guards added and their proof, the hooks installed (or why none) and their proof, the rules added (IDs), the open entries in `AGENT-NOTES.md`, and what is still review-held. Keep it short.

## Do not

- Do not centralise unrelated facts into one module or store.
- Do not merge two look-alike calculations without measuring that they answer the same question.
- Do not leave a copy "for now"; delete it in the same change that binds its consumers.
- Do not write a bare count in the ledger; write the pattern that finds the consumers.
- Do not add guards the project does not need, and do not claim a guard covers what it cannot detect.
- Do not install hooks the project does not need, write hooks that fall back to a default, or turn hooks off to get work through.
- Do not fix side findings mid-task, and do not let the notes file become an archive.
- Do not copy the rules into several files.

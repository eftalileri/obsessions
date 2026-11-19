# Rule block for the user's rules file

Add this to the project's rules file (`CLAUDE.md`, `AGENTS.md`, `.cursor/rules`, or the project constitution). Adapt IDs to the file's existing style (keep them stable once written). Replace the bracketed paths with the real ones. Drop `SSOT-9` if no guards were added.

Single source: each rule lives only here. Code comments, docs, prompts, the ledger and sub-agent briefs refer to the ID (e.g. "see SSOT-4") and never restate or paraphrase it. When a rule changes, only this block changes.

If the project has no rules file, propose creating `AGENTS.md` (or `CLAUDE.md` for Claude Code) with this block and wait for the user's yes.

```markdown
## Single source of truth (SSOT)

SSOT violations are forbidden. Ledger: `[SSOT.md]`. Notes: `[AGENT-NOTES.md]`. Guards: `[path to lint rules / scripts]`.

- **SSOT-1 One authority.** Every value, calculation, list, rule, type and operation has exactly one authority. Everything else reads it from there; it does not change it or keep a copy. If two places disagree, the authority is right. One authority per fact, not one place for everything.
- **SSOT-2 Ledger.** `[SSOT.md]` has one row per authority: the fact, the authority file and function, the contract, the consumers (as a find pattern), the related rule, the guard. A stale or missing row is a bug.
- **SSOT-3 Before work: does it exist?** Before writing code, trace the flow end to end and search the ledger. Exists → use it. In the code but not in the ledger → add the row. Missing → create it in one place and add the row.
- **SSOT-4 No copies.** Two places doing the same job are bound to one authority and the copies are deleted in the same change. No "copy it for now, fix later". Before merging two look-alikes, measure that they answer the same question.
- **SSOT-5 Before finishing: who else uses it?** List every consumer of each fact the change touched; all go through the authority; the ledger row is current.
- **SSOT-6 Derived values.** A derived value is computed by one derivation, never written back and never accepted as input. A stored copy has one writer that recomputes it, and a drift check against its source. A frozen copy (value at a past moment) is its own fact.
- **SSOT-7 Bugs are classes.** When fixing a bug, find every copy of the wrong logic in the whole project, count them, bind all of them to the authority, and prove the result on the screen.
- **SSOT-8 Side findings.** A problem outside the task goes to `[AGENT-NOTES.md]`, not fixed mid-task, and only when what it breaks, where, how many, not already recorded, and worth fixing are all clear. Security, money, data loss, silent failure and SSOT violations are never filtered out. Finished entries are deleted. Return to the notes when the task is done.
- **SSOT-9 Guards.** Guards in `[path]` must pass before commit/merge. A guard is never edited to whitelist one file. What no guard covers is marked `review` in the ledger.
- **SSOT-10 Hooks.** Hooks in `[path]` are never disabled, edited or bypassed to get work through; a hook that blocks wrongly is repaired, with the user's approval. Drop this rule if no hooks were installed.
```

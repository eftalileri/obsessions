# ssot-system

An Agent Skill that installs a single source of truth (SSOT) system in your project: every fact, calculation and rule lives in one place, and everything else reads it from there.

## What it does

1. Scans your project and lists the places that keep the same fact or calculation separately, with counts. No change without your approval.
2. Creates a ledger (`SSOT.md`): one row per fact, saying where its single source is and who uses it.
3. Binds the copies to the single source one fact at a time, after checking they really answer the same question, and deletes the copies.
4. Keeps derived values derived and checks stored copies against their source.
5. Adds guards where they pay off, so new copies are stopped before they are saved.
6. Optionally installs agent hooks (if your tool supports them, e.g. Claude Code): the agent cannot edit code before it has read the ledger and rules, cannot push to main without passing checks, and cannot switch the hooks off. Every hook fails closed and is proven on a bad and a good action.
7. Sets up an agent notes file (`AGENT-NOTES.md`) for problems found along the way.
8. Writes the rules once into your rules file (`CLAUDE.md`, `AGENTS.md`), including the two questions for every task: "Does this already exist?" and "Who else uses this, and are they all updated?"
9. Proves it works on a real task.

If your project has real users, cleanup can create new bugs until it is finished. The skill asks before it starts.

## How to use

Copy the `ssot-system` folder into your agent's skills folder (for Claude Code: `.claude/skills/`). Then tell your agent: "Use the ssot-system skill to set up a single source of truth in my project."

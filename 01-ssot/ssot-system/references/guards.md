# Guards: stopping new copies

A guard is a small automatic check that runs before work is saved or merged and rejects a new copy. A rule the agent read can still be skipped; a guard does not skip. But every guard is paid for on every run and every refactor, so add one only where it earns its place.

## Pick the strongest mechanism first

Close the class with the highest rung that closes it:

1. **Design or types that make the copy impossible.** One exported function with no alternative path; a closed union with a total map (`contracts.md`); a required parameter instead of a default; a branded or opaque type only the authority can create; one function that performs the check and the effect together.
2. **A row in an existing table-driven lint rule** that already asks the same question.
3. **A new lint rule** (syntax tree) for a condition one file shows.
4. **A cross-file script** only when no single file can decide it.
5. **A stated limit**: the rule stays, review holds it, and the ledger's Guard cell says `review`.

Choosing a lower rung means saying why each higher rung cannot close the class.

## When a guard is worth adding

All four must hold:

- **Real, recurring risk**, with evidence (it happened, or copies of this kind already exist).
- **Serious harm** (wrong money, wrong access, lost or corrupted data, a silent wrong answer to users).
- **Nothing stronger already prevents it** (the compiler, an existing rule).
- **Small cost** (fast, few false positives, little maintenance).

If any fails, do not build it. `review` in the Guard cell is the honest default, not a failure.

## Table-driven rules

Prefer one rule with many rows over one rule per fact. A new protected fact becomes a row, not a new rule body:

```js
// "Is this authority function called outside its owner files?" (single caller)
{ id: 'SC-003', kind: 'call', match: { names: ['remainingSessions'] },
  owners: ['server/packages/**'], files: ['server/**', 'client/**', 'jobs/**'],
  message: 'Use remainingSessions — SSOT.md #12.' }

// "Is this field written outside its writer files?" (single writer)
{ id: 'SW-001', fields: ['remainingSessionsCache'], surfaces: ['update'],
  owners: ['server/packages/recomputePackage.ts'], files: ['server/**', 'jobs/**'],
  message: 'Only recomputePackage writes this field — SSOT.md #12.' }
```

Typical row kinds:

- **Single writer** — a protected field may be written only in the listed files.
- **Single caller** — an authority function may be called only from the listed places (when callers must be fenced; usually anyone may call it).
- **Model mutation access** — writes to a listed table or model run only from its service.
- **Known stale form** — a specific syntactic shortcut known to give a second answer is allowed only in the owner files.

Rules for the table:

- Owner globs are as narrow as possible. Every listed path must match a real file when the rule loads; a dead entry is removed in the same change that deletes or renames its file (otherwise a file later created under that name silently inherits write access).
- One resolved row per protected symbol; two conflicting rows are forbidden.
- Never edit the rule body to whitelist one file. A verified false positive gets a line-level exception with its reason, naming the row.

## Known limits (write them down)

- A rule that matches by name does not see a copy re-implemented under another name. Review and the census catch that.
- "This is the only place that computes X" is not decidable from one file. Two functions can compute the same value with no shared syntax.
- Parity with a system outside the repository (a provider console, another repo) cannot be checked here.
- Whether a product rule is correct cannot be checked by restating it in a guard; that would be a second authority.

Where a guard cannot cover something, the Guard cell says what is covered and what is review-held. Zero findings must not be mistaken for coverage.

## The ledger checker (optional)

A read-only script that checks the ledger itself:

- the routing table links resolve and every area file is routed;
- table headers name the columns by role; each row has the header's cell count;
- row numbers are unique; ids are `N` or a tombstone `~~N~~`;
- every non-struck path in Authority cells and contract blocks exists (globs match at least one file);
- every Consumers cell starts with a find pattern or `no consumers — <reason>`;
- every row links its own contract block and every block is linked from exactly one row.

It reports every defect with `file:line` in one run and exits non-zero. Whether it runs as a CI gate is the user's choice; the ledger's shape is otherwise held by review. A checker that stops at the first defect hides the rest.

## Proving a guard

Without creating permanent test files, prove each guard against the real tree:

1. **Positive:** it fires on a known bad sample (for example the pre-fix code, extracted to a scratch location).
2. **Negative:** it reports zero false positives on the clean tree.
3. **Ratchet:** add a temporary violation, see the gate fail, revert, confirm the diff is empty.

Then confirm it is wired into pre-commit or CI. A guard that is not run protects nothing.

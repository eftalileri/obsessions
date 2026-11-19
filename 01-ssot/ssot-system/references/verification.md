# Verification

Claims are not proof. Each step below ends with evidence the user can see.

## 1. The ledger is true

- Every path in every Authority cell and contract block exists (or is struck through as removed).
- Row numbers are unique; every row links its contract block; every block is linked from one row.
- For each row, run its consumer pattern now. Every hit goes through the authority; none recomputes or copies the fact. Show the command and its output for at least the rows resolved in this session.
- If the ledger checker was added, show its run with zero defects.

## 2. Each resolved fact is one fact

For every fact resolved in Phase 3, show:

- the census before (places, `file:line`, count) and after (only the authority, plus consumers that read it);
- the deleted copies (diff);
- the equivalence evidence: equal outputs on the real inputs, or the named mode added for a real difference;
- one real record (e.g. Emma's package) showing the same value on every screen, report and export that displays it.

## 3. Derived values stay derived

- Request schemas do not accept derived fields.
- Each stored copy has one writer; every input-changing path calls it (list them).
- The drift check runs and reports zero mismatches, or reports the real ones with ids and both values.

## 4. Guards work

For each guard:

- **Positive:** fires on a known bad sample.
- **Negative:** zero findings on the clean tree.
- **Ratchet:** a temporary violation makes the gate fail; revert; the diff is empty.
- It runs in pre-commit or CI (show the config line).

List what is review-held, so zero findings is not read as full coverage.

Hooks, if installed: each blocks a known bad action, stays silent on a good one, and blocks when its state or config is missing (`hooks.md` section 11). Show the results.

## 5. The two questions work

Give the agent a small task that touches a registered fact, in a fresh conversation, with only the rules file loaded. Example: "Add each package's remaining sessions to the report." Show that it:

- searches the ledger first and names the row;
- reuses the authority instead of writing a new calculation;
- lists the consumers before finishing;
- the value in the report equals the value on the customer screen.

## 6. Rules are written once

The rule block exists in exactly one file. Search the repository for distinctive phrases of each rule; any second copy (in a comment, a doc, a prompt file) is replaced by a pointer to the ID.

## "Done" means observed

A fact is resolved when the census shows one authority and the screens agree on real data, not when the commit message says so. A later report of the same wrong number means a copy was missed: re-run the census by meaning, not by name.

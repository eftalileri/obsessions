# Hooks: stopping the agent before it starts

A rule the agent read can still be skipped. A guard (`guards.md`) catches the copy when the work is saved. A hook acts earlier: it stops the agent's tool call before the edit happens, for example until the agent has read the ledger and the rules that apply. Cost: reading spends context and credits on every task. Payoff: fewer revision rounds, because the agent starts from the ledger instead of from memory.

Install only the gates the project needs. The agent writes the real scripts for the user's project; the snippets below are skeletons.

## 1. Detect the tool first

Before writing anything, find out which agent tool the user runs and whether it has a hook system.

- Claude Code: hooks are registered in `.claude/settings.json` (project, shared) or `.claude/settings.local.json`. Check the tool's current hooks documentation before writing; the details in section 2 were checked against it, but versions change.
- Another tool: check its documentation or ask the tool. If it has no way to run a script before a tool call, say so to the user in plain words and stop this phase. Fall back to guards and rules only (`guards.md`, `rules-template.md`). Never write files that look like hooks but are never run.

Also ask: Node or Python for the scripts (whichever the project already has). Use the same language for every hook.

## 2. Hook mechanics (Claude Code)

- Events used here: `PreToolUse` (before a tool call; can block), `PostToolUse` (after a successful call), `ConfigChange` (a settings file changed during the session; can block), `PostCompact` and `SessionStart` with matcher `compact` (after a compaction; used to clear read records, section 5).
- Registration: event, then a matcher on the tool name (`Edit|Write|MultiEdit|NotebookEdit`, `Bash`, `Read`), then `{ "type": "command", "command": "...", "timeout": <seconds> }`.
- Input: one JSON object on stdin with `session_id`, `cwd`, `tool_name`, `tool_input` (`file_path`, `command`), `tool_use_id`, `transcript_path`, and `agent_id` when the call comes from a sub-agent.
- Output: exit 0 allows. Exit 2 blocks and shows stderr to the agent. Any other exit code is a non-blocking error: the call goes ahead. So every hook command ends with `|| exit 2`, which turns a crash into a block.
- A timed-out hook does not block by default: the call goes through. Use two defences: the script sets its own deadline well below `timeout` and blocks when it is hit; and where the tool version supports it, add `"onFailure": "block"` to the hook (the documentation names Claude Code 2.1.295 as the first version).
- `${CLAUDE_PROJECT_DIR}` is the project root where the session started; use it in commands instead of relative paths.

`PostToolUse` input carries `tool_input`, `tool_response`, `tool_use_id` and `duration_ms`. `ConfigChange` input carries `source` (`user_settings`, `project_settings`, `local_settings`, ...) and `file_path`; this is what a working system reads, and the documentation page for it was only partly read. Before relying on any field, print one real input for each event (section 11, item 8) and read it.

## 3. Fail closed: no assumptions, no defaults, no silent fallbacks

This is a requirement for every hook, not a style choice.

- Parse stdin strictly. Not JSON, a missing field the hook needs, an unknown tool name, an unreadable state file, a state file with an unknown shape: block, with a message.
- Oversized input (a very long command) and a hit deadline: block. Never "skip the check".
- No default identity, no default path, no default rules file. The hook reads them from the project's hook config file (section 4) and blocks when the config is missing or lists a path that does not exist.
- Identity (who is acting) comes only from hook input: `agent_id` if present, else `session_id`. Never from an argument the agent typed.
- No `try { ... } catch { return }`. A caught error ends in a block.

Skeleton (Node; the same shape in Python):

```js
// hooks/lib.mjs
import { readFileSync } from 'node:fs';
export function block(reason, unblock) {
  process.stderr.write(`HOOK: ${reason}\nTo unblock: ${unblock}\n`);
  process.exit(2);
}
export function run(body) {
  const timer = setTimeout(() => block('hook hit its own deadline', 'retry; the hook fails closed'), 8000);
  try {
    const raw = readFileSync(0, 'utf8');
    if (raw.length > 1_000_000) block('hook input too large to inspect', 'use a smaller command');
    body(JSON.parse(raw));
  } catch (e) {
    block(`hook failed (${e.message})`, 'fix the cause above and retry; the hook fails closed');
  }
  clearTimeout(timer);
}
export const actor = i => (i.agent_id ? `agent:${i.agent_id}` : `main:${i.session_id}`);
```

A synchronous hang cannot be cut by a timer in the same process. If a hook calls slow programs (git, network), run the body in a child process and kill it before the registered `timeout`, then block.

## 4. One config file, written once

`hooks/config.json` (name it to fit the project) holds every path the hooks need: the ledger, the rules file, the source globs the read gate protects, the doc globs it ignores, the guard command for the push gate. Hooks read it; none hard-codes a path. It is itself a protected file (section 8), and the ledger gets a row for it.

## 5. Read gate (always installed)

Whenever the user's tool supports hooks, install the read gate. It is not optional; only its scope (which source paths, which documents) is the project's choice. The push gate and self-protection stay ask-first, the lookup gate stays optional.

Goal: before the first edit to project source, the agent has read `SSOT.md` and the rules file in full in this session, and has read them again if they changed since.

How it proves this, from real state and not from the agent's word:

- A `PostToolUse` hook on `Read` writes a record per actor: `{ file, sha256 of the content, line range read, time }`, into the hook state folder (section 8). Failed reads write nothing. The hook input carries `tool_input` (with `file_path`, and `offset` and `limit` when the agent gave them) and `tool_response`.
- A read counts only when it covers the whole file: a read with no `offset` and no `limit` (and a file shorter than the tool's default read cap; check that cap in the tool's documentation), or several recorded ranges at the same sha256 that together cover every line. `Read SSOT.md limit:1` does not unlock anything. A file longer than the cap needs ranged reads until the union covers it; the block message says which lines are missing.
- A `PreToolUse` hook on `Edit|Write|MultiEdit|NotebookEdit` checks, for every required document, that this actor's records at the file's current sha256 cover the whole file. A document edited after it was read needs a new read.
- Documentation and the hook state itself are not gated by this hook. Source globs come from the config.
- With a split ledger, require the entry file and the rules file always; the area files are covered by the lookup gate (section 6), because the hook cannot know the task.
- Bash can write files too. The hook also reads `Bash` commands: a redirect (`>`, `>>`, `tee`), `sed -i`, `perl -i`, or an interpreter run that names a source path counts as an edit. State the limit (section 8): a command assembled at runtime is not seen.

Block message, for example: `HOOK: SSOT.md was not read by this agent (or changed since). To unblock: read SSOT.md and AGENTS.md with the Read tool, then retry the edit.`

After the conversation is compacted, the agent forgets what it read while the records remain. In Claude Code, register a hook on `PostCompact` and another on `SessionStart` with matcher `compact`; both delete the read records of that session (and of its sub-agents), so the agent must read again before its next edit. A failed delete blocks (section 3). In another tool, look for an equivalent event; if there is none, tell the user this limit.

## 6. Ledger-lookup gate (optional)

Use when reading the whole ledger is too heavy, or the user wants proof that the agent looked up the facts for this task.

- The project gets a small command, for example `node tools/ssot-lookup.mjs --facts <words>`, that searches the ledger and prints only the matching rows and their contract blocks. It reads the ledger and writes nothing.
- A `PostToolUse` hook on `Bash` matches the exact command form (an anchored pattern on plain characters; anything else does not count) and, after success, records `{ actor, checkout, facts, row numbers, time }`.
- The edit hook blocks until a record exists for this actor and this checkout. The unblock message is the exact command to run.
- Identity and checkout come from hook input (`agent_id`/`session_id`, `cwd`), never from the command's arguments.
- Known limit: the hook proves the lookup ran, not that the words fit the task. Review holds that; say so in the rules.

## 7. Push and merge gate (ask the user first)

Goal: nothing reaches the main branch unless the project's checks and guards pass.

- A `PreToolUse` hook on `Bash` reads the command and blocks a `git push` that targets the main branch (or a merge command that does) unless a check receipt exists: the guard command from the config ran green on exactly the commits being pushed (record the commit id and the result; compare on push).
- It blocks every form that skips the git hook or hides the target: `--no-verify`, `-c core.hooksPath=...`, other `-c` config overrides, `--receive-pack`, an unresolvable remote or directory, git run through a shell string, remotes changed in the same command, aliases that expand to those. Resolve aliases and read the real target; do not trust the branch name typed.
- It blocks `--no-verify` in any command.
- A git `pre-push` hook (or CI) runs the same check, so a push from a terminal outside the agent is held too. The hook stops the agent; the git hook stops everyone.
- Parsing shell text is the hard part. Refuse what the parser cannot read; do not guess it.

Ask the user before installing: it changes how they push.

## 8. Self-protection (ask the user first)

Without this, the agent can switch the gates off. Install it with the first gate, after the user agrees.

- The file hook blocks edits to: the hook scripts, the hook config, the hook state folder, the settings files that register them (`.claude/settings.json`, `.claude/settings.local.json`, the user-level settings), and the git hook directory.
- The Bash hook blocks commands that name those paths in a write form, and any command that sets `disableAllHooks`.
- A `ConfigChange` hook (matchers `user_settings`, `project_settings`, `local_settings`) blocks a change whose resulting content sets `disableAllHooks` to anything but false, removes or alters the registered hooks, or adds an `env` block (it reaches every hook process). It appends every change to a log in the state folder. Judge the parsed content, not the written text.
- The state folder lives outside the project tree (for example under the user's home, per project), or inside it and ignored by git; either way the file and Bash hooks refuse agent writes to it. Only hook processes write there.
- To change a hook the user does it from their own terminal, reviewed, and restarts the session. Say this in every self-protection block message.
- Known limits, written down for the user: a new session started after a change loads the new settings; a Bash command assembled at runtime (variables, encoded text, a script written to disk and then run) is not seen by a text check. Hooks reduce skipped rules; they are not a security boundary against a determined agent.

## 9. Messages

Every block names the reason and the exact action that unblocks it: a command to run, a file to read, or "ask the owner". A block with only a reason makes the agent guess, and the guess is how bypasses start. Keep each message to two lines.

## 10. Install steps

1. Detect the tool (section 1). Tell the user the result.
2. Install the read gate without asking (when the tool supports hooks). Propose the rest: lookup gate (large ledgers), push gate (a main branch that ships), self-protection (with any gate). Wait for the user's choice; ask before the push gate and self-protection.
3. Write the scripts and `hooks/config.json`; register them in the settings file; add the ledger rows for the config and the scripts' owners; add the hook rule to the rules file (`rules-template.md`).
4. Prove each hook (section 11) and show the user the results.
5. Tell the user: restart the session so the hooks load; if a hook blocks work wrongly, report it. The fix is to repair the hook, never to turn hooks off.

## 11. Proof

Without permanent test files, run each hook by hand against its real script, feeding stdin JSON (`echo '<json>' | node hooks/pre-edit.mjs; echo $?`), and then once in a live session. Show every result.

For each hook:

1. **Bad action blocked:** an edit of source with no read record: exit 2 and the message. A push to main without a receipt: exit 2. A write to a hook script: exit 2. A settings change with `disableAllHooks: true`: exit 2.
2. **Good action silent:** after the read record exists, the same edit exits 0 with no output. A push after a green check passes.
3. **Missing state fails closed:** remove the state folder or the config in a scratch copy: the hook blocks with a message, it does not allow. Feed invalid JSON, empty input, an unknown tool name and oversized input: each blocks.
4. **Partial read stays blocked:** read `SSOT.md` with `limit: 1` (and again with an `offset` that skips the top): the edit still blocks and the message names the missing lines. A whole read, or ranged reads that together cover the file, unlock it.
5. **Compaction clears the records:** run the `PostCompact` and `SessionStart` (`compact`) hook scripts by hand: the records are gone and the next edit blocks until the documents are read again.
6. **Modified document:** change `SSOT.md` after the read record: the next edit blocks until it is read again. Revert and confirm the diff is empty.
7. **Live check:** in a fresh session the agent tries to edit source first. It is blocked, reads the ledger, and the retry passes. Show the transcript lines.
8. **Registered and loaded:** show the settings lines, and the printed real stdin for one call of each event used, so the field names in the scripts are the ones the tool really sends.

A hook that is not registered, or whose command path is wrong, protects nothing. A wrong path with `|| exit 2` blocks every tool call, so test the registration before the session depends on it.

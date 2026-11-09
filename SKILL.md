---
name: herdr-new-project
description: Provisions a brand new project end-to-end via herdr — creates the directory, a new tab in the calling pane's own herdr workspace, a Claude Code agent started and renamed to match the project (before it's briefed, so session-name-based logging and labels agree from the first prompt), remote control enabled, then handed off to a project-provisioning skill (if you have one) and given the user's brief. Use when - 'new agent session', 'new project session', 'start a new project', 'create a new session for <project>'.
compatibility: Requires the `herdr` CLI (v0.8.2+) with a running herdr server, and Claude Code. A project-provisioning skill (called `new-project` below) is preferred, for repo/docs registration, and a session-end sync skill (`sync-session`) is preferred for logging; both are optional — without them the project is still created, renamed and briefed, just without that provisioning. Creates a new <projects-root>/<name> directory, herdr tab, and Claude Code agent per invocation.
license: MIT
metadata:
  author: herdr-new-project contributors
  version: 1.0.0
  tags: herdr, new-project, project-provisioning, agents, sessions, tabs, terminal
---

# Herdr New Project

## Starting a new project session

Trigger phrases: "new agent session", "new project session", "start a new project", "create a new session for X". This workflow provisions a brand new project end-to-end and hands it to its own Claude Code agent — it is a separate capability from pane labeling. It uses herdr primitives and, preferably, the `new-project` and `sync-session` skills.

**Defaults — assume these unless the user says otherwise:**

- **Same herdr space, as a new tab — not a new workspace.** "Same space" means literally that: a new *tab* inside the calling pane's own workspace (`$HERDR_WORKSPACE_ID`), never a separate `herdr workspace create`.
- **Same model.** Do not pass `--model`. A plain `claude` launch already uses whatever `/model` last saved as the default, so starting one with no model flag already matches "the model it's currently using."
- **Generic agent.** `--kind claude` with no subagent/persona name — just Claude Code, same as every other project session.
- **Background by default.** `--no-focus` — do not steal the user's focus from their current pane unless they asked to switch to it.
- **Remote control on by default.** Every new project session gets Claude Code's own `/remote-control` turned on (see Step 5) so it's reachable from the Claude phone/desktop app and claude.ai, not just this herdr pane — unless the user says they don't want that for this one.

**If the user's request already names the project and gives a brief, skip straight to execution. Otherwise ask for both** — a project name (slug-able) and a one- or two-sentence brief of what it's for. Do not guess a name or invent scope from nothing.

### Execution order

The rename MUST happen before the brief is sent — not after. The preferred `new-project` and `sync-session` skills name progress-log and index entries from the Claude Code session's name, so the session must already be named correctly before it starts doing anything that gets logged or labeled under that name.

1. **Create the directory.**
   ```bash
   mkdir -p <projects-root>/<name>
   ```

2. **Create a tab in the calling pane's own workspace**, cwd'd into the new directory — not a new workspace:
   ```bash
   herdr tab create --workspace "$HERDR_WORKSPACE_ID" --cwd <projects-root>/<name> --label <name> --no-focus
   ```
   Read `.result.root_pane.pane_id` from the response.

3. **Start the agent.** herdr agent names are stricter than project names — lowercase letters/digits/`-`/`_`, 1-32 chars, must start with a letter. Derive one from the project slug (lowercase, non-matching chars → `-`, collapse repeats, strip leading/trailing `-`; prefix `p-` if the result starts with a digit; truncate to 32).
   ```bash
   herdr agent start <sanitized-name> --kind claude --pane <pane-id>
   ```
   A first-ever launch in a new directory almost always blocks on Claude Code's folder-trust prompt (`agent_not_ready` from `agent start`, or `agent_status: blocked`). Confirm with `herdr agent read <id> --source recent-unwrapped --lines 40` — if it shows "Quick safety check" / "trust this folder", this is always a folder you just created, so answer it immediately:
   ```bash
   herdr agent send-keys <id> down enter
   ```
   Then poll `herdr agent get <id>` until `agent_status` is `idle`.

4. **Rename the session — before anything else is sent:**
   ```bash
   herdr agent prompt <id> "/rename <name>"
   ```
   **Do not use `--wait` here, and do not pass `--timeout` either** (this herdr rejects `--timeout` without `--wait`: `--timeout requires --wait`, and nothing is sent). `/rename` is a local command, not a real agent turn, so it never produces the `working`/`blocked` activity herdr's `--wait` gate looks for — a `--wait` call would reliably return `agent_prompt_stalled` (exit 1) within ~5s even though the rename succeeded instantly. Send it bare (it returns immediately) and confirm with a read instead of trusting the exit code:
   ```bash
   herdr agent read <id> --source visible --lines 40
   ```
   Look for `Session renamed to: <name>` and the pane's header line showing `<name>`.

5. **Enable remote control — after the rename, never before:**
   ```bash
   herdr agent prompt <id> "/remote-control"
   ```
   Same rule as `/rename`: it's a local command, not a real agent turn, so **do not use `--wait`** — send it bare and confirm with a read instead of trusting the exit code:
   ```bash
   herdr agent read <id> --source visible --lines 40
   ```
   Use `--source visible` — `recent-unwrapped` did not show the local-command output. Look for a line starting `/remote-control is active` followed by the claude.ai link for the session and keep that URL for the final report.

   **Order matters and is easy to get backwards.** Some setups have a global hook (not part of this skill) that watches prompts for `/remote-control` and, once seen, silently appends `-r` to the very next `/rename <name>` it sees, in any pane, because its "RC is active" flag is one shared sentinel file rather than per-pane state. Enable remote control *before* the rename and the new session ends up named `<name>-r`, which breaks the exact-name match the preferred `new-project`/`sync-session` skills' naming depends on. Doing the rename first and remote-control second sidesteps this: a bare `/remote-control` prompt never touches the session name.


6. **Send the brief, instructing the new agent to initialize itself via the `new-project` skill, then actually get moving:**
   ```bash
   herdr agent prompt <id> "You are initializing a new project called \"<name>\". Brief from the user: <brief>.

   First, run the new-project skill to provision this project (the local directory already exists at <projects-root>/<name>; this registers the project: repo, docs and instructions entry):
   /new-project <name> --tool claude --path <projects-root>/<name> --description \"<brief>\"

   Then get on with the brief above now — don't just stop after provisioning. If the brief gives you enough to act on, start: explore relevant existing systems/config, make a plan, begin implementing. If it is genuinely too vague to take a first concrete step (no clear starting point, config location, or first deliverable), don't sit idle — ask the user targeted clarifying questions about what to build or do first." --wait --timeout 300000
   ```
   If the `new-project` skill isn't installed, drop its paragraph from the prompt and send just the brief and the get-moving instruction; the rest of the workflow is unchanged.
   Use a generous timeout — `new-project` does real network I/O (repo creation and push, docs writes, a sync push), plus whatever exploration the agent does before it stops to report or ask.

7. **Report back**: tab ID, pane ID, agent name, the agent's final status (`herdr agent get <id>`), the remote-control URL from Step 5, and a one-line pointer to the tab so the user can jump to it.

### Known limitations

- Step 5's `/remote-control` is Claude Code's own account-level remote control (phone/desktop app, claude.ai) — unrelated to herdr's *named persistent herdr session* (`herdr --session <name>`), which this workflow still does not and cannot set up: that command launches an interactive TUI and cannot be scripted from here. If the user wants that too, tell them the exact command to run themselves.
- If you use a rename-suffix hook like the one described in Step 5, its sentinel is global, not per-pane, and may be cleared whenever *any* pane starts. Don't rely on it surviving between this workflow's Step 5 and Step 6/7 if the user is actively opening other panes at the same time — if in doubt, re-read the pane after Step 5 to confirm RC actually activated rather than assuming it did.
- If `herdr agent start` fails for a reason other than the trust dialog (e.g. a stale pane, a name collision), do not loop blindly — read the pane, diagnose, and only retry once the actual cause is understood.
- **If a tab already ended up in the wrong workspace** (e.g. an earlier run of this workflow, or a manual `workspace create`), fix it without restarting the agent: `herdr pane move <pane-id> --new-tab --workspace <correct-workspace-id> --label <name> --no-focus`. This preserves the live agent process and its state; it also auto-closes the now-empty source workspace when that was its only tab. Confirm with `herdr workspace get <old-id>` returning `workspace_not_found`.

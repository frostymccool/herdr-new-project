# herdr-new-project

A Claude Code skill that starts a brand new project in one step, inside herdr, the terminal multiplexer for coding agents.

## The problem

Starting a new project with an AI coding agent is a string of fiddly manual steps: make the folder, open a terminal tab, launch the agent, click through the folder-trust prompt, name the session, turn on remote control, then paste in what you want built. Do them in the wrong order and things quietly break. For example, naming the session after enabling remote control can leave you with a `-r` suffix on the name, and the project then gets logged under the wrong name. Run a dozen sessions and these steps get skipped or done inconsistently.

## What it does

Say "start a new project called X, it's for Y". The skill then:

1. creates the project directory;
2. opens a new tab for it in your current herdr workspace;
3. starts a Claude Code agent there and answers the folder-trust prompt;
4. renames the session to match the project, **before** anything else is sent;
5. turns on Claude Code remote control, so the session is reachable from your phone or claude.ai;
6. hands the agent your brief, optionally via a project-provisioning skill of yours, and tells it to start work or ask clarifying questions rather than sit idle;
7. reports back the tab, pane, agent name and remote-control link.

## Requirements

- the `herdr` CLI (v0.8.2 or newer) with a running herdr server
- Claude Code
- optional: a project-provisioning skill (`new-project`) and a session-end sync skill (`sync-session`); without them the project is still created, renamed and briefed

## Install

Copy or symlink this folder into `~/.claude/skills/herdr-new-project`.

## Reference

The full step-by-step workflow, defaults, gotchas and known limitations are in [SKILL.md](SKILL.md).

## License

MIT

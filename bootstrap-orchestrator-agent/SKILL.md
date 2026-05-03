---
name: bootstrap-orchestrator-agent
description: Scaffolds and validates a Claude Code orchestrator subagent that coordinates a fleet of per-repo project agents. Use when setting up multi-repo agent coordination, creating an orchestrator for an existing fleet of project agents, scaffolding a top-level coordinator that decomposes cross-repo work, or wiring up a multi-agent setup where one session delegates to per-repo specialists. Do not use unless at least one `<repo>-agent.md` already exists somewhere reachable — run `bootstrap-project-agent` on each target repo first.
---

# bootstrap-orchestrator-agent

## Overview

Scaffolds and validates a Claude Code subagent intended to run as the **main session agent** (via `claude --agent <name>` or the `agent` setting in `.claude/settings.json`), coordinating a fleet of per-repo project agents that the user has previously bootstrapped. Use it once per workspace when you have two or more `<repo>-agent.md` files and want a single coordinator that can dispatch work across them.

**Output is a [Claude Code subagent](https://code.claude.com/docs/en/sub-agents) intended for main-session use, not delegated use.** The orchestrator coordinates; the project agents do the per-repo work. For the project agent template this skill composes with, see [`bootstrap-project-agent`](../bootstrap-project-agent/). For a worked example of orchestrator output, see [`references/orchestrator-example.md`](references/orchestrator-example.md).

**Background:** subagents cannot spawn other subagents — that is why the orchestrator is *not* itself a delegated subagent but the **top-level session**. See [main-session agent docs](https://code.claude.com/docs/en/sub-agents#run-an-agent-as-the-main-session). For inter-session coordination across separate sessions, see [agent teams](https://code.claude.com/docs/en/agent-teams) — a different feature.

## Orchestrator template (defaults this skill enforces)

| Item | Value |
|------|--------|
| Path | `~/.claude/agents/<name>.md` (user-scope) or `<workspace>/.claude/agents/<name>.md` (workspace-scope) |
| `name` | user-chosen; default `orchestrator` |
| `model` | `opus` (synthesis is the orchestrator's value-add) unless user overrides |
| `tools` | **required:** `Agent(<agent1>, <agent2>, …), Read, Grep, Glob, Bash, TodoWrite` |
| **Forbidden in `tools`** | `Edit`, `Write`, `MultiEdit`, `NotebookEdit` (orchestrator coordinates; never edits) |
| `isolation` | **must NOT be set** — orchestrator runs in the operator's session, not a worktree |
| `memory` | omit, or `user` if you want cross-project orchestration learnings |
| Omit from frontmatter | `color`, `permissionMode`, `metadata`, `license`, `disallowedTools`, `background`, `effort` |
| `description` | Single-line plain YAML. Opens with `Use when` or `Use proactively` (typically `Use as the main session agent when…`). Includes negative trigger + ends with `See "When invoked" in the agent body for worked scenarios.` |

The `Agent(...)` allowlist is the load-bearing safety: it restricts the orchestrator to dispatching only to the specific project agents you registered. A bare `Agent` would let the orchestrator spawn any agent type it knows about — a sprawl risk.

Run `scripts/validate-orchestrator.sh` on the draft; **pass = zero warnings** before shipping.

## Procedure

1. **Discover existing project agents.** Walk likely locations to find candidate files:

```bash
# Project agents in a workspace-style layout (e.g. ~/projects/*/.claude/agents/*-agent.md)
find "$WORKSPACE_DIR" -mindepth 4 -maxdepth 4 -type f -path '*/.claude/agents/*-agent.md' 2>/dev/null
# Plus any explicit non-workspace repos the user names
find "$HOME" -maxdepth 4 -type f -path '*/.claude/agents/*-agent.md' 2>/dev/null
```

For each match, **parse the YAML frontmatter and extract the `name:` field.** That value — not the filename, not the directory name, not the repo name — is what `Agent(...)` will resolve against when the orchestrator dispatches. Common gotcha: a file at `<repo>/.claude/agents/<repo>-agent.md` may declare `name: <something-different>` inside its frontmatter (e.g. legacy agents from prior conventions). The frontmatter `name:` is the authoritative identifier; mismatches mean the orchestrator's `Agent(<inferred-name>)` will silently resolve to nothing.

Build a registry of `{ frontmatter_name, repo_path, description }` tuples. Where filename and frontmatter name disagree, surface the divergence to the user before continuing — they may want to rename the agent file to match, or accept the canonical frontmatter name.

If zero project agents are found: refuse and tell the user to run `bootstrap-project-agent` on each target repo first. The orchestrator without specialists is a coordinator with no team.

2. **Ask once** — (a) where to write the orchestrator (`~/.claude/agents/<name>.md` user-scope or `<dir>/.claude/agents/<name>.md` workspace-scope; default user-scope), (b) what to call it (default `orchestrator`), (c) any discovered project agents to *exclude* (default: include all), (d) model override (default `opus`).

3. **Generate** — feed the registry + answers into the meta-prompt below.

4. **Validate** — `scripts/validate-orchestrator.sh <path>`; fix until clean.

5. **Write** — `mkdir -p` the destination directory and write the file. Don't commit.

6. **Tell the operator how to invoke it:**

```bash
claude --agent <name>          # one-off session as the orchestrator
# or persist by adding to .claude/settings.json:
#   { "agent": "<name>" }      # makes every `claude` in this dir use the orchestrator
```

## Meta-prompt (feed registry + four answers)

```
You are emitting ONE Claude Code subagent file: an ORCHESTRATOR for a fleet of per-repo project agents.

Authoritative field semantics: https://code.claude.com/docs/en/sub-agents — do not invent fields.
Main-session-agent context: https://code.claude.com/docs/en/sub-agents#run-an-agent-as-the-main-session

INPUTS:
- registry: list of {agent_name, repo_path, description} for each discovered project agent
- name: orchestrator name (lowercase, hyphens)
- model: inherit | opus | sonnet | haiku (default opus)

Frontmatter (required):
- name: <chosen name>
- model: <chosen model>
- tools: Agent(<comma-separated agent_names from registry>), Read, Grep, Glob, Bash, TodoWrite
- description: single-line plain scalar; 50–1024 chars; start "Use when" or "Use as the main session agent when"; name 2–4 trigger scenarios using the registered project names; one negative trigger ("do not use for single-repo work — dispatch to the relevant project agent directly"); end exactly: See "When invoked" in the agent body for worked scenarios.

Forbidden in frontmatter:
- isolation (NEVER set — orchestrator runs in the operator's session, not a worktree)
- Edit, Write, MultiEdit, NotebookEdit in tools (orchestrator coordinates; does not edit)
- color, permissionMode, metadata, license, disallowedTools, background, effort

Body — use these headings in order:

# <Title>

(One paragraph: what this orchestrator coordinates and why it exists in operator vocabulary. One paragraph: "You run as the main session agent. The project agents run as subagents you invoke via the Agent tool. Subagents cannot spawn further subagents, so any cross-repo coordination must happen here.")

## The specialists

(One subsection per registered project agent, drawn verbatim from the registry — agent name, repo path, one-line domain summary from the agent's own description, vocabulary that should route to it.)

## When invoked

(2–4 bold-led prose-bullet scenarios. Anchor on cross-repo work, dependency rippling, parallel tasks. Include one negative scenario: "single-repo work goes directly to the specialist, not through this orchestrator".)

## Dispatch protocol

(A self-briefing the orchestrator gives itself: each dispatch is self-contained; specialists do not see your conversation; pack 1-sentence task + why-now + inputs + acceptance criteria + out-of-scope + reporting requirements into every dispatch prompt.)

## Parallel vs. sequential dispatch

(Parallel = single message, multiple Agent calls, when tasks are independent. Sequential when one specialist's output is the next specialist's input. Name TodoWrite as the tracking tool for in-flight dispatches.)

## Handling specialist responses

(Three sub-cases: clarifying question — answer if inferable, otherwise surface to operator verbatim. Refusal — load-bearing, do not re-frame. Partial work / blocker — route to the right owner; do not silently re-dispatch.)

## Return format to the operator

Your final report must include:
1. What was done in each repo (commands run, tests run with results, files changed with absolute paths, branch / PR links if any).
2. Anything the operator must do next that no specialist could do (homelab pulls, one-way doors, credentials, cross-repo decisions).
3. Any contradictions surfaced between specialists, with how they were reconciled.
4. Match every involved repo's output discipline (no co-author tags / emojis / hyperbole if any specialist's CLAUDE.md says so — strictest wins).

Constraints: body 1500–15000 chars; no absolute paths leaking the operator's home directory beyond what comes from registry repo paths; do not invent specialists not in the registry; do not refer to the orchestrator as a "manager" or specialists as "reports" — they are peers being asked to do focused work; if any specialist's CLAUDE.md forbids emojis/co-author tags, the orchestrator body itself follows that. Regenerate at most twice if validation fails; then stop and report failures.
```

## Pitfalls (keep short)

| Wrong move | Right move |
|------------|------------|
| `tools: Agent` (unrestricted) | `tools: Agent(<specific-names>)` — restrict to your registered fleet |
| Setting `isolation: worktree` on the orchestrator | Never — the orchestrator IS the operator's session, not a worker |
| Including `Edit` / `Write` in `tools` | Drop them — orchestrators coordinate; specialists edit |
| Calling the orchestrator a "manager" or specialists "reports" | They are peers. The orchestrator dispatches focused work, doesn't supervise |
| Paraphrasing a specialist's report | Quote verbatim. Integration is the orchestrator's value, not retelling |
| Re-dispatching when a specialist refuses out-of-scope work | The refusal is load-bearing. Surface to the operator with the cited reason |
| Inventing a new specialist not in the registry | Refuse. Tell the operator to run `bootstrap-project-agent` on the new repo first |
| Inferring agent names from filenames or repo names | Parse the `name:` field from each agent's frontmatter. `Agent(...)` resolves against the frontmatter name; mismatches silently fail to dispatch |

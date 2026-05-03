---
name: bootstrap-orchestrator-agent
description: Scaffolds and validates a Claude Code orchestrator subagent that coordinates a fleet of per-repo project agents. Use when setting up multi-repo agent coordination, creating an orchestrator for an existing fleet of project agents, scaffolding a top-level coordinator that decomposes cross-repo work, or wiring up a multi-agent setup where one session delegates to per-repo specialists. Do not use unless at least one `<repo>-agent.md` already exists somewhere reachable — run `bootstrap-project-agent` on each target repo first.
---

# bootstrap-orchestrator-agent

## Overview

Scaffolds and validates a Claude Code subagent intended to run as the **main session agent** (via `claude --agent <name>` or the `agent` setting in `.claude/settings.json`), coordinating a fleet of per-repo project agents that the user has previously bootstrapped. Use it when you have two or more `<repo>-agent.md` files and want a coordinator that can dispatch work across them. Run it again to create *additional* orchestrators targeting different specialist subsets — e.g. one orchestrator for the frontend stack and another for the data pipeline.

Because Claude Code's subagent discovery walks **up** from CWD only, the skill also creates user-scope symlinks (`~/.claude/agents/<specialist>.md` → per-repo file) so the orchestrator can resolve specialists from any working directory. Symlinking is the default; opt out per-run if you have a different setup.

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

2. **Select which specialists this orchestrator should include.** Show the discovered registry to the user as a numbered list (each entry: agent name + repo path + first ~80 chars of description). Ask: *"Which of these should this orchestrator coordinate? (default: all; or comma-separated indices, e.g. `1,3,4`)"*.

   Why this is a separate step: the user may want **multiple orchestrators** on one machine, each targeting a different specialist subset (e.g. one orchestrator for the frontend stack, one for the data pipeline). Defaulting to "all" makes the simple case one keystroke; subset selection makes the customized case explicit.

3. **Ask once** — (a) where to write the orchestrator (`~/.claude/agents/<name>.md` user-scope or `<dir>/.claude/agents/<name>.md` workspace-scope; default user-scope), (b) what to call it (default `orchestrator` for a single orchestrator; for multiples, suggest a scoped name like `frontend-orchestrator`), (c) model override (default `opus`), (d) make selected specialists discoverable from any CWD via user-scope symlinks (default: yes — see step 7).

4. **Generate** — feed the *selected* subset of the registry + answers into the meta-prompt below. The orchestrator's `Agent(...)` allowlist contains only the selected specialists' frontmatter names.

5. **Validate** — `scripts/validate-orchestrator.sh <path>`; fix until clean.

6. **Write the orchestrator file** — `mkdir -p` the destination directory and write. Don't commit.

7. **Make specialists discoverable from any CWD** (if user accepted in step 3d). Claude Code's subagent discovery walks **up** from CWD, never **down** into sibling repos — so a per-repo project agent at `<repo>/.claude/agents/<repo>-agent.md` is invisible to an orchestrator session running from anywhere outside that repo. To bridge the gap without moving the canonical files, create a user-scope symlink for each selected specialist:

```bash
for agent in "${selected_agents[@]}"; do
  src="<repo_path>/.claude/agents/<frontmatter_name>.md"   # canonical, per-repo
  dst="$HOME/.claude/agents/<frontmatter_name>.md"          # user-scope visibility
  if [ -L "$dst" ] && [ "$(readlink "$dst")" = "$src" ]; then
    : # already correct, skip silently
  elif [ -e "$dst" ]; then
    echo "WARN: $dst already exists (and points elsewhere or is a real file). Refusing to overwrite. Resolve manually."
  else
    ln -s "$src" "$dst"
  fi
done
```

The skill must NOT silently overwrite an existing file or symlink with a different target — surface the conflict and let the user decide. Multiple orchestrators sharing the same specialist (e.g. `alpha-agent` is in two orchestrators' selections) reuse the same symlink, which is correct: the symlink keys on the agent name, not on which orchestrator references it.

8. **Tell the operator how to invoke and clean up:**

```bash
# Launch:
claude --agent <name>          # one-off session as the orchestrator
# or persist for the current dir by adding to .claude/settings.json:
#   { "agent": "<name>" }

# To remove the symlinks created in step 7 (if uninstalling this orchestrator):
find "$HOME/.claude/agents" -type l -lname '*/.claude/agents/*-agent.md' -delete
# (or just `rm` the specific ones you no longer want)
```

Surface a one-line summary of what was created: the orchestrator file path, the selected specialists, and the symlinks (if any).

## Meta-prompt (feed selected-subset registry + answers)

```
You are emitting ONE Claude Code subagent file: an ORCHESTRATOR for a fleet of per-repo project agents.

Authoritative field semantics: https://code.claude.com/docs/en/sub-agents — do not invent fields.
Main-session-agent context: https://code.claude.com/docs/en/sub-agents#run-an-agent-as-the-main-session

INPUTS:
- registry: list of {agent_name, repo_path, description} for the SELECTED subset of project agents (not all discovered ones — only the ones the user chose to include in this orchestrator)
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
| Defaulting to "include every discovered specialist" without asking | Ask explicitly. The user may want multiple orchestrators targeting different subsets (frontend stack vs data pipeline). Defaulting to all is fine; *not asking* is the bug |
| Silently overwriting an existing symlink at `~/.claude/agents/<name>.md` that points elsewhere | Refuse and surface the conflict. The user may have a different orchestrator pointing the same name at a different repo — clobbering breaks their other setup |
| Generating an orchestrator without creating the symlinks | `Agent(<specialist-name>)` will silently fail to resolve at dispatch time because Claude Code's discovery walks UP from CWD, not DOWN into sibling repos. Either symlink (default), document the user-scope-only path, or refuse to ship |

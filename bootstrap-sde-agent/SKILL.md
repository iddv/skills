---
name: bootstrap-sde-agent
description: Use when bootstrapping, scaffolding, or creating a project-scoped SDE-style Claude Code subagent for a repository. Triggers include phrases like "create an agent for this repo", "set up a project SDE", "make an agent that knows this codebase", "bootstrap a team agent", or onboarding a new project into an orchestrator that delegates to per-repo subagents. Use proactively when the current working directory is a git repository with steering context (CLAUDE.md, AGENTS.md, .kiro/) but no comprehensive SDE-style agent at .claude/agents/.
license: MIT
metadata:
  author: iddv
  version: "0.1.0"
---

# bootstrap-sde-agent

## Overview

Generates a project-scoped, SDE-style Claude Code subagent for the current repository by inspecting the repo's existing context (CLAUDE.md, AGENTS.md, `.kiro/steering/`, README, manifest, structure) and writing `.claude/agents/<repo>-sde.md`.

**Core principle: the generated agent is a thin operational layer over the repo's own steering documents, not a parallel system that drifts from them.**

The skill exists because unaided agents reliably make three specific mistakes when asked to write a project SDE agent:
1. They omit `isolation: worktree` and rationalize it as "an orchestrator concern" — it is a documented frontmatter field.
2. They open the `description` with the agent's *role* rather than the official trigger pattern (`Use this agent when…` / `use proactively`), degrading auto-delegation.
3. They duplicate the rules from CLAUDE.md into the agent body, where they immediately drift.

## When to use

- Onboarding a new repo into a multi-repo orchestrator setup
- The user explicitly asks to "create an agent for this repo" / "bootstrap a project SDE" / "scaffold a team agent"
- A repo has rich CLAUDE.md / `.kiro/` context but no `.claude/agents/` yet
- You are about to hand-write a per-repo specialist subagent — use this instead

**Do NOT use when:**
- The user wants a task-specific agent (code-reviewer, test-writer, security-reviewer) — those belong in different templates
- The repo has no CLAUDE.md, AGENTS.md, README, or other steering — refuse and ask the user to write one first; the agent body would be hollow and drift on first contact

## Quick reference: defaults the skill enforces

| Field | Default | Why this default |
|-------|---------|------------------|
| `name` | `<reponame>-sde` | Stable, project-scoped, easy to @-mention |
| `description` opener | `Use this agent when…` | Anthropic's official auto-delegation trigger pattern |
| `model` | `inherit` | Orchestrator picks the tier per task; agent does not second-guess |
| `tools` | omitted (inherit all) | SDE work uses many tools; allowlists go stale; restrict via settings.json instead |
| `isolation` | `worktree` | User-stated requirement; closes the most common rationalization gap |
| `memory` | `project` | Lets agent accumulate repo-specific learnings; survives `git clone` |
| `color` | omitted | Cosmetic; not in agentskills.io spec |
| Body opens with | "First moves, every invocation" pointing at CLAUDE.md / `.kiro/steering/` | Defer; do not duplicate |
| Body includes | "When invoked" prose-bullet scenarios | Anchors auto-delegation; mirrors `description` triggers |
| Body ends with | "Return format to the orchestrator" | Structured handoff; subagent runs in a separate session and the parent only sees the return value |

## Workflow

See [workflow.md](workflow.md) for the full step-by-step procedure including the meta-prompt used to generate the agent body. Summary:

1. **Verify environment**: `git rev-parse --show-toplevel`, confirm cwd is a repo root, refuse if `.claude/agents/<repo>-sde.md` already exists (offer `--force` only on explicit user request).
2. **Inspect**: read CLAUDE.md, AGENTS.md, `.kiro/steering/*.md`, `.kiro/specs/*/spec.json`, README*, manifest (`package.json` / `pyproject.toml` / `go.mod` / `Cargo.toml`), top-level `ls -F`. Capture project vocabulary (proper nouns, service names, "the X" phrases).
3. **Confirm**: ask the user three questions — (a) any scope/exclusions, (b) any feedback or preferences for this agent, (c) override default model from `inherit`?
4. **Generate**: feed the inspection bundle into the meta-prompt in `workflow.md`. Output is markdown frontmatter + body.
5. **Post-process**: enforce defaults from the table above. Verify the body opens with the "First moves" / read-CLAUDE.md instruction. Verify it ends with a "Return format" section.
6. **Validate**: run `scripts/validate-agent.sh <path>`. Must exit 0 with zero warnings.
7. **Write**: `.claude/agents/<repo>-sde.md`.
8. **Show**: `cat` the file back. Briefly explain choices made (what inspection surfaced, where defaults were overridden, what the orchestrator should know about invoking this agent).

## Common mistakes

| Mistake | Fix |
|---------|-----|
| Omitting `isolation: worktree` claiming "that's an orchestrator concern" | It is a frontmatter field. Set it. |
| Description starting with the agent's role ("SDE on the X team") | Lead with `Use this agent when…`. Routing is on triggers, not titles. |
| Duplicating the rules from CLAUDE.md into the body | Defer instead. CLAUDE.md is the source of truth; the agent points at it and re-reads it every invocation. |
| Adding `tools` allowlist for "least privilege" | SDE work uses many tools; allowlists are brittle. Inherit. Block dangerous bash via `settings.json` permissions. |
| Forgetting to brief the agent that it does not see parent conversation | One sentence: "You are invoked by an orchestrator in a different session. Treat the prompt you receive as the full statement of work." |
| Skipping the "Return format" section | The orchestrator only sees the return value. Specify: what was done, what was verified, what's next, surfaced one-way doors, files changed. |
| Generating without inspecting `.kiro/` if present | `.kiro/steering/*.md` and `.kiro/specs/` carry load-bearing project intent. Skip them and the agent will reinvent decisions the project has already made. |
| Hard-coding absolute paths from one machine | Use repo-relative paths in the body. The agent runs wherever the orchestrator clones. |

## Anti-patterns from baseline testing

Without this skill, a competent subagent asked to write an SDE agent for `parallax` produced this verbatim rationalization:

> "Worktree isolation: No. Subagent definitions don't configure worktrees — that's an orchestrator concern."

This is the most common failure mode. The user's whole reason for asking for an SDE agent is *isolation*: they want the SDE to never touch the working tree of the parent session. Burying that decision in the orchestrator's per-call parameters means it gets forgotten, fights bug #27881 (worktree-CWD drift after compaction), and lets the agent commit directly to the protected branch. **`isolation: worktree` in the agent's frontmatter is the correct default.**

The same baseline started its description with `"SDE on the parallax team. Use this agent for any work touching the parallax repo at /home/iddv/workspace/parallax — the Fleet of long-running intern agents…"`. The role-first opener is a trigger-degraded form. Lead with `Use this agent when` and treat the role as something the body establishes.

## Validation

`scripts/validate-agent.sh <path>` checks:
- YAML frontmatter is well-formed
- `name` matches `<repo>-sde` and the file's path
- `description` starts with `Use this agent when` (or `Use proactively`)
- `isolation: worktree` is present
- `memory: project` is present
- `model` is one of `inherit | sonnet | opus | haiku` (or omitted)
- Body contains a "When invoked" section
- Body contains a "Return" section
- Body length is between 500 and 10,000 characters

Generated agents must pass with zero warnings.

## Why this skill exists

You can hand-write a project SDE agent and it will probably be 80% correct. The remaining 20% — worktree isolation, trigger-shaped description, deferral to CLAUDE.md, structured return — are exactly the parts that make the agent reliably *invoked* and reliably *safe*. Those are also the parts an unaided agent rationalizes away first under any time pressure. This skill makes them defaults.

---
name: bootstrap-sde-agent
description: Use when bootstrapping or scaffolding a project-scoped SDE-style Claude Code subagent for a repository, or onboarding a new repo into a multi-repo orchestrator. Triggers include phrases like "create an agent for this repo", "set up a project SDE", "make an agent that knows this codebase", "bootstrap a team agent". Use proactively when starting work in a git repository that has steering context (CLAUDE.md, AGENTS.md, .kiro/) but no .claude/agents/ directory yet.
---

# bootstrap-sde-agent

**REQUIRED BACKGROUND:** Generated agents default to `isolation: worktree`. If you don't know what that means, read `superpowers:using-git-worktrees` first.

**Announce at start:** "I'm using the bootstrap-sde-agent skill to scaffold a project SDE for `<repo>`."

## What this generates

A **Claude Code subagent file** at `<repo>/.claude/agents/<repo>-sde.md`. This is NOT a SKILL.md — both file types use YAML frontmatter, but they are governed by different specifications, live in different directories, and behave differently. See [`references/subagent-format.md`](references/subagent-format.md) for the field-by-field reference, and [code.claude.com/docs/en/sub-agents](https://code.claude.com/docs/en/sub-agents) for the authoritative spec.

## Overview

The generated agent is a **thin operational layer over the repo's own steering documents (CLAUDE.md, AGENTS.md, `.kiro/steering/`)**, not a parallel system that drifts from them. It encodes the project's vocabulary, conventions, and non-negotiable rules so an orchestrator can delegate work into the repo without re-briefing each time.

## When to use

- Onboarding a new repo into a multi-repo orchestrator setup
- The user explicitly asks to create / scaffold / bootstrap a project SDE agent
- A repo has rich CLAUDE.md / `.kiro/` context but no `.claude/agents/` yet

**Do NOT use when:**
- The user wants a task-specific agent (code-reviewer, test-writer) — different template
- The repo has no CLAUDE.md, AGENTS.md, README, or other steering — refuse and ask the user to write one first; the agent body would be hollow

## Quick reference: defaults the skill enforces

| Field | Default | Why |
|-------|---------|-----|
| `name` | `<reponame>-sde` | Stable, project-scoped, easy to @-mention |
| `description` opener | `Use when…` | Anthropic's auto-delegation trigger pattern |
| `model` | `inherit` | Orchestrator picks tier per task |
| `tools` | omitted | SDE work uses many tools; allowlists go stale; restrict via settings.json |
| `isolation` | `worktree` | User requirement; closes the most common rationalization gap |
| `memory` | `project` | Lets agent accumulate repo-specific learnings; survives clone |
| `color` / `permissionMode` | omitted | Not in agentskills.io spec |
| Body opens with | "First moves" pointing at CLAUDE.md / `.kiro/steering/` | Defer; do not duplicate |
| Body includes | `## When invoked` (prose-bullet scenarios) | Anchors auto-delegation |
| Body ends with | `## Return format to the orchestrator` | Structured handoff |

## Workflow

See [workflow.md](workflow.md) for the step-by-step procedure and the meta-prompt. Summary:

1. Verify cwd is a git repo root (handle worktrees correctly via `git rev-parse --git-common-dir`)
2. Inspect: CLAUDE.md, AGENTS.md, `.kiro/steering/*.md`, README*, manifest, top-level structure
3. Confirm 3 short questions with user (scope, feedback, model override)
4. Generate via the meta-prompt
5. Validate via `scripts/validate-agent.sh`
6. Write to `<repo>/.claude/agents/<repo>-sde.md`
7. `cat` the file back; explain decisions

## Red flags — stop and reconsider

These rationalizations come from the RED-phase baseline (an unaided agent asked to write an SDE agent for a real Python project). All produce subtly wrong agents. Treat them as automatic stops.

| Rationalization | Reality |
|-----------------|---------|
| "Subagent definitions don't configure worktrees — that's an orchestrator concern." | `isolation` is a documented frontmatter field. Setting it makes worktree isolation the default for every invocation, which is what the user wants. |
| "I'll inline CLAUDE.md so the agent always has it." | The agent re-reads CLAUDE.md every invocation. Inlining causes drift the moment CLAUDE.md is edited. Defer; don't duplicate. |
| "Description should describe the agent's role and expertise." | Description is a routing rule, not a job title. Open with `Use when…` and list trigger keywords. |
| "I'll tighten `tools` for least-privilege." | SDE work uses many tools, including ones not yet announced. Allowlists silently break on new tool releases. Inherit; restrict via `settings.json`. |
| "I'll add memory later if needed." | `memory: project` is cheap, survives `git clone`, and lets the agent accumulate codebase learnings. Default it on. |
| "The agent can figure out it's a subagent from context." | It can't — it doesn't see the parent's conversation. State explicitly: "You are invoked by an orchestrator in a different session. The prompt is the full statement of work." |
| "Skip the Return section — the agent will report naturally." | The orchestrator only sees the return value. Without a structured Return section, every invocation produces a different shape. |

## Common mistakes

| Mistake | Fix |
|---------|-----|
| Hard-coded absolute paths in the body | Use repo-relative paths; the agent runs wherever the orchestrator clones |
| Generating one SDE per service in a monorepo | One SDE per repo. Anthropic warns specialist sprawl dilutes auto-delegation |
| Skipping `.kiro/specs/` inspection because "specs aren't operational" | Active specs tell the agent what's in flight. List names; don't load full bodies |
| Paraphrasing CLAUDE.md rules into the agent body | Quote verbatim. Paraphrase weakens; verbatim survives drift |

## Validation

`scripts/validate-agent.sh <path>` enforces every default in the Quick reference table. Generated agents must pass with zero warnings. See [`references/rationale.md`](references/rationale.md) for *why* each check exists.

## References

- [`references/subagent-format.md`](references/subagent-format.md) — field-by-field reference for the agent file format this skill produces, plus the skill-vs-subagent distinction
- [`references/rationale.md`](references/rationale.md) — why each enforced default exists; RED-phase baseline narrative
- [`workflow.md`](workflow.md) — full step-by-step procedure with the meta-prompt
- [`examples/example-agent.md`](examples/example-agent.md) — illustrative output, with framing
- **Official:** Claude Code subagent spec — <https://code.claude.com/docs/en/sub-agents>
- **Official:** Agent Skills spec (governs *this* file) — <https://agentskills.io/specification>
- **Official:** Worktrees (relevant because we default to `isolation: worktree`) — <https://code.claude.com/docs/en/worktrees>
- Anthropic's blog on subagents — <https://claude.com/blog/subagents-in-claude-code>

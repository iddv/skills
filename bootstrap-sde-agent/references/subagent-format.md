# Claude Code subagent file format — quick reference

The artifact this skill produces is a **Claude Code subagent file**, not a skill. They look similar (both are markdown with YAML frontmatter) but are governed by different specifications, live in different directories, and are loaded differently.

| Aspect | Subagent (what this skill produces) | Skill (e.g. this very file) |
|--------|-------------------------------------|------------------------------|
| Lives in | `<repo>/.claude/agents/` (project) or `~/.claude/agents/` (user) | `<repo>/.claude/skills/` or `~/.claude/skills/` or a plugin's `skills/` |
| Loaded when | Claude decides to delegate based on the `description` field, or invoked explicitly via `@agent-<name>` | Auto-loaded into the active conversation when a prompt matches its `description` |
| Spec | [Claude Code Subagents](https://code.claude.com/docs/en/sub-agents) | [Agent Skills](https://agentskills.io/specification) |
| Body becomes | The subagent's full system prompt for its own context window | Reference content injected into the parent conversation |
| Can use frontmatter fields like | `model`, `tools`, `isolation`, `memory`, `color`, `mcpServers`, `hooks` | `license`, `compatibility`, `metadata`, `allowed-tools` |

**The two specs share `name` and `description`. Almost everything else is different.**

## Subagent frontmatter — official fields

Quoted from <https://code.claude.com/docs/en/sub-agents> (verify there for the current canonical version).

| Field | Required | Purpose |
|-------|----------|---------|
| `name` | Yes | Unique identifier; lowercase letters, numbers, hyphens. Used in `@agent-<name>` invocation. |
| `description` | Yes | When Claude should delegate to this subagent. **Load-bearing for auto-routing.** |
| `tools` | No | Tools the subagent can use. Inherits all tools if omitted. |
| `disallowedTools` | No | Tools to deny, removed from inherited or specified list. |
| `model` | No | `sonnet` / `opus` / `haiku` / a full model ID / `inherit`. Defaults to `inherit`. |
| `permissionMode` | No | `default` / `acceptEdits` / `auto` / `dontAsk` / `bypassPermissions` / `plan`. Ignored for plugin subagents. |
| `maxTurns` | No | Maximum agentic turns before the subagent stops. |
| `skills` | No | Skills to load into the subagent's context at startup. **Subagents do NOT inherit skills from the parent conversation.** |
| `mcpServers` | No | MCP servers available to this subagent. Ignored for plugin subagents. |
| `hooks` | No | Lifecycle hooks scoped to this subagent. Ignored for plugin subagents. |
| `memory` | No | `user` / `project` / `local`. Enables cross-session learning. |
| `background` | No | `true` to always run as a background task. Default: `false`. |
| `effort` | No | `low` / `medium` / `high` / `xhigh` / `max`. Overrides session effort level. |
| `isolation` | No | Set to `worktree` to run in a temporary git worktree (isolated copy of the repository, auto-cleaned if no changes). |
| `color` | No | `red` / `blue` / `green` / `yellow` / `purple` / `orange` / `pink` / `cyan`. Display color in task list and transcript. |
| `initialPrompt` | No | Auto-submitted as the first user turn when this agent runs as the main session agent (via `--agent` or the `agent` setting). |

## What `bootstrap-sde-agent` enforces

| Field | This skill's default | Why |
|-------|----------------------|-----|
| `name` | `<repo>-sde` | Project-scoped, easy to @-mention |
| `description` | "Use when…" + concrete triggers + canonical pointer | Load-bearing for routing |
| `model` | `inherit` | Orchestrator picks tier per task |
| `isolation` | `worktree` | Closes the most common rationalization gap |
| `memory` | `project` | Lets the agent accumulate codebase-specific learnings; survives clone |
| `tools`, `color`, `permissionMode`, `mcpServers`, `hooks`, `background`, `effort`, `disallowedTools`, `initialPrompt`, `maxTurns`, `skills` | omitted | Out of scope for the SDE pattern; restrict via `settings.json` if needed |

## Memory directories

When `memory: project` is set, the subagent reads/writes:

```
<repo>/.claude/agent-memory/<name-of-agent>/
```

This directory is in the repo and travels with `git clone`. The `local` variant uses `.claude/agent-memory-local/<name>/` and is gitignored by convention. The `user` variant uses `~/.claude/agent-memory/<name>/` and is shared across all projects.

## Discovery & precedence

Claude resolves subagents in this order (highest priority first):

1. Managed settings (organization)
2. `--agents` CLI flag (current session)
3. `<repo>/.claude/agents/` (project)
4. `~/.claude/agents/` (user)
5. Plugin's `agents/` directory

Same name in higher-priority scope wins. Generated SDE agents go in `<repo>/.claude/agents/` so they travel with the repo.

## Invocation

- **Automatic** — Claude reads the `description` field and delegates when a task matches.
- **Explicit** — `@agent-<name>` (or just mentioning the agent's name in natural language).
- **Session-wide** — `claude --agent <name>` makes the entire session use this subagent's system prompt, tools, and model.

## Critical isolation properties

- Subagents do NOT see the parent's conversation. The system prompt + the per-call prompt + cwd/env are the only inputs.
- Subagents return only their final summary to the parent — verbose tool output stays in the subagent's own context.
- Subagents cannot spawn other subagents. The orchestrator must be the top-level session.

## Source documents

- **Subagents (the artifact this skill produces)** — <https://code.claude.com/docs/en/sub-agents>
- **Skills (what this skill itself is)** — <https://agentskills.io/specification>
- **Agent teams (different from subagents)** — <https://code.claude.com/docs/en/agent-teams>
- **Worktrees (relevant because we default to `isolation: worktree`)** — <https://code.claude.com/docs/en/worktrees>
- **Anthropic blog: How and when to use subagents** — <https://claude.com/blog/subagents-in-claude-code>

For the complete frontmatter reference and authoritative behavior, always check <https://code.claude.com/docs/en/sub-agents> — the spec evolves.

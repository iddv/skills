---
name: bootstrap-sde-agent
description: Scaffolds and validates a project-scoped Claude Code SDE subagent at `<repo>/.claude/agents/<repo>-sde.md` from a repo's existing steering. Use when creating an SDE for this repo, onboarding a new repo into a multi-repo orchestrator, scaffolding a per-team subagent, generating a project-aware agent that knows this codebase, or bootstrapping the agent surface for an existing project. Do not use for task-only agents (code-reviewer, test-writer, etc.) or empty repos that have no CLAUDE.md, AGENTS.md, .kiro/steering/, or README — add steering first.
---

# bootstrap-sde-agent

## Overview

Scaffolds and validates a Claude Code SDE-style subagent for one repo at `<repo>/.claude/agents/<repo>-sde.md`, with the SDE template baked in (`isolation: worktree`, `memory: project`, deferral to the repo's own CLAUDE.md). Use it once per repo when onboarding a project into a multi-repo orchestrator setup, or any time you would otherwise hand-write a per-repo specialist subagent.

**Output is a [Claude Code subagent](https://code.claude.com/docs/en/sub-agents), not a Skill** ([different spec](https://agentskills.io/specification)). For field meanings, routing, memory paths, and edge cases, read the official subagent doc — this skill only pins the **SDE template** and the **procedure**. For a worked example of a passing output, see [`examples/example-agent.md`](examples/example-agent.md).

**Background:** default `isolation: worktree` — if unclear, read `superpowers:using-git-worktrees` and [Worktrees](https://code.claude.com/docs/en/worktrees). [Agent teams](https://code.claude.com/docs/en/agent-teams) are a different feature.

## SDE template (defaults not spelled out in one place upstream)

| Item | Value |
|------|--------|
| Path | `<repo>/.claude/agents/<basename>-sde.md` |
| `name` | `<basename>-sde` (lowercase, hyphens) |
| `model` | `inherit` unless user overrides |
| `isolation` | `worktree` |
| `memory` | `project` |
| Omit from frontmatter | `tools`, `color`, `permissionMode`, `metadata`, `license` (inherit tools; tune permissions in [settings](https://code.claude.com/en/settings); optional UI fields are template noise) |
| `description` | Single-line plain YAML. Opens with `Use when` or `Use proactively`. Includes negative trigger + ends with `See "When invoked" in the agent body for worked scenarios.` — see [automatic delegation](https://code.claude.com/docs/en/sub-agents#understand-automatic-delegation) |

Run `scripts/validate-agent.sh` on the draft; **pass = zero warnings** before shipping.

## Procedure

1. **Resolve `<repo>`** — must be a git checkout. Prefer main checkout when inside a linked worktree (do **not** rely on `--show-toplevel` alone):

```bash
git rev-parse --is-inside-work-tree >/dev/null 2>&1 || { echo "Not a git repo"; exit 1; }
COMMON_DIR="$(git rev-parse --git-common-dir)"
if [ "$COMMON_DIR" = ".git" ] || [ "$COMMON_DIR" = "$(git rev-parse --git-dir)" ]; then
  REPO_ROOT="$(git rev-parse --show-toplevel)"
else
  REPO_ROOT="$(cd "$COMMON_DIR/.." && pwd)"
fi
```

Refuse if `.claude/agents/<basename>-sde.md` already exists unless `--force`. Refuse if there is no steering (`CLAUDE.md` / `AGENTS.md` / README / `.kiro/` all absent).

2. **Inspect (read-only)** — gather text for generation: `CLAUDE.md` (and nested up to depth 3 if your rules say so), `AGENTS.md`, `.kiro/steering/*.md`, `.kiro/specs/*/spec.json` (names only), README + any `CONTRIBUTING`/`DEVELOPMENT`/`HACKING`, one top-level manifest (`package.json` / `pyproject.toml` / `go.mod` / `Cargo.toml` / …), `ls` of repo root, recent `git log --oneline -20`, `.claude/settings*.json` if present. Extract proper nouns for the `description` triggers.

3. **Ask once** — (a) paths not to touch, (b) free-text prefs, (c) model override (`inherit` default). If no answer, use defaults.

4. **Generate** — use the meta-prompt below + official doc for anything not listed.

5. **Validate** — `scripts/validate-agent.sh <path>`; fix until clean.

6. **Write** — `mkdir -p "$REPO_ROOT/.claude/agents"` then write the file; do not commit for the user.

## Meta-prompt (feed inspection bundle + three answers)

```
You are emitting ONE Claude Code subagent file for repo <basename>, path .claude/agents/<basename>-sde.md.

Authoritative field semantics: https://code.claude.com/docs/en/sub-agents — do not invent fields.

Frontmatter (required): name <basename>-sde; model inherit OR user override; isolation worktree; memory project.
Omit: tools, color, permissionMode, metadata, license.
description: single-line plain scalar; 50–1024 chars; start Use when or Use proactively; project vocabulary; one "do not invoke for…"; end exactly: See "When invoked" in the agent body for worked scenarios.

Body — use these headings in order:
# <Title> SDE
(two short paragraphs: repo purpose from README/CLAUDE; orchestrator isolation — no parent chat, prompt is full SOW.)

## First moves, every invocation
(numbered repo-relative paths: CLAUDE.md, AGENTS.md, .kiro/steering if any, etc.)

## What you are working with
(layout, commands from manifest; if .kiro/specs exists list active spec names only)

## How work happens here
(quote CLAUDE.md rules verbatim — no paraphrase; omit if silent on a topic)

## When invoked
(2–4 bold-led scenario bullets for routing)

## Engineering conventions
(stack, tests, lint; optional "Do not modify" from user exclusions)

## Return format to the orchestrator
(numbered: what changed/verified; handoffs; one-way doors; match repo output discipline)

Constraints: body 500–10000 chars; repo-relative paths only; no invented rules; if CLAUDE forbids emojis/co-author, obey in body. Regenerate at most twice if validation fails; then stop and report failures.
```

## Pitfalls (keep short)

| Wrong move | Right move |
|------------|------------|
| `isolation` "is orchestration only" | It is official frontmatter; SDE uses `worktree`. |
| Inline full CLAUDE.md in body | Defer to files; avoids drift. |
| One SDE per package in a monorepo | One per repo; see [blog](https://claude.com/blog/subagents-in-claude-code) on too many specialists. |
| Paraphrase CLAUDE rules | Quote verbatim. |

# bootstrap-sde-agent — Workflow

Detailed procedure for generating a project-scoped SDE subagent. Read after `SKILL.md`.

## Step 1 — Verify environment

```bash
# Detect a git repo at all
git rev-parse --is-inside-work-tree >/dev/null 2>&1 || {
  echo "Not a git repo. Refusing."; exit 1
}

# Find the *main* repo root, even when invoked from inside a worktree.
# git-common-dir returns <main-repo>/.git when inside a worktree, or .git when at main.
COMMON_DIR="$(git rev-parse --git-common-dir)"
if [ "$COMMON_DIR" = ".git" ] || [ "$COMMON_DIR" = "$(git rev-parse --git-dir)" ]; then
  REPO_ROOT="$(git rev-parse --show-toplevel)"
else
  # Inside a linked worktree: git-common-dir points at <main>/.git
  REPO_ROOT="$(cd "$COMMON_DIR/.." && pwd)"
  echo "Note: invoked from inside a worktree. Writing to main checkout: $REPO_ROOT"
fi

REPO_NAME="$(basename "$REPO_ROOT")"
AGENT_PATH="$REPO_ROOT/.claude/agents/${REPO_NAME}-sde.md"

if [ -f "$AGENT_PATH" ]; then
  echo "Agent already exists at $AGENT_PATH. Refuse unless user explicitly asked --force."
  exit 1
fi
```

Refuse politely if cwd is not a git repo. The agent body needs *something* to specialize on. **Always derive `REPO_ROOT` from `git rev-parse --git-common-dir`, never from `--show-toplevel` alone** — `--show-toplevel` returns the worktree path when invoked from inside one, and the agent would land in `<worktree>/.claude/agents/<worktree-basename>-sde.md` instead of the main checkout. Refuse if an SDE agent already exists at that path; offer to overwrite only if the user explicitly asks.

## Step 2 — Inspect the repo

Read every file below that exists. **All inspection is read-only.** Don't summarize as you go — collect raw text in a single context bundle that the meta-prompt will consume.

| Source | Why it matters |
|--------|----------------|
| `CLAUDE.md` (root, plus any `**/CLAUDE.md` up to depth 3) | Authoritative working rules. Must be deferred to, not duplicated. |
| `AGENTS.md` (root) | Same role as CLAUDE.md for agents that follow that convention. |
| `.kiro/steering/*.md` | Curated project knowledge: product, tech, structure, custom roles. Load-bearing if present. |
| `.kiro/specs/*/spec.json` | Active specifications. The agent should know what's in flight. List names only; don't load full spec bodies. |
| `README.md`, `README.*` | Surface-level identity, quick-start, dev workflow. |
| `DEVELOPMENT.md`, `DEV_PROMPT.md`, `CONTRIBUTING.md` | Often where the actual workflow lives if README is marketing-flavored. |
| `package.json` / `pyproject.toml` / `go.mod` / `Cargo.toml` / `Gemfile` / `composer.json` | Tech stack, scripts, test/lint commands. |
| `ls -F "$REPO_ROOT"` (top-level only) | Repo layout; informs the agent's mental model. |
| `git log --oneline -20` | Recent activity vocabulary; surfaces in-flight work. |
| `.claude/settings*.json` | Existing permissions/hooks. The new agent should not contradict them. |
| `.claude/worktrees/` (existence only) | If present, this repo already does worktree-driven agent work; mention it in the body. |
| `.worktrees/` (existence only) | Same as above with the alternate convention. |

**Capture project vocabulary explicitly.** Walk the README and CLAUDE.md and list every proper noun, "the X" phrase, internal service name, and domain term. These become the trigger keywords in the description.

If `CLAUDE.md`, `AGENTS.md`, README, and `.kiro/` are all absent: stop and tell the user "this repo has no steering context; write a CLAUDE.md or README first, then re-invoke." A hollow SDE agent is worse than none — it will drift on first contact.

## Step 3 — Confirm with the user

Three short questions, one message:

1. **Scope or exclusions.** "Anything in this repo the agent should explicitly *not* touch?" (Common answers: RFC docs, contracts directory, infra, secrets.)
2. **Feedback or preferences.** "Anything specific you want this agent to know — non-obvious from the steering docs — about how you work in this repo?" (Optional. Free text. Often empty.)
3. **Model override.** "Default is `model: inherit` so the orchestrator picks per task. Override to `sonnet`, `opus`, `haiku`, or a specific model ID? (default: inherit)"

Do not ask anything else. The meta-prompt will infer everything else from the inspection bundle. If the user does not respond, proceed with all defaults.

## Step 4 — Generate via the meta-prompt

Feed the inspection bundle plus the user's three answers into the meta-prompt below. The meta-prompt is adapted from Anthropic's `agent-development` plugin's `agent-creation-system-prompt.md`, narrowed to the SDE-style use case and pre-loaded with the defaults from `SKILL.md`'s quick-reference table.

### The meta-prompt

```
You are generating a project-scoped Claude Code subagent definition (an "SDE" — Software Development Engineer — for one specific repo).

INPUT: A bundle of files from the target repo (CLAUDE.md, AGENTS.md, .kiro/steering/, README, manifest, structure, vocabulary), plus three answers from the user (scope/exclusions, preferences, model override).

OUTPUT: A markdown subagent file with YAML frontmatter and a body, ready to write to .claude/agents/<repo>-sde.md.

REQUIRED FRONTMATTER (do not negotiate these):
- name: <repo>-sde   (where <repo> is the main repo's directory basename, lowercased, hyphens-only)
- description: a SINGLE-LINE PLAIN SCALAR (no `|`, no `>`, no quoted multiline). Starts with "Use when" — flat prose, 50-1024 chars, naming 2-4 trigger scenarios using the project's actual vocabulary (proper nouns, service names, domain terms surfaced in the bundle). Include a sentence describing what NOT to invoke this agent for. End with: 'See "When invoked" in the agent body for worked scenarios.'
- model: inherit  (or whatever the user overrode to)
- isolation: worktree
- memory: project

DO NOT add `tools` (omit so the agent inherits all). DO NOT add `color`. DO NOT add `permissionMode`. DO NOT add `metadata` or `license`.

BODY STRUCTURE (use these exact headings; fill content from the inspection bundle):

# <Repo Name> SDE

[One paragraph: what this repo is, in language drawn from the README/CLAUDE.md/VISION-style docs. Concrete, no hyperbole.]

[One paragraph: brief — "You are invoked by an orchestrator running in a different session. You do not see its conversation. Treat the prompt you receive as the full statement of work, ask a clarifying question only if a one-way door is genuinely ambiguous, and otherwise execute end-to-end and report back."]

## First moves, every invocation

[Numbered list of which files in this repo to read every session before doing substantive work. Include CLAUDE.md, AGENTS.md, .kiro/steering/* if present, plus any "always-loaded" docs the project's CLAUDE.md itself names. Use repo-relative paths, not absolute.]

[One sentence on what to read on demand vs always.]

## What you are working with

[Concrete, factual layout: production code dirs, test dirs, config, scripts, runtime/build commands. Pull names verbatim from the manifest. Include common dev/test/build commands.]

## How work happens here

[Quote the non-negotiable rules from CLAUDE.md verbatim — DO NOT paraphrase. Cover: source-of-truth conventions (e.g. GitHub-as-truth), output discipline (commits, attribution, hyperbole, honesty), one-way vs two-way doors, scope refusal. If CLAUDE.md is silent on a category, omit it; do not invent rules.]

## When invoked

[2-4 prose bullets describing scenarios this agent should handle. Each bullet starts with a bold short scenario name followed by what the agent should do. Examples:
- "**Direct repo work.** A change is requested to a path under [repo conventions]. Read the genome, do the work, run tests, return."
- "**Operational question.** The orchestrator asks about deploy state, cron schedule, or runtime budget. Surface the answer with the relevant file/command and any reminders the operator needs."]

## Engineering conventions

[Tech stack details: language version, package manager, test runner, lint/format. Pulled from the manifest. Project-specific conventions if surfaced (e.g. "experiments/ may import from src/ but never the reverse").]

[If the user provided scope/exclusions in step 3, add a "Do not modify" subsection here.]

## Return format to the orchestrator

Your final report must include:
1. What you did, what you verified, and how (commands run, tests run with their result, files changed with repo-relative paths).
2. Anything the orchestrator must do next that you could not do from this session.
3. Any tenet-tension or one-way door you flagged or resolved, with reasoning.
4. Honor the repo's output discipline (no co-author tags / emojis / hyperbole if the project's CLAUDE.md says so).

[End. No motivational closing.]

CONSTRAINTS:
- Body length: 500–10,000 characters.
- Use repo-relative paths, never absolute machine paths.
- Quote CLAUDE.md verbatim; do not paraphrase its rules.
- Do not invent rules the project hasn't declared.
- If `.kiro/specs/` exists, name the active specs in "What you are working with"; do not summarize their contents.
- If the project's CLAUDE.md forbids emojis/hyperbole/co-author tags, the body itself must follow those rules.
```

## Step 5 — Post-process

After generation, programmatically verify (don't trust the model):

- Frontmatter contains `isolation: worktree` and `memory: project` exactly as written.
- Description is a single-line plain scalar (no `|`, no `>`).
- Description starts with `Use when` or `Use proactively` (case-insensitive on the first non-whitespace token).
- Description ends with: `See "When invoked" in the agent body for worked scenarios.`
- Description includes at least one sentence describing what NOT to invoke this agent for (heuristic: contains the token `Do not` or `Don't` or `not for`).
- Frontmatter does NOT contain `tools`, `color`, `permissionMode`, `metadata`, or `license`.
- Body contains `## When invoked` and `## Return format to the orchestrator` (or close variant — `## Return format` matches).
- Body length 500–10,000 chars.
- No absolute paths matching `/home/`, `/Users/`, `/root/`, or any `[A-Za-z]:\` Windows path.
- If repo's CLAUDE.md contains the substring "no emojis" or "no co-author", grep the body for emojis / `Co-Authored-By` and fail if found.

If any check fails: regenerate with a corrective addendum to the prompt, **don't** post-process-fix in code. Post-process fixing produces files that look right but didn't go through the model's reasoning. **Cap regeneration at 2 attempts.** On the third failure, surface the specific check failures to the user and let them decide.

## Step 6 — Validate

```bash
scripts/validate-agent.sh "$AGENT_PATH"
```

Must exit 0 with zero warnings. If it warns, fix and re-validate before writing.

## Step 7 — Write

```bash
mkdir -p "$REPO_ROOT/.claude/agents"
# write file
```

Don't commit. Don't push. The user reviews, then commits.

## Step 8 — Show and explain

`cat` the written file. Then in under 200 words, tell the user:

- What inspection surfaced (which steering docs were present, vocabulary captured)
- Where defaults were overridden (only the user-driven ones from step 3)
- Two or three things the orchestrator should know about invoking this agent

End. Do not include a checklist for "next steps" — the user knows what to do next.

## Edge cases

**Monorepo with multiple deployable services** (e.g. coordinator + node-client + frontend). Generate one SDE agent for the *whole repo*, not one per service. The body's "What you are working with" section enumerates the services. Anthropic explicitly warns: "Excessive specialist agents dilute automatic delegation reliability."

**Project lives outside `~/workspace` or has multiple checkouts** (e.g. `~/streamr` and `~/workspace/streamr`). Use the cwd's repo root only. The orchestrator handles cross-location dispatch by absolute path.

**Project has its own existing agent infrastructure** (`.claude/.worker-state.local.json`, custom prompts, ralph-loop, etc.). The generated SDE agent should *reference* that infrastructure (point at `DEVELOPMENT.md` or `DEV_PROMPT.md` in "First moves"), not replace it. The skill's job is to add an orchestrator-facing entry point, not to rewire the project's internal automation.

**Project's CLAUDE.md uses `@file` includes** (e.g. `@docs/VISION.md`). Those resolve transitively when the agent is invoked — don't try to inline them. List the top-level CLAUDE.md in "First moves" and let the include chain do its job.

**Conflict between user's stated preference and the project's CLAUDE.md**. The project's CLAUDE.md wins. Note the conflict to the user in the step-8 explanation; let them resolve it.

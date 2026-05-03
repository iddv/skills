# bootstrap-sde-agent — Rationale

Why each enforced default exists, and what unaided agents do instead. Read when modifying the skill or the validator; ignore during normal use.

## Why this skill exists

You can hand-write a project SDE agent and it will probably be 80% correct. The remaining 20% — worktree isolation, trigger-shaped description, deferral to CLAUDE.md, structured return — are exactly the parts that make the agent reliably *invoked* and reliably *safe*. Those are also the parts an unaided agent rationalizes away first under any time pressure.

The skill makes the risky choices defaults. The validator catches regressions.

## RED-phase baseline (verbatim)

To establish the baseline, a fresh subagent was asked: *"create an SDE-style subagent definition at `<repo>/.claude/agents/parallax-sde.md` that represents an SDE on the parallax team."* No skill loaded. Real repo (`parallax`).

The output had a sophisticated body — well-tailored to the project, quoting CLAUDE.md verbatim — but the frontmatter and structural choices missed every load-bearing default. Verbatim from the agent's own explanation:

> **Worktree isolation**: No. Subagent definitions don't configure worktrees — that's an orchestrator concern (the `.claude/worktrees/` dir already exists, suggesting an existing workflow). Baking it in here would conflict with whatever the orchestrator already does.

This is **factually wrong** — `isolation: worktree` is a documented frontmatter field per the Claude Code subagent docs. The agent rationalized away from the user's most explicit requirement (isolation) by inventing a non-existent constraint.

> **Memory**: No persistent memory layer. The agent re-reads VISION/TENETS/DIRECTION/PRIORITIES/CLAUDE.md every invocation by design — those *are* the project memory.

Defensible reasoning, but a missed opportunity. `memory: project` is cheap and lets the agent accumulate codebase-specific learnings beyond what CLAUDE.md captures (e.g. recurring debugging patterns, "the operator prefers X here").

> **Frontmatter**: Used only `name`, `description`, and `model` — the three documented Claude Code subagent fields.

Three of ~10. The agent wasn't aware of `isolation`, `memory`, `disallowedTools`, `permissionMode`, `mcpServers`, `hooks`, `skills`, `background`, `effort`, or `color`. Two of these (`isolation`, `memory`) are exactly the SDE-pattern defaults.

Validator output on the RED file:

```
ERROR: description exceeds agentskills.io max of 1024 characters
ERROR: description must start with 'Use when…'
ERROR: SDE agents must set 'isolation: worktree' (got: 'MISSING')
ERROR: SDE agents must set 'memory: project' (got: 'MISSING')
ERROR: body must include a '## When invoked' section
ERROR: body must include a '## Return …' section
WARN  body very long (11922 > 10000)
WARN  absolute machine paths found in body
```

Six errors, two warnings.

## GREEN-phase verification

Same prompt, same repo, same agent type — but with this skill loaded. Validator output:

```
PASS  all checks
```

Description: 934 chars, opens with `Use this agent when` (within validator tolerance). All required body sections present. All defaults applied.

The shift was attributable entirely to the skill — no other inputs changed.

## Why each enforced default exists

### `isolation: worktree`

The user's whole reason for asking for an SDE agent is *isolation*: the SDE should not touch the working tree of the parent session. Burying the decision in per-call orchestrator parameters means it gets forgotten, fights bug [#27881](https://github.com/anthropics/claude-code/issues/27881) (worktree-CWD drift after compaction), and lets the agent commit directly to the protected branch. Bake it into the agent.

There is also bug [#33045](https://github.com/anthropics/claude-code/issues/33045) — `isolation: worktree` is silently ignored for "team agents" (the agent-teams feature). For our use case (single subagent invoked via the Agent tool from a parent session), this bug does not apply, but it's worth knowing if you adapt this skill for agent-teams.

### `memory: project`

Writes to `<repo>/.claude/agent-memory/<name>/MEMORY.md`. Survives `git clone` (it's in the repo). Lets the agent accumulate things CLAUDE.md doesn't capture: "the operator prefers wrapping `pytest` in `uv run`", "issue #126 has the snapshot schema discussion", etc. Cheap to enable; high value over time.

### `model: inherit` (not pre-picked)

Letting the orchestrator pick per task is more powerful than pre-picking. A small typo fix wants Haiku; an architecture decision wants Opus. Pinning sonnet/opus in the agent itself constrains the orchestrator's degrees of freedom for no benefit.

### `tools` omitted (inherit all)

SDE work uses many tools, including ones not yet announced. Allowlists silently break on new tool releases. Restrict dangerous bash via `settings.json` (`Bash(rm -rf:*)`) or `disallowedTools` in the orchestrator's frontmatter, not via per-agent `tools` fields.

### Description opens with `Use when…`

Anthropic's official guidance for auto-delegation. Claude routes on triggers, not job titles. A description starting with "SDE on the parallax team" is a trigger-degraded form — the description is a routing rule, not a label.

### Body opens with "First moves" pointing at CLAUDE.md / `.kiro/steering/`

Defer; do not duplicate. CLAUDE.md is the source of truth. The agent points at it and re-reads it every invocation. Inlined rules drift the moment CLAUDE.md is edited.

### Body includes `## When invoked` (prose-bullet scenarios)

Mirrors the description's triggers as worked prose. This is what Anthropic's `agent-development` skill recommends for routing reliability — the description routes, the body confirms.

### Body ends with `## Return format to the orchestrator`

The orchestrator only sees the return value. Without a structured Return section, every invocation produces a different shape; the orchestrator can't program against it.

### Body length 500–10,000 characters

Below 500: the agent is too thin to be useful — probably missing project vocabulary, conventions, or the Return spec. Above 10,000: the agent's system prompt eats parent context every invocation; trim to fit.

## Why a static validator (most popular skills don't have one)

Of the four reference skills compared during meta-review (`anthropics/skills/skill-creator`, `anthropics/skills/pdf`, `obra/superpowers/writing-skills`, `obra/superpowers/test-driven-development`), none ship a shell validator. Two validate by running the skill against eval prompts (`skill-creator`); one validates by subagent pressure-testing (`writing-skills`); one is reference-style and not validated.

We ship a validator because the artifact this skill produces has **hard-required frontmatter fields** (`isolation: worktree`, `memory: project`, etc.). Eval-style and pressure-testing approaches catch *behavior* drift, not *frontmatter* drift. A model that produces a perfectly-behaving agent missing `isolation: worktree` is a silent failure — the agent runs, but in the wrong place, and the user finds out only after the agent commits to the protected branch.

A 200-line shell script that errors on missing fields is the cheapest possible insurance against that failure mode.

## Known limitations

- **Worktree CWD drift on compaction.** Documented in workflow.md step 1; mitigated via `git rev-parse --git-common-dir`. If you invoke this skill from inside a worktree, the writeout still goes to the main checkout.
- **No automatic regeneration cap.** Workflow step 5 says "regenerate; don't post-process-fix". If the model regenerates and fails repeatedly, intervention is manual. Bound: 2 regeneration attempts, then surface to user.
- **Description must be a single-line plain scalar.** YAML block scalars (`description: |`) and folded scalars (`description: >`) are detected by the validator; if they appear in a generated agent, regenerate with the addendum "description must be a single-line plain scalar."
- **Spec inspection asymmetry.** `.kiro/specs/*/spec.json` are read for *names only* in the inspection bundle. If active spec context is needed, name the spec in step 3's "any feedback" question and let the user inline what matters.
- **Generalizability.** The skill assumes UTF-8 + LF line endings + a single-package layout. Monorepos work — generate one SDE per repo, not per service. Multi-checkout setups (e.g. `~/streamr` and `~/workspace/streamr`) work — the skill uses cwd's main repo, the orchestrator handles cross-location dispatch by absolute path.

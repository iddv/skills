# skills

Claude Code skills published to skills.sh.

## Install

```bash
npx skills add iddv/skills
```

Or install a specific skill by directory name from a clone:

```bash
git clone https://github.com/iddv/skills.git
cp -r skills/bootstrap-project-agent ~/.claude/skills/
```

## Skills

| Skill | Purpose |
|-------|---------|
| [bootstrap-project-agent](bootstrap-project-agent/) | Inspect a repo's existing context (CLAUDE.md, AGENTS.md, .kiro/, README, manifest) and generate a project-scoped Claude Code subagent at `.claude/agents/<repo>-agent.md`. Defaults to `isolation: worktree` and `memory: project`. |
| [bootstrap-orchestrator-agent](bootstrap-orchestrator-agent/) | Companion to `bootstrap-project-agent`. Discovers the per-repo project agents you've registered and generates an orchestrator subagent that runs as the **main session agent** (`claude --agent <name>`) and dispatches cross-repo work to those specialists. Restricts the `Agent(...)` allowlist to your registered fleet. |

## Conventions

- Each skill lives in its own directory at the repo root.
- The directory name matches the `name` field in `SKILL.md`'s frontmatter.
- Each skill follows the [Agent Skills specification](https://agentskills.io/specification).
- Each skill includes a `validate-*.sh` script under `scripts/` where applicable.

## License

MIT — see [LICENSE](LICENSE).

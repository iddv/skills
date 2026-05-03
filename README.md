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
| [bootstrap-project-agent](bootstrap-project-agent/) | Inspect a repo's existing context (CLAUDE.md, AGENTS.md, .kiro/, README, manifest) and generate a project-scoped Claude Code subagent at `.claude/agents/<repo>-project.md`. Defaults to `isolation: worktree` and `memory: project`. |

## Conventions

- Each skill lives in its own directory at the repo root.
- The directory name matches the `name` field in `SKILL.md`'s frontmatter.
- Each skill follows the [Agent Skills specification](https://agentskills.io/specification).
- Each skill includes a `validate-*.sh` script under `scripts/` where applicable.

## License

MIT — see [LICENSE](LICENSE).

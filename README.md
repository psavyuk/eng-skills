# eng-skills

Personal, opinionated rules and skills for the engineering disciplines I work in. Designed to be loaded by Claude Code as skills and read directly by humans (or any other coding assistant) as plain markdown.

## Layout

```
.
├── skills/                       # Claude Code skill format (frontmatter + body)
│   ├── python-data-engineer/SKILL.md
│   ├── backend-engineer/SKILL.md
│   ├── react-frontend/SKILL.md
│   ├── terraform-devops/SKILL.md
│   └── cicd/SKILL.md
├── python-data-engineer/RULES.md  # IDE-agnostic reference (longer, narrative)
├── backend-engineer/RULES.md
├── react-frontend/RULES.md
├── terraform-devops/RULES.md
└── cicd/RULES.md
```

- `skills/<topic>/SKILL.md` — tight, trigger-friendly. Loaded by Claude Code via the [skill-creator](https://github.com/anthropics/skills) conventions. Frontmatter has `name` and `description` so the harness can decide when to apply it.
- `<topic>/RULES.md` — the longer reference: rationale, anti-patterns, tool choices. Read these directly, or paste into any IDE/agent.

## Using with Claude Code

Symlink (or copy) the `skills/` directory into your user-level skills folder so Claude Code picks them up:

```bash
ln -s ~/Development/eng-skills/skills ~/.claude/skills/eng-skills
```

Then in any project you can invoke them with `/skill python-data-engineer` (or rely on auto-trigger via the `description` field).

## Using with other agents / IDEs

Point your tool's rules config at the relevant `<topic>/RULES.md`. Examples:

- Cursor: copy or include from `.cursor/rules/`.
- Aider: pass via `--read`.
- Plain prompt: paste the file into the system prompt.

## Updating

Rules are living documents. When a rule causes friction or proves wrong:

1. Edit the file.
2. Commit with a one-line reason (e.g. `react: drop CSS modules rule — Tailwind won this`).
3. Push.

Avoid documenting things that aren't durable preferences — those belong in the project's own CLAUDE.md / README, not here.

## Topics

| Topic | Skill | Rules |
|---|---|---|
| Python data engineering | [skills/python-data-engineer/SKILL.md](skills/python-data-engineer/SKILL.md) | [python-data-engineer/RULES.md](python-data-engineer/RULES.md) |
| Backend engineering | [skills/backend-engineer/SKILL.md](skills/backend-engineer/SKILL.md) | [backend-engineer/RULES.md](backend-engineer/RULES.md) |
| React frontend | [skills/react-frontend/SKILL.md](skills/react-frontend/SKILL.md) | [react-frontend/RULES.md](react-frontend/RULES.md) |
| Terraform / DevOps | [skills/terraform-devops/SKILL.md](skills/terraform-devops/SKILL.md) | [terraform-devops/RULES.md](terraform-devops/RULES.md) |
| CI/CD | [skills/cicd/SKILL.md](skills/cicd/SKILL.md) | [cicd/RULES.md](cicd/RULES.md) |

## License

MIT — see [LICENSE](LICENSE).

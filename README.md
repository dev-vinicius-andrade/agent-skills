# agent-skills

A marketplace of agent skills: opinionated, project-agnostic guidelines that tell coding agents **how** to do a kind of work (refactoring first) the way I want it done, in any repository.

Every skill is a plain folder with a `SKILL.md`, so it works with Claude Code, the Claude apps, and any other agent that can read Markdown instructions.

## Naming: `<action>:<stack>`

Skills are grouped by **action**; each action is one plugin, each stack-specific skill lives inside it.

| Invoked as | Plugin (action) | Skill (stack) | Folder |
|------------|-----------------|---------------|--------|
| `refactor:dotnet8` | `refactor` | `dotnet8` | `skills/refactor/dotnet8/` |
| `develop:dotnet8` | `develop` | `dotnet8` | `skills/develop/dotnet8/` |
| `bootstrap:dotnet9-api` *(planned)* | `bootstrap` | `dotnet9-api` | `skills/bootstrap/dotnet9-api/` |
| `bootstrap:dotnet8-worker` *(planned)* | `bootstrap` | `dotnet8-worker` | `skills/bootstrap/dotnet8-worker/` |

## Skills

| Skill | What it guides | Path |
|-------|----------------|------|
| `refactor:dotnet8` | Behavior-preserving refactoring of layered/DDD .NET 8 backends | [skills/refactor/dotnet8](skills/refactor/dotnet8/SKILL.md) |
| `develop:dotnet8` | Compact rules for writing new features and fixes in layered/DDD .NET 8 backends | [skills/develop/dotnet8](skills/develop/dotnet8/SKILL.md) |

## Install

### Claude Code (plugin marketplace)

```text
/plugin marketplace add dev-vinicius-andrade/agent-skills
/plugin install refactor@agent-skills
/plugin install develop@agent-skills
```

Installing a plugin installs every skill of that action (e.g. all `refactor:*` skills). Invoke one with `/refactor:dotnet8`, or let the agent pick it from its description.

Update later with `/plugin marketplace update agent-skills`.

### Claude apps

Zip a skill folder (e.g. `skills/refactor/dotnet8/`) and upload it under **Settings → Capabilities → Skills**.

### Other agents

Copy or symlink the skill folder into wherever your agent loads instructions from, or reference its `SKILL.md` from your project's `AGENTS.md`:

```markdown
When refactoring .NET code, follow `path/to/agent-skills/skills/refactor/dotnet8/SKILL.md`.
```

## How skills behave in a project

Skills hold defaults, not project facts. Each one starts by discovering the target repository (solution, layers, docs, existing conventions) and applies this precedence:

1. Explicit instructions in the task.
2. Conventions the repository documents (`AGENTS.md`, `CLAUDE.md`, `README`, `docs/`, `.editorconfig`, ...).
3. The skill's defaults.

## Repository layout

```text
.claude-plugin/marketplace.json   marketplace catalog (one plugin per action)
skills/<action>/<stack>/SKILL.md      skill entry point (frontmatter + workflow)
skills/<action>/<stack>/references/   detailed recipes loaded on demand
template/SKILL.md                 starting point for a new skill
AGENTS.md                         rules for adding or editing skills
```

## Contributing

See [AGENTS.md](AGENTS.md).

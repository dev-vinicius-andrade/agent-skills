# Authoring guide for this repository

Rules for any agent (or human) adding or editing skills here.

## Principles

- **Project-agnostic.** A skill must work in any repository. No company, product, solution, namespace, path, branch, or domain model names from a specific codebase. Use placeholders (`{Company}.{Product}`, `<solution>`) and made-up illustrative names, and say they are illustrative.
- **Discover, then apply.** A skill tells the agent how to learn the target repository (structure, docs, conventions, commands) before acting, and states the precedence: task instructions > repository conventions > skill defaults.
- **Guidelines, not essays.** Imperative, short, checkable rules. Every rule should be verifiable in a review.
- **Progressive disclosure.** `SKILL.md` holds the workflow, the strongest rules and a checklist (aim for under ~200 lines). Long recipes and examples go in `references/*.md`, linked from `SKILL.md`.

## Naming: `<action>:<stack>`

- **Action** = verb for the kind of work (`refactor`, `bootstrap`, `review`, `migrate`, ...). One plugin per action.
- **Stack** = what it applies to, including the major version when rules depend on it (`dotnet8`, `dotnet9-api`, `dotnet8-worker`, `react19`).
- Folder: `skills/<action>/<stack>/`. Frontmatter `name: <stack>` (lowercase letters, digits, hyphens; no colon). Claude Code exposes it as `<action>:<stack>`.
- Because the bare `name` loses the action outside Claude Code, start the `description` with "<Action> guideline for <stack> (invoked as <action>:<stack>)."

## Adding a skill

1. Copy `template/SKILL.md` to `skills/<action>/<stack>/SKILL.md`. The folder name must equal the frontmatter `name`.
2. Write a `description` that says what the skill does **and when to use it** (trigger phrases) and when not to. This is what agents use to decide to load it.
3. Add `references/` files if needed and link each one from `SKILL.md`.
4. Register it in `.claude-plugin/marketplace.json`: append `"./skills/<action>/<stack>"` to the `skills` of the plugin named `<action>`, or, for a new action, add a plugin entry (`name: "<action>"`, `source: "./"`, `strict: false`, `skills: [...]`) and list the skill in its description.
5. Add a row to the Skills table in `README.md`.
6. Bump the action plugin's `version` when any of its skills changes.

## Before committing

- `grep -ri` the skill for names from the project it was born in; none may remain.
- `SKILL.md` frontmatter has only `name` and `description` (plus optional `license`, `allowed-tools`, `metadata`).
- All relative links resolve.
- `marketplace.json` is valid JSON.

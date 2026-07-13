# Python Skills

Reusable Codex skill definitions for opinionated Python backend work.

This repository contains local skill files that can be installed into a Codex
skills directory and reused across projects. The current skills focus on
type-safe, validation-heavy Python development and a consistent FastAPI/backend
project layout.

## Included Skills

| Skill | Purpose |
|---|---|
| `python-programmer` | Production-grade Python implementation guidance: typed functions, Pydantic at I/O boundaries, immutable internal models, `returns`-style result handling, `pytest`, `ruff`, `ty`, and `basedpyright`. |
| `python-project-structure` | Mandatory backend/FastAPI directory layout and architectural rules separating domain logic, infrastructure I/O, API boundaries, tests, migrations, scripts, and docs. |

## Repository Layout

```text
.
├── README.md
└── skills/
    ├── python-programmer/
    │   └── SKILL.md
    └── python-project-structure/
        └── SKILL.md
```

Each skill is a directory containing a `SKILL.md` file with YAML front matter
and the instructions Codex should follow when the skill is selected.

## Installation

Copy the skill directories into your Codex skills directory:

```bash
mkdir -p ~/.codex/skills
cp -R skills/python-programmer ~/.codex/skills/
cp -R skills/python-project-structure ~/.codex/skills/
```

If your environment uses a different `CODEX_HOME`, install into that location
instead:

```bash
mkdir -p "$CODEX_HOME/skills"
cp -R skills/python-programmer "$CODEX_HOME/skills/"
cp -R skills/python-project-structure "$CODEX_HOME/skills/"
```

Restart Codex after installation if the skills are not detected immediately.

## Usage

Invoke a skill by name when asking Codex to work on a Python project:

```text
Use $python-programmer to implement this service with typed errors and tests.
```

```text
Use $python-project-structure to scaffold the backend layout before adding code.
```

The two skills are designed to work together:

- `python-project-structure` defines where code belongs.
- `python-programmer` defines how the Python code should be written and verified.

## Engineering Defaults

These skills encode the following defaults:

- Prefer pure functions and small modules over broad object-oriented designs.
- Use Pydantic for validated external I/O contracts.
- Use frozen dataclasses, enums, protocols, and typed errors for internal domain code.
- Keep domain logic free from FastAPI, SQLModel, SDKs, HTTP clients, credentials, and other I/O concerns.
- Model failures explicitly with typed error variants and `returns.Result`.
- Add or update tests for behavior changes.
- Run `pytest`, `ruff`, `ty`, and `basedpyright` before finishing work when available.

## Updating Skills

Edit the relevant `SKILL.md` file directly:

```text
skills/<skill-name>/SKILL.md
```

Keep each skill focused and actionable. A good skill should describe concrete
decision rules, project conventions, and verification steps rather than broad
style preferences.

## Public Repo Notes

This repository intentionally contains only reusable skill instructions. It does
not include project code, package metadata, or runtime dependencies.

If you extend it, keep generated files, secrets, local logs, virtual
environments, and editor-specific state out of the repository.

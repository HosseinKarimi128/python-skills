# Python Skills

Reusable skill definitions for opinionated Python work.

This repository contains local skill files that can be installed into a skills
directory supported by your coding agent, or used directly by agents that can
load local skill definitions. The current skills focus on type-safe,
validation-heavy Python development and a consistent backend project layout,
with room to expand into other Python topics and verification workflows over
time.

## Included Skills

| Skill | Purpose |
|---|---|
| `python-programmer` | Production-grade Python implementation guidance: typed functions, Pydantic at I/O boundaries, immutable internal models, `returns`-style result handling, `pytest`, `ruff`, `ty`, and `basedpyright`. |
| `python-project-structure` | Mandatory backend directory layout and architectural rules separating domain logic, infrastructure I/O, API boundaries, tests, migrations, scripts, and docs. |

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
and the instructions a compatible coding agent should follow when the skill is
selected.

## Installation

Copy the skill directories into the skills directory used by your agent:

```bash
mkdir -p <skills-dir>
cp -R skills/python-programmer <skills-dir>/
cp -R skills/python-project-structure <skills-dir>/
```

If your agent loads skills from the repository directly, you can usually skip
this step and point it at the local `skills/` directory instead.

## Usage

Invoke a skill by name when asking a compatible agent to work on a Python
project:

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

The skills are intentionally written to be broadly useful across coding agents
and Python project types. They are not limited to Codex, and they are not
limited to FastAPI. The current scope reflects the skills already present in
this repository; future additions may cover other Python domains, tooling, and
linting or verification rules that help agents follow the skills consistently.

If you extend it, keep generated files, secrets, local logs, virtual
environments, and editor-specific state out of the repository.

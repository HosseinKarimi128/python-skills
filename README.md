# Python Skills

Reusable skill definitions for opinionated Python backend engineering.

This repository currently exposes one canonical Python backend skill. It combines
project structure, PostgreSQL-first architecture, functional programming rules,
typing, Pydantic I/O boundaries, `returns` control flow, logging, testing, and
quality gates in a single source of truth so coding agents do not have to reconcile
overlapping skill documents.

## Included Skill

| Skill | Purpose |
|---|---|
| `postgres-first-python` | **Postgres-First Python** — canonical PostgreSQL-first Python backend guidance: project structure, SQL-first persistence, Pydantic external I/O, domain dataclasses/enums/protocols, functions-only workflows, `returns.Result`/`Maybe`, `singledispatch`, structured logging, `pytest`, `ruff`, `ty`, and `basedpyright`. |

## Repository Layout

```text
.
├── README.md
└── skills/
    └── postgres-first-python/
        ├── SKILL.md
        └── agents/
            └── openai.yaml
```

`skills/postgres-first-python/SKILL.md` is the single authoritative rule document.

## Installation

Copy the skill into the skills directory used by your coding agent:

```bash
mkdir -p <skills-dir>
cp -R skills/postgres-first-python <skills-dir>/
```

If your agent loads skills directly from this repository, point it at `skills/`
instead.

## Usage

```text
Use $postgres-first-python to implement or refactor this Python backend.
```

The skill covers both architecture and programming behavior, including:

- simple CRUD/query/report: `API -> SQL -> PostgreSQL`;
- non-trivial business work: `API -> workflow -> domain / SQL / gateway`;
- all runtime SQL in `.sql` files;
- SQL migrations as schema history;
- no ORM/repository/mapper/query-layer ceremony by default;
- Pydantic for every external I/O boundary;
- data-only domain `Enum`, frozen `dataclass`, and `Protocol` classes;
- functions-only workflow modules with helper/pipeline rules;
- typed expected failures through `returns.Result` / `Maybe`;
- `functools.singledispatch` for type-driven domain behavior;
- structured JSON + PostgreSQL logging with redaction/correlation;
- focused `pytest` tests and `ruff`, `ty`, `basedpyright` delivery gates.

## Updating the Skill

Edit only the canonical rule document:

```text
skills/postgres-first-python/SKILL.md
```

Keep rules concrete, internally consistent, and agent-actionable. Avoid adding
parallel documents that restate the same conventions, because duplicated rules can
drift over time.

## Public Repo Notes

This repository intentionally contains reusable skill instructions rather than
application code or runtime dependencies. Keep generated files, secrets, local logs,
virtual environments, and editor-specific state out of the repository.

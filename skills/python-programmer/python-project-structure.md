---
name: python-project-structure

description: >
  Brief project-structure summary for the python-programmer skill. The dedicated
  python-project-structure skill is authoritative.
---

# Project Structure Summary

Follow the dedicated `python-project-structure` skill. Keep this programming skill
aligned with its PostgreSQL-first, low-ceremony layout:

```text
src/myapp/
├── api/        # HTTP + Pydantic boundary models
├── workflow/   # non-trivial orchestration; functions only
├── domain/     # Enum, frozen dataclass, Protocol classes only
├── db/         # generic PostgreSQL execution/lifecycle
├── gateway/    # external systems + Pydantic provider payloads
├── error.py    # typed application failure values
├── config.py
├── state.py
└── main.py

sql/
├── migrations/
├── <capability>/
├── functions/
├── views/
├── triggers/
└── seeds/
```

Normal execution paths:

```text
simple CRUD/query/report:
api -> db/sql -> .sql -> PostgreSQL

non-trivial business work:
api -> workflow -> domain / db/sql / gateway
```

Rules:

- Keep API, workflow, domain, and gateway flat by default.
- Do not force simple CRUD through `workflow/`.
- `domain/` contains no executable functions and no Pydantic.
- `workflow/` contains no classes.
- All runtime SQL lives in `.sql` files.
- PostgreSQL is intentionally first-class; do not hide it behind repositories.
- Do not add ORM models, repositories, mappers, Python query layers, repository UoW,
  mandatory ports, or Alembic Python migrations by default.
- Add hierarchy or abstraction only when real complexity requires it.

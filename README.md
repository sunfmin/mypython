# mypython

Python coding standards as a Claude Code agent skill, plus this repo's agent config.

**Two laws:** uv runs everything; every fact has exactly one home.

- `skills/mypython/SKILL.md` — the skill
- `docs/agents/` — issue tracker, triage labels, and domain-doc conventions

## Install

```bash
npx skills add sunfmin/mypython -g
```

## Validation

Verified against uv 0.11.32, ruff, and ty:

- the canonical `pyproject.toml` passes `uv sync --locked`, `ruff format --check`,
  `ruff check`, `ty check`, and `pytest`;
- the gates catch real violations — `assert_never` names the missing enum member,
  `NewType` mixups error, `uv sync --locked` fails on lock drift;
- trigger tested 13/13 (6 should-fire, 7 decoys clean).

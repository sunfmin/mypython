---
name: python
description: Python development standards — uv is the only entrypoint (never pip, python, venv, conda, poetry, or requirements.txt) and every fact has exactly one home. This skill should be used whenever writing, editing, running, testing, debugging, reviewing, refactoring, or packaging Python code; creating or changing any .py file, pyproject.toml, or uv.lock; adding, removing, or upgrading a dependency; writing a standalone or one-off script; or setting up Python CI. It covers uv, ruff, and ty commands, PEP 723 inline script metadata, dependency groups, deriving versions and schemas rather than copying them, idempotent convergence, and making illegal states unrepresentable with Literal, StrEnum, NewType, assert_never, and pydantic.
---

# Python

Two laws. Everything else follows.

1. **uv runs everything.** There is no other way to invoke Python.
2. **Every fact has exactly one home.** Copies drift silently; derive instead.

Law 2 is the single-source-of-truth philosophy. Each rule below names the duty it moves
off human memory and onto a mechanism that cannot forget — compiler, type, lock, or CI gate.

Every claim below was verified against uv 0.11.32, ruff, and ty on 2026-07-26.

---

## 0. The uv mandate

Requires **uv ≥ 0.11** (`uv check` and `uv audit` are recent). Pin it — see §7.
Not installed? `brew install uv`, or `curl -LsSf https://astral.sh/uv/install.sh | sh`.
uv also manages interpreters (`uv python install`) — never install Python via brew or
pyenv for a project; that's a second home for "which Python".

| Never | Always | Why |
|---|---|---|
| `pip install x` | `uv add x` | pip mutates an env with no record; `uv add` writes pyproject **and** uv.lock — the change has a home |
| `python foo.py` | `uv run foo.py` | bare `python` is whatever's on PATH; `uv run` resolves the declared interpreter + deps |
| `python -m venv` / `virtualenv` | `uv sync` | `.venv` is a derived artifact, never hand-built |
| `source .venv/bin/activate` | `uv run <cmd>` | activation is ambient state a later command can silently miss |
| `requirements.txt`, `requirements-dev.txt` | `[project.dependencies]`, `[dependency-groups]` | a second dep list is a drift bomb |
| `setup.py`, `setup.cfg` | `pyproject.toml` | one home for metadata |
| `pip freeze > requirements.txt` | `uv.lock` (committed) | freeze snapshots one machine's env; the lock is cross-platform and derived from the declaration |
| `conda`, `poetry`, `pipenv`, `pdm` | uv | one resolver, one lock format |
| `pipx install x` | `uvx x` / `uv tool install x` | tools stay out of the project env |
| `os.system("pip ...")` in code | declare the dep | runtime installs are imperative steps (violates §3) |

**`uv run` self-heals.** It locks and syncs before executing, so the environment
converges to the declaration on every invocation. You never need to remember to
install anything — that duty is gone (§4).

### The verbs

| Task | Command |
|---|---|
| Run code | `uv run <script.py>` / `uv run -m pkg` / `uv run pytest` |
| Add / remove dep | `uv add x` / `uv add --group dev x` / `uv remove x` |
| Converge env | `uv sync` (exact: prunes extras) |
| Reproducible install | `uv sync --locked` ← **CI uses this** |
| Format | `uv run ruff format` |
| Lint | `uv run ruff check --fix` |
| Type check | `uv run ty check` |
| Test | `uv run pytest` |
| Vulnerability audit | `uv audit` |
| Bump version | `uv version --bump patch\|minor\|major` |
| Throwaway tool | `uvx <tool>` |

`uv format` and `uv check` exist as zero-config shortcuts (they wrap ruff and ty),
but **prefer `uv run ruff …` / `uv run ty …`**: that way the tool version lives in
`uv.lock` alongside everything else, so your editor, your terminal, and CI cannot
disagree. A tool version resolved outside the lock is a second home (§1).

### Standalone scripts — PEP 723, never a stray venv

A script that needs deps declares them **in the script**:

```python
# /// script
# requires-python = ">=3.12"
# dependencies = ["httpx", "rich"]
# ///

import httpx
```

Run it with `uv run fetch.py`. Add deps with `uv add --script fetch.py httpx`.
Never hand-edit the block — `uv add --script` owns it.

The dependency lives next to the code that imports it (§6). No README line saying
"first pip install httpx" — that instruction is a duty parked on a human, and it rots.

### Landing in a project that isn't on uv yet

Check this **before** writing code. Migrate first, then work.

| Found | Move |
|---|---|
| `requirements.txt` | `uv init --bare` → `uv add -r requirements.txt` → delete the txt |
| Poetry (`[tool.poetry]`) | `uvx migrate-to-uv` — rewrites to PEP 621 `[project]` |
| pipenv / pdm | `uvx migrate-to-uv` |
| loose scripts, no project | PEP 723 header + `uv run script.py` |
| already on uv | nothing to do |

Never bolt uv on beside a live Poetry or pip setup. Two dependency lists is precisely the
failure this skill exists to prevent — and the migration is a two-minute command.

If migrating is genuinely out of scope right now, **say so explicitly** and follow the
project's existing tooling consistently. Do not half-convert.

### Precedence

These rules govern **new** code. When editing an existing file, match its surrounding
style — do not reformat, re-type, or restructure a module as a drive-by; that buries the
actual change in noise.

But never *add* a new violation to an old file: no fresh `pip install`, no new
`requirements.txt` entry, no bare `str` where the set is closed. Leave existing ones alone
unless you were asked to clean them up.

---

## 1. One home

**pyproject.toml is the only config file.** Delete `setup.py`, `setup.cfg`, `.flake8`,
`.isort.cfg`, `pytest.ini`, `tox.ini`, `mypy.ini`, `requirements*.txt`.
Ruff, ty, and pytest all read `pyproject.toml`.

**Dev deps go in `[dependency-groups]`** (PEP 735), not `[project.optional-dependencies]`.
Groups are never shipped in the built wheel; extras are.

```toml
[dependency-groups]
dev = ["pytest>=8", "ruff>=0.14", "ty>=0.0.1a1"]
test = ["pytest-cov"]
```

`uv sync` installs `dev` by default. `uv sync --no-dev` for production images.

**Config values get one home too.** Never scatter `os.environ.get(...)` across modules —
each call site is a place the default can disagree.

```python
# ✅ one home; parsed once, at the boundary
from pydantic_settings import BaseSettings

class Settings(BaseSettings):
    database_url: str
    timeout_seconds: int = 30

settings = Settings()  # type: ignore[call-arg]
```

```python
# ❌ three homes for one fact, three defaults that can drift
timeout = int(os.environ.get("TIMEOUT", "30"))
```

---

## 2. Derive, don't copy

**Version — the canonical Python drift bug.** `pyproject.toml` holds the version.
`__version__` derives from it:

```python
# ✅ src/mypkg/__init__.py
from importlib.metadata import version

__version__ = version("mypkg")
```

```python
# ❌ now bump it in two places forever
__version__ = "1.2.3"
```

Bump with `uv version --bump minor`, never by hand-editing.

**Python version — set it once.** `[project] requires-python` is the one home.
Both Ruff and ty *infer their target version from it*. So:

```toml
[project]
requires-python = ">=3.12"
```

and **do not** set `[tool.ruff] target-version` or `[tool.ty.environment] python-version` —
each would be a second home that silently wins over the first.

`.python-version` is **not** a duplicate — it's a different fact (*which interpreter dev
and CI actually use*, a choice inside the supported range). Commit it.

**Derive collections from the enum**, never a parallel list:

```python
# ✅ one home
class Status(StrEnum):
    PENDING = "pending"
    DONE = "done"

ALL_STATUSES = [s.value for s in Status]
```

```python
# ❌ add a member, forget the list
ALL_STATUSES = ["pending", "done"]
```

**Other derivations:** JSON Schema from `Model.model_json_schema()`, not hand-written.
CLI flags from type hints (Typer/`argparse` off a dataclass), not a parser mirroring a
config class. API clients from OpenAPI via `datamodel-code-generator`. `uv.lock` from
`pyproject.toml` — **never hand-edit the lock**.

---

## 3. Declare state, not steps

`pyproject.toml` + `uv.lock` **declare** the environment; `uv sync` finds the path there
from whatever state the machine is in. A `setup.sh` full of `pip install` lines assumes a
starting state and corrupts anything else.

```dockerfile
# ✅ declarative, cached, reproducible
COPY pyproject.toml uv.lock ./
RUN uv sync --locked --no-dev
```

Schema changes go through Alembic migrations, never `Base.metadata.create_all()` against a
real database — `create_all` is a no-op on an existing table, so it silently skips your
column change.

Prefer declarative shapes in code too: `@dataclass` / pydantic models over
`__init__` bodies that assign twelve attributes.

---

## 4. Idempotent — re-run changes nothing

`uv sync` converges. Running it twice is a no-op. Hold your own code to that bar.

**Never branch on "did this already run?"** — that asks the process to remember across
crashes, which it cannot.

```python
# ❌ read-modify-write: a retry double-applies
row = db.get(id); row.count += 1; db.save(row)

# ✅ atomic upsert — converge to the declared state
db.execute(
    insert(Counter).values(id=id, count=1)
    .on_conflict_do_update(index_elements=["id"], set_={"count": Counter.count + 1})
)
```

For anything that crosses a network, retries are guaranteed (at-least-once is all the world
offers). Buy exactly-once *effect* with an idempotency key, not exactly-once delivery:

```python
client.post("/charges", json=payload, headers={"Idempotency-Key": request_id})
```

**Codegen must be idempotent and gated.** Any generator you write: running it twice
produces a byte-identical tree. CI proves it (§7).

`functools.cache` and any materialized table are *derived views* — the world is never
rebuilt from them.

---

## 5. Make illegal states unrepresentable

A runtime `if` trusts every future caller to remember the check. A type trusts no one.

**Never a bare `str` for a closed set:**

```python
# ✅ the set is the type
class Status(StrEnum):
    PENDING = "pending"
    DONE = "done"

# also fine for small closed sets
Mode = Literal["fast", "safe"]
```

**Exhaustiveness — the compiler patrols the call sites, not you.** Add a member, and every
unhandled `match` becomes a type error:

```python
from typing import assert_never

def label(s: Status) -> str:
    match s:
        case Status.PENDING: return "waiting"
        case Status.DONE:    return "finished"
        case _ as unreachable: assert_never(unreachable)
```

**Parse, don't validate.** Untyped data is parsed **once** at the boundary and never again:

```python
# ✅ dict[str, Any] dies at the door
def handle(raw: bytes) -> Response:
    req = CreateUser.model_validate_json(raw)   # parse once
    return create(req)                          # everything downstream is typed
```

```python
# ❌ every function re-checks, and one of them forgets
def create(data: dict[str, Any]) -> dict[str, Any]:
    if "email" not in data: raise ValueError(...)
```

**Distinguish IDs by type**, so you can't pass the wrong one:

```python
UserId = NewType("UserId", int)
OrderId = NewType("OrderId", int)
def cancel(o: OrderId) -> None: ...
cancel(user_id)  # ty: error
```

**Model states as a union, not a pile of Optionals.** `Optional[str] × 3` = 8 states, 5 of
them nonsense. A discriminated union has exactly the states that exist:

```python
@dataclass(frozen=True)
class Pending: submitted_at: datetime

@dataclass(frozen=True)
class Failed: submitted_at: datetime; error: str

Job = Pending | Failed  # "failed with no error" is now unwriteable
```

Default to `@dataclass(frozen=True, slots=True)` and `Final` for module constants.

**When a refactor makes a bug class impossible, delete its guard test and say so** in the
commit message. The structure holds the line now; the test is dead weight implying the risk
is still live.

Suppress narrowly and never blanket-ignore: `# ty: ignore[unresolved-attribute]`.

---

## 6. Colocate truth with the thing

A side dict keyed by name is a join you maintain by hand — it has two failure modes a field
can't have (orphan key, missing key) plus a typo-able string key.

```python
# ❌ two structures to keep in step
class Color(Enum): RED = "red"; BLUE = "blue"
LABELS = {"red": "Red", "blue": "Blue"}      # typo-able, silently incomplete
```

```python
# ✅ the fact lives on the member
class Color(Enum):
    RED = ("red", "Red")
    BLUE = ("blue", "Blue")

    def __init__(self, code: str, label: str) -> None:
        self.code = code
        self.label = label
```

Same move elsewhere: constraints as `Annotated[int, Field(ge=0)]` on the field, not a
validator function far away; fixtures in the `conftest.py` nearest the tests that use them;
`@property` on the model instead of `compute_x(model)` in a utils module.

---

## 7. Hand-sync unavoidable? Gate it

Some copies can't be removed — a README example, a committed generated file. The copy is
forced, so its freshness must not rest on memory. Make the build red instead.

```yaml
- run: uv sync --locked          # lock matches pyproject, or fail
- run: uv run ruff format --diff # formatting is committed, or fail
- run: uv run ruff check
- run: uv run ty check
- run: uv run pytest
- run: uv audit
- run: uv run python -m tools.codegen && git diff --exit-code   # generated files current
```

`uv sync --locked` is the whole point: it **errors** if `uv.lock` is stale rather than
quietly updating it. Never use `--frozen` in CI (it skips the check entirely).

**Test the docs.** README code blocks rot invisibly. Execute them:

```python
# tests/test_docs.py
import pathlib, pytest
from mktestdocs import check_md_file

@pytest.mark.parametrize("f", pathlib.Path("docs").glob("**/*.md"), ids=str)
def test_docs(f): check_md_file(fpath=f, memory=True)
```

**Pin the toolchain**, so "works on my machine" is structural rather than hoped-for:

```toml
[tool.uv]
required-version = ">=0.11"
```

Commit `.python-version` and `uv.lock`. In GitHub Actions use `astral-sh/setup-uv` pinned
to a SHA, and `uv python install` to honour `.python-version`.

---

## 8. Extract a package — but earn it

Rule 1 pushed across a repo boundary. Copying logic into a second repo is the worst drift:
the person who'd "remember to update the other copy" doesn't know it exists.

**But this rule has a brake.** Extraction buys a dependency edge (version skew, breaking
changes rippling outward). Duplicate until the **third** use reveals the real shape, then
extract. A wrong abstraction costs more than the duplication it replaced.

Inside one repo, use a uv workspace rather than a published package:

```toml
[tool.uv.workspace]
members = ["packages/*"]

[tool.uv.sources]
shared = { workspace = true }
```

One lockfile for the whole workspace, edits picked up immediately, no publish step, no
version skew. Publish to an index only when a genuinely external consumer needs it.

---

## Canonical pyproject.toml

```toml
[project]
name = "mypkg"
version = "0.1.0"
requires-python = ">=3.12"        # ONE home — ruff and ty both infer from this
dependencies = ["httpx>=0.27"]

[dependency-groups]
dev = ["pytest>=8", "pytest-cov", "ruff>=0.14", "ty"]

[tool.uv]
required-version = ">=0.11"

[tool.ruff]
line-length = 88
# no target-version — inferred from requires-python

[tool.ruff.lint]
select = ["E", "F", "I", "B", "UP", "SIM", "RUF", "ANN", "PTH"]
ignore = ["E501"]                 # the formatter owns line length

[tool.ty.rules]
all = "error"

[tool.pytest.ini_options]
addopts = "--strict-markers --strict-config"
testpaths = ["tests"]

[build-system]
requires = ["uv_build>=0.11"]
build-backend = "uv_build"
```

Libraries: keep runtime constraints loose (`>=`), never pin — pinning in a library forces
conflicts on consumers. `uv.lock` still gets committed; it governs *your* dev/CI env, not
your consumers'.

---

## Definition of done

Nothing is finished until all of these pass:

```bash
uv sync --locked
uv run ruff format --check
uv run ruff check
uv run ty check
uv run pytest
```

## Smell → fix

| Smell | Duty parked on a human | Move |
|---|---|---|
| `pip install` in a README or Dockerfile | "install the right things" | `uv add` + `uv sync --locked` |
| `__version__ = "1.2.3"` | "bump both" | `importlib.metadata.version()` |
| `requirements-dev.txt` | "update both lists" | `[dependency-groups]` |
| `target-version` beside `requires-python` | "keep them equal" | delete it; it's inferred |
| `dict[str, Any]` past the boundary | "re-validate everywhere" | parse into a model once |
| `if x not in ("a","b"): raise` | "call the check" | `Literal` / `StrEnum` |
| `match` with no `assert_never` | "update every branch" | `assert_never` |
| `LABELS = {enum_value: ...}` | "keep keys in step" | attribute on the enum member |
| `row.count += 1; save()` | "don't double-apply" | atomic upsert / append+fold |
| `if already_ran: return` | "know if it ran" | converge instead |
| `# remember to update the docs` | that IS the smell | generate it, or test it in CI |
| same helper copied to a 3rd repo | invisible cross-boundary drift | now it's earned — extract |

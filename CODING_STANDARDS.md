# Coding Standards

**Audience:** coding agents working in this repo, and me.
**Stack:** Python 3.11+, JavaScript (Node 20+), typically Dockerised, self-hosted.
**Scale:** single maintainer, small local apps. Rules are calibrated for that — not for a team.

## How to read this

- **MUST** — do not deviate. If you think you need to, stop and ask.
- **SHOULD** — the default. Deviate only with a stated reason in the commit or PR body.
- **PREFER** — a tiebreaker for when you'd otherwise pick arbitrarily.

**Precedence when rules conflict:** existing code in the file you're editing > this document > your defaults. Do not reformat or restyle surrounding code to match this doc as a side effect of an unrelated change.

---

## 1. Agent working rules

These matter more than anything below. Style problems are cheap to fix; the failures in this section are the expensive ones.

**MUST NOT** widen the scope of a task without asking. Fix the thing asked for. Do not opportunistically refactor, rename, upgrade dependencies, reformat untouched files, or "tidy" adjacent code. If you spot something worth changing, say so and leave it.

**MUST NOT** add a dependency without asking. Reach for the standard library first. If a dependency is genuinely warranted, name it, say what it replaces, and wait. This applies to transitive weight too — do not pull in a framework to solve a twenty-line problem.

**MUST NOT** claim work is complete without running it. "Should work" is not done. Run the code, run the tests, show the output. If you cannot run it in your environment, say explicitly that it is unverified and what needs checking.

**MUST NOT** write silent fallbacks that mask failure. No `except Exception: pass`. No empty catch blocks. No defaulting to a placeholder value when the real one is missing. A component that fails should fail loudly, not degrade into pretending.

```python
# Never
try:
    cfg = load_config()
except Exception:
    cfg = {}          # app now runs with silently wrong behaviour

# Instead
cfg = load_config()   # let it raise; the entry point decides what to do
```

**MUST NOT** leave stub implementations presented as finished. If something is a placeholder, mark it `raise NotImplementedError` or `# TODO:` — never a function that returns a plausible-looking hardcoded value.

**MUST NOT** invent APIs, config keys, env var names, or CLI flags. If you are unsure whether a method exists, check the source or docs. A confidently wrong function signature costs more debugging time than asking.

**SHOULD** ask rather than guess when a requirement is ambiguous and the choice is hard to reverse — schema shape, file format, external interface, anything that persists to disk or the network. Guess freely on things that are cheap to change.

**SHOULD** state assumptions inline when you do proceed on a guess. One line: `# Assuming UTC; no timezone specified in the source data.`

**MUST** make destructive operations obvious and opt-in. Anything that deletes files, drops tables, overwrites data, or mutates remote state needs a `--dry-run` default or an explicit confirmation flag. Never write a script that destroys data on a bare invocation.

---

## 2. Architecture

**MUST NOT** create circular imports. The import graph is a DAG.

**SHOULD** separate business logic from I/O. Logic that decides should not also be the code that fetches, writes, or transmits — that's what makes it testable and what lets you swap the source later.

```
models/     data structures, validation — no I/O
services/   business logic — takes data in, returns data out
adapters/   database, HTTP clients, file and message-bus I/O
handlers/   entry points: routes, CLI commands, schedulers
```

Use directories when a layer has more than ~300 lines or three concepts; a single `services.py` is fine below that. Do not build the full tree for a 200-line script — a single module is the correct structure for a small tool.

**MUST** externalise environment-specific values: hostnames, ports, paths, credentials, feature flags. No deployment detail hardcoded in source.

**MUST** validate configuration at startup and fail immediately with a message naming the missing or invalid key. A missing env var should not surface as a `NoneType` error forty seconds into a run.

```python
from pydantic_settings import BaseSettings

class Settings(BaseSettings):
    database_url: str
    api_key: str
    log_level: str = "INFO"

    class Config:
        env_file = ".env"

settings = Settings()  # raises at import with a readable error
```

**PREFER** passing dependencies as constructor or function arguments over module-level globals and singletons. It's the difference between a testable unit and one that needs a live database.

---

## 3. Python

### Tooling

**MUST** use `ruff` for both linting and formatting. It replaces black, isort, flake8, and most of pylint, in one tool.

```bash
ruff format .        # formatting
ruff check --fix .   # lint + autofix, includes import sorting
```

**MUST** add type hints to function signatures. `mypy` in strict mode on `src/`.

**MUST NOT** silence the type checker with `# type: ignore` to make an error go away. If it is genuinely unavoidable (untyped third-party library), use the specific form with a reason: `# type: ignore[attr-defined]  # boto3 stubs incomplete`.

Formatting is whatever `ruff format` produces. Do not argue with it, do not hand-format around it, and do not add style rules here that duplicate it — quote style, line breaks, and trailing commas are settled by the tool.

### Structure

```
project/
├── src/
│   ├── config.py       # settings object, validated at import
│   ├── models.py       # data structures
│   ├── services.py     # business logic
│   ├── adapters.py     # external I/O
│   ├── exceptions.py   # app exception hierarchy
│   └── main.py         # entry point, logging setup
├── tests/
│   ├── conftest.py
│   └── test_*.py
├── pyproject.toml
└── .env.example        # every key, no real values
```

### Errors

**MUST** define an app exception hierarchy rooted in a single base class, so callers can catch app errors distinctly from bugs.

```python
class AppError(Exception):
    """Base for all application errors."""

class ConfigError(AppError):
    """Configuration missing or invalid."""

class ValidationError(AppError):
    """Input failed validation."""

class ExternalServiceError(AppError):
    """A third-party service failed or was unreachable."""
```

**MUST NOT** use bare `except:` — it swallows `KeyboardInterrupt` and `SystemExit`.

**MUST** preserve the original error when re-raising as an app error:

```python
try:
    resp = httpx.get(url, timeout=10)
    resp.raise_for_status()
except httpx.HTTPError as exc:
    raise ExternalServiceError(f"Weather API unreachable: {url}") from exc
```

The `from exc` is not optional. Without it you lose the traceback that tells you what actually broke.

**SHOULD** document raised exceptions in the docstring when a function raises something a caller must handle.

### Logging

**MUST** use the `logging` module, not `print()`, in anything that runs unattended.

**MUST** configure logging only at the entry point. Library modules do `logger = logging.getLogger(__name__)` and nothing else — no `basicConfig`, no handlers.

**SHOULD** use `logger.exception()` inside an `except` block — it captures the traceback. `logger.error()` elsewhere.

**MUST NOT** log secrets, tokens, full auth headers, or personal data. Log the key name, not the value.

**PREFER** structured context over string interpolation where the logs get parsed:

```python
logger.info("Sync completed", extra={"files": 412, "bytes": 1_048_576, "target": "b2"})
```

### Async

**SHOULD** use `asyncio` only for genuinely concurrent I/O. A script that makes three sequential HTTP calls does not need it — async has a real readability and debugging cost, and a partial async conversion is worse than none.

**MUST NOT** call blocking code inside a coroutine. Use `asyncio.to_thread()` for unavoidable sync calls.

---

## 4. JavaScript

Plain JavaScript is the default here. TypeScript is worth it for anything with a real domain model or more than a couple of modules — but it's a choice per project, not a blanket rule.

**MUST** use ESLint with flat config (`eslint.config.js`) and Prettier. Note ESLint 9 removed `.eslintrc.*` support.

**SHOULD** enable type checking on plain JS via JSDoc rather than skipping types entirely — most of the benefit, none of the build step:

```javascript
// @ts-check

/**
 * @param {string} id
 * @returns {Promise<User|null>}
 */
async function getUser(id) { /* ... */ }
```

With `"checkJs": true` in `jsconfig.json`, `tsc --noEmit` will check it.

**If using TypeScript:** `"strict": true`, no `any` (use `unknown` and narrow), explicit return types on exported functions.

**MUST** use ES modules (`import`/`export`), not CommonJS, in new code.

**MUST** use `const` by default, `let` where reassigned, never `var`.

**MUST NOT** leave floating promises. Every promise is awaited, returned, or explicitly handled with `.catch()`. Enable `no-floating-promises` if on TypeScript.

**MUST** throw `Error` instances, never strings — strings carry no stack trace.

```javascript
class AppError extends Error {
  constructor(message, { code, cause } = {}) {
    super(message, { cause });
    this.name = this.constructor.name;
    this.code = code;
  }
}
```

`cause` is native since Node 16 and is the JS equivalent of `raise ... from exc`. Use it.

**SHOULD** use `pino` for structured logging in long-running services. `console.log` is fine in a build script or a one-shot CLI; it is not fine in a daemon.

---

## 5. Testing

No coverage percentage target. Coverage numbers get gamed and produce tests for getters while the actual logic goes untested.

**MUST** test: anything with branching logic, anything parsing external input, anything doing arithmetic that matters, and every bug you fix (write the failing test first).

**SHOULD NOT** test: thin wrappers, straight-through delegation, framework glue, and code that only calls out.

**MUST** mock external I/O in unit tests — network, database, filesystem, clock. A unit test that needs the internet is an integration test.

**MUST** make tests deterministic. Freeze time (`freezegun`, `vi.useFakeTimers()`), seed randomness, never depend on test execution order.

**SHOULD** name tests as a sentence describing the behaviour, not the method:

```python
def test_returns_none_when_user_not_found(): ...   # good
def test_get_user_2(): ...                          # useless in a failure report
```

**PREFER** a small number of tests over realistic data to a large number over trivial data.

Tools: `pytest` and `vitest`.

---

## 6. Naming

| Kind | Python | JavaScript |
|---|---|---|
| Variables, functions | `snake_case` | `camelCase` |
| Classes, types | `PascalCase` | `PascalCase` |
| Constants | `UPPER_SNAKE` | `UPPER_SNAKE` |
| Private | `_leading_underscore` | `#privateField` |
| Files | `snake_case.py` | `kebab-case.js` |

**MUST** prefix booleans with `is`, `has`, `can`, or `should`.

**SHOULD** name for the domain, not the type: `vessel_mmsi` beats `id_string`, `retry_delay_seconds` beats `delay`.

**SHOULD** put units in the name where ambiguity is possible: `timeout_seconds`, `size_bytes`, `distance_nm`. This class of bug is expensive and entirely preventable.

Short names are fine where scope is short and meaning is obvious — `i`, `db`, `id`, `fp`. Length should track scope.

---

## 7. Security

**MUST NOT** commit secrets. Ever. `.env` in `.gitignore`, `.env.example` committed with every key present and no real values.

**MUST** use parameterised queries. No string interpolation into SQL, shell commands, or file paths from external input.

```python
cursor.execute("SELECT * FROM vessels WHERE mmsi = ?", (mmsi,))   # yes
cursor.execute(f"SELECT * FROM vessels WHERE mmsi = {mmsi}")      # no
```

**MUST** validate external input at the boundary — user input, API responses, file contents, message payloads. An upstream API changing its schema should produce a clear validation error, not a corrupted row.

**MUST NOT** use `shell=True` in `subprocess` with any interpolated value. Pass a list.

**SHOULD** set explicit timeouts on every network call. The default in most HTTP libraries is no timeout, which hangs forever.

---

## 8. Comments and documentation

**MUST** explain *why*, not *what*. Code states what it does; comments cover what the code cannot.

```python
# Bad
count += 1  # increment count

# Good
# B2 rate-limits at 100 req/s per bucket; batching keeps us under it
# without needing a token bucket.
BATCH_SIZE = 50
```

**SHOULD** docstring anything whose behaviour isn't obvious from name and signature. `def get_user(user_id: str) -> User | None` needs no docstring — a docstring restating the name is noise.

**SHOULD** keep a `README.md` covering: what it is, how to run it, required env vars, how to run the tests. Four sections. This is the one piece of documentation that consistently pays for itself.

**MUST** update comments and docstrings when changing the code they describe. A stale comment is worse than no comment because it is trusted.

---

## Appendix: configuration

`pyproject.toml`:

```toml
[tool.ruff]
line-length = 100
target-version = "py311"

[tool.ruff.lint]
select = ["E", "F", "I", "N", "UP", "B", "S", "A", "C4", "SIM", "RUF"]
ignore = ["S101"]  # assert is fine in tests

[tool.ruff.lint.per-file-ignores]
"tests/*" = ["S", "ARG"]

[tool.mypy]
strict = true
python_version = "3.11"

[[tool.mypy.overrides]]
module = ["untyped_lib.*"]
ignore_missing_imports = true

[tool.pytest.ini_options]
testpaths = ["tests"]
addopts = "-q --strict-markers"
```

Ruff rule sets worth knowing: `B` catches mutable default arguments and similar bugs; `S` is bandit's security checks; `UP` modernises syntax as you upgrade Python; `SIM` catches needlessly convoluted logic.

`eslint.config.js`:

```javascript
import js from "@eslint/js";

export default [
  js.configs.recommended,
  {
    languageOptions: { ecmaVersion: 2023, sourceType: "module" },
    rules: {
      "no-unused-vars": ["error", { argsIgnorePattern: "^_" }],
      "no-console": ["warn", { allow: ["warn", "error"] }],
      "prefer-const": "error",
      eqeqeq: ["error", "always"],
    },
  },
];
```

`.pre-commit-config.yaml` — run `pre-commit autoupdate` to set revisions rather than copying pinned versions, which go stale fast:

```yaml
repos:
  - repo: https://github.com/astral-sh/ruff-pre-commit
    rev: ""   # pre-commit autoupdate fills this
    hooks:
      - id: ruff
        args: [--fix]
      - id: ruff-format
  - repo: https://github.com/pre-commit/pre-commit-hooks
    rev: ""
    hooks:
      - id: end-of-file-fixer
      - id: trailing-whitespace
      - id: check-yaml
      - id: check-added-large-files
      - id: detect-private-key
```

`detect-private-key` and `check-added-large-files` are the two that will actually save you.

---

## Review checklist

Before saying a change is done:

- [ ] Ran it. Saw it work. Not "should work".
- [ ] Tests pass — including ones I didn't touch.
- [ ] `ruff check` / `eslint` clean.
- [ ] No new dependency added without asking.
- [ ] Nothing changed beyond what was asked.
- [ ] No secrets, no hardcoded paths, no `.env` staged.
- [ ] Failures are loud — nothing silently swallowed or defaulted.
- [ ] Comments and docstrings match what the code now does.

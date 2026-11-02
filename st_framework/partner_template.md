# Test Automation Framework Template

- **Purpose:**
	- An architectural and implementation baseline for new Python/pytest test automation frameworks.
	- It covers service/API testing and laboratory instrument automation (PyVISA/SCPI).
- **Origin:**
	- Derived from the review of an existing API test framework (`partner_report.md`).
	- Each rule exists to prevent a problem that was observed in practice; Section 14 maps the rules back to those problems.
- **Out of scope:**
	- Browser/UI automation.
	- Selenium, WebDriver, and related tooling are deliberately excluded and must not be added to a framework built from this template.

---

## 0. How to use this document

| Keyword                     | Meaning                                                                                                  |
|:----------------------------|:---------------------------------------------------------------------------------------------------------|
| **MUST** / **MUST NOT**     | Mandatory standard. Deviations need a written justification in the project README and reviewer approval. |
| **SHOULD** / **SHOULD NOT** | Strong recommendation. Deviate only for a concrete reason.                                               |
| **MAY**                     | Optional. Adopt when it fits the project.                                                                |

Rules are numbered (for example `CFG-3`) so code reviews and pull requests can cite them.

**Applicability tags:** `[ALL]` applies to every project, `[API]` to service/HTTP testing, and `[LAB]` to instrument and laboratory automation.

---

## 1. Core principles

1. **One source of truth.** Each setting, environment name, and secret is resolved once, validated once, and read from one place.
2. **No hidden shared state.** Clients, test data, and parameters are immutable or owned by exactly one fixture. Nothing is mutated "for the next test".
3. **Clients do I/O, tests assert.** Transport and driver layers return results or raise typed errors. They never contain test-outcome logic.
4. **Fail loudly, fail early, fail with context.** Configuration errors fail at startup with a clear message. Runtime errors keep their root cause.
5. **Determinism over retries.** Synchronize on observable state (polling with a deadline). Retries are narrow, bounded, and visible in reports.
6. **Cleanup is guaranteed.** Every acquired resource (session, instrument, created entity) is released in `finally` or by a fixture teardown, including when setup fails halfway.
7. **The framework is code under test.** It has its own unit tests, linters, type checks, and pinned dependencies.
8. **Use the ecosystem.** Use pytest's built-in logging, stash, markers, and xdist instead of hand-rolled singletons, runners, and report parsers.

---

## 2. Recommended project structure `[ALL]`

```
<project>/
├── pyproject.toml              # deps, pytest, ruff, mypy config (single file)
├── README.md                   # setup, config precedence, how to run, markers
├── .env.example                # documented keys, NO values; real .env is git-ignored
├── config/
│   ├── base.yaml               # non-secret defaults
│   ├── dev.yaml | stage.yaml | prod.yaml | lab-<bench>.yaml
│   └── schema/                 # OpenAPI specs or other contracts, versioned locally
├── src/<framework>/            # installable package (pip install -e .)
│   ├── config/                 # Settings model, loader, precedence, validation
│   ├── transport/              # HTTP client, retry policy, contract validation
│   ├── instruments/            # VISA session manager, SCPI base driver, drivers/
│   ├── clients/                # domain/service clients built on transport (composition)
│   ├── data/                   # test-data models, builders, factories
│   ├── logging/                # setup, redaction filter, formatters
│   ├── reporting/              # artifact attachment, report plugins (RP, Slack, ...)
│   ├── utils/                  # polling/wait helpers, ids, time
│   └── pytest_plugin.py        # hooks + core fixtures, registered via entry point
├── tests/                      # product tests ONLY
│   ├── conftest.py             # thin: area-wide fixtures only
│   └── <area>/<feature>/test_*.py
├── framework_tests/            # unit tests for src/<framework> (no network, no hardware)
├── tools/                      # one-off scripts (never imported by the framework)
└── reports/                    # generated, git-ignored
```

**Mandatory rules**

- **ARCH-1 MUST:** Framework code lives in an installable package (`src/` layout). Tests import `<framework>.*`. There is no `sys.path` manipulation anywhere.
- **ARCH-2 MUST:** Dependency direction is `tests → clients → transport/instruments → config/logging/utils`. The framework MUST NOT import from `tests/`, and test modules MUST NOT import from other test modules. Shared helpers belong in the framework package or a `conftest.py`.
- **ARCH-3 MUST:** There is one implementation per concern. Superseded layers (for example an old client next to a new one) MUST be deleted at the end of the migration, not kept "temporarily".
- **ARCH-4 MUST:** There are no wildcard imports (`from x import *`).
- **ARCH-5 SHOULD:** The core fixtures and hooks ship as a pytest plugin (`pytest_plugin.py`, registered through `[project.entry-points.pytest11]`), so root `conftest.py` files stay small.
- **ARCH-6 SHOULD:** Tooling that tests don't need at runtime (test-case exporters, migration scripts) goes in `tools/` with its own optional dependency group.

---

## 3. Module responsibilities `[ALL]`

| Module             | Owns                                                                                                 | Must not                                                                         |
|:-------------------|:-----------------------------------------------------------------------------------------------------|:---------------------------------------------------------------------------------|
| `config`           | Loading, merging, validating, and exposing a typed, read-only `Settings` object                      | Read `os.environ` outside the loader, or mutate settings after startup           |
| `transport` (HTTP) | Sessions, per-request headers, timeouts, retry policy, contract validation, request/response logging | Assert test outcomes, or hold per-test state                                     |
| `instruments`      | VISA resource-manager lifecycle, SCPI I/O, error-queue checks, locking, simulation backend           | Contain test logic, or leave sessions open                                       |
| `clients`          | Domain operations (`create_user`, `set_voltage`) composed from transport/driver objects              | Contain `assert`, or carry flags like `expect_failure=True` that change behavior |
| `data`             | Immutable test-data models, builders, unique-ID generation                                           | Be mutated by fixtures or tests                                                  |
| `logging`          | Logger setup, redaction, formatting                                                                  | Implement singletons or buffer logs privately                                    |
| `reporting`        | Attaching artifacts, publishing results to external systems                                          | Parse reports to decide pass/fail (exit codes decide)                            |
| `pytest_plugin`    | CLI options, settings into `config.stash`, core fixtures, marker registration                        | Hold business logic                                                              |

---

## 4. Configuration and parameter passing `[ALL]`

### 4.1 Precedence (MUST be documented in README)

```
CLI option  >  environment variable  >  config/<env>.yaml  >  config/base.yaml  >  code default
```

- **CFG-1 MUST:** Settings are resolved **once**, in `pytest_configure`, into a single typed, frozen `Settings` object stored in `config.stash`. Everything else reads from that object.
- **CFG-2 MUST:** Validation happens **after** all sources are merged. An override MUST NOT bypass validation (for example a valid CLI value replaced by an unvalidated environment variable).
- **CFG-3 MUST:** When two sources disagree on a safety-relevant setting (target environment, lab bench, destructive mode), the run fails with an explicit message instead of picking one silently.
- **CFG-4 MUST:** Derived values (base URLs, secret names, instrument addresses) come from `Settings`. They MUST NOT be recomputed from `os.environ` in fixtures.
- **CFG-5 MUST:** Booleans from strings go through one parser that accepts `true/false/1/0/yes/no` (case-insensitive) and rejects anything else. Patterns like `os.environ.get("X", False)` are forbidden, because `"false"` is truthy.
- **CFG-6 MUST:** Missing required values fail at startup with a message naming the key and where to set it. Values that only some features need are validated lazily, when the feature is first used, so unrelated runs don't fail.
- **CFG-7 MUST:** Destructive or production targets need an explicit opt-in flag (for example `--allow-prod`). Tests that must never run there are excluded through markers (Section 9).
- **CFG-8 SHOULD:** Use `pydantic-settings` (or a frozen `dataclass` plus an explicit loader) to get types, defaults, and error messages for free.
- **CFG-9 MAY:** Print the resolved, redacted settings in the report header (`pytest_report_header`).

```python
# src/<framework>/pytest_plugin.py
import pytest
from <framework>.config import Settings, load_settings

SETTINGS = pytest.StashKey[Settings]()

def pytest_addoption(parser: pytest.Parser) -> None:
    g = parser.getgroup("framework")
    g.addoption("--env", dest="env", default=None, help="Target environment (dev|stage|prod|lab-*)")
    g.addoption("--allow-prod", action="store_true", default=False)

def pytest_configure(config: pytest.Config) -> None:
    # load_settings merges defaults < yaml < env vars < CLI, then validates (raises on conflict)
    config.stash[SETTINGS] = load_settings(cli_env=config.getoption("env"),
                                           allow_prod=config.getoption("allow_prod"))

@pytest.fixture(scope="session")
def settings(pytestconfig: pytest.Config) -> Settings:
    return pytestconfig.stash[SETTINGS]
```

```python
# src/<framework>/config/settings.py
from pydantic import SecretStr, field_validator
from pydantic_settings import BaseSettings, SettingsConfigDict

class Settings(BaseSettings):
    model_config = SettingsConfigDict(env_prefix="FW_", frozen=True, extra="forbid")

    env: str
    base_url: str | None = None
    api_password: SecretStr | None = None       # SecretStr never prints its value
    http_timeout_s: float = 20.0
    visa_backend: str = "@py"                   # "@sim" for simulation, "" for system VISA
    allow_prod: bool = False

    @field_validator("env")
    @classmethod
    def _known_env(cls, v: str) -> str:
        allowed = {"dev", "stage", "prod"} | {"lab-a", "lab-b"}
        if v.lower() not in allowed:
            raise ValueError(f"Unknown env {v!r}; expected one of {sorted(allowed)}")
        return v.lower()
```

### 4.2 Secrets

- **SEC-1 MUST:** Secrets come only from environment variables or a secret store. They MUST NOT appear in source, test data, or committed config. The local `.env` is git-ignored, and `.env.example` documents the keys.
- **SEC-2 MUST:** Secrets are typed as `SecretStr` (or equivalent) and redacted by a logging filter (Section 7). Secrets SHOULD NOT be placed in URL query strings. When an API requires it, the URL redactor MUST cover that parameter.
- **SEC-3 SHOULD NOT:** Don't build an in-process "encryption" of secrets in environment variables. It adds failure modes without real protection. Rely on CI secret masking and redaction instead.

### 4.3 Test data

- **DATA-1 MUST:** Test data is immutable (frozen dataclasses, `MappingProxyType`, or pydantic models with `frozen=True`). Fixtures that need variations build a **new** object (`dataclasses.replace(...)`, `model_copy(update=...)`).
- **DATA-2 MUST:** Unique identifiers (users, devices, entities) come from one helper that includes a run ID and a worker ID, so parallel runs and leftovers can be identified and swept.
- **DATA-3 SHOULD:** Keep environment-specific data in YAML or JSON validated against the data model, not in executable Python modules.

---

## 5. Fixture design, lifecycle, and resource management `[ALL]`

### 5.1 Rules

- **FIX-1 MUST:** A fixture depends only on fixtures of **equal or broader** scope. Dynamic scopes (`scope=callable`) are allowed only when every dependency supports the broadest possible result. Add a framework test that runs collection under each scope mode.
- **FIX-2 MUST:** A fixture MUST NOT mutate an object it received from another fixture. Copy, then modify.
- **FIX-3 MUST:** Each authenticated identity is its own client instance. Two fixtures MUST NOT re-authenticate the same underlying object with different credentials.
- **FIX-4 MUST:** Setup that acquires several resources registers each one for cleanup **as soon as it is acquired** (an `ExitStack` or a cleanup registry). Partial setup failures don't leak resources.
- **FIX-5 MUST NOT:** Don't use `assert` in teardown. Teardown failures are logged and collected, and reported once as a warning or error, so a passed test isn't turned into an error by a cleanup hiccup. Use a session-level "leak report" for anything not cleaned up.
- **FIX-6 MUST:** Don't duplicate fixtures per scope (`thing` and `func_thing` twins). Use one **factory fixture** and expose scoped wrappers only when needed.
- **FIX-7 SHOULD:** Prefer factory fixtures (`make_user()`) over fixtures that return a single pre-made object. The factory tracks everything it creates and deletes it at teardown.
- **FIX-8 SHOULD:** Parametrize indirect fixtures with `request.getfixturevalue(request.param)` plus an allow-list, not `if/elif` chains.
- **FIX-9 SHOULD:** Do network or hardware discovery (geo-IP lookups, `*IDN?`, spec download) **once per session** in a session fixture. Never do it in a constructor.
- **FIX-10 MAY:** Use `pytest --setup-plan` in CI to catch scope problems early.

### 5.2 Patterns

```python
# Factory fixture with guaranteed cleanup (FIX-4, FIX-5, FIX-7)
import contextlib, logging
log = logging.getLogger(__name__)

@pytest.fixture
def make_user(api_client, run_ids):
    created: list[str] = []

    def _make(**overrides) -> User:
        user = api_client.users.create(UserSpec.build(run_ids, **overrides))
        created.append(user.id)               # registered immediately after acquisition
        return user

    yield _make

    for user_id in reversed(created):
        try:
            api_client.users.delete(user_id)
        except Exception:                      # teardown never asserts
            log.warning("Cleanup failed for user %s", user_id, exc_info=True)
```

```python
# Multi-resource setup without leaks (FIX-4)
@pytest.fixture
def two_factor_user(api_client, make_user):
    with contextlib.ExitStack() as stack:
        t1 = api_client.templates.create(...); stack.callback(api_client.templates.delete, t1.id)
        t2 = api_client.templates.create(...); stack.callback(api_client.templates.delete, t2.id)
        user = make_user()
        yield user                             # ExitStack unwinds even if setup failed midway
```

```python
# Temporary state change that always reverts
@contextlib.contextmanager
def without_auth(client):
    saved = client.auth
    client.auth = None
    try:
        yield client
    finally:
        client.auth = saved
```

---

## 6. Transport and driver layers

### 6.1 HTTP transport `[API]`

- **HTTP-1 MUST:** Request headers are built **fresh for every request** from the client defaults plus per-call overrides. Client defaults are never mutated by a request.
- **HTTP-2 MUST:** Every outbound call has an explicit timeout (connect and read), from `Settings`. That includes auxiliary calls: notification webhooks, report uploads, mock services, and discovery endpoints.
- **HTTP-3 MUST:** Use one `requests.Session` (or `httpx.Client`) per client instance for connection reuse, and close it in fixture teardown.
- **HTTP-4 MUST:** The retry policy is **narrow and explicit**:
  - Retry only on transport errors (connect or read timeout) and on configured status codes (by default `429, 502, 503, 504`).
  - Cap it by attempt count **and** by total time (by default 3 attempts, 30–60 s).
  - Business errors (auth failures, validation errors, rate-limit result codes in the body) are **not** retried by default. A call opts in explicitly.
  - Every retry is logged at `WARNING` with the reason, and counted in the run summary so masked instability stays visible.
  - Retry state (back-off) is per client and MUST NOT be stored in `os.environ` or other global state.
- **HTTP-5 MUST:** Contract validation (OpenAPI or similar):
  - Catch the validator library's **actual** exception types. Add a framework test that feeds a deliberately invalid response and asserts the raised error type and message.
  - Run validation **after** retry classification, so transient gateway errors aren't reported as contract failures.
  - Skipping validation is a per-call, documented argument, never a global mutable flag.
  - Load specs once per session, preferably from a pinned local copy, falling back to a URL.
- **HTTP-6 MUST:** Raise a typed exception hierarchy (`TransportError`, `HttpStatusError(status, body)`, `ContractError(details)`, `RetryExhausted(history)`) and chain causes with `raise ... from err`. Never re-wrap everything as a bare `Exception("Unexpected error")`.
- **HTTP-7 SHOULD:** Clients return a small response object, or the raw response plus helpers. The "expect non-2xx" decision is the test's (`client.get(..., expected_status=401)`), not a set of boolean flags.
- **HTTP-8 SHOULD:** Keep `make_request`-style signatures small (under 8 parameters). Group rarely used options into a `RequestOptions` dataclass.
- **HTTP-9 MAY:** Use `tenacity` for retry policies, `httpx` for HTTP/2 or async, and `respx`/`responses` for transport unit tests.

```python
# Minimal shape of a transport call (HTTP-1, HTTP-2, HTTP-4, HTTP-6)
def request(self, method: str, path: str, *, headers: Mapping[str, str] | None = None,
            json: Any = None, params: Mapping[str, Any] | None = None,
            retry: RetryPolicy | None = None, validate: bool = True) -> Response:
    merged = {**self._default_headers, **(headers or {})}          # never mutates defaults
    policy = retry or self._default_retry
    for attempt in policy.attempts():                               # bounded by count and deadline
        try:
            resp = self._session.request(method, self._url(path), headers=merged, json=json,
                                         params=params, timeout=self._timeout)
        except (requests.ConnectionError, requests.Timeout) as err:
            attempt.fail(err); continue                             # logs WARNING, sleeps back-off
        if policy.should_retry(resp):
            attempt.fail(f"HTTP {resp.status_code}"); continue
        self._log_exchange(resp)
        if validate:
            self._contract.validate(resp)                           # raises ContractError
        return resp
    raise RetryExhausted(policy.history)
```

### 6.2 Instrument automation: PyVISA / SCPI `[LAB]`

- **LAB-1 MUST:** There is exactly one `pyvisa.ResourceManager` per process, owned by a **session fixture** and closed at session end. Instrument sessions are opened through it and closed in teardown (or with context managers). Constructors MUST NOT open sessions implicitly.
- **LAB-2 MUST:** Every resource sets `timeout` (ms) explicitly from `Settings`, plus `read_termination` and `write_termination` for message-based instruments. Long operations (sweeps, calibration, self-test) use a scoped, temporarily raised timeout that is restored afterwards.
- **LAB-3 MUST:** After configuration commands, and on every failure path, the driver checks the instrument error queue (`SYST:ERR?`, drained until `0,"No error"`) and raises a typed `InstrumentError(code, message, command)`. Silent SCPI errors are the lab equivalent of unvalidated API responses.
- **LAB-4 MUST:** Synchronize with `*OPC?` (or status-byte or service-request polling) and deadline-based waits. `time.sleep()` is never used to "let the instrument settle".
- **LAB-5 MUST:** Identify the instrument at session start (`*IDN?`) and compare it with the expected model or firmware from config. Fail fast on a mismatch. Record IDN, serial, and firmware in the report header.
- **LAB-6 MUST:** Put the instrument in a known state at start (`*RST; *CLS`, or a documented project preset) and in a **safe state** at teardown: outputs off, sources at zero, interlocks respected. The safe-state teardown runs even if the test or setup failed.
- **LAB-7 MUST:** Catch `pyvisa.errors.VisaIOError` at the driver boundary and re-raise it as a framework `InstrumentIOError` with the resource name and command attached, chaining the cause.
- **LAB-8 MUST:** A physical instrument is used by one worker at a time. Under parallel execution, use exclusive VISA locks or a cross-process file lock keyed by resource name. Otherwise, mark hardware tests to run in a single dedicated worker group.
- **LAB-9 SHOULD:** Drivers expose typed, unit-aware methods (`set_voltage(volts: float)`) built on a small `ScpiInstrument` base. Raw `write`/`query` stays an escape hatch, not the main API.
- **LAB-10 SHOULD:** Support a simulation backend (`pyvisa-sim` with a YAML device description, `visa_backend="@sim"`) so driver logic and framework tests run in CI without hardware.
- **LAB-11 SHOULD:** Retry transient bus errors (for example timeouts on LAN instruments) a small, bounded number of times, with a reconnect step. Never retry commands that are not idempotent (output enable, trigger) without re-establishing a known state first.
- **LAB-12 MAY:** Log every SCPI exchange at `DEBUG` (`>> MEAS:VOLT?` / `<< 1.2034`), and attach the transcript to failing tests.

```python
# src/<framework>/instruments/base.py
import contextlib, logging
import pyvisa
from pyvisa.errors import VisaIOError

log = logging.getLogger(__name__)

class InstrumentError(RuntimeError): ...
class InstrumentIOError(InstrumentError): ...

class ScpiInstrument:
    def __init__(self, resource: pyvisa.resources.MessageBasedResource, name: str):
        self._res, self.name = resource, name

    def write(self, cmd: str) -> None:
        log.debug(">> %s %s", self.name, cmd)
        try:
            self._res.write(cmd)
        except VisaIOError as err:
            raise InstrumentIOError(f"{self.name}: write {cmd!r} failed") from err

    def query(self, cmd: str) -> str:
        try:
            reply = self._res.query(cmd).strip()
        except VisaIOError as err:
            raise InstrumentIOError(f"{self.name}: query {cmd!r} failed") from err
        log.debug("<< %s %s", self.name, reply)
        return reply

    def check_errors(self, context: str = "") -> None:
        errors = []
        for _ in range(50):                                   # bounded drain
            code, _, msg = self.query("SYST:ERR?").partition(",")
            if int(code) == 0:
                break
            errors.append(f"{code}: {msg}")
        if errors:
            raise InstrumentError(f"{self.name} {context}: {'; '.join(errors)}")

    def wait_complete(self) -> None:
        self.query("*OPC?")                                   # blocks until operations finish

    @contextlib.contextmanager
    def timeout(self, ms: int):
        old = self._res.timeout
        self._res.timeout = ms
        try:
            yield
        finally:
            self._res.timeout = old

    def safe_state(self) -> None:                             # override per driver: outputs off, etc.
        self.write("*RST")
```

```python
# Fixture lifecycle (LAB-1, LAB-5, LAB-6)
@pytest.fixture(scope="session")
def visa_rm(settings):
    rm = pyvisa.ResourceManager(settings.visa_backend)
    yield rm
    rm.close()

@pytest.fixture(scope="session")
def psu(visa_rm, settings):
    res = visa_rm.open_resource(settings.psu.address, timeout=settings.psu.timeout_ms,
                                read_termination="\n", write_termination="\n")
    dev = PowerSupply(res, name="psu")
    try:
        dev.verify_identity(expected=settings.psu.model)
        dev.write("*RST;*CLS"); dev.check_errors("init")
        yield dev
    finally:
        try:
            dev.safe_state()
        except InstrumentError:
            log.error("PSU safe-state failed", exc_info=True)
        res.close()
```

---

## 7. Logging `[ALL]`

- **LOG-1 MUST:** Use the standard `logging` module with module-level loggers (`logging.getLogger(__name__)`). There are no custom logger singletons, and no private log buffering or replay. pytest already captures logs per test and shows them for failures.
- **LOG-2 MUST:** Configure output through pytest options in `pyproject.toml` (`log_level`, `log_file`, `log_file_level`, `log_cli`). Library code never adds handlers. Only the plugin or entry point configures handlers.
- **LOG-3 MUST:** A redaction `logging.Filter` is installed on **every** handler, including file, console, and external reporters. It masks configured secret values and sensitive keys in JSON, form data, headers, and URL query strings. If redaction fails, the record is dropped with a warning; the secret is never sent through.
- **LOG-4 MUST:** Use standard levels only, consistently:
  - `DEBUG` for wire-level detail (HTTP bodies, SCPI transcripts).
  - `INFO` for test steps and domain actions.
  - `WARNING` for retries, slow operations, and cleanup failures.
  - `ERROR` for failures with context.
  Custom numeric levels SHOULD NOT be used.
- **LOG-5 MUST:** Use lazy formatting (`log.info("Created %s", user_id)`). Never use `print()` in framework or test code. CI lint enforces this with ruff `T20`.
- **LOG-6 MUST:** Under parallel execution, every log file or artifact path includes the worker ID (`PYTEST_XDIST_WORKER`) and is created with `pathlib`. Processes never truncate or append to a shared file.
- **LOG-7 SHOULD:** Include the test node ID in log records, through a `LoggerAdapter` or a filter populated from a `pytest_runtest_setup` hook, so the merged worker logs can be filtered per test.
- **LOG-8 SHOULD:** Write request/response exchanges as a reproducible artifact (a redacted curl command or a SCPI transcript) **per test**, and attach it only to failed tests.

```toml
# pyproject.toml
[tool.pytest.ini_options]
log_level = "INFO"
log_format = "%(asctime)s %(levelname)-7s %(name)s: %(message)s"
log_file = "reports/run.log"          # plugin rewrites to reports/run-<worker>.log under xdist
log_file_level = "DEBUG"
```

```python
class RedactingFilter(logging.Filter):
    def __init__(self, secrets: Iterable[str], keys: Iterable[str]):
        super().__init__()
        self._values = [s for s in secrets if s]
        self._key_re = re.compile(rf'("?({"|".join(map(re.escape, keys))})"?\s*[:=]\s*)("[^"]*"|[^&\s,]+)', re.I)

    def filter(self, record: logging.LogRecord) -> bool:
        msg = record.getMessage()
        for v in self._values:
            msg = msg.replace(v, "[REDACTED]")
        record.msg, record.args = self._key_re.sub(r"\1[REDACTED]", msg), None
        return True
```

---

## 8. Reporting and artifacts `[ALL]`

- **REP-1 MUST:** **The pytest exit code is the source of truth.** CI fails when any test run (including any worker or shard) exits non-zero, or when an expected report is missing. Summaries and notifications are informational only.
- **REP-2 MUST:** Produce one machine-readable report format per run: JUnit XML (`--junitxml`) by default. Pick one JSON plugin if JSON is needed, and use it everywhere. Parsers MUST NOT assume a structure from a different plugin.
- **REP-3 MUST NOT:** Don't overwrite `report.nodeid` or other pytest identity fields. Descriptions, ticket IDs, and requirement links go in `user_properties`, markers, or the reporting tool's attributes.
- **REP-4 MUST:** Failure details include:
  - The assertion message.
  - The last request/response or SCPI exchange (redacted).
  - The resolved environment or bench.
  - For lab runs, instrument identity.
  Attach these as artifacts from a `pytest_runtest_makereport` hook.
- **REP-5 MUST:** Notification and report-portal integrations are optional plugins, enabled by explicit flags. Their failure (network, auth) is logged as a warning and never changes the test outcome or crashes the session. They make no calls when disabled, and they don't require their credentials when disabled.
- **REP-6 MUST:** The run summary includes counts of passed, failed, error, skipped, xfailed, **retries performed**, and **cleanup failures**, so instability is visible even when everything passes.
- **REP-7 SHOULD:** Use `pytest-html` or Allure for human-readable reports, ReportPortal (`pytest-reportportal`) for history and trends, and a single summary webhook post at the end of the run.
- **REP-8 SHOULD:** Archive `reports/` (logs, JUnit XML, artifacts, measurement data for lab runs) as CI artifacts with a retention policy.
- **REP-9 MAY:** For lab projects, store raw measurement data (CSV or Parquet with units and instrument metadata) next to the report, so results can be re-analyzed without re-running.

```python
@pytest.hookimpl(hookwrapper=True)
def pytest_runtest_makereport(item, call):
    outcome = yield
    report = outcome.get_result()
    if report.when == "call" and report.failed:
        exchange = item.stash.get(LAST_EXCHANGE, None)       # set by the transport/driver layer
        if exchange:
            report.sections.append(("last exchange", exchange.redacted()))
```

---

## 9. Test design and reliability `[ALL]`

### 9.1 Organization

- **TST-1 MUST:** `tests/` mirrors the product: area → feature → endpoint or capability. One test module per feature, and test names that describe behavior (`test_disabled_user_cannot_log_in`).
- **TST-2 MUST:** Enable `--strict-markers` and `--strict-config`. All markers are registered once, in `pyproject.toml`, with a description. Use a small, stable taxonomy: `smoke`, `regression`, `destructive`, `slow`, `hardware`, `prod_safe`, `env(...)`. Don't create one marker per ticket; ticket IDs go in `user_properties` or a single `jira("ID")` marker.
- **TST-3 MUST:** Environment and bench gating uses marker arguments evaluated against `Settings` in `pytest_collection_modifyitems` (for example `@pytest.mark.env(exclude=["prod"])`). Don't hand-code `if env == ...` skips inside tests.

### 9.2 Isolation and assertions

- **TST-4 MUST:** Each test is independent and order-agnostic. It creates what it needs through factories and leaves nothing behind (factory teardown plus a session-level sweep for leftovers tagged with the run ID). Verify periodically with `pytest-randomly` (or `-p random_order`).
- **TST-5 MUST:** Assertions live in tests or in dedicated assertion helpers (`assert_subscriber_matches(expected, actual)`), never in clients or drivers. Non-trivial assertions include a message with the actual values.
- **TST-6 SHOULD:** Use `pytest.mark.parametrize` with `ids=` for readable case names. Use indirect parametrization for cross-cutting dimensions (auth method, instrument model).
- **TST-7 SHOULD:** Comparisons of numeric measurements use explicit tolerances (`pytest.approx(expected, rel=..., abs=...)`) taken from the spec. Tolerances are never tuned until a test passes.

### 9.3 Waiting, flakiness, and skips

- **TST-8 MUST NOT:** Don't use `time.sleep()` for synchronization in tests or fixtures. Use a single `wait_until(predicate, timeout, interval, description)` helper that raises a descriptive `TimeoutError`. Allowed exception: deliberate time-based behavior (token expiry), and then the duration comes from config, is as short as the system allows, and the test is marked `slow`.
- **TST-9 MUST:** Polling external systems (mailboxes, queues, instrument status) reuses one connection per wait, uses back-off (for example 0.5 s → 5 s), and has a deadline. It never opens a new connection on every iteration.
- **TST-10 MUST:** Every `skip` and `xfail` carries a reason **with a ticket reference**. Placeholder reasons ("TBD", "investigate") are rejected in review. Use `xfail(strict=True)` for known bugs, so a fix is noticed.
- **TST-11 MUST:** Reruns (`pytest-rerunfailures`) are allowed only with a `flaky` marker that names a ticket. A global `--reruns` setting is not allowed. Report rerun counts (REP-6). Review flaky tests on a fixed cadence and either fix them or quarantine them into a separate, non-gating job.
- **TST-12 SHOULD:** Tests that exceed a duration threshold are marked `slow` and excluded from the default pull-request job. Use `pytest-timeout` with a global default (for example 300 s) so a hung test fails instead of blocking CI.

```python
def wait_until(predicate: Callable[[], T | None], *, timeout: float, interval: float = 0.5,
               max_interval: float = 5.0, what: str) -> T:
    deadline = time.monotonic() + timeout
    while True:
        result = predicate()
        if result:
            return result
        if time.monotonic() >= deadline:
            raise TimeoutError(f"Timed out after {timeout}s waiting for {what}")
        time.sleep(interval)
        interval = min(interval * 2, max_interval)
```

---

## 10. Parallel execution and runners `[ALL]`

- **PAR-1 MUST:** Use `pytest-xdist` (`-n auto` / `--dist loadgroup`) for parallelism. Don't write a custom thread or subprocess runner that splits by folder and merges reports by hand.
- **PAR-2 MUST:** Tests that must not run concurrently (shared accounts, rate-limited endpoints, physical instruments) use `@pytest.mark.xdist_group("<resource>")`, or run in a separate serial job. "Run last" ordering is expressed through markers and job configuration, never through hard-coded path lists.
- **PAR-3 MUST:** Session-wide one-time setup under xdist (creating shared entities, downloading specs) uses a file lock (`filelock`) plus a cache in `tmp_path_factory.getbasetemp().parent`, so it runs once across workers.
- **PAR-4 MUST:** If a wrapper script is still needed (for example to choose suites in CI), it passes through pytest's exit code, applies a total timeout, and never reports success when a shard produced no results.
- **PAR-5 SHOULD:** Hardware tests run with one worker per bench (or `xdist_group` per instrument). API tests may scale freely.

---

## 11. Coding conventions `[ALL]`

**Mandatory**

- **CODE-1 MUST:** Python ≥ 3.11. Type hints on all public functions, methods, and fixtures in `src/`. `mypy --strict` (or pyright) passes on `src/`. Tests are at least type-checked for imports.
- **CODE-2 MUST:** `ruff` (lint and format) is enforced in pre-commit and CI. The baseline rule set is `E, F, W, I, B, UP, SIM, PT, T20, S, BLE, RUF`. Key rules:
  - `B006`: no mutable default arguments.
  - `BLE001`: no blind `except Exception` without re-raise or logging.
  - `T20`: no `print`.
  - `PT`: pytest style.
- **CODE-3 MUST:** Use PEP 8 naming: `snake_case` functions, `PascalCase` classes, `UPPER_CASE` constants. Don't mix styles. Avoid double-underscore name mangling; use a single leading underscore for internals.
- **CODE-4 MUST:** Abstract bases use `abc.ABC` and `@abstractmethod`, so a missing implementation fails at instantiation instead of silently doing nothing. Prefer `typing.Protocol` for interfaces that tests or fakes implement.
- **CODE-5 MUST:** Prefer **composition over inheritance**. A client aggregates sub-clients (`api.users`, `api.sessions`) that share one transport. Multiple inheritance from more than one concrete class is not allowed.
- **CODE-6 MUST:** Exceptions:
  - Use a framework exception hierarchy rooted at `FrameworkError`.
  - Catch only the exceptions you can handle.
  - Re-raise with `from err`.
  - Never swallow exceptions silently. `except: pass` is forbidden.
  - Don't replace an informative exception with a generic `pytest.fail("X failed")`; use `pytest.fail(..., pytrace=True)` with the original chained, or let it propagate.
- **CODE-7 MUST:** No I/O in constructors (network, file, VISA). Construction is cheap and side-effect free. Acquisition happens in explicit `open()`/`connect()` calls or in fixtures.
- **CODE-8 MUST:** No module-level mutable globals for runtime state (counters, delay managers, caches keyed by instances). State lives in objects owned by fixtures. `functools.cache` is used only on pure functions with hashable, long-lived arguments, never on instance methods.
- **CODE-9 MUST:** Use `pathlib.Path` for all file paths, and one `PROJECT_ROOT` constant defined in a single module.
- **CODE-10 MUST:** No dead code. Commented-out blocks, `if 1 == 0:` blocks, unused fixtures, and unused helpers are deleted; history is in git. CI enforces this with `vulture` or ruff `ERA`.

**Recommended**

- **CODE-11 SHOULD:** Functions stay under about 50 lines with 6 or fewer parameters. Replace boolean-flag parameters that switch behavior with separate methods or an options object.
- **CODE-12 SHOULD:** Every public module, class, and fixture has a short docstring explaining *what it provides and its lifecycle*. Docstrings are documentation, never report identifiers.
- **CODE-13 MAY:** Use `dataclasses` or pydantic models for request and response payloads in domain clients, and parse responses into models at the client boundary.

---

## 12. Dependencies, quality gates, and CI `[ALL]`

- **QA-1 MUST:** All dependencies are declared once, in `pyproject.toml`, with a lock file (`uv.lock`, `poetry.lock`, or `pip-tools`). There are no duplicate or unpinned runtime entries. Every directly imported package is declared, including transitive ones you import (for example `jsonschema`). Use optional groups for `lab` (`pyvisa`, `pyvisa-py`, `pyvisa-sim`), `reporting`, and `dev`.
- **QA-2 MUST:** `framework_tests/` covers the framework itself, with no network and no hardware:
  - Config precedence and validation, including the "conflict fails" case.
  - The boolean parser.
  - The retry policy (retries on the right statuses, stops at the cap, no retry on business errors).
  - Contract-validation error types.
  - Redaction (secrets in JSON, form data, headers, and URLs).
  - Header immutability across requests.
  - Fixture scope compatibility (collection under every scope mode).
  - SCPI driver behavior against `pyvisa-sim` (error-queue drain, timeout restore, safe state).
- **QA-3 MUST:** The CI pipeline per pull request is:
  1. ruff lint and format check.
  2. mypy.
  3. `framework_tests`.
  4. `pytest --collect-only --strict-markers` on `tests/`.
  5. The smoke suite against a non-production target.

  Full regression and hardware suites run on schedule or on demand.
- **QA-4 MUST:** CI configuration lives in the repository and is code-reviewed like everything else.
- **QA-5 SHOULD:** Use pre-commit hooks for ruff, mypy (fast mode), secret scanning (`detect-secrets` or `gitleaks`), and end-of-file and whitespace fixes.
- **QA-6 SHOULD:** Enable Dependabot or Renovate for dependency updates, and run `pip-audit` in CI.

```toml
# pyproject.toml (excerpt)
[project.optional-dependencies]
lab = ["pyvisa>=1.14", "pyvisa-py>=0.7", "pyvisa-sim>=0.6"]
dev = ["ruff", "mypy", "pytest-randomly", "pytest-timeout", "pytest-xdist", "pre-commit"]

[tool.pytest.ini_options]
addopts = "--strict-markers --strict-config -ra"
testpaths = ["tests"]
timeout = 300
markers = [
  "smoke: fast, safe checks run on every PR",
  "destructive: creates/deletes data or drives outputs; never on prod",
  "hardware: requires a physical instrument",
  "slow: > 60 s; excluded from PR runs",
  "env(include=None, exclude=None): environment gating",
  "flaky(ticket): known-flaky test; allowed reruns, tracked",
]

[tool.ruff.lint]
select = ["E","F","W","I","B","UP","SIM","PT","T20","S","BLE","RUF","ERA"]
```

---

## 13. Definition of done for a new framework

A framework built from this template is ready for production use when all of these are true:

- [ ] `pip install -e .[dev]` and one documented command run the smoke suite. The README documents the configuration precedence and every setting.
- [ ] Settings are resolved once, typed, frozen, and validated. Conflicting environment sources fail the run. Production needs an explicit opt-in.
- [ ] No secrets in the repository. Redaction is verified by framework tests. Secret scanning runs in pre-commit and CI.
- [ ] Every outbound call (HTTP, VISA, IMAP, webhooks) has a timeout. The retry policy is bounded, narrow, logged, and counted.
- [ ] Fixtures pass a scope-compatibility check. No fixture mutates another fixture's object. Every created resource is cleaned up with `finally`, `ExitStack`, or factory teardown.
- [ ] Instruments: one ResourceManager per process, identity verified, error queue checked, safe state guaranteed at teardown, a simulation backend available in CI.
- [ ] Logging uses the stdlib and pytest's capture, with per-worker files and no `print`.
- [ ] CI is gated on the pytest exit code. Reports are in one format and archived. Retries and cleanup failures appear in the summary.
- [ ] `--strict-markers` is on. Every skip, xfail, and flaky mark references a ticket. There are no hard sleeps outside documented time-based tests.
- [ ] ruff, mypy, and `framework_tests` pass in CI. Dependencies are pinned through a lock file.

---

## 14. Lessons learned: source findings mapped to standards

| Problem observed in the reviewed framework                                                                                      | Prevented by                   |
|:--------------------------------------------------------------------------------------------------------------------------------|:-------------------------------|
| CLI environment validated, then silently overridden by an environment variable; secrets derived from a different source         | CFG-1, CFG-2, CFG-3, CFG-4     |
| `os.environ.get("FLAG", False)` treated `"false"` as enabled                                                                    | CFG-5                          |
| Unused integrations still required their secrets, and a missing local env file crashed import                                   | CFG-6, REP-5                   |
| Request code mutated the client's default headers, so overrides leaked into later requests                                      | HTTP-1, FIX-2                  |
| Blanket 25-attempt / 300 s retry, including business errors, with global delay state in `os.environ`                            | HTTP-4, CODE-8, REP-6          |
| Validation caught the wrong exception type and ran before retry classification                                                  | HTTP-5, HTTP-6, QA-2           |
| Network call with no timeout inside a client constructor                                                                        | HTTP-2, CODE-7, FIX-9          |
| `@abstractmethod` without `ABC`, so teardown called an empty stub                                                               | CODE-4                         |
| Five-way multiple inheritance "god" client; assertions embedded in client methods via boolean flags                             | CODE-5, HTTP-7, TST-5, CODE-11 |
| `thing`/`func_thing` fixture twins, with copy-paste wiring to the wrong scoped dependency                                       | FIX-6, FIX-7, QA-2             |
| Dynamic session scope with function-scoped dependencies (`ScopeMismatch`)                                                       | FIX-1, FIX-10                  |
| Several login fixtures re-authenticated one shared client; parameter dicts mutated in place; test data held in a mutable module | FIX-2, FIX-3, DATA-1           |
| Resources created before `try`; asserts in teardown; created users never deleted                                                | FIX-4, FIX-5, TST-4            |
| Custom logger singleton whose buffered logs were never emitted                                                                  | LOG-1, LOG-2                   |
| Shared debug file truncated and appended by parallel processes; backslash paths broken on POSIX                                 | LOG-6, CODE-9                  |
| `report.nodeid` overwritten with docstrings                                                                                     | REP-3, CODE-12                 |
| Two JSON report plugins with incompatible structures                                                                            | REP-2                          |
| Custom folder runner: wrong "run last" path, exit codes ignored, success reported when a shard produced no report               | PAR-1, PAR-2, PAR-4, REP-1     |
| 101 hard sleeps (up to 15 min); IMAP re-login every 0.1 s                                                                       | TST-8, TST-9, TST-12           |
| 160 skips (many "TBD"), unregistered markers, one marker per ticket                                                             | TST-2, TST-10, TST-11          |
| Framework importing from tests; legacy and new API layers coexisting; wildcard imports; `sys.path` hacks                        | ARCH-1 to ARCH-4               |
| Duplicated or unpinned requirements, missing direct dependencies, no lint, type, or self-tests                                  | QA-1 to QA-4                   |
| (Lab, preventive) Unchecked SCPI errors, open sessions, sleeps for settling, shared instruments in parallel runs                | LAB-1 to LAB-12, PAR-5         |

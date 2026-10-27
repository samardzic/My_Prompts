# Test Automation Framework Template

**Purpose:** an architectural and implementation baseline for new Python/pytest test automation projects, covering general system tests, API/service tests, and laboratory equipment automation (PyVISA/SCPI).
**Derived from:** lessons recorded in `snd_report.md`. Every standard here exists to prevent a problem observed in practice (see [Appendix A](#appendix-a--lessons-learned-traceability)).
**Applies to:** Python ≥ 3.11 and pytest ≥ 8. Browser and UI automation is out of scope.

## How to read this document

| Keyword | Meaning |
|---|---|
| **MUST** / **MUST NOT** | Mandatory standard. A project that deviates needs a written, reviewed justification in its README. |
| **SHOULD** / **SHOULD NOT** | Strong recommendation. Deviate only for a concrete reason. |
| **MAY** | Optional. Adopt when the project's size or needs justify it. |

Mandatory items are also collected in the [compliance checklist](#13-compliance-checklist).

---

## 1. Core principles

1. **Portable by default.** A fresh clone runs on any machine or CI agent with one install command and one config selection. Nothing depends on an absolute path, a user name, or a personal account.
2. **Fail loudly, early, and specifically.** Misconfiguration fails at session start with a clear message. Errors keep their type and cause. Nothing returns `None` to signal failure.
3. **Every resource has an owner.** Every opened instrument, session, file, process or created test entity is released by a fixture teardown or context manager, *including when the test fails*.
4. **Importing does nothing.** Importing a module never opens ports, creates directories, starts processes, reads secrets or prints.
5. **Tests state intent; the framework does the work.** Tests read as specifications. Communication, waiting, retries and cleanup belong to the framework layers.
6. **Observable by design.** Every run leaves a log, a machine-readable result and failure diagnostics, without anyone editing the code.
7. **Secrets never enter the repository.**

---

## 2. Project structure

### 2.1 Recommended layout

```
<project>/
├── pyproject.toml              # MUST: metadata, deps, pytest/ruff/mypy config
├── README.md                   # MUST: setup, run, configure, add a test
├── .gitignore                  # MUST: artifacts/, .env, caches, venvs
├── .pre-commit-config.yaml     # SHOULD: ruff, mypy, secret scanner
├── .env.example                # MUST if secrets are used: names only, no values
├── config/
│   ├── default.toml            # shared defaults
│   └── environments/
│       ├── sim.toml            # simulated instruments, used by CI on every commit
│       ├── bench-01.toml       # one file per physical bench or target environment
│       └── staging.toml
├── src/<fw_name>/              # the framework, an installable package
│   ├── config.py               # load, merge, validate → typed Settings
│   ├── paths.py                # root-relative paths; artifact dir factory (no side effects)
│   ├── errors.py               # exception hierarchy
│   ├── logging_setup.py        # one logging configuration entry point
│   ├── waits.py                # wait_until / retry helpers
│   ├── instruments/            # instrument drivers (PyVISA/SCPI)
│   │   ├── base.py             # ScpiInstrument base class
│   │   ├── power_supply.py
│   │   ├── multimeter.py
│   │   └── registry.py         # config "driver" name → class
│   ├── clients/                # non-instrument interfaces: REST, DUT console, DB, mail
│   ├── flows/                  # multi-step domain operations composed from drivers/clients
│   ├── data/                   # test-data builders, unique-ID factories
│   └── reporting/              # measurement records, result writers
├── tests/
│   ├── conftest.py             # root: options, settings, logging, artifacts, hooks
│   ├── plugins/                # fixture modules registered via pytest_plugins
│   ├── framework/              # self-tests of the framework (sim backend, no hardware)
│   └── <feature_area>/test_*.py
├── tools/                      # scripts and utilities; NOT collected by pytest
└── artifacts/                  # gitignored; one sub-directory per run
```

### 2.2 Module responsibilities and dependency rules

| Layer | Responsibility | May import | MUST NOT |
|---|---|---|---|
| `tests/` | Express test intent, parameters and assertions | everything below | contain protocol strings, raw I/O, sleeps, or cleanup outside fixtures |
| `tests/conftest.py`, `tests/plugins/` | Wire configuration to objects; own lifecycles | framework package | contain business logic or measurement logic |
| `flows/` | Reusable multi-step operations (e.g. "power-cycle DUT and wait for boot") | drivers, clients, waits | assert test outcomes; import `pytest` |
| `instruments/`, `clients/` | One class per device or service; encapsulate protocol details | config types, errors, waits | import `pytest`, test modules or `conftest`; read CLI options; open resources at import |
| `config.py`, `paths.py`, `errors.py`, `waits.py` | Foundations | stdlib, small third-party libraries | import anything above them |

**MUST**
- Dependencies flow downward only. A driver never inherits from or imports a test base class, fixture or `conftest`.
- The framework is an installable package (`pip install -e .`). Tests import it by package name, never through `sys.path` manipulation or the current working directory.
- `tests/` contains only collected tests, fixtures and test data. Experiments go to `tools/` or a branch.

**SHOULD**
- Keep one driver class per instrument model or family, plus a common base.
- Introduce `flows/` as soon as two tests repeat the same sequence of three or more steps.

---

## 3. Tooling and dependencies

**MUST**
- Declare dependencies in `pyproject.toml`, and pin them with a lock or constraints file (`uv lock`, `pip-tools` or `poetry.lock`).
- List only libraries the code imports. Review unused dependencies at every upgrade.
- Pick exactly one tool per concern (one parallelisation plugin, one HTML reporter, one HTTP client).
- Run `ruff` (lint and format) and `mypy` in CI. The build fails on errors.

**SHOULD**
- Update dependencies at least quarterly. Do not let pins drift for years.
- Run a secret scanner (`gitleaks` or `detect-secrets`) in pre-commit and in CI.

**Baseline `pyproject.toml` excerpt**

```toml
[project]
name = "fw_name"
requires-python = ">=3.11"
dependencies = [
  "pytest>=8",
  "pyvisa>=1.13",
  "pyvisa-py>=0.7",        # pure-Python VISA backend (no vendor VISA needed)
]

[project.optional-dependencies]
sim = ["pyvisa-sim"]
dev = ["ruff", "mypy", "pre-commit"]

[tool.pytest.ini_options]
testpaths = ["tests"]
addopts = "--strict-markers --strict-config -ra"
xfail_strict = true
markers = [
  "hardware: requires physical instruments",
  "requires(*names): instruments the test needs",
  "slow: runtime > 60 s",
]
```

Notes:
- `--strict-markers` and `--strict-config` turn typos into errors.
- `xfail_strict` makes an unexpectedly passing `xfail` fail, so stale markers get noticed.
- Do **not** put environment selection, report paths or machine-specific values in `addopts`.

---

## 4. Configuration and parameter passing

### 4.1 Precedence model

There is one well-defined precedence chain, applied in one place (`config.py`):

```
config/default.toml  <  config/environments/<env>.toml  <  environment variables  <  CLI options
```

**MUST**
- Load and validate configuration **once per session** into an immutable, typed `Settings` object. Every consumer receives values from that object. No module reads config files, environment variables or CLI options on its own.
- Every value that changes between environments (addresses, VISA resource strings, endpoints, limits, accounts) lives in config. Do not hardcode it in code.
- Unknown environments, missing files, missing keys and wrong types raise `ConfigError` at session start, naming the file and the key.
- Every CLI option has a meaningful `help` text, and its default is the *effective* default. Nothing in `addopts` silently overrides it.
- Derive paths from the package or repository root (`Path(__file__).resolve()`), or from `pytestconfig.rootpath`. Never use absolute or user-specific paths.

**MUST NOT**
- Commit credentials, tokens, client secrets or personal e-mail addresses. Config files reference secrets by **name**; values come from environment variables or the CI secret store.
- Keep config sections the code never reads. Remove dead config, or the code that should be using it.

### 4.2 Typed settings

```python
# src/fw_name/config.py
from __future__ import annotations

import os
import tomllib
from dataclasses import dataclass
from pathlib import Path

from fw_name.errors import ConfigError

CONFIG_DIR = Path(__file__).resolve().parents[2] / "config"


@dataclass(frozen=True)
class InstrumentConfig:
    driver: str                      # key into instruments.registry
    resource: str                    # VISA resource string, e.g. "TCPIP0::10.0.0.5::INSTR"
    timeout_ms: int = 5000
    read_termination: str = "\n"
    write_termination: str = "\n"


@dataclass(frozen=True)
class Settings:
    env: str
    simulate: bool
    instruments: dict[str, InstrumentConfig]


def secret(name: str) -> str:
    """Return a secret from the environment, failing clearly if absent."""
    value = os.environ.get(name)
    if not value:
        raise ConfigError(f"Required secret '{name}' is not set in the environment")
    return value


def _merge(base: dict, override: dict) -> dict:
    merged = dict(base)
    for key, value in override.items():
        if isinstance(value, dict) and isinstance(merged.get(key), dict):
            merged[key] = _merge(merged[key], value)
        else:
            merged[key] = value
    return merged


def _read(path: Path) -> dict:
    if not path.is_file():
        raise ConfigError(f"Config file not found: {path}")
    with path.open("rb") as fh:
        return tomllib.load(fh)


def available_envs() -> list[str]:
    return sorted(p.stem for p in (CONFIG_DIR / "environments").glob("*.toml"))


def load_settings(env: str, *, simulate: bool | None = None) -> Settings:
    raw = _merge(_read(CONFIG_DIR / "default.toml"),
                 _read(CONFIG_DIR / "environments" / f"{env}.toml"))
    try:
        instruments = {name: InstrumentConfig(**cfg)
                       for name, cfg in raw.get("instruments", {}).items()}
        return Settings(
            env=env,
            simulate=raw.get("simulate", False) if simulate is None else simulate,
            instruments=instruments,
        )
    except TypeError as exc:   # unknown or missing keys in an instrument table
        raise ConfigError(f"Invalid configuration for env '{env}': {exc}") from exc
```

```toml
# config/environments/bench-01.toml
simulate = false

[instruments.psu]
driver = "keysight_e36xx"
resource = "TCPIP0::192.168.10.21::inst0::INSTR"
timeout_ms = 3000

[instruments.dmm]
driver = "keysight_344xx"
resource = "USB0::0x2A8D::0x0101::MY12345678::INSTR"
```

**SHOULD**
- Use `pydantic` or `attrs` validators once settings grow beyond a few flat fields.
- Identify serial and USB devices by stable identity (VISA resource with serial number, or USB VID/PID **and** serial number), not by enumeration order or a "first match" scan.

**MAY**
- Support a `--set key=value` CLI option for ad-hoc overrides, parsed into the same merge step.

### 4.3 CLI options

```python
# tests/conftest.py
import os

import pytest

from fw_name.config import Settings, available_envs, load_settings


def pytest_addoption(parser: pytest.Parser) -> None:
    group = parser.getgroup("framework")
    group.addoption("--env", default=os.environ.get("TEST_ENV", "sim"),
                    choices=available_envs(),
                    help="Target environment (config/environments/<env>.toml). Default: $TEST_ENV or 'sim'.")
    group.addoption("--simulate", action="store_true", default=None,
                    help="Force the simulated VISA backend regardless of environment config.")


@pytest.fixture(scope="session")
def settings(pytestconfig: pytest.Config) -> Settings:
    return load_settings(pytestconfig.getoption("--env"),
                         simulate=pytestconfig.getoption("--simulate"))
```

---

## 5. Fixture design and resource lifecycle

### 5.1 Rules

**MUST**
- Any fixture that acquires a resource uses `yield` and releases the resource in a `finally` block, or delegates to a context manager. A fixture never returns an open resource without a teardown.
- Acquire and release a resource in the **same** fixture. Do not open in one fixture and close in another.
- Choose scope deliberately:

  | Scope | Use for |
  |---|---|
  | `session` | settings, VISA `ResourceManager`, instrument connections, authenticated API sessions |
  | `module` / `class` | expensive shared state that tests in that file treat as read-only |
  | `function` (default) | anything a test mutates: instrument state, created entities, temp dirs |

- When a long-lived connection is shared, a **function-scoped** fixture restores a known state before each test (reset and configure) and makes the device safe after it.
- Register cleanup of entities a test creates (users, files, DUT configuration) **at creation time** through a fixture. Never write cleanup as the last statements of a test body.
- Per-test files go to `tmp_path` or a per-test artifact directory, never to a shared fixed folder.

**MUST NOT**
- Use `request.cls.<attr> = ...` injection or a base test class to share resources. Request fixtures by argument.
- Import fixtures from `conftest` modules. Expose shared fixtures through `pytest_plugins = ["tests.plugins.instruments", ...]` in the root `conftest.py`.

### 5.2 Cleanup registry

```python
# tests/plugins/cleanup.py
import contextlib
from collections.abc import Iterator

import pytest


@pytest.fixture
def cleanup() -> Iterator[contextlib.ExitStack]:
    """Register undo actions as soon as something is created.
    All callbacks run in LIFO order even if the test or earlier callbacks fail."""
    with contextlib.ExitStack() as stack:
        yield stack
```

```python
def test_profile_upload(api, cleanup):
    profile = api.create_profile(name=unique_name("profile"))
    cleanup.callback(api.delete_profile, profile.id)      # registered immediately
    ...
```

**SHOULD**
- Prefer API- or command-level setup and teardown over long multi-step sequences that replay user workflows.
- Provide a separate sweeper job for leftovers from aborted runs, keyed by the run-ID prefix (see §9.3). It is a safety net, not the primary cleanup.

---

## 6. Instrument automation (PyVISA / SCPI)

### 6.1 Standards

**MUST**
- Talk to instruments through PyVISA (`pyvisa` with a vendor VISA or `pyvisa-py`). For non-VISA serial devices, use a driver class with the same interface over `pyserial`.
- Configure termination characters, timeout and encoding explicitly per instrument from config. Do not rely on library defaults.
- Use real control characters (`"\n"`, `"\r\n"`), never escaped literals (`"\\n"`). Add a driver self-test that asserts the exact bytes written (a simulation or mock transport makes this easy).
- Give every I/O call a bounded timeout. Replace `sleep` followed by "read whatever is buffered" with `query()`, `read_until()` with a timeout, or `*OPC?` synchronisation.
- After configuration sequences, and at teardown, drain the instrument error queue (`SYST:ERR?`) and raise `InstrumentError` if it is non-empty.
- Every driver implements `safe_state()` (outputs off, sources to zero, loads disabled). Fixtures call it in teardown even when the test failed.
- Drivers are context managers whose `__exit__` returns `None` or `False`. **Never suppress exceptions.**
- Keep instrument model differences inside the driver. Tests use semantic methods (`psu.set_voltage(12.0)`), never raw SCPI strings.

**SHOULD**
- Verify identity on connect (`*IDN?`) against the expected model or vendor from config, and log the full IDN (including firmware version) once per session.
- Use `*RST; *CLS` for a known start state, followed by explicit configuration of every setting the test relies on.
- Use exclusive VISA locks (`lock_excl()`) when several processes could reach the same bench.
- Run a simulated backend (`pyvisa-sim`, or a fake driver) in CI on every commit, so driver and fixture code is exercised without hardware.

**MAY**
- Wrap instrument I/O in an optional SCPI trace logger at `DEBUG` level, written to a per-instrument log.

### 6.2 Base driver

```python
# src/fw_name/instruments/base.py
from __future__ import annotations

import logging
from types import TracebackType
from typing import Self

import pyvisa
from pyvisa.resources import MessageBasedResource

from fw_name.config import InstrumentConfig
from fw_name.errors import InstrumentError

MAX_ERROR_READS = 32


class ScpiInstrument:
    """Common SCPI behaviour. Subclasses add model-specific commands and safe_state()."""

    def __init__(self, resource: MessageBasedResource, name: str) -> None:
        self._res = resource
        self.name = name
        self.log = logging.getLogger(f"{__name__}.{name}")

    @classmethod
    def open(cls, rm: pyvisa.ResourceManager, cfg: InstrumentConfig, name: str) -> Self:
        try:
            res = rm.open_resource(
                cfg.resource,
                timeout=cfg.timeout_ms,
                read_termination=cfg.read_termination,
                write_termination=cfg.write_termination,
            )
        except pyvisa.Error as exc:
            raise InstrumentError(f"{name}: cannot open '{cfg.resource}'") from exc
        if not isinstance(res, MessageBasedResource):
            res.close()
            raise InstrumentError(f"{name}: '{cfg.resource}' is not message-based")
        inst = cls(res, name)
        inst.log.info("Connected: %s", inst.query("*IDN?"))
        return inst

    # --- I/O -----------------------------------------------------------
    def write(self, command: str) -> None:
        self.log.debug(">> %s", command)
        try:
            self._res.write(command)
        except pyvisa.Error as exc:
            raise InstrumentError(f"{self.name}: write '{command}' failed") from exc

    def query(self, command: str) -> str:
        self.log.debug(">> %s", command)
        try:
            reply = self._res.query(command).strip()
        except pyvisa.Error as exc:   # includes VisaIOError timeouts
            raise InstrumentError(f"{self.name}: query '{command}' failed") from exc
        self.log.debug("<< %s", reply)
        return reply

    # --- common SCPI ---------------------------------------------------
    def reset(self) -> None:
        self.write("*RST")
        self.write("*CLS")
        self.query("*OPC?")           # block until the reset completes

    def check_errors(self) -> None:
        errors = []
        for _ in range(MAX_ERROR_READS):
            reply = self.query("SYST:ERR?")     # e.g. '+0,"No error"' / '-113,"Undefined header"'
            if int(reply.split(",", 1)[0]) == 0:
                break
            errors.append(reply)
        if errors:
            raise InstrumentError(f"{self.name}: instrument errors {errors}")

    def safe_state(self) -> None:
        raise NotImplementedError

    # --- lifecycle -----------------------------------------------------
    def close(self) -> None:
        self._res.close()
        self.log.info("Disconnected")

    def __enter__(self) -> Self:
        return self

    def __exit__(self, exc_type: type[BaseException] | None,
                 exc: BaseException | None, tb: TracebackType | None) -> None:
        self.close()                          # returns None → exceptions propagate
```

```python
# src/fw_name/instruments/power_supply.py
from fw_name.instruments.base import ScpiInstrument


class ScpiPowerSupply(ScpiInstrument):
    def set_output(self, *, voltage: float, current_limit: float) -> None:
        self.write(f"VOLT {voltage:.4f}")
        self.write(f"CURR {current_limit:.4f}")

    def enable(self, on: bool) -> None:
        self.write(f"OUTP {'ON' if on else 'OFF'}")

    def measure_voltage(self) -> float:
        return float(self.query("MEAS:VOLT?"))

    def safe_state(self) -> None:
        self.enable(False)
        self.write("VOLT 0")
```

### 6.3 Instrument fixtures

```python
# tests/plugins/instruments.py
from collections.abc import Iterator

import pytest
import pyvisa

from fw_name.config import Settings
from fw_name.instruments.power_supply import ScpiPowerSupply


@pytest.fixture(scope="session")
def visa_rm(settings: Settings) -> Iterator[pyvisa.ResourceManager]:
    rm = pyvisa.ResourceManager("@sim" if settings.simulate else "")
    try:
        yield rm
    finally:
        rm.close()


@pytest.fixture(scope="session")
def _psu_session(visa_rm, settings) -> Iterator[ScpiPowerSupply]:
    if "psu" not in settings.instruments:
        pytest.skip(f"No 'psu' configured for env '{settings.env}'")
    psu = ScpiPowerSupply.open(visa_rm, settings.instruments["psu"], name="psu")
    try:
        yield psu
    finally:
        try:
            psu.safe_state()
        finally:
            psu.close()


@pytest.fixture
def psu(_psu_session: ScpiPowerSupply) -> Iterator[ScpiPowerSupply]:
    """Known state before each test; safe state and an empty error queue after."""
    _psu_session.reset()
    yield _psu_session
    _psu_session.safe_state()
    _psu_session.check_errors()
```

**SHOULD**
- Have a hardware CI job fail, rather than skip, when a required instrument is missing, so a mis-wired bench does not produce a green run. Make this configurable (for example a `--require-hw` option that turns `pytest.skip` into `pytest.fail`).

### 6.4 Physical timing

- **MUST:** Wait for *conditions* (an `*OPC?` response, a stable measurement, a DUT status), not for guessed durations.
- **MAY:** When a datasheet or specification mandates a settling time, use a named constant with a reference, e.g. `RELAY_SETTLE_S = 0.05  # relay datasheet t_settle max`. This is the only accepted form of fixed delay.

---

## 7. Logging

**MUST**
- Configure logging **once**, through pytest's logging options plus at most one `pytest_configure` hook. Never add handlers in constructors or library code.
- Each module uses `log = logging.getLogger(__name__)`. Drivers and clients may add a child logger per instance (`<module>.<name>`).
- Use `print()` only in `tools/` scripts. Framework and test code logs.
- Use log levels consistently:

  | Level | Content |
  |---|---|
  | `DEBUG` | Raw I/O (SCPI traffic, HTTP request and response summaries), polling iterations |
  | `INFO` | Test-relevant actions and results: connections, configuration applied, measurements with units |
  | `WARNING` | Recoverable anomalies: retries, unexpected-but-tolerated responses |
  | `ERROR` | Failures the framework is about to raise, with context |

- Write a full `DEBUG` log file per run into the run's artifact directory. Console output stays at `INFO` or above.
- Never log secrets. Redact tokens and passwords in client request logging.

```toml
# pyproject.toml
[tool.pytest.ini_options]
log_level = "DEBUG"                       # what is captured and shown for failing tests
log_cli_level = "INFO"                    # live console output when run with -o log_cli=true
log_file_level = "DEBUG"
log_file_format = "%(asctime)s.%(msecs)03d %(levelname)-8s [%(name)s] %(message)s"
log_file_date_format = "%Y-%m-%d %H:%M:%S"
```

```python
# tests/conftest.py: one artifact directory per run, created at configure time (not import)
from datetime import datetime
from pathlib import Path

import pytest

RUN_DIR = pytest.StashKey[Path]()


def pytest_configure(config: pytest.Config) -> None:
    run_id = datetime.now().strftime("%Y%m%d-%H%M%S")
    run_dir = Path(config.rootpath, "artifacts", run_id)
    run_dir.mkdir(parents=True, exist_ok=True)
    config.stash[RUN_DIR] = run_dir
    logging_plugin = config.pluginmanager.get_plugin("logging-plugin")
    if logging_plugin is not None:
        logging_plugin.set_log_path(str(run_dir / "session.log"))   # pytest API (marked experimental)
```

**SHOULD**
- Log measurement values with units and limits (`"VOUT=11.98 V (limits 11.80..12.20)"`).
- Under `pytest-xdist`, include the worker ID in the log file name.

---

## 8. Reporting and artifacts

### 8.1 Standards

**MUST**
- Every CI run produces **JUnit XML** (`--junitxml=artifacts/<run>/junit.xml`). A human-readable report (pytest-html or Allure) is optional, in addition to the XML.
- Pass report and artifact paths on the command line or derive them from the root. Never hardcode them in `addopts`.
- On failure, attach the diagnostics needed to triage without rerunning: relevant instrument state and error queue, last N lines of the DUT console, request/response summaries, and the location of the log file.
- Reporting hooks **never raise**. Every diagnostic collector is optional and guarded: a failing collector adds a "diagnostic unavailable: <reason>" note instead of crashing the session.
- Hooks do not assume a fixture exists (`item.funcargs.get(...)`, not `item.funcargs[...]`).
- Record measurement results in a structured form (value, unit, limits, verdict) through `record_property` and/or a per-run results file (CSV, JSON or a database).

### 8.2 Fixture-registered diagnostics

Fixtures register what they can contribute. The hook stays generic:

```python
# tests/plugins/diagnostics.py
from collections.abc import Callable

import pytest

DIAGNOSTICS = pytest.StashKey[list[tuple[str, Callable[[], str]]]]()


def add_diagnostic(request: pytest.FixtureRequest, name: str, collect: Callable[[], str]) -> None:
    request.node.stash.setdefault(DIAGNOSTICS, []).append((name, collect))


@pytest.hookimpl(hookwrapper=True)
def pytest_runtest_makereport(item: pytest.Item, call: pytest.CallInfo):
    outcome = yield
    report = outcome.get_result()
    if report.failed and report.when in ("setup", "call"):
        for name, collect in item.stash.get(DIAGNOSTICS, []):
            try:
                content = collect()
            except Exception as exc:          # diagnostics must never break reporting
                content = f"diagnostic unavailable: {exc!r}"
            report.sections.append((f"diagnostic: {name}", content))
```

```python
@pytest.fixture
def psu(_psu_session, request):
    add_diagnostic(request, "psu errors", lambda: _psu_session.query("SYST:ERR?"))
    ...
```

### 8.3 Measurement records (recommended)

```python
# src/fw_name/reporting/measurement.py
from dataclasses import dataclass


@dataclass(frozen=True)
class Measurement:
    name: str
    value: float
    unit: str
    low: float | None = None
    high: float | None = None

    @property
    def passed(self) -> bool:
        return (self.low is None or self.value >= self.low) and \
               (self.high is None or self.value <= self.high)
```

```python
def test_output_regulation(psu, dmm, record_property):
    psu.set_output(voltage=12.0, current_limit=1.0)
    psu.enable(True)
    m = Measurement("vout", dmm.measure_dc_voltage(), "V", low=11.8, high=12.2)
    record_property(m.name, f"{m.value} {m.unit}")
    assert m.passed, m
```

**SHOULD**
- Keep the artifact directory self-contained per run (`junit.xml`, `report.html`, `session.log`, `results.json`, captured traces). CI archives the whole directory.
- Make the post-processing (Excel export, e-mail notification) a CI step or `tools/` script that consumes the JUnit XML or results file. It is not a test.

---

## 9. Error handling and test reliability

### 9.1 Exception hierarchy

```python
# src/fw_name/errors.py
class FrameworkError(Exception):
    """Base for all framework errors."""

class ConfigError(FrameworkError): ...
class InstrumentError(FrameworkError): ...
class ClientError(FrameworkError): ...
class WaitTimeoutError(FrameworkError): ...
```

**MUST**
- When translating an exception, raise a framework exception **with context and `from exc`**. Never use a bare `raise Exception("...")` that discards the original type.
- Do not wrap a call in `try/except` only to re-raise with a vaguer message. Catch only what you can handle or meaningfully enrich.
- Never use bare `except:`. Never use `except Exception: pass`, or print-and-continue.
- Never return `None` (or `False`) to signal "operation failed". Raise.
- Validate external responses (status codes, parse results, instrument replies) at the boundary, in the client or driver, and raise a specific error there.

### 9.2 Waiting and polling

```python
# src/fw_name/waits.py
import time
from collections.abc import Callable
from typing import TypeVar

from fw_name.errors import WaitTimeoutError

T = TypeVar("T")


def wait_until(predicate: Callable[[], T], *, timeout: float, interval: float = 0.2,
               description: str) -> T:
    """Poll until predicate returns a truthy value; return it, or raise with context."""
    deadline = time.monotonic() + timeout
    while True:
        result = predicate()
        if result:
            return result
        if time.monotonic() >= deadline:
            raise WaitTimeoutError(f"Timed out after {timeout}s waiting for: {description}")
        time.sleep(interval)
```

**MUST**
- Tests contain no `time.sleep()`. All waiting goes through `wait_until` or a driver's synchronisation method, with an explicit timeout and description.
- Every external wait (mailbox polling, service readiness, DUT boot) has a deadline.

### 9.3 Isolation and data

**MUST**
- Each test is runnable alone and in any order. `pytest-randomly` or `-p random_order` SHOULD be run periodically to prove this.
- Generated identifiers are collision-resistant, and carry a run prefix so that sweepers can find leftovers:
  ```python
  RUN_TAG = uuid.uuid4().hex[:6]
  def unique_name(prefix: str) -> str:
      return f"{prefix}-{RUN_TAG}-{uuid.uuid4().hex[:8]}"
  ```
- Tests share no mutable files. Use `tmp_path` or per-test directories.
- Physical instruments are exclusive resources. Parallel execution (`pytest-xdist`) is only allowed when each worker gets its own bench or resources, or when a lock serialises access.
- Shared service accounts are read-only for tests. Tests that mutate data create their own entities through fixtures.

**SHOULD**
- Use `@pytest.mark.parametrize` (with readable `ids=`) for variants instead of copied test files.
- Keep tests short and single-purpose. Build preconditions through API/driver calls in fixtures, not by replaying long sequences.

### 9.4 Flakiness policy

**MUST**
- Do not apply reruns globally. If a test may rerun (`pytest-rerunfailures`), it carries an explicit `@pytest.mark.flaky(reruns=N)` and a linked ticket, and the reruns are visible in the report.
- Unfinished tests are marked `@pytest.mark.skip(reason="<ticket>")` or `xfail(strict=True, reason=...)`. Do not leave unmarked placeholder files.
- CI checks the collected test count (`pytest --collect-only -q`). Unintended drops, such as a misnamed class or a broken import, fail the build.

---

## 10. Coding conventions

**MUST**
- PEP 8 naming throughout, enforced by `ruff`: `snake_case` modules, functions and variables, `PascalCase` classes, `UPPER_CASE` constants. Use a test file name pattern of `test_<area>_<behaviour>.py`. Ticket IDs go in markers or docstrings, not as the only identifier.
- Type hints on all public functions and methods in `src/`. `mypy` passes in CI.
- No wildcard imports. No imports of private modules (`_name`).
- **No import-time side effects:** no I/O, device discovery, directory creation, network calls or printing at module level. Module level may only define constants, classes and functions.
- Boolean and mode switches are keyword-only (`def allocate(item, *, verify: bool = False)`) or, better, separate methods. Avoid calls like `do(x, None, True)`.
- Methods that create something **return** it (`profile = api.create_profile(...)`). Do not expose results through side-effect attributes.
- Drivers, clients and flows do not assert. They return values or raise framework errors. Tests assert.
- Assertions carry context: `assert m.passed, m` or `assert actual == expected, f"{name}: ..."`.
- Shared logic is extracted once (§2.2). Copies that differ only in a parameter are not allowed: parameterise them.
- No commented-out code or debug prints in `main`. Version control keeps history.

**SHOULD**
- Use comments to explain *why*. Use docstrings on public APIs, and don't restate the code.
- Aim for functions under ~40 lines and avoid nesting deeper than 3 levels.
- Favour composition over inheritance. Inheritance is for genuine "is-a" relations (e.g. `ScpiPowerSupply(ScpiInstrument)`).

**MAY**
- Use `Protocol` classes to define instrument capabilities (e.g. `VoltageSource`) so tests can depend on behaviour rather than on a concrete model.

---

## 11. CI/CD integration

**MUST**
- The CI entry point propagates pytest's exit code. Wrapper scripts never force success (no `exit 0` or `exit /b 0`).
- Parameterise one entry point (`--env`, marker expression) instead of keeping copied scripts per environment.
- Every commit triggers lint, type check, framework self-tests and the simulation suite (`--env sim`). Hardware suites run on labelled lab agents.
- Secrets are injected by the CI secret store as environment variables.
- The run's artifact directory is archived on every run, pass or fail.

**SHOULD**
- Create the virtual environment or container from the lock file in CI. Do not rely on a pre-provisioned venv at a fixed path.
- Run the secret scanner on every push and block merges on findings.

**Minimal launcher example (cross-platform via `nox` or `tox`, or a CI YAML step):**

```bash
pytest --env "${TEST_ENV:-sim}" -m "${TEST_MARKERS:-not hardware}" \
       --junitxml "artifacts/junit.xml"
# The step's exit status is pytest's exit status. Do not mask it.
```

---

## 12. Documentation

**MUST:** The README covers prerequisites, install (`pip install -e .[dev,sim]`), how to select an environment, how to supply secrets (referencing `.env.example`), how to run subsets (markers), where artifacts appear, and how to add an instrument driver and a test.

**SHOULD:** Keep a short `docs/architecture.md` with the layer diagram from §2.2, and a changelog for framework-level changes.

---

## 13. Compliance checklist

Use this when reviewing a new framework or a pull request that changes framework code.

**Structure and tooling**
- [ ] Installable package, dependencies pinned, no unused dependencies
- [ ] `ruff` + `mypy` + secret scanner run in CI and pass
- [ ] Downward-only dependencies; drivers and clients don't import pytest or conftest
- [ ] No experimental or placeholder code in collected paths

**Configuration**
- [ ] Single typed `Settings`, loaded once and validated at session start
- [ ] No absolute or user-specific paths; all paths root-relative
- [ ] No secrets in the repository; secrets come from the environment or CI store
- [ ] CLI defaults are the effective defaults; `addopts` carries no environment values

**Lifecycle**
- [ ] Every resource fixture uses `yield` plus `finally` and releases what it acquired
- [ ] Created entities are cleaned up through fixtures registered at creation time
- [ ] No module performs I/O at import

**Instruments**
- [ ] Explicit terminations and timeouts per instrument; real control characters
- [ ] Error queue checked after configuration and at teardown
- [ ] `safe_state()` implemented and called on teardown
- [ ] Context managers never suppress exceptions
- [ ] Simulation backend exercised in CI

**Logging and reporting**
- [ ] Logging configured once; module loggers; no `print` in framework or test code
- [ ] Per-run DEBUG log file; JUnit XML produced
- [ ] Failure diagnostics attached; hooks guarded and never raise

**Reliability**
- [ ] No `time.sleep` in tests; all waits use deadlines
- [ ] Tests are order-independent; IDs are unique and run-tagged
- [ ] No global reruns; flaky tests are marked and tracked
- [ ] Collected-test count is verified in CI
- [ ] CI propagates the pytest exit code

---

## Appendix A – Lessons-learned traceability

Each problem observed in `snd_report.md` maps to the standard that prevents it.

| Observed problem (report ref.) | Preventing standard |
|---|---|
| Credentials, tokens and a client secret committed (F-01) | §4.1 no secrets; §3 secret scanner; §11 CI secret store |
| Hardcoded absolute project roots and report paths (F-02) | §1.1, §4.1 root-relative paths; §8.1 paths from CLI |
| Reporting hook crashes on tests without the expected fixture (F-03) | §8.1 hooks never raise; `funcargs.get`; §8.2 registered diagnostics |
| Cleanup as the last test statements; skipped on failure (F-04) | §5.1 cleanup at creation time; §5.2 `ExitStack` registry |
| Resource opened in one fixture and closed in another; leaks on setup failure (F-05) | §5.1 same-fixture acquire and release, `yield` plus `finally` |
| Serial ports opened and infinite loop started at import; escaped `"\\r\\n"` terminator; sleep-then-read (F-06) | §6.1 explicit terminations, bounded I/O, self-test of bytes written; §10 no import side effects |
| Context manager `__exit__` returning `True` (F-06) | §6.1 never suppress exceptions |
| Test class silently not collected (F-07) | §9.4 collected-count check; §3 `--strict-markers`/`--strict-config` |
| CI wrapper always exits 0; global reruns hide flakiness (F-08) | §11 exit-code propagation; §9.4 flakiness policy |
| Loggers created per object and never used; duplicate handlers; `print` (F-09) | §7 single configuration, module loggers, per-run log file |
| CLI options ignored by some modules; config sections unused; hardcoded endpoints (F-10, F-11) | §4.1 single `Settings` object, no dead config |
| Errors swallowed and `None` returned; unbounded waits (F-12) | §9.1 raise, never return `None`; §9.2 deadlines |
| Fixture crashing when an optional option is absent (F-13) | §4.1 validated settings with effective defaults |
| Shared files and folders between tests (F-14) | §5.1 `tmp_path`; §9.3 no shared mutable files |
| Collision-prone random data (F-15) | §9.3 UUID-based, run-tagged identifiers |
| Fixed sleeps throughout (F-16) | §6.4, §9.2 condition-based waiting |
| 300+ generic `raise Exception(...)` wrappers (F-17) | §9.1 typed hierarchy, `from exc`, no vague re-wrapping |
| Components instantiating peers; hidden workflows (F-18) | §2.2 `flows/` layer, downward-only dependencies |
| Positional booleans; results via side-effect attributes (F-19) | §10 keyword-only switches, return created objects |
| Dead code, sandbox and TODO tests collected; unused dependencies (F-20, F-21) | §2.2 `tools/` outside collection; §9.4 explicit skip/xfail; §3 dependency hygiene |
| Missing machine-readable results (F-22) | §8.1 JUnit XML, structured measurements |
| Copy-paste variant tests (F-23) | §9.3 parametrisation; §10 no parameter-only copies |

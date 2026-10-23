We need to define a stable integration boundary between PyTest/Python and our C-based tests before the overall test framework
architecture and parameter set are fully known.


**_Current constraints:_**
- Other team members need to start developing C tests immediately.
- The Python/PyTest framework will continue evolving in parallel.
- We do not yet know all configuration parameters, runtime data, commands, or results that will eventually need to cross the Python/C boundary.
- I want to avoid modifying existing C tests every time the Python framework or its configuration model changes.
- The C tests should focus primarily on the actual test/core functionality rather than framework orchestration,
- environment setup, reporting, or infrastructure concerns.


**_Desired execution flow:_**
```text
           PyTest
              │
              ▼
           Python test/framework
              │
              ├── configure environment
              ├── configure instruments/devices
              └── prepare test inputs
              │
              ▼
┌─────────────────────────────┐
│ Stable Python ↔ C interface │
│                             │
│ configuration / input       │
│ execution control           │
│ results                     │
│ logs / diagnostics          │
└─────────────────────────────┘
              │
              ▼
┌─────────────────────────────┐
│        C Test Core          │
│                             │
│ Actual test functionality   │
└─────────────────────────────┘
              │
              ▼
   Results + logs + diagnostics
              │
              ▼
      Interface/adaptor
              │
              ▼
      PyTest verdict/report
```


**_Primary design goal:_**
- Changes to the Python framework, configuration structure, reporting, logging, or peripheral functionality should have
little or ideally no impact on the C test core.
- At the same time, we should not try to predict and hard-code every possible future parameter into the C API today.


**Please propose the simplest viable architecture that provides this separation.**

**_Specifically address:_**
- What should be the stable contract between Python and C?
- Which responsibilities should belong to:
	- PyTest/test case
	- Python framework
	- Python-to-C adapter/interface
	- C test runner/interface layer
	- C test core
- How should configuration and parameters be passed without continuously changing C test function signatures or CLI arguments?
- How should C return:
	- execution status
	- test verdict
	- structured results
	- diagnostics
	- logs/artifacts
- Should the initial interface use CLI arguments, configuration files, JSON, stdin/stdout, or another mechanism? Explain the trade-offs.
- How should the interface evolve when new parameters or result fields are added?
- Should the contract be versioned? If yes, propose a simple versioning strategy.
- How can C developers start implementing tests now without waiting for the full Python framework design?
- What parts should we implement now, what should we only design for, and what would currently be over-engineering?

**Important:**
- Challenge this architecture if there is a better boundary. Do not assume that my proposed "interface layer" is necessarily the correct solution.

**Provide:**
- a recommended component architecture
- an execution/data-flow diagram
- an example interface contract
- one small C-side example
- one small Python/PyTest-side example
- expected evolution of the interface when new parameters are introduced

This aligns particularly well with your existing requirement that the framework execute C binaries, supply parameters,
capture stdout/stderr and exit codes, preserve artifacts, and remain flexible enough for structured results rather than
relying only on exit codes. framework_rules


Design the framework so that team-specific or format-specific parsing logic does not leak into individual PyTest test cases.
When proposing the architecture, determine the appropriate abstraction boundary for this variability.
Consider interfaces, adapters, strategies, pluggable parsers, and configuration-driven parser selection where they provide concrete value.
The framework should ideally normalize heterogeneous test outputs into a common internal result model that can represent:
- execution status,
- test verdict,
- failure reason,
- warnings/errors,
- measurements or relevant result data,
- diagnostic information,
- and references to raw logs/artifacts.

Discuss:
1. Whether parsers should be organized per team, per test type, or per output/result format.
2. How the appropriate parser should be selected for a test.
3. Where parser-specific logic should live.
4. How new teams and new output formats can be added without modifying existing tests.
5. How to distinguish a parsing failure from a DUT/test failure.
6. How the architecture could gradually move from parsing human-readable logs toward structured machine-readable results.  
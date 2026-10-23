# SoC Validation Test Strategy

## 1. Purpose

This document summarizes a practical strategy for integrating existing C code into a Python and `pytest`-based SoC validation framework.

The recommended approach is to keep hardware-near and timing-sensitive functionality in C, while Python controls test execution, test equipment, result collection, PASS/FAIL evaluation, and report generation.

> **DUT**: Device Under Test, in this case the SoC or target board being validated.
>
> **CPython**: The standard Python implementation written primarily in C.

---

## 2. Recommended Strategy

Do not automatically rewrite all C code in Python. Instead, divide responsibilities according to what each language handles well.

| Layer                    | Recommended language    | Main responsibilities                                                       |
|:-------------------------|:------------------------|:----------------------------------------------------------------------------|
| Test orchestration       | Python                  | Test sequencing, configuration, retries, cleanup                            |
| Test framework           | `pytest`                | Parameterization, fixtures, assertions, PASS/FAIL status                    |
| Test equipment control   | Python                  | PSU, oscilloscope, DMM, UART, JTAG, VISA/SCPI control                       |
| Low-level test execution | C                       | Register access, DMA, interrupts, memory tests, timing-sensitive operations |
| Result exchange          | JSON and process status | Structured measurements, errors, status, metadata                           |
| Reporting                | Python                  | JUnit XML, HTML, Markdown, Quarto, Excel, database output                   |

### High-Level Architecture

```text
┌─────────────────────────┐
│     Python / pytest     │
└─────────────────────────┘
            │
            ▼
┌─────────────────────────┐
│ Configure Test Equipment│
└─────────────────────────┘
            │
            ▼
┌─────────────────────────┐
│ Execute C Test Program  │
└─────────────────────────┘
            │
            ▼
┌─────────────────────────┐
│   Collect Test Output   │
└─────────────────────────┘
            │
            ▼
┌─────────────────────────┐
│ Evaluate PASS / FAIL    │
└─────────────────────────┘
            │
            ▼
┌─────────────────────────┐
│    Generate Report      │
└─────────────────────────┘
```

---

## 3. Where the C Integration Happens

In the following Python call, all visible code is Python:

```python
result = subprocess.run(
    [
        "sram_test.exe",
        "--controller", "SRAMC1",
        "--address", "0x40000000",
        "--size", "0x1000",
    ],
    capture_output=True,
    text=True,
    timeout=60,
)
```

The C integration is the executable:

```text
sram_test.exe
```

The executable is compiled from C source code:

```text
┌─────────────────────┐
│    sram_test.c      │
└─────────────────────┘
          │
          │ C compiler
          ▼
┌─────────────────────┐
│    sram_test.exe    │
└─────────────────────┘
```

Python does not execute the C source directly. `subprocess.run()` starts the compiled C program as a separate operating-system process.

### Process-Level Integration

```text
┌─────────────────────┐                           ┌─────────────────────┐
│  Python / pytest    │ ─── subprocess.run() ──►  │   sram_test.exe     │
└─────────────────────┘                           └─────────────────────┘
                                                            │
                                                            ▼
                                                  ┌─────────────────────┐
                                                  │      SoC / DUT      │
                                                  └─────────────────────┘
```

Running the Python list is equivalent to entering this command in PowerShell:

```powershell
.\sram_test.exe --controller SRAMC1 --address 0x40000000 --size 0x1000
```

---

## 4. Parameterizing the C Executable

Yes, `sram_test.exe` can and should be parameterized. The executable is built once and then called repeatedly with different test configurations.

### Example Command-Line Interface

```text
sram_test.exe
    │
    ├── --controller SRAMC1
    ├── --test memory
    ├── --address 0x40000000
    ├── --size 0x1000
    ├── --pattern 0xAAAAAAAA
    ├── --iterations 100
    ├── --ecc enable
    └── --output result.json
```

### Suggested Parameters

| Parameter      | Example       | Purpose                               |
|:---------------|:--------------|:--------------------------------------|
| `--controller` | `SRAMC1`      | Select the SRAM controller            |
| `--test`       | `memory`      | Select the test operation             |
| `--address`    | `0x40000000`  | Set the test start address            |
| `--size`       | `0x1000`      | Set the tested memory range           |
| `--pattern`    | `0xAAAAAAAA`  | Select the memory data pattern        |
| `--iterations` | `100`         | Set the number of repetitions         |
| `--ecc`        | `enable`      | Enable or disable ECC-related testing |
| `--output`     | `result.json` | Select the structured result file     |

> **ECC**: Error-Correcting Code, logic used to detect and possibly correct memory data errors.

### Example C Argument Reception

The C application receives parameters through `argc` and `argv`:

```c
#include <stdio.h>

int main(int argc, char *argv[])
{
    for (int i = 0; i < argc; i++)
    {
        printf("argv[%d] = %s\n", i, argv[i]);
    }

    return 0;
}
```

For this command:

```powershell
.\sram_test.exe --controller SRAMC1 --address 0x40000000
```

C receives:

```text
argv[0] = sram_test.exe
argv[1] = --controller
argv[2] = SRAMC1
argv[3] = --address
argv[4] = 0x40000000
```

The production C implementation should validate every parameter, reject unsupported combinations, and return a non-zero exit code for invalid input or failed execution.

---

## 5. Parameterization with pytest

`pytest.mark.parametrize` can run the same test logic with multiple SRAM controllers, addresses, sizes, and patterns.

```python
import subprocess
import pytest


@pytest.mark.parametrize(
    "controller,address,size,pattern",
    [
        ("SRAMC1", "0x40000000", "0x1000", "0xAAAAAAAA"),
        ("SRAMC1", "0x40001000", "0x1000", "0x55555555"),
        ("SRAMC2", "0x50000000", "0x2000", "0xFFFFFFFF"),
        ("SRAMC3", "0x60000000", "0x4000", "0x00000000"),
    ],
)
def test_sram(controller, address, size, pattern):
    result = subprocess.run(
        [
            "sram_test.exe",
            "--controller", controller,
            "--address", address,
            "--size", size,
            "--pattern", pattern,
        ],
        capture_output=True,
        text=True,
        timeout=60,
        check=False,
    )

    print(result.stdout)
    assert result.returncode == 0, result.stderr
```

### Parameterized Execution Graph

```text
┌─────────────────────┐
│       pytest        │
└─────────────────────┘
          │
          ├──────────────► SRAMC1 / 0xAAAAAAAA
          │
          ├──────────────► SRAMC1 / 0x55555555
          │
          ├──────────────► SRAMC2 / 0xFFFFFFFF
          │
          └──────────────► SRAMC3 / 0x00000000
                              │
                              ▼
                    ┌─────────────────────┐
                    │    sram_test.exe    │
                    └─────────────────────┘
```

---

## 6. Input and Output Interface

Python and the C executable communicate in two directions.

```text
┌─────────────────────┐
│  Python Test Input  │
└─────────────────────┘
          │
          │ Command-line arguments
          ▼
┌─────────────────────┐
│    C Executable     │
└─────────────────────┘
          │
          │ stdout / stderr / exit code / JSON
          ▼
┌─────────────────────┐
│ Python Test Result  │
└─────────────────────┘
```

### Python Input to C

| Mechanism              | Use                                                                            |
|:-----------------------|:-------------------------------------------------------------------------------|
| Command-line arguments | Controller, address, size, pattern, iterations                                 |
| Environment variables  | Stable environment configuration; not preferred for individual test parameters |
| Configuration file     | Large or reusable test configurations                                          |
| Standard input         | Interactive or streamed commands; usually unnecessary for simple tests         |

### C Output to Python

| Output                   | Python access                  | Recommended use                             |
|:-------------------------|:-------------------------------|:--------------------------------------------|
| Standard output          | `result.stdout`                | Human-readable diagnostics and logs         |
| Standard error           | `result.stderr`                | Error details and failure diagnostics       |
| Exit code                | `result.returncode`            | Primary execution status                    |
| JSON file or JSON stdout | `json.load()` / `json.loads()` | Structured measurements and result metadata |
| Artifact files           | File paths recorded in JSON    | Memory dumps, traces, screenshots, raw logs |

### Recommended Exit-Code Convention

| Exit code | Meaning                              |
|----------:|:-------------------------------------|
|       `0` | Test executed and passed             |
|       `1` | Test executed and failed             |
|       `2` | Invalid command-line parameters      |
|       `3` | DUT communication failure            |
|       `4` | Test setup or initialization failure |
|       `5` | Internal C application error         |

The exact mapping is project-specific, but it must be documented and stable.

---

## 7. Structured Result Format

Use JSON for machine-readable output rather than relying only on text parsing.

```json
{
  "schema_version": "1.0",
  "test": "SRAMC1_MEMORY_TEST",
  "controller": "SRAMC1",
  "address": "0x40000000",
  "size_bytes": 4096,
  "pattern": "0xAAAAAAAA",
  "iterations": 100,
  "error_count": 0,
  "result": "PASS",
  "artifacts": {
    "c_log": "artifacts/sramc1_c.log",
    "uart_log": "artifacts/sramc1_uart.log"
  }
}
```

Python parsing example:

```python
import json
import subprocess

result = subprocess.run(
    ["sram_test.exe", "--output", "result.json"],
    capture_output=True,
    text=True,
    timeout=60,
    check=False,
)

with open("result.json", encoding="utf-8") as file:
    data = json.load(file)

assert result.returncode == 0, result.stderr
assert data["result"] == "PASS"
assert data["error_count"] == 0
```

### Why Structured Output Is Preferred

| Text-only output | Structured JSON output |
|---|---|
| Easy for a person to read | Easy for Python and people to process |
| Parsing may break when wording changes | Stable field-based interface |
| Measurements may lose type information | Numbers, strings, arrays, and objects are explicit |
| Harder to version | Can include `schema_version` |
| Useful for diagnostics | Useful for reports, databases, and traceability |

---

## 8. Test Equipment Integration

Python should normally configure and control host-connected laboratory equipment before starting the C test.

```text
┌─────────────────────┐                           ┌─────────────────────┐
│  Python / pytest    │ ───── SCPI / VISA ─────►  │ PSU / Scope / DMM   │
└─────────────────────┘                           └─────────────────────┘
          │
          │ subprocess
          ▼
┌─────────────────────┐                           ┌─────────────────────┐
│    C Test Program   │ ─── UART / JTAG / API ─►  │      SoC / DUT      │
└─────────────────────┘                           └─────────────────────┘
```

> **SCPI**: Standard Commands for Programmable Instruments, a command language used by laboratory equipment.
>
> **VISA**: Virtual Instrument Software Architecture, an API layer for communicating with instruments.

A typical `pytest` fixture sequence is:

1. Connect to the PSU and communication interfaces.
2. Configure voltage and current limits.
3. Power on and reset the DUT.
4. Start UART or other hardware log capture.
5. Run `sram_test.exe`.
6. Collect C output, DUT logs, and equipment measurements.
7. Evaluate results.
8. Power down or restore a known state even when the test fails.

---

## 9. Example Robust Python Runner

```python
from pathlib import Path
import json
import subprocess


def run_sram_test(
    executable: Path,
    controller: str,
    address: int,
    size: int,
    pattern: int,
    output_file: Path,
    timeout_s: int = 60,
) -> dict:
    command = [
        str(executable),
        "--controller", controller,
        "--address", f"0x{address:08X}",
        "--size", f"0x{size:X}",
        "--pattern", f"0x{pattern:08X}",
        "--output", str(output_file),
    ]

    try:
        completed = subprocess.run(
            command,
            capture_output=True,
            text=True,
            timeout=timeout_s,
            check=False,
        )
    except subprocess.TimeoutExpired as error:
        raise AssertionError(
            f"C test timed out after {timeout_s} seconds"
        ) from error

    if not output_file.exists():
        raise AssertionError(
            "C test did not create the expected JSON result file.\n"
            f"Exit code: {completed.returncode}\n"
            f"stdout:\n{completed.stdout}\n"
            f"stderr:\n{completed.stderr}"
        )

    with output_file.open(encoding="utf-8") as file:
        data = json.load(file)

    assert completed.returncode == 0, (
        f"C test failed with exit code {completed.returncode}\n"
        f"stdout:\n{completed.stdout}\n"
        f"stderr:\n{completed.stderr}"
    )

    return {
        "result": data,
        "stdout": completed.stdout,
        "stderr": completed.stderr,
        "command": command,
    }
```

Important implementation points:

- Pass arguments as a list, not as one manually assembled shell string.
- Keep `shell=False`, which is the default, unless a shell is specifically required.
- Always use a timeout.
- Preserve both `stdout` and `stderr` as artifacts.
- Check the exit code and the structured result.
- Include the exact command and parameter set in the report.
- Use a unique output directory for each test invocation to avoid file collisions.

---

## 10. Reporting Strategy

The report should combine results from Python, C, the DUT, and the test equipment.

```text
┌─────────────────────┐
│ Test Configuration  │
└─────────────────────┘
          │
          ▼
┌─────────────────────┐
│ Equipment Readings  │
└─────────────────────┘
          │
          ▼
┌─────────────────────┐
│ C and DUT Results   │
└─────────────────────┘
          │
          ▼
┌─────────────────────┐
│ pytest Assertions   │
└─────────────────────┘
          │
          ▼
┌─────────────────────┐
│ Final Test Report   │
└─────────────────────┘
```

### Recommended Report Content

| Section             | Content                                                 |
|:--------------------|:--------------------------------------------------------|
| Identification      | Test ID, requirement ID, build, board, silicon revision |
| Configuration       | Controller, address, size, pattern, iterations          |
| Environment         | Tool versions, C executable version, Python version     |
| Equipment           | Instrument identity, voltage/current configuration      |
| Execution           | Start/end timestamps, duration, exact command           |
| Results             | Exit code, JSON result, measurements, error count       |
| Assertions          | Expected and actual values                              |
| Artifacts           | C log, UART log, JTAG log, dumps, screenshots           |
| Verdict             | PASS, FAIL, ERROR, SKIPPED                              |
| Failure diagnostics | stderr, timeout details, cleanup status                 |

### Suggested Artifact Layout

```text
Execution
│
├──► Test Case 001
│    ├──► result.json
│    ├──► c_stdout.log
│    ├──► c_stderr.log
│    ├──► uart.log
│    ├──► equipment.json
│    └──► screenshots
│
├──► Test Case 002
│    ├──► result.json
│    ├──► c_stdout.log
│    └──► uart.log
│
└──► Summary
     ├──► junit.xml
     ├──► report.html
     └──► report.md
```

---

## 11. Alternative C Integration Methods

Executing an `.exe` is not the only option.

| Method                | Integration level | Advantages                                                           | Trade-offs                                                             |
|:----------------------|:------------------|:---------------------------------------------------------------------|:-----------------------------------------------------------------------|
| `subprocess` + `.exe` | Process           | Simple isolation, easy log capture, works with existing applications | Process startup overhead; interface limited to arguments/files/streams |
| `ctypes` + DLL        | Function          | Direct C function calls; lower call overhead                         | Requires stable C API and careful type/memory handling                 |
| `cffi` + DLL          | Function          | Convenient C declarations and native-library access                  | Adds a dependency and build considerations                             |
| CPython extension     | Interpreter       | Deep Python integration and potentially high performance             | More complex build, packaging, and debugging                           |

### DLL-Based Alternative

```text
┌─────────────────────┐                           ┌─────────────────────┐
│  Python / pytest    │ ─────── ctypes ─────────► │     test.dll        │
└─────────────────────┘                           └─────────────────────┘
                                                            │
                                                            ▼
                                                  ┌─────────────────────┐
                                                  │      SoC / DUT      │
                                                  └─────────────────────┘
```

Start with `subprocess` when the C code already exists as a standalone program. Consider a DLL when direct function-level calls are required and the C API can be designed as a stable interface.

---

## 12. What Should Stay in C and What Should Move to Python

| Functionality                                   | Keep in C |       Use Python        |
|:------------------------------------------------|:---------:|:-----------------------:|
| Bare-metal firmware                             |    Yes    | No for standard CPython |
| Boot and startup code                           |    Yes    |           No            |
| Memory-mapped register access inside the target |    Yes    |      Not directly       |
| Interrupt Service Routines                      |    Yes    |           No            |
| DMA configuration inside the SoC                |    Yes    |           No            |
| Tight real-time loops                           |    Yes    |       Usually no        |
| Existing qualified low-level tests              |    Yes    |  Wrap and orchestrate   |
| Host-side UART access                           | Optional  |           Yes           |
| PSU and instrument control                      | Optional  |           Yes           |
| Test sequencing                                 | Optional  |           Yes           |
| Parameter matrix generation                     | Optional  |           Yes           |
| Logging and artifact management                 | Optional  |           Yes           |
| Result analysis                                 | Optional  |           Yes           |
| Report generation                               | Optional  |           Yes           |

> **DMA**: Direct Memory Access, a hardware mechanism that transfers data without continuous CPU involvement.

The practical target architecture is therefore:

```text
┌─────────────────────┐
│ Python Orchestration│
└─────────────────────┘
          │
          ▼
┌─────────────────────┐
│ pytest Test Control │
└─────────────────────┘
          │
          ├──────────────────────► Test Equipment
          │
          ▼
┌─────────────────────┐
│ C Test Application  │
└─────────────────────┘
          │
          ▼
┌─────────────────────┐
│      SoC / DUT      │
└─────────────────────┘
          │
          ▼
┌─────────────────────┐
│ Results & Artifacts │
└─────────────────────┘
          │
          ▼
┌─────────────────────┐
│ Validation Report   │
└─────────────────────┘
```

---

## 13. Key Risks and Controls

| Risk                                    | Effect                          | Recommended control                                      |
|:----------------------------------------|:--------------------------------|:---------------------------------------------------------|
| C program hangs                         | Test execution never completes  | Enforce a Python timeout and terminate the child process |
| Human-readable output changes           | Parser starts failing           | Use versioned JSON output                                |
| Result file from previous run is reused | False result                    | Create a unique per-test output directory                |
| Invalid address or size reaches C       | Crash or unsafe hardware access | Validate in Python and again in C                        |
| C exit code and JSON verdict disagree   | Ambiguous result                | Treat disagreement as framework ERROR                    |
| Parallel tests access one DUT           | Resource conflicts              | Serialize DUT access or use resource locks               |
| Equipment remains powered after failure | Unsafe or inconsistent setup    | Put cleanup in `pytest` fixture teardown/finalizer       |
| stdout buffer becomes large             | Memory usage increases          | Redirect large logs to files or stream them              |
| Executable version is unknown           | Poor reproducibility            | Record executable hash/version in every run              |
| Parameters are not traceable            | Result cannot be reproduced     | Store exact command and configuration in the report      |

---

## 14. Recommended Decision

Use the following as the initial implementation:

```text
┌─────────────────────┐
│ Python + pytest     │
└─────────────────────┘
          │
          ├──► Configure PSU and instruments
          ├──► Configure communication interfaces
          ├──► Build test parameter set
          │
          ▼
┌─────────────────────┐
│ subprocess.run()    │
└─────────────────────┘
          │
          ▼
┌─────────────────────┐
│ Parameterized C EXE │
└─────────────────────┘
          │
          ├──► Perform low-level SoC operations
          ├──► Return documented exit code
          └──► Generate versioned JSON result
          │
          ▼
┌─────────────────────┐
│ Python Evaluation   │
└─────────────────────┘
          │
          ├──► Preserve logs and artifacts
          ├──► Apply pytest assertions
          └──► Generate JUnit / HTML / Markdown report
```

This approach provides a clean separation:

- **Python and pytest own orchestration, equipment control, traceability, and reporting.**
- **C owns low-level SoC operations and existing hardware test logic.**
- **A documented command-line and JSON interface forms the contract between them.**

---

## 15. Implementation Checklist

- [ ] Compile the existing C test into `sram_test.exe`.
- [ ] Define and document all command-line parameters.
- [ ] Validate parameters in the C application.
- [ ] Define stable exit codes.
- [ ] Add versioned JSON output.
- [ ] Preserve stdout and stderr.
- [ ] Create one artifact directory per pytest test instance.
- [ ] Add `pytest` fixtures for equipment setup and cleanup.
- [ ] Add `pytest.mark.parametrize` test matrices.
- [ ] Add timeout and process-failure handling.
- [ ] Record software, firmware, board, silicon, and equipment versions.
- [ ] Generate JUnit XML and the selected user-facing report format.
- [ ] Verify that failures always leave the DUT and equipment in a safe known state.



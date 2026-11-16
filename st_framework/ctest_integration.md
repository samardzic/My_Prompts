I want to simulate pytest interaction with c-language compiled test.
C Test needs to be parametrized - test itself has a runner with buitin parametrization - I just need to pass parametrization parameters
Test executed also needs to return some log/output.
Suggest solution without coding




# Integrating pytest with a Compiled C Test Runner

I want to design an integration between Python/pytest and an existing test executable compiled from C source code.

Please analyze the requirements and propose an appropriate architecture and implementation approach.

**Important: Do not write any code yet. I want a conceptual solution, architecture proposal, and discussion of alternatives.**

## 1. Existing C test infrastructure

I have an existing C-language test application that is compiled into an executable.

The executable already contains its own test runner and built-in parametrization mechanism.

The C test runner is responsible for:
- Executing the actual test logic.
- Handling test-specific parametrization.
- Running the test with the supplied parameters.
- Producing execution logs and test output.
- Returning an execution result/status.

I do not want to reimplement the C test runner or its parametrization logic in Python.

## 2. Python / pytest responsibilities

I want pytest to act as an orchestration layer around the compiled C executable.

Pytest should:
- Define and manage test cases.
- create env (hardware)
- configure env and check availibility
- Provide parametrization values to the C executable.
- Launch the compiled executable with the appropriate arguments.
- Wait for execution to complete.
- Capture stdout and stderr.
- Collect execution logs and results.
- Determine whether the test passed or failed.
- Integrate the execution results into pytest reporting.

The actual test execution must remain inside the C executable.

## 3. Test parametrization

The C executable already supports parametrization internally.

I only need pytest to supply the parameters required by the C test runner.

The proposed solution should explain:
- How pytest parameters should be represented.
- How parameters should be passed to the executable.
- How multiple parameter combinations should be managed.
- How pytest can execute the same C test with different parameter sets.
- How parameter values should appear in pytest test reports.

Consider whether command-line arguments, configuration files, JSON input, or another communication mechanism would be most appropriate.

## 4. Execution output and logging

The compiled C executable produces logs and execution output.

I need to understand how Python should collect and manage this information.

Consider:
- Capturing stdout and stderr.
- Preserving the original C test logs.
- Separating test execution output from diagnostic information.
- Handling large amounts of output.
- Saving logs to individual files for each parametrized test execution.
- Making logs accessible when a pytest test fails.

## 5. Execution status and error handling

Explain how pytest should determine the result of the C test execution.

Consider:
- Process exit codes.
- Explicit PASS/FAIL information in the output.
- Crashes and abnormal termination.
- Execution timeouts.
- Missing or malformed output.
- Infrastructure errors versus actual test failures.

I want to distinguish between:
1. The C test executed successfully and passed.
2. The C test executed successfully but reported a failure.
3. The C executable failed to execute or crashed.
4. The execution infrastructure encountered an error.

## 6. Architecture and design alternatives

Compare possible integration approaches, including:

- Python `subprocess` with pytest parametrization.
- A reusable Python wrapper around the C executable.
- Pytest fixtures for executable management and configuration.
- Other approaches if they provide meaningful advantages.

Discuss the advantages, disadvantages, maintainability and scalability of each approach.

## 7. Expected outcome

Provide:

1. A recommended architecture.
2. A conceptual execution flow diagram.
3. Suggested responsibilities for pytest, the Python wrapper and the C test runner.
4. Recommended parameter-passing mechanism.
5. Recommended output/logging mechanism.
6. Recommended result/status reporting mechanism.
7. Potential limitations and edge cases.

The design should support future integration of multiple compiled test executables, different test suites and potentially hardware-dependent tests.

Keep the solution modular, maintainable and suitable for a professional QA automation framework.
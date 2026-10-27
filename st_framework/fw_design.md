## Furniture design

- Env is python, pytest, visa, py-visa, pyserial, pyvisa-sim (device simulator)
- reporting is with quarto, pytest-html, (suggest additional)
- data sourcing would be through excel document
- logging needs to be fully covered since it is one of the crucial components of the framework.
results and debug is dove through gathered logs and logs parsing

# Hardware / Instrument Integration

The framework must support laboratory equipment such as:

- Power supply units (PSUs)
- Oscilloscopes
- Digital multimeters (DMMs)
- Other instruments that may be added later

The framework should support:

<br/>









# Claude Code Prompt: Test Automation Framework Analysis

Analyze the existing test automation framework and generate a Markdown report named `snd_report.md` in the project root.

## Areas to Analyze

1. **Framework architecture**
   - Project structure, module organization, dependencies, and separation of responsibilities.
   - Overall design, maintainability, and extensibility.

2. **Logging and reporting**
   - Logging configuration, levels, consistency, and usefulness for debugging.
   - Report generation, test results, failure details, and artifact handling.

3. **Configuration and parameter passing**
   - How parameters flow through `conftest.py`, pytest fixtures, command-line options, configuration files, and framework classes.
   - Fixture dependencies, scopes, setup/teardown, and resource management.
   - Identify confusing, fragile, or error-prone parameter flows.

4. **OOP and coding practices**
   - Use of encapsulation, inheritance, composition, abstraction, and separation of responsibilities.
   - Code duplication, complexity, exception handling, naming, type hints, and general Python best practices.
   - Identify inappropriate abstractions or unnecessary complexity.

5. **Test execution and reliability**
   - Test organization, isolation, parametrization, assertions, timeouts, and potential flaky behavior.
   - Error handling and cleanup of resources.

6. **Hardware and instrument automation**
   - If applicable, review PyVISA, SCPI communication, instrument initialization, timeouts, error handling, and connection cleanup.

7. **Other relevant findings**
   - Identify additional issues or improvement opportunities not explicitly listed above.

## Framework Maturity Assessment

Evaluate the overall maturity of the framework using a scale from Level 1 to Level 5:

- **Level 1:** Ad hoc
- **Level 2:** Basic
- **Level 3:** Established
- **Level 4:** Mature
- **Level 5:** Advanced

Explain the assigned level using evidence from the codebase. Highlight the framework's strengths, weaknesses, and the main factors limiting its maturity.

## Findings and Recommendations

For each significant issue, include:

- Description and category.
- Severity: Critical, High, Medium, or Low.
- Relevant file path and class/function, including line numbers where possible.
- Explanation of the problem and its practical impact.
- A concrete recommendation for improvement.

Distinguish confirmed defects from potential risks and optional improvements. Prioritize recommendations by importance.

## Report Structure

Generate `snd_report.md` with these sections:

1. Executive summary.
2. Framework architecture.
3. Logging and reporting.
4. Configuration, `conftest.py`, and fixtures.
5. OOP and coding practices.
6. Test reliability and hardware automation, where applicable.
7. Identified issues and improvements.
8. Framework maturity assessment.
9. Prioritized recommendations.
10. Conclusion.

## Instructions

- Inspect the actual codebase before drawing conclusions.
- Support findings with evidence; do not invent issues or make unsupported assumptions.
- Keep the report concise, technical, and actionable.
- Include code examples or diagrams only when they clarify an important finding.
- Do not modify existing source code or configuration files.
- Do not install dependencies or perform destructive operations.
- Generate only the requested report, `snd_report.md`, in the project root.
- Verify the report exists and contains all required sections before finishing.
- Provide a brief final summary of the report's location and key findings.






## Refine prompt and create template
from 'snd_report.md' generate document that would be the base template for the future projects.
Disregard the part related to selenium/webdriver handling.
Im gonna use new fremwork without this two components.

Using the analysis in snd_report.md, create a reusable test automation framework template document that will serve as the architectural and implementation baseline for future projects.

Requirements:
Extract the most valuable architectural decisions, coding practices, configuration patterns, fixture design, logging, reporting, and error-handling recommendations from snd_report.md.
Convert the findings into practical guidelines and standards for a new framework, rather than simply summarizing the analysis.
Exclude all Selenium and WebDriver-related components, concepts, dependencies, and recommendations. The new framework will not use either technology.
Retain recommendations that are applicable to general Python test automation, pytest, PyVISA, SCPI communication, and laboratory equipment automation, where relevant.
Define recommended project structure, responsibilities of major modules, configuration and parameter passing, fixture lifecycle, resource management, logging, reporting, and coding conventions.
Clearly distinguish mandatory standards from optional recommendations.
Include concise Python examples or configuration snippets only where they clarify how a recommended pattern should be implemented.
Address the weaknesses identified in snd_report.md by establishing practices that help prevent similar problems in future projects.

Output:
Generate a Markdown document named framework_template.md in the project root.
The document should be practical, technology-appropriate, reusable across future projects, and detailed enough to guide framework implementation without being tied to the original project's specific classes, modules, or configuration.
Do not modify existing source code or snd_report.md. Use the analysis report as the primary source, applying engineering judgment where necessary.
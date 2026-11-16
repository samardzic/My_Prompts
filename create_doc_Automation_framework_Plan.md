Im requested to create timeline/estimates for my automation activities.
This assumes all of the steps in process of creating the framework from ground up
I have 130 - 150 man days for these activities.
Im not writing the actual tests but more of support and analysis structure.
I need to create test framework design document.
I already have 3 framework templates (the way that framework is designed) from previous project.
- "partner template.md"
- "snd_template.md"


This is the manager request:
```
- I’ll need a high-level schedule, nothing too detailed,
but at least enough to understand when you will need support from SW and when we will need support from your team.
- For each part of the test automation, I need to know which equipment/platform will be used to debug and validate it.
The goal is to be as ready as possible when the chip arrives
```

Areas that I need to cover:
- Framework design

- Framework implementation
	- writing the actual code
- Data sourcing and data management
	- test inputs, test outputs, data maintenance

- Virtual hardware Simulator
	- During the initial development phase, real hardware may not be available.

- Hardware / Instrument Integration
	- The framework must support laboratory equipment

Instrument to DUT integration
	- various ways of of devices interacting with equipment peripherals

- Framework to ctest integration

- Logging
	- The framework needs a centralized logging architecture.

- Log Parsing
	- The framework must provide a log parsing mechanism.

- Test Reporting
	- Test reports should be generated automatically in several format options



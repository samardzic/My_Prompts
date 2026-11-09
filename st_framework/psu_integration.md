# PSU Device Management – Python / PyVISA / NI-VISA

I want to implement a hardware device management layer in an existing Python-based test automation framework.
The implementation will initially focus on **Programmable Power Supply Units (PSUs)**, with the architecture designed to support
additional measurement instruments (DMM, SMU, oscilloscope, electronic load, etc.) in the future.

Important: Keep all changes and implementation in "root folder/psu_devices" during this testing phase. Later on this will be integrated into the whole of the framework

## 1. Technology stack

Use:
- Python
- PyVISA
- NI-VISA as the primary VISA backend
- pytest for testing




## 2. Architecture and hardware abstraction

Design a modular and extensible architecture that separates:

- Generic PSU functionality and interface definitions.
- Manufacturer/model-specific PSU implementations.
- VISA communication and resource management.
- Device configuration.
- Test framework integration.

Different PSU manufacturers and models may implement different SCPI commands and support different capabilities.

The framework must hide these differences behind a **unified PSU API**.

The test code should not need to know which manufacturer or model is being used.




## 3. PSU identification and configuration

The PSU manufacturer and model must be configurable parameters.

For example:

```python
psu = PSU(
    manufacturer="Keysight",
    model="E36312A",
    resource="USB0::...::INSTR"
)
```

The framework should use this information to instantiate the appropriate model-specific driver.
Where possible, support automatic instrument identification using the VISA `*IDN?` query and validate that the connected
instrument matches the configured manufacturer and model.

Do not assume that all instruments support identical SCPI commands.



## 4. Unified PSU API

Define a common interface for all supported PSUs.
At minimum, consider these operations:

- connect()
- disconnect()
- identify()
- reset()
- output_on()
- output_off()
- set_voltage()
- get_voltage()
- set_current_limit()
- get_current_limit()
- measure_voltage()
- measure_current()
- get_output_state()

The API should expose consistent method names, parameters, return types and exception behavior across different PSU models.

If a particular PSU does not support an operation, handle this explicitly rather than silently ignoring it.




## 5. Model-specific implementations

Implement the architecture so that each PSU model has its own driver class.

For example:

```text
devices/
    psu/
        base.py
        factory.py
        models/
            keysight_e36312a.py
            keysight_e3648a.py
            rigol_dp832.py
```

The generic PSU interface should contain no manufacturer-specific SCPI commands.

Individual model drivers should implement the commands required by their respective instruments.

## 6. VISA communication

Create a reusable VISA communication layer responsible for:

- Opening and closing VISA sessions.
- Configuring timeouts.
- Sending commands.
- Reading responses.
- Executing SCPI queries.
- Handling communication errors.
- Managing resource cleanup.

Use NI-VISA as the default backend, but avoid tightly coupling the PSU implementation to NI-VISA-specific functionality.





## 7. Error handling and safety

Implement appropriate handling for:

- Instrument not found.
- Incorrect manufacturer/model.
- VISA communication timeout.
- Invalid command.
- Unsupported functionality.
- Instrument disconnection.
- Invalid voltage/current parameters.

Ensure that PSU outputs are safely disabled during appropriate shutdown and error-handling paths.

Do not automatically enable PSU outputs during device initialization.




## 8. Testing

Use pytest to test the implementation.

Tests should include:

- Unit tests for the generic PSU interface.
- Unit tests for individual model drivers.
- Mocked VISA communication.
- Instrument identification.
- Command generation and response parsing.
- Error handling.
- Integration tests that can run against physical instruments when available.

The framework must allow development and testing without requiring a physically connected PSU.

## 9. Implementation approach

Implement this incrementally.
Start by proposing the architecture, class structure, interfaces and configuration approach.
Explain the design decisions and trade-offs before implementing the code.
Then implement one concrete PSU model as the initial reference implementation.
Keep the architecture simple, maintainable and extensible. Avoid unnecessary abstractions and overengineering.

**Important:** The unified API must remain independent of the underlying PSU manufacturer and model.
Adding a new PSU should primarily require implementing a new model-specific driver, without modifying existing test cases or the generic PSU interface.


These are some sources that you can consult:
https://github.com/tektronix/keithley
https://github.com/InterplanetaryLab/keithley_2231A-pyvisa
https://github.com/jesseadamczyk/keithley2700-tutorial/tree/main#keithley2700-tutorial
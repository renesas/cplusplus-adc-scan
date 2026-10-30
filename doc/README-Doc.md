# C++

## Overview

This SBB demonstrates how to use Renesas RA Flexible Software Package (FSP) ADC HAL APIs from C++. The ADC peripheral is wrapped by the `adc_ep` class, and the reference application creates an ADC object and invokes its member functions to initialize and control scanning and read ADC data.

The example reads the MCU internal temperature-sensor ADC channel and provides a serial-terminal interface to start a scan and, where the configured conversion mode supports it, stop a scan. ADC results and status information are displayed on the terminal.

This document covers only the following boards:

- FPB-RA0L1
- FPB-RA2E3
- FPB-RA4E2
- FPB-RA6E2
- FPB-RA8E1

## What Use Case Does This SBB Demonstrate?

The SBB demonstrates a reusable C++ wrapper around the FSP ADC HAL. It separates ADC control from the terminal-driven reference application so that the ADC control methods can be reused by another application.

The demonstrated flow is:

1. Initialize the UART console used by the reference application.
2. Accept a terminal command to start ADC scanning.
3. Open and configure the ADC through FSP.
4. Start conversion using the ADC driver supported by the target MCU.
5. Read the MCU die-temperature ADC channel.
6. Display the raw ADC result on the terminal.
7. In a continuous/sequential mode, continue reading until the user requests a stop.
8. In a single/one-shot mode, automatically stop reading after the conversion and close the ADC module.

The SBB abstracts differences among the FSP ADC driver variants through macros in `adc_ep.h`. Depending on the target MCU, the implementation selects `r_adc`, `r_adc_b`, or `r_adc_d` APIs at build time.

## Supported Reference Boards

| Board | ADC mode information used by this SBB | Board-specific setup |
|---|---|---|
| FPB-RA0L1 | ADC_D. Default is Sequential mode; One-shot can be selected in RA Configurator. | Place jumper J9 on pins 1-2 to use on-board debug functionality. |
| FPB-RA2E3 | Uses the applicable FSP ADC driver configuration for the device. | Close E25, E28, E46, and E47 to use J-Link OB VCOM for the serial terminal. |
| FPB-RA4E2 | Uses the applicable FSP ADC driver configuration for the device. | No additional hardware connection is identified in the supplied README beyond connecting the USB debug port. |
| FPB-RA6E2 | Uses the applicable FSP ADC driver configuration for the device. | No additional hardware connection is identified in the supplied README beyond connecting the USB debug port. |
| FPB-RA8E1 | Uses the applicable FSP ADC driver configuration for the device. | No additional hardware connection is identified in the supplied README beyond connecting the USB debug port. |

> The supplied README states that the default example configuration is Continuous Scan for boards generally, while FPB-RA0L1 defaults to ADC_D Sequential mode.

## SBB Architecture

The source is divided into two logical layers.

### Reusable SBB core

`adc_ep.h` and `adc_ep.cpp` contain the ADC C++ wrapper and the ADC control logic. This layer calls the configured FSP ADC instance and hides driver-family differences behind common macros.

The class provides operations to:

- initialize the ADC module;
- start an ADC scan;
- read the configured internal temperature channel;
- stop a supported ongoing scan; and
- close the ADC module.

### Demo/reference application

`adc_example_app.h` and `adc_example_app.cpp` implement the terminal-driven demonstration. This code is an example of how to consume the reusable ADC class and is not required when integrating the ADC wrapper into a different application.

The example application:

- initializes `uart_console`;
- prints the SBB banner, FSP version, description, and menu;
- accepts terminal commands;
- calls the `adc_ep` public methods;
- prints ADC readings and errors; and
- waits one second between displayed readings while reads remain enabled.

`hal_entry.cpp` is expected to call the single reference-application entry point `adc_example_app_run()`.

## Core Code Behavior

### Starting a scan

`adc_ep::adc_scan_start()` performs the core start sequence:

1. Opens the ADC module through `init_adc_module()`.
2. Applies the configured channel/scan configuration.
3. Performs ADC calibration when the selected ADC implementation requires the calibration path compiled by this source.
4. Starts the ADC scan.
5. For ADC_D in One-shot mode, waits for completion and performs a second scan because the source states that the initial internal-temperature conversion result cannot be used.
6. Sets `g_ready_to_read` so the application can read ADC data.

### Reading ADC data

`adc_ep::adc_read_data()` reads `ADC_CHANNEL_TEMPERATURE` and copies the raw result to the caller-provided `uint16_t` buffer.

In Single Scan or ADC_D One-shot mode, the function clears `g_ready_to_read` and closes the ADC after the read. In a continuous/sequential configuration, the reference application can continue reading while the flag remains set.

### Stopping a scan

`adc_ep::adc_scan_stop()` stops the active scan, clears `g_ready_to_read`, and closes the ADC module. The reference application's Stop command is intended for the continuous/sequential mode, not the single/one-shot mode.

## Public Functions to Look at First

### Reusable SBB API

```cpp
fsp_err_t adc_ep::init_adc_module(void);
fsp_err_t adc_ep::adc_scan_start(void);
fsp_err_t adc_ep::adc_read_data(uint16_t * p_data);
fsp_err_t adc_ep::adc_scan_stop(void);
static void adc_ep::deinit_adc_module(void);
```

For normal integration, start with:

1. `adc_scan_start()` to configure and begin conversion.
2. `adc_read_data()` to retrieve the ADC result.
3. `adc_scan_stop()` when using a mode that continues scanning.
4. `deinit_adc_module()` when explicit cleanup is needed.

`init_adc_module()` is public, but `adc_scan_start()` already calls it as part of the demonstrated sequence.

### Reference-application API

```cpp
void adc_example_app_run(void);
```

Use this entry point only when running the supplied terminal demonstration.

## Reference Board Configuration Assumptions

### ADC configuration

The SBB expects FSP-generated ADC configuration objects declared through `hal_data.h`, including the ADC control/configuration objects and the appropriate scan/channel configuration for the target ADC implementation.

The source selects among these FSP ADC modules according to BSP peripheral definitions:

- `r_adc_d` when `BSP_PERIPHERAL_ADC_D_PRESENT` is enabled;
- `r_adc_b` when `BSP_PERIPHERAL_ADC_B_PRESENT` is enabled;
- otherwise `r_adc`.

For FPB-RA0L1, the supplied README states that ADC_D defaults to Sequential mode. To use One-shot mode, change **Conversion operation** for the ADC_D driver in RA Configurator.

For the other boards covered by this document, the supplied README states the project default generally as Continuous Scan and instructs users to select **Mode = Single Scan** in RA Configurator to switch modes, subject to the ADC driver available on the target.

### ADC channel

The reusable source explicitly reads:

```cpp
ADC_CHANNEL_TEMPERATURE
```

Therefore, the reference configuration is built around the MCU internal temperature-sensor ADC input, not an external analog-input pin.

### UART console

The demo/reference application depends on `console/uart_console.h`, `printf()`, `getchar()`, and the user stdio routing used by the project.

The supplied README specifies these serial-terminal settings:

- Speed: 115200 baud
- Data: 8 bits
- Parity: none
- Stop bits: 1
- Flow control: none
- Terminal receive newline: CR
- Terminal transmit newline: CR
- Local echo: enabled

For FPB-RA2E3, close E25, E28, E46, and E47 to route the serial terminal through J-Link OB VCOM.

For FPB-RA0L1, place J9 on pins 1-2 to use on-board debugging.

## What Must Be Changed for a Custom Board?

The supplied source and README establish the reference configuration but do not define a complete custom-board porting procedure. When moving the SBB to another board, review the following integration points against the new FSP configuration and hardware design.

### Board and ADC instance configuration

Review the ADC instance generated by RA Configurator and ensure that the project exposes the objects used by the SBB, including `g_adc_ctrl`, `g_adc_cfg`, and the appropriate channel/scan configuration object.

If the target MCU uses a different ADC peripheral variant, verify that the BSP peripheral definitions select the intended `r_adc`, `r_adc_b`, or `r_adc_d` path.

### ADC channel

The current SBB reads the internal temperature channel. If the application requires another analog source, update the ADC channel configuration and the channel supplied to `ADC_READ()`.

For an external ADC input, configure the corresponding analog-capable pin in RA Configurator according to the target device and board schematic.

### Scan mode

Select the required scan/conversion mode in RA Configurator. Application control flow must match that mode because single/one-shot operation automatically ends the demonstrated read sequence, while continuous/sequential operation remains active until stopped.

### Clocks and timing

The source does not provide board-independent clock values. Preserve a valid FSP clock configuration for the selected ADC peripheral and review conversion/calibration timing whenever the clock configuration changes.

The demo application also intentionally inserts a one-second software delay between displayed ADC readings:

```cpp
R_BSP_SoftwareDelay(ADC_READ_DELAY, BSP_DELAY_UNITS_SECONDS);
```

Change this demo delay if the application requires another reporting cadence. It is not an ADC conversion-rate specification.

### Memory

The supplied files do not define special heap, stack, or buffer-size requirements. The reusable class stores a small ADC result value and uses the FSP-generated ADC control/configuration objects. Re-evaluate memory configuration if the SBB is moved into an RTOS task or combined with additional middleware.

### Task priorities and RTOS integration

The supplied reference application is a polling `while (true)` loop and does not define RTOS task priorities. If the SBB is integrated into an RTOS application, task creation, priorities, synchronization, and blocking behavior must be provided by that application. These parameters are not specified by the supplied SBB.

### Middleware and console setup

The ADC core does not require the terminal UI. If the target application does not need the demonstration console, replace or remove `adc_example_app.*`, `uart_console`, `printf()`, and `getchar()` integration while retaining the reusable `adc_ep` layer.

## Dependencies

### FSP and BSP

The SBB depends on Renesas RA FSP and the generated project configuration exposed through `hal_data.h`.

The code directly uses or maps to the following APIs, depending on the selected ADC peripheral:

- `R_ADC_Open`, `R_ADC_ScanCfg`, `R_ADC_ScanStart`, `R_ADC_ScanStop`, `R_ADC_Read`, `R_ADC_Close`, `R_ADC_StatusGet`, and `R_ADC_Calibrate` when applicable;
- equivalent `R_ADC_B_*` APIs for ADC_B; and
- equivalent `R_ADC_D_*` APIs for ADC_D.

It also uses:

- `R_FSP_VersionGet()` in the demo application;
- `R_BSP_SoftwareDelay()` for the demo's one-second display interval; and
- BSP peripheral-presence macros to select the ADC implementation.

### Services/middleware

The demo/reference application depends on the project UART-console service:

```text
console/uart_console.h
```

It uses `uart_console_init()`, `uart_console_has_char()`, and `uart_console_flush()` plus standard C I/O calls routed by the project's stdio implementation.

### Hardware resources

The demonstrated configuration requires:

- one supported RA MCU/board;
- the MCU ADC and internal temperature sensor channel;
- a USB debug/programming connection; and
- a UART/VCOM path when running the terminal demo.

## Demonstrated Behavior and Performance

The supplied project demonstrates these behaviors:

- C++ encapsulation of ADC HAL access;
- selection of ADC driver family at build time;
- start and stop terminal commands;
- raw MCU die-temperature ADC readings displayed on a serial terminal;
- single/one-shot and continuous/sequential control-flow handling; and
- ADC calibration where the compiled target-specific path requires it.

The reference application waits one second between displayed readings. This is an application delay and should not be interpreted as ADC throughput.

The source comments state that the calibration path can take **24 ms to 780 ms based on clock**. No additional measured throughput, CPU utilization, memory footprint, ADC accuracy, temperature accuracy, interrupt latency, or power benchmark is provided in the supplied files. Those values should therefore not be treated as characterized SBB performance.

## File Ownership and Modification Guidance

| File/category | Role | Modification guidance |
|---|---|---|
| `adc_ep.h` | Reusable ADC C++ interface, driver-selection macros, common definitions | Modify when adapting or extending the reusable SBB API or supported ADC behavior. |
| `adc_ep.cpp` | Reusable ADC implementation | Modify when changing core ADC behavior. Keep local changes isolated where practical to simplify later upgrades. |
| `adc_example_app.h` | Demo/reference application interface and terminal strings | Demo-oriented. Modify or replace for the product application's user interface. |
| `adc_example_app.cpp` | Demo/reference application control loop, UART interaction, printing, and menu | Demo-oriented. Safe to replace when integrating the reusable core into a product application. |
| `hal_data.h` and FSP-generated configuration | FSP-generated project interface/configuration used by the SBB | Treat as generated/preset content. Make peripheral configuration changes through RA Configurator rather than hand-maintaining generated output. |
| `readme.txt` | Source documentation used to describe the supplied example | Documentation/reference content. This Markdown document consolidates the relevant information for the five boards in scope. |

## Adopting a Newer SBB Release After Local Modifications

The supplied files do not define a formal upgrade or merge policy. To minimize integration effort, keep application-specific changes outside the reusable SBB core whenever possible.

A practical source-ownership boundary is:

- keep `adc_ep.h` and `adc_ep.cpp` close to the released SBB unless core behavior must change;
- put product-specific workflows, presentation, command handling, RTOS integration, and policy in the application layer rather than `adc_ep`;
- keep board/peripheral selections in the FSP/RA Configurator configuration rather than hard-coding generated settings into the SBB; and
- preserve local SBB changes as small, reviewable commits so they can be compared with a newer release.

When adopting a newer SBB release, compare the newer reusable `adc_ep` files with the locally modified versions, reapply only the still-needed local changes, regenerate or reconcile the FSP configuration for the target board, and rerun the application's ADC start/read/stop behavior on hardware.

## Reference Application Operation

1. Connect the target board's USB debug port to the host PC.
2. Apply the board-specific jumper or VCOM settings listed above.
3. Build and program the project using its configured Renesas RA/FSP development environment.
4. Open the serial terminal using 115200, 8 data bits, no parity, 1 stop bit, and no flow control.
5. Run the application.
6. Use the terminal menu:
   - Press `1` to start ADC scanning.
   - Press `2` to stop scanning when using the continuous/sequential mode supported by the configured target.
   - Use another key to display the menu again.

## Integration Summary

For a product integration, the primary reusable portion is the `adc_ep` class. The `adc_example_app` layer demonstrates how to call it but couples the example to a polling terminal interface. Configure the required ADC instance and scan mode in FSP, retain the internal-temperature channel only if that is the desired signal, and replace the demo console/control loop with the product application's own control path.

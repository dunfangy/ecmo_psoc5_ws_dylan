# PSoC Firmware Optimization — First Revision

## Overview

This revision primarily addresses higher-priority issues that may affect data correctness, communication availability, or normal system operation, as summarized in Part 1. Part 2 outlines lower-priority improvements to reliability, timing, and maintainability, which are deferred for future consideration.

## 1. Changes Completed in This Revision

### 1.1 OEM Response Completeness Check

**Code area:** `main.c` — `OEM_Read2()` and its callers for status, scale factor, flow, phase, and amplitude queries. Corresponds to Record Issue 1 / Plan A1.

**Problem and consequences:** The function returned success even when fewer than two response bytes had arrived. It could read uninitialized array elements and pass invalid values to its callers, bypassing their reception-failure branches. This could affect initialization, flow scaling, measurements, and warning status.

**Correction strategy:** Check the received-byte count before writing output values. Return failure unless both bytes have arrived, allowing existing callers to handle incomplete responses. This addresses response completeness; UART error recovery and delayed-response handling remain separate work.

### 1.2 Initial Raw-Pressure Channel Selection

**Code area:** `Pressure_Transducer.c` — `init_pressure_sensors()`, in relation to the alternating calls to `read_pressure()`. Corresponds to Record Issue 2 / Plan A4.

**Problem and consequences:** `AMux_Start()` disconnected every channel. Since `read_pressure()` read the current ADC input before switching to the next channel, the first Before reading had no selected sensor input and could be transmitted as an apparently valid measurement.

**Correction strategy:** Explicitly select AMux channel 0, the raw Before pressure input, during initialization. The existing alternating Before/After sequence is retained. These AMux channels are separate from the SAR channels named `ADC_CH0` and `ADC_CH1`.

### 1.3 Raw-Pressure Result Width and Warning Range

**Code area:** `Pressure_Transducer.h`, `Pressure_Transducer.c`, and `main.c` — pressure reading interface, intermediate storage, and pressure-voltage warning check. Corresponds to Record Issue 3 / Plan A5.

**Problem and consequences:** The original warning limits of −0.1 to 5.1 V did not match the configured nominal differential range of approximately ±0.064 V. In addition, the delta-sigma ADC can produce overrange counts exceeding a signed 16-bit container. Truncation could reverse the apparent sign and produce an apparently in-range voltage, concealing the original overrange result.

**Correction strategy:** Use the 32-bit result interface and signed 32-bit variables throughout the pressure-count path, then compare the converted voltage against the nominal ±0.064 V range. The ADC resolution remains 16 bits; only result handling becomes wider. The transmitted fields remain floating-point voltages, not calibrated pressure values.

### 1.4 SAR Upper-Endpoint Warning

**Code area:** `main.c` — the four-channel SAR voltage conversion and warning loop. Corresponds to Record Issue 4 / Plan A5.

**Problem and consequences:** With the current 12-bit, nominal 0–5 V configuration, the maximum count converts to approximately 4.998779 V. The original upper warning threshold of 5.1 V could therefore miss a reading already at the upper endpoint.

**Correction strategy:** Use an upper threshold of 4.9987 V and retain the −0.1 V abnormal-negative-value check. A direct count-based alternative was also documented, but the voltage-based option was selected. The warning identifies an endpoint condition requiring investigation; it does not prove that the external input exceeded its range. The fixed threshold must be reconsidered if the reference or calibration changes.

### 1.5 SAR Scan Consistency and Bounded Waiting

**Code area:** `TopDesign.cysch` — `ADC_TS410` sampling mode; `main.c` — initialization, timer interrupt, SAR acquisition, voltage packing, and CH3 change checking. Corresponds to Record Issue 5 / Plan A6 and the SAR portion of A7.

**Problem and consequences:** During continuous scanning, DMA could overwrite the result buffer between separate channel reads, mixing scan rounds within one packet. The blocking completion call could also wait indefinitely if acquisition never completed.

**Correction strategy:** Use software-triggered single scans initiated by the main loop. Poll completion a limited number of times and read all four results after completion, without delays between channel reads. On the first timeout, stop SAR acquisition and retain a fault state until restart. Send floating-point NaN values for all four SAR fields and skip CH3 change checking when acquisition fails.

NaN is assigned to the floating-point voltage, not the integer count array. Under the selected design, acquisition failure alone does not set `ADCOORWarn`; invalidity is expressed through the NaN fields. The polling loop limits waiting attempts but is not a strict 10 ms wall-clock deadline. A single scan contains four sequential samples, not four simultaneous samples.

## 2. Potential Directions for Future Optimization

### 2.1 Raw-Pressure Acquisition Timing

Move pressure conversion and channel-switching delays out of the timer interrupt, and introduce bounded waits for the delta-sigma ADC. This would reduce interrupt execution time and address the blocking pressure-acquisition path that remains after the SAR changes.

### 2.2 Sample Consistency and Traceability

Protect the Before/After pressure pair from being updated while the main loop reads it. Sequence numbers, timestamps, and event counters could make overwritten samples, skipped acquisitions, and differences in data age easier to identify.

### 2.3 Fault Reporting and Recovery

Distinguish acquisition timeouts, invalid measurements, and range warnings more clearly. Bounded retries could improve OEM initialization and allow controlled SAR recovery instead of requiring a program restart after the first SAR timeout.

### 2.4 CRC Coverage

Include the actual status/opcode byte in the CRC calculation instead of a fixed substitute. This would improve detection of corrupted status information and requires coordinated firmware and Raspberry Pi updates with shared protocol test vectors.

### 2.5 Packet Capacity and Length Handling

Align payload limits, frame-buffer capacity, and UART length types, and reject unsupported lengths before writing or sending a frame. The current 48-byte measurement payload fits, but the maximum-length definitions contain a boundary issue that could affect future packet extensions.

### 2.6 Channel Definitions and Calibration

Document each channel's source, unit, divider ratio, polarity, and calibration parameters. Keep ADC voltage, external input voltage, and calibrated pressure or flow clearly distinguished, and confirm unresolved physical wiring when supporting evidence becomes available.

### 2.7 UART Electrical Compatibility

When hardware becomes available, verify the actual I/O supply, resistor assembly, and signal levels at the Raspberry Pi interface. Firmware voltage settings alone cannot confirm electrical compatibility; this remains an interface verification task.

### 2.8 Code Structure and Configuration

Separate acquisition, OEM communication, and packet handling into focused modules, centralize configuration, and remove unused declarations and excessive debug output. Keep application logic in handwritten files and modify component settings through PSoC Creator to preserve changes during code regeneration.


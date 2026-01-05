# Architecture – Rybno PLC & SCADA (TwinCAT 3)

## Goal
This project controls a micro biogas plant and a granulation line. The PLC uses:
- TwinCAT 3 (IEC 61131-3, Structured Text)
- SCADA/HMI: Promotic
- Communication PLC↔SCADA: OPC UA
- Field communication: Modbus RTU (inverters + other devices)

---

## Core design approach
### Drives are implemented using an OOP architecture + Bridge pattern
We separated:
- **Drive drivers** (hardware abstraction)
- **Drive control** (logic layer)

This separation is intentional to:
- reuse control logic across different drive hardware implementations
- keep hardware-specific details isolated in driver classes
- keep SCADA mapping consistent (commands/status independent from hardware)

---

## Drive drivers (hardware abstraction)
We have three main drivers:
- `FB_Drive_1c`  
  Contactor-based drive, **one direction** (single contactor).
- `FB_Drive_2c`  
  Contactor-based drive, **forward/reverse** (two contactors). No frequency control here like in VFDs.
- `FB_DriveInv`  
  Inverter/VFD-based drive.
  - Frequency setpoint is written via Modbus
  - Status is read via Modbus
  - Forward/reverse command may still be wired (digital I/O), depending on installation

Drivers include:
- hardware I/O handling
- interlocks at driver/hardware level (where applicable)
- SCADA-mapped status and commands (through GVL structures)

---

## Drive control (logic layer)
- `FB_DriveControl` is an **abstract** control class used to control different drive drivers.
- Control layer implements:
  - mode handling (AUTO/MANUAL)
  - command arbitration
  - higher-level permissives/interlocks (process level)
  - standard behaviors independent from hardware

### Operating modes
- **AUTO**:
  - used when an upstream PLC logic (master automation FBs) commands the drive
  - AUTO does not mean SCADA direct control; it is for higher-level automation code
- **MANUAL**:
  - direct operator control from Promotic SCADA (buttons, setpoints)
  - exchanged via OPC UA using GVL-mapped structures

---

## Modbus architecture
### Inverters / VFDs
- One shared Modbus RTU port handles all inverters.
- A shared service/operation FB exists:
  - `FB_ModbusInverterOperation`
  - It manages communication and scheduling for multiple inverters on one port.

### Other Modbus devices
A second Modbus port is used for other devices, e.g.:
- biogas analyzer
- Simex load cell / weighing transmitter
- CHP (combined heat and power unit)

---

## SCADA / OPC UA mapping
- SCADA Promotic exchanges data through OPC UA.
- PLC uses **GVL structures** to expose:
  - commands
  - statuses
  - setpoints
  - alarms
- The purpose is to keep a stable interface and decouple SCADA from internal PLC implementation.

---

## Master automation (process logic)
- Higher-level automation (granulation process, sequences, plant logic) is implemented as classic IEC logic:
  - Structured Text FBs (not necessarily OOP)
- The PLC program is organized into subprograms.
- OOP is used mainly for device-level components (drives, Modbus services, etc.).

---

## Sensors / I/O types
- Analog sensors: pressure, temperature, etc.
- Binary sensors: e.g. rotary level switches (MIN/MAX), two-state inputs.
- Additional helper/utility functions exist (common functions, conversions, alarms, etc.).

---

## Expectations for changes
When modifying code or adding features:
1. Preserve the Bridge separation:
   - do not mix driver and control responsibilities
2. Keep SCADA interface stable (GVL structures)
3. Add new device types by extending drivers or services, not by branching in master logic
4. Prefer maintainable, testable state machines for process-level automations

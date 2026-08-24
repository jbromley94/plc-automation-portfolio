# I/O and Variable List

## 1. Scope

This document records the simulated commands, valve outputs, countdown values, internal variables, timing parameters, and external symbols used by `twoWaterTanksFuncAndBlockContinued`.

No variable is mapped to a fixed IEC address such as `%IX`, `%QX`, `%IW`, or `%QW`. The current integration boundary is CODESYS Symbol Configuration.

## 2. Tank 1 Commands

| Signal | Type | Active transition | Initial value | Current source | Intended role |
|---|---|---|---:|---|---|
| `Fill` | `BOOL` | False → true | `FALSE` | Online value / symbol client | Request one 15-second fill pulse |
| `Discharge` | `BOOL` | True → false | `FALSE` | Online value / symbol client | Request one 10-second discharge pulse |

`Discharge` must first be made true and then false. A maintained active-high pushbutton therefore requests discharge on release.

## 3. Tank 2 Commands

| Signal | Type | Active transition | Initial value | Current source | Intended role |
|---|---|---|---:|---|---|
| `Fill2` | `BOOL` | False → true | `FALSE` | Online value / symbol client | Request one 8-second fill pulse |
| `Discharge2` | `BOOL` | True → false | `FALSE` | Online value / symbol client | Request one 12-second discharge pulse |

## 4. Valve Outputs

| Signal | Type | True meaning | Logic source | Current destination |
|---|---|---|---|---|
| `Fill_Valve` | `BOOL` | Tank 1 fill pulse active | `FB_Tank_1.TP_0.Q` | Online value / symbol client |
| `Discharge_Valve` | `BOOL` | Tank 1 discharge pulse active | `FB_Tank_1.TP_1.Q` | Online value / symbol client |
| `Fill_Valve2` | `BOOL` | Tank 2 fill pulse active | `FB_Tank_2.TP_0.Q` | Online value / symbol client |
| `Discharge_Valve2` | `BOOL` | Tank 2 discharge pulse active | `FB_Tank_2.TP_1.Q` | Online value / symbol client |

These are software command values, not confirmed valve positions. There is no open/closed limit switch, solenoid current, flow, pressure, or actuator-fault feedback.

## 5. Countdown Values

| Signal | Type | Value meaning | Written by | Idle behaviour |
|---|---|---|---|---|
| `Timer` | `INT` | Tank 1 whole seconds remaining | Fill and discharge `FC_Timer` calls | Retains last written value |
| `Timer2` | `INT` | Tank 2 whole seconds remaining | Fill and discharge `FC_Timer` calls | Retains last written value |

The displays truncate fractional seconds. Valve status is required to interpret whether the value belongs to a fill phase, discharge phase, or idle retained result.

## 6. `PLC_PRG` Internal Variables

| Variable | Type | Purpose | Externally exposed? |
|---|---|---|---:|
| `FB_Tank_1` | `FB_Tank` | Tank 1 state and timing instance | No |
| `Fill_ET` | `TIME` | Tank 1 fill elapsed time | No |
| `Discharge_ET` | `TIME` | Tank 1 discharge elapsed time | No |
| `FB_Tank_2` | `FB_Tank` | Tank 2 state and timing instance | No |
| `Fill_ET2` | `TIME` | Tank 2 fill elapsed time | No |
| `Discharge_ET2` | `TIME` | Tank 2 discharge elapsed time | No |

The four commands, four valve outputs, and two countdown values are also declared locally in `PLC_PRG` and are listed separately above because they form the external interface.

## 7. `FB_Tank` Variables

| Variable | Direction | Type | Default | Purpose |
|---|---|---|---:|---|
| `Fill` | Input | `BOOL` | `FALSE` | Rising-edge fill request |
| `Fill_PT` | Input | `TIME` | `T#10S` | Fill pulse preset |
| `Discharge` | Input | `BOOL` | `FALSE` | Falling-edge discharge request |
| `Discharge_PT` | Input | `TIME` | `T#10s` | Discharge pulse preset |
| `Fill_Valve` | Output | `BOOL` | `FALSE` | Fill pulse status/command |
| `Fill_ET` | Output | `TIME` | `T#0s` | Fill elapsed time |
| `Discharge_ET` | Output | `TIME` | `T#0s` | Discharge elapsed time |
| `Discharge_Valve` | Output | `BOOL` | `FALSE` | Discharge pulse status/command |
| `TP_0` | Internal | `TP` | Reset | Fill timer instance |
| `TP_1` | Internal | `TP` | Reset | Discharge timer instance |

## 8. `FC_Timer` Variables

| Variable | Direction | Type | Purpose |
|---|---|---|---|
| `Max_Time` | Input | `TIME` | Configured pulse duration |
| `Current_Time` | Input | `TIME` | Current timer elapsed value |
| `FC_Timer` | Return | `INT` | Whole seconds remaining |
| `Var1` | Internal | `TIME` | `Max_Time - Current_Time` |
| `Inv_Time_INT` | Internal | `INT` | Remaining milliseconds converted to signed 16-bit integer |

Current presets are safe for the `INT` intermediate. If the function is reused for a duration above approximately 32.767 seconds, change the conversion design before relying on the result.

## 9. Timing Configuration

| Tank | Phase | Request event | Preset source | Effective preset | Valve output | Elapsed value | Countdown |
|---|---|---|---|---:|---|---|---|
| 1 | Fill | Rising edge of `Fill` | Explicit call input | 15 s | `Fill_Valve` | `Fill_ET` | `Timer` |
| 1 | Discharge | Falling edge of `Discharge` | `FB_Tank` default | 10 s | `Discharge_Valve` | `Discharge_ET` | `Timer` |
| 2 | Fill | Rising edge of `Fill2` | Explicit call input | 8 s | `Fill_Valve2` | `Fill_ET2` | `Timer2` |
| 2 | Discharge | Falling edge of `Discharge2` | Explicit call input | 12 s | `Discharge_Valve2` | `Discharge_ET2` | `Timer2` |

Tank 1's unconnected discharge-preset input should be visually confirmed after import and tested for an effective ten-second pulse.

## 10. Symbol Configuration

The export selects the following paths. `VarAccess = 3` and `MaxVarAccess = 3` are recorded as read/write access.

| CODESYS symbol | Type | PLC role | Recommended client use | Current access |
|---|---|---|---|---|
| `Application.PLC_PRG.Fill` | `BOOL` | Tank 1 command | Write request / read status | Read/write |
| `Application.PLC_PRG.Discharge` | `BOOL` | Tank 1 command | Write request / read status | Read/write |
| `Application.PLC_PRG.Fill_Valve` | `BOOL` | Tank 1 output | Read only | Read/write |
| `Application.PLC_PRG.Discharge_Valve` | `BOOL` | Tank 1 output | Read only | Read/write |
| `Application.PLC_PRG.Timer` | `INT` | Tank 1 countdown | Read only | Read/write |
| `Application.PLC_PRG.Fill2` | `BOOL` | Tank 2 command | Write request / read status | Read/write |
| `Application.PLC_PRG.Discharge2` | `BOOL` | Tank 2 command | Write request / read status | Read/write |
| `Application.PLC_PRG.Fill_Valve2` | `BOOL` | Tank 2 output | Read only | Read/write |
| `Application.PLC_PRG.Discharge_Valve2` | `BOOL` | Tank 2 output | Read only | Read/write |
| `Application.PLC_PRG.Timer2` | `INT` | Tank 2 countdown | Read only | Read/write |

OPC UA support is enabled in the `Symbols` object.

The elapsed-time variables and function-block instances are correctly excluded from the present external interface. If diagnostic elapsed values are later exposed, grant read-only access unless there is a defined commissioning need.

## 11. Factory I/O Mapping Position

The export contains no `FIO` object. No change is required solely to use the current Symbol Configuration.

A future Factory I/O integration can map:

| Factory I/O role | CODESYS symbols |
|---|---|
| Operator inputs | `Fill`, `Discharge`, `Fill2`, `Discharge2` |
| Actuator indications | Four valve output variables |
| Numeric displays | `Timer`, `Timer2` |

If actual tank-level sensors are added, declare and document new input symbols rather than inferring level from elapsed time.

## 12. Missing Process Signals

The current design intentionally omits signals that a physical tank normally requires.

| Signal category | Current status |
|---|---|
| Low, high, and high-high level switches | Not modelled |
| Analogue level transmitter | Not modelled |
| Flow or pressure feedback | Not modelled |
| Fill/discharge valve position feedback | Not modelled |
| Pump run, trip, and overload feedback | Not modelled |
| Leak or bund detection | Not modelled |
| Stop, emergency, permissive, and reset inputs | Not modelled |
| Busy, done, rejected-command, and fault status | Not modelled |

## 13. Addressing and Electrical Assumptions

| Item | Current status |
|---|---|
| Fixed digital input addresses | Not configured |
| Fixed digital output addresses | Not configured |
| Analogue input/output addresses | Not configured |
| PNP/NPN wiring | Not applicable to current simulation |
| Sourcing/sinking interface | Not applicable to current simulation |
| Valve voltage and fail position | Not defined |
| Output interposing relays | Not defined |
| Safety I/O | Not present |

These details must be engineered before adapting the project to physical hardware.

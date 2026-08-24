# I/O, Variable, and Symbol List

## 1. Scope

This document records the commands, timed valve outputs, countdown values, internal variables, presets, and external symbol paths used by `twoTanksFBDlanguageRefactor`.

No variable is mapped to a fixed IEC address such as `%IX`, `%QX`, `%IW`, or `%QW`. The current integration boundary is CODESYS Symbol Configuration.

## 2. Command Variables

| Program | Signal | Type | Request transition | Initial value | Current source | Intended action |
|---|---|---|---|---:|---|---|
| `Tank1` | `Fill` | `BOOL` | False → true | `FALSE` | Online value / symbol client | Start one 15-second LD fill pulse |
| `Tank1` | `Discharge` | `BOOL` | True → false | `FALSE` | Online value / symbol client | Start one 10-second LD discharge pulse |
| `Tank2` | `Fill2` | `BOOL` | False → true | `FALSE` | Online value / symbol client | Start one 8-second FBD fill pulse |
| `Tank2` | `Discharge2` | `BOOL` | False → true | `FALSE` | Online value / symbol client | Start one 12-second FBD discharge pulse |

The discharge interaction is inconsistent between programs:

- Tank 1 reacts when an active-high momentary input is released;
- Tank 2 reacts when it is pressed.

## 3. Valve Outputs

| Program | Signal | Type | True meaning | Logic source | Current destination |
|---|---|---|---|---|---|
| `Tank1` | `Fill_Valve` | `BOOL` | Tank 1 fill pulse active | `FB_Tank_1.TP_0.Q` | Online value / symbol client |
| `Tank1` | `Discharge_Valve` | `BOOL` | Tank 1 discharge pulse active | `FB_Tank_1.TP_1.Q` | Online value / symbol client |
| `Tank2` | `Fill_Valve2` | `BOOL` | Tank 2 fill pulse active | `FB_Tank_FBDlanguage_0.TP_0.Q` | Online value / symbol client |
| `Tank2` | `Discharge_Valve2` | `BOOL` | Tank 2 discharge pulse active | `FB_Tank_FBDlanguage_0.TP_1.Q` | Online value / symbol client |

These are unconfirmed software commands. There is no valve-position, solenoid-current, flow, pressure, or actuator-fault feedback.

## 4. Countdown Values

| Program | Signal | Type | Intended meaning | Implemented source | Idle behaviour |
|---|---|---|---|---|---|
| `Tank1` | `Timer` | `INT` | Tank 1 whole seconds remaining | Correct fill/discharge references | Retains last written value |
| `Tank2` | `Timer2` | `INT` | Tank 2 whole seconds remaining | Discharge correct; fill uses incorrect 15-second reference | Retains last written value |

Both displays truncate fractional seconds. Valve status is required to determine the active phase.

## 5. Elapsed-Time Variables

| Program | Signal | Type | Source | External symbol? |
|---|---|---|---|---:|
| `Tank1` | `Fill_ET` | `TIME` | `FB_Tank_1.TP_0.ET` | No |
| `Tank1` | `Discharge_ET` | `TIME` | `FB_Tank_1.TP_1.ET` | No |
| `Tank2` | `Fill_ET2` | `TIME` | `FB_Tank_FBDlanguage_0.TP_0.ET` | No |
| `Tank2` | `Discharge_ET2` | `TIME` | `FB_Tank_FBDlanguage_0.TP_1.ET` | No |

## 6. Timing Configuration

| Tank / language | Phase | Request edge | Valve preset | Countdown maximum | Consistent? |
|---|---|---|---:|---:|---:|
| Tank 1 / LD | Fill | Rising | 15 s | 15 s | Yes |
| Tank 1 / LD | Discharge | Falling | Default 10 s | 10 s | Yes |
| Tank 2 / FBD | Fill | Rising | 8 s | **15 s** | **No** |
| Tank 2 / FBD | Discharge | Rising | 12 s | 12 s | Yes |

Tank 2's fill countdown input must be changed to `T#8S` before it can represent the valve pulse accurately.

## 7. LD `FB_Tank` Variables

| Variable | Direction | Type | Default | Purpose |
|---|---|---|---:|---|
| `Fill` | Input | `BOOL` | `FALSE` | Rising-edge fill request |
| `Fill_PT` | Input | `TIME` | `T#10S` | Fill pulse preset |
| `Discharge` | Input | `BOOL` | `FALSE` | Falling-edge discharge request |
| `Discharge_PT` | Input | `TIME` | `T#10s` | Discharge pulse preset |
| `Fill_Valve` | Output | `BOOL` | `FALSE` | Fill pulse command/status |
| `Fill_ET` | Output | `TIME` | `T#0s` | Fill elapsed time |
| `Discharge_ET` | Output | `TIME` | `T#0s` | Discharge elapsed time |
| `Discharge_Valve` | Output | `BOOL` | `FALSE` | Discharge pulse command/status |
| `TP_0` | Internal | `TP` | Reset | Fill timer instance |
| `TP_1` | Internal | `TP` | Reset | Discharge timer instance |

## 8. FBD `FB_Tank_FBDlanguage` Variables

| Variable | Direction | Type | Default | Purpose |
|---|---|---|---:|---|
| `Fill` | Input | `BOOL` | `FALSE` | Rising-edge fill request |
| `Fill_PT` | Input | `TIME` | `T#10S` | Fill pulse preset |
| `Discharge` | Input | `BOOL` | `FALSE` | Rising-edge discharge request |
| `Discharge_PT` | Input | `TIME` | `T#10S` | Discharge pulse preset |
| `Discharge_Valve` | Output | `BOOL` | `FALSE` | Discharge pulse command/status |
| `Fill_ET` | Output | `TIME` | `T#0s` | Fill elapsed time |
| `Fill_Valve` | Output | `BOOL` | `FALSE` | Fill pulse command/status |
| `Discharge_ET` | Output | `TIME` | `T#0s` | Discharge elapsed time |
| `TP_0` | Internal | `TP` | Reset | Fill timer instance |
| `TP_1` | Internal | `TP` | Reset | Discharge timer instance |

The output ordering differs from `FB_Tank`; `Tank2` maps each named output to the correct destination.

## 9. Timer Functions

| Function | Language | Inputs | Return | Intermediate declarations | Implemented calculation |
|---|---|---|---|---|---|
| `FC_Timer` | LD | `Max_Time`, `Current_Time` | `INT` | `Var1`, `Inv_Time_INT` used | `TO_INT(Max - Current) / 1000` |
| `FC_Timer_FBDlanguage` | FBD | `Max_Time`, `Current_Time` | `INT` | `Var1`, `Inv_Time_INT` unused | `TO_INT(Max - Current) / 1000` |

The signed 16-bit intermediate limits positive remaining time to approximately 32.767 seconds. Current presets are within range.

## 10. Unused `Tank1` Declarations

The following variables are declared in `Tank1` but are not connected in any of its three networks:

| Variable | Type | Likely origin |
|---|---|---|
| `FB_Tank_2` | `FB_Tank` | Former second instance in combined `PLC_PRG` |
| `Fill2` | `BOOL` | Former Tank 2 command |
| `Discharge2` | `BOOL` | Former Tank 2 command |
| `Fill_Valve2` | `BOOL` | Former Tank 2 output |
| `Fill_ET2` | `TIME` | Former Tank 2 elapsed value |
| `Discharge_ET2` | `TIME` | Former Tank 2 elapsed value |
| `Discharge_Valve2` | `BOOL` | Former Tank 2 output |
| `Timer2` | `INT` | Former Tank 2 countdown |

The active FBD variables with these short names belong to the separate `Tank2` namespace.

## 11. Symbol Configuration

The export selects these ten paths. `VarAccess = 3` and `MaxVarAccess = 3` are recorded as read/write access.

| CODESYS symbol | Type | PLC role | Recommended client use | Current access |
|---|---|---|---|---|
| `Application.Tank1.Fill` | `BOOL` | Tank 1 command | Write request / read status | Read/write |
| `Application.Tank1.Discharge` | `BOOL` | Tank 1 command | Write request / read status | Read/write |
| `Application.Tank1.Fill_Valve` | `BOOL` | Tank 1 output | Read only | Read/write |
| `Application.Tank1.Discharge_Valve` | `BOOL` | Tank 1 output | Read only | Read/write |
| `Application.Tank1.Timer` | `INT` | Tank 1 countdown | Read only | Read/write |
| `Application.Tank2.Fill2` | `BOOL` | Tank 2 command | Write request / read status | Read/write |
| `Application.Tank2.Discharge2` | `BOOL` | Tank 2 command | Write request / read status | Read/write |
| `Application.Tank2.Fill_Valve2` | `BOOL` | Tank 2 output | Read only | Read/write |
| `Application.Tank2.Discharge_Valve2` | `BOOL` | Tank 2 output | Read only | Read/write |
| `Application.Tank2.Timer2` | `INT` | Tank 2 countdown | Read only | Read/write |

OPC UA support is enabled. Elapsed-time values, function-block instances, and unused `Tank1` declarations are not selected.

## 12. Symbol Migration from the Preceding Project

| Previous path | Current path |
|---|---|
| `Application.PLC_PRG.Fill` | `Application.Tank1.Fill` |
| `Application.PLC_PRG.Discharge` | `Application.Tank1.Discharge` |
| `Application.PLC_PRG.Fill_Valve` | `Application.Tank1.Fill_Valve` |
| `Application.PLC_PRG.Discharge_Valve` | `Application.Tank1.Discharge_Valve` |
| `Application.PLC_PRG.Timer` | `Application.Tank1.Timer` |
| `Application.PLC_PRG.Fill2` | `Application.Tank2.Fill2` |
| `Application.PLC_PRG.Discharge2` | `Application.Tank2.Discharge2` |
| `Application.PLC_PRG.Fill_Valve2` | `Application.Tank2.Fill_Valve2` |
| `Application.PLC_PRG.Discharge_Valve2` | `Application.Tank2.Discharge_Valve2` |
| `Application.PLC_PRG.Timer2` | `Application.Tank2.Timer2` |

Update all Factory I/O, KEPServerEX, OPC, HMI, test, and trace configurations that use the former namespace.

## 13. Factory I/O Position

The export contains no `FIO` object. No separate mapping POU is required solely to use the current symbols.

Before integration:

- correct Tank 2's fill countdown;
- choose a consistent discharge edge;
- regenerate symbols; and
- verify the final names from the running application.

## 14. Missing Process Signals

| Signal category | Current status |
|---|---|
| Low, high, and high-high level switches | Not modelled |
| Analogue level transmitters | Not modelled |
| Flow and pressure feedback | Not modelled |
| Valve position feedback | Not modelled |
| Leak or bund detection | Not modelled |
| Stop, emergency, reset, and permissive inputs | Not modelled |
| Busy, done, rejected-command, and fault status | Not modelled |

## 15. Addressing and Electrical Assumptions

| Item | Current status |
|---|---|
| Fixed digital input addresses | Not configured |
| Fixed digital output addresses | Not configured |
| Analogue input/output addresses | Not configured |
| PNP/NPN wiring | Not applicable to current simulation |
| Valve voltage and fail position | Not defined |
| Output interposing and protection | Not defined |
| Safety I/O | Not present |

These details must be engineered before any physical adaptation.

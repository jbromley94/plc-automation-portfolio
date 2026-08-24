# Control Philosophy and Refactor Comparison

## 1. Purpose

This document describes `twoTanksFBDlanguageRefactor` as reconstructed from its native CODESYS export.

The project is a side-by-side graphical-language exercise. It preserves the original Ladder Diagram implementation for Tank 1 and introduces parallel Function Block Diagram POUs for Tank 2. The intended process remains two independent, timed valve simulations with whole-second countdown values.

The review distinguishes visual refactoring from behavioural equivalence. The exported FBD path contains two observable differences from the preceding LD implementation:

- Tank 2 discharge changed from falling-edge to rising-edge operation; and
- Tank 2's eight-second fill pulse is paired with a fifteen-second countdown reference.

## 2. Execution Model

| Property | Configured value |
|---|---|
| CODESYS profile | V3.5 SP22 Patch 3 |
| Runtime target | CODESYS Control Win V3 x64 |
| Device version | 3.5.22.30 |
| Task | `MainTask` |
| Task type | Cyclic |
| Interval | 20 ms |
| Priority | 1 |
| Watchdog | Disabled |
| Program order | `Tank1`, then `Tank2` |

The application contains twelve implementation networks across six POUs. `Tank1` and `Tank2` are both called every normal task scan.

## 3. POU Architecture

| POU | Kind | Language | Networks | Runtime role |
|---|---|---|---:|---|
| `FB_Tank` | Function block | LD | 2 | Original timed-valve logic for Tank 1 |
| `FC_Timer` | Function | LD | 1 | Original remaining-seconds calculation for Tank 1 |
| `Tank1` | Program | LD | 3 | Calls the original LD POUs |
| `FB_Tank_FBDlanguage` | Function block | FBD | 2 | Refactored timed-valve logic for Tank 2 |
| `FC_Timer_FBDlanguage` | Function | FBD | 1 | Refactored remaining-seconds calculation for Tank 2 |
| `Tank2` | Program | FBD | 3 | Calls the refactored FBD POUs |

The language-specific names are useful for this comparison exercise. In a consolidated application, the implementation language should not normally be encoded in the public functional name.

## 4. Main Task and Program Split

The preceding project used one six-network `PLC_PRG` containing both tank instances. This export separates those calls:

| MainTask position | Program | Active implementation |
|---:|---|---|
| 1 | `Tank1` | Original LD `FB_Tank` and `FC_Timer` |
| 2 | `Tank2` | New FBD `FB_Tank_FBDlanguage` and `FC_Timer_FBDlanguage` |

The two programs use local variables and do not exchange process data. The task order therefore does not create a normal cross-tank dependency, although it remains relevant to trace timestamps and future shared-resource logic.

## 5. Original LD Function Block

`FB_Tank` is retained from the preceding project.

### 5.1 Interface

| Variable | Direction | Type | Default | Meaning |
|---|---|---|---:|---|
| `Fill` | Input | `BOOL` | `FALSE` | Rising-edge fill request |
| `Fill_PT` | Input | `TIME` | `T#10S` | Fill pulse duration |
| `Discharge` | Input | `BOOL` | `FALSE` | Falling-edge discharge request |
| `Discharge_PT` | Input | `TIME` | `T#10s` | Discharge pulse duration |
| `Fill_Valve` | Output | `BOOL` | `FALSE` | Fill pulse status/command |
| `Fill_ET` | Output | `TIME` | `T#0s` | Fill elapsed time |
| `Discharge_ET` | Output | `TIME` | `T#0s` | Discharge elapsed time |
| `Discharge_Valve` | Output | `BOOL` | `FALSE` | Discharge pulse status/command |

### 5.2 Networks

| Network | Reconstructed expression |
|---:|---|
| 1 | `TP_0(IN := R_EDGE(Fill) AND NOT Discharge_Valve, PT := Fill_PT)` |
| 2 | `TP_1(IN := F_EDGE(Discharge) AND NOT Fill_Valve, PT := Discharge_PT)` |

The exported graphical flags are `16` for the fill rising edge and `32` for the discharge falling edge.

## 6. Refactored FBD Function Block

`FB_Tank_FBDlanguage` has the same four inputs, four outputs, defaults, and two `TP` instances. Its graphical networks implement:

| Network | Reconstructed expression |
|---:|---|
| 1 | `TP_0(IN := R_EDGE(Fill) AND NOT Discharge_Valve, PT := Fill_PT)` |
| 2 | `TP_1(IN := R_EDGE(Discharge) AND NOT Fill_Valve, PT := Discharge_PT)` |

Both command operands use exported flag `16`, so both are rising-edge requests.

The output declaration order differs from the LD function block:

1. `Discharge_Valve`;
2. `Fill_ET`;
3. `Fill_Valve`;
4. `Discharge_ET`.

`Tank2` connects each output to the matching named variable, so no incorrect output route was found. The unusual order still increases visual comparison effort and should be normalised if both versions are retained.

## 7. Valve Interlocks and Arbitration

Both function blocks prevent their own fill and discharge valve pulses from overlapping:

- the fill request is ANDed with `NOT Discharge_Valve`;
- the discharge request is ANDed with `NOT Fill_Valve`; and
- the fill network executes before the discharge network.

For either tank, a request received while the opposite valve is active is discarded, not queued. A same-phase edge received while its `TP` is already active does not create a later pulse.

If both configured request edges occur in one idle scan, fill starts in network 1. Network 2 then reads the new fill output and blocks discharge.

The two tanks do not interlock each other. Both may fill or discharge concurrently.

## 8. Original LD Timer Function

`FC_Timer` retains the preceding three-stage LD calculation and writes its local intermediates:

```text
Var1         := Max_Time - Current_Time
Inv_Time_INT := TO_INT(Var1)
FC_Timer     := Inv_Time_INT / 1000
```

It returns truncated whole seconds as an `INT`.

## 9. Refactored FBD Timer Function

`FC_Timer_FBDlanguage` expresses the same nominal calculation as one nested FBD chain:

```text
FC_Timer_FBDlanguage := TO_INT(Max_Time - Current_Time) / 1000
```

The function still declares `Var1 : TIME` and `Inv_Time_INT : INT`, but neither is connected in the FBD network. They are unused remnants of the LD implementation.

Both timer functions convert the remaining milliseconds to signed 16-bit `INT` before division. Their safe positive duration range is therefore approximately 0 to 32.767 seconds. All current presets are inside this range, but the implementation is not safely reusable for longer times.

## 10. `Tank1` Program

`Tank1` contains three LD networks:

| Network | Call | Connections |
|---:|---|---|
| 1 | `FB_Tank_1 : FB_Tank` | `Fill`, `T#15S`, `Discharge`, default discharge preset; outputs to Tank 1 variables |
| 2 | `FC_Timer` | Enabled by `Fill_Valve`; `T#15S - Fill_ET`; result to `Timer` |
| 3 | `FC_Timer` | Enabled by `Discharge_Valve`; `T#10S - Discharge_ET`; result to `Timer` |

Implemented Tank 1 behaviour:

- fill starts on the rising edge of `Fill` and lasts 15 seconds;
- discharge starts on the falling edge of `Discharge` and lasts 10 seconds; and
- `Timer` represents whichever phase is active.

The discharge-preset pin is unconnected and relies on the `FB_Tank` default of `T#10s`. The separate countdown call uses an explicit `T#10S`.

## 11. Unused `Tank1` Declarations

`Tank1` reuses the declaration formerly attached to the combined `PLC_PRG`. It still declares these unused Tank 2 items:

- `FB_Tank_2`;
- `Fill2` and `Discharge2`;
- `Fill_Valve2` and `Discharge_Valve2`;
- `Fill_ET2` and `Discharge_ET2`; and
- `Timer2`.

None appears in the three `Tank1` networks, and none is selected under the `Tank1` symbol group.

This creates duplicate names such as `Tank1.Fill2` and the active `Tank2.Fill2` in different program namespaces. Remove the unused declarations to prevent browsing and maintenance ambiguity.

## 12. `Tank2` Program

`Tank2` contains three FBD networks:

| Network | Call | Connections |
|---:|---|---|
| 1 | `FB_Tank_FBDlanguage_0` | Always enabled; `Fill2`, `T#8S`, `Discharge2`, `T#12S`; all outputs mapped by name |
| 2 | `FC_Timer_FBDlanguage` | Enabled by `Fill_Valve2`; **`T#15S - Fill_ET2`**; result to `Timer2` |
| 3 | `FC_Timer_FBDlanguage` | Enabled by `Discharge_Valve2`; `T#12S - Discharge_ET2`; result to `Timer2` |

Implemented Tank 2 behaviour:

- fill starts on the rising edge of `Fill2` and lasts 8 seconds;
- discharge starts on the rising edge of `Discharge2` and lasts 12 seconds;
- the discharge countdown is consistent with its pulse; and
- the fill countdown is inconsistent with its pulse.

## 13. Tank 2 Fill Countdown Mismatch

The valve timer and display calculation use different maximum times:

| Item | Configured value |
|---|---:|
| `FB_Tank_FBDlanguage_0.Fill_PT` | 8 s |
| Fill `FC_Timer_FBDlanguage.Max_Time` | 15 s |

The expected static progression is therefore approximately:

| Fill phase point | `Fill_ET2` | Calculated `Timer2` |
|---|---:|---:|
| Start | 0 s | 15 |
| Mid-pulse | 4 s | 11 |
| Just before 8-second completion | Just under 8 s | Approximately 7 |
| Idle after completion | No enabled fill call | Last value retained |

The exact sampled sequence depends on task timing and must be traced. The configuration is nevertheless unambiguously inconsistent in the export.

Required correction: connect `T#8S` to the Tank 2 fill countdown call, then verify an 8-to-0 display and repeat regression tests.

## 14. Countdown Selection and Idle Behaviour

Each tank uses one integer for both phase countdowns.

- The fill function call is enabled by the fill valve output.
- The discharge function call is enabled by the discharge valve output.
- Mutual exclusion normally prevents both calls from writing in one scan.
- When both valves are false, neither call explicitly writes the countdown.

The value therefore retains its last result while idle. A client needs valve status to determine the current phase and whether the integer is live.

## 15. Behavioural Comparison

| Behaviour | Tank 1 / LD | Tank 2 / FBD | Equivalent? |
|---|---|---|---:|
| Fill request edge | Rising | Rising | Yes |
| Fill pulse | 15 s | 8 s | Intentionally different preset |
| Fill countdown matches pulse | Yes | No: 15 s reference vs 8 s pulse | **No** |
| Discharge request edge | Falling | Rising | **No** |
| Discharge pulse | 10 s | 12 s | Intentionally different preset |
| Local valve interlock | Present | Present | Yes |
| Fill-before-discharge priority | Present | Present | Yes |
| Countdown formula | Intermediate LD chain | Direct FBD chain | Nominally equivalent |
| Countdown numeric range | 16-bit intermediate | 16-bit intermediate | Yes |

## 16. Symbol Configuration and Namespace Migration

The export exposes ten read/write variables in two symbol groups:

| Group | Selected variables |
|---|---|
| `Tank1` | `Fill`, `Discharge`, `Fill_Valve`, `Discharge_Valve`, `Timer` |
| `Tank2` | `Fill2`, `Discharge2`, `Fill_Valve2`, `Discharge_Valve2`, `Timer2` |

The preceding application exposed the same short variable names beneath `Application.PLC_PRG`. Any Factory I/O, OPC, KEPServerEX, HMI, trace, or script using the old paths requires migration.

All selected variables use access value `3`, recorded here as read/write. External write permission is broader than necessary for the four valve outputs and two countdown values.

## 17. Startup and Recovery

The program variables have no explicit initialisers, so the normal initial values are zero or false.

There is no application-level start, stop, emergency, reset, alarm acknowledgement, or recovery state. Edge-memory initialisation with command signals already true at startup must be runtime tested for both language paths.

## 18. Physical I/O and Factory I/O Boundary

No fixed IEC addresses and no `FIO` object are present.

The selected symbols are sufficient for a simple external simulation. Before mapping a client, correct the Tank 2 countdown and decide whether the unequal discharge-edge conventions are intentional.

The project has no tank-level feedback. A visual water level would be open loop unless new feedback signals and control requirements are added.

## 19. Safety and Process Limitations

The application has no:

- high, low, or independent high-high level protection;
- valve-position or flow feedback;
- leak, pressure, or actuator diagnostics;
- process timeout fault separate from the command pulse;
- stop, emergency, reset, or safe recovery sequence;
- physical output design; or
- runtime verification evidence.

Timed Boolean outputs in a Windows soft PLC do not constitute safe physical tank control.

## 20. Improvement Backlog

- Correct Tank 2 fill `Max_Time` to `T#8S`.
- Choose and document one discharge edge convention.
- Apply identical input traces to both language implementations and compare outputs scan by scan.
- Consolidate the two versions after the learning comparison to prevent further drift.
- Remove eight unused `Tank1` declarations.
- Remove or reconnect the unused FBD timer locals.
- Centralise presets so a valve pulse and its display cannot use different literals.
- Decide and document the final external namespaces.
- Restrict symbol access by interface role.
- Use a wider timer intermediate and define an explicit idle value.
- Add process feedback, permissives, diagnostics, and recovery.
- Add network titles and comments in CODESYS.

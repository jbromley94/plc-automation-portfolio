# Control Philosophy

## 1. Purpose

This document describes the implemented behaviour of `twoWaterTanksFuncAndBlockContinued` as reconstructed from its native CODESYS export.

The project demonstrates reusable PLC software rather than a complete water process. Two independent tank instances accept Boolean fill and discharge commands, energise their valve outputs for fixed pulse times, and publish a whole-second countdown.

There are no level sensors, flow sensors, pumps, alarms, physical addresses, process permissives, or explicit safety functions.

## 2. Execution Model

| Property | Configured value |
|---|---|
| CODESYS profile | V3.5 SP22 Patch 3 |
| Runtime target | CODESYS Control Win V3 x64 |
| Device version | 3.5.22.30 |
| Application program | `PLC_PRG` |
| Language | Ladder Diagram (LD) |
| Task | `MainTask` |
| Task type | Cyclic |
| Interval | 20 ms |
| Priority | 1 |
| Watchdog | Disabled |

`PLC_PRG` contains six networks. It calls two other POUs whose implementations contain three more networks, giving nine application LD networks in total.

No explicit enumerated state machine is implemented. Each tank's active state is held inside its two `TP` instances.

## 3. Software Architecture

| POU | Kind | Networks | Responsibility |
|---|---|---:|---|
| `FB_Tank` | Function block | 2 | Create mutually exclusive timed fill and discharge valve pulses |
| `FC_Timer` | Function returning `INT` | 1 | Calculate truncated whole seconds remaining |
| `PLC_PRG` | Program | 6 | Instantiate two tanks, set presets, and route countdown results |

The central learning outcome is instance reuse: `FB_Tank_1` and `FB_Tank_2` execute the same compiled logic while retaining independent timer state.

## 4. `FB_Tank` Interface

### 4.1 Inputs

| Input | Type | Default | Implemented use |
|---|---|---:|---|
| `Fill` | `BOOL` | `FALSE` | Rising edge requests a fill pulse |
| `Fill_PT` | `TIME` | `T#10S` | Fill pulse duration |
| `Discharge` | `BOOL` | `FALSE` | Falling edge requests a discharge pulse |
| `Discharge_PT` | `TIME` | `T#10s` | Discharge pulse duration |

### 4.2 Outputs

| Output | Type | Meaning |
|---|---|---|
| `Fill_Valve` | `BOOL` | Fill `TP.Q`; true while the fill pulse is active |
| `Fill_ET` | `TIME` | Fill `TP.ET`; elapsed fill time |
| `Discharge_ET` | `TIME` | Discharge `TP.ET`; elapsed discharge time |
| `Discharge_Valve` | `BOOL` | Discharge `TP.Q`; true while the discharge pulse is active |

### 4.3 Internal state

| Variable | Type | Role |
|---|---|---|
| `TP_0` | `TP` | Fill pulse timer |
| `TP_1` | `TP` | Discharge pulse timer |

Because `FB_Tank` is a function block, each instance owns a separate copy of `TP_0`, `TP_1`, and their edge history.

## 5. `FB_Tank` Network Traceability

| Network | Reconstructed expression | Outputs |
|---:|---|---|
| 1 | `TP_0(IN := R_EDGE(Fill) AND NOT Discharge_Valve, PT := Fill_PT)` | `Q → Fill_Valve`, `ET → Fill_ET` |
| 2 | `TP_1(IN := F_EDGE(Discharge) AND NOT Fill_Valve, PT := Discharge_PT)` | `Q → Discharge_Valve`, `ET → Discharge_ET` |

The native graphical flags are `16` on `Fill` and `32` on `Discharge`, corresponding to rising- and falling-edge contacts in the exported LD representation. These symbols should be visually confirmed after import and included in the runtime evidence.

## 6. Fill Operation

The fill network executes first in the function block.

1. A false-to-true transition on `Fill` generates a one-scan request.
2. A normally closed `Discharge_Valve` contact permits the request only when discharge is inactive.
3. `TP_0` sets `Fill_Valve` for `Fill_PT`.
4. `Fill_ET` reports the elapsed pulse time.
5. At the preset time, `Fill_Valve` returns false automatically.

`Fill` is not a maintained run command. Holding it true does not extend or repeat the pulse. It must return false and then rise again before another fill request can be created.

A second fill edge received while `TP_0` is already active is not queued for later execution.

## 7. Discharge Operation

The discharge network executes after the fill network.

1. A true-to-false transition on `Discharge` generates a one-scan request.
2. A normally closed `Fill_Valve` contact permits the request only when fill is inactive.
3. `TP_1` sets `Discharge_Valve` for `Discharge_PT`.
4. `Discharge_ET` reports the elapsed pulse time.
5. At the preset time, `Discharge_Valve` returns false automatically.

This convention is intentionally documented literally: changing `Discharge` from false to true does not start the pulse. A conventional active-high momentary button starts discharge on release, when its value falls back to false.

The initial false value does not itself constitute a falling edge. The input must first become true and then false.

A second discharge edge received while `TP_1` is active is not queued.

## 8. Mutual Exclusion and Command Arbitration

The interlock is local to each tank instance.

| Condition when an edge arrives | Implemented result |
|---|---|
| Fill edge; both valves off | Fill pulse starts |
| Discharge edge; both valves off | Discharge pulse starts |
| Fill edge while discharge valve is on | Fill request is rejected |
| Discharge edge while fill valve is on | Discharge request is rejected |
| New edge for the valve already pulsing | `TP` remains on its current pulse; request is not queued |

If a fill rising edge and discharge falling edge occur in the same scan while both valves are initially off, fill has effective priority:

1. Network 1 starts `Fill_Valve`.
2. Network 2 then reads `Fill_Valve = TRUE` and blocks discharge.

This priority comes from scan order rather than an explicit arbiter or state machine.

Rejected requests are silent. There is no busy output, queue, alarm, or command acknowledgement.

## 9. `FC_Timer` Interface and Calculation

### 9.1 Interface

| Item | Type | Purpose |
|---|---|---|
| `Max_Time` | `TIME` input | Configured pulse duration |
| `Current_Time` | `TIME` input | Current `TP.ET` value |
| `FC_Timer` | `INT` return | Truncated remaining seconds |
| `Var1` | `TIME` local | Intermediate remaining time |
| `Inv_Time_INT` | `INT` local | Remaining milliseconds after conversion |

### 9.2 Network operation

The one network performs three chained operations:

```text
Var1        := Max_Time - Current_Time
Inv_Time_INT := TO_INT(Var1)
FC_Timer    := Inv_Time_INT / 1000
```

The result is not rounded; integer division truncates the fractional part. For example, 7.980 seconds remaining is displayed as `7`.

The conversion occurs before division. An IEC `INT` is signed 16-bit, so a positive millisecond value must not exceed 32,767. The current 8-, 10-, 12-, and 15-second presets are inside this range. A future preset above approximately 32.767 seconds can overflow before the `/ 1000` operation.

`TP.ET` should remain between zero and its preset during normal operation. `FC_Timer` contains no explicit clamp for `Current_Time > Max_Time` or for a negative/overflowed result.

## 10. `PLC_PRG` Network Traceability

| Network | Call | Key connections |
|---:|---|---|
| 1 | `FB_Tank_1` | `Fill`, `T#15S`, `Discharge`, default `Discharge_PT`; outputs to Tank 1 variables |
| 2 | `FC_Timer` | Enabled by `Fill_Valve`; `T#15S - Fill_ET`; result to `Timer` |
| 3 | `FC_Timer` | Enabled by `Discharge_Valve`; `T#10S - Discharge_ET`; result to `Timer` |
| 4 | `FB_Tank_2` | `Fill2`, `T#8S`, `Discharge2`, `T#12S`; outputs to Tank 2 variables |
| 5 | `FC_Timer` | Enabled by `Fill_Valve2`; `T#8S - Fill_ET2`; result to `Timer2` |
| 6 | `FC_Timer` | Enabled by `Discharge_Valve2`; `T#12S - Discharge_ET2`; result to `Timer2` |

Tank 1's `Discharge_PT` pin is unconnected. Its instance therefore relies on the declaration initializer `T#10s`. The corresponding `FC_Timer` call uses the same 10-second value explicitly.

## 11. Configured Tank Instances

| Property | Tank 1 | Tank 2 |
|---|---|---|
| Instance | `FB_Tank_1` | `FB_Tank_2` |
| Fill request | `Fill` rising edge | `Fill2` rising edge |
| Fill preset | `T#15S` | `T#8S` |
| Fill output | `Fill_Valve` | `Fill_Valve2` |
| Discharge request | `Discharge` falling edge | `Discharge2` falling edge |
| Discharge preset | Default `T#10s` | `T#12S` |
| Discharge output | `Discharge_Valve` | `Discharge_Valve2` |
| Shared phase display | `Timer` | `Timer2` |

There is no connection between the two tanks. The project does not transfer water from Tank 1 to Tank 2 or coordinate their capacity.

## 12. Countdown Selection and Idle Behaviour

Each tank has one countdown variable shared by its fill and discharge phases.

The fill `FC_Timer` call is enabled only while the fill valve is true. The discharge call is enabled only while the discharge valve is true. Under the implemented mutual exclusion, only one call writes the tank's countdown during a normal scan.

When both valves are false, both calls are disabled. No network explicitly writes zero or an idle sentinel, so the countdown retains its last written value. With normal 20 ms execution, the final active calculation is expected to reach zero near pulse completion, but this must be confirmed at runtime rather than assumed as an explicit reset.

Using one variable for both phases also means an external client must inspect the valve outputs to know whether the value belongs to filling, discharging, or an idle retained result.

## 13. Concurrency

The two `FB_Tank` instances share code but not state.

Valid concurrent combinations include:

- Tank 1 filling while Tank 2 fills;
- Tank 1 filling while Tank 2 discharges;
- Tank 1 discharging while Tank 2 fills; and
- one tank active while the other remains idle.

The one-valve-at-a-time rule applies within an instance only. There is no site-wide utility limit, pump interlock, flow limit, or shared-resource arbitration.

## 14. Startup and Recovery

All `PLC_PRG` variables have no explicit initializer, so their normal initial values are zero or false.

On a clean start:

- both valve outputs are false;
- both countdown integers are zero;
- a false fill command has not generated a rising edge; and
- a false discharge command has not generated a falling edge.

There is no start, stop, emergency, reset, alarm acknowledgement, or retained recovery state. Restart behaviour after a download or runtime interruption should be verified with the command signals in both false and true states.

## 15. External Interface

The `Symbols` object exposes ten `PLC_PRG` variables and has OPC UA support enabled.

All selected variables use access value `3`, documented here as read/write. This includes PLC-generated outputs and countdown values. Those values are rewritten by PLC logic during execution, so external writes may be transient or can interfere with testing and interlock interpretation.

A production-style interface should separate command authority from status observation and grant the minimum required client permissions.

## 16. Physical I/O and Factory I/O Boundary

No fixed IEC addresses are present, and the export does not contain an `FIO` object.

For the current simulated project, a separate `FIO` mapping POU is optional: Factory I/O or another client can use the selected symbols directly. If the project is later connected to physical inputs and outputs, add a deliberate I/O abstraction layer that handles polarity, forcing policy, simulation mode, and hardware diagnostics.

## 17. Safety and Process Limitations

This project is not suitable for direct physical use because it has no:

- high-high level trip or independent overfill protection;
- low-level or dry-run protection;
- valve position or flow feedback;
- maximum fill/discharge timeout distinct from the operating command;
- leak, sensor disagreement, or actuator fault detection;
- stop or emergency function;
- manual recovery state;
- physical I/O design; or
- runtime verification evidence.

Timed energisation alone does not establish a safe or repeatable tank level.

## 18. Improvement Backlog

- Confirm whether falling-edge discharge is deliberate; otherwise change it to rising edge.
- Add `Busy`, `Done`, `CommandRejected`, and `Fault` outputs to `FB_Tank`.
- Make arbitration explicit instead of relying on network order.
- Decide whether requests received while busy should be rejected or queued.
- Wire all presets explicitly at each call site, even when they match defaults.
- Give fill and discharge separate countdown/status values or publish an explicit phase.
- Use `DINT` or a duration-safe conversion in `FC_Timer`, with range clamping.
- Restrict symbol writes to command variables.
- Add level feedback, plausibility checks, process permissives, timeouts, and fault reset.
- Add network titles and comments in CODESYS.
- Capture build, trace, and interface evidence against the verification plan.

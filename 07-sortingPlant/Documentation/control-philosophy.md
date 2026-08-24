# Control Philosophy

## 1. Purpose

This document describes the control behaviour reconstructed from the supplied `sortingPlant.export` file.

The application is a CODESYS learning project for a loading, lifting, horizontal transport, storage, and return sequence. Native SFC provides the sequence structure, while FBD, LD, and ST implement run control, step actions, numeric target handling, and movement-complete transitions.

The source calls the project `sortingPlant`, but no product property is measured and no destination is selected from a classification rule. The current destination policy is simply `NextPosition := NextPosition + 1` once per transport step.

## 2. Source and Verification Boundary

Only a portable CODESYS export was supplied:

- `CODESYS/sortingPlant.export`

No native `.project`, Factory I/O scene, KEPServerEX configuration, wiring schedule, electrical design, risk assessment, or online trace was provided.

This documentation is therefore a static reconstruction. Project hierarchy, declarations, graphical operands, SFC properties, transitions, actions, task settings, symbol selections, and source hashes are confirmed from the export. Compilation, generated-code behaviour, signal polarity, motion response, timing, communication, and physical results remain subject to the tests in `verification.md`.

## 3. Configured Execution Environment

| Property | Exported value |
|---|---|
| CODESYS profile | V3.5 SP22 Patch 3 |
| Target | CODESYS Control Win V3 x64 |
| Device version | 3.5.22.30 |
| Task | `MainTask` |
| Task type | Cyclic |
| Interval | 20 ms |
| Priority | 1 |
| Watchdog | Disabled |
| Program order | `SFC_PRG`, then `Control` |
| Task group | `IEC-Tasks` |

CODESYS Control Win is described in the export as a Windows soft PLC with non-real-time capabilities. The configured 20 ms interval is nominal; host scheduling jitter should be measured if time-sensitive behaviour matters.

## 4. Application Architecture

| Object | Kind | Language | Role |
|---|---|---|---|
| `FIO` | Global variable list | ST declaration | Qualified-only interface for 26 scene, command, actuator, light, and position values |
| `Control` | Program | FBD | Reset-dominant run latch and intended SFC initialisation pulse |
| `SFC_PRG` | Program | SFC | Active process sequence |
| `Emergency_active` | SFC child action | ST | Clears Boolean actuators/lights and commands target 0 |
| `Reset_active` | SFC child action | ST | Clears Boolean actuators/lights and conditionally commands target 55 |
| `Transporting_entry` | SFC entry action | LD | Increments `NextPosition` once on step activation |
| `Transporting_active` | SFC step action | LD | Moves `NextPosition` into `FIO.TargetPosition` |
| `GoToInitalPosition_active` | SFC step action | LD | Moves literal 55 into `FIO.TargetPosition` |
| `End_MovingX` | SFC transition object | FBD | Falling-edge detection for transport completion |
| `EndMovingX1` | SFC transition object | FBD | Falling-edge detection for return-home completion |
| `Symbols` | Symbol Configuration | — | Publishes all `FIO` variables read/write; OPC UA support enabled |
| `MainTask` | Task | — | Calls both programs every 20 ms |

The spelling `GoToInitalPosition` is the source spelling and is retained throughout the documentation.

## 5. Resolved Libraries

| Library or placeholder | Exported resolution |
|---|---|
| `IoStandard` | 3.5.22.0, System |
| `3SLicense` | 0.0.0.0, CODESYS |
| `CAA Device Diagnosis` | 3.5.22.0, CAA Technical Workgroup |
| `Standard` | Version placeholder; exported redirection resolves to 3.5.22.0 |
| `Breakpoint Logging Functions` | Version placeholder |
| `IecSfc` | 4.4.0.0, System |
| `Analyzation` | 4.1.0.0, System |
| `IecVarAccess` | 3.3.1.20, System |

`IecSfc` supports the SFC action qualifiers. `IecVarAccess` is present with the Symbol Configuration feature.

## 6. Program Declarations

### 6.1 `Control`

```iecst
PROGRAM Control
VAR
    RS_0: RS;
    RunningMode: BOOL;
    R_TRIG_0: R_TRIG;
END_VAR
```

### 6.2 `SFC_PRG`

```iecst
PROGRAM SFC_PRG
VAR
    NextPosition: WORD := 0;
    F_TRIG_0: F_TRIG;
    F_TRIG_1: F_TRIG;
END_VAR
VAR_INPUT
    SFCinit: BOOL;
END_VAR
```

`NextPosition` is retained as program state during ordinary scans. The export does not assign it in the reset or emergency actions.

## 7. Run/Stop Philosophy

### 7.1 Reset-dominant run latch

The first `Control` FBD network is equivalent to:

```iecst
RunningMode := RS_0(
    SET := FIO.Start,
    RESET1 := NOT FIO.Stop
              OR NOT FIO.EmergencyStop
              OR FIO.Reset
).Q1;
```

The exact generated syntax may differ, but these are the configured operands, inversions, and pins.

The signal convention implied by this network is:

| Signal | Healthy/inactive value | Active/fault value | Effect |
|---|---:|---:|---|
| `FIO.Stop` | `TRUE` | `FALSE` | False resets `RunningMode` |
| `FIO.EmergencyStop` | `TRUE` | `FALSE` | False resets `RunningMode` |
| `FIO.Reset` | `FALSE` | `TRUE` | True resets `RunningMode` |
| `FIO.Start` | `FALSE` | `TRUE` | True sets `RunningMode` when no reset condition exists |

This resembles normally closed healthy-true stop channels at the software interface, but it is not proof of the actual Factory I/O mapping or electrical wiring.

`RS` is reset-dominant. If `Start` and any reset condition are true in the same call, `RunningMode` remains or becomes false.

### 7.2 Level-sensitive start consequence

`FIO.Start` drives the set input directly; there is no rising-edge detector. If `Start` remains true while a stop, emergency, or reset condition clears, the run latch can set without a fresh button edge.

For controlled restart, use a defined start pulse or edge, require all permissives healthy, and require a deliberate new acknowledgement after the inhibiting condition has cleared.

### 7.3 What `RunningMode` actually stops

`Control.RunningMode` appears in only one SFC transition: `Normal_Sequence → Loading`.

Once the chart has left `Normal_Sequence`, neither stop nor emergency state is checked in the receiving, transporting, storing, or return-home path. Resetting `RunningMode` prevents the next cycle from starting after the chart returns, but does not by itself stop the cycle already in progress.

This distinction is central to the current control philosophy:

- `RunningMode` is a cycle-start permissive/latch;
- it is not an immediate sequence stop command; and
- it is not an output safety override.

## 8. Intended SFC Initialisation

The second `Control` network is equivalent to:

```iecst
SFC_PRG.SFCinit := R_TRIG_0(
    CLK := NOT FIO.EmergencyStop OR FIO.Reset
).Q;
```

The intent appears to be a one-scan SFC initialisation request when:

- the emergency-stop signal changes from healthy true to false; or
- reset changes from false to true.

### 8.1 Exported SFC flag configuration

The `SFC_PRG` properties contain these key values:

| Setting | Exported value |
|---|---|
| Use default SFC settings | `FALSE` |
| `Init` — Use | `FALSE` |
| `Init` — Declare | `TRUE` |
| All other optional SFC flags — Use | `FALSE` |
| Calculate active transitions only | `FALSE` |

CODESYS requires an SFC flag to be declared **and activated**. A flag that is only declared has no SFC function. The manually declared `SFCinit` input is writable from `Control`, but `Init — Use = FALSE` means it does not reset the chart in this exported revision.

### 8.2 Combined-edge limitation

Even after the SFC flag is activated, a single `R_TRIG` monitors the OR of two conditions.

If `NOT EmergencyStop` is already true, pressing `Reset` does not create a new false-to-true edge at the OR output. Similarly, overlapping events are merged into one high level. If separate event acknowledgement matters, use separate edge detectors or implement explicit level-based reset logic with a documented priority.

### 8.3 Correction options

An engineering revision should choose and test one clear model:

1. activate `SFCInit` in the SFC POU settings, retain the explicit `VAR_INPUT`, and disable automatic `Declare` so there is exactly one externally writable flag;
2. use `SFCReset` if processing of the initial step during reset is the required behaviour;
3. implement an explicit high-priority stop/fault branch in the chart; or
4. place safe-output arbitration outside the sequence and treat chart reset only as recovery coordination.

Regardless of the software choice, emergency stopping of hazardous motion must be implemented independently in safety-rated hardware and logic.

## 9. Initial and Emergency Branch

`Initial` is the sole initial step. It applies:

```text
S FIO.ResetLight
```

The following `Branch0` is an alternative branch with `Parallel = FALSE`.

| Branch order | Guard | Destination |
|---:|---|---|
| 1 | `NOT FIO.EmergencyStop` | `Emergency` |
| 2 | `TRUE` | `Reset` |

CODESYS evaluates alternative branch transitions from left to right. The first true branch opens.

Therefore:

- `EmergencyStop = FALSE` selects `Emergency`; and
- `EmergencyStop = TRUE` falls through to the unconditional `Reset` branch.

### 9.1 Emergency step

`Emergency` calls the ST action `Emergency_active` for as long as the step is active:

```iecst
FIO.EntryConveyor := FALSE;
FIO.ExitConveyor := FALSE;
FIO.ForksLeft := FALSE;
FIO.ForksRight := FALSE;
FIO.Lift := FALSE;
FIO.LoadConveyor := FALSE;
FIO.ResetLight := FALSE;
FIO.StartLight := FALSE;
FIO.StopLight := FALSE;
FIO.UnloadConveyor := FALSE;
FIO.TargetPosition := 0;
```

The exit guard is:

```iecst
FIO.EmergencyStop AND FIO.Reset
```

The operator must therefore restore the healthy emergency signal and assert reset before the chart can enter `Reset`.

### 9.2 Reachability limitation

The `Emergency` step is connected only to the initial branch. There is no global branch from `Loading`, either macro, `Transporting`, or `GoToInitalPosition` into `Emergency`.

Because the intended `SFCinit` input is inactive, an emergency event during an operating cycle does not make this action execute. Static source review therefore cannot claim that the listed false assignments are enforced during a mid-cycle emergency.

## 10. Reset Philosophy

`Reset` calls `Reset_active`:

```iecst
FIO.EntryConveyor := FALSE;
FIO.ExitConveyor := FALSE;
FIO.ForksLeft := FALSE;
FIO.ForksRight := FALSE;
FIO.Lift := FALSE;
FIO.LoadConveyor := FALSE;
FIO.ResetLight := FALSE;
FIO.StartLight := FALSE;
FIO.StopLight := FALSE;
FIO.UnloadConveyor := FALSE;

IF FIO.AtMiddle THEN
    FIO.TargetPosition := 55;
END_IF
```

The code block preserves the exported text: the final `END_IF` has no semicolon. CODESYS ST normally uses `END_IF;`. Record the baseline build result and add the missing terminator in a corrected revision if the compiler rejects it.

The transition to `Normal_Sequence` is:

```iecst
NOT FIO.MovingX
```

### 10.1 Retained target concern

When `AtMiddle = FALSE`, `Reset_active` makes no assignment to `TargetPosition`. The value retains its previous value, which may have come from emergency target 0, a storage target, a prior home command, or an external write.

### 10.2 Reset-complete concern

The reset transition proves only that `MovingX` is false. It does not prove:

- a specific home sensor is active;
- `TargetPosition = 55` was issued;
- the target was accepted;
- X motion started and completed;
- Z motion is stationary; or
- forks and lift feedback match the commanded reset state.

If `MovingX` is already false, the chart can leave `Reset` after its first eligible transition check.

## 11. Ready and Loading Behaviour

### 11.1 `Normal_Sequence`

The ready/waiting step applies:

| Qualifier | Variable | Effect |
|---|---|---|
| `R` | `FIO.ResetLight` | Reset stored reset-light action |
| `N` | `FIO.StartLight` | Start light active while the step is active |
| `R` | `FIO.StopLight` | Reset stored stop-light action |

The chart waits for `Control.RunningMode`.

### 11.2 `Loading`

When `RunningMode` is observed true, `Loading` becomes active and applies:

| Qualifier | Variable | Effect |
|---|---|---|
| `N` | `FIO.EntryConveyor` | Active while loading step is active |
| `N` | `FIO.LoadConveyor` | Active while loading step is active |
| `S` | `FIO.StopLight` | Stored on until an `R` association executes |

The exit guard is exactly:

```iecst
NOT FIO.AtLoad
```

If `AtLoad` is a conventional active-high product-present sensor and is false before a product arrives, this guard is already true and loading can end almost immediately. An active-low scene mapping could make the source correct. The polarity must be confirmed against the actual Factory I/O driver and an online trace.

`StopLight` remains stored during receiving, transporting, storing, and return home, then receives `R` on the next activation of `Normal_Sequence`. Its implemented behaviour is closer to a cycle-active/busy indication than an indication that the machine is stopped. Confirm the intended name and colour semantics.

## 12. Receiving Macro

The `Receiving` macro is part of the same SFC chart; it is not a separately called program.

### 12.1 `Receiving1`

- `S FIO.ForksLeft`
- `R FIO.Lift`
- wait for `FIO.AtLeft`

The left forks are stored active and the lift action is reset. With no maximum time, a missing `AtLeft` signal can hold the step indefinitely.

### 12.2 `Receiving2`

- minimum active time `T#2s`
- `S FIO.Lift`
- following guard is literal `TRUE`

The unconditional transition cannot pass until the 2 s minimum step time has elapsed.

### 12.3 `Receiving3`

- minimum active time `T#2.5s`
- `R FIO.ForksLeft`
- following guard is literal `TRUE`

This produces a further 2.5 s dwell before the macro exits to `Transporting`.

The macro therefore contains at least 4.5 s of configured dwell after `AtLeft`, excluding task quantisation and runtime jitter.

## 13. Destination and Transport Behaviour

`Transporting` has both an entry action and a step action.

### 13.1 Entry action

The LD `ADD` network performs once when the step activates:

```iecst
NextPosition := NextPosition + 1;
```

The first normal cycle changes the initial value from 0 to 1.

### 13.2 Active action

The LD `MOVE` network runs while the step is active:

```iecst
FIO.TargetPosition := NextPosition;
```

### 13.3 Completion transition

The transition object `End_MovingX` uses `F_TRIG_0`:

```iecst
End_MovingX := F_TRIG_0(CLK := FIO.MovingX).Q;
```

The chart label uses `END_MovingX`; IEC identifiers are case-insensitive, so it refers to the child transition object `End_MovingX`.

The transition is true for the falling edge of `MovingX`, interpreted as movement completion. If `MovingX` never becomes true after the command, there is no subsequent falling edge and the chart waits indefinitely.

### 13.4 Destination management gaps

No source logic:

- limits `NextPosition` to available locations;
- skips occupied locations;
- checks a full-store condition;
- maps a product class to a destination;
- resets or re-homes the index;
- detects rejection of `TargetPosition`;
- prevents reuse after a WORD wrap; or
- validates that a position target is within the Factory I/O model's accepted range.

This must be settled before presenting the project as a complete sorting or storage controller.

## 14. Storing Macro

### 14.1 `Storing1`

- `S FIO.ForksRight`
- wait for `FIO.AtRight`

The right forks remain stored active until the later reset association. No maximum time protects the wait for `AtRight`.

### 14.2 `Storing2`

- minimum active time `T#3s`
- `R FIO.Lift`
- following transition is `TRUE`

The lift action is reset and the step dwells for at least 3 s.

### 14.3 `Storing3`

- minimum active time `T#3s`
- `R FIO.ForksRight`
- following transition is `TRUE`

The right-fork action is reset and the step dwells for at least another 3 s before `GoToInitalPosition` activates.

The storing macro contains at least 6 s of configured dwell after `AtRight`.

## 15. Return-to-Initial-Position Behaviour

While `GoToInitalPosition` is active, its LD action continuously executes:

```iecst
FIO.TargetPosition := 55;
```

The transition object `EndMovingX1` uses a separate falling-edge instance:

```iecst
EndMovingX1 := F_TRIG_1(CLK := FIO.MovingX).Q;
```

On the falling edge, the chart jumps directly to `Normal_Sequence`.

The numeric meaning of target `55` is not defined in the export. It is treated in this documentation as an intended initial/home target because of the action name, but that interpretation requires confirmation against the scene.

`NextPosition` is not reset when the mechanism returns to target 55. The next product therefore advances to the next numeric destination rather than starting again at 1.

## 16. SFC Qualifiers and Output Ownership

The chart uses:

- `N`: active while the associated step is active;
- `S`: stored active until an overriding reset; and
- `R`: overriding reset.

Several variables are controlled both by qualifier associations and by direct ST assignments:

| Variable | IEC qualifier owners | Direct ST owners |
|---|---|---|
| `ForksLeft` | `S` in `Receiving1`; `R` in `Receiving3` | False in `Emergency_active` and `Reset_active` |
| `ForksRight` | `S` in `Storing1`; `R` in `Storing3` | False in `Emergency_active` and `Reset_active` |
| `Lift` | `R` in `Receiving1`; `S` in `Receiving2`; `R` in `Storing2` | False in `Emergency_active` and `Reset_active` |
| `ResetLight` | `S` in `Initial`; `R` in `Normal_Sequence` | False in both ST actions |
| `StartLight` | `N` in `Normal_Sequence` | False in both ST actions |
| `StopLight` | `R` in `Normal_Sequence`; `S` in `Loading` | False in both ST actions |

CODESYS processes step actions before IEC actions in the SFC scan order. Where both mechanisms write the same Boolean, qualifier processing occurs later in that chart evaluation, and stored qualifier state can persist beyond the step that originally set it.

The exact online result through reset and emergency transitions should be captured before relying on the direct false assignments as an override. A clearer architecture gives each output one owner, with separate sequence requests feeding a final permissive/safety arbitration layer.

## 17. Task-Order Behaviour

`MainTask` calls:

1. `SFC_PRG`;
2. `Control`.

### 17.1 Start example

1. An external client sets `FIO.Start = TRUE` before a task cycle.
2. `SFC_PRG` executes first and sees the previous value of `Control.RunningMode`.
3. `Control` then sets `RunningMode` if all reset inputs are healthy.
4. `SFC_PRG` can use the new value on the next task cycle.

### 17.2 Emergency/reset pulse example

1. `EmergencyStop` becomes false or `Reset` becomes true.
2. The current SFC scan executes before `Control` generates the rising-edge pulse.
3. `Control` writes `SFC_PRG.SFCinit` at the end of the task program list.
4. The SFC reads the value on the next cycle.
5. In the current export the flag is inactive, so no chart reset occurs.

Reordering the calls can reduce the ordinary command latency, but must be done deliberately and regression-tested. It does not turn application code into a safety system.

## 18. Interface Scope and Unused Signals

The following declared scene-to-PLC variables are not referenced by active logic:

- `AtEntry`;
- `AtExit`;
- `AtUnload`;
- `Auto`;
- `Manual`; and
- `MovingZ`.

The following actuator variables are only written false by the reset/emergency actions and are never commanded true:

- `ExitConveyor`; and
- `UnloadConveyor`.

There is no automatic/manual mode selection despite the declared mode signals. There is no exit/unload sequence despite the declared sensors and conveyor output.

Unused signals can be legitimate placeholders for later lessons, but they should be labelled as future scope rather than implied to be implemented.

## 19. Diagnostics and Fault Handling

The exported revision has:

- no maximum step times;
- all optional SFC flag `Use` settings false, including `Error`, `ErrorStep`, and `CurrentStep`;
- no fault POU or fault-state branch;
- no timeout transition around `AtLeft`, `AtRight`, or `MovingX`;
- no plausibility checks between position sensors;
- no target acceptance or range check;
- no debounce or filtering;
- no task watchdog; and
- empty source comments on SFC elements and FBD networks.

The minimum step times control dwell but do not detect a failure. Add maximum times and a documented fault/recovery route before equipment use.

## 20. Recommended Remediation Order

1. Preserve and build-test the supplied export as the baseline.
2. Confirm every Factory I/O polarity, especially `EmergencyStop`, `Stop`, `AtLoad`, and `MovingX`.
3. Activate and prove the chosen SFC reset flag, or replace it with a clearer sequence-reset design.
4. Add an effective stop/emergency response from every operating state and an independent safe-output layer.
5. Define controlled-restart behaviour and edge-qualify `Start`.
6. Define home position, reset completion, and X/Z motion permissives.
7. Add maximum step times, transition timeouts, fault diagnostics, and recovery rules.
8. Bound and validate `NextPosition` against actual storage capacity.
9. Give every actuator output one control owner.
10. Reduce external symbol write permissions.
11. Implement or remove unused mode and unload signals.
12. Capture dated build, watch, trace, scene, and communication evidence.

## 21. Safety Boundary

`EmergencyStop` is a standard Boolean in a non-safety CODESYS application. Neither its name nor the `Emergency` step provides a safety integrity level.

For physical equipment, hazardous-energy removal and motion stopping must be enforced independently by a validated safety system. The standard PLC may coordinate status and restart only after the safety system reports a verified safe condition.

## 22. CODESYS References

- [SFC flags](https://content.helpme-codesys.com/en/CODESYS%20SFC/_cds_sfc_sfc_flags.html)
- [Qualifiers for actions in SFC](https://content.helpme-codesys.com/en/CODESYS%20SFC/_cds_sfc_action_qualifier.html)
- [SFC element properties](https://content.helpme-codesys.com/en/CODESYS%20SFC/_cds_sfc_element_properties.html)
- [Processing order in SFC](https://content.helpme-codesys.com/en/CODESYS%20SFC/_cds_sfc_sequence_of_processing.html)
- [SFC alternative branches](https://content.helpme-codesys.com/en/CODESYS%20SFC/_cds_sfc_element_branch.html)

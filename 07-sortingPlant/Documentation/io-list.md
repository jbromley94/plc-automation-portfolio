# I/O, Variables, and External Interface

## 1. Scope

This document inventories the variables and external interface reconstructed from `sortingPlant.export`.

The application contains a qualified-only global variable list named `FIO`. It serves as a software interface between the PLC logic and an intended simulation or external client. The export contains no fixed IEC addresses, field-device modules, Factory I/O scene, or KEPServerEX project.

The word “I/O” in this document therefore describes logical direction, not proven physical wiring.

## 2. `FIO` Global Variable List

The declaration begins with:

```iecst
{attribute 'qualified_only'}
VAR_GLOBAL
```

PLC code must reference each value with the list qualifier, for example:

```iecst
FIO.Start
FIO.AtLoad
FIO.EntryConveyor
FIO.TargetPosition
```

The list contains 25 Boolean variables and one `WORD`, for 26 variables total.

## 3. Addressing and Mapping Status

| Concern | Static finding |
|---|---|
| Fixed `%I` addresses | None found |
| Fixed `%Q` addresses | None found |
| Fixed `%M` addresses | None found |
| Device I/O mapping | None included |
| Factory I/O scene/driver | Not supplied |
| KEPServerEX configuration | Not supplied |
| Symbol Configuration | Present as `Symbols` |
| OPC UA support | Enabled in `Symbols` |
| Direct I/O access | Disabled in `Symbols` |
| Published scope | All 26 `FIO` variables |
| Configured access | Read/write for all selected variables |

Do not invent fixed addresses from this list. Use symbolic communication or deliberately add and document a separate I/O map.

## 4. Scene-to-PLC Signals

The following 15 Boolean values are declared under the source comment `//Sensors and PBs`.

All are selected in `Symbols` with configured read/write access. For an external simulation, the expected operational direction is generally external client → PLC, so external write access can be appropriate. Unused values should still be removed from the published interface until implemented.

| Variable | Type | Intended role inferred from name | PLC use in supplied logic | Source polarity/behaviour | Recommended client access |
|---|---|---|---|---|---|
| `FIO.AtEntry` | `BOOL` | Entry presence/position sensor | **Unused** | No source semantics | Remove until used, or write from scene/read for diagnostics |
| `FIO.AtExit` | `BOOL` | Exit presence/position sensor | **Unused** | No source semantics | Remove until used, or write from scene/read for diagnostics |
| `FIO.AtLeft` | `BOOL` | Fork/mechanism at left position | Ends `Receiving1` | Active-high guard: transition when true | Scene write; PLC/client read |
| `FIO.AtLoad` | `BOOL` | Load station presence/position | Ends `Loading` through `NOT FIO.AtLoad` | Source treats false as transition-true | Scene write; PLC/client read; verify polarity |
| `FIO.AtMiddle` | `BOOL` | Middle/reference position sensor | In `Reset_active`, permits target 55 assignment | Target assigned only when true | Scene write; PLC/client read; confirm meaning |
| `FIO.AtRight` | `BOOL` | Fork/mechanism at right position | Ends `Storing1` | Active-high guard: transition when true | Scene write; PLC/client read |
| `FIO.AtUnload` | `BOOL` | Unload presence/position sensor | **Unused** | No source semantics | Remove until unload sequence exists |
| `FIO.Auto` | `BOOL` | Automatic-mode selector | **Unused** | No mode logic | Remove until mode arbitration exists |
| `FIO.EmergencyStop` | `BOOL` | Emergency-stop status | Initial branch, emergency acknowledgement, run-latch reset, reset-pulse source | Healthy appears to be true; false means emergency active | Scene/safety-status write only in simulation; never treat as safety over OPC |
| `FIO.Manual` | `BOOL` | Manual-mode selector | **Unused** | No mode logic | Remove until mode arbitration exists |
| `FIO.MovingX` | `BOOL` | Horizontal-axis moving status | Reset exit uses `NOT`; two completion transitions use falling edges | True while moving, falling edge interpreted as complete | Scene write; PLC/client read |
| `FIO.MovingZ` | `BOOL` | Vertical-axis moving status | **Unused** | No source semantics | Remove until Z-axis interlocks exist |
| `FIO.Reset` | `BOOL` | Reset pushbutton/command | Resets run latch; contributes to `SFCinit` pulse; acknowledges emergency | Active-high level | Momentary scene/client write; PLC/client read |
| `FIO.Start` | `BOOL` | Start pushbutton/command | Sets reset-dominant run latch | Active-high and level-sensitive | Momentary scene/client write; edge policy recommended |
| `FIO.Stop` | `BOOL` | Stop circuit/pushbutton status | Inverted into run-latch reset | Healthy appears true; false requests stop | Scene write; PLC/client read; verify mapping |

### 4.1 Important polarity tests

Four signals need explicit polarity evidence before sequence testing:

| Signal | Source assumption or guard | Required test |
|---|---|---|
| `EmergencyStop` | Healthy true; emergency false | Operate and release the scene device; record both values |
| `Stop` | Healthy true; stop request false | Operate and release the scene device; record both values |
| `AtLoad` | Loading exits while false | Determine whether false means “at load” in this scene or whether the guard is inverted incorrectly |
| `MovingX` | True during motion; falling edge completes | Confirm a clear false → true → false sequence for every accepted target |

## 5. PLC-to-Scene Commands and Indicators

The following 11 values are declared under `// Actuators`.

For a normal external architecture, the PLC owns these values and the external scene consumes them. They should therefore be readable by clients but not externally writable, except under a deliberate protected commissioning mode.

| Variable | Type | Initial value | PLC writers | Implemented behaviour | Recommended client access |
|---|---|---:|---|---|---|
| `FIO.EntryConveyor` | `BOOL` | `FALSE` | `N` action in `Loading`; false in emergency/reset ST | Active while loading step is active | Read-only |
| `FIO.ExitConveyor` | `BOOL` | `FALSE` | False in emergency/reset ST only | Never commanded true | Read-only or remove until implemented |
| `FIO.ForksLeft` | `BOOL` | `FALSE` | `S` in `Receiving1`; `R` in `Receiving3`; false in emergency/reset ST | Stored left-fork command during receive sequence | Read-only |
| `FIO.ForksRight` | `BOOL` | `FALSE` | `S` in `Storing1`; `R` in `Storing3`; false in emergency/reset ST | Stored right-fork command during store sequence | Read-only |
| `FIO.Lift` | `BOOL` | `FALSE` | `R` in `Receiving1`; `S` in `Receiving2`; `R` in `Storing2`; false in emergency/reset ST | Stored lift command through transport until store step 2 | Read-only |
| `FIO.LoadConveyor` | `BOOL` | `FALSE` | `N` action in `Loading`; false in emergency/reset ST | Active while loading step is active | Read-only |
| `FIO.ResetLight` | `BOOL` | `FALSE` | `S` in `Initial`; `R` in `Normal_Sequence`; false in emergency/reset ST | Qualifier/direct-write multiple ownership | Read-only |
| `FIO.StartLight` | `BOOL` | `FALSE` | `N` in `Normal_Sequence`; false in emergency/reset ST | Active while waiting in normal sequence | Read-only |
| `FIO.StopLight` | `BOOL` | `FALSE` | `R` in `Normal_Sequence`; `S` in `Loading`; false in emergency/reset ST | Stored from loading until next normal step; behaves like busy/cycle indication | Read-only |
| `FIO.TargetPosition` | `WORD` | `0` | Emergency target 0; conditional reset target 55; transport target `NextPosition`; return target 55 | Numeric horizontal destination command | Read-only; protect against external writes |
| `FIO.UnloadConveyor` | `BOOL` | Numeric `0` in source | False in emergency/reset ST only | Never commanded true | Read-only or remove until implemented |

`UnloadConveyor` is declared as `BOOL := 0` rather than `BOOL := FALSE`. Build verification should confirm whether the selected compiler accepts the implicit literal conversion without a warning.

## 6. Program-Local Variables

These values are not selected in the exported Symbol Configuration.

### 6.1 `Control`

| Variable | Type | Role | External symbol status |
|---|---|---|---|
| `RS_0` | `RS` | Reset-dominant run latch instance | Not selected |
| `RunningMode` | `BOOL` | Latched cycle-start permissive | Not selected |
| `R_TRIG_0` | `R_TRIG` | Rising edge for combined emergency/reset expression | Not selected |

### 6.2 `SFC_PRG`

| Variable | Type | Initial value | Role | External symbol status |
|---|---|---:|---|---|
| `NextPosition` | `WORD` | `0` | Increments once per transport step | Not selected |
| `F_TRIG_0` | `F_TRIG` | Standard default | Falling edge for transport completion | Not selected |
| `F_TRIG_1` | `F_TRIG` | Standard default | Falling edge for return-home completion | Not selected |
| `SFCinit` | `BOOL` input | Standard default | Written by `Control`; SFC `Init` use disabled | Not selected |

The SFC element attributes also store `Symbol = 0`, and no optional current-step diagnostic flag is enabled. External clients therefore have no deliberately published sequence state, `RunningMode`, or `NextPosition` diagnostic in the baseline.

## 7. Symbol Configuration Detail

The object is named `Symbols` and contains one selected group, `FIO`.

| Property | Exported value |
|---|---|
| Selected variable count | 26 |
| Selected variable types | 25 `BOOL`, 1 `WORD` |
| `VarAccess` | `3` for every variable |
| `MaxVarAccess` | `3` for every variable |
| Effective UI access | Read/write |
| `SupportOPCUA` | `True` |
| `EnableDirectIoAccess` | `False` |
| Export symbol comments | `False` |

The expected symbolic namespace is:

```text
Application.FIO.<variable>
```

Examples:

| PLC variable | Expected external path |
|---|---|
| `FIO.EmergencyStop` | `Application.FIO.EmergencyStop` |
| `FIO.Start` | `Application.FIO.Start` |
| `FIO.AtLoad` | `Application.FIO.AtLoad` |
| `FIO.EntryConveyor` | `Application.FIO.EntryConveyor` |
| `FIO.TargetPosition` | `Application.FIO.TargetPosition` |

Generated paths can vary with gateway, server, namespace, application name, and client driver. Treat these as expected paths and replace them with browse-proven paths after an error-free build.

## 8. Recommended Access Matrix

The baseline configuration is intentionally broad. A production-style revision should use this policy:

| Interface class | Examples | External read | External write |
|---|---|---:|---:|
| Simulated sensors/status | `AtLeft`, `AtRight`, `MovingX` | Yes | Yes from the authorised scene only |
| Operator commands | `Start`, `Reset` | Yes | Yes, momentary and authorised |
| Healthy-true stop status | `Stop`, `EmergencyStop` | Yes | Yes from the simulation driver only; never a real safety channel |
| PLC actuator commands | conveyors, forks, lift | Yes | No |
| PLC indicators | lights | Yes | No |
| Motion destination | `TargetPosition` | Yes | No |
| Internal diagnostics | `RunningMode`, current step, `NextPosition`, fault code | Yes if deliberately added | No |
| Unused placeholders | mode, entry/exit/unload, `MovingZ` | No until implemented | No |

Where the tooling cannot enforce per-symbol direction cleanly, separate external-input and PLC-output GVLs to make permissions and ownership obvious.

## 9. Multiple-Writer Review

The external symbol configuration permits writes to every PLC output. In addition, six qualifier-controlled Boolean outputs are also assigned directly by SFC ST actions.

This creates three potential writer classes:

1. SFC IEC action control (`N`, `S`, or `R`);
2. direct PLC assignments in `Emergency_active` and `Reset_active`; and
3. any external OPC or simulation client because the symbol is read/write.

An externally written value may be overwritten on the next PLC/SFC execution, may temporarily mask the commanded state, or may interfere with edge-based tests. Do not use unrestricted client writes to simulate an output override.

The corrected design should provide one PLC owner for each output, expose it read-only, and implement any maintenance override through an authenticated mode with explicit permissives and indication.

## 10. Proposed Diagnostic Additions

A deliberately designed read-only diagnostics group would improve commissioning without exposing control internals for writing. Suggested values include:

- `RunningMode`;
- enumerated main state;
- enumerated macro substate;
- `NextPosition`;
- requested and accepted target position;
- motion-started and motion-complete status;
- reset active/complete;
- stop and emergency status;
- timeout/fault active;
- fault code and first-out cause; and
- task/runtime heartbeat.

If native SFC flags are used, activate only the required flags and publish them read-only.

## 11. Factory I/O Mapping Workflow

1. Build the imported project without errors.
2. Start CODESYS Control Win and download the exact documented revision.
3. Browse the runtime symbols from the selected Factory I/O driver.
4. Record every generated path rather than typing paths from memory.
5. Map scene sensors and controls to the scene-to-PLC group.
6. Map PLC commands and indicators to the PLC-to-scene group.
7. Confirm Boolean polarity one signal at a time.
8. Confirm `TargetPosition` data type, byte order, accepted range, and target meaning.
9. Test quality/stale behaviour on disconnect and runtime restart.
10. Remove all forces and online writes after commissioning.
11. Save the scene and a mapping table in `FactoryIO/`.

## 12. KEPServerEX / OPC Client Guidance

If KEPServerEX or another OPC client is added:

- document the exact driver and endpoint;
- use encrypted and authenticated communication where supported;
- assign a least-privilege account;
- reject writes to PLC output and diagnostic tags;
- rate-limit or edge-process momentary commands;
- define quality and stale-data behaviour;
- prove reconnect behaviour after PLC and server restarts;
- record certificate handling without committing private keys; and
- bind the tag export to a named PLC source revision.

OPC communication must never be the sole path for an emergency-stop function.

## 13. Interface Acceptance Criteria

The interface can be marked verified when:

- all 26 baseline variables are accounted for;
- unused values are removed or clearly labelled as future scope;
- exact runtime paths are browse-proven;
- scene directions and polarities are recorded;
- output symbols reject unauthorised writes;
- `TargetPosition` range and position `55` are defined;
- disconnect, stale data, and restart tests pass;
- no force or manual write remains active; and
- the saved Factory I/O/OPC configuration matches the documented PLC hash or revision.

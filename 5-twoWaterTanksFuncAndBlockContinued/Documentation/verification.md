# Verification

## 1. Verification Status

This revision contains a static review of the native CODESYS V3.5 SP22 Patch 3 export.

The POU declarations, nine graphical networks, edge modifiers, function/function-block calls, timer presets, output routes, selected symbols, and task configuration were traced from the export. The application has not been executed in this documentation environment, so runtime results remain pending.

## 2. Reviewed Configuration

| Item | Static finding |
|---|---|
| Runtime target | CODESYS Control Win V3 x64, device version 3.5.22.30 |
| Program | `PLC_PRG` |
| Reusable POUs | `FB_Tank` and `FC_Timer` |
| Language | LD for all three POUs |
| Networks | 2 in `FB_Tank`, 1 in `FC_Timer`, 6 in `PLC_PRG` |
| Task | `MainTask`, cyclic, 20 ms, priority 1 |
| Watchdog | Disabled |
| Timer instances | Two `TP` instances per `FB_Tank`; two tank instances |
| Tank 1 presets | Fill 15 s; discharge default 10 s |
| Tank 2 presets | Fill 8 s; discharge 12 s |
| External symbols | Ten read/write `PLC_PRG` variables |
| OPC UA symbol support | Enabled |
| Physical I/O mapping | None found |
| Network titles/comments | Empty |

## 3. Test Preconditions

For a clean test session:

1. Import `CODESYS/twoWaterTanksFuncAndBlockContinued.export` into a compatible Patch 3 project.
2. Confirm that the target is CODESYS Control Win V3 x64.
3. Build with zero errors and zero unresolved-library placeholders.
4. Confirm that `PLC_PRG` is assigned to cyclic `MainTask` at 20 ms.
5. Log in, download, and place the application in Run.
6. Set `Fill = FALSE`, `Discharge = FALSE`, `Fill2 = FALSE`, and `Discharge2 = FALSE`.
7. Confirm all four valve outputs are false.
8. Use a trace or watch list containing all ten exported variables plus the four elapsed-time variables if available.
9. Record actual pulse duration with timestamps rather than relying only on the integer countdown.
10. Remove forces before each new test.

The Windows soft PLC is non-real-time. Evaluate pulse duration against the configured preset with an allowance appropriate to observed task jitter; do not claim deterministic hardware timing from this runtime.

## 4. Functional Test Matrix

| ID | Test | Expected result | Static review | Runtime |
|---|---|---|---|---|
| VT-01 | Import and build the export | Application builds without errors | Export parses and dependencies are listed | Pending |
| VT-02 | Inspect application hierarchy | `FB_Tank`, `FC_Timer`, `PLC_PRG`, `Symbols`, and `MainTask` are present | Consistent | Pending |
| VT-03 | Start with all four commands false | All valves remain false; countdowns initialise to zero | Consistent with declarations | Pending |
| VT-04 | Change `Fill` false → true while Tank 1 is idle | `Fill_Valve` starts; `Discharge_Valve` remains false | Rising-edge flag and interlock present | Pending |
| VT-05 | Measure Tank 1 fill pulse | Fill output remains active for approximately 15 s | Explicit `T#15S` | Pending |
| VT-06 | Hold `Fill` true beyond completion | Fill does not retrigger | Edge-driven request | Pending |
| VT-07 | Return `Fill` false, then true | A second 15-second fill pulse starts | Edge can rearm | Pending |
| VT-08 | Change `Discharge` false → true while idle | No discharge pulse starts | Falling-edge convention | Pending |
| VT-09 | Change `Discharge` true → false while idle | `Discharge_Valve` starts; `Fill_Valve` remains false | Falling-edge flag and interlock present | Pending |
| VT-10 | Measure Tank 1 discharge pulse | Discharge remains active for approximately 10 s | Call relies on default `T#10s` | Pending |
| VT-11 | Trigger `Fill2` rising while Tank 2 is idle | `Fill_Valve2` starts for approximately 8 s | Explicit `T#8S` | Pending |
| VT-12 | Trigger `Discharge2` falling while Tank 2 is idle | `Discharge_Valve2` starts for approximately 12 s | Explicit `T#12S` | Pending |
| VT-13 | Fill both tanks concurrently | Both fill outputs can be active; countdowns remain independent | Separate instances | Pending |
| VT-14 | Fill one tank and discharge the other | Opposite phases can run concurrently across instances | No cross-instance interlock | Pending |
| VT-15 | Request discharge while the same tank is filling | Discharge does not start and does not start automatically after fill | Normally closed fill interlock; no queue | Pending |
| VT-16 | Request fill while the same tank is discharging | Fill does not start and does not start automatically after discharge | Normally closed discharge interlock; no queue | Pending |
| VT-17 | Generate both request edges in one idle scan | Fill starts and discharge remains off | Fill network executes first | Pending |
| VT-18 | Generate a discharge edge on the scan fill completes | Confirm discharge can start in that scan | Predicted from network order | Pending |
| VT-19 | Generate a fill edge on the scan discharge completes | Confirm fill edge is rejected | Predicted from network order | Pending |
| VT-20 | Retrigger the same phase while its `TP` is active | Existing pulse is not extended and no later pulse is queued | Standard `TP` behaviour expected | Pending |
| VT-21 | Observe a fill countdown | Value starts near the preset and decreases in truncated whole seconds | `FC_Timer` formula traced | Pending |
| VT-22 | Observe a discharge countdown | The same tank display changes source to discharge timing | Two gated writes traced | Pending |
| VT-23 | Observe countdown after valve turns off | Record whether final value is zero and confirm it then holds | No explicit idle assignment | Pending |
| VT-24 | Browse Symbol Configuration | Exactly ten documented names are available | Configured | Pending |
| VT-25 | Check client write permissions | Commands and PLC-generated values all appear writable | Access value `3` on all symbols | Pending |
| VT-26 | Attempt a normal client write to a valve/countdown, then remove it | Observe PLC overwrite behaviour; no force remains | PLC writes these values during enabled logic | Pending |
| VT-27 | Verify task monitoring | `PLC_PRG` executes cyclically at the configured 20 ms interval | Configured | Pending |
| VT-28 | Restart with `Fill` already true | Confirm edge initialisation and whether an unintended fill begins | Requires runtime confirmation | Pending |
| VT-29 | Restart with `Discharge` already false | No false falling-edge pulse should be generated | Expected from edge memory initialisation | Pending |
| VT-30 | Inspect graphical edge symbols | Fill contacts are rising-edge and discharge contacts are falling-edge | Modifier flags are present | Pending |
| VT-31 | Search device mapping | No fixed IEC addresses or `FIO` object are present | None found in export | Pending |

Tests VT-18 and VT-19 require deliberate signal timing and a trace resolution capable of identifying the exact completion scan. They are useful for documenting the existing implementation but should not become a required operator technique.

## 5. Static Findings

### VF-01: Discharge is release-triggered

**Severity:** Functional interface concern.

Fill uses a rising-edge contact, while discharge uses a falling-edge contact. If both commands are connected to ordinary active-high momentary controls, fill starts on press and discharge starts on release.

Confirm that this asymmetry is intentional. If not, change the discharge contacts to rising-edge behaviour and repeat all discharge and arbitration tests.

### VF-02: Busy-time requests are discarded

**Severity:** Sequence and operator-expectation concern.

The opposite valve output blocks a request edge while a tank is busy. Same-phase retriggers are also not queued by the active `TP`.

The function block provides no acknowledgement or rejected-command status. An HMI can therefore show that a command was operated without explaining why no later action occurs.

### VF-03: Arbitration depends on scan order

**Severity:** Maintainability concern.

Fill logic executes before discharge logic. Fill wins simultaneous idle requests, and the two pulse-completion boundary cases are asymmetric.

If one phase should have priority, encode that requirement explicitly. If neither should have priority, detect the conflict and reject both with a diagnostic.

### VF-04: Tank 1 discharge preset is implicit

**Severity:** Configuration traceability concern.

`FB_Tank_1.Discharge_PT` is unconnected, so the instance uses the function-block declaration default of `T#10s`. The associated countdown call separately uses `T#10S`.

The values currently agree, but a future change to either location could create a mismatch. Wire the call input explicitly or centralise the preset in one named constant.

### VF-05: One retained countdown represents two phases

**Severity:** Interface clarity concern.

`Timer` is written by Tank 1's fill and discharge calls; `Timer2` is written by the corresponding Tank 2 calls. Neither value has an explicit idle write.

An external client needs valve status to determine the active phase and whether a displayed value is current. Add a phase/status variable, a valid bit, or separate countdown values.

### VF-06: `FC_Timer` has a short numeric range

**Severity:** Reuse limitation.

The function converts remaining milliseconds to signed 16-bit `INT` before dividing by 1000. Its safe positive range is therefore approximately 0 to 32.767 seconds.

The current 15-second maximum is safe. Reuse with longer presets can overflow. Use a wider intermediate such as `DINT`, then validate and clamp the return range.

### VF-07: External write access is broader than the interface requires

**Severity:** Integration and control-authority concern.

All ten symbols are read/write, including valve outputs and countdowns. A normal external client should generally write only the four command variables.

Restrict status values to read-only where the CODESYS/client design permits, and distinguish intentional commissioning forces from ordinary communications.

### VF-08: No closed-loop process protection exists

**Severity:** High if adapted to real equipment; expected limitation for this exercise.

The application has no tank level, valve feedback, flow confirmation, timeout fault, stop, emergency, or reset logic. A timed output cannot prove that a tank filled, discharged, or remained within a safe level.

Do not connect this application directly to physical valves.

### VF-09: The implementation is visually undocumented

**Severity:** Maintainability concern.

All exported network titles and comments are empty. The POU and variable names provide some intent, but edge conventions, rejected-command behaviour, and the timer formula are not explained inside CODESYS.

Add network titles and concise comments, then export a new source-of-truth file.

### VF-10: No direct-open project file is included

**Severity:** Packaging limitation.

Only the portable `.export` was supplied. This is sufficient to reconstruct the application objects, but reviewers must create a compatible project and import it before building.

After runtime verification, optionally add a clean `.project` created from the same tested revision while retaining the portable export.

## 6. Evidence to Capture

Add the following portfolio evidence to `Media/`:

- Patch 3 build result with zero errors;
- application tree containing both reusable POUs and the task;
- `FB_Tank` declaration and both edge-modified networks;
- `FC_Timer` calculation network;
- Tank 1 fill trace showing request edge, 15-second output, elapsed time, and countdown;
- Tank 1 discharge trace showing true-to-false request and 10-second output;
- Tank 2 fill and discharge traces at 8 and 12 seconds;
- concurrent operation of both function-block instances;
- rejected busy-time command with proof that it is not queued;
- simultaneous request trace demonstrating fill priority;
- countdown idle-retention result;
- Symbol Configuration with all ten names and access rights; and
- `MainTask` interval and live cycle-time evidence.

Name evidence with its test ID, for example `VT-15-discharge-rejected-during-fill.png`.

## 7. Completion Criteria

The project can be marked runtime verified when:

- the export imports and builds without errors in the documented version;
- every applicable test has a dated result and evidence reference;
- all four configured pulse durations have been measured;
- the falling-edge discharge convention is either accepted as a requirement or corrected and retested;
- busy-time and simultaneous-command behaviour is explicitly accepted or improved;
- the Tank 1 default discharge preset is confirmed and made traceable;
- countdown idle behaviour and numeric range are documented in the client interface;
- external write permissions follow the intended control-authority model;
- edge modifiers are visually confirmed;
- no project claim implies level control or physical safety that the source does not implement; and
- the saved project, export, Symbol Configuration, and documentation all describe the same tested revision.

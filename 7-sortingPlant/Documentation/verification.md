# Verification and Correction Plan

## 1. Verification Status

This revision contains a static review of the supplied CODESYS V3.5 SP22 Patch 3 export.

The XML was parsed and inspected for project hierarchy, target, libraries, declarations, FBD/LD operands and inversions, SFC steps, macros, branches, transitions, action qualifiers, child actions, task configuration, symbol selection, access rights, and fixed addresses.

The project has not been compiled or executed in this documentation environment. Build results, generated-code behaviour, Factory I/O polarity, motion response, timing, and OPC communication remain pending until recorded in CODESYS.

## 2. Supplied Source Integrity

| File | Size | SHA-256 |
|---|---:|---|
| `CODESYS/sortingPlant.export` | 502,655 bytes | `dd148ee32a82b5107f7103998c0683652487d7729fa663fe9131f27aaeb89b57` |

The repository filename is simplified from the uploaded `sortingPlant(1).export`; the contents are unchanged.

Recalculate after cloning or downloading:

```bash
sha256sum CODESYS/sortingPlant.export
```

A deliberately corrected export should have a new filename or revision tag and a separately recorded hash. Do not silently replace the baseline evidence.

## 3. Reviewed Baseline Configuration

| Item | Static finding |
|---|---|
| Profile | CODESYS V3.5 SP22 Patch 3 |
| Target | CODESYS Control Win V3 x64, version 3.5.22.30 |
| Main programs | `SFC_PRG`, `Control` |
| Task | `MainTask`, cyclic 20 ms, priority 1 |
| Program order | `SFC_PRG` then `Control` |
| Watchdog | Disabled |
| Global interface | Qualified-only `FIO`, 26 variables |
| Symbol object | `Symbols`; all 26 variables read/write |
| OPC UA | Enabled |
| Direct I/O access | Disabled |
| Fixed IEC addresses | None found |
| Initial step | `Initial` |
| Initial branch | Alternative, emergency guard first, literal-true fallback second |
| Main process states | `Reset`, `Normal_Sequence`, `Loading`, `Receiving`, `Transporting`, `Storing`, `GoToInitalPosition` |
| Receiving minimum times | 2 s and 2.5 s |
| Storing minimum times | 3 s and 3 s |
| Maximum step times | None |
| Motion completion | Two separate `F_TRIG` instances on `FIO.MovingX` |
| SFC `Init` flag | `Use = FALSE`, `Declare = TRUE` |
| Explicit `SFCinit` declaration | `VAR_INPUT SFCinit : BOOL` |
| Other optional SFC flags | All `Use = FALSE` |
| Calculate active transitions only | `FALSE` |
| Native `.project` | Not supplied |
| Factory I/O / KEPServerEX | Not supplied |

## 4. Verification Phases

| Phase | Purpose |
|---|---|
| A — baseline recovery | Prove file integrity, clean import, object inventory, and build result |
| B — baseline controls | Prove run-latch truth table, task-order delay, and initial/reset behaviour |
| C — baseline process cycle | Trace loading, receiving, transporting, storing, and return home |
| D — baseline abnormal cases | Demonstrate the exact current stop, emergency, timeout, edge, and retained-state behaviour |
| E — corrected sequence control | Activate the SFC flag correctly, add effective stop/fault handling, and regression-test |
| F — destination and motion hardening | Add range, capacity, feedback, and timeout policies |
| G — integration and security | Verify Factory I/O/OPC paths, polarity, least privilege, reconnect, and stale data |

Keep evidence from the baseline phases. It shows what the supplied source actually did and makes the later engineering corrections credible.

## 5. General Preconditions

1. Use CODESYS Development System V3.5 SP22 Patch 3 where possible.
2. Install or select CODESYS Control Win V3 x64 version 3.5.22.30, or record the exact compatible substitute.
3. Preserve `sortingPlant.export` as a read-only baseline copy.
4. Create a compatible standard project and import the export.
5. Resolve device and library prompts without silently changing source logic.
6. Build and record every error and warning.
7. Confirm the task interval, priority, watchdog, and POU order before running.
8. Use simulation or a disconnected Factory I/O scene first; do not connect physical machinery.
9. Remove all external writes and forces before each test unless explicitly required.
10. Use a watch list and trace sampled at task resolution or faster.
11. Record CODESYS IDE/runtime versions, operating system, test date, source hash, and scene revision.
12. Reset the runtime/test state between cases where stored qualifiers or edge-detector memory could affect the result.

## 6. Recommended Watch and Trace List

### 6.1 Commands and permissives

- `FIO.Start`
- `FIO.Stop`
- `FIO.EmergencyStop`
- `FIO.Reset`
- `Control.RunningMode`
- `SFC_PRG.SFCinit`

### 6.2 Position and movement

- `FIO.AtLoad`
- `FIO.AtLeft`
- `FIO.AtMiddle`
- `FIO.AtRight`
- `FIO.MovingX`
- `FIO.MovingZ`
- `SFC_PRG.NextPosition`
- `FIO.TargetPosition`
- `End_MovingX`
- `EndMovingX1`

### 6.3 Outputs

- both conveyor commands used by loading;
- both fork commands;
- lift;
- all three lights;
- exit/unload conveyors; and
- current active SFC step/substep from the online editor.

If transition-object variables cannot be added directly, monitor the two `F_TRIG` instances or capture the transition highlighting at the same time resolution.

## 7. Baseline Test Matrix

### 7.1 Source, import, and build

| ID | Test | Expected/result to record | Static review | Runtime |
|---|---|---|---|---|
| VT-01 | Recalculate export hash and size | Match section 2 | Recorded | Pending |
| VT-02 | Import into a clean compatible project | Import completes without source-recovery loss | XML parses | Pending |
| VT-03 | Compare application tree | `FIO`, Library Manager, `Control`, `SFC_PRG`, `Symbols`, Task Configuration present | Confirmed | Pending |
| VT-04 | Inspect SFC child objects | Five actions and two transition objects match documentation | Confirmed | Pending |
| VT-05 | Inspect target | Control Win V3 x64, 3.5.22.30 | Confirmed | Pending |
| VT-06 | Resolve libraries | Resolutions recorded; no unintended source changes | Inventory complete | Pending |
| VT-07 | Build baseline | Zero errors; all warnings copied to evidence | Cannot prove statically | Pending |
| VT-08 | Inspect task | Cyclic 20 ms, priority 1, watchdog disabled | Confirmed | Pending |
| VT-09 | Inspect program order | `SFC_PRG` first, `Control` second | Confirmed | Pending |
| VT-10 | Inspect FIO declaration | 26 variables, qualified-only, types/initial values match | Confirmed | Pending |
| VT-11 | Search addresses and mappings | No fixed `%I`, `%Q`, `%M`, or mapped device I/O | Confirmed | Pending |
| VT-12 | Inspect symbols | All 26 `FIO` values selected read/write; OPC UA on; direct I/O off | Confirmed | Pending |
| VT-13 | Inspect SFC settings | `UseDefaults = FALSE`; all flags `Use = FALSE`; Init `Declare = TRUE` | Confirmed | Pending |
| VT-14 | Inspect SFC topology | Main chart, alternative branch, both macros, jump, times, actions match | Reconstructed | Pending |

### 7.2 Run latch and task order

| ID | Test | Expected/result to record | Static review | Runtime |
|---|---|---|---|---|
| VT-15 | Healthy idle: `Stop=TRUE`, `EmergencyStop=TRUE`, `Reset=FALSE`, `Start=FALSE` | `RunningMode` remains false | Consistent | Pending |
| VT-16 | Pulse `Start` with all reset inputs healthy | `RunningMode` sets true | Configured | Pending |
| VT-17 | Assert `Start` and `Reset` together | Reset-dominant result: `RunningMode=FALSE` | Configured `RS` | Pending |
| VT-18 | Change `Stop` true → false | `RunningMode` resets | Inverted stop input | Pending |
| VT-19 | Change `EmergencyStop` true → false | `RunningMode` resets | Inverted emergency input | Pending |
| VT-20 | Assert `Reset` | `RunningMode` resets | Configured | Pending |
| VT-21 | Hold `Start=TRUE`, clear a stop/reset condition | Determine whether latch re-sets without a new start edge | Level-sensitive concern | Pending |
| VT-22 | Trace start across task calls | `Control` updates after SFC; loading transition sees new latch next task | Program-order inference | Pending |
| VT-23 | Measure task interval and jitter | Nominal 20 ms; actual distribution recorded | Configured only | Pending |

### 7.3 Initial, emergency, and reset states

| ID | Test | Expected/result to record | Static review | Runtime |
|---|---|---|---|---|
| VT-24 | Fresh start with `EmergencyStop=FALSE` | Initial left branch activates `Emergency` | Configured | Pending |
| VT-25 | Observe `Emergency_active` | Listed outputs false and target 0, subject to qualifier interaction | Source assignments confirmed | Pending |
| VT-26 | Keep emergency false and pulse reset | Emergency step does not exit because healthy signal is false | Guard confirmed | Pending |
| VT-27 | Restore emergency true without reset | Emergency step remains active | Guard confirmed | Pending |
| VT-28 | Set emergency true and reset true | Chart enters `Reset` | Guard confirmed | Pending |
| VT-29 | Fresh start with emergency healthy | Literal fallback selects `Reset` | Branch order confirmed | Pending |
| VT-30 | Reset with `AtMiddle=TRUE` | `TargetPosition` becomes 55 while action active | Source confirmed | Pending |
| VT-31 | Reset with `AtMiddle=FALSE` | Record retained `TargetPosition`; action makes no assignment | Retention found | Pending |
| VT-32 | Enter reset with `MovingX=FALSE` | Chart can advance at first eligible transition check | Guard confirmed | Pending |
| VT-33 | Enter reset with `MovingX=TRUE` | Remain until false | Guard confirmed | Pending |
| VT-34 | Prove reset completion position | Record actual position when chart reaches normal | No position proof in source | Pending |

### 7.4 Ready, loading, and receiving

| ID | Test | Expected/result to record | Static review | Runtime |
|---|---|---|---|---|
| VT-35 | Observe `Normal_Sequence` | Reset/stop lights reset; start light active | Qualifiers confirmed | Pending |
| VT-36 | Start a cycle | Chart enters `Loading` after latch/task-order timing | Configured | Pending |
| VT-37 | Observe loading outputs | Entry and load conveyors active; stop light stored | Qualifiers confirmed | Pending |
| VT-38 | Enter loading with `AtLoad=FALSE` | Record whether transition opens immediately | Guard is `NOT AtLoad` | Pending |
| VT-39 | Enter loading with `AtLoad=TRUE` | Guard false; chart should wait while true | Guard confirmed | Pending |
| VT-40 | Determine actual Factory I/O load polarity | Document truth table and correct source/mapping if required | Unknown statically | Pending |
| VT-41 | Enter `Receiving1` with `AtLeft=FALSE` | Wait indefinitely; forks left stored, lift reset | Configured | Pending |
| VT-42 | Set `AtLeft=TRUE` | Advance to `Receiving2` | Configured | Pending |
| VT-43 | Measure `Receiving2` dwell | At least 2 s plus task quantisation | Configured | Pending |
| VT-44 | Measure `Receiving3` dwell | At least 2.5 s plus task quantisation | Configured | Pending |
| VT-45 | Trace left forks and lift | Forks set then reset; lift set and remains stored after receive | Qualifier sequence confirmed | Pending |
| VT-46 | Hold `AtLeft=TRUE` before receiving | Record immediate level-sensitive transition behaviour | No edge qualification | Pending |

### 7.5 Transport, storing, and return home

| ID | Test | Expected/result to record | Static review | Runtime |
|---|---|---|---|---|
| VT-47 | First activation of `Transporting` | `NextPosition` changes 0 → 1 once | Entry action confirmed | Pending |
| VT-48 | Hold `Transporting` for multiple scans | Index does not continue incrementing while step remains active | Entry action semantics | Pending |
| VT-49 | Observe transport target | `TargetPosition = NextPosition` | MOVE confirmed | Pending |
| VT-50 | Target accepted and `MovingX` rises then falls | `End_MovingX` pulses and enters storing | F_TRIG confirmed | Pending |
| VT-51 | Motion never starts (`MovingX` stays false) | No falling edge; transport waits indefinitely | Edge limitation | Pending |
| VT-52 | Motion starts but never ends | Transport waits indefinitely | No timeout | Pending |
| VT-53 | Enter `Storing1` with `AtRight=FALSE` | Forks right stored; step waits | Configured | Pending |
| VT-54 | Set `AtRight=TRUE` | Advance to `Storing2` | Configured | Pending |
| VT-55 | Measure `Storing2` dwell | At least 3 s; lift reset | Configured | Pending |
| VT-56 | Measure `Storing3` dwell | At least 3 s; forks right reset | Configured | Pending |
| VT-57 | Observe return-home target | Target becomes 55 | MOVE confirmed | Pending |
| VT-58 | Complete home motion false → true → false | `EndMovingX1` pulses; jump to `Normal_Sequence` | Configured | Pending |
| VT-59 | Return motion never starts | Chart waits indefinitely because no falling edge | Edge limitation | Pending |
| VT-60 | Start second cycle | `NextPosition` advances from 1 to 2; it was not reset at home | Source confirmed | Pending |
| VT-61 | Repeat across all real storage locations | Record accepted range and first invalid/full condition | No bound in source | Pending |
| VT-62 | Trace total minimum dwell | At least 10.5 s plus process/task time | Sum confirmed | Pending |

### 7.6 Baseline stop, emergency, reset, and ownership tests

| ID | Test | Expected/result to record | Static review | Runtime |
|---|---|---|---|---|
| VT-63 | Assert `Stop=FALSE` during receiving | `RunningMode` resets; active sequence has no stop transition and may continue | High-priority finding | Pending |
| VT-64 | Assert emergency false during transport | `RunningMode` resets and `SFCinit` pulse is written later; chart is not initialised as exported | High-priority finding | Pending |
| VT-65 | Assert reset during storing | Run latch resets; chart reset effect absent because Init Use is false | High-priority finding | Pending |
| VT-66 | Watch `SFC_PRG.SFCinit` | One-scan pulse can occur, but active step does not reset | Flag configuration confirms | Pending |
| VT-67 | Hold emergency false, then press reset | Combined OR stays true, so no distinct second `R_TRIG` edge | Edge-merging concern | Pending |
| VT-68 | Trace lights through initial/emergency/reset/normal | Determine qualifier versus direct-write result | Multiple writers found | Pending |
| VT-69 | Interrupt after `S ForksLeft` or `S Lift` | Record whether stored action remains active without reaching its `R` owner | Persistence concern | Pending |
| VT-70 | Write an actuator externally while PLC runs | Baseline permits write; record overwrite/race, then remove authority | Security/control concern | Pending |

## 8. Corrected SFC Initialisation Procedure

Create a separately named engineering revision.

1. Open `SFC_PRG` properties and select SFC Settings.
2. Keep **Use defaults** disabled if POU-specific settings are intended.
3. Set the `Init` flag **Use** option to `TRUE`.
4. Because `SFCinit` is manually declared as `VAR_INPUT`, set its automatic **Declare** option to `FALSE`.
5. Build and confirm there is one externally writable `SFC_PRG.SFCinit`, with no implicit variable shadowing or duplicate declaration.
6. Confirm `Control` still writes the intended instance variable.
7. Trace a reset pulse while each main and macro state is active.
8. Record the first cycle in which the chart returns to `Initial`.
9. Record whether initial-step actions execute during and after the pulse.
10. Trace all stored IEC actions and direct outputs during initialisation.
11. Verify that `NextPosition` follows the deliberately chosen retain/reset policy.
12. Repeat the complete normal cycle afterward.

Do not stop at “the highlighted step moved.” Prove the actuator image, qualifier memory, position index, and controlled-restart behaviour.

## 9. Stop and Emergency Correction Procedure

Activating `SFCInit` improves the intended chart recovery but is not sufficient for safe machine control.

1. Define requirements for controlled stop, emergency status, reset, recovery, and restart.
2. Implement safe-output enforcement outside the process sequence.
3. Add a high-priority transition or supervisory state that is effective from every process step.
4. Decide whether an ordinary stop finishes the current motion, stops after the current macro, or aborts immediately to a controlled-stop state.
5. Require a new start edge after stop/emergency/reset recovery.
6. Ensure a held `Start` cannot cause automatic restart.
7. Add motion feedback and timeout handling to the stop path.
8. Test stop and emergency at every state and at each dwell boundary.
9. Test simultaneous start/stop/reset/emergency combinations.
10. Validate that standard PLC logic only coordinates with, and never replaces, the independent safety system.

## 10. `AtLoad` Polarity Correction

1. Keep the baseline transition `NOT FIO.AtLoad` for the first trace.
2. Observe the scene value with no product, while arriving, and when fully at load.
3. Record whether the sensor is normally false/true and whether the driver inverts it.
4. If product-at-load is true, change the guard to `FIO.AtLoad` in a new revision.
5. If the scene intentionally supplies active-low, retain the guard and add an explicit polarity comment.
6. Repeat loading with sensor initially clear, product arrival, stuck on, and stuck off.
7. Add a loading timeout and fault route.

## 11. Destination and Motion Hardening Procedure

1. Document the valid Factory I/O target range and the meaning of target 55.
2. Define storage capacity and occupied/free location ownership.
3. Replace unbounded incrementing with a validated destination-selection function.
4. Define behaviour when all destinations are full.
5. Define whether reset/emergency preserves or clears `NextPosition`.
6. Add target-command acceptance feedback where available.
7. Require motion start within a bounded time after a target change.
8. Require motion complete within a bounded travel time.
9. Add sensor plausibility and X/Z interlocks.
10. Test targets 0, 1, last valid, first invalid, 55, and type/overflow boundaries.
11. Test feedback stuck false, stuck true, chattering, and delayed beyond timeout.

## 12. Symbol and Integration Verification

| ID | Test | Acceptance requirement |
|---|---|---|
| IT-01 | Build before browsing | Error-free build and downloaded exact revision |
| IT-02 | Browse `Application.FIO` | All intended paths recorded from client browse |
| IT-03 | Compare 26 baseline variables | Type and direction match `io-list.md` |
| IT-04 | Prove scene input writes | Only authorised input/command group accepts writes |
| IT-05 | Prove output protection | Conveyors, forks, lift, lights, and target reject client writes |
| IT-06 | Check unused symbols | Removed or explicitly marked future scope |
| IT-07 | Check update rate | Suitable relative to 20 ms PLC task; no missed pulses |
| IT-08 | Disconnect client | PLC enters documented stale/disconnected behaviour |
| IT-09 | Restart PLC runtime | Client reconnects without unintended command replay |
| IT-10 | Restart client/server | PLC remains controlled; momentary commands do not stick |
| IT-11 | Inspect OPC security | Authentication, encryption, certificate trust, and least privilege recorded |
| IT-12 | Close out writes/forces | No force, online write, or maintenance override remains |

## 13. Static Findings

### VF-01: SFC initialisation flag is inactive

**Severity:** Definite functional configuration defect relative to the apparent intent.

`Control` writes `SFC_PRG.SFCinit`, but `Init — Use = FALSE`. CODESYS documents that declared-but-not-enabled flags have no processing function.

### VF-02: Automatic and manual declaration settings conflict

**Severity:** Configuration/shadowing concern.

The POU manually declares `SFCinit` as `VAR_INPUT`, while `Init — Declare = TRUE`. CODESYS guidance states that automatic Declare should be deselected for a manually declared flag, otherwise the automatic flag can overwrite the manual declaration.

### VF-03: Emergency step is not reachable from running states

**Severity:** High functional concern; critical if misrepresented as safety behaviour.

`Emergency` is connected only to the initial alternative branch. No active process state has an emergency transition.

### VF-04: `RunningMode` is only a cycle-start gate

**Severity:** High stop-behaviour concern.

Once loading begins, the chart does not check `RunningMode` until it returns to `Normal_Sequence`.

### VF-05: Held start can permit automatic restart

**Severity:** Controlled-restart concern.

`Start` is level-sensitive. When reset inputs clear, a still-true set input can set the run latch without a fresh edge.

### VF-06: Program order adds a task-cycle dependency

**Severity:** Timing/maintainability concern.

`SFC_PRG` runs before `Control`, so it consumes latch and pulse values from the prior control call.

### VF-07: Loading guard polarity is unproven

**Severity:** Functional sequence concern.

The exact guard is `NOT FIO.AtLoad`. With conventional active-high sensing it can leave loading before arrival.

### VF-08: Reset target can remain stale

**Severity:** Motion command concern.

`Reset_active` writes target 55 only when `AtMiddle` is true. Otherwise it retains the previous command.

### VF-09: Reset completion does not prove position

**Severity:** High for motion equipment.

The reset exit condition is only `NOT MovingX`; stopped is not equivalent to homed.

### VF-10: Destination index is unbounded

**Severity:** Functional/data-range concern.

`NextPosition` increments once per cycle with no capacity, range, occupied-location, wrap, or reset policy.

### VF-11: Movement completion has no start acknowledgement or timeout

**Severity:** High availability/motion concern.

Both transitions require a falling edge of `MovingX`. If motion never starts or never finishes, the sequence stalls indefinitely.

### VF-12: Stored actions can remain active during stalls

**Severity:** Output-state concern.

Fork and lift `S` actions depend on later normal-path `R` actions. Missing sensors/edges and ineffective chart reset can leave the stored command active.

### VF-13: Outputs have multiple writers

**Severity:** Determinism and maintainability concern.

Six Boolean outputs are written by SFC qualifiers and direct ST actions. Every output is also externally writable through `Symbols`.

### VF-14: Symbol permissions are broader than required

**Severity:** Control-authority and OT-security concern.

All sensor values and all PLC actuator/target values are read/write. PLC-owned outputs should be read-only to clients.

### VF-15: Declared scope exceeds implemented scope

**Severity:** Documentation/integration concern.

Six scene inputs are unused. Exit and unload conveyors are never commanded true. Auto/manual and unload logic are absent.

### VF-16: Diagnostics and timeout handling are absent

**Severity:** High for equipment; expected limitation for a learning baseline.

Maximum step times, SFC error/current-step flags, fault states, and task watchdog are all absent or disabled.

### VF-17: Stateful transition code generation needs an online trace

**Severity:** Verification concern.

`Calculate active transitions only = FALSE`, and transition objects contain `F_TRIG` state. Prove when each instance executes and retains history.

### VF-18: Integration evidence is absent

**Severity:** Project-completeness limitation.

No Factory I/O scene, KEPServerEX configuration, fixed addresses, native project, build record, or runtime trace was supplied.

### VF-19: Source comments are empty

**Severity:** Maintainability concern.

The SFC element comments and FBD network titles/comments do not explain polarity, target 55, dwell choices, destination policy, or reset intent.

### VF-20: `Reset_active` may be missing an ST terminator

**Severity:** Baseline build concern.

The exported action ends with `END_IF` and no semicolon. Record the unmodified import/build result. If the selected compiler reports a syntax error, change it to `END_IF;` in a separately identified corrected revision.

## 14. Evidence to Capture

Store evidence in `Media/` using the test ID in the filename:

- clean import and application tree;
- build result and warning list;
- target and library resolution;
- task configuration and program order;
- SFC settings showing Init Use/Declare before and after correction;
- full main chart plus receiving/storing macros;
- run-latch truth-table trace;
- start-to-loading task delay;
- initial emergency and reset routes;
- `AtLoad` polarity trace;
- all four minimum-time measurements;
- both X-motion handshakes and failure cases;
- destination progression and capacity boundary;
- stop/emergency/reset during every active state;
- stored-action/multiple-writer output trace;
- corrected SFC initialisation trace;
- corrected global stop/fault handling;
- browse-proven symbol paths and write rejection;
- disconnect/reconnect and stale-data behaviour; and
- final force/write/override closeout.

Example filename: `VT-64-emergency-during-transport-baseline.png`.

## 15. Completion Criteria

The project can be marked fully documented and runtime-verified when:

- the baseline export passes integrity checking and clean import;
- the imported project builds without errors and warnings are resolved or justified;
- target, libraries, task, program order, GVL, symbols, chart, macros, times, actions, and transitions match the static documentation;
- the current inert `SFCinit` behaviour is demonstrated and preserved as baseline evidence;
- the corrected SFC flag uses one externally writable declaration and reliably initialises the chart;
- stop and emergency response is effective from every operating state without relying on standard software for the safety function;
- controlled restart requires a fresh deliberate command;
- `AtLoad`, stop, emergency, and motion polarities are browse- and trace-proven;
- reset proves a defined home state rather than merely no X movement;
- all sensor/motion waits have bounded timeout and fault handling;
- destination selection respects real capacity and valid ranges;
- stored actions and actuator ownership are deterministic through faults and resets;
- external output writes are rejected;
- Factory I/O/OPC restart, disconnect, and stale-data tests pass;
- no forces or commissioning overrides remain; and
- dated evidence is committed alongside the exact tested source revision.

## 16. Safety Boundary

Perform these tests in simulation or on a safely isolated training setup. Standard CODESYS logic, OPC tags, and a Factory I/O Boolean named `EmergencyStop` are not substitutes for safety-rated hardware, logic, validation, and hazardous-energy control.

## 17. CODESYS References

- [SFC flags](https://content.helpme-codesys.com/en/CODESYS%20SFC/_cds_sfc_sfc_flags.html)
- [SFC Settings and Use/Declare behaviour](https://content.helpme-codesys.com/en/CODESYS%20SFC/_cds_dlg_properties_sfc_settings.html)
- [SFC processing order](https://content.helpme-codesys.com/en/CODESYS%20SFC/_cds_sfc_sequence_of_processing.html)
- [SFC step times](https://content.helpme-codesys.com/en/CODESYS%20SFC/_cds_sfc_element_properties.html)
- [SFC action qualifiers](https://content.helpme-codesys.com/en/CODESYS%20SFC/_cds_sfc_action_qualifier.html)

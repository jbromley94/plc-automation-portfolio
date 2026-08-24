# Verification and Correction Plan

## 1. Verification Status

This revision contains a static review of the supplied CODESYS V3.5 SP22 Patch 3 export and its accompanying native project file.

The application hierarchy, two POU declarations, SFC topology, branch type and order, transition expressions, IEC action associations, task configuration, library resolutions, and lack of external symbols or fixed I/O were inspected from the XML export.

The project has not been compiled or executed in this documentation environment. All build, online, timing, and generated-runtime results remain pending until recorded in CODESYS.

## 2. Supplied Source Integrity

| File | Size | SHA-256 |
|---|---:|---|
| `CODESYS/introToSFC.project` | 270,032 bytes | `3eb6fa16e2bdcd5dfa1fb39d58fb9a8024aa661160c122ee5121ff4a4287e40a` |
| `CODESYS/introToSFC.export` | 223,879 bytes | `d906fba5895139a666fc459e5e8da53f2f0d78bf780f8f4bfdcc6bb3708d1d3e` |

Recalculate these values after cloning or downloading if source integrity needs to be demonstrated. A deliberately edited CODESYS revision will have different hashes and should be labelled separately rather than replacing the as-supplied evidence silently.

## 3. Reviewed Configuration

| Item | Static finding |
|---|---|
| Profile | CODESYS V3.5 SP22 Patch 3 |
| Target | CODESYS Control Win V3 x64, version 3.5.22.30 |
| Active POU | `PLC_PRG`, SFC |
| Comparison POU | `PLC_PRG_ST`, Structured Text |
| Task | `MainTask`, cyclic, 20 ms, priority 1 |
| Scheduled program list | `PLC_PRG` only |
| Watchdog | Disabled |
| SFC steps | `Step0`, `Step1`, `Step2`, `Step3` |
| Initial step | `Step0` |
| Branch | Alternative, left path first |
| Route guards | `Trans0A`, `Trans0B` |
| Completion guards | `Trans1`, `Trans2` |
| Return guard | Literal `TRUE` |
| Action associations | `Step0: N A`; `Step1: S B`; `Step2: S C`; `Step3: R B, N C` |
| Step minimum/maximum times | None |
| Optional SFC flags | Not enabled for use |
| Symbol Configuration | Not present |
| Fixed IEC addresses | None found |
| Factory I/O / KEPServerEX | Not configured |

## 4. Test Phases

| Phase | Purpose |
|---|---|
| A — source and build | Prove file integrity, open/import behaviour, POU inventory, and task configuration |
| B — as-exported SFC | Record the exact current SFC behaviour, including `Step3` action `N C` and the absent `R C` |
| C — corrected SFC | Replace `N C` with `R C` if symmetric cleanup is the requirement, then regression-test both routes |
| D — as-exported ST | Prove that the ST POU is unscheduled and capture its missing state-1 transition in a controlled comparison build |
| E — corrected ST equivalence | Add the state-1 assignment and recovery policy, schedule deliberately, and compare traces |
| F — optional integration | Add and verify symbols, Factory I/O, or KEPServerEX only after the PLC interface is settled |

Keep evidence from the as-exported phases. It demonstrates the learning and debugging process and provides a baseline for regression testing.

## 5. General Preconditions

1. Use CODESYS Development System V3.5 SP22 Patch 3 where possible.
2. Install or select CODESYS Control Win V3 x64 version 3.5.22.30 or document any compatible replacement.
3. Preserve the supplied `.project` and `.export` files as the baseline revision.
4. Open the native project, or create a compatible project and import the export.
5. Resolve library placeholders without changing application logic.
6. Build with zero errors and record all warnings.
7. Confirm `MainTask` is cyclic at 20 ms and calls only `PLC_PRG`.
8. Start the runtime, log in, download, and place the application in Run.
9. Add `PLC_PRG.A`, `B`, `C`, and all four `PLC_PRG.Trans*` variables to a watch list.
10. Monitor the highlighted active step in the online SFC editor.
11. Add a trace with a sampling interval suitable for a 20 ms task.
12. Set all four transition guards false before each test unless the test states otherwise.
13. Remove forces and online writes after every test.
14. Record the CODESYS version, runtime version, date, and test revision with the evidence.

## 6. Functional and Regression Test Matrix

| ID | Test | Expected result | Static review | Runtime |
|---|---|---|---|---|
| VT-01 | Recalculate baseline hashes | Both files match the values in section 2 | Values recorded | Pending |
| VT-02 | Open native `.project` | Project opens without source recovery errors | File supplied | Pending |
| VT-03 | Import `.export` into a clean compatible project | Export recreates the documented application objects | XML parses correctly | Pending |
| VT-04 | Compare native and imported object trees | Both show the same documented POUs, task, and chart | Export inventory complete | Pending |
| VT-05 | Build baseline | Zero errors; warnings recorded | Cannot be proven statically | Pending |
| VT-06 | Inspect target | Control Win V3 x64, version 3.5.22.30 | Configured | Pending |
| VT-07 | Inspect task | Cyclic 20 ms, priority 1, watchdog off, only `PLC_PRG` | Configured | Pending |
| VT-08 | Inspect SFC topology | Four steps, two-path alternative branch, true return jump | Reconstructed | Pending |
| VT-09 | Inspect qualifier boxes | `N A`, `S B`, `S C`, `R B`, `N C` | Reconstructed | Pending |
| VT-10 | Start with guards false | `Step0` active and sequence waits | Consistent | Pending |
| VT-11 | Observe initial action | `A` active; `B` and `C` initially inactive | Consistent | Pending |
| VT-12 | Pulse `Trans0A` only | Left branch activates `Step1` | Configured | Pending |
| VT-13 | Observe `Step1` | `A` inactive and `B` stored active | Configured | Pending |
| VT-14 | Return `Trans0A` false while in `Step1` | No effect on current step | Guard belongs to `Step0` branch | Pending |
| VT-15 | Pulse `Trans1` | Chart activates `Step3` | Configured | Pending |
| VT-16 | Observe left-route `Step3` actions | `B` resets; `C` is active through `N` while `Step3` is active | Important current behaviour | Pending |
| VT-17 | Observe automatic return | Chart returns to `Step0` through literal `TRUE` | Configured | Pending |
| VT-18 | Complete clean right route | `Step0 → Step2 → Step3 → Step0` | Configured | Pending |
| VT-19 | Observe `Step2` | `C` becomes stored active through `S` | Configured | Pending |
| VT-20 | Observe `C` after right-route return | Record whether `C` remains active; no `R C` exists | Stored-reset omission found | Pending |
| VT-21 | Set both route guards true in `Step0` | Left `Trans0A` route opens | Left-to-right priority | Pending |
| VT-22 | Hold both route guards true across multiple loops | Left route continues to win; right route starved | Expected from priority | Pending |
| VT-23 | Hold `Trans0A` and `Trans1` true | Chart cycles rapidly through left path | Level-sensitive guards | Pending |
| VT-24 | Hold `Trans0B` and `Trans2` true with `Trans0A` false | Chart cycles rapidly through right path | Level-sensitive guards | Pending |
| VT-25 | Leave `Trans1` false in `Step1` | Chart waits indefinitely; no timeout fault | No max time | Pending |
| VT-26 | Leave `Trans2` false in `Step2` | Chart waits indefinitely; no timeout fault | No max time | Pending |
| VT-27 | Restart/reset application with all guards false | Record initial step and action values | Runtime confirmation required | Pending |
| VT-28 | Watch dormant `PLC_PRG_ST` over several task cycles | Its logic does not advance because it is unscheduled | Task list proves omission | Pending |
| VT-29 | Compare identical POU-local names | SFC and ST copies change independently | Separate local scopes | Pending |
| VT-30 | Temporary controlled scheduling of as-exported ST | ST executes only after deliberate task addition | Required for comparison | Pending |
| VT-31 | Apply `PLC_PRG_ST.Trans0A` in state 0 | State stays 0; `A` ends false for the active scan | Missing `state := 1` | Pending |
| VT-32 | Apply both ST route guards in state 0 | `Trans0A` branch wins but state still stays 0 | `IF/ELSIF` traced | Pending |
| VT-33 | Apply `PLC_PRG_ST.Trans0B` only | State becomes 2 and `C` becomes true | Configured in ST | Pending |
| VT-34 | Apply `PLC_PRG_ST.Trans2` | State becomes 3, then state-3 scan clears `B` and `C` and assigns 0 | Configured in ST | Pending |
| VT-35 | Force an invalid ST state in a controlled simulation | Outputs and state hold because no `ELSE` exists | Recovery missing | Pending |
| VT-36 | Change SFC `Step3` action from `N C` to `R C` | Corrected project builds | Proposed correction | Pending |
| VT-37 | Repeat left route after SFC correction | `Step3` does not assert `C`; both branch actions end reset | Regression target | Pending |
| VT-38 | Repeat right route after SFC correction | Stored `C` resets in `Step3` and is false on return | Regression target | Pending |
| VT-39 | Add `state := 1` to ST `Trans0A` branch | Corrected project builds | Proposed correction | Pending |
| VT-40 | Run corrected ST left route | State 1 becomes reachable; `B` activates then clears in state 3 | Regression target | Pending |
| VT-41 | Add and test invalid-state recovery | Invalid state clears outputs and enters documented recovery/fault path | Improvement required | Pending |
| VT-42 | Compare corrected SFC and ST input traces | Intended state/output behaviour matches at documented scan boundaries | Equivalence target | Pending |
| VT-43 | Search Symbol Configuration | No object exists in baseline | Confirmed statically | Pending |
| VT-44 | Search fixed addresses and device mappings | No `%I`, `%Q`, `%M`, or mapped I/O exists | Confirmed statically | Pending |
| VT-45 | Add minimal active-program symbols | Four guards writable; three action values read-only | Proposed interface | Pending |
| VT-46 | Browse generated external paths | Paths match the final documented mapping | Requires error-free build | Pending |
| VT-47 | Restart external client and PLC runtime | Mapping reconnects without writes to action variables | Integration acceptance | Pending |
| VT-48 | Enable and test appropriate SFC diagnostics | Current-step/fault data is observable without changing sequence semantics | Future improvement | Pending |
| VT-49 | Enable task watchdog with justified settings | No normal overrun; fault response documented | Future improvement | Pending |
| VT-50 | Review all forces and external write permissions | No active force or unintended write authority remains | Commissioning closeout | Pending |

## 7. Baseline SFC Test Procedure

### 7.1 Initial condition

1. Reset or freshly download the baseline application.
2. Set all transition guards false.
3. Run for at least five task cycles.
4. Record the active step and `A`, `B`, `C`.
5. Expected static result: `Step0` and `A` active; `B` and `C` inactive before any stored action history.

### 7.2 Left route

1. From `Step0`, pulse `Trans0A` true for at least one task cycle.
2. Return `Trans0A` false.
3. Confirm `Step1` becomes active and `B` is stored true.
4. Pulse `Trans1`.
5. Capture every sample through `Step3` and the return to `Step0`.
6. Confirm `B` resets in `Step3`.
7. Determine the duration and final value of `C`, because `Step3` applies `N C` even on the left route.

### 7.3 Right route

1. Begin from a clean application reset with `Trans0A` false.
2. Pulse `Trans0B`.
3. Confirm `Step2` becomes active and `C` is stored true.
4. Pulse `Trans2`.
5. Capture `C` through `Step3` and at least five scans after returning to `Step0`.
6. Record the exact as-built result before changing the qualifier.

### 7.4 Simultaneous route conditions

1. Return to a clean `Step0`.
2. Set `Trans0A` and `Trans0B` true before the same task edge.
3. Confirm only `Step1` opens.
4. Repeat at least three times.
5. Hold both true through a full loop and verify persistent left-route priority.

## 8. SFC Correction Procedure

Use this procedure only if the intended requirement is that `C` clears at convergence.

1. Save a new project revision.
2. In `Step3`, change the `C` qualifier from `N` to `R`.
3. Add a comment stating that `Step3` clears both stored branch actions.
4. Build with zero errors.
5. Repeat VT-10 through VT-27.
6. Confirm the left route no longer asserts `C` in `Step3`.
7. Confirm the right route clears stored `C` before the sequence settles in `Step0`.
8. Export the corrected project and preserve new source hashes.

If the stored `C` behaviour was intentional, retain it but document the reset owner and add an explicit tested reset path elsewhere.

## 9. ST Comparison Procedure

Do not silently add `PLC_PRG_ST` to the production task and call the two programs equivalent.

1. Save a separate comparison revision.
2. First observe the baseline task list and prove the POU is dormant.
3. Add the ST POU to a temporary task or replace the SFC task call.
4. Keep all ST tests in simulation and use its fully qualified local variables.
5. Record the as-exported `Trans0A` failure.
6. Add `state := 1` to the first branch.
7. Add an invalid-state recovery policy.
8. Decide whether outputs are assigned by full state image or explicit latch/reset logic.
9. Run the same guard traces used for the SFC.
10. Compare outputs at task boundaries, accounting for SFC activation/deactivation processing.

## 10. Static Findings

### VF-01: Stored action `C` has no overriding reset

**Severity:** Functional sequence concern.

`Step2` stores `C` using `S`. No `R C` action exists. `Step3` uses `N C`, which also activates `C` while the convergence step is active.

If both branch outputs are intended to clear at convergence, replace `N C` with `R C` and complete VT-36 through VT-38.

### VF-02: ST state 1 is unreachable

**Severity:** Definite functional defect in the comparison implementation.

The `Trans0A` branch does not assign `state := 1`. Normal source execution never enters the `B` state.

### VF-03: ST comparison is unscheduled

**Severity:** Execution/configuration limitation.

`MainTask` calls only `PLC_PRG`. The ST program cannot provide runtime evidence until deliberately scheduled.

### VF-04: Same variable names identify separate memory

**Severity:** Commissioning and test clarity concern.

Each program owns its own `A`, `B`, `C`, and `Trans*` variables. Writing a guard in the wrong namespace does not control the active chart.

### VF-05: ST has no invalid-state recovery

**Severity:** Robustness concern.

The `CASE` statement lacks an `ELSE` branch. Values outside 0–3 hold indefinitely with retained outputs.

### VF-06: Transition guards are level-sensitive

**Severity:** Interface-design concern.

Held guards can cause immediate transitions and rapid cyclic operation. No edge detection, debounce, acknowledgement, or command queue is implemented.

### VF-07: No timeout or fault path exists

**Severity:** High if adapted to equipment; expected limitation for a basic exercise.

`Step0`, `Step1`, or `Step2` can wait forever. Step time limits, SFC error flags, and recovery transitions are not configured.

### VF-08: Task watchdog is disabled

**Severity:** Diagnostic limitation.

No configured task overrun response is present. A justified watchdog setting should be tested before machine use.

### VF-09: External interface is absent

**Severity:** Integration limitation, not a PLC logic defect.

No Symbol Configuration, fixed I/O, Factory I/O scene, or KEPServerEX project exists. Watch/force testing is currently the only defined way to stimulate guards.

### VF-10: Source comments are empty

**Severity:** Maintainability concern.

The alternative-branch priority, qualifier intent, and SFC/ST comparison are not documented inside the CODESYS elements.

### VF-11: No stop, reset, or safety behaviour exists

**Severity:** High for physical use.

The program is an educational Boolean sequence, not a complete control or safety system.

## 11. Evidence to Capture

Store evidence in `Media/` using the test ID:

- native-project open and clean import;
- build result and warnings;
- application tree and 20 ms task configuration;
- full SFC with action qualifiers visible;
- left-route trace including the `Step3` value of `C`;
- right-route trace showing the final stored-state behaviour of `C`;
- simultaneous-guard left priority;
- held-guard rapid cycling;
- dormant ST watch values;
- as-exported ST left-route failure;
- corrected SFC `R C` trace;
- corrected ST state-1 route;
- invalid-state recovery;
- final Symbol Configuration and client browse; and
- force/write-access closeout.

Example filename: `VT-20-right-route-c-after-return.png`.

## 12. Completion Criteria

The project can be marked documented and runtime-verified when:

- both supplied source files pass integrity checking or a new revision is explicitly identified;
- the native project and imported export build without errors;
- the application tree, POU languages, task interval, task program list, steps, transitions, and qualifiers match the documentation;
- both route sequences have task-resolution traces;
- left priority is demonstrated when both route guards are true;
- the intended `C` behaviour is explicitly decided, coded, and proven;
- the ST state-1 assignment is corrected before equivalence is claimed;
- the ST comparison is deliberately scheduled and tested in a controlled revision;
- invalid-state, held-guard, reset, timeout, and fault policies are either implemented or clearly retained as limitations;
- any external interface exposes only the intended namespace with appropriate access rights;
- all evidence is dated and linked to test IDs;
- no forces remain active; and
- no statement claims physical or safety capability that the source does not provide.

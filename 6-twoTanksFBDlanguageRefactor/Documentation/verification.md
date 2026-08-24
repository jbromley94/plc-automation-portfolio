# Verification and Regression Plan

## 1. Verification Status

This revision contains a static review of the native CODESYS V3.5 SP22 Patch 3 export and a comparison with the preceding all-LD two-tank project.

All six POU declarations, twelve graphical networks, edge modifiers, function/function-block calls, timer values, symbol groups, and task settings were traced from the export. The application has not been executed in this documentation environment, so runtime results remain pending.

Static review found a definite Tank 2 fill-countdown mismatch and an LD/FBD discharge-edge difference. The test plan intentionally captures current as-built behaviour before correction, then requires regression testing after correction.

## 2. Reviewed Configuration

| Item | Static finding |
|---|---|
| Runtime target | CODESYS Control Win V3 x64, device version 3.5.22.30 |
| Programs | `Tank1` in LD; `Tank2` in FBD |
| Reusable function blocks | `FB_Tank` in LD; `FB_Tank_FBDlanguage` in FBD |
| Timer functions | `FC_Timer` in LD; `FC_Timer_FBDlanguage` in FBD |
| Networks | Twelve total: 2 + 2 + 1 + 1 + 3 + 3 |
| Task | `MainTask`, cyclic, 20 ms, priority 1 |
| Task order | `Tank1`, then `Tank2` |
| Watchdog | Disabled |
| Tank 1 presets | Fill 15 s; discharge default 10 s |
| Tank 2 valve presets | Fill 8 s; discharge 12 s |
| Tank 2 countdown references | Fill **15 s**; discharge 12 s |
| External symbols | Ten read/write variables across two program groups |
| OPC UA symbol support | Enabled |
| Physical I/O mapping | None found |
| Network titles/comments | Empty in all twelve networks |

## 3. Test Phases

Use two explicit evidence phases:

| Phase | Purpose |
|---|---|
| A — as exported | Prove and record the current behaviour, including the countdown defect and unequal discharge edges |
| B — corrected revision | Verify the eight-second Tank 2 fill countdown and whichever discharge convention is selected |

Do not overwrite Phase A evidence. It demonstrates the review and refactor-debugging process.

## 4. Test Preconditions

1. Import `CODESYS/twoTanksFBDlanguageRefactor.export` into a compatible Patch 3 project.
2. Confirm the CODESYS Control Win V3 x64 target.
3. Build with zero errors and resolve any library placeholders.
4. Confirm all six POUs and their expected LD/FBD view modes.
5. Confirm that `MainTask` calls `Tank1` followed by `Tank2` every 20 ms.
6. Log in, download, and place the application in Run.
7. Set `Fill`, `Discharge`, `Fill2`, and `Discharge2` false.
8. Confirm all four valve outputs are false.
9. Create a trace containing all ten exposed variables and, where possible, the four elapsed-time values.
10. Record output transition timestamps; do not assess pulse duration only from the integer display.
11. Remove all forces before each test.

The Windows soft PLC is non-real-time. Use a pulse-duration tolerance based on measured task jitter and report the tolerance with the evidence.

## 5. Functional and Regression Test Matrix

| ID | Test | Expected result | Static review | Runtime |
|---|---|---|---|---|
| VT-01 | Import and build | Application builds without errors | Export parses correctly | Pending |
| VT-02 | Inspect POU inventory | Six documented POUs are present | Consistent | Pending |
| VT-03 | Inspect POU languages and networks | LD/FBD modes and 12-network total match documentation | Consistent | Pending |
| VT-04 | Inspect task configuration | `Tank1` then `Tank2`, cyclic 20 ms, priority 1 | Configured | Pending |
| VT-05 | Start with all commands false | All valves off; countdown integers initialise to zero | Consistent with declarations | Pending |
| VT-06 | Raise `Tank1.Fill` while idle | Tank 1 fill starts; discharge stays off | LD rising edge present | Pending |
| VT-07 | Measure Tank 1 fill | Valve pulse approximately 15 s; `Timer` counts about 15 to 0 | Presets agree | Pending |
| VT-08 | Hold `Tank1.Fill` true through completion | No automatic retrigger | Edge-driven request | Pending |
| VT-09 | Raise `Tank1.Discharge` while idle | No discharge starts | LD uses falling edge | Pending |
| VT-10 | Lower `Tank1.Discharge` | Tank 1 discharge starts | Falling edge present | Pending |
| VT-11 | Measure Tank 1 discharge | Valve pulse approximately 10 s; `Timer` counts about 10 to 0 | Default and display agree | Pending |
| VT-12 | Raise `Tank2.Fill2` while idle | Tank 2 fill starts; discharge stays off | FBD rising edge present | Pending |
| VT-13 | Measure Tank 2 fill output | Valve pulse approximately 8 s | `Fill_PT = T#8S` | Pending |
| VT-14 | Observe Tank 2 fill display before correction | `Timer2` starts near 15, not 8 | **Defect confirmed statically** | Pending |
| VT-15 | Observe Tank 2 display at fill completion | Output stops at 8 s; display expected to retain near 7 | **Defect predicted statically** | Pending |
| VT-16 | Correct fill countdown input to `T#8S` and rebuild | Project builds; no unrelated logic changes | Required correction | Pending |
| VT-17 | Repeat Tank 2 fill after correction | Valve runs 8 s and display counts about 8 to 0 | Required regression result | Pending |
| VT-18 | Raise `Tank2.Discharge2` while idle | Tank 2 discharge starts immediately | FBD uses rising edge | Pending |
| VT-19 | Measure Tank 2 discharge | Valve pulse approximately 12 s; `Timer2` counts about 12 to 0 | Presets agree | Pending |
| VT-20 | Lower and hold `Tank2.Discharge2` after completion | No second discharge occurs until another rising edge | Edge-driven request | Pending |
| VT-21 | Apply the same press/release discharge trace to both tanks | Tank 1 starts on release; Tank 2 starts on press | **Language paths differ** | Pending |
| VT-22 | Choose one discharge convention and update both versions | Both tanks react on the documented transition | Improvement required | Pending |
| VT-23 | Repeat identical discharge trace after correction | LD and FBD outputs have equivalent edge behaviour | Regression target | Pending |
| VT-24 | Request Tank 1 discharge while filling | Request is rejected and not queued | Interlock traced | Pending |
| VT-25 | Request Tank 2 discharge while filling | Request is rejected and not queued | Interlock traced | Pending |
| VT-26 | Request fill while the same tank discharges | Fill request is rejected and not queued in both versions | Interlock traced | Pending |
| VT-27 | Generate both valid Tank 1 edges in one idle scan | Fill wins; discharge remains off | LD network order | Pending |
| VT-28 | Generate both Tank 2 rising edges in one idle scan | Fill wins; discharge remains off | FBD network order | Pending |
| VT-29 | Exercise request edges on pulse-completion scans | Confirm documented fill/discharge boundary asymmetry | Predicted from network order | Pending |
| VT-30 | Operate both tanks concurrently | Outputs and countdowns remain independently controlled | Separate programs/instances | Pending |
| VT-31 | Observe each countdown while idle | Last written value is retained | No explicit idle write | Pending |
| VT-32 | Compare LD and FBD timer functions with identical valid inputs | Both return the same whole-second value | Formulas are nominally equivalent | Pending |
| VT-33 | Browse symbols | Ten names appear beneath `Tank1` and `Tank2` | Configured | Pending |
| VT-34 | Browse old `Application.PLC_PRG.*` paths | Old paths are absent from this application | Namespace changed | Pending |
| VT-35 | Verify normal client write permissions | All ten selected values appear writable | Access value `3` | Pending |
| VT-36 | Inspect `Tank1` declaration usage | Eight copied Tank 2 declarations have no network references | Confirmed statically | Pending |
| VT-37 | Inspect FBD timer locals | `Var1` and `Inv_Time_INT` are declared but unused | Confirmed statically | Pending |
| VT-38 | Restart with each command already true | Record edge initialisation for LD and FBD paths | Runtime confirmation required | Pending |
| VT-39 | Inspect graphical edge markers | LD discharge is falling; FBD discharge is rising | Modifier flags traced | Pending |
| VT-40 | Search device mapping | No fixed IEC addresses or `FIO` object are present | None found | Pending |

VT-29 and VT-38 require a sufficiently detailed trace. They are verification of the implementation, not recommended operator techniques.

## 6. Static Findings

### VF-01: Tank 2 fill countdown uses the wrong maximum

**Severity:** Functional display defect.

The Tank 2 valve pulse receives `T#8S`, while its enabled fill countdown receives `T#15S`. The display cannot reach zero when the valve pulse completes.

Correction: connect `T#8S` to `FC_Timer_FBDlanguage.Max_Time`, rebuild, and complete VT-16 and VT-17.

### VF-02: Discharge command semantics differ by language path

**Severity:** Operator-interface and regression defect unless explicitly intended.

Tank 1 discharge uses a falling edge; Tank 2 discharge uses a rising edge. An identical active-high button therefore starts the two tanks at different moments.

Choose one requirement and apply it consistently. Do not label this as behavioural equivalence until VT-23 passes.

### VF-03: Duplicated implementations have already drifted

**Severity:** Architecture and maintainability concern.

The project contains two tank function blocks and two countdown functions. The countdown literal and discharge edge diverged during the refactor.

Side-by-side duplication is useful temporarily for learning and comparison. After verification, consolidate both tank instances onto one source of truth or maintain automated equivalence tests.

### VF-04: `Tank1` contains eight unused declarations

**Severity:** Maintainability and namespace clarity concern.

The former combined program declaration was retained after the logic split. It includes an unused second LD function-block instance and seven unused Tank 2 variables.

Remove them after confirming no watch list or client depends on the unused namespace.

### VF-05: FBD timer locals are unused

**Severity:** Code-cleanliness concern.

`FC_Timer_FBDlanguage` directly nests its calculation, leaving `Var1` and `Inv_Time_INT` disconnected. Remove them or deliberately expose intermediates for diagnostics.

### VF-06: Symbol paths are integration-breaking changes

**Severity:** Integration concern.

The previous paths were under `Application.PLC_PRG`. This export uses `Application.Tank1` and `Application.Tank2`.

Update Factory I/O, KEPServerEX, OPC, HMI, test, and trace clients together with the PLC revision.

### VF-07: External write access is broader than required

**Severity:** Control-authority concern.

All ten symbols are read/write. Only the four command variables normally require client writes; valve outputs and countdowns should be observational.

### VF-08: Tank 1 discharge preset is implicit

**Severity:** Configuration traceability concern.

Tank 1 leaves `Discharge_PT` unconnected and relies on the LD function-block default of ten seconds, while the countdown call contains a separate explicit ten-second literal.

Use one named preset as the source for both connections.

### VF-09: Both countdown functions have a 32.767-second range limit

**Severity:** Reuse limitation.

Both functions convert remaining milliseconds to signed 16-bit `INT` before dividing by 1000. Current presets are safe, but longer future durations can overflow.

Use a wider intermediate and validate the returned range.

### VF-10: No closed-loop process protection exists

**Severity:** High if adapted to physical equipment; expected limitation for this exercise.

There is no level, flow, valve, leak, stop, emergency, reset, or fault feedback. Timed output operation cannot prove safe filling or discharge.

### VF-11: All network titles and comments are empty

**Severity:** Maintainability concern.

The language comparison, edge difference, and required countdown correction are not explained within the graphical source.

### VF-12: No direct-open project file is included

**Severity:** Packaging limitation.

Only the portable `.export` was supplied. Reviewers must create a compatible project and import it before building.

## 7. Evidence to Capture

Store evidence in `Media/` with the corresponding test ID:

- successful Patch 3 import and build;
- the six-POU tree and 20 ms task order;
- paired screenshots of equivalent LD and FBD networks;
- Tank 1 fill and release-triggered discharge traces;
- Tank 2 fill output with the incorrect countdown before correction;
- corrected Tank 2 8-to-0 fill trace;
- Tank 2 rising-edge discharge trace;
- identical discharge command traces before and after semantic alignment;
- busy-time command rejection and simultaneous-request priority;
- concurrent operation of both programs;
- symbol browser showing the new namespaces; and
- cleaned declarations and commented networks in the corrected revision.

Example filename: `VT-17-tank2-corrected-fill-countdown.png`.

## 8. Completion Criteria

The refactor can be marked behaviourally verified when:

- the export imports and builds without errors;
- all applicable tests have dated results and evidence references;
- Tank 2 fill uses the same eight-second source for its pulse and countdown;
- both tanks use the deliberately selected discharge edge;
- identical valid input traces produce equivalent LD/FBD control behaviour apart from intentional presets and names;
- busy-time and simultaneous-command behaviour is explicitly accepted or improved;
- unused declarations are removed or justified;
- external symbol clients use the final namespaces and permissions;
- edge modifiers are visually confirmed;
- pulse durations and countdowns are measured on the running task;
- no claim implies physical level control or safety that the source does not implement; and
- the saved project, export, symbols, documentation, and evidence describe the same tested revision.

# Engineering Review

## Executive assessment

The strongest part of this project is its architecture: one reusable equipment module, a separate pair coordinator and a dedicated operator-control layer combine to create balanced production with diagnosable failure modes. That is a materially stronger controls example than two copied sequences that happen to run at the same time.

The supplied files also show realistic development-stage limitations. The control concept is suitable for virtual commissioning and portfolio demonstration, but it needs communications supervision, improved diagnostics, corrected metadata/mapping and a genuine safety design before any physical implementation.

## Effective design decisions

### One generic cell FB

`FB_MachiningCell` is instantiated once for lids and once for bases. Product type is selected through `xProduceLids`, while all state, timeout, fault and count logic remains common.

Benefits:

- Fixes and improvements are made once.
- Both cells have identical recovery behaviour.
- The design can scale to further identical machines.
- Differences remain visible as parameters instead of copied code drift.

### Held industrial handshakes

The coordinator holds both requests throughout `WaitPairComplete`; each cell holds Done throughout `Complete`. This is scan-safe and tolerates different completion times. It also avoids pulse stretching between modules within the same PLC.

### Explicit enum state machines

Named, strict enums make online state diagnosis clearer than scattered latches. Non-consecutive numeric values leave room for additional states, while 900 is visibly distinct as the fault state.

### Timers called every scan

Every state timer is executed every scan with a state-specific `IN`. This avoids a common IEC timer error in which an FB is called only inside a branch and retains confusing internal state.

### Default outputs plus final interlock

The cell assigns deterministic output defaults before its state case and then applies a final enable/fault/reset interlock afterwards. This provides two layers of output control and reduces dependence on the previous scan's assignments.

### Centralised boundary conversion

Active-low sensor, emergency-stop and Stop-button signals are converted when entering the internal modules. External OPC tags are written together at the end of `PLC_PRG`. Internal control logic can therefore use positive semantic names such as `xAtEntry` and `xEmergencyStop`.

### Controlled production stop

A routine Stop blocks the next pair while allowing the current work to complete and clear the final outfeed. This models a production stop rather than treating every Stop request as an immediate sequence abort.

### Independent production counters

Cell counts, pair count and final throughput provide useful cross-checks. The final counter uses an edge detector, so one stationary product is not counted every scan.

## Traceability from feature to implementation

| Engineering feature | Implementation evidence |
| --- | --- |
| Reusable equipment module | `fbLidsCell` and `fbBasesCell` are both `FB_MachiningCell` instances |
| Balanced production | `FB_PairCoordinator` asserts identical held requests and waits for both Done signals |
| Controlled stop | `xStopPending`, `xAllowNewPair` and `fbStopOutfeedClearance` |
| Deterministic recovery | Reset priority, 500 ms `TP`, state reset and latched Reset Required |
| Fault diagnosis | Codes 1, 101, 102, 103, 104 and 999 |
| Scan-cycle awareness | Held handshakes; 20 ms cyclic task; edge-triggered commands and count |
| I/O abstraction | Qualified `FIO` GVL and final `PLC_PRG` mapping section |
| Simulation integration | 53 Factory I/O OPC DA items through `PTC.KepwareServer` |

## Confirmed review findings

### 1. Base Opened feedback is mis-mapped

The latest Factory I/O scene maps `xBasesCenterOpened` to the base **Progress** integer, the same point used by `diBasesCenterProgress`. The actual binary Opened point at BI 9 is unmapped.

Current impact: none on sequencing, because `xOpened` is not referenced inside `FB_MachiningCell`.

Required action before use: bind `xBasesCenterOpened` to `Machining Center 2 (Opened)`, GUID `377c4656-24e1-4963-9dca-25054aef73e2`, then prove both Boolean states.

### 2. Some source comments no longer match behaviour

| Source comment | Actual latest implementation |
| --- | --- |
| Emitters/remover are forced on in the supplied scene | Every latest-scene `UseForcedValue` flag is false; PLC commands are effective |
| Busy or the falling entry signal confirms pickup | The transition checks `xBusy` only |
| `diTotalPartsCounterDisplay` sits in the GVL input section | `PLC_PRG` writes it and Factory I/O consumes it as an integer display output |

These are documentation/code-quality defects rather than runtime faults. Update the comments in the next code revision so they remain trustworthy.

### 3. Reserved feedback is published but unused

`xOpened`, `diProgress` and `rFactoryIOTimeScale` are exchanged externally but have no control effect. This is acceptable as a deliberate diagnostic reserve, but it should be labelled clearly or removed until it has a defined purpose.

Good next uses:

- Inhibit Start and alarm on a machining-centre door-open condition.
- Display progress on an HMI and use it only for information, not sequence completion.
- Scale timeout expectations or warn when the simulation time scale is outside the configured range.

### 4. Communications health is not supervised

The control logic reads values but not OPC quality or message age. A separate heartbeat is required to distinguish a healthy static process from stale communication.

Recommended design:

1. Generate a changing heartbeat at the supervisory/scene side.
2. Detect transitions or a rolling count in the PLC.
3. Start a communication timeout when it stops changing.
4. Latch a dedicated communications fault.
5. Apply a documented safe output state and require Reset after recovery.

### 5. Controlled Stop has no maximum drain time

The stop sequence can remain pending forever if the final count never reaches `pairs × 2` or the final beam never clears. Add a controlled-stop timeout and diagnostic code that identifies which completion condition failed.

### 6. Diagnostic variables are local only

Cell states, cell fault codes, coordinator state, pair count, Run, Stop Pending and Plant Faulted are locals in `PLC_PRG`; the current symbol selection publishes only `FIO`. This limits external HMI/SCADA diagnosis.

Recommended action: create a read-only qualified diagnostics GVL or extend the symbol interface with explicit status fields.

### 7. All published `FIO` variables have read/write access

The logical direction is documented in comments but not enforced by IEC addresses or symbol permissions. Tighten external access where the communication stack supports it, particularly for scene-to-PLC feedback and safety-related status signals.

### 8. Bases-entry sensor range deserves review

The saved bases-entry retroreflective sensor range is 6 m while its reflector is approximately 1 m away; the other entry, exit and removal sensors are set to about 1 to 1.18 m. Because a retroreflective beam terminates at its reflector, this is not evidence of stray detection, but it is an inconsistent setting. Consider reducing the range to roughly 1–1.2 m and re-prove the input after the change.

### 9. Inactive driver maps are stale

Factory I/O retains older profiles for Modbus, Siemens, Allen-Bradley and other drivers. They contain fewer points than the 53-item OPC configuration. Treat OPC Client DA as the only complete saved map; rebuild and test any protocol migration.

### 10. Stop is edge-detected rather than level-interlocked

The normally closed scene Stop input is inverted and passed through `R_TRIG`. Its false transition therefore creates one controlled-stop request, but the healthy raw level is not included in the Start condition or final plant-enable expression. Because Reset has higher priority and clears Stop Pending, holding Stop false while issuing Reset and then Start can allow Run to relatch after the original edge has been consumed.

Required correction: include a healthy Stop-circuit level in the run permissive, define whether a Stop-circuit loss sets Reset Required, and add regression tests for press, hold, release, disconnection and Reset/Start interactions. Do not rely on the current software Stop logic as a safety function.

## Known limitations

| Area | Current limitation | Consequence |
| --- | --- | --- |
| Safety | Emergency stop is software simulation over OPC | Not safety-rated and unsuitable for real hazardous motion |
| Runtime | Control Win is a non-real-time Windows soft PLC | Windows scheduling is non-deterministic; timing suitability requires measurement and validation |
| Task supervision | PLC task watchdog is disabled | A task overrun is not supervised by the supplied configuration |
| Communications | No quality/heartbeat watchdog | Stale or default external values may not be distinguished reliably |
| Stop input | Edge request only; healthy level is not a maintained permissive | Run can relatch after Reset while Stop remains false |
| Modes | Manual is an inhibit only | No jog, maintenance or recovery movement is available |
| Mid-cycle mode change | Manual or an invalid selector combination clears Run immediately but retains sequence state | This bypasses controlled draining; a later valid Auto + Start can resume the retained cycle |
| Outfeed | Exit conveyors, emitters and remover broadly follow plant enable | No zone control, accumulation logic or downstream interlock |
| Stop | No drain timeout | Controlled Stop may remain pending indefinitely |
| Diagnostics | Fault codes/states are not in the external GVL | Scene/HMI cannot display detailed internal diagnosis without expansion |
| Persistence | Counters are non-retentive and clear on Reset | Production totals are not suitable for reporting/OEE |
| Process selection | `xProduceLidsCmd` remains configured when disabled/faulted | Intentional static selection, but not part of the motion interlock |
| Timing | Values are tuned to the virtual scene | Physical machinery would require measured limits and validation |
| Test evidence | No execution log is embedded in the supplied files | Formal acceptance results must be recorded separately |

## Improvement roadmap

### Priority 1 — correctness and diagnosis

- Correct the base Opened OPC mapping.
- Update the three stale/misleading source comments.
- Make the healthy Stop level a maintained run permissive and prove Reset/Start interactions.
- Add an OPC heartbeat and latched communication fault.
- Add a controlled-stop timeout with an actionable diagnostic.
- Publish plant state, both cell states, both fault codes, coordinator state, pair count, Run and Stop Pending as read-only diagnostics.
- Prove and record every active-low input and all 53 OPC items.

### Priority 2 — operating capability

- Implement a proper Manual/Maintenance mode with permissive-controlled jog commands.
- Replace broad outfeed enable with zone occupancy and downstream-available interlocks.
- Add an operator alarm summary, first-out indication and alarm history.
- Separate Reset Sequence from Reset Production Totals.
- Make lifetime or shift counters retentive where appropriate.
- Enable and tune the task watchdog after measuring scan execution time.

### Priority 3 — platform progression

- Migrate the external interface to OPC UA where suitable; the CODESYS symbol configuration already records OPC UA support.
- Commission the reusable cell FB against a physical PLC and representative 24 VDC I/O.
- Create a safety requirements specification and implement safety functions with suitable hardware independently of standard control code.
- Add automated simulation regression tests for state transitions, timeout faults and counter invariants.
- Add SCADA trends for cycle time, blocked/starved time and per-cell utilisation.

## Portfolio value

This project demonstrates more than basic sequencing:

- IEC 61131-3 Structured Text organised into reusable function blocks.
- Parallel machine coordination using held, scan-safe handshakes.
- Ready/request/done handshakes that respect PLC scan behaviour.
- State-specific timeouts and latched fault codes.
- Distinction between routine stop, emergency stop, pause and reset.
- Active-low signal normalisation and final output interlocking.
- OPC DA integration and a point-by-point virtual-commissioning plan.
- Honest engineering review of defects, assumptions and required next steps.

The strongest interview explanation is: **two copies of one equipment module are orchestrated by a separate coordinator, allowing matched production and controlled recovery even when the cells complete at different times.**

## Release checklist for the next revision

- [ ] Base Opened mapping corrected and proved.
- [ ] Source comments reconciled with the latest scene.
- [ ] All commissioning tests have Result and Evidence entries.
- [ ] Ten-pair soak test meets the counter invariant.
- [ ] Communication-loss behaviour tested and documented.
- [ ] No active forces or simulated failures remain.
- [ ] Diagnostics published to the intended HMI/SCADA layer.
- [ ] README version/path table updated if Kepware names change.

[Back to the project README](../README.md)

# Commissioning and Testing

## Purpose

This document separates configuration checks, point-to-point proving and functional acceptance. Do not begin automatic production until the basic I/O states and active-low devices have been proved.

The attached source files do not contain runtime logs or test evidence. Result and evidence fields below are deliberately left open rather than presenting proposed tests as completed tests.

## Required software and files

| Item | Supplied or expected configuration |
| --- | --- |
| CODESYS export | `CODESYS/twinCellBalancedProduction.export` |
| Factory I/O scene | `FactoryIO/twinCellBalancedProduction.factoryio` |
| CODESYS engineering profile | V3.5 SP22 Patch 3 |
| Runtime | CODESYS Control Win V3 x64 3.5.22.30 or a deliberately migrated target |
| OPC server | Kepware exposing OPC DA server `PTC.KepwareServer` |
| Saved namespace | `Channel2.project10.Application.FIO.*` |
| PLC task | Cyclic `MainTask`, 20 ms, priority 1 |

The Kepware project/configuration is an external dependency and is not included in this repository.

## 1. CODESYS preparation

1. Open a CODESYS installation compatible with V3.5 SP22 Patch 3.
2. The export contains the full top-level `Device`, `PLC Logic` and `Application` tree. For a complete import, begin with an Empty Project. When migrating into an existing compatible target, select only the required objects beneath its existing Application to avoid duplicate device or application nodes.
3. Confirm the imported application contains:
   - `E_MachiningCellState`
   - `E_PairCoordinatorState`
   - `Commands`
   - `FIO`
   - `FB_MachiningCell`
   - `FB_OperatorControl`
   - `FB_PairCoordinator`
   - `FB_ThroughputCounter`
   - `PLC_PRG`
4. Confirm the target is `CODESYS Control Win V3 x64`, or perform and record any device migration.
5. Select the active local runtime rather than relying on the machine-specific communication identity stored in the export.
6. Confirm `MainTask` is cyclic at `T#20MS`, priority 1, and calls `PLC_PRG`.
7. Update the application, resolve libraries and compile with no errors.
8. Confirm all 53 variables in qualified GVL `FIO` are selected in Symbol Configuration.
9. Start the runtime, log in, download and put the application into Run.
10. Verify online that `xResetRequired` is initially true and `xEnablePlant` is false.

The export has no physical `%I`/`%Q` addresses. Successful compilation alone does not prove that the external OPC path is correct.

## 2. Kepware and OPC checks

1. Start Kepware and open the project that contains `Channel2.project10`.
2. Confirm the CODESYS device is connected and the `Application.FIO` symbols are browsable.
3. Verify read/write access for the required Boolean, DINT and REAL tags.
4. Confirm there are no stale tags from an older CODESYS download.
5. Use the OPC Quick Client or equivalent diagnostics to verify live value changes.
6. Keep the saved `Channel2.project10.Application.FIO.*` namespace, or update all 53 Factory I/O items if the Kepware channel/device names change.

Minimum live checks after opening and connecting the scene, but before selecting Auto:

| Tag | Expected observation |
| --- | --- |
| `FIO.xFactoryIORunning` | Changes with Factory I/O Run/Stop once connected |
| `FIO.xTwinCellEmergencyStop` | True when the scene emergency stop is released |
| `FIO.xTwinCellStopPB` | True when the normally closed Stop device is released |
| `FIO.xTwinCellStartPB` | Momentarily true when Start is pressed |
| `FIO.xTwinCellResetPB` | Momentarily true when Reset is pressed |

## 3. Factory I/O preparation

1. Open `FactoryIO/twinCellBalancedProduction.factoryio` in Factory I/O 2.5.10 or a deliberately validated later version.
2. Open the driver configuration and select **OPC Client DA**.
3. Select `PTC.KepwareServer`.
4. Confirm the saved browser filters are `Starts with: Channel2` and `Contains: FIO`.
5. Confirm 53 mapped OPC items are present.
6. Verify good quality for every signal required by the current sequence.
7. Confirm there are no active forces, open circuits or short circuits.
8. Correct the known `xBasesCenterOpened` mapping before using that feedback. For V1 it may be explicitly accepted as an unused known issue because the cell FB does not reference `xOpened`.
9. Put the simulation into Run.

## 4. Static I/O proving

Prove one signal at a time with automatic operation disabled.

### Inputs to the PLC

| Check | Expected raw state |
| --- | --- |
| Emergency stop released / pressed | True / false |
| Stop released / pressed | True / false |
| Start released / pressed | False / true |
| Reset released / pressed | False / true |
| Auto selected | Auto true, Manual false |
| Manual selected | Auto false, Manual true |
| Each retro beam clear / blocked | True / false |
| Machining centre idle / running | Busy false / true |
| Healthy / station error | Has Error false / true |

### Outputs from the PLC

With a controlled CODESYS force in the virtual environment or a dedicated commissioning path, prove the following. An ordinary watch write is insufficient because `PLC_PRG` rewrites these outputs every 20 ms.

Keep each machining centre's Start, Stop and Reset commands mutually exclusive. With Auto disabled, Stop is normally asserted; prove these commands through the normal sequence where possible, or use a controlled force that explicitly makes both competing commands false before asserting the command under test. Never assert more than one of the three together.

- Lid and base raw conveyors move in the intended direction.
- Both exit stages per lane move parts toward the merge.
- The shared exit conveyor moves parts toward the final beam and remover.
- Lid selection produces a lid and base selection produces a base.
- Start, Stop and Reset reach the correct machining centre only.
- Both emitters and the remover respond to their mapped commands.
- Start, Stop and Reset lamps follow the intended outputs.
- All five counter displays show the correct value.

Remove every temporary force and return all watch-list command variables to false before automatic testing.

## 5. Safe first automatic cycle

1. Confirm the scene is Running and not Paused.
2. Release the emergency stop.
3. Select Auto; confirm Manual is false.
4. Press Reset once.
5. Confirm:
   - `xResetRequired = FALSE`
   - both cell states = `Stopped` (0)
   - coordinator state = `Stopped` (0)
   - both cell fault codes = 0
   - all production counters = 0
6. Press Start once.
7. Observe both cells move through `Ready → FeedingRaw → WaitingForPickup → Processing → Complete`.
8. Confirm neither cell starts a second product while waiting for the other.
9. Confirm both cell exits clear and the coordinator observes the two-second merge clearance.
10. Confirm two distinct edges at the final sensor and a total-part count of 2.

## Acceptance-test record

Complete the Result and Evidence columns during commissioning. Suitable evidence includes screenshots of online states, a short Factory I/O video, an OPC quality capture and counter values before/after the test.

| ID | Test / method | Expected result | Result | Evidence |
| --- | --- | --- | --- | --- |
| TC-01 | Cold start; press Start before Reset | Plant remains disabled and Reset lamp remains on | — | — |
| TC-02 | Start, wait for both requests to assert, then press Stop so no second pair is admitted | Exactly one lid and one base complete; pair count = 1; final total = 2; plant stops after clearance | — | — |
| TC-03 | Ten-pair soak | Lid = base = pairs = 10; final total = 20; no unplanned fault | — | — |
| TC-04 | Press Stop while a pair is active | No new pair starts; active pair reaches removal; plant stops after final beam is clear for 2 s | — | — |
| TC-05 | Press Stop while no pair is active | Stop Pending asserts; no new pair starts; Run clears after the two-second safe-clearance condition | — | — |
| TC-06 | Press emergency stop during production | Plant enable drops immediately; cell commands enter safe state; Reset Required latches | — | — |
| TC-07 | Try Reset while emergency stop remains active | No reset pulse is produced | — | — |
| TC-08 | Select Manual, neither mode, then both signals | Automatic enable remains false in every invalid selection | — | — |
| TC-09 | Pause during an active state, then resume | Motion/process commands de-energise, machining Stop asserts, state is retained and timers reset; sequence re-enables after resume | — | — |
| TC-10 | Hold one part in the final beam | Total increments once, not once per 20 ms scan | — | — |
| TC-11 | Suppress entry detection for more than 15 s | Affected cell faults with code 101; complete plant requires Reset | — | — |
| TC-12 | Prove entry, then suppress Busy for more than 15 s | Affected cell faults with code 102 | — | — |
| TC-13 | Allow Busy, then suppress exit detection for more than 90 s | Affected cell faults with code 103 | — | — |
| TC-14 | Hold the cell exit occupied for 10 s in Complete | Affected cell faults with code 104 | — | — |
| TC-15 | Assert machining-centre Has Error | Affected cell faults with code 1 and coordinator faults on propagation | — | — |
| TC-16 | Reset after each injected fault | States return to Stopped; faults/codes and all counters clear; Reset does not start plant | — | — |
| TC-17 | Interrupt OPC communication during production | Observe and record value quality and actual command behaviour; compare with safe-state requirement | — | — |
| TC-18 | Correct and prove base Opened mapping | `xBasesCenterOpened` follows BI 9, not the progress integer | — | — |
| TC-19 | Virtual-only: hold the normally closed Stop input false, issue Reset, then attempt Start | Desired behaviour is to remain disabled; current V1 may relatch Run after the original Stop edge has been consumed, so record the test as failed until a level-held Stop permissive is implemented | — | — |

## Fault-injection guidance

Use only in the virtual scene and record every temporary change.

| Fault | Injection approach | Restore before Reset |
| --- | --- | --- |
| 101, no material at entry | Keep the entry beam clear after the coordinator requests the affected cell; disable its emitter/feed path or temporarily prevent the sensor transition | Restore emitter, conveyor and sensor mapping |
| 102, pickup not confirmed | Allow entry detection, then prevent the affected machining centre from asserting Busy | Restore Start/Busy path |
| 103, no product at exit | Allow Busy, then prevent the finished product or exit signal from arriving | Restore outfeed and sensor path |
| 104, exit obstructed | Hold a completed part at the affected exit beam | Clear the physical blockage |
| 1, station error | Use Factory I/O failure injection or a controlled temporary signal force | Clear the station error and remove force |
| 999, invalid enum | Development-only online write to an invalid state value | Restore a valid enum and remove write/force |

If `xHasError` remains true after the reset pulse, fault code 1 will immediately return.

## Controlled-stop acceptance detail

A controlled stop is complete only when all four conditions are satisfied:

```text
xStopPending
AND coordinator state = Ready
AND final count >= completed pairs × 2
AND final raw beam signal = TRUE continuously for 2 seconds
```

If Stop Pending never clears, inspect those conditions individually rather than repeatedly pressing Stop or Reset.

## Communications-loss test

The application has no explicit OPC quality or heartbeat watchdog. A real communications interruption may leave values at driver-dependent defaults or last-known states. TC-17 is therefore a required design-discovery test, not a behaviour that should be assumed safe.

For a physical implementation, add a toggling heartbeat or monotonically changing counter, supervise its timeout in the PLC, and force an independently defined safe state on loss.

## Completion criteria

Virtual commissioning is complete when:

- All required OPC items show good quality.
- Every active-low input has been proved in both states.
- The base Opened mapping is corrected or formally accepted as unused.
- Normal and controlled-stop tests pass.
- All five user-reachable fault paths recover through one deliberate Reset.
- The ten-pair soak finishes with `lids = bases = pairs` and `total = pairs × 2`.
- No temporary forces or failure injections remain active.
- Test evidence is stored with the project.

[Back to the project README](../README.md)

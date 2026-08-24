# Commissioning and Testing

## Purpose

This guide separates configuration checks, point-to-point proving and functional acceptance. It is written for the supplied virtual environment; it is not a physical-machine commissioning procedure.

The source files contain no runtime log, completed test sheet or video evidence. Result and Evidence fields are intentionally blank so proposed tests are not presented as passed tests.

## Required software and files

| Item | Supplied or expected configuration |
| --- | --- |
| CODESYS export | `CODESYS/assemblyAndPackoutCell.export` |
| Factory I/O scene | `FactoryIO/assemblyAndPackoutCell.factoryio` |
| CODESYS profile | V3.5 SP22 Patch 3 |
| Runtime | CODESYS Control Win V3 x64 3.5.22.30, or a deliberately migrated target |
| OPC server | Kepware exposing OPC DA server `PTC.KepwareServer` |
| Saved namespace | `Channel2.project11.Application.FIO.*` |
| PLC task | Cyclic `MainTask`, 20 ms, priority 1 |

The Kepware project/configuration is an external dependency and is not included.

## 1. CODESYS preparation

1. Open a CODESYS installation compatible with V3.5 SP22 Patch 3.
2. The export contains the full top-level `Device`, `PLC Logic` and `Application` tree. For a complete import, begin with an Empty Project. When migrating into an existing compatible target, select only the required objects beneath its existing Application to avoid duplicate device/application nodes.
3. Confirm the imported Application contains:
   - `E_AssemblerState`
   - `AssemblyConstants`
   - `Commands`
   - `FIO`
   - `FB_AssemblerCell`
   - `FB_OperatorControl`
   - `PLC_PRG`
4. Confirm or migrate the `CODESYS Control Win V3 x64` device.
5. Rescan and select the intended runtime rather than relying on the machine-specific communication identity stored in the export.
6. Confirm `MainTask` is cyclic at `T#20MS`, priority 1, calls `PLC_PRG` and has its watchdog disabled in the supplied configuration.
7. Resolve libraries, compile with no errors, log in and download.
8. Confirm all 40 variables in the `FIO` symbol group are selected.
9. Put the application into Run.
10. Record the cold-start values of state, fault code, Run, Stop Pending and Reset Required.

No `%I` or `%Q` variables are present. Successful compilation does not prove the OPC path.

The supplied `xResetRequired` has no explicit true initialiser, so an unmodified cold start may accept Start before Reset. Use Reset deliberately and decide whether a known-reset requirement should be added in the next revision.

## 2. Kepware and OPC checks

1. Start Kepware and open the project containing `Channel2.project11`.
2. Confirm the intended CODESYS runtime/device is connected.
3. Browse all 40 `Application.FIO` symbols.
4. Confirm Boolean, DINT and REAL values have good quality and expected read/write access.
5. Check for stale names from an older CODESYS download.
6. Use an OPC diagnostic client to observe live transitions.
7. Retain the saved namespace or update all 40 Factory I/O item paths consistently if the channel/device name changes.

Minimum live checks after opening and connecting the scene, before Auto:

| Tag | Expected observation |
| --- | --- |
| `FIO.xFactoryIORunning` | Follows Factory I/O Run/Stop |
| `FIO.xFactoryIOPaused` | Follows Factory I/O Pause |
| `FIO.xEmergencyStop` | True when released/healthy |
| `FIO.xStopPB` | True when released/healthy |
| `FIO.xStartPB` | Momentarily true when pressed |
| `FIO.xResetPB` | Momentarily true when pressed |
| `FIO.xAutoModeSelected` / `xManualModeSelected` | Mutually exclusive for normal selector positions |

## 3. Factory I/O preparation

1. Open `FactoryIO/assemblyAndPackoutCell.factoryio` in Factory I/O 2.5.10 or a deliberately validated later version.
2. Select **OPC Client DA** and `PTC.KepwareServer`.
3. Confirm the browser filters are `Starts with: Channel2` and `Contains: FIO`.
4. Confirm all 40 OPC items are present and have good quality.
5. Confirm there are no active Factory I/O forces, open circuits or short circuits.
6. Clear the conveyors, clamps, gripper and leaving sensor before the first sequence.
7. Put the scene into Run with Auto not yet selected.

Five outputs retain saved true force values, but `UseForcedValue` is false for every point. Do not enable those force flags.

## 4. Static input proving

Prove one point at a time with automatic operation disabled.

| Check | Expected raw state |
| --- | --- |
| Emergency stop released / pressed | True / false |
| Stop released / pressed | True / false |
| Start released / pressed | False / true |
| Reset released / pressed | False / true |
| Auto selected | Auto true, Manual false |
| Manual selected | Auto false, Manual true |
| Lid/base absent / at place | False / true |
| Lid/base unclamped / clamped | False / true |
| Base positioner below / at raised limit | False / true |
| X or Z stationary / moving | False / true |
| Gripper empty / lid detected | False / true |
| Leaving sensor clear / occupied | False / true |

Record both the Factory I/O point and corresponding `FIO` value. A visual animation without an online tag transition is not a point-to-point proof.

## 5. Output proving

Use the normal sequence wherever possible. If a controlled CODESYS force or dedicated commissioning path is needed, use it only in the virtual environment. An ordinary watch write is insufficient because `PLC_PRG` rewrites outputs every 20 ms.

Prove:

- each emitter and conveyor reaches the intended lane;
- lid and base clamps act independently and return correct feedback;
- base-positioner raise reaches its limit;
- X=true selects the base side and X=false the lid/home side;
- Z=true lowers and Z=false raises;
- Grab acquires/releases a lid and Item Detected changes;
- both removers operate on the intended lane;
- Start, Stop and Reset lamps follow their commands;
- the display follows `diProductsCompleted`.

Keep motion, clamp and gripper commands in a physically coherent combination. Never test opposing operator commands simultaneously. Remove every temporary force and return all `Commands` variables to false before automatic testing.

## 6. Safe first automatic cycle

1. Confirm the scene is Running and not Paused.
2. Release the emergency stop and Stop button.
3. Select Auto only.
4. Press Reset once.
5. Confirm state `Stopped` (0), fault code 0, count 0 and Run false.
6. Press Start once.
7. Observe the full sequence through Homing, feed, clamp, pickup, transfer, release, return and outfeed.
8. Confirm the occupied infeed lane stops while the other continues if arrival times differ.
9. Confirm the lid remains detected while it is transferred.
10. Confirm the assembled product physically reaches and clears the leaving sensor and remover.
11. Confirm exactly one display increment.
12. Inspect the next cycle for leftover parts or a positioner/clamp state inconsistent with the PLC.

## Acceptance-test record

Complete Result and Evidence during execution. Useful evidence includes online state/fault captures, OPC-quality screenshots, short scene videos and before/after counter values.

| ID | Test / method | Expected or observation target | Result | Evidence |
| --- | --- | --- | --- | --- |
| TC-01 | Import/build, reselect runtime and inspect task | Clean build; intended runtime; 20 ms cyclic task, priority 1 | — | — |
| TC-02 | Prove all 40 OPC items | Correct point, direction, type, polarity and good quality | — | — |
| TC-03 | Reset, then run one nominal cycle | All 15 nonfault states occur in order; one product reaches remover; count = 1 | — | — |
| TC-04 | Delay one infeed part | Occupied lane stops; missing lane continues; sequence waits for both | — | — |
| TC-05 | Run ten cycles | Ten leaving edges and count = 10; record WIP, colour combinations and any faults | — | — |
| TC-06 | Suppress one at-place input for more than 15 s | Fault 101; cell commands de-energise | — | — |
| TC-07 | Prevent one clamp proof for more than 5 s | Fault 102 | — | — |
| TC-08 | Prevent gripper Item Detected from settling | Fault 203 within 5 s | — | — |
| TC-09 | Hold Lid Clamped true after its command releases | Fault 204 | — | — |
| TC-10 | Hold either Moving input true during Homing; in commanded travel, suppress the required Moving pulse or hold Moving true | Corresponding fault 201, 202 or 205–209; record exact phase | — | — |
| TC-11 | Prevent base positioner raised-limit proof | Fault 210 | — | — |
| TC-12 | Prevent the leaving edge for more than 10 s | Desired fault 301; **the supplied revision is expected to remain waiting because the wrong timer Q is checked** | — | — |
| TC-13 | Make Part Leaving rise before state 140 and remain high | Record consumed-edge stall; input must fall/rise again in the supplied revision | — | — |
| TC-14 | Hold Part Leaving high after a valid edge | Counter increments once, not once per scan | — | — |
| TC-15 | Press Stop during pickup or transfer | Current product reaches leaving-sensor rising edge; Run then clears | — | — |
| TC-16 | Press Stop while waiting with no complete product active | Record that the supplied revision can start/finish another product to obtain its completion pulse | — | — |
| TC-17 | Hold Stop false, issue Reset, then attempt Start | Desired inhibit; the supplied revision may relatch Run after the original Stop edge is consumed | — | — |
| TC-18 | Press Start while Stop Pending | The supplied revision cancels Stop Pending; record against required operator policy | — | — |
| TC-19 | Emergency stop during each motion phase | Enable and commands drop; Reset Required latches; no Reset accepted until released | — | — |
| TC-20 | Pause during lid transfer, then resume | The supplied revision resets sequence to Stopped and automatically restarts Homing because Run stays latched | — | — |
| TC-21 | Select Manual, neither mode and both signals mid-cycle | Run clears; cell returns Stopped; fresh Start required after valid Auto | — | — |
| TC-22 | Press routine Reset mid-cycle | The supplied revision aborts immediately and clears count; document residual physical WIP | — | — |
| TC-23 | Interrupt OPC/runtime communication | Record quality, retained/default values and actual output behaviour | — | — |
| TC-24 | Verify physical outfeed clearance after completion edge | Product continues through sensor, conveyor/chute and remover before outputs stop | — | — |

## Fault-injection guidance

Use only in the virtual environment and record every temporary change.

| Fault group | Controlled injection | Restore before Reset |
| --- | --- | --- |
| 101 | Prevent one part reaching its at-place sensor | Restore emitter, conveyor and sensor path |
| 102 / 204 | Prevent a clamp from setting or releasing | Restore clamp command/feedback and part alignment |
| 201–202, 205–210 | Hold moving feedback true during Homing, prevent a required movement/stop transition, or prevent the positioner limit | Restore axis/positioner feedback and safe physical state |
| 203 | Prevent stable gripper item detection | Restore lid position and gripper feedback |
| 301 | Prevent/advance the leaving edge | Restore sensor; fix the wrong timer-Q reference before formal acceptance |
| 999 | Development-only online write of an invalid enum | Restore valid state and remove write/force |

## Release gates before claiming commissioning complete

- Change state 140 to check `fbProductExitTimeout.Q` and prove fault 301.
- Replace the completion pulse with a maintained safe-stop condition that covers true idle/empty and confirmed outfeed clearance.
- Hold the healthy Stop level in the run permissive and prove held/disconnected cases.
- Decide and test the response to Pause, runtime loss and mid-cycle Reset.
- Stretch or otherwise acknowledge the one-scan Factory I/O reset command across the OPC update rate.
- Add communications quality/heartbeat supervision.
- Publish state, fault code, Run and Stop Pending for diagnosis.
- Complete every Result/Evidence field relevant to the intended release.

[Back to the project README](../README.md)

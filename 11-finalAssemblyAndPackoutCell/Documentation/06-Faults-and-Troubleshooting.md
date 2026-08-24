# Faults and Troubleshooting

## Fault model

`FB_AssemblerCell` latches its state at `Faulted` and retains `uiFaultCode` until Reset. The final interlock forces every cell-owned command false in the same scan that a timeout faults. On the following scan, plant fault reaches `FB_OperatorControl`, which clears Run and Stop Pending and sets Reset Required.

The emitters and removers are driven outside the cell FB from automatic enable, so they can remain on for the single fault-detection scan before operator control receives the new fault.

## Fault-code reference

| Code | State / meaning | Trigger | First checks |
| ---: | --- | --- | --- |
| 0 | No cell fault | Reset/default | None |
| 101 | Parts not at assembly positions | Both at-place inputs not true within 15 s | Emitters, lane conveyors, sensor alignment, blockage, OPC quality |
| 102 | Parts not both clamped | Both clamp proofs not true within 5 s | Part alignment, clamp commands, clamp feedback |
| 201 | Homing movement timeout | Homing does not reach 500 ms stationary before 5 s motion timer | X/Z motion feedback, obstruction, command direction |
| 202 | Lowering-to-lid timeout | Z motion not observed and completed within 5 s | Z command and Moving Z |
| 203 | Lid pickup not confirmed | Item Detected does not remain true for 300 ms within 5 s | Lid position, Grab command, gripper sensor |
| 204 | Lid clamp failed to release | Lid Clamped does not become false within 5 s | Clamp output, feedback, mechanical interference |
| 205 | Raise-with-lid timeout | Z motion handshake not completed within 5 s | Moving Z, held lid, obstruction |
| 206 | Move-to-base timeout | X motion handshake not completed within 5 s | Moving X, travel obstruction |
| 207 | Lower-to-base timeout | Z motion handshake not completed within 5 s | Moving Z, base/lid alignment |
| 208 | Raise-from-base timeout | Z motion handshake not completed within 5 s | Moving Z, join/release interference |
| 209 | Return-home timeout | X motion handshake not completed within 5 s | Moving X, travel obstruction |
| 210 | Base positioner failed to raise | Raised-limit input not true within 5 s | Positioner command, limit feedback, clamp/product obstruction |
| 301 | Intended product-exit timeout | Intended after 10 s without a leaving edge | **Unreachable in the supplied revision because state 140 checks the wrong timer Q** |
| 999 | Invalid state | Enum outside defined values | Online write/force, unsafe edit, memory corruption |

## Recovery procedure

1. Record the state, fault code, all relevant sensor values and OPC quality before resetting.
2. Stop or secure the virtual process and inspect the position of the base, lid and gantry.
3. Correct the initiating condition and clear unsafe residual WIP.
4. Confirm the emergency-stop and Stop raw contacts are both true/healthy.
5. Confirm Auto=true and Manual=false.
6. Confirm the scene is Running and not Paused.
7. Remove all temporary forces or failure injections.
8. Press Reset once.
9. Confirm state `Stopped`, fault code 0, count 0, Run false and Stop Pending false.
10. Confirm the physical scene is also in a state from which Homing and feeding can restart safely.
11. Press Start.

Reset clears the product count. It can also abort a normal active cycle, so capture evidence and clear WIP before using it.

The Factory I/O scene-reset output is only one PLC scan and is issued only when Reset Required was already true. Confirm that the OPC path actually transmits it; do not assume the visual scene reset merely because the PLC fault cleared.

## Symptom-based troubleshooting

### Plant will not start

Check these online values:

| Condition | Required state |
| --- | --- |
| `FIO.xFactoryIORunning` | True |
| `FIO.xFactoryIOPaused` | False |
| `FIO.xEmergencyStop` | True: released/healthy |
| `FIO.xStopPB` | True: released/healthy |
| `FIO.xAutoModeSelected` | True |
| `FIO.xManualModeSelected` | False |
| `xAssemblerFaulted` | False |
| `xResetRequired` | False |

Then confirm Start changes false → true → false. A Start edge consumed while emergency stop, pause, scene Stop, invalid mode or fault blocks it must be released and pressed again.

### Plant can restart while Stop remains pressed

This is a confirmed limitation in the supplied revision. The healthy-high Stop signal is inverted and passed through `R_TRIG`; it creates one request edge but is not included as a maintained enable permissive. Reset can clear Stop Pending, and a later Start can relatch Run while the raw Stop remains false.

Keep `FIO.xStopPB = TRUE` before Reset or Start. In a corrected version, include the healthy Stop level in the run permissive and define the required reset policy for a failed Stop circuit.

### Cell remains in Homing

- Confirm X and Z commands are false, selecting the raised/lid-side defaults.
- Observe Moving X and Moving Z.
- If a Moving signal stays true, inspect obstruction and axis travel; code 201 should result after 5 s.
- If both Moving signals remain false, Homing should advance after 500 ms—even when endpoint position is not actually proven.

A stuck-false movement input can therefore create false Homing success rather than a fault. Add endpoint sensors for a stronger design.

### Cell faults with 101

- Confirm both emitters are enabled.
- Confirm each conveyor runs only until its own at-place input becomes true.
- Follow each part to the diffuse sensor.
- Verify `xLidAtPlace` and `xBaseAtPlace` are positive-logic detections.
- Check for excess parts or a part that has passed the sensor.

The timer measures continuous enabled time in `WaitingForParts`. Pause, simulation Stop or lost enable sends the cell back to `Stopped` rather than preserving elapsed time.

### Cell faults with 102

- Confirm each part is aligned in its positioning bar.
- Confirm both clamp outputs become true.
- Confirm each lane conveyor continues until its own clamp proof.
- Prove `xLidClamped` and `xBaseClamped` independently.

### Cell remains in GrippingLid or faults with 203

- Confirm `xGrabCmd` is true.
- Confirm the picker is lowered onto the lid.
- Observe `xItemDetected`; it must remain true continuously for 300 ms.
- Inspect lid alignment and ensure the lid clamp has not released prematurely.

The program proves pickup at this state only. It does not continuously fault if the lid is later lost during transfer.

### Lid clamp will not release

In `ReleasingLidClamp`, the lid-clamp command returns to its default false while base clamp, Grab and Z-down remain asserted. If `xLidClamped` does not become false within 5 s, code 204 results.

Check the clamp output, feedback polarity and whether the gripped lid is mechanically trapping the positioner.

### Axis state times out

For faults 202 and 205–209, verify the intended handshake:

```text
state entry → axis command → Moving TRUE → Moving FALSE
```

A pulse missed by the 20 ms task, a stuck-true signal or a command that causes no movement can all prevent completion. State 203 shares the five-second grab timeout; code 210 uses the same motion-timeout FB for the base-positioner limit.

### Base positioner does not release the product

- Confirm the base-clamp command has returned false.
- Confirm `xBasesPositionerRaiseCmd` is true in `ReleasingProduct`.
- Prove `xBasesPositionerAtLimit` at the raised position.
- Check for a misassembled product obstructing the bar.

The program does not explicitly prove base-clamp release before raising the positioner.

### Cell remains in WaitingForProductExit

This state runs the base conveyor and waits for a rising `xPartLeaving` edge. Inspect:

- base conveyor command;
- base-positioner raised command/position;
- part alignment and chute path;
- leaving-sensor false → true transition;
- whether the edge occurred before state 140.

The 10-second `fbProductExitTimeout` is called, but the supplied revision tests `fbMotionTimeout.Q` instead. Fault 301 is therefore unreachable and the state can wait forever. Correct the reference before relying on exit supervision.

### Counter does not match physical output

- Confirm the product physically crosses the leaving sensor.
- Verify one false → true transition per product.
- Check whether the sensor was already true before state 140.
- Check for Reset, which clears the count.
- Confirm `diProductCounterDisplay` matches the internal counter.

The count proves sensor detections, not verified lid/base attachment. A single base, separated lid or malformed product can still create an edge.

### Controlled Stop does not complete

Stop Pending clears only when `xProductCompletePulse` is true. That pulse exists for one scan after a leaving-sensor rising edge.

- Check whether the cell is still feeding or assembling a product.
- Check fault state and product-leaving input.
- Check the early-edge and broken-timeout conditions above.
- Confirm a new Start edge has not cancelled Stop Pending.

If Stop is requested while no product is active, the supplied revision can continue feeding and complete another product rather than stopping immediately at an empty boundary.

### Product stops at the leaving sensor

The completion pulse is created on sensor arrival, not clearance. A pending Stop can disable the base conveyor and finished-product remover on the next scan. If geometry or inertia does not carry the product clear, it may remain at or after the sensor.

The corrected stop design should require the sensor to clear and the outfeed to run for a proved clearance time.

### Pause or simulation Stop causes a sequence restart

Pause or `xFactoryIORunning = FALSE` removes enable, which resets any non-faulted state to `Stopped`. Run remains latched. When the simulation resumes, the cell can automatically enter Homing without a fresh Start.

Inspect and clear physical WIP before resume. If retained pause/resume is required, implement an explicit paused state and recovery policy.

### OPC items are Bad or Unknown

1. Confirm the Control Win runtime and application are running.
2. Rescan/reselect the target after moving the project to another PC.
3. Rebuild/download if Symbol Configuration changed.
4. Confirm Kepware browses `Application.FIO` under its configured device.
5. Confirm the channel/device names still form `Channel2.project11`.
6. Confirm Factory I/O uses `PTC.KepwareServer`.
7. Compare all item names, types and directions with the tag map.

## Communications limitation

The PLC does not supervise OPC quality, message age or a heartbeat. The target also has its task watchdog disabled, and the saved device stop/reset behaviour records `KeepCurrentValues`. Test actual outputs and client values during application Stop, runtime loss and OPC disconnection.

A static last-known value is not proof of healthy communication. Add a changing heartbeat, timeout, latched communications fault and documented safe-output policy before physical use.

## Safety note

The emergency-stop and Stop paths in this project are ordinary PLC logic exchanged through OPC DA in a simulation. They are not safety-rated. A physical implementation requires a risk assessment, suitable safeguarding and independently validated safety functions.

[Back to the project README](../README.md)

# Simulation Verification Record

## Test Summary

| Item | Detail |
| --- | --- |
| Application | `tankAndSortingPlant` |
| Environment | CODESYS V3.5 SP22 Patch 3 |
| Runtime | CODESYS Control Win V3 x64 simulation |
| Language | IEC 61131-3 Structured Text |
| Test method | Manual write-values commissioning with online monitoring |
| Build result | 0 errors, 0 warnings |
| Overall result | Core sequence and high-high fault/reset behaviour passed |

These tests verify the PLC logic in isolation. They do not yet verify Factory I/O geometry, OPC communication, real field wiring or safety hardware.

## Initial Test Conditions

Before starting the automatic sequence, the following feedback was established:

| Signal | Initial value | Meaning |
| --- | ---: | --- |
| `xPlantEnableTest` | `TRUE` | Plant modules enabled |
| `xAutoRunTest` | `FALSE` | No automatic transaction requested |
| `xPlantResetTest` | `FALSE` | Reset inactive |
| `xContainerAtFillTest` | `TRUE` | Empty container available at tank |
| `xTransferPathClearTest` | `TRUE` | Downstream path available |
| `rTankLevelPctTest` | `0.0` | Tank empty |
| `xTankLevelHighHighTest` | `FALSE` | Overfill switch healthy |
| `xCraneAtHomeTest` | `TRUE` | Crane at position 55 |
| `xForkAtMiddleTest` | `TRUE` | Fork carriage centred |
| Other fork/motion/load feedback | `FALSE` | No movement or product initially detected |

Expected stable conditions were observed: coordinator stopped, tank idle/ready, storage idle/ready, and all 54 rack positions free.

## Functional Acceptance Tests

| ID | Test | Expected result | Observed result | Status |
| --- | --- | --- | --- | --- |
| FAT-01 | Enable automatic operation | Coordinator reserves slot 1 and requests a tank batch | Slot 1 became `Reserved`; tank entered `Filling` and fill valve opened | Pass |
| FAT-02 | Increase tank level to `80.0%` | Fill valve closes and tank enters `BatchReady` | State and output matched expectation | Pass |
| FAT-03 | Maintain container, path and storage permissives | Coordinator permits discharge | Coordinator entered `DischargingBatch`; tank entered `Discharging`; discharge valve opened | Pass |
| FAT-04 | Reduce tank level to `5.0%` | Tank confirms discharge, closes valve and returns idle | Completion handshake was observed and storage entered `ReceivingProduct` | Pass |
| FAT-05 | Leave auto selected after discharge | Tank may prepare batch 2 while storage handles product 1 | Concurrent refill behaviour was observed | Pass |
| FAT-06 | Simulate product arrival and crane feedback in sequence | Storage progresses through load, lift, retract, travel, rack deposit and return-home states | All expected storage states and actuator commands were observed | Pass |
| FAT-07 | Apply X/Z movement feedback active, then inactive | Travel completes only after motion has actually been observed | Storage remained in travel until the active-to-inactive sequence occurred | Pass |
| FAT-08 | Confirm fork retraction after deposit | Rack inventory commits only after physical deposit confirmation | Slot 1 changed from `Reserved` to `Occupied`; counts became 53 free, 1 occupied, 0 reserved | Pass |
| FAT-09 | Remove auto during the active transaction | Current product finishes; no new transaction starts | Product 1 completed and crane returned home; prepared batch 2 remained safely at `BatchReady` | Pass |
| FAT-10 | Activate tank high-high input | Tank and coordinator fault; both tank valves close immediately | Fault propagation and safe outputs matched expectation | Pass |
| FAT-11 | Clear high-high, keep auto off and apply reset | Faults clear without restarting production or erasing inventory | Modules recovered; occupied slot count remained unchanged | Pass |

## Storage Sequence Evidence

The following ordered state progression was exercised manually:

1. `ReceivingProduct`
2. `ExtendingToLoad`
3. `LiftingProduct`
4. `RetractingWithProduct`
5. `MovingToRack`
6. `ExtendingToRack`
7. `LoweringProduct`
8. `RetractingAfterDeposit`
9. `ReturningHome`
10. `Complete`
11. `Idle`

The movement test deliberately set `xMovingXTest` or `xMovingZTest` true before returning it false. This confirmed that the `xMotionStarted` guard prevents a stationary crane from being mistaken for one that has completed a commanded move.

## Final Stable Condition

After completing one product transaction and performing the fault/reset test, the following behaviour was confirmed:

- storage returned home with forks centred;
- all Boolean movement commands were false;
- rack position 1 remained occupied;
- no reservation remained active;
- the next prepared tank batch could remain parked safely at `BatchReady`;
- automatic production did not restart while `xAutoRunTest` was false.

## Test Notes

- The current `T#30M` timeouts are intentionally long so a person can change feedback values one step at a time without creating nuisance timeouts.
- Earlier short timeout settings correctly produced latched timeout faults during manual commissioning. Production values still require formal tuning when movement is automated.
- Writing `xAutoRunTest := TRUE` after recovery correctly begins another production transaction. This is expected behaviour, not an unintended restart.
- A clean simulation restart may reinitialize rack data because persistent inventory has not yet been implemented.

## Remaining Verification

| Area | Required test |
| --- | --- |
| Factory I/O | Verify every sensor/actuator tag, polarity, scale and mechanical sequence |
| Kepware | Verify OPC quality, read/write direction, update rate and recovery after disconnect |
| Timing | Measure worst-case fill, discharge and crane steps; replace commissioning timeouts |
| Capacity | Fill all 54 positions and prove rack-full blocks new work without faulting |
| Recovery | Define and test reservation reconciliation after interruption/power loss |
| Persistence | Decide whether inventory is retained, reconstructed or operator-confirmed after restart |
| Extended faults | Deliberately prove each fill, discharge, storage-step and invalid-operation fault |
| HMI | Verify command permissions, indication, alarm acknowledgement and reset workflow |
| Safety | Outside simulation scope; requires a separate risk-based safety validation |

## Suggested Evidence for the Portfolio

When Factory I/O and Kepware are connected, capture:

1. CODESYS online view showing the coordinator and module states.
2. Kepware tag quality with live values.
3. Factory I/O tank filling and high-high trip response.
4. Container transfer and complete crane storage cycle.
5. Rack count before reservation, during reservation and after deposit commit.
6. Controlled stop with a prepared batch waiting safely.
7. A short end-to-end video placed in the `Media` folder.

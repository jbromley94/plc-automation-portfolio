# Logical I/O Schedule

## Scope

The current project has no final hardware or Factory I/O addresses. `PLC_PRG` uses temporary commissioning variables so the complete sequence can be exercised directly in CODESYS.

This schedule records the logical interface now. During Factory I/O and Kepware integration, each commissioning variable will be replaced or mapped to an external tag while the function-block interfaces remain unchanged.

## Plant Controls and Permissives

| Current signal | Type | Direction | Meaning | Future source |
| --- | --- | --- | --- | --- |
| `xPlantEnableTest` | `BOOL` | Input | Enables the coordinated plant modules | Operator/HMI enable |
| `xAutoRunTest` | `BOOL` | Input | Requests continuous automatic production | Operator/HMI auto command |
| `xPlantResetTest` | `BOOL` | Input | Requests fault reset while auto is off | Reset pushbutton/HMI |
| `xContainerAtFillTest` | `BOOL` | Input | Confirms a container is correctly positioned below the tank | Fill-position photoelectric sensor |
| `xTransferPathClearTest` | `BOOL` | Input | Confirms downstream transfer can accept a filled container | Conveyor/buffer clear logic |

## Tank Process I/O

### Feedback into the PLC

| Current signal | Type | Meaning | Future source |
| --- | --- | --- | --- |
| `rTankLevelPctTest` | `REAL` | Scaled tank level from `0.0` to `100.0` percent | Factory I/O analog level transmitter |
| `xTankLevelHighHighTest` | `BOOL` | Independent overfill trip input | High-high level switch |

### Commands from the PLC

| Module signal | Type | Meaning | Future destination |
| --- | --- | --- | --- |
| `fbTank.xFillValveCmd` | `BOOL` | Opens the tank inlet/fill valve | Factory I/O fill valve |
| `fbTank.xDischargeValveCmd` | `BOOL` | Opens the tank outlet/discharge valve | Factory I/O discharge valve |

### Configured tank values

| Setting | Current value | Purpose |
| --- | --- | --- |
| `rFillSetpointPct` | `80.0` | Routine level at which filling stops and the batch becomes ready |
| `rEmptySetpointPct` | `5.0` | Level at or below which discharge is considered complete |
| `tFillTimeout` | `T#30M` | Manual commissioning timeout; requires tuning during integration |
| `tDischargeTimeout` | `T#30M` | Manual commissioning timeout; requires tuning during integration |

The analog setpoint controls normal filling. `xTankLevelHighHighTest` is a separate trip and should remain independent when external I/O is added.

## Storage and Crane I/O

### Feedback into the PLC

| Current signal | Type | Meaning | Future source |
| --- | --- | --- | --- |
| `xCraneAtHomeTest` | `BOOL` | Crane is at target/home position `55` | Home position confirmation |
| `xForkAtLeftTest` | `BOOL` | Fork carriage is fully extended toward the loading side | Left limit sensor |
| `xForkAtMiddleTest` | `BOOL` | Fork carriage is retracted/centred for travel | Middle limit sensor |
| `xForkAtRightTest` | `BOOL` | Fork carriage is fully extended into the rack | Right limit sensor |
| `xMovingXTest` | `BOOL` | Crane reports horizontal travel active | Factory I/O crane feedback |
| `xMovingZTest` | `BOOL` | Crane reports vertical travel active | Factory I/O crane feedback |
| `xProductAtLoadTest` | `BOOL` | Product has reached the crane loading position | Load-position photoelectric sensor |

### Commands from the PLC

| Module signal | Type | Meaning | Future destination |
| --- | --- | --- | --- |
| `fbStorage.xEntryConveyorCmd` | `BOOL` | Runs the entry conveyor toward the crane | Entry conveyor motor |
| `fbStorage.xLoadConveyorCmd` | `BOOL` | Runs the conveyor at the crane loading point | Load conveyor motor |
| `fbStorage.xForksLeftCmd` | `BOOL` | Extends the forks toward the product-loading side | Fork left command |
| `fbStorage.xForksRightCmd` | `BOOL` | Extends the forks toward the storage rack | Fork right command |
| `fbStorage.xLiftCmd` | `BOOL` | Holds the product at the lifted travel height | Lift command |
| `fbStorage.wTargetPositionCmd` | `WORD` | Selects rack target `1..54` or home target `55` | Crane target-position command |

### Configured storage values

| Setting | Current value | Purpose |
| --- | --- | --- |
| `wHomePosition` | `55` | Position used for the crane home/loading location |
| `tLiftTime` | `T#1S` | Simulated time to lift a product clear of its support |
| `tLowerTime` | `T#1S` | Simulated time to lower a product into its rack position |
| `tStepTimeout` | `T#30M` | Manual commissioning timeout; requires tuning during integration |

## Coordinator-to-Module Commands

| Coordinator output | Receiving input | Purpose |
| --- | --- | --- |
| `xModuleEnable` | All module `xEnable` inputs | Common equipment enable |
| `xModuleReset` | All module `xReset` inputs | Common coordinated reset |
| `xRackReserveRequest` | `FB_RackManager.xReserveRequest` | Allocate the next free rack slot |
| `xRackCommitRequest` | `FB_RackManager.xCommitRequest` | Mark a deposited product's slot occupied |
| `xRackReleaseRequest` | `FB_RackManager.xReleaseRequest` | Return an unused reservation to free state |
| `xTankStartBatchReq` | `FB_Tank.xStartBatchReq` | Start preparing a batch |
| `xTankDischargePermit` | `FB_Tank.xDischargePermit` | Permit discharge into a correctly positioned container |
| `xStorageStoreRequest` | `FB_Storage.xStoreRequest` | Receive and store one product using the active reservation |

## Module Status Returned to the Coordinator

| Module | Status | Purpose |
| --- | --- | --- |
| Tank | `xReady` | Tank is enabled, healthy, idle and able to accept a command |
| Tank | `xBatchReady` | Fill setpoint has been reached and the batch is waiting |
| Tank | `xDischargeComplete` | Low-level condition was reached during the permitted discharge |
| Tank | `xFaulted` | One or more tank faults are latched |
| Storage | `xReady` | Crane is enabled, healthy and idle at home |
| Storage | `xDepositComplete` | Product has been deposited and forks retracted; inventory may commit |
| Storage | `xStoreComplete` | Deposit is complete and the crane has returned home |
| Storage | `xFaulted` | Reservation or movement timeout fault is latched |
| Rack manager | `xReserveAck` | Reservation request has completed |
| Rack manager | `xCommitAck` | Occupancy commit has completed |
| Rack manager | `xReleaseAck` | Reservation release has completed |
| Rack manager | `xReservationValid` | `wReservedPosition` identifies the active reservation |
| Rack manager | `wReservedPosition` | Active rack destination from `1..54` |
| Rack manager | `xRackFull` | No free rack positions remain |
| Rack manager | `xFaulted` | An invalid or conflicting inventory operation occurred |

## Diagnostic Signals

| Module | Signals |
| --- | --- |
| Coordinator | `eState`, `xFaulted`, `xRunning`, `xSequenceFault`, `xWaitingForContainer`, `xWaitingForStorage` |
| Tank | `eState`, `xBusy`, `xTankEmpty`, `xHighHighFault`, `xFillTimeoutFault`, `xDischargeTimeoutFault` |
| Storage | `eState`, `xBusy`, `xReservationFault`, `xStepTimeoutFault` |
| Rack manager | `aSlotState[1..54]`, `uiFreeCount`, `uiReservedCount`, `uiOccupiedCount`, `uiUnknownCount`, `xInvalidOperationFault` |

These values are suitable for future HMI, Kepware and troubleshooting views. They are status/diagnostic tags rather than physical outputs.

## Mapping Rules for the Next Stage

1. Confirm each Factory I/O tag's data type and safe state before mapping it.
2. Keep simulation tag names separate from module interfaces; change the mapping layer, not the equipment logic.
3. Scale the analog tank level once at the boundary and pass percentage into `FB_Tank`.
4. Do not use the analog fill setpoint as the high-high trip; retain a separate Boolean safety/process switch.
5. Verify actuator polarity, especially if any field output is fail-open or active-low.
6. Tune timeouts from measured worst-case motion times plus an engineering margin.
7. Test every feedback signal independently before starting the automatic sequence.

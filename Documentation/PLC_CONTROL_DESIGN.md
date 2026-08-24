# PLC Control Design

## Export profile

The supplied CODESYS export is readable XML created with **CODESYS V3.5 SP22 Patch 3**. The configured target is **CODESYS Control Win V3 x64 3.5.22.30**.

`MainTask` is cyclic, priority 1, with a 20 ms interval. It calls `PLC_PRG`.

## Source inventory

| Type | Count | Contents |
|---|---:|---|
| Program | 1 | `PLC_PRG` |
| Global variable lists | 3 | `FIO`, `PlantConstants`, `SCADA` |
| Function blocks | 15 | Equipment, transfer, operator and plant controllers |
| Qualified enums | 14 | Explicit equipment, phase, plant and inventory states |

### Function-block responsibilities

| Function block | Responsibility |
|---|---|
| `FB_MachiningCell` | Controls one machining-centre lane |
| `FB_TwinCellUnit` | Coordinates the lid and base machining cells |
| `FB_PairDiverter` | Alternates/routes paired parts through the pusher zone |
| `FB_TransferCoordinator` | Reusable request/ready/permit/ack transfer handshake |
| `FB_AssemblerCell` | Executes the X/Z clamp, pick, place and release sequence |
| `FB_AssemblyUnit` | Wraps assembly-cell status and boundary control |
| `FB_PackoutLoader` | Transfers assembled product to a carrier |
| `FB_PackoutCarrierStation` | Emits, positions and transfers carriers |
| `FB_Tank` | Controls filling, batch-ready and discharge states |
| `FB_Storage` | Moves a carrier through the crane deposit sequence |
| `FB_RackManager` | Reserves, commits and counts 54 logical slots |
| `FB_BatchStorageUnit` | Coordinates tank, storage and inventory as one unit |
| `FB_MasterOperatorControl` | Master mode, run latch, start/reset edges and controlled stop |
| `FB_UnitOperatorControl` | Reusable local/master unit boundary |
| `FB_PlantCoordinator` | Overall plant state and production-phase sequencing |

## State-machine model

Every important machine exposes a qualified enum instead of unexplained integers in the application code. Examples include:

- `E_PlantState`: Stopped, Ready, Starting, Running, Stopping, Faulted and RecoveryRequired.
- `E_ProductionPhase`: Idle, ReservingStorage, ProducingPair, Assembling, WaitingForPackout, Batching, Storing and Complete.
- Detailed equipment states for machining, diverter, assembler, packout loader/carrier, tank and storage.
- `E_RackSlotState`: Free, Occupied, Reserved and Unknown.

The state machines generally follow four useful rules:

1. Set non-active commands to safe defaults before evaluating the current state.
2. Advance only when the required feedback and handshake conditions are true.
3. Use timeouts and specific module fault codes for stalled operations.
4. Treat an unexpected enum value as a fault rather than silently continuing.

## Input normalisation

The `FIO` GVL is a symbolic process interface; it is not a direct `%I`/`%Q` hardware map. Raw scene conventions are normalised near the top of `PLC_PRG`.

```iecst
rTankLevelPct := FIO.rTankLevelV * 10.0;
xPackoutPalletAtLoad := NOT FIO.xPackoutPalletAtLoadBeamClear;
```

This prevents raw device naming or active-low sensor semantics from leaking throughout the control modules.

## Safety, fault and stop behaviour

The three simulated emergency-stop channels are combined explicitly:

```iecst
xAllEmergencyStopsHealthy :=
    FIO.xMasterEmergencyStopHealthy
    AND FIO.xPlantEmergencyStopHealthy
    AND FIO.xTwinCellEmergencyStopHealthy;
```

Subsystem faults are aggregated into `xAnyFault` before the plant coordinator is executed.

The master control block uses:

- `R_TRIG` instances for Start and Reset;
- XOR validation so exactly one mode is selected;
- an automatic run latch;
- reset-required behaviour following a safety loss or fault;
- a controlled-stop request rather than an abrupt mid-transaction stop.

When Auto is removed during a production transaction, the plant enters stop-pending behaviour and finishes the current transaction before returning to stopped.

## Transfer handshakes

The pair-transfer and product-transfer boundaries use separate instances of `FB_TransferCoordinator`.

The handshake vocabulary is deliberately visible to the HMI:

| Signal | Meaning |
|---|---|
| Request | Upstream wants to transfer material |
| Ready | Downstream is available |
| Permit | Transfer is authorised |
| Ack | Downstream has accepted the transaction |
| Complete | Physical transfer has completed |

Keeping these signals explicit makes commissioning easier than hiding every boundary inside one large state machine.

## Rack reservation and commit

`FB_RackManager` contains:

```iecst
aSlotState : ARRAY[1..54] OF E_RackSlotState;
```

Its transaction model is a strong part of the project:

1. A reserve request scans from slot 1 to 54 and marks the first free slot `Reserved`.
2. The storage sequence receives that target and performs the physical deposit.
3. Only a later commit request changes `Reserved` to `Occupied`.
4. If recovery begins after a deposit may have happened, the slot becomes `Unknown` rather than incorrectly `Free`.
5. Free, occupied, reserved and unknown totals are recalculated every scan.

Ordinary control reset does not erase occupied inventory.

## Analogue tank control

The project includes continuous values in both directions:

| Signal | Direction |
|---|---|
| `rTankLevelV` | Factory I/O to PLC |
| `rTankFlowV` | Factory I/O to PLC |
| `rTankFillValveCmdV` | PLC to Factory I/O |
| `rTankDischargeValveCmdV` | PLC to Factory I/O |

Normal fill control therefore uses analogue feedback and commands, while a separate `xTankLevelHighHigh` Boolean provides an independent overfill condition.

## SCADA command contract

The `SCADA` GVL distinguishes maintained and momentary writes.

Maintained:

```iecst
xIgnitionAutoModeReq : BOOL;
```

Momentary:

```iecst
xResetReq : BOOL;
xStartReq : BOOL;
xStopReq  : BOOL;
```

After all consuming FBs run, the program clears the momentary values:

```iecst
SCADA.xResetReq := FALSE;
SCADA.xStartReq := FALSE;
SCADA.xStopReq := FALSE;
```

This is why an Ignition One-Shot Button is appropriate for Start, Stop and Reset, while the Auto switch uses a bidirectional maintained binding.

## Why the replacement Auto symbol worked

The export contains both `SCADA.xAutoModeSelected` and `SCADA.xIgnitionAutoModeReq`. The old symbol was present but unused by executable logic and exported with current access `1`; the replacement has read/write access `3` and is consumed by `PLC_PRG`.

The working merge is:

```iecst
xAutoModeSelected :=
    FIO.xMasterAutoModeSelected
    OR SCADA.xIgnitionAutoModeReq;
```

The earlier write failure was therefore not caused by the three physical control panels. The old symbol is now a confusing legacy declaration and should be removed or explicitly deprecated in a later cleanup.

## Design constraints

- Numeric fault codes are module-specific, not globally unique identifiers.
- `FIO` is symbolic and externally mapped; it should not be described as direct hardware addressing.
- The watchdog is disabled in this simulation export.
- Rack inventory is not retained across PLC reinitialisation; production hardening would require retention or startup reconciliation.

See [Review and Next Steps](REVIEW_AND_NEXT_STEPS.md) for the full prioritised backlog.

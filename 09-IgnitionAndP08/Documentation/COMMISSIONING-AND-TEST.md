# Commissioning and Test Plan

## Preconditions

- CODESYS Control Win V3 x64 is installed and running.
- CODESYS application builds without errors and is online in Run.
- Kepware channel/device is `Project8.SCADATankAndSortingPlant`.
- Kepware values under `Application.FIO` and `Application.SCADA` show Good quality.
- Factory I/O uses the corrected scene and its OPC client shows connected.
- Ignition OPC connection `SCADA` is connected.
- Ignition provider `Sample_Tags` contains the supplied core tags with Good quality.

## Recommended startup order

1. Start CODESYS Control Win.
2. Open/import the CODESYS project, log in, download if required, and Run.
3. Start Kepware and verify fresh values/timestamps in its OPC Quick Client.
4. Open the corrected Factory I/O scene and connect the driver.
5. Start the Factory I/O simulation.
6. Open the Ignition project/session.
7. Confirm the E-stop and Stop circuits are healthy.
8. Select Auto on the Factory I/O local panel.
9. Press Reset to acknowledge the startup/reset-required condition.
10. Press Start.

## Core acceptance test

### 1. Communications

- [ ] Factory I/O driver connected to `PTC.KepwareServer`
- [ ] Ignition `SCADA` OPC connection connected
- [ ] Perspective footer reports `OPC DATA GOOD`
- [ ] `xFactoryIORunning = TRUE`

### 2. Operator control

- [ ] Local Auto selection is visible in Perspective
- [ ] Reset clears `xResetRequired`
- [ ] Start latches automatic operation
- [ ] Stop prevents a new transaction; an active transaction completes before the plant enters `Stopped`
- [ ] E-stop removes all process/motion commands and requires acknowledgement

### 3. Tank fill

During `E_TankState.Filling`, verify:

| Value | Expected |
|---|---:|
| `FIO.xFillValveCmd` | `TRUE` |
| `FIO.rFillValveCmdV` | about `10.0 V` |
| `FIO.rTankLevelV` | increasing |
| `FIO.rTankLevelPct` | `rTankLevelV × 10` |

- [ ] Physical Factory I/O tank visibly fills
- [ ] Perspective tank rises smoothly
- [ ] At approximately `8.0 V / 80%`, state becomes `BatchReady`
- [ ] Fill command returns to `0.0 V`

### 4. Batch discharge

- [ ] Empty container is emitted and reaches the fill position
- [ ] Discharge starts only after the coordinator confirms the required container/path/storage conditions
- [ ] Tank level falls
- [ ] At or below `5%`, discharge completes and valve closes

The entry checks are transactional rather than continuously re-interlocked: after the coordinator enters `DischargingBatch`, its discharge permit remains asserted until the batch completes or a higher-level fault/recovery condition removes outputs.

### 5. Storage transaction

- [ ] A free rack slot is reserved before production proceeds
- [ ] Product transfers to the storage load point
- [ ] Crane completes load, move, deposit, retract, and home sequence
- [ ] Reserved count returns to zero after commit
- [ ] Occupied count increments by one
- [ ] Free count decrements by one

### 6. Stop and restart

- [ ] Normal Stop prevents another transaction and allows an active transaction to finish before entering `Stopped`
- [ ] Restart after a normal Stop requires a new Start edge; Reset is needed only after a reset-required or fault condition
- [ ] Interrupted material is not silently assigned to a free rack slot

## Tank troubleshooting matrix

| Fill command | Raw level | Percentage | Interpretation |
|---:|---:|---:|---|
| `0.0 V` | `0.0 V` | `0%` | PLC output interlock active or no fill command requested |
| `10.0 V` | `0.0 V` | `0%` | Factory I/O valve/level mapping or physical scene issue |
| `10.0 V` | Rising | Rising ×10 | Analogue chain is functioning |
| Any | Rising | `0%` | PLC percentage calculation/publication issue |
| Correct in PLC | Correct in PLC | Bad/null in Ignition | Ignition tag path, provider, or OPC connection issue |

If `rFillValveCmdV` is unexpectedly zero, check:

```text
xEmergencyActive
xResetRequired
xPlantFaulted
xRecoveryRequired
xPlantEnable
```

If `rTankLevelV` stays zero, inspect the Factory I/O mapping and confirm it uses **Tank 0 (Level Meter)**, not the flow meter.

## Fault-injection tests

- [ ] Hold/force the tank feedback below setpoint and confirm fill timeout after two minutes
- [ ] Activate high-high and confirm immediate latched tank fault and closed valves
- [ ] Interrupt an active transaction and confirm recovery-required behaviour
- [ ] Block a storage motion step and confirm step timeout after two minutes
- [ ] Fill all rack slots and confirm rack-full blocks new production without reporting an equipment fault

Use Factory I/O’s failure/forcing features only for controlled testing and remove every force afterward.

## Evidence to capture for the portfolio

- Desktop responsive dashboard during filling and during storage
- Mobile dashboard view
- CODESYS online watch of the four tank analogue/Boolean signals
- Kepware Quick Client with Good values
- Factory I/O driver mapping for level and valve points
- Completed cycle with changed rack counts
- One controlled fault and successful recovery

# Factory I/O and OPC Mapping

## Canonical namespaces

Kepware channel/device prefix:

```text
Project8.SCADATankAndSortingPlant
```

`Project8` is the retained legacy channel name from the earlier project. Project 09 does not require it to be renamed; every consumer must simply use the same exact channel/device path and casing.

Published CODESYS namespaces:

```text
Project8.SCADATankAndSortingPlant.Application.FIO.<symbol>
Project8.SCADATankAndSortingPlant.Application.SCADA.<symbol>
```

Ignition paths:

```text
[Sample_Tags]FIO/<symbol>
[Sample_Tags]SCADA/<symbol>
```

Factory I/O uses `OPC Client DA/UA` with server `PTC.KepwareServer`. Ignition uses the OPC UA connection name `SCADA`.

## Physical inputs

| Factory I/O input | Address/type | CODESYS symbol | Status |
|---|---:|---|---|
| Start | BI 0 | `FIO.xStartPB` | Mapped |
| Stop | BI 1 | `FIO.xStopCircuitHealthy` | Mapped as healthy NC-style circuit |
| Reset | BI 2 | `FIO.xResetPB` | Mapped |
| Emergency stop | BI 3 | `FIO.xEmergencyStopHealthy` | Mapped as healthy circuit |
| Manual | BI 4 | — | Unused |
| At Entry | BI 5 | — | Unused |
| Load Beam Clear | BI 6 | `FIO.xLoadBeamClear` | Mapped |
| At Unload | BI 7 | — | Unused |
| At Exit | BI 8 | — | Unused |
| At Left | BI 9 | `FIO.xForkAtLeft` | Mapped |
| At Middle | BI 10 | `FIO.xForkAtMiddle` | Mapped |
| At Right | BI 11 | `FIO.xForkAtRight` | Mapped |
| Moving Z | BI 12 | `FIO.xMovingZ` | Mapped |
| Moving X | BI 13 | `FIO.xMovingX` | Mapped |
| Auto | BI 14 | `FIO.xAutoModeSelected` | Mapped |
| Container At Fill | BI 15 | `FIO.xContainerAtFill` | Mapped |
| Tank Level High High | BI 16 | `FIO.xTankLevelHighHigh` | Mapped |
| Tank 0 Level Meter | AI 0 | `FIO.rTankLevelV` | **Corrected mapping** |
| Tank 0 Flow Meter | AI 1 | — | Intentionally unmapped |
| Factory I/O Run | Internal | `FIO.xFactoryIORunning` | Mapped |

The flow meter can later be published as a separate `REAL`, for example `FIO.rTankFlowV`. It must not share `rTankLevelV`.

## Physical outputs

| CODESYS symbol | Factory I/O actuator | Address/type |
|---|---|---:|
| `FIO.xLoadConveyorCmd` | Load Conveyor | BO 0 |
| `FIO.xStartLightCmd` | Start light | BO 1 |
| `FIO.xStopLightCmd` | Stop light | BO 2 |
| `FIO.xEntryConveyorCmd` | Entry Conveyor | BO 3 |
| `FIO.xResetLightCmd` | Reset light | BO 4 |
| — | Exit Conveyor | BO 5, unused |
| `FIO.axRollerConveyorCmd[0]` | Roller Conveyor 0 | BO 6 |
| `FIO.axStackRedCmd[0]` | Stack Light 0 Red | BO 7 |
| — | Remover 1 | BO 8, unused |
| `FIO.axRollerConveyorCmd[1]` | Roller Conveyor 1 | BO 9 |
| `FIO.axRollerConveyorCmd[2]` | Roller Conveyor 2 | BO 10 |
| — | Unload Conveyor | BO 11, unused |
| `FIO.axRollerConveyorCmd[3]` | Roller Conveyor 3 | BO 12 |
| `FIO.xContainerEmitterCmd` | Emitter 0 | BO 13 |
| `FIO.axStackYellowCmd[0]` | Stack Light 0 Yellow | BO 14 |
| `FIO.xForksLeftCmd` | Forks Left | BO 15 |
| `FIO.xForksRightCmd` | Forks Right | BO 16 |
| `FIO.xLiftCmd` | Lift | BO 17 |
| `FIO.axStackGreenCmd[0]` | Stack Light 0 Green | BO 18 |
| `FIO.axStackRedCmd[1]` | Stack Light 1 Red | BO 19 |
| `FIO.axStackYellowCmd[1]` | Stack Light 1 Yellow | BO 20 |
| `FIO.axStackGreenCmd[1]` | Stack Light 1 Green | BO 21 |
| `FIO.axStackRedCmd[2]` | Stack Light 2 Red | BO 22 |
| `FIO.axStackYellowCmd[2]` | Stack Light 2 Yellow | BO 23 |
| `FIO.axStackGreenCmd[2]` | Stack Light 2 Green | BO 24 |
| `FIO.axStackRedCmd[3]` | Stack Light 3 Red | BO 25 |
| `FIO.axStackYellowCmd[3]` | Stack Light 3 Yellow | BO 26 |
| `FIO.axStackGreenCmd[3]` | Stack Light 3 Green | BO 27 |
| `FIO.rFillValveCmdV` | Tank 0 Fill Valve | AO 0 |
| `FIO.rDischargeValveCmdV` | Tank 0 Discharge Valve | AO 1 |
| `FIO.wTargetPositionCmd` | Crane Target Position | Integer output 0 |

Stack-light indices are:

```text
[0] infeed/emitter
[1] tank/filling position
[2] control panel
[3] outfeed/remover
```

## Public status and diagnostics

The `FIO` GVL also exposes PLC-derived values used by Ignition and commissioning:

- Plant/tank/storage states
- Tank level percentage
- Rack free, reserved, occupied, and unknown counts
- Reserved/target positions
- Plant, tank, storage, rack, timeout, high-high, sequence, and recovery flags
- Auto-run, enable, reset, reset-required, emergency-active, and readiness flags

The corrected Factory I/O driver contains 78 Project8 FIO items: 62 scalar variables plus 16 expanded array elements. The 22 stale `Channel2.Device1` items have been removed.

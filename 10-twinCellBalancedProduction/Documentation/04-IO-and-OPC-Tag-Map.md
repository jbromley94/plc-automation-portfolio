# I/O and OPC Tag Map

## Saved integration

| Property | Value |
| --- | --- |
| Factory I/O driver | OPC Client DA |
| OPC DA server / ProgID | `PTC.KepwareServer` |
| Saved item prefix | `Channel2.project10.Application.FIO.` |
| Saved mapped items | 53 of 100 maximum |
| Factory I/O component points | 45: 17 binary inputs, 2 integer inputs, 21 binary outputs and 5 integer outputs |
| Virtual simulation points | 8: Run, Pause, Reset and Time Scale status/commands plus Camera command |
| CODESYS external interface | Qualified GVL `FIO` |
| CODESYS symbol exposure | All 53 `FIO` variables selected; serialized access 3/3; export flag `SupportOPCUA=True` (no active OPC UA server/use is established by these files) |
| Direct IEC addresses | None; there are no `%I` or `%Q` declarations in the supplied export |

“Factory I/O → PLC” means the scene or simulator writes a value that the PLC reads. “PLC → Factory I/O” means the PLC writes a command or display value that the scene consumes.

## Authoritative saved OPC mapping

The table follows the item order stored in the latest Factory I/O scene. Prefix every tag with `Channel2.project10.Application.FIO.` when reproducing the saved Kepware path.

| Item | Direction | Type | `FIO` tag | Factory I/O point | Logic / use |
| ---: | --- | --- | --- | --- | --- |
| 0 | Factory I/O → PLC | BOOL | `xTwinCellStopPB` | Twin Cell Stop button, BI 15 | Normally closed; inverted in `PLC_PRG` |
| 1 | PLC → Factory I/O | BOOL | `xTwinCellStopLightCmd` | Stop-button lamp, BO 19 | On while stopped or Stop Pending |
| 2 | Factory I/O → PLC | BOOL | `xTwinCellStartPB` | Twin Cell Start button, BI 14 | Positive-logic momentary input |
| 3 | PLC → Factory I/O | BOOL | `xTwinCellStartLightCmd` | Start-button lamp, BO 18 | On only while enabled and not Stop Pending |
| 4 | Factory I/O → PLC | BOOL | `xTwinCellResetPB` | Twin Cell Reset button, BI 16 | Positive-logic momentary input |
| 5 | PLC → Factory I/O | BOOL | `xTwinCellResetLightCmd` | Reset-button lamp, BO 20 | On when Reset Required or plant faulted |
| 6 | Factory I/O → PLC | BOOL | `xTwinCellManualModeSelected` | Twin Cell Manual, BI 13 | Must be false for valid Auto; no manual sequence implemented |
| 7 | Factory I/O → PLC | BOOL | `xTwinCellEmergencyStop` | Twin Cell emergency stop, BI 11 | Healthy/released high; inverted to internal emergency active |
| 8 | Factory I/O → PLC | BOOL | `xTwinCellAutoModeSelected` | Twin Cell Auto, BI 12 | Must be true while Manual is false |
| 9 | Factory I/O → PLC | BOOL | `xTotalPartsAtRemoval` | Total parts at removal, BI 10 | Clear-beam high; inverted before edge counting |
| 10 | PLC → Factory I/O | BOOL | `xRemoverCmd` | Remover, BO 6 | Follows plant enable |
| 11 | PLC → Factory I/O | BOOL | `xLidsRawConveyorCmd` | Lids raw conveyor, BO 16 | State-controlled by lid cell |
| 12 | PLC → Factory I/O | BOOL | `xLidsExitConveyor2Cmd` | Lids exit conveyor 2, BO 14 | Follows plant enable |
| 13 | PLC → Factory I/O | BOOL | `xLidsExitConveyor1Cmd` | Lids exit conveyor 1, BO 5 | Follows plant enable |
| 14 | PLC → Factory I/O | BOOL | `xLidsEmitterCmd` | Lids emitter, BO 4 | Follows plant enable |
| 15 | PLC → Factory I/O | BOOL | `xLidsCenterStopCmd` | Lids centre Stop, BO 2 | Asserted when disabled/faulted, except during Reset |
| 16 | PLC → Factory I/O | BOOL | `xLidsCenterStartCmd` | Lids centre Start, BO 1 | Asserted in `WaitingForPickup` |
| 17 | PLC → Factory I/O | BOOL | `xLidsCenterResetCmd` | Lids centre Reset, BO 3 | Follows 500 ms PLC reset pulse |
| 18 | PLC → Factory I/O | BOOL | `xLidsCenterProduceLidsCmd` | Lids centre `produceLids`, BO 0 | Held true by lid-cell instance |
| 19 | Factory I/O → PLC | BOOL | `xLidsCenterOpened` | Machining Centre 1 Opened, BI 8 | Passed to cell FB but currently unused |
| 20 | Factory I/O → PLC | BOOL | `xLidsCenterHasError` | Lids centre Has Error, BI 1 | Direct fault input; produces code 1 |
| 21 | Factory I/O → PLC | BOOL | `xLidsCenterBusy` | Lids centre Busy, BI 0 | Confirms pickup/start |
| 22 | Factory I/O → PLC | BOOL | `xLidsAtExit` | Lids at exit, BI 6 | Clear-beam high; inverted before use |
| 23 | Factory I/O → PLC | BOOL | `xLidsAtEntry` | Lids at entry, BI 2 | Clear-beam high; inverted before use |
| 24 | Factory I/O → PLC | BOOL | `xFactoryIORunning` | Factory I/O Running status | Required for plant enable |
| 25 | PLC → Factory I/O | BOOL | `xFactoryIORunCmd` | Factory I/O Run command | Present but assigned false in V1 |
| 26 | PLC → Factory I/O | BOOL | `xFactoryIOResetCmd` | Factory I/O Reset command | 500 ms reset pulse only when recovery was required |
| 27 | Factory I/O → PLC | BOOL | `xFactoryIOReset` | Factory I/O Reset status | Combined with operator Reset sources |
| 28 | Factory I/O → PLC | BOOL | `xFactoryIOPaused` | Factory I/O Paused status | Removes plant enable without clearing Run latch |
| 29 | PLC → Factory I/O | BOOL | `xFactoryIOPauseCmd` | Factory I/O Pause command | Present but assigned false in V1 |
| 30 | PLC → Factory I/O | BOOL | `xExitConveyorCmd` | Shared exit conveyor, BO 13 | Follows plant enable |
| 31 | PLC → Factory I/O | BOOL | `xBasesRawConveyorCmd` | Bases raw conveyor, BO 17 | State-controlled by base cell |
| 32 | PLC → Factory I/O | BOOL | `xBasesExitConveyor2Cmd` | Bases exit conveyor 2, BO 15 | Follows plant enable |
| 33 | PLC → Factory I/O | BOOL | `xBasesExitConveyor1Cmd` | Bases exit conveyor 1, BO 12 | Follows plant enable |
| 34 | PLC → Factory I/O | BOOL | `xBasesEmitterCmd` | Bases emitter, BO 11 | Follows plant enable |
| 35 | PLC → Factory I/O | BOOL | `xBasesCenterStopCmd` | Bases centre Stop, BO 9 | Asserted when disabled/faulted, except during Reset |
| 36 | PLC → Factory I/O | BOOL | `xBasesCenterStartCmd` | Bases centre Start, BO 8 | Asserted in `WaitingForPickup` |
| 37 | PLC → Factory I/O | BOOL | `xBasesCenterResetCmd` | Bases centre Reset, BO 10 | Follows 500 ms PLC reset pulse |
| 38 | PLC → Factory I/O | BOOL | `xBasesCenterProduceLidsCmd` | Bases centre native `produceLids`, BO 7 | Held false by base-cell instance to select bases |
| 39 | Factory I/O → PLC | BOOL mapped from DINT | `xBasesCenterOpened` | **Machining Centre 2 Progress, II 1** | **Saved mapping defect; actual Opened BI 9 is unmapped** |
| 40 | Factory I/O → PLC | BOOL | `xBasesCenterHasError` | Bases centre Has Error, BI 4 | Direct fault input; produces code 1 |
| 41 | Factory I/O → PLC | BOOL | `xBasesCenterBusy` | Bases centre Busy, BI 3 | Confirms pickup/start |
| 42 | Factory I/O → PLC | BOOL | `xBasesAtExit` | Bases at exit, BI 7 | Clear-beam high; inverted before use |
| 43 | Factory I/O → PLC | BOOL | `xBasesAtEntry` | Bases at entry, BI 5 | Clear-beam high; inverted before use |
| 44 | Factory I/O → PLC | REAL | `rFactoryIOTimeScale` | Factory I/O Time Scale | Published but currently unused by control logic |
| 45 | PLC → Factory I/O | DINT | `diTwinCellLidsCounterDisplay` | Panel lids counter, IO 3 | Mirrors lid-cell count |
| 46 | PLC → Factory I/O | DINT | `diTwinCellBasesCounterDisplay` | Panel bases counter, IO 4 | Mirrors base-cell count |
| 47 | PLC → Factory I/O | DINT | `diTotalPartsCounterDisplay` | Total-parts counter, IO 2 | Mirrors final throughput count |
| 48 | PLC → Factory I/O | DINT | `diLidsCounterDisplay` | Lids lane counter, IO 0 | Mirrors lid-cell count |
| 49 | Factory I/O → PLC | DINT | `diLidsCenterProgress` | Machining Centre 1 Progress, II 0 | Passed to cell FB but currently unused |
| 50 | PLC → Factory I/O | DINT | `diFactoryIOCameraPositionCmd` | Factory I/O Camera command | Present but assigned 0 in V1 |
| 51 | PLC → Factory I/O | DINT | `diBasesCounterDisplay` | Bases lane counter, IO 1 | Mirrors base-cell count |
| 52 | Factory I/O → PLC | DINT | `diBasesCenterProgress` | Machining Centre 2 Progress, II 1 | Correct progress mapping; currently unused by cell logic |

## Signal polarity and normal state

| Signal family | Raw normal state | Program conversion | Internal meaning |
| --- | --- | --- | --- |
| Entry and exit retroreflective sensors | `TRUE` while beam is clear | `NOT FIO.x...` | `TRUE` while a part blocks the beam |
| Final removal sensor | `TRUE` while beam is clear | `xDetected := NOT FIO.xTotalPartsAtRemoval` | Rising detection edge when a part enters the beam |
| Emergency stop | `TRUE` while released/healthy | `xEmergencyStop := NOT FIO.xTwinCellEmergencyStop` | `TRUE` when emergency is active |
| Stop pushbutton | `TRUE` while healthy/not pressed | `xStopPB := ... OR NOT FIO.xTwinCellStopPB`, then `R_TRIG` | A false raw input creates one Stop request edge; it is not a maintained run permissive |
| Start and Reset pushbuttons | `FALSE` while released | Direct, then `R_TRIG` | One command per press |
| Busy and Has Error | `FALSE` when inactive | Direct | Positive-logic feedback |

The emergency-stop condition is level-interlocked, but the normally closed Stop signal is only edge-detected. A transition to false requests Stop once; if Reset later clears Stop Pending while the raw input remains false, a subsequent Start edge can latch Run. Treat a false or bad Stop signal as a commissioning hold point and correct the logic to include the healthy Stop level in the run permissive before any physical implementation. OPC value quality is not supervised separately.

## Confirmed mapping defect

The saved OPC item for `xBasesCenterOpened` is connected to the integer **Machining Centre 2 Progress** point, GUID `25f154b6-b59e-42e5-9f1d-da761139ac2d`. The correct binary point is:

| Property | Correct value |
| --- | --- |
| Point name | `Machining Center 2 (Opened)` |
| Type/address | Binary input 9 |
| GUID | `377c4656-24e1-4963-9dca-25054aef73e2` |
| Current OPC state | Unmapped |

`diBasesCenterProgress` correctly uses the progress GUID, so Items 39 and 52 currently target the same scene point. This does not affect the current sequence because `xOpened` and `diProgress` are unused inside `FB_MachiningCell`. Correct Item 39 before using door-open feedback for permissives or alarms.

## Source-direction observations

- `diTotalPartsCounterDisplay` is placed beneath the “inputs from Factory I/O” comment in the `FIO` declaration, but `PLC_PRG` writes it and the scene maps it to an integer display. Its actual direction is PLC → Factory I/O.
- The CODESYS implementation comment says the emitters and remover are forced on in the supplied scene. In the latest Factory I/O file, every `UseForcedValue` flag is false. The comment is stale; the mapped PLC commands are active.
- The latest scene contains no active open-circuit, short-circuit or forced-value simulation on any component point.
- All seven belt-conveyor encoders are disabled; there is no conveyor speed or distance feedback.

## Driver-profile caution

Factory I/O serializes settings for several drivers even when they are not selected. The non-OPC profiles in this scene contain older, incomplete maps—typically 9 binary inputs, 15 binary outputs and 2 numeric outputs. They omit later panel, emitter/remover, display, door/progress and simulator-control signals.

Do not switch from OPC Client DA and assume those profiles are equivalent. Rebuild and prove the complete mapping for any replacement protocol.

## Point-to-point proving checklist

Before automatic commissioning:

1. Confirm every required OPC item reports good quality.
2. Confirm the emergency-stop raw value is true when released and false when pressed.
3. Confirm the Stop raw value is true when released and false when pressed.
4. Confirm all five retroreflective signals are true with a clear beam and false with a part present.
5. Confirm each Start/Stop/Reset command reaches the intended machining centre only.
6. Confirm the lid `produceLids` output is true and the base `produceLids` output is false.
7. Jog or force each conveyor command individually during commissioning and verify physical scene direction.
8. Verify all five integer displays receive the intended counter.
9. Correct the base Opened mapping or explicitly document it as unused.
10. Remove every temporary force before returning to automatic operation.

[Back to the project README](../README.md)

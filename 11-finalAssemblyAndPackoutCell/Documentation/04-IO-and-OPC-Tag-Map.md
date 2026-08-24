# I/O and OPC Tag Map

## Saved communications path

```text
Factory I/O scene point
    ↕ OPC Client DA
PTC.KepwareServer
    ↕ Channel2.project11.Application.FIO.*
CODESYS FIO global variable list
    ↕ explicit PLC_PRG mapping
FB_OperatorControl / FB_AssemblerCell
```

| Property | Saved value |
| --- | --- |
| Selected Factory I/O driver | OPC Client DA |
| OPC server | `PTC.KepwareServer` |
| Namespace prefix | `Channel2.project11.Application.FIO.` |
| Items | 40 |
| Scene → PLC | 19: 18 BOOL and 1 REAL |
| PLC → scene | 21: 19 BOOL and 2 DINT |
| Physical mapped points | 32 |
| Virtual simulation points | 8 |

Each OPC item name is the namespace prefix above plus the listed PLC variable. The 40 saved display names match the 40 variables selected under the CODESYS `FIO` symbol group one-for-one.

## Complete saved OPC map

| Item | Direction | Type | PLC variable | Factory I/O point | Address / control use |
| ---: | --- | --- | --- | --- | --- |
| 0 | PLC → Factory I/O | BOOL | `xUnusedLidRemoverCmd` | Remover 2 | BO 6; lid-lane remover; follows automatic enable |
| 1 | Factory I/O → PLC | BOOL | `xStopPB` | Stop | BI 4; healthy/released high; inverted and edge-detected |
| 2 | PLC → Factory I/O | BOOL | `xStopLightCmd` | Stop light | BO 5; on while not running or Stop Pending |
| 3 | Factory I/O → PLC | BOOL | `xStartPB` | Start | BI 2; positive-logic momentary command |
| 4 | PLC → Factory I/O | BOOL | `xStartLightCmd` | Start light | BO 3; on while enabled and not Stop Pending |
| 5 | Factory I/O → PLC | BOOL | `xResetPB` | Reset | BI 3; positive-logic momentary command |
| 6 | PLC → Factory I/O | BOOL | `xResetLightCmd` | Reset light | BO 4; Reset Required or cell fault |
| 7 | Factory I/O → PLC | BOOL | `xPartLeaving` | Part leaving | BI 15; positive detection; rising-edge count/completion |
| 8 | Factory I/O → PLC | BOOL | `xMovingZ` | Moving Z | BI 1; axis-motion status |
| 9 | Factory I/O → PLC | BOOL | `xMovingX` | Moving X | BI 6; axis-motion status |
| 10 | PLC → Factory I/O | BOOL | `xMoveZCmd` | Move Z | BO 0; false=raised, true=lowered |
| 11 | PLC → Factory I/O | BOOL | `xMoveXCmd` | Move X | BO 1; false=lid/home, true=base side |
| 12 | Factory I/O → PLC | BOOL | `xManualModeSelected` | Manual selector | BI 9; must be false for Auto operation |
| 13 | PLC → Factory I/O | BOOL | `xLidsPositionerRaiseCmd` | Lid positioner raise | BO 18; mapped but PLC writes false |
| 14 | PLC → Factory I/O | BOOL | `xLidsEmitterCmd` | Lid emitter | BO 12; follows automatic enable |
| 15 | PLC → Factory I/O | BOOL | `xLidsConveyorCmd` | Lid conveyor | BO 11; state-controlled |
| 16 | Factory I/O → PLC | BOOL | `xLidClamped` | Lid clamped | BI 13; clamp proof |
| 17 | PLC → Factory I/O | BOOL | `xLidClampCmd` | Clamp lid | BO 19; state-controlled |
| 18 | Factory I/O → PLC | BOOL | `xLidAtPlace` | Lid at place | BI 8; diffuse sensor, detection high |
| 19 | Factory I/O → PLC | BOOL | `xItemDetected` | Gripper item detected | BI 7; must remain high for grab settle |
| 20 | PLC → Factory I/O | BOOL | `xGrabCmd` | Grab | BO 2; vacuum/gripper command |
| 21 | PLC → Factory I/O | BOOL | `xFinishedProductRemoverCmd` | Remover 1 | BO 14; base-lane finished-product remover |
| 22 | Factory I/O → PLC | BOOL | `xFactoryIORunning` | Factory I/O Running | Virtual; required for automatic enable |
| 23 | PLC → Factory I/O | BOOL | `xFactoryIORunCmd` | Factory I/O Run | Virtual; PLC writes false |
| 24 | PLC → Factory I/O | BOOL | `xFactoryIOResetCmd` | Factory I/O Reset | Virtual; one-scan pulse only after abnormal recovery |
| 25 | Factory I/O → PLC | BOOL | `xFactoryIOReset` | Factory I/O Reset | Virtual; combined with Reset sources |
| 26 | Factory I/O → PLC | BOOL | `xFactoryIOPaused` | Factory I/O Paused | Virtual; removes enable but retains Run latch |
| 27 | PLC → Factory I/O | BOOL | `xFactoryIOPauseCmd` | Factory I/O Pause | Virtual; PLC writes false |
| 28 | Factory I/O → PLC | BOOL | `xEmergencyStop` | Emergency-stop contact | BI 5; healthy/released high; inverted in PLC |
| 29 | PLC → Factory I/O | BOOL | `xBasesPositionerRaiseCmd` | Base positioner raise | BO 16; asserted during product release/outfeed |
| 30 | Factory I/O → PLC | BOOL | `xBasesPositionerAtLimit` | Base positioner at limit | BI 11; raised-position proof |
| 31 | PLC → Factory I/O | BOOL | `xBasesEmitterCmd` | Base emitter | BO 15; follows automatic enable |
| 32 | PLC → Factory I/O | BOOL | `xBasesConveyorCmd` | Base conveyor | BO 13; state-controlled |
| 33 | Factory I/O → PLC | BOOL | `xBaseClamped` | Base clamped | BI 12; clamp proof |
| 34 | PLC → Factory I/O | BOOL | `xBaseClampCmd` | Clamp base | BO 17; held through pickup/placement |
| 35 | Factory I/O → PLC | BOOL | `xBaseAtPlace` | Base at place | BI 0; diffuse sensor, detection high |
| 36 | Factory I/O → PLC | BOOL | `xAutoModeSelected` | Auto selector | BI 14; must be true while Manual is false |
| 37 | Factory I/O → PLC | REAL | `rFactoryIOTimeScale` | Factory I/O Time Scale | Virtual; published but unused by PLC logic |
| 38 | PLC → Factory I/O | DINT | `diProductCounterDisplay` | Product counter display | Integer output 0; mirrors completed count |
| 39 | PLC → Factory I/O | DINT | `diFactoryIOCameraPositionCmd` | Factory I/O Camera Position | Virtual; PLC writes 0 |

## Signal polarity and normal state

| Signal | Raw normal/active state | Internal treatment |
| --- | --- | --- |
| Emergency stop | True while released/healthy | Inverted into active emergency condition; level-interlocked |
| Stop pushbutton | True while released/healthy | Inverted into a Stop command, then passed through `R_TRIG` |
| Start and Reset | False while released; true while pressed | Direct positive logic, then edge-detected |
| At-place sensors | True while an object is detected | Direct positive logic |
| Part leaving | True while an object is detected | Rising edge gives one count/completion pulse |
| Clamp proofs | True when clamped | Direct positive logic |
| Moving X/Z | True while the respective axis moves | Must be observed true then false for most transitions |
| Item detected | True while the gripper detects the lid | Must remain true for the 300 ms grab dwell |

The normally closed Stop input is not a maintained permissive. After its false transition has generated one edge, Reset can clear Stop Pending and a later Start can latch Run while the raw Stop value remains false. The [engineering review](07-Engineering-Review.md) treats this as a confirmed control limitation.

## Physical-point coverage

The scene defines 39 physical I/O points:

| Point type | Defined | Mapped to OPC |
| --- | ---: | ---: |
| Binary inputs | 18 | 15 |
| Binary outputs | 20 | 16 |
| Integer outputs | 1 | 1 |
| Total | 39 | 32 |

Seven physical points are not present in the OPC map:

| Equipment | Unmapped points | Reason / current status |
| --- | --- | --- |
| Gantry arm rotation | BI 16, BO 7 and BO 8 | Reserved rotation function not used by the sequence |
| Gantry gripper rotation | BI 17, BO 9 and BO 10 | Reserved rotation function not used by the sequence |
| Lid positioner | At-limit BI 10 | PLC keeps lid-positioner raise false and uses lid-clamped feedback |

## Mapping audit

The latest saved map has:

- no duplicate OPC display names;
- no dangling scene-point keys;
- no wrong-direction bindings;
- matching Factory I/O point kinds and CODESYS data types;
- all 40 selected `FIO` variables represented exactly once.

The older protocol maps retained inside the scene expose fewer points and should not be treated as commissioned alternatives. Rebuild and prove any migration away from the selected OPC Client DA configuration.

## Saved force and fault flags

Across all 39 physical points:

- `UseForcedValue = False`
- `OpenCircuit = False`
- `ShortCircuit = False`

Five outputs retain a stored `ForcedValue = True` value—both emitters, the base conveyor and both removers—but those values are inactive because `UseForcedValue` is false. PLC commands are therefore effective in the supplied scene.

## Symbol-access observations

All 40 `FIO` variables are selected with access level 3/3 in CODESYS. Logical direction is established by the program and this document, not enforced by symbol permissions. Internal state, cell fault code, Run, Stop Pending, Ready and Busy are not included in the selected symbol set.

## Point-to-point proving order

1. Connect the PLC runtime to Kepware and confirm the `Application.FIO` symbols browse correctly.
2. Connect Factory I/O to `PTC.KepwareServer` and verify good quality.
3. Prove emergency-stop and Stop healthy-high polarity before enabling Auto.
4. Prove Start, Reset, Auto and Manual individually.
5. Prove both at-place sensors and both clamp proofs.
6. Prove Moving X, Moving Z and Item Detected while exercising the gantry through a controlled virtual test.
7. Prove the leaving sensor false → true → false and confirm exactly one counter increment.
8. Prove each state-controlled output and then both removers/emitters.
9. Remove every temporary force before automatic testing.

[Back to the project README](../README.md)

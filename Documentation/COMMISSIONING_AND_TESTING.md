# Commissioning and Testing

This document is a repeatable demonstration and regression checklist for the simulated system.

## 1. Prerequisites

- CODESYS Development System compatible with V3.5 SP22 Patch 3.
- CODESYS Control Win V3 x64 runtime.
- Factory I/O compatible with scene version 2.5.10.
- Kepware server with the Project 12 CODESYS and OPC routes configured.
- Ignition Gateway with Perspective, the `[Sample_Tags]P12/SCADA` tags and tank history enabled.

## 2. Cold-start order

1. Start the CODESYS runtime.
2. Open the CODESYS project, log in and run the application.
3. Start Kepware and confirm the CODESYS channel/device is connected.
4. Open the Factory I/O scene.
5. Select its Kepware OPC DA driver and connect.
6. Open the Ignition Gateway and confirm the Kepware OPC UA connection.
7. Open the Perspective session.

The exact application order is less important than confirming each boundary before issuing machine commands.

## 3. Communications checks

| Check | Expected result | Result |
|---|---|---|
| CODESYS application | `MainTask` running | ☐ |
| Kepware device | Connected and item quality Good | ☐ |
| Factory I/O driver | Connected, no unresolved mappings | ☐ |
| Ignition Factory I/O KPI | `ONLINE` | ☐ |
| Tank instantaneous value | Perspective tank matches live PLC percentage | ☐ |
| Tank historian | New points appear after value changes | ☐ |
| SCADA command tags | Read/write quality Good | ☐ |

If a tag reads correctly but rejects writes, test the item at the Kepware OPC client layer before changing Ignition. A good read proves connectivity, not write access.

## 4. Normal startup

1. Release the Twin Cell, Plant and Master simulated emergency stops.
2. Select Twin Cell `Plant/Auto`.
3. Select Plant `Auto`.
4. Select Master `Auto`, or turn on the Ignition Automatic Mode switch.
5. Confirm the Operating Mode KPI becomes `AUTO`.
6. Press Reset momentarily.
7. Confirm fault/recovery indications clear and Start becomes available.
8. Press Start momentarily.
9. Confirm Plant State becomes `RUNNING` and the Production Phase advances.

## 5. Full-cycle checks

| Stage | Evidence to observe | Result |
|---|---|---|
| Twin Cell | Lid/base machining cycle and pair count advance | ☐ |
| Pair Sorter | Alternating routing and complete pair at assembly | ☐ |
| Assembly | Clamp, lid pick/place, release and product count | ☐ |
| Product Transfer | Request, ready, permit and acknowledge sequence | ☐ |
| Packout | Carrier presented, product loaded and vision confirmation | ☐ |
| Tank | Level rises, batch-ready occurs, then level falls on discharge | ☐ |
| Storage reservation | Reserved count becomes 1 and a target position is displayed | ☐ |
| Crane deposit | Crane moves to target and completes the deposit | ☐ |
| Commit | Reserved returns to 0 and Occupied increments | ☐ |
| Completed load | Plant completed-load count increments | ☐ |

## 6. SCADA control checks

### Automatic mode

1. Turn Automatic Mode on in Ignition.
2. Confirm `SCADA.xIgnitionAutoModeReq` changes to true.
3. Confirm `SCADA.xAutoModeActive` reflects the PLC result.
4. Turn the request off and verify controlled-stop behaviour if the plant is mid-cycle.

### Start, Stop and Reset

1. Watch the corresponding SCADA request tag.
2. Press the Perspective One-Shot Button.
3. Confirm the request goes true briefly and the PLC clears it.
4. Confirm the resulting plant state changes, not merely the button colour.

Start, Stop and Reset are commands; their success is verified from PLC state feedback.

## 7. Controlled-stop test

1. Start a normal automatic cycle.
2. Wait until a production transaction is active.
3. Press Stop or remove automatic mode.
4. Confirm `xPlantStopPending` becomes true.
5. Confirm the active transaction completes safely.
6. Confirm the plant returns to `STOPPED` without beginning a new cycle.

## 8. E-stop and recovery test

This tests simulation logic only; it is not a safety validation.

1. Run the plant.
2. Operate one simulated emergency stop.
3. Confirm E-stop health becomes unhealthy and running commands are removed.
4. Confirm the plant reports Faulted or Recovery Required.
5. Release the emergency stop.
6. Confirm the system does not restart automatically.
7. Select valid modes, press Reset, then press Start.
8. Confirm a deliberate operator action is required before production resumes.

Repeat for each of the three panels.

## 9. Tank-history test

1. Confirm `rTankLevelPct` changes during fill and discharge.
2. In the tag editor, confirm History Enabled and the storage provider are valid.
3. In Perspective, confirm the chart history binding returns rows.
4. Compare the latest chart value with the cylindrical tank.
5. Let at least one complete fill/discharge cycle run.
6. Confirm the graph shows the rising, plateau and falling profile.

Historian data persists at the Gateway/database layer. Reimporting the Perspective project alone does not restore old samples.

## 10. Storage-integrity tests

### Normal reserve and commit

1. Record Free, Occupied, Reserved and Unknown totals.
2. Trigger one storage transaction.
3. Confirm one Free slot becomes Reserved.
4. Complete the physical deposit.
5. Confirm Reserved becomes Occupied.
6. Confirm all four counts still total 54.

### Interrupted reservation

1. Begin a deposit.
2. Interrupt the sequence after material may have entered the rack.
3. Apply the defined recovery/reset procedure.
4. Confirm the uncertain slot is marked Unknown rather than Free.

This protects the logical inventory from claiming a potentially occupied slot is available.

## 11. Demonstration evidence to capture

- Wide SCADA screenshot while running.
- Mobile SCADA screenshot at the responsive route.
- Factory I/O overall camera.
- Pair-transfer or assembly close-up.
- Tank trend showing a full fill/discharge profile.
- Storage summary before reservation, during reservation and after commit.
- CODESYS online view of plant state and phase.
- Optional short video covering startup through one stored load.

## 12. Test record

| Field | Value |
|---|---|
| Test date |  |
| CODESYS version |  |
| Factory I/O version |  |
| Ignition version |  |
| Operator |  |
| Full cycle pass | ☐ Yes / ☐ No |
| Controlled stop pass | ☐ Yes / ☐ No |
| E-stop recovery pass | ☐ Yes / ☐ No |
| Historian pass | ☐ Yes / ☐ No |
| Storage reconciliation pass | ☐ Yes / ☐ No |
| Notes |  |

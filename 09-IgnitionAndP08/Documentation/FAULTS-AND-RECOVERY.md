# Faults and Recovery

## Fault model

This project publishes Boolean diagnostic tags and numeric state enums. It does not use a single numeric fault-code register.

| Diagnostic | Trigger or meaning |
|---|---|
| `xHighHighFault` | Tank high-high input; latched |
| `xFillTimeoutFault` | Enabled filling exceeds two minutes |
| `xDischargeTimeoutFault` | Enabled discharge exceeds two minutes |
| `xTankFaulted` | Tank high-high, fill timeout, or discharge timeout |
| `xStepTimeoutFault` | Active storage movement step exceeds two minutes |
| `xStorageFaulted` | Storage reservation/step failure |
| `xRackFaulted` | Invalid rack reservation or commit transaction |
| `xSequenceFault` | Lost/invalid reservation state or inconsistent coordinator transaction acknowledgement |
| `xRecoveryRequired` | Emergency or enable loss during an active transaction |
| `xPlantFaulted` | Combined rack, recovery, sequence, storage, or tank fault |

## Important non-fault conditions

- `xRackFull` blocks new production but is not an equipment fault.
- `xResetRequired` is an acknowledgement/safety gate. It forces all outputs off even though it is not included in `xPlantFaulted`.
- `Unknown` rack slots are quarantined inventory, not necessarily failed equipment.

## Immediate safe actions

When emergency, reset-required, plant-faulted, or recovery-required is active, `PLC_PRG` removes all process/motion commands by commanding:

- Both tank valve voltages to `0.0 V`
- All conveyors off
- Emitter off
- Crane lift/forks off
- Numerical target to zero
- All roller commands off

Panel and stack indication lights remain active so the stopped/fault state can still be shown.

The software also prevents fill and discharge commands from being true together.

## Reset behaviour

Reset is accepted only when the relevant live fault condition has been removed. For the tank, a live high-high input prevents fault clearance.

Recommended sequence:

1. Stop the process and identify the cause.
2. Remove the physical/simulated cause.
3. Verify E-stop and Stop circuits are healthy.
4. Inspect material and reservation state.
5. Press Reset once.
6. Confirm fault flags clear and state returns to a stopped baseline.
7. Re-select Auto/Start only when the process is safe.

SCADA reset is a momentary `Application.SCADA.xResetReq`; CODESYS clears it after consumption.

## Transaction recovery

The rack manager distinguishes between:

- Reservation certainly unused → return slot to `Free`
- Reservation may contain physical material → mark slot `Unknown`

This conservative behaviour prevents the software from allocating a possibly occupied location after an interrupted transfer. An operator should inspect/reconcile unknown slots before returning them to service.

## HMI indication scope

The current Perspective footer shows state-derived fault status and OPC data quality. It is not a full alarm subsystem. A production-oriented extension should configure tag alarms for each fault Boolean and provide:

- Alarm Status Table
- Alarm Journal Table
- Acknowledgement permissions
- Shelving rules
- Audit records
- Historian/trend context around trips

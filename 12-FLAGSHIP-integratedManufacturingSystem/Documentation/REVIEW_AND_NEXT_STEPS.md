# Review and Next Steps

The supplied project is operational and portfolio-ready as an integrated simulation. This list separates quick presentation cleanup from changes that alter controller behaviour.

## A. Quick cleanup before public release

These are low-risk, mostly cosmetic changes.

| Item | Current finding | Suggested action |
|---|---|---|
| Ignition project metadata | `project.json` still names Project 9 | Set title/description to Project 12 |
| Mobile Start text colour | `FFFFFF` lacks `#` | Change to `#FFFFFF` |
| CSS radius units | Six live-status group values use `7p`; Reset banner uses `6p` | Change all to `7px`/`6px` |
| KPI icon fault status | Shared map uses `fault`, pages send `bad` | Rename/add the `bad` case |
| Allocation bar | Label calculates percentage; bar receives occupied + reserved raw count | Set `max=54` or bind the calculated percentage |
| Component name | `RecoveryRequiredLED'` has a stray apostrophe | Rename without the apostrophe in both views |
| Mobile breakpoint child | Desktop has explicit `large`; mobile has no position object | Set mobile child size to `small` |
| Factory I/O description | Scene description is blank | Add the description in `FACTORY_IO_SCENE.md` |

## B. Latent Ignition cleanup

These do not block the current screen but should be corrected before reusing the views.

### Canonicalise tag paths

The export contains these variants:

```text
[Sample_Tags]P12/...
[Sample_Tags]/P12/...
[sample_tags]/P12/...
[sample_Tags]/P12/...
```

Use one canonical form everywhere:

```text
[Sample_Tags]P12/...
```

Direct control and history bindings are already canonical; most differences are in display expressions, headers and some Twin Cell stage bindings.

### Hidden mobile diagnostic LEDs

The mobile `LiveStatusPanel` is currently hidden. Nine diode-on colour expressions inside it still test `xPlantRunning` rather than their own signal:

- Plant Faulted
- Transfer Request
- Transfer Ready
- Transfer Permit
- Transfer Acknowledge
- Storage Ready
- Storage Busy
- Storage Faulted
- Rack Full

Correct them before making the panel visible or copying it into another project.

### Remove unused view/route

- `P12/Shared/Navigation` is empty and unused.
- `/responsive` duplicates the root responsive route and has a blank title.

Deleting them is optional, but it reduces ambiguity in a portfolio export.

## C. PLC cleanup without intended behaviour change

| Item | Finding | Recommendation |
|---|---|---|
| Legacy Auto symbol | `SCADA.xAutoModeSelected` is unused and exports with read-only current access | Remove or mark deprecated after confirming no external client uses it |
| Dead enums/states | `E_PackoutState`, `E_PairCoordinatorState` and some enum members are not reached | Remove only after confirming they are not reserved for the next iteration |
| Tank diagnostics | Batch unit flattens any tank fault to code 610 | Expose the tank sub-code/cause to SCADA |
| Long `PLC_PRG` | Well-sectioned but large | Move normalisation, SCADA publication and output mapping to named actions/programs in P12MKII |

## D. Behavioural fixes to validate carefully

These require regression testing.

### 1. Retain or reconcile rack inventory

`aSlotState : ARRAY[1..54]` is not `RETAIN`/`PERSISTENT`. A PLC reinitialisation can therefore return the logical model to all-free even when the simulated/physical rack contains stock.

Production-style options:

- retain/persist the array with a versioned initialisation strategy;
- reconstruct from reliable position sensors;
- restore from a database/MES record;
- start in Unknown and run a reconciliation procedure.

### 2. Carrier-emission recovery latch

`FB_PackoutCarrierStation.xCarrierEmitIssued` sets when the emit pulse fires and clears when a pallet reaches the load point. Its reset branch does not visibly clear that latch. If carrier arrival times out, a reset may leave later emission inhibited.

Reproduce the timeout deliberately, then add a defined reset/recovery assignment if confirmed.

### 3. Commissioning permit

`xOutputPermit := TRUE` is hardcoded. Existing safety and module gates still constrain outputs, but the name suggests a live commissioning interlock.

P12.5 should drive it from an explicit commissioning/configuration state or rename it to avoid overstating its function.

### 4. Watchdog

The CODESYS task watchdog is disabled. That is acceptable for this simulation build, but a production-oriented version should configure and test appropriate task monitoring.

### 5. Hardcoded/uncommissioned outputs

Several Factory I/O lifecycle, panel-light, assembly-positioner and storage exit/unload commands remain fixed. Either commission them or document them explicitly as outside the demonstrated scope.

## E. Suggested P12MKII priorities

1. Rack retention/reconciliation.
2. Confirm and fix carrier-emission recovery.
3. Remove legacy/dead interface elements.
4. Introduce explicit commissioning permits.
5. Add focused state-machine and handshake tests.
6. Refactor scan sections into clearer units without changing behaviour.
7. Add alarm history and module-specific diagnostic detail.
8. Export the full Kepware and Ignition Gateway configuration.

## Change-control rule

After any behaviour-changing edit:

1. save a dated CODESYS/Factory I/O/Ignition set;
2. rebuild symbol configuration if interfaces changed;
3. refresh Kepware/OPC items;
4. rerun normal, controlled-stop, E-stop, historian and storage-integrity tests;
5. update the manifest and screenshots only after the integrated test passes.

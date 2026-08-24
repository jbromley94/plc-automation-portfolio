# Engineering Review

## Executive assessment

The strongest aspect of this project is the step up from single-lane conveyor logic to coordinated material handling. Two asynchronous infeeds, two clamps, a vacuum gripper and two axes are organised into a readable equipment state machine, while operator latching and external I/O remain separate.

The supplied files also preserve development-stage defects and design limitations. They make the project useful as an engineering-review example, but the current revision should be presented as a virtual prototype with a defined improvement plan—not as fully commissioned or production-ready control software.

## Effective design decisions

### Equipment logic is modular

`FB_AssemblerCell` owns the assembly sequence and `FB_OperatorControl` owns operator state. `PLC_PRG` is left to coordinate calls and map external I/O. That separation makes faults and sequencing easier to review than a monolithic program.

### Independent infeed control

In `WaitingForParts`, each conveyor stops when its own sensor detects a part. In `ClampingParts`, each conveyor continues only until its own clamp feedback is true. This handles unequal arrival time without relying on identical delays.

### Feedback-qualified motion

Most axis transitions require motion to be observed true and then false. This avoids advancing immediately on a stale stationary input and is a useful industrial handshake pattern.

### Command continuity across states

The base clamp remains asserted throughout the pickup/placement motion, and Grab remains asserted from pickup until the lid is released. Commands are therefore tied to process intent rather than isolated one-state pulses.

### Deliberate defaults and fault override

All cell commands default false at the start of each call. A second final interlock forces them off if the state becomes Faulted. A timeout therefore removes cell commands in the same detection scan.

### Central timing constants and diagnostics

Seven timings are stored in `AssemblyConstants`, and sequence failures receive specific codes rather than a single generic fault bit.

### Scan-aware completion handoff

The product-complete pulse is generated after operator control runs. A pending Stop sees the FB output on the next scan, before a new cell state can issue commands. This uses deterministic call order rather than an external pulse that could be missed between modules.

## Traceability

| Engineering feature | Implementation evidence |
| --- | --- |
| State-based sequencing | 16-value `E_AssemblerState` and `FB_AssemblerCell` case statement |
| Independent infeed | Per-lane conveyor commands use `NOT xLidAtPlace` / `NOT xBaseAtPlace` |
| Clamp proof | `xLidClamped` and `xBaseClamped` gate transfer |
| Feedback-qualified axes | `xMotionObserved` plus Moving true→false transitions |
| Pickup proof | `xItemDetected` held for the 300 ms grab settle |
| Fault diagnosis | Codes 101, 102, 201–210, intended 301 and 999 |
| Operator separation | `FB_OperatorControl` manages edges, Run and Stop Pending |
| I/O abstraction | 40-variable `FIO` GVL and final mapping section |
| Simulation integration | Factory I/O OPC Client DA through `PTC.KepwareServer` |

## Confirmed source findings

### 1. Product-exit fault 301 is unreachable

`fbProductExitTimeout` runs in `WaitingForProductExit` with the intended 10-second setting, but the state checks `fbMotionTimeout.Q`. The motion timer is disabled and reset in that state, so fault 301 cannot occur.

Impact: a missing leaving edge can hold the cell indefinitely with the base conveyor, base-positioner raise and selected peripheral outputs active.

Required correction: replace the state-140 condition with `fbProductExitTimeout.Q`, then execute tests for missing, late, stuck-high and early leaving signals.

### 2. A leaving edge can be consumed before it is needed

`rtPartLeaving` runs in every state. If the input rises before `WaitingForProductExit`, its one edge is consumed. The input must become false and rise again after state entry to complete/count the product.

Combined with the broken timeout, this creates a permanent-wait scenario. Gate the edge detector to the valid state, latch an early arrival for evaluation, or use a level/clearance sequence appropriate to the sensor geometry.

### 3. Controlled Stop uses arrival, not clearance

`xSafeToStop` is the one-scan `xProductCompletePulse`. The pulse is generated on the rising edge of Part Leaving, proving that an object reached the sensor—not that it cleared the sensor, conveyor, chute or remover.

When Stop Pending is cleared on the following scan, the base conveyor and finished-product remover lose enable. A stopped product may therefore remain in the outfeed.

Required correction: build a maintained safe-boundary condition that includes product completion, leaving-sensor clearance and a proved outfeed-clearance time.

### 4. Stop while idle can create another product

There is no separate idle/empty safe-stop level. If Stop is pressed in `WaitingForParts`, automatic enable remains true until the next completion pulse. Emitters and conveyors can feed and assemble another product merely to satisfy the stop condition; missing material can instead lead to fault 101.

Required correction: define safe stop for both idle and active states and inhibit new WIP as soon as Stop Pending becomes true.

### 5. Stop is an edge request, not a maintained permissive

The normally closed Stop input is inverted and passed through `R_TRIG`. Once the edge is consumed, Reset can clear Stop Pending and a later Start can relatch Run while the raw Stop remains false.

Required correction: include the healthy Stop level in the final run permissive, define reset-required behaviour for Stop-circuit loss and regression-test hold/release/disconnection cases.

### 6. Start can cancel Stop Pending

A Start edge does not require Stop Pending to be false. Its branch sets Run true and Stop Pending false. The source comment describes this as resuming without Reset, but the operator policy should explicitly decide whether cancellation is allowed and how it is indicated.

### 7. Pause/runtime loss restarts the logical sequence

Pause or loss of Factory I/O Running removes enable but preserves the Run latch. `FB_AssemblerCell` then returns any non-faulted state to `Stopped`. When the scene resumes, retained Run automatically starts Homing without another Start.

Impact: clamps or Grab can drop immediately, while physical WIP remains in place and the sequence later restarts over it.

Required correction: implement a defined paused/aborted state and a recovery decision based on actual axis, clamp, gripper and part positions.

### 8. Routine Reset can abort an active cycle

Reset has high priority whenever emergency stop is not active. It clears Run, state, fault and production count without a safe-boundary check. The Factory I/O scene is reset only if Reset Required was already true, so a routine Reset can discard the PLC model while leaving physical scene parts behind.

Required correction: separate Fault Reset, Sequence Initialise and Production Counter Reset, and inhibit routine reset during unsafe motion unless an explicit abort procedure is followed.

### 9. Homing and endpoints are inferred

Homing accepts 500 ms with both Moving inputs false. It does not require a movement or dedicated home-position proof. Other motion states prove only that movement started and stopped.

Required correction for a stronger design: add endpoint inputs, validate target position after motion and fault inconsistent command/position combinations.

### 10. Assembly success is not proved

Grab releases for a fixed 300 ms dwell and the sequence assumes Factory I/O joins correctly aligned parts. It does not prove attachment, product quality or continued lid presence during transfer. The leaving sensor can count any detected object.

The scene enables blue and green lids and bases independently but has no colour identification or recipe matching.

### 11. Reset output is only one PLC scan

The internal reset pulse lasts one 20 ms call. The external `xFactoryIOResetCmd` is asserted for that one scan only during abnormal recovery. An asynchronous Kepware/OPC/Factory I/O path can miss a short pulse unless its update timing is proved.

Recommended action: stretch or acknowledge the external command independently of the internal one-scan reset event.

### 12. Communications health is unsupervised

There is no OPC quality, heartbeat or message-age input. The task watchdog is disabled, and the saved device stop/reset behaviour records `KeepCurrentValues`. Actual output/client behaviour during runtime and communications loss must be tested.

### 13. Diagnostics remain local

State, fault code, Ready, Busy, Run, Stop Pending and Reset Required are not published in the selected symbol group. Fault finding therefore depends on a CODESYS online session rather than a scene/HMI diagnostic interface.

### 14. Broad peripheral enable can create accumulation

Both emitters and both removers follow automatic enable rather than state or demand. They remain enabled during a pending Stop and have no lane-full/downstream-available interlock. The lid conveyor also preloads the next lid during product exit.

## Known limitations

| Area | Current limitation | Consequence |
| --- | --- | --- |
| Safety | Emergency stop and Stop use ordinary PLC logic over OPC DA | Not safety-rated and unsuitable for hazardous physical motion |
| Product exit | Wrong timer Q is checked | Fault 301 cannot occur; state 140 can wait forever |
| Counting | Global rising edge can occur before valid state | Early edge can be lost; count proves detection, not quality |
| Controlled stop | Completion pulse only; no idle or outfeed-clear condition | New WIP may start and product may stop before clearing |
| Stop input | Edge-only, not held as permissive | Run can relatch while Stop remains false |
| Pause/runtime loss | State resets but Run remains latched | Automatic Homing restart over residual WIP |
| Reset | Immediate abort and count clear | Physical and logical state can diverge |
| Position proof | Moving status only | Stopped does not necessarily mean correct endpoint |
| Product quality | No join or colour verification | Counter can include mismatched or incomplete products |
| Communications | No heartbeat, quality or stale-data supervision | Static bad/last-known data is not distinguished |
| Diagnostics | Internal state/faults not published | Limited HMI/SCADA fault isolation |
| Peripheral control | Emitters/removers follow broad enable | No demand or accumulation management |
| Runtime | Non-real-time Windows soft PLC; watchdog disabled | Timing/overrun suitability requires measurement |
| Symbols | All selected FIO items have read/write access | Logical direction is not externally enforced |
| Persistence | Count is non-retentive and Reset clears it | Not suitable for production reporting/OEE |
| Packout scope | Outfeed/removal only | No box, label, seal or pallet operation |
| Test evidence | No execution record embedded | Acceptance results must be produced separately |

## Improvement roadmap

### Priority 1 — correctness and safe recovery

- Fix state 140 to use `fbProductExitTimeout.Q`.
- Redesign the leaving-sensor logic so an early edge cannot be lost.
- Create separate safe-stop conditions for idle and active operation.
- Require confirmed sensor clearance and outfeed run-on before stopping.
- Hold the healthy Stop level in the run permissive.
- Define whether Start may cancel Stop Pending.
- Stretch/acknowledge the external Factory I/O reset command.
- Define Pause, runtime-loss and mid-cycle Reset recovery policies.

### Priority 2 — diagnosis and process robustness

- Publish state, fault code, Run, Stop Pending, Ready and Busy as read-only diagnostics.
- Add an OPC heartbeat and latched communications fault.
- Add endpoint sensors or stronger target-position validation.
- Monitor gripper item presence throughout transfer.
- Add an assembly-success or product-quality check.
- Separate sequence reset from production-count reset.
- Add demand/full-line control for emitters and removers.
- Enable and tune the task watchdog after measuring execution time.

### Priority 3 — capability and platform progression

- Add recipe/colour identification if matched variants are required.
- Implement controlled Manual/Maintenance jog with movement permissives.
- Extend true packout with inspection, reject, box, label or pallet handling.
- Add automatic regression tests for all state transitions and fault codes.
- Trend cycle time, waiting time, motion time and fault frequency in SCADA.
- Consider OPC UA after separately configuring, securing and proving it; the symbol-support flag alone is not an operating connection.

## Portfolio value

This project demonstrates:

- coordinated control of two asynchronous material streams;
- feedback-based clamping, pickup and multi-axis handling;
- a readable enum state machine with centralised time settings;
- correct command continuity across multi-step equipment actions;
- scan-cycle reasoning and edge-triggered event handling;
- state-specific diagnostics and virtual fault-injection planning;
- a complete saved symbolic map across CODESYS, Kepware and Factory I/O;
- a transparent engineering review that identifies defects and defines a credible remediation plan.

That final point supports a useful interview discussion: explaining why fault 301 is unreachable and how the Stop/pause recovery risks should be corrected is stronger engineering evidence than presenting the prototype as flawless.

[Back to the project README](../README.md)

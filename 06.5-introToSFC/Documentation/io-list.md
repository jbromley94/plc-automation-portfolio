# I/O, Variable, and External Interface List

## 1. Scope

This document inventories every user-declared variable in the supplied `introToSFC` export and separates three concepts that are easy to confuse:

- program-local variables used by the control logic;
- externally published symbols used by software clients; and
- fixed IEC addresses used for physical or mapped I/O.

The export contains the first category only. It has no Symbol Configuration object, no `FIO` object, and no `%I`, `%Q`, or `%M` addresses.

## 2. Interface Summary

| Interface layer | Current status |
|---|---|
| User-declared variables | Seven in `PLC_PRG`; eight separate variables in `PLC_PRG_ST` |
| Scheduled program | `PLC_PRG` only |
| Symbol Configuration | Not present |
| OPC UA / external symbol list | Not configured |
| Fixed IEC addresses | None |
| Device I/O modules | None below the soft-PLC device |
| Factory I/O mapping | None |
| KEPServerEX tags | None |

## 3. Active SFC Variables

All variables in this table belong to `PLC_PRG`, the program called by `MainTask`.

| Variable | Type | Initial value | Logical role | Written by SFC | Fixed address | External symbol |
|---|---|---:|---|---:|---|---|
| `A` | `BOOL` | `FALSE` | Initial-step action/status | Yes, `N` in `Step0` | None | Not configured |
| `B` | `BOOL` | `FALSE` | Left-route stored action/status | Yes, `S` in `Step1`; `R` in `Step3` | None | Not configured |
| `C` | `BOOL` | `FALSE` | Right-route stored/non-stored action/status | Yes, `S` in `Step2`; `N` in `Step3` | None | Not configured |
| `Trans0A` | `BOOL` | `FALSE` | Left route guard from `Step0` | No | None | Not configured |
| `Trans0B` | `BOOL` | `FALSE` | Right route guard from `Step0` | No | None | Not configured |
| `Trans1` | `BOOL` | `FALSE` | Left-route completion guard | No | None | Not configured |
| `Trans2` | `BOOL` | `FALSE` | Right-route completion guard | No | None | Not configured |

`Trans0A`, `Trans0B`, `Trans1`, and `Trans2` are not declared as `VAR_INPUT`; they are ordinary program-local variables. During development they can be manipulated from an online watch or force list. They become a formal software interface only if deliberately published through Symbol Configuration or connected to mapped I/O.

## 4. Unscheduled ST Variables

`PLC_PRG_ST` declares a separate namespace. None of these variables are read or written by `PLC_PRG`.

| Variable | Type | Explicit initial value | Logical role | Fixed address | External symbol |
|---|---|---:|---|---|---|
| `A` | `BOOL` | No (`FALSE` by default) | State-0 output | None | Not configured |
| `B` | `BOOL` | No (`FALSE` by default) | State-1 output | None | Not configured |
| `C` | `BOOL` | No (`FALSE` by default) | State-2 output | None | Not configured |
| `state` | `INT` | `0` | Manual state selector, intended range 0–3 | None | Not configured |
| `Trans0A` | `BOOL` | No (`FALSE` by default) | Intended state 0-to-1 guard | None | Not configured |
| `Trans0B` | `BOOL` | No (`FALSE` by default) | State 0-to-2 guard | None | Not configured |
| `Trans1` | `BOOL` | No (`FALSE` by default) | State 1-to-3 guard | None | Not configured |
| `Trans2` | `BOOL` | No (`FALSE` by default) | State 2-to-3 guard | None | Not configured |

Because `PLC_PRG_ST` is not scheduled, changing its transition variables does not advance either implementation. It only changes dormant memory values unless the POU is later added to a task.

## 5. Namespace Separation

The following names are not aliases:

| Active SFC location | Separate ST location |
|---|---|
| `PLC_PRG.A` | `PLC_PRG_ST.A` |
| `PLC_PRG.B` | `PLC_PRG_ST.B` |
| `PLC_PRG.C` | `PLC_PRG_ST.C` |
| `PLC_PRG.Trans0A` | `PLC_PRG_ST.Trans0A` |
| `PLC_PRG.Trans0B` | `PLC_PRG_ST.Trans0B` |
| `PLC_PRG.Trans1` | `PLC_PRG_ST.Trans1` |
| `PLC_PRG.Trans2` | `PLC_PRG_ST.Trans2` |

A watch list should include the POU prefix to prevent an operator from writing the wrong copy.

## 6. Action Ownership

| Variable | Source of true state | Source of false/reset state | Review note |
|---|---|---|---|
| `A` | `Step0` active through `N` | `Step0` deactivation | Non-stored action |
| `B` | `Step1` through `S` | `Step3` through `R` | Explicit set/reset pair |
| `C` | `Step2` through `S`; also `Step3` through `N` | No `R C` association exists | Stored reset is missing |

External clients should not write `A`, `B`, or `C` during normal operation. Doing so creates multiple control authorities and can hide the real qualifier behaviour.

## 7. Transition Inputs

| Guard | Evaluated while | Meaning to assign during integration | Priority |
|---|---|---|---:|
| `Trans0A` | `Step0` active | Select/permit left sequence | 1 |
| `Trans0B` | `Step0` active | Select/permit right sequence | 2 |
| `Trans1` | `Step1` active | Left sequence complete | — |
| `Trans2` | `Step2` active | Right sequence complete | — |

The guards are level-sensitive. A connected pushbutton or sensor that stays true can pass a transition immediately when its preceding step becomes active. If one event per actuation is required, add edge handling or acknowledgement logic deliberately.

## 8. Current Startup and Reset Values

On application initialisation, the declared values are expected to be:

| Variable group | Expected initial value |
|---|---:|
| `PLC_PRG.A`, `B`, `C` before first SFC processing | `FALSE` |
| All `PLC_PRG.Trans*` guards | `FALSE` |
| All `PLC_PRG_ST` Booleans | `FALSE` |
| `PLC_PRG_ST.state` | `0` |

On the first active SFC processing cycle, `Step0` is activated and its `N A` association should make `PLC_PRG.A` active. Build and runtime tests must capture the exact first-scan observation.

The source contains no operator reset input. Application reset/restart behaviour must not be presented as a substitute for a designed machine reset.

## 9. Proposed Symbol Configuration

Do not create an external interface simply by selecting every variable. A minimal interface for the active SFC would be:

| Variable | Proposed client access | Reason |
|---|---|---|
| `PLC_PRG.Trans0A` | Read/write | External left-route request or condition |
| `PLC_PRG.Trans0B` | Read/write | External right-route request or condition |
| `PLC_PRG.Trans1` | Read/write | External left completion condition |
| `PLC_PRG.Trans2` | Read/write | External right completion condition |
| `PLC_PRG.A` | Read-only | PLC-generated state/action indication |
| `PLC_PRG.B` | Read-only | PLC-generated state/action indication |
| `PLC_PRG.C` | Read-only | PLC-generated state/action indication |

Do not publish the `PLC_PRG_ST` group until that program is corrected and intentionally scheduled. Publishing dormant duplicate variables would make client configuration ambiguous.

After adding Symbol Configuration and completing an error-free build, verify the actual generated browse paths. Paths similar to `Application.PLC_PRG.Trans0A` may be expected, but the runtime browser is the source of truth.

## 10. Suggested Factory I/O Mapping

No Factory I/O scene is currently included. If a simple visual demonstration is added, one possible mapping is:

| PLC variable | Suggested scene role | Data direction |
|---|---|---|
| `PLC_PRG.Trans0A` | Left-route selection button or sensor | Factory I/O → PLC |
| `PLC_PRG.Trans0B` | Right-route selection button or sensor | Factory I/O → PLC |
| `PLC_PRG.Trans1` | Left-route completion sensor | Factory I/O → PLC |
| `PLC_PRG.Trans2` | Right-route completion sensor | Factory I/O → PLC |
| `PLC_PRG.A` | Ready/route-selection indicator | PLC → Factory I/O |
| `PLC_PRG.B` | Left-route indicator or simulated actuator | PLC → Factory I/O |
| `PLC_PRG.C` | Right-route indicator or simulated actuator | PLC → Factory I/O |

This mapping is a proposed demonstration interface, not an extracted as-built mapping. The `FactoryIO/` folder needs only its placeholder README until a real scene and verified mapping exist.

## 11. Proposed Physical I/O Design Rules

If the learning sequence is adapted to physical I/O:

- declare clear input and output names instead of `A`, `B`, `C`, and `Trans*`;
- map addresses through the device tree or a documented I/O abstraction layer;
- separate raw inputs from qualified control conditions;
- derive outputs from a defined safe-state policy;
- add actuator and process feedback;
- add timeout, fault, reset, and stop behaviour;
- prevent an external symbol write from bypassing physical interlocks; and
- document signal voltage, normal state, fail state, and safety classification.

## 12. Proposed Engineering I/O List Template

Use this structure when real signals are introduced:

| Tag | Description | Direction | Type | Address / symbol | Normal state | Fail-safe state | Owner | Test ID |
|---|---|---|---|---|---|---|---|---|
| To be assigned | Left route request | Input | `BOOL` | To be assigned | False | False | Process / operator | To be assigned |
| To be assigned | Right route request | Input | `BOOL` | To be assigned | False | False | Process / operator | To be assigned |
| To be assigned | Left route complete | Input | `BOOL` | To be assigned | False | Application-specific | Process | To be assigned |
| To be assigned | Right route complete | Input | `BOOL` | To be assigned | False | Application-specific | Process | To be assigned |
| To be assigned | Left action command | Output | `BOOL` | To be assigned | False | False | PLC | To be assigned |
| To be assigned | Right action command | Output | `BOOL` | To be assigned | False | False | PLC | To be assigned |

## 13. Change-Control Checks

Repeat the following after any interface change:

1. build without errors;
2. verify the `MainTask` program list;
3. refresh Symbol Configuration;
4. browse the final runtime paths;
5. confirm write access only on intended command variables;
6. test each transition independently;
7. test simultaneous route requests and confirm left priority;
8. confirm the corrected reset behaviour for `C`;
9. restart the runtime and repeat the browse/write test; and
10. update Factory I/O, KEPServerEX, HMI, trace, and documentation files together.

## 14. Safety Boundary

No variable in this export is a validated physical signal. The names and proposed mappings must not be interpreted as proof of safe machine I/O. External write access is a commissioning feature, not a safety mechanism.

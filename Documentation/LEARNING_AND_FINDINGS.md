# Learning and Findings

This is the candid engineering log for Project 12: what worked, what caused confusion, what the exports prove, and the rule I would carry into the next project.

## 1. `EXTENDS` and `SUPER^()`

### What they mean

CODESYS supports single inheritance between function blocks:

```iecst
FUNCTION_BLOCK FB_Derived EXTENDS FB_Base
```

The derived block receives the base block's data and methods. Inside the derived FB or its methods, `SUPER^` dereferences the base instance so an overridden implementation can call base behaviour.

Conceptual example:

```iecst
METHOD PUBLIC Reset

SUPER^.Reset();
// Add the derived block's extra reset work here.
```

Official references:

- [Extension of a Function Block — CODESYS](https://content.helpme-codesys.com/en/CODESYS%20Development%20System/_cds_extending_function_block.html)
- [Pointer: SUPER — CODESYS](https://content.helpme-codesys.com/en/CODESYS%20Development%20System/_cds_pointer_super.html)

### What Project 12 actually uses

The supplied export contains **zero** `EXTENDS`, `SUPER` or `SUPER^` occurrences. P12 uses composition:

- `FB_TwinCellUnit` owns two machining-cell instances.
- `FB_AssemblyUnit` owns an assembler-cell instance.
- `FB_BatchStorageUnit` owns the tank, storage and rack manager.

This is appropriate. A tank is not a specialised form of a storage crane, and an assembly unit is not a specialised machining cell. They collaborate or are owned by a higher-level unit.

### Rule for the next project

Use inheritance only for a genuine **is-a** relationship with stable shared behaviour. Use composition for equipment that is contained, coordinated or exchanged independently. Do not introduce `EXTENDS` solely to reduce a few repeated lines.

## 2. `{attribute 'analysis' := '-33'}`

### Correct syntax

The note:

```iecst
{attribute 'analysis' := '-33-'}
```

contains an extra trailing hyphen. The whole-object form is:

```iecst
{attribute 'analysis' := '-33'}
```

SA0033 is the CODESYS static-analysis rule for unused variables. The attribute disables that rule for the containing programming object.

For a smaller implementation scope, CODESYS also supports:

```iecst
{analysis -33}
// Deliberately external or generated declarations.
{analysis +33}
```

Official references:

- [Attribute: analysis — CODESYS](https://content.helpme-codesys.com/en/CODESYS%20Static%20Analysis/_san_attribute_analysis.html)
- [SA0033: Unused variables — CODESYS](https://content.helpme-codesys.com/en/CODESYS%20Static%20Analysis/_san_rule_sa0033.html)

### Why it came up around `FIO`

The `FIO` GVL is an external symbolic contract. Some members are mapped to Factory I/O or reserved for commissioning even when the PLC code does not currently consume them. Static analysis can therefore report them as unused despite their external purpose.

### Finding from the export

No `attribute 'analysis'` or `{analysis ...}` pragma exists in the delivered P12 export.

### Rule for the next project

Do not silence SA0033 across an entire GVL just to obtain a clean build. First remove stale declarations. If an external mapping is deliberately unused internally, suppress the rule narrowly and add a comment explaining the external consumer.

## 3. Grey or apparently blank function blocks

### What the export proved

None of the 15 P12 function blocks is blank. Every FB in the supplied export has both a non-empty declaration and a non-empty implementation. The smallest implementation still contains substantial source.

The grey/blank appearance was therefore an editor or object-selection presentation issue, not lost code in the saved export.

### Useful CODESYS distinction

A grey POU can indicate that it is not reached by the compiled call graph. A function block type becomes active through an **instance that is called**; merely defining the FB does not execute it. A blank implementation pane can also result from selecting a declaration-only child object, an inherited item, the wrong editor pane, or a transient Designer layout state.

### Diagnostic checklist

1. Confirm the selected object is the FB itself, not a child/interface item.
2. Switch between declaration and implementation panes.
3. Search the project for an instance declaration such as `fbUnit : FB_Unit;`.
4. Confirm that instance is called in executable code.
5. Build and inspect warnings/call-tree status.
6. Export the object and check whether implementation text is present.
7. Reopen/reset the editor layout before assuming source was lost.

### Rule for the next project

Treat UI colour as a clue, not proof. Verify the call graph and the saved/exported source before recreating a POU.

## 4. The failed Auto-mode write

### Symptom

Ignition and the OPC client could read `xAutoModeSelected`, but writes failed with a bad/unsupported result. Kepware reported a synchronous write failure and `HRESULT 0x80004005`.

`0x80004005` is a generic failure result. It told us the write failed, but not the root cause by itself.

### What the CODESYS export revealed

Two similar SCADA variables existed:

```iecst
xAutoModeSelected    : BOOL := FALSE;
xIgnitionAutoModeReq : BOOL;
```

Both source declarations carried a `readwrite` attribute, but the generated symbol metadata differed:

| Variable | Generated current access | Used by executable logic |
|---|---:|---|
| `SCADA.xAutoModeSelected` | 1 | No |
| `SCADA.xIgnitionAutoModeReq` | 3 | Yes |

The replacement variable was read/write in the generated symbol and was actually consumed:

```iecst
xAutoModeSelected :=
    FIO.xMasterAutoModeSelected
    OR SCADA.xIgnitionAutoModeReq;
```

### Finding

The three physical control panels did **not** cause the OPC write failure. The old symbol was stale/unused and exported differently. Creating a dedicated HMI request variable established a clean command boundary and solved the write path.

### Rule for the next project

Name remote commands as requests, for example `xHmiAutoModeReq`, and keep PLC state feedback separate, for example `xAutoModeActive`. Never bind an HMI control to a value that looks like calculated state.

## 5. Resetting CODESYS, Kepware and OPC cleanly

During symbol changes, repeatedly changing the HMI first wastes time because downstream caches can still expose an old item definition.

### Recovery order

1. Save and rebuild the CODESYS application.
2. Rebuild/update Symbol Configuration.
3. Download/login and run the latest application.
4. Confirm the symbol's generated access in CODESYS.
5. Refresh or recreate the Kepware item.
6. Restart/reinitialise the Kepware device only if the refreshed item still uses stale metadata.
7. Test a manual write in the Kepware OPC client.
8. Refresh/rebrowse the Ignition OPC item.
9. Confirm Ignition tag `CanWrite` and quality.
10. Test the Perspective binding last.

### Why this order matters

Each step proves one boundary. A successful Kepware-client write isolates the remaining issue to Ignition. A failed Kepware-client write means Perspective styling or binding syntax is irrelevant until the lower layer is fixed.

### Preservation note

A full reset/recommission can destroy useful state or configuration. Save the CODESYS project/archive, Factory I/O scene, Kepware configuration and Ignition Gateway/project resources before performing destructive recovery.

## 6. Momentary commands versus maintained state

Start, Stop and Reset are events; Auto selection is state.

The final design reflects that distinction:

- Start/Stop/Reset use Ignition One-Shot Buttons and PLC edge detection.
- The PLC clears the request bits after all consumers have sampled them.
- Auto uses a maintained bidirectional Boolean request.
- The HMI displays separate PLC feedback to show whether Auto is actually active.

This avoids stuck pushbuttons and prevents a screen control from pretending a command succeeded before the PLC confirms it.

## 7. Direct bindings, expression bindings and tag paths

Direct bindings expect a tag path, not an Ignition expression wrapped in braces. Expression bindings use `{[Provider]path}` syntax inside an expression.

The debugging lesson was to choose the binding type first:

- direct tag binding: `[Sample_Tags]P12/SCADA/xTagName`;
- expression binding: `if({[Sample_Tags]P12/SCADA/xTagName}, "ON", "OFF")`.

The export still contains mixed provider capitalisation and optional leading slashes in some display expressions. They work in the current environment but should be normalised to one canonical form before reuse.

## 8. Historian-backed charts

A populated binding preview proves that the historian query returns a dataset, but a Time Series Chart also needs the expected column shape and a valid series configuration.

The final chart uses the same `rTankLevelPct` source as the tank graphic, a 10-minute real-time range, MinMax aggregation and 2-second polling.

The project ZIP does not include the Gateway history provider or stored samples. Ignition's official documentation distinguishes project exports from Gateway-level configuration: [Project Export and Import](https://www.docs.inductiveautomation.com/docs/8.1/platform/projects/project-export-and-import).

### Rule for the next project

Test the history chain from bottom to top: live tag → History Enabled/storage provider → returned dataset → series binding → chart axes and size.

## 9. Responsive does not mean “shrink desktop”

The first mobile attempts compressed six stage cards and the desktop header into a narrow viewport. The successful design created a real mobile composition:

- two-column KPI cards;
- horizontally scrollable process stages;
- hidden low-priority diagnostic LEDs;
- stacked tank and chart;
- reordered Operator Controls and Storage Summary;
- compact header with secondary metadata removed.

### Rule for the next project

Responsive layout is a hierarchy decision. Decide what must remain visible, what can scroll, what can move and what can be omitted before tuning font sizes.

## 10. Storage needed a transaction, not just a count

A naive storage implementation would increment Occupied as soon as a target is selected. P12 instead uses reserve then commit:

- reservation claims capacity;
- physical storage uses the target;
- commit occurs only after completion;
- uncertain interrupted deposits become Unknown.

This is a useful general pattern for warehouse, batching and material-tracking systems where logical state must not get ahead of physical reality.

## 11. What I would preserve unchanged

- Typed state enums and explicit operator text mappings.
- Composition of unit controllers.
- Reusable transfer and operator-control FBs.
- Input normalisation near the composition root.
- Reserve/commit/unknown inventory model.
- Parameterised Perspective KPI and FlowStage views.
- Shared desktop/mobile style classes.
- Separate HMI request and PLC feedback tags.

## 12. What I would change first in P12MKII

1. Retain or reconcile the 54-slot rack inventory at startup.
2. Remove the legacy `SCADA.xAutoModeSelected` variable.
3. Resolve the carrier-emission recovery latch after a timeout.
4. Make output commissioning permits explicit rather than hardcoded.
5. Split the long `PLC_PRG` publication/mapping sections into clearer actions or programs.
6. Standardise all Ignition tag paths and status keys.
7. Add an automated test matrix for state transitions and handshakes.

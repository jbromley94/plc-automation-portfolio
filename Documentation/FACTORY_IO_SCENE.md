# Factory I/O Scene

## File summary

The supplied `.factoryio` file is a valid Factory I/O 2.5.10 scene saved on 21 August 2026 at 16:37:54. Although the extension is application-specific, its contents are UTF-8 XML.

The scene contains:

- 184 physical objects;
- 13 named camera presets;
- 158 physical I/O points;
- 154 configured OPC mapping entries;
- three operator-control stations;
- no duplicate or unresolved mapping references.

## Physical plant inventory

| Asset | Count |
|---|---:|
| Machining centres | 2 |
| Two-axis pick-and-place units | 2 |
| Driven conveyor sections | 25 |
| Positioning/clamping bars | 2 |
| Pair-routing pusher | 1 |
| Analogue process tank | 1 |
| Stacker crane | 1 |
| Storage racks | 3 |
| Emitters | 3 |
| Product/carrier remover | 1 |
| Retroreflective sensors | 12 |
| Diffuse sensors | 4 |
| Capacitive high-high sensor | 1 |
| Vision sensor | 1 |
| Three-colour stack lights | 5 |
| Digital counter displays | 5 |

## Production route

1. Blue and green raw materials are emitted onto separate lid and base conveyors.
2. Two machining centres process the parts in parallel.
3. Exit conveyors feed a count sensor, junction and pusher-based pair sorter.
4. The two streams enter separate assembly conveyors and positioning/clamping stations.
5. An X/Z pick-and-place unit transfers the lid onto the base.
6. Three buffer conveyors move the assembled product to packout.
7. A carrier emitter and infeed conveyor present an empty carrier.
8. A second X/Z pick-and-place loads the product; a vision sensor confirms loading.
9. Four transfer conveyors move the carrier through the tank station.
10. The tank provides level, flow, fill/discharge and high-high signals.
11. Entry, load, unload and exit conveyors feed the stacker crane.
12. The crane receives a target position and deposits the carrier into the rack system.

The physical scene contains three rack assets. The 54-slot inventory is a PLC/SCADA logical model rather than 54 individually modelled Factory I/O locations.

## Physical I/O profile

| Type | Count |
|---|---:|
| Boolean outputs | 83 |
| Boolean inputs | 63 |
| Integer outputs | 6 |
| Integer inputs | 2 |
| Analogue inputs | 2 |
| Analogue outputs | 2 |
| **Total** | **158** |

## OPC mapping profile

| Subsystem | Factory I/O to PLC | PLC to Factory I/O | Total |
|---|---:|---:|---:|
| Twin Cell | 18 | 23 | 41 |
| Pair Sorter | 4 | 5 | 9 |
| Assembly | 10 | 9 | 19 |
| Packout | 6 | 13 | 19 |
| Tank | 4 | 9 | 13 |
| Storage | 9 | 12 | 21 |
| Plant panel | 6 | 6 | 12 |
| Master panel | 6 | 6 | 12 |
| Factory I/O runtime | 4 | 4 | 8 |
| **Total** | **67** | **87** | **154** |

The embedded driver uses:

```text
PTC.KepwareServer
Channel2.project12.Application.FIO...
```

Of the 158 physical points, 146 are mapped. The remaining 12 are optional rotation and gripper-rotation capabilities on the two pick-and-place components. The project uses X, Z and Grab only, so these are intentionally unused rather than missing mappings.

## Operator stations

The Twin Cell, Plant and Master panels each contain:

- an Auto/Manual or Plant/Local selector;
- a simulated emergency stop;
- illuminated Start, Stop and Reset pushbuttons.

All nine Start/Stop/Reset pushbuttons are configured as momentary controls, matching the PLC edge-trigger and SCADA one-shot strategy.

## Camera presets

The 13 named camera positions include views for the Twin Cell, Assembly, Master Control Panel, batch/storage areas and overall plant. These are useful for demonstrations and targeted commissioning evidence.

## Suggested scene description

The current scene-level description is blank. A concise replacement is:

> Project 12 — Integrated Manufacturing System. CODESYS-controlled twin-cell machining, pair routing, robotic assembly, robotic packout, analogue tank batching and automated rack storage, connected through Kepware OPC DA.

## Source-control note

The scene embeds a large base64 image, so line-based Git diffs are noisy even though the file is XML. Treating `*.factoryio` as binary in `.gitattributes` keeps the repository history readable.

## Cosmetic-only cleanup

Several raw component labels contain minor inconsistencies such as `Pik&Place`, `Movin Z`, `Rot8`, varying spaces and one missing closing parenthesis. The PLC-facing names are clean, so these do not affect operation.

## What this file proves

The scene proves the physical topology, component configuration, I/O profile and mapping structure. It does not prove PLC state logic, timeout behaviour, storage-allocation rules, runtime cycle success, or Ignition historian behaviour; those claims are supported by the other exports and test evidence.

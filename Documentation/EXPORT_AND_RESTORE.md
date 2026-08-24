# Export, Backup and Restore

The three supplied files preserve the main application work, but they are three different types of export. They should not be treated as one complete machine backup.

## Supplied files

| File | Contains | Does not contain |
|---|---|---|
| `P12_IntegratedManufacturingSystem_2026-08-21_1700.zip` | Ignition project metadata, Perspective page/session configuration, 8 views, 9 style classes and global project properties | Gateway tag providers/tags, OPC connection, historian provider/database, Kepware configuration |
| `integratedManufacturingSystem-21-08-2026-1637pm.export` | CODESYS device/application objects, task, `PLC_PRG`, 15 FBs, 14 enums and 3 GVLs | A portable project archive with every referenced device/library installer and workstation setting |
| `integratedManufacturingSystem21-08-2026-1637pm.factoryio` | Factory I/O scene, components, camera presets, driver selection and 154 mappings | PLC code, Kepware server configuration, Ignition project or historical data |

## Ignition project-resource ZIP

### Restore/import

1. Open the target Ignition project in Designer.
2. Select **File → Import**.
3. Select the project ZIP.
4. Review the resource tree and collision policy.
5. Import the P12 Perspective Properties, Project Properties, Style Classes and Views.
6. Save the imported project to the Gateway.

The supplied export is internally complete for its P12 view/style references.

### What must be recreated separately

- `[Sample_Tags]P12/SCADA` tag folder and OPC item paths;
- tag permissions and write access;
- tank tag History Enabled settings;
- historian storage provider/database;
- Kepware OPC UA connection;
- any Gateway authentication, certificates or network settings.

Ignition documents that project exports do not include Gateway-level connections, Tag Providers or tags: [Project Export and Import](https://www.docs.inductiveautomation.com/docs/8.1/platform/projects/project-export-and-import).

### Better full-machine backup

Take a Gateway `.gwbk` backup in addition to the project ZIP when you need the complete Ignition environment. A Gateway restore overwrites the existing Gateway rather than merging, so back up the target before restoring: [Gateway Backup and Restore](https://www.docs.inductiveautomation.com/docs/8.1/platform/gateway/gateway-backup-and-restore).

## CODESYS `.export`

The supplied file is a native XML object export. It can be inspected in source control and imported into an open CODESYS project.

### Import

1. Use a compatible CODESYS installation.
2. Create/open the destination project and select the intended insertion point.
3. Select **Project → Import**.
4. Choose the `.export` file.
5. Review available objects and conflicts.
6. Import the Device/Application tree.
7. Check referenced libraries and target compatibility.
8. Build before connecting to a runtime.

Official reference: [Exporting and Importing Projects — CODESYS](https://content.helpme-codesys.com/en/CODESYS%20Development%20System/_cds_project_export_import.html).

### Better portable backup

For transfer to another workstation, also create a `.projectarchive`:

1. Select **File → Project Archive → Save/Send Archive**.
2. Include referenced devices, libraries, download information and relevant profiles.
3. Add the Factory I/O scene, mapping notes and documentation as additional files if useful.
4. Add a dated comment.
5. Save the archive beside the ordinary `.project` file.

CODESYS recommends a project/archive for exchange between development systems: [Saving/Sending the Project Archive](https://content.helpme-codesys.com/en/CODESYS%20Development%20System/_cds_saving_project_archive.html).

### Save strategy

Maintain all three:

- editable `.project` working file;
- dated `.projectarchive` recovery/transfer package;
- plain-text `.export` for inspection, documentation and source review.

## Factory I/O scene

### Open

1. Start Factory I/O.
2. Select **File → Open**.
3. Browse to the `.factoryio` file or copy it into the configured My Scenes directory.
4. Open the scene.
5. Open **Drivers**, select OPC Client DA and verify `PTC.KepwareServer`.
6. Check mapping quality before entering Run mode.

Factory I/O documents opening and maintaining customised scenes under My Scenes: [Scenes — Factory I/O](https://docs.factoryio.com/manual/scenes/).

### Save safely

- Use a dated filename before making structural or mapping changes.
- Keep the working scene and a known-good commissioning copy.
- Record the Factory I/O version.
- Export a mapping/tag list when making major I/O changes.
- Treat the file as binary in Git because its embedded base64 image makes text diffs noisy.

Suggested description to add inside the scene:

> Project 12 — Integrated Manufacturing System. CODESYS-controlled twin-cell machining, pair routing, robotic assembly, robotic packout, analogue tank batching and automated rack storage, connected through Kepware OPC DA.

## Kepware and OPC

Detailed Kepware configuration was intentionally left outside this documentation package. To make the workstation reproducible later, preserve at least:

- Kepware project/configuration export;
- channel and device names;
- CODESYS endpoint/device settings;
- OPC UA endpoint and certificate/trust notes;
- Factory I/O OPC DA server selection;
- Ignition OPC connection name;
- a mapping export or screenshots of write access and item paths.

## Recommended backup set

| Artifact | Frequency | Reason |
|---|---|---|
| CODESYS `.project` | Every work session | Editable source |
| CODESYS `.export` | Milestones | Reviewable object/source snapshot |
| Factory I/O `.factoryio` | Before mapping/scene changes | Physical plant and mapping recovery |
| Ignition project ZIP | UI milestones | Mergeable project resources |
| Kepware configuration | Before channel/item changes | Communications recovery |
| Test record/screenshots | Each release | Evidence that the integrated version worked |

## Naming convention

Use sortable timestamps and avoid ambiguous “final” filenames:

```text
P12_Codesys_2026-08-21_1637.export
P12_Codesys_2026-08-21_1637.projectarchive
P12_FactoryIO_2026-08-21_1637.factoryio
P12_Ignition_2026-08-21_1700.zip
P12_Kepware_2026-08-21_1700.opf
```

The aboive is primarliy for testing and restoration purposes, if any part of the chain fails, so that you can better stay aligned. All major files have now omitted the datestamp at the end of the file name, as can be seen throughout the project.

## Minimum restore verification

After restoring, do not assume success because the files opened. Repeat the communications, Auto, Reset, Start, full-cycle, historian and reserve/commit checks in [Commissioning and Testing](COMMISSIONING_AND_TESTING.md).

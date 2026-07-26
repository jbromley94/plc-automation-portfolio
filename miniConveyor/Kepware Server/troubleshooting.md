# KEPServerEX Troubleshooting

## Problem

After configuring KEPServerEX to communicate with a CODESYS Control Win V3 runtime, the PLC tags were successfully discovered and imported into KEPServerEX.

However, the tags displayed the following behaviour in the OPC Quick Client:

- **Value:** `Unknown`
- **Quality:** `Bad`
- **Update Count:** `1`

Changes made to the Boolean variables in CODESYS were not reflected in KEPServerEX.

---

## Initial Configuration

The system consisted of:

- CODESYS 3.5
- CODESYS Control Win V3 x64
- KEPServerEX
- CODESYS V3 Ethernet driver
- OPC Quick Client

The CODESYS variables were configured as Boolean values and exposed through the CODESYS Symbol Configuration.

Example variable:

```text
CODESYS:
Application.PLC_PRG.a

KEPServerEX:
Name: FIO_a
Address: Application.PLC_PRG.a
Data Type: Boolean
Client Access: Read/Write
Scan Rate: 100 ms
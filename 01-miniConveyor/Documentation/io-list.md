# I/O List

## Overview

This document lists the inputs, outputs and control signals used by the mini-conveyor PLC application.

The project uses CODESYS to control a simulated conveyor system. The PLC communicates with the simulated field devices through the configured I/O mapping.

---

## Inputs

| Address / Tag | Data Type | Description |
|---|---|---|
| `a` | BOOL | Part detection sensor |
| `b` | BOOL | Start / input sensor |
| `c` | BOOL | Stop / input sensor |
| `aa` | BOOL | Part detection / sensor input |
| `bb` | BOOL | Part detection / sensor input |
| `cc` | BOOL | Part detection / sensor input |

---

## Outputs

| Address / Tag | Data Type | Description |
|---|---|---|
| `...` | BOOL | Conveyor motor output |
| `...` | BOOL | Conveyor actuator |
| `...` | BOOL | Indicator / output |
| `...` | BOOL | Diverter / actuator |

---

## Internal Variables

| Variable | Data Type | Description |
|---|---|---|
| `...` | BOOL | Internal control state |
| `...` | BOOL | Conveyor sequence state |
| `...` | BOOL | Part detection state |

---

## Communication Tags

The following PLC variables are exposed through the CODESYS Symbol Configuration and monitored using KEPServerEX.

| CODESYS Variable | KEPServerEX Tag | Data Type | Description |
|---|---|---|---|
| `Application.PLC_PRG.a` | `FIO_a` | BOOL | Part detection sensor |
| `Application.PLC_PRG.aa` | `FIO_aa` | BOOL | Part detection sensor |
| `Application.PLC_PRG.b` | `FIO_b` | BOOL | Input signal |
| `Application.PLC_PRG.bb` | `FIO_bb` | BOOL | Input signal |
| `Application.PLC_PRG.c` | `FIO_c` | BOOL | Input signal |
| `Application.PLC_PRG.cc` | `FIO_cc` | BOOL | Input signal |

---

## Notes

All Boolean I/O signals are represented as `BOOL` values within the CODESYS application.

The selected variables are exposed through the CODESYS Symbol Configuration and accessed by KEPServerEX using the CODESYS V3 Ethernet driver.

KEPServerEX is configured to communicate with the CODESYS Control Win runtime through the CODESYS Gateway.

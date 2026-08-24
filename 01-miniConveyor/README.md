# Mini Conveyor Automation System

## Overview

This project is a simulated industrial conveyor control system developed using CODESYS.

The project demonstrates the use of PLC programming to control a conveyor process based on sensor inputs and actuator outputs. The PLC application was developed in CODESYS and executed using CODESYS Control Win V3.

KEPServerEX was integrated to provide an OPC communication layer between the CODESYS runtime and an OPC client, allowing PLC variables to be monitored in real time.

The project was developed as part of my portfolio to demonstrate practical PLC programming, industrial communication, troubleshooting and automation engineering concepts.

---

## Technologies

- CODESYS 3.5
- CODESYS Control Win V3 x64
- Structured Text / Ladder Logic
- KEPServerEX
- CODESYS V3 Ethernet Driver
- CODESYS Gateway
- OPC Quick Client
- Git / GitHub

---

## System Architecture

The overall communication architecture is:

```text
+-----------------------+
|     Conveyor System   |
|  Sensors & Actuators  |
+-----------+-----------+
            |
            v
+-----------------------+
|   CODESYS PLC Logic   |
|      PLC_PRG          |
+-----------+-----------+
            |
            v
+-----------------------+
| CODESYS Control Win   |
|        V3 x64         |
+-----------+-----------+
            |
            v
+-----------------------+
|   CODESYS Gateway     |
|   127.0.0.1 : 1217   |
+-----------+-----------+
            |
            v
+-----------------------+
|      KEPServerEX      |
| CODESYS V3 Ethernet   |
+-----------+-----------+
            |
            v
+-----------------------+
|   OPC Quick Client    |
| Live PLC Monitoring   |
+-----------------------+
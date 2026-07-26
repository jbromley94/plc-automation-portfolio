# KEPServerEX Integration

## Overview

This project uses KEPServerEX to expose PLC variables from the CODESYS
Control Win runtime for monitoring through an OPC client.

## Architecture

CODESYS Control Win
        |
        | CODESYS V3 Ethernet
        |
CODESYS Gateway
127.0.0.1:1217
        |
        |
KEPServerEX
        |
        | OPC
        |
OPC Quick Client

## Configuration

Driver:
CODESYS

Device Model:
CODESYS V3 Ethernet

Address Type:
Logical Address / PLC Name

Gateway:
Enabled

Gateway Address:
127.0.0.1

Gateway Port:
1217

## Example Tag

| KEPServerEX Tag | CODESYS Variable | Type |
|---|---|---|
| FIO_a | Application.PLC_PRG.a | Boolean |

## Validation

The connection was verified using OPC Quick Client.

- Quality: Good
- Values updated correctly
- Update count increased when PLC variables changed
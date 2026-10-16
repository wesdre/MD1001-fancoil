# MD1001 Fancoil

Information about fancoils with the **MD1001** controller board (such as the
Climasol LTV-080/CW) and how to integrate them into Home Assistant over
RS485 / Modbus RTU.

| Folder | Contents |
|---|---|
| [`esphome/`](esphome/) | ESPHome configuration for an M5Stack Atom Lite + Atomic RS485 Base |
| [`documents/`](documents/) | [Modbus register map of the MD1001](documents/MD1001-modbus-registers.md) |

## In short

- Modbus RTU, 9600 8N1, slave address 15
- Read with function code 03, write with 16 (06 is not supported)
- Readable block 28301–28327: power, mode, fan speed, setpoints and all panel
  parameters P01–P20
- The board answers with address byte `0x00`, which standard Modbus libraries
  reject – the ESPHome configuration handles that itself

## License

[GPL-3.0](LICENSE)

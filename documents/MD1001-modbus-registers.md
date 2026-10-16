# MD1001 fancoil controller – Modbus register map

Reverse-engineered in October 2026 on a **Climasol LTV-080/CW** water-cooled
fancoil with an **MD1001-type controller board**. The full 16-bit address range
was scanned register by register, and each readable register was then matched
to a function on the control panel. The "measured" column comes from a single
read of the whole block.

Other fancoils built around the same MD1001 board will very likely use the same
map, but that has not been verified.

## Device

| | |
|---|---|
| Model | Climasol LTV-080/CW, water-cooled fancoil |
| Capacity | 3500 W cooling, 7200 W heating, 580 m³/h, 25 W |
| Controller | MD1001 or a close variant |

## Quick start

To rebuild this you only need four things.

1. Connect to the separate **A** and **B** terminals of the unit. Connect
   **GND** as well.
2. Talk Modbus RTU at **9600 baud, 8 data bits, no parity, 1 stop bit**,
   slave address **15**.
3. Read the block **28301 to 28327** with function code **03**. Write only with
   function code **16**; function code 06 does not exist on this board.
4. **Ignore the address byte in the reply.** It is always `0x00`, not `0x0F`.

## Connection

| | |
|---|---|
| Physical layer | RS485, separate A and B terminals on the unit |
| Protocol | Modbus RTU |
| Serial settings | 9600 baud, 8 data bits, no parity, 1 stop bit |
| Slave address | 15, set with parameter P14 on the panel |
| Response time | about 28 ms after the end of the request |

## Supported function codes

| Code | Function | Supported |
|---|---|---|
| 0x03 | Read holding registers | yes |
| 0x10 | Write multiple registers | yes, also with count 1 |
| 0x01 | Read coils | no, exception 2 on every address |
| 0x02 | Read discrete inputs | no, exception 2 on every address |
| 0x04 | Read input registers | no, exception 2 on every address |
| 0x06 | Write single register | **no, exception 1, illegal function** |
| 0x11 | Report server ID | no, no reply |
| 0x2B/14 | Read device identification | no, no reply |

So: read only with 03, write only with 16.

## Quirk in the reply frame

The board puts address byte `0x00` in its reply instead of `0x0F`. The CRC is
calculated over that `0x00` and is valid, so this is not line noise but how the
firmware answers.

```
sent       0F 03 00 00 00 08 45 22     read 8 registers from 0, slave 15
received   00 83 02 91 31              exception 2, address byte is 00
```

The CRC over `0F 83 02` would be `A1 32`, while the frame contains `91 31`,
which is exactly the CRC over `00 83 02`.

Consequence: every Modbus library that checks the reply address – nearly all
of them, including ESPHome's `modbus_controller` – rejects every reply from
this device. Frames have to be built and parsed by hand, which is what the
ESPHome configuration in this repository does.

## Example frames

```
read the whole block, 27 registers from 28301
  0F 03 6E 8D 00 1B 88 2C

read one register, 28301
  0F 03 6E 8D 00 01 09 E7

write 23.0 °C to 28311, heating setpoint
  0F 10 6E 97 00 01 02 00 E6 F8 9B

write 0 to 28301, unit off
  0F 10 6E 8D 00 01 02 00 00 7B 2B
```

## Register map

The only readable area is **28301 to 28327** (0x6E8D to 0x6EA7), 27 registers.
Every other address in the full range 0 to 65535 returns exception 2.

For the parameters: **register = 28307 + parameter number**, where the
parameter number is the one shown on the control panel.

All temperatures are in tenths of a degree Celsius: 250 means 25.0 °C.

The "write" column only says "tested" for registers that were actually written
with confirmation on the panel display.

| Register | Hex | Par. | Meaning | Values | Measured | Write |
|---|---|---|---|---|---|---|
| 28301 | 0x6E8D | – | Power | 0 = off, 1 = on | 1 | tested |
| 28302 | 0x6E8E | – | Mode | 0 = auto, 1 = cool, 2 = dry, 3 = fan only, 4 = heat | 4 | tested |
| 28303 | 0x6E8F | – | Fan speed | 2 = low, 3 = medium, 4 = high, 6 = auto. 5 does not exist | 2 | tested |
| 28304 | 0x6E90 | – | **unknown** | always 0 | 0 | not tested |
| 28305 | 0x6E91 | – | **unknown** | always 0 | 0 | not tested |
| 28306 | 0x6E92 | – | **unknown** | always 0 | 0 | not tested |
| 28307 | 0x6E93 | – | **unknown** | always 0 | 0 | not tested |
| 28308 | 0x6E94 | P01 | Maximum setpoint | tenths °C | 300 | not tested |
| 28309 | 0x6E95 | P02 | Minimum setpoint | tenths °C | 180 | not tested |
| 28310 | 0x6E96 | P03 | Cooling setpoint | tenths °C | 195 | tested |
| 28311 | 0x6E97 | P04 | Heating setpoint | tenths °C | 200 | tested |
| 28312 | 0x6E98 | P05 | Auto mode setpoint, cooling side | tenths °C | 260 | tested |
| 28313 | 0x6E99 | P06 | Auto mode setpoint, heating side | tenths °C | 250 | tested |
| 28314 | 0x6E9A | P07 | Coil temperature below which the fan stops when heating | tenths °C | 190 | not tested |
| 28315 | 0x6E9B | P08 | Stop the fan when the cooling coil is too warm | 0 or 1 | 1 | not tested |
| 28316 | 0x6E9C | P09 | Allow ultra-low fan speed | 0 or 1 | 0 | not tested |
| 28317 | 0x6E9D | P10 | Water valve or 24 V relay present | 0 or 1 | 0 | not tested |
| 28318 | 0x6E9E | P11 | Floor heating | 0 or 1 | 0 | not tested |
| 28319 | 0x6E9F | P12 | Display in Fahrenheit | 0 = Celsius, 1 = Fahrenheit | 0 | not tested |
| 28320 | 0x6EA0 | P13 | This unit is the master controller | 0 or 1 | 0 | not tested |
| 28321 | 0x6EA1 | P14 | Slave address | 0 to 99 | 15 | not tested |
| 28322 | 0x6EA2 | P15 | Key lock | 0 or 1 | 0 | not tested |
| 28323 | 0x6EA3 | P16 | Remote control works while the panel is locked | 0 or 1 | 1 | not tested |
| 28324 | 0x6EA4 | P17 | Password required to lock | 0 or 1 | 0 | not tested |
| 28325 | 0x6EA5 | P18 | Password 1 | 1 to 50 | 50 | not tested |
| 28326 | 0x6EA6 | P19 | Password 2 | 1 to 50 | 50 | not tested |
| 28327 | 0x6EA7 | P20 | Fan pulses in auto mode | 0 or 1 | 0 | not tested |

## Which setpoint belongs to which mode

The unit has four setpoints and uses one at a time, depending on register
28302.

| Mode in 28302 | Active setpoint | Register |
|---|---|---|
| 4, heat | P04 | 28311 |
| 1, cool | P03 | 28310 |
| 2, dry | P03 | 28310 |
| 0, auto | P05 and P06 | 28312 and 28313 |
| 3, fan only | none, no temperature control | – |

Only the heat row is proven beyond doubt: after writing 230 to 28311 the
display showed 23 °C in heat mode. The other rows follow from the parameter
descriptions in the manual and are plausible but not independently confirmed.

A written setpoint is clamped by P02 and P01, on the test unit 18.0 and
30.0 °C. Values outside that range were not tried.

## How the mapping was confirmed

The register-to-parameter mapping is fixed because three values were known
independently before the registers were read:

- P13 had just been changed by hand from 1 to 0 on the panel. Register 28320
  read 0.
- P14 is the slave address 15 the unit answers on. Register 28321 read 15.
- P18 and P19 are both at their factory value of 50. Registers 28325 and 28326
  both read 50.

The three control registers were confirmed in both directions: first by
operating the panel while reading the block, where exactly one register moved
each time, and then the other way round, by writing from the bus and checking
on the display that the setting really changed. The meaning of each mode and
fan speed value was read off the display that way.

## Writing

Writing only works with function code 16. Function code 06 returns exception
1, illegal function.

For register 28311 a range of values was tried. All were accepted and stayed,
also when read again up to forty seconds later:

| Written | Read back |
|---|---|
| 220 | 220 |
| 200 | 200 |
| 190 | 190 |
| 180 | 180 |
| 250 | 250 |
| 260 | 260 |

Over dozens of writes no write was ever rejected.

The board stores a new setpoint immediately, but appears to apply it to its
control loop only after another event. Writing the current mode to 28302 again
right after a setpoint change makes it reload; the ESPHome configuration does
this automatically.

## Completeness of the scan

The full 16-bit address range was scanned register by register with function
code 03 and count 1:

| | |
|---|---|
| Addresses scanned | 65536, i.e. all |
| Exception 02, illegal data address | 65509 |
| Readable registers | 27 |
| No reply | 0 |

65509 plus 27 is exactly 65536. Every address answered and the only readable
registers are the block 28301 to 28327.

Two extra checks:

- **Slave addresses 1 to 247 scanned.** Only address 15 answers, so there is no
  second logical device, such as a separate sensor module, on the bus.
- **Device identification tried** with function code 17 and with function code
  43 subcode 14. Neither answers.

## What is missing

The **measured room temperature** is not in the block. The panel shows it, so
the board has it, but it is not available at any address. The same goes for the
**coil temperature**, which the board clearly uses (see P07 and P08).

There is also no **status register** telling whether the unit is actually
heating or cooling at that moment. When the setpoint was lowered from 250 to
180, the only register in the whole block that changed was 28311.

The meaning of **28304 to 28307** is unknown. They read 0 in every mode and at
every fan speed.

## Note on regulation without a valve

With P10 at 0 the board drives no water valve or relay. The unit then cannot
shut off the hot water and regulates with the fan only. With the fan speed on
auto the fan usually keeps running and only changes speed, so the unit keeps
giving off heat as long as hot water flows through the coil.

Setting **28301 to 0** is therefore the reliable way to stop the unit – more
reliable than lowering the setpoint.

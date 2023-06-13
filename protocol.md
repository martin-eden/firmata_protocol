# Firmata protocol

## Preface

This is my interpretation of "Firmata" protocol. It is based on
[Firmata protocol v2.6.0][proto] and [C++ implementation for Arduino][impl].

[proto]: https://github.com/firmata/protocol/blob/master/protocol.md
[impl]: https://github.com/firmata/arduino

----------------------------------------------------------------------

## Protocol

Firmata is command-data protocol with byte granularity. All "command" bytes
have 8-th bit set, so lie in range `80`..`FF`. All data bytes have 8-th bit
clear, so lie in range `00`..`7F`.

There are two types of Firmata commands: fixed-length and variable-length.

Fixed-length command is command byte and fixed number of data bytes.
Number of data bytes depends of command.

Variable-length command is embraced between `F0` (Sysex.Start) and `F7`
(Sysex.End) bytes. Inside there are command byte and a variable number
of data bytes.

----------------------------------------------------------------------

## Commands

* Pin management
  * Mode
    * [<] [List all pins, all modes](#get_pins_modes)
    * [<] [List analog pins](#get_analog_pins_mapping)
    * [>] [Set pin mode](#set_pin_mode)
  * Value
    * [<] [Get pin state](#get_pin_state)
    * [>] [Set digital value](#set_pin_value_digital)
    * [>] [Set analog value](#set_pin_value_analog)
  * Reporting
    * [>] [Set analog report interval](#set_sampling_interval)
    * [>] [Setup digital port value reporting](#setup_digital_port_reporting)
    * [>] [Setup analog pin value reporting](#setup_analog_pin_reporting)
* [I2C](#i2c_preface)
  * [>] [Init](#i2c_init)
  * [<] [Read](#i2c_read)
  * [>] [Write](#i2c_write)
* Misc
  * [>] [Reset](#reset)
  * [<] [Get version](#get_firmware_version)
  * [<] [Get version (v2)](#get_firmware_name_version)
  * [<] [String reply](#string_reply)

----------------------------------------------------------------------

### Set pin mode <a name="set_pin_mode"/>

Set mode for given pin. Usually mode is input or output.

```
  ╭────╮ ╭───────╮ ╭──────╮
→ │ F4 ├─┤ Pin # ├─┤ Mode │
  ╰────╯ ╰───────╯ ╰──────╯
```

```
← ∅
```

__Mode__<a name="pin_modes"/>

  Value | Description
 -------|-----------------
  `00`  | Digital input
  `01`  | Digital output
  `02`  | Analog input
  `03`  | PWM
  `04`  | Servo
  `05`  | Shift
  `06`  | I2C
  `07`  | One-wire
  `08`  | Stepper
  `09`  | Encoder
  `0A`  | Serial
  `0B`  | Digital input-pullup

----------------------------------------------------------------------

### Get pins modes <a name="get_pins_modes"/>

For each pin on board list all modes it can support.

```
  ╭────╮ ╭────╮ ╭────╮
→ │ F0 ├─┤ 6B ├─┤ F7 │
  ╰────╯ ╰────╯ ╰────╯
```

```
  ╭────╮ ╭────╮                                   ╭────╮   ╭────╮
← │ F0 ├─┤ 6C ├─┬─┬─────────────────────────────┬─┤ 7F ├─┬─┤ F7 │
  ╰────╯ ╰────╯ │ │   ╭──────╮ ╭────────────╮   │ ╰────╯ │ ╰────╯
                │ ╰─┬─┤ Mode ├─┤ Resolution ├─┬─╯        │
                │   ↑ ╰──────╯ ╰────────────╯ │          │
                │   ╰───── #(Pin modes) ──────╯          │
                ╰────────────── #Pins ───────────────────╯
```

[__Mode__](#pin_modes) - same mapping as for "Set pin mode".

__Resolution__ - number of bits in value for given mode.

  Mode # | Mode name             | Resolution bits
 --------|-----------------------|----------------
   0     | Digital input         | 1
   1     | Digital output        | 1
   2     | Analog input          | 10
   3     | PWM                   | 8
   4     | Servo                 | 14
   6     | I2C                   | 1
   10    | Serial                | 4: Bit.0: 0 - RX, 1 - TX, Bit.1 .. Bit.3 - UART port number.
   11    | Digital input-pullup  | 1

----------------------------------------------------------------------

### Get analog pins mapping <a name="get_analog_pins_mapping"/>

Returns byte array _A_ with length of _#Pins_. For given index _i_,
(A[i] == 7F) means that pin _i_ doesn't support analog mode. Other
value of A[i] means __analog pin index__: 0 - A0, 1 - A1, etc.

```
  ╭────╮ ╭────╮ ╭────╮
→ │ F0 ├─┤ 69 ├─┤ F7 │
  ╰────╯ ╰────╯ ╰────╯
```

```
  ╭────╮ ╭────╮     ╭──────────────────╮     ╭────╮
← │ F0 ├─┤ 6A ├─┬─┬─┤ Analog pin index ├─┬─┬─┤ F7 │
  ╰────╯ ╰────╯ ↑ │ ╰──────────────────╯ │ │ ╰────╯
                │ │ ╭────╮               │ │
                │ ╰─┤ 7F ├───────────────╯ │
                │   ╰────╯                 │
                ╰──────── #Pins ───────────╯
```

----------------------------------------------------------------------

### Set digital pin value <a name="set_pin_value_digital"/>

```
  ╭────╮ ╭───────╮ ╭────────────────╮
→ │ F5 ├─┤ Pin # ├─┤ LOW/HIGH (0/1) │
  ╰────╯ ╰───────╯ ╰────────────────╯
```

```
← ∅
```

----------------------------------------------------------------------

### Set analog pin value <a name="set_pin_value_analog"/>

```
  ╭────╮ ╭────╮ ╭───────╮ ╭──────────────────╮           ╭────╮
→ │ F0 ├─┤ 6F ├─┤ Pin # ├─┤ Value.Bit.0 .. 6 ├──────┬────┤ F7 │
  ╰────╯ ╰────╯ ╰───────╯ ╰─┬────────────────╯      │    ╰────╯
                            │ ╭───────────────────╮ │
                            ╰─┤ Value.Bit.7 .. 13 ├─┤
                              ╰─┬─────────────────╯ │
                                │                   │
                                ╰─ ... ─────────────╯
```

```
← ∅
```

----------------------------------------------------------------------

### Setup digital port value reporting <a name="setup_digital_port_reporting"/>

```
  ╭───────────────╮ ╭──────────────────────╮
→ │ D0 + (Port #) ├─┤ Enable/disable (1/0) │
  ╰───────────────╯ ╰──────────────────────╯
```
If enabled you will start getting following message every time firmware
main loop executed (as fast as possible):
```
     ╭───────────────╮ ╭──────────────────────╮ ╭───────────────────────╮
← ─┬─┤ 90 + (Port #) ├─┤ PortValue.Bit.0 .. 6 ├─┤ PortValue.Bit.7 (0/1) ├─╮
   ↑ ╰───────────────╯ ╰──────────────────────╯ ╰───────────────────────╯ │
   ╰──────────────────────────────────────────────────────────────────────╯
```

__Port #__ - value between 0 and 15.

"Port value" means value of eight digital pins. Port 0 describes pins 0 to 7,
port 1 -- pins 8 to 15 and so on.

Command capacity is 16 ports or 128 digital pins.

----------------------------------------------------------------------

### Setup analog pin value reporting <a name="setup_analog_pin_reporting"/>

```
  ╭─────────────────────╮ ╭──────────────────────╮
→ │ C0 + (Analog pin #) ├─┤ Enable/disable (1/0) │
  ╰─────────────────────╯ ╰──────────────────────╯
```
If enabled, you will start getting this message every 19ms. Analog query delay
may be changed via [set sampling interval](#set_sampling_interval) command.
```
     ╭─────────────────────╮ ╭────────────╮ ╭─────────────╮
← ─┬─┤ E0 + (Analog pin #) ├─┤ Bit.0 .. 6 ├─┤ Bit.7 .. 13 ├─ delay ─╮
   ↑ ╰─────────────────────╯ ╰────────────╯ ╰─────────────╯         │
   ╰────────────────────────────────────────────────────────────────╯
```

__Analog pin #__ - value between 0 and 15. 0 means A0, 1 - A1, etc.

----------------------------------------------------------------------

### Get pin state <a name="get_pin_state"/>

```
  ╭────╮ ╭────╮ ╭───────╮ ╭────╮
→ │ F0 ├─┤ 6D ├─┤ Pin # ├─┤ F7 │
  ╰────╯ ╰────╯ ╰───────╯ ╰────╯
```

```
  ╭────╮ ╭────╮ ╭───────╮ ╭──────────╮ ╭──────────────────╮        ╭────╮
← │ F0 ├─┤ 6E ├─┤ Pin # ├─┤ Pin mode ├─┤ State.Bit.0 .. 6 ├──────┬─┤ F7 │
  ╰────╯ ╰────╯ ╰───────╯ ╰──────────╯ ╰─┬────────────────╯      │ ╰────╯
                                         │ ╭───────────────────╮ │
                                         ╰─┤ State.Bit.7 .. 13 ├─┤
                                           ╰─┬─────────────────╯ │
                                             │                   │
                                             ╰─ ... ─────────────╯
```

__State__ - pin state value, meaning depends of pin mode:

  Mode                                | Value or meaning
  ------------------------------------|----------------------------------------
  Digital output, PWM, servo          | Value previously written to pin
  Analog input                        | 0
  Digital input, digital input-pullup | 0/1 - pullup resistor enabled/disabled

----------------------------------------------------------------------

## I2C <a name="i2c_preface"/>

Firmata protocol has extension for working with I2C devices.

Here I'll describe just core functionality needed for work. Although
protocol supports 10-bit device addresses and continious reading, these
will not be decribed now. Official description is available [there](https://github.com/firmata/protocol/blob/master/i2c.md).

We have some _DeviceId_ - 7-bit integer, _DataOffset_ - 8-bit integer and
_DataLength_ - 8-bit integer.

Before any reads or writes you need to send [init command](#i2c_init).

----------------------------------------------------------------------

### I2C Init <a name="i2c_init"/>

```
  ╭────╮ ╭────╮ ╭────╮
→ │ F0 ├─┤ 78 ├─┤ F7 │
  ╰────╯ ╰────╯ ╰────╯
```

```
← ∅
```

----------------------------------------------------------------------

### I2C read <a name="i2c_read"/>

```
  ╭────╮╭────╮╭──────────╮╭───╮╭───────────────────────╮╭──────────────────╮╭───────────────────────╮╭──────────────────╮╭────╮
→ │ F0 ├┤ 76 ├┤ DeviceId ├┤ 8 ├┤ DataOffset.Bit.0 .. 6 ├┤ DataOffset.Bit.7 ├┤ DataLength.Bit.0 .. 6 ├┤ DataLength.Bit.7 ├┤ F7 │
  ╰────╯╰────╯╰──────────╯╰───╯╰───────────────────────╯╰──────────────────╯╰───────────────────────╯╰──────────────────╯╰────╯
```

If there is no error, response is
```
  ╭────╮╭────╮╭──────────╮╭───╮╭───────────────────────╮╭──────────────────╮ ╭─────────────────╮╭────────────╮ ╭────╮
← │ F0 ├┤ 77 ├┤ DeviceId ├┤ 0 ├┤ DataOffset.Bit.0 .. 6 ├┤ DataOffset.Bit.7 ├┬┤ Data.Bit.0 .. 6 ├┤ Data.Bit.7 ├┬┤ F7 │
  ╰────╯╰────╯╰──────────╯╰───╯╰───────────────────────╯╰──────────────────╯↑╰─────────────────╯╰────────────╯│╰────╯
                                                                            ╰─────────────────────────────────╯
                                                                                        #DataLength
```

If there _is_ error, there will be [string message](#string_reply) with
text like `I2C: Too many bytes received` or `I2C: Too few bytes received`
_AND_ normal response with data.

----------------------------------------------------------------------
### I2C write <a name="i2c_write"/>

```
  ╭────╮╭────╮╭──────────╮╭───╮╭───────────────────────╮╭──────────────────╮ ╭─────────────────╮╭────────────╮ ╭────╮
→ │ F0 ├┤ 76 ├┤ DeviceId ├┤ 0 ├┤ DataOffset.Bit.0 .. 6 ├┤ DataOffset.Bit.7 ├┬┤ Data.Bit.0 .. 6 ├┤ Data.Bit.7 ├┬┤ F7 │
  ╰────╯╰────╯╰──────────╯╰───╯╰───────────────────────╯╰──────────────────╯↑╰─────────────────╯╰────────────╯│╰────╯
                                                                            ╰─────────────────────────────────╯
                                                                                        #DataLength
```

```
← ∅
```
----------------------------------------------------------------------

### System reset <a name="reset"/>

Reset to initial state and execute initialization sequence.

```
  ╭────╮
→ │ FF │
  ╰────╯
```

```
← ∅
```

----------------------------------------------------------------------

### Get firmware version <a name="get_firmware_version"/>

Report major and minor version of protocol in two 7-bit bytes.

```
  ╭────╮
→ │ F9 │
  ╰────╯
```

```
  ╭────╮ ╭───────────────╮ ╭───────────────╮
← │ F9 ├─┤ Major version ├─┤ Minor version │
  ╰────╯ ╰───────────────╯ ╰───────────────╯
```

----------------------------------------------------------------------

### Get firmware name and version <a name="get_firmware_name_version"/>

Every byte of ASCII name is split in two: first byte contains lower 7
bits, second contains 8-th bit (at bit 0).

```
  ╭────╮ ╭────╮ ╭────╮
→ │ F0 ├─┤ 79 ├─┤ F7 │
  ╰────╯ ╰────╯ ╰────╯
```

```
  ╭────╮ ╭────╮ ╭───────────────╮ ╭───────────────╮                                           ╭────╮
→ │ F0 ├─┤ 79 ├─┤ Major version ├─┤ Minor version ├──┬──────────────────────────────────────┬─┤ F7 │
  ╰────╯ ╰────╯ ╰───────────────╯ ╰───────────────╯  │   ╭─────────────────╮ ╭────────────╮ │ ╰────╯
                                                     ╰─┬─┤ Char.Bit.0 .. 6 ├─┤ Char.Bit.7 ├─┤
                                                       ↑ ╰─────────────────╯ ╰────────────╯ │
                                                       ╰──────── #(Firmware name) ──────────╯
```

__Firmware name__ - name of the Firmata client file, minus the file
extension. So for `StandardFirmata.ino` _firmware_name_ is
`StandardFirmata`. But in practice name is __with__ `.ino` extension
as implementation code strips only `.cpp` extension.

----------------------------------------------------------------------

### Set analog sampling interval <a name="set_sampling_interval"/>

Set the interval at which analog pins with enabled reporting are queried.

Default value is __19 ms__.

Used in [enable/disable analog pin value reporting](#analog_pin_reporting)
command.

```
  ╭────╮ ╭────╮ ╭────────────────────────╮ ╭─────────────╮ ╭────╮
→ │ F0 ├─┤ 7A ├─┤ Sampling_ms.Bit.0 .. 6 ├─┤ Bit.7 .. 13 ├─┤ F7 │
  ╰────╯ ╰────╯ ╰────────────────────────╯ ╰─────────────╯ ╰────╯
```

```
← ∅
```

----------------------------------------------------------------------

### String reply <a name="string_reply"/>

This is possible reply to some commands. Typically it means that error
occurred and string contains reason.

Maximum message length for Arduino Uno version is 30 characters.

```
→ ∅
```

```
  ╭────╮ ╭────╮                                             ╭────╮
← │ F0 ├─┤ 71 ├──┬────────────────────────────────────────┬─┤ F7 │
  ╰────╯ ╰────╯  │   ╭─────────────────╮ ╭────────────╮   │ ╰────╯
                 ╰─┬─┤ Char.Bit.0 .. 6 ├─┤ Char.Bit.7 ├─┬─╯
                   ↑ ╰─────────────────╯ ╰────────────╯ │
                   ╰──────────── #Message ──────────────╯

```

----------------------------------------------------------------------

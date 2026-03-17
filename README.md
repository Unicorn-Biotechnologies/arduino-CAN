# Arduino CAN

An Arduino library for sending and receiving data using CAN bus via a [Microchip MCP2515](http://www.microchip.com/wwwproducts/en/en010406) SPI CAN controller.

> **Fork note:** This is a fork of [sandeepmistry/arduino-CAN](https://github.com/sandeepmistry/arduino-CAN). The ESP32 built-in SJA1000 CAN controller is **not supported** — only MCP2515-based hardware is. See the divergences section below for other changes.

## Compatible Hardware

* MCP2515-based boards and shields
  * [Arduino MKR CAN shield](https://store.arduino.cc/arduino-mkr-can-shield)

## Wiring (MCP2515)

| MCP2515 | Arduino |
| :-----: | :-----: |
| VCC | 5V |
| GND | GND |
| SCK | SCK |
| SO | MISO |
| SI | MOSI |
| CS | 10 (ESP32: 5) |
| INT | 2 (ESP32: 34) |

`CS` and `INT` pins can be changed with `CAN.setPins(cs, irq)`. `INT` is only needed for receive callback mode and must be interrupt-capable.

**Note:** Logic level converters are required for 3.3 V boards.

**Note:** On ESP32, `onReceive()` callbacks do not use `usingInterrupt()` (not supported by the ESP32 Arduino core). Avoid accessing SPI directly inside the callback — use a flag instead.

## Usage

The `CAN` global is an `MCP2515Class` constructed with the default `SPI` object. For a different SPI bus, construct your own instance:

```cpp
#include <MCP2515.h>

SPIClass spi{HSPI};
MCP2515Class myCAN{spi};
```

### Basic send

```cpp
#include <CAN.h>

CAN.begin(500E3);

CAN.beginPacket(0x12);
CAN.write(0xDE);
CAN.write(0xAD);
CAN.endPacket();
```

### Basic receive (polling)

```cpp
int packetSize = CAN.parsePacket();
if (packetSize) {
    long id = CAN.packetId();
    while (CAN.available()) {
        CAN.read();
    }
}
```

## API

See [API.md](API.md) for the full base API. Changes and additions specific to this fork are documented below.

### Constructor

```cpp
explicit MCP2515Class(SPIClass& spi);
```

`SPIClass` is injected rather than using the global `SPI` object. The pre-built global `CAN` uses `SPI`.

### `begin`

```cpp
int begin(long baudRate);
int begin(long baudRate, bool stayInConfigurationMode);
```

`stayInConfigurationMode = true` leaves the chip in configuration mode after init, allowing filter registers to be configured before going on-bus. Call `switchToNormalMode()` when ready.

RXB0 overflow rollover into RXB1 is **enabled by default**.

### Mode switching

```cpp
bool switchToNormalMode();
bool switchToConfigurationMode();
```

Exposed as public methods for use alongside `setFilterRegisters()` or `begin(..., true)`.

### TX timeout

```cpp
void setTxTimeout(unsigned timeout);  // milliseconds, default 50
```

`endPacket()` aborts transmission and returns failure if the MCP2515 does not complete (or fail) the send within `timeout` ms. This prevents blocking indefinitely when the bus is disconnected.

### Low-level filter configuration

```cpp
boolean setFilterRegisters(
    uint16_t mask0,   uint16_t filter0, uint16_t filter1,
    uint16_t mask1,   uint16_t filter2, uint16_t filter3,
    uint16_t filter4, uint16_t filter5,
    bool allowRollover);
```

Direct access to all MCP2515 mask and filter registers for standard (11-bit) IDs.

- `mask0` applies to `filter0` and `filter1` (RXB0).
- `mask1` applies to `filter2`–`filter5` (RXB1).
- `allowRollover` controls whether RXB0 overflows into RXB1.

Switches to configuration mode internally; returns `false` if the mode switch fails.

### Diagnostics

```cpp
void dumpImportantRegisters(Stream& out);  // TEC, REC, CANINTE, CANINTF, EFLG with decoded flags
void dumpRegisters(Stream& out);           // All 128 registers as hex
```

## Divergences from upstream

| Area | Change |
|---|---|
| ESP32 SJA1000 | Removed entirely (`ESP32SJA1000.cpp/h` deleted) |
| Constructor | `SPIClass&` injected; no longer uses global `SPI` implicitly |
| `begin()` | Added `stayInConfigurationMode` overload |
| TX | Timeout + abort on bus error (`setTxTimeout`) |
| Filters | `setFilterRegisters()` for full mask/filter control |
| Mode | `switchToNormalMode()` / `switchToConfigurationMode()` are public |
| Receive callback | `usingInterrupt` skipped on ESP32 |
| Default pins (ESP32) | CS = 5, INT = 34 |
| RXB0 rollover | Enabled by default in `begin()` |
| Diagnostics | `dumpImportantRegisters()` added |

## Examples

See [examples](examples) folder.

## License

This library is [licensed](LICENSE) under the [MIT Licence](http://en.wikipedia.org/wiki/MIT_License).

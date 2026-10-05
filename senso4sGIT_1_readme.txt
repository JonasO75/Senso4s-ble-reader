# Senso4s BLE Reader for ESPHome

An ESPHome configuration for receiving and decoding **Senso4s BLE advertising packets**.

The project is intended for monitoring Senso4s gas scales locally using an ESP32 with Bluetooth support. It does **not** communicate with the Senso4s device through GATT and does not modify or configure the scale.

The configuration simply listens for the BLE manufacturer advertisements broadcast by the Senso4s.

## Features

* Detects Senso4s BLE advertisements
* Supports multiple Senso4s devices
* Reads gas level
* Reads Senso4s battery level
* Detects Senso4s model
* Detects usage mode
* Exposes the raw B2/B3 value
* Provides an experimental remaining-time estimate
* Logs the complete BLE packet for further reverse engineering
* Works with both observed Senso4s PLUS and BASIC packets

## Requirements

* ESP32 with Bluetooth Low Energy support
* ESPHome
* Wi-Fi connection
* A Senso4s BLE gas scale

The example configuration uses:

* ESP32-S3
* ESP-IDF framework
* ESPHome

Other ESP32 boards with BLE support should also be usable, provided they are supported by ESPHome.

## Installation

Copy the YAML configuration to your ESPHome configuration directory.

For example:

```text
senso4s-ble-reader.yaml
```

The configuration expects the following secrets:

```yaml
wifi_ssid
wifi_password
ota_password
```

These should be defined in your ESPHome `secrets.yaml`.

Example:

```yaml
wifi_ssid: "MyWiFi"
wifi_password: "MyPassword"
ota_password: "MyOTAPassword"
```

Do not put real passwords directly into the YAML file when publishing it to GitHub.

## Senso4s BLE packet

Observed Senso4s advertisements contain a 12-byte manufacturer-data payload.

Manufacturer ID:

```text
09CC
```

Observed packet layout:

```text
B0 B1 B2 B3 B4 B5 B6 B7 B8 B9 B10 B11
```

### Verified fields

| Byte   | Meaning                    | Status       |
| ------ | -------------------------- | ------------ |
| B0     | Flags / model + usage mode | Verified     |
| B1     | Gas level (%)              | Verified     |
| B2     | Part of prognosis value    | Experimental |
| B3     | Part of prognosis value    | Experimental |
| B4     | Battery level (%)          | Verified     |
| B5     | Unknown                    | Not decoded  |
| B6-B11 | Device MAC address         | Verified     |

## B0 - Model and usage mode

B0 contains two separate pieces of information.

The upper nibble is used to identify the model:

```text
B0 >> 4
```

Observed interpretation:

```text
0x0 - 0x7   PLUS
0x8         BASIC
other       UNKNOWN
```

For example, an observed BASIC packet was:

```text
81 56 FF FF 48 00 E3 53 6C 63 03 AA
```

Here:

```text
B0 = 81
```

The upper nibble is:

```text
8
```

which identifies the device as BASIC.

The lower nibble contains the usage mode:

```text
B0 & 0x0F
```

Observed values:

| Value | Usage       |
| ----: | ----------- |
|     1 | BBQ         |
|     2 | Camping     |
|     3 | Caravanning |
|     4 | Heating     |
|     5 | Household   |

Other values are currently reported as `UNKNOWN`.

## B1 - Gas level

B1 contains the gas level as a percentage.

Example:

```text
B1 = 56
```

means:

```text
Gas level = 86 %
```

Values from 0 to 100 are published as the ESPHome sensor:

```text
Senso4s Gas Level
```

## B4 - Battery level

B4 contains the battery level as a percentage.

Example:

```text
B4 = 48
```

means:

```text
Battery = 72 %
```

The value is published as:

```text
Senso4s Battery
```

## B6-B11 - Device MAC address

The last six bytes contain the device MAC address.

For example:

```text
B6-B11:

E3 53 6C 63 03 AA
```

corresponds to:

```text
E3:53:6C:63:03:AA
```

This has been verified against the BLE address of an observed Senso4s device.

The YAML does not filter on MAC address. This allows it to detect multiple Senso4s devices.

## B2/B3 - Experimental prognosis value

B2 and B3 form a 16-bit little-endian value:

```text
value = B2 + (B3 * 256)
```

The resulting value is exposed as:

```text
Senso4s Prognosis Counter
```

Observed data suggests that this value is related to the remaining-time prognosis shown by the Senso4s system.

An experimental interpretation currently used by the YAML is:

```text
1 counter step ≈ 15 minutes
```

which gives:

```text
hours = counter / 4
```

The YAML therefore exposes:

```text
Senso4s Estimated Remaining Time
```

### Important

The B2/B3 interpretation is **experimental**.

It has been derived from observed data and should not be considered an official or confirmed Senso4s protocol specification.

More testing is required to determine exactly what B2 and B3 represent.

## FF FF prognosis value

An observed value of:

```text
B2 = FF
B3 = FF
```

is treated as an unavailable prognosis value.

The estimated remaining time is then reported as unavailable.

Example:

```text
81 56 FF FF 48 00 E3 53 6C 63 03 AA
```

This packet represents:

```text
Model:       BASIC
Gas:         86 %
Battery:     72 %
Prognosis:   unavailable
MAC:         E3:53:6C:63:03:AA
```

## Example packet

A complete observed packet:

```text
81 56 FF FF 48 00 E3 53 6C 63 03 AA
```

can be interpreted as:

```text
B0 = 81
B1 = 56
B2 = FF
B3 = FF
B4 = 48
B5 = 00
B6 = E3
B7 = 53
B8 = 6C
B9 = 63
B10 = 03
B11 = AA
```

Decoded:

```text
Model:     BASIC
Usage:     BBQ
Gas:       86 %
Battery:   72 %
Prognosis: unavailable
MAC:       E3:53:6C:63:03:AA
```

## ESPHome entities

The configuration creates the following entities:

### Sensors

```text
Senso4s Gas Level
Senso4s Battery
Senso4s Prognosis Counter
Senso4s Estimated Remaining Time
```

### Text sensors

```text
Senso4s Model
Senso4s Usage
```

## Logging

The configuration logs the complete received packet when a known value changes.

Example:

```text
RAW: 81 56 FF FF 48 00 E3 53 6C 63 03 AA
```

It also prints the individual bytes and decoded values.

A periodic log is generated every 60 seconds containing the latest received packet.

This makes the configuration useful not only as a reader, but also as a tool for further Senso4s BLE reverse engineering.

## Multiple Senso4s devices

The configuration listens only for manufacturer ID:

```text
09CC
```

It does not specify a MAC address.

This means an ESP32 running this configuration can receive advertisements from multiple Senso4s devices.

For installations with multiple scales, the MAC address contained in B6-B11 can be used to distinguish the individual devices.

## What is currently known

### Verified

* Manufacturer ID `09CC`
* 12-byte advertisement payload
* B0 contains model and usage information
* B1 contains gas percentage
* B4 contains battery percentage
* B6-B11 contain the device MAC address
* PLUS devices have been observed
* BASIC devices have been observed
* `FF FF` can occur in B2/B3 when no prognosis is available

### Experimental

* Exact meaning of B2/B3
* Relationship between B2/B3 and remaining time
* The assumption that one counter step represents approximately 15 minutes
* Some of the B0 usage-mode mappings

### Unknown

* B5
* The complete meaning of all B0 flag combinations
* The complete B2/B3 algorithm
* Whether additional packet formats exist
* Whether other Senso4s models use different packet layouts

## Disclaimer

This project is an independent reverse-engineering and monitoring project.

It is based on observed BLE advertising data and is not an official Senso4s integration or protocol specification.

No GATT writes are performed by this configuration.

Use the official Senso4s application for calibration, configuration and device setup.

## License

Choose a license appropriate for your project.

For example, if you want others to freely use, modify and redistribute the configuration, you can use the MIT License.

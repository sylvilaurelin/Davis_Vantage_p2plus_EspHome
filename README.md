# ESPHome Davis Vantage Pro2 / Vue Receiver using ESP32 + CC1101

Receive a **Davis Vantage Pro2 / Vantage Vue EU 868 MHz** outdoor transmitter directly with an **ESP32 + CC1101**, decode the weather packets in ESPHome, and publish the measurements to Home Assistant.

No Davis console is required.

This configuration follows the Davis 868 MHz frequency-hopping sequence in real time. After locking onto a valid Davis packet, the ESP32 retunes the CC1101 to the frequency expected for the next transmission.

> **Important:** This configuration is for the **European 868 MHz Davis system**. It is not a drop-in configuration for US 915 MHz or other regional versions.

---

## Features

The current YAML provides:

- Davis EU 868 MHz five-channel frequency hopping
- Automatic synchronization after startup
- Missed-packet recovery
- Davis CRC-16 validation
- Bit-alignment recovery
- Per-hop reception statistics
- Home Assistant entities for:
  - Outdoor temperature
  - Outdoor humidity
  - Wind speed
  - Wind gust
  - Wind direction
  - Solar radiation
  - UV index
  - Rain accumulation
  - Transmitter battery-low status
  - RF RSSI
  - RF LQI
- ESPHome native API
- OTA updates
- Optional ESPHome web server
- Optional Bluetooth proxy

Packet type 5 is currently received and CRC-validated but is **not yet decoded into a rain-rate entity**.

---

## Tested configuration

Development testing was performed with:

- ESP32
- CC1101 868 MHz module
- Davis outdoor transmitter using Station ID `0`
- ESPHome 2026.8.2
- ESP-IDF framework

With the final RF settings, the receiver reached **98.6% cumulative valid packet reception** during one development test, including several consecutive one-minute windows at **100% reception on all five hopping frequencies**.

Actual performance will depend on antenna, CC1101 module quality, wiring, interference, distance, and installation.

---

# Hardware

## Required

- ESP32 development board
- CC1101 RF module suitable for the 868 MHz band
- 868 MHz antenna
- 3.3 V power for the CC1101
- Jumper wires or a PCB

A CC1101 module designed for 868/915 MHz operation is recommended.

> **Do not power the CC1101 from 5 V.**  
> Use **3.3 V** power and 3.3 V logic.

---

# Wiring

The included YAML uses the following ESP32 pins:

| CC1101 pin | ESP32 pin | Function |
|---|---:|---|
| VCC | 3.3 V | Power |
| GND | GND | Ground |
| SCK / CLK | GPIO18 | SPI clock |
| MISO / SO | GPIO19 | SPI MISO |
| MOSI / SI | GPIO23 | SPI MOSI |
| CSN / SS | GPIO5 | Chip select |
| GDO0 | GPIO4 | Packet interrupt / packet-ready signal |
| GDO2 | Not connected | Not used |

### Wiring diagram

```text
ESP32                         CC1101
-----                         ------

3.3V  ----------------------> VCC
GND   ----------------------> GND

GPIO18 ----------------------> SCK
GPIO19 <---------------------- MISO / SO
GPIO23 ----------------------> MOSI / SI
GPIO5  ----------------------> CSN / SS
GPIO4  <---------------------- GDO0

                               ANT ---- 868 MHz antenna
```

Keep the SPI and GDO0 wires reasonably short.

If the CC1101 supply is noisy or the module is unstable, adding local decoupling close to the module can help, for example:

```text
3.3 V ----+---- CC1101 VCC
          |
        100 nF
          |
         GND

and optionally approximately 10 µF between 3.3 V and GND.
```

---

# RF configuration

The working CC1101 settings are:

```yaml
frequency: 868.317250MHz
modulation_type: GFSK
symbol_rate: 19200
fsk_deviation: 9.5kHz
filter_bandwidth: 102kHz
magn_target: 33dB

packet_mode: true
packet_length: 10

sync_mode: 16/16
sync1: 0xCB
sync0: 0x89

crc_enable: false
whitening: false
num_preamble: 2
```

The CC1101 hardware CRC is disabled because the Davis packet uses its own CRC format, which is checked in the ESPHome lambda.

---

# Davis EU frequency hopping

The receiver follows this five-frequency sequence:

| Hop | Frequency |
|---:|---:|
| 0 | 868.077250 MHz |
| 1 | 868.317250 MHz |
| 2 | 868.557250 MHz |
| 3 | 868.197250 MHz |
| 4 | 868.437250 MHz |

Sequence:

```text
868.077250
    ↓
868.317250
    ↓
868.557250
    ↓
868.197250
    ↓
868.437250
    ↓
868.077250
```

The included YAML initially listens on:

```text
868.317250 MHz
```

Once a CRC-valid packet from the configured Davis transmitter is received, the receiver knows its current hop position and starts following the sequence.

---

# Packet timing

The supplied YAML is configured for:

```text
Davis packet Station ID = 0
```

For this transmitter ID, the packet period is:

```text
2.5625 seconds
```

The ESPHome state machine uses:

```text
2563 ms
```

and re-phases itself whenever a CRC-valid Davis packet is received, preventing timer error from accumulating.

If an expected packet is not received, the watchdog advances to the next hop while keeping the original transmitter timebase.

After 25 consecutive missed transmissions, the receiver abandons its timing lock and returns to the acquisition frequency.

---

# Station ID

The current decoder intentionally accepts only:

```cpp
unit_id == 0
```

This prevents another nearby Davis transmitter from taking over the hopping synchronization.

If your ISS uses another transmitter ID, you must change both:

1. The accepted `unit_id`
2. The packet timing

Nominal packet periods are:

| Packet Station ID | Period |
|---:|---:|
| 0 | 2.5625 s |
| 1 | 2.6250 s |
| 2 | 2.6875 s |
| 3 | 2.7500 s |
| 4 | 2.8125 s |
| 5 | 2.8750 s |
| 6 | 2.9375 s |
| 7 | 3.0000 s |

The current YAML has the `2563 ms` value in both the successful-packet re-phase logic and missed-packet watchdog. Both values must be changed for another transmitter ID.

---

# Decoded packet types

The current decoder handles the following Davis packet types:

| Type | Measurement |
|---:|---|
| 4 | UV index |
| 5 | Rain-tip timing / rain-rate related packet — received but not yet published |
| 6 | Solar radiation |
| 8 | Outdoor temperature |
| 9 | Wind gust |
| 10 | Outdoor humidity |
| 14 | Rain bucket tip counter |

Wind speed and wind direction are carried in the common portion of valid packets and are therefore updated much more frequently than when listening on only one fixed RF channel.

---

# Home Assistant entities

The YAML creates these main entities:

| Entity name | Description |
|---|---|
| Davis Temperature | Outdoor temperature |
| Davis Humidity | Outdoor humidity |
| Davis Wind Speed | Wind speed |
| Davis Wind Gust | Wind gust |
| Davis Wind Direction | Degrees + compass direction |
| Davis Solar Radiation | Solar radiation |
| Davis UV Index | UV index |
| Davis Daily Rain | Rain accumulator |
| Davis Battery Low | ISS battery-low flag |
| Davis RSSI | RF signal strength |
| Davis LQI | CC1101 link-quality indicator |

---

# Important note about rain

The current entity is named:

```text
Davis Daily Rain
```

but the ESPHome implementation itself is currently a **running accumulator since ESP boot**.

The global is initialized as:

```yaml
- id: rain_total_mm
  type: float
  initial_value: '0.0'
```

It is not persisted across reboot and there is no midnight reset in the YAML.

For a true daily rainfall value, it is recommended to either:

- create the daily total in Home Assistant from a monotonic rain counter, or
- add persistence and a daily reset strategy to the ESPHome configuration.

The decoder currently assumes:

```text
1 bucket tip = 0.2 mm
```

and uses the rolling 7-bit Davis rain-tip counter.

---

# Reception statistics

Every 60 seconds the YAML logs per-hop diagnostics.

Example:

```text
DAVIS_STATS:
H0 868.077271 | 60s ok 5/5 100% crc 0 miss 0
H1 868.317261 | 60s ok 5/5 100% crc 0 miss 0
H2 868.557251 | 60s ok 5/5 100% crc 0 miss 0
H3 868.197266 | 60s ok 5/5 100% crc 0 miss 0
H4 868.437256 | 60s ok 5/5 100% crc 0 miss 0
ALL | 60s ok 25/25 100.0% crc 0 miss 0
```

Meanings:

- `ok` — CRC-valid Davis packet
- `crc` — strong, time-plausible packet that failed Davis CRC
- `miss` — no usable packet was received in the expected slot

The statistics also include cumulative counts since boot.

These diagnostics are especially useful for checking:

- antenna placement
- individual hopping frequencies
- interference
- marginal CC1101 modules
- receiver placement

---

# Installing

## 1. Copy the YAML

Use the supplied ESPHome YAML as the device configuration.

Before publishing or installing it on another device, change at least:

```yaml
esphome:
  name: your-davis-receiver
  friendly_name: Davis Receiver
```

The original YAML contains installation-specific names.

---

## 2. Create `secrets.yaml`

At minimum, the configuration expects secrets for Wi-Fi, API encryption and OTA.

Example:

```yaml
wifi_ssid: "YOUR_WIFI_NAME"
wifi_password: "YOUR_WIFI_PASSWORD"

esphome_web_4763f4__encryption_key: "YOUR_ESPHOME_API_KEY"
OTA_password: "YOUR_OTA_PASSWORD"
```

You may also rename the API encryption secret in the main YAML to something more generic before publishing.

---

## 3. Compile and upload

Compile the configuration with ESPHome and upload it to the ESP32.

After the first USB installation, OTA updates can be used.

---

## 4. Watch the logs

On startup, the receiver waits on the acquisition frequency until it receives a CRC-valid Station ID 0 packet.

A successful lock looks similar to:

```text
DAVIS_HOP: LOCKED on hop 1; following Davis EU sequence
```

After locking, valid packets should normally appear approximately every:

```text
2.5625 seconds
```

The per-hop statistics begin reporting every 60 seconds.

---

# Optional ESPHome features

The supplied YAML also contains:

```yaml
bluetooth_proxy:
  active: true

web_server:
  port: 80
```

Neither is required for Davis reception.

They can be removed if you want a smaller, dedicated weather receiver configuration.

---

# Troubleshooting

## No packets at all

Check:

- CC1101 is an 868/915 MHz version
- CC1101 is powered from 3.3 V
- SPI wiring is correct
- GDO0 is connected to GPIO4
- antenna is suitable for 868 MHz
- Davis system is the EU 868 MHz model
- Davis transmitter Station ID is 0

---

## Receives only every ~12.8 seconds

That usually means the receiver is effectively hearing only one of the five Davis hopping frequencies.

A correctly synchronized Station ID 0 receiver should normally see a transmission about every 2.5625 seconds.

---

## Good RSSI but many bad packets

Verify the RF parameters, especially:

```yaml
fsk_deviation: 9.5kHz
filter_bandwidth: 102kHz
magn_target: 33dB
packet_length: 10
sync_mode: 16/16
```

Also check the antenna and power supply.

---

## Frequent loss of hopping synchronization

Use the `DAVIS_STATS` output to determine whether the problem is:

- one specific hop frequency
- CRC failures
- complete misses
- RF signal strength
- interference

Do not tune the frequencies based only on RSSI. CRC-valid reception percentage is the more useful metric.

---

# Notes about the 10-byte Davis packet

The CC1101 is configured to receive a fixed 10-byte packet.

The decoder uses the first eight bytes for the primary weather payload and CRC. Bytes 8 and 9 are retained so bit-alignment recovery has access to the complete received data and so direct-ISS/repeater framing can be diagnosed.

For the direct ISS used during development, valid packets normally had:

```text
FF FF
```

in the final two decoded bytes.

The YAML logs a warning if a CRC-valid packet has a different tail.

---

# Antenna notes

A simple 868 MHz quarter-wave wire is approximately:

```text
86 mm
```

A half-wave dipole is approximately:

```text
2 × 86 mm
```

after allowing for real-world construction and tuning.

For best results:

- keep the antenna away from large metal surfaces
- avoid placing the ESP32/CC1101 inside a closed steel enclosure
- orient the antenna appropriately for the Davis transmitter
- use short coax if the antenna must be moved away from electronics
- do not place the antenna directly next to the ESP32 Wi-Fi antenna

The exact best orientation depends on the installation.

---

# Current limitations

- EU 868 MHz frequency set only
- Station ID 0 only unless timing/code is changed
- Rain total resets when the ESP reboots
- No automatic midnight rain reset
- Packet type 5 rain-rate data is not yet exposed as an entity
- No support yet for multiple Davis transmitters
- Repeater behavior has not been extensively tested
- This is a reverse-engineered receiver and is not an official Davis Instruments implementation

---


---

# References / prior art

This project builds on publicly documented and reverse-engineered Davis/CC1101 work, including:

- ESPHome CC1101 component documentation  
  https://esphome.io/components/cc1101/

- CC1101 Weather Receiver by cmatteri  
  https://github.com/cmatteri/CC1101-Weather-Receiver

- DavisRFM69 by dekay  
  https://github.com/dekay/DavisRFM69

These projects were useful references for Davis packet framing, radio parameters, CRC handling and frequency-hopping behavior.

---

# Disclaimer

This project is not affiliated with or endorsed by Davis Instruments or ESPHome.

Use it at your own risk, especially if the weather data is used for safety-critical decisions.

# Pool Pump RPM Regulator

ESPHome firmware for an ESP32 that talks RS485 to a **Kripsol** pool pump's
built-in frequency converter (VFD), using the **iSAVER** protocol, and exposes
full RPM control and live status to Home Assistant. One self-contained YAML
file, no external ESPHome components, no cloud dependency — control keeps
working over the local network even if Home Assistant itself is down.

## Features

- **Live status polling** — RPM, on/off state, and error code read from the
  pump once per second over RS485
- **RPM control** — set any speed from 1200–2900 via a number entity, three
  preset buttons (ECO / NORMAL / MAX), or a plain on/off power switch
- **Runs independently of Home Assistant** — pump control keeps working
  during a Home Assistant outage or a device reboot; nothing about the pump's
  operation depends on an active API connection
- **Setpoint survives reboot** — the last commanded RPM is restored after a
  power cycle, without ever being able to restore into a stuck "off" state
- **Full error decoding** — all 16 iSAVER error bits are decoded to a
  human-readable text sensor, not just a raw error code
- **Filter duration timer** — run at the current speed for a set number of
  hours, then automatically drop back to ECO
- **UART Debug Logging switch** — see raw RS485 traffic in the log when
  troubleshooting, toggled from Home Assistant with no reflash needed
- **Automation-friendly** — two native API services (`set_pump_rpm`,
  `set_filter_duration`) for scripting from Home Assistant automations, in
  addition to the regular entities

## Hardware

### Bill of materials

| Part | Example used | Notes |
| --- | --- | --- |
| ESP32 dev board | ELEGOO `EL-SM-009` | Any `esp32dev`-compatible board works |
| TTL-to-RS485 adapter | Jopto (ASIN `B096ZXXKCR`) | Any TTL↔RS485 module works |
| Cabling | — | RS485 twisted pair between adapter and pump, jumpers for the rest |
| Power | — | USB power supply + cable for the ESP32 |

### Wiring

| ESP32 pin | Function | Goes to |
| --- | --- | --- |
| `GPIO17` | UART TX | RS485 adapter RX |
| `GPIO16` | UART RX | RS485 adapter TX |
| `GPIO05` | RS485 DE/RE (write-enable) | RS485 adapter DE/RE, if present |
| `GND` | Ground | Shared with pump and RS485 adapter |
| `3.3V` / `5V` | Power | RS485 adapter (depends on adapter's voltage) |

`GPIO05` is a strapping pin — ESPHome warns about it on every build, expected
on this board. Note: an earlier build of this project used an adapter with no
DE/RE line at all (auto-direction-sensing); the current firmware assumes one
is present and toggles it around every write. Its state is briefly undefined
between power-on and ESPHome's own setup — harmless in practice since the
polling loop re-asserts the correct state within a second of boot.

## Protocol

Sources: [htilly/ha-esp32-variable-speed-drive-esphome](https://github.com/htilly/ha-esp32-variable-speed-drive-esphome),
[backuprestore/isaver-isaverx-RS485-modbus](https://github.com/backuprestore/isaver-isaverx-RS485-modbus),
and an iSAVER RS485 protocol PDF. Treat the PDF as a starting point, not
ground truth — its own spec claims a 9-byte response with a trailing CRC,
which is wrong for this pump (see below).

Called "Modbus" in the source material, but it's a custom protocol, not
standard Modbus RTU.

**Read status — function `0xC3`:**
```
Request  (8 bytes): AA C3 07 D1 00 00 0D 4D
Response (7 bytes): AA C3 [err_hi] [err_lo] [on_off] [rpm_hi] [rpm_lo]
```

**Write RPM — function `0xD0`, register `0x0BB9`:**
```
Request  (8 bytes): AA D0 0B B9 [rpm_hi] [rpm_lo] [crc_lo] [crc_hi]
ACK:                AA D0 0B B9 00 02 00
```

Verified on the bus, all of it counter-intuitive:

- **Responses carry no CRC.** Requests do — CRC-16/Modbus, poly `0xA001`,
  sent **little-endian** (lo byte first). Validating a CRC on responses
  silently rejects every frame.
- **Stray bytes surround every response** (`F8`, `FF`, `FE`, `F0`, `E0`, …)
  from RS485 bus turnaround — search for the `AA C3` header, never assume a
  fixed offset.
- **RPM encoding:** `1` = OFF, `1200`–`2900` = run. The OFF check must come
  before any clamping, or an off command gets clamped up to minimum speed.
- **Priming:** the pump runs at max (2900 RPM) for a few minutes after every
  start, then settles at the setpoint — a high reading right after a start
  is normal, not a bug.

## Requirements

- [ESPHome](https://esphome.io/) 2026.8 or newer (`pip install esphome`, or
  the Home Assistant ESPHome add-on / dashboard)
- A Kripsol pool pump with an iSAVER-protocol frequency converter, wired as above
- Home Assistant is optional — the device works standalone via its own web UI,
  but exposes all entities over the native API when HA is present

## Installation

1. Clone this repo and copy `secrets.yaml.example` to `secrets.yaml`, filling
   in your own values (see **Configuration** below).
2. Validate the config without touching any hardware:
   ```
   esphome config pump-rpm-regulator.yaml
   ```
3. First flash requires a USB connection:
   ```
   esphome run pump-rpm-regulator.yaml
   ```
4. Every later update goes over OTA using the same command — no cable needed
   as long as the device is already on the network.

After any change, run through the **Verification checklist** below against
the real device before trusting it — this firmware has no automated test
suite; the pump itself is the only thing that proves a change works.

## Configuration

All secrets live in `secrets.yaml` (gitignored — never commit it). Copy
`secrets.yaml.example` and fill in:

| Key | Used for |
| --- | --- |
| `wifi_ssid` / `wifi_password` | Home WiFi credentials |
| `api_key` | Native API encryption key (generate per ESPHome docs) |
| `ota_password` | Password for OTA firmware updates |
| `static_ip` / `gateway` / `subnet` | Fixed IP configuration for the device |
| `ap_fallback_password` | Password for the fallback AP if WiFi is unreachable |

`api_key_presence` also exists in `secrets.yaml.example` but isn't currently
referenced anywhere in the YAML — left over from an earlier iteration.

## Entities exposed

Entity IDs are prefixed with the device name (`pump_rpm_regulator_`) by Home
Assistant's ESPHome integration — e.g. the RPM setpoint is
`number.pump_rpm_regulator_pool_pump_rpm_setpoint`, not just
`number.pool_pump_rpm_setpoint`. Tables below show the name after that prefix.

### Sensors

| Entity (after `sensor.pump_rpm_regulator_`) | Description |
| --- | --- |
| `pool_pump_rpm` | Actual RPM read from the pump's status frame |
| `pool_pump_filter_time_remaining` | Minutes left on the filter-duration timer |
| `wifi_signalstyrka` | WiFi signal strength (dBm) |
| `esp_uptime` | Seconds since last boot — the fastest way to spot a silent restart |
| `esp_temperatur` | ESP32 internal temperature |

### Text sensors

| Entity (after `text_sensor.pump_rpm_regulator_`) | Description |
| --- | --- |
| `modbus_status` | `Online`, `Waiting for packet`, or `Offline` (after 5 consecutive failed polls) |
| `pool_pump_error` | Decoded iSAVER error text, or `No Error` |
| `esp_ip_adress` | Current IP address |
| `esphome_version` | Firmware version and build info |
| `esp_reset_reason` | Why the device last restarted — OTA, brownout, watchdog, or software crash |

### Binary sensor

| Entity | Description |
| --- | --- |
| `binary_sensor.pump_rpm_regulator_esp_status` | Whether an API client (e.g. Home Assistant) is currently connected |

### Number

| Entity (after `number.pump_rpm_regulator_`) | Range | Description |
| --- | --- | --- |
| `pool_pump_rpm_setpoint` | 1200–2900, step 50 | Target RPM. Survives reboot |
| `pool_pump_filter_duration` | 0–24 h, step 0.5 | Runs at current speed for this long, then drops to ECO (1300 RPM) |

### Switch

| Entity (after `switch.pump_rpm_regulator_`) | Description |
| --- | --- |
| `pool_pump_power` | On = 1800 RPM, Off = pump stopped. Always boots to whatever the pump itself reports, never commands it on boot |
| `uart_debug_logging` | Toggles raw RS485 hex logging on/off at runtime, no reflash needed. Off by default on every boot |

### Button

| Entity (after `button.pump_rpm_regulator_`) | Sets |
| --- | --- |
| `pool_pump_eco` | 1300 RPM |
| `pool_pump_normal` | 1800 RPM |
| `pool_pump_max` | 2900 RPM |

### Automation services

Exposed via the ESPHome native API (`esphome.pump_rpm_regulator_<service>` in
Home Assistant):

| Service | Parameter | Description |
| --- | --- | --- |
| `set_pump_rpm` | `rpm` (int) | Same effect as setting the **Pool Pump RPM Setpoint** number |
| `set_filter_duration` | `hours` (float) | Same effect as setting the **Pool Pump Filter Duration** number |

These duplicate what the number entities already do via HA's standard
`number.set_value` service — kept for now in case an existing automation
calls them directly.

## How it keeps running without Home Assistant

Three deliberate settings, each guarding against a real failure this project
hit once:

- `api: reboot_timeout: 0s` — the device never reboots itself for lack of an
  API client, so a Home Assistant outage can't interrupt pump control
- The **Pool Pump Power** switch boots with `restore_mode: DISABLED` — it never
  commands the pump on startup; the real on/off state arrives from the pump's
  own status frame a second later
- The RPM setpoint only restores on boot if the stored value is a real
  running speed (`>= 1200`) — a stuck "off" value can never be replayed

## Troubleshooting

| Problem | Solution |
| --- | --- |
| `Modbus Status` shows `Offline` | Check RS485 wiring (TX/RX not swapped, DE/RE on `GPIO05` connected) and that the pump is powered |
| `Modbus Status` stuck on `Waiting for packet` | Fewer than 5 consecutive polls have failed yet — wait a few seconds, or check wiring if it doesn't recover |
| Setpoint slider shows a value the pump isn't actually running at | Shouldn't happen since the `min_value: 1200` fix — if seen, the slider was set from an automation calling `set_value` with something below 1200 |
| Pump won't turn off, or turns on at minimum speed unexpectedly | Check the **Pool Pump Power** switch's `restore_mode` is still `DISABLED` — see "How it keeps running" above |
| Need to see raw RS485 traffic | Turn on **UART Debug Logging**, watch the log, turn it back off when done |
| Device reboots every ~15 minutes | `api: reboot_timeout` got reset to its default — must stay `0s` |
| Web UI reachable but device shows as unavailable in HA | The web UI (port 80) does **not** count as an API client — check the native API port (6053) isn't blocked |

## Verification checklist

Run through this against the real device after any change — there's no
automated test suite, so this is the only thing that proves a change works:

1. Flash, then confirm the build timestamp in the web UI is today's
2. `Modbus Status` → `Online`, `Pool Pump Error` → `No Error`
3. Press ECO (1300 RPM): pump audibly changes speed, RPM sensor updates
4. Power off: verify the pump actually stops, not just drops to minimum speed
5. Reboot with the pump running — it must keep running
6. Disable the Home Assistant integration, leave the pump running, and check
   that uptime passes 900s without a restart (don't open the log stream
   during this test — it counts as an API client and resets the timer)
7. Compare the RPM sensor against the pump's own display

## Notable fixed issues

- **Offline for two months while the pump was answering fine** — the parser
  validated a CRC on the pump's status responses, but this pump's responses
  don't carry one; every frame was silently rejected
- **Pump switched itself off after ~15 minutes** — `api: reboot_timeout`
  defaulted to 15 minutes, so any Home Assistant outage rebooted the device,
  which then booted into an "off" state that got written back to flash and
  never recovered — see "How it keeps running" above for the three guards
  that now prevent this
- **Setpoint slider could show a value the pump never got** — picking
  anything from 2–1199 RPM got silently rounded up to 1200 before being sent,
  but the displayed setpoint kept the un-rounded number; fixed by raising the
  slider's minimum to 1200

## Status

Verified working against real hardware as of 2026-09-13: `Modbus Status`
Online, no error, correct RPM.

## Development

Single self-contained YAML file — no build tooling beyond ESPHome itself.
`esphome compile pump-rpm-regulator.yaml` builds without flashing; on Windows
this needs [long path support enabled](https://learn.microsoft.com/windows/win32/fileio/maximum-file-path-limitation)
or `ESPHOME_ESP_IDF_PREFIX` set to a short path, or it fails on ESP-IDF's own
toolchain paths. A GitHub Actions workflow
([.github/workflows/validate.yaml](.github/workflows/validate.yaml)) runs the
same config + compile check on Linux for every push, where that path issue
doesn't exist.

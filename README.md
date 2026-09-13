# Pool Pump RPM Regulator

ESPHome firmware for an ESP32 that controls a Kripsol pool pump's frequency
converter (VFD) over RS485, using a custom Modbus-RTU-style protocol, and exposes
RPM control and live status to Home Assistant. One self-contained YAML file, no
external ESPHome components.

The pump keeps running whether or not Home Assistant is up — see
[CLAUDE.md](CLAUDE.md) for the full protocol writeup, hardware wiring, and the
bugs already found and fixed.

## What it does

- Reads the pump's live status (on/off, RPM, error code) once per second over
  RS485 and publishes it to Home Assistant.
- Lets you set a target RPM (1200–2900) or turn the pump off from Home
  Assistant, a number/switch entity, or the device's own web UI.
- Survives a Home Assistant outage and a device reboot without stopping the
  pump or losing the last setpoint.

## Hardware

| Item | Value |
| --- | --- |
| Board | ESP32 Dev Module (`esp32dev`, framework: arduino) |
| UART | TX `GPIO17`, RX `GPIO16`, 1200 baud, 8N1 |
| RS485 DE/RE | `GPIO05` |
| Pump | Kripsol pool pump — RS485 talks to its frequency converter (VFD) |

## Building and flashing

1. Install [ESPHome](https://esphome.io/) (`pip install esphome`, or use the
   ESPHome dashboard/add-on).
2. Copy `secrets.yaml.example` to `secrets.yaml` and fill in your own WiFi
   credentials and API/OTA keys (`secrets.yaml` is gitignored — never commit
   it).
3. Validate the config without touching any hardware:
   ```
   esphome config pump-rpm-regulator.yaml
   ```
4. Build and flash:
   ```
   esphome run pump-rpm-regulator.yaml
   ```
   First flash needs a USB connection; later updates go over OTA.

After any change, run through the [Verification
checklist](CLAUDE.md#verification-checklist-after-any-change) in CLAUDE.md
against the real device before trusting it.

## Status

Verified working against real hardware as of 2026-09-13 — see CLAUDE.md's
"Current status" section for details.

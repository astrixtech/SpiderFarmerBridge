# Spider Farmer GGS — ESPHome BLE bridge

Connect a [Spider Farmer](https://www.spider-farmer.com/) **GGS** grow controller (`SF-GGS-CB` and compatible builds) to **Home Assistant** over **BLE** using a single [ESPHome](https://esphome.io/)-supported **ESP32-family** module (Wi‑Fi + BLE). The configuration **reads** state from the GGS and exposes it through the **native ESPHome API** (temperature, humidity, VPD, and equipment status such as fan, light, and blower) so you can see what the controller is doing—see **Design intent** below. No cloud account is required for this path.

## Design intent: monitoring, not actuation

The goal of this bridge is **observability in Home Assistant** (dashboards, history, and alerts from live data), **not** to **change** grow parameters or equipment from Home Assistant. Avoid using HA to switch equipment on or off, change levels, or otherwise “drive” the tent from automations; perform those changes through **Spider Farmer** (e.g. **Planting Plans**, the official app, or the controller UI as intended by the product).

**Why:** Planting Plans and the controller’s own schedules rely on a single, consistent path for setpoints, timing, and actuation. **Controlling the same equipment from a second system** (including toggling blower, light, or fan from Home Assistant or the mobile app while a plan is active) can **clash with scheduled stages**, **override planned environment steps**, and **break the continuity** of the grow program you configured in Spider Farmer. The maintainable pattern is: **set programs and equipment policy in Spider Farmer**; use Home Assistant here to **view** state and trends so monitoring does not fight the plan.

## What this repository contains

| Path | Description |
|------|-------------|
| `spider_farmer_ggs_ble.yaml` | ESPHome configuration for the bridge |
| `secrets.example.yaml` | Template for Wi‑Fi credentials; copy to `secrets.yaml` |
| `tools/ggs_ble.py` | Optional: command-line helper to scan/connect and read notifications (Python 3 + [Bleak](https://github.com/hbldh/bleak)) |

## Requirements

- **Hardware:** An Espressif module with **both Wi‑Fi and Bluetooth LE** (this bridge does not use GPIO, so any supported SoC works if the build fits in flash). Set **`esphome_variant`** and **`flash_size`** in the YAML to match your chip and module (see [ESPHome — ESP32 `variant`](https://esphome.io/components/esp32.html#variants)).
- **Not suitable:** **`esp32s2`** (no Bluetooth). Double-check **H2 / P4** class parts: use a variant that matches what ESPHome supports for Wi‑Fi + BLE in your use case.
- **Controller:** Spider Farmer GGS (BLE).
- **Home system:** [Home Assistant](https://www.home-assistant.io/) with the [ESPHome](https://www.home-assistant.io/integrations/esphome/) integration.
- **Phone app (for setup):** [nRF Connect](https://www.nordicsemi.com/Products/Development-tools/nrf-connect-for-mobile) (iOS or Android) to read the controller’s BLE address and GATT UUIDs if yours differ from the defaults in the YAML.

## Quick start

1. **Clone** this repository (or copy the files into your ESPHome config directory).

2. **Secrets:** copy `secrets.example.yaml` to `secrets.yaml` and set your Wi‑Fi SSID and password.

3. **Edit** `spider_farmer_ggs_ble.yaml` **substitutions** at the top:
   - `esphome_variant` — SoC id for [ESPHome’s `esp32` `variant`](https://esphome.io/components/esp32.html#variants) (e.g. `esp32` for original ESP32, `esp32c3` for ESP32‑C3, `esp32s3` for ESP32‑S3). Must match your hardware.
   - `flash_size` — size of the **flash** chip on the module (e.g. `4MB`, `8MB`). This is not “RAM size”; it must match the module or flashing/OTA can fail.
   - `ggs_ble_mac` — BLE address of the GGS from nRF Connect (replace the placeholder `aa:bb:cc:dd:ee:ff`).
   - `ggs_service_uuid`, `ggs_notify_char_uuid`, `ggs_write_char_uuid` — only if your GATT profile differs from the defaults.
   - **Optional tuning** (left at values that are stable in the field; change only if you need to): `api_*`, `ble_connection_timeout`, `ggs_scan_retry_interval`, `ggs_status_interval` — see the comments in the YAML. To use ESPHome stock API numbers, set those keys to the defaults documented in the [API component](https://esphome.io/components/api.html) and the [`esp32_ble` component](https://esphome.io/components/esp32_ble.html) pages.

   | Goal | Set in substitutions |
   |------|------------------------|
   | Original ESP32 dev board | `esphome_variant: esp32`, `flash_size: 4MB` (or match your module) |
   | ESP32‑C3 (e.g. supermini) | `esphome_variant: esp32c3`, `flash_size: 4MB` typical |
   | ESP32‑S3 | `esphome_variant: esp32s3`, `flash_size` per module (often 8MB) |

4. **Optional `board` line:** this bridge does not use GPIO. If you **add** components that use pins, set `board: <PlatformIO board id>` under the `esp32:` key so pin labels match your devkit (see [board list](https://registry.platformio.org/platforms/platformio/espressif32/boards) and [ESP32 component](https://esphome.io/components/esp32.html)). Example: `board: esp32-c3-devkitm-1` for a common C3 board.

5. **Build and flash** (from the directory that contains the YAML and `secrets.yaml`):
   - ESPHome Dashboard: add this package and use **Install**.
   - CLI: `esphome run spider_farmer_ggs_ble.yaml`

6. **Home Assistant:** **Settings → Devices & services → Add integration → ESPHome** and add the device (hostname, IP, or `*.local` as prompted).

7. **Entities:** the YAML uses short names (e.g. `Temperature`, `Blower`). Home Assistant prefixes them with the device name, e.g. `binary_sensor.spider_farmer_bridge_blower` (depending on your `device_name` / friendly name).

## BLE address and GATT profile (nRF Connect)

**Only one BLE central at a time:** while you use nRF Connect (or `ggs_ble.py`), nothing else should be connected to the GGS over Bluetooth — for example, close the **Spider Farmer** mobile app and any other phone/tablet that might hold a link to the controller. Otherwise connect/scan can fail or show incomplete services.

1. Open **nRF Connect**, scan, and find the GGS (often shown as `SF-GGS-CB` or similar).
2. Note the **MAC address** and set `ggs_ble_mac` in the YAML (lowercase, colon-separated, is common).
3. Connect, open the custom service, and note **notify** and **write** characteristics.

Many units use this GATT layout (if yours matches, you can leave the UUID substitutions as-is):

| Role | Example UUID |
|------|----------------|
| Service | `000000ff-0000-1000-8000-00805f9b34fb` |
| Notify | `0000ff01-0000-1000-8000-00805f9b34fb` |
| Write | `0000ff02-0000-1000-8000-00805f9b34fb` |

## Optional: `tools/ggs_ble.py`

Use on a PC with **Bluetooth enabled** (built-in radio or a USB Bluetooth adapter), **Python 3**, and `pip install bleak` to discover the address or inspect notifications outside Home Assistant. Run with `--help` for options.

## Disclaimer

This is an independent, non-commercial project. It is not affiliated with, endorsed by, or supported by Spider Farmer. Use at your own risk. Reverse engineering and use of product protocols may be restricted in your region or by the manufacturer’s terms; you are responsible for compliance with applicable law and licenses.

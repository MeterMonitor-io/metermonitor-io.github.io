# MeterMonitor

**AI-powered water meter reading for Home Assistant.**

MeterMonitor reads analog water meters from camera images — no on-device processing required. A camera (ESP32-CAM or any Home Assistant camera entity) sends images to the MeterMonitor server, which extracts the digits using AI and publishes readings back to Home Assistant via MQTT.

---

## What MeterMonitor does

- Reads rotating-wheel and 7-segment water meters via camera
- Publishes readings to Home Assistant as MQTT sensors (auto-discovery)
- Tracks consumption history with charts
- Corrects common misreads automatically (digit roll-over, rotation indicators)
- Supports multiple meters and multiple camera sources

## Where to start

| I want to… | Go to |
|---|---|
| Install MeterMonitor | [Installation](installation.md) |
| Set up my ESP32 camera | [ESP32 / Camera Setup](esp32-setup.md) |
| Configure my first meter | [User Guide](user-guide.md) |
| Adjust settings | [Configuration Reference](configuration.md) |
| Fix a problem | [Troubleshooting](troubleshooting.md) |

## Requirements

**Home Assistant Add-on** (easiest):
- Home Assistant OS or Supervised
- Mosquitto broker add-on
- `amd64` or `aarch64` (ARM64)

**Standalone (Docker or Python)**:
- Docker, or Python 3.12+
- Any MQTT broker

## Getting help

Open an issue at the [GitHub repository](https://github.com/MeterMonitor-io/metermonitor-ha/issues).

## Credits

Inspired by [AI-on-the-edge-device](https://github.com/jomjol/AI-on-the-edge-device).
Training data by [haverland/collectmeterdigits](https://github.com/haverland/collectmeterdigits).

# ESP32 / Camera Setup

MeterMonitor accepts images from any source — the **official MQTT-based software** for the ESP32, an **ESP32-CAM with ESPHome** using a **Home Assistant camera entity**, or any **HTTP endpoint**. This guide covers the ESP32 path.

---

## Hardware

> **Recommended solution: [MeterMonitor-esp](https://github.com/MeterMonitor-io/MeterMonitor-esp)**
> A ready-to-flash ESPHome configuration and 3D-printable mount designed specifically for MeterMonitor. This is the easiest way to get started — no custom ESPHome config required.

**Recommended board:** AI-Thinker ESP32-CAM (~€5–10)
- Built-in OV2640 camera (2MP)
- GPIO4 flash LED (optional but useful)
- Supports external antenna

Any ESP32 board with an OV2640 camera will work. Ensure at least 4MB flash and PSRAM enabled.

**Power:** Use a stable 5V supply (≥500mA). Voltage drops during image capture cause reboots — use a short USB cable or a regulated 5V rail.

---

## ESPHome firmware

MeterMonitor uses the **Home Assistant Camera** source to poll the ESP32's camera entity — no custom MQTT code is needed.

### Minimal ESPHome configuration

```yaml
esphome:
  name: watermeter
  friendly_name: Watermeter

esp32:
  board: esp32cam
  framework:
    type: arduino

psram:
  mode: quad
  speed: 80MHz

logger:
  baud_rate: 0          # disable serial (conflicts with camera pins)

api:

ota:
  platform: esphome

wifi:
  ssid: !secret wifi_ssid
  password: !secret wifi_password

# ── Camera ────────────────────────────────────────────────
esp32_camera:
  name: "Watermeter Camera"
  external_clock:
    pin: GPIO0
    frequency: 20MHz
  i2c_pins:
    sda: GPIO26
    scl: GPIO27
  data_pins: [GPIO5, GPIO18, GPIO19, GPIO21, GPIO36, GPIO39, GPIO34, GPIO35]
  vsync_pin: GPIO25
  href_pin: GPIO23
  pixel_clock_pin: GPIO22
  power_down_pin: GPIO32
  resolution: 640x480
  jpeg_quality: 8
  max_framerate: 1 fps
  idle_framerate: 0.1 fps

# ── Flash LED (optional) ──────────────────────────────────
output:
  - platform: gpio
    pin: GPIO4
    id: led_flash

light:
  - platform: binary
    name: "Watermeter LED"
    output: led_flash
    restore_mode: ALWAYS_OFF
```

Flash initial firmware via USB/FTDI, then use ESPHome OTA for all future updates.

### Image settings

| Setting | Recommendation |
|---|---|
| Resolution | `640x480` (VGA) — good balance of quality and speed |
| JPEG quality | `8`–`12` — lower number = better quality, larger file |
| Max framerate | `1 fps` — MeterMonitor only needs one image per poll |

---

## Sending images to MeterMonitor

### Option A — Direct MQTT (push - no capture on demand)

If you want the ESP32 to push images directly, publish to `MeterMonitor/<device-name>` with this payload:

```json
{
  "name": "watermeter",
  "WiFi-RSSI": -55,
  "picture": {
    "timestamp": "2025-06-01T10:00:00",
    "data": "<base64-encoded jpeg>"
  }
}
```

`name` and `picture.data` are required. Timestamp and RSSI are optional. 

**Recommended solution: [MeterMonitor-esp](https://github.com/MeterMonitor-io/MeterMonitor-esp)** - it implements a very power-efficient MQTT option and provides a simple way to integrate with MeterMonitor.

---

### Option B — Home Assistant Camera source (pull - capture on demand)

1. Add the ESP32 to Home Assistant via the ESPHome integration
2. In MeterMonitor, go to **Add watermeter → Source-Type "Home Assistant""**
3. Set the camera entity ID (e.g. `camera.watermeter_camera`)
4. Set a poll interval in minutes
5. If you have a flash LED, set the flash entity ID and a delay of `~1000ms` (maybe longer for better results or bad connection)

MeterMonitor will poll the camera, optionally trigger the flash, and process the image.

### Option C - HTTP Polling (pull - capture on demand)

MeterMonitor can poll an HTTP endpoint for images. This is useful if you have a custom camera setup or want to integrate with a different system.
1. Go to **Add watermeter → Source-Type "HTTP""**
2. Set the HTTP endpoint URL (e.g. `http://192.168.1.100/camera.jpg`)
3. Set a poll interval in minutes
4. Set your custom headers or body to send with the GET-request.

MeterMonitor will poll the camera and process the image.

## Camera positioning

- **Distance:** 15–25 cm from the meter face
- **Angle:** As perpendicular as possible — avoid more than ~30° of tilt
- **Entire display must be visible** including all digits
- **Stable mount** — movement between captures should be minimal

### Lens focus

Most ESP32-CAM lenses are pre-focused at ~30 cm. If your image is blurry at the meter distance, carefully rotate the lens:
- Counterclockwise → focus farther
- Clockwise → focus closer

Lock with a small drop of superglue once focused.

### Lighting

- Consistent lighting gives better results than a flash (which can cause glare)
- If ambient light is unreliable, configure the flash LED and set a delay of 1000–2000 ms to let the LED stabilise before capture
- Avoid direct sunlight on the meter face (shadows across digits)

---

## Troubleshooting

| Symptom | Fix |
|---|---|
| Blurry image | Adjust lens focus; check camera distance |
| Dark image | Enable flash or increase `brightness` in ESPHome camera settings |
| Glare / reflections | Reposition camera angle; use indirect lighting |
| ESP32 reboots during capture | Improve power supply; add a 100–470 µF capacitor near the power pins |
| WiFi not connecting | Use 2.4 GHz; check `secrets.yaml` credentials; consider static IP |
| Camera not initialised | Ensure PSRAM is enabled; disable serial logging (`baud_rate: 0`) |

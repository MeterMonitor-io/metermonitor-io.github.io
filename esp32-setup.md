# ESP32-CAM Hardware and Firmware Setup

Complete guide to setting up ESP32-CAM devices for MeterMonitor.

## Table of Contents

1. [Hardware Requirements](#hardware-requirements)
2. [Camera Selection](#camera-selection)
3. [ESPHome Installation](#esphome-installation)
4. [Configuration](#configuration)
5. [Camera Positioning](#camera-positioning)
6. [LED Flash Usage](#led-flash-usage)
7. [MQTT Configuration](#mqtt-configuration)
8. [Troubleshooting](#troubleshooting)

---

## Hardware Requirements

### Required Components

**ESP32-CAM Module**:
- AI-Thinker ESP32-CAM or compatible
- ESP32 chip with camera interface
- Minimum 4MB flash
- PSRAM recommended for better performance

**Camera Module**:
- OV2640 camera (included with most ESP32-CAM boards)
- 2MP resolution sufficient
- Fixed focus or adjustable focus lens

**Power Supply**:
- 5V power supply (minimum 500mA)
- USB to FTDI adapter for initial programming
- Stable power supply critical (voltage drops cause reboots)

**Optional but Recommended**:
- LED flash (GPIO4 on most boards)
- External antenna for better WiFi range
- 3D-printed mount or case
- Lens adjustment tool (for focus)

### Recommended Boards

**AI-Thinker ESP32-CAM**:
- Most common and well-supported
- Built-in antenna + external antenna connector
- GPIO4 LED flash output
- Price: ~$10 USD

**Alternatives**:
- ESP32-CAM-MB (with USB programmer)
- TTGO T-Camera
- M5Stack Camera modules

---

## Camera Selection

### OV2640 Camera Module

The standard OV2640 camera is suitable for most water meters:
- Resolution: Up to 1600x1200 (UXGA)
- For MeterMonitor: 640x480 (VGA) recommended
- Field of view: ~70 degrees
- Fixed or adjustable focus

### Lens Considerations

**Fixed Focus**:
- Pre-focused at ~30cm
- No adjustment needed
- Suitable if meter distance is optimal

**Adjustable Focus**:
- Rotate lens to focus
- Use small wrench or pliers (gentle!)
- Focus on meter display before mounting
- Lock with small drop of glue if needed

**Wide-Angle vs. Standard**:
- Standard lens (default): Best for 10-30cm distance
- Wide-angle: Useful if space is constrained
- Telephoto: For long-distance installations (rare)

---

## ESPHome Installation

### Prerequisites

- ESPHome installed (via Home Assistant add-on or standalone)
- FTDI USB adapter (if using bare ESP32-CAM)
- ESPHome dashboard access

### Initial Flashing

**Step 1: Connect ESP32-CAM to Computer**

Using FTDI adapter:
1. Connect FTDI to ESP32-CAM:
   - FTDI GND → ESP32 GND
   - FTDI 5V → ESP32 5V
   - FTDI TX → ESP32 U0RXD
   - FTDI RX → ESP32 U0TXD
2. Connect GPIO0 to GND (boot mode)
3. Connect FTDI to computer USB

**Step 2: Create ESPHome Configuration**

1. Open ESPHome dashboard
2. Click **+ NEW DEVICE**
3. Name device (e.g., "watermeter")
4. Select ESP32
5. Choose ESP32CAM board

**Step 3: Flash Firmware**

1. Click **INSTALL** on device
2. Select **Plug into this computer**
3. Choose correct serial port
4. Wait for compilation and upload
5. Disconnect GPIO0 from GND
6. Press RESET button
7. Device should boot and connect to WiFi

### OTA Updates

After initial flashing, updates can be done wirelessly:
1. Modify configuration in ESPHome
2. Click **INSTALL**
3. Select **Wirelessly**
4. Updates apply without physical connection

---

## Configuration

### Example ESPHome YAML

Based on `/home/phiph/Repositories/metermonitor-managementserver/esphome_esp32cam_example.yaml`:

```yaml
esphome:
  name: watermeter
  friendly_name: Watermeter

esp32:
  board: esp32cam
  framework:
    type: arduino

# Enable PSRAM for better camera performance
psram:
  mode: quad
  speed: 80MHz

# Disable serial logging (conflicts with camera)
logger:
  baud_rate: 0

# Enable Home Assistant API
api:

# Enable OTA updates
ota:
  platform: esphome

# WiFi configuration
wifi:
  ssid: !secret wifi_ssid
  password: !secret wifi_password

  # Optional: Static IP for reliability
  # manual_ip:
  #   static_ip: 192.168.1.100
  #   gateway: 192.168.1.1
  #   subnet: 255.255.255.0

# LED Flash configuration
output:
  - platform: gpio
    pin: GPIO4
    id: led_flash

light:
  - platform: binary
    name: "Watermeter LED"
    output: led_flash
    restore_mode: ALWAYS_OFF

# Camera configuration
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

  # Image settings
  resolution: 640x480      # VGA resolution (optimal)
  jpeg_quality: 8          # 1-63, lower = better quality
  max_framerate: 1 fps     # Low framerate for stability
  idle_framerate: 0.1 fps  # Very low when idle

  # Optional: Vertical flip
  # vertical_flip: true

  # Optional: Horizontal mirror
  # horizontal_mirror: true
```

### Resolution Selection

| Resolution | Size | Use Case | Memory Usage |
|------------|------|----------|--------------|
| 320x240 (QVGA) | Small | Testing only | Low |
| 640x480 (VGA) | **Recommended** | Standard meters | Medium |
| 800x600 (SVGA) | Medium | Detailed meters | High |
| 1024x768 (XGA) | Large | High detail needed | Very High |
| 1600x1200 (UXGA) | Very Large | Rarely needed | Maximum |

**Recommendation**: Use 640x480 (VGA) for best balance of quality and performance.

### Image Quality Settings

**JPEG Quality**:
- Range: 1-63 (lower = better quality, larger file)
- Recommended: 8-12
- Too high (>15): Compression artifacts
- Too low (<5): Large files, slow upload

**Frame Rate**:
- `max_framerate`: Rate during active capture
- `idle_framerate`: Rate when idle
- Lower values reduce power consumption
- MeterMonitor only needs 1 image per capture

### Advanced Camera Settings

```yaml
esp32_camera:
  # ... other settings ...

  # Brightness (-2 to 2)
  brightness: 0

  # Contrast (-2 to 2)
  contrast: 0

  # Saturation (-2 to 2)
  saturation: 0

  # Special effect (none, negative, grayscale, etc.)
  special_effect: none

  # White balance (auto, sunny, cloudy, office, home)
  wb_mode: auto

  # Auto exposure control
  aec_mode: auto

  # Auto gain control
  agc_mode: auto
```

---

## Camera Positioning

### Mounting Guidelines

**Distance from Meter**:
- Optimal: 15-25cm
- Minimum: 10cm (too close = out of focus)
- Maximum: 40cm (too far = digits too small)

**Angle**:
- Perpendicular to meter face (90°)
- Avoid angles >30° (causes distortion)
- Align camera parallel to digit row

**Stability**:
- Secure mount (no vibration)
- Fixed position (no movement between readings)
- Consider meter box vibrations (water flow)

**Lighting**:
- Consistent lighting preferred
- Avoid direct sunlight on meter
- Avoid shadows across digits
- Use LED flash if lighting varies

### Mounting Solutions

**3D-Printed Mounts**:
- Design files available online
- Customizable for specific meters
- Can integrate LED flash

**Commercial Mounts**:
- Camera brackets
- Adjustable arms
- Magnetic mounts

**DIY Solutions**:
- Hot glue (temporary)
- Velcro strips
- Cable ties
- Foam tape (vibration dampening)

### Testing Position

Before permanent mounting:
1. Capture test image
2. Check in MeterMonitor:
   - All digits visible
   - Clear digit edges
   - No glare/reflections
   - Even lighting
3. Adjust position if needed
4. Permanently mount once satisfied

---

## LED Flash Usage

### Hardware Connection

Most ESP32-CAM boards have flash output on GPIO4:
- Connect LED to GPIO4
- Use current-limiting resistor (100-330Ω)
- White LED recommended (5000-6500K)

**Pre-installed Flash**:
- Some boards include built-in flash LED
- Usually on GPIO4
- No additional hardware needed

### ESPHome Flash Configuration

```yaml
output:
  - platform: gpio
    pin: GPIO4
    id: led_flash

light:
  - platform: binary
    name: "Watermeter LED"
    output: led_flash
    restore_mode: ALWAYS_OFF  # Important: flash off by default
```

### MeterMonitor Flash Integration

**Using HA Camera Source**:
1. Create HA Camera source in MeterMonitor
2. Set `flash_entity_id`: `light.watermeter_led`
3. Set `flash_delay_ms`: 500-2000ms
4. MeterMonitor will:
   - Turn on flash
   - Wait specified delay
   - Capture image
   - Turn off flash

**Flash Timing**:
- `flash_delay_ms`: Time to wait after turning on flash
- Too short: Flash not fully lit
- Too long: Unnecessary delay
- Recommended: 1000ms (1 second)

### When to Use Flash

**Use Flash When**:
- Meter in dark location
- Inconsistent ambient lighting
- Shadow problems
- Low-light environment

**Don't Use Flash When**:
- Good consistent lighting
- Meter has backlight
- Flash causes reflections/glare
- Power consumption is critical

---

## MQTT Configuration

### ESPHome MQTT Setup

**Option 1: Home Assistant API** (Recommended if using HA):
```yaml
api:
  # Auto-discovered by Home Assistant
```

**Option 2: Direct MQTT**:
```yaml
mqtt:
  broker: 192.168.1.10
  port: 1883
  username: !secret mqtt_username
  password: !secret mqtt_password
  discovery: true
  discovery_prefix: homeassistant
```

### Custom MQTT Image Upload

To send images directly to MeterMonitor via MQTT, you'll need custom ESPHome code:

```yaml
esphome:
  # ... other config ...
  on_boot:
    - priority: -100
      then:
        - script.execute: upload_image

script:
  - id: upload_image
    mode: restart
    then:
      - delay: 60s  # Initial delay
      - while:
          condition:
            lambda: 'return true;'
          then:
            # Capture image
            - logger.log: "Capturing image..."

            # Publish to MQTT (requires custom component)
            # See: https://github.com/your-custom-component

            - delay: 300s  # Wait 5 minutes between captures
```

**Note**: Direct MQTT image upload requires custom ESPHome component. Alternatively, use Home Assistant Camera integration with MeterMonitor polling.

### MQTT Topic Structure

If implementing custom MQTT upload:

**Topic**: `MeterMonitor/{device_name}`

**Payload** (JSON):
```json
{
  "name": "watermeter",
  "WiFi-RSSI": -45,
  "picture": {
    "timestamp": "2026-01-28T10:30:00",
    "format": "jpg",
    "data": "base64_encoded_image_data"
  }
}
```

Minimal accepted payload is:
- `name`
- `picture.data`

Optional:
- `WiFi-RSSI`
- `picture.timestamp` (ISO or Unix seconds/ms)
- `picture.format`

`picture.data` can be either raw base64 or `data:image/...;base64,...`.

### MeterMonitor MQTT Configuration

In MeterMonitor settings (`settings.json` or add-on config):

```json
{
  "mqtt": {
    "broker": "192.168.1.10",
    "port": 1883,
    "topic": "MeterMonitor/#",
    "username": "esp",
    "password": "esp"
  }
}
```

---

## Troubleshooting

### Camera Not Detected

**Symptoms**: ESPHome fails to initialize camera

**Solutions**:
1. Check all camera connections (especially data pins)
2. Verify PSRAM is enabled in config
3. Try different JPEG quality settings
4. Disable serial logging (`baud_rate: 0`)
5. Check power supply (minimum 500mA at 5V)

### Poor Image Quality

**Symptoms**: Blurry, dark, or noisy images

**Solutions**:
1. **Blurry**: Adjust lens focus
2. **Dark**: Increase brightness setting or use flash
3. **Noisy**: Improve lighting, lower ISO (auto exposure)
4. **Washed out**: Decrease brightness or avoid direct light
5. **Color issues**: Adjust white balance mode

### Brownout/Reboot Issues

**Symptoms**: ESP32 reboots when taking pictures

**Solutions**:
1. Use better power supply (>500mA)
2. Shorter USB cable (reduces voltage drop)
3. Add capacitor near ESP32 power pins (100-470µF)
4. Lower resolution or JPEG quality
5. Check for loose connections

### WiFi Connection Problems

**Symptoms**: Device doesn't connect to WiFi

**Solutions**:
1. Verify WiFi credentials in `secrets.yaml`
2. Check WiFi signal strength (use external antenna)
3. Use 2.4GHz network (5GHz not supported)
4. Set static IP if DHCP issues
5. Check router compatibility (some routers have issues)

### MQTT Upload Failures

**Symptoms**: Images not appearing in MeterMonitor

**Solutions**:
1. Verify MQTT broker is running
2. Check MeterMonitor MQTT configuration
3. Ensure topic matches configuration
4. Test with MQTT client (mosquitto_sub)
5. Check image size (too large may timeout)

### Flash Not Working

**Symptoms**: LED doesn't turn on or stays on

**Solutions**:
1. Verify GPIO4 connection
2. Check `restore_mode: ALWAYS_OFF` in config
3. Test manually in Home Assistant
4. Check LED polarity (if external LED)
5. Verify current-limiting resistor

### Focus Problems

**Symptoms**: Digits not sharp

**Solutions**:
1. Rotate lens carefully (counterclockwise = farther, clockwise = closer)
2. Capture test images while adjusting
3. Use fixed distance (15-25cm)
4. Clean lens if dirty/smudged
5. Replace camera module if damaged

### Memory Issues

**Symptoms**: Out of memory errors, crashes

**Solutions**:
1. Ensure PSRAM enabled
2. Lower resolution (640x480)
3. Increase JPEG quality number (more compression)
4. Reduce `max_framerate`
5. Disable unused ESPHome components

---

## Power Consumption

### Typical Power Usage

- Idle: 50-100mA at 5V
- During capture: 200-400mA at 5V
- With flash: +50-200mA (depends on LED)

**For Battery Operation**:
- Not recommended (high power usage)
- Deep sleep between captures possible but complex
- Consider USB power or power adapter

### Power Supply Options

**USB Power**:
- Most reliable
- Use quality USB adapter (>500mA)
- Short cable (<1m) to reduce voltage drop

**External Power Supply**:
- 5V DC regulated supply
- Minimum 1A capacity recommended
- Connect to 5V and GND pins

**Power over Ethernet (PoE)**:
- Requires PoE splitter (48V → 5V)
- Good for remote installations
- More expensive but very reliable

---

## Advanced Topics

### Custom Firmware

For advanced users, custom firmware can:
- Send images directly via MQTT
- Implement adaptive capture intervals
- Add motion detection
- Optimize power consumption

**Resources**:
- ESPHome custom components
- Arduino ESP32 Camera library
- AI-on-the-edge-device project (inspiration)

### Multiple Cameras

To monitor multiple meters:
1. Create separate ESPHome configurations
2. Use unique device names
3. Configure separate MQTT topics or HA camera entities
4. Create separate sources in MeterMonitor

### Remote Access

For remote installations:
- Use Home Assistant Nabu Casa (cloud access)
- VPN to home network
- Port forwarding (less secure)
- Dynamic DNS service

---

## Additional Resources

- [ESPHome ESP32 Camera Documentation](https://esphome.io/components/esp32_camera.html)
- [ESP32-CAM Pinout Reference](https://randomnerdtutorials.com/esp32-cam-ai-thinker-pinout/)
- [OV2640 Datasheet](https://www.uctronics.com/download/cam_module/OV2640DS.pdf)
- [User Guide](user-guide.md)
- [Troubleshooting Guide](troubleshooting.md)

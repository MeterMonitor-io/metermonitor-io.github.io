# User Guide

This guide walks through everything you can do in the MeterMonitor web interface.

---

## Discovery

When MeterMonitor receives its first image from a device it creates an entry in the **Discovery** tab. New meters appear here automatically — you don't need to create them manually.

### MQTT Setup Helper

The Discovery tab includes an **MQTT Setup Helper** panel that shows:

- The broker address and topic MeterMonitor is listening on
- The exact JSON payload format expected from your device

**Expected MQTT payload** (topic: `MeterMonitor/<device-name>`):

```json
{
  "name": "unique-device-name",
  "WiFi-RSSI": -57,
  "picture": {
    "format": "jpeg",
    "timestamp": "2025-06-01T10:00:00",
    "data": "iVBORw0KGgoAAAANSUhEUgAA..."
  }
}
```

Required fields: `name`, `picture.data`.
`WiFi-RSSI`, `picture.format`, and `picture.timestamp` are optional.
`picture.data` can be raw base64 or a data-URI (`data:image/jpeg;base64,...`).

---

## Meter Setup

Click a discovered meter to start setup. This configures how images are processed.

### Step 1 — Choose a display extraction method

MeterMonitor needs to locate the digit display within the camera image. Four methods are available:

#### YOLO (default — recommended)
Automatic AI-based detection. Works for most standard water meters with no configuration needed.

- Enable **Rotated 180°** if the camera is mounted upside-down.
- Enable **Extended last digit** if the rightmost digit is getting cut off.

#### ORB (template-based)
Uses a reference image and manually marked corner points to find the display. More reliable when YOLO fails (unusual meters, difficult lighting).

1. Capture or upload a clear reference image.
2. Drag the four corner handles to mark the exact edges of the digit display (clockwise: top-left → top-right → bottom-right → bottom-left).
3. Name and save the template.
4. Templates can be reused across meters of the same model.

> **Tip:** Mark the digit area only — exclude the meter housing. Leave a small margin around the digits.

**Per-digit bounding boxes (optional)**

If digits have varying sizes, unusual spacing, or ORB accuracy is still low after marking the display area, switch to **Each digit** mode. Instead of marking the overall display, you draw an individual bounding box for each digit:

1. Change the segment mode to **Each digit** in the configurator.
2. Draw one bounding quad per digit (in order, left to right).
3. The number of quads must match the **Segments** count.
4. Save the template.

This mode is useful for meters where digits are not evenly spaced or overlap with non-digit elements.

#### Static Rectangle
Crops a fixed rectangle from every image — no detection involved. Use this when the camera is rigidly mounted and the display is always at the same position.

1. Enter the X/Y coordinates and width/height of the crop rectangle in pixels.
2. The crop is applied to every image without any matching.

Good choice for IP cameras or when both YOLO and ORB are unreliable.

**Per-digit bounding boxes (optional)**

Static Rectangle also supports **Each digit** mode — define individual crop regions for each digit instead of one shared rectangle. This is useful when digits are non-uniformly spaced or some digits need a different crop:

1. Change the segment mode to **Each digit**.
2. Define a bounding quad for each digit position.
3. The number of quads must match the **Segments** count.

#### Bypass
Passes the entire image directly to digit recognition without any cropping. Only useful for testing or when images are already pre-cropped.

---

### Step 2 — Configure thresholds

Thresholds convert the grayscale digit image to a clean black-and-white image for recognition.

#### Automatic threshold search

Click **Search Thresholds** and choose a step count (3–25). A higher count gives better results but takes longer. The system tests many combinations and shows the best result with sample digit previews.

#### Manual adjustment

- **Threshold Low / High** — the grayscale brightness range (0–255) that is treated as "digit". Pixels within this range are set to white; everything else is black.
- **Threshold Last Low / High** — separate thresholds for the decimal-part digits (rightmost). Useful when those digits are printed in a different color or have different contrast.

> **Reading the preview:** white pixels should show only the digit shape on a black background. If the digit looks broken, raise `threshold_high`. If there is too much noise, lower it.

#### Islanding Padding

Percentage of the digit area to ignore at the edges (default 20%). Increase this if edge artifacts (lines, borders) are being picked up as digit pixels.

#### Decimal position

Use the decimal control (the `.` with arrows) to set how many of the rightmost digits are after the decimal point. This also determines which digits use the `threshold_last_*` values.

---

### Step 3 — Per-digit model selection

Below the digit preview thumbnails, each digit has a small icon you can click to toggle its recognition model:

- **Rotating wheel** (default) — for classic analog drums that rotate continuously
- **7-segment** — for LCD/7-segment style digits

Mixed setups are supported: you can use rotating-wheel for most digits and 7-segment for just the last one (or vice versa).

---

### Step 4 — Layout settings

| Setting | When to use |
|---|---|
| **Segments** | Set this to the number of digits your meter shows (typically 6–8) |
| **Rotated 180°** | Camera is mounted upside-down |
| **Shrink last 3** | Fractional digits are narrower than the main digits on your meter |
| **Extended last digit** | The rightmost digit (rotation indicator) is getting cut off at the edge |

---

### Step 5 — Error correction mode

#### Full mode (recommended)
Comprehensive correction applied to every reading:
- Rotation-class digits ('r') replaced with the last known value
- Readings that decrease are rejected or corrected
- Readings with an impossible flow rate are rejected
- Low-confidence digits fall back to the previous prediction
- Roll-over transitions (9→0) handled correctly

Configure **Max flow rate** (m³/h) to match your installation's realistic peak. The default (1.0 m³/h) works for most households.

#### Light mode
Minimal correction — only rotation-class and explicitly denied digits are replaced. No flow-rate validation. Use this when debugging or when the full algorithm is rejecting valid readings.

---

### Step 6 — Set initial value

Enter the current meter reading. This is used as the starting point for consumption tracking and error correction. Once saved, setup is complete and the meter appears in the **Watermeters** tab.

---

## Reading history and evaluations

### History chart

Each meter shows a chart of readings over time. Green points are automatic readings; blue points are manual entries. Hover over a point to see the exact value, confidence, and timestamp.

### Evaluation details

Click any reading in the evaluation list to see:
- Individual digit images (colored and thresholded)
- Top-3 predictions and confidence per digit
- Correction metadata: flow rate, delta, rejection reason (if any)

### Confidence scores

| Range | Meaning |
|---|---|
| > 0.9 | Excellent |
| 0.8–0.9 | Good |
| 0.6–0.8 | Acceptable — monitor for issues |
| < 0.6 | Poor — check thresholds and lighting |

### Manual entries

If a reading is wrong, you can add a manual correction:
1. Go to the meter's history
2. Click **Add Manual Entry**
3. Enter the correct value and timestamp

Manual entries are marked separately in the chart and are used by the correction algorithm as reference points.

---

## Sources

Sources define how images reach MeterMonitor. Navigate to the **Sources** tab to manage them.

### MQTT source
Created automatically when an ESP32 or other device publishes its first image. You can enable/disable it but cannot create one manually.

### Home Assistant Camera source
Polls a Home Assistant camera entity on a schedule.

**Setup:**
1. **Sources → Create Source → HA Camera**
2. Set the meter name, camera entity ID (e.g. `camera.watermeter`), and poll interval (seconds)
3. Optionally configure a flash entity and flash delay (ms) — MeterMonitor will turn on the light before each capture and off afterwards
4. Click **Test Capture** to verify, then enable the source

### HTTP source
Fetches an image from an HTTP endpoint on a schedule.

**Setup:**
1. **Sources → Create Source → HTTP**
2. Set the URL, poll interval, and any required headers or request body
3. The endpoint must return a JPEG or PNG image

---

## Home Assistant integration

After meter setup is complete, MeterMonitor publishes readings to MQTT. Home Assistant picks these up via MQTT auto-discovery and creates a sensor automatically:

- Entity: `sensor.watermeter_<name>_value`
- Unit: m³
- Device class: `water`
- State class: `total_increasing`

If auto-discovery doesn't work, add manually in `configuration.yaml`:

```yaml
mqtt:
  sensor:
    - name: "Water Meter"
      state_topic: "homeassistant/sensor/watermeter_kitchen/state"
      unit_of_measurement: "m³"
      device_class: "water"
      state_class: "total_increasing"
      json_attributes_topic: "homeassistant/sensor/watermeter_kitchen/attributes"
```

---

## Dataset export

Export digit images from stored evaluations — useful for reviewing misclassifications or contributing to model training.

1. Meter detail → **Export Dataset**
2. Download the ZIP file — it contains labeled digit images organised by predicted digit class

---

## Common tasks

### Improving recognition accuracy

1. Check lighting — consistent, shadow-free illumination gives the best results
2. Run **Threshold Search** (automatic is a good starting point)
3. Verify the number of **Segments** matches your meter
4. If YOLO detection keeps failing, try the **ORB** or **Static Rectangle** extractor
5. Increase `conf_threshold` to reject low-quality readings automatically

### After changing thresholds or extractor

Click **Reset Corr. Alg.** in the meter context menu (three-dot menu) so the correction algorithm starts fresh with the new configuration.

### Monitoring MQTT connection

A red notification in the top-right corner means MeterMonitor lost connection to the MQTT broker. Reconnection is automatic with exponential backoff. Check the [Troubleshooting](troubleshooting.md) guide if it doesn't recover.

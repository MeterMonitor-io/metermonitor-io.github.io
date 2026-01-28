# MeterMonitor User Guide

Complete guide to using MeterMonitor for automated water meter reading.

## Table of Contents

1. [Getting Started](#getting-started)
2. [Initial Meter Discovery](#initial-meter-discovery)
3. [Meter Setup](#meter-setup)
4. [Configuration](#configuration)
5. [Reading History](#reading-history)
6. [Dataset Export](#dataset-export)
7. [Common Tasks](#common-tasks)

---

## Getting Started

MeterMonitor is an AI-powered system that reads analog water meters using ESP32 cameras. The system processes images on a central server, reducing device power consumption while providing accurate readings integrated with Home Assistant.

### Prerequisites

- Running MeterMonitor instance (Home Assistant add-on or standalone)
- ESP32-CAM device with ESPHome firmware (see [esp32-setup.md](esp32-setup.md))
- MQTT broker configured and running
- Basic understanding of your water meter layout

---

## Initial Meter Discovery

When MeterMonitor receives its first image from an ESP32 device, it automatically appears in the **Discovery** section.

### Discovery Process

1. Navigate to the **Discovery** tab in the web interface
2. New meters appear with:
   - Meter name (from MQTT topic)
   - Last image timestamp
   - WiFi signal strength (RSSI)
   - Source type (MQTT, HA Camera, or HTTP)

3. Click on a discovered meter to begin setup

### What Happens During Discovery

- Meter entry is created in the database
- Default settings are applied:
  - Threshold: 0-100 (low), 0-100 (high)
  - Islanding padding: 20%
  - Segments: 7 digits
  - ROI extractor: YOLO (automatic detection)
  - Correctional algorithm: Full mode enabled

---

## Meter Setup

Setup is required before MeterMonitor can track consumption. This process calibrates the system to your specific meter.

### Step 1: Select ROI Extractor

Choose how the display region should be detected:

#### YOLO (Automatic Detection)
- **Best for**: Standard meter displays with clear digit regions
- **Advantages**: Fully automatic, no manual configuration
- **Requirements**: Meter display must be clearly visible
- **Settings**:
  - `rotated_180`: Enable if meter is upside-down
  - `extended_last_digit`: Extends detection area for rightmost digit

#### ORB (Template-Based)
- **Best for**: Non-standard meters, poor lighting, or YOLO detection failures
- **Advantages**: Works with difficult meter layouts, robust to lighting changes
- **Requirements**: Reference image with clear meter display
- **Process**:
  1. Capture reference image
  2. Mark 4 corner points of the digit display (clockwise from top-left)
  3. Save template
  4. Select template in meter settings

#### Bypass
- **Best for**: Testing or when display is already extracted
- **Use case**: Advanced debugging only

### Step 2: Configure Thresholds

Thresholds determine how digits are extracted from the image.

#### Automatic Threshold Search

1. Click **Search Thresholds** in setup
2. Select search steps (3-25):
   - Lower values: Faster but less accurate
   - Higher values: Slower but more precise
3. System tests combinations and selects optimal values
4. Review results:
   - Average confidence score
   - Sample digit images
5. Apply if satisfactory

#### Manual Threshold Adjustment

**Main Digit Thresholds**:
- `threshold_low`: Minimum pixel brightness (0-255)
- `threshold_high`: Maximum pixel brightness (0-255)
- Adjust to isolate digit pixels from background

**Last Digit Thresholds** (if different):
- `threshold_last_low`: Separate threshold for rightmost digit
- `threshold_last_high`: Often needs different values due to rotation indicator

**Islanding Padding**:
- Percentage of digit area to ignore at edges (default: 20%)
- Prevents edge artifacts from affecting recognition

**Tips**:
- Use the preview to see thresholded digits
- White pixels should show only the digit shape
- Black background indicates good thresholding
- If digits are broken/incomplete, increase `threshold_high`
- If too much noise, decrease `threshold_high`

### Step 3: Configure Layout

**Segments**: Number of digits on your meter (typically 7)

**Layout Options**:
- `shrink_last_3`: Reduces width of last 3 digits (for compact layouts)
- `extended_last_digit`: Extends rightmost digit capture area
- `rotated_180`: Flips image if meter is upside-down

### Step 4: Set Correctional Algorithm

Choose between two correction modes:

#### Full Mode (Recommended)
Applies comprehensive corrections:
- Replaces rotation class ('r') with last known value
- Validates positive flow (consumption only increases)
- Checks max flow rate (rejects impossible increases)
- Falls back to previous predictions on low confidence
- Handles digit roll-over (9→0 transitions)

**Configuration**:
- `max_flow_rate`: Maximum expected flow in m³/h (default: 1.0)
- Set based on your household's peak consumption

#### Light Mode
Minimal corrections only:
- Replaces rotation class ('r') with last value
- Replaces denied/low-confidence digits with last value
- No flow rate validation
- Faster processing

**When to use**:
- Testing and debugging
- Unusual consumption patterns
- Manual flow validation

### Step 5: Set Initial Value

1. Enter current meter reading manually
2. Format: Enter full reading including leading zeros
3. Click **Finish Setup**

This value is used as the baseline for:
- Consumption tracking
- Correctional algorithm validation
- History chart starting point

### Step 6: Verify Setup

After setup completion:
1. Meter appears in **Watermeters** tab
2. Check first reading appears in history
3. Verify evaluation result shows correct digits
4. Monitor confidence scores (should be >0.8 for good readings)

---

## Configuration

### Source Management

Sources define how images reach MeterMonitor.

#### MQTT Source
Automatically created when ESP32 device sends first image.

**Configuration**:
- Cannot be created via UI (automatic only)
- Can be disabled/enabled
- View last success timestamp and errors

#### Home Assistant Camera Source
Poll camera entities in Home Assistant at regular intervals.

**Setup**:
1. Navigate to **Sources** tab
2. Click **Create Source**
3. Select source type: `HA Camera`
4. Configure:
   - Meter name (must match existing meter)
   - Camera entity ID (e.g., `camera.watermeter`)
   - Flash entity ID (optional LED control)
   - Flash delay (ms to wait after turning on flash)
   - Poll interval (seconds between captures)
5. Test capture to verify configuration
6. Enable source

**Flash Configuration**:
- Useful for ESP32-CAM with LED flash
- Turns on light before capture
- Automatically turns off after capture
- Configurable delay for flash stabilization

#### HTTP Source
Fetch images from HTTP endpoints.

**Setup**:
1. Create source with type `HTTP`
2. Configure:
   - URL (http:// or https://)
   - Custom headers (optional JSON object)
   - Request body (optional)
   - Poll interval
3. Endpoint must return image data (JPEG/PNG)

**Example Use Cases**:
- IP cameras with snapshot URLs
- Custom image capture services
- Webhook-based image delivery

### ROI Extractor Settings

Can be changed at any time in meter settings:

1. Navigate to meter details
2. Click **Edit Settings**
3. Change `roi_extractor`:
   - `yolo`: Automatic detection
   - `orb`: Template-based (requires template)
   - `bypass`: No extraction
4. For ORB, select template from dropdown
5. Save changes

**Note**: Changing ROI extractor marks all evaluations as outdated. Re-evaluation will occur automatically on next image.

### Template Management

Templates are reference configurations for ORB-based extraction.

#### Creating Templates

1. Navigate to meter setup or settings
2. Select **ORB** extractor
3. Capture or upload reference image
4. Mark 4 corner points:
   - Top-left
   - Top-right
   - Bottom-right
   - Bottom-left
5. Preview extracted region
6. Name template and save

#### Template Point Editor

Interactive editor features:
- Drag points to adjust corners
- Zoom in/out for precision
- Reset to default positions
- Real-time preview of extraction

**Tips**:
- Choose an image with clear digit display
- Ensure all digits are within marked region
- Avoid including frame/housing in region
- Leave small margin around digits

#### Editing Templates

Templates are immutable after creation. To modify:
1. Create new template with updated points
2. Update meter settings to use new template
3. Delete old template if no longer needed

---

## Reading History

### Viewing History

**History Chart**:
- X-axis: Timestamp
- Y-axis: Meter reading (m³)
- Line color: Green for automatic, Blue for manual entries
- Hover for details: Exact value, confidence, time

**History Table**:
- Columns: Value, Timestamp, Confidence, Manual flag
- Sort by any column
- Export to CSV

**Confidence Scores**:
- Range: 0.0 to 1.0
- >0.9: Excellent recognition
- 0.8-0.9: Good recognition
- 0.6-0.8: Acceptable, monitor for issues
- <0.6: Poor recognition, check thresholds/lighting

### Evaluation Details

Each reading has detailed evaluation data:

**View Evaluation**:
1. Click on evaluation in list
2. Dialog shows:
   - Colored digit images (original segments)
   - Thresholded digits (after binary threshold)
   - Predictions (top 3 for each digit)
   - Confidence per digit
   - Correctional metadata

**Correctional Metadata** (Full mode only):
- `flow_rate_m3h`: Calculated flow rate
- `delta_m3`: Change since last reading
- `delta_raw`: Raw digit difference
- `time_diff_min`: Time since last reading
- `rejection_reason`: Why reading was rejected (if applicable)
- `fallback_digit_count`: Digits corrected from previous value
- `digits_changed_vs_last`: Digits different from last reading
- `digits_changed_vs_top_pred`: Corrections applied to predictions

### Manual Entries

Add manual readings to correct errors:

1. Navigate to meter history
2. Click **Add Manual Entry**
3. Enter correct reading
4. Set timestamp (defaults to now)
5. Save

Manual entries:
- Override incorrect automatic readings
- Used by correctional algorithm as reference
- Marked differently in charts
- Cannot be automatically deleted

### Clearing History

Delete all readings for a meter:
1. Go to meter details
2. Click context menu (three dots)
3. Select **Clear History**
4. Confirm deletion

**Warning**: This action cannot be undone. Export data first if needed.

---

## Dataset Export

Export digit images for training custom models or analysis.

### Export Process

1. Navigate to meter details
2. Click **Export Dataset**
3. System collects:
   - Colored digit images
   - Thresholded digit images
   - Labels (predicted digits)
4. Download ZIP file

### Dataset Structure

```
meter_name_dataset.zip
├── color/
│   ├── 0/
│   │   ├── 0_metername_hash1.png
│   │   └── 0_metername_hash2.png
│   ├── 1/
│   ├── ...
│   └── r/  (rotation class)
└── th/
    ├── 0/
    │   ├── 0_metername_hash1.png
    │   └── 0_metername_hash2.png
    ├── 1/
    └── ...
```

**File Naming**: `{label}_{metername}_{crc32hash}.png`
- Label: Predicted digit (0-9 or r)
- Meter name: Sanitized meter identifier
- Hash: CRC32 checksum for uniqueness

### Using Exported Data

**Model Training**:
- Images are pre-segmented and labeled
- Suitable for CNN training datasets
- Thresholded images match model input format

**Quality Analysis**:
- Review misclassifications
- Identify problematic digit patterns
- Assess threshold effectiveness

**Dataset Management**:
- Delete dataset: Removes all exported images
- Download: Creates fresh ZIP from stored evaluations
- Exports respect `max_evals` setting (configurable in settings)

---

## Common Tasks

### Improving Recognition Accuracy

**Low Confidence Scores**:
1. Check lighting - ensure consistent illumination
2. Adjust thresholds - run automatic search
3. Try ORB extractor if YOLO fails
4. Increase `conf_threshold` to reject low-quality readings

**Incorrect Readings**:
1. Verify segment count matches meter
2. Check `rotated_180` setting
3. Review denied digits in evaluations
4. Adjust correctional algorithm mode

**Intermittent Failures**:
1. Check WiFi signal strength (RSSI)
2. Verify camera positioning (stable mount)
3. Review flash timing (if using LED)
4. Monitor source errors in source list

### Changing Meter Configuration

**Safe Changes** (no re-evaluation needed):
- Poll interval
- Flash settings
- Source enable/disable

**Changes Requiring Re-evaluation**:
- Thresholds
- ROI extractor
- Segment count
- Layout options

To trigger re-evaluation:
1. Change settings
2. Click **Reset Corr. Alg.** in context menu
3. Wait for next image or trigger manual capture

### Monitoring System Health

**Check MQTT Connection**:
- Alerts appear in top-right corner
- Red alert: MQTT disconnected
- Reconnection is automatic with exponential backoff

**Source Health**:
- Navigate to **Sources** tab
- Check `last_success_ts` column
- Review `last_error` for failures
- Enable/disable sources as needed

**Database Maintenance**:
- History and evaluations are automatically limited by `max_history` and `max_evals`
- Oldest entries are pruned automatically
- Manual cleanup: Delete meter to remove all associated data

### Integration with Home Assistant

**MQTT Discovery**:
- Automatic sensor creation (if Mosquitto add-on used)
- Sensor name: `sensor.watermeter_{metername}_value`
- Attributes: Confidence, timestamp, source type

**Manual YAML Configuration** (if discovery disabled):

```yaml
mqtt:
  sensor:
    - name: "Water Meter Reading"
      state_topic: "homeassistant/sensor/watermeter_kitchen/state"
      unit_of_measurement: "m³"
      device_class: "water"
      state_class: "total_increasing"
      json_attributes_topic: "homeassistant/sensor/watermeter_kitchen/attributes"
```

**Using Camera Integration**:
- Create HA Camera source instead of MQTT
- MeterMonitor polls camera entity
- Useful for existing camera entities
- Supports flash control for ESP32-CAM

---

## Tips and Best Practices

### Camera Positioning
- Mount camera perpendicular to meter face
- Distance: 10-30cm depending on lens
- Avoid reflections and glare
- Ensure entire digit display is visible
- Use flash in low-light conditions

### Threshold Tuning
- Start with automatic search
- Fine-tune manually if needed
- Different meters may need different values
- Last digit often needs separate thresholds
- Preview thresholded images to verify

### Correctional Algorithm
- Use Full mode for most cases
- Set realistic max_flow_rate
- Light mode for testing/debugging
- Manual entries help algorithm learn

### Performance
- Limit `max_evals` to reduce database size
- Export datasets before clearing evaluations
- Use YOLO extractor when possible (faster than ORB)
- Increase poll intervals to reduce processing load

### Troubleshooting
- Check evaluation details for insight into failures
- Monitor confidence trends over time
- Review rejection reasons in evaluation metadata
- Use manual entries to correct persistent errors

---

## Next Steps

- [ESP32 Hardware Setup](esp32-setup.md)
- [ROI Extractor Guide](roi-extractors.md)
- [API Reference](api-reference.md)
- [Troubleshooting Guide](troubleshooting.md)
- [Advanced Topics](advanced-topics.md)

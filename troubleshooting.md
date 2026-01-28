# MeterMonitor Troubleshooting Guide

Comprehensive troubleshooting guide for common issues.

## Table of Contents

1. [Installation Issues](#installation-issues)
2. [MQTT Connection Problems](#mqtt-connection-problems)
3. [Image Quality and Detection](#image-quality-and-detection)
4. [Recognition Accuracy](#recognition-accuracy)
5. [Threshold Tuning](#threshold-tuning)
6. [Performance and Memory](#performance-and-memory)
7. [Database Errors](#database-errors)
8. [Home Assistant Integration](#home-assistant-integration)
9. [Source and Polling Issues](#source-and-polling-issues)
10. [Reporting Issues](#reporting-issues)

---

## Installation Issues

### Home Assistant Add-on Won't Start

**Symptoms**: Add-on shows error state or fails to start

**Solutions**:

1. **Check logs** in add-on interface:
   ```
   Configuration → Add-ons → MeterMonitor → Log
   ```

2. **Common causes**:
   - Invalid configuration (check MQTT settings)
   - Port conflict (default 8070)
   - Insufficient memory (minimum 512MB RAM)

3. **Verify MQTT broker**:
   - Ensure Mosquitto add-on is installed and running
   - Check MQTT credentials in configuration

4. **Reset to defaults**:
   - Remove custom configuration
   - Restart add-on
   - Reconfigure step by step

### Standalone Installation Fails

**Symptoms**: Dependencies won't install or runtime errors

**Solutions**:

1. **Check Python version**:
   ```bash
   python3 --version  # Requires Python 3.8+
   ```

2. **Install system dependencies**:
   ```bash
   # Ubuntu/Debian
   sudo apt-get update
   sudo apt-get install python3-pip python3-dev build-essential libopencv-dev

   # Alpine (Docker)
   apk add python3 py3-pip gcc musl-dev opencv-dev
   ```

3. **Use virtual environment**:
   ```bash
   python3 -m venv .venv
   source .venv/bin/activate
   pip install -r requirements.txt
   ```

4. **Check requirements.txt versions**:
   - onnxruntime may fail on some platforms
   - Try platform-specific wheels
   - Use `pip install --upgrade pip`

### Docker Build Fails

**Symptoms**: Docker build errors or missing files

**Solutions**:

1. **Check Docker version** (requires BuildKit):
   ```bash
   docker --version  # Requires 19.03+
   docker buildx version
   ```

2. **Enable BuildKit**:
   ```bash
   export DOCKER_BUILDKIT=1
   docker build -t metermonitor .
   ```

3. **Frontend build fails**:
   - Check Node.js version in frontend container
   - Ensure `yarn.lock` is present
   - Try: `cd frontend && yarn install && yarn build`

4. **Multi-platform build**:
   ```bash
   docker buildx build --platform linux/amd64,linux/arm64 -t metermonitor .
   ```

### Port Already in Use

**Symptoms**: "Address already in use" error

**Solutions**:

1. **Check what's using port 8070**:
   ```bash
   # Linux
   sudo netstat -tulpn | grep 8070
   sudo lsof -i :8070

   # Kill process
   sudo kill -9 <PID>
   ```

2. **Change MeterMonitor port**:
   - Home Assistant: Edit add-on configuration
   - Standalone: Modify `settings.json` → `http.port`

3. **Docker port mapping**:
   ```bash
   docker run -p 8071:8070 metermonitor  # Map to different host port
   ```

---

## MQTT Connection Problems

### MQTT Connection Failed

**Symptoms**: Red alert "Failed to connect to MQTT broker"

**Solutions**:

1. **Verify broker is running**:
   ```bash
   # Test with mosquitto_sub
   mosquitto_sub -h 192.168.1.10 -p 1883 -t "test/#" -u username -P password
   ```

2. **Check MeterMonitor configuration**:
   - Correct broker IP/hostname
   - Correct port (1883 default, 8883 for TLS)
   - Valid username and password
   - Topic pattern includes wildcard (`MeterMonitor/#`)

3. **Firewall issues**:
   ```bash
   # Check if port is accessible
   telnet 192.168.1.10 1883
   nc -zv 192.168.1.10 1883
   ```

4. **Home Assistant Mosquitto**:
   - Default IP: `172.30.32.1` (for add-ons)
   - Check Mosquitto add-on configuration
   - Verify user credentials in Mosquitto settings

### MQTT Reconnection Loop

**Symptoms**: Constantly connecting/disconnecting

**Solutions**:

1. **Check logs** for specific error:
   - Authentication failure
   - Connection refused
   - Timeout

2. **Broker overload**:
   - Check broker resource usage
   - Reduce client count
   - Increase broker limits in configuration

3. **Network stability**:
   - Verify network connectivity
   - Check for packet loss
   - Use wired connection if possible

### Messages Not Received

**Symptoms**: ESP32 sends messages but MeterMonitor doesn't process them

**Solutions**:

1. **Verify topic match**:
   - ESP32 publishes to: `MeterMonitor/kitchen_meter`
   - MeterMonitor subscribes to: `MeterMonitor/#`
   - Topics are case-sensitive!

2. **Check message format** (see [API Reference](api-reference.md#mqtt-message-format)):
   ```bash
   mosquitto_sub -h broker_ip -t "MeterMonitor/#" -v
   ```
   Verify JSON structure is correct

3. **MQTT source disabled**:
   - Check **Sources** tab in UI
   - Ensure MQTT source for meter is enabled
   - Re-enable if disabled

4. **Message size too large**:
   - Check broker max message size
   - Reduce image resolution/quality
   - Default MQTT limit: 256MB (usually sufficient)

---

## Image Quality and Detection

### No Bounding Box Detected

**Symptoms**: Error "No display detected in image"

**Solutions**:

1. **Check image quality**:
   - View latest image in meter details
   - Ensure display is visible and clear
   - Check focus (adjust lens if blurry)

2. **Lighting issues**:
   - Too dark: Enable flash or add lighting
   - Too bright: Reduce exposure or move camera
   - Glare: Reposition camera to avoid reflections

3. **Camera positioning**:
   - Entire display must be visible
   - Distance: 15-25cm recommended
   - Angle: Perpendicular to meter face

4. **Try different ROI extractor**:
   - Switch from YOLO to ORB (see [roi-extractors.md](roi-extractors.md))
   - Create template for ORB
   - ORB is more robust to lighting

5. **Check evaluation details**:
   - View colored/thresholded digits
   - Identify specific problem area

### Blurry Images

**Symptoms**: Images are out of focus

**Solutions**:

1. **Adjust lens focus** (ESP32-CAM):
   - Rotate lens carefully (counterclockwise = farther)
   - Capture test images while adjusting
   - Lock focus with glue when optimal

2. **Camera shake/vibration**:
   - Secure mount (no movement)
   - Check for vibration from water flow
   - Add dampening material

3. **Motion blur**:
   - ESP32 camera settings (too high framerate)
   - Ensure low framerate (1 fps or less)
   - Increase exposure time if too dark

4. **Lens quality**:
   - Some OV2640 lenses are lower quality
   - Consider replacement lens
   - Clean lens if dirty/scratched

### Dark Images

**Symptoms**: Images are underexposed

**Solutions**:

1. **Enable LED flash**:
   - Configure flash entity in HA Camera source
   - Set appropriate flash delay (1000-2000ms)
   - Verify flash LED is working

2. **ESPHome camera settings**:
   ```yaml
   esp32_camera:
     brightness: 1  # Increase (range: -2 to 2)
     contrast: 0
   ```

3. **Add external lighting**:
   - LED strip or bulb near meter
   - Consistent lighting better than flash
   - Avoid shadows across digits

4. **Adjust threshold search**:
   - Lower brightness images need different thresholds
   - Run automatic threshold search
   - Adjust `threshold_high` manually

### Glare and Reflections

**Symptoms**: Bright spots obscuring digits

**Solutions**:

1. **Reposition camera**:
   - Change angle to avoid direct reflection
   - Move slightly left/right or up/down
   - Test different mounting positions

2. **Diffuse lighting**:
   - Use diffuser on flash LED
   - Replace direct light with indirect
   - Avoid flash if possible (use ambient)

3. **Polarizing filter** (advanced):
   - Add polarizing filter to camera lens
   - Reduces reflections
   - May require custom mount

4. **Change meter cover** (if possible):
   - Matte finish better than glossy
   - Some meters have replaceable covers

---

## Recognition Accuracy

### Low Confidence Scores

**Symptoms**: Readings accepted but confidence < 0.8

**Solutions**:

1. **Improve image quality** (see above sections)

2. **Tune thresholds**:
   - Run automatic threshold search
   - Verify thresholded digits are clean (white on black)
   - Adjust `islanding_padding` if digits cut off

3. **Check segment count**:
   - Must match number of digits on meter
   - Include all digits (including last rotation indicator)

4. **Set confidence threshold**:
   ```json
   {
     "conf_threshold": 0.7  // Reject readings below 70%
   }
   ```
   - Lower = more readings accepted (may be incorrect)
   - Higher = fewer readings (only high confidence)

5. **Review predictions** in evaluation details:
   - Check top 3 predictions per digit
   - Identify consistently problematic digits
   - May indicate threshold or segmentation issue

### Incorrect Digit Recognition

**Symptoms**: Wrong digits read (e.g., 5 read as 6)

**Solutions**:

1. **Check thresholding**:
   - View thresholded digits in evaluation
   - Should show clear digit shape
   - If broken/noisy, adjust thresholds

2. **Segmentation issues**:
   - Verify segment width is correct
   - Check `shrink_last_3` setting
   - Ensure digits not cut off at edges

3. **Similar digits confused**:
   - 5/6, 3/8, 1/7 commonly confused
   - Improve image quality (focus, lighting)
   - Retrain model (advanced, see [development.md](development.md))

4. **Rotation indicator ('r' class)**:
   - Last digit may show partial rotation
   - Correctional algorithm handles this
   - Check if `extended_last_digit` helps

5. **Export dataset** for analysis:
   - Review exported digit images
   - Identify patterns in misclassifications
   - May reveal systematic issue

### Readings Not Updating

**Symptoms**: Same value repeated, no consumption tracked

**Solutions**:

1. **Check correctional algorithm**:
   - Verify `use_correctional_alg` is enabled
   - Check `max_flow_rate` isn't too restrictive
   - Review rejection reasons in evaluation metadata

2. **Negative flow rejection**:
   - Algorithm rejects decreasing values by default
   - Check `allow_negative_correction` in config
   - Add manual entry if meter was read backwards

3. **No water usage**:
   - Meter may genuinely not have changed
   - Verify by checking meter physically
   - Normal for low-usage periods

4. **Time difference issues**:
   - Check timestamps in evaluation metadata
   - Ensure ESP32 time is correct (NTP sync)
   - Algorithm requires positive time difference

5. **Last digit not changing**:
   - Smallest digit may not move for hours
   - This is normal behavior
   - Wait for higher-order digit to change

### Erratic Readings (Jumping Values)

**Symptoms**: Values jump up/down unpredictably

**Solutions**:

1. **Use Full correctional algorithm**:
   - Enable `use_correctional_alg: true`
   - Set realistic `max_flow_rate`
   - Algorithm filters impossible changes

2. **Improve detection stability**:
   - Better ROI extraction (try ORB if using YOLO)
   - More stable camera mounting
   - Consistent lighting

3. **Check thresholds**:
   - Inconsistent thresholding causes digit flickering
   - Run threshold search
   - Use fixed lighting for consistency

4. **Manual corrections**:
   - Add manual history entries for correct values
   - Algorithm uses these as reference points
   - Helps "anchor" future readings

---

## Threshold Tuning

### Automatic Threshold Search Returns Poor Results

**Symptoms**: Search completes but digits not well extracted

**Solutions**:

1. **Increase search steps**:
   - Default: 10 steps
   - Try: 15-20 steps (slower but more thorough)
   - Maximum: 25 steps

2. **Improve source image**:
   - Capture better quality reference image
   - Adjust lighting before search
   - Ensure focus is sharp

3. **Manual tuning** after search:
   - Use search result as starting point
   - Fine-tune by ±5 in each direction
   - Preview results in real-time

4. **Search gets stuck**:
   - Timeout: Increase (in code, default 30s)
   - Reduce steps temporarily
   - Check CPU usage (may be overloaded)

### Thresholds Work for Some Digits, Not Others

**Symptoms**: First digits good, last digit bad (or vice versa)

**Solutions**:

1. **Use separate thresholds for last digit**:
   ```json
   {
     "threshold_low": 0,
     "threshold_high": 100,
     "threshold_last_low": 0,    // Different for last digit
     "threshold_last_high": 120  // Different for last digit
   }
   ```

2. **Last digit considerations**:
   - Often has rotation indicator (different appearance)
   - May need higher threshold
   - Use `extended_last_digit` for better capture

3. **Lighting gradient**:
   - If lighting uneven across display
   - Adjust camera angle for uniform lighting
   - Consider multiple light sources

### Thresholded Digits Look Good But Recognition Fails

**Symptoms**: Binary digits are clear but confidence is low

**Solutions**:

1. **Check segment alignment**:
   - Digits might be cut off at segment boundaries
   - Adjust `shrink_last_3` setting
   - Verify segment count matches meter

2. **Digit size**:
   - Too small/large for model expectations
   - Model trained on ~28x28 pixels
   - Adjust camera distance

3. **Model mismatch**:
   - Model trained on different digit style
   - Consider custom model training
   - Export dataset and review

4. **Inverted digits**:
   - Check if thresholded digits are inverted (black on white)
   - System expects white on black
   - Algorithm auto-inverts but verify in evaluation

---

## Performance and Memory

### High CPU Usage

**Symptoms**: Server slow, high CPU load

**Solutions**:

1. **Reduce polling frequency**:
   - Increase `poll_interval_s` for HA Camera/HTTP sources
   - Reduce MQTT message frequency on ESP32
   - Stagger polling for multiple meters

2. **Optimize ROI extractor**:
   - YOLO is faster than ORB
   - Use Bypass for testing (no extraction)

3. **Limit evaluations**:
   ```json
   {
     "max_evals": 50  // Reduce from default 100
   }
   ```

4. **Disable unused features**:
   - Reduce `max_history`
   - Clear old evaluations

5. **Hardware upgrade**:
   - ONNX runtime is CPU-intensive
   - Multi-core helps (2-4 cores recommended)
   - Consider GPU inference (requires CUDA setup)

### High Memory Usage

**Symptoms**: Out of memory errors, crashes

**Solutions**:

1. **Reduce image resolution**:
   - ESP32: Use 640x480 instead of higher
   - HA Camera: Check camera resolution settings

2. **Limit stored evaluations**:
   ```json
   {
     "max_evals": 50,
     "max_history": 100
   }
   ```

3. **Clear old data**:
   - Delete old evaluations
   - Export and remove datasets
   - Vacuum database (see below)

4. **ONNX model memory**:
   - Models loaded once at startup (~300MB)
   - Shared across all meters
   - Cannot reduce without changing models

5. **Docker memory limits**:
   - Increase Docker container memory
   - Minimum: 512MB
   - Recommended: 1GB+

### Slow Image Processing

**Symptoms**: Long delay between image capture and result

**Solutions**:

1. **Profile bottleneck**:
   - Check logs for timing information
   - ROI extraction: 100-400ms
   - Digit recognition: 50-100ms
   - Thresholding: 10-50ms

2. **Database optimization** (see below)

3. **Network latency**:
   - MQTT broker on same network
   - HA API calls can be slow (10s timeout)
   - Use local broker, not cloud

4. **Multiple concurrent requests**:
   - Limit simultaneous processing
   - Stagger polling intervals
   - Queue requests (not yet implemented)

---

## Database Errors

### Database Locked

**Symptoms**: "Database is locked" errors

**Solutions**:

1. **Close other connections**:
   - Only one process should access DB
   - Stop duplicate MeterMonitor instances
   - Close database browser tools

2. **SQLite WAL mode** (not implemented):
   - Allows concurrent reads
   - Requires code change

3. **Increase timeout**:
   - Modify `sqlite3.connect(db_file, timeout=30)`
   - Higher timeout for busy systems

### Database Corruption

**Symptoms**: "Database disk image is malformed"

**Solutions**:

1. **Backup immediately**:
   ```bash
   cp watermeters.sqlite watermeters.sqlite.backup
   ```

2. **Attempt repair**:
   ```bash
   sqlite3 watermeters.sqlite "PRAGMA integrity_check;"
   sqlite3 watermeters.sqlite ".dump" | sqlite3 watermeters_fixed.sqlite
   ```

3. **Restore from backup** (if available)

4. **Start fresh** (loses all data):
   - Delete `watermeters.sqlite`
   - Restart MeterMonitor (recreates DB)
   - Reconfigure meters

### Migration Failures

**Symptoms**: Error during database migration on startup

**Solutions**:

1. **Check logs** for specific migration that failed

2. **Manual migration**:
   - Review `/db/migrations.py`
   - Apply failed migration manually via SQLite

3. **Backup and reset**:
   - Backup database
   - Delete problematic table
   - Let migration recreate

4. **Report issue** with logs and database schema

### Database Growing Too Large

**Symptoms**: Database file > 1GB

**Solutions**:

1. **Clear old evaluations**:
   - Per meter: DELETE via API
   - All: Manually via SQLite

2. **Reduce retention**:
   ```json
   {
     "max_evals": 50,
     "max_history": 100
   }
   ```

3. **Vacuum database**:
   ```bash
   sqlite3 watermeters.sqlite "VACUUM;"
   ```
   Reclaims deleted space

4. **Export and purge datasets**:
   - Export dataset via API
   - Delete dataset
   - Reduces stored base64 images

---

## Home Assistant Integration

### Sensors Not Auto-Discovered

**Symptoms**: No sensor entities in Home Assistant

**Solutions**:

1. **Check MQTT discovery** is enabled:
   - Mosquitto add-on configuration
   - `discovery: true` (default)
   - `discovery_prefix: homeassistant`

2. **Verify MQTT connection**:
   - MeterMonitor connected to same broker
   - Check alert in MeterMonitor UI

3. **Manual discovery trigger**:
   - Restart meter (re-sends discovery)
   - Finish setup again
   - Check MQTT topic:
     ```
     homeassistant/sensor/watermeter_{name}/config
     ```

4. **Manual YAML configuration** (fallback):
   ```yaml
   mqtt:
     sensor:
       - name: "Water Meter Reading"
         state_topic: "homeassistant/sensor/watermeter_kitchen/state"
         unit_of_measurement: "m³"
         device_class: "water"
         state_class: "total_increasing"
   ```

### HA Camera Capture Fails

**Symptoms**: "Capture failed" error for HA Camera source

**Solutions**:

1. **Check HA API connection**:
   - Navigate to `/api/ha/status`
   - Verify `ok: true`
   - Check `base_url` and token configuration

2. **Token issues**:
   - **Add-on**: Use `use_supervisor_token: true` (default)
   - **Standalone**: Generate long-lived access token
   - Token must have `homeassistant_api` permission

3. **Camera entity problems**:
   - Verify entity ID is correct
   - Check camera is accessible in HA UI
   - Some cameras require stream setup

4. **Timeout**:
   - Increase `request_timeout_s` (default: 10s)
   - Some cameras are slow to respond

5. **Network connectivity**:
   - Addon: Use `http://supervisor/core`
   - Standalone: Use `http://homeassistant.local:8123`
   - Firewall may block requests

### Flash Control Not Working

**Symptoms**: Flash entity doesn't turn on during capture

**Solutions**:

1. **Verify entity ID**:
   - Check light entity exists in HA
   - Test manual control in HA UI

2. **Check flash delay**:
   - May be too short (light not fully on)
   - Increase `flash_delay_ms` (try 2000ms)

3. **Entity state**:
   - Must be controllable (not unavailable)
   - ESPHome device must be online

4. **Service call permissions**:
   - Check token has permission to call `light.turn_on`

---

## Source and Polling Issues

### Source Shows Errors

**Symptoms**: `last_error` field populated in source list

**Solutions**:

1. **Review specific error message**

2. **Common errors**:
   - **"Connection timeout"**: Network issue, check connectivity
   - **"401 Unauthorized"**: HA token invalid
   - **"Entity not found"**: Camera entity ID wrong
   - **"Invalid image data"**: HTTP endpoint not returning image

3. **Test source**:
   - Use **Trigger Capture** button
   - Check logs for detailed error

4. **Disable problematic source**:
   - Prevents repeated failures
   - Fix issue then re-enable

### Polling Not Triggering

**Symptoms**: Source enabled but not capturing

**Solutions**:

1. **Check `poll_interval_s`**:
   - Must be > 0 for polling
   - Verify in source configuration

2. **Check `last_success_ts`**:
   - Shows last successful capture time
   - Next poll: `last_success_ts + poll_interval_s`

3. **Polling service running**:
   - Check logs for "POLLING" entries
   - Restarts required after config change

4. **Source type**:
   - Only `ha_camera` and `http` support polling
   - `mqtt` is push-only (no polling)

### Multiple Sources for Same Meter

**Symptoms**: Conflicting sources updating same meter

**Solutions**:

1. **Disable redundant sources**:
   - Each meter should have one active source
   - Multiple sources will overwrite each other

2. **Use cases for multiple sources**:
   - One disabled (backup configuration)
   - Testing different capture methods

3. **MQTT vs HA Camera**:
   - MQTT: Real-time push from ESP32
   - HA Camera: Polling Home Assistant entity
   - Choose one per meter

---

## Reporting Issues

### Collecting Logs

**Home Assistant Add-on**:
1. Configuration → Add-ons → MeterMonitor → Log
2. Copy full log output
3. Include in issue report

**Standalone**:
1. Terminal output shows logs
2. Redirect to file: `python run.py > metermonitor.log 2>&1`
3. Include relevant portions

**Log Levels**:
- `[INIT]`: Startup configuration
- `[MQTT]`: MQTT connection/messages
- `[POLLING]`: Polling service activity
- `[HTTP]`: API requests
- `[CorrectionAlg]`: Correction decisions
- `[ERROR]`: Errors and exceptions

### Information to Include

When reporting issues, provide:

1. **MeterMonitor version**:
   - Check `/api/config` endpoint
   - Or add-on info page

2. **Environment**:
   - Home Assistant add-on vs standalone vs Docker
   - Platform: amd64, aarch64, etc.
   - RAM available

3. **Configuration**:
   - Relevant settings (redact passwords)
   - Meter settings (thresholds, ROI extractor)
   - Source configuration

4. **Symptoms**:
   - What you expected
   - What actually happened
   - Steps to reproduce

5. **Logs**:
   - Full log output (last 100 lines minimum)
   - Include timestamps

6. **Screenshots**:
   - Meter image
   - Evaluation details
   - Error messages

7. **Database state** (if relevant):
   - Number of meters
   - Number of evaluations
   - Database file size

### Creating GitHub Issue

1. Visit: https://github.com/phiph-s/metermonitor-managementserver/issues
2. Click **New Issue**
3. Use descriptive title
4. Fill template with information above
5. Add relevant labels

### Community Support

- GitHub Discussions
- Home Assistant Community Forum
- Discord (if available)

---

## Additional Resources

- [User Guide](user-guide.md)
- [ESP32 Setup Guide](esp32-setup.md)
- [ROI Extractor Guide](roi-extractors.md)
- [API Reference](api-reference.md)
- [Development Guide](development.md)
- [Advanced Topics](advanced-topics.md)

### External Resources

- [ESPHome Documentation](https://esphome.io/)
- [Home Assistant MQTT Integration](https://www.home-assistant.io/integrations/mqtt/)
- [SQLite Documentation](https://www.sqlite.org/docs.html)

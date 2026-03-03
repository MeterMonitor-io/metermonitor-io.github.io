# Troubleshooting

---

## MQTT problems

### "Failed to connect to MQTT broker" alert

1. Verify the broker is running (for the Mosquitto add-on, check its status in HA)
2. Check `mqtt.broker`, `mqtt.port`, `mqtt.username`, `mqtt.password` in your configuration
3. Default HA Mosquitto address: `172.30.32.1`, port `1883`
4. Test reachability: `telnet <broker_ip> 1883`

### Meter not appearing in Discovery

1. Verify the ESP32 (or other device) publishes to a topic matching your `mqtt.topic` pattern (default `MeterMonitor/#`)
2. Topics are case-sensitive
3. Check the **MQTT Setup Helper** in the Discovery tab for the exact payload format
4. Subscribe to the topic manually to confirm messages arrive: `mosquitto_sub -h <broker> -t "MeterMonitor/#" -v`

### MQTT source disabled / no new images

- Navigate to **Sources** and check the `last_error` field
- Re-enable the source if it was auto-disabled after repeated failures

---

## Image quality and detection

### "No display detected in image"

1. Open the meter detail and view the latest captured image
2. Ensure the entire digit display is visible and not obscured
3. Check focus (blurry images fail detection)
4. Improve lighting — very dark or very bright images both cause failures
5. If YOLO keeps failing, switch to the **ORB** or **Static Rectangle** extractor

### Blurry images

- Adjust the ESP32-CAM lens (rotate carefully — counterclockwise moves focus farther)
- Ensure the mount is rigid and doesn't vibrate
- Check `max_framerate` is low (`1 fps`) to avoid motion blur

### Dark images

- Enable the flash LED via a HA Camera source (set `flash_delay_ms` ≥ 1000)
- Increase `brightness` in the ESPHome camera configuration
- Add ambient lighting

### Glare or reflections

- Reposition the camera — even a few centimetres can eliminate a reflection
- Replace a direct flash with indirect lighting
- Diffuse the LED with tape or a diffuser

---

## Recognition accuracy

### Low confidence scores

1. Run **Threshold Search** (Setup view → Search Thresholds, use 15–20 steps)
2. Check that the digit segment count matches your meter
3. Review the thresholded digit images in evaluation details — the digit shape should appear white on black
4. Increase `conf_threshold` in per-meter settings to automatically reject low-quality readings

### Wrong digits read

1. Check the thresholded images — if they look garbled, adjust thresholds
2. Make sure segment count and decimal position are correct
3. Verify the per-digit model (rotating wheel vs 7-segment) matches your meter's digit type
4. Enable `extended_last_digit` if the last digit is being cut off at the edge

### Readings stuck / not updating

1. Check `max_flow_rate` — if set too low, even valid readings are rejected
2. Review the `rejection_reason` in evaluation metadata
3. Add a manual history entry to give the correction algorithm a fresh reference point
4. Try switching to **Light correction mode** temporarily to see if readings come through

### Erratic / jumping values

1. Enable **Full correction mode** with a realistic `max_flow_rate`
2. Improve mounting stability
3. Run threshold search — inconsistent thresholds cause flickering

---

## Home Assistant integration

### Sensor not auto-discovered

1. Ensure MQTT discovery is enabled in Mosquitto (`discovery: true`)
2. Finish the meter setup (auto-discovery only happens after setup is complete)
3. Check MeterMonitor is connected to the same broker as HA
4. As a fallback, add the sensor manually — see the [User Guide](user-guide.md#home-assistant-integration)

### HA Camera capture fails

1. Navigate to `/api/ha/status` in MeterMonitor to verify the HA API connection
2. For add-on installs: `use_supervisor_token` should be `true` (default)
3. For standalone installs: generate a long-lived access token in HA Profile → Security
4. Verify the camera entity ID exists and is accessible
5. Increase `request_timeout_s` if the camera is slow to respond

### Flash not working

1. Confirm the light entity ID is correct and the device is online
2. Increase `flash_delay_ms` to 2000 ms — the LED may need more time to reach full brightness
3. Test the light manually in HA to confirm it works

---

## Performance

### High CPU / slow processing

- Increase poll intervals — MeterMonitor doesn't need more than one image per minute for most setups
- YOLO is faster than ORB; use YOLO when possible
- Reduce `max_evals` to lower database I/O

### Database growing large

Reduce `max_evals` and `max_history` in configuration. Export datasets before clearing evaluations.

---

## Collecting logs for a bug report

**Home Assistant Add-on:** Settings → Add-ons → MeterMonitor → **Log** tab

**Docker:** `docker logs -f metermonitor`

**Manual install:** Terminal output, or redirect: `python run.py > metermonitor.log 2>&1`

Include in your bug report:
- MeterMonitor version (shown in the web UI header)
- Deployment type (add-on / Docker / manual)
- Relevant log lines (include timestamps)
- Meter settings (thresholds, ROI extractor) with passwords redacted
- Screenshot of the failing evaluation (if applicable)

Open issues at: https://github.com/MeterMonitor-io/metermonitor-ha/issues

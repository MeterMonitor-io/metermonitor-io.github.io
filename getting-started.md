# Getting Started

The fastest way to get MeterMonitor running is as a **Home Assistant Add-on**.

## Step 1 — Add the repository

1. In Home Assistant, go to **Settings → Add-ons → Add-on Store**
2. Click ⋮ → **Repositories**
3. Paste: `https://github.com/MeterMonitor-io/metermonitor-ha`
4. Click **Add**, then reload the page

> **Dev / preview builds:** use the `develop` branch URL to get pre-release versions.

## Step 2 — Install and configure

1. Find **MeterMonitor** in the store and click **Install**
2. Go to the **Configuration** tab and set your MQTT credentials:

```yaml
mqtt:
  broker: 172.30.32.1       # Mosquitto add-on default
  port: 1883
  topic: MeterMonitor/#
  username: your_mqtt_user
  password: your_mqtt_password
```

3. Click **Save**

## Step 3 — Start

1. Click **Start**
2. Enable **Start on boot** and **Watchdog**
3. Click **Open Web UI** once started

## Step 4 — Set up your first meter

See the [User Guide](user-guide.md) for the full setup walkthrough (discovery → thresholds → initial value).

---

## Alternative: Docker

```bash
mkdir ~/metermonitor-data
# Create ~/metermonitor-data/options.json (see Configuration Reference)

docker run -d \
  --name metermonitor \
  --restart unless-stopped \
  -p 8070:8070 \
  -v ~/metermonitor-data:/data \
  -e TZ=Europe/Berlin \
  ghcr.io/metermonitor-io/metermonitor:latest
```

Open `http://localhost:8070`

For a full Docker or manual Python setup, see [Installation](installation.md).

---

## Common first-time issues

**MQTT connection fails**
- Verify the Mosquitto add-on is running
- Double-check username/password
- Default HA Mosquitto broker IP is `172.30.32.1`

**No images received / meter not appearing in Discovery**
- Verify the ESP32 publishes to a topic matching `MeterMonitor/#`
- Check the MQTT Setup Helper in the Discovery tab — it shows the exact expected payload format and your broker address

**Detection not working after setup**
- Try the automatic **Threshold Search** in the Setup view
- Make sure the entire digit display is visible and the image is in focus

# Getting Started

This guide will help you get MeterMonitor up and running quickly.

## Overview

MeterMonitor processes images from ESP32 cameras to read analog water meter values using AI. The system consists of:

1. **ESP32-CAM device** - Captures meter images
2. **MQTT broker** - Transmits images
3. **MeterMonitor server** - Processes images and extracts readings
4. **Web interface** - Configuration and monitoring

## Prerequisites

Before you begin, ensure you have:

- Home Assistant installation (for add-on deployment) **OR** Docker/Python environment (for standalone)
- MQTT broker (Mosquitto recommended)
- ESP32-CAM device with ESPHome firmware **OR** existing Home Assistant camera entity
- Basic knowledge of MQTT and Home Assistant (if using add-on)

## Quick Start: Home Assistant Add-on

### Step 1: Install the Add-on

1. Open Home Assistant
2. Navigate to **Settings** → **Add-ons** → **Add-on Store**
3. Click the menu (⋮) and select **Repositories**
4. Add this repository URL: `https://github.com/phiph-s/metermonitor-managementserver`
5. Find **MeterMonitor** in the add-on list and click **Install**

### Step 2: Configure MQTT

1. Ensure the **Mosquitto broker** add-on is installed and running
2. In the MeterMonitor add-on configuration, set:
   ```yaml
   mqtt:
     broker: 172.30.32.1
     port: 1883
     topic: MeterMonitor/#
     username: your_mqtt_user
     password: your_mqtt_password
   ```

### Step 3: Start the Add-on

1. Click **Start**
2. Enable **Start on boot** and **Watchdog**
3. Click **Open Web UI** to access the interface

### Step 4: Set Up Your First Meter

See the [User Guide](user-guide.md#initial-meter-setup) for detailed setup instructions.

## Quick Start: Standalone Docker

### Step 1: Clone and Build

```bash
git clone https://github.com/phiph-s/metermonitor-managementserver.git
cd metermonitor-managementserver
docker build -t metermonitor .
```

### Step 2: Create Configuration

Create a directory for persistent data:

```bash
mkdir ~/metermonitor-data
```

Create `~/metermonitor-data/options.json`:

```json
{
  "max_history": 200,
  "max_evals": 100,
  "mqtt": {
    "broker": "your.mqtt.broker",
    "port": 1883,
    "topic": "MeterMonitor/#",
    "username": "your_username",
    "password": "your_password"
  },
  "allow_negative_correction": true,
  "publish_to": "homeassistant/sensor/watermeter_{device}/",
  "dbfile": "/data/metermonitor.db",
  "http": {
    "enabled": true,
    "host": "0.0.0.0",
    "port": 8070
  },
  "secret_key": "change_me_to_something_secure",
  "enable_auth": false
}
```

### Step 3: Run Container

```bash
docker run -d \
  --name metermonitor \
  -p 8070:8070 \
  -v ~/metermonitor-data:/data \
  metermonitor
```

### Step 4: Access Web Interface

Open your browser to `http://localhost:8070`

## Quick Start: Manual Python Installation

### Step 1: Clone and Install Dependencies

```bash
git clone https://github.com/phiph-s/metermonitor-managementserver.git
cd metermonitor-managementserver
pip install -r requirements.txt
```

### Step 2: Build Frontend

```bash
cd frontend
yarn install
yarn build
cd ..
```

### Step 3: Configure

Edit `settings.json` with your MQTT credentials and preferences.

### Step 4: Run

```bash
python3 run.py
```

Access the web interface at `http://localhost:8070`

## Next Steps

After completing the quick start:

1. **Configure your ESP32-CAM** - See [ESP32 Setup Guide](esp32-setup.md)
2. **Set up your first meter** - Follow the [User Guide](user-guide.md)
3. **Configure thresholds** - Use the automatic threshold search feature
4. **Set up Home Assistant sensors** - MQTT auto-discovery handles this automatically

## Common First-Time Issues

### MQTT Connection Failed

- Verify MQTT broker is running and accessible
- Check username/password are correct
- Ensure topic pattern is correct (default: `MeterMonitor/#`)
- Check firewall rules allow MQTT port (default: 1883)

### No Images Received

- Verify ESP32 is connected to WiFi
- Check ESP32 MQTT configuration matches server topic
- Confirm MQTT credentials on ESP32 match broker settings
- Check ESP32 logs for errors

### Detection Not Working

- Ensure image quality is sufficient (good lighting, clear view)
- Try automatic threshold search in the web interface
- Verify meter is properly framed in the camera view
- Consider using template-based ROI extraction for difficult angles

## Where to Go Next

- [Full Installation Guide](installation.md) - Detailed installation for all platforms
- [Configuration Reference](configuration.md) - Complete configuration options
- [User Guide](user-guide.md) - Comprehensive usage instructions
- [ESP32 Setup](esp32-setup.md) - ESP32-CAM hardware and firmware setup

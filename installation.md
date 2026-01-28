# Installation Guide

This comprehensive guide covers all installation methods for MeterMonitor.

## Table of Contents

- [Home Assistant Add-on Installation](#home-assistant-add-on-installation)
- [Docker Installation](#docker-installation)
- [Manual Python Installation](#manual-python-installation)
- [Configuration](#configuration)
- [Post-Installation](#post-installation)

## Home Assistant Add-on Installation

### Prerequisites

- Home Assistant OS or Home Assistant Supervised
- MQTT broker (Mosquitto add-on recommended)
- Architecture: `amd64` or `aarch64` (ARM64)

### Step 1: Add Repository

1. Navigate to **Settings** → **Add-ons** → **Add-on Store**
2. Click the **menu button** (⋮) in the top right
3. Select **Repositories**
4. Add the repository URL:
   ```
   https://github.com/phiph-s/metermonitor-managementserver
   ```
5. Click **Add** and wait for the repository to load

### Step 2: Install Add-on

1. Refresh the add-on store page
2. Search for **MeterMonitor**
3. Click on the add-on card
4. Click **Install**
5. Wait for the installation to complete (may take 5-10 minutes)

### Step 3: Configure

Before starting, configure the add-on:

#### Minimal Configuration

```yaml
mqtt:
  broker: 172.30.32.1
  port: 1883
  topic: MeterMonitor/#
  username: your_mqtt_username
  password: your_mqtt_password
```

#### Advanced Configuration

```yaml
max_history: 200
max_evals: 100
mqtt:
  broker: 172.30.32.1
  port: 1883
  topic: MeterMonitor/#
  username: your_mqtt_username
  password: your_mqtt_password
allow_negative_correction: true
publish_to: homeassistant/sensor/watermeter_{device}/
dbfile: /data/metermonitor.db
homeassistant:
  use_supervisor_token: true
  url: http://supervisor/core
  token: ""
  request_timeout_s: 10
```

### Step 4: Start the Add-on

1. Click **Start**
2. Monitor the **Log** tab for startup messages
3. Once you see "Setup complete", the add-on is ready
4. Enable **Start on boot** for automatic startup
5. Enable **Watchdog** for automatic restart on crashes

### Step 5: Access Web Interface

Click **Open Web UI** or navigate to:
```
http://homeassistant.local:8070
```

Or use the Ingress interface (recommended):
```
Settings → Add-ons → MeterMonitor → OPEN WEB UI
```

## Docker Installation

### Prerequisites

- Docker Engine 20.10+
- Docker Compose (optional)
- MQTT broker access

### Method 1: Docker CLI

#### Step 1: Create Data Directory

```bash
mkdir -p ~/metermonitor-data
cd ~/metermonitor-data
```

#### Step 2: Create Configuration File

Create `options.json`:

```json
{
  "max_history": 200,
  "max_evals": 100,
  "mqtt": {
    "broker": "mqtt.example.com",
    "port": 1883,
    "topic": "MeterMonitor/#",
    "username": "mqtt_user",
    "password": "mqtt_password"
  },
  "allow_negative_correction": true,
  "publish_to": "homeassistant/sensor/watermeter_{device}/",
  "dbfile": "/data/metermonitor.db",
  "http": {
    "enabled": true,
    "host": "0.0.0.0",
    "port": 8070
  },
  "ingress": false,
  "secret_key": "change_this_to_a_random_string",
  "enable_auth": false,
  "output_dataset": "/data/output_dataset",
  "homeassistant": {
    "use_supervisor_token": false,
    "url": "",
    "token": "",
    "request_timeout_s": 10
  }
}
```

#### Step 3: Build Image

```bash
git clone https://github.com/phiph-s/metermonitor-managementserver.git
cd metermonitor-managementserver
docker build -t metermonitor:latest .
```

#### Step 4: Run Container

```bash
docker run -d \
  --name metermonitor \
  --restart unless-stopped \
  -p 8070:8070 \
  -v ~/metermonitor-data:/data \
  -e TZ=Europe/Berlin \
  metermonitor:latest
```

#### Step 5: Verify

Check logs:
```bash
docker logs -f metermonitor
```

Access interface:
```
http://localhost:8070
```

### Method 2: Docker Compose

Create `docker-compose.yml`:

```yaml
version: '3.8'

services:
  metermonitor:
    build: .
    container_name: metermonitor
    restart: unless-stopped
    ports:
      - "8070:8070"
    volumes:
      - ./data:/data
    environment:
      - TZ=Europe/Berlin
      - PYTHONUNBUFFERED=1
    depends_on:
      - mosquitto

  mosquitto:
    image: eclipse-mosquitto:2
    container_name: mosquitto
    restart: unless-stopped
    ports:
      - "1883:1883"
      - "9001:9001"
    volumes:
      - ./mosquitto/config:/mosquitto/config
      - ./mosquitto/data:/mosquitto/data
      - ./mosquitto/log:/mosquitto/log
```

Start services:
```bash
docker-compose up -d
```

## Manual Python Installation

### Prerequisites

- Python 3.12 or higher
- pip package manager
- Node.js 18+ and yarn (for frontend)
- Git

### Step 1: Install System Dependencies

#### Ubuntu/Debian
```bash
sudo apt update
sudo apt install -y python3.12 python3-pip python3-venv nodejs yarn git
```

#### Fedora/RHEL
```bash
sudo dnf install -y python3.12 python3-pip nodejs yarn git
```

#### macOS (with Homebrew)
```bash
brew install python@3.12 node yarn git
```

#### Windows
1. Install [Python 3.12](https://www.python.org/downloads/)
2. Install [Node.js](https://nodejs.org/)
3. Install [Git](https://git-scm.com/)
4. Install yarn: `npm install -g yarn`

### Step 2: Clone Repository

```bash
git clone https://github.com/phiph-s/metermonitor-managementserver.git
cd metermonitor-managementserver
```

### Step 3: Create Virtual Environment

```bash
python3.12 -m venv .venv
source .venv/bin/activate  # On Windows: .venv\Scripts\activate
```

### Step 4: Install Python Dependencies

```bash
pip install --upgrade pip
pip install -r requirements.txt
```

### Step 5: Build Frontend

```bash
cd frontend
yarn install
yarn build
cd ..
```

### Step 6: Create Configuration

Copy and edit `settings.json`:

```bash
cp settings.json.example settings.json
# Edit settings.json with your preferred editor
```

Example `settings.json`:

```json
{
  "max_history": 200,
  "max_evals": 100,
  "mqtt": {
    "broker": "localhost",
    "port": 1883,
    "topic": "MeterMonitor/#",
    "username": "mqtt_user",
    "password": "mqtt_pass"
  },
  "allow_negative_correction": true,
  "publish_to": "homeassistant/sensor/watermeter_{device}/",
  "dbfile": "data/metermonitor.db",
  "http": {
    "enabled": true,
    "host": "0.0.0.0",
    "port": 8070
  },
  "ingress": false,
  "secret_key": "your-secret-key-here",
  "enable_auth": false
}
```

### Step 7: Run Application

```bash
python run.py
```

Or with auto-reload for development:

```bash
uvicorn run:app --reload --host 0.0.0.0 --port 8070
```

### Step 8: Install as System Service (Optional)

#### systemd (Linux)

Create `/etc/systemd/system/metermonitor.service`:

```ini
[Unit]
Description=MeterMonitor AI Water Meter Reader
After=network.target

[Service]
Type=simple
User=YOUR_USER
WorkingDirectory=/path/to/metermonitor-managementserver
Environment="PATH=/path/to/metermonitor-managementserver/.venv/bin"
ExecStart=/path/to/metermonitor-managementserver/.venv/bin/python run.py
Restart=on-failure
RestartSec=10

[Install]
WantedBy=multi-user.target
```

Enable and start:

```bash
sudo systemctl daemon-reload
sudo systemctl enable metermonitor
sudo systemctl start metermonitor
```

Check status:

```bash
sudo systemctl status metermonitor
```

View logs:

```bash
sudo journalctl -u metermonitor -f
```

## Configuration

### Configuration File Structure

MeterMonitor uses different configuration files depending on the deployment method:

- **Home Assistant Add-on**: Configuration UI → `options.json`
- **Docker**: `~/metermonitor-data/options.json`
- **Manual**: `settings.json`

### Configuration Options Reference

See [Configuration Guide](configuration.md) for detailed option descriptions.

## Post-Installation

### 1. Verify Installation

Check that the application is running:

```bash
# Docker
docker logs metermonitor

# Manual
# Check terminal output or systemd logs
```

Expected output:
```
[INIT] Running as Home Assistant addon, using options.json
[INIT] Loaded config: ...
[MIGRATION] Database migrations complete
[MQTT] Connecting to MQTT broker
[MQTT] Successfully connected to MQTT broker
[HTTP] Setup complete
[INIT] Started setup server on http://0.0.0.0:8070
```

### 2. Access Web Interface

Open your browser:
- **Local**: `http://localhost:8070`
- **Network**: `http://your-server-ip:8070`
- **HA Add-on**: Use Ingress or port 8070

### 3. Configure MQTT Broker

Ensure your MQTT broker is running and accessible:

```bash
# Test with mosquitto_sub
mosquitto_sub -h localhost -p 1883 -u username -P password -t 'MeterMonitor/#' -v
```

### 4. Set Up First Meter

See [User Guide - Initial Setup](user-guide.md#initial-meter-setup)

## Troubleshooting Installation

### Add-on Won't Start

**Check Logs:**
```
Settings → Add-ons → MeterMonitor → Log tab
```

**Common Issues:**
- MQTT broker not running → Start Mosquitto add-on
- Invalid MQTT credentials → Verify username/password
- Port conflict → Check if port 8070 is in use

### Docker Container Exits

**Check Logs:**
```bash
docker logs metermonitor
```

**Common Issues:**
- Configuration file not found → Ensure `options.json` exists in mounted volume
- Invalid JSON → Validate `options.json` syntax
- Permission issues → Check volume mount permissions

### Python Installation Errors

**Missing Dependencies:**
```bash
# Ubuntu/Debian
sudo apt install -y build-essential python3-dev

# Fedora
sudo dnf install -y gcc python3-devel
```

**OpenCV Issues:**
```bash
pip install opencv-python-headless==4.11.0.86
```

**ONNX Runtime Issues:**
```bash
pip install onnxruntime==1.19.2 --no-cache-dir
```

### Frontend Not Loading

**Check frontend build:**
```bash
cd frontend
yarn build
ls -la dist/  # Should contain index.html and assets/
```

**Clear browser cache:**
- Hard refresh: Ctrl+Shift+R (Windows/Linux) or Cmd+Shift+R (macOS)

## Updating

### Home Assistant Add-on

1. Navigate to **Settings** → **Add-ons** → **MeterMonitor**
2. Click **Update** if available
3. Restart the add-on

### Docker

```bash
cd metermonitor-managementserver
git pull
docker build -t metermonitor:latest .
docker stop metermonitor
docker rm metermonitor
# Run docker run command again (see above)
```

### Manual Installation

```bash
cd metermonitor-managementserver
source .venv/bin/activate
git pull
pip install -r requirements.txt --upgrade
cd frontend
yarn install
yarn build
cd ..
# Restart application
```

## Uninstallation

### Home Assistant Add-on

1. Stop the add-on
2. Click **Uninstall**
3. Optionally remove repository from add-on store

### Docker

```bash
docker stop metermonitor
docker rm metermonitor
docker rmi metermonitor:latest
rm -rf ~/metermonitor-data  # Warning: Deletes all data!
```

### Manual Installation

```bash
# Stop systemd service if installed
sudo systemctl stop metermonitor
sudo systemctl disable metermonitor
sudo rm /etc/systemd/system/metermonitor.service

# Remove application
rm -rf ~/metermonitor-managementserver
```

## Next Steps

- [Configure your ESP32-CAM device](esp32-setup.md)
- [Follow the User Guide](user-guide.md)
- [Explore Configuration Options](configuration.md)
- [Set up Home Assistant integration](user-guide.md#home-assistant-integration)

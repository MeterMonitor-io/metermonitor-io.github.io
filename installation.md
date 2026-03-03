# Installation

MeterMonitor can run as a Home Assistant Add-on, as a Docker container, or as a standalone Python application.

---

## Home Assistant Add-on

### Prerequisites

- Home Assistant OS or Supervised
- Mosquitto broker add-on installed and running
- Architecture: `amd64` or `aarch64` (ARM64)

### Install

1. **Settings → Add-ons → Add-on Store** → ⋮ → **Repositories**
2. Add `https://github.com/MeterMonitor-io/metermonitor-ha`, click **Add**
3. Find **MeterMonitor**, click **Install** (may take a few minutes)
4. Open the **Configuration** tab and set at minimum:

```yaml
mqtt:
  broker: 172.30.32.1
  port: 1883
  topic: MeterMonitor/#
  username: your_mqtt_user
  password: your_mqtt_password
```

5. Click **Start** → enable **Start on boot** + **Watchdog** → **Open Web UI**

For all available configuration options see [Configuration Reference](configuration.md).

### Updating

Go to **Settings → Add-ons → MeterMonitor** and click **Update** when one is available.

---

## Docker

### Create a data directory and config file

```bash
mkdir -p ~/metermonitor-data
```

Create `~/metermonitor-data/options.json`:

```json
{
  "mqtt": {
    "broker": "your.mqtt.broker",
    "port": 1883,
    "topic": "MeterMonitor/#",
    "username": "mqtt_user",
    "password": "mqtt_password"
  },
  "http": {
    "enabled": true,
    "host": "0.0.0.0",
    "port": 8070
  },
  "ingress": false,
  "max_history": 200,
  "max_evals": 100
}
```

### Run

```bash
docker run -d \
  --name metermonitor \
  --restart unless-stopped \
  -p 8070:8070 \
  -v ~/metermonitor-data:/data \
  -e TZ=Europe/Berlin \
  ghcr.io/metermonitor-io/metermonitor:latest
```

Open `http://localhost:8070`. Check logs with `docker logs -f metermonitor`.

### Docker Compose

```yaml
version: '3.8'
services:
  metermonitor:
    image: ghcr.io/metermonitor-io/metermonitor:latest
    container_name: metermonitor
    restart: unless-stopped
    ports:
      - "8070:8070"
    volumes:
      - ./data:/data
    environment:
      - TZ=Europe/Berlin
```

---

## Manual Python installation

### Prerequisites

- Python 3.12+
- Node.js 18+ and yarn (to build the frontend)
- An MQTT broker

### Steps

```bash
git clone https://github.com/MeterMonitor-io/metermonitor-ha.git
cd metermonitor-ha

python3.12 -m venv .venv
source .venv/bin/activate

pip install -r requirements.txt

cd frontend
yarn install
yarn build
cd ..
```

Copy `settings.json.example` to `settings.json` and fill in your MQTT details, then:

```bash
python run.py
```

Open `http://localhost:8070`.

### Run as a systemd service (Linux)

Create `/etc/systemd/system/metermonitor.service`:

```ini
[Unit]
Description=MeterMonitor
After=network.target

[Service]
Type=simple
User=YOUR_USER
WorkingDirectory=/path/to/metermonitor-ha
Environment="PATH=/path/to/metermonitor-ha/.venv/bin"
ExecStart=/path/to/metermonitor-ha/.venv/bin/python run.py
Restart=on-failure

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now metermonitor
```

---

## Troubleshooting installation

| Symptom | Likely cause | Fix |
|---|---|---|
| Add-on won't start | MQTT broker not running | Start the Mosquitto add-on |
| Docker container exits immediately | Bad `options.json` | Run `python -m json.tool options.json` to validate |
| Frontend not loading | Build not present | Run `cd frontend && yarn build` |
| Port already in use | Another process on 8070 | Change `http.port` in config |

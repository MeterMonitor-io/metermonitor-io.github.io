# Configuration Reference

Configuration is stored as JSON. The location depends on how you run MeterMonitor:

| Deployment | Config location |
|---|---|
| Home Assistant Add-on | Configuration tab in the add-on UI |
| Docker | `/data/options.json` (mounted volume) |
| Manual Python | `settings.json` in project root |

---

## MQTT

```json
"mqtt": {
  "broker": "172.30.32.1",
  "port": 1883,
  "topic": "MeterMonitor/#",
  "username": "your_user",
  "password": "your_password"
}
```

| Key | Default | Description |
|---|---|---|
| `broker` | `172.30.32.1` | Hostname or IP of your MQTT broker. The default works for the Mosquitto add-on inside Home Assistant. |
| `port` | `1883` | MQTT port. Use `8883` for TLS. |
| `topic` | `MeterMonitor/#` | Subscription pattern. Must end in a wildcard to receive from all devices. |
| `username` | — | Broker authentication username. |
| `password` | — | Broker authentication password. |

---

## HTTP server

```json
"http": {
  "enabled": true,
  "host": "0.0.0.0",
  "port": 8070
}
```

| Key | Default | Description |
|---|---|---|
| `enabled` | `true` | Set to `false` to disable the web interface (MQTT-only mode). |
| `host` | `0.0.0.0` | Bind address. Use `127.0.0.1` to restrict to localhost. |
| `port` | `8070` | Web interface port. Must match Docker port mapping. |

---

## Home Assistant integration

```json
"homeassistant": {
  "use_supervisor_token": true,
  "url": "http://supervisor/core",
  "token": "",
  "request_timeout_s": 10
}
```

| Key | Default | Description |
|---|---|---|
| `use_supervisor_token` | `true` | When running as an HA add-on, leave this `true` — credentials are injected automatically. |
| `url` | `http://supervisor/core` | HA API base URL. Only relevant if using HA Camera sources or MQTT auto-discovery. For external installs use e.g. `http://homeassistant.local:8123`. |
| `token` | `""` | Long-lived access token. Only needed when `use_supervisor_token` is `false`. |
| `request_timeout_s` | `10` | Seconds to wait for HA API responses. Increase if you see timeout errors. |

---

## Data retention

| Key | Default | Description |
|---|---|---|
| `max_history` | `200` | Maximum history entries kept per meter. Oldest are pruned automatically. |
| `max_evals` | `100` | Maximum evaluation records kept per meter. Evaluations include digit images and are larger than history entries. |

---

## MQTT publishing

| Key | Default | Description |
|---|---|---|
| `publish_to` | `homeassistant/sensor/watermeter_{device}/` | MQTT topic template for outgoing readings. `{device}` is replaced by the meter name. This path also controls Home Assistant MQTT auto-discovery. |

---

## Error correction

| Key | Default | Description |
|---|---|---|
| `allow_negative_correction` | `true` | Allow the correction algorithm to adjust readings that appear to go backwards (caused by misread digits). Set to `false` for strict monotonic enforcement. |

---

## Authentication (optional)

| Key | Default | Description |
|---|---|---|
| `enable_auth` | `false` | Require a `Secret` header on all API requests. |
| `secret_key` | `"change_me"` | The shared secret value. Change this if enabling auth on a public-facing instance. |

---

## Per-meter settings

These settings are configured per meter during the **Setup wizard** — the multi-step flow that opens when you click a newly discovered meter in the Discovery tab. They are not part of the main config file.

After initial setup you can revisit and adjust them by clicking the meter in the **Watermeters** tab and going to **Setup**.

| Setting | Configured in setup step | Description |
|---|---|---|
| `roi_extractor` | Step 1 – Digit Extraction | Detection method: `yolo` (automatic), `orb` (template), `static_rect` (fixed crop), or `bypass` |
| `segments` | Step 1 – Digit Extraction | Number of digits on the meter display |
| `rotated_180` | Step 1 – Digit Extraction | Rotate the image 180° (for upside-down cameras) |
| `extended_last_digit` | Step 1 – Digit Extraction | Extend the capture area of the last digit |
| `shrink_last_3` | Step 1 – Digit Extraction | Reduce width of the rightmost 3 digits (for compact meter layouts) |
| `decimals` | Step 2 – Digit Isolation | How many of the rightmost digits are after the decimal point |
| `threshold_low` / `threshold_high` | Step 2 – Digit Isolation | Grayscale range for isolating digits (0–255) |
| `threshold_last_low` / `threshold_last_high` | Step 2 – Digit Isolation | Separate thresholds for the decimal-part digits |
| `max_flow_rate` | Step 3 – Evaluation Settings | Maximum realistic flow in m³/h — readings exceeding this are rejected |
| `conf_threshold` | Step 3 – Evaluation Settings | Minimum confidence (0.0–1.0) to accept a reading; `null` disables the check |
| `use_correctional_alg` | Step 3 – Evaluation Settings | `true` for full error correction, `false` for light mode |

---

## Minimal working configuration

```json
{
  "mqtt": {
    "broker": "172.30.32.1",
    "port": 1883,
    "topic": "MeterMonitor/#",
    "username": "mqtt_user",
    "password": "mqtt_pass"
  }
}
```

All other values use their defaults.

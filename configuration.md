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

These are configured per meter through the web interface (meter detail → **Edit Settings**), not in the main config file.

| Setting | Description |
|---|---|
| `roi_extractor` | Detection method: `yolo` (automatic), `orb` (template), `static_rect` (fixed crop), or `bypass` |
| `segments` | Number of digits on the meter display |
| `decimals` | How many of the rightmost digits are after the decimal point |
| `threshold_low` / `threshold_high` | Grayscale range for isolating digits (0–255) |
| `threshold_last_low` / `threshold_last_high` | Separate thresholds for the decimal-part digits |
| `max_flow_rate` | Maximum realistic flow in m³/h — readings exceeding this are rejected |
| `conf_threshold` | Minimum confidence (0.0–1.0) to accept a reading; `null` disables the check |
| `use_correctional_alg` | `true` for full error correction, `false` for light mode |
| `rotated_180` | Rotate the image 180° (for upside-down cameras) |
| `shrink_last_3` | Reduce width of the rightmost 3 digits (for compact meter layouts) |
| `extended_last_digit` | Extend the capture area of the last digit |

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

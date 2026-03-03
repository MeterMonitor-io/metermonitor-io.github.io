# Configuration Reference

Complete reference for all MeterMonitor configuration options.

## Configuration File Location

- **Home Assistant Add-on**: Configuration UI (Settings → Add-ons → MeterMonitor → Configuration tab)
- **Docker**: `/data/options.json` (mounted volume)
- **Manual Install**: `settings.json` in project root

## Configuration Format

Configuration is stored in JSON format with the following structure:

```json
{
  "max_history": 200,
  "max_evals": 100,
  "mqtt": { ... },
  "allow_negative_correction": true,
  "publish_to": "string",
  "dbfile": "string",
  "http": { ... },
  "homeassistant": { ... },
  "ingress": false,
  "secret_key": "string",
  "enable_auth": false,
  "output_dataset": "string"
}
```

## Top-Level Configuration

### `max_history`

Maximum number of history entries to retain per water meter.

- **Type**: Integer
- **Default**: `200`
- **Range**: 1 - unlimited
- **Description**: When the history for a meter exceeds this limit, the oldest entries are automatically deleted. Lower values save disk space but reduce historical data availability.
- **Example**: `"max_history": 500`

### `max_evals`

Maximum number of evaluation records to retain per water meter.

- **Type**: Integer
- **Default**: `100`
- **Range**: 1 - unlimited
- **Description**: Evaluations contain detailed processing information including digit images, predictions, and confidence scores. They consume more storage than history entries.
- **Example**: `"max_evals": 200`

### `allow_negative_correction`

Enable correction of readings that appear to go backwards.

- **Type**: Boolean
- **Default**: `true`
- **Description**: When `true`, the error correction algorithm can adjust readings that are lower than the previous reading (e.g., due to misread digits). When `false`, backwards readings are rejected entirely.
- **Use Case**: Set to `false` if your meter physically cannot go backwards and you want strict monotonic enforcement.
- **Example**: `"allow_negative_correction": false`

### `publish_to`

MQTT topic template for publishing meter readings.

- **Type**: String
- **Default**: `"homeassistant/sensor/watermeter_{device}/"`
- **Placeholders**:
  - `{device}` - Replaced with meter name
- **Description**: Defines the MQTT topic pattern where readings are published. Used for Home Assistant MQTT discovery.
- **Example**: `"publish_to": "meters/{device}/state"`

### `dbfile`

Path to SQLite database file.

- **Type**: String
- **Default**:
  - Add-on: `"/data/metermonitor.db"`
  - Manual: `"data/metermonitor.db"`
- **Description**: Location of the SQLite database storing all meter data, settings, evaluations, and history.
- **Note**: Use `/data/` prefix in Docker/Add-on for persistence
- **Example**: `"dbfile": "/data/watermeters.sqlite"`

### `ingress`

Enable Home Assistant Ingress mode.

- **Type**: Boolean
- **Default**: `true` (add-on), `false` (standalone)
- **Description**: When enabled, restricts HTTP access to Home Assistant Ingress IP (172.30.32.2). Automatically set by Home Assistant addon configuration.
- **Warning**: Do not manually enable unless running as HA add-on
- **Example**: `"ingress": false`

### `secret_key`

Secret key for API authentication.

- **Type**: String
- **Default**: `"change_me"`
- **Description**: Used to authenticate API requests when `enable_auth` is true. Must be included in the `Secret` header of API requests.
- **Security**: Change this to a random, secure string in production
- **Example**: `"secret_key": "Xr9Kp2LmQ8vN5wZ3bT7yH4jF6"`

### `enable_auth`

Enable API authentication requirement.

- **Type**: Boolean
- **Default**: `false`
- **Description**: When `true`, all API endpoints require the `Secret` header matching `secret_key`.
- **Recommendation**: Enable for public-facing deployments
- **Example**: `"enable_auth": true`

### `output_dataset`

Directory path for training dataset export.

- **Type**: String
- **Default**: `"/data/output_dataset"`
- **Description**: Location where digit images are exported when using the dataset upload feature. Used for training model improvements.
- **Example**: `"output_dataset": "/data/training_data"`

## MQTT Configuration

### `mqtt.broker`

MQTT broker hostname or IP address.

- **Type**: String
- **Default**: `"172.30.32.1"` (HA add-on internal network)
- **Description**: Address of the MQTT broker to connect to.
- **Examples**:
  - Local: `"localhost"` or `"127.0.0.1"`
  - HA Add-on: `"172.30.32.1"` (Mosquitto add-on)
  - Remote: `"mqtt.example.com"`

### `mqtt.port`

MQTT broker port.

- **Type**: Integer
- **Default**: `1883`
- **Common Values**:
  - `1883` - MQTT (unencrypted)
  - `8883` - MQTTS (TLS/SSL)
  - `9001` - WebSocket
- **Example**: `"port": 8883`

### `mqtt.topic`

MQTT topic pattern to subscribe to.

- **Type**: String
- **Default**: `"MeterMonitor/#"`
- **Wildcards**:
  - `#` - Multi-level wildcard (all subtopics)
  - `+` - Single-level wildcard
- **Description**: Pattern for receiving meter images from ESP32 devices.
- **Examples**:
  - All meters: `"MeterMonitor/#"`
  - Specific: `"MeterMonitor/basement/#"`
  - Multiple: Subscribe programmatically or use wildcards

### `mqtt.username`

MQTT broker authentication username.

- **Type**: String
- **Default**: `"esp"`
- **Required**: If broker has authentication enabled
- **Example**: `"username": "mqtt_user"`

### `mqtt.password`

MQTT broker authentication password.

- **Type**: String
- **Default**: `"esp"`
- **Required**: If broker has authentication enabled
- **Security**: Store securely, avoid committing to version control
- **Example**: `"password": "secure_password_here"`

### Complete MQTT Example

```json
{
  "mqtt": {
    "broker": "192.168.1.100",
    "port": 1883,
    "topic": "home/watermeters/#",
    "username": "meter_reader",
    "password": "sup3r_s3cr3t"
  }
}
```

## HTTP Configuration

### `http.enabled`

Enable HTTP server for web interface and API.

- **Type**: Boolean
- **Default**: `true`
- **Description**: When `false`, only MQTT handler runs (headless mode).
- **Use Case**: Disable for MQTT-only deployments without web interface
- **Example**: `"enabled": false`

### `http.host`

HTTP server bind address.

- **Type**: String
- **Default**: `"0.0.0.0"` (all interfaces)
- **Options**:
  - `"0.0.0.0"` - Listen on all network interfaces
  - `"127.0.0.1"` - Localhost only (secure, local access only)
  - Specific IP - Bind to specific network interface
- **Security**: Use `"127.0.0.1"` with reverse proxy for added security
- **Example**: `"host": "127.0.0.1"`

### `http.port`

HTTP server port.

- **Type**: Integer
- **Default**: `8070`
- **Range**: 1-65535 (use 1024+ for non-root)
- **Note**: Must match port mapping in Docker/HA addon configuration
- **Example**: `"port": 8080`

### Complete HTTP Example

```json
{
  "http": {
    "enabled": true,
    "host": "0.0.0.0",
    "port": 8070
  }
}
```

## Home Assistant Integration

### `homeassistant.use_supervisor_token`

Use Home Assistant Supervisor token for authentication.

- **Type**: Boolean
- **Default**: `true`
- **Description**: When `true`, uses the `SUPERVISOR_TOKEN` environment variable (automatically available in HA add-ons). When `false`, uses the manual `token` field.
- **Recommendation**: Leave `true` for HA add-on deployments
- **Example**: `"use_supervisor_token": false`

### `homeassistant.url`

Home Assistant API base URL.

- **Type**: String
- **Default**: `"http://supervisor/core"` (add-on)
- **Description**: Base URL for Home Assistant API requests.
- **Options**:
  - Add-on: `"http://supervisor/core"` (recommended)
  - Local: `"http://homeassistant.local:8123"`
  - IP: `"http://192.168.1.100:8123"`
- **Note**: Do not include trailing slash
- **Example**: `"url": "http://192.168.1.50:8123"`

### `homeassistant.token`

Home Assistant long-lived access token.

- **Type**: String
- **Default**: `""` (empty, uses supervisor token)
- **Description**: Long-lived access token for HA API authentication. Only required if `use_supervisor_token` is `false`.
- **How to Generate**:
  1. In Home Assistant, go to Profile
  2. Scroll to "Long-Lived Access Tokens"
  3. Click "Create Token"
  4. Copy and paste the token here
- **Security**: Treat like a password, never commit to repositories
- **Example**: `"token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."`

### `homeassistant.request_timeout_s`

Timeout for Home Assistant API requests.

- **Type**: Integer (seconds)
- **Default**: `10`
- **Range**: 1-120
- **Description**: Maximum time to wait for HA API responses before timing out.
- **Recommendation**: Increase if experiencing timeout errors on slow networks
- **Example**: `"request_timeout_s": 30`

### Complete Home Assistant Example

#### For Add-on (Recommended)
```json
{
  "homeassistant": {
    "use_supervisor_token": true,
    "url": "http://supervisor/core",
    "token": "",
    "request_timeout_s": 10
  }
}
```

#### For Manual/External HA Integration
```json
{
  "homeassistant": {
    "use_supervisor_token": false,
    "url": "http://192.168.1.100:8123",
    "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiI...",
    "request_timeout_s": 15
  }
}
```

## Per-Meter Settings

Per-meter settings are configured via the web interface or API, stored in the `settings` table.

### Threshold Settings

#### `threshold_low` / `threshold_high`

Grayscale thresholding range for digits 1-4 (or 1-N if not using `threshold_last`).

- **Type**: Integer (0-255)
- **Defaults**: `low=0`, `high=100`
- **Description**: Defines the grayscale range to isolate digit pixels. Pixels in this range are considered part of the digit.
- **Tuning**: Use the web interface threshold picker or automatic threshold search

#### `threshold_last_low` / `threshold_last_high`

Grayscale thresholding range for the decimal digits (right side of the decimal separator).

- **Type**: Integer (0-255)
- **Defaults**: `low=0`, `high=100`
- **Description**: Separate thresholds for fractional digits, which often have different lighting.
- **Use Case**: When the last digits have different contrast (e.g., red numbers)

#### `decimals`

Number of decimal digits in the meter display.

- **Type**: Integer
- **Default**: `3`
- **Range**: `0..segments`
- **Description**: Controls decimal separator position and how many rightmost digits use `threshold_last_*`.

### Segmentation Settings

#### `segments`

Number of digit columns to extract from the meter display.

- **Type**: Integer
- **Default**: `7`
- **Range**: 1-15
- **Description**: How many individual digits to extract and classify.
- **Example**: `7` for a meter showing `1234.567`

#### `islanding_padding`

Padding percentage for connected component extraction.

- **Type**: Integer (percentage)
- **Default**: `20`
- **Range**: 0-50
- **Description**: Defines the inner region where digit components must exist to be considered valid. Higher values are more restrictive.
- **Tuning**: Increase if noise/artifacts are being classified as digits

#### `rotated_180`

Rotate image 180 degrees before processing.

- **Type**: Boolean
- **Default**: `false`
- **Description**: Use when the camera is mounted upside-down.

#### `shrink_last_3`

Apply different widths to last 3 digits.

- **Type**: Boolean
- **Default**: `false`
- **Description**: Compensates for meters where fractional digits are narrower.

#### `extended_last_digit`

Use extended ROI for the last digit.

- **Type**: Boolean
- **Default**: `false`
- **Description**: Extracts a slightly larger region for the last digit to capture rotation indicators.

### ROI Extraction Settings

#### `roi_extractor`

Region of Interest extraction method.

- **Type**: String
- **Default**: `"yolo"`
- **Options**:
  - `"yolo"` - Automatic detection using YOLOv11-OBB
  - `"orb"` - Template-based ORB feature matching
  - `"bypass"` - No extraction, use full image
- **Description**: Selects the algorithm for locating the meter display region.

#### `template_id`

Template UUID for ORB extractor.

- **Type**: String (UUID) or `null`
- **Default**: `null`
- **Required**: When `roi_extractor="orb"`
- **Description**: References a template in the `templates` table with precomputed ORB features.
- **Example**: `"550e8400-e29b-41d4-a716-446655440000"`

### Validation Settings

#### `max_flow_rate`

Maximum allowed flow rate in m³/hour.

- **Type**: Float
- **Default**: `1.0`
- **Range**: 0.01-100.0
- **Description**: Used by error correction to reject readings with unrealistic flow rates.
- **Calculation**: `(delta_reading / time_diff_hours)`
- **Example**: `2.5` for high-flow applications

#### `conf_threshold`

Minimum confidence threshold for accepting readings.

- **Type**: Float or `null`
- **Default**: `null` (no threshold)
- **Range**: 0.0-1.0
- **Description**: Reject evaluations where average confidence is below this threshold. When `null`, all readings are accepted regardless of confidence.
- **Example**: `0.7` to require 70% confidence

#### `use_correctional_alg`

Enable error correction algorithm.

- **Type**: Boolean
- **Default**: `true`
- **Options**:
  - `true` - Full correction with flow validation and fallback
  - `false` - Light correction (only rotation class replacement and low-confidence fixes)
- **Description**: Controls the aggressiveness of the error correction system.

## Configuration Templates

### Minimal Home Assistant Add-on
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

### Production Standalone
```json
{
  "max_history": 500,
  "max_evals": 200,
  "mqtt": {
    "broker": "mqtt.example.com",
    "port": 8883,
    "topic": "meters/#",
    "username": "production_user",
    "password": "s3cur3_p4ssw0rd"
  },
  "allow_negative_correction": false,
  "publish_to": "home/sensors/water/{device}/state",
  "dbfile": "/data/production.db",
  "http": {
    "enabled": true,
    "host": "0.0.0.0",
    "port": 8070
  },
  "homeassistant": {
    "use_supervisor_token": false,
    "url": "http://homeassistant.local:8123",
    "token": "YOUR_LONG_LIVED_TOKEN",
    "request_timeout_s": 15
  },
  "secret_key": "RANDOM_SECURE_KEY_HERE",
  "enable_auth": true,
  "output_dataset": "/data/training_exports"
}
```

### Development / Testing
```json
{
  "max_history": 50,
  "max_evals": 50,
  "mqtt": {
    "broker": "localhost",
    "port": 1883,
    "topic": "test/#",
    "username": "",
    "password": ""
  },
  "allow_negative_correction": true,
  "publish_to": "test/meter_{device}/",
  "dbfile": "test_data.db",
  "http": {
    "enabled": true,
    "host": "127.0.0.1",
    "port": 8070
  },
  "ingress": false,
  "enable_auth": false,
  "output_dataset": "./test_output"
}
```

## Environment Variables

Some configuration can be overridden via environment variables:

| Variable | Description | Example |
|----------|-------------|---------|
| `SUPERVISOR_TOKEN` | HA Supervisor token (automatic in add-on) | Auto-set |
| `TZ` | Timezone for timestamps | `Europe/Berlin` |
| `PYTHONUNBUFFERED` | Disable Python output buffering | `1` |

## Configuration Validation

### Home Assistant Add-on

The add-on uses `config.json` schema validation:

```json
{
  "schema": {
    "max_history": "int(1,)",
    "max_evals": "int(1,)",
    "mqtt": {
      "broker": "str",
      "port": "port",
      "topic": "str",
      "username": "str",
      "password": "str"
    },
    "allow_negative_correction": "bool",
    "publish_to": "str",
    "dbfile": "str",
    "homeassistant": {
      "use_supervisor_token": "bool",
      "url": "str?",
      "token": "str?",
      "request_timeout_s": "int(1,)"
    }
  }
}
```

### Manual Validation

Validate JSON syntax:

```bash
python -m json.tool settings.json
```

## Troubleshooting Configuration

### Invalid JSON Syntax

**Error**: `json.decoder.JSONDecodeError`

**Solution**: Validate JSON syntax, common issues:
- Missing commas between items
- Trailing commas (not allowed in strict JSON)
- Unquoted strings
- Unescaped special characters

### MQTT Connection Failed

**Check**:
- `mqtt.broker` is reachable (ping test)
- `mqtt.port` is correct
- `mqtt.username` and `mqtt.password` match broker config
- Broker has anonymous access disabled (if using auth)

### Home Assistant API 401 Errors

**Check**:
- `homeassistant.token` is valid and not expired
- `homeassistant.url` is correct and reachable
- If using `use_supervisor_token=true`, ensure running as add-on

### Database Locked Errors

**Check**:
- Only one instance is running
- `dbfile` path is writable
- No file system errors (disk full, permissions)

## Next Steps

- [User Guide](user-guide.md) - Learn how to use the web interface
- [ESP32 Setup](esp32-setup.md) - Configure your camera devices
- [API Reference](api-reference.md) - Integrate with external systems

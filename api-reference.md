# MeterMonitor API Reference

Complete REST API documentation for MeterMonitor v3.1.x.

## Table of Contents

1. [Authentication](#authentication)
2. [Base URL](#base-url)
3. [Error Handling](#error-handling)
4. [Endpoints](#endpoints)
   - [Watermeters](#watermeters)
   - [Settings](#settings)
   - [History](#history)
   - [Evaluations](#evaluations)
   - [Sources](#sources)
   - [Templates](#templates)
   - [Home Assistant Integration](#home-assistant-integration)
   - [Dataset Export](#dataset-export)
   - [Discovery](#discovery)
   - [Alerts](#alerts)
5. [MQTT Message Format](#mqtt-message-format)

---

## Authentication

MeterMonitor supports optional API authentication via secret key.

### Configuration

**Home Assistant Add-on**:
- Authentication is disabled by default (protected by Ingress)
- Requests must originate from Ingress IP (172.30.32.2)

**Standalone**:
- Set `enable_auth: true` in `settings.json`
- Configure `secret_key` (change from default!)

### Using Authentication

Include secret key in request headers:

```http
Secret: your_secret_key_here
```

**Example with curl**:
```bash
curl -H "Secret: your_secret_key" http://localhost:8070/api/watermeters
```

---

## Base URL

**Home Assistant Add-on**: `http://homeassistant.local:8123/api/hassio_ingress/{token}/`
- Access via Ingress (no direct port access)
- Token is managed automatically by supervisor

**Standalone**: `http://localhost:8070/`
- Default port: 8070 (configurable in `settings.json`)
- Host: 0.0.0.0 (all interfaces)

**API Prefix**: All endpoints start with `/api/`

---

## Error Handling

### Standard Error Response

```json
{
  "detail": "Error message describing what went wrong"
}
```

### HTTP Status Codes

| Code | Meaning | Common Causes |
|------|---------|---------------|
| 200 | OK | Request succeeded |
| 400 | Bad Request | Invalid parameters, missing required fields |
| 401 | Unauthorized | Authentication failed or missing |
| 403 | Forbidden | IP restriction (Ingress mode) |
| 404 | Not Found | Resource doesn't exist |
| 500 | Internal Server Error | Server-side processing error |
| 502 | Bad Gateway | Home Assistant API unreachable |

---

## Endpoints

### Watermeters

#### List All Watermeters

```http
GET /api/watermeters
```

Returns all configured watermeters (setup completed).

**Response**:
```json
{
  "watermeters": [
    [
      "kitchen_meter",           // name
      "2026-01-28T10:30:00",    // last picture timestamp
      -45,                       // WiFi RSSI
      1234567,                   // last history value
      ["base64...", "base64..."], // thresholded digits (inverted)
      true                       // has bounding box
    ]
  ]
}
```

#### Get Watermeter Details

```http
GET /api/watermeters/{name}
```

**Parameters**:
- `name`: Watermeter identifier (path parameter)

**Response**:
```json
{
  "name": "kitchen_meter",
  "picture_number": 1542,
  "WiFi-RSSI": -45,
  "picture": {
    "format": "jpg",
    "timestamp": "2026-01-28T10:30:00",
    "width": 640,
    "height": 480,
    "length": 12543,
    "data": "base64_encoded_image_data",
    "data_bbox": "base64_encoded_bbox_image"
  },
  "dataset_present": true
}
```

#### Delete Watermeter

```http
DELETE /api/watermeters/{name}
```

Deletes watermeter and all associated data (settings, history, evaluations, sources).

**Response**:
```json
{
  "message": "Watermeter deleted",
  "name": "kitchen_meter"
}
```

#### Get Watermeter History

```http
GET /api/watermeters/{name}/history
```

**Response**:
```json
{
  "history": [
    [1234567, "2026-01-28T10:30:00", 0.95, false],
    // [value, timestamp, confidence, manual]
  ]
}
```

---

### Settings

#### List All Settings

```http
GET /api/settings
```

**Response**:
```json
{
  "settings": [
    {
      "name": "kitchen_meter",
      "threshold_low": 0,
      "threshold_high": 100,
      "threshold_last_low": 0,
      "threshold_last_high": 100,
      "islanding_padding": 20,
      "segments": 7,
      "rotated_180": false,
      "shrink_last_3": false,
      "extended_last_digit": false,
      "max_flow_rate": 1.0,
      "conf_threshold": null,
      "roi_extractor": "yolo",
      "template_id": null,
      "segment_mode": "display",
      "digit_models": null,
      "decimals": 3,
      "use_correctional_alg": true
    }
  ]
}
```

#### Get Settings for Watermeter

```http
GET /api/watermeters/{name}/settings
GET /api/settings/{name}
```

Both endpoints return the same data.

**Response**:
```json
{
  "threshold_low": 0,
  "threshold_high": 100,
  "threshold_last_low": 0,
  "threshold_last_high": 100,
  "islanding_padding": 20,
  "segments": 7,
  "shrink_last_3": false,
  "extended_last_digit": false,
  "max_flow_rate": 1.0,
  "rotated_180": false,
  "conf_threshold": null,
  "roi_extractor": "yolo",
  "template_id": null,
  "segment_mode": "display",
  "digit_models": null,
  "decimals": 3,
  "use_correctional_alg": true
}
```

#### Update Settings

```http
PUT /api/watermeters/{name}/settings
POST /api/settings
```

**Request Body**:
```json
{
  "name": "kitchen_meter",
  "threshold_low": 0,
  "threshold_high": 120,
  "threshold_last_low": 0,
  "threshold_last_high": 100,
  "islanding_padding": 20,
  "segments": 7,
  "rotated_180": false,
  "shrink_last_3": false,
  "extended_last_digit": false,
  "max_flow_rate": 1.0,
  "conf_threshold": 0.6,
  "roi_extractor": "yolo",
  "template_id": null,
  "segment_mode": "display",
  "digit_models": null,
  "decimals": 3,
  "use_correctional_alg": true
}
```

**Response**:
```json
{
  "message": "Settings updated",
  "name": "kitchen_meter"
}
```

#### Search Optimal Thresholds

```http
POST /api/watermeters/{name}/search_thresholds
```

Performs grid search to find optimal threshold values.

**Request Body**:
```json
{
  "steps": 10  // 3-25, higher = more accurate but slower
}
```

**Response**:
```json
{
  "threshold": 85,
  "threshold_last": 90,
  "avg_confidence": 0.943,
  "sample_digits": ["base64...", "base64..."],
  "search_time_ms": 1234
}
```

Or on error:
```json
{
  "error": "No watermeter picture found"
}
```

---

### History

#### Create Manual History Entry

```http
POST /api/history
```

**Request Body**:
```json
{
  "value": 1234567,
  "timestamp": "2026-01-28T10:30:00"
}
```

**Response**:
```json
{
  "message": "History entry created"
}
```

#### Delete History for Watermeter

```http
DELETE /api/history/{name}
```

**Response**:
```json
{
  "message": "History deleted",
  "name": "kitchen_meter"
}
```

---

### Evaluations

#### Get Evaluations

```http
GET /api/watermeters/{name}/evals
GET /api/watermeters/{name}/evals?amount=50
GET /api/watermeters/{name}/evals?amount=50&from_id=1234
```

**Query Parameters**:
- `amount`: Maximum evaluations to return (optional)
- `from_id`: Return evaluations older than this ID for pagination (optional)

**Response**:
```json
{
  "evals": [
    {
      "id": 1234,
      "colored_digits": ["base64...", "base64..."],
      "th_digits": ["base64...", "base64..."],
      "th_digits_inverted": ["base64...", "base64..."],
      "predictions": [
        [["5", 0.98], ["6", 0.01], ["8", 0.01]],
        [["2", 0.95], ["7", 0.03], ["3", 0.02]]
      ],
      "timestamp": "2026-01-28T10:30:00",
      "result": 1234567,
      "total_confidence": 0.93,
      "used_confidence": 0.95,
      "outdated": false,
      "denied_digits": [false, false, false, false, false, false, false],
      "denied_digits_count": 0,
      "flow_rate_m3h": 0.012,
      "delta_m3": 0.003,
      "delta_raw": 3,
      "time_diff_min": 15.0,
      "rejection_reason": null,
      "negative_correction_applied": false,
      "fallback_digit_count": 0,
      "digits_changed_vs_last": 1,
      "digits_changed_vs_top_pred": 0,
      "prediction_rank_used_counts": [7, 0, 0],
      "timestamp_adjusted": false
    }
  ]
}
```

#### Get Single Evaluation

```http
GET /api/watermeters/{name}/evals/{eval_id}
```

**Response**: Same structure as single evaluation object above.

#### Get Evaluation Count

```http
GET /api/watermeters/{name}/evals/count
```

**Response**:
```json
{
  "count": 1234
}
```

#### Delete All Evaluations

```http
DELETE /api/watermeters/{name}/evals
```

**Response**:
```json
{
  "message": "Deleted 1234 evaluations",
  "count": 1234
}
```

#### Re-evaluate Latest Picture

```http
POST /api/watermeters/{name}/evaluations/reevaluate
```

Processes the latest image with current settings.

**Response**:
```json
{
  "result": true
}
```

Or on error:
```json
{
  "result": false,
  "error": "No display detected in image"
}
```

#### Mark Evaluations as Outdated

```http
POST /api/watermeters/{name}/evaluations/mark-outdated
```

Marks all evaluations as outdated to trigger re-evaluation with new settings.

**Response**:
```json
{
  "result": true,
  "count": 1234
}
```

#### Get Sample Evaluation

```http
POST /api/watermeters/{name}/evaluations/sample
POST /api/watermeters/{name}/evaluations/sample/{offset}
```

Returns random (offset=-1) or specific historic evaluation re-processed with current settings.

**Parameters**:
- `offset`: 0 = latest, 1 = second latest, -1 = random (optional, default: -1)

**Response**: Same as evaluation object with additional processing metadata.

---

### Sources

#### List All Sources

```http
GET /api/sources
```

**Response**:
```json
{
  "sources": [
    {
      "id": 1,
      "name": "kitchen_meter",
      "source_type": "mqtt",
      "enabled": true,
      "poll_interval_s": null,
      "config": null,
      "last_success_ts": "2026-01-28T10:30:00",
      "last_error": null,
      "created_ts": "2026-01-20T08:00:00",
      "updated_ts": "2026-01-28T10:30:00"
    },
    {
      "id": 2,
      "name": "bathroom_meter",
      "source_type": "ha_camera",
      "enabled": true,
      "poll_interval_s": 600,
      "config": {
        "camera_entity_id": "camera.bathroom_meter",
        "flash_entity_id": "light.bathroom_meter_led",
        "flash_delay_ms": 1000
      },
      "last_success_ts": "2026-01-28T10:25:00",
      "last_error": null,
      "created_ts": "2026-01-22T12:00:00",
      "updated_ts": "2026-01-28T10:25:00"
    }
  ]
}
```

#### Create Source

```http
POST /api/sources
```

**Request Body (HA Camera)**:
```json
{
  "name": "bathroom_meter",
  "source_type": "ha_camera",
  "enabled": true,
  "poll_interval_s": 600,
  "config": {
    "camera_entity_id": "camera.bathroom_meter",
    "flash_entity_id": "light.bathroom_meter_led",
    "flash_delay_ms": 1000
  }
}
```

**Request Body (HTTP)**:
```json
{
  "name": "garage_meter",
  "source_type": "http",
  "enabled": true,
  "poll_interval_s": 300,
  "config": {
    "url": "http://192.168.1.100/snapshot.jpg",
    "headers": {
      "Authorization": "Bearer token123"
    },
    "body": null
  }
}
```

**Response**:
```json
{
  "message": "Source created"
}
```

#### Update Source

```http
PUT /api/sources/{source_id}
```

**Request Body**:
```json
{
  "enabled": true,
  "poll_interval_s": 900,
  "config": {
    "camera_entity_id": "camera.new_bathroom_meter"
  }
}
```

All fields are optional; only provided fields are updated.

**Response**:
```json
{
  "message": "Source updated"
}
```

#### Delete Source

```http
DELETE /api/sources/{source_id}
```

**Response**:
```json
{
  "message": "Source deleted",
  "id": 2
}
```

#### Trigger Source Capture

```http
POST /api/sources/{source_id}/capture
```

Manually triggers capture and processing for a source.

**Response**:
```json
{
  "message": "Capture and processing triggered"
}
```

---

### Templates

#### Get Template

```http
GET /api/templates/{template_id}
```

**Response**:
```json
{
  "id": "550e8400-e29b-41d4-a716-446655440000",
  "name": "kitchen_meter_template",
  "created_at": "2026-01-20T10:00:00",
  "reference_image_base64": "base64_encoded_image",
  "image_width": 640,
  "image_height": 480,
  "config": {
    "display_corners": [
      [100.0, 150.0],
      [500.0, 150.0],
      [500.0, 300.0],
      [100.0, 300.0]
    ],
    "target_width": 400,
    "target_height": 150,
    "target_width_ext": 480,
    "target_height_ext": 180
  }
}
```

#### Create Template

```http
POST /api/templates
```

**Request Body**:
```json
{
  "name": "new_meter_template",
  "extractor": "orb",
  "reference_image_base64": "base64_encoded_image",
  "image_width": 640,
  "image_height": 480,
  "display_corners": [
    [0.15625, 0.3125],
    [0.78125, 0.3125],
    [0.78125, 0.625],
    [0.15625, 0.625]
  ]
}
```

**Note**: `display_corners` can be in normalized (0-1) or pixel coordinates. Values ≤2.0 are treated as normalized.

**Response**:
```json
{
  "id": "550e8400-e29b-41d4-a716-446655440000",
  "name": "new_meter_template"
}
```

---

### Home Assistant Integration

#### Get HA Connection Status

```http
GET /api/ha/status
```

**Response**:
```json
{
  "configured": true,
  "base_url": "http://supervisor/core",
  "use_supervisor_token": true,
  "supervisor_token_present": true,
  "manual_token_present": false,
  "ok": true
}
```

#### List HA Cameras

```http
GET /api/ha/cameras
```

**Response**:
```json
{
  "cameras": [
    {
      "entity_id": "camera.bathroom_meter",
      "name": "Bathroom Meter Camera",
      "suggested_flash_entity_id": "light.bathroom_meter_led"
    },
    {
      "entity_id": "camera.garage_meter",
      "name": "Garage Meter Camera",
      "suggested_flash_entity_id": null
    }
  ]
}
```

#### Call HA Service

```http
POST /api/ha/service
```

**Request Body**:
```json
{
  "domain": "light",
  "service": "turn_on",
  "entity_id": "light.bathroom_meter_led"
}
```

**Response**:
```json
{
  "result": [
    {
      "entity_id": "light.bathroom_meter_led",
      "state": "on"
    }
  ]
}
```

#### Capture from HA Camera or HTTP

```http
POST /api/capture-now
```

**Request Body (HA Camera)**:
```json
{
  "cam_entity_id": "camera.bathroom_meter",
  "flash_entity_id": "light.bathroom_meter_led",
  "flash_delay_ms": 1000
}
```

**Request Body (HTTP)**:
```json
{
  "http_url": "http://192.168.1.100/snapshot.jpg",
  "http_headers": {
    "Authorization": "Bearer token"
  },
  "http_body": null
}
```

**Response**:
```json
{
  "result": true,
  "data": "base64_encoded_image",
  "format": "jpg",
  "flash_used": true
}
```

---

### Dataset Export

#### Upload Dataset

```http
POST /api/dataset/upload
```

**Request Body**:
```json
{
  "name": "kitchen_meter",
  "labels": ["5", "2", "3", "4", "5", "6", "7"],
  "colored": ["base64...", "base64...", ...],
  "thresholded": ["base64...", "base64...", ...]
}
```

Arrays must have equal length. Labels: 0-9 or 'r'.

**Response**:
```json
{
  "saved": 7,
  "output_root": "/data/output_dataset"
}
```

#### Download Dataset

```http
GET /api/dataset/{name}/download
```

Returns ZIP file with dataset structure.

**Response**: Binary ZIP data with headers:
```
Content-Type: application/zip
Content-Disposition: attachment; filename=kitchen_meter_dataset.zip
```

#### Delete Dataset

```http
DELETE /api/dataset/{name}
```

**Response**:
```json
{
  "message": "Dataset deleted",
  "name": "kitchen_meter"
}
```

---

### Discovery

#### Get Discovered Watermeters

```http
GET /api/discovery
```

Returns watermeters that have not completed setup.

**Response**:
```json
{
  "watermeters": [
    ["new_meter", "2026-01-28T10:30:00", -45, "mqtt"],
    // [name, timestamp, rssi, source_type]
  ]
}
```

#### Setup New Watermeter

```http
POST /api/setup
```

**Request Body**:
```json
{
  "name": "new_meter",
  "picture_number": 1,
  "WiFi_RSSI": -45,
  "picture": {
    "format": "jpg",
    "timestamp": "2026-01-28T10:30:00",
    "width": 640,
    "height": 480,
    "length": 12543,
    "data": "base64_encoded_image"
  }
}
```

**Response**:
```json
{
  "message": "Watermeter configured",
  "name": "new_meter"
}
```

#### Finish Setup

```http
POST /api/setup/{name}/finish
```

**Request Body**:
```json
{
  "value": 1234567,
  "timestamp": "2026-01-28T10:30:00"
}
```

**Response**:
```json
{
  "message": "Setup completed"
}
```

#### Re-enable Setup Mode

```http
POST /api/setup/{name}/enable
```

Moves watermeter back to discovery state.

**Response**:
```json
{
  "message": "Setup completed"
}
```

---

### Alerts

#### Get Current Alerts

```http
GET /api/alerts
```

Returns system-wide alerts (MQTT connection, polling errors, etc.).

**Response**:
```json
{
  "alerts": {
    "mqtt": "Reconnecting to MQTT broker",
    "polling_bathroom_meter": "Polling failed for source 'bathroom_meter': Connection timeout"
  }
}
```

---

### Configuration

#### Get Configuration

```http
GET /api/config
```

Returns server configuration (excluding secret key).

**Response**:
```json
{
  "max_history": 200,
  "max_evals": 100,
  "mqtt": {
    "broker": "172.30.32.1",
    "port": 1883,
    "topic": "MeterMonitor/#",
    "username": "esp",
    "password": "esp"
  },
  "allow_negative_correction": true,
  "publish_to": "homeassistant/sensor/watermeter_{device}/",
  "dbfile": "/data/metermonitor.db",
  "homeassistant": {
    "use_supervisor_token": true,
    "url": "http://supervisor/core",
    "request_timeout_s": 10
  }
}
```

---

## MQTT Message Format

MeterMonitor listens on the configured MQTT topic (default: `MeterMonitor/#`).

### Incoming Message Format

ESP32 devices should publish messages in this format:

**Topic**: `MeterMonitor/{device_name}`

**Payload**:
```json
{
  "name": "kitchen_meter",
  "WiFi-RSSI": -45,
  "picture": {
    "timestamp": "2026-01-28T10:30:00",
    "format": "jpg",
    "data": "base64_encoded_image_data"
  }
}
```

**Field Descriptions**:
- `name`: Unique meter identifier (alphanumeric, underscores, hyphens)
- `WiFi-RSSI` (optional): Signal strength in dBm
- `picture.timestamp` (optional): ISO-8601 or Unix timestamp (seconds/ms). Missing/invalid values are replaced with server time.
- `picture.format` (optional): Image format hint (`jpg`, `jpeg`, `png`, ...). If omitted, the server detects format.
- `picture.data` (**required**): Base64 image bytes. Both raw base64 and `data:image/...;base64,...` are accepted.

Server-generated fields:
- `picture_number` is managed by MeterMonitor.
- `picture.width`, `picture.height`, `picture.length` are derived from decoded image data.

---

## Realtime Updates (WebSocket)

When automatic evaluations complete (MQTT / polling / HTTP capture), the backend emits realtime events.

```text
GET /api/ws/evaluations?secret=...
```

Event payload:

```json
{
  "type": "evaluation_created",
  "name": "kitchen_meter",
  "timestamp": "2026-03-03T14:10:00"
}
```

Notes:
- Manual reevaluation in setup/UI does not emit this event.
- Client should refresh meter/setup views only if `name` matches current meter.

### Outgoing Messages (HA Discovery)

MeterMonitor publishes to configured topic (default: `homeassistant/sensor/watermeter_{device}/`).

**Discovery Topic**: `homeassistant/sensor/watermeter_{device}/config`

**Discovery Payload**:
```json
{
  "name": "Kitchen Meter",
  "unique_id": "watermeter_kitchen_meter_value",
  "state_topic": "homeassistant/sensor/watermeter_kitchen_meter/state",
  "json_attributes_topic": "homeassistant/sensor/watermeter_kitchen_meter/attributes",
  "unit_of_measurement": "m³",
  "device_class": "water",
  "state_class": "total_increasing",
  "device": {
    "identifiers": ["watermeter_kitchen_meter"],
    "name": "Kitchen Meter",
    "model": "MeterMonitor",
    "manufacturer": "MeterMonitor"
  }
}
```

**State Topic**: `homeassistant/sensor/watermeter_{device}/state`

**State Payload**: `"0.123"` (current reading in m³)

**Attributes Topic**: `homeassistant/sensor/watermeter_{device}/attributes`

**Attributes Payload**:
```json
{
  "confidence": 0.95,
  "timestamp": "2026-01-28T10:30:00",
  "unit_of_measurement": "m³"
}
```

---

## Rate Limiting

No built-in rate limiting. Recommended practices:
- Limit ESP32 upload frequency to 1-5 minutes
- Limit HA camera polling to 10+ minute intervals
- Avoid rapid API calls (use pagination for large datasets)

---

## Versioning

API version follows MeterMonitor version defined in `config.json`.

Breaking changes are documented in CHANGELOG.md.

---

## Additional Resources

- [User Guide](user-guide.md)
- [ESP32 Setup](esp32-setup.md)
- [Troubleshooting](troubleshooting.md)

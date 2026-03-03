# Architecture

This document describes the technical architecture and design of MeterMonitor.

## System Overview

```
┌─────────────┐      MQTT       ┌──────────────────┐
│ ESP32-CAM   │ ─────────────> │  MeterMonitor    │
│  or HA Cam  │                │     Server       │
└─────────────┘                └──────────────────┘
                                        │
                    ┌───────────────────┼───────────────────┐
                    │                   │                   │
                    ▼                   ▼                   ▼
            ┌──────────────┐   ┌──────────────┐   ┌──────────────┐
            │   MQTT       │   │  FastAPI     │   │   Vue 3      │
            │   Handler    │   │  REST API    │   │   Frontend   │
            └──────────────┘   └──────────────┘   └──────────────┘
                    │                   │
                    └─────────┬─────────┘
                              ▼
                    ┌──────────────────┐
                    │  MeterPredictor  │
                    │   (AI Models)    │
                    └──────────────────┘
                              │
                    ┌─────────┴─────────┐
                    │                   │
                    ▼                   ▼
            ┌──────────────┐   ┌──────────────┐
            │  YOLOv11     │   │  CNN Digit   │
            │  (ROI)       │   │  Classifier  │
            └──────────────┘   └──────────────┘
                              │
                              ▼
                    ┌──────────────────┐
                    │    SQLite DB     │
                    │  (History, etc)  │
                    └──────────────────┘
                              │
                              ▼
                    ┌──────────────────┐
                    │   MQTT Publish   │
                    │  (HA Discovery)  │
                    └──────────────────┘
```

## Core Components

### 1. Entry Point (`run.py`)

The main application entry point that:
- Loads configuration from `settings.json` or `/data/options.json` (HA addon mode)
- Merges with default HA settings
- Runs database migrations
- Initializes the MQTT handler in a background thread
- Starts the FastAPI HTTP server (if enabled)
- Initializes polling handler for camera sources

### 2. MQTT Handler (`lib/mqtt_handler.py`)

Handles MQTT communication:

**Responsibilities:**
- Connects to MQTT broker with authentication
- Subscribes to configured topic (default: `MeterMonitor/#`)
- Receives base64-encoded images from ESP32 devices
- Validates message format
- Stores/updates watermeter records in database
- Triggers image processing via MeterPredictor
- Publishes MQTT discovery messages for Home Assistant
- Implements reconnection logic with exponential backoff
- Manages alert states for frontend display

**Message Format:**
```json
{
  "name": "watermeter_1",
  "WiFi-RSSI": -56,
  "picture": {
    "format": "jpg",
    "timestamp": "2026-01-28T10:30:00",
    "data": "base64_encoded_image_data"
  }
}
```

Notes:
- `WiFi-RSSI`, `picture.format`, `picture.timestamp` are optional.
- `picture.timestamp` accepts ISO and Unix (s/ms).
- `picture.data` accepts raw base64 and data-URI format.
- `picture_number`, image dimensions and length are derived/stored server-side.

### 3. HTTP Server (`lib/http_server.py`)

FastAPI-based REST API providing:

**Core Endpoints:**
- `/api/watermeters` - CRUD operations for water meters
- `/api/settings` - Threshold and configuration management
- `/api/history` - Reading history
- `/api/evaluations` - Evaluation results and re-evaluation
- `/api/sources` - Image source management (MQTT, HA Camera, HTTP)
- `/api/templates` - ROI template management
- `/api/ha/*` - Home Assistant integration endpoints
- `/api/dataset/*` - Training dataset export

**Features:**
- CORS middleware for frontend access
- Optional authentication via secret key
- Ingress IP restriction for Home Assistant addon mode
- Automatic API documentation at `/docs`
- Static file serving for Vue.js frontend

### 4. Meter Predictor (`lib/meter_processing/meter_processing.py`)

The core AI processing component using ONNX Runtime:

**Workflow:**
1. **ROI Extraction** - Extract display region using selected extractor:
   - `YOLOExtractor`: YOLOv11-OBB for automatic detection
   - `ORBExtractor`: Template-based feature matching
   - `BypassExtractor`: Use full image (no detection)

2. **Perspective Transform** - Straighten rotated meter displays

3. **Segmentation** - Split into individual digit columns

4. **Brightness Normalization** - Adjust each segment to target brightness

5. **Thresholding** - Binary conversion with connected component analysis

6. **Digit Classification** - CNN prediction for each digit (0-9, r=rotation)

7. **Error Correction** - Apply correctional algorithm:
   - **Full Mode**: Complete validation with flow rate check, max rate limits, fallback handling
   - **Light Mode**: Only replace rotation class and low-confidence digits

**Models:**
- `models/yolo-best-obb-2.onnx` - YOLOv11 Oriented Bounding Box detection
- `models/best_model.onnx` - CNN digit classifier (11 classes: 0-9, rotation)

**Optimizations:**
- ONNX Runtime for ~70% less memory vs TensorFlow/PyTorch
- Disabled memory arena and pattern for minimal footprint
- Explicit garbage collection after operations
- Singleton pattern for shared model instances

### 5. ROI Extractors (`lib/meter_processing/roi_extractors/`)

Pluggable region-of-interest extraction strategies:

#### YOLOExtractor
- Uses YOLOv11 OBB model for automatic display detection
- Applies perspective transform to straighten rotated displays
- Supports extended last digit for better fractional reading
- Handles multiple detection confidences

#### ORBExtractor
- Template-based matching using ORB features
- User-defined reference image and corner points
- Feature detection and homography computation
- Robust to lighting variations and partial occlusions
- Precomputed keypoints/descriptors for performance

#### BypassExtractor
- No detection, uses full input image
- Useful for pre-cropped images or testing

### 6. Polling Handler (`lib/polling_handler.py`)

Manages scheduled image capture from sources:

**Features:**
- Background thread polling enabled sources
- Configurable per-source intervals (seconds)
- Supports Home Assistant camera entities and HTTP endpoints
- Flash light control for HA camera sources
- Error tracking and retry logic
- Last success timestamp tracking

### 7. Database Layer (`db/migrations.py`)

SQLite database with automatic migrations:

**Tables:**
- `watermeters` - Meter metadata and latest image
- `settings` - Per-meter configuration (thresholds, segments, ROI method)
- `evaluations` - Full evaluation history with metadata
- `history` - Value history (readings accepted into timeline)
- `sources` - Image source definitions (MQTT, HA Camera, HTTP)
- `templates` - ROI template definitions with precomputed features

**Migration System:**
- Automatic schema updates on startup
- Column additions with default values
- Data transformations for schema changes
- Backwards compatible

### 8. Frontend (`frontend/`)

Modern Vue 3 SPA with Vite:

**Key Components:**
- `DiscoveryView.vue` - New meter discovery and onboarding
- `MeterView.vue` - Meter dashboard and list
- `SetupView.vue` - Detailed meter configuration
- `MeterDetails.vue` - Meter info, charts, evaluations
- `ThresholdPicker.vue` - Visual threshold adjustment
- `ROIExtractorSelect.vue` - ROI method selector with template editor
- `EvaluationResultList.vue` - Infinite scroll evaluation history
- `MeterCharts.vue` - Usage and confidence charts (Chart.js)
- `SourceCollapse.vue` - Source management UI

**State Management:**
- Component-local state with Vue 3 Composition API
- API calls via fetch to backend REST endpoints
- Reactive updates on data changes

## Data Flow

### Image Processing Pipeline

```
1. Image Capture
   ├─ ESP32-CAM → MQTT → MQTTHandler
   ├─ HA Camera → PollingHandler → capture_from_ha_source
   └─ HTTP → PollingHandler → capture_from_http_source
                ↓
2. Database Storage
   - Store raw image and metadata in watermeters table
   - Create or update MQTT source entry
                ↓
3. ROI Extraction (MeterPredictor)
   ├─ Load settings (roi_extractor, template_id, etc)
   ├─ Initialize extractor (YOLO/ORB/Bypass)
   ├─ Extract display region
   └─ Apply brightness normalization
                ↓
4. Segmentation
   ├─ Split into N columns (default: 7)
   ├─ Apply shrink_last_3 if enabled
   └─ Handle extended_last_digit
                ↓
5. Thresholding
   ├─ Convert to grayscale
   ├─ Apply threshold_low/high (or threshold_last for configured decimal digits)
   ├─ Connected component analysis
   └─ Island extraction with padding
                ↓
6. Digit Classification
   ├─ Resize to 40x64
   ├─ Normalize to [0,1]
   ├─ Per-digit model inference (rotating or 7-segment ONNX)
   └─ Get top-3 predictions per digit
                ↓
7. Error Correction (history_correction.py)
   ├─ Replace rotation class ('r') with previous digit
   ├─ Apply confidence-based corrections
   ├─ Validate flow rate (Full mode)
   ├─ Check max_flow_rate constraints (Full mode)
   └─ Store correction metadata
                ↓
8. Storage & Publishing
   ├─ Insert evaluation record
   ├─ Update history if accepted
   ├─ Publish to MQTT (homeassistant/sensor/...)
   └─ Push realtime event over `/api/ws/evaluations` (automatic captures only)
```

### Configuration Flow

```
User Action (Frontend)
        ↓
    REST API Call
        ↓
   HTTP Server
        ↓
  Database Update
        ↓
  Mark Evaluations Outdated
        ↓
  Re-evaluation Trigger (optional)
```

## Technology Stack

### Backend
- **Python 3.12** - Runtime
- **FastAPI** - REST API framework
- **Paho MQTT** - MQTT client library
- **ONNX Runtime** - AI model inference
- **OpenCV** - Image processing
- **SQLite** - Database
- **Uvicorn** - ASGI server

### Frontend
- **Vue 3** - UI framework
- **Vite** - Build tool and dev server
- **Chart.js** - Data visualization
- **Axios** - HTTP client
- **Lodash** - Utility library

### AI Models
- **YOLOv11-OBB** - Oriented bounding box detection
- **Custom CNN** - Digit classification (trained on haverland dataset)

### Deployment
- **Docker** - Containerization
- **Home Assistant Add-on** - Native HA integration
- **ESPHome** - ESP32 firmware framework

## Security Considerations

### Authentication
- Optional secret key authentication for API endpoints
- Ingress IP restriction in Home Assistant addon mode (172.30.32.2)
- Home Assistant Supervisor token support

### Data Storage
- SQLite database with proper foreign keys
- Base64 image encoding in database
- No plaintext password storage (MQTT credentials in config only)

### Network
- MQTT over TCP (consider TLS in production)
- HTTP server (consider reverse proxy with HTTPS)
- Configurable timeouts for HA API requests

## Performance Characteristics

### Memory Usage
- **ONNX Runtime**: ~70% less RAM than TensorFlow/PyTorch
- **Singleton Models**: Shared between MQTT and HTTP handlers
- **Explicit GC**: After batch operations

### Processing Speed
- **YOLO Detection**: ~100-500ms (CPU-dependent)
- **Digit Classification**: ~50-100ms for 7 digits
- **Total Pipeline**: ~200-800ms per image

### Scalability
- **Single-threaded**: MQTT in background thread, HTTP via Uvicorn workers
- **Database**: SQLite (consider PostgreSQL for multi-instance)
- **Concurrent Meters**: Tested with 5+ meters, no theoretical limit

## Extensibility Points

### Custom ROI Extractors
Implement `lib/meter_processing/roi_extractors/base.py`:

```python
class CustomExtractor(BaseROIExtractor):
    def extract(self, image):
        # Your extraction logic
        return cropped, extended, bbox
```

Register in `__init__.py` and reference in settings.

### Custom Correction Algorithms
Extend `lib/history_correction.py` functions for custom validation logic.

### Additional Image Sources
Add new source types in `lib/capture_utils.py` and update API validation.

## Deployment Architectures

### Home Assistant Add-on (Recommended)
```
┌──────────────────────────────────────┐
│       Home Assistant Host            │
│  ┌────────────────────────────────┐  │
│  │   MeterMonitor Add-on          │  │
│  │  (Docker Container)            │  │
│  │   - FastAPI Server             │  │
│  │   - MQTT Handler               │  │
│  │   - SQLite DB                  │  │
│  └────────────────────────────────┘  │
│  ┌────────────────────────────────┐  │
│  │   Mosquitto Add-on             │  │
│  │   (MQTT Broker)                │  │
│  └────────────────────────────────┘  │
└──────────────────────────────────────┘
            ↑
        MQTT Images
            │
    ┌───────┴────────┐
    │   ESP32-CAM    │
    └────────────────┘
```

### Standalone Docker
```
┌─────────────────┐    MQTT     ┌──────────────┐
│   ESP32-CAM     │ ──────────> │ MQTT Broker  │
└─────────────────┘             └──────────────┘
                                      ↓
                          ┌─────────────────────┐
                          │  MeterMonitor       │
                          │  Docker Container   │
                          │  Port: 8070         │
                          │  Volume: /data      │
                          └─────────────────────┘
```

### Python Development
```
┌─────────────────────────┐
│  Development Machine    │
│  - run.py               │
│  - Uvicorn server       │
│  - Hot reload enabled   │
│  - Local SQLite         │
└─────────────────────────┘
         ↑
    HTTP/MQTT
         │
   Testing Tools
```

## Future Enhancements

Potential architectural improvements:

1. **Message Queue**: Replace direct MQTT with Redis/RabbitMQ for better scalability
2. **Distributed Processing**: Worker pools for parallel image processing
3. **TimeSeries DB**: InfluxDB for better history query performance
4. **Model Registry**: Versioned model management with A/B testing
5. **WebSocket Updates**: Real-time frontend updates instead of polling
6. **Multi-tenancy**: Support for multiple users with isolation
7. **Cloud Storage**: S3/MinIO for image archival

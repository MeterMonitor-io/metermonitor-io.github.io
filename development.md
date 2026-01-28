# MeterMonitor Development Guide

Complete guide for developers contributing to MeterMonitor.

## Table of Contents

1. [Development Environment](#development-environment)
2. [Project Structure](#project-structure)
3. [Code Style](#code-style)
4. [Testing](#testing)
5. [Building](#building)
6. [Contributing](#contributing)
7. [Custom ROI Extractors](#custom-roi-extractors)
8. [API Extensions](#api-extensions)
9. [Frontend Development](#frontend-development)

---

## Development Environment

### Prerequisites

**Required**:
- Python 3.8+ (3.10+ recommended)
- Node.js 16+ and Yarn (for frontend)
- Git
- SQLite3

**Recommended**:
- VS Code or PyCharm
- Docker (for testing builds)
- MQTT broker (Mosquitto) for testing

### Initial Setup

**1. Clone Repository**:
```bash
git clone https://github.com/phiph-s/metermonitor-managementserver.git
cd metermonitor-managementserver
```

**2. Python Environment**:
```bash
# Create virtual environment
python3 -m venv .venv
source .venv/bin/activate  # Linux/Mac
# or
.venv\Scripts\activate  # Windows

# Install dependencies
pip install -r requirements.txt
```

**3. Frontend Setup**:
```bash
cd frontend
yarn install
```

**4. Configuration**:
```bash
# Copy example settings
cp settings.json.example settings.json

# Edit settings.json with your MQTT broker details
```

**5. Database Setup**:
```bash
# Database created automatically on first run
# Located at: data/watermeters.sqlite
```

### IDE Configuration

**VS Code** (`.vscode/settings.json`):
```json
{
  "python.pythonPath": ".venv/bin/python",
  "python.linting.enabled": true,
  "python.linting.pylintEnabled": false,
  "python.linting.flake8Enabled": true,
  "python.formatting.provider": "black",
  "editor.formatOnSave": true,
  "[python]": {
    "editor.rulers": [120]
  }
}
```

**PyCharm**:
1. Open project
2. File → Settings → Project → Python Interpreter
3. Add interpreter → Existing environment → Select `.venv/bin/python`
4. Enable PEP 8 inspections

---

## Project Structure

```
metermonitor-managementserver/
├── run.py                          # Main entry point
├── requirements.txt                # Python dependencies
├── settings.json                   # Standalone configuration
├── config.json                     # Home Assistant add-on config
├── ha_default_settings.json        # HA-specific defaults
├── Dockerfile                      # Docker build definition
├── .dockerignore                   # Docker ignore rules
├── .gitignore                      # Git ignore rules
├── README.md                       # Project readme
├── CHANGELOG.md                    # Version history
│
├── db/                             # Database layer
│   └── migrations.py               # Database migrations
│
├── lib/                            # Core backend logic
│   ├── functions.py                # Core processing functions
│   ├── history_correction.py      # Correctional algorithm
│   ├── http_server.py              # FastAPI REST API
│   ├── mqtt_handler.py             # MQTT client
│   ├── polling_handler.py          # Polling service
│   ├── capture_utils.py            # Image capture utilities
│   ├── ha_auth.py                  # Home Assistant authentication
│   ├── ha_flash_suggestion.py      # Flash entity suggestion
│   ├── global_alerts.py            # Alert management
│   ├── threshold_optimizer.py      # Threshold search
│   ├── model_singleton.py          # Model instance management
│   │
│   └── meter_processing/           # Image processing
│       ├── meter_processing.py     # Main processing pipeline
│       ├── loss_fn.py              # Loss functions (legacy)
│       │
│       └── roi_extractors/         # ROI extraction modules
│           ├── __init__.py         # Extractor exports
│           ├── base.py             # Base classes
│           ├── yolo_extractor.py   # YOLO-based extraction
│           ├── orb_extractor.py    # ORB-based extraction
│           └── bypass_extractor.py # Bypass (no extraction)
│
├── models/                         # AI models
│   ├── yolo-best-obb-2.onnx       # YOLO detection model
│   └── best_model.onnx            # Digit classifier model
│
├── frontend/                       # Vue 3 web interface
│   ├── src/
│   │   ├── App.vue                # Root component
│   │   ├── main.js                # Entry point
│   │   ├── router.js              # Vue Router config
│   │   ├── components/            # Vue components
│   │   ├── views/                 # Page views
│   │   └── assets/                # Static assets
│   │
│   ├── public/                    # Public files
│   ├── dist/                      # Build output (generated)
│   ├── package.json               # npm dependencies
│   ├── yarn.lock                  # Yarn lock file
│   └── vite.config.js             # Vite build config
│
├── tools/                         # Utility scripts
│   ├── mqtt_bulk_sender.py        # Bulk MQTT sender
│   └── mqtt_image_collector.py    # Image collection tool
│
├── docs/                          # Documentation
│   ├── user-guide.md
│   ├── api-reference.md
│   ├── esp32-setup.md
│   ├── roi-extractors.md
│   ├── troubleshooting.md
│   ├── database-schema.md
│   ├── development.md             # This file
│   └── advanced-topics.md
│
└── translations/                  # i18n (add-on only)
    ├── en.yaml
    └── de.yaml
```

### Key Modules

**Backend**:
- `run.py`: Initializes MQTT, polling, HTTP server
- `lib/http_server.py`: FastAPI endpoints
- `lib/mqtt_handler.py`: MQTT message processing
- `lib/meter_processing/meter_processing.py`: Image processing pipeline
- `lib/history_correction.py`: Correctional algorithm (full/light modes)

**Frontend**:
- `frontend/src/main.js`: Vue app initialization
- `frontend/src/router.js`: Route definitions
- `frontend/src/components/`: Reusable components
- `frontend/src/views/`: Page views (Watermeters, Discovery, Sources, etc.)

**Database**:
- `db/migrations.py`: Schema creation and migration logic
- SQLite database: `data/watermeters.sqlite`

---

## Code Style

### Python

**Style Guide**: PEP 8 with modifications

**Line Length**: 120 characters (not 79)

**Formatting**: Use `black` formatter
```bash
black lib/ db/ run.py
```

**Linting**: Use `flake8`
```bash
flake8 lib/ db/ run.py --max-line-length=120
```

**Type Hints**: Encouraged but not required
```python
def process_image(img: np.ndarray, threshold: int = 100) -> tuple[list, float]:
    pass
```

**Docstrings**: Use for public functions
```python
def reevaluate_latest_picture(db_file: str, name: str, predictor, config, **kwargs):
    """
    Re-evaluate the latest picture for a watermeter.

    Args:
        db_file: Path to SQLite database
        name: Watermeter name
        predictor: MeterPredictor instance
        config: Configuration dict
        **kwargs: Additional options

    Returns:
        Tuple of (target_brightness, confidence, bbox_image_base64)
    """
    pass
```

**Imports**: Group by standard, third-party, local
```python
import os
import sqlite3
from datetime import datetime

import cv2
import numpy as np
from PIL import Image

from lib.functions import reevaluate_latest_picture
from lib.model_singleton import get_meter_predictor
```

### JavaScript/Vue

**Style Guide**: Airbnb JavaScript Style Guide (relaxed)

**Formatting**: Use Prettier
```bash
cd frontend
yarn prettier --write "src/**/*.{js,vue}"
```

**Linting**: ESLint (if configured)
```bash
yarn lint
```

**Component Structure**:
```vue
<template>
  <div class="component-name">
    <!-- Template -->
  </div>
</template>

<script>
export default {
  name: 'ComponentName',
  props: {
    propName: {
      type: String,
      required: true
    }
  },
  data() {
    return {
      localState: null
    }
  },
  methods: {
    methodName() {
      // Method implementation
    }
  }
}
</script>

<style scoped>
.component-name {
  /* Scoped styles */
}
</style>
```

---

## Testing

### Manual Testing

**1. Start Development Server**:
```bash
# Terminal 1: Backend
python run.py

# Terminal 2: Frontend (dev mode)
cd frontend
yarn dev
```

**2. Access UI**:
- Backend API: http://localhost:8070/api/
- Frontend Dev: http://localhost:5173/ (Vite dev server)
- API Docs: http://localhost:8070/docs (FastAPI auto-generated)

**3. Test MQTT**:
```bash
# Send test message
mosquitto_pub -h localhost -t "MeterMonitor/test_meter" \
  -m '{"name":"test_meter","picture_number":1,"WiFi-RSSI":-45,"picture":{"timestamp":"2026-01-28T10:00:00","format":"jpg","width":640,"height":480,"length":1234,"data":"base64_encoded_image"}}'
```

### Automated Testing

**Unit Tests** (not yet implemented):
```bash
pytest tests/
```

**Integration Tests** (not yet implemented):
```bash
pytest tests/integration/
```

### Test Coverage

**Coverage Tool**:
```bash
pip install pytest-cov
pytest --cov=lib tests/
```

**Target Coverage**: 80%+ (goal, not yet achieved)

---

## Building

### Frontend Build

```bash
cd frontend
yarn build
```

Output: `frontend/dist/` (static files served by FastAPI)

### Docker Build

**Local Build**:
```bash
docker build -t metermonitor:dev .
```

**Multi-Platform Build**:
```bash
docker buildx create --use
docker buildx build --platform linux/amd64,linux/arm64 \
  -t metermonitor:dev .
```

**Test Docker Image**:
```bash
docker run -d -p 8070:8070 \
  -v $(pwd)/data:/data \
  -e MQTT_BROKER=host.docker.internal \
  metermonitor:dev
```

### Home Assistant Add-on Build

**Local Testing**:
1. Copy repository to HA add-ons folder:
   ```bash
   cp -r . /usr/share/hassio/addons/local/metermonitor/
   ```
2. Rebuild add-on in HA UI
3. Test changes

**CI/CD** (GitHub Actions):
- `.github/workflows/` contains build workflows
- Automatic builds on push to main
- Publishes to GitHub Container Registry

---

## Contributing

### Contribution Workflow

**1. Fork Repository**:
- Visit: https://github.com/phiph-s/metermonitor-managementserver
- Click **Fork**

**2. Create Branch**:
```bash
git checkout -b feature/your-feature-name
# or
git checkout -b fix/issue-number-description
```

**3. Make Changes**:
- Follow code style guidelines
- Add tests if applicable
- Update documentation

**4. Commit**:
```bash
git add .
git commit -m "feat: Add new ROI extractor based on SIFT

- Implement SIFTExtractor class
- Add configuration options
- Update documentation

Closes #123"
```

**Commit Message Format**:
```
<type>(<scope>): <subject>

<body>

<footer>
```

**Types**:
- `feat`: New feature
- `fix`: Bug fix
- `docs`: Documentation only
- `style`: Code style (formatting, no logic change)
- `refactor`: Code refactoring
- `perf`: Performance improvement
- `test`: Adding tests
- `chore`: Maintenance tasks

**5. Push and Create PR**:
```bash
git push origin feature/your-feature-name
```
- Open PR on GitHub
- Fill out PR template
- Link related issues

### PR Review Process

1. **Automated Checks**:
   - Docker build passes
   - Linting passes (if configured)

2. **Code Review**:
   - Maintainer reviews changes
   - May request modifications

3. **Testing**:
   - Test in local environment
   - Verify no regressions

4. **Merge**:
   - Squash and merge (typically)
   - Update CHANGELOG.md

---

## Custom ROI Extractors

### Creating a Custom Extractor

**1. Implement Base Class**:

```python
# lib/meter_processing/roi_extractors/custom_extractor.py

from lib.meter_processing.roi_extractors.base import ROIExtractor
import cv2
import numpy as np

class CustomExtractor(ROIExtractor):
    """
    Custom ROI extractor using [your method].
    """

    def __init__(self, config_dict=None):
        super().__init__()
        self.config = config_dict or {}
        # Initialize your extractor

    def extract(self, input_image):
        """
        Extract display region from image.

        Args:
            input_image: PIL.Image or numpy array

        Returns:
            Tuple of:
            - rotated_cropped_img: Display region (numpy array, BGR)
            - rotated_cropped_img_ext: Extended region (or None)
            - boundingboxed_image: Original with bbox drawn (base64)
        """
        # Convert PIL to numpy if needed
        if hasattr(input_image, 'convert'):
            input_image = cv2.cvtColor(np.array(input_image), cv2.COLOR_RGB2BGR)

        # Your extraction logic here
        # ...

        # Example: Return full image (bypass behavior)
        return input_image, None, None
```

**2. Register Extractor**:

```python
# lib/meter_processing/roi_extractors/__init__.py

from .yolo_extractor import YOLOExtractor
from .orb_extractor import ORBExtractor
from .bypass_extractor import BypassExtractor
from .custom_extractor import CustomExtractor  # Add import

__all__ = [
    'YOLOExtractor',
    'ORBExtractor',
    'BypassExtractor',
    'CustomExtractor'  # Export
]
```

**3. Integrate into Pipeline**:

```python
# lib/meter_processing/meter_processing.py

# In extract_display_and_segment method:
if roi_extractor == "custom":
    extractor = CustomExtractor(config_dict)
```

**4. Add API Support**:

```python
# lib/http_server.py

# Update SettingsRequest model:
class SettingsRequest(BaseModel):
    # ...
    roi_extractor: Optional[str] = None  # Allow "custom"
```

**5. Test**:
```python
# Test script
from lib.meter_processing.meter_processing import MeterPredictor
from lib.meter_processing.roi_extractors import CustomExtractor
from PIL import Image

predictor = MeterPredictor()
extractor = CustomExtractor()
image = Image.open('test_image.jpg')

# Test extraction
result = extractor.extract(image)
print(f"Extracted region shape: {result[0].shape}")
```

### Extractor Best Practices

- **Error Handling**: Set `self.last_error` on failure
- **Performance**: Optimize for <500ms extraction time
- **Robustness**: Handle various image sizes and qualities
- **Documentation**: Docstrings for all public methods
- **Configuration**: Use `config_dict` for parameters

---

## API Extensions

### Adding New Endpoints

**1. Define Model** (if needed):
```python
# lib/http_server.py

class NewFeatureRequest(BaseModel):
    param1: str
    param2: int
    optional_param: Optional[bool] = False
```

**2. Implement Endpoint**:
```python
@app.post("/api/new-feature", dependencies=[Depends(authenticate)])
def new_feature_endpoint(payload: NewFeatureRequest):
    """
    Description of what this endpoint does.
    """
    # Validate input
    if payload.param2 < 0:
        raise HTTPException(status_code=400, detail="param2 must be positive")

    # Database access
    db = db_connection()
    cursor = db.cursor()

    # Business logic
    result = process_new_feature(payload.param1, payload.param2)

    # Return response
    return {"status": "success", "result": result}
```

**3. Document Endpoint**:
- Update `docs/api-reference.md`
- FastAPI auto-generates docs at `/docs`

**4. Add Frontend Support** (if needed):
```javascript
// frontend/src/components/NewFeature.vue

async submitNewFeature() {
  try {
    const response = await fetch('/api/new-feature', {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
        'Secret': this.secretKey
      },
      body: JSON.stringify({
        param1: this.param1,
        param2: this.param2
      })
    });

    if (!response.ok) throw new Error('Request failed');
    const data = await response.json();
    console.log('Success:', data);
  } catch (error) {
    console.error('Error:', error);
  }
}
```

---

## Frontend Development

### Development Server

```bash
cd frontend
yarn dev
```

- Runs on http://localhost:5173
- Hot module replacement (HMR)
- Proxies API requests to backend

### Build for Production

```bash
yarn build
```

Output: `frontend/dist/`

### Component Development

**Create New Component**:
```vue
<!-- frontend/src/components/NewComponent.vue -->

<template>
  <div class="new-component">
    <h2>{{ title }}</h2>
    <button @click="handleClick">Click Me</button>
  </div>
</template>

<script>
export default {
  name: 'NewComponent',
  props: {
    title: {
      type: String,
      required: true
    }
  },
  methods: {
    handleClick() {
      this.$emit('clicked', 'Button was clicked');
    }
  }
}
</script>

<style scoped>
.new-component {
  padding: 20px;
}
</style>
```

**Use in View**:
```vue
<!-- frontend/src/views/NewView.vue -->

<template>
  <div>
    <NewComponent title="My Title" @clicked="onClicked" />
  </div>
</template>

<script>
import NewComponent from '@/components/NewComponent.vue'

export default {
  components: {
    NewComponent
  },
  methods: {
    onClicked(message) {
      console.log(message);
    }
  }
}
</script>
```

### State Management

Currently uses **component state** (no Vuex/Pinia).

For shared state, consider:
- Props and events (current approach)
- Composables (Vue 3 Composition API)
- Pinia (if complexity grows)

### API Client

**Fetch API** used directly (no Axios).

**Example**:
```javascript
async fetchWatermeters() {
  const response = await fetch('/api/watermeters');
  const data = await response.json();
  this.watermeters = data.watermeters;
}
```

---

## Additional Resources

- [User Guide](user-guide.md)
- [API Reference](api-reference.md)
- [Database Schema](database-schema.md)
- [Troubleshooting](troubleshooting.md)
- [Advanced Topics](advanced-topics.md)

### External Documentation

- [FastAPI](https://fastapi.tiangolo.com/)
- [Vue 3](https://vuejs.org/)
- [ONNX Runtime](https://onnxruntime.ai/)
- [OpenCV Python](https://docs.opencv.org/4.x/d6/d00/tutorial_py_root.html)
- [SQLite](https://www.sqlite.org/docs.html)

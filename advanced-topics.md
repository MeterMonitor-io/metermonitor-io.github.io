# Advanced Topics

Advanced configuration, deployment strategies, and optimization techniques for MeterMonitor.

## Table of Contents

1. [Custom Correction Algorithms](#custom-correction-algorithms)
2. [Multi-Instance Deployment](#multi-instance-deployment)
3. [External Database (PostgreSQL)](#external-database-postgresql)
4. [Reverse Proxy Configuration](#reverse-proxy-configuration)
5. [TLS/SSL for MQTT](#tlsssl-for-mqtt)
6. [Performance Tuning](#performance-tuning)
7. [Backup Strategies](#backup-strategies)
8. [Model Retraining](#model-retraining)

---

## Custom Correction Algorithms

### Understanding Correction Modes

MeterMonitor includes two built-in correction modes:

**Full Mode** (default):
- Comprehensive validation and correction
- Positive flow check (consumption only increases)
- Max flow rate validation
- Fallback to lower-ranked predictions
- Handles digit roll-over

**Light Mode**:
- Minimal correction
- Only replaces rotation class and denied digits
- No flow rate validation
- Faster processing

### Implementing Custom Correction

Create custom correction logic by extending `/lib/history_correction.py`:

```python
# lib/history_correction_custom.py

import sqlite3
from datetime import datetime

def custom_correct_value(db_file: str, name: str, new_eval, config):
    """
    Custom correction algorithm.

    Args:
        db_file: Database path
        name: Meter name
        new_eval: Evaluation tuple (colored, th, predictions, timestamp, result, ...)
        config: Configuration dict

    Returns:
        dict with:
            - accepted: bool
            - value: int or None
            - total_confidence: float
            - used_confidence: float
            - [custom metadata fields]
    """
    with sqlite3.connect(db_file) as conn:
        cursor = conn.cursor()

        # Get last reading
        cursor.execute(
            "SELECT value, timestamp FROM history WHERE name = ? ORDER BY ROWID DESC LIMIT 1",
            (name,)
        )
        last_row = cursor.fetchone()

        if not last_row:
            return {
                "accepted": False,
                "value": None,
                "total_confidence": 0.0,
                "used_confidence": 0.0,
                "rejection_reason": "no_history"
            }

        last_value = last_row[0]
        last_time = datetime.fromisoformat(last_row[1])
        new_time = datetime.fromisoformat(new_eval[3])

        # Time validation
        time_diff_min = (new_time - last_time).total_seconds() / 60.0
        if time_diff_min <= 0:
            return {
                "accepted": False,
                "value": None,
                "total_confidence": 0.0,
                "used_confidence": 0.0,
                "rejection_reason": "invalid_time"
            }

        # Your custom logic here
        predictions = new_eval[2]
        corrected_value = ""
        total_conf = 1.0

        for i, pred_list in enumerate(predictions):
            if len(pred_list) == 0:
                return {
                    "accepted": False,
                    "value": None,
                    "total_confidence": 0.0,
                    "used_confidence": 0.0,
                    "rejection_reason": "empty_prediction"
                }

            # Example: Always use top prediction
            digit, conf = pred_list[0]
            corrected_value += digit
            total_conf *= conf

        # Custom validation (example: only accept monotonic increase)
        new_value = int(corrected_value)
        if new_value < last_value:
            return {
                "accepted": False,
                "value": None,
                "total_confidence": total_conf,
                "used_confidence": total_conf,
                "rejection_reason": "non_monotonic"
            }

        return {
            "accepted": True,
            "value": new_value,
            "total_confidence": total_conf,
            "used_confidence": total_conf,
            "custom_field": "custom_value"  # Add your metadata
        }
```

**Integrate Custom Correction**:

```python
# lib/functions.py

from lib.history_correction import correct_value
from lib.history_correction_custom import custom_correct_value

def reevaluate_latest_picture(db_file, name, predictor, config, **kwargs):
    # ... existing code ...

    # Use custom correction
    use_custom = config.get('use_custom_correction', False)

    if use_custom:
        correction_result = custom_correct_value(db_file, name, new_eval, config)
    else:
        use_full = settings[14]  # use_correctional_alg
        correction_result = correct_value(
            db_file, name, new_eval,
            allow_negative_correction=config.get('allow_negative_correction', False),
            max_flow_rate=max_flow_rate,
            use_full_correction=use_full
        )

    # ... rest of code ...
```

**Configuration**:
```json
{
  "use_custom_correction": true
}
```

### Correction Algorithm Tips

**Considerations**:
- Always validate time difference (positive, reasonable)
- Handle empty predictions gracefully
- Consider confidence thresholds
- Log rejection reasons for debugging
- Test with historical data

**Common Patterns**:
- **Weighted predictions**: Use confidence-weighted average
- **Temporal smoothing**: Apply moving average over time
- **External validation**: Query external API for plausibility
- **Machine learning**: Train model to predict corrections

---

## Multi-Instance Deployment

### Running Multiple MeterMonitor Instances

**Use Cases**:
- High availability (failover)
- Load distribution
- Geographic distribution
- Testing/production separation

### Shared Database Approach

**Architecture**:
```
┌──────────────┐     ┌──────────────┐
│ Instance 1   │────▶│   Shared     │
│ (Server A)   │     │   Database   │
└──────────────┘     │  (Network)   │
                     │              │
┌──────────────┐     │              │
│ Instance 2   │────▶│              │
│ (Server B)   │     └──────────────┘
└──────────────┘
```

**Limitations with SQLite**:
- SQLite doesn't support concurrent writes well
- Database locking issues likely
- **Not recommended** for production

### Separate Database Approach (Recommended)

**Architecture**:
```
┌──────────────┐     ┌──────────────┐
│ Instance 1   │────▶│   DB 1       │
│ (Meter A,B)  │     │              │
└──────────────┘     └──────────────┘

┌──────────────┐     ┌──────────────┐
│ Instance 2   │────▶│   DB 2       │
│ (Meter C,D)  │     │              │
└──────────────┘     └──────────────┘
```

**Configuration**:

Instance 1:
```json
{
  "dbfile": "/data/metermonitor_instance1.db",
  "mqtt": {
    "topic": "MeterMonitor/instance1/#"
  },
  "http": {
    "port": 8070
  }
}
```

Instance 2:
```json
{
  "dbfile": "/data/metermonitor_instance2.db",
  "mqtt": {
    "topic": "MeterMonitor/instance2/#"
  },
  "http": {
    "port": 8071
  }
}
```

**Benefits**:
- No database conflicts
- Independent scaling
- Isolated failures

**Drawbacks**:
- Separate UIs
- Duplicate model loading (memory)

### Load Balancer Approach

**Not recommended** due to:
- Stateful processing (image→evaluation pipeline)
- Database locking with shared SQLite
- Model loading overhead

If needed, use:
- Separate instances per meter group
- Route by meter name in load balancer
- Sticky sessions (not helpful here)

---

## External Database (PostgreSQL)

### Migrating to PostgreSQL

**Why PostgreSQL**:
- Better concurrent access
- Advanced features (replication, clustering)
- Larger dataset support
- Network accessibility

**Challenges**:
- MeterMonitor is SQLite-specific
- Schema conversion required
- Connection pooling needed
- **Not officially supported**

### DIY PostgreSQL Migration

**1. Schema Conversion**:

```sql
-- PostgreSQL schema (example)
CREATE TABLE watermeters (
    name TEXT PRIMARY KEY,
    picture_number INTEGER NOT NULL,
    wifi_rssi INTEGER NOT NULL,
    picture_format TEXT NOT NULL,
    picture_timestamp TIMESTAMP NOT NULL,
    picture_width INTEGER NOT NULL,
    picture_height INTEGER NOT NULL,
    picture_length INTEGER NOT NULL,
    picture_data TEXT NOT NULL,
    setup BOOLEAN DEFAULT FALSE,
    picture_data_bbox BYTEA
);

-- Repeat for other tables...
```

**2. Code Changes**:

Replace all `sqlite3.connect()` with PostgreSQL connector:

```python
# Before (SQLite)
import sqlite3
conn = sqlite3.connect(db_file)

# After (PostgreSQL)
import psycopg2
conn = psycopg2.connect(
    host="postgres_host",
    database="metermonitor",
    user="postgres",
    password="password"
)
```

**3. Query Adjustments**:

- `?` placeholders → `%s` (PostgreSQL style)
- `AUTOINCREMENT` → `SERIAL`
- `datetime('now')` → `NOW()`
- `PRAGMA` commands → PostgreSQL equivalents

**4. Migration Script**:

```python
# migrate_to_postgres.py

import sqlite3
import psycopg2

# Export SQLite data
sqlite_conn = sqlite3.connect('watermeters.sqlite')
sqlite_cursor = sqlite_conn.cursor()

# Connect to PostgreSQL
pg_conn = psycopg2.connect("host=localhost dbname=metermonitor user=postgres password=password")
pg_cursor = pg_conn.cursor()

# Migrate each table
tables = ['watermeters', 'settings', 'history', 'evaluations', 'sources', 'templates']

for table in tables:
    sqlite_cursor.execute(f"SELECT * FROM {table}")
    rows = sqlite_cursor.fetchall()

    for row in rows:
        placeholders = ','.join(['%s'] * len(row))
        pg_cursor.execute(f"INSERT INTO {table} VALUES ({placeholders})", row)

pg_conn.commit()
```

**Not officially supported**: This is advanced DIY territory.

---

## Reverse Proxy Configuration

### Why Use a Reverse Proxy

- **SSL/TLS termination**
- **Authentication layer**
- **Rate limiting**
- **Caching**
- **Load balancing** (if multi-instance)

### NGINX Configuration

**Basic Proxy**:

```nginx
# /etc/nginx/sites-available/metermonitor

server {
    listen 80;
    server_name metermonitor.example.com;

    location / {
        proxy_pass http://localhost:8070;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        # WebSocket support (if needed in future)
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
    }
}
```

**With SSL (Let's Encrypt)**:

```nginx
server {
    listen 443 ssl http2;
    server_name metermonitor.example.com;

    ssl_certificate /etc/letsencrypt/live/metermonitor.example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/metermonitor.example.com/privkey.pem;

    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers HIGH:!aNULL:!MD5;

    location / {
        proxy_pass http://localhost:8070;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto https;
    }
}

# Redirect HTTP to HTTPS
server {
    listen 80;
    server_name metermonitor.example.com;
    return 301 https://$host$request_uri;
}
```

**Enable Configuration**:
```bash
sudo ln -s /etc/nginx/sites-available/metermonitor /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl reload nginx
```

### Caddy Configuration

**Simpler alternative** with automatic HTTPS:

```caddy
# /etc/caddy/Caddyfile

metermonitor.example.com {
    reverse_proxy localhost:8070
}
```

Caddy automatically:
- Obtains SSL certificate (Let's Encrypt)
- Redirects HTTP to HTTPS
- Renews certificates

**Start Caddy**:
```bash
sudo systemctl start caddy
sudo systemctl enable caddy
```

### Authentication Layer

**NGINX Basic Auth**:

```nginx
location / {
    auth_basic "MeterMonitor";
    auth_basic_user_file /etc/nginx/.htpasswd;

    proxy_pass http://localhost:8070;
    # ... other proxy settings ...
}
```

**Create Password File**:
```bash
sudo apt-get install apache2-utils
sudo htpasswd -c /etc/nginx/.htpasswd admin
```

**Note**: This is in addition to MeterMonitor's built-in authentication.

---

## TLS/SSL for MQTT

### Securing MQTT Communication

**Why TLS for MQTT**:
- Encrypted data transmission
- Authentication of broker
- Protection against eavesdropping

### Mosquitto TLS Configuration

**1. Generate Certificates**:

```bash
# Using Let's Encrypt (if broker has public domain)
sudo certbot certonly --standalone -d mqtt.example.com

# Or self-signed (for local network)
openssl req -new -x509 -days 3650 -extensions v3_ca \
  -keyout ca.key -out ca.crt

openssl genrsa -out server.key 2048
openssl req -new -out server.csr -key server.key
openssl x509 -req -in server.csr -CA ca.crt -CAkey ca.key \
  -CAcreateserial -out server.crt -days 3650
```

**2. Configure Mosquitto**:

```conf
# /etc/mosquitto/conf.d/tls.conf

listener 8883
protocol mqtt

cafile /etc/mosquitto/certs/ca.crt
certfile /etc/mosquitto/certs/server.crt
keyfile /etc/mosquitto/certs/server.key

require_certificate false
use_identity_as_username false

# Optional: Disable unencrypted port
# listener 1883
# protocol mqtt
```

**3. Restart Mosquitto**:
```bash
sudo systemctl restart mosquitto
```

**4. Update MeterMonitor Configuration**:

```json
{
  "mqtt": {
    "broker": "mqtt.example.com",
    "port": 8883,
    "topic": "MeterMonitor/#",
    "username": "esp",
    "password": "esp",
    "tls": {
      "enabled": true,
      "ca_certs": "/path/to/ca.crt",
      "certfile": null,
      "keyfile": null
    }
  }
}
```

**Note**: TLS configuration requires code changes to `lib/mqtt_handler.py`:

```python
# lib/mqtt_handler.py

if config.get('mqtt', {}).get('tls', {}).get('enabled'):
    import ssl
    tls_config = config['mqtt']['tls']
    self.client.tls_set(
        ca_certs=tls_config.get('ca_certs'),
        certfile=tls_config.get('certfile'),
        keyfile=tls_config.get('keyfile'),
        cert_reqs=ssl.CERT_REQUIRED,
        tls_version=ssl.PROTOCOL_TLS
    )
```

**Currently not implemented** in MeterMonitor. Contribution welcome!

---

## Performance Tuning

### Optimizing ONNX Inference

**1. Use Optimized ONNX Runtime**:

```bash
# CPU optimized
pip install onnxruntime

# GPU (requires CUDA)
pip uninstall onnxruntime
pip install onnxruntime-gpu
```

**2. Session Options**:

```python
# lib/meter_processing/meter_processing.py

sess_options = ort.SessionOptions()
sess_options.graph_optimization_level = ort.GraphOptimizationLevel.ORT_ENABLE_ALL
sess_options.intra_op_num_threads = 4  # Adjust based on CPU cores
sess_options.inter_op_num_threads = 2
```

**3. Model Quantization** (advanced):

```python
# Quantize ONNX model (INT8)
import onnxruntime.quantization

onnxruntime.quantization.quantize_dynamic(
    'best_model.onnx',
    'best_model_quantized.onnx',
    weight_type=onnxruntime.quantization.QuantType.QUInt8
)
```

Smaller model, faster inference, slight accuracy loss.

### Database Optimization

**1. Enable WAL Mode** (Write-Ahead Logging):

```sql
PRAGMA journal_mode=WAL;
```

Benefits:
- Concurrent reads while writing
- Better performance
- Reduced locking

**2. Increase Cache Size**:

```sql
PRAGMA cache_size=-64000;  -- 64MB cache
```

**3. Create Indexes**:

```sql
CREATE INDEX idx_history_name_timestamp ON history(name, timestamp);
CREATE INDEX idx_evals_name_id ON evaluations(name, id DESC);
CREATE INDEX idx_sources_enabled ON sources(enabled);
```

**4. Regular Maintenance**:

```sql
VACUUM;
ANALYZE;
```

### Image Processing Optimization

**1. Reduce Resolution** (ESP32):

```yaml
esp32_camera:
  resolution: 640x480  # Lower than 800x600 or 1024x768
```

**2. Adjust JPEG Quality**:

```yaml
esp32_camera:
  jpeg_quality: 10  # Balance quality vs size
```

**3. ROI Extractor Choice**:
- YOLO: Faster (~100-200ms)
- ORB: Slower (~200-400ms)

**4. Batch Processing** (not implemented):

Process multiple images in parallel using threading/multiprocessing.

### System-Level Tuning

**1. CPU Governor**:

```bash
# Set to performance mode
echo performance | sudo tee /sys/devices/system/cpu/cpu*/cpufreq/scaling_governor
```

**2. Swap Configuration**:

```bash
# Reduce swappiness (prefer RAM)
sudo sysctl vm.swappiness=10
```

**3. Docker Resource Limits**:

```yaml
# docker-compose.yml
services:
  metermonitor:
    image: metermonitor:latest
    deploy:
      resources:
        limits:
          cpus: '2'
          memory: 2G
        reservations:
          cpus: '1'
          memory: 1G
```

---

## Backup Strategies

### Automated Backup Scripts

**Daily Backup with Rotation**:

```bash
#!/bin/bash
# /usr/local/bin/metermonitor_backup.sh

BACKUP_DIR="/backups/metermonitor"
DB_PATH="/data/watermeters.sqlite"
RETENTION_DAYS=30

# Create backup directory
mkdir -p "$BACKUP_DIR"

# Backup filename with timestamp
BACKUP_FILE="$BACKUP_DIR/watermeters_$(date +%Y%m%d_%H%M%S).sqlite"

# Perform backup
sqlite3 "$DB_PATH" ".backup '$BACKUP_FILE'"

# Compress backup
gzip "$BACKUP_FILE"

# Delete old backups
find "$BACKUP_DIR" -name "watermeters_*.sqlite.gz" -mtime +$RETENTION_DAYS -delete

echo "Backup completed: ${BACKUP_FILE}.gz"
```

**Cron Job**:
```bash
# Run daily at 2 AM
0 2 * * * /usr/local/bin/metermonitor_backup.sh >> /var/log/metermonitor_backup.log 2>&1
```

### Off-Site Backup

**Using Rclone** (to cloud storage):

```bash
# Install rclone
curl https://rclone.org/install.sh | sudo bash

# Configure remote (interactive)
rclone config

# Sync backups to cloud
rclone sync /backups/metermonitor remote:metermonitor_backups --log-file=/var/log/rclone.log
```

**Automate with Cron**:
```bash
# Upload to cloud daily at 3 AM
0 3 * * * rclone sync /backups/metermonitor remote:metermonitor_backups --log-file=/var/log/rclone.log
```

### Disaster Recovery Plan

**1. Document Configuration**:
- Save `settings.json` or add-on configuration
- Export meter settings via API
- Document camera positions/templates

**2. Test Restores**:
- Regularly test backup restoration
- Verify data integrity after restore

**3. Recovery Steps**:
```bash
# 1. Stop MeterMonitor
sudo systemctl stop metermonitor

# 2. Restore database
cp /backups/metermonitor/watermeters_20260128.sqlite /data/watermeters.sqlite

# 3. Restore configuration
cp /backups/metermonitor/settings.json /path/to/settings.json

# 4. Start MeterMonitor
sudo systemctl start metermonitor
```

---

## Model Retraining

### When to Retrain Models

**YOLO Detection Model**:
- Add support for new meter types
- Improve detection accuracy
- Handle unusual layouts

**Digit Classifier**:
- Different digit fonts/styles
- Improve recognition accuracy
- Reduce misclassifications

### Collecting Training Data

**1. Export Datasets** (via API):

```bash
curl -O http://localhost:8070/api/dataset/kitchen_meter/download
```

**2. Manual Labeling**:
- Review exported digit images
- Correct mislabeled data
- Add manual labels

**3. Data Augmentation**:
- Rotation, scaling, noise
- Brightness adjustments
- Synthetic data generation

### Retraining Digit Classifier

**1. Prepare Dataset**:

```
training_data/
├── 0/
│   ├── img001.png
│   ├── img002.png
│   └── ...
├── 1/
│   └── ...
└── r/  # Rotation class
    └── ...
```

**2. Training Script** (example with TensorFlow/Keras):

```python
# train_classifier.py

import tensorflow as tf
from tensorflow import keras
import numpy as np
from pathlib import Path

# Load data
def load_dataset(data_dir):
    images = []
    labels = []
    class_names = sorted([d.name for d in Path(data_dir).iterdir() if d.is_dir()])

    for class_idx, class_name in enumerate(class_names):
        class_dir = Path(data_dir) / class_name
        for img_path in class_dir.glob('*.png'):
            img = tf.keras.preprocessing.image.load_img(img_path, target_size=(28, 28), color_mode='grayscale')
            img_array = tf.keras.preprocessing.image.img_to_array(img) / 255.0
            images.append(img_array)
            labels.append(class_idx)

    return np.array(images), np.array(labels), class_names

X_train, y_train, classes = load_dataset('training_data')

# Build model
model = keras.Sequential([
    keras.layers.Conv2D(32, (3, 3), activation='relu', input_shape=(28, 28, 1)),
    keras.layers.MaxPooling2D((2, 2)),
    keras.layers.Conv2D(64, (3, 3), activation='relu'),
    keras.layers.MaxPooling2D((2, 2)),
    keras.layers.Flatten(),
    keras.layers.Dense(128, activation='relu'),
    keras.layers.Dropout(0.5),
    keras.layers.Dense(len(classes), activation='softmax')
])

model.compile(optimizer='adam', loss='sparse_categorical_crossentropy', metrics=['accuracy'])

# Train
model.fit(X_train, y_train, epochs=50, validation_split=0.2, batch_size=32)

# Export to ONNX
import tf2onnx

spec = (tf.TensorSpec((None, 28, 28, 1), tf.float32, name="input"),)
output_path = "best_model_custom.onnx"
tf2onnx.convert.from_keras(model, input_signature=spec, output_path=output_path)

print(f"Model exported to {output_path}")
```

**3. Replace Model**:

```bash
cp best_model_custom.onnx models/best_model.onnx
```

**4. Restart MeterMonitor**

### Retraining YOLO Model

**More Complex** - requires:
- Annotated images (bounding boxes)
- YOLOv11 training pipeline
- GPU for training

**High-Level Steps**:

1. **Annotate Images** (using LabelImg or similar)
2. **Prepare Dataset** (YOLO format)
3. **Train with Ultralytics**:
   ```python
   from ultralytics import YOLO

   model = YOLO('yolo11n-obb.pt')
   model.train(data='meter_dataset.yaml', epochs=100, imgsz=640)
   ```
4. **Export to ONNX**:
   ```python
   model.export(format='onnx')
   ```
5. **Replace Model**:
   ```bash
   cp runs/obb/train/weights/best.onnx models/yolo-best-obb-2.onnx
   ```

**Detailed Guide**: Beyond scope of this document. See [Ultralytics Documentation](https://docs.ultralytics.com/).

---

## Additional Resources

- [User Guide](user-guide.md)
- [API Reference](api-reference.md)
- [ESP32 Setup](esp32-setup.md)
- [ROI Extractors](roi-extractors.md)
- [Troubleshooting](troubleshooting.md)
- [Database Schema](database-schema.md)
- [Development Guide](development.md)

### External Resources

- [ONNX Runtime Performance Tuning](https://onnxruntime.ai/docs/performance/)
- [SQLite Optimization](https://www.sqlite.org/optoverview.html)
- [NGINX Reverse Proxy](https://docs.nginx.com/nginx/admin-guide/web-server/reverse-proxy/)
- [Mosquitto TLS](https://mosquitto.org/man/mosquitto-tls-7.html)
- [Ultralytics YOLOv11](https://docs.ultralytics.com/)

# Database Schema Documentation

Complete documentation of MeterMonitor's SQLite database structure.

## Table of Contents

1. [Overview](#overview)
2. [Schema Diagram](#schema-diagram)
3. [Tables](#tables)
4. [Relationships](#relationships)
5. [Migrations](#migrations)
6. [Backup and Restore](#backup-and-restore)
7. [Direct SQL Queries](#direct-sql-queries)

---

## Overview

MeterMonitor uses SQLite as its database backend. The database stores:
- Water meter configurations
- Image data and metadata
- Recognition evaluations
- Historical readings
- Image sources (MQTT, HA Camera, HTTP)
- ROI extraction templates

**Database File Location**:
- Home Assistant Add-on: `/data/watermeters.sqlite`
- Standalone: `data/watermeters.sqlite` (configurable in `settings.json`)

**SQLite Version**: 3.x (compatible with any SQLite3 client)

---

## Schema Diagram

```
┌─────────────────┐
│   watermeters   │──┐
│                 │  │
│ PK: name        │  │
│     picture_*   │  │
│     setup       │  │
│     picture_*   │  │
│     bbox        │  │
└─────────────────┘  │
         │           │
         │ (1:N)     │ (1:N)
         │           │
         ├───────────┼──────────────┬────────────────┬────────────────┐
         │           │              │                │                │
┌────────▼────────┐  │  ┌──────────▼──────┐ ┌──────▼──────┐ ┌──────▼──────┐
│    settings     │  │  │    history      │ │ evaluations │ │   sources   │
│                 │  │  │                 │ │             │ │             │
│ PK: name        │  │  │ PK: id          │ │ PK: id      │ │ PK: id      │
│ FK: name        │  │  │ FK: name        │ │ FK: name    │ │ FK: name    │
│     thresholds  │  │  │     value       │ │     digits  │ │     type    │
│     segments    │  │  │     timestamp   │ │     preds   │ │     config  │
│     extractor   │  │  │     confidence  │ │     result  │ │     polling │
│     template_id │──┼──┼─────────────────┘ │     metadata│ └─────────────┘
└─────────────────┘  │  │                   └─────────────┘
                     │  │
                     │  │ (0:1)
                     │  │
              ┌──────▼──┴────┐
              │   templates  │
              │              │
              │ PK: id       │
              │     name     │
              │     ref_img  │
              │     config   │
              │     precomp  │
              └──────────────┘
```

---

## Tables

### watermeters

Stores water meter devices and their latest images.

**Columns**:

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `name` | TEXT | PRIMARY KEY | Unique meter identifier |
| `picture_number` | INTEGER | NOT NULL | Monotonic counter from ESP32 |
| `wifi_rssi` | INTEGER | NOT NULL | WiFi signal strength (dBm) |
| `picture_format` | TEXT | NOT NULL | Image format (jpg, png) |
| `picture_timestamp` | TEXT | NOT NULL | ISO 8601 timestamp |
| `picture_width` | INTEGER | NOT NULL | Image width in pixels |
| `picture_height` | INTEGER | NOT NULL | Image height in pixels |
| `picture_length` | INTEGER | NOT NULL | Base64 data length |
| `picture_data` | TEXT | NOT NULL | Base64-encoded image |
| `setup` | BOOLEAN | DEFAULT 0 | 0=in setup, 1=configured |
| `picture_data_bbox` | BLOB | NULLABLE | Image with bounding box drawn |

**Notes**:
- One row per meter
- `picture_data` updated on each image receipt
- `picture_data_bbox` generated after ROI extraction
- `setup=0`: Meter appears in discovery
- `setup=1`: Meter appears in main watermeter list

**Example**:
```sql
SELECT name, picture_timestamp, wifi_rssi, setup
FROM watermeters
WHERE setup = 1;
```

---

### settings

Configuration settings for each water meter.

**Columns**:

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `name` | TEXT | PRIMARY KEY, FK | Meter name (references watermeters) |
| `threshold_low` | INTEGER | NOT NULL | Minimum pixel brightness for main digits |
| `threshold_high` | INTEGER | NOT NULL | Maximum pixel brightness for main digits |
| `threshold_last_low` | INTEGER | NOT NULL | Minimum brightness for last digit |
| `threshold_last_high` | INTEGER | NOT NULL | Maximum brightness for last digit |
| `islanding_padding` | INTEGER | NOT NULL | Padding percentage for islanding (0-100) |
| `segments` | INTEGER | NOT NULL | Number of digit segments (typically 7) |
| `rotated_180` | BOOLEAN | NOT NULL | Whether to rotate image 180° |
| `shrink_last_3` | BOOLEAN | NOT NULL | Shrink width of last 3 digits |
| `extended_last_digit` | BOOLEAN | NOT NULL | Extend last digit extraction area |
| `max_flow_rate` | FLOAT | NOT NULL | Maximum flow rate in m³/h |
| `conf_threshold` | REAL | NULLABLE | Minimum confidence threshold (0-1) |
| `roi_extractor` | TEXT | DEFAULT 'yolo' | ROI extractor type (yolo/orb/bypass) |
| `template_id` | TEXT | NULLABLE | Template ID for ORB extractor |
| `use_correctional_alg` | BOOLEAN | DEFAULT true | Use full (true) or light (false) correction |

**Foreign Key**:
- `name` references `watermeters(name)`

**Notes**:
- One row per meter
- Created automatically on meter discovery
- Default values:
  - Thresholds: 0-100
  - Islanding padding: 20%
  - Segments: 7
  - ROI extractor: yolo
  - Max flow rate: 1.0 m³/h
  - Correctional algorithm: full (true)

**Example**:
```sql
SELECT name, roi_extractor, template_id, use_correctional_alg
FROM settings
WHERE roi_extractor = 'orb';
```

---

### history

Historical water meter readings.

**Columns**:

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `id` | INTEGER | PRIMARY KEY AUTOINCREMENT | Unique history entry ID |
| `name` | TEXT | NOT NULL, FK | Meter name |
| `value` | INTEGER | NOT NULL | Reading value (raw digits) |
| `confidence` | REAL | NOT NULL | Total confidence (0-1) |
| `target_brightness` | REAL | NOT NULL | Target brightness for normalization |
| `timestamp` | TEXT | NOT NULL | ISO 8601 timestamp |
| `manual` | BOOLEAN | NOT NULL | Manual entry (true) or automatic (false) |
| `used_confidence` | REAL | DEFAULT -1.0 | Confidence of digits actually used |

**Foreign Key**:
- `name` references `watermeters(name)`

**Indexes** (recommended, not auto-created):
```sql
CREATE INDEX idx_history_name ON history(name);
CREATE INDEX idx_history_timestamp ON history(timestamp);
CREATE INDEX idx_history_name_timestamp ON history(name, timestamp);
```

**Notes**:
- Ordered by timestamp (implicit via ID)
- Limited by `max_history` config setting
- Oldest entries auto-pruned
- Manual entries marked with `manual=1`
- `used_confidence` tracks confidence of digits after correction

**Example**:
```sql
SELECT value, timestamp, confidence, manual
FROM history
WHERE name = 'kitchen_meter'
ORDER BY timestamp DESC
LIMIT 100;
```

---

### evaluations

Detailed evaluation data for each image processed.

**Columns**:

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `id` | INTEGER | PRIMARY KEY AUTOINCREMENT | Unique evaluation ID |
| `name` | TEXT | NOT NULL, FK | Meter name |
| `colored_digits` | TEXT | NULLABLE | JSON array of base64 digit images (colored) |
| `th_digits` | TEXT | NULLABLE | JSON array of base64 thresholded digits |
| `th_digits_inverted` | TEXT | NULLABLE | JSON array of inverted thresholded digits |
| `predictions` | TEXT | NULLABLE | JSON array of prediction arrays per digit |
| `timestamp` | DATETIME | NOT NULL | Evaluation timestamp |
| `result` | INTEGER | NULLABLE | Final reading value (raw digits) |
| `total_confidence` | REAL | NULLABLE | Product of all digit confidences |
| `used_confidence` | REAL | DEFAULT -1.0 | Confidence after correction |
| `outdated` | BOOLEAN | DEFAULT 0 | Marked for re-evaluation |
| `denied_digits` | TEXT | NULLABLE | JSON array of boolean flags per digit |
| `denied_digits_count` | INTEGER | NULLABLE | Count of denied digits |
| `flow_rate_m3h` | REAL | NULLABLE | Calculated flow rate (m³/h) |
| `delta_m3` | REAL | NULLABLE | Change since last reading (m³) |
| `delta_raw` | INTEGER | NULLABLE | Raw digit difference |
| `time_diff_min` | REAL | NULLABLE | Time since last reading (minutes) |
| `rejection_reason` | TEXT | NULLABLE | Why reading was rejected |
| `negative_correction_applied` | BOOLEAN | NULLABLE | Negative correction used |
| `fallback_digit_count` | INTEGER | NULLABLE | Digits corrected from previous value |
| `digits_changed_vs_last` | INTEGER | NULLABLE | Digits different from last reading |
| `digits_changed_vs_top_pred` | INTEGER | NULLABLE | Corrections applied to predictions |
| `prediction_rank_used_counts` | TEXT | NULLABLE | JSON array: [rank0_count, rank1_count, rank2_count] |
| `timestamp_adjusted` | BOOLEAN | NULLABLE | Timestamp was adjusted to current time |

**Foreign Key**:
- `name` references `watermeters(name)`

**JSON Structure Examples**:

**`colored_digits` / `th_digits` / `th_digits_inverted`**:
```json
["base64_img1", "base64_img2", ..., "base64_img7"]
```

**`predictions`**:
```json
[
  [["5", 0.98], ["6", 0.01], ["8", 0.01]],  // Digit 1: Top 3 predictions
  [["2", 0.95], ["7", 0.03], ["3", 0.02]],  // Digit 2
  ...
]
```

**`denied_digits`**:
```json
[false, false, true, false, false, false, false]  // Digit 3 denied
```

**`prediction_rank_used_counts`**:
```json
[5, 2, 0]  // 5 digits used rank 0 (top pred), 2 used rank 1, 0 used rank 2
```

**Indexes** (recommended):
```sql
CREATE INDEX idx_evals_name ON evaluations(name);
CREATE INDEX idx_evals_outdated ON evaluations(outdated);
CREATE INDEX idx_evals_name_id ON evaluations(name, id);
```

**Notes**:
- Limited by `max_evals` config setting
- `outdated=1`: Re-evaluation needed (settings changed)
- Correctional metadata columns added in v3.0.0+
- `rejection_reason` values:
  - `no_history`: No previous reading
  - `time_diff_zero`: Invalid time difference
  - `fallback_digit`: Prediction failed (empty)
  - `positive_flow_check`: Negative flow detected
  - `max_flow_rate`: Flow rate exceeded limit
  - `confidence_threshold`: Below minimum confidence

**Example**:
```sql
SELECT id, result, total_confidence, flow_rate_m3h, rejection_reason
FROM evaluations
WHERE name = 'kitchen_meter'
  AND outdated = 0
ORDER BY id DESC
LIMIT 50;
```

---

### sources

Image source configurations (MQTT, HA Camera, HTTP).

**Columns**:

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `id` | INTEGER | PRIMARY KEY AUTOINCREMENT | Unique source ID |
| `name` | TEXT | NOT NULL, FK | Meter name |
| `source_type` | TEXT | NOT NULL | Source type: mqtt, ha_camera, http |
| `enabled` | BOOLEAN | DEFAULT 1 | Source enabled/disabled |
| `poll_interval_s` | INTEGER | NULLABLE | Polling interval in seconds (N/A for mqtt) |
| `config_json` | TEXT | NULLABLE | JSON configuration (entity IDs, URLs, etc.) |
| `last_success_ts` | TEXT | NULLABLE | Last successful capture timestamp |
| `last_error` | TEXT | NULLABLE | Last error message |
| `created_ts` | TEXT | DEFAULT datetime('now') | Creation timestamp |
| `updated_ts` | TEXT | DEFAULT datetime('now') | Last update timestamp |

**Foreign Key** (implicit):
- `name` references `watermeters(name)`

**JSON Structure for `config_json`**:

**MQTT** (auto-created, no config):
```json
null
```

**HA Camera**:
```json
{
  "camera_entity_id": "camera.bathroom_meter",
  "flash_entity_id": "light.bathroom_meter_led",
  "flash_delay_ms": 1000
}
```

**HTTP**:
```json
{
  "url": "http://192.168.1.100/snapshot.jpg",
  "headers": {
    "Authorization": "Bearer token123"
  },
  "body": null
}
```

**Indexes** (recommended):
```sql
CREATE INDEX idx_sources_name ON sources(name);
CREATE INDEX idx_sources_type ON sources(source_type);
CREATE INDEX idx_sources_enabled ON sources(enabled);
```

**Notes**:
- Multiple sources can exist for same meter
- Only one should be enabled at a time
- MQTT sources auto-created on first message
- HA Camera and HTTP sources created via API

**Example**:
```sql
SELECT name, source_type, enabled, poll_interval_s, last_success_ts, last_error
FROM sources
WHERE enabled = 1
  AND source_type IN ('ha_camera', 'http');
```

---

### templates

ROI extraction templates for ORB extractor.

**Columns**:

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `id` | TEXT | PRIMARY KEY | UUID (e.g., 550e8400-e29b-41d4-a716-...) |
| `name` | TEXT | NOT NULL | User-friendly template name |
| `created_at` | TEXT | NOT NULL | ISO 8601 creation timestamp |
| `reference_image_base64` | TEXT | NOT NULL | Base64-encoded reference image |
| `image_width` | INTEGER | NOT NULL | Reference image width |
| `image_height` | INTEGER | NOT NULL | Reference image height |
| `config_json` | TEXT | NOT NULL | JSON configuration (corners, sizes, etc.) |
| `precomputed_data_base64` | TEXT | NULLABLE | Serialized ORB keypoints/descriptors |

**JSON Structure for `config_json`**:
```json
{
  "display_corners": [
    [100.0, 150.0],  // Top-left
    [500.0, 150.0],  // Top-right
    [500.0, 300.0],  // Bottom-right
    [100.0, 300.0]   // Bottom-left
  ],
  "target_width": 400,
  "target_height": 150,
  "target_width_ext": 480,
  "target_height_ext": 180,
  "min_inliers": 10,
  "inlier_ratio_threshold": 0.3,
  "max_reprojection_error": 3.0,
  "matching_mask_padding": 10,
  "orb_nfeatures": 2000,
  "orb_scale_factor": 1.2,
  "orb_nlevels": 8
}
```

**Notes**:
- Templates are immutable (create new, delete old)
- `precomputed_data_base64` contains pickled ORB data
- Referenced by `settings.template_id`
- Can be shared across multiple meters

**Example**:
```sql
SELECT id, name, created_at, image_width, image_height
FROM templates
ORDER BY created_at DESC;
```

---

## Relationships

### Entity Relationship Summary

```
watermeters (1) ──< (N) settings
watermeters (1) ──< (N) history
watermeters (1) ──< (N) evaluations
watermeters (1) ──< (N) sources
settings    (N) ──> (0:1) templates
```

### Cascade Behavior

**When deleting a watermeter**:
- Settings: Manual DELETE required (or use API endpoint)
- History: Manual DELETE required
- Evaluations: Manual DELETE required
- Sources: Manual DELETE required

**API endpoint** `/api/watermeters/{name}` handles cascading deletes:
```sql
DELETE FROM watermeters WHERE name = ?;
DELETE FROM evaluations WHERE name = ?;
DELETE FROM history WHERE name = ?;
DELETE FROM settings WHERE name = ?;
DELETE FROM sources WHERE name = ?;
```

**When deleting a template**:
- Settings referencing it: `template_id` becomes NULL (or should be updated)

---

## Migrations

### Migration System

Migrations run automatically on startup via `/db/migrations.py`.

**Migration Function**: `run_migrations(db_file)`

**Migration Process**:
1. Check if tables exist → Create if not
2. Check column existence → Add missing columns
3. Transform data if schema changed
4. Log migration actions

### Migration History

**v1.x → v2.x**:
- Added `picture_data_bbox` column to `watermeters`
- Added `conf_threshold` to `settings`
- Added ID primary keys to `history` and `evaluations`

**v2.x → v3.0.0**:
- Added `roi_extractor`, `template_id` to `settings`
- Added `sources` table
- Added `templates` table
- Added `denied_digits`, `th_digits_inverted` to `evaluations`
- Added `used_confidence` to `history` and `evaluations`
- Added correctional metadata columns to `evaluations`

**v3.0.x → v3.1.x**:
- Added `use_correctional_alg` to `settings`
- Added timestamp adjustment tracking

### Custom Migrations

To add custom migration:

1. Edit `/db/migrations.py`
2. Add column check:
```python
cursor.execute("PRAGMA table_info(tablename)")
columns = [info[1] for info in cursor.fetchall()]
if 'new_column' not in columns:
    cursor.execute("ALTER TABLE tablename ADD COLUMN new_column TYPE DEFAULT value")
    print("[MIGRATION] Added 'new_column' to 'tablename'")
```
3. Restart MeterMonitor

---

## Backup and Restore

### Manual Backup

**SQLite file copy** (simplest):
```bash
# Stop MeterMonitor first!
cp watermeters.sqlite watermeters_backup_$(date +%Y%m%d).sqlite
```

**SQLite dump** (portable):
```bash
sqlite3 watermeters.sqlite .dump > backup.sql
```

### Automated Backup

**Cron job example**:
```bash
0 2 * * * /usr/bin/sqlite3 /path/to/watermeters.sqlite ".backup '/path/to/backups/watermeters_$(date +\%Y\%m\%d).sqlite'"
```

**Docker volume backup**:
```bash
docker run --rm -v metermonitor_data:/data -v $(pwd):/backup alpine \
  tar czf /backup/metermonitor_data_backup.tar.gz /data
```

### Restore

**From file copy**:
```bash
# Stop MeterMonitor
cp watermeters_backup.sqlite watermeters.sqlite
# Start MeterMonitor
```

**From SQL dump**:
```bash
sqlite3 watermeters_new.sqlite < backup.sql
mv watermeters_new.sqlite watermeters.sqlite
```

### Incremental Backup

SQLite supports online backup via:
- `BACKUP` command (requires SQLite 3.27.0+)
- Vacuum into: `VACUUM INTO 'backup.sqlite'`

**Not yet integrated** into MeterMonitor UI.

---

## Direct SQL Queries

### Common Queries

**List all meters with last reading**:
```sql
SELECT
    w.name,
    w.picture_timestamp,
    h.value AS last_reading,
    h.confidence,
    h.timestamp AS last_history_timestamp
FROM watermeters w
LEFT JOIN history h ON h.name = w.name AND h.id = (
    SELECT id FROM history WHERE name = w.name ORDER BY id DESC LIMIT 1
)
WHERE w.setup = 1;
```

**Meter statistics**:
```sql
SELECT
    name,
    COUNT(*) AS eval_count,
    AVG(total_confidence) AS avg_confidence,
    SUM(CASE WHEN rejection_reason IS NOT NULL THEN 1 ELSE 0 END) AS rejections,
    SUM(CASE WHEN outdated = 1 THEN 1 ELSE 0 END) AS outdated_count
FROM evaluations
GROUP BY name;
```

**Find evaluations with low confidence**:
```sql
SELECT
    id, name, result, total_confidence, timestamp, rejection_reason
FROM evaluations
WHERE total_confidence < 0.7
  AND outdated = 0
ORDER BY total_confidence ASC
LIMIT 50;
```

**History consumption over time**:
```sql
SELECT
    name,
    DATE(timestamp) AS date,
    MAX(value) - MIN(value) AS daily_consumption,
    COUNT(*) AS readings
FROM history
WHERE name = 'kitchen_meter'
  AND timestamp >= datetime('now', '-30 days')
GROUP BY name, date
ORDER BY date;
```

**Source health check**:
```sql
SELECT
    name,
    source_type,
    enabled,
    ROUND((julianday('now') - julianday(last_success_ts)) * 1440) AS minutes_since_success,
    last_error
FROM sources
WHERE enabled = 1
  AND last_success_ts IS NOT NULL;
```

### Database Maintenance

**Vacuum** (reclaim space):
```sql
VACUUM;
```

**Analyze** (update statistics):
```sql
ANALYZE;
```

**Integrity check**:
```sql
PRAGMA integrity_check;
```

**Table sizes**:
```sql
SELECT
    name,
    SUM(pgsize) / 1024.0 / 1024.0 AS size_mb
FROM dbstat
GROUP BY name
ORDER BY size_mb DESC;
```

**Orphaned records** (no corresponding meter):
```sql
-- Orphaned settings
SELECT name FROM settings WHERE name NOT IN (SELECT name FROM watermeters);

-- Orphaned history
SELECT DISTINCT name FROM history WHERE name NOT IN (SELECT name FROM watermeters);

-- Orphaned evaluations
SELECT DISTINCT name FROM evaluations WHERE name NOT IN (SELECT name FROM watermeters);

-- Orphaned sources
SELECT DISTINCT name FROM sources WHERE name NOT IN (SELECT name FROM watermeters);
```

**Clean up orphans**:
```sql
DELETE FROM settings WHERE name NOT IN (SELECT name FROM watermeters);
DELETE FROM history WHERE name NOT IN (SELECT name FROM watermeters);
DELETE FROM evaluations WHERE name NOT IN (SELECT name FROM watermeters);
DELETE FROM sources WHERE name NOT IN (SELECT name FROM watermeters);
```

---

## Additional Resources

- [User Guide](user-guide.md)
- [API Reference](api-reference.md)
- [Development Guide](development.md)
- [Troubleshooting](troubleshooting.md)
- [SQLite Documentation](https://www.sqlite.org/docs.html)

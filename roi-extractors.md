# ROI Extractor Guide

Comprehensive guide to Region of Interest (ROI) extraction methods in MeterMonitor.

## Table of Contents

1. [Overview](#overview)
2. [Extractor Types](#extractor-types)
3. [YOLO Extractor](#yolo-extractor)
4. [ORB Template Extractor](#orb-template-extractor)
5. [Bypass Extractor](#bypass-extractor)
6. [Performance Comparison](#performance-comparison)
7. [Troubleshooting](#troubleshooting)

---

## Overview

ROI (Region of Interest) extractors identify and extract the digit display region from water meter images. This is the critical first step before digit recognition.

### Why ROI Extraction Matters

**Without ROI Extraction**:
- Image contains meter housing, background, shadows
- Digit recognition would process irrelevant areas
- Results probably unusable, unless MeterMonitor receives pre-processed images

**With ROI Extraction**:
- Only digit display region processed
- Can handle movement of the camera with some extractors.

### Extraction Pipeline

```
Input Image
    ↓
ROI Extractor (YOLO/ORB/Static)
    ↓
Extracted Display Region (Straightened)
    ↓
Segmentation (Split into digits)
    ↓
Thresholding (Binary conversion)
    ↓
Digit Recognition (CNN classifier)
```

---

## Extractor Types

MeterMonitor supports three ROI extraction methods:

| Extractor  | Method             | Use Case                                                             | Configuration |
|------------|--------------------|----------------------------------------------------------------------|---------------|
| **YOLO**   | AI-based detection | Standard meters, good images                                         | Automatic |
| **ORB**    | Template matching  | Most reliable, requires distinct features outside the display region | Manual template |
| **Static** | Static cutting     | For very stable camera setups that don't move.                       | Manual template |
| **Bypass** | No extraction      | For pre-cropped images                                               | None |

### Selection Criteria

**Choose YOLO when**:
- Meter has standard rectangular display
- Good image quality and lighting
- Camera position is stable
- You want zero-configuration solution
- **Test with your own meter – some work great, some don't.**

**Choose ORB when**:
- YOLO detection fails consistently/too often
- Non-standard meter layout
- Extreme lighting conditions (very dark/bright)
- Camera position varies slightly between captures
- You need fine-tuned control

**Choose Static when**:
- Camera position is very stable – then it's super reliable
- Extremely fast (no AI/ORB)

**Choose Bypass when**:
- Testing digit recognition only
- Images are already cropped to display
- Debugging extraction issues
- Custom pre-processing pipeline

---

## YOLO Extractor

### How It Works

YOLO (You Only Look Once) is a deep learning model trained to detect meter displays:

1. **Detection**: Scans image for display region
2. **Oriented Bounding Box (OBB)**: Identifies rotated rectangle around digits
3. **Perspective Transform**: Straightens display to frontal view
4. **Extraction**: Crops and returns display region

### Model Details

- Architecture: YOLOv11 with OBB support
- Training: Trained on diverse water meter images
- Input: 640x640 RGB image
- Output: Rotated bounding box coordinates
- Format: ONNX (optimized for CPU inference)

### Configuration Options

**In Meter Settings**:

```json
{
  "roi_extractor": "yolo",
  "rotated_180": false,
  "extended_last_digit": false
}
```

**Parameters**:

#### `rotated_180`
- **Type**: Boolean
- **Default**: `false`
- **Purpose**: Handles upside-down meters
- **Effect**: Rotates entire image 180° before detection
- **Use when**: Meter is physically inverted in image

#### `extended_last_digit`
- **Type**: Boolean
- **Default**: `false`
- **Purpose**: Extends extraction area for rightmost digit
- **Effect**: Captures 20% more area on right side
- **Use when**: Last digit (rotation indicator) is cut off

### Advantages

- **Zero configuration**: Works immediately
- **Robust to position**: Handles camera movement
- **Fast inference**: ~100-200ms on modern CPU
- **Rotation handling**: Detects displays at any angle
- **Perspective correction**: Straightens tilted displays

### Limitations

- **Standard layouts only**: May fail on unusual meters
- **Requires clear display**: Low contrast problematic
- **Model dependency**: Fixed behavior, can't fine-tune per meter
- **Glare sensitivity**: Reflections can confuse detection
- **Training data bias**: Works best on meter types in training set

### Troubleshooting YOLO Detection

**No detection (bbox not found)**:
1. Check image quality (blur, low light)
2. Verify entire display is visible
3. Try increasing image brightness
4. Consider using ORB extractor instead

**Incorrect detection**:
1. Enable `rotated_180` if meter inverted
2. Check for glare/reflections (reposition camera)
3. Ensure background doesn't have digit-like patterns
4. Try different camera angle

**Partial detection**:
1. Enable `extended_last_digit` for cut-off right edge
2. Adjust camera to include full display
3. Check model is detecting full display rectangle

---

## ORB Template Extractor

### How It Works

ORB (Oriented FAST and Rotated BRIEF) uses feature matching:

1. **Template Creation**: User marks display corners on reference image
2. **Feature Extraction**: ORB finds distinctive keypoints in template
3. **Matching**: For each new image, match keypoints to template
4. **Homography**: Calculate perspective transform from matches
5. **Extraction**: Apply transform to extract display region

### When to Use ORB

**ORB excels when**:
- YOLO consistently fails
- Meter has unique/non-standard layout
- Extreme lighting variations (ORB is robust)
- Camera position varies within small range
- You need deterministic results

**Real-world examples**:
- Meters with surrounding patterns that confuse YOLO
- Very old meters with unusual digit layouts
- Industrial meters with non-standard displays
- Low-light environments where YOLO detection fails

### Creating Templates (in Setup)

#### Step 1: Capture Reference Image

**Requirements**:
- Clear view of entire display
- Good lighting (representative of typical conditions)
- All digits visible
- Display not obscured

**Tips**:
- Use automatic capture or manual test
- Ensure focus is sharp
- Avoid extreme angles
- If using flash, capture with flash on

#### Step 2: Mark Display Corners

**Point Order** (clockwise from top-left):
1. Top-left corner of digit display
2. Top-right corner
3. Bottom-right corner
4. Bottom-left corner

**Interactive Editor**:
- Drag points to adjust
- Zoom for precision
- Preview extracted region in real-time
- Points shown as numbered circles

**Tips**:
- Mark actual digit area (not meter housing)
- Include small margin (1-2mm) around digits
- Ensure rectangle encompasses all digits
- Avoid including frame/background

#### Step 3: Save Template

After saving ("applying") the template, it will be used for all incoming images from that meter.

### ORB Algorithm Details

**Feature Detection**:
- Detects corner-like keypoints
- Extracts rotation-invariant descriptors

**Matching**:
- Brute-force matcher with Hamming distance
- Lowe's ratio test for robust matching
- RANSAC for outlier rejection

**Homography Estimation**:
- Minimum 4 matches required (default: 10)
- Reprojection error threshold: 3 pixels
- Validates positive flow (no inverted transforms)

### Advantages

- **Lighting robust**: Works in variable lighting
- **Fine control**: User marks exact region
- **Explainable**: Visual template editor
- **Deterministic**: Same features always detected

### Limitations

- **Setup time**: Requires manual template creation
- **Template dependency**: Must recreate if meter replaced
- **Feature-dependent**: Fails if display has no texture/features
- **Slower**: ~200-400ms per image (vs YOLO ~100-200ms)
- **Homography fails**: Insufficient matches → no extraction

### Troubleshooting ORB Extraction

**Not enough matches**:
1. Recapture reference image with better lighting
2. Ensure template has textured patterns (not blank)
3. Check camera position hasn't drastically changed

**Incorrect extraction**:
1. Verify corner points are correct
2. Ensure corners are in clockwise order
3. Check for glare/occlusion in current vs reference
4. Recapture template with representative lighting

**Homography fails**:
1. Too few features matched (see "not enough matches")
2. Camera moved significantly from template position
3. Extreme lighting change (very rare with ORB)
4. Recapture template closer to current conditions


---

## Static Extractor

The static extractor is configured in the same way as the ORB extractor.
Instead of matching features, it uses a pre-defined region of interest (ROI) for extraction.
The ROI is defined by a set of four corner points, which are used to crop the image for display region extraction.

## Advantages
1. Robust to lighting changes
2. Faster than ORB extractor
3. No need for feature detection and matching
4. Suitable for images that never change perspective

---

## Bypass Extractor

### Purpose

Bypass extractor performs no ROI extraction. The entire input image is used as the "display region."

### Use Cases

**1. Testing Digit Recognition**:
- Isolate recognition from extraction issues
- Test thresholds without detection noise
- Benchmark recognition accuracy

**2. Pre-cropped Images**:
- Images already cropped to display
- Custom preprocessing pipeline
- Integration with external systems

**3. Debugging**:
- Determine if issue is extraction or recognition
- Test different image sources
- Validate processing pipeline

### Configuration

```json
{
  "roi_extractor": "bypass"
}
```

**No additional parameters**. Image is passed directly to segmentation.

### Behavior

```
Input Image (entire image)
    ↓
Segmentation (split into segments)
    ↓
Thresholding
    ↓
Digit Recognition
```

**Important**: Image should be:
- Display only (no background)
- Frontal view (not tilted)
- Approximately rectangular
- Correct orientation

### Limitations

- No automatic detection
- No perspective correction
- No rotation handling
- Camera must be perfectly aligned

---

## Choosing the Right Extractor

### Decision Tree

```
Start
  ↓
Is meter a standard rectangular display?
  ├─ Yes → Try YOLO
  │   ↓
  │   Does YOLO detect consistently?
  │   ├─ Yes → Use YOLO ✓
  │   └─ No → Try ORB
  │       ↓
  │       Does ORB extract correctly?
  │       ├─ Yes → Use ORB ✓
  │       └─ No → Check camera/lighting
  │
  └─ No → Use ORB
      ↓
      Create template for non-standard layout ✓
```

---

## Best Practices

### General

- **Test both extractors** during initial setup
- **Monitor confidence scores** to detect degradation
- **Recapture templates** if meter hardware changes
- **Document working configuration** for future reference

### YOLO

- Use high-quality images (640x480 minimum)
- Ensure good lighting (consistent across captures)
- Avoid glare and reflections
- Center display in camera frame
- Use `extended_last_digit` only if needed

### ORB

- Capture template in representative conditions
- Mark corners precisely (use zoom)
- Use textured reference image (not blank display)
- Test template with multiple images before deployment
- Recapture if camera position changes significantly

### Performance

- YOLO: Preferred for standard meters (faster)
- ORB: Use when accuracy more important than speed
- Bypass: Only for testing or custom pipelines

---


## Troubleshooting

### YOLO Issues

**Problem**: No bounding box detected

**Solutions**:
1. Check image quality (blur, focus, lighting)
2. Verify display is fully visible in frame
3. Increase image brightness/contrast
4. Try ORB extractor
5. Review evaluation detail images

**Problem**: Bounding box in wrong location

**Solutions**:
1. Remove background clutter (reposition camera)
2. Check for patterns that look like digits
3. Ensure only one display in frame
4. Adjust camera angle

**Problem**: Rotation incorrect

**Solutions**:
1. Enable `rotated_180` setting
2. Verify meter orientation
3. Check YOLO detected rotation angle

### ORB Issues

**Problem**: Insufficient matches

**Solutions**:
1. Recapture reference with better lighting
2. Ensure display has textured features
3. Check camera hasn't moved too far
4. Clean camera lens
5. Adjust template corners to include more features

**Problem**: Homography estimation fails

**Solutions**:
1. Increase `min_inliers` (if too strict)
2. Recapture template closer to current conditions
3. Check for occlusion (dirt, condensation on lens)
4. Verify template corners are correct

**Problem**: Distorted extraction

**Solutions**:
1. Verify corner points in correct order (clockwise)
2. Check corners mark rectangle (not skewed polygon)
3. Ensure camera position similar to template
4. Recapture template with better corner placement

### General Extraction Issues

**Problem**: Extracted region too small/large

**Solutions**:
1. For YOLO: Adjust camera distance
2. For ORB: Remark template corners
3. Check segment count matches meter

**Problem**: Partial digit extraction

**Solutions**:
1. Enable `extended_last_digit`
2. Adjust camera framing (more margin)
3. For ORB: Extend corner points outward

**Problem**: Inconsistent extraction

**Solutions**:
1. Improve mounting stability (reduce vibration)
2. Use more consistent lighting
3. For YOLO: Consider switching to ORB (more stable)
4. Check camera auto-focus is disabled (if applicable)

---

## Additional Resources

- [User Guide](user-guide.md)
- [API Reference](api-reference.md)
- [Troubleshooting Guide](troubleshooting.md)

### Research Papers

- **YOLO**: [YOLOv11 Documentation](https://docs.ultralytics.com/)
- **ORB**: [ORB: An Efficient Alternative to SIFT or SURF](https://www.willowgarage.com/sites/default/files/orb_final.pdf)
- **Homography**: [Multiple View Geometry in Computer Vision](https://www.robots.ox.ac.uk/~vgg/hzbook/)

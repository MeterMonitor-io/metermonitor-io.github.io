# MeterMonitor Documentation

Welcome to the MeterMonitor documentation. This guide provides comprehensive information about installing, configuring, and using the MeterMonitor Home Assistant addon for AI-powered analog water meter reading.

## Documentation Structure

- **[Getting Started](getting-started.md)** - Quick start guide and installation instructions
- **[Architecture](architecture.md)** - System architecture and design overview
- **[Installation](installation.md)** - Detailed installation instructions for different deployment methods
- **[Configuration](configuration.md)** - Complete configuration reference
- **[User Guide](user-guide.md)** - Step-by-step guide for using the web interface
- **[API Reference](api-reference.md)** - Complete REST API documentation
- **[ESP32 Setup](esp32-setup.md)** - Guide for setting up ESP32-CAM devices
- **[ROI Extractors](roi-extractors.md)** - Region of Interest extraction methods
- **[Troubleshooting](troubleshooting.md)** - Common issues and solutions
- **[Development](development.md)** - Guide for developers and contributors
- **[Database Schema](database-schema.md)** - Database structure documentation
- **[Advanced Topics](advanced-topics.md)** - Advanced configuration and customization

## What is MeterMonitor?

MeterMonitor is an AI-powered backend system for reading analog water meters via ESP32 cameras. Instead of running OCR on-device, this solution offloads image processing and digit recognition to a server, significantly reducing device power consumption while providing superior accuracy.

### Key Features

- **AI-Based Detection**: YOLOv11 for display region detection
- **CNN Digit Recognition**: Trained neural network for digit classification with error correction
- **MQTT Integration**: Seamless image ingestion from ESP32 devices
- **Multiple Sources**: Support for MQTT, Home Assistant cameras, and HTTP endpoints
- **Template-Based ROI**: Manual region selection using ORB feature matching
- **Per-Digit Models**: Mix rotating-digit and 7-segment models per position
- **Configurable Decimals**: Decimal separator position configurable per meter
- **Realtime UI Updates**: WebSocket event stream for automatic evaluation updates
- **FastAPI Backend**: High-performance REST API with automatic documentation
- **Vue 3 Frontend**: Modern, responsive web interface
- **Home Assistant Integration**: Native addon with MQTT auto-discovery
- **History Tracking**: Comprehensive evaluation and reading history
- **Flow Rate Validation**: Configurable max flow rate checking with correction algorithms

## Quick Links

- [Main Repository](https://github.com/phiph-s/metermonitor-managementserver)
- [Home Assistant Add-on Store](https://github.com/phiph-s/metermonitor-managementserver)
- [Issue Tracker](https://github.com/phiph-s/metermonitor-managementserver/issues)

## System Requirements

### For Home Assistant Add-on
- Home Assistant OS or Supervised
- Architecture: `amd64` or `aarch64` (ARM)
- Minimum 2GB RAM recommended
- MQTT broker (e.g., Mosquitto add-on)

### For Standalone Deployment
- Python 3.12+
- 2GB+ RAM
- Linux, Windows, or macOS
- MQTT broker access

## Getting Help

- Check the [Troubleshooting Guide](troubleshooting.md) for common issues
- Review the [API Reference](api-reference.md) for integration questions
- Open an [issue on GitHub](https://github.com/phiph-s/metermonitor-managementserver/issues) for bug reports or feature requests

## License

This project was developed as a Master Project at Hochschule RheinMain.

## Acknowledgments

This project is heavily inspired by:
- [AI-on-the-edge-device](https://github.com/jomjol/AI-on-the-edge-device)
- Training dataset by [haverland](https://github.com/haverland/collectmeterdigits)

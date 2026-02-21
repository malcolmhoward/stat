# S.T.A.T. - System Telemetry and Analytics Tracker

## Table of Contents

- [Overview](#overview)
- [Hardware](#hardware)
  - [Power Monitors](#power-monitors)
  - [Battery Management](#battery-management)
  - [Platform Detection](#platform-detection)
  - [I2C Bus Configuration](#i2c-bus-configuration)
- [Communication](#communication)
  - [MQTT Topics](#mqtt-topics)
  - [JSON Payload Format](#json-payload-format)
  - [Subscribing Components](#subscribing-components)
- [System Monitoring](#system-monitoring)
- [Battery Profiles](#battery-profiles)
- [Service Mode](#service-mode)
- [Troubleshooting](#troubleshooting)
- [Related Components](#related-components)

---

## Overview

S.T.A.T. is the telemetry backbone of the O.A.S.I.S. wearable computing platform. Telemetry — the automated collection and transmission of measurements from remote hardware — is how the suit knows its own status. S.T.A.T. serves as the diagnostic heartbeat: collecting hardware metrics and broadcasting them across the network so every component stays informed.

**What it monitors:**
- Power: voltage, current, power draw (via INA238 and INA3221)
- Battery: state of charge, time remaining, cell-level health (via Daly BMS)
- System: CPU usage, memory usage, fan speed, temperature

**How it shares data:**
- Publishes structured JSON telemetry to MQTT topics
- Any O.A.S.I.S. component (or external tool) can subscribe
- Designed for real-time consumption — typically 1-second update intervals

**Where it runs:**
- NVIDIA Jetson (Nano, Orin Nano, NX) — primary target
- Raspberry Pi — supported
- Any Linux system with I2C support

---

## Hardware

### Power Monitors

S.T.A.T. supports two types of I2C power monitors and auto-detects which are available:

#### INA238 (Single-Channel)

High-precision power monitor communicating via direct I2C:

| Parameter | Default | ARK Board Default |
|-----------|---------|-------------------|
| I2C Address | `0x45` | `0x45` |
| I2C Bus | `/dev/i2c-1` | `/dev/i2c-7` |
| Shunt Resistor | 0.0003 ohm | 0.001 ohm |
| Max Current | 327.68 A | 10.0 A |

Provides: bus voltage, current, power, die temperature.

#### INA3221 (Three-Channel)

Multi-channel power monitor accessed via Linux sysfs/hwmon interface:

- Monitors up to three power rails simultaneously
- Each channel reports voltage, current, and power independently
- Auto-detected at `/sys/bus/i2c/drivers/ina3221`

#### Unified Monitoring

When multiple sources are available, S.T.A.T. combines data for a unified view — prioritizing the most accurate source for each metric and combining INA238 precision with BMS cell-level data.

### Battery Management

#### Daly Smart BMS Integration

S.T.A.T. provides comprehensive integration with Daly Smart BMS via serial (UART):

| Parameter | Default |
|-----------|---------|
| Serial Port | `/dev/ttyTHS1` |
| Baud Rate | 9600 |
| Poll Interval | 1000 ms |

**Capabilities:**
- Individual cell voltage tracking
- Cell balancing status monitoring
- Charge/discharge MOSFET state
- Multiple temperature sensors
- Fault detection and categorization (by severity)
- Cell deviation analysis with WARNING/CRITICAL thresholds

**Cell Health Analysis:**
1. Calculates statistics: min, max, average, delta across all cells
2. Detects deviations from average using configurable thresholds
3. Categorizes each cell as NORMAL, WARNING, or CRITICAL
4. Publishes detailed health data via MQTT

#### Battery Time Estimation

S.T.A.T. estimates remaining runtime using:
1. Chemistry-specific discharge curves (Li-ion, LiPo, LiFePO4, NiMH, Lead-Acid)
2. Temperature compensation (reduced capacity at lower temperatures)
3. Actual measured current draw
4. Cell configuration (series/parallel count)
5. Adaptive smoothing to prevent erratic jumps

### Platform Detection

S.T.A.T. auto-detects the running platform:

1. **ARK Electronics Jetson Carrier**: Detected by reading the AT24CSW010 EEPROM serial number on I2C. Applies optimized defaults (bus `/dev/i2c-7`, shunt 0.001 ohm, max current 10.0 A).
2. **Generic Jetson**: Detected via `/etc/nv_tegra_release`.
3. **Raspberry Pi**: Detected via `/proc/cpuinfo`.
4. **Generic Linux**: Fallback with standard I2C defaults.

### I2C Bus Configuration

```bash
# Verify I2C bus availability
ls /dev/i2c-*

# Scan for devices on bus 7 (ARK boards)
i2cdetect -y 7

# Scan for devices on bus 1 (generic)
i2cdetect -y 1

# Read INA238 manufacturer ID to verify communication
i2cget -y 7 0x45 0x3e w
```

**Required permissions:** User must be in the `i2c` group:
```bash
sudo usermod -a -G i2c $USER
# Log out and back in for changes to take effect
```

---

## Communication

### MQTT Topics

S.T.A.T. publishes all telemetry to a configurable base topic (default: `stat`). Data types are distinguished by the `"device"` and `"type"` fields in each JSON payload.

| Data Type | device | type | Content |
|-----------|--------|------|---------|
| Battery (INA238) | `Battery` | `INA238` | Voltage, current, power, temperature, SOC, time remaining |
| Power (INA3221) | `Power` | `INA3221` | Per-channel voltage, current, power |
| BMS | `BMS` | `Daly` | Cell voltages, temperatures, FET status, faults |
| Battery Health | `Health` | `Battery` | Cell statistics, deviation analysis, fault categorization |
| System Metrics | `System` | `Metrics` | CPU usage, memory usage, fan speed |
| Unified Battery | `Battery` | `Unified` | Combined data from all sources with prioritization |

### JSON Payload Format

All data is published as JSON. Example INA238 battery payload:

```json
{
  "device": "Battery",
  "type": "INA238",
  "voltage": 15.842,
  "current": 1.512,
  "power": 23.953,
  "temperature": 46.38,
  "battery_level": 76.0,
  "time_remaining_min": 303.5,
  "time_remaining_fmt": "5:03",
  "battery_status": "NORMAL"
}
```

### Subscribing Components

Any MQTT client can subscribe to S.T.A.T. telemetry:

```bash
# Subscribe to all S.T.A.T. data
mosquitto_sub -t stat

# Subscribe with verbose output (shows topic)
mosquitto_sub -v -t stat
```

**O.A.S.I.S. consumers:**
- **MIRAGE**: Subscribes to display battery level, power draw, and temperature on the HUD
- **DAWN**: Subscribes to answer voice queries like "What's my battery level?"

---

## System Monitoring

Beyond power and battery, S.T.A.T. monitors system health:

| Metric | Source | Published Field |
|--------|--------|-----------------|
| CPU Usage | `/proc/stat` | Percentage per core and total |
| Memory Usage | `/proc/meminfo` | Used, free, total, percentage |
| Fan Speed | Platform-specific sysfs | RPM, PWM percentage |
| Temperature | `/sys/class/thermal/` | Degrees Celsius |

---

## Battery Profiles

S.T.A.T. includes predefined battery profiles selectable via `--battery <name>`:

| Name | Cells | Capacity | Chemistry |
|------|-------|----------|-----------|
| `4S_Li-ion` | 4S1P | 2,600 mAh | Li-ion |
| `5S_Li-ion` | 5S1P | 2,600 mAh | Li-ion |
| `6S_Li-ion` | 6S1P | 2,600 mAh | Li-ion |
| `2S_LiPo` | 2S1P | 5,000 mAh | LiPo |
| `3S_LiPo` | 3S1P | 5,000 mAh | LiPo |
| `6S_LiPo` | 6S1P | 5,000 mAh | LiPo |
| `4S2P_Samsung50E` | 4S2P | 10,000 mAh | Li-ion |
| `3S_5200mAh_LiPo` | 3S1P | 5,200 mAh | LiPo |
| `3S_2200mAh_LiPo` | 3S1P | 2,200 mAh | LiPo |
| `3S_1500mAh_LiPo` | 3S1P | 1,500 mAh | LiPo |

Custom profiles can be specified via CLI options (`--battery-cells`, `--battery-capacity`, `--battery-chemistry`, etc.).

List all available profiles: `./oasis-stat --list-batteries`

---

## Service Mode

S.T.A.T. can run as a systemd service for continuous background monitoring:

```bash
# Start the service
sudo systemctl start oasis-stat

# Check status
sudo systemctl status oasis-stat

# View logs
sudo journalctl -u oasis-stat

# Enable on boot
sudo systemctl enable oasis-stat
```

Configuration is loaded from `/etc/oasis/stat.conf`:
```ini
# MQTT Settings
MQTT_HOST=localhost
MQTT_PORT=1883
MQTT_TOPIC=stat
```

The service runs as the `oasis` user with security hardening: `ProtectSystem=full`, `ProtectHome=true`, `PrivateTmp=true`, `NoNewPrivileges=true`.

---

## Troubleshooting

### I2C Device Not Detected

```bash
# Verify the I2C bus exists
ls /dev/i2c-*

# Scan for devices (use bus 7 for ARK boards, bus 1 for generic)
i2cdetect -y 7
i2cdetect -y 1

# Check permissions
groups  # Should include 'i2c'
sudo usermod -a -G i2c $USER  # Add if missing
```

**Common causes:** Wrong bus number, device not powered, loose wiring, missing `i2c` group membership.

### MQTT Broker Connection Failed

```bash
# Check if mosquitto is running
sudo systemctl status mosquitto

# Start mosquitto
sudo systemctl start mosquitto

# Test publishing manually
mosquitto_pub -t test -m "hello"

# Test subscribing
mosquitto_sub -t stat
```

**Common causes:** Mosquitto not installed (`sudo apt install mosquitto`), broker not running, wrong host/port.

### BMS Serial Communication Issues

```bash
# Check serial port exists
ls -l /dev/ttyTHS*
ls -l /dev/ttyUSB*

# Check permissions
groups  # Should include 'dialout'
sudo usermod -a -G dialout $USER  # Add if missing

# Test serial communication
stty -F /dev/ttyTHS1 9600
```

**Common causes:** Wrong serial port, incorrect baud rate, missing `dialout` group membership, BMS not powered on.

### systemd Service Issues

```bash
# Check service status and recent logs
sudo systemctl status oasis-stat
sudo journalctl -u oasis-stat --since "5 minutes ago"

# Verify config file
cat /etc/oasis/stat.conf

# Verify binary is installed
which oasis-stat
```

**Common causes:** Binary not installed (`sudo make install`), config file missing, `oasis` user not created.

### Build Issues

```bash
# Install dependencies (Ubuntu/Debian)
sudo apt-get install build-essential cmake libmosquitto-dev libjson-c-dev

# Clean rebuild
rm -rf build && mkdir build && cd build
cmake .. && make -j$(nproc)
```

---

## Related Components

| Component | Relationship |
|-----------|-------------|
| **MIRAGE** | Displays S.T.A.T. telemetry on the HUD — battery level, power draw, temperature indicators. Subscribes to MQTT topics for real-time data. |
| **DAWN** | Enables voice queries for system status — "What's my battery level?", "How much power am I using?". Subscribes to S.T.A.T. MQTT topics. |
| **SPARK** | Complementary power monitoring — SPARK handles armor-segment-level power (SPI-based), while S.T.A.T. handles system-level power (I2C-based). Both publish to MQTT for a unified power picture. |
| **S.C.O.P.E.** | Meta-repo that coordinates all O.A.S.I.S. components. S.T.A.T. is tracked as a submodule and participates in ecosystem-wide standards (foundation files, documentation, deployment). |

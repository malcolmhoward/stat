# CLAUDE.md - LLM Integration Guide

## Project Overview

**S.T.A.T.** (System Telemetry and Analytics Tracker) is the hardware monitoring and telemetry component of the O.A.S.I.S. ecosystem. It collects power, battery, thermal, and system metrics from I2C sensors and Daly BMS, then broadcasts structured telemetry via MQTT for consumption by other components.

## Build System

| Aspect | Details |
|--------|---------|
| Language | C (C99) |
| Build Tool | CMake (3.10+) |
| Target Platform | Linux with I2C support (NVIDIA Jetson, Raspberry Pi) |
| Key Dependencies | Mosquitto (MQTT), json-c |
| Binary | `oasis-stat` |

### Building

```bash
mkdir build && cd build
cmake -DCMAKE_BUILD_TYPE=Release ..
make -j$(nproc)
```

### Installation

```bash
# Binary only
sudo make install

# Full installation (binary + systemd service + config)
sudo ./install.sh
```

### Configuration

- `config/stat.conf`: MQTT settings (host, port, topic) — installed to `/etc/oasis/stat.conf`
- `config/oasis-stat.service`: systemd unit file — installed to `/etc/systemd/system/`
- CLI options override config file values

## Key Files

| File | Purpose |
|------|---------|
| `src/oasis-stat.c` | Main entry point, CLI parsing, event loop |
| `src/ina238.c` / `.h` | INA238 single-channel power monitor driver (I2C) |
| `src/ina3221.c` / `.h` | INA3221 three-channel power monitor driver (sysfs) |
| `src/daly_bms.c` / `.h` | Daly Smart BMS integration (serial) |
| `src/battery_model.c` / `.h` | Battery chemistry models, SOC curves, time estimation |
| `src/ark_detection.c` / `.h` | ARK Electronics Jetson carrier auto-detection (EEPROM) |
| `src/mqtt_publisher.c` / `.h` | MQTT telemetry publishing (JSON payloads) |
| `src/i2c_utils.c` / `.h` | I2C bus utilities |
| `src/cpu_monitor.c` / `.h` | CPU usage tracking |
| `src/memory_monitor.c` / `.h` | Memory usage tracking |
| `src/fan_monitor.c` / `.h` | Fan speed monitoring |
| `src/system_temp_monitor.c` / `.h` | System temperature monitoring |
| `src/logging.c` / `.h` | Logging utilities |
| `include/ina238_registers.h` | INA238 register definitions |

## Architecture

### Data Pipeline

```
I2C Sensors ──→ Driver Layer ──→ Processing ──→ MQTT Publisher
  INA238           ina238.c        battery_model.c    mqtt_publisher.c
  INA3221          ina3221.c       cpu_monitor.c
  EEPROM           ark_detection.c memory_monitor.c
                                   fan_monitor.c
Serial ────→ daly_bms.c ──────→ system_temp_monitor.c
  Daly BMS
```

### Hardware Auto-Detection

S.T.A.T. auto-detects available hardware in this order:
1. ARK Electronics Jetson carrier (EEPROM serial number on I2C)
2. INA3221 via sysfs (`/sys/bus/i2c/drivers/ina3221`)
3. INA238 via direct I2C access
4. Daly BMS on serial ports (if `--bms-enable`)

### Communication (MQTT)

S.T.A.T. publishes JSON telemetry to configurable MQTT topics:

```
S.T.A.T. ──MQTT──→ MIRAGE (HUD displays battery/power)
S.T.A.T. ──MQTT──→ DAWN (voice queries for battery status)
S.T.A.T. ──MQTT──→ Any subscriber (structured JSON)
```

Published data types: battery (INA238), multi-channel power (INA3221), BMS cell data (Daly), system metrics (CPU/memory/fan/temp), unified battery status.

## Hardware Dependencies

| Hardware | Purpose | Interface | Configuration |
|----------|---------|-----------|---------------|
| INA238 | Single-channel power monitor | I2C | Address `0x45`, configurable shunt/current |
| INA3221 | Three-channel power monitor | sysfs (hwmon) | Auto-detected |
| Daly Smart BMS | Cell-level battery monitoring | Serial (UART) | Port, baud rate configurable |
| ARK Electronics carrier | Jetson carrier board | I2C (EEPROM) | Auto-detected on `/dev/i2c-7` |

## O.A.S.I.S. Ecosystem Context

S.T.A.T. is part of the O.A.S.I.S. ecosystem coordinated by [S.C.O.P.E.](https://github.com/malcolmhoward/the-oasis-project-meta-repo):

| Component | Interaction |
|-----------|-------------|
| **MIRAGE** | Displays telemetry on HUD (battery level, power, temperature) |
| **DAWN** | Voice queries for battery status, system health |
| **SPARK** | Complementary power monitoring (armor-level vs system-level) |

For ecosystem-level coordination, see S.C.O.P.E. roadmaps and documentation.

## Common Tasks

### Adding a New Sensor

1. Create `src/new_sensor.c` and `include/new_sensor.h`
2. Add source file to `SOURCES` in `CMakeLists.txt`
3. Add header to `HEADERS` in `CMakeLists.txt`
4. Initialize in `oasis-stat.c` main loop
5. Add MQTT publishing in `mqtt_publisher.c`

### Adding a New Battery Profile

1. Add profile to the battery configurations array in `battery_model.c`
2. Define cells, capacity, chemistry, voltage range
3. Profile is automatically available via `--battery <name>` CLI option

### Adding a New MQTT Data Type

1. Create JSON payload structure in `mqtt_publisher.c`
2. Add `"device"` and `"type"` fields for consumer identification
3. Publish to the configured topic

### Debugging

- I2C: `i2cdetect -y <bus>` to scan for devices
- MQTT: `mosquitto_sub -t stat/#` to monitor published telemetry
- BMS: Check serial port permissions (`dialout` group)
- Service: `journalctl -u oasis-stat` for systemd logs

## Testing

Currently manual testing on hardware. Key verification:
- `i2cdetect` confirms sensor presence
- `mosquitto_sub` confirms MQTT publishing
- `--list-batteries` shows all battery profiles

## Conventions

- C99 standard
- 4-space indentation
- `snake_case` for functions and variables
- `UPPER_CASE` for constants and macros
- Compiler flags: `-Wall -Wextra -Wpedantic`

## Security Considerations

- MQTT: Currently unencrypted on local network — production should use TLS
- I2C/serial: Requires `i2c` and `dialout` group membership
- systemd service: Runs as `oasis` user with `ProtectSystem=full`, `NoNewPrivileges=true`
- No secrets in source code — MQTT config via environment file

---

*For contribution guidelines, see [CONTRIBUTING.md](CONTRIBUTING.md).*
*For ecosystem coordination, see [S.C.O.P.E.](https://github.com/malcolmhoward/the-oasis-project-meta-repo).*

## Branch Naming Convention

**Critical**: Branch names must include the GitHub issue number being addressed.

### Format
```
feat/<component>/<issue#>-<short-description>
```

### Before Creating a Branch

1. **Identify the issue** you're working on (check GitHub Issues)
2. **Use that issue's number** in the branch name
3. **Verify** the issue number matches the work being done

### Examples
```bash
# Check available issues first
gh issue list

# Create branch with correct issue number
git checkout -b feat/stat/<issue#>-description
```

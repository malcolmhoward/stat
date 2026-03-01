# S.T.A.T. Docker Guide

Docker configurations for building and running S.T.A.T. on multiple platforms.

## Overview

S.T.A.T. provides three platform-specific Dockerfiles:

| Dockerfile | Base Image | Target Platform | Use Case |
|-----------|-----------|-----------------|----------|
| `Dockerfile.dev` | `ubuntu:22.04` | x86/x64 | Development, CI/CD, build verification |
| `Dockerfile.jetson` | `nvcr.io/nvidia/l4t-base:r35.4.1` | NVIDIA Jetson | Production with I2C sensor access |
| `Dockerfile.rpi` | `arm64v8/debian:bookworm-slim` | Raspberry Pi (ARM64) | Production with I2C sensor access |

Each Dockerfile is fully self-contained — see [ADR-0005](https://github.com/malcolmhoward/the-oasis-project-meta-repo/blob/main/coordination/decisions/adr/0005-dockerfile-independence.md) for the rationale.

> **Note**: These containers build the actual S.T.A.T. C application. Mock hardware (simulated I2C sensors) is not yet supported — see the [O.A.S.I.S. meta-repo](https://github.com/malcolmhoward/the-oasis-project-meta-repo) for mock hardware plans (meta-issue #30).

## Prerequisites

### All Platforms

- Docker installed and running
- S.T.A.T. source code cloned

### NVIDIA Jetson

1. **JetPack SDK** installed on host
2. **NVIDIA Docker runtime**:
   ```bash
   sudo apt-get update
   sudo apt-get install -y docker.io nvidia-docker2
   sudo systemctl restart docker
   ```
3. **I2C configured** (verify with `ls /dev/i2c-*`)

### Raspberry Pi

1. **64-bit OS** required (Raspberry Pi OS 64-bit or Ubuntu for Pi)
2. **Docker installed**:
   ```bash
   curl -fsSL https://get.docker.com | sh
   sudo usermod -aG docker $USER  # re-login after
   ```
3. **I2C enabled**: `sudo raspi-config` → Interface Options → I2C → Enable
4. **I2C bus** typically at `/dev/i2c-1` (verify with `ls /dev/i2c-*`)

## Building

```bash
# Development (x86/x64)
docker build -f Dockerfile.dev -t stat:dev .

# NVIDIA Jetson (build on Jetson hardware)
docker build -f Dockerfile.jetson -t stat:jetson .

# Raspberry Pi (build on Pi hardware)
docker build -f Dockerfile.rpi -t stat:rpi .
```

## Running

### Development Container

```bash
docker run --rm -it stat:dev
```

The dev container starts Mosquitto in the background and drops to a shell with the built S.T.A.T. binary at `/opt/stat/build/oasis-stat`.

### Jetson Container

```bash
# ARK Electronics carrier (I2C bus 7)
docker run --rm -it \
  --network host \
  --device=/dev/i2c-7 \
  stat:jetson

# Generic Jetson (I2C bus 1)
docker run --rm -it \
  --network host \
  --device=/dev/i2c-1 \
  stat:jetson

# With Daly BMS serial access
docker run --rm -it \
  --network host \
  --device=/dev/i2c-7 \
  --device=/dev/ttyTHS1 \
  stat:jetson

# Override MQTT settings
docker run --rm -it \
  --network host \
  --device=/dev/i2c-7 \
  -e MQTT_HOST=192.168.1.100 \
  -e MQTT_PORT=1883 \
  -e MQTT_TOPIC=oasis/stat \
  stat:jetson
```

### Raspberry Pi Container

```bash
# Standard RPi I2C bus
docker run --rm -it \
  --network host \
  --device=/dev/i2c-1 \
  stat:rpi

# With Daly BMS serial access
docker run --rm -it \
  --network host \
  --device=/dev/i2c-1 \
  --device=/dev/ttyAMA0 \
  stat:rpi

# Override MQTT settings
docker run --rm -it \
  --network host \
  --device=/dev/i2c-1 \
  -e MQTT_HOST=192.168.1.100 \
  -e MQTT_PORT=1883 \
  -e MQTT_TOPIC=oasis/stat \
  stat:rpi
```

> **Note**: ARK Electronics carrier auto-detection does not apply on Raspberry Pi. The container uses standard I2C bus 1.

## I2C Device Access

S.T.A.T. requires I2C access for power monitor sensors. In Docker, this requires `--device` flags to pass through specific I2C buses from the host.

| Platform | I2C Bus | Sensors | Flag |
|----------|---------|---------|------|
| ARK Electronics Jetson carrier | `/dev/i2c-7` | INA238, EEPROM | `--device=/dev/i2c-7` |
| Generic Jetson / Raspberry Pi | `/dev/i2c-1` | INA238, INA3221 | `--device=/dev/i2c-1` |

To verify I2C devices on the host before running:
```bash
# List available I2C buses
ls /dev/i2c-*

# Scan for devices (ARK carrier)
i2cdetect -y 7

# Scan for devices (generic)
i2cdetect -y 1
```

## Multi-Component Development

For running S.T.A.T. alongside other O.A.S.I.S. components (MIRAGE, DAWN), use the `docker-compose.yml` in the [S.C.O.P.E. meta-repo](https://github.com/malcolmhoward/the-oasis-project-meta-repo). The compose configuration orchestrates multiple component containers with shared MQTT networking.

## Platform Differences

| Feature | Dockerfile.dev | Dockerfile.jetson | Dockerfile.rpi |
|---------|---------------|-------------------|----------------|
| Architecture | x86/x64 | ARM64 (Tegra) | ARM64 (RPi) |
| Base image | ubuntu:22.04 | nvcr.io/nvidia/l4t-base | arm64v8/debian:bookworm-slim |
| I2C access | Not available (no hardware) | Pass-through via `--device` | Pass-through via `--device` |
| Serial access | Not available | Pass-through for Daly BMS | Pass-through for Daly BMS |
| ARK carrier detection | N/A | Auto-detects bus 7 | Not applicable |
| Use case | Build verification, CI | Production telemetry (Jetson) | Production telemetry (RPi) |
| Entrypoint | Interactive shell | `oasis-stat --service` | `oasis-stat --service` |

## Troubleshooting

### Permission Denied on I2C Devices

```bash
# On the host, verify device permissions
ls -l /dev/i2c-*

# Add user to i2c group (then re-login)
sudo usermod -aG i2c $USER
```

### No I2C Devices in Container

Ensure you pass the correct `--device` flag. The dev container does not have I2C access — it is for build verification only.

### MQTT Connection Refused

The containers start Mosquitto locally. For external brokers, override with environment variables:
```bash
docker run --rm -it \
  --network host \
  -e MQTT_HOST=your-broker-host \
  stat:jetson
```

### Build Failures

1. Verify network access for `apt-get` during build
2. Check that CMakeLists.txt is not excluded by `.dockerignore`
3. Review build output: `docker build --no-cache -f Dockerfile.dev -t stat-dev .`

## Docker vs Native

**Quick guidance**:
- **Use Docker** for development, CI/CD, and deployments where containerization is preferred
- **Use native** (`./install.sh`) for production on dedicated hardware where direct I2C access and systemd integration are needed

The native install script (`install.sh`) sets up the systemd service, creates the `oasis` user, configures group permissions, and installs the config file — things that are handled differently in a container environment.

## Security Considerations

- Use specific `--device` flags instead of `--privileged` where possible
- The Jetson container runs as root by default — consider adding a non-root user for production
- MQTT is unencrypted by default — use TLS in production
- The `stat.conf` file contains no secrets (only MQTT host/port/topic)

---

*Part of the [O.A.S.I.S. Project](https://github.com/The-OASIS-Project). For ecosystem orchestration, see [S.C.O.P.E.](https://github.com/malcolmhoward/the-oasis-project-meta-repo).*

# Debian Package Build System

Build Debian packages for UGREEN LED controller in Docker.

This document covers **building packages**, not installing them on the NAS. For the manual Debian 13 installation, kernel headers, DKMS verification, I2C initialization, and service startup, see the [Debian 13 installation guide](../../debian13-installation.md). Package installation can configure and start systemd services; review the package scripts before installing on an active system.

## Quick Start

Build all packages for Debian Bookworm (default):
```bash
./docker-run.sh
```

Packages will be created in `./build/` directory.

## Build Options

**Custom Debian version:**
```bash
DEBIAN_VERSION=bullseye ./docker-run.sh
DEBIAN_VERSION=trixie ./docker-run.sh
```

**Selective package builds:**
```bash
# Build only one package
./docker-run.sh --only-dkms
./docker-run.sh --only-cli
./docker-run.sh --only-utils
./docker-run.sh --only-systemd

# Build all except one
./docker-run.sh --skip-systemd
```

**Build a specific git ref:** `docker-run.sh` copies the current checkout and uses the local build mode. Select the desired ref in a separate clean checkout before running it; passing a ref argument does not switch the copied checkout in that mode.

## Output Files

After building, you'll find these packages in `./build/`:
- `led-ugreen-dkms_0.3-1_all.deb` (architecture-independent DKMS source package)
- `led-ugreen-cli_0.3_amd64.deb`
- `led-ugreen-utils_0.3_amd64.deb`
- `led-ugreen-systemd_0.3_all.deb`

## Package Structure

Four packages are built:

### 1. led-ugreen-dkms
Kernel module with DKMS support.
- Configured to rebuild on kernel updates when matching headers and a working build environment are available
- Creates module autoload configuration
- **No dependencies** (except dkms)

### 2. led-ugreen-cli
Command-line interface tool for manual LED control.
- Standalone binary
- Conflicts with kernel module (can't use simultaneously)
- **No dependencies** on other LED packages

### 3. led-ugreen-utils
Monitoring scripts and compiled utilities.
- Disk activity monitoring
- Network activity monitoring (single and multi-interface)
- Power LED control
- Hardware probing
- **Depends on:** led-ugreen-dkms

### 4. led-ugreen-systemd
Systemd service files and auto-configuration.
- Interactive installation with debconf
- Auto-detects disk mapping
- Auto-detects network interfaces
- Auto-enables services
- **Depends on:** led-ugreen-utils, led-ugreen-dkms

## Testing

Test package installation in Docker:
```bash
./build-and-test.sh
```

This builds all packages and tests installation in a clean Debian container.

## Platform Support

The DKMS package (`led-ugreen-dkms`) is architecture-independent (`all`) as it contains only source code.

The CLI and utils packages are built for **amd64** (x86_64); the systemd package is architecture-independent (`all`).

The build system uses `--platform linux/amd64` to ensure correct architecture even when building on ARM-based Macs.

## Supported Debian Versions

- Debian 11 (Bullseye)
- Debian 12 (Bookworm) - default
- Debian 13 (Trixie)

## Requirements

- Docker
- Bash

No other dependencies needed on the host system.

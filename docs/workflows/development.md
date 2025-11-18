# Development Workflow Documentation

## Overview

This document describes the typical development workflows for working with the PlutoSDR firmware, from setup through testing and deployment.

## Developer Setup Workflow

### Step 1: Clone Repository

```bash
# Clone with all submodules
git clone --recursive https://github.com/analogdevicesinc/plutosdr-fw.git
cd plutosdr-fw

# Verify submodule status
git submodule status

# Expected output:
#  [hash] u-boot-xlnx (detached at ...)
#  [hash] buildroot (HEAD -> ...)
#  [hash] linux (HEAD -> ...)
#  [hash] hdl (HEAD -> ...)
```

### Step 2: Configure Environment

```bash
# Source environment setup
source setup_env.sh

# Follow displayed instructions
export VIVADO_SETTINGS=/opt/Xilinx/Vivado/2023.2/settings64.sh
source $VIVADO_SETTINGS
export CROSS_COMPILE=arm-linux-gnueabihf-

# Or save to .bashrc for persistence
cat >> ~/.bashrc << 'EOF'
# PlutoSDR Build Environment
if [ -f ~/plutosdr-fw/setup_env.sh ]; then
    source ~/plutosdr-fw/setup_env.sh
fi
EOF
```

### Step 3: First-Time Build

```bash
# Build toolchain (one-time, ~30 minutes)
make TOOLCHAIN

# Full firmware build
make all

# This creates:
# ├─ build/pluto.frm (main firmware)
# ├─ build/pluto.dfu (DFU format)
# ├─ build/boot.bin (bootloader)
# └─ [other artifacts]
```

## Incremental Development Workflow

### Scenario: Modify Linux Kernel

```
Developer Goal: Change kernel configuration or add driver
│
├─ Step 1: Modify Linux source
│  └─ Edit: linux/drivers/iio/...
│  └─ Or: linux/arch/arm/configs/zynq_pluto_defconfig
│
├─ Step 2: Rebuild kernel only
│  └─ make -C linux -j$(nproc) zImage
│  └─ Time: ~5-10 minutes
│
├─ Step 3: Rebuild rootfs with kernel
│  └─ make build/rootfs.cpio.gz
│  └─ Time: ~2-3 minutes
│
├─ Step 4: Rebuild firmware
│  └─ make build/pluto.dfu
│  └─ Time: ~1-2 minutes
│
└─ Step 5: Test on hardware
   └─ Deploy via DFU or MSD
   └─ See: Deployment Methods guide
```

### Scenario: Modify FPGA Design

```
Developer Goal: Change RF signal processing in FPGA
│
├─ Step 1: Modify HDL source
│  └─ Edit: hdl/projects/pluto/system_bd.tcl
│  └─ Or: hdl/library/axi_ad9361/*.v
│
├─ Step 2: Rebuild FPGA
│  └─ make -C hdl/projects/pluto
│  └─ Time: ~15-20 minutes (P&R intensive)
│
├─ Step 3: Regenerate FSBL
│  └─ xsct scripts/create_fsbl_project.tcl
│  └─ Time: ~2-3 minutes
│
├─ Step 4: Rebuild boot image
│  └─ make build/boot.bin
│  └─ Time: <1 minute
│
├─ Step 5: Rebuild firmware
│  └─ make build/pluto.dfu
│  └─ Time: ~1-2 minutes
│
└─ Step 6: Test on hardware
   └─ Deploy via DFU or MSD
   └─ Monitor with: dmesg, iio_info
```

### Scenario: Modify U-Boot

```
Developer Goal: Change bootloader behavior or add command
│
├─ Step 1: Modify U-Boot source
│  └─ Edit: u-boot-xlnx/board/xilinx/zynq/
│  └─ Or: u-boot-xlnx/configs/zynq_pluto_defconfig
│
├─ Step 2: Rebuild U-Boot
│  └─ make -C u-boot-xlnx -j$(nproc)
│  └─ Time: ~3-5 minutes
│
├─ Step 3: Regenerate boot image
│  └─ bootgen -image scripts/pluto.bif -o build/boot.bin
│  └─ Time: <1 minute
│
├─ Step 4: Rebuild firmware
│  └─ make build/pluto.dfu
│  └─ Time: ~1-2 minutes
│
└─ Step 5: Test bootloader behavior
   └─ Use JTAG bootstrap for testing
   └─ Serial console access via: picocom /dev/ttyUSB0
```

## Testing Workflow

### Unit Testing

```bash
#!/bin/bash
# Test individual firmware components

# 1. Build-time checks
echo "Running build-time checks..."
make clean
make all 2>&1 | tee build.log

# Parse build warnings
grep -i warning build.log | head -20

# Check for errors
if grep -qi error build.log; then
    echo "Build errors detected!"
    exit 1
fi

# 2. Verify firmware artifacts
echo "Verifying firmware artifacts..."
ls -lah build/pluto.{frm,dfu}
sha256sum build/pluto.{frm,dfu}

# 3. Inspect FIT image contents
echo "Inspecting FIT image..."
u-boot-xlnx/tools/mkimage -l build/pluto.itb
```

### Hardware Integration Testing

```bash
#!/bin/bash
# Test firmware on actual PlutoSDR hardware

DEVICE_IP="192.168.2.1"
DEVICE_USER="root"
DEVICE_PASS="analog"

# 1. Deploy firmware
./download_and_test.sh

# 2. Verify device boots
sleep 5
if ping -c 1 $DEVICE_IP >/dev/null; then
    echo "✓ Device online"
else
    echo "✗ Device not responding"
    exit 1
fi

# 3. Check device version
sshpass -p "$DEVICE_PASS" ssh -o StrictHostKeyChecking=no \
    $DEVICE_USER@$DEVICE_IP \
    "cat /sys/firmware/devicetree/base/model"
# Expected: "PlutoSDR" or similar

# 4. Test USB IIO interface
iio_info | grep "ad9361" && echo "✓ RF transceiver accessible"

# 5. Test basic RF operation
iio_attr -r ad9361-phy frequency | grep "AD9361_TX"

# 6. Check system logs for errors
sshpass -p "$DEVICE_PASS" ssh -o StrictHostKeyChecking=no \
    $DEVICE_USER@$DEVICE_IP \
    "dmesg | tail -20"
```

## Git Workflow

### Feature Development

```bash
# Create feature branch from main
git checkout -b feature/my-feature main

# Make changes
vim linux/drivers/iio/...

# Test changes
make clean && make all

# Commit changes
git add -A
git commit -m "feat: add new RF feature

- Detailed description of changes
- Benefits or improvements
- Any breaking changes
"

# Push feature branch
git push origin feature/my-feature

# Create pull request on GitHub
# (for code review)
```

### Bug Fix Workflow

```bash
# Create bugfix branch
git checkout -b bugfix/issue-123 main

# Apply minimal fix
vim scripts/get_default_envs.sh

# Test fix thoroughly
make clean && make all
./download_and_test.sh

# Commit with issue reference
git commit -m "fix: resolve issue #123 in environment extraction

Fixes #123
- Root cause: ...
- Solution: ...
- Testing: Verified with ...
"

# Push and create PR
git push origin bugfix/issue-123
```

### Update Submodules

```bash
# Update all submodules to latest upstream
git submodule update --remote

# Or update specific submodule
cd linux && git pull origin 2018_R1

# Commit submodule updates
cd ..
git add linux
git commit -m "chore: update Linux kernel to latest 2018_R1"

# Push submodule changes
git push origin
```

## CI/CD Workflow

### Pre-commit Checks

```bash
#!/bin/bash
# Pre-commit hook (save as .git/hooks/pre-commit)
#!/bin/bash

echo "Running pre-commit checks..."

# 1. Verify build still works
echo "Building firmware..."
make clean
make all -j$(nproc) || {
    echo "Build failed!"
    exit 1
}

# 2. Check for large files
echo "Checking for large files..."
git diff --cached --name-only | while read file; do
    SIZE=$(wc -c < "$file")
    if [ $SIZE -gt 10485760 ]; then  # 10 MB
        echo "Warning: Large file committed: $file"
    fi
done

# 3. Verify no credentials in commits
echo "Scanning for credentials..."
git diff --cached | grep -E "password|secret|token" && {
    echo "Error: Credentials detected in commit!"
    exit 1
}

echo "Pre-commit checks passed!"
exit 0
```

### CI/CD Pipeline (GitHub Actions Example)

```yaml
# .github/workflows/build.yml
name: Build and Test

on: [push, pull_request]

jobs:
  build:
    runs-on: ubuntu-latest

    steps:
    - uses: actions/checkout@v2
      with:
        submodules: 'recursive'

    - name: Install dependencies
      run: |
        sudo apt-get update
        sudo apt-get install -y \
          build-essential \
          gcc-arm-linux-gnueabihf \
          device-tree-compiler

    - name: Setup environment
      run: |
        source setup_env.sh || true
        export PATH=/opt/Xilinx/Vivado/2023.2/bin:$PATH || true

    - name: Build firmware
      run: |
        make TOOLCHAIN
        make all

    - name: Verify artifacts
      run: |
        ls -lah build/pluto.{frm,dfu,dfu}
        sha256sum build/*.dfu

    - name: Upload artifacts
      uses: actions/upload-artifact@v2
      with:
        name: firmware-artifacts
        path: build/pluto.*.dfu

    - name: Upload to release
      if: startsWith(github.ref, 'refs/tags/')
      run: |
        python3 CI/upload_to_artifactory.py \
          --server_path="pluto/releases/${{ github.ref_name }}" \
          --local_path="build/" \
          --token="${{ secrets.ARTIFACTORY_TOKEN }}"
```

## Documentation Workflow

### Update Documentation After Changes

```bash
# When code changes, update corresponding docs

# 1. Edit source code
vim hdl/projects/pluto/system_bd.tcl

# 2. Update architecture documentation
vim docs/architecture/fpga-design.md

# 3. Update component documentation
vim docs/components/fpga-hdl.md

# 4. Commit with documentation
git add hdl/projects/pluto/system_bd.tcl docs/
git commit -m "feat: update FPGA signal processing

- Added new FIR filter stage for improved SNR
- Updated block diagram in architecture docs
- Updated signal flow in component docs
"
```

## Performance Profiling Workflow

### Identify Build Bottlenecks

```bash
#!/bin/bash
# Profile build time

echo "Profiling build stages..."

# Time each major stage
stages=(
    "TOOLCHAIN"
    "u-boot-xlnx/u-boot"
    "linux/arch/arm/boot/zImage"
    "build/rootfs.cpio.gz"
    "hdl/projects/pluto"
    "build/boot.bin"
    "build/pluto.itb"
)

for stage in "${stages[@]}"; do
    echo "Building: $stage"
    START=$(date +%s)
    make $stage -j$(nproc)
    END=$(date +%s)
    echo "  Time: $((END - START)) seconds"
    echo ""
done
```

### Optimize Parallel Builds

```bash
# Test different parallelization levels
for NCORES in 1 2 4 8 16; do
    echo "Testing with $NCORES cores..."
    START=$(date +%s)
    make clean && make -j$NCORES all
    END=$(date +%s)
    echo "  Time: $((END - START)) seconds"
done

# Results show optimal core count
# (typically near physical core count)
```

## Deployment Workflow for Releases

```bash
#!/bin/bash
# Release preparation and deployment

VERSION="0.39"

# 1. Update version in code
sed -i "s/VERSION=.*/VERSION=$VERSION/" Makefile

# 2. Build final firmware
make clean
make all

# 3. Generate checksums
sha256sum build/pluto.*.dfu > build/SHA256SUMS
gpg --detach-sign build/SHA256SUMS

# 4. Create release notes
cat > RELEASE_NOTES.md << EOF
# PlutoSDR Firmware v$VERSION

## Features
- [List new features]

## Bug Fixes
- [List bugs fixed]

## Known Issues
- [Any known issues]

## Download
- [Link to download]

## Installation
See [deployment documentation](docs/build-system/deployment.md)
EOF

# 5. Tag release
git tag -a "v$VERSION" -m "Release v$VERSION"

# 6. Upload to artifactory
python3 CI/upload_to_artifactory.py \
    --server_path="pluto/releases/v$VERSION" \
    --local_path="build/" \
    --properties="git_sha=$(git rev-parse --short HEAD)"

# 7. Publish release notes
# (Manual: GitHub releases page)
```

## Related Documentation

- [Build Process](../build-system/build-process.md) - Build system details
- [Deployment Methods](../build-system/deployment.md) - Firmware update methods
- [Scripts and Automation](../components/scripts-automation.md) - Available scripts

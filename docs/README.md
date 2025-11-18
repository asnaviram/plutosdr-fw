# PlutoSDR Firmware Documentation

Complete technical documentation for the PlutoSDR firmware project, covering architecture, components, build systems, and workflows.

## Documentation Structure

- **[Architecture](./architecture/)** - System architecture, block diagrams, and design overview
  - [System Overview](./architecture/system-overview.md)
  - [Hardware Architecture](./architecture/hardware-architecture.md)
  - [FPGA Design](./architecture/fpga-design.md)

- **[Components](./components/)** - Detailed component documentation
  - [Build System](./components/build-system.md)
  - [Hardware Configuration](./components/hardware-configuration.md)
  - [Scripts and Automation](./components/scripts-automation.md)
  - [FPGA and HDL](./components/fpga-hdl.md)

- **[Build System](./build-system/)** - Build process and compilation guide
  - [Build Process](./build-system/build-process.md)
  - [Build Targets](./build-system/build-targets.md)
  - [Deployment Methods](./build-system/deployment.md)

- **[Workflows](./workflows/)** - Development and operational workflows
  - [Development Workflow](./workflows/development.md)
  - [Firmware Update Process](./workflows/firmware-update.md)
  - [CI/CD Integration](./workflows/cicd.md)

- **[API Reference](./api-reference/)** - API and interface documentation
  - [Device Interfaces](./api-reference/device-interfaces.md)
  - [Boot Sequence](./api-reference/boot-sequence.md)

## Quick Start

For a quick overview, start with [System Overview](./architecture/system-overview.md).

For build instructions, see [Build Process](./build-system/build-process.md).

For deployment, see [Deployment Methods](./build-system/deployment.md).

## Project Information

**Repository**: PlutoSDR Firmware (plutosdr-fw)
**Targets**: PlutoSDR (RevA/B/C), SidekiqZ2
**Platform**: Xilinx Zynq-7010
**Toolchain**: Vivado 2023.2, ARM GCC 7.3, Buildroot

## Key Components

- **Bootloader**: U-Boot (Xilinx variant)
- **OS**: Linux Kernel 2018 R1
- **Filesystem**: Buildroot-based minimal rootfs
- **FPGA Design**: Vivado HDL project with AD9361 RF transceiver
- **RF Transceiver**: AD9361 with IIO driver support

## Documentation Generation Date

This documentation was automatically generated with comprehensive analysis of the codebase.

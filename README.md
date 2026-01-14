# Project 22: PCIe XDMA Test Pattern Generator

## Part of FPGA Trading Systems Portfolio

This project is part of a complete end-to-end trading system:
- **Main Repository:** [fpga-trading-systems](https://github.com/adilsondias-engineer/fpga-trading-systems)
- **Project Number:** 22 of 30
- **Category:** FPGA Core
- **Dependencies:** Project 21 (PCIe infrastructure)

---

## Overview

PCIe Gen2 test pattern generator for verifying XDMA C2H streaming. This project provides a minimal PCIe design for validating the XDMA driver and host-side DMA operations before integrating with the full trading pipeline.

Key features:
- PCIe Gen2 x1/x4 support (5.0 GT/s)
- Continuous AXI-Stream test pattern generation
- XDMA C2H streaming mode verification
- Driver and host application testing

## Architecture

```
+-------------------+
|   Host PC         |
|   (Linux)         |
|                   |
|  /dev/xdma0_c2h_0 |
+--------+----------+
         |
         | PCIe Gen2 x1/x4
         v
+-------------------+     +------------------------+
|   XDMA IP Core    | <-- | axi_stream_test_pattern|
|   (C2H Stream)    |     | (250 MHz axi_aclk)     |
+-------------------+     +------------------------+
         |                          |
         v                          v
    S_AXIS_C2H               m_axis_tvalid = '1'
    (64-bit)                 (continuous)
```

Key difference from Project 23: Test pattern runs directly in XDMA clock domain (250 MHz axi_aclk) with no CDC crossing required.

## XDMA Configuration

```
PCIe Link:
  - Max Link Speed: 5.0 GT/s (Gen2)
  - Max Link Width: X4
  - Device ID: 0x7024
  - Vendor ID: 0x10EE (Xilinx)

AXI Interface:
  - Data Width: 64-bit
  - Clock Frequency: 250 MHz
  - Mode: AXI-Stream (C2H streaming)
```

## Test Pattern Format

The test pattern generator outputs 48-byte packets matching the BBO format:

| Offset | Size | Field | Value |
|--------|------|-------|-------|
| 0-7 | 8 | Symbol | "TESTAAPL" |
| 8-11 | 4 | Bid Price | Incrementing counter |
| 12-15 | 4 | Bid Size | Fixed value |
| 16-19 | 4 | Ask Price | Incrementing counter + offset |
| 20-23 | 4 | Ask Size | Fixed value |
| 24-27 | 4 | Spread | Fixed offset |
| 28-47 | 20 | Timestamps/Pad | Zeros |

## Linux Driver Setup (CRITICAL)

### Driver Selection

**IMPORTANT**: Linux has TWO different `xdma` drivers. You MUST use the correct one:

| Driver | Type | Purpose | Device Nodes |
|--------|------|---------|--------------|
| **Kernel built-in** `xdma.ko` | Platform driver | DMA engines in SoCs (Zynq, etc.) | None for PCIe |
| **Xilinx dma_ip_drivers** `xdma.ko` | PCIe driver | PCIe endpoint DMA (what we need!) | `/dev/xdma*` |

### Check Current Driver

```bash
# Check if wrong driver is loaded
lsmod | grep xdma
modinfo xdma 2>/dev/null | grep -E "filename|description"

# If you see "platform" or no PCIe mentions, it's the WRONG driver
```

### Install Correct XDMA Driver

```bash
# 1. Clone Xilinx dma_ip_drivers (if not already done)
cd /work/projects
git clone https://github.com/Xilinx/dma_ip_drivers

# 2. Build the PCIe XDMA driver
cd dma_ip_drivers/XDMA/linux-kernel/xdma
make clean
make

# 3. Unload wrong driver (if loaded)
sudo rmmod xdma 2>/dev/null

# 4. Load correct driver
sudo insmod /work/projects/dma_ip_drivers/XDMA/linux-kernel/xdma/xdma.ko

# 5. Verify device nodes created
ls -la /dev/xdma*
```

### Verification Checklist

```bash
# 1. Check PCIe device is detected
lspci -d 10ee:
# Expected: XX:00.0 Memory controller: Xilinx Corporation Device 7024

# 2. Check driver is bound
lspci -d 10ee: -k
# Expected: Kernel driver in use: xdma

# 3. Check device nodes exist
ls /dev/xdma0_* | head -5
# Expected: /dev/xdma0_c2h_0, /dev/xdma0_h2c_0, /dev/xdma0_control, etc.

# 4. Test register access (read XDMA identifier)
/work/projects/dma_ip_drivers/XDMA/linux-kernel/tools/reg_rw /dev/xdma0_control 0x0 w
# Expected: 0x1fc00006 (XDMA block identifier)

# 5. Test DMA read (should return data if FPGA is streaming)
dd if=/dev/xdma0_c2h_0 of=/tmp/test.bin bs=64 count=1
xxd /tmp/test.bin
```

### Build XDMA Test Tools

```bash
cd /work/projects/dma_ip_drivers/XDMA/linux-kernel/tools
make

# Available tools:
# - reg_rw        : Read/write XDMA registers
# - dma_to_device : H2C DMA test (host → FPGA)
# - dma_from_device: C2H DMA test (FPGA → host)
# - performance   : DMA bandwidth test
```

### Troubleshooting

| Symptom | Cause | Solution |
|---------|-------|----------|
| `lspci -d 10ee:` returns nothing | No power or bitstream not loaded | Connect 12V power, program FPGA |
| `/dev/xdma*` not created | Wrong driver loaded | Unload kernel driver, load dma_ip_drivers version |
| `reg_rw` returns all 0xFF | PCIe link not trained | Check power, rescan PCIe bus |
| DMA read hangs | FPGA not streaming data | Check FPGA design has data source |
| Permission denied | Need root or udev rules | `sudo` or add udev rule (see below) |

### Udev Rules (Optional)

Create `/etc/udev/rules.d/99-xdma.rules` for non-root access:

```
# Xilinx XDMA PCIe devices
SUBSYSTEM=="xdma", MODE="0666", GROUP="xdma"
```

Then: `sudo udevadm control --reload-rules && sudo udevadm trigger`

### Device Files

```bash
# DMA channels
/dev/xdma0_c2h_0    # Card-to-Host (BBO stream)
/dev/xdma0_h2c_0    # Host-to-Card (commands)
/dev/xdma0_user     # Control registers (mmap)

# Events/Interrupts
/dev/xdma0_events_0 # BBO available interrupt
```

### Example Usage (C++)

```cpp
#include <fcntl.h>
#include <sys/mman.h>
#include <unistd.h>

// Open DMA channel for BBO stream
int fd_c2h = open("/dev/xdma0_c2h_0", O_RDONLY);

// Read BBO update (36 bytes)
BBOData bbo;
ssize_t n = read(fd_c2h, &bbo, sizeof(BBOData));

// Memory-mapped control registers
int fd_user = open("/dev/xdma0_user", O_RDWR);
uint32_t* ctrl = (uint32_t*)mmap(NULL, 4096,
    PROT_READ | PROT_WRITE, MAP_SHARED, fd_user, 0);

// Enable BBO streaming
ctrl[1] = 0x01;  // CONTROL register, bit 0 = enable

// Read status
uint32_t status = ctrl[2];  // STATUS register
```

## Build Instructions

### FPGA Bitstream

```bash
cd 21-pcie-gpu-bridge

# Step 1: Create block design with XDMA IP
vivado -mode batch -source scripts/create_block_design.tcl

# Step 2: Add custom BBO stream logic
vivado -mode batch -source scripts/add_custom_logic.tcl

# Step 3: Run implementation and generate bitstream (from Vivado GUI or TCL)
vivado vivado_project/pcie_gpu_bridge.xpr
# In Vivado: Run Synthesis → Run Implementation → Generate Bitstream
```

### Linux Driver

```bash
# Clone Xilinx XDMA driver
git clone https://github.com/Xilinx/dma_ip_drivers
cd dma_ip_drivers/XDMA/linux-kernel/xdma

# Build and install
make
sudo make install
sudo modprobe xdma
```

### Test Application

```bash
cd test
make
./pcie_loopback_test
```

## Files

```
21-pcie-gpu-bridge/
├── src/
│   ├── pcie_bbo_top.vhd          # Top-level PCIe BBO wrapper (integrates all modules)
│   ├── bbo_axi_stream.vhd        # BBO to 128-bit AXI-Stream converter
│   ├── bbo_cdc_fifo.vhd          # Clock domain crossing FIFO (200 MHz → XDMA)
│   ├── control_registers.vhd     # AXI-Lite slave (config & status)
│   ├── latency_calculator.vhd    # 4-point latency measurement (min/max/last)
│   └── xdma_wrapper.cpp          # C++ XDMA wrapper class
├── include/
│   ├── pcie_types.h              # C++ type definitions
│   └── xdma_wrapper.h            # XDMA C++ wrapper class
├── constraints/
│   └── ax7203_pcie.xdc           # PCIe pin constraints
├── scripts/
│   ├── create_block_design.tcl   # Block design generation (XDMA + AXI infrastructure)
│   ├── add_custom_logic.tcl      # Add custom BBO logic to block design
│   └── install_xdma_driver.sh    # Linux driver installation script
├── driver/
│   └── xdma_patches/             # Any Linux-specific patches
├── test/
│   ├── tb_bbo_axi_stream.vhd     # Testbench: AXI-Stream converter
│   ├── tb_bbo_cdc_fifo.vhd       # Testbench: CDC FIFO
│   ├── pcie_loopback_test.cpp    # Basic loopback test
│   └── Makefile
└── docs/
    └── pcie_integration_notes.md # Detailed integration notes
```

## Custom Logic Architecture

```
┌───────────────────────────────────────────────────────────────────────────────┐
│                           pcie_bbo_top                                         │
│  ┌─────────────────────────────────────────────────────────────────────────┐  │
│  │                                                                         │  │
│  │  ┌─────────────┐    ┌─────────────────┐    ┌───────────────────────┐  │  │
│  │  │ bbo_cdc_fifo│───▶│ bbo_axi_stream  │───▶│  AXI-Stream to C2H   │  │  │
│  │  │ (CDC 200→   │    │ (128-bit beats) │    │  (Block Design FIFO) │  │  │
│  │  │  XDMA clk)  │    │                 │    │                      │  │  │
│  │  └──────▲──────┘    └─────────────────┘    └───────────────────────┘  │  │
│  │         │                    │                                         │  │
│  │         │           ┌────────▼────────┐                               │  │
│  │  From   │           │latency_calculator│                               │  │
│  │  Order  │           │ (min/max/last)  │                               │  │
│  │  Book   │           └────────┬────────┘                               │  │
│  │         │                    │                                         │  │
│  │         │           ┌────────▼────────┐    ┌───────────────────────┐  │  │
│  │         │           │control_registers│◀───│  AXI-Lite from Host   │  │  │
│  │         │           │ (AXI-Lite slave)│    │  (via XDMA M_AXI_LITE)│  │  │
│  │         │           └─────────────────┘    └───────────────────────┘  │  │
│  │         │                                                              │  │
│  └─────────┼──────────────────────────────────────────────────────────────┘  │
│            │                                                                  │
│  Trading   │  clk_trading (200 MHz)                                          │
│  Logic ────┴──────────────────────────────────────────────────────────────── │
└───────────────────────────────────────────────────────────────────────────────┘
```

## Integration with Project 20

This project extends Project 20 (Order Book) by adding PCIe output alongside the existing UDP output:

1. **BBO FIFO Output**: Connects to both UDP TX (existing) and PCIe C2H (new)
2. **Control Interface**: Replaces/augments UART configuration with PCIe
3. **Shared Trading Logic**: Order book and BBO detection unchanged

## Hardware Setup (CRITICAL)

### Power Requirements

**IMPORTANT**: The AX7203 requires external 12V power for PCIe operation!

The GTP transceivers used for PCIe draw ~1-1.5W additional power compared to
non-GTP designs (like Project 20 Ethernet). Without external power:
- LEDs appear "half-bright"
- PCIe device not detected (`lspci -d 10ee:` returns nothing)
- XDMA driver won't load

**Power options (diode-OR on board):**
1. **CN1**: External 12V DC barrel jack
2. **J14**: ATX PSU 4-pin Molex connector
3. **PCIe slot**: 12V from PCIe edge connector

For reliable operation, connect CN1 or J14 in addition to PCIe slot power.

### GTP Lane Constraints (CRITICAL)

The XDMA IP auto-generates GTP lane constraints with the **WRONG order** for AX7203.
This is handled automatically by the TCL script, but for manual builds:

**Auto-generated (WRONG for AX7203):**
```
Lane 0 → X0Y7
Lane 1 → X0Y6
Lane 2 → X0Y5
Lane 3 → X0Y4
```

**AX7203 correct order:**
```
Lane 0 → X0Y5
Lane 1 → X0Y4
Lane 2 → X0Y6
Lane 3 → X0Y7
```

**Solution:**
1. After `generate_target`, disable the auto-generated constraint file:
   ```tcl
   set_property IS_ENABLED false [get_files *pcie2_ip-PCIE_X0Y0.xdc]
   ```
2. Use `constraints/ax7203_pcie.xdc` which has the correct lane order

### JTAG vs Flash Boot

- For **JTAG programming**: Keep JTAG cable connected
- For **Flash boot** (cold boot from SPI flash): **Disconnect JTAG cable**
  - JTAG cable prevents FPGA from booting from flash
  - Program flash → Power off → Disconnect JTAG → Power on

## Current Status

PCIe Gen2 streaming verified and working.

```
$ lspci -d 10ee:
11:00.0 Memory controller: Xilinx Corporation Device 7024

$ cat /sys/bus/pci/devices/0000:11:00.0/current_link_*
5 GT/s PCIe     (Gen2)
1               (x1 lane via TB4)
```

## Purpose

This project serves as a reference implementation for:
1. Validating XDMA driver installation
2. Testing PCIe link training (Gen1/Gen2)
3. Verifying C2H streaming before integrating with trading logic
4. Baseline for CDC FIFO debugging (no CDC in this design)

For the full trading pipeline with Ethernet/ITCH/Order Book, see Project 23.

## Dependencies

- Vivado 2019.1+
- Xilinx dma_ip_drivers (XDMA Linux driver)
- GCC for host tools

---

**Status:** Completed and tested on hardware

**Created:** December 2024

**Last Updated:** December 2024

**Author:** Adilson Dias

**Project:** https://github.com/adilsondias-engineer/fpga-trading-systems

# TinyUSB USB Host Mass Storage File Explorer Example for Alif Semiconductor Ensemble E7

A USB **host** example for the [Alif Semiconductor](https://www.alifsemi.com)
Ensemble **E7 DevKit** (`AE722F80F55D5LS`), built on
[TinyUSB](https://github.com/hathach/tinyusb).

The Ensemble xHCI controller acts as the USB host: attach a USB flash drive and
the example mounts its FAT filesystem and gives you an interactive shell on the
serial console, with `ls`, `cd`, `pwd`, `cat`, `cp`, `mkdir` and `rm`.

| | |
|---|---|
| **Target device** | Alif Ensemble `AE722F80F55D5LS`, Cortex-M55 HP and HE cores |
| **Target board** | Alif Ensemble E7 DevKit Gen 2 (`DevKit-E7`) |
| **USB role** | Host, via the Ensemble xHCI controller (`hcd_ensemble.c`) |
| **USB classes** | Mass storage (MSC), Hub |
| **Filesystem** | FatFs R0.15 — FAT12/16/32 with long file names (exFAT disabled) |
| **Console** | embedded-cli shell over the PRG USB serial port |
| **Published by** | Sirin Software, with Alif Semiconductor's permission |
| **Status** | Development snapshot, provided as-is — see below |

---

## Publication and collaboration statement

This example was developed as part of the collaboration between **Sirin
Software** and **Alif Semiconductor**, and is published by Sirin Software with
Alif Semiconductor's permission.

This repository is **not** an official Alif Semiconductor examples repository.

## Snapshot / As-Is

> This repository provides **development snapshots** for the Alif Semiconductor
> Ensemble platform. They are provided **as-is** for demonstration and development
> purposes and should be considered snapshots rather than finalized demos.
>
> Customer support is not provided for these snapshots. Future finalized versions
> may be supported separately.

---

## Target Setup Requirements

### Hardware

- **Alif Ensemble E7 DevKit Gen2**
- **Micro-USB cable** (included with the DevKit)
- A **host computer** (Linux, macOS or Windows) with terminal emulation software
- One or more **USB devices to attach**: a USB-to-serial adapter, a USB flash
  drive, a USB keyboard or mouse, optionally through a USB hub
- Optional: **JTAG debugger**, such as SEGGER J-Link or Arm ULINKpro

### USB Connections

The DevKit provides two USB interfaces used by the example:

#### PRG USB (Micro-USB)

Connect a USB cable from the host computer to the **PRG USB** connector.

This interface provides a two-channel USB-to-UART adapter:
- used by the SETOOLS applications to program images into MRAM.
- used for the example's console output and keyboard input. The example
  configures it as 115200 baud, 8-N-1.

#### HOST USB (USB-A receptacle)

This interface is connected directly to the Ensemble SoC. In this example it
operates in **host** role, so it is where the USB device under test is
attached rather than being connected to the host computer.

The USB host interface is powered only from the DC Jack. Connect a 5 V DC power
supply to the J9 connector on the DevKit. The interface provides the required
VBUS power to the attached USB device.

### GPIO used by the example

| Function | Board definition | GPIO |
|---|---|---|
| Status LED | `BOARD_LEDRGB1_R_*` | GPIO port 6, pin 2 (RGB LED 1, red) |

---

## Getting started

Clone with submodules

```bash
git clone --recurse-submodules <repository-url>
```

The example is built with the CMSIS solution flow described below. There is no
CMake, FreeRTOS or Zephyr build in this repository.

---

## CMSIS solution build (VS Code / CMSIS-Toolbox)

### Prerequisites

- VS Code with the recommended extensions:
  `arm.environment-manager`, `arm.cmsis-csolution`,
  `ms-vscode.cpptools-extension-pack`, `marus25.cortex-debug`.
- The tools installed by the Arm Environment Manager: CMake ≥ 3.28.4,
  Ninja ≥ 1.12.0,`arm-none-eabi-gcc` ≥ 13.2.1-0, CMSIS-Toolbox ≥ 2.6.1.
- CMSIS packs (installed by the task below): `ARM::CMSIS@6.2.0`,
  `ARM::CMSIS-FreeRTOS@11.2.0`, `AlifSemiconductor::Ensemble@2.0.2`.
- For the Security Toolkit programming tasks: the Alif Security Toolkit
  (SETOOLS) installed locally, and a VS Code setting `alif.setools.root`
  pointing at its installation directory. This setting is not stored in the
  repository; add it to your user or workspace settings.

Setting up the Alif VS Code environment itself is documented by Alif in the
[VS Code Getting Started Template](https://github.com/alifsemi/alif_vscode-template);
follow that first.

### Build

1. Open the repository folder in VS Code (**File → Open Folder**).
2. When the **Arm Environment Activation** prompt appears, choose
   **Allow For Current Workspace**.
3. **F1 → Tasks: Run Task → First time pack installation** (only needed once
   per machine; it runs `cpackget` to install the three CMSIS packs listed above).
4. **F1 → CMSIS: Manage Solution Settings** and select a target type:
   `E7-HP` (Cortex-M55 High Performance core) or `E7-HE` (High Efficiency core).
5. Build with the CMSIS extension's build (hammer) button, or
   **F1 → Tasks: Run Task → cmsis-csolution.build: Build**.

Build types are `release` (balanced optimisation, debug info off) and `debug`
(no optimisation, debug info on). GCC is the selected compiler.

### Output location

```
out/<project>/<target>/<build-type>/<project>.elf
out/<project>/<target>/<build-type>/<project>.bin
```

for example
`out/host_msc_file_explorer/E7-HP/release/host_msc_file_explorer.elf`.

### Program the board

Two paths are pre-configured in [`.vscode/tasks.json`](.vscode/tasks.json):

**Alif Security Toolkit (writes to MRAM)**

**F1 → Tasks: Run Task → Program with Security Toolkit**.
Variants for selecting a COM port and for installing debug stubs are provided
as separate tasks. The MRAM load addresses come from the per-core
configuration files in [`.alif/`](.alif): `0x80200000` for M55_HP and
`0x80000000` for M55_HE.

**pyOCD over CMSIS-DAP**

**F1 → Tasks: Run Task → CMSIS Load** (or `CMSIS Load+Run`). Related tasks:
`CMSIS Run`, `CMSIS Erase`, `CMSIS TargetInfo`.

### Debug

[`.vscode/launch.json`](.vscode/launch.json) provides a J-Link configuration
("Alif Ensemble Debug (Cortex-Debug)", using
[`.alif/JLinkDevices.xml`](.alif/JLinkDevices.xml) and RTT console) and pyOCD
gdb launch/attach configurations for the M55_HP and M55_HE cores.

---

## The example in detail

TinyUSB runs in host role on the Ensemble xHCI controller. `main()` initialises
the board, brings up the host stack with `tusb_init()`, initialises the file
explorer, and then runs `tuh_task()`, the explorer task and the LED blinker in
a super-loop. There is no RTOS.

Three pieces are stacked on top of each other:

- the TinyUSB **mass storage host driver**, which enumerates the drive and
  issues SCSI read and write commands;
- **FatFs** (Elm Chan's R0.15), which turns those block operations into a FAT
  filesystem, wired up through the disk I/O glue in
  [`msc_app.c`](host_msc_file_explorer/src/msc_app.c);
- **embedded-cli**, which provides the line editing, history and command
  dispatch for the shell on the serial console.

The project modifications place the FatFs state in the memory section the
Ensemble USB controller can reach: the `FATFS` work areas and the `FIL` file
objects used by the `cat` and `cp` commands are declared `static` and attributed
with `CFG_TUH_MEM_SECTION`, instead of living on the stack as they do upstream.

This example is a useful check of sustained bulk transfers through the Alif host
controller, since reading and copying files drives long multi-block SCSI
transfers rather than the short control traffic of enumeration alone.

**Available commands**

| Command | Purpose |
|---|---|
| `ls [path]` | list a directory |
| `cd <path>` | change the current directory |
| `pwd` | print the current directory |
| `cat <file>...` | print file contents to the console |
| `cp <src> <dst>` | copy a file |
| `mkdir <path>` | create a directory |
| `rm <path>` | remove a file or directory |

embedded-cli also provides a built-in `help` command that lists the above.

**Run.**

1. Program the board and reset it.
2. Connect **PRG USB** to the host computer and open the serial port at
   115200 8-N-1.
3. Attach a FAT-formatted USB flash drive to **HOST USB**.
4. Use the shell prompt on the console.

**Expected result.**

- The console prints `TinyUSB Host MassStorage Explorer Example` on start-up.
- The status LED blinks, indicating the loop is running.
- Attaching the drive prints the mass storage mount messages, and the drive is
  mounted as a FatFs volume.
- The shell prompt accepts the commands above; `ls` lists the drive's root
  directory, `cat` prints a text file, `cp` copies one.
- Removing the drive unmounts the volume.
- TinyUSB's own logging is off by default (`CFG_TUSB_DEBUG 0` in
  [`tusb_config.h`](host_msc_file_explorer/src/tusb_config.h)); raise it to see
  stack level traces on the same serial port.

---

## Licensing

This repository contains software from multiple sources and under multiple
applicable licences. **No single licence covers the whole repository.**

- **TinyUSB source and TinyUSB-derived source** (the example sources under
  `host_msc_file_explorer/src/`, and the `lib/tinyusb` submodule except where
  noted): MIT Licence — see [`LICENSE-MIT.txt`](LICENSE-MIT.txt) and the notices
  in the individual files. The copyright notices name Ha Thach (tinyusb.org).
- **Alif Semiconductor material** (the Ensemble xHCI host controller sources
  under `lib/tinyusb/src/portable/xhci/`, the Alif BSP inside `lib/tinyusb`,
  the `lib/alif_common-app-utils` submodule, the CMSIS pack configuration under
  `host_msc_file_explorer/RTE/`, and the project scaffolding derived from the
  Alif VS Code template): the Alif Semiconductor Software License Agreement —
  see [`LICENSE-ALIF.txt`](LICENSE-ALIF.txt) and the notices in the individual
  files.
- **Azure RTOS USBX material.** Four files compiled into this example
  (`ux_system_initialize.c` and the three `ux_utility_memory_*` files in
  `lib/tinyusb/src/portable/xhci/`) carry a Microsoft Corporation copyright
  notice and refer to the Microsoft Software License Terms for Microsoft Azure
  RTOS. That licence text is not included in this repository. See
  [`THIRD_PARTY_NOTICES.txt`](THIRD_PARTY_NOTICES.txt) before redistributing or
  building on this code.
- **FatFs** (`lib/tinyusb/lib/fatfs`): the FatFs licence, a one-clause
  BSD-style permissive licence. Copyright (C) 2022, ChaN. See
  `lib/tinyusb/lib/fatfs/LICENSE.txt`.
- **embedded-cli** (`lib/tinyusb/lib/embedded-cli`): MIT Licence. Copyright (c)
  2021 Sviatoslav Kokurin.
- **Other third-party components** (Arm CMSIS under Apache-2.0): see
  [`THIRD_PARTY_NOTICES.txt`](THIRD_PARTY_NOTICES.txt) and the components' own
  licence files.

Modifications developed for the Alif Semiconductor platform are marked as such
in the affected files; they do not replace the upstream copyright or licence
notices in those files, which continue to apply to the original material.

---

## Support and issues

This repository is a **published snapshot**. It is maintained on a best-effort
basis by Sirin Software and is not covered by an Alif Semiconductor
customer-support commitment.

- Questions or problems with the content of **this repository** can be raised
  through this repository's issue tracker on GitHub. Opening an issue here does
  not create any support obligation for Alif Semiconductor.
- Questions about **Alif Semiconductor devices, SDKs, packs or tools** belong in
  Alif's own support channels; see [alifsemi.com](https://www.alifsemi.com).
- Questions about **TinyUSB itself** belong upstream at
  [hathach/tinyusb](https://github.com/hathach/tinyusb).

---

## References

- [Alif Ensemble E7 DevKit documentation](https://alifsemi.com/support/kits/ensemble-e7devkit/)
- [Alif VS Code Getting Started Template](https://github.com/alifsemi/alif_vscode-template)
- [TinyUSB documentation](https://docs.tinyusb.org)
- [FatFs documentation](http://elm-chan.org/fsw/ff/)
- [embedded-cli](https://github.com/funbiscuit/embedded-cli)
- [CMSIS-Toolbox](https://github.com/Open-CMSIS-Pack/cmsis-toolbox)

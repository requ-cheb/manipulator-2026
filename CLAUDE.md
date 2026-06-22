# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project overview

This is an Infineon ModusToolbox firmware project for PSOC Edge E84 MCU. It demonstrates a three-project multi-core structure targeting `KIT_PSE84_AI` (default) with GCC_ARM toolchain.

## Build commands

Run from the top-level directory. Requires ModusToolbox v3.6+ installed at `~/ModusToolbox/`.

```bash
make build                          # build all three sub-projects
make build -j12                     # parallel build (12 jobs)
make program                        # build and flash via KitProg3/OpenOCD
make clean                          # clean all build artifacts
make erase                          # erase device flash
make erase MTB_ERASE_EXT_MEM=1     # erase device + external QSPI flash
make getlibs                        # fetch/update library dependencies
make modlibs                        # launch Library Manager GUI
```

Override build variables:
```bash
make build TOOLCHAIN=ARM            # ARM, IAR, LLVM_ARM, GCC_ARM (default)
make build TARGET=KIT_PSE84_EVAL_EPC2
make build CONFIG=Release           # Debug (default) or Release
make build VERBOSE=1                # show full compiler command lines
```

To build a single sub-project:
```bash
cd proj_cm33_s && make build
cd proj_cm33_ns && make build
cd proj_cm55 && make build
```

## Architecture

### Multi-core structure

All PSOC Edge E84 applications use a mandatory three-project layout:

| Project | Core | Security | Description |
|---------|------|----------|-------------|
| `proj_cm33_s` | CM33 | Secure (SPE) | Configures TrustZone security boundaries via `cybsp_init`, then transfers control to NS |
| `proj_cm33_ns` | CM33 | Non-Secure (NSPE) | Initializes peripherals, RTC, LPTimer (MCWDT0), enables CM55 via `Cy_SysEnableCM55()`, runs FreeRTOS |
| `proj_cm55` | CM55 | Non-Secure | Initializes peripherals, LPTimer (MCWDT1), runs FreeRTOS |

### Boot sequence

ROM boot → Secure Enclave (RoT) → `proj_cm33_s` → `proj_cm33_ns` → `proj_cm55` (launched by NS via `Cy_SysEnableCM55`)

All three images are stored on external QSPI flash and executed in XIP (Execute-in-Place) mode.

### Post-build image assembly

`configs/boot_with_extended_boot.json` drives three post-build steps:
1. **Sign** `proj_cm33_s.hex` → adds MCUboot metadata header (0x400 bytes)
2. **Relocate** `proj_cm33_ns.hex` → shifts to S-BUS programmable address
3. **Merge** all three signed/relocated hex files → `build/app_combined.hex`

This combined hex is what gets programmed to the device.

### Makefile hierarchy

- `Makefile` — top-level `APPLICATION` type, lists `MTB_PROJECTS`
- `common_app.mk` — locates `CY_TOOLS_DIR` (ModusToolbox installation)
- `common.mk` — shared per-project settings: `TARGET`, `TOOLCHAIN`, `CONFIG`
- `proj_*/Makefile` — project-specific settings: `CORE`, `COMPONENTS`, `DEFINES`

Change `TARGET` or `TOOLCHAIN` in `common.mk` to apply across all sub-projects.

### Key per-project settings

- `proj_cm33_s`: `VCORE_ATTRS=SECURE`, `COMPONENTS+=SECURE_DEVICE`
- `proj_cm33_ns`: `COMPONENTS+=FREERTOS RTOS_AWARE`, `CORE_NAME=CM33_0`
- `proj_cm55`: `COMPONENTS+=FREERTOS RTOS_AWARE`, `CORE=CM55`, `CORE_NAME=CM55_0`

Both non-secure projects use FreeRTOS tickless idle via MCWDT-backed LPTimer. FreeRTOS config is in `proj_cm33_ns/FreeRTOSConfig.h` and `proj_cm55/FreeRTOSConfig.h` (50 KB heap, heap_3 allocator, deep sleep enabled when idle).

### BSP

`bsps/TARGET_APP_KIT_PSE84_AI/` contains the board support package: pin definitions, system init (`cybsp.c`/`system_edge.c`), partition table (`partition_edge.h`), and peripheral configs. Peripheral resource assignments (LPTimer IRQs, GPIO pins, etc.) are referenced via `CYBSP_*` macros defined here.

### Debugging

`openocd.tcl` configures KitProg3/SWD for dual-core debugging (CM33 + CM55). Both cores use RTOS-aware mode (`-rtos auto`). VS Code launch configs are in `.vscode/launch.json` and `proj_cm55/.vscode/launch.json`. Open the project via `PSOC_Edge_Empty_Application.code-workspace`.

Serial output: 115200 baud, 8N1 on KitProg3 COM port.

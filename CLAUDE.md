# LinuxCNC Project — Claude Instructions

## Project Overview

This repository is a fork of the official LinuxCNC repository. The focus is on integrating a custom STM32H723-based control board using the Remora firmware over SPI.

## Computers

- **Development machine:** Debian 12 — code is written here, LinuxCNC is **not** installed
- **Test machine:** Raspberry Pi 5, Debian 13 — LinuxCNC 2.9.8 is installed and runs here

## Hardware Setup

- **CNC Control Board:** Custom design, MCU: STM32H723
- **Firmware:** [Remora](https://github.com/scottalford75/Remora) — communicates with LinuxCNC via SPI
- **Communication:** Remora SPI (between RPi5 and control board)

## UI

- **Screen:** qtdragon vertical (qtdragon_hd_vert)
- **Repository path:** `share/qtvcp/screens/qtdragon_hd_vert/`
- **Installed path (test machine):** `/usr/share/qtvcp/screens/qtdragon_hd_vert/`

## Directory Structure

```
configTest/
  oldConfigTest/      # Legacy reference configs — DO NOT modify
  verticalConfigTest/ # Active development configs — primary working directory
```

### configTest/oldConfigTest

Contains old LinuxCNC configuration files used for initial control board testing. **Read-only reference.** Never make code changes here. Only consult when comparing against current behavior or recovering a previous setting.

### configTest/verticalConfigTest

Active LinuxCNC configuration for the current machine setup. All new development, tuning, and feature work happens here.

## Development Guidelines

- When exploring configs, always start from `configTest/verticalConfigTest`.
- If a behavior is unclear, check `configTest/oldConfigTest` for historical context — but do not edit it.
- HAL and INI changes should be validated against the Remora SPI component behavior.
- QTVcp UI changes go into the qtdragon_hd_vert screen directory.
- Do not touch upstream LinuxCNC source files unless the task explicitly requires it.

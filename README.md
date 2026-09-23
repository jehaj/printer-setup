# Ender 5 Pro Reference and Configuration Guide

Documentation and configuration reference for an upgraded Creality Ender 5 Pro running Marlin 2.1.x.

---

## 1. Hardware Specifications

| Component | Identifier / Value | Notes |
| --- | --- | --- |
| Motherboard | Creality v4.2.2 | 32-bit silent board |
| Microcontroller (MCU) | STM32F103RET6 | 512 KB flash memory |
| Stepper Drivers | TMC2208 Standalone | Marked with letter `A` on the SD slot |
| Bed Probe | Antclabs BLTouch | Connected to dedicated 5-pin port |
| Physical Probe Offset | X: -45 mm, Y: -6 mm, Z: ~ -2.0 mm | Manually measured physical mounting distance |
| Kinematics | Cartesian (bed drops along Z) | Build volume: 220 x 220 x 300 mm |

---

## 2. Filament Drying

Filament absorbs ambient moisture, which causes steam bubbles in the melt zone, stringing, and sagging overhangs.

### Target Parameters

* PLA: 45 to 50 C for 4 to 6 hours (temperatures above 55 C will soften and fuse the spool).
* PETG: 60 to 65 C for 6 hours.

### Heated Bed Drying Method

1. Place the spool flat on the heated bed.
2. Cover the spool with a cardboard box that has 3 to 4 ventilation holes cut into the top.
3. Set bed temperature to 50 C and leave for 4 to 6 hours.

### Brittle PLA Check

Bend the leading 50 mm of the filament. Wet PLA snaps cleanly like dry pasta. Dried PLA flexes before snapping.

---

## 3. Slicer Setup

Modern slicers such as [OrcaSlicer](https://github.com/SoftFever/OrcaSlicer?utm_source=gemini) or [PrusaSlicer](https://www.prusa3d.com/page/prusaslicer_424/?utm_source=gemini) handle overhang slowdowns, cooling ramps, and acceleration much better than legacy profiles.

### Start G-code

This sequence heats the bed first to allow thermal expansion, keeps the nozzle below melting point to prevent oozing during probing, runs bed leveling, and finishes hotend heating before the prime lines.

```gcode
G90 ; Use absolute coordinates
M83 ; Extruder relative mode

; Heat bed and soft-warm nozzle
M140 S{first_layer_bed_temperature[0]} ; Set target bed temperature
M104 S150 ; Set idle nozzle temperature to prevent oozing
M190 S{first_layer_bed_temperature[0]} ; Wait for bed temperature to stabilize

; Homing and leveling
G28 ; Home all axes
G29 ; Auto bed leveling (creates new mesh)

; Position and final heat
G1 Z50 F240 ; Drop bed for clearance
G1 X2 Y10 F3000 ; Move to start position
M104 S{first_layer_temperature[0]} ; Set target nozzle temperature
M109 S{first_layer_temperature[0]} ; Wait for nozzle temperature to stabilize

; Prime lines
G1 Z0.28 F240
G92 E0
G1 Y140 E10 F1500 ; Extrude first prime line
G1 X2.3 F5000
G92 E0
G1 Y10 E10 F1200 ; Extrude second prime line
G92 E0

```

### End G-code

```gcode
G91 ; Relative positioning
G1 E-2 F2700 ; Retract filament slightly
G1 Z10 F240 ; Drop bed down
G90 ; Absolute positioning
G1 X0 Y220 F3000 ; Present print
M106 S0 ; Turn off part fan
M104 S0 ; Turn off hotend heater
M140 S0 ; Turn off bed heater
M84 X Y E ; Disable motors (Z kept engaged to prevent bed drop)

```

---

## 4. Firmware Build and Configuration

### Prerequisites

* [Visual Studio Code](https://code.visualstudio.com/?utm_source=gemini)
* [PlatformIO IDE Extension](https://platformio.org/install/ide?install=vscode&utm_source=gemini)
* Source Code: [Marlin 2.1.x](https://github.com/MarlinFirmware/Marlin/releases?utm_source=gemini)
* Config Templates: [Marlin Configurations](https://github.com/MarlinFirmware/Configurations/releases?utm_source=gemini) (`config/examples/Creality/Ender-5 Pro/CrealityV422/`)

### File Modifications

#### `platformio.ini`

Set the build environment for the STM32F103RET6 chip:

```ini
default_envs = STM32F103RE_creality

```

#### `Marlin/Configuration.h`

1. **Board and Steppers**
The `A` marking on the SD card reader indicates TMC2208 drivers. In Creality 4.2.2 boards, these operate without UART serial communication. Setting them to standalone avoids compilation warnings.
```cpp
#define MOTHERBOARD BOARD_CREALITY_V422
#define X_DRIVER_TYPE  TMC2208_STANDALONE
#define Y_DRIVER_TYPE  TMC2208_STANDALONE
#define Z_DRIVER_TYPE  TMC2208_STANDALONE
#define E0_DRIVER_TYPE TMC2208_STANDALONE

```


2. **Probe and Homing**
[Marlin BLTouch Documentation](https://www.google.com/search?q=https://marlinfw.org/docs/configuration/configuration.html%2523bltouch&utm_source=gemini)
Uncomment `ENDER5_USE_BLTOUCH` if present in your example file, or define the following directly:
```cpp
#define BLTOUCH
#define USE_PROBE_FOR_Z_HOMING
#define Z_SAFE_HOMING

```


3. **Nozzle-to-Probe Offsets**
Physical distance measured from the nozzle tip to the BLTouch probe tip:
```cpp
#define NOZZLE_TO_PROBE_OFFSET { -45, -6, 0 }

```


* X = -45 mm (probe is 45 mm left of nozzle)
* Y = -6 mm (probe is 6 mm in front of nozzle)
* Z = 0 mm (Z-offset must be calibrated and saved in EEPROM, not hardcoded here)


4. **Bed Leveling and Insets**
[Marlin Bed Leveling Documentation](https://marlinfw.org/docs/features/auto_bed_leveling.html?utm_source=gemini)
```cpp
#define AUTO_BED_LEVELING_BILINEAR
#define RESTORE_LEVELING_AFTER_G28

```


5. **Hotend Temperature Control (MPC)**
[Marlin MPC Documentation](https://marlinfw.org/docs/features/model_predictive_control.html?utm_source=gemini)
Model Predictive Control models heat transfer directly and avoids sudden temperature drops when the cooling fan ramps up.
```cpp
#define MPCTEMP
// Comment out PIDTEMP when MPCTEMP is used

```


6. **Bed Tramming via LCD**
[Marlin Tramming Documentation](https://marlinfw.org/docs/gcode/G035.html?utm_source=gemini)
Because the probe is offset by X-45, an inset of 45 mm prevents the probe from trying to measure past the physical reach of the carriage.
```cpp
#define LCD_BED_TRAMMING
#define BED_TRAMMING_INSET_LFRB { 45, 45, 45, 45 }
#define BED_TRAMMING_USE_PROBE
#define BED_TRAMMING_INCLUDE_CENTER
#define BED_TRAMMING_AUDIO_FEEDBACK

```


7. **Acceleration Strategy**
```cpp
#define S_CURVE_ACCELERATION

```


*Note: If you plan to test Input Shaping (`#define INPUT_SHAPING_X / Y`) later, `S_CURVE_ACCELERATION` must be commented out because the two features are mutually exclusive.*

#### `Marlin/Configuration_adv.h`

Enable the interactive offset calibration tool:

```cpp
#define PROBE_OFFSET_WIZARD

```

---

## 5. Flashing and Calibration Routine

### Flashing Procedure

1. Build the firmware in PlatformIO (`PlatformIO: Build`).
2. Locate the output file at `.pio/build/STM32F103RE_creality/firmware.bin`.
3. Format an SD card (FAT32, 4096-byte cluster size).
4. Rename the binary to a name not previously used by the bootloader (example: `firmware_2026A.bin`).
5. Insert the SD card into the powered-down printer, then turn on the power. Wait 15 seconds for the screen to initialize.

### Post-Flash Calibration Order

Run through these steps in sequence after flashing a new build:

1. **Clear EEPROM:**
Navigate on the screen to `Configuration -> Advanced Settings -> Initialize EEPROM` (or run `M502` followed by `M500` via terminal). This clears legacy values that cause extreme offsets.
2. **Tune Hotend MPC:**
Run the autotune routine from the terminal:
```gcode
M306 E0 T ; Autotune MPC on hotend
M500      ; Save constants to EEPROM

```


3. **Bed Tramming:**
Navigate to `Motion -> Bed Tramming`. Adjust the corner hand screws until the audible confirmation tone indicates all corners are level relative to each other.
4. **Calibrate Z-Probe Offset:**
* Place a sheet of standard paper on the center of the bed.
* Navigate to `Configuration -> Advanced Settings -> Probe Offsets -> Z-Probe Wizard`.
* Lower the nozzle until it creates light sliding resistance on the paper.
* Finish the wizard and accept the value. Expected target is typically between -1.50 mm and -3.00 mm.


5. **Commit Settings to Memory:**
Navigate to `Configuration -> Store Settings` (or run `M500`). Listen for the confirmation beep. Reboot the printer and verify that the stored offset remains unchanged.

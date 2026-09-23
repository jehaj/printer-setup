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
G1 Z20 F240 ; Drop bed 20mm for clearance
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

### Pre-Configured Defaults (No Edits Needed)

The `config/examples/Creality/Ender-5 Pro/CrealityV422/` example configuration already defines the following:
* **Motherboard:** `#define MOTHERBOARD BOARD_CREALITY_V4` (matches v4.2.2 silent board).
* **Stepper Drivers:** `TMC2208_STANDALONE` for X, Y, Z, and E0.
* **Default Steps/mm:** `{ 80, 80, 800, 93 }` (configured for T8x2 lead screw).
* **Bed Leveling Restore:** `RESTORE_LEVELING_AFTER_G28` is enabled by default.
* **Mesh Tilt Extrapolation:** `EXTRAPOLATE_BEYOND_GRID` is enabled by default under bilinear leveling.
* **Configuration_adv.h Settings:** `BABYSTEPPING` (with double-click LCD shortcut), `PROBE_OFFSET_WIZARD`, and `BABYSTEP_ZPROBE_OFFSET` are automatically activated once the probe is enabled. **No manual edits are required in `Configuration_adv.h`.**

> [!NOTE]
> **Z-Axis Steps Verification (800 vs. 400):** Newer Ender 5 Pro v4.2.2 models use a T8x2 lead screw (800 steps/mm), which is already set in the example config. Older models use T8x4 (400 steps/mm). Verify physical bed travel after flashing (see Section 5, Step 2).
>
> [!WARNING]
> **Do NOT enable Linear Advance (`LIN_ADVANCE`):** The TMC2208 drivers on the v4.2.2 board operate in standalone StealthChop mode without UART communication. Rapid step-direction pulses from Linear Advance frequently cause the extruder driver to freeze mid-print. See the [Marlin Linear Advance Documentation](https://marlinfw.org/docs/features/lin_advance.html).

### Required File Modifications

#### 1. `platformio.ini` (Root Marlin Directory)

Set the default build environment for the STM32F103RET6 chip:

```ini
default_envs = STM32F103RE_creality
```

#### 2. `Marlin/Configuration.h`

Make only the following edits to `Configuration.h`:

1. **Activate BLTouch and Leveling (Line ~26)**
   Uncomment the convenience macro. This automatically activates `BLTOUCH`, `AUTO_BED_LEVELING_BILINEAR`, `Z_SAFE_HOMING`, `G26_MESH_VALIDATION`, `LCD_BED_LEVELING`, and `PROBE_OFFSET_WIZARD`:
   ```cpp
   #define ENDER5_USE_BLTOUCH
   ```

2. **Dedicated 5-Pin Probe Port Wiring Check (Line ~1319)**
   [Marlin BLTouch Documentation](https://marlinfw.org/docs/configuration/configuration.html#bltouch)  
   When using the dedicated 5-pin BLTouch port on the v4.2.2 board, Marlin must read the probe signal from pin `PB1` instead of `PA7`. **Comment out** this line:
   ```cpp
   //#define Z_MIN_PROBE_USES_Z_MIN_ENDSTOP_PIN
   ```
   *(If using an adapter that plugs the 2-pin black/white wire into the mechanical Z-stop socket, leave this line enabled).*

   > [!IMPORTANT]
   > **High-Air Probe Safety Test:**  
   > On the Ender 5 Pro, the bed rises **upward** toward the stationary nozzle gantry during Z-homing. Before letting the nozzle get anywhere near the bed, test the probe high in the air:
   > * Manually lower the bed (or jog Z down) so there is at least 100 mm of clearance below the nozzle.
   > * Initiate Z homing (`G28 Z`).
   > * While the bed is still rising and 50–100 mm away from the nozzle, **trigger the Z-probe pin with your finger**.
   > * If the Z-axis does **not** stop immediately, cut power to the printer immediately! This means your probe wiring (`PB1` vs. `PA7`) or logic inversion (`Z_MIN_PROBE_ENDSTOP_INVERTING`) is wrong. You can also verify probe states while stationary by sending `M119` (Endstop Status) via terminal.

3. **Physical Nozzle-to-Probe Offsets (Line ~1530)**
   Update the default offsets (`{-40, -13, -1.45}`) with your physical mounting distance:
   ```cpp
   #define NOZZLE_TO_PROBE_OFFSET { -45, -6, 0 }
   ```
   * X = -45 mm (probe is 45 mm left of nozzle)
   * Y = -6 mm (probe is 6 mm in front of nozzle)
   * Z = 0 mm (calibrated post-flash via Z-Probe Wizard and stored in EEPROM)

4. **Mesh Grid Density (Line ~2014)**
   [Marlin Bed Leveling Documentation](https://marlinfw.org/docs/features/auto_bed_leveling.html)  
   Increase probing resolution from 3x3 (9 points) to 5x5 (25 points):
   ```cpp
   #define GRID_MAX_POINTS_X 5
   ```

5. **Hotend Temperature Control — Model Predictive Control (Lines ~524 & ~677)**
   [Marlin MPC Documentation](https://marlinfw.org/docs/features/model_predictive_control.html)  
   Model Predictive Control models heat transfer directly, avoiding sudden temperature dips when part-cooling fans ramp up.  
   * Comment out `PIDTEMP` (Line ~524):
     ```cpp
     //#define PIDTEMP
     ```
   * Uncomment `MPCTEMP` (Line ~677):
     ```cpp
     #define MPCTEMP
     ```
   *(Note: `MPC_HEATER_POWER { 40.0f }` and `MPC_INCLUDE_FAN` are already enabled by default inside the `MPCTEMP` block. You can also optionally uncomment `#define MPC_AUTOTUNE_MENU` at Line ~717 to run tuning directly from the LCD).*

6. **Probe-Assisted Bed Tramming via LCD (Lines ~2092–2100)**
   [Marlin Tramming Documentation](https://marlinfw.org/docs/gcode/G035.html)  
   `LCD_BED_TRAMMING` is already enabled by default. Update the corner insets and enable automated probe measurement:
   ```cpp
   #define BED_TRAMMING_INSET_LFRB { 45, 45, 45, 45 }
   #define BED_TRAMMING_INCLUDE_CENTER
   #define BED_TRAMMING_USE_PROBE
   #define BED_TRAMMING_AUDIO_FEEDBACK
   ```

7. **Acceleration Strategy (Line ~1303)**
   Uncomment S-curve acceleration for smoother motion and reduced vibration:
   ```cpp
   #define S_CURVE_ACCELERATION
   ```
   *(Note: If you plan to test Input Shaping later, `S_CURVE_ACCELERATION` must remain commented out as the two features are mutually exclusive).*

---

## 5. Flashing and Calibration Routine

### Calibration Guides and Reference Tools

Marlin includes references to several foundational calibration tools and guides directly in `Configuration.h`:

* **Calculators & Technical Walkthroughs:**
  * [Průša 3D Printer Calculator](https://blog.prusa3d.com/calculator_3416/): Calculate leadscrew pitch, motor steps/mm, and optimal layer heights.
  * [Teaching Tech 3D Printer Calibration](https://teachingtechyt.github.io/calibration.html): Interactive step-by-step calibration suite covering E-steps, first layer, slicer flow, and retraction.
  * [RepRap Calibration Guide](https://reprap.org/wiki/Calibration): General machine and motion calibration theory.
  * [Triffid Hunter's Calibration Guide](https://reprap.org/wiki/Triffid_Hunter%27s_Calibration_Guide): Detailed reference for fine-tuning E-steps, steps/mm, and extrusion widths.
  * [RepRap Log Phase Calibration Guide (Archive)](https://web.archive.org/web/20220907014303/sites.google.com/site/repraplogphase/calibration-of-your-reprap): Classic mechanical and bed alignment reference.
  * [Marlin Calibration Video Guide](https://youtu.be/wAL9d7FgInk): Visual walkthrough of 3D printer calibration routines.

* **Calibration Test Objects:**
  * [Thingiverse: 20 mm Calibration Cube (Thing 5573)](https://www.thingiverse.com/thing:5573): Standard test print for measuring dimensional accuracy and verifying X/Y/Z steps.
  * [Thingiverse: Precision Bed Leveling Test (Thing 1278865)](https://www.thingiverse.com/thing:1278865): Multi-point thin square pattern for validating bed mesh extrapolation and first layer squish.

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

2. **Verify Z-Axis Travel Distance (800 vs. 400 Check):**
Before homing or printing, verify that the Z lead screw steps match your physical hardware (calculate or verify pitches with the [Průša Calculator](https://blog.prusa3d.com/calculator_3416/)):
* Place a small ruler next to the bed or note the starting height.
* Send `G1 Z10 F200` via terminal or jog Z up by 10 mm via the LCD.
* Measure the physical bed movement:
  * If it moved **10 mm**, your steps are correct (800 steps/mm).
  * If it moved only **5 mm**, your firmware is set to 400 but needs 800: send `M92 Z800` then `M500`.
  * If it moved **20 mm**, your firmware is set to 800 but needs 400: send `M92 Z400` then `M500`.

3. **Tune Hotend MPC:**
Ensure the hotend is completely cold (ambient room temperature) before starting:
```gcode
M306 E0 T ; Autotune MPC on hotend
M500      ; Save constants to EEPROM
```

4. **Calibrate Extruder E-Steps (100 mm Test):**
Ensure the extruder feeds precisely the length of filament requested by the slicer (detailed instructions available at [Teaching Tech Extruder Calibration](https://teachingtechyt.github.io/calibration.html#esteps) and [Triffid Hunter's Calibration Guide](https://reprap.org/wiki/Triffid_Hunter%27s_Calibration_Guide#E_steps_fine_tuning)):
* Heat the hotend to your printing temperature (e.g. 200 °C for PLA).
* Measure and mark the filament with a fine marker at **100 mm** and **120 mm** from the extruder intake hole.
* Send the following via terminal to extrude 100 mm slowly:
  ```gcode
  M83          ; Relative extruder mode
  G1 E100 F100 ; Extrude 100 mm at 100 mm/min
  ```
* Measure the remaining distance from the intake hole to the 120 mm mark:
  * If exactly 20 mm remains, exactly 100 mm was extruded (`93.0` steps/mm is accurate).
  * If the actual extruded length differs, calculate the correction:
    $$\text{New E-steps} = \frac{100 \times \text{Current E-steps}}{\text{Actual Length Extruded}}$$
  * Update and save: send `M92 E<new_value>` then `M500`.

5. **High-Air Z-Probe Safety Test (Air-Trigger Check):**
Before allowing the printer to auto-home against the bed or execute probing routines:
* Drop the bed manually or jog Z down to ensure at least 100 mm of clearance below the nozzle.
* Initiate Z homing (`G28 Z` via terminal or LCD).
* As the bed rises toward the nozzle and the BLTouch deploys its pin, **trigger the probe pin with your finger** while the bed is still 50 to 100 mm away from the nozzle.
* **Pass:** The bed stops moving upward immediately and retracts slightly.
* **Fail:** If the bed does not stop immediately, cut power to the printer immediately! This indicates incorrect probe pin routing (`PB1` vs. `PA7` / `Z_MIN_PROBE_USES_Z_MIN_ENDSTOP_PIN`) or inverted probe logic (`Z_MIN_PROBE_ENDSTOP_INVERTING`). You can also verify probe trigger states safely while stationary by sending `M119` (Endstop Status) via terminal.

6. **Bed Tramming:**
Navigate to `Motion -> Bed Tramming`. Adjust the corner hand screws until the audible confirmation tone indicates all corners are level relative to each other.

7. **Calibrate Z-Probe Offset:**
* Place a sheet of standard paper on the center of the bed.
* Navigate to `Configuration -> Advanced Settings -> Probe Offsets -> Z-Probe Wizard`.
* Lower the nozzle until it creates light sliding resistance on the paper.
* Finish the wizard and accept the value. Expected target is typically between -1.50 mm and -3.00 mm.

8. **Commit Settings to Memory:**
Navigate to `Configuration -> Store Settings` (or run `M500`). Listen for the confirmation beep. Reboot the printer and verify that the stored offset remains unchanged.

9. **First Layer Live Babystepping:**
During your first test print (such as the [Bed Leveling Calibration Pattern](https://www.thingiverse.com/thing:1278865) or a skirt, following the [Teaching Tech First Layer Guide](https://teachingtechyt.github.io/calibration.html#firstlayer)), double-click the LCD knob to open the babystep menu. Adjust Z live in 0.01 mm increments until the extrusion lines gently squish together without overlapping or peeling. Marlin will automatically offer to commit the babystepped offset to EEPROM.



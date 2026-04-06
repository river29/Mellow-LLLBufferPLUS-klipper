# Klipper Configuration for Mellow LLL Buffer Plus

Complete Klipper configuration for the Mellow LLL Filament Plus Buffer with automatic filament feeding and buffer management.

> **Note:** This is the Klipper configuration. For the Buffer Plus firmware source code, see the [main repository README](../README.md).

## Features

- ✅ **Automatic Buffer Control** - Fills buffer automatically when filament is detected
- ✅ **Smart Feed Bursts** - HALL2 sensor triggers small feed bursts during printing
- ✅ **Overfill Protection** - HALL1 prevents buffer from jamming into extruder
- ✅ **Manual Feed/Retract** - Physical buttons for manual filament loading
- ✅ **Filament Runout Detection** - Optional pause on filament runout

## Hardware Setup

### Sensor Configuration
- **ENDSTOP3 (PB7)**: Filament entrance sensor - detects when filament is loaded
- **HALL3 (PB4)**: Initial fill sensor - switches from continuous to burst mode
- **HALL2 (PB3)**: Primary buffer control - triggers feed bursts when neck extends
- **HALL1 (PB2)**: Overfill limiter - prevents buffer from over-filling

### Button Configuration
- **Feed Button (PB12)**: Manual continuous feed (hold to feed)
- **Retract Button (PB13)**: Manual continuous retract (hold to retract)

---

## Installation

### Step 1: Flash Katapult Bootloader (Recommended)

Katapult (formerly CanBoot) allows easy firmware updates without needing to press physical buttons or enter DFU mode.

#### 1.1: Build Katapult

```bash
cd ~
git clone https://github.com/Arksine/katapult
cd katapult
make menuconfig
```

**Katapult Configuration:**
- Micro-controller Architecture: `STMicroelectronics STM32`
- Processor model: `STM32F072`
- Build Katapult deployment application: `Do Not build`
- Clock Reference: `8 MHz crystal`
- Communication interface: `USB (on PA11/PA12)`
- Application start offset: `8KiB offset`
- USB ids: Leave default or customize
- Support bootloader entry on rapid double click: `[*]` ✓ (Enable this!)
- Enable bootloader entry on button (or gpio) state (Do not enable this)
- Enable Status LED `[*]`
- (PA8)   Status LED GPIO Pin

```bash
make clean
make
```

#### 1.2: Enter DFU Mode

The LLL Buffer Plus needs to be put into DFU (Device Firmware Update) mode:

**Method 1: Jumper BOOT0 to 3.3V**
1. Push and hold the boot button
2. Push the reset button
3. Release the boot button

**Method 2: BOOT Button (if accessible)**
1. Disconnect USB
2. Hold the **BOOT button** on the board
3. Connect USB while holding BOOT
4. Release BOOT button


#### 1.3: Verify DFU Mode

```bash
lsusb | grep DFU
```

You should see something like:
```
Bus 001 Device 015: ID 0483:df11 STMicroelectronics STM Device in DFU Mode
```

If not detected, try:
```bash
sudo dfu-util -l
```

#### 1.4: Flash Katapult

```bash
cd ~/katapult
sudo dfu-util -a 0 -D ~/katapult/out/katapult.bin --dfuse-address 0x08000000:force:mass-erase:leave -d 0483:df11
```

You should see output ending with:
```
File downloaded successfully
```

#### 1.5: Verify Katapult

Disconnect and reconnect USB. Check for Katapult device:

```bash
ls /dev/serial/by-id/
```

You should see something like:
```
usb-katapult_stm32f072xb_XXXXXX-if00
```

---

### Step 2: Build and Flash Klipper Firmware

#### 2.1: Build Klipper

```bash
cd ~/klipper
make menuconfig
```

**Klipper Configuration:**
- Micro-controller Architecture: `STMicroelectronics STM32`
- Processor model: `STM32F072`
- Bootloader offset: `8KiB bootloader` (for Katapult)
- Clock Reference: `8 MHz crystal`
- Communication interface: `USB (on PA11/PA12)`

**Important:** The bootloader offset MUST match what you set in Katapult (8KiB)!

```bash
make clean
make
```

#### 2.2: Flash Klipper via Katapult

Find your device ID:
```bash
ls /dev/serial/by-id/
```

Flash using Katapult's flashtool:
```bash
python3 ~/katapult/scripts/flashtool.py -f ~/klipper/out/klipper.bin -d /dev/serial/by-id/usb-katapult_stm32f072xb_XXXXXX-if00
```

Or using `make flash`:
```bash
make flash FLASH_DEVICE=/dev/serial/by-id/usb-katapult_stm32f072xb_XXXXXX-if00
```

You should see:
```
Attempting to connect to bootloader
Katapult Connected
Protocol: 1.0.0
Flashing '/home/pi/klipper/out/klipper.bin'...
[##################################################]
Write complete: X pages
Verifying...
Verification Complete
CRC: 0xXXXXXXXX
Flashing successful
```

#### 2.3: Verify Klipper

Disconnect and reconnect USB. Check the device ID changed:

```bash
ls /dev/serial/by-id/
```

You should now see:
```
usb-Klipper_stm32f072xb_XXXXXX-if00
```

---

### Step 3: Configure Klipper

#### 3.1: Copy Configuration File

```bash
cp mellow.cfg ~/printer_data/config/
```

#### 3.2: Update printer.cfg

Add to your main `printer.cfg`:
```cfg
[include mellow.cfg]
```

#### 3.3: Update MCU Serial ID

Edit `mellow.cfg` and update the serial path:

```cfg
[mcu LLL_PLUS]
serial: /dev/serial/by-id/usb-Klipper_stm32f072xb_XXXXXX-if00
restart_method: command
```

Replace `XXXXXX` with your actual device ID from Step 2.3.

#### 3.4: Restart Klipper

```
FIRMWARE_RESTART
```

Check the Klipper web interface - you should see the LLL_PLUS MCU connected!

---

### Alternative: Flash Klipper Without Katapult

If you prefer not to use Katapult, you can flash Klipper directly:

#### Build Klipper (No Bootloader)

```bash
cd ~/klipper
make menuconfig
```

**Settings:**
- Micro-controller Architecture: `STMicroelectronics STM32`
- Processor model: `STM32F072`
- Bootloader offset: `No bootloader`
- Clock Reference: `8 MHz crystal`
- Communication interface: `USB (on PA11/PA12)`

```bash
make clean
make
```

#### Flash via DFU

1. Enter DFU mode (see Step 1.2)
2. Flash:
   ```bash
   make flash FLASH_DEVICE=0483:df11
   ```

> **Note:** Without Katapult, future firmware updates will require entering DFU mode manually each time.

---

## How It Works

### Initial Loading
1. Insert filament into entrance sensor (ENDSTOP3/PB7)
2. Buffer starts **continuous feeding** automatically
3. When neck reaches top (HALL3 triggers), switches to **burst mode**

### During Printing
1. Printer pulls filament → Buffer neck extends
2. When neck reaches mid-point (HALL2 releases) → **15mm feed burst**
3. Neck retracts back into housing
4. Repeat as needed

### Overfill Protection
1. If buffer overfills and neck extends too far (HALL1 releases)
2. **Auto-feed pauses** until neck retracts
3. Prevents jamming against extruder

---

## Configuration Tuning

### Adjust Feed Burst Amount
Change the burst size in `_BUFFER_FEED_BURST` macro:
```cfg
[gcode_macro _BUFFER_FEED_BURST]
gcode:
    {% if printer["gcode_macro _BUFFER_AUTO_CONTROL"].overfill_lock == 0 %}
        ACTIVATE_EXTRUDER EXTRUDER=extruder1
        M83
        G1 E15 F3000  # ← Change E15 to desired burst amount (mm)
        M118 Buffer: Feed burst complete
    {% endif %}
```

### Adjust Feed Speed
Change feed/retract speed (currently 3000 mm/min = 50 mm/s):
```cfg
G1 E10 F3000  # Change F3000 to desired speed (mm/min)
```

Common speeds:
- `F1800` = 30 mm/s (slower, more reliable)
- `F3000` = 50 mm/s (default)
- `F6000` = 100 mm/s (faster, may skip)

### Motor Current
Adjust TMC2208 current if motor is too weak or overheating:
```cfg
[tmc2208 extruder1]
uart_pin: LLL_PLUS:PB1
run_current: 0.35  # Increase up to 0.5 if motor skips, decrease to 0.25 if overheating
stealthchop_threshold: 999999
```

### Rotation Distance Calibration

To calibrate your buffer motor for accurate feeding:

1. **Mark the filament** 120mm from the entrance sensor
2. **Heat your hotend** (if min_extrude_temp is set)
3. **Activate the buffer extruder:**
   ```
   ACTIVATE_EXTRUDER EXTRUDER=extruder1
   ```
4. **Feed 100mm:**
   ```
   M83
   G1 E100 F300
   ```
5. **Measure** the actual distance the mark moved
6. **Calculate new rotation distance:**
   ```
   new_rotation_distance = current_rotation_distance * (100 / actual_distance_moved)
   ```
   
   Example: If mark moved 95mm instead of 100mm:
   ```
   new_rotation_distance = 18.86 * (100 / 95) = 19.85
   ```

7. **Update config:**
   ```cfg
   [extruder1]
   rotation_distance: 19.85  # Your calculated value
   ```

8. **Restart and test again** until accurate

---

## Troubleshooting

### Flashing Issues

**DFU device not detected:**
- Check USB cable (must be data cable, not charge-only)
- Try different USB port
- Check `lsusb` without grep to see all devices
- Verify BOOT0 is properly jumpered to 3.3V
- Try both BOOT button methods

**"Cannot open DFU device":**
```bash
sudo dfu-util -a 0 -D ~/katapult/out/katapult.bin --dfuse-address 0x08000000:force:mass-erase:leave -d 0483:df11
```
Run with `sudo` if permission denied.

**Katapult not appearing after flash:**
- Disconnect and reconnect USB
- Wait 5-10 seconds
- Check `dmesg | tail` for USB events
- Reflash Katapult - it may not have written correctly

**Klipper flash fails via Katapult:**
- Verify bootloader offset matches (8KiB in both Katapult and Klipper)
- Try entering Katapult manually: Double-tap reset button quickly
- Reflash Katapult and try again

### Buffer Operation Issues

**Buffer feeds continuously and won't stop:**
- Check HALL3 sensor is working: `QUERY_ENDSTOPS`
- Verify neck can physically reach HALL3 when extended
- Check sensor wiring and polarity
- Look for "HALL3 TRIGGERED" message in console

**HALL2 bursts happen too frequently:**
- Increase burst amount (E15 → E20 or E25)
- Check reverse bowden tube tension
- Verify printer is actually consuming filament

**Buffer overfills (HALL1 warning):**
- Decrease burst amount (E15 → E10)
- Check that printer is pulling filament from buffer
- Verify no clogs in bowden tube
- Check extruder is actually feeding

**Manual buttons don't work:**
- Verify button wiring to PB12 (feed) and PB13 (retract)
- Check console for "button pressed/released" messages
- Ensure buttons are wired normally-open (NO)
- Test with `QUERY_ENDSTOPS` while pressing

**MCU not detected after flashing Klipper:**
- Verify Klipper firmware is flashed (not Arduino or Katapult)
- Check USB connection
- Run `ls /dev/serial/by-id/` to find device
- Check `dmesg | tail` for USB enumeration errors
- Reflash Klipper firmware

**"Option 'step_pin' is not valid in section 'extruder X'":**
- Ensure section is named `[extruder1]` not `[extruder filament_buffer]`
- Klipper only supports numbered extruders: `extruder`, `extruder1`, `extruder2`, etc.

**TMC UART errors:**
- Verify UART pin is correct: `uart_pin: LLL_PLUS:PB1`
- Check TMC2208 is properly seated
- Verify run_current is not too low (minimum ~0.2)

---

## Updating Firmware (with Katapult)

Once Katapult is installed, updating Klipper is easy:

1. **Rebuild Klipper:**
   ```bash
   cd ~/klipper
   make clean
   make
   ```

2. **Flash via Katapult:**
   ```bash
   python3 ~/katapult/scripts/flashtool.py -f ~/klipper/out/klipper.bin -d /dev/serial/by-id/usb-Klipper_stm32f072xb_XXXXXX-if00
   ```

3. **Or use double-tap reset:**
   - Quickly press reset button twice
   - Device enters Katapult mode for 5 seconds
   - Flash using the Katapult device ID

No need to open the case or press BOOT buttons! 🎉

---

## Credits

Klipper configuration developed by [@ss1gohan13](https://github.com/ss1gohan13) for the Mellow LLL Filament Plus Buffer.

Hardware and original firmware by [Mellow 3D](https://github.com/mellow-3d).

Special thanks to:
- James on the Klipper Discord
- Ian on the Klipper Discord
- [Arksine](https://github.com/Arksine) for Katapult bootloader
- [Klipper](https://github.com/Klipper3d/klipper) team

## License

MIT License - Feel free to use and modify!





# ADD MDM Module Clog Detection
Feature Introduction
The FLY-LLL PLUS Buffer can be used in conjunction with the FLY-MDM Filament Runout/Clog Sensor to achieve real-time monitoring and automatic handling of extruder clog conditions.

Core Features
Clog Detection: The MDM module monitors the buffer's filament status to detect clogs.
Unified Runout/Clog Handling: Filament runout detection is also handled by the MDM module, with signals sent via the buffer.
Important Note: When using the MDM module, all filament runout/clog detection signals are sent to the mainboard via the buffer. The mainboard cannot distinguish whether the signal originates from a runout or a clog.

Firmware Requirements
The buffer firmware version must be V1.1.5 or higher.
Hardware Wiring
1. MDM Module to Buffer Connection
The MDM module communicates directly with the buffer to detect clog status:

MDM Module to Buffer Connection Diagram
2. Buffer to Mainboard Connection
The mainboard sends signals to the buffer.

Buffer Signal Forwarding Wiring Diagram
3. Buffer Filament Runout Detection Wiring
The buffer's filament runout detection must be connected to the mainboard; otherwise, it will not function properly.


Specific Connection Method:

Buffer Pin	Function Description	Connection Suggestion
STEP	Extruder Stepper Signal Monitoring	Connect to a free PWM, RGB, or 12864 interface on the mainboard.
DIR	Extruder Direction Signal Monitoring	Connect to a free limit switch interface on the mainboard.
Tip: The servo port of a BL-Touch can also be used for STEP signal monitoring.

Klipper Configuration
Pre-configuration Preparation
Before adding the MDM module configuration, ensure the following are correctly configured:

Basic extruder configuration
Basic buffer functionality configuration
Note: Filament runout detection now follows the path: MDM module → Buffer → Mainboard.
1. Buffer Monitor Configuration (for Clog Detection)
Add the following configuration to your Klipper configuration file (e.g., printer.cfg) to monitor extruder status:

 [extruder_stepper buffer_monitor]
 extruder: extruder           # Name of the associated main extruder
 step_pin: PE10               # Replace with the actual pin connected to buffer PA5
 dir_pin: PD4                 # Replace with the actual pin connected to buffer PB11
 rotation_distance: 17.472    # Replace with your extruder's actual value
 gear_ratio: 50:10            # Replace with your extruder's actual gear ratio
 microsteps: 16               # Replace with your extruder's actual microstep setting
full_steps_per_rotation: 200 # Standard stepper motor is 200 steps/revolution

Complete Configuration Example
# Main Extruder Configuration
 [extruder]
 step_pin: PB13
 dir_pin: PB12
 enable_pin: !PB14
 microsteps: 16
 rotation_distance: 17.472
 gear_ratio: 50:10
 nozzle_diameter: 0.4
 filament_diameter: 1.75
 heater_pin: PA1
 sensor_type: ATC Semitec 104GT-2
 sensor_pin: PC1
 control: pid
 pid_Kp: 21.527
 pid_Ki: 1.063
 pid_Kd: 108.982
 min_temp: 0
 max_temp: 280

[extruder_stepper buffer_monitor]
extruder: extruder
step_pin: PE10      # Connected to buffer PA5
dir_pin: PD4        # Connected to buffer PB11
rotation_distance: 17.472
gear_ratio: 50:10
microsteps: 16
full_steps_per_rotation: 200

[filament_switch_sensor Material_breakage_detection]
pause_on_runout: true
switch_pin: ^PA0   # Please replace with the pin you actually use
runout_gcode:
    PAUSE
    RESPOND MSG="Filament runout detected, print paused"
insert_gcode:
    RESPOND MSG="Filament inserted, preparing to resume print"
event_delay: 2.0    # Event trigger delay (seconds)
pause_delay: 2.0     # Pause command delay (seconds)
debounce_delay: 2.0  # Debounce delay (seconds)

Buffer Configuration
Important Notes
If the extruder configuration does not have gear_ratio, change both Driver Gear Number and Driven Gear Number to 1.
Stepper Parameter Calculator
200
Steps per motor revolution
16
Driver microstepping
22
Travel distance per revolution (mm)
1
Driver gear teeth count
1
Driven gear teeth count
145,455
Pulses per mm (step/mm)
Parameter Description
Function Description	Configuration Command (Please input in the serial tool)	Default Value	Unit	Remarks
View all current parameters	
info
-	-	Send the command to read all current configurations.
Set motor pulse count	
steps 145.455
916	-	Set the number of pulses required for the motor to move per millimeter.
Set encoder detection distance	
encoder 1.73
1.73	mm	Set the filament movement distance represented by each encoder signal.
Set operation timeout	
timeout 60000
60000	ms	Set the automatic stop time when no trigger is detected to prevent continuous extrusion.
Set error scaling factor	
scale 2.0
2.0	-	Allowable Error = encoder value X scale value.
Example: 1.73 * 2.0 = 3.46 mm
Set speed control command	
speed 260
260	mm	Set the buffer running speed, maximum 600 (rpm). Firmware needs to be updated to V1.1.1.
Operation Notes:

Command Format: In the "Configuration Command" column of the table above, the entire line of command (e.g., steps 916) is the content that needs to be entered in full.
Sending Method: After entering the command in the send area of the serial assistant, click the Send button.
Automatic Save: After the command is sent successfully, the parameters take effect immediately and are saved automatically. No additional save operation is required.
Confirm Configuration: After modifying any parameter, you can send the info command to query all current parameters to verify if the configuration is correct.
Important Notes
After remembering the set parameters, you can configure the buffer via the link below.
Buffer Configuration
Functionality Testing
1. Connection Test
Complete the connection between the MDM module and the buffer.
Complete the signal wire connection between the buffer and the mainboard.
Verify all connections are secure.
2. Full Process Test
Start a test print.
Simulate a clog condition (operate with care).
Observe:
Whether the MDM module detects the issue.
Whether the buffer forwards the signal.
Whether the mainboard receives the signal.

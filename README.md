## Fork fix: fw-v250dwk (Leonardo + AS5600 + single-axis joystick)

This fork adds one extra firmware build, **fw-v250dwk**, for this hardware combination:

- Arduino Leonardo (ATmega32U4)
- AS5600 magnetic encoder as the steering sensor
- **one** analog joystick axis used as both throttle and brake
- H-bridge motor driver in PWM+- mode (tested with an L298N module)

Everything else is the unmodified firmware from [ranenbg/Arduino-FFB-wheel](https://github.com/ranenbg/Arduino-FFB-wheel). This fork is not affiliated with or endorsed by its authors.

### What changed

| File | Change |
|---|---|
| `brWheel_my/brWheel_my.ino` | With `USE_AS5600` and `USE_SPLITAXIS` both enabled, the HID report now sends the split throttle value (`gasAxis`) on the Z axis. Before, the throttle half of the split axis was computed but never sent in the single-AS5600 case. |
| `arduino-1.8.5/libraries/AS5600/AS5600.cpp` | `getCumulativePosition()` now reads the `RAW ANGLE` register instead of `ANGLE`. |
| `brWheel_my/leonardo hex/fw_v250/brWheel_my.ino.leonardo_v250dwk.hex` | The compiled build (no bootloader included). |

**Why RAW ANGLE:** the `ANGLE` register is scaled by the chip's `ZPOS`/`MPOS`/`MANG` settings. If a module has been programmed with a partial range, `ANGLE` stays clamped outside that range while `RAW ANGLE` keeps changing. On the fork author's module (`ZPOS` = 1666, `MPOS` = 672) the steering axis froze for part of each turn around the 4095 to 0 wrap point. Reading `RAW ANGLE` fixed it there. The trade-off is that `RAW ANGLE` does not go through the chip's output filter, so the value can be slightly noisier.

**Version letters** (printed by the `V` serial command): `d` = no optical encoder, `w` = AS5600, `k` = split axis (one analog input becomes brake and throttle).

### Wiring

| Part | Connection |
|---|---|
| AS5600 | VCC to 5V, GND and DIR to GND, SDA to pin 2, SCL to pin 3 |
| Joystick (one axis) | outer legs to 5V and GND, wiper to **A1** |
| Motor driver (PWM+-) | pin 9 and pin 10 are the two PWM outputs, pin 11 is the enable output |

Joystick behavior: the split point is fixed at the middle of the ADC range (512). Pushing toward 5V gives **throttle (Z axis)**, pushing toward GND gives **brake (Y axis)**. Swap the 5V and GND legs to reverse the direction.

L298N example: pin 9 to IN1, pin 10 to IN2, pin 11 to ENA (remove the ENA jumper), common GND, separate motor supply. Keeping the ENA jumper also works but leaves the motor always enabled, so it brakes instead of free-wheeling at zero force.

**Safety:** a force feedback motor can hurt you. First test with low voltage, no load, a firmly mounted motor and a way to cut the power immediately.

### Flashing

1. Download the `.hex` from the [Releases](../../releases) page (SHA-256: `<fill in after the final build>`).
2. Use XLoader (included in the `XLoader` folder): baud 57600, device ATmega32U4.
3. Press reset on the Leonardo, select the new COM port and press Upload within a few seconds.
4. Open a serial terminal on the board's COM port (line ending: carriage return), send `V`. The answer must be `fw-v250dwk`. If it is not, the wrong file was flashed.

### First start and configuration

- Use [Arduino-FFB-gui](https://github.com/ranenbg/Arduino-FFB-gui) for configuration. A virtual joystick such as **vJoy** can be picked by the GUI instead of the Arduino. Disable it while using the GUI.
- On a board that never ran this firmware family, the EEPROM is empty and defaults are loaded (CPR 4096 for AS5600, PWM+- mode, 7.8 kHz).
- The firmware loads defaults only when the stored version number differs from the firmware's. A board that previously ran another `v250` build keeps its old EEPROM values (for example CPR 2400 from an optical encoder build). Check **CPR = 4096** in the GUI after flashing.
- Pedal limits are also stored in EEPROM. `YR` prints them (brake min, brake max, ...). For the split-axis brake use `YA 0` and `YB 1023`, then save to EEPROM.

### Known limitations

- Leonardo only. Not tested on Pro Micro.
- The throttle half of the split axis has no calibration. The brake half uses the manual pedal limits above. The `a` (autocalibration) option is not part of this build.
- No dead zone around the joystick center. If the stick does not rest at exactly 512, a small brake or throttle value may remain at rest.
- Some joysticks reach the end of their electrical range before the end of their mechanical travel, so the axis saturates early. This is a property of the joystick, not of the firmware.
- `RAW ANGLE` is unfiltered (see above).

### Build it yourself

Follow *How to compile the source* above (Arduino IDE 1.8.19, Arduino AVR Boards 1.6.21, modified core), then in `brWheel_my/Config.h`:

- comment out `#define USE_QUADRATURE_ENCODER`
- uncomment `#define USE_AS5600`
- uncomment `#define USE_SPLITAXIS`

Select **Arduino Leonardo**, compile, and use *Sketch, Export Compiled Binary* (take the file without `with_bootloader`). Flash it as described above and check that `V` answers `fw-v250dwk`.

## Credits

- FFB HID and USB core for Arduino by: Peter Barrett
- BRWheel firmware by: Tero Loimuneva, Saku Kekkonen, Etienne Saint-Paul, and Fernando Igor
https://github.com/fernandoigor/BRWheel/tree/alphatest
- Arduino-FFB-wheel firmware and Arduino-FFB-gui by: Milos Rankovic (ranenbg)
https://github.com/ranenbg/Arduino-FFB-wheel
https://github.com/ranenbg/Arduino-FFB-gui
- AS5600 library by: Rob Tillaart (MIT license, see `arduino-1.8.5/libraries/AS5600/LICENSE`)
- Other bundled libraries (HX711_ADC, Adafruit ADS1X15, Adafruit MCP4725, digitalWriteFast, ...) keep their own authors and licenses, see each library folder.
- fw-v250dwk fork changes by: Sor76

The original copyright and permission notice from the firmware authors is kept unchanged in the headers of the source files (for example `brWheel_my/brWheel_my.ino`) and applies to this fork as well.

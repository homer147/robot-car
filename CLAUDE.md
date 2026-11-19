# CLAUDE.md

## What this is

An obstacle-avoiding car, built over Christmas 2026 with my son nearly 4.5 years old. Its purpose is to **learn**: C++ for me, electronics for both of us. The car is how we get there, not the goal.

I'm building Pip, a children's watch on an ESP32-S3, whose firmware is C++ on the Arduino framework with PlatformIO (`~/repos/pip`, firmware in `~/repos/pip/firmware`). I can't read C++ well yet. This project should leave me able to read Pip's firmware and follow how it's built.

## Teaching mode (always on in this repo)

- **I write the code; you guide.** For each step: explain the idea, show a small example that is *not* our code, then let me write it. Review what I wrote and explain the compiler errors rather than fixing them.
- Write code into the repo only when I ask you to ("just show me", "write it"). Even then, keep it short and explain each line.
- One new idea at a time. When I hit something new (a pointer, `const`, a reference, a header file), stop and explain it in a few lines before moving on.
- Keep it to a session's size: something my son can see working (a light, a number, wheels turning) at the end of each one.
- My son does the building, wiring and testing, and makes the design choices (what the car does at a wall, its sounds and lights). Suggest parts he can do and ways to explain what's happening to a child.
- **Relate it to Pip.** When a concept appears in Pip's firmware, point me to where (read `~/repos/pip/firmware`; never change anything in `~/repos/pip`). The main parallels:
  - The car's driving logic is a state machine, like `lib/watch_state` and `lib/pairing_flow`.
  - Pip keeps pure logic in `lib/` (no Arduino calls), behind small interfaces ("ports"), with the hardware code in `src/`. That lets `lib/` be unit-tested on the Mac (`platformio test -e native`). We'll do the same in stage 5.
  - Pip's `main.cpp` and `WatchApp` only wire the pieces together, and its board-specific pins live in one `board_config.h`.
  - Timing uses `millis()`, never `delay()`, so the program keeps reading buttons and sensors while it waits.

## Toolchain

Same as Pip, so what I learn carries over:

- PlatformIO (VS Code extension or CLI), `platform = espressif32 @ 7.1.3`, `framework = arduino`, `board = esp32-s3-devkitc-1`, `monitor_speed = 115200`.
- The board is a Waveshare ESP32-S3-Zero, which has no PlatformIO board of its own. We borrow `esp32-s3-devkitc-1` (as Pip does for its Waveshare kit) and correct the flash size, since the Zero has 4 MB, not the DevKitC's 8 MB:
  - `board_upload.flash_size = 4MB`
  - `board_build.partitions = default.csv` (the standard 4 MB layout)
- One USB-C port, the S3's own USB (no separate UART chip), so `-DARDUINO_USB_CDC_ON_BOOT=1` as Pip uses. If an upload won't start: hold BOOT, tap RESET, release BOOT. The serial monitor may drop when the board resets.
- Later: a `[env:native]` for unit tests, like Pip's.

## Hardware

Ordered from Core Electronics on 2026-11-19:

- **Waveshare ESP32-S3-Zero** (WS-25081, ESP32-S3FH4R2: 4 MB flash, 2 MB PSRAM), with 2.54 mm male headers to solder on the two long sides only. The short end (GPIO 14–16) stays unsoldered, or those pins would share a breadboard row. It has a built-in WS2812 RGB LED (GPIO 21; check the pinout card), a stage-6 extra. Powered from a USB power bank.
- **Half-size breadboard** (400 tie points). The Zero and the TB6612 both straddle the centre gap. One side's rails carry 3.3 V from the Zero ("the brain"), the other side's the AA pack ("the muscles"), with one wire joining the two ground rails.
- **Pololu TB6612FNG carrier** (POLOLU-713; loose headers to solder). A 100 µF electrolytic (stripe = −) and a 0.1 µF ceramic across VMOT and GND. Two spare 0.1 µF can be soldered across the motor terminals if motor noise resets the board or upsets the readings.
- **Two TT gear motors** (Adafruit ADA3777, 200 RPM, 3–6 V, wires attached), **two 63 mm wheels** (ADA3766) and a **Pololu ½" metal ball caster** (POLOLU-953): differential drive, steering like a tank or wheelchair.
- **4×AA holder with switch** (ADA830). Alkalines give about 6 V (fine for every part), NiMH about 4.8 V. Don't mix types or old with new.
- **RCWL-1601** ultrasonic sensor, 3–5 V (runs on 3.3 V, so no voltage divider), on 4 male-to-female jumpers.
- Later: **FS90 servo** (180°, powered from the AA rail) to turn the sensor, and a **PS1240 passive piezo buzzer** (no polarity; the code sets the pitch).
- Wires: a pre-cut jumper kit for the breadboard, male-to-female jumpers for the sensor, male-to-male for the servo.
- Not from Core: alkaline AAs, M3 bolts for the motors (measure first), a USB-C data cable.

Chassis: 3D-printed (Bambu P2S, build123d). Measure the motors, wheels, caster, battery holder and breadboard before finalising; the caster's height sets the motor height. Drive wheels at the front (near the sensor) is the forgiving layout, but it's my son's choice: give the chassis caster holes at both ends so he can try both.

Safety rules to check every wiring change against:
- ESP32 pins are 3.3 V and **not** 5 V tolerant. A plain HC-SR04's echo pin needs a voltage divider (1 kΩ / 2 kΩ).
- Motors and servo take their power from the AA pack, never from the ESP32's 3.3 V pin.
- All grounds connected together (ESP32, TB6612, AA pack).
- The 100 µF capacitor has polarity: long leg to +, stripe to ground.
- Avoid strapping pins (GPIO 0, 3, 45, 46) and the USB pins (19, 20); on the Zero that means skipping GPIO 3 on the edge. Record the pins we choose in `include/pins.h` and in the table below.

| Signal | GPIO |
| --- | --- |
| (fill in as we wire) | |

## Stages

Each teaches one C++ idea:

1. **Blink an LED:** a PlatformIO project, `setup()` / `loop()`, `pinMode`, `digitalWrite`, then blinking with `millis()` instead of `delay()`.
2. **Measure distance and print it:** functions, types (`int`, `float`, `unsigned long`), `const`, the serial monitor. Is the reading believable? (Pip's `lib/clock_check` asks the same about its clock.)
3. **Drive the motors:** PWM (`ledcWrite`; check which Arduino core the platform brings, since 2.x and 3.x set up PWM differently), an `enum class Direction`, a `Motor` class (header and `.cpp`, constructor, member functions).
4. **Avoid obstacles:** a state machine (`enum class` state plus `switch`) driven by time and distance. Drive, see a wall, back up, turn, drive again.
5. **The Pip way:** move the decision logic into `lib/` with no Arduino calls, give it a tiny interface for "distance" and "motors", and unit-test it on the Mac with Unity.
6. **Extras:** a servo sweep to choose the clearer side, a buzzer with the car's own sounds, or a 3D-printed sensor mount or bumper (I have a Bambu P2S and design in build123d).

## Progress

- (Update after each session: what we built, what I learned, what's next.)
- 2026-11-19: parts ordered from Core Electronics. Some were backordered, so everything ships together, probably in early December. Next: solder the Zero's and the TB6612's headers, then stage 1.

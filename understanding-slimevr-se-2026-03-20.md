# SlimeVR-Tracker-nRF Understanding Notes

Date: 2026-03-20

This note summarizes the project understanding from today's discussion, with focus on the `boards/riven/slimevr_se` board and the `MMC5983MA` timeout issue.

## 1. Project Scope

- Workspace root: `E:\Documents\Projects\slimeVR\tracker`
- Main project: `SlimeVR-Tracker-nRF`
- Main business code: `SlimeVR-Tracker-nRF/src`
- Other top-level folders in the workspace are mostly Zephyr/Nordic SDK/modules/libraries.

The project is a Zephyr/NCS firmware for SlimeVR tracker hardware based on Nordic nRF SoCs.

## 2. How To Read This Project

Treat it as four layers:

1. Board / SoC description
2. System / power / retained state
3. Sensor scan / acquisition / calibration / fusion
4. Connection / packetization / ESB or HID output

Do not treat it as a simple single-threaded `main() -> loop` project.
The real behavior is driven by:

- `SYS_INIT(...)`
- `K_THREAD_DEFINE(...)`
- runtime thread creation

## 3. Important Top-Level Files

- `CMakeLists.txt`
  - pulls in all `src/*.c`
  - also links `Fusion/` and `vqf-c/`
- `prj.conf`
  - Zephyr/NCS feature switches
- `Kconfig`
  - app-level config entries, including sensor ODR, fusion mode, HID, power options

Key Kconfig entries discussed:

- `USE_SLIMENRF_CONSOLE`
- `USER_SHUTDOWN`
- `USE_IMU_WAKE_UP`
- `SENSOR_ACCEL_ODR`
- `SENSOR_GYRO_ODR`
- `SENSOR_USE_MAG`
- `SENSOR_USE_VQF`
- `CONNECTION_OVER_HID`

## 4. Boot And Runtime Flow

The startup path is roughly:

```text
Boot
-> Zephyr SYS_INIT handlers
-> retained/NVS/config restore
-> main() handles reset/button/DFU/shutdown logic
-> sensor scan thread starts
-> sensor_scan() identifies IMU and MAG
-> sensor_init() configures sensor + fusion
-> sensor_loop() reads FIFO and produces orientation
-> connection thread chooses packet types
-> ESB or HID sends packets
```

Important insight:

- `main()` is not the main business loop.
- `main()` mostly decides reset behavior, shutdown behavior, and special boot actions.

## 5. Main Runtime Actors

Think of these as the core long-lived threads:

- `sensor_loop`
  - the main producer of orientation data
- `connection_thread`
  - packet scheduling and serialization
- `esb_thread`
  - pairing and radio-link state
- `power_thread`
  - battery, charger, dock, sleep, shutdown
- `calibration_thread`
  - calibration state machine
- `status_thread`
  - status LED behavior
- `button_thread`
  - press / multi-press / hold behavior

Priority overview:

- priorities are defined in `src/thread_priority.h`
- lower numeric value means higher priority
- most support threads are priority `6`
- `sensor_loop` is `7`
- `connection_thread` is `8`

This means:

- power / calibration / button / status / ESB logic can preempt the sensor loop
- sensor loop can preempt connection packetization

So logs do not always appear in a simple straight-line order.

## 6. Core File Map

### System / boot / retained

- `src/main.c`
  - boot decision logic
- `src/system/system.c`
  - retained/NVS handling, GPIO setup, button logic, reset modes
- `src/system/power.c`
  - sleep / WOM / shutdown / reboot / charger / battery / dock handling
- `src/retained.c`
  - retained RAM validation and checksum
- `src/config.c`
  - runtime settings backed by retained + NVS

### Sensor path

- `src/sensor/sensor.c`
  - scan, init, loop, fusion flow
- `src/sensor/interface.c`
  - SPI/I2C/ext abstraction
- `src/sensor/sensors.h`
  - scan tables and driver mapping
- `src/sensor/calibration.c`
  - calibration logic and calibration thread
- `src/sensor/fusion/*`
  - fusion algorithm backends

### Connection path

- `src/connection/connection.c`
  - packet building and send scheduling
- `src/connection/esb.c`
  - pairing and ESB transmission
- `src/hid.c`
  - HID report buffering
- `src/usb.c`
  - USB connect/disconnect handling

## 7. Sensor Data Path

The main sensor path is:

```text
sensor_request_scan()
-> sensor_scan()
-> sensor_init()
-> sensor_loop()
-> sensor_calibration_process_*
-> sensor_fusion->update_*
-> sensor_fusion->get_quat()
-> connection_update_sensor_data()
-> connection_thread()
-> esb_write() / hid_write_packet_n()
```

What happens in practice:

1. detect IMU and MAG
2. choose driver implementation from tables
3. configure ODR / full-scale / fusion
4. read IMU FIFO
5. read MAG
6. apply calibration
7. run fusion
8. emit quaternion and linear acceleration
9. package and transmit

## 8. DTS As A Hardware Contract

The app is not just "using DTS normally". It has hard expectations about node names and properties.

Important examples:

- `aliases.sw0`
  - used for button logic
- `/zephyr,user/int0-gpios`
  - used for IMU interrupt / wake
- `/zephyr,user/dock-gpios`
  - optional dock sense
- `/zephyr,user/chg-gpios`
  - charge state input
- `/zephyr,user/stby-gpios`
  - standby / charged state input
- `/zephyr,user/clk-gpios`
  - optional external clock enable
- `/zephyr,user/dcdc-gpios`
  - regulator control
- `/zephyr,user/ldo-gpios`
  - regulator control
- `/zephyr,user/chg_en-gpios`
  - charger enable control
- `/zephyr,user/imu-ldo-en-gpios`
  - IMU power control
- `imu` / `imu_spi`
  - IMU bus node
- `mag` / `mag_spi`
  - MAG bus node
- `/battery-divider`
  - battery ADC path
- `retainedmemdevice`
  - retained RAM alias from SoC overlay

When changing DTS, the question is not only "does it compile", but also "does it satisfy the names and GPIO semantics the app expects".

## 9. Board Focus: `boards/riven/slimevr_se`

Current board file:

- `boards/riven/slimevr_se/slimevr_se.dts`

Key facts:

- IMU and MAG are both on `i2c0`
- addresses are left at `reg = <0>` to allow runtime scan
- IMU interrupt:
  - `int0-gpios = <&gpio0 12 0>`
- MAG interrupt:
  - `mag-int-gpios = <&gpio1 0 0>`
- IMU LDO control:
  - `imu-ldo-en-gpios = <&gpio0 8 (GPIO_OPEN_DRAIN | GPIO_PULL_UP)>`
- battery divider exists

Important note:

- `mag-int-gpios` was newly added by you
- it is not yet integrated into the main business logic in a meaningful way

## 10. Current Status Of `mag-int-gpios`

The codebase currently knows that the property may exist, but does not actually use it for behavior.

Observed state:

- `src/system/power.c` checks whether `mag_int_gpios` exists and defines `MAG_INT_EXISTS`
- no real GPIO configuration, callback, wait path, or sensor flow currently depends on it
- there is no proper sensor-layer abstraction for magnetometer interrupt readiness

So as of today:

- `mag-int-gpios` is present in DTS
- `mag-int-gpios` is not yet a working runtime feature

Conclusion:

- it will not currently solve the `MMC5983MA` timeout issue by itself

## 11. `MMC5983MA` Timeout Analysis

Relevant driver:

- `src/sensor/mag/MMC5983MA.c`

Observed log:

- `MMC5983MA: mmc_mag_read: Read timeout`

Important interpretation:

- this is not the same as an I2C communication failure
- the timeout is produced while waiting for measurement-ready status in one-shot mode
- communication failures in this driver are logged as `Communication error`

### Driver behavior

`mmc_mag_read()` does this:

1. if one-shot was triggered, poll the status register
2. wait for ready bit
3. timeout after 2 ms
4. read 7 bytes of magnetometer output

This means the timeout says:

- a one-shot measurement was expected to complete
- the ready bit did not become ready before the current 2 ms timeout

### Why one-shot is being used

The sensor framework dynamically changes magnetometer mode:

- in `src/sensor/sensor.c`, the code may call `sensor_mag->update_odr(INFINITY, ...)`
- for `MMC5983MA`, this maps to one-shot mode
- then the main loop does:
  - `mag_oneshot()`
  - later `mag_read()`

So the repeated timeout strongly suggests:

- the board is currently hitting the one-shot path
- and the 2 ms ready wait is too aggressive or otherwise not robust enough for this setup

## 12. What The Timeout Most Likely Is Not

From today's reading, the timeout is less likely to mean:

- DTS node missing
- `mag-int-gpios` missing
- the MAG device is completely absent

Because if the device were simply not communicating, the more likely logs would be from failed register access rather than repeated ready-bit timeout.

## 13. Two Practical Next Steps

### Option A: quick stabilization

Use the smallest possible driver fix first:

- increase the one-shot wait timeout in `MMC5983MA.c`
- or avoid one-shot mode and keep the sensor in continuous mode for now

Why this is attractive:

- minimal code change
- matches current architecture
- likely enough to stop the spam and verify the rest of the path

### Option B: proper `mag-int-gpios` integration

If the goal is to make `mag-int-gpios` real, that needs a broader change:

1. extend the magnetometer abstraction in `src/sensor/sensor.h`
2. add GPIO setup and synchronization for MAG INT in `src/sensor/sensor.c`
3. teach `MMC5983MA` driver to configure and use the interrupt behavior
4. define how the main loop waits for or consumes MAG-ready signaling

This is a cleaner long-term solution, but it is not a small patch.

## 14. Best Mental Model For Debugging Tomorrow

When something breaks, check in this order:

1. DTS contract
   - are the expected nodes and GPIO polarities correct
2. scan
   - was IMU/MAG actually detected
3. init
   - did ODR/FS/fusion setup complete
4. runtime acquisition
   - is FIFO being read, is MAG being read, are interrupts firing
5. fusion output
   - is quaternion being updated
6. connection output
   - are packets being scheduled and sent

Specific heuristics:

- "sensor detected but no orientation"
  - suspect FIFO, IMU interrupt, or `fifo_process`
- "USB info works but ESB tracking does not"
  - inspect pairing / ESB thread / connection scheduling
- "boots then immediately sleeps or powers off"
  - inspect dock / charge GPIO polarity and power thread logic
- repeated `MMC5983MA` read timeout
  - first suspect one-shot readiness timing, not yet `mag-int-gpios`

## 15. Recommended Resume Plan For Tomorrow

Suggested order:

1. confirm the exact timeout pattern in logs
2. decide whether tomorrow's goal is:
   - quick fix for `MMC5983MA`
   - or full `mag-int-gpios` integration
3. if quick fix:
   - inspect one-shot timeout and continuous-mode behavior
4. if full integration:
   - design the magnetometer interrupt abstraction first

## 16. Short Summary

- The project is thread-driven, not `main()`-driven.
- The key runtime path is `sensor_loop -> connection_thread -> esb/hid`.
- `boards/riven/slimevr_se` currently uses I2C for both IMU and MAG.
- `mag-int-gpios` exists in DTS but is not yet truly wired into business logic.
- `MMC5983MA: mmc_mag_read: Read timeout` currently points more to one-shot ready timing than to a missing DTS property.
- The safest next move is probably a minimal MMC5983MA stabilization first, then proper MAG interrupt integration if still needed.

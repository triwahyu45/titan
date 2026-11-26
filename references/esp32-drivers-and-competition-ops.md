# ESP32 / Arduino C++ Competition Drivers & Field Ops

Battle-tested ESP32 C++ driver patterns for omni/transporter robots, IMUs, PCA9685 motor buses, Bluepad32 gamepads, and bare-metal RMT WS2812 LEDs.

---

## 1. Bare-Register MPU6050/6500/9250/Clone IMU Driver

- **Clone-Safe `WHO_AM_I` Check**: Accept `0x68`, `0x70`, `0x71`, `0x72`, `0x73` so clone IMU modules initialize reliably without silent failure.
- **Fast Low-Latency Config**:
  - 14-byte I2C burst read starting at `0x3B` (accel + temp + gyro).
  - `DLPF = 0x02` (`~98 Hz` / `8 ms` delay), `SMPLRT_DIV = 0x04` (`~200 Hz`), `±500 dps` (`65.5 LSB/dps`), `±4g` (`8192 LSB/g`).
  - Complementary filter (`alpha = 0.96`) for roll/pitch; gyro Z EMA low-pass (`0.85 * filt + 0.15 * raw`) + `0.25 dps` deadzone + `wrap180()`.
  - Persist calibrated gyro offsets in ESP32 NVS (`Preferences`) so match reboots skip waiting for stationary calibration.

---

## 2. PCA9685 + ZK-5AD (`TA6586`) / TB6612 Motor Driver Gotchas

- **ZK-5AD (`TA6586`) Pin Logic**: Unlike TB6612, ZK-5AD has no separate speed-only PWM pin. Send PWM duty directly to `IN1` (forward) or `IN2` (reverse) while holding `ENA` (`PWM` pin) **HIGH**.
  - **Short-Brake**: `IN1=HIGH, IN2=HIGH, ENA=HIGH` (if `ENA=LOW`, motor coasts instead of braking).
- **Non-Linear Low-RPM Motor Deadband (`20–25%`)**:
  - Cheap geared motors exhibit higher actual RPM than linear feedforward at `<50%` PWM; closed-loop RPM PID with `KI > 0` winds up and stalls inside the deadband.
  - Prefer open-loop joystick PWM or a heading-hold **snap floor (`>= 35%`)** strictly above the motor deadband, paired with slew-rate limiting (`~800 RPM/s`) and encoder fault checks enabled only on pins with reliable pull-ups.

---

## 3. Dual-Boot Competition vs WiFi Web-Config Mode (`Preferences`)

- Store `boot.mode` (`0 = Competition / Bluepad32`, `1 = WiFi WebServer + mDNS Config`) in NVS `Preferences` so WiFi radio never interferes with Bluetooth gamepad latency during matches, while PID gains, servo bounds, and anti-tip pitch/roll limits remain live-tunable over HTTP in the pit.
- **Arena Gamepad Safety**: Whitelist the exact controller MAC (`<CONTROLLER_BT_MAC>`) in `Bluepad32`, add pitch/roll anti-tip lifter throttling, and use delta-angle tracking on right-stick absolute heading to avoid `±180°` wrap jumps.

---

## 4. Zero-Jitter Bare-Metal ESP32 RMT Driver for WS2812

- Use ESP32 hardware `RMT` (`clk_div = 4` → `20 MHz` / `50 ns` ticks: `T0H=350ns/T0L=800ns`, `T1H=700ns/T1L=600ns`, `reset=300us`).
- Call `rmt_wait_tx_done(channel, portMAX_DELAY)` before refilling the buffer, then transmit asynchronously (`rmt_write_items(..., false)`) so LED updates never disable interrupts or starve motor/Bluepad32 loops.

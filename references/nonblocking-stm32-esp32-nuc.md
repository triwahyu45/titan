# Non-Blocking STM32, ESP32 FreeRTOS & 3-Tier NUC Robotics Architecture

Field-proven non-blocking firmware and 3-tier comms pattern for multi-motor omni/holonomic robots (NUC/MiniPC ↔ ESP32 Bridge ↔ STM32F4 Motion Controller).

---

## 1. STM32F4 Zero-Blocking Super-Loop (`while(1)`)

Never block the main control loop with `HAL_Delay()` during motion, homing, or UART polling. Run a deterministic tick-scheduled super-loop (`HAL_GetTick()` or hardware timer flag):

- **Timer Init vs Loop**: Call `HAL_TIM_PWM_Start()` and `HAL_TIM_Encoder_Start()` **once** in initialization. Inside `while(1)`, only update duty cycle via `__HAL_TIM_SET_COMPARE(&htim, channel, pwm)`.
- **State-Machine Homing (`Set_Limit`)**: Replace blocking `while (!limit_pressed)` loops with an explicit enum state (`HOMING_IDLE`, `HOMING_SEEK`, `HOMING_BACKOFF`, `HOMING_DONE`) checked each loop tick against limit-switch GPIO.
- **`smoothSpeed` Slew-Rate Ramp**: Limit acceleration per tick (`current += clamp(target - current, -max_delta, +max_delta)`) to prevent wheel slip, brownouts, and gearbox shock.
- **Stiction Floor Compensation**: Add minimum deadband PWM (`if (fabs(out) > eps) out += copysign(MIN_PWM, out)`) so low-speed PID corrections overcome static friction.

---

## 2. 3-Wheel Dead-Wheel Odometry & Field-Oriented Control

For an omni-wheel base with Left (`dn_L`), Right (`dn_R`), and perpendicular Auxiliary (`dn_aux`) dead-wheel encoders (`L` = track width, `B` = aux wheel offset from rotation center):

```c
float dtheta = cm_per_tick * (float)(dn_R - dn_L) / L;
float dx     = cm_per_tick * (float)(dn_L + dn_R) * 0.5f;
float dy     = cm_per_tick * ((float)dn_aux - (float)(dn_R - dn_L) * (B / L));

// Midpoint integration reduces arc discretization error
float theta_mid = pos.h + dtheta * 0.5f;
pos.x += dx * cosf(theta_mid) - dy * sinf(theta_mid);
pos.y += dx * sinf(theta_mid) + dy * cosf(theta_mid);
pos.h += dtheta;
```

- **Field-Oriented Transform**: Rotate desired global `(vx_world, vy_world)` by `-pos.h` before feeding inverse kinematics (`vx_robot = vx_w * cos(h) + vy_w * sin(h)`, `vy_robot = -vx_w * sin(h) + vy_w * cos(h)`).

---

## 3. ESP32 Dual-Core FreeRTOS Bridge (`xTaskCreatePinnedToCore`)

Isolate time-critical gamepad/UART streaming from slow blocking sensors (e.g., `NewPing` ultrasonic):

- **Core 0 (Fast Loop)**: PS4/BLE controller polling + fixed-rate UART packet TX to STM32/NUC.
- **Core 1 (Slow Loop)**: Ultrasonic (`NewPing`), battery ADC, LED status, and diagnostics.
- **Edge-Triggered Button Latch**: Track `prev_btn` state (`pressed_edge = curr_btn && !prev_btn`) instead of raw level checks so single presses do not fire repeatedly across ticks.
- **Fixed `snprintf` TX Buffers**: Format packets into static `char tx_buf[128]` with `snprintf(...)`—avoid Arduino `String` concatenation inside loops.

---

## 4. 3-Tier NUC / MiniPC Serial & `*...#` Framing

- **Persistent Udev Symlinks**: Bind `/dev/ESP32` (`115200` baud) and `/dev/FTDI1` (`921600` baud) via USB vendor/product/serial udev rules so port numbers never swap on reboot.
- **`*...#` Circular Framing**: Wrap telemetry/command payloads as `*<payload>#`. Parse incoming bytes into a fixed ring buffer on interrupt/DMA, resetting cleanly on overflow or missing terminator.

---

## 5. Five Verified Field Firmware Pitfalls

| # | Pitfall | Root Cause | Verified Fix |
|---|---------|-----------|--------------|
| 1 | **Copy-paste PID returns wrong axis** | Duplicated `PID_X()` into `PID_Y()` but left `return out_x;` or wrong integral/derivative state | Parameterize a single `PID_Compute(PID_t *pid, float err, float dt)` struct function—never duplicate axis code |
| 2 | **`HAL_UART_Receive_IT()` inside `while(1)`** | Re-arming IT in the main loop drops bytes and triggers ORE overrun flags under burst traffic | Use `HAL_UARTEx_ReceiveToIdle_DMA()` + circular ring buffer; re-arm only in the RX callback |
| 3 | **`HAL_TIM_PWM_Start()` per loop iteration** | Calling HAL start every tick resets/glitches timer state and wastes CPU | Start PWM/Encoder timers once in `MX_*_Init()` / setup; update duty only via `__HAL_TIM_SET_COMPARE()` |
| 4 | **ESP32 shared globals & `String` heap fragmentation** | Unprotected cross-core writes + dynamic `String` concat cause torn reads and random Guru Meditation panics | Use `portMUX_TYPE` (`taskENTER_CRITICAL`) or FreeRTOS queues + fixed `char[]` with `snprintf()` |
| 5 | **Python `serial.Serial()` inside `while True` & unbounded `.append()`** | Re-opening serial every iteration resets DTR/MCU; unbounded lists cause OOM on long runs | Open `serial.Serial` once outside the loop (with try/except reconnect) and store history in `collections.deque(maxlen=N)` |

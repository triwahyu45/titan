# Robotics Control, Sensor Fusion, Planning & Physics Simulation

Field-verified control-theory, navigation, telemetry, and pre-hardware `Python/NumPy` simulation rules for differential-drive, balancing, and autonomous mobile robots.

---

## 1. Line Follower Geometry & Rollover Physics

- **Sensor Boom Lookahead vs Motor Lag (`~30 ms`)**:
  - Too short (`20 mm`) → robot oscillates side-to-side and loses straight lines at `1 m/s`.
  - Too long (`140 mm`) → robot cuts corners (`~25 mm` inside a `150 mm` curve vs `6 mm` at `60 mm` boom).
  - Higher target speed (`2 m/s+`) requires longer lookahead; real tyres saturate around `~1 g` lateral grip.
- **Differential Turn Rate**: `omega = (v_R - v_L) / track_width`. Narrower track turns faster (`0.5 m/s` delta at `100 mm` = `286 deg/s` vs `150 mm` = `191 deg/s`) but tips easier.
- **Lateral Rollover Limit**: `a_tip = (track_width / 2) / h_CG * g`. Keep chassis plate flat and battery low (`100 mm` track + `20 mm` CG = `2.5 g` rollover vs `60 mm` CG = `0.83 g`).

---

## 2. Real-World PID Pitfalls & Verified Fixes

| Pitfall | Mechanism | Verified Fix |
|---|---|---|
| **Integral Windup** | When actuator output saturates (e.g., motor capped at `3 V` or max PWM) while robot is held/blocked, `I += e * dt` keeps growing and catapults past setpoint on release | **Conditional integration**: freeze `I` accumulation whenever motor output is saturated (still allow unwinding if `sign(e)` opposes `I`) |
| **Derivative Noise Chatter** | Raw `Kd * (e_k - e_{k-1}) / dt` amplifies high-frequency sensor/encoder jitter into violent motor chatter | **Exponential low-pass filter** before `D`: `smooth = 0.75 * smooth + 0.25 * raw` (`alpha ≈ 0.25`) |

---

## 3. IMU Sensor Fusion (Accel + Gyro + Mag)

- **Accelerometer**: Measures linear acceleration + gravity (`roll`/`pitch` only, **zero yaw/heading**); noisy under vibration/jolts.
- **Gyroscope**: Measures angular rate smoothly and instantaneously, but integrating small bias errors causes continuous angle **drift** (`~1°/s` uncalibrated).
- **Sensor Fusion**: Combine via complementary filter, Madgwick, or EKF (high-pass gyro for fast transients + low-pass accel to cancel roll/pitch drift; add a magnetometer in 9-DOF chips like BNO055 to anchor yaw).

---

## 4. Non-Minimum Phase Control (Self-Balancing Robot)

- Required steady lean angle for forward acceleration `a` is `tan(theta) ≈ a / g` (`0.5 m/s²` → `~3°`).
- Starting upright, the only way to tip the center of gravity forward is to **pull the wheels backward first** (`~15 mm` in `~0.125 s` for `0 → 0.5 m/s`), then drive forward to chase the fall.
- **Hard Physics Law**: No controller can eliminate this initial opposite-direction kickback (*non-minimum phase*); only commanding gentler acceleration shrinks it.

---

## 5. Path Planning (`Nav2 Smac-2D A*`) & Telemetry (`MCAP`)

- **Nav2 Planner Choice**: Stock Nav2 `navfn` defaults to Dijkstra (`~1,546` expanded cells). Use **Smac Planner 2D** with **octile distance** heuristic `f = g + h` over an inflated costmap (`~222–474` expanded cells) with parent-pointer backtracking.
- **Telemetry Format (`MCAP`)**: Record multi-sensor fleet/robot logs in **MCAP** (`.mcap`) instead of ad-hoc CSV/JSON dumps so timestamps stay indexed for root-cause correlation (e.g., correlating battery voltage sag `15.9V → 13.8V` with lockstep motor RPM collapse).

---

## 6. Pre-Hardware Physics Verification (`Python/NumPy RK4 500 Hz`)

- Validate control laws in a lightweight `500 Hz` **Python/NumPy RK4** ODE model using real datasheet motor torque, back-EMF, voltage ceiling, and measured part masses before flashing hardware.
- Downsample state snapshots to `60 fps` JSON to drive `STUDIO` (`SIM-GRID` Three.js/Remotion) for visual debugging of overshoot, windup, and trajectory tracking.

# CHIOS Firmware Architecture & Control Algorithms

Reverse-engineered from `Firmware` & `RobotUi` in the Ichibot Ultimate 5 Max simulator (`robotics.ichibot.id/simulator`), cross-checked with ESP32 `Chios 8.0.2 Beta` firmware metadata (`ESP-IDF v4.4.7`, Arduino lib-builder, SPIFFS + WiFi + MQTT `umqtt.ichibot.id`).

## Critical Pitfalls & Verified Fixes

- **[Default REA `capture_browser_scenario` 64 KiB body cap]** -> Truncates or omits Next.js Turbopack chunks (>64 KB) -> **Hard Rule**: Always pass `capture.limits` with `max_network_body_bytes: 4194304`, `max_network_body_allocation_bytes: 16777216`, and `max_total_event_allocation_bytes: 33554432`.
- **[`TA` / `TB` in `mS` mode runs `2×` longer than entered]** -> Firmware applies `TIMER_DELAY_MULTIPLY = 2` (`2 * TA`, `2 * TB`), whereas `D` (`mode_delay == 0`) runs `1 * D` ms -> **Hard Rule**: Divide desired milliseconds by `2` when setting `TA`/`TB` in `mS` mode.
- **[UI labels `mode_TA/TB == 1` as `"cm"` while firmware compares millimeters]** -> `odoPathMm()` returns `mm` and is compared directly (`Math.abs(odoPathMm()) >= Math.abs(TA)`); likewise `encL`/`encR` compare `encDistMm()` in `mm`, not raw encoder ticks -> **Hard Rule**: Treat `TA`/`TB`/`D` distance values as **millimeters** when porting or tuning.
- **[`SENSOR_MODE[0]` and `SENSOR_MODE[13]` visual vs `detectRaw` mismatch]** -> UI displays `SENSOR_MODE[0] = 0x0618` (`___■■____■■___` EQUAL) and `SENSOR_MODE[13] = 0x2000` (`■_____________` OR), but `detectRaw` actually evaluates `0x3333` (`■■__■■__■■__■■` EQUAL) for value `0` and `0x3000` (`■■____________` OR) for value `13` -> **Hard Rule**: Match `detectRaw` exact bitmasks in firmware, not the visual preview table.
- **[Reversed sensor bit order]** -> `bit 13` (`0x2000`, `S14`) is **FAR LEFT** (`+y = +61.8 mm`), and `bit 0` (`0x0001`, `S1`) is **FAR RIGHT** (`-y = -61.8 mm`) -> **Hard Rule**: Map `S14 -> MSB (13)` and `S1 -> LSB (0)`.
- **[`rpmWheel` `30/π` scaling quirk]** -> `rpmWheel` computes `(deltaTicks / PPR) / dt / π * 30` (which is `RPM / 2π`), compensated by `rpmMultiplierX10 = 11` (`1.1x`) -> **Hard Rule**: Keep `rpmMultiplierX10` coupled to the exact `rpmWheel` scale factor when porting.
- **[Auto Line-Finder skipped unexpectedly]** -> After `D` completes, the `0.8 * L, 0.8 * R` line-search loop is skipped if `L == 0 || R == 0`, OR if `stopAtStep == 0 && indexNow >= stopAtIndex`, OR if next index is `sensor_type == 0` with `TA == 0 && TB == 0` -> **Hard Rule**: Never leave `L=0` or `R=0` on a turn if you expect Auto Line-Finder to catch the line.
- **[Calibration floor/ceiling shrink]** -> `calMin = trunc(2.3 * rawMin)` and `calMax = trunc(0.9 * rawMax)`, and `readLineSensor` ignores any channel where `limBlack >= calMax` (black) or `limWhite <= calMin` (white) -> **Hard Rule**: Ensure `rawMax > 2.6 * rawMin` contrast during calibration.

---
## 1. Hardware & Coordinate Frame

- **MCU Target (Physical)**: ESP32-WROOM (`Chios 8.0.2 Beta`), SPIFFS for plan/settings persistence, 128×128 monochrome OLED (`SH1107`-style buffer), 6 tactile buttons (`UPL`, `OKL`, `DWNL`, `UPR`, `OKR`, `DWNR`), active buzzer (`2700 Hz`), RGB/status LED (`action=#ef4444`, `sa=#22c55e`, `sb=#3b82f6`), 14-channel IR sensor array, 2× DC motors with quadrature encoders (`192 PPR`), optional fan & gripper.
- **Body Coordinate Frame**: Origin `(0, 0)` is the midpoint of the rear drive axle.
  - `+x` = Forward (mm), `+y` = Left (mm), `heading` = yaw angle (rad, CCW positive).
  - Chassis footprint corners (`R` in mm): `[160, -98]`, `[160, 98]`, `[-27, 98]`, `[-27, -98]`.
  - Front skid / ball caster at `x = +110 mm` relative to axle (`skidX = (110 - comOffsetMm)/1000`).
  - Default drivetrain: `wheelDiameterMm = 38`, `wheelbaseMm = 168`, `encoderPpr = 192`.

### 14-Channel Curved Sensor Arc (`SENSOR_LAYOUT`, mm from axle)
`bit 0` (`S1`) is **Far Right (`-y`)**; `bit 13` (`S14`) is **Far Left (`+y`)**:

| Bit | Label | `x` (mm) | `y` (mm) | Side | Bit | Label | `x` (mm) | `y` (mm) | Side |
|---:|:---|---:|---:|:---|---:|:---|---:|---:|:---|
| `0` | S1 | `103.8` | `-61.8` | Far Right | `7` | S8 | `142.8` | `+4.75` | Center-Left |
| `1` | S2 | `121.1` | `-57.0` | Outer Right | `8` | S9 | `142.8` | `+14.25` | Inner Left |
| `2` | S3 | `134.7` | `-46.2` | Mid Right | `9` | S10 | `142.8` | `+24.4` | Mid Left |
| `3` | S4 | `142.8` | `-35.3` | Right | `10` | S11 | `142.8` | `+35.3` | Left |
| `4` | S5 | `142.8` | `-24.4` | Mid Right | `11` | S12 | `134.7` | `+46.2` | Mid Left |
| `5` | S6 | `142.8` | `-14.25` | Inner Right | `12` | S13 | `121.1` | `+57.0` | Outer Left |
| `6` | S7 | `142.8` | `-4.75` | Center-Right | `13` | S14 | `103.8` | `+61.8` | Far Left |

---

## 2. Sensor Calibration & Binarization

1. **Auto-Rotate Calibration (`calibrateRotate`)**:
   - Robot spins in place: `setMotor(+speedCalibrate, -speedCalibrate)` (default `speedCalibrate = 40`).
   - Tracks `calMinSample[0..13]` and `calMaxSample[0..13]` from 12-bit ADC (`0..4095`).
   - Stops when `|odoThetaDeg()| >= 720°` OR `elapsed >= 4000 ms`.
2. **Threshold Calculation (`applyLimits`)** for slot `cal ∈ [0..4]`:
   ```c
   calMin[ch][cal] = (int)(2.3f * rawMin[ch]);   // 23 * rawMin / 10
   calMax[ch][cal] = (int)(0.9f * rawMax[ch]);   //  9 * rawMax / 10
   int span = calMax[ch][cal] - calMin[ch][cal];
   limBlack[ch][cal] = (int)(span * (100 - sensorSensitivity) / 100.0f) + calMin[ch][cal];
   limWhite[ch][cal] = (int)(span * sensorSensitivity / 100.0f) + calMin[ch][cal];
   ```
   Default `sensorSensitivity = 18` (`82%` of span for black threshold, `18%` for white threshold).
3. **Binarization (`readLineSensor(line_color, NUM_CAL)`)**:
   - `line_color == 0` (**BLACK line**): bit `n` is `1` iff `adc[n] > limBlack[n][cal] && limBlack[n][cal] < calMax[n][cal]`.
   - `line_color == 1` (**WHITE line**): bit `n` is `1` iff `adc[n] < limWhite[n][cal] && limWhite[n][cal] > calMin[n][cal]`.

---

## 3. Line Error Extraction (`followLine`)

### Mode 0: `FL.CENTER` (27-Entry Priority Table)
Evaluated top-to-bottom; first match sets `errPos ∈ [-16, +16]` and resets `maskShift = 0`:

| Priority | Bits `(a, b)` | `errPos` | Priority | Bits `(a, b)` | `errPos` | Priority | Bits `(a, b)` | `errPos` |
|---:|---|---:|---:|---|---:|---:|---|---:|
| 1 | `7 & 6` | `0` | 10 | `4` | `-5` | 19 | `11` | `+9` |
| 2 | `6` | `-1` | 11 | `9` | `+5` | 20 | `2 & 1` | `-10` |
| 3 | `7` | `+1` | 12 | `4 & 3` | `-6` | 21 | `11 & 12` | `+10` |
| 4 | `6 & 5` | `-2` | 13 | `9 & 10` | `+6` | 22 | `1` | `-11` |
| 5 | `7 & 8` | `+2` | 14 | `3` | `-7` | 23 | `12` | `+11` |
| 6 | `5` | `-3` | 15 | `10` | `+7` | 24 | `1 & 0` | `-12` |
| 7 | `8` | `+3` | 16 | `3 & 2` | `-8` | 25 | `12 & 13` | `+12` |
| 8 | `5 & 4` | `-4` | 17 | `10 & 11` | `+8` | 26 | `0` | `-16` |
| 9 | `8 & 9` | `+4` | 18 | `2` | `-9` | 27 | `13` | `+16` |

### Mode 1 (`FL.LEFT`) & Mode 2 (`FL.RIGHT`) — Fork-Biasing `maskShift`
1. Masks `sensorBits`:
   - `FL.LEFT`: `c = sensorBits & ((0x3FFF << maskShift) & 0x3FFF)` (ignores sensors to the right of previous outer-left hit).
   - `FL.RIGHT`: `c = sensorBits & (0x3FFF >> maskShift)` (ignores sensors to the left of previous outer-right hit).
2. Priority scan (for `FL.LEFT`; `FL.RIGHT` mirrors bits `13 - k` and negates `errPos`):
   - `bit 13` -> `errPos=+13, maskShift=11` | `bit 12` -> `+10, 10` | `bit 11` -> `+8, 9` | `bit 10` -> `+6, 8` | `bit 9` -> `+4, 7` | `bit 8` -> `+2, 0`
   - `(bit 6 & bit 7)` -> `0, 0` | `bit 7` -> `+1, 0` | `bit 6` -> `-1, 0` | `bit 5` -> `-2, 0` | `bit 4` -> `-4, 0` | `bit 3` -> `-6, 0` | `bit 2` -> `-8, 0` | `bit 1` -> `-10, 0` | `bit 0` -> `-13, 0`.
3. **Lost-Line Recovery Kick**: If `c == 0 && -8 < errPos < 8`, force `errPos = (FL.LEFT ? -13 : +13)` and `maskShift = 0`.
4. **Global Lost-Line Timeout**: If raw `sensorBits == 0` continuously for `>= settings.lostLine` ms (default `300 ms`, `0` = disabled), stop motors and set `lostLine = true`.

---

## 4. Dual Run Control: LQR vs PID & Asymmetric Mixing

Loop executes whenever `millis() - lastProcess > DT` (note strict `>` in simulator, so `DT = 2 ms` ticks every `2.5 ms` or `2 ms` depending on step; `dt_s = (now - lastProcess) / 1000.0`). Also sets `rpmSetInterval(DT)`.

### A. LQR (`runControl == 0`, Default)
Analytic 32-iteration Riccati solver `lqrSolve(Q0, Q1, Q2, R)` (`R = max(1, R)`):
```c
float rv = sqrtf(Q0 * R), k1_term = sqrtf(R * Q1), k2_term = 0.0f;
for (int i = 0; i < 32; i++) {
    k2_term = sqrtf(2.0f * R * k1_term + R * Q2);
    k1_term = sqrtf(2.0f * rv * k2_term + R * Q1);
}
k0 = rv / R;  k1 = k1_term / R;  k2 = k2_term / R;
```
- Derivative with optional 1st-order LPF (`Tf` in ms): `h = Tf / 1000.0`. If `h <= 0`: `edot = rawEdot`; else `edot += (dt / (dt + h)) * (rawEdot - edot)`.
- Integral (`k0 > 0`): Zeroed if `(sensorBits & 0x01E0) != 0` (center 4 sensors `S6..S9` on line). Otherwise accumulated and clamped to `±(0.5 * Max / k0)`.
- Output: `u = k0 * integral + k1 * errPos + k2 * edot`.

**Factory LQR Presets** (`R = 10, Q0 = 0, Tf = 0, DT = 2`):
| Slot | `Q1` | `Q2` | `Max` | `Min` | Solved `k1` | Solved `k2` |
|---:|---:|---:|---:|---:|---:|---:|
| LQR 1 | `122` | `740` | `230` | `100` | `3.49` | `9.00` |
| LQR 2 | `276` | `3135` | `255` | `150` | `5.25` | `18.00` |
| LQR 3 | `490` | `7150` | `255` | `170` | `7.00` | `27.00` |
| LQR 4 | `766` | `12785` | `255` | `200` | `8.75` | `36.00` |
| LQR 5 | `1322` | `20020` | `255` | `255` | `11.50` | `45.00` |

### B. PID (`runControl == 1`)
- `P = errPos * Kp`, `D = (errPos - lastError) * Kd / dt_s`.
- Integral: Zeroed if `(sensorBits & 0x00C0) != 0` (center 2 sensors `S7, S8`). Otherwise `sum += errPos * dt_s; I = sum * Ki`.
- **Factory PID Presets** (`DT = 2`): `(Kp, Ki, Kd, Max, Min)` = `(4,4,10,230,100)`, `(6,6,20,255,150)`, `(8,10,30,255,170)`, `(10,16,40,255,200)`, `(13,18,50,255,255)`.

### C. Asymmetric `1.7×` Differential Steering Mix
```c
if (u >= 0) { // Line is to the left (+errPos) -> slow/brake left wheel harder
    L = baseSpeed - 1.7f * u;
    R = baseSpeed + u;
} else {      // Line is to the right (-errPos) -> slow/brake right wheel harder
    L = baseSpeed - u;        // -u is positive
    R = baseSpeed + 1.7f * u; // 1.7*u is negative
}
L = clamp(L, -Min, +Max);
R = clamp(R, -Min, +Max);
setMotor((int)L, (int)R);
```

---

## 5. Inner-Loop RPM PI Control & Odometry

- When `enableRpm == true`, `setMotor(cmdL, cmdR)` sets target `ref = cmd * (rpmMultiplierX10 / 10.0)` (default `1.1x`).
- Every `rpmInterval` ms (`5 ms` during actions, `DT` during `followLine`):
  - `meas = (rawTicks - lastRawTicks) / encoderPpr / dt_s / PI * 30.0`
  - If `ref == 0`: `sum = 0; pwm = 0`.
  - Else: `err = ref - meas`; `sum = clamp(sum + err * dt_s, -16.4516129f, +16.4516129f)`; `pwm = clamp(1.76f * err + 15.5f * sum, -255, 255)`.
- **Odometry**:
  - `ticksToMeters(t) = (t / encoderPpr) * PI * (wheelDiameterMm / 1000.0)`
  - `dL = ticksToMeters(encL - prevL)`, `dR = ticksToMeters(encR - prevR)`
  - `odoPath += 0.5 * (dL + dR)` (reported in `mm` via `odoPathMm()`)
  - `odoTheta += (dR - dL) / (wheelbaseMm / 1000.0)` (reported in `deg` via `odoThetaDeg() = -odoTheta * 180 / PI`)

---

## 6. Plan / Index Execution Pipeline (`*run` & `*action`)

1. **Start / Checkpoint**:
   - If `selectedCp == 0`: start at `indexNow = 0`, force trigger `e = 1` (executes `i:000` immediately).
   - If `selectedCp > 0`: `indexNow = cpIndex[selectedCp]`, execute `cpActions[selectedCp]`, then `indexNow++` and wait for trigger (`e = 0`).
2. **Main Loop**:
   - While trigger `detectNext(indexes[indexNow], sensorBits)` is `0`: call `followLine(defaultSpeed, RUN_MODE, PID_NUM)`.
   - `detectNext` requires **3 consecutive matching reads** (`MAX_SENSOR_DETECT_REPEAT = 3`) for `sensor_type == 1`.
3. **Action Execution (`action(idx)`)**:
   - **Jump (`ACT == 5`)**: Sets `jumpPlan = clamp(L, 0, 8)`, `jumpIndex = clamp(R, 0, 150)` and returns immediately.
   - **Brake (`B > 0`)**: `setMotor(-defaultSpeed, -defaultSpeed)` for `B` ms.
   - **Reset**: `resetEncoders()`, `resetOdometry()`, `rpmSetInterval(5)`.
   - **Turn/Motion (`D > 0`)**: `setMotor(L, R)` until `mode_delay` target reached: `0` = `D` ms, `1` = `|odoThetaDeg()| >= |D|`, `2` = `|encDistMm(L)| >= |D|`, `3` = `|encDistMm(R)| >= |D|`.
   - **Auto Line-Finder**: After `D > 0`, if `L != 0 && R != 0` and not stopping at Action and not chaining into `sensor_type==0, TA==0, TB==0`: run `setMotor(0.8*L, 0.8*R)` until `readLineSensor(line_color, NUM_CAL)` matches (`&= 0x1FF0` if `L < R`, `&= 0x03FE` if `L > R`).
   - **Actuators**: If `gripper_status` changed, stop motors & wait `700 ms`. If `fan_status ∈ [1..9]`, stop motors & wait `1000 * fan_status` ms.
   - **Speed A (`TA > 0`)**: Reset controllers & odometry. If `throttleMode > 0` (`[0,1,2,3]`), start speed at `max(40, trunc(0.5 * SA))` and increment by `throttleStep` every `2` loops up to `SA`. Run `followLine(speed, RUN_MODE, PID_NUM)` until `mode_TA` target (`0` = `2 * TA` ms, `1` = `odoPathMm`, `2` = `encDistMm(L)`, `3` = `encDistMm(R)`).
   - **Speed B (`TB > 0`)**: Reset controllers (and odometry if `mode_TB != 0`). Run `followLine(SB, RUN_MODE, PID_NUM)` until `mode_TB` target (`0` = `2 * TB` ms, `1..3` = distance) OR if `SB_MODE == 1` (`FREE`) and `detectNext(indexes[indexNow + 1])` fires early (returns trigger immediately!).
   - **Stop Check**: `stopAtStep` (`0` = after Action, `1` = after Speed A, `2` = after Speed B) halts robot when `indexNow >= stopAtIndex`.

### Mirror Index Transform (`mirrorIndex`)
When mirroring a Plan (left ↔ right):
- `ACT`: `1 (LEFT) <-> 2 (RIGHT)`
- `L, R` (when `ACT != 5`): swap `L <-> R`
- `mode_delay, mode_TA, mode_TB`: `2 (encL) <-> 3 (encR)`; if `1 (degree/cm)`, negate `D / TA / TB` (`-val`)
- `RUN_MODE`: `1 (FL.LEFT) <-> 2 (FL.RIGHT)`
- `sensor_value` (when `sensor_type == 1`): single-sensor pairs `23..29 <-> 36..30`, asymmetric mask pairs `6..9 <-> 13..16`, `10..12 <-> 17..19`; otherwise find index with matching `SENSOR_MODE` and flipped `LSM` (`0 <-> 1`).

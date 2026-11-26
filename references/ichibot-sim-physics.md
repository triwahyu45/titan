# Physics Engine, Track Sampling & Simulator Model

Derived from `S`, `T`, `TrackSampler`, `TRACKS`, and `CompetitionTracker` in `robotics.ichibot.id/simulator`.

---

## 1. 2D Rigid-Body Vehicle Dynamics (`dt = 0.25 ms` / `4 kHz`)

### Default Physical Parameters (`defaultPhysics`)
| Parameter | Value | Parameter | Value |
|---|---|---|---|
| `massKg` | `0.39 kg` | `iz` | `0.0037 * massKg` (`0.001443 kg·m²`) |
| `batteryV` | `12.0 V` | `batteryRintOhm` | `0.15 Ω` |
| `motorStallTorqueKgCm` | `0.8 kg·cm` | `motorNoLoadRpm` | `2856 RPM` |
| `motorRatedV` | `12.6 V` | `jw` (wheel inertia) | `3e-5 kg·m²` |
| `gearLossTorqueKgCm` | `0.1 kg·cm` | `gearEfficiency` | `0.5` |
| `tireMu` | `1.3` | `tireSlipMs` | `0.02 m/s` |
| `skidMu` | `0.15` | `driveWeightShare` | `0.75` (`25%` on front caster) |
| `comOffsetMm` | `30 mm` (ahead of axle) | `sensorNoise` | `0.04` |
| `loopPeriodMs` | `0.5 ms` (`2 kHz` FW) | `runVariation` | `1` |

### Force, Torque & Integration Equations
1. **3-Contact-Point Friction** at `[axleX, +halfTrack]`, `[axleX, -halfTrack]`, `[skidX, 0]` (`axleX = -0.03 m`, `skidX = +0.08 m`, `halfTrack = 0.084 m`):
   - Contact velocity: `vx = u - r * py`, `vy = v + r * px`.
   - Drive wheels (`e < 2`): slip `sx = w * r_w - vx`, `sy = -vy`, regularized norm `n = sqrt(sx² + sy² + tireSlipMs²)`, max friction `F_max = tireMu * nWheel * grip[e] * (1 + gripNoise[e])`, force `(Fx, Fy) = F_max * (sx, sy) / n`. Track `gripUse = hypot(Fx, Fy) / F_max`.
   - Front skid (`e == 2`): `F_skid = skidMu * nSkid`, opposing `(vx, vy)` with `25e-6` regularization.
2. **DC Motor & Back-Driving Drag**:
   - `kV = stallTorque / motorRatedV`, `kW = stallTorque / omegaNoLoad`, `rMotor = motorRatedV / (2.6 * motorRatedV / 6)`.
   - `tau_motor = gain * (kV * V_applied - kW * w)`. When braking (`tau_motor * w < 0`), multiplied by `1 + ((1 / gearEfficiency)² - 1) * (1 - |pwm| / 255)`.
   - Gear drag: `tLoss * tanh(w / 2)`.
   - Wheel angular acceleration: `dw/dt = (tau_motor - r_w * Fx_tire - tau_loss) / jw`.
3. **Rigid-Body & Battery Sag**:
   - `du/dt = sumFx / m + v * r`, `dv/dt = sumFy / m - u * r`, `dr/dt = sumMz / iz`.
   - Battery voltage sags with `tau = 2 ms` toward `V_nom - R_int * (I_L + I_R + 0.15)`.

---

## 2. Stochastic Run Variation & Optical Sensor Model

- **Per-Run Variation (`runVariation = 1`)**: Mulberry32 PRNG seeds per-run asymmetry: motor gain `±1%`, wheel radius `±0.2%`, tire grip `±1.5%`, start pose jitter `±0.75 mm` and `±0.3°`.
- **Grip Noise**: Ornstein-Uhlenbeck process (`sigma = 0.02`, `tau = 50 ms`) active only during `ACT` phases (`gripNoiseActive = true`).
- **Track Sampling (`TrackSampler`)**:
  - **Vector Tracks**: 40 mm spatial hash grid of line segments; exact point-to-segment distance `d`.
  - **Raster Tracks**: 2 mm/px grayscale SDF PNG; bilinear interpolation `(pixel - 128) / 4` gives signed edge distance in mm.
- **Reflectance & 12-bit ADC**:
  - Optical blur via Gaussian `erf`: `reflectance = 0.92 - 0.86 * (erf((halfWidth + 0.5 - d)/2) - erf((-halfWidth - 0.5 - d)/2))` (inverted inside `invertZones`).
  - Normalized line darkness `n = (0.92 - reflectance) / 0.86`.
  - Per-channel 12-bit ADC: `adc[ch] = clamp(round(S[ch] + (R[ch] - S[ch]) * n + uniform(-1,1) * sensorNoise * 3000), 0, 4095)`, where `S[ch] ∈ [300, 480]` (white) and `R[ch] ∈ [2700, 3400]` (black).

---

## 3. Built-In Tracks & Reference Plans

### 9 Built-In Tracks
1. `oval` (**Oval Latihan**): 2× `1200 mm` straights + 2× `R350 mm` 180° turns, `lineWidth = 20 mm`.
2. `siku` (**Siku 90° dan Persimpangan**): 90° right/left turns + `+` cross.
3. `warna` (**Zona Warna Terbalik**): Loop with `invertZones: [{x:300, y:1050, w:700, h:300}]`.
4. `technowars` (**Technowars 5**): `2400 × 4051 mm` raster SDF.
5. `krpy` (**KRPY Track 1**): `6600 × 3400 mm` (Red & Blue starts).
6. `elsemar` (**ELSEMAR 2026**): `4500 × 3500 mm` (Red & Blue starts).
7. `pe2026` (**PE 2026 LF Advance**): `4500 × 8000 mm` (Red & Blue starts).
8. `brc2026` (**Arena LF BRC 2026**): `6000 × 3000 mm` (Red & Blue starts).
9. `kompetisi1` (**Kompetisi Ichibot #1**): `5800 × 3420 mm`, `lineWidth = 20 mm`, `112` competition waypoint groups (Red & Blue starts).

### 4 Built-In Example Plans (Patterns)
- **P1 (`oval`)**: `stopAtIndex=0, stopAtStep=1, defaultSpeed=90`, `i:000: sensor_type=0, TA=5510, mode_TA=3` (pure line-following by right-encoder mm).
- **P3 (`siku`)**: `stopAtIndex=4, stopAtStep=0, defaultSpeed=60`, uses `PID_NUM=1`, `sensor_value=8` (90° right), `15` (90° left), `5` (cross), `mode_delay=1/2/3` & `mode_TA=2/3`.
- **P4 (`warna`)**: `stopAtIndex=2, stopAtStep=1, defaultSpeed=80`, `i:001` switches `line_color=1` (WHITE) with `SB=100, TB=99999, SB_MODE=1 (FREE)` until `i:002` (`sensor_value=4`) switches back to `line_color=0` (BLACK).
- **P6 (`krpy`)**: `stopAtIndex=7, stopAtStep=1, defaultSpeed=60`, chains 4 `sensor_type=0` (`Motion`) segments in a row (`i:001..i:004`) by timing `TA` precisely before turns, then switches to `PID_NUM=4` for high-speed finale.

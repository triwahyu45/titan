# Physics Engine, Track Sampling & Simulator Model

Derived from `S`, `T`, `TrackSampler`, `TRACKS`, and `CompetitionTracker` in `robotics.ichibot.id/simulator`.

### 4. Exact Mathematical Invariance Transfer (`RPM Multiply` Scaling)
- Scaling `rpmMultiplierX10` from `M0` (e.g. `11` = `1.1x`) to `M1` (`r = M0 / M1`) preserves closed-loop trajectory physics when **both** Plan and LQR are scaled:
  - `SA *= r`, `SB *= r`, `L *= r`, `R *= r`, `defaultSpeed *= r`
  - `LQR.Q1 *= r^2`, `LQR.Q2 *= r^2`, `LQR.Max *= r`, `LQR.Min *= r` (keeping `Q0=0, Tf=0, DT=2`).
- **Offline Local Simulator Mirror (`LF Ichibot/local_simulator/`)**: Pure Node HTTP server (`server.mjs`) serving mirrored Next.js static chunks + `tracks/` + `competition.json` with local `/api/local/bootstrap` injecting `ichibot-simulator-v1` into `localStorage` and stubbing `/api/simulator/competition/submit` locally.

---

## 1. 2D Rigid-Body Vehicle Dynamics (`dt = 0.25 ms` / `4 kHz`)

### Default Physical Parameters (`defaultPhysics`)
| Parameter | Value | Parameter | Value |
|### 4. Exact Mathematical Invariance Transfer (`RPM Multiply` Scaling)
- Scaling `rpmMultiplierX10` from `M0` (e.g. `11` = `1.1x`) to `M1` (`r = M0 / M1`) preserves closed-loop trajectory physics when **both** Plan and LQR are scaled:
  - `SA *= r`, `SB *= r`, `L *= r`, `R *= r`, `defaultSpeed *= r`
  - `LQR.Q1 *= r^2`, `LQR.Q2 *= r^2`, `LQR.Max *= r`, `LQR.Min *= r` (keeping `Q0=0, Tf=0, DT=2`).
- **Offline Local Simulator Mirror (`LF Ichibot/local_simulator/`)**: Pure Node HTTP server (`server.mjs`) serving mirrored Next.js static chunks + `tracks/` + `competition.json` with local `/api/local/bootstrap` injecting `ichibot-simulator-v1` into `localStorage` and stubbing `/api/simulator/competition/submit` locally.

---|### 4. Exact Mathematical Invariance Transfer (`RPM Multiply` Scaling)
- Scaling `rpmMultiplierX10` from `M0` (e.g. `11` = `1.1x`) to `M1` (`r = M0 / M1`) preserves closed-loop trajectory physics when **both** Plan and LQR are scaled:
  - `SA *= r`, `SB *= r`, `L *= r`, `R *= r`, `defaultSpeed *= r`
  - `LQR.Q1 *= r^2`, `LQR.Q2 *= r^2`, `LQR.Max *= r`, `LQR.Min *= r` (keeping `Q0=0, Tf=0, DT=2`).
- **Offline Local Simulator Mirror (`LF Ichibot/local_simulator/`)**: Pure Node HTTP server (`server.mjs`) serving mirrored Next.js static chunks + `tracks/` + `competition.json` with local `/api/local/bootstrap` injecting `ichibot-simulator-v1` into `localStorage` and stubbing `/api/simulator/competition/submit` locally.

---|### 4. Exact Mathematical Invariance Transfer (`RPM Multiply` Scaling)
- Scaling `rpmMultiplierX10` from `M0` (e.g. `11` = `1.1x`) to `M1` (`r = M0 / M1`) preserves closed-loop trajectory physics when **both** Plan and LQR are scaled:
  - `SA *= r`, `SB *= r`, `L *= r`, `R *= r`, `defaultSpeed *= r`
  - `LQR.Q1 *= r^2`, `LQR.Q2 *= r^2`, `LQR.Max *= r`, `LQR.Min *= r` (keeping `Q0=0, Tf=0, DT=2`).
- **Offline Local Simulator Mirror (`LF Ichibot/local_simulator/`)**: Pure Node HTTP server (`server.mjs`) serving mirrored Next.js static chunks + `tracks/` + `competition.json` with local `/api/local/bootstrap` injecting `ichibot-simulator-v1` into `localStorage` and stubbing `/api/simulator/competition/submit` locally.

---|### 4. Exact Mathematical Invariance Transfer (`RPM Multiply` Scaling)
- Scaling `rpmMultiplierX10` from `M0` (e.g. `11` = `1.1x`) to `M1` (`r = M0 / M1`) preserves closed-loop trajectory physics when **both** Plan and LQR are scaled:
  - `SA *= r`, `SB *= r`, `L *= r`, `R *= r`, `defaultSpeed *= r`
  - `LQR.Q1 *= r^2`, `LQR.Q2 *= r^2`, `LQR.Max *= r`, `LQR.Min *= r` (keeping `Q0=0, Tf=0, DT=2`).
- **Offline Local Simulator Mirror (`LF Ichibot/local_simulator/`)**: Pure Node HTTP server (`server.mjs`) serving mirrored Next.js static chunks + `tracks/` + `competition.json` with local `/api/local/bootstrap` injecting `ichibot-simulator-v1` into `localStorage` and stubbing `/api/simulator/competition/submit` locally.

---|
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

### 4. Exact Mathematical Invariance Transfer (`RPM Multiply` Scaling)
- Scaling `rpmMultiplierX10` from `M0` (e.g. `11` = `1.1x`) to `M1` (`r = M0 / M1`) preserves closed-loop trajectory physics when **both** Plan and LQR are scaled:
  - `SA *= r`, `SB *= r`, `L *= r`, `R *= r`, `defaultSpeed *= r`
  - `LQR.Q1 *= r^2`, `LQR.Q2 *= r^2`, `LQR.Max *= r`, `LQR.Min *= r` (keeping `Q0=0, Tf=0, DT=2`).
- **Offline Local Simulator Mirror (`LF Ichibot/local_simulator/`)**: Pure Node HTTP server (`server.mjs`) serving mirrored Next.js static chunks + `tracks/` + `competition.json` with local `/api/local/bootstrap` injecting `ichibot-simulator-v1` into `localStorage` and stubbing `/api/simulator/competition/submit` locally.

---

## 2. Stochastic Run Variation & Optical Sensor Model

- **Per-Run Variation (`runVariation = 1`)**: Mulberry32 PRNG seeds per-run asymmetry: motor gain `±1%`, wheel radius `±0.2%`, tire grip `±1.5%`, start pose jitter `±0.75 mm` and `±0.3°`.
- **Trail Slip Color Thresholds (`38e1f401dc10e491.js:1145-1146`)**: Simulator renders trail segments using `slip = min(1, max(gripUseL, gripUseR))`:
  - `slip > 0.95`: `#ef4444` (**Red** — saturated/skidding)
  - `0.75 < slip <= 0.95`: `#f59e0b` (**Amber**)
  - `slip <= 0.75`: `#10b981` (**Green** — clean grip)
  - **Low-Slip Tuning Rule**: High-PWM turns (`150/10`) + aggressive `PID_NUM=3` drive ~47% red slip; dropping corner PWM to `70/10` (or `-14/70`) and using `PID_NUM=1` (straights) / `PID_NUM=2` (oscillators/resistors) slashes red slip to `~13.5%` (`~74%` green) with 100% finish reliability.
- **Grip Noise**: Ornstein-Uhlenbeck process (`sigma = 0.02`, `tau = 50 ms`) active only during `ACT` phases (`gripNoiseActive = true`).
- **Track Sampling (`TrackSampler`)**:
  - **Vector Tracks**: 40 mm spatial hash grid of line segments; exact point-to-segment distance `d`.
  - **Raster Tracks**: 2 mm/px grayscale SDF PNG; bilinear interpolation `(pixel - 128) / 4` gives signed edge distance in mm.
- **Reflectance & 12-bit ADC**:
  - Optical blur via Gaussian `erf`: `reflectance = 0.92 - 0.86 * (erf((halfWidth + 0.5 - d)/2) - erf((-halfWidth - 0.5 - d)/2))` (inverted inside `invertZones`).
  - Normalized line darkness `n = (0.92 - reflectance) / 0.86`.
  - Per-channel 12-bit ADC: `adc[ch] = clamp(round(S[ch] + (R[ch] - S[ch]) * n + uniform(-1,1) * sensorNoise * 3000), 0, 4095)`, where `S[ch] ∈ [300, 480]` (white) and `R[ch] ∈ [2700, 3400]` (black).

### 4. Exact Mathematical Invariance Transfer (`RPM Multiply` Scaling)
- Scaling `rpmMultiplierX10` from `M0` (e.g. `11` = `1.1x`) to `M1` (`r = M0 / M1`) preserves closed-loop trajectory physics when **both** Plan and LQR are scaled:
  - `SA *= r`, `SB *= r`, `L *= r`, `R *= r`, `defaultSpeed *= r`
  - `LQR.Q1 *= r^2`, `LQR.Q2 *= r^2`, `LQR.Max *= r`, `LQR.Min *= r` (keeping `Q0=0, Tf=0, DT=2`).
- **Offline Local Simulator Mirror (`LF Ichibot/local_simulator/`)**: Pure Node HTTP server (`server.mjs`) serving mirrored Next.js static chunks + `tracks/` + `competition.json` with local `/api/local/bootstrap` injecting `ichibot-simulator-v1` into `localStorage` and stubbing `/api/simulator/competition/submit` locally.

---

## 3. Built-In Tracks & Reference Plans

### 10 Built-In Tracks (`robotics.ichibot.id/simulator`)
1. `oval` (**Oval Latihan**): `2× 1200 mm` straights + `2× R350 mm` 180° turns, `lineWidth = 20 mm`.
2. `siku` (**Siku 90° dan Persimpangan**): 90° right/left turns + `+` cross, `lineWidth = 20 mm`.
3. `warna` (**Zona Warna Terbalik**): Loop with `invertZones: [{x:300, y:1050, w:700, h:300}]`, `lineWidth = 20 mm`.
4. `technowars` (**Technowars 5**): `2400 × 4051 mm` raster SDF (`lineWidth = 18 mm`). Features bottom `2× T` + S-curves + Diode + Square Loop + White-box Invert + Dashed line + Triangle/Square/AND-gate + Coils + Resistor + Butterfly + Top Circle (`sv = 15/8`, `PID = 2..4`).
5. `krpy` (**KRPY Track 1**): `6600 × 3400 mm` (`lineWidth = 18 mm`, Red/Blue mirror `vertikal`). Zigzags -> AND-gate + Helmet -> Capacitor + Coils + Dashed -> Op-Amp + Sine-Wave -> Invert Resistor + Hex -> `2026` -> Checkerboard -> `INSPIRE` (`K-R-P-Y` center).
6. `krpy2` (**KRPY Track 2**): `6600 × 4400 mm` (`lineWidth = 20 mm`, Red/Blue mirror `vertikal`). Extended `krpy` with lower loop: Ground/Vertical Resistor -> `U-W-U` turns -> Dashed-Circle -> Invert White-Dashed-Line -> Central Black Box Finish.
7. `elsemar` (**ELSEMAR 2026**): `4500 × 3500 mm` (`lineWidth = 18 mm`, Red/Blue mirror `putar` 180°). Start -> Resistor -> Butterfly -> Star -> `ESC#14` -> Hex -> S-curve -> Diamond -> `HMTE` + Hourglass + Resistor -> `UNS` + Double-Diamonds -> Checkerboard + Spiral + Central Lightbulb Finish.
8. `pe2026` (**PE 2026 LF Advance**): `4500 × 8000 mm` (`lineWidth = 18 mm`, Red/Blue mirror `horizontal`). Battery + Resistor -> `HMTE` -> Diode + Loops + Capacitor -> Logic Gates -> Honeycomb Invert -> Transistor -> `PE2026` + Variable Resistor -> 3-Circle Venn -> White-Zigzag Invert -> Inductor + Bull-Head -> Finish.
9. `brc2026` (**Arena LF BRC 2026**): `6000 × 3000 mm` (`lineWidth = 18 mm`, Red/Blue mirror `vertikal`). Diamond -> `2026` -> Invert Resistor -> Sine-Wave -> Circle-X -> Hexagon + Dashed Y-fork + Coils -> Invert Double-Butterfly + Dashed-Circle -> Loop-pin -> `BRC` + Checker-arc + `BRC` Crown + Finish.
10. `kompetisi1` (**Kompetisi Ichibot #1**): `5800 × 3420 mm` (`lineWidth = 20 mm`, Red/Blue mirror `vertikal`), `102` waypoint groups (`111` waypoints). Combines 4 Zigzags + Diamond + Black Box + 3-Circle Venn + Hourglass + `2026` + `ICHIBOT` + Top Sine-Wave.

### Competition Anti-Shortcut Rule (`CompetitionTracker` + `SENSOR_BOARD`)
- Live simulator checks waypoints (`g = 1..total`, `r = 15 mm`) against the **47-vertex concave `SENSOR_BOARD` polygon** (not full-body `ROBOT_CORNERS`, which is only used for the final Finish line cross).
- Because `SENSOR_BOARD` is concave (T-shaped front sensor wing), `pointHitsPoly` **must** use ray-casting + segment-distance (`(a.y > py) !== (b.y > py) && ...`) rather than a convex-only winding check.
- Hitting a later waypoint group `g > reached + 1` before `reached` triggers `shortcutAt = e + 1` (`Pelanggaran shortcut`).

### 4 Built-In Example Plans (Patterns)
- **P1 (`oval`)**: `stopAtIndex=0, stopAtStep=1, defaultSpeed=90`, `i:000: sensor_type=0, TA=5510, mode_TA=3` (pure line-following by right-encoder mm).
- **P3 (`siku`)**: `stopAtIndex=4, stopAtStep=0, defaultSpeed=60`, uses `PID_NUM=1`, `sensor_value=8` (90° right), `15` (90° left), `5` (cross), `mode_delay=1/2/3` & `mode_TA=2/3`.
- **P4 (`warna`)**: `stopAtIndex=2, stopAtStep=1, defaultSpeed=80`, `i:001` switches `line_color=1` (WHITE) with `SB=100, TB=99999, SB_MODE=1 (FREE)` until `i:002` (`sensor_value=4`) switches back to `line_color=0` (BLACK).
- **P6 (`krpy`)**: `stopAtIndex=7, stopAtStep=1, defaultSpeed=60`, chains 4 `sensor_type=0` (`Motion`) segments in a row (`i:001..i:004`) by timing `TA` precisely before turns, then switches to `PID_NUM=4` for high-speed finale.

---

## Verified Simulator Import & Play-to-Play Variation Rules (`f3455ebd8e7291ed.js`)

1. **`Import Excel` (`ty()`) Ignores `Setting Robot`**:
   - Importing an `.xlsx` backup into `robotics.ichibot.id/simulator` **only** applies `s.plans` (`Plan 1..9`, `settings`, `indexes`, `cpActions`) and **ignores `s.robotSettings`** (`RPM Multiply`, `LQR 1..5`, `PID 1..5`, `Throttle`, `Sensor Sensitivity`).
   - Any non-default `RPM Multiply` or `LQR` tuning **must** be entered manually in the simulator's OLED menu before clicking Play (or pre-loaded via local state).
2. **Exact Play-to-Play Randomization (`runVariation = 1`)**:
   - Every Play click draws fresh PRNG values for 6 exact physical perturbations:
     - `gain`: `[1 ± 1.0%, 1 ± 1.0%]` (left/right motor torque asymmetry)
     - `wheel`: `[1 ± 0.2%, 1 ± 0.2%]` (left/right wheel radius tolerance)
     - `grip`: `[1 ± 1.5%, 1 ± 1.5%]` (left/right tire friction)
     - `gripNoise`: OU process `σ = 2.0%` (clipped `±6%`), active **only** while `gripNoiseActive == true` during index `action()`
     - Start pose jitter: `±0.75 mm` X/Y and `±0.3°` heading
     - `sensorNoise`: `0.04` (`±120` ADC counts) per sensor read
   - **Fast Verification Shortcut**: Instead of running 100+ blind Monte Carlo seeds on every candidate, test the **16 deterministic extreme corners** of these 6 factors first (`CORNER_16`), then confirm on `UNSEEN_200` + `openpyxl` Excel round-trip.

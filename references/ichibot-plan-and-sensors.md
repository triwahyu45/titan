# Sensor Modes (0–36), Plan/Index Schema, OLED UI & Excel Backup

---

## 1. Complete 37-Mode Sensor Trigger Table (`SENSOR_MODE[0..36]`)

Bit `13` (leftmost `■`) = `S14` (Far Left); Bit `0` (rightmost `■`) = `S1` (Far Right).
`LSM` modes: `0 = OR` (any masked bit `1`), `1 = XOR` (both left & right groups in `detectRaw` must have ≥1 bit `1`), `2 = EQUAL` (exact 14-bit match).

> [!WARNING]
> **Two Firmware Quirks Verified in `detectRaw`:**
> - **Mode `0`**: Visual preview shows `0x0618` (`___■■____■■___` EQUAL), but `detectRaw` checks `sensorBits === 0x3333` (`■■__■■__■■__■■` EQUAL).
> - **Mode `13`**: Visual preview shows `0x2000` (`■_____________` OR), but `detectRaw` checks `(sensorBits & 0x3000) != 0` (`■■____________` OR).

| # | Visual (`S14..S1`) | Hex | Dec | `LSM` | Exact `detectRaw` Condition | Mirror | Typical Track Use-Case |
|---:|:---|---:|---:|:---|:---|---:|:---|
| `0` | `___■■____■■___` | `0x0618` | `1560` | `EQUAL` | `e === 13107` (`0x3333`) | `0` | Exact double-line pattern (quirk) |
| `1` | `■____________■` | `0x2001` | `8193` | `XOR` | `(e&0x2000) && (e&0x0001)` | `1` | Wide outer-edge cross / T-junction |
| `2` | `■■__________■■` | `0x3003` | `12291` | `XOR` | `(e&0x3000) && (e&0x0003)` | `2` | Cross / T-junction (outer 2+2) |
| `3` | `■■■________■■■` | `0x3807` | `14343` | `XOR` | `(e&0x3800) && (e&0x0007)` | `3` | Cross / T-junction (outer 3+3) |
| `4` | `■■■■______■■■■` | `0x3c0f` | `15375` | `XOR` | `(e&0x3c00) && (e&0x000f)` | `4` | Cross / T-junction (outer 4+4) |
| `5` | `■■■■■____■■■■■` | `0x3e1f` | `15903` | `XOR` | `(e&0x3e00) && (e&0x001f)` | `5` | Wide cross / T-junction (5+5) |
| `6` | `_____________■` | `0x0001` | `1` | `OR` | `(e&0x0001) != 0` | `13` | Rightmost `S1` hits right branch |
| `7` | `____________■■` | `0x0003` | `3` | `OR` | `(e&0x0003) != 0` | `14` | Right `S1..S2` hit right branch |
| `8` | `___________■■■` | `0x0007` | `7` | `OR` | `(e&0x0007) != 0` | `15` | Right `S1..S3` hit 90° right |
| `9` | `_________■■■■■` | `0x001f` | `31` | `OR` | `(e&0x001f) != 0` | `16` | Right `S1..S5` hit wide right |
| `10` | `_____■■■■___■■` | `0x01e3` | `483` | `XOR` | `(e&0x0003) && (e&0x01e0)` | `17` | Center + Right Y/T-branch |
| `11` | `____■■■■___■■_` | `0x03c6` | `966` | `XOR` | `(e&0x0006) && (e&0x03c0)` | `18` | Center + Mid-Right Y-fork |
| `12` | `____■■■■__■■__` | `0x03cc` | `972` | `XOR` | `(e&0x000c) && (e&0x03c0)` | `19` | Center + Inner-Right Y-fork |
| `13` | `■_____________` | `0x2000` | `8192` | `OR` | `(e&0x3000) != 0` (`S13\|S14`) | `6` | Leftmost `S13\|S14` hits left branch |
| `14` | `■■____________` | `0x3000` | `12288` | `OR` | `(e&0x3000) != 0` | `7` | Left `S13..S14` hit left branch |
| `15` | `■■■___________` | `0x3800` | `14336` | `OR` | `(e&0x3800) != 0` | `8` | Left `S12..S14` hit 90° left |
| `16` | `■■■■■_________` | `0x3e00` | `15872` | `OR` | `(e&0x3e00) != 0` | `9` | Left `S10..S14` hit wide left |
| `17` | `■■___■■■■_____` | `0x31e0` | `12768` | `XOR` | `(e&0x3000) && (e&0x01e0)` | `10` | Center + Left Y/T-branch |
| `18` | `_■■___■■■■____` | `0x18f0` | `6384` | `XOR` | `(e&0x1800) && (e&0x00f0)` | `11` | Center + Mid-Left Y-fork |
| `19` | `__■■__■■■■____` | `0x0cf0` | `3312` | `XOR` | `(e&0x0c00) && (e&0x00f0)` | `12` | Center + Inner-Left Y-fork |
| `20` | `■■■■■■■■■■■■■■` | `0x3fff` | `16383` | `EQUAL` | `e === 16383` | `20` | All 14 sensors on line (finish/bar) |
| `21` | `______________` | `0x0000` | `0` | `EQUAL` | `e === 0` | `21` | All 14 sensors blank (gap / invert) |
| `22` | `■■■■■■■■■■■■■■` | `0x3fff` | `16383` | `OR` | `(e&0x3fff) != 0` | `22` | Any sensor sees line (re-acquire) |
| `23..36` | Single bit `0..13` | `1<<k` | `2^k` | `OR` | `(e & (1<<(val-23))) != 0` | `36..23` | Individual sensor `S1`..`S14` |

---

## 2. Complete 21-Field Index Schema

Every Plan (`1..9`) has `151` Indexes (`indexNo = 0..150`) + `10` Checkpoint Actions (`cp = 0..9`).

| Field | Label | Range / Enum | Default | Notes |
|---|---|---|---|---|
| `sensor_type` | Trigger Type | `0..5` | `1` (`0` at `i:000` & CP) | `0`: Motion (no sensor), `1`: If Line Sensor, `2`: A0<, `3`: A0>, `4`: A15<, `5`: A15> |
| `sensor_value` | Trigger Value | `0..36` (type 1) / `0..255` (analog) | `0` | Index into `SENSOR_MODE[0..36]` or 8-bit analog threshold |
| `ACT` | Action | `0..5` | `0` | `0`: FORWARD, `1`: Turn LEFT, `2`: Turn RIGHT, `3`: BACKWARD, `4`: PAUSE, `5`: JUMP PLAN |
| `B` | Brake | `0..99` ms | `0` | Reverse pulse at `-defaultSpeed` before `ACT` |
| `D` | Delay | `0..9999` (`-500..500` if `mode_delay==1`) | `100` | Duration/angle/distance for `(L, R)` |
| `mode_delay` | Delay Mode | `0..3` | `0` | `0`: mS, `1`: degree (`odoThetaDeg`), `2`: encL (mm), `3`: encR (mm) |
| `L`, `R` | Left / Right | `-255..255` (`0..8` / `0..150` if `ACT==5`) | `50, 50` | Motor speeds/PWM; when `ACT==5`: `L = targetPlan (0..8)`, `R = targetIndex (0..150)` |
| `fan_status` | Fan | `0..9` | `0` | `0`: OFF, `1..9`: ON for `1..9` seconds (stationary) |
| `gripper_status` | Gripper | `0..6` | `0` | `0`: STANDBY, `1..3`: PICK 1..3, `4..6`: DROP 1..3 (`700 ms` if changed) |
| `SA`, `SB` | Speed A / B | `0..255` | `90`, `10` | Line-following base speeds for Phase A & Phase B |
| `TA`, `TB` | Timer A / B | `0..99999` (`-99999..99999` if mode `1`) | `0`, `0` | Phase A/B limit (`mS` mode runs `2 * TA` / `2 * TB` ms!) |
| `mode_TA`, `mode_TB` | Timer Mode | `0..3` | `0`, `0` | `0`: mS (`×2`), `1`: mm (UI says `cm`), `2`: encL (mm), `3`: encR (mm) |
| `SB_MODE` | SB Mode | `0..1` | `0` | `0`: LOCK (ignore next trigger during `TB`), `1`: FREE (exit early on next trigger) |
| `line_color` | Line Color | `0..1` | `0` | `0`: BLACK line, `1`: WHITE line |
| `RUN_MODE` | Run Mode | `0..2` | `0` | `0`: `FL.CENTER`, `1`: `FL.LEFT`, `2`: `FL.RIGHT` |
| `PID_NUM` | Control Slot | `0..4` | `2` | Selects `LQR/PID` slot `1..5` |
| `NUM_CAL` | Calib Slot | `0..4` | `0` | Selects calibration slot `CAL0..CAL4` |

**Action Defaults (`DEFAULT_ACTION`)**: `0: (50,50,90)`, `1: (-100,150,90)`, `2: (150,-100,90)`, `3: (-50,-50,90)`, `4: (0,0,100)`.
**Checksum Payload**: Concatenate `[index, sensor_type, sensor_value, ..., NUM_CAL].join(",")` for `index = 0..150` with no separator between rows.

---

## 3. Excel Backup Format (`ichibot-robot-backup-v1`)

Filename: `<robotCode>_Backup_<YYYY-MM-DD>.xlsx`. Five sheet types:
1. **`Info`**: `field, value` (`format = "ichibot-robot-backup-v1"`, `robotCode`, `exportedAt`, `settingSchemaVersion`, `settingFwVersion`, `settingFwVersionCode`).
2. **`Pengaturan Plan`**: `planNo (1..9)`, `planName`, `stopAtIndex`, `stopAtStep (0..2)`, `selectedCpNumber (0..9)`, `inverseStatus (0..1)`, `defaultSpeed`, `cpIndex` (JSON array of 10 ints), `version`, `schemaVersion`, `fwVersion`.
3. **`Plan 1` .. `Plan 9`**: Header `["indexNo", ...21 Index fields]` with 151 rows (`0..150`).
4. **`Checkpoint`**: Header `["planNo", "cp", "runAfter", ...21 Index fields]` (`cp = 1..9`, `runAfter = cpIndex[cp]`).
5. **`Setting Robot`**: Flattened dotted/indexed keys (`part ∈ {"core", "tuning", "sensor"}`, `key`, `type`, `value`).

---

## 4. 128×128 OLED & 6-Button Menu Tree

**Buttons**: `UPL` (`↑`), `DWNL` (`↓`), `OKL` (`Enter`, hold `900 ms` in Plan Editor -> Advance Index), `UPR` (`+`/`→`), `DWNR` (`-`/`←`), `OKR` (`Esc`/`Backspace`; hold & release on Home -> **Ready -> Running**).

- **Home Screen**: Plan selector (`P1..P9`), `Stop i:XXX [ACTION|SPEED A|SPEED B]`, `Checkpoint [START|1..9]`, `to MAIN MENU`, `V:XXX` (`defaultSpeed`), plus live 14-bar sensor monitor, battery voltage, WiFi/BT/Inverse icons.
- **Main Menu**:
  1. `SET PLAN X`: **Normal** (`T, A..I` 2-page view) or **Expert** (compact 16-field grid). Hold `OKL` -> **Advance Menu** (`Copy`, `Paste`, `Delete`, `Insert Before`, `Mirror Index`).
  2. `CALIBRATE`: `Auto Rotate` -> Step 1 (center on line) -> Spin `720°`/`4s` -> Step 2 (`Sensitivity %`) -> Step 3 (`CAL0..4`) -> `Check Sensor`.
  3. `HARDWARE`: `Line Sensor` (`Calibration`, `Check Sensor` with live bars + `Geleng`/rotate), `Motor DC` (`Speed`, `Delay`, `Mode`, `Test Now`, `RPM Control On/Off`, `RPM Multiply`, `Throttle`), `Encoder` (`Enc Val`, `Odo Dist`, `Odo Angle`, `PPR`, `Wheelbase`, `Wheel Diameter`), `Buzzer`, `Run Control` (`LQR`/`PID`), `LQR/PID Setting`.
  4. `ADVANCE`: `Checkpoint`, `Mirror Plan` (`Range 1..5` + `Mirror Now`), `Plan Display` (`Normal`/`Expert`), `Reset Setting`, `Reset Data Plan`, `Default Speed`, `Lost Line`, `Speed Calibrate`, `Robot Name`, `Plan Name`.

### Verified 6-Button Entry Shortcuts (`c42b584801af4d19.js`) & Built-In Plans
- **Wrap-Around (`E(..., wrap=true)`)**:
  - `stopAtIndex`: `150` + `UPR` once -> `0`.
  - `TB` (and `TA`): `0` + `DWNR` once -> `99999`.
  - `sensor_value`: `0` + `DWNR` wraps `0 -> 36 -> ...` (faster for modes `20..36`, e.g. `21` is 16 `DWNR` presses).
- **Hold Acceleration (`>1000ms` pointerdown, `90ms` repeat)**: Steps by `10` on `D`, `SA`, `TA`, `SB`, `TB` and by `5` on `L`, `R`, `stopAtIndex`, `selIndex`.
- **Action Preset Rule**: Changing `ACT` overwrites `L`, `R`, and `D` with `DEFAULT_ACTION[ACT]` — always set `ACT` **before** tuning `L`, `R`, or `D`.
- **Built-In Training Plans (`38e1f401dc10e491.js`)**: Default firmware pre-populates **Plan 1** (`Oval ikut garis`), **Plan 3** (`Siku & persimpangan`), **Plan 4** (`Ganti warna garis`), and **Plan 6** (`KRPY`). **Plan 2** is empty by default — use Plan 2 for custom competition tracks like **Technowars 5**.
- **Corner & `"Cari garis"` Physics**: Because sensors sit `142.8 mm` ahead of the rear axle, pure spin turns (`L=-100, R=150`) miss perpendicular lines ahead; arc turns (`L=10, R=70` / `L=70, R=10` with `D=50..70, mode_delay=1`) advance into the corner and trigger `"Cari garis"` (`0.8 * L/R` masked by `8176` or `1022`) reliably with minimal tire slip.
- **Zero-Loop Obstacle Rules**:
  - **Symmetric Hourglass / Circle**: Use `RUN_MODE=0 (FL.CENTER)` with `PID_NUM=1..2` (using `FL.LEFT` on an hourglass triggers 1–2 spinning loops on the left wall).
  - **Final Circle + T-Bar Finish**: Use `RUN_MODE=1 (FL.LEFT)` with gentle `SA=45, PID_NUM=1` to exit cleanly without 50/50 branch coin-flips, then park with `ACT=0` gentle arc (`L=45, R=65, D=135, mode_delay=2`) onto the finish bar.

---

## 5. Setting Robot Tuning Playbook (LQR, RPM Multiplier, Throttle & Multi-Seed Stability)

Verified via 30-to-50-seed Monte Carlo simulations (`runVariation = 1`, `sensorNoise = 0.04`) on Ichibot Ultimate 5 Max firmware:

1. **`PID_NUM` is 0-Indexed (`0..4`, and `5` wraps to `0`)**:
   - `followLine()` indexes `lqr[s]` directly (`if (s >= 5) s = 0`).
   - `0`: softest (`Q1=122, Q2=740`), `1`: medium (`276/3135`), `2`: stiff (`490/7150`), `3`: very stiff (`766/12785` — S-curve / coils), `4`: ultra-stiff (`1322/20020` — sharp Resistor corners).
2. **`RPM Multiplier` (`rpmMultiplierX10`) Couples Speed AND Steering Gain**:
   - `setMotor(r, n)` multiplies **both** base speed `SA/SB` **and** LQR steering output `out` by `rpmMultiplierX10 / 10`.
   - Running a `1.1`-tuned plan on a simulator set to `1.5` increases steering aggressiveness by `1.36×` and causes `0/30` crashes on zigzags; even after re-tuning with default LQR, `1.5` drops 50-seed accuracy from **68% (`34/50`, 27.47s best)** down to **20% (`10/50`, 27.76s best)**.
   - **Rule:** Keep `RPM Control = ON`, **`RPM Multiplier = 1.1`** (default), and **`Throttle = 2`** (default; `Throttle = 0` drops stability from `19/30` to `14/30`).
3. **LQR `Tf` (Derivative Filter) & `DT` Guardrails**:
   - **Keep `Tf = 0 ms`** — any `Tf >= 4 ms` introduces derivative phase lag and causes `0/30` failures on sharp zigzags.
   - **Keep `DT = 2 ms`** — `DT = 1 ms` with default `Q1/Q2` amplifies `sensorNoise = 0.04` jitter (`21/30 -> 3/30`).
4. **Anti-Overshoot Corner Braking & Fast Pivot Turns**:
   - End `TA` (`mode_TA=1`, mm) **150–250 mm before 90° corners** and brake via `SB = 40..50, TB = 99999, SB_MODE = 1 (FREE)` to eliminate wide corner overshoot ("loncat") before the turn trigger.
   - Fast sweep-pivot turns (`L = -20, R = 145, D = 62, mode_delay = 1`) cut ~0.1s per open corner; keep gentler arcs (`L = 10, R = 105, D = 64`) on sensitive corners (e.g. Capacitor entry).
5. **High-Reliability Trigger Patterns on Complex Tracks**:
   - **White-line-in-Black-Box entry:** Use `sv = 3` (`■■■________■■■` XOR) instead of `sv = 20` (all-black), because the white centerline prevents all 14 sensors from reading black simultaneously.
   - **Mid-track Encoder Re-Sync:** Insert a `sv = 15` re-sync on crossbars (e.g., `H` in `ICHIBOT`) + short `33 mm` `FL.RIGHT` offset before switching to `FL.LEFT` to zero out accumulated wheel-slip drift.
   - **Closed-Loop Exits (`O` -> `T`):** Force a short `180 mm` `RUN_MODE = 2 (FL.RIGHT)` segment to guarantee exiting right instead of looping around the `O`.
   - **Axle vs. Sensor Visual Offset:** The 14-sensor bar sits `142.8 mm` ahead of the rear axle; rear-axle trajectory plots naturally trace an inner arc at angled junctions (`_/-`) even when the front sensor bar tracks the exact black line (`Shortcut = 0`).

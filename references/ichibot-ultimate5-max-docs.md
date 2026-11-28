# Ichibot Ultimate 5 Max & SE/5R — Official CHIOS Reference

Synthesized from the official `robotics.ichibot.id/docs/ultimate-5-max` documentation (34 pages). Use alongside [`ichibot-chios-firmware.md`](ichibot-chios-firmware.md) (reverse-engineered firmware math), [`ichibot-plan-and-sensors.md`](ichibot-plan-and-sensors.md) (37 sensor modes & 21-field schema), and [`ichibot-sim-physics.md`](ichibot-sim-physics.md) (simulator physics & tracks).

---

## 1. Hardware Specifications & Wiring

- **MCU & Display**: ESP32 dual-core Xtensa LX6 up to `240 MHz`; OLED `1.3"` (or `0.96" 128×64` on Ultimate 5 Max vs `1.5" 128×128` on Ultimate 5 Max SE / 5R) running **CHIOS** (Ichibot Operating System).
- **Sensors & Actuators**: `14×` white SMD LEDs + `14×` SMD photodiodes; `2×` 12V DC motors with internal gearbox (`2500 RPM`) + quadrature encoders; `20 × 20 × 7 cm` chassis; Magcharge round docking connector.
- **Battery (`3×` LiPo / 3S 400mAh)**:
  - `12.6V` (`4.2V/cell`) = Full | `11.1V` (`3.7V/cell`) = Nominal / recharge below this | `10.5V` (`3.5V/cell`) = Stop use immediately.
  - USB power only (`< 5V`) displays `USB` instead of voltage; battery calibration requires battery power.
- **Servo Header (near main LCD)**: `B` = Ground (brown), `R` = `5V` (red), `Y` = Data/Pin (orange).
- **Driver Motor & Step-Up (`XL6009`)**:
  - Motor supply `10–13V` (with optional `XL6009` step-up tuned **outside** robot first to `13V–14V max`).
  - **CRITICAL**: Never let the capacitor on the `XL6009` step-up detach/break — output voltage will spike `> 30V` and fatally damage robot electronics.
  - For `24V` motor input on driver board, change voltage divider resistors to `R1 = 15k`, `R2 = 3k9` (`5V` output at `24V` input).
- **Maintenance**:
  - Wheel/tire removal uses `1.5 mm` hex key (included).
  - Stiff/dragging new wheels usually indicate the wheel is pressed too far inward against the hub/gearbox.
  - Dead robot on battery: check battery cell voltage, fuse, and white power switch.

---

## 2. Calibration & Hardware Tests (`HARDWARE` / `CALIBRATE`)

### Line Sensor Calibration (`MAIN MENU › CALIBRATE` or `HARDWARE › Line Sensor`)
- **Two Modes**:
  1. `Auto Rotate`: Place sensor bar centered perpendicular over straight line -> `Waiting..` (1s) -> spins ~2 rotations (max 4s) -> set `Sensitivity` (`0–99%`, default `18%`) -> save to `CAL:0..4` -> opens `Check Sensor`.
     - Tune rotation speed in `ADVANCE › Speed calibrate` (`10–255`, default `40` on 5 Max/SE, `60` on 5R). Raise if incomplete rotation; lower if wheels slip/drift off line.
  2. `Push Manual`: Place behind line -> push straight back-and-forth by hand over line -> set `Sensitivity` -> save to `CAL:0..4`.
- **Sensitivity Guide**:
  - Higher (`25–35%`): thin, faded lines or uneven lighting.
  - Default (`18%`): balanced for most tracks.
  - Lower (`10%`): noisy/stained floors, board seams, or glare.
- **Check Sensor**: Displays `Cal:0..4`, `Sensitivity%`, `BLACK/WHITE` preview, active sensor `00..13` (`00` = rightmost), `Read`, `High`, `Low`, `Limt`. Short-press `OKL` cycles sensor `00..13`. Changing `Sensitivity` or re-calibrating overwrites any manual per-sensor `Limt` edits.

### Battery Calibration (`HARDWARE › Battery`)
- Measure live battery with multimeter (`DC 20V`) while powered on:
  `new_factor = current_factor × (multimeter_V / screen_V)` (range `0.70–1.30`, website `Sync Plan › Setting` enters `70–130`).
- Calibration is board-specific (no need to repeat when swapping batteries).

### Motor DC & Encoder Tests (`HARDWARE › Motor DC` / `Encoder`)
- **Motor DC**: `Speed Test` (`0–255`), `Delay Test` (`1–20s`), `Test Mode` (`FWD-BWD`, `RGT-LFT`, `LFT-RGT`), `RPM Control` (`On/Off`, **active during index `Action` & Motor Test only — NOT during `followLine`**), `RPM Multiply`, `Throttle` (`Off`, `Slow`, `Medium`, `Fast` — ramps `Speed A` start to prevent wheelspin/skipping on slick tracks).
- **Encoder**: Push straight `200 mm` -> verify `Odo Dist` ≈ `200 mm` (adjust `Wheel Diameter`); rotate `360°` in place -> verify `Odo Angle` ≈ `360` (adjust `Wheelbase`). Invert wrong-sign wheel via `Enc L/R Invert ?`.

---

## 3. CHIOS Menu, Path Planning & Advanced Features

- **Buttons**: `UPL/DWNL` (cursor), `OKL` (select/enter; hold `1s` in `SET PLAN` -> `Advance Index Menu`), `UPR/DWNR` (change value; hold = fast step), `OKR` (back; hold on Home until `Ready to go` & release = Run; press `OKL`/`OKR` to stop).
- **Plan Display (`ADVANCE › Plan Display`)**: `Normal` (labeled `T..I` blocks) or `Expert` (single-screen grid).
- **Advance Index Menu (`SET PLAN` -> hold `OKL` or `...` in Block `I`)**: `Copy Index`, `Paste Index` (works across Plans 1–9), `Delete Index` (shifts later indexes up, confirms `YES/NO`), `Insert Before` (pushes `i:150` out), `Mirror Index` (no confirmation).
- **Copy Plan (`ADVANCE › Copy Plan`)**: Copies `From -> To` across up to 5 `On/Off` index ranges (`000–150`). Copies `STOP ROBOT` & `Checkpoint` (`Run After` within range) + `Default Speed` (always), **not** Plan Name.
- **Checkpoint (`ADVANCE › Checkpoint` or `OKL` on Home `CP1..9`)**:
  - `Run After` (`0–150`): index assumed already completed; run resumes waiting for `Run After + 1`.
  - `With Action`: executes `b -> Action -> Gripper -> Fan -> Speed A -> Speed B` immediately on start (trigger locked to `Motion`).
  - Tip: Set `STOP ROBOT` equal to `Run After` to test the checkpoint action alone; always return Home selector to `START` after testing.
- **Inverse / Mirror Plan (`ADVANCE › Mirror Plan`)**: Flips `Turn LEFT <-> RIGHT`, swaps `L <-> R`, mirrors `SENSOR_MODE`, swaps `encL <-> encR` & `FL.LEFT <-> FL.RIGHT`, negates `degree`/`cm`. Does **not** flip `Motion`/`A0`/`A15` triggers, `mS` delays, `SA/SB`, `line_color`, `fan`, `gripper`, `LQR/PID`, or `CAL`.
- **Jump Plan (`ACT = JUMP to`)**: Jumps mid-run without stopping to `P (1..9), i (000..150)` and **executes target index immediately** without waiting for its trigger; `STOP ROBOT` of target plan applies.
- **Warna Garis (Block `C`)**: Applies from when that index **executes** through `Speed A/B` and while waiting for the **next** index trigger. Place boundary index using `Pola 04` (`4 left XR 4 right`) right at Black<->White floor transitions.
- **Lost Line (`ADVANCE › Lost Line`)**: `OFF` or `10..5000 mS` (default `300 mS`). Counted **only** during `followLine` (`Speed A`, `Speed B`, between indexes) — **never** during index `Action`.
- **Transporter Servo (`HARDWARE › Gripper`, Block `E`)**: Left socket = `Lift` (smaller angle = arm up); Right socket = `Grip` (smaller angle = open). `7` presets: `Stand by`, `Pick 1–3`, `Drop 1–3`.
- **LED Indicator (`HARDWARE › LED Indicator`)**: Separate colors for `Action`, `SA`, `SB`, `Finish` (`+ RAINBOW`) — record slow-motion phone video to visually inspect phase timing on track.

---

## 4. Connectivity, Website Sync, Firmware & Reset

- **WiFi (`HARDWARE › Wifi`)**: **`2.4 GHz` only** (5 GHz unsupported). Keep SSID & password `<= 19 chars` for reliable auto-reconnect. **WiFi and Bluetooth share one radio** — enabling Bluetooth (`PS3` / `BLE`) disables WiFi until `Connect Now` restarts into WiFi mode.
- **Website Sync (`robotics.ichibot.id/sync-plan/ID_ROBOT`)**:
  - `Read Plan from Robot` pulls robot -> website; table edits auto-push while robot is WiFi-connected and idle; `Send Plan to Robot` pushes active tab + Settings.
  - `Export` downloads full `.xlsx` backup (`Plan 1..9` + `Setting Robot`); `Import` loads `.xlsx` to website (still requires `Send Plan to Robot` per tab to flash robot). Note: the web **Simulator's** `Import Excel` only reads `Plan 1..9` and ignores `Setting Robot`.
  - Direct robot-to-robot transfer via `Send` using target ESP32 ID (`MAIN MENU › ABOUT`).
- **Firmware Update (`robotics.ichibot.id/firmware`)**: Flash CHIOS directly via USB in browser (`Connect Robot -> Flash Firmware`).
- **Factory Reset**:
  - `ADVANCE › Reset Setting`: clears all robot settings (calibration, LQR/PID, servo, WiFi, etc.), **keeps** Plans 1–9 & registration.
  - `ADVANCE › Reset Data Plan`: clears 1 plan only, **keeps** all settings.
  - **Hardware Button Reset**: Hold `UPL + UPR` while powering on -> release at `Factory Reset Mode` -> separate `YES/NO` prompts for `ALL PLAN` and `SETTING`.

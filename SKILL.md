---
name: TITAN
description: Robotics, Edge CV & Embedded Systems Ops playbook covering Linux/NUC ROS2 & omni/swerve kinematics, non-blocking STM32F4/ESP32 FreeRTOS firmware & 3-wheel dead-wheel odometry, PID anti-windup/derivative filtering, IMU fusion, Nav2 A*/MCAP, NumPy 500 Hz RK4 sim, YOLOv8/OpenCV FPS optimization, ESP32/STM32 PlatformIO serial & OLED workflows, Ichibot/CHIOS 14-sensor Line Follower firmware (LQR/PID/RPM PI, Plan/Index, 2D tire-slip simulator), H-1 competition recovery, and CAD-to-Simulation (FORGE + MuJoCo).
---

# Titan — Robotics, Edge CV & Embedded Ops

End-to-end field playbook for autonomous robots, edge vision nodes, and microcontroller hardware.

---

## Modules

1. **Edge CV & Linux/NUC Ops** ([`references/edge-cv-and-nuc.md`](references/edge-cv-and-nuc.md)):
   - YOLOv8 / OpenCV FPS optimization (ONNX/TensorRT/OpenVINO, zero-copy capture, threaded inference), Ubuntu headless/GUI scaling, Tailscale P2P, and H-1 competition emergency recovery.
2. **Embedded, Kinematics, Control & CAD-to-Sim** ([`references/embedded-and-cad-sim.md`](references/embedded-and-cad-sim.md), [`references/nonblocking-stm32-esp32-nuc.md`](references/nonblocking-stm32-esp32-nuc.md), [`references/robotics-sim-and-control.md`](references/robotics-sim-and-control.md)):
   - ESP32/STM32 PlatformIO flash & serial verification, 128×64 OLED bitmap/video conversion, omni/swerve kinematics sanity checks, and parametric FreeCAD (`FORGE`) → MuJoCo simulation workflow.
   - Zero-`HAL_Delay()` STM32F4 super-loop, 3-wheel dead-wheel midpoint odometry, ESP32 dual-core `xTaskCreatePinnedToCore`, 3-tier NUC `/dev/ESP32` + `/dev/FTDI1` `*...#` framing, and 5 verified field firmware pitfalls.
   - Line follower boom/rollover physics, PID conditional anti-windup & `alpha=0.25` derivative smoothing, IMU sensor fusion, non-minimum phase balancing, Nav2 Smac-2D A*, MCAP telemetry, and `500 Hz` NumPy RK4 verification.
3. **Ichibot / CHIOS Line Follower Suite**:
   - [`references/ichibot-chios-firmware.md`](references/ichibot-chios-firmware.md): 14-sensor curved arc geometry, auto-rotate calibration, `FL.CENTER/LEFT/RIGHT` error extraction, `lqrSolve` + `PID` + `1.7×` steering mix, inner-loop RPM PI, odometry, `*action`/`*run` state machine, Mirror transform, and critical firmware quirks.
   - [`references/ichibot-plan-and-sensors.md`](references/ichibot-plan-and-sensors.md): All 37 `SENSOR_MODE` bitmasks (`OR/XOR/EQUAL`), 21-field Index schema, Checkpoints, Checksum, `ichibot-robot-backup-v1` Excel `.xlsx` format, and 128×128 OLED 6-button menu tree.
   - [`references/ichibot-sim-physics.md`](references/ichibot-sim-physics.md): 4 kHz 2D rigid-body tire-slip physics, DC motor & battery-sag model, stochastic run variation, SDF reflectance/ADC noise, 9 built-in tracks, and 4 reference example plans.

---

## Mandatory Self-Audit & Revise-Before-Delivery Gate

Before declaring any robotics, CV, or embedded task done:
1. Run the actual build/flash/test command and inspect live serial output (`pio device monitor`) or real inference FPS logs — never assume success.
2. For CAD-to-MuJoCo assets, visually inspect rendered simulation frames to verify collision meshes and joint limits.

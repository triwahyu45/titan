---
name: TITAN
description: Robotics, Edge CV & Embedded Systems Ops playbook organized by language and platform — STM32 C HAL, ESP32/Arduino C++ FreeRTOS & drivers (MPU6050/PCA9685/Bluepad32/WS2812 RMT), Linux/NUC Python & ROS2 (V4L2 60fps, vectorized lane tracking, Tustin PID, YOLOv8), Control/Planning/RK4 Sim (PID anti-windup, IMU fusion, Nav2 A*, MCAP, Curriculum RL, FORGE->MuJoCo), and Ichibot/CHIOS C++ Line Follower.
---

# Titan — Robotics, Edge CV & Embedded Ops

End-to-end field playbook for autonomous robots, edge vision nodes, and microcontroller hardware.

---

## Routing Matrix by Language & Platform

| Language / Platform | Sub-Module(s) | Key Coverage |
| :--- | :--- | :--- |
| **STM32 (`Bahasa C` / HAL)** | [`references/nonblocking-stm32-esp32-nuc.md`](references/nonblocking-stm32-esp32-nuc.md), [`references/embedded-and-cad-sim.md`](references/embedded-and-cad-sim.md) | Zero-`HAL_Delay()` super-loop, 3-wheel dead-wheel midpoint odometry, CAN-bus bitmasks + `CAN_Fail()` watchdog, DMA UART, & field pitfalls |
| **ESP32 / Arduino (`C++` / FreeRTOS / PlatformIO)** | [`references/esp32-drivers-and-competition-ops.md`](references/esp32-drivers-and-competition-ops.md), [`references/nonblocking-stm32-esp32-nuc.md`](references/nonblocking-stm32-esp32-nuc.md), [`references/embedded-and-cad-sim.md`](references/embedded-and-cad-sim.md) | Bare-register MPU6050/clone IMU + NVS offsets, PCA9685 + ZK-5AD brake/deadband, dual-boot NVS WiFi/Bluepad32, WS2812 RMT, dual-core `xTaskCreatePinnedToCore`, & OLED |
| **Linux / NUC (`Python` / `C++` ROS2 & Edge CV)** | [`references/edge-cv-and-nuc.md`](references/edge-cv-and-nuc.md), [`references/nonblocking-stm32-esp32-nuc.md`](references/nonblocking-stm32-esp32-nuc.md) | V4L2 60fps threaded camera, sub-ms Ridge lane tracker, YOLOv8 TensorRT/OpenVINO, OS `atexit`/`SIGINT` E-stop, ROS2 Tustin PID, & `/dev/ESP32` udev `*...#` framing |
| **Control, Planning & Sim (`Python/NumPy` & `FORGE`→MuJoCo)** | [`references/robotics-sim-and-control.md`](references/robotics-sim-and-control.md), [`references/embedded-and-cad-sim.md`](references/embedded-and-cad-sim.md) | Line follower boom/rollover physics, PID anti-windup & `alpha=0.25` D-filter, IMU fusion, Nav2 Smac-2D A*, MCAP, `500 Hz` NumPy RK4, Curriculum RL, & FreeCAD→MuJoCo |
| **Ichibot / CHIOS (`C++` Line Follower)** | [`references/ichibot-chios-firmware.md`](references/ichibot-chios-firmware.md), [`references/ichibot-plan-and-sensors.md`](references/ichibot-plan-and-sensors.md), [`references/ichibot-sim-physics.md`](references/ichibot-sim-physics.md), [`references/ichibot-ultimate5-max-docs.md`](references/ichibot-ultimate5-max-docs.md) | 14-sensor curved arc, LQR/PID/RPM PI, 37 trigger masks, 21-field Plan/Index, `.xlsx` backup, 4 kHz tire-slip simulator, & official Ultimate 5 Max docs |

---

## Mandatory Self-Audit & Revise-Before-Delivery Gate

Before declaring any robotics, CV, or embedded task done:
1. Run the actual build/flash/test command and inspect live serial output (`pio device monitor`) or real inference FPS logs — never assume success.
2. For CAD-to-MuJoCo assets, visually inspect rendered simulation frames to verify collision meshes and joint limits.

# TITAN — Robotics, Edge CV & Embedded Systems Ops

AI Agent Skill for **Antigravity**, **Claude Code**, and **OpenAI Codex**. Field playbook for Linux/NUC ROS2, omni/swerve kinematics, YOLOv8/OpenCV FPS optimization, ESP32/STM32 PlatformIO serial & OLED workflows, and FreeCAD-to-MuJoCo simulation.

---

## Quick Install (Claude Code / Codex / Antigravity)

Clone this skill into your agent's global or project skills directory:

| Agent | Global Install Path | Project-Local Path |
| :--- | :--- | :--- |
| **Antigravity** | Global Antigravity skills dir | `.agents/skills/TITAN` |
| **Claude Code** | `~/.claude/skills/TITAN` | `.claude/skills/TITAN` |
| **OpenAI Codex** | `~/.codex/skills/TITAN` | `.agents/skills/TITAN` |

```bash
# Example (Antigravity / Claude Code / Codex — pick your path):
git clone https://github.com/triwahyu45/TITAN.git ~/.claude/skills/TITAN
```

---

## Requirements & Dependencies (For Agents)

- **Runtime / Tools**: **PlatformIO CLI** (`pio`) for ESP32/STM32; **Python 3.10+** (`ultralytics`, `opencv-python`, `onnxruntime`/`tensorrt`, `mujoco`) for CV & sim; **ROS2** on Ubuntu/NUC
- **MCP Servers**: Optional: `freecad` (if generating CAD meshes from FreeCAD)
- **Related Skills**: Optional pairing with [`forge`](https://github.com/triwahyu45/FORGE) for parametric CAD before MuJoCo export
- **API Keys / Secrets**: None

---

## Included Files

- `SKILL.md` — Field verification rules & Self-Audit Gate
- `references/edge-cv-and-nuc.md` — YOLOv8/OpenCV FPS tuning, Ubuntu/NUC ops, & competition recovery
- `references/embedded-and-cad-sim.md` — PlatformIO flash/serial, 128x64 OLED, kinematics, & FreeCAD->MuJoCo
- `references/embedded-and-cad-sim.md` — PlatformIO, OLED, & CAD-to-Sim
- `references/nonblocking-stm32-esp32-nuc.md` — STM32F4 C HAL, CAN watchdog, ESP32 FreeRTOS, 3-wheel odometry, & 5 pitfalls
- `references/esp32-drivers-and-competition-ops.md` — Bare-register IMU, PCA9685/ZK-5AD, Bluepad32/NVS dual-boot, & WS2812 RMT
- `references/edge-cv-and-nuc.md` — V4L2 60fps camera, Ridge lane tracker, YOLOv8, OS E-stop, & ROS2 Tustin PID
- `references/robotics-sim-and-control.md` — Line follower geometry/rollover, PID windup/noise, IMU fusion, Nav2 A*/MCAP, & 500 Hz NumPy RK4 sim
- `references/ichibot-chios-firmware.md` — CHIOS 14-sensor LF firmware, LQR/PID/RPM PI, odometry, & quirks
- `references/ichibot-plan-and-sensors.md` — 37 sensor trigger masks, 21-field Plan/Index, OLED UI, & `.xlsx` backup
- `references/ichibot-sim-physics.md` — 4 kHz 2D tire-slip physics, SDF sensor model, & built-in tracks/plans

## When to Trigger / Search Keywords

`titan`, `ros2`, `yolov8`, `opencv fps`, `esp32`, `stm32`, `freertos`, `odometry`, `pid`, `imu`, `a*`, `mcap`, `rk4`, `bluepad32`, `pca9685`, `ws2812 rmt`, `can bus`, `v4l2`, `lane tracker`, `tustin pid`, `platformio`, `omni kinematics`, `swerve`, `mujoco`, `nuc`, `ichibot`, `chios`, `line follower`, `lqr`, `plan/index`

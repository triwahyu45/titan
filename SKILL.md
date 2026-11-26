---
name: TITAN
description: Robotics, Edge CV & Embedded Systems Ops playbook covering Linux/NUC ROS2 & omni/swerve kinematics, YOLOv8/OpenCV FPS optimization, ESP32/STM32 PlatformIO serial & OLED workflows, H-1 competition recovery, and CAD-to-Simulation (FreeCAD MCP + MuJoCo).
---

# Titan — Robotics, Edge CV & Embedded Ops

End-to-end field playbook for autonomous robots, edge vision nodes, and microcontroller hardware.

---

## Modules

1. **Edge CV & Linux/NUC Ops** ([`references/edge-cv-and-nuc.md`](references/edge-cv-and-nuc.md)):
   - YOLOv8 / OpenCV FPS optimization (ONNX/TensorRT/OpenVINO, zero-copy capture, threaded inference), Ubuntu headless/GUI scaling, Tailscale P2P, and H-1 competition emergency recovery.
2. **Embedded, Kinematics & CAD-to-Sim** ([`references/embedded-and-cad-sim.md`](references/embedded-and-cad-sim.md)):
   - ESP32/STM32 PlatformIO flash & serial verification, 128×64 OLED bitmap/video conversion, omni/swerve kinematics sanity checks, and parametric FreeCAD (`freecad-forge-mcp`) → MuJoCo simulation workflow.
---

## Mandatory Self-Audit & Revise-Before-Delivery Gate

Before declaring any robotics, CV, or embedded task done:
1. Run the actual build/flash/test command and inspect live serial output (`pio device monitor`) or real inference FPS logs — never assume success.
2. For CAD-to-MuJoCo assets, visually inspect rendered simulation frames to verify collision meshes and joint limits.

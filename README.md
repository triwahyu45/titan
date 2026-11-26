# TITAN — Robotics, Edge CV & Embedded Systems Ops

AI Agent Skill for **Antigravity**, **Claude Code**, and **OpenAI Codex**. Field playbook for Linux/NUC ROS2, omni/swerve kinematics, YOLOv8/OpenCV FPS optimization, ESP32/STM32 PlatformIO serial & OLED workflows, and FreeCAD-to-MuJoCo simulation.

---

## Quick Install (Claude Code / Codex / Antigravity)

Clone this skill into your agent's global or project skills directory:

| Agent | Global Install Path | Project-Local Path |
| :--- | :--- | :--- |
| **Antigravity** | `~/.gemini/config/skills/TITAN` | `.agents/skills/TITAN` |
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

## When to Trigger / Search Keywords

`titan`, `ros2`, `yolov8`, `opencv fps`, `esp32`, `stm32`, `platformio`, `omni kinematics`, `swerve`, `mujoco`, `nuc`

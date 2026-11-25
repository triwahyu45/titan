# Embedded, Kinematics & CAD-to-Simulation

## 1. ESP32 / STM32 PlatformIO & Serial Verification
- **Build & Flash**: `pio run -t upload` with explicit `upload_port` and `monitor_speed = 115200`.
- **Serial Verification**: Capture boot banner and sensor/actuator telemetry over serial immediately after flash; never assume flash succeeded without reading serial output.
- **OLED Displays (128×64 SSD1306/SH1106)**: Convert frames to 1-bit packed byte arrays (`XBM` / horizontal byte-aligned) and cap I2C clock at `400kHz–800kHz` for smooth frame rates.

## 2. Kinematics & CAD-to-Simulation (`FreeCAD` + `MuJoCo`)
- **Omni / Swerve Kinematics**: Verify wheel angle sign conventions (`atan2(vy, vx)`), chassis-to-wheel velocity matrix symmetry, and motor saturation clamping (`max_rpm`) in unit tests before hardware deployment.
- **CAD-to-Sim Loop**: Author parametric parts in FreeCAD (`freecad-forge-mcp`) → export clean watertight STEP/STL meshes → normalize inertia/collision geometry in MuJoCo (`MJCF`) and verify joint limits visually.

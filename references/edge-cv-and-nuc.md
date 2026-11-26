# Edge CV & Linux/NUC Field Ops

## 1. YOLOv8 / OpenCV FPS Optimization
- **Decouple Capture & Inference**: Read camera frames in a dedicated producer thread (`queue.Queue(maxsize=1)` dropping stale frames) so inference never stalls on old buffered frames.
- **Export for Target Accelerator**: Prefer TensorRT (`.engine`, FP16) on NVIDIA GPUs, OpenVINO on Intel NUC iGPUs, or ONNX Runtime over raw PyTorch `.pt` for 2–4× speedup.
- **Resize Before Pre-Processing**: Match camera capture resolution (`640x480` or `640x360`) to model input (`imgsz=640` or `416`) to avoid CPU resize bottlenecks.

## 2. Headless Mini-PC / NUC Competition Readiness
- **Network & Remote Access**: Lock static LAN IP + Tailscale P2P fallback; verify SSH and `systemctl` service auto-start on boot.
- **Display & Power**: Disable sleep/suspend (`systemctl mask sleep.target suspend.target`), set performance governor, and verify virtual display/X11 forwarding if GUI tools (RViz2/OpenCV debug) are needed.
- **H-1 Recovery Checklist**: Keep a known-good backup branch/tag, verify camera `/dev/video*` & serial `/dev/ttyUSB*` udev rules, and run a 60-second cold-boot end-to-end smoke test.

---

## 3. Zero-Latency 60 FPS V4L2 / DSHOW Threaded Camera

- **Linux V4L2 Auto-Detect**: Scan `/sys/class/video4linux/video*/name` to locate the exact USB camera node (e.g., Logitech Brio) rather than hardcoding `/dev/video0`.
- **Lock Manual Exposure & Disable Dynamic FPS**: Set `MJPG`, `CAP_PROP_BUFFERSIZE = 1`, `CAP_PROP_AUTO_EXPOSURE = 1` (manual), and `CAP_PROP_AUTOFOCUS = 0`; on Linux run `v4l2-ctl -d <node> -c exposure_dynamic_framerate=0` so dark competition venues never silently drop capture FPS.
- **Background Frame-Grabber Thread**: Read frames in a background `threading.Thread` with non-blocking auto-reconnect on consecutive drops (saves `30–45 ms` per control cycle).

---

## 4. Sub-Millisecond Vectorized Lane Tracker (No Heavy DL Needed)

1. **ROI Pre-Crop (`>60%` faster)**: Crop to the active road band (`y` in `[200, 350]` at `640×360`) before `cvtColor` and `GaussianBlur`.
2. **O(N) Median Floor (`<0.01 ms`)**: Compute 4× subsampled row floor via `np.partition` instead of slow full-frame adaptive thresholding.
3. **Vectorized Run-Length Clustering (`<0.30 ms`)**: Use `np.flatnonzero`, `np.diff`, and `np.split` per slice, then trace chains bottom-up (`|Δx| <= 35 px`) to prevent curb latching on tight hairpins.
4. **Ridge-Regularized 4×4 Normal Equations (`<0.02 ms`)**: Solve `(A^T A + λI)θ = A^T Y` with Tikhonov curvature penalty `λ` on `a²` and smooth quadratic↔linear turning-point blending.

---

## 5. OS-Level Competition Fail-Safe E-Stop & Dead-Man Switch

- Register `atexit`, `SIGINT`, and `SIGTERM` handlers in the NUC/MiniPC Python controller to switch the MCU to Mode 0 (`Manual`) and burst-send `PWM = 0, steer = 0.0` (3× with `5 ms` spacing) on any exit or crash.
- Pair autonomous mode with an analog-trigger dynamic throttle (`R2`) and toggle/hold override so the human operator can cut propulsion instantaneously.

---

## 6. ROS2 Band-Limited Tustin Discrete PID (Zero Derivative Kick)

Use trapezoidal integration with integrator clamp (`limMinInt..limMaxInt`) and a derivative-on-measurement band-limited differentiator (`tau` low-pass) to eliminate setpoint derivative kick:
```python
integrator = clamp(integrator + 0.5 * Ki * T * (err + prev_err), limMinInt, limMaxInt)
diff = -(2.0 * Kd * (meas - prev_meas) + (2.0 * tau - T) * diff) / (2.0 * tau + T)
out = clamp(Kp * err + integrator + diff, limMin, limMax)
```

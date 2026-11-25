# Edge CV & Linux/NUC Field Ops

## 1. YOLOv8 / OpenCV FPS Optimization
- **Decouple Capture & Inference**: Read camera frames in a dedicated producer thread (`queue.Queue(maxsize=1)` dropping stale frames) so inference never stalls on old buffered frames.
- **Export for Target Accelerator**: Prefer TensorRT (`.engine`, FP16) on NVIDIA GPUs, OpenVINO on Intel NUC iGPUs, or ONNX Runtime over raw PyTorch `.pt` for 2–4× speedup.
- **Resize Before Pre-Processing**: Match camera capture resolution (`640x480` or `640x360`) to model input (`imgsz=640` or `416`) to avoid CPU resize bottlenecks.

## 2. Headless Mini-PC / NUC Competition Readiness
- **Network & Remote Access**: Lock static LAN IP + Tailscale P2P fallback; verify SSH and `systemctl` service auto-start on boot.
- **Display & Power**: Disable sleep/suspend (`systemctl mask sleep.target suspend.target`), set performance governor, and verify virtual display/X11 forwarding if GUI tools (RViz2/OpenCV debug) are needed.
- **H-1 Recovery Checklist**: Keep a known-good backup branch/tag, verify camera `/dev/video*` & serial `/dev/ttyUSB*` udev rules, and run a 60-second cold-boot end-to-end smoke test.

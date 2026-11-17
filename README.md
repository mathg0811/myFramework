# visionarch

**A model-agnostic, low-latency edge vision inference pipeline.**

[한국어 요약 (Korean summary)](README.ko.md)

Models are treated as replaceable commodities. This project focuses on the system around them:

- **Model-agnostic boundaries** — inference runtime, pre/post-processing, business logic, and I/O are strictly separated behind Protocols. Swapping YOLOX → RT-DETR, or ONNX Runtime → TensorRT, does not touch upper layers.
- **Defenses against AI uncertainty** — every model output passes a guardrail chain (confidence floors, geometric sanity, domain rules, numeric checks) and a fallback policy before reaching downstream logic.
- **Low latency on constrained hardware** — an asynchronous producer–consumer pipeline from day one; FP16/INT8 via ONNX Runtime / TensorRT in Phase 1, CUDA-stream and memory-bandwidth optimization in Phase 2.

## Status
Step 1 — prototype scope and interface design. See [`docs/ARCHITECTURE.md`](docs/ARCHITECTURE.md).

| Area | Prototype targets |
|------|-------------------|
| Platforms | Windows + NVIDIA GPU, CPU (Linux CI); Jetson later |
| Tasks | Object detection (YOLOX, RT-DETR), human pose (RTMPose, top-down); segmentation and VLM planned |
| Runtime | ONNX Runtime (CPU / CUDA / TensorRT EP); native TensorRT later |
| Toolchain | Python 3.11, `uv` |

## Roadmap
1. Architecture & interface design
2. Core pipeline + per-module unit / contract tests
3. Phase 1 optimization (runtime, quantization) + benchmarks
4. AI-uncertainty guardrails integration
5. DevOps (Docker, CI) + troubleshooting scenarios

## Repository conventions
Working guidelines and the decision log live in [`CLAUDE.md`](CLAUDE.md). Git-Flow branching (`main` / `dev` / `feature/*`), Conventional Commits, and contract tests in every PR.

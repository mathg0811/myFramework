# visionarch (한국어 요약)

> 이 문서는 요약본입니다. 모든 정식 문서는 영어로 작성되며, 기준 문서는 [README.md](README.md)와 [`docs/ARCHITECTURE.md`](docs/ARCHITECTURE.md)입니다.

**모델 비종속(Model-Agnostic) · 저지연 엣지 비전 추론 파이프라인**

AI 모델은 언제든 교체 가능한 부품으로 취급하고, 그 주변 시스템에 집중합니다.

- **모델 비종속 경계** — 추론 런타임, 전/후처리, 비즈니스 로직, I/O를 Protocol 뒤로 엄격히 분리. YOLOX → RT-DETR, ONNX Runtime → TensorRT 교체 시 상위 레이어 수정 없음.
- **AI 불확실성 방어** — 모든 모델 출력은 가드레일 체인(컨피던스, 기하학적 검증, 도메인 룰, 수치 검사)과 폴백 정책을 통과한 뒤에만 하류로 전달.
- **저지연 · 자원 최적화** — 초기부터 비동기 Producer-Consumer 구조. Phase 1: ONNX Runtime / TensorRT 기반 FP16·INT8, Phase 2: CUDA Stream · 메모리 대역폭 최적화.

## 현재 단계
Step 1 — 프로토타입 범위 정의 및 인터페이스 설계

- 플랫폼: Windows + NVIDIA GPU, CPU (Jetson은 이후 확장)
- 태스크: Object Detection (YOLOX, RT-DETR), Human Pose (RTMPose, Top-down) / Segmentation · VLM 예정
- 툴체인: Python 3.11, `uv`

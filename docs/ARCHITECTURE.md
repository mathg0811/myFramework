# Architecture & Interface Design — v0.2

> Status: **ACCEPTED (baseline)** — merged via PR #5. Further changes go through PRs with a doc diff.
> Related decisions: `CLAUDE.md` D-001 – D-012
> Open follow-up: model-graph (chaining) interface for top-down pose (D-011) — see §5.8.

---

## 1. Prototype Scope

| Area | Prototype (v0) | Later |
|------|----------------|-------|
| Input | Video file, image directory | USB/CSI camera, RTSP |
| Tasks | Object detection (YOLOX, RT-DETR), human pose (RTMPose, top-down) | Instance segmentation, VLM |
| Runtime | ONNX Runtime (CPU EP, CUDA EP), FakeBackend | Native TensorRT, Jetson (aarch64) |
| Precision | FP32, FP16 | INT8 (calibration) |
| Pipeline | Thread-based producer–consumer, batch = 1 | Micro-batching, CUDA streams, IOBinding zero-copy |
| Guardrails | Confidence / NMS / geometry / class allowlist, backend fallback | Temporal consistency (tracking), VLM output schema validation |
| Platforms | Windows 10/11 x64, Linux x64 (CI) | Jetson Orin |

---

## 2. Layers & Dependency Direction

```
                 ┌───────────────────────────────────────────────┐
  apps / CLI     │  apps/  (run_video.py, bench.py)              │
                 └───────────────────────┬───────────────────────┘
                                         │ uses
                 ┌───────────────────────▼───────────────────────┐
  orchestration  │  pipeline/  Stage, BoundedQueue, Runner       │
                 └──┬──────────┬──────────────┬───────────┬──────┘
                    │          │              │           │
         ┌──────────▼──┐ ┌─────▼──────┐ ┌─────▼─────┐ ┌───▼────────┐
  modules│  io/        │ │  tasks/    │ │guardrails/│ │ telemetry/ │
         │ Source/Sink │ │ TaskModel  │ │ Guard,    │ │ Timer,     │
         │             │ │ Pre / Post │ │ Fallback  │ │ Metrics    │
         └──────┬──────┘ └─────┬──────┘ └─────┬─────┘ └───┬────────┘
                │        ┌─────▼──────┐       │           │
                │        │  runtime/  │       │           │
                │        │ Backend    │       │           │
                │        │ (ORT/TRT/  │       │           │
                │        │  Fake)     │       │           │
                │        └─────┬──────┘       │           │
                └──────────────┴──────┬───────┴───────────┘
                               ┌──────▼──────┐
  contracts                    │   core/     │  types, Protocols, errors, registry
                               │ (numpy only)│  ← no third-party dependencies
                               └─────────────┘
```

**Rule**: no imports against the arrows. `tasks/` knows only the backend *Protocol*, never a concrete backend (onnxruntime, ...). Concrete implementations are injected through the `registry` and config.

---

## 3. Data Flow (Runtime View)

```
 VideoFileSource ──Q1──▶ Preprocess ──Q2──▶ Infer ──Q3──▶ Postprocess ──Q4──▶ Guardrails ──Q5──▶ Sink
  (decode thread)        (letterbox,        (Backend      (decode, NMS,        (verdict,          (json/video/
                          normalize)         .infer)       inverse xform)       fallback)          metrics)

  A FramePacket travels through every queue and is filled in progressively:
  { frame, frame_id, pts, preprocess_meta, raw_outputs, result, verdict, timings{stage: ns} }
```

- Q1–Q5: `BoundedQueue(maxsize, policy)` — policies: `BLOCK` (default; deterministic for file input) / `DROP_OLDEST` (live) / `DROP_NEWEST`
- Shutdown: source EOF → `EndOfStream` sentinel propagates → stages stop in order (graceful shutdown)
- Errors: a stage exception is wrapped per packet as `PacketError` and forwarded downstream (no whole-pipeline crash); only fatal errors stop the Runner

> ⚠️ **Design warning 1 — lost preprocessing metadata**: if letterbox scale/pad does not reach the Postprocessor, bbox coordinates are wrong (the most common bug of this kind). `PreprocessMeta` is therefore a mandatory packet field, and a contract test verifies the round trip.
>
> ⚠️ **Design warning 2 — the GIL**: Phase 1 is thread-based. onnxruntime `run()` and OpenCV decode/resize release the GIL, so threads suffice. Python-level NMS/post-processing loops hold the GIL and must be vectorized with numpy. Windows has no `fork`, making multiprocessing expensive; process separation is a Phase 2 topic.

---

## 4. Folder Structure

```
myFramework/
├── CLAUDE.md                     # living working guidelines
├── README.md / README.ko.md
├── pyproject.toml                # src layout, extras: [cpu], [gpu], [dev]
├── docs/
│   ├── ARCHITECTURE.md           # this document
│   ├── adr/                      # Architecture Decision Records
│   └── troubleshooting/          # incident / fix history
├── src/visionarch/
│   ├── core/                     # ★ contract layer — no dependency except numpy
│   │   ├── types.py              #   Frame, TensorSpec, Device, Precision
│   │   ├── results.py            #   task result types (tagged union)
│   │   ├── interfaces.py         #   all Protocol definitions
│   │   ├── errors.py             #   error taxonomy
│   │   └── registry.py           #   name → implementation registration / lookup
│   ├── runtime/                  # inference backends (tensors in → tensors out)
│   │   ├── fake.py               #   deterministic FakeBackend (CI)
│   │   ├── onnxruntime_backend.py#   CPU / CUDA / TensorRT EP
│   │   └── tensorrt_backend.py   #   (Step 3) native TRT + engine cache
│   ├── tasks/                    # per-task pre/post-processing + TaskModel assembly
│   │   ├── common/               #   letterbox, normalize, nms (numpy)
│   │   ├── detection/            #   YOLOX / RT-DETR adapters
│   │   ├── pose/                 #   RTMPose adapter (top-down)
│   │   ├── segmentation/         #   (planned)
│   │   └── vlm/                  #   (planned) includes generation loop
│   ├── io/
│   │   ├── sources/              #   video_file.py, image_dir.py
│   │   └── sinks/                #   json_sink.py, video_overlay_sink.py
│   ├── guardrails/
│   │   ├── rules/                #   confidence, geometry, class_allowlist, ...
│   │   ├── chain.py              #   run guard chain & merge verdicts
│   │   └── fallback.py           #   policy on backend failure
│   ├── pipeline/
│   │   ├── packet.py             #   FramePacket
│   │   ├── queue.py              #   BoundedQueue + backpressure policies
│   │   ├── stage.py              #   Stage base (thread worker)
│   │   └── runner.py             #   assembly / start / shutdown
│   ├── telemetry/                #   per-stage latency, queue depth, FPS
│   └── config/                   #   pydantic schemas + YAML loader
├── configs/                      # run presets (det_cpu.yaml, det_cuda_fp16.yaml)
├── tests/
│   ├── contract/                 # ★ shared Protocol contract suites
│   ├── unit/
│   ├── integration/              # pipeline E2E with FakeBackend
│   └── fixtures/                 # short test videos, golden results
├── benchmarks/                   # latency / throughput scripts + recorded results
├── tools/                        # model export, ONNX validation, engine build
├── docker/                       # (Step 5)
└── .github/workflows/            # CI: Linux + Windows × CPU; GPU on self-hosted
```

---

## 5. Core Interfaces

> These are **contract definitions (signatures)**, not implementations. They move into `src/visionarch/core/` in issue #4.

### 5.1 Basic types (`core/types.py`)

```python
class Device(str, Enum):
    CPU = "cpu"
    CUDA = "cuda"          # Jetson is CUDA too (unified memory exposed as a capability)

class Precision(str, Enum):
    FP32 = "fp32"; FP16 = "fp16"; INT8 = "int8"

@dataclass(frozen=True)
class TensorSpec:
    name: str
    shape: tuple[int, ...]   # -1 = dynamic axis
    dtype: np.dtype

@dataclass(frozen=True)
class Frame:
    image: np.ndarray        # HxWxC, uint8, BGR (fixed convention)
    frame_id: int            # monotonically increasing within a source
    pts_ns: int              # source timestamp
    source_id: str
```

> ⚠️ Phase 2 readiness: assuming tensors are always `np.ndarray` breaks the interface once GPU-resident (zero-copy) buffers arrive.
> Backend I/O is therefore typed as the alias `TensorLike` (Phase 1 = `np.ndarray`, Phase 2 = DLPack-compatible device buffer), and backends expose `supports_device_buffers` as a capability.

### 5.2 Result types — task-agnostic (`core/results.py`)

```python
class TaskType(str, Enum):
    DETECTION = "detection"
    KEYPOINTS = "keypoints"          # human pose
    INSTANCE_SEGMENTATION = "instance_segmentation"
    VLM = "vlm"

@dataclass(frozen=True)
class Detection:
    bbox_xyxy: tuple[float, float, float, float]  # absolute pixels, original frame
    score: float                                  # [0, 1]
    class_id: int
    label: str | None = None

@dataclass(frozen=True)
class DetectionResult:
    task: Literal[TaskType.DETECTION]
    detections: tuple[Detection, ...]

@dataclass(frozen=True)
class KeypointsResult:               # human pose (skeleton definition, e.g. COCO-17, comes from model metadata)
    task: Literal[TaskType.KEYPOINTS]
    detections: tuple[Detection, ...]            # person bboxes
    keypoints: tuple[np.ndarray, ...]            # per instance (K, 3) = x, y (original pixels), score
    skeleton_id: str                             # e.g. "coco17"

@dataclass(frozen=True)
class SegmentationResult:            # planned — extends detection
    task: Literal[TaskType.INSTANCE_SEGMENTATION]
    detections: tuple[Detection, ...]
    masks: "MaskEncoding"            # RLE by default (memory), dense on demand

@dataclass(frozen=True)
class VLMResult:                     # planned
    task: Literal[TaskType.VLM]
    text: str
    structured: Mapping[str, Any] | None   # JSON-mode output (subject to schema validation)
    finish_reason: Literal["stop", "length", "timeout", "error"]

TaskResult = DetectionResult | KeypointsResult | SegmentationResult | VLMResult   # tagged union
```

### 5.3 Inference backend — the runtime boundary (`core/interfaces.py`)

```python
@dataclass(frozen=True)
class BackendCapabilities:
    devices: frozenset[Device]
    precisions: frozenset[Precision]
    dynamic_shapes: bool
    supports_device_buffers: bool    # IOBinding / zero-copy (Phase 2)

@dataclass(frozen=True)
class BackendConfig:
    model_path: Path
    device: Device
    precision: Precision
    device_index: int = 0
    options: Mapping[str, Any] = field(default_factory=dict)  # backend-specific options, isolated

class InferenceBackend(Protocol):
    """Tensors in → tensors out. Knows nothing about tasks or model semantics."""
    capabilities: BackendCapabilities
    def load(self, config: BackendConfig) -> None: ...
    @property
    def input_specs(self) -> Sequence[TensorSpec]: ...
    @property
    def output_specs(self) -> Sequence[TensorSpec]: ...
    def infer(self, inputs: Mapping[str, TensorLike]) -> dict[str, TensorLike]: ...
    def warmup(self, iterations: int = 3) -> None: ...
    def close(self) -> None: ...      # also usable as a context manager
```

### 5.4 Task model — the model boundary (`core/interfaces.py`)

```python
class Preprocessor(Protocol):
    def __call__(self, frame: Frame) -> tuple[dict[str, TensorLike], PreprocessMeta]: ...

class Postprocessor(Protocol[R_co]):
    def __call__(self, outputs: Mapping[str, TensorLike],
                 meta: PreprocessMeta) -> R_co: ...

class TaskModel(Protocol[R_co]):
    """The only 'model' abstraction the pipeline knows.
    Hides whether it is a single forward pass (detection) or a generation loop (VLM)."""
    task: TaskType
    model_id: str                         # for result traceability (name + version + precision)
    def predict(self, frames: Sequence[Frame],
                request: TaskRequest | None = None) -> Sequence[R_co]: ...
```

> 💡 **Why two levels of abstraction (Backend vs TaskModel)?**
> Detection and segmentation are a single `Pre → Backend.infer → Post` pass, but a VLM needs a *token generation loop + KV cache + prompt input*,
> which a single `infer(tensor) -> tensor` cannot express. With the backend as the runtime boundary and `TaskModel` as the model boundary,
> YOLOX → RT-DETR (swap TaskModel), ORT → TensorRT (swap backend), and detection → VLM (add a TaskType) are each independent.
> `TaskRequest` is the extension point for *non-frame inputs* such as VLM prompts or open-vocabulary class lists.

### 5.5 I/O (`core/interfaces.py`)

```python
@dataclass(frozen=True)
class SourceInfo:
    source_id: str; width: int; height: int
    fps: float | None; total_frames: int | None; is_live: bool

class FrameSource(Protocol):
    info: SourceInfo
    def open(self) -> None: ...
    def read(self) -> Frame | None: ...    # None = EOF
    def close(self) -> None: ...

class ResultSink(Protocol):
    def write(self, packet: "FramePacket") -> None: ...
    def flush(self) -> None: ...
    def close(self) -> None: ...
```

### 5.6 Guardrails & fallback (`core/interfaces.py`)

```python
class VerdictLevel(str, Enum):
    PASS = "pass"; FLAGGED = "flagged"; REJECTED = "rejected"

@dataclass(frozen=True)
class GuardrailVerdict:
    level: VerdictLevel
    rule_id: str
    reasons: tuple[str, ...]
    filtered: TaskResult | None     # set when the rule refined the result (e.g. removed low-confidence boxes)

class Guardrail(Protocol):
    rule_id: str
    applies_to: frozenset[TaskType]
    def check(self, result: TaskResult, ctx: GuardContext) -> GuardrailVerdict: ...
    # ctx: original frame size, previous N results (temporal rules), model metadata

class FallbackPolicy(Protocol):
    def on_error(self, error: InferenceError, packet: "FramePacket") -> FallbackAction: ...
    # FallbackAction: RETRY | SWITCH_BACKEND (cuda → cpu) | USE_LAST_GOOD | EMIT_EMPTY | ABORT
```

Default prototype guardrail set:
| rule_id | Check | Effect |
|---------|-------|--------|
| `confidence_floor` | drop score < τ | refine |
| `geometry_sanity` | bbox outside frame / inverted (x2 < x1) / zero area / abnormal aspect ratio | refine + FLAGGED |
| `class_allowlist` | drop out-of-domain classes | refine |
| `count_cap` | abnormally many detections (hallucination burst) | FLAGGED |
| `person_cap` | too many persons for top-down pose (latency bound, D-011) | refine (keep top-k) + FLAGGED |
| `numeric_sanity` | NaN/Inf outputs (typical FP16 overflow symptom) | REJECTED → fallback |

### 5.7 Error taxonomy (`core/errors.py`)

```
VisionArchError
├── ConfigError                # startup validation failure (fail fast)
├── ContractViolation          # an implementation broke a contract (shape / dtype / range)
├── SourceError                # decode failure, abnormal EOF
└── InferenceError
    ├── BackendUnavailable     # CUDA / driver / DLL load failure → CPU fallback candidate
    ├── InferenceTimeout
    └── NumericalInstability   # NaN / Inf
```

### 5.8 Model graph (open — D-011)
Top-down pose chains two models: `detector → crop persons → pose`. Draft direction, to be finalized in its own PR:
- A `CompositeTaskModel` implements the same `TaskModel` Protocol, so the pipeline stays unaware of chaining.
- Intermediate results (person boxes) pass through guardrails (`confidence_floor`, `person_cap`) *before* the second model runs.
- Crops are batched into a single pose call to bound per-frame latency.

---

## 6. Test Strategy — Contract First

- `tests/contract/test_backend_contract.py`: a shared suite parametrized over every `InferenceBackend`
  (spec consistency, dtype preservation, repeat-call determinism, error after `close`, harmless warmup)
- `tests/contract/test_preprocess_roundtrip.py`: letterbox → inverse round-trip error < 0.5 px
- `tests/integration/`: pipeline E2E with `FakeBackend` and a short fixture video; verifies frame_id order and count
- CI matrix: `{ubuntu, windows} × {py3.11}`, CPU only. GPU contract tests via `-m gpu` on the self-hosted runner (D-008)

---

## 7. Risks & Early Warnings

| # | Risk | Mitigation |
|---|------|------------|
| R1 | On Windows, onnxruntime-gpu CUDA/cuDNN DLL mismatch silently falls back to CPU | Verify the active EP after load; raise `BackendUnavailable` if it differs from the requested device (no silent fallback) |
| R2 | TensorRT engines depend on GPU architecture and TRT version → not portable | Engine cache key = (model hash, GPU SM, TRT version, precision) |
| R3 | NaN / accuracy loss in FP16 | `numeric_sanity` guard + accuracy regression vs FP32 (benchmarks/) |
| R4 | File decoding outruns inference → unbounded queues, memory growth | Bounded queues + BLOCK policy, queue-depth telemetry |
| R5 | Per-frame numpy allocations waste GC time and memory bandwidth | Phase 1: reusable preprocess output buffers / Phase 2: pinned memory + IOBinding |
| R6 | Model licensing (Ultralytics is AGPL-3.0) | Apache-2.0 reference models only (D-005) |
| R7 | VLM latency (hundreds of ms to seconds) does not fit a frame-synchronous pipeline | Separate asynchronous VLM branch (keyframe sampling) that never blocks the main loop |
| R8 | Top-down pose latency grows with person count | `person_cap` guardrail + batched crops (§5.8) |

---

## 8. Next Steps
1. #6 — English-only repository (this change)
2. #3 — project skeleton + CI (uv, Linux / Windows)
3. #4 — `core` contracts in code + contract tests + `FakeBackend`
4. Model-graph interface for top-down pose (§5.8)
5. `pipeline/` (queue, stage, runner) + `VideoFileSource` + JSON sink + E2E tests
6. `onnxruntime_backend` (CPU / CUDA) + first detection adapter

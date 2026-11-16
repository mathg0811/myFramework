# Architecture & Interface Design — Draft v0.1

> 상태: **DRAFT / RFC** — 사용자 리뷰 후 확정. 확정 전까지 구현 코드 작성 금지.
> 관련 의사결정: `CLAUDE.md` D-001 ~ D-004

---

## 1. 프로토타입 범위 (Scope)

| 항목 | 프로토타입(v0) 포함 | 이후 단계 |
|------|---------------------|-----------|
| 입력 | Video file, Image dir | USB/CSI, RTSP |
| 태스크 | Object Detection | Instance Segmentation, VLM |
| 런타임 | ONNX Runtime (CPU EP, CUDA EP), FakeBackend | Native TensorRT, Jetson(aarch64) |
| 정밀도 | FP32, FP16 | INT8 (calibration) |
| 파이프라인 | 스레드 기반 Producer-Consumer, batch=1 | Micro-batching, CUDA Stream, IOBinding zero-copy |
| 가드레일 | Confidence/NMS/Geometry/Class allowlist, Backend fallback | Temporal consistency(tracking), VLM 출력 스키마 검증 |
| 플랫폼 | Windows 10/11 x64, Linux x64 (CI) | Jetson Orin |

---

## 2. 레이어 구조 & 의존성 방향

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
                               │ (numpy only)│  ← 서드파티 의존 금지
                               └─────────────┘
```

**규칙**: 화살표 역방향 import 금지. `tasks/`는 `runtime/`의 *Protocol*만 알고 구체 백엔드(onnxruntime 등)를 import 하지 않는다. 구체 구현은 `registry` + config로 주입한다.

---

## 3. 데이터 흐름 (Runtime View)

```
 VideoFileSource ──Q1──▶ Preprocess ──Q2──▶ Infer ──Q3──▶ Postprocess ──Q4──▶ Guardrails ──Q5──▶ Sink
  (decode thread)        (letterbox,        (Backend      (decode, NMS,        (verdict,          (json/video/
                          normalize)         .infer)       inverse xform)       fallback)          metrics)

  FramePacket 이 모든 큐를 통과하며 점점 채워진다:
  { frame, frame_id, pts, preprocess_meta, raw_outputs, result, verdict, timings{stage: ns} }
```

- Q1~Q5: `BoundedQueue(maxsize, policy)` — 정책: `BLOCK`(기본, 파일 입력 결정성 보장) / `DROP_OLDEST`(라이브용) / `DROP_NEWEST`
- 종료: Source EOF → `EndOfStream` sentinel 전파 → 각 Stage 순차 종료 (graceful shutdown)
- 에러: Stage 예외는 패킷 단위로 `PacketError`로 감싸 하류로 전달(파이프라인 전체 crash 방지), 치명 에러만 Runner 중단

> ⚠️ **설계 경고 1 — 전처리 메타 유실**: letterbox scale/pad 정보가 Postprocessor까지 전달되지 않으면 bbox 좌표가 틀어진다(가장 흔한 버그). 그래서 `PreprocessMeta`를 패킷에 강제 포함하고 계약 테스트로 왕복 변환을 검증한다.
>
> ⚠️ **설계 경고 2 — GIL**: Phase 1은 스레드 기반. onnxruntime `run()`과 OpenCV decode/resize는 GIL을 해제하므로 스레드로 충분하다. 단, Python 레벨 NMS/후처리 루프는 GIL을 잡으므로 numpy 벡터화 필수. Windows는 `fork`가 없어 multiprocessing 비용이 크므로 프로세스 분리는 Phase 2 검토 항목.

---

## 4. 폴더 구조 (제안)

```
myFramework/
├── CLAUDE.md                     # 살아있는 작업 지침
├── README.md
├── pyproject.toml                # src 레이아웃, extras: [cpu], [gpu], [dev]
├── docs/
│   ├── ARCHITECTURE.md           # 본 문서
│   ├── adr/                      # Architecture Decision Records
│   └── troubleshooting/          # 장애/해결 이력
├── src/edgevision/               # (패키지명 가칭, Q-001)
│   ├── core/                     # ★ 계약 레이어 — numpy 외 의존성 0
│   │   ├── types.py              #   Frame, TensorSpec, Device, Precision
│   │   ├── results.py            #   Detection/Segmentation/VLM 결과 타입 (Tagged Union)
│   │   ├── interfaces.py         #   모든 Protocol 정의
│   │   ├── errors.py             #   에러 분류 체계
│   │   └── registry.py           #   이름 → 구현 클래스 등록/조회
│   ├── runtime/                  # 추론 엔진 백엔드 (텐서 in → 텐서 out)
│   │   ├── fake.py               #   결정적 출력 FakeBackend (CI용)
│   │   ├── onnxruntime_backend.py#   CPU / CUDA / TensorRT EP
│   │   └── tensorrt_backend.py   #   (Step 3) Native TRT + engine cache
│   ├── tasks/                    # 태스크별 전/후처리 + TaskModel 조립
│   │   ├── common/               #   letterbox, normalize, nms (numpy)
│   │   ├── detection/            #   YOLOX/RT-DETR 등 어댑터
│   │   ├── segmentation/         #   (예정)
│   │   └── vlm/                  #   (예정) 생성 루프 포함
│   ├── io/
│   │   ├── sources/              #   video_file.py, image_dir.py
│   │   └── sinks/                #   json_sink.py, video_overlay_sink.py
│   ├── guardrails/
│   │   ├── rules/                #   confidence, geometry, class_allowlist, ...
│   │   ├── chain.py              #   Guard 체인 실행 & verdict 병합
│   │   └── fallback.py           #   Backend 장애 시 대체 정책
│   ├── pipeline/
│   │   ├── packet.py             #   FramePacket
│   │   ├── queue.py              #   BoundedQueue + 백프레셔 정책
│   │   ├── stage.py              #   Stage 베이스 (스레드 워커)
│   │   └── runner.py             #   조립/기동/종료
│   ├── telemetry/                #   stage 레이턴시, 큐 깊이, FPS
│   └── config/                   #   pydantic 스키마 + YAML 로더
├── configs/                      # 실행 프리셋 (det_cpu.yaml, det_cuda_fp16.yaml)
├── tests/
│   ├── contract/                 # ★ Protocol 공용 계약 테스트 스위트
│   ├── unit/
│   ├── integration/              # FakeBackend로 파이프라인 E2E
│   └── fixtures/                 # 짧은 테스트 영상, golden 결과
├── benchmarks/                   # 레이턴시/처리량 측정 스크립트 + 결과 기록
├── tools/                        # 모델 export, ONNX 검증, engine build
├── docker/                       # (Step 5)
└── .github/workflows/            # CI: Linux+Windows × CPU, GPU는 self-hosted
```

---

## 5. 핵심 인터페이스 초안

> 아래는 **계약 정의(시그니처)** 이며 구현이 아니다. 리뷰 후 `src/edgevision/core/`에 그대로 옮긴다.

### 5.1 기본 타입 (`core/types.py`)

```python
class Device(str, Enum):
    CPU = "cpu"
    CUDA = "cuda"          # Jetson도 CUDA (unified memory 여부는 capability로 표현)

class Precision(str, Enum):
    FP32 = "fp32"; FP16 = "fp16"; INT8 = "int8"

@dataclass(frozen=True)
class TensorSpec:
    name: str
    shape: tuple[int, ...]   # -1 = dynamic axis
    dtype: np.dtype

@dataclass(frozen=True)
class Frame:
    image: np.ndarray        # HxWxC, uint8, BGR (규약 고정)
    frame_id: int            # 소스 내 단조 증가
    pts_ns: int              # 원본 타임스탬프
    source_id: str
```

> ⚠️ Phase 2 대비: 텐서를 `np.ndarray`로만 가정하면 GPU 상주 버퍼(zero-copy)로 확장할 때 인터페이스가 깨진다.
> 그래서 Backend 입출력 타입은 `TensorLike`(Phase 1 = `np.ndarray`, Phase 2 = DLPack 호환 Device 버퍼)라는 별칭으로 정의해 두고, Backend capability로 `supports_device_buffers`를 노출한다.

### 5.2 결과 타입 — 태스크 비종속 (`core/results.py`)

```python
class TaskType(str, Enum):
    DETECTION = "detection"
    INSTANCE_SEGMENTATION = "instance_segmentation"
    VLM = "vlm"

@dataclass(frozen=True)
class Detection:
    bbox_xyxy: tuple[float, float, float, float]  # 원본 프레임 절대 픽셀
    score: float                                  # [0, 1]
    class_id: int
    label: str | None = None

@dataclass(frozen=True)
class DetectionResult:
    task: Literal[TaskType.DETECTION]
    detections: tuple[Detection, ...]

@dataclass(frozen=True)
class SegmentationResult:            # 예정 — Detection 확장
    task: Literal[TaskType.INSTANCE_SEGMENTATION]
    detections: tuple[Detection, ...]
    masks: "MaskEncoding"            # RLE 기본 (메모리 절약), 필요시 dense

@dataclass(frozen=True)
class VLMResult:                     # 예정
    task: Literal[TaskType.VLM]
    text: str
    structured: Mapping[str, Any] | None   # JSON 모드 출력 (스키마 검증 대상)
    finish_reason: Literal["stop", "length", "timeout", "error"]

TaskResult = DetectionResult | SegmentationResult | VLMResult   # Tagged Union
```

### 5.3 추론 백엔드 — 런타임 경계 (`core/interfaces.py`)

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
    options: Mapping[str, Any] = field(default_factory=dict)  # 백엔드 고유 옵션 격리

class InferenceBackend(Protocol):
    """텐서 in → 텐서 out. 태스크/모델 의미를 전혀 모른다."""
    capabilities: BackendCapabilities
    def load(self, config: BackendConfig) -> None: ...
    @property
    def input_specs(self) -> Sequence[TensorSpec]: ...
    @property
    def output_specs(self) -> Sequence[TensorSpec]: ...
    def infer(self, inputs: Mapping[str, TensorLike]) -> dict[str, TensorLike]: ...
    def warmup(self, iterations: int = 3) -> None: ...
    def close(self) -> None: ...      # 컨텍스트 매니저 지원
```

### 5.4 태스크 모델 — 모델 경계 (`core/interfaces.py`)

```python
class Preprocessor(Protocol):
    def __call__(self, frame: Frame) -> tuple[dict[str, TensorLike], PreprocessMeta]: ...

class Postprocessor(Protocol[R_co]):
    def __call__(self, outputs: Mapping[str, TensorLike],
                 meta: PreprocessMeta) -> R_co: ...

class TaskModel(Protocol[R_co]):
    """파이프라인이 아는 유일한 '모델' 추상화.
    단일 forward(Detection)든 생성 루프(VLM)든 내부 사정은 숨긴다."""
    task: TaskType
    model_id: str                         # 결과 추적용 (모델명+버전+정밀도)
    def predict(self, frames: Sequence[Frame],
                request: TaskRequest | None = None) -> Sequence[R_co]: ...
```

> 💡 **왜 2단 추상화인가 (Backend vs TaskModel)**:
> Detection/Segmentation은 `Pre → Backend.infer → Post`의 단일 패스지만, VLM은 *토큰 생성 루프 + KV cache + 프롬프트 입력*이 필요해
> `infer(tensor)->tensor` 하나로 표현할 수 없다. Backend를 런타임 경계로, TaskModel을 모델 경계로 분리하면
> YOLO→RT-DETR(TaskModel 교체), ORT→TensorRT(Backend 교체), Detection→VLM(TaskType 추가)이 각각 독립적으로 가능하다.
> `TaskRequest`는 VLM 프롬프트/open-vocab 클래스 목록 등 *프레임 외 입력*을 담는 확장 지점이다.

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

### 5.6 가드레일 & 폴백 (`core/interfaces.py`)

```python
class VerdictLevel(str, Enum):
    PASS = "pass"; FLAGGED = "flagged"; REJECTED = "rejected"

@dataclass(frozen=True)
class GuardrailVerdict:
    level: VerdictLevel
    rule_id: str
    reasons: tuple[str, ...]
    filtered: TaskResult | None     # 규칙이 결과를 정제(예: 저신뢰 박스 제거)한 경우

class Guardrail(Protocol):
    rule_id: str
    applies_to: frozenset[TaskType]
    def check(self, result: TaskResult, ctx: GuardContext) -> GuardrailVerdict: ...
    # ctx: 원본 frame 크기, 이전 N프레임 결과(temporal rule용), 모델 메타

class FallbackPolicy(Protocol):
    def on_error(self, error: InferenceError, packet: "FramePacket") -> FallbackAction: ...
    # FallbackAction: RETRY | SWITCH_BACKEND(cuda→cpu) | USE_LAST_GOOD | EMIT_EMPTY | ABORT
```

프로토타입 기본 가드레일 세트:
| rule_id | 내용 | 결과 |
|---------|------|------|
| `confidence_floor` | score < τ 제거 | 정제 |
| `geometry_sanity` | bbox가 프레임 밖/역전(x2<x1)/면적 0/비정상 종횡비 | 정제 + FLAGGED |
| `class_allowlist` | 도메인 외 클래스 제거 | 정제 |
| `count_cap` | 비정상 다수 검출(환각성 폭주) | FLAGGED |
| `numeric_sanity` | NaN/Inf 출력 (FP16 overflow 대표 증상) | REJECTED → Fallback |

### 5.7 에러 분류 (`core/errors.py`)

```
EdgeVisionError
├── ConfigError                # 기동 시점 검증 실패 (fail-fast)
├── ContractViolation          # 구현체가 계약 위반 (shape/dtype/범위)
├── SourceError                # 디코드 실패, EOF 이상
└── InferenceError
    ├── BackendUnavailable     # CUDA/드라이버/DLL 로드 실패 → CPU 폴백 후보
    ├── InferenceTimeout
    └── NumericalInstability   # NaN/Inf
```

---

## 6. 테스트 전략 — 계약 중심

- `tests/contract/test_backend_contract.py`: 모든 `InferenceBackend` 구현체에 parametrize 되는 공용 스위트
  (spec 일치, dtype 보존, 반복 호출 결정성, close 후 사용 시 에러, warmup 무해성)
- `tests/contract/test_preprocess_roundtrip.py`: letterbox → 역변환 왕복 오차 < 0.5px
- `tests/integration/`: `FakeBackend` + 짧은 fixture 영상으로 파이프라인 E2E, frame_id 순서/개수 보존 검증
- CI 매트릭스: `{ubuntu, windows} × {py3.11}` CPU 전용. GPU 계약 테스트는 `-m gpu` (self-hosted, Q-005)

---

## 7. 리스크 & 선제 경고

| # | 리스크 | 대응 |
|---|--------|------|
| R1 | Windows에서 onnxruntime-gpu의 CUDA/cuDNN DLL 버전 불일치 → 조용히 CPU로 떨어짐 | 로드 후 실제 활성 EP를 검증, 요청 device와 다르면 `BackendUnavailable` (silent fallback 금지) |
| R2 | TensorRT 엔진은 GPU 아키텍처·TRT 버전에 종속 → 이식 불가 | 엔진 캐시 키 = (모델 해시, GPU SM, TRT 버전, precision) |
| R3 | FP16에서 NaN/정확도 저하 | `numeric_sanity` 가드 + FP32 대비 정확도 회귀 테스트 (benchmarks/) |
| R4 | 파일 입력은 디코드가 추론보다 빨라 큐 폭주 → 메모리 증가 | Bounded queue + BLOCK 정책, 큐 깊이 텔레메트리 |
| R5 | 매 프레임 numpy 할당으로 GC/메모리 대역폭 낭비 | Phase 1: 전처리 출력 버퍼 재사용 풀 / Phase 2: pinned memory + IOBinding |
| R6 | 모델 라이선스 (Ultralytics AGPL-3.0) | 레퍼런스 모델은 Apache-2.0 계열 권장 (Q-003) |
| R7 | VLM은 레이턴시가 수백 ms~초 단위로 프레임 동기 파이프라인과 맞지 않음 | VLM은 별도 비동기 브랜치(키프레임 샘플링) 로 설계 예정 — 메인 루프를 막지 않음 |

---

## 8. 다음 작업 (제안 순서)
1. 본 문서 리뷰 & Open Questions(Q-001~Q-006) 결정
2. `pyproject.toml` + 빈 패키지 뼈대 + CI(lint/test, Linux+Windows) — PR #1
3. `core/` 타입·Protocol·에러 + 계약 테스트 + `FakeBackend` — PR #2
4. `pipeline/` (queue, stage, runner) + `VideoFileSource` + JSON Sink + E2E 테스트 — PR #3
5. `onnxruntime_backend` (CPU/CUDA) + Detection 어댑터 1종 — PR #4

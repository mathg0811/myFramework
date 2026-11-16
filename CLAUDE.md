# CLAUDE.md — 프로젝트 작업 지침 (Living Document)

> 이 문서는 AI 에이전트(테크 리드 역할)와 개발자가 공유하는 **살아있는 작업 지침**이다.
> 의사결정이 바뀌거나 새 규칙이 생기면 반드시 이 문서와 하단 Changelog를 함께 갱신한다.

- 문서 버전: v0.2 (2026-11-16)
- 현재 단계: **Step 1 — 프로토타입 범위 정의 & 인터페이스 설계 (Draft)**
- 상세 설계: [`docs/ARCHITECTURE.md`](docs/ARCHITECTURE.md)

---

## 1. 프로젝트 한 줄 정의
모델 비종속(Model-Agnostic)·저지연 엣지 비전 추론 파이프라인. 모델은 교체 가능한 부품이며,
**시스템 아키텍처 / 인터페이스 계약 / 불확실성 방어 / 하드웨어 최적화**가 본 프로젝트의 산출물이다.

## 2. 확정된 의사결정 (Decision Log)
| ID | 결정 | 근거 / 비고 | 일자 |
|----|------|-------------|------|
| D-001 | 1차 검증 타깃: **Windows + NVIDIA GPU** 와 **CPU** 두 경로 동시 검증. Jetson 등은 이후 확장 가능하도록 구조만 확보 | Backend 추상화 + Device 추상화로 흡수 | 2026-11-16 |
| D-002 | 언어: **Python first, C++/CUDA later** | 계약·테스트를 Python으로 먼저 고정, Phase 2에서 핫패스만 교체 | 2026-11-16 |
| D-003 | 첫 태스크: **Object Detection**. 단, Segmentation·VLM이 곧 붙을 수 있도록 결과 타입/모델 인터페이스를 태스크 비종속으로 설계 | 결과는 Tagged Union, VLM은 생성형 루프를 수용하는 `TaskModel` 레벨에서 추상화 | 2026-11-16 |
| D-004 | 주 입력: **Video file (재현 가능한 테스트)** | 기본 백프레셔 정책 = BLOCK(무손실·결정적). 라이브 소스용 DROP 정책은 인터페이스만 정의 | 2026-11-16 |
| D-005 | 레퍼런스 모델: **YOLOX + RT-DETR (Detection)**, **Human Pose Estimation 1종 추가** (후보: RTMPose / ViTPose, Apache-2.0) | 서로 다른 아키텍처(CNN/Transformer)·태스크로 Model-Agnostic 경계를 실증. Ultralytics(AGPL) 제외. `TaskType.KEYPOINTS` 추가 | 2026-11-16 |
| D-006 | GPU 런타임 우선순위: **ONNX Runtime(CPU/CUDA/TensorRT EP) 먼저 → Native TensorRT 백엔드는 Step 3** | 하나의 백엔드로 CPU/GPU 두 경로를 먼저 검증 | 2026-11-16 |
| D-007 | 브랜치 전략: **Git-Flow** (`main` / `develop` / `feature/*` / `release/*` / `hotfix/*`) | 상세 규칙은 §5.1 | 2026-11-16 |
| D-008 | GPU CI: **사용자 로컬 GPU 노트북을 self-hosted runner로 등록** (잠정, 셋업 대기) | CPU CI는 GitHub-hosted, GPU 잡은 수동/야간 트리거로 학습 작업과 충돌 회피 | 2026-11-16 |

## 3. 미결 사항 (Open Questions) — 사용자 결정 필요
- Q-001 패키지명 — 사용자 제안 `visionarchitect` 검토 중 (대안 제시함)
- Q-002 Python 버전 / 패키지 매니저 — `uv` vs `pip` 설명 후 결정 대기 (제안: Python 3.11 + uv)
- Q-007 Pose 모델 최종 선택 (RTMPose: 경량·엣지 친화 / ViTPose: Transformer 계열) 및 Top-down(검출→포즈) vs Bottom-up 구조
- Q-008 리포지토리 공개 여부 (public이면 self-hosted runner 보안 정책 강화 필요)

## 4. 개발 철칙 (반드시 지킬 것)
1. **문서 선행**: 새 모듈은 인터페이스 정의 → 문서(ARCHITECTURE.md 또는 ADR) → 계약 테스트 → 구현 순서.
2. **의존성 방향 고정**: `core` ← `runtime` / `tasks` / `io` / `guardrails` ← `pipeline` ← `apps`.
   `core`는 numpy 외 어떤 서드파티(onnxruntime, tensorrt, cv2, torch)도 import 하지 않는다. (CI에서 import-linter로 강제 예정)
3. **계약 테스트 의무화**: 각 Protocol마다 `tests/contract/`에 공용 테스트 스위트를 두고, 모든 구현체는 이를 통과해야 한다. PR마다 포함.
4. **GPU 없이도 CI가 돌아야 한다**: `FakeBackend`(결정적 출력)로 파이프라인 전체를 검증. GPU 테스트는 `@pytest.mark.gpu`로 분리.
5. **좌표/단위 규약**: 모든 공개 결과의 bbox는 *원본 프레임 기준 절대 픽셀, xyxy, float32*. 역변환(letterbox 해제)은 Postprocessor 책임.
6. **모델 출력은 신뢰하지 않는다**: 모든 결과는 Guardrail 체인을 통과한 뒤에만 Sink/비즈니스 로직으로 전달된다.
7. **Windows 호환성**: `multiprocessing`은 spawn 전제, 경로는 `pathlib`, CUDA DLL 로딩 경로 명시, 쉘 스크립트 대신 Python 툴링.
8. **점진적 실행**: 한 PR = 한 관심사. 뼈대 → 계약 → Fake 구현 → 실제 구현 순.

## 5. 작업 프로세스
- Issue 등록 → 브랜치 생성 → 구현 + 테스트 → PR(인터페이스 변경 시 문서 diff 필수) → 리뷰 → 머지
- 트러블슈팅은 `docs/troubleshooting/YYYY-MM-DD-<slug>.md`에 증상/원인/조치/재발방지로 기록
- 아키텍처 결정은 `docs/adr/NNNN-<slug>.md`로 기록하고 위 Decision Log에 한 줄 추가

### 5.1 브랜치 규칙 (Git-Flow)
| 브랜치 | 분기 원천 | 머지 대상 | 규칙 |
|--------|-----------|-----------|------|
| `main` | — | — | 릴리스 태그(`vX.Y.Z`)만 존재. 직접 push 금지, 보호 브랜치 |
| `develop` | `main` | — | 통합 브랜치. PR + CI 통과 + 리뷰 1건 필수 |
| `feature/<issue#>-<slug>` | `develop` | `develop` | 이슈 1개 = 브랜치 1개. Squash merge |
| `release/vX.Y.0` | `develop` | `main` + `develop` | 기능 동결, 버그픽스·문서만. 머지 후 태그 |
| `hotfix/vX.Y.Z-<slug>` | `main` | `main` + `develop` | 트러블슈팅 문서(`docs/troubleshooting/`) 필수 동반 |
- 커밋 메시지: Conventional Commits (`feat:`, `fix:`, `docs:`, `test:`, `refactor:`, `ci:`, `perf:`)
- 버전: SemVer. `0.x` 동안은 인터페이스 breaking change 허용하되 CHANGELOG에 명시
- AI 에이전트 세션 브랜치(`claude/*`)는 `feature/*`와 동일하게 취급하여 `develop`으로 PR

## 6. AI 에이전트 행동 지침
- 제안 전에 사용자가 결정해야 할 부분이 있으면 **먼저 질문**한다.
- 결함·누락된 예외 처리·메모리/레이턴시 병목이 보이면 능동적으로 경고하고 대안을 제시한다.
- 이 문서를 의사결정이 생길 때마다 갱신한다.

---

## Changelog
- v0.1 (2026-11-16): 초안 작성. D-001~D-004 반영, Open Questions Q-001~Q-006 등록.
- v0.2 (2026-11-16): D-005~D-008 추가(모델 3종, ORT 우선, Git-Flow, self-hosted GPU runner). Git-Flow 규칙 §5.1 추가. Q-007, Q-008 등록.

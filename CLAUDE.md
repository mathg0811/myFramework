# CLAUDE.md — Project Working Guidelines (Living Document)

> Shared working guidelines for the AI agent (acting as tech lead) and the developer.
> Whenever a decision changes or a new rule is introduced, update this document **and** the Changelog at the bottom.

- Document version: v0.7 (2026-11-17)
- Current phase: **Step 1 — Prototype scope & interface design**
- Detailed design: [`docs/ARCHITECTURE.md`](docs/ARCHITECTURE.md)

---

## 1. Project in One Sentence
A model-agnostic, low-latency edge vision inference pipeline. Models are replaceable commodities;
the real deliverables are **system architecture, interface contracts, defenses against AI uncertainty, and hardware optimization**.

### 1.1 Vision (north star)
A **general-purpose perception → reasoning → action framework**: it must be able to grow into robotics
(3D perception + VLA action decisions) and multimodal video understanding (e.g. sports analysis) without
rewriting its core. Only one target is implemented at a time, but every change must respect this extensibility.

### 1.2 What the portfolio must demonstrate
1. **Separation** — how the pipeline is decomposed and how modules run independently
2. **Performance** — measured, reproducible latency/throughput results (benchmarks, not claims)
3. **Safety nets** — how the system detects and contains AI failures, demonstrated with failure-injection benchmarks
4. **Structural uncertainty handling** — the architecture itself (not individual models) makes uncertainty manageable

### 1.3 Extensibility Principles (D-014)
Adopted as guiding principles; each is detailed in `docs/ARCHITECTURE.md` when the relevant module is designed.

| # | Principle | Meaning |
|---|-----------|---------|
| P1 | **Observation, not Frame** | Inputs are timestamped multimodal bundles (N cameras, depth, point clouds, proprioception, text). A single image is just one modality |
| P2 | **Outputs may be actions** | `ActionResult` joins the result union; every action passes a mandatory safety shield (limits, rate limits, workspace bounds). *The model proposes, the system disposes* |
| P3 | **Time is first-class** | Clock domains, multi-sensor synchronization, per-stage deadlines (real-time) vs throughput (offline) |
| P4 | **Open and closed loop** | The runner supports feedback (action → environment → next observation) for robotics; video analysis is open loop |
| P5 | **Location-agnostic compute** | Models are referenced by manifest + URI (local / S3 / GCS); backends may be local or remote; no cloud SDK in `core` |
| P6 | **Training ⟂ inference** | Training and inference meet only at the model-artifact contract (ONNX + manifest + evaluation report). Inference never imports training code |

### 1.4 Framework vs Projects (D-017)
`visionarch` is a **framework**; concrete work is done in **projects** built on it.
- Projects are **config-driven**: components are selected by registry name and assembled from typed, validated configs (YAML → pydantic). No project-specific code in the framework.
- A project owns its configs, data manifests, training recipes, evaluation reports, and benchmarks (`projects/<name>/`).
- First project (D-015): **volleyball pose** — collect and curate volleyball video data, fine-tune the pose model to improve accuracy over the pretrained baseline, deploy it through the pipeline; later extend toward robot actions in simulation.

## 2. Decision Log
| ID | Decision | Rationale / Notes | Date |
|----|----------|-------------------|------|
| D-001 | Primary validation targets: **Windows + NVIDIA GPU** and **CPU**, validated side by side. Jetson etc. must remain possible via structure only | Absorbed by the Backend and Device abstractions | 2026-11-16 |
| D-002 | Language: **Python first, C++/CUDA later** | Lock contracts and tests in Python; replace only hot paths in Phase 2 | 2026-11-16 |
| D-003 | First task: **Object Detection**, with task-agnostic result types / model interfaces so Segmentation and VLM can follow | Results are a tagged union; VLM's generative loop is abstracted at the `TaskModel` level | 2026-11-16 |
| D-004 | Primary input: **video files (reproducible tests)** | Default backpressure policy = BLOCK (lossless, deterministic). DROP policies for live sources are interface-only for now | 2026-11-16 |
| D-005 | Reference models: **YOLOX + RT-DETR (detection)** plus **one human pose estimation model** | Different architectures (CNN / Transformer) and tasks prove the model-agnostic boundary. Ultralytics (AGPL-3.0) excluded. Adds `TaskType.KEYPOINTS` | 2026-11-16 |
| D-006 | GPU runtime priority: **ONNX Runtime (CPU / CUDA / TensorRT EP) first → native TensorRT backend in Step 3** | Validate both CPU and GPU paths with a single backend first | 2026-11-16 |
| D-007 | Branching: **Git-Flow** (`main` / `dev` / `feature/*` / `release/*` / `hotfix/*`) | Integration branch named `dev` rather than `develop` (2026-11-17). Rules in §5.1 | 2026-11-16 |
| D-008 | GPU CI: **developer's local Windows GPU laptop registered as a self-hosted runner** (setup pending) | CPU CI on GitHub-hosted runners; GPU jobs only via `workflow_dispatch` or the `gpu-test` label to avoid clashing with training. Assumes no outside contributors — revisit runner security when collaboration starts | 2026-11-16 |
| D-009 | Package name: **`visionarch`** (`import visionarch`) | Distinctive, short import path | 2026-11-17 |
| D-010 | Toolchain: **Python 3.11 + `uv`** (`uv.lock` must be committed) | Reproducible dependencies across laptop / CI / Jetson. `pyproject.toml` stays standard, so falling back to pip is possible | 2026-11-17 |
| D-011 | Pose: **RTMPose, top-down** (detector → crop → pose) | Demonstrates multi-model graphs (model chaining). Latency scales with person count → per-frame person cap guardrail required | 2026-11-17 |
| D-012 | Repository language: **English only**; Korean summary in `README.ko.md` only | Portfolio readiness; a single source of truth avoids translation drift. Developer ↔ agent conversation may stay in Korean. Pre-policy commit history is kept as-is | 2026-11-17 |
| D-013 | License: **Apache-2.0** | Matches the Apache-2.0 reference models; permissive for portfolio use | 2026-11-17 |
| D-014 | Adopt extensibility principles **P1–P6** (§1.3) | Keeps the core general enough for robotics/VLA and multimodal video while implementing one target at a time. Details designed per module | 2026-11-17 |
| D-015 | First reference scenario: **volleyball video pose estimation**, including data collection/curation and fine-tuning; robotics comes second | Builds on detection + pose + video files (D-003–D-005), fits 4–8 GB VRAM, runs in CI without hardware | 2026-11-17 |
| D-016 | Robotics: **simulation only** (e.g. MuJoCo / Isaac Sim) with small VLA models | No physical robot; 7B-class VLAs do not fit the target VRAM | 2026-11-17 |
| D-017 | **Framework / project separation**, config-driven assembly (registry + typed YAML configs) | Projects reuse the framework without modifying it; experiments are reproducible from config | 2026-11-17 |

## 3. Open Questions / Pending Actions
- Developer laptop GPU model and VRAM (`nvidia-smi`) — input for latency/memory targets
- Volleyball data source and its license (self-recorded vs public datasets vs broadcast footage)
- Training environment (Windows native vs WSL2/Docker vs cloud) and training stack
- Data/annotation tooling and dataset versioning (to be decided in the data-management design)
- Developer action: configure branch protection for `main` and `dev`
  (require PR + status checks; approvals = 0 while working solo).

## 4. Engineering Rules (mandatory)
1. **Docs first**: interface definition → doc (ARCHITECTURE.md or ADR) → contract tests → implementation.
2. **Fixed dependency direction**: `core` ← `runtime` / `tasks` / `io` / `guardrails` ← `pipeline` ← `apps`.
   `visionarch.core` must not import any third-party package other than numpy (onnxruntime, tensorrt, cv2, torch, ...). Enforced in CI via import-linter.
3. **Contract tests are mandatory**: every Protocol has a shared suite in `tests/contract/` that all implementations must pass. Included in every PR.
4. **CI must run without a GPU**: the whole pipeline is validated with a deterministic `FakeBackend`. GPU tests are isolated with `@pytest.mark.gpu`.
5. **Coordinate convention**: every public bbox is *absolute pixels in the original frame, xyxy, float32*. Undoing letterbox is the Postprocessor's responsibility.
6. **Never trust model output**: every result passes the guardrail chain before reaching sinks or business logic.
7. **Windows compatibility**: assume `multiprocessing` spawn, use `pathlib`, make CUDA DLL search paths explicit, prefer Python tooling over shell scripts.
8. **Incremental delivery**: one PR = one concern. Skeleton → contracts → fake implementation → real implementation.
9. **Language**: all tracked content in English (D-012).

## 5. Workflow
- Issue → branch → implementation + tests → PR (interface changes require a doc diff) → review → merge
- **Issues are living records**: update the description/checklist as the situation changes. When scope drifts, close the issue with a note
  and open a new one rather than letting it grow (one issue = one concern). Every manual close states what resolved it.
- Closing keywords (`Closes #N`) fire only when a PR merges into the default branch (`dev`) and are evaluated at merge time;
  if one does not fire, close the issue manually with a reference to the PR.
- Troubleshooting records go to `docs/troubleshooting/YYYY-MM-DD-<slug>.md` (symptom / root cause / fix / prevention)
- Architecture decisions go to `docs/adr/NNNN-<slug>.md`, with a one-line entry in the Decision Log above

### 5.1 Branching Rules (Git-Flow)
| Branch | Created from | Merges into | Rules |
|--------|--------------|-------------|-------|
| `main` | — | — | Release tags (`vX.Y.Z`) only. No direct pushes; protected |
| `dev` | `main` | — | Integration branch (default PR target). PR + green CI required |
| `feature/<issue#>-<slug>` | `dev` | `dev` | One issue = one branch. Squash merge |
| `release/vX.Y.0` | `dev` | `main` + `dev` | Feature freeze; bug fixes and docs only. Tag after merge |
| `hotfix/vX.Y.Z-<slug>` | `main` | `main` + `dev` | Must include a troubleshooting record in `docs/troubleshooting/` |
- Commit messages: Conventional Commits (`feat:`, `fix:`, `docs:`, `test:`, `refactor:`, `ci:`, `perf:`, `chore:`)
- Versioning: SemVer. Breaking interface changes are allowed during `0.x` but must be noted in the CHANGELOG
- AI agent session branches (`claude/*`) are treated as `feature/*` branches and PR into `dev`

## 6. AI Agent Conduct
- Ask the developer first whenever a decision is theirs to make, before proposing.
- Proactively flag defects, missing error handling, and memory/latency bottlenecks, with alternatives.
- Keep this document updated whenever a decision is made.
- Converse with the developer in Korean; write everything committed to the repository in English.

---

## Changelog
- v0.1 (2026-11-16): Initial draft. D-001–D-004; open questions Q-001–Q-006.
- v0.2 (2026-11-16): Added D-005–D-008 (reference models, ORT first, Git-Flow, self-hosted GPU runner) and branching rules §5.1.
- v0.3 (2026-11-17): Added D-009 (`visionarch`), D-010 (Python 3.11 + uv), D-011 (RTMPose top-down). Integration branch renamed `develop` → `dev`.
- v0.4 (2026-11-17): Added D-012 (English-only repository). Translated the whole document to English. Clarified solo-developer branch protection (approvals = 0).
- v0.5 (2026-11-17): Added issue hygiene rules (§5): living issues, split on scope drift, manual-close notes.
- v0.6 (2026-11-17): Added vision (§1.1) and portfolio goals (§1.2); D-013 (Apache-2.0); new open questions.
- v0.7 (2026-11-17): Added extensibility principles P1–P6 (§1.3), framework/project separation (§1.4); D-014–D-017 (volleyball pose as first project, simulation-only robotics, config-driven projects).

# Plan — Notion 페이지 "(협동2) Humble→Jazzy 마이그레이션" Section 3 작성 (Concise)

## Context

대상 Notion 페이지: <https://www.notion.so/teamsparkx/2-Humble-Jazzy-36c563918e59805997c4e8e533067303>

페이지의 현재 상태:
- Section 1 (humble 셋업) / Section 2 (jazzy 목표 + 컨테이너) — 작성 완료
- **Section 3 (필요 기술 스택 및 고려사항)** = `<empty-block/>` 1개만 있는 빈 섹션
- 부록: 2 mention-page (Docker 코드 전달 / host PC ↔ container 통신) — 보존

본 plan 의 의도: Section 3 채워서 Phase 4 진입 전 학습 / 결정 가이드 역할 수행. **Concise** (1 page 수준, reference 카드 성격). 이전 comprehensive 안 (~2-3 page) 은 사용자 피드백으로 폐기.

**핵심 추가 요인**: 실제 사용될 application 코드 2 묶음:
- `/home/rokey/ros2_jazzy_test/cobot2_ws.zip` — ROS2 워크스페이스, 5개 ROS2 package (`voice_processing`, `robot_control`, `object_detection`, `od_msg`, `pick_and_place_text/voice`, `rokey`)
- `/home/rokey/ros2_jazzy_test/corecode.zip` — tutorial / 연구 자료 (Calibration / DRL / YOLO / VoiceProcessing)

이게 Section 3 의 일부 항목 (특히 custom message 정합성, gripper 통신 경로, 컨테이너 분할 경계) 의 결정 근거 됨.

---

## Section 3 의 최종 구조 (압축)

```
3. 필요 기술 스택 및 고려사항
├─ 3-1. 필수 기술 스택 (단일 표 1개 + cobot2_ws 매핑 표 1개)
├─ 3-2. 핵심 고려사항 (7개 항목, 각 2~3줄)
└─ 3-3. 추가 학습 권장 (3 단계, 단계별 4~6개 bullet)
```

총 분량 = 한 페이지 스크롤 안에 들어오는 수준. Comprehensive 안의 표 5개 + 단락 10개 → 표 2개 + 단락 7개 + bullet 3 단계로 압축.

---

## 3-1. 필수 기술 스택 — 단일 통합 표

각 row = 1 기술. column = `기술 / Layer / 본 프로젝트 역할`. Layer = `HW interface / Host / Container / Network / Build·Ops` 5종 중 1개.

| 기술 | Layer | 본 프로젝트 역할 |
|------|-------|---------------|
| Ubuntu 24.04 (noble) + ROS2 jazzy (apt) | Host | system Python (rclpy / launch / colcon) + `/opt/ros/jazzy/` |
| NVIDIA driver + CUDA toolkit (host nvcc) | Host | colcon 빌드 시 CUDA 의존 ROS2 노드 컴파일 |
| Docker CE + buildx + compose plugin | Host | 컨테이너 runtime + 다중 컨테이너 운영 |
| librealsense2 SDK (Intel Noble repo) | Host | RealSense 커널 모듈 + udev rule |
| colcon workspace (`~/cobot2_ws/`) | Host | `doosan-robot2` + 본 프로젝트 cobot2_ws 5개 패키지 빌드 |
| NVIDIA Container Toolkit | HW interface | GPU device + driver lib 자동 mount → yolo 컨테이너 |
| USB device passthrough (`/dev/bus/usb`) | HW interface | RealSense D435I → yolo 컨테이너 |
| PulseAudio/PipeWire socket mount | HW interface | 내장 마이크 → voice 컨테이너 |
| `ros:jazzy-ros-base-noble` (명시 태그) | Container base | yolo / voice image 의 공통 base |
| PyTorch (cu126 wheel 후보) + ultralytics + opencv-python + pyrealsense2 | Container (yolo) | object detection 추론 |
| langchain + openai + PyAudio + openwakeword + tflite-runtime | Container (voice) | wake-word + STT + LLM 호출 |
| numpy<2 마지막 재핀 | Container 공통 | ultralytics 호환 (transitive silent upgrade 차단) |
| ROS2 RMW (Fast-DDS or CycloneDDS, 둘 중 1개로 통일) | Network | DDS 미들웨어 |
| `ROS_DOMAIN_ID` + `network_mode: host` | Network | 모든 노드 동일 domain + multicast discovery 자연 동작 |
| Dockerfile multi-stage + `.dockerignore` + BuildKit | Build·Ops | image build, secret 차단, layer cache |
| docker-compose (`profiles`, `env_file`) | Build·Ops | yolo/voice 동시 운영, `.env` runtime 주입 |
| Docker Hub (public registry, semver + git SHA 태그) | Build·Ops | image 배포, `latest` 금지 |
| bash + shellcheck + `set -euo pipefail` + `apt-mark hold` | Build·Ops | host installer 안정성 |
| apt keyring (`/etc/apt/keyrings/` + `signed-by=`) | Build·Ops | 외부 repo 키 등록 (`apt-key add` deprecated) |

### 3-1-a. 본 프로젝트 application 코드 매핑 (`cobot2_ws/`)

| 패키지 (cobot2_ws/) | 책임 | 배포 위치 |
|-------------------|------|----------|
| `object_detection/` (detection.py, yolo.py, realsense.py) | YOLO 추론 + RealSense frame grab | yolo container |
| `voice_processing/` (get_keyword.py, MicController.py, stt.py, wakeup_word.py, hello_rokey_8332_32.tflite) | wake-word + STT + keyword 추출 | voice container |
| `od_msg/` (CMake 패키지, custom .msg) | object detection 결과 message 타입 | **host + 모든 container 공통 빌드** (interface 정합성) |
| `robot_control/` (robot_control.py, onrobot.py) | DSR + onrobot gripper 제어 | host (DSR_NODE 와 같은 위치 — 실기 Modbus/TCP 직접 접근) |
| `pick_and_place_text/voice/` | voice / text 입력 → robot 동작 통합 task | host (orchestrator 역할, 두 container 의 topic 구독 + robot_control 호출) |
| `rokey/` | 메타 / 공통 utility | host |

`corecode.zip` 은 컨테이너 적재 대상 아님 — 참고 자료 / 연구용 (Calibration / DRL / 학습 tutorial). 단 일부 (onrobot.py, realsense.py) 가 cobot2_ws 와 중복 — 추후 단일 source 화 검토.

---

## 3-2. 핵심 고려사항 — 7 항목 (각 2~3줄)

각 항목 = 짧은 단락 (왜 함정인가 + 회피 방법 핵심 1줄). 내부 룰 ID / ADR 번호 인용 금지 — substance 만 서술.

### 3-2-1. ROS distro 단일 진실 소스 + 호환 매트릭스
- `humble` / `jazzy` 같은 distro 문자열을 스크립트마다 박지 않음. `ROS_DISTRO` 환경변수 하나 정의하고 모두 참조 → 차기 distro 마이그레이션 시 1곳만 변경.
- Ubuntu / ROS / CUDA / PyTorch / numpy / ultralytics 버전 조합은 1곳에 매트릭스로 기록 (이미 `docs/COMPATIBILITY.md` 운영). 변경 시 매트릭스 동시 갱신.

### 3-2-2. Idempotency + 재시작 안전성
- 같은 스크립트 N회 실행해도 동일 결과 — apt source / keyring / `git clone` 중복 사전 체크.
- `apt install pkg=VERSION` 만으로는 후속 `apt upgrade -y` 가 핀 풀어버림 → **`apt-mark hold`** 까지 호출해야 진짜 잠금.
- 설치 실패 / reboot 발생 시 마지막 성공 단계를 state file 에 기록, 재실행 시 다음 단계부터 `[n/total]` 진행.

### 3-2-3. Secret hygiene (Docker Hub publish 안전성)
- API key 가 image layer 에 박히면 public registry 노출이 비가역. 3중 차단: (a) `.dockerignore` 에 `.env`, `*.key` 명시 (b) Dockerfile 에서 `ARG`/`ENV` 로 API key 안 받음, runtime env injection 만 (c) multi-stage build.
- publish 전 mandatory grep: `docker history --no-trunc <image> | grep -iE 'OPENAI|API_KEY|TOKEN|SECRET'` → 0건이어야 push.

### 3-2-4. 책임 분리 — host vs container layer 의 명시적 2-step
- `bash install.sh` = host PC layer 만 (driver / Docker / ROS2 / CUDA / colcon / RealSense SDK).
- `docker compose pull && docker compose up -d` = application layer — 사용자 별도 명령. install.sh 가 컨테이너 layer 호출 안 함.
- cobot2_ws 의 host 측 패키지 (`robot_control`, `pick_and_place_*`, `rokey`, `od_msg`) 는 install.sh 가 colcon 으로 빌드. container 측 (`object_detection`, `voice_processing`) 은 Dockerfile 안에서 colcon 빌드.

### 3-2-5. DDS discovery — `network_mode: host` 필수 + custom interface 정합성
- Docker bridge 는 UDP multicast forward 안 함 → bridge 모드에선 DDS discovery 실패. `network_mode: host` 가 단순한 해결책.
- 모든 노드 (host + 모든 container) 의 `ROS_DOMAIN_ID` + `RMW_IMPLEMENTATION` 가 **완전 동일**해야. Fast-DDS / CycloneDDS 혼합 = 같은 topic 도 못 봄.
- **본 프로젝트 특이사항**: `od_msg` (custom message 타입) 가 host + 두 container 모두에서 동일 정의로 빌드되어야 함. 한쪽에서 message 정의가 다르면 publish/subscribe 가 type 체크 단계에서 실패 — `od_msg` 패키지는 단일 source (cobot2_ws/od_msg) 에서 각 환경이 빌드해서 가져감.

### 3-2-6. GPU / Audio / USB passthrough 함정
- GPU: NVIDIA Container Toolkit + host driver 가 container 안 CUDA runtime 메이저 (cu126 등) 보다 같거나 새로워야. RTX 4060 arch = sm_89.
- Audio: raw `/dev/snd` mount 는 host PipeWire 와 충돌 → **PulseAudio/PipeWire socket mount** + UID 매칭 (`user: "1000:1000"`).
- USB: `--device /dev/bus/usb` + host 의 librealsense2 udev rule 사전 설치. 노트북 USB 3.0 포트 대역폭/전력 제한 — RealSense + 기타 device 공존 시 USB 3.0 허브 권장.

### 3-2-7. 산업 protocol 과 DDS 의 공존 + Linux 전용 제약
- Doosan controller ↔ host DSR_NODE = **TCP socket port 12345 (DRFL)**. DDS 아님. DSR_NODE ↔ 다른 ROS2 노드 = DDS. 한 시스템에 두 layer 공존.
- `robot_control/onrobot.py` 는 Modbus/TCP 로 onrobot gripper 직접 통신 — 또 다른 산업 protocol. 따라서 `robot_control` 노드는 host 에서 직접 실기 IP 접근하는 게 자연.
- `network_mode: host` 완전 동작 환경 = **Linux 만**. macOS / Windows Docker Desktop 에서 host network 동작 보장 안 됨. 본 프로젝트 검증 환경 = Ubuntu 24.04.

---

## 3-3. 추가 학습 권장 기술 — 3 단계 분류

각 단계 = 4~6개 bullet. 각 bullet = 기술명 한 줄 (왜 필요한지 짧게).

### 3-3-A. 필수 (Phase 4 진입 전)

- **Dockerfile multi-stage + layer caching 전략**: `COPY requirements.txt` 를 `COPY src/` 앞에 → requirements 안 바뀌면 pip layer cache, 코드 수정 시 rebuild 수 초.
- **NVIDIA Container Toolkit 설치 + `docker run --gpus all nvidia/cuda nvidia-smi` smoke test**.
- **docker-compose `profiles` + `env_file`**: DSR emulator `profiles: [dev]`, `.env` runtime 주입.
- **PulseAudio socket passthrough 실습**: `pactl list sources short` 마이크 인식 + 컨테이너에서 `parecord` smoke test.
- **ROS2 launch.py + colcon `--symlink-install`**: 컨테이너 entrypoint 의 `ros2 launch <pkg> <file>.launch.py`, dev 모드에서 코드 수정 즉시 반영.
- **custom message (`od_msg`) 빌드 파이프라인**: 같은 .msg 가 host + 두 container 에서 빌드되어 type hash 일치 확인 (`ros2 interface show od_msg/...`).

### 3-3-B. 권장 (구현 품질 / 디버깅 효율)

- **Fast-DDS vs CycloneDDS 차이**: 본 프로젝트가 둘 중 어느 것을 표준으로 정할지 — QoS / discovery latency 비교.
- **DDS QoS policy**: reliability (RELIABLE/BEST_EFFORT), durability (TRANSIENT_LOCAL/VOLATILE) — yolo 의 bbox stream 은 BEST_EFFORT, robot_command 는 RELIABLE 등.
- **PyTorch CUDA wheel 매칭**: cu118 / cu121 / cu126 / cu130 wheel 가용성 매트릭스. PyTorch wheel 가용 = CUDA 메이저 결정 입력.
- **bash defensive scripting**: `set -euo pipefail` 의 unbound var 함정, `${VAR:?missing}`, `trap ERR`.
- **컨테이너 디버깅**: `docker exec -it`, `docker logs --tail`, `ros2 topic hz <t>`, `ros2 node info`.

### 3-3-C. 선택 (운영 안정화)

- **non-root container user**: `USER 1000` — root 권한 최소화 + audio socket UID 매칭 자연.
- **healthcheck**: 컨테이너 self-check (예: `ros2 topic list` 가 자기 topic 포함), `docker ps` 로 외부 모니터.
- **vulnerability scanning**: `trivy image <image>:<tag>` — base image + pip 의존성 CVE 정기 점검.
- **CI 자동화**: pre-commit / GitHub Actions 로 shellcheck + secret grep + image scan. 현재는 수동 — 노트북 수 늘면 검토.
- **observability**: journald log driver, `docker stats`. 단일 노트북 scope 엔 과한 수준.

---

## 작성 컨벤션 (Notion 본문)

1. **내부 ID 인용 금지**: `Hard Rule #N`, `ADR-NNN`, `Phase 2-X`, `M1`/`M2` 등 본 레포 내부 축약어 금지. 그 룰의 substance / 사실만 서술.
2. **언어**: 한국어 설명 + 영어 식별자 (변수명 / 패키지명 / 명령어 / 환경변수).
3. **표 형식**: Notion `<table header-row="true">` — 기존 Section 1-2 / 2-3 스타일 일관.
4. **mention-page 부록**: page 끝 2개 mention 그대로 유지. Section 3 본문에서 "자세한 사항은 부록 참조" 1~2회 자연 언급.
5. **mermaid 미사용**: Section 3 은 텍스트 + 표 중심. Section 2 의 mermaid 와 시각 분리.
6. **코드 블록**: shell / yaml / dockerfile 언어 태그 명시.

---

## 작성 메커니즘 (Notion MCP)

대상 page URL = `https://www.notion.so/teamsparkx/2-Humble-Jazzy-36c563918e59805997c4e8e533067303`.

**현재 상태**: Section 3 본문 = `<empty-block/>` 1개. 부록 영역 (mention-page 2개 + `<empty-block/>`) 은 보존.

**삽입 방식**: `mcp__claude_ai_Notion__notion-update-page` 의 `insert_content`. 큰 payload 차단 사례가 있어서 청크 분할.

**청크 분할안 (3 chunk)** — concise 분량이라 이전 5 chunk → 3 chunk 로 축소:
1. Chunk 1: 3-1 통합 표 + 3-1-a cobot2_ws 매핑 표
2. Chunk 2: 3-2 7개 고려사항 단락
3. Chunk 3: 3-3 3 단계 학습 권장 bullet

각 청크 ~40줄 이하. 첫 청크 통과 시 나머지 동일 패턴.

**삽입 위치**: `insert_content` 의 `after_block_id` 옵션 — `# 3. 필요 기술 스택 및 고려사항` heading block 바로 뒤. 기존 `<empty-block/>` 는 첫 chunk 들어간 후 잔존해도 무해.

---

## Verification

1. **Notion fetch 재실행**: Section 3 본문에 표 2개 (3-1, 3-1-a) + 7개 단락 (3-2) + 3 단계 bullet (3-3-A/B/C) 모두 존재.
2. **내부 ID 인용 grep**: 응답 text 에서 `Hard Rule #`, `ADR-`, `Phase 2-/3-/4-X`, `M1`/`M2`/`M3` 패턴 = 0건.
3. **부록 보존**: page 마지막의 `### 부록` + 2개 mention-page 변경 없음.
4. **표 column count 검증**: 3-1 (3 column) / 3-1-a (3 column) row 의 cell 수 일치, backtick 코드 cell 깨짐 없음.
5. **사용자 read-through**: Notion app 에서 page 열어 시각 검토.

---

## 실행 시 의존성

- 사용자 plan 승인 후 실행 (현재 plan mode).
- 실패 시 rollback: Notion page version history 에서 직전 버전 restore.

---

## 변경 사항 (이전 안 대비)

| 항목 | 이전 (Comprehensive) | 현재 (Concise) |
|------|-------------------|---------------|
| 3-1 표 | 5개 (layer 별 분할) | **단일 통합 표 1개** + cobot2_ws 매핑 표 1개 |
| 3-2 단락 | 10개 | **7개** (audio/USB 통합, distro+매트릭스 통합, industry+platform 통합) |
| 3-3 bullet | 3 단계 × 7~13개 | 3 단계 × **4~6개** |
| cobot2_ws / corecode 반영 | 없음 | **3-1-a 표 + 3-2-4 / 3-2-5 / 3-2-7 본문에 자연 인용** |
| 청크 수 | 5 chunk | **3 chunk** |
| 총 분량 (Notion 본문 기준) | ~2-3 page | **~1 page** |

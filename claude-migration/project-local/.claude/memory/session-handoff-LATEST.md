# Session Handoff — LATEST

> 매 세션 종료 전 `/session-checkpoint` 로 갱신. 글로벌 `SessionStart` hook 이 자동 로드.
> Forward-looking only — 본 세션에서 한 일이 아니라 다음 세션이 할 일.

## Last updated
2026-05-28 — 배포 코드(`cobot2_ws.zip` / `corecode.zip`, repo 루트 untracked) 분석 + 노션 아키텍처 페이지(`36c563918e59805997c4e8e533067303`) 1-1/2-1/2-2/섹션3 검증·정정. **Phase 4 통신 아키텍처 결정 3건 확정** (RealSense=host realsense2_camera publish / robot_control=host / app 통신=ROS2 service 유지) → 상세·근거·코드분석은 `.claude/memory/project_phase4-container-architecture.md`. **여전히 M2 미진입** — 아래 Next Actions 그대로.
> 이전(2026-05-27): Phase 1 + M1 완료(헬퍼 5개), ADR-007/008/010, git baseline `v0.1.0`, 책임 분리 원칙 확정.

---

## Next Actions (priority order)

1. **M2 — System layer (a01 영역)**:
   - 신규 작성: `resources/{ros2-install,ros2-desktop-main,nvidia-driver-install,docker-install}.sh`
   - `a01-prerequirements.sh` 갱신 — `source resources/config.sh` + state 헬퍼 (step_begin/step_end_ok) + confirm_or_abort 호출
   - apt-mark hold: nvidia-driver-* / docker-ce / docker-ce-cli / containerd.io (MIGRATION_NOTES § 11)
   - keyring 일관성: 모든 외부 repo 키링을 `/etc/apt/keyrings/` 로 통일 (MIGRATION_NOTES § 3.2)
   - `apt-key add` (Gazebo, backup/ros2-install.sh:26) 제거 — `curl | gpg --dearmor` + `signed-by=`
   - reboot 직전 `confirm_or_abort` 호출 (Hard Rule #9)
   - 시작 명령: `bash backup/ros2-install.sh` 를 reference 로 참조하면서 `resources/ros2-install.sh` 신규 작성
2. **M3 진입 전 ADR-006 (CUDA 메이저 결정)** 필요:
   - Noble repo 에 `cuda-toolkit-12-4` 부재. 가용: 12-5 / 12-6 / 12-8 / 12-9 / 13-0 / 13-1 / 13-2
   - PyTorch wheel 가용성 검증: `curl -fsIL https://download.pytorch.org/whl/cu126/torch/` 등
   - config.sh 의 `CUDA_VERSION` 변수 채울 값 결정 (apt 패키지명 형식, 예: `12-6`)
3. **M3 — Compute / Robot / Camera (a02 영역)**:
   - venv 부트스트랩 **없음** (ADR-008). a02 첫 단계 = `cuda-keyring_1.1-1_all.deb` 다운로드 + `apt install cuda-toolkit-${CUDA_VERSION}`
   - DSR project clone `-b jazzy` (브랜치 활성 확인됨, commit `816ecb5d`)
   - RealSense: Noble 정식 librealsense2 분기 (Intel Artifactory repo, 활성 확인 2026-05-25)
4. **M4 — a06 정리 결정 (사용자 재질문 필요)**: host pip 단계 제거 (ADR-008) 후 a06 / python-dependency.sh 의 잔여 책임 — (a) 삭제 / (b) 최소 wrapper / (c) Phase 4 컨테이너 launcher
5. **M5 — install.sh 통합 + 원본 a01-a06 처리 결정**: (a) 삭제 / (b) backup/ 이동 / (c) thin symlink
6. **Phase 4 — 컨테이너 빌드 + 종단 검증 (ROADMAP endpoint)**:
   - 통신/배치 결정 3건 **확정됨** (RealSense host node / robot_control host / app 통신 service) — `project_phase4-container-architecture.md` 참조
   - 실제 작업점: `pick_and_place_voice` 마스터 패키지를 컨테이너별 분리 (yolo←object_detection / voice←voice_processing / host←robot_control). od_msg 는 host + yolo 컨테이너 양쪽 빌드
   - yolo 컨테이너 deps: PyTorch(cuXXX wheel) + ultralytics + opencv + cv_bridge (pyrealsense2 불필요). voice: langchain + openai + PyAudio + openwakeword(+tflite, 모델 COPY)
   - 빌드 시 RMW + ROS_DOMAIN_ID host/컨테이너 완전 일치, `network_mode: host`

---

## Open Decisions

- **ADR-006 (CUDA 메이저)**: 12-6 vs 12-8 vs 13-x 결정 (Noble repo 에 12-4 부재). PyTorch wheel 가용성으로 결정. M3 진입 전 필수.
- **두 `dsr-project-install*.sh` 통합 여부**: `_25` variant 가 dead code 인지. 현재 `backup/dsr-project-install_25.sh` 보존 상태. M3 진입 시 사용자에게 재질문.
- **Phase 4 디자인 결정 (진입 시) — 별도 ADR 후보**:
  - (a) base image: `ros:jazzy-ros-base-noble` (CPU+ROS2, pip torch wheel 이 CUDA runtime 동봉 → GPU 가능) vs `nvidia/cuda:*-runtime-ubuntu24.04` + ROS2 수동. **여전히 open** (섹션3은 ros base 가정)
  - (b) ROS2 통신 network mode: `network_mode: host` 권장 (DDS multicast 자연, 단일 노트북 표준) — 진입 시 최종 확정
  - ~~(c) install.sh 끝단 자동 호출~~ → **분리 유지 (2026-05-27)**. pull-first 분기는 wrapper (`containers/up.sh` 가칭)
  - ~~(d) Docker Hub publish + secret hygiene~~ → **ADR-007 (2026-05-27) 결정 완료**
  - ~~(e) app 통신 모델 (service vs topic)~~ → **service 유지 (2026-05-28)**. 코드가 `/get_3d_position`(od_msg/SrvDepthPosition) + `/get_keyword`(Trigger) serve, robot_control 이 client
  - ~~(f) RealSense 처리~~ → **host realsense2_camera publish + yolo 컨테이너 subscribe (2026-05-28)**. 컨테이너 pyrealsense2/USB passthrough 불필요
  - ~~(g) robot_control 위치~~ → **host 유지 (2026-05-28)**. DSR_ROBOT2(TCP 12345) + RG2(Modbus 502) 실기 직접 접근
  - (h) **신규 open**: colcon ws 이름 `cobot_ws` vs `cobot2_ws` 표준 미정 (노션 3-1 표 vs 다이어그램 불일치). M2 진입 전 정하면 깔끔
  - (i) **신규 open**: `pick_and_place_voice` 마스터 패키지 컨테이너별 분리 방식 — Dockerfile/colcon 실제 작업점
- **M4 a06 잔여 책임 결정**: 삭제 / 최소 wrapper / Phase 4 launcher 중 — M4 진입 시 사용자에게 재질문.
- **M5 원본 a01-a06 처리**: 삭제 / backup/ 이동 / thin symlink 중 — M5 진입 시 사용자에게 재질문.

---

## Remaining Issues

(없음 — 활성 버그/blocker 없음. M1 검증 모두 PASS.)

---

## Context Notes

### 본 세션 확립된 새 사실 (Phase 2~4 작업 시 전제)

- **doosan-robot2 controller 통신** = **TCP socket port 12345 via DRFL** (Doosan Robot Framework Library, libpoco 의존). DDS 아님 — DDS 는 ROS2 노드 간 (DSR_NODE ↔ T_STATE/T_CMD) 통신.
- **doosan-robot2 jazzy 브랜치 commit**: `816ecb5d` (2026-05-26 확인). humble 은 `ec924254` (노션 검증본 commit 과 일치).
- **Noble apt repo 가용 CUDA**: 12-5 / 12-6 / 12-8 / 12-9 / 13-0 / 13-1 / 13-2. **12-4 부재** → 단순 URL 갱신으로 끝나지 않음, 메이저 변경 결정 필요.
- **librealsense2 Noble repo**: Intel Artifactory `https://librealsense.intel.com/Debian/apt-repo/dists/noble` 활성, last-modified 2026-05-25.
- **ROS2 jazzy Noble Packages.gz**: 1.87MB, last-modified 2026-05-01. 활성.
- **Voice 입력 소스**: 노트북 내장 마이크 (USB 외장 아님). PulseAudio/PipeWire socket mount 결정.
- **DSR emulator 사용 모드**: 실기 우선, emulator 거의 미사용 → docker-compose `profiles: [dev]` 격리.

### 배포 코드 분석 (2026-05-28) — Phase 4 전제

- **`cobot2_ws.zip` / `corecode.zip`** = repo 루트에 존재 (untracked). 압축 해제 분석 완료. 상세는 `.claude/memory/project_phase4-container-architecture.md`.
- `cobot2_ws` = colcon ws. `pick_and_place_voice` 가 마스터 패키지 (object_detection + voice_processing + robot_control 묶음, entry_points 정의). `od_msg` = CMake srv 패키지.
- 데이터 흐름: 내장 mic→PyAudio→openwakeword(tflite,로컬)→Whisper(STT)→gpt-4o(langchain)→`/get_keyword` → robot_control 이 키워드별 `/get_3d_position` 호출 → `T_gripper2camera.npy` camera→base 변환 → movej/movel + RG2.
- `corecode` = 컨테이너 비적재 (개발/연구용: Calibration→`T_gripper2camera.npy`, OD_Tutorial→`yolov8n_tools_0122.pt`, VoiceProcessing standalone, DRL 노트북).
- 실재 토픽은 `/camera/camera/*` 와 `/dsr01/joint_states` 둘뿐 (노션의 `/detect/bbox`·`/voice/text`·`/robot_command` 은 실재 안 함 → service 로 정정 완료).

### Phase 2 / M2 진입점 (humble 파일은 모두 `backup/` 에 있음)

- `backup/ros2-humble-desktop-main.sh:12,14` — `CHOOSE_ROS_DISTRO=humble`, `TARGET_OS=jammy`
- `backup/ros2-install.sh:10,16,19-22,26,29` — `ros-humble-*` 약 17개 + Gazebo `apt-key add`
- `backup/cuda-pytorch-install.sh:3-7` — `ubuntu2204` 5건, CUDA 12.4 local installer (Noble 부재 → ADR-006)
- `backup/docker-install.sh:22` — Docker jammy-pin (`5:23.0.6-1~ubuntu.22.04~jammy`)
- `backup/dsr-project-install.sh:4,9-23,38,39` — clone 브랜치 + 패키지 15개 + ros distro path
- `a05-realsense02.sh:1,11` — `ros-humble-realsense2-*`, `ROS_DISTRO=humble` (top-level, 미이동)
- `a06-Voice.sh:5-6` — numpy 1.24.4 force-reinstall (ADR-002 + ADR-008 영향, 호스트 pip 제거)
- top-level `a01-a06.sh` 는 일시 깨진 상태 (호출 대상이 backup/ 으로 이동됨) — M2 진입 시 a01 신규 작성으로 해소

### M1 산출 (현재 사용 가능)

- `resources/config.sh` — distro/OS 강제 export (jazzy/noble), 단일 진실 소스
- `resources/state.sh` — `step_begin <n> <total> <name>` / `step_end_ok` / `step_end_fail` / `step_should_skip` / `state_dump`
- `resources/confirm.sh` — `confirm_or_abort "msg"`, `ASSUME_YES=1` override
- `resources/env-load.sh` — `_load_env <file>` (source 안 함, 안전 파싱), `_require_env`
- `resources/activate.sh` — non-interactive 셸용 ROS2 source 만 (venv 라인 제거됨, ADR-008)
- shellcheck `-x` exit 0 검증 완료. shellcheck 시스템 설치됨 (apt).

### git 운영 (2026-05-27 시작, ADR-010)

- `git init -b main` 완료. remote 없음 (`git remote -v` = 빈 출력, 유지 필수).
- baseline annotated tag `v0.1.0` 부착 — 7 분할 commit 위에. M2 작업 시 destructive 동작 전 안전망 commit 활용 가능.
- commit/tag 메시지 정책: 외부 사람이 봐도 이해 가능. 내부 마일스톤 (M1/M2…) / 결정 기록 번호 (ADR-NNN) / 단계 (Phase N) / 룰 ID (Hard Rule #N) 미사용. 한국어 회화 + 영어 식별자 혼용.
- 실험적 변경은 short-lived feature branch (`migrate-system-layer` 등) → fast-forward merge.

### 책임 분리 원칙 (2026-05-27 사용자 결정)

- `bash install.sh` = host PC layer only (NVIDIA driver / Docker CE / ROS2 jazzy / CUDA toolkit / DSR colcon / RealSense SDK).
- `docker compose pull && docker compose up -d` = application layer (yolo / voice 컨테이너) — 사용자 별도 명령.
- 신규 노트북 setup 흐름이 명시적 2-step. install.sh 가 컨테이너 layer 호출 안 함.
- 영향: a06 잔여 책임은 최소 wrapper 로 좁아짐 (`.env` placeholder 검증 + Docker Hub login 안내만). 자세한 사항은 ROADMAP 2-10 / 2-15 / Phase 4 도입부.

### 작업 참조용 노션 페이지 (2026-05-27 작성)

- "Docker 컨테이너에서 코드를 실행하는 구조와 코드 전달 방법" — `36d563918e59803da5b9c761da8a05d0`
- "host PC ↔ 컨테이너, 컨테이너 ↔ 컨테이너 통신 구조 — 일반 소켓 vs DDS" — `36d563918e5980f8866ccfe64de99119`
- Phase 4 진입 시 Dockerfile / docker-compose.yml 작성 시 참조.

### 함정 (다시 발생하면 다음 세션 피하기)

- mermaid edge label 안에 `--` (shell flag) 들어가면 화살표 토큰 오인 → pipe 문법 `-->|label|` 사용 (lessons L-002)
- 큰 페이로드 Notion `insert_content` → Cloudflare WAF 차단 → 5~7개 청크 분할 (lessons L-001)
- config.sh 의 distro/OS 핀은 `:=` 가 아니라 `export X=Y` 강제 — 사용자 셸이 humble 로 오염되어도 강제 jazzy 적용
- shellcheck SC1090/SC1091 false-positive (동적 source 경로) → 파일 상단 `# shellcheck source-path=SCRIPTDIR` + 동적 라인 위 `# shellcheck disable=SC1090,SC1091`

---

## Current Focus

- **Top priority**: M2 진입 — `a01-prerequirements.sh` 신규 작성 + 4 resource 파일 (`ros2-install`, `ros2-desktop-main`, `nvidia-driver-install`, `docker-install`) 신규 작성. backup/ 의 humble 파일들이 참조 기준.
- **Friction**: M3 진입 전 ADR-006 (CUDA 메이저) 결정 필요 — Noble repo 가용 매트릭스는 fetch 완료, PyTorch wheel 매칭은 별도 검증 필요.

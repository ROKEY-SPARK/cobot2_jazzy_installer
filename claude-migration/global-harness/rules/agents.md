# Agent Orchestration

## Available Agents

| Agent | Does | Does NOT | Hands off to |
|-------|------|----------|-------------|
| `rl-theory-verifier` (project) | 새 알고리즘 구현 후 `docs/concepts/<algo>.md`와 코드를 대조해 이론적 누락 / 불일치 검출 | 코드 수정, 새 알고리즘 작성, 학습 실행 | 이슈 발견 시 사용자에게 보고 (사용자가 수정) |
| `robot-safety-verifier` (project) | Smartwarehouse 정적 검증 — motion limits (`vel`/`acc`), `REAL_ROBOT` 게이팅, 하드코딩 좌표 / 경로, `best.pt` 덮어쓰기 검출. severity (CRITICAL/IMPORTANT) 부여 | 코드 수정, 동적 시뮬레이션, DSR API 디버깅 | CRITICAL → 사용자 / orchestrator correction loop, IMPORTANT → verification 단계로 |
| `notebook-colab-verifier` (project) | ai_advanced_lecture 노트북 정적 검증 — Colab T4 호환성 (VRAM, 시스템콜, apt 호출), hardcoded `/home/`·`/Users/`·`C:\` 경로, 하드코딩 `.cuda()` / `.to('cuda')`, 학생 배포본 정답 셀 노출 검출. severity (CRITICAL/IMPORTANT/MINOR) 부여 | 코드 수정, 실제 Colab 실행, 데이터셋 다운로드 | CRITICAL → 사용자 / orchestrator correction loop, MINOR → verification 단계로 |
| `orchestrator` (global) | 플랜 실행 추적, gate 순서 강제 (`code-reviewer → verification → done`), 리포트 검증 | 코드 작성, 플랜 수정, 자동 drift 수정 (Light) | gate 실패 → 사용자, 단순 미완료 → 구현자 재호출 |
| `code-reviewer` (global) | 구조적 코드 리뷰 — Hard Rules 1-6, CLI 도메인 (idempotent / exit code / dry-run), 글로벌 Tier 0 (secrets / AI attribution / LLM API) 검사. severity 등급 (CRITICAL/IMPORTANT/MINOR) 부여 | 코드 직접 수정, 새 코드 작성 | CRITICAL 발견 → orchestrator correction loop, MINOR → verification 통과 |
| `verification` (global) | 완료 선언 전 mandatory checklist — tests pass, diff scope, hygiene, Hard Rules, reproducibility | 버그 수정, 코드 변경 | 실패 → 구현자, Hard Rule 위반 → BLOCK + 사용자 |

위치:
- `rl-theory-verifier`: `/home/rokey/RL_study/.claude/agents/rl-theory-verifier.md` (RL_study 레포 전용 — 본 프로젝트 무관)
- `robot-safety-verifier`: `/home/rokey/Smartwarehouse/.claude/agents/robot-safety-verifier.md` (Smartwarehouse 레포 전용)
- `notebook-colab-verifier`: `/home/rokey/ai_advanced_lecture/.claude/agents/notebook-colab-verifier.md` (ai_advanced_lecture 레포 전용)
- `orchestrator` / `code-reviewer` / `verification`: `/home/rokey/.claude/agents/` (global, 모든 프로젝트 공통)

## Routing Rules

### RL theory (RL_study 레포 전용)
- 새 알고리즘 구현이 끝나거나 사용자가 "이론과 일치하는지 확인" / "검증" / "이론 대조" / "verifier" 같은 키워드로 요청할 때 → `rl-theory-verifier` 호출.
- 단순 syntax / lint 이슈는 verifier가 다루지 않는다 — 이론적 정확성만.
- 검증 대상이 docs/concepts/에 개념 노트 없이 코드만 있을 경우 → 노트 작성을 먼저 권장하고 verifier 호출 거부 (Hard Rule #9 일치).

### Robot safety (Smartwarehouse 레포 전용)
- `movel`/`movej`/`DSR_ROBOT2` 호출이 포함된 코드 변경 직후 → `robot-safety-verifier` 호출.
- 사용자 키워드: "로봇 안전 검증", "motion 검증", "robot-safety", "real robot 배포 전 점검" → 호출.
- CRITICAL severity (motion limits 누락 / `REAL_ROBOT` 게이팅 누락) → 즉시 BLOCK + 사용자 보고. 절대 우회 금지.
- `code-reviewer`와 직교 — robot-safety-verifier는 도메인 정적 분석, code-reviewer는 일반 구조 검토.

### Notebook Colab compatibility (ai_advanced_lecture 레포 전용)
- `*.ipynb` 또는 노트북에서 import 되는 `*.py` 모듈을 수정한 직후 → `notebook-colab-verifier` 호출.
- 사용자 키워드: "Colab 호환성 검증", "notebook 검증", "colab-compat", "학생 배포 전 점검", "노트북 검증" → 호출.
- CRITICAL severity (Colab T4 비호환 / hardcoded `/home/` 경로 / 하드코딩 `.cuda()`) → 즉시 BLOCK + 사용자 보고.
- `code-reviewer` 와 직교 — notebook-colab-verifier 는 노트북 셀 단위 정밀 정적 분석, code-reviewer 는 일반 구조 + ML Lecture profile 일괄 검사. 두 에이전트가 같은 5규칙을 다른 입자 (granularity) 로 본다.
- 검사 제외 경로: `solutions/`, `_instructor_only/`, `docs/`, `original_reference/`.

### Orchestration (모든 프로젝트)
- 다단계 구현 / 플랜 실행 / "task 진행 상황" / "gate 통과했는지" / "done 선언해도 되나" → `orchestrator` 호출.
- 단일 파일 사소한 수정은 orchestrator 우회 가능 (overhead 비용 > 가치).
- orchestrator 호출 후 gate 순서는 immutable: `[implement] → code-reviewer → verification → done`.

### Code review (모든 프로젝트)
- 코드를 작성·수정한 직후, 또는 사용자가 "리뷰해줘" / "review" / "코드 점검" 키워드 → `code-reviewer` 호출.
- code-reviewer는 절대 코드를 수정하지 않는다 — 발견·등급·제안만.
- CRITICAL severity 발견 시 orchestrator의 correction loop로 위임. MINOR만 있으면 verification으로 진행.

### Verification (모든 프로젝트)
- "done", "완료", "마무리", "merge 준비" 키워드 또는 모든 코드 변경 직후 마지막 단계 → `verification` 호출.
- verification 통과 없이 done 선언 금지 (Tier 1).
- Hard Rule 위반 발견 시 즉시 BLOCK + 사용자 보고.

## Project Profiles (conditional rule activation)

글로벌 에이전트 (`code-reviewer`, `orchestrator`)에 도메인별 규칙 블록이 있고, 각 블록은 프로젝트가 명시적으로 opt-in해야 발동한다. opt-in 방법은 마커 파일 생성:

```bash
mkdir -p .claude/profiles
touch .claude/profiles/<profile-name>.md   # 파일 존재 = 활성
```

현재 정의된 프로파일:

| Profile marker | 활성 시 적용되는 검증 |
|----------------|--------------------|
| `.claude/profiles/cli-automation.md` | exit code 규율, --dry-run, help text, idempotency |
| `.claude/profiles/isaac-sim-worldgen.md` | USD World Generation 6개 Hard Rules (manifest, SI 단위, Visual/Collision 분리 등) |
| `.claude/profiles/ml-lecture.md` | PyTorch + Jupyter 강의 자료 — Colab T4 호환성, hardcoded 경로, device 분기, lecture/solution 분리, beginner-readable code |

**기본값**: 마커 없음 → 도메인 블록 스킵, universal check만 수행.
이 컨벤션은 false-positive CRITICAL이 무관한 프로젝트의 머지를 막는 사고를 방지한다.

프로파일 추가 시: 글로벌 에이전트 파일에 새 블록을 "Activation gate: `.claude/profiles/<name>.md` exists" 형식으로 추가하고 이 표에도 등록.

## Tier Priorities
- **Tier 0**: Hard Rules — immutable, no agent can override. ai-constitution.md IV절 참조.
- **Tier 1**: 코드 변경 후 verification, experiment logging.
- **Tier 2**: 다중 시드 평가, feature 영향 점검.
- **Tier 4**: Style (append-only logs, notebook 셀 구조).

상위 tier가 항상 우선. 동일 tier 충돌 시 더 보수적인 옵션 선택.

## Voice Guidelines

**Agent → User:**
- 결과 먼저, 설명은 그 다음 (결론 → 근거 → 다음 단계).
- 불확실하면 "unknown"이라 명시 — 추측 금지.
- 코드 블록은 변경된 부분만 (전체 파일 출력 금지).

**Agent → Agent (subagent dispatch):**
- 프롬프트에 전체 컨텍스트 포함 (파일 읽기 위임 금지).
- 절대 경로만 사용.
- 상태 코드 반환: `DONE | DONE_WITH_CONCERNS | NEEDS_CONTEXT | BLOCKED`.
- **Return compression 규칙**: 압축된 요약 + 상태 코드만. raw output, full file contents, verbose execution logs 절대 반환 금지. 깊은 검색 결과는 핵심 발견만.

**금지 패턴:**
- Sycophantic openers ("좋은 질문입니다!", "물론이죠!")
- Closing filler ("도움이 되었기를", "추가 질문 있으시면 알려주세요")
- 과도한 이모지

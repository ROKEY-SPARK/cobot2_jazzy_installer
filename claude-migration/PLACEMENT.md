# Claude 데이터 마이그레이션 — 배치 가이드

이 번들은 `ros2_jazzy_test` 프로젝트의 Claude Code 컨텍스트를 새 Ubuntu 24.04 머신으로 옮기기 위한 것. 아래 4개 묶음을 각 위치에 풀면 새 머신의 Claude 가 동일 컨텍스트로 이어서 작업한다.

> 생성 시점 환경: user=`rokey`, 프로젝트 경로=`/home/rokey/ros2_jazzy_test`.

## 배치 위치

| 번들 폴더 | 새 머신 배치 위치 | 비고 |
|-----------|------------------|------|
| `project-local/.claude/` | `<프로젝트>/.claude/` | 프로젝트 폴더와 함께 이동하면 이미 제자리. 별도 이동 시에만 복사 |
| `global-project-data/` | `~/.claude/projects/-home-rokey-ros2-jazzy-test/` | **경로 키 주의** (아래 참조). memory + 세션 transcript |
| `global-plan/*.md` | `~/.claude/plans/` | 진행 중 plan 파일 |
| `global-harness/` | `~/.claude/` (rules / agents / skills / settings.json) | **모든 프로젝트 공통** AI 설정. merge 주의 (아래) |

## 적용 예시 (user=rokey, 동일 경로 가정)

```bash
# 0. 프로젝트 폴더를 새 머신 ~/ros2_jazzy_test 로 옮긴 뒤 (이 번들 포함)
cd ~/ros2_jazzy_test/claude-migration

# 1. 프로젝트 로컬 (폴더째 옮겼으면 생략 가능)
cp -a project-local/.claude ~/ros2_jazzy_test/

# 2. 글로벌 프로젝트 데이터 (memory + transcript)
mkdir -p ~/.claude/projects/-home-rokey-ros2-jazzy-test
cp -a global-project-data/. ~/.claude/projects/-home-rokey-ros2-jazzy-test/

# 3. plan
mkdir -p ~/.claude/plans && cp -a global-plan/*.md ~/.claude/plans/

# 4. 글로벌 harness (아래 merge 주의 먼저 읽을 것)
cp -a global-harness/rules global-harness/agents global-harness/skills ~/.claude/
# settings.json 은 덮어쓰기 대신 비교 후 병합 권장
```

## ⚠️ 주의 2가지

### 1. auto-memory 경로 키
`~/.claude/projects/` 아래 폴더명 `-home-rokey-ros2-jazzy-test` 는 **프로젝트 절대경로**(`/home/rokey/ros2_jazzy_test`)에서 파생된 키다. 새 머신의 user 명이나 경로가 다르면 (예: `/home/ubuntu/...`) Claude 가 이 폴더를 자동으로 못 찾는다.
- 동일 경로(`/home/rokey/ros2_jazzy_test`) → 그대로 동작.
- 다른 경로 → 폴더명을 새 경로 기준으로 변경. 규칙: 절대경로의 `/` 를 `-` 로 치환 (앞 `/` 포함). 예: `/home/ubuntu/ros2_jazzy_test` → `-home-ubuntu-ros2-jazzy-test`.

### 2. `settings.json` 병합
`global-harness/settings.json` 은 사용자 글로벌 설정 전체다. 새 머신에 이미 `~/.claude/settings.json` 이 있으면 **통째로 덮어쓰지 말 것** — 두 파일을 비교해 필요한 키만 병합. (secret 점검 완료 — 자격증명 없음.)

## 번들 내용 요약

- **project-local/.claude/**: `memory/` (MEMORY.md, session-handoff-LATEST.md, project/feedback 노트), `profiles/cli-automation.md`, `settings.local.json` (권한 allowlist)
- **global-project-data/**: `memory/` (auto-memory: MEMORY.md, feedback-prefer-concise.md), 세션 transcript `*.jsonl` (원본 대화 로그, 약 8.8M — 핵심 맥락은 handoff/memory 에 이미 증류됨), subagent 로그
- **global-plan/**: Section 3 작성 plan
- **global-harness/**: `rules/` (ai-constitution, output-style, agents), `agents/` (orchestrator, code-reviewer, verification), `skills/`, `settings.json`

## Secret 점검 결과 (포함 전 확인 완료)

- `.env` 실파일 없음 (`.env.example` 템플릿만 프로젝트에 존재, 번들엔 미포함).
- transcript `.jsonl` 의 `OPENAI_API_KEY=` / `sk-` 매치는 전부 placeholder·템플릿 `${...}`·예시값(`sk-test-abc123`)·논의된 정규식 문자열 → 실제 key 아님.
- `settings.json` / `settings.local.json` / `rules` / `agents` / `skills` 내 평문 자격증명 없음.

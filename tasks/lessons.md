# tasks/lessons.md — AI 행동 교정 규칙

> 반복 실수가 발생하면 여기에 기록 → 다음 세션 SessionStart hook이 자동 노출.
> 작성 패턴:
> - **Symptom**: 어떤 잘못된 행동이 반복되었는가
> - **Why it happened**: 모델이 그 행동을 한 추정 원인
> - **Correction**: 다음부터 어떻게 행동해야 하는가
> - **Trigger**: 이 교정이 발동되어야 하는 상황 (키워드 / 작업 종류)

---

## L-001: Notion `insert_content` 큰 페이로드 → Cloudflare 차단

- **Symptom**: 한 번에 ~10KB+ 크기의 markdown 을 `mcp__claude_ai_Notion__notion-update-page` `insert_content` 로 보내면 Cloudflare 가 차단 응답 (HTML "Sorry, you have been blocked", Ray ID 동반).
- **Why it happened**: 페이로드 크기 + 일부 WAF 트리거 패턴 (shell 명령 라인 / `<script>` 유사 토큰 등) 의심.
- **Correction**: 5-7개 섹션으로 분할하여 `insert_content` 를 순차 호출 (각 청크 ~50줄 이하 권장). 첫 청크가 통과하면 나머지도 같은 호출 패턴으로 안전.
- **Trigger**: "노션에 보고서 만들어줘", "notion 페이지에 정리", "notion 페이지에 넣어줘", `mcp__claude_ai_Notion__notion-update-page` 의 `insert_content` 호출.

---

## L-002: Mermaid edge label 안의 `--` → 화살표 토큰으로 오인 → 구문 오류

- **Symptom**: 노션 / GitHub 의 mermaid 렌더러가 다음 같은 라인에서 "syntax error" 표시:
  ```
  SRC -- docker compose build --pull --> BUILD
  ```
  사용자 의도: `--` 사이의 문자열 ("docker compose build --pull") 이 edge label. 그러나 라벨 안의 `--pull` 의 `--` 가 mermaid parser 에게 화살표 시작 토큰으로 보임 → edge 정의가 깨짐.
- **Why it happened**: Mermaid 의 `A -- text --> B` 문법은 양쪽 `--` 를 edge 경계로 인식. 라벨 자체에 `--` 가 들어가면 parser 가 어디서 라벨이 끝나는지 헷갈림. CLI flag (`--pull`, `--force`, `--no-deps` 등) 이 흔한 trigger.
- **Correction**: **pipe 문법으로 작성**: `A -->|text with --| B`. pipe `|` 가 명시적 경계라 라벨 안에 `--` 가 자유롭게 들어감. 또는 라벨에서 `--` 를 단일 `-` / em-dash (`—`) 로 변경해 회피 (단 의미 손실 가능).
- **Trigger**: Mermaid flowchart 의 edge label 에 shell 명령 / CLI flag / 옵션 (`--pull`, `--force`, `--no-deps`, `--system-site-packages`) 을 넣을 때. 노션에 mermaid 다이어그램 작성 / GitHub README 의 flowchart 작성 시.

---

## L-003: 다중 편집 Notion 페이지 → `update_content` 직전 재-fetch 필수

- **Symptom**: 세션 초반에 fetch 한 페이지 내용으로 `update_content` 의 `old_str` 을 구성했는데, 그 사이 다른 담당자가 섹션을 새로 채워 넣어 (예: `<empty-block/>` → 충실한 표) old_str 매칭 실패 (`No matches found`). 매칭 실패가 안 났다면 남의 작업을 덮어쓸 뻔함.
- **Why it happened**: Notion 페이지는 공동 편집 대상. 세션 시작 시점 fetch 는 스냅샷이라 시간이 지나면 stale. 담당자 2명 이상이면 분 단위로 내용이 바뀜.
- **Correction**: 공유 Notion 페이지를 `update_content` / `replace_content` 로 수정하기 직전 **항상 재-fetch** 하여 현재 상태로 old_str 을 재구성. 특히 빈 섹션(`<empty-block/>`)을 채울 때는 누가 이미 채웠는지 확인 후 진행 — 덮어쓰기 대신 surgical 정정.
- **Trigger**: 담당자 2명 이상인 Notion 페이지 수정, 세션 중반 이후의 `mcp__claude_ai_Notion__notion-update-page` `update_content`/`replace_content`, `<empty-block/>` 를 콘텐츠로 교체하려는 경우.

---

## 예상 후보 (발생 시 정식 항목으로 승격)

본 프로젝트가 Phase 2 마이그레이션 단계에서 실제 작업이 진행되면, 발견되는 반복 실수를 이 파일에 누적.
- distro 문자열을 하드코딩으로 다시 박는 경우 (Hard Rule #1 위반)
- `pip install` 을 numpy 재핀 단계 **앞**에 추가하는 경우 (ADR-002 위반)
- Docker 이미지를 무태그 또는 `latest` 로 두는 경우 (Hard Rule #6 위반)
- bash 스크립트 최상단에 `set -euo pipefail` 누락 (Hard Rule #5 위반)

# Output Style

## Conversation Language
- 사용자와의 대화는 **한국어**로 한다.
- 코드 / 식별자 / API 호출은 **영어** (변수명 / 함수명 / 클래스명 / 파일명).
- 한국어 변수명 / 함수명 금지.

## Notebook Markdown
- `.ipynb` 마크다운 셀: **한국어 설명 + 영어 용어 혼용**.
- 핵심 RL 용어는 영어 그대로 사용 후 첫 등장 시 한국어 부연 (예: "Bellman equation (벨만 방정식)").
- 수식은 LaTeX (`$...$` / `$$...$$`).
- 셀 단위 마크다운 헤더 (`##`, `###`)로 섹션 구분.

## Documentation (`docs/concepts/`, `docs/references/`)
- 한국어 설명 + 영어 용어. 한국어 단독 / 영어 단독 모두 지양.
- 인용 시 출처 명시 (논문 / 책 / URL + 챕터·페이지).

## Internal Harness References (문서 / 주석 / 노트북 공통)
- 문서, 코드 주석 (docstring / inline comment), 노트북 마크다운 셀에 harness 내부 룰을 **직접 인용하지 않는다**.
- 금지 예: "Hard Rule #4 — git 추적 금지", "Tier 0 #3 위반", "see `.claude/rules/ai-constitution.md`", "ai-constitution.md IV절".
- 이유: 룰은 재정렬 · 추가 · 삭제되어 번호와 위치가 시간에 따라 변한다. 문서 / 주석은 silently stale 되고, 학습 노트(예: Notion paste)는 외부 컨텍스트가 끊긴 채 의미를 잃는다.
- 대안: 그 룰의 **원인 또는 사실 자체**를 직접 서술한다.
  - ❌ "git 추적 금지 (Hard Rule #4)" → ✅ "이 디렉토리는 colcon 산출물이라 재생성 가능 — 소스만 추적".
  - ❌ "Hard Rule #5 위반" → ✅ "`package.xml` 의존성 누락은 타 환경 빌드 실패로 직결".
- 예외: `.claude/` 내부 룰 파일끼리의 cross-reference 는 허용 (시스템 자체 일관성). CLAUDE.md → ai-constitution.md 같은 단일 참조 링크도 허용.

## Response Style
- 결과 먼저, 추론은 그 다음 (Conclusion-first).
- 짧고 직접적으로. Filler 금지: "도움이 되었기를", "혹시 추가 질문이...", "Great question!"
- 불확실하면 "unknown" 명시 — 추측한 내용을 사실처럼 단정 짓지 않음.
- 코드 변경은 diff 또는 변경된 부분만 표시 — 전체 파일 출력 금지 (사용자 요청 시 예외).
- 이모지 절제. 사용자 명시적 요청 시에만 사용.

## Shell Command Formatting
- Bash / sh 명령어에서 **줄바꿈 continuation 은 백슬래시 하나 `\`** — `\\` 아님.
- `\\` 는 리터럴 백슬래시 1개로 해석되어 명령어가 깨진다. 줄 끝의 단일 `\` 만 다음 줄로 이어주는 line continuation.
- 사용자가 그대로 복사해서 실행할 수 있어야 한다. 마크다운 / 문서 / 채팅 메시지 / 코드 블록 모두 동일.
- ❌ `ros2 pkg create my_pkg \\` `--build-type ament_python \\` `--dependencies rclpy`
- ✅ `ros2 pkg create my_pkg \` `--build-type ament_python \` `--dependencies rclpy`
- 단일 라인으로 충분한 명령은 line continuation 자체를 쓰지 않는다.

## Explanations
- 사용자는 ML 석사 수준 배경 보유 — 기초 ML 개념(loss, gradient, network architecture) 설명 생략 가능.
- RL 도메인 개념은 입문 단계라 가정해 자세히 설명. 단, 한 번 설명한 개념은 반복하지 않는다 (memory / 이전 노트 참조).

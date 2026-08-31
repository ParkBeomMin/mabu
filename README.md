# claude-harness

Claude Code에서 도메인 맞춤 **에이전트 하네스**(에이전트 정의 + 스킬 + 오케스트레이션)를
설계·생성·진화시키는 메타 스킬. `/harness 결제 모듈 리뷰 하네스 만들어줘`처럼 쓴다.

[revfactory/harness](https://github.com/revfactory/harness)의 방법론을 이어받아
**Claude Code v2.1.178+ 실행 계층에 맞게 전면 재작성**한 버전이다.

## 원본과 다른 점

| | 원본 | 이 버전 |
|---|---|---|
| 기본 실행 모드 | 에이전트 팀 (`TeamCreate` — v2.1.178에서 제거됨) | **Workflow** 결정론적 오케스트레이션 (`agent()`/`pipeline()`/`parallel()`) |
| 팀 모드 | 무조건 최우선 | 조건부 — 환경 감사에서 자동 팀 형성이 켜져 있을 때만 |
| 모델 배정 | 전 에이전트 `opus` 강제 | 작업 성격별 model/effort (기본 `inherit`) |
| 오케스트레이터 | 스킬 산문으로 흐름 기술 | 진입 스킬 + `workflows/*.workflow.mjs` 2계층 — 흐름은 코드가 강제 |
| 신기능 | — | 에이전트 `skills` 사전 로드·`isolation: worktree`·`memory`, 스킬 `context: fork`·`allowed-tools` 반영 |
| 예시 | 가상 팀 예시 | 실운영 하네스 2종 (생성-검증 루프 / 팬아웃 파이프라인) |

핵심 관점: **판단은 에이전트에, 흐름은 스크립트에.** "누가 다음에 뭘 하나"를 모델 재량에
두면 실행마다 결과가 바뀐다. Workflow가 순서·병렬·루프 상한·산출물 스키마를 강제하고,
모델은 각 단계 안의 판단만 맡는다.

## 설치

```bash
git clone https://github.com/ParkBeomMin/claude-harness
cp -r claude-harness/skills/harness ~/.claude/skills/   # 심볼릭 링크 말고 복사
```

## 구조

```
skills/harness/
├── SKILL.md                          # Phase 0~7 워크플로우 (308줄)
└── references/
    ├── agent-design-patterns.md      # 실행 모드·패턴·에이전트 frontmatter 전체
    ├── orchestrator-template.md      # Workflow 스크립트 + 진입 스킬 템플릿
    ├── skill-writing-guide.md        # description·본문·스킬 frontmatter
    ├── skill-testing-guide.md        # with/without 비교, 트리거 검증
    ├── qa-agent-guide.md             # 경계면 교차 비교 QA
    └── harness-examples.md           # 실전 예시 2종
```

## 라이선스

Apache-2.0. 원본 저작자 표시는 [NOTICE](NOTICE) 참조.

# 기성 스킬 조달 — 만들기 전에 물려라

하네스가 스킬을 새로 쓰는 것은 **조달이 실패한 뒤의 일**이다. 사용자 환경에는 이미
검증된 스킬 생태계(플러그인·전역 스킬)가 깔려 있고, 에이전트 정의의 `skills:` 필드가
그것들을 물려주는 공식 통로다. 직접 그린 근육맵을 MIT 라이브러리 발견 후 갈아엎는 일을
스킬에서 반복하지 마라.

## 목차
1. 인벤토리 — 무엇이 깔려 있나 확인
2. 연결 방법과 함정
3. 작업 유형 → 기성 스킬 매핑
4. 역할 충돌 정리
5. 이식성 — 다른 머신에서 열 때

## 1. 인벤토리

Phase 0 감사에서 세 곳을 훑는다:

| 위치 | 무엇 | 참조 형식 |
|------|------|----------|
| `~/.claude/skills/` | 사용자 전역 스킬 | `skill-name` |
| `프로젝트/.claude/skills/` | 프로젝트 스킬 | `skill-name` (동명이면 스코프 규칙 적용) |
| 플러그인 (`~/.claude/plugins/installed_plugins.json`) | 예: superpowers | `plugin-name:skill-name` |

인벤토리 없이 설계하면 두 가지 실패가 나온다 — 이미 있는 스킬의 열화판을 새로 쓰거나,
없는 스킬을 전제한다. 둘 다 Phase 0 환경 감사가 막아야 할 일이다.

## 2. 연결 방법과 함정

```yaml
---
name: implement-worker
description: ...
skills:
  - superpowers:test-driven-development   # 플러그인 스킬
  - eli5                                  # 전역 스킬
---
## 작업 원칙
- 구현 전 superpowers:test-driven-development의 규율을 따른다  ← 본문 병기 (보험)
```

- **`skills:` 필드는 팀원(in-process)·포크에서 무시된다.** 사전 로드에만 의존하지 말고
  본문에 "따르라/읽으라"를 병기한다 — 실행 모드에 따라 한쪽만 작동한다
- Workflow 스크립트에서는 `agent(..., {agentType: 'implement-worker'})`로 커스텀 에이전트를
  쓰면 그 정의의 `skills:`가 함께 적용된다
- 기성 스킬을 물렸으면 새 스킬에 같은 내용을 **다시 쓰지 마라.** 하네스 스킬에는
  도메인 고유 지식만 남긴다 — 중복은 두 사본이 갈라지는 순간 부채가 된다

## 3. 작업 유형 → 기성 스킬 매핑

에이전트 역할을 정의할 때 이 표를 먼저 본다 (superpowers 6.x 기준):

| 에이전트의 작업 | 물릴 스킬 | 효과 |
|----------------|----------|------|
| 설계·기획의 발산 단계 | `superpowers:brainstorming` | 첫 안에 수렴하기 전에 대안 강제 |
| 디버깅·원인 조사 | `superpowers:systematic-debugging` | 추측 수정 대신 가설-검증 루프 |
| 구현 (테스트 가능한 코드) | `superpowers:test-driven-development` | 빨강→초록→리팩터 규율 |
| 계획 수립 / 계획 실행 | `superpowers:writing-plans` / `executing-plans` | 계획 형식·실행 이탈 방지 |
| 완료 보고 직전 검증 | `superpowers:verification-before-completion` | "됐다"고 말하기 전에 실측 |
| 코드 리뷰 요청·수신 | `superpowers:requesting-code-review` / `receiving-code-review` | 리뷰 프로토콜 |
| 병렬 파일 수정 | `superpowers:using-git-worktrees` (또는 `isolation: worktree`) | 충돌 격리 |
| 사용자 대면 설명·보고·문서 | `eli5` (설치돼 있다면) | 청중 눈높이 보정 — 보고 에이전트의 만성 문제인 전문용어 과다를 잡는다 |

QA 에이전트에는 systematic-debugging + verification-before-completion 조합이 기본값으로
좋다 — QA의 실패 모드(존재 확인만 하고 통과)를 정확히 겨냥한다.

## 4. 역할 충돌 정리

- `superpowers:writing-skills` vs 이 하네스의 Phase 4 — **하네스가 우선한다.** 하네스는
  에이전트-스킬 연결과 오케스트레이션까지 다루는 상위 워크플로우다. writing-skills는
  개별 스킬 문장을 다듬을 때 보조 참고로만
- `superpowers:subagent-driven-development` / `dispatching-parallel-agents` vs Workflow —
  Workflow가 있으면 Workflow. 저 스킬들은 Workflow 도구가 없는 환경의 수동 패턴이다
- 트리거 검증(Phase 6-4) 시 물린 기성 스킬과 새 스킬의 description이 서로 침범하지
  않는지 near-miss 쿼리에 포함한다

## 5. 이식성

`skills:`에 적은 스킬이 그 머신에 없으면 조용히 빠진다. 하네스를 다른 환경으로 가져갈
것을 대비해:

1. 하네스 CLAUDE.md 포인터에 **요구 스킬을 명시**한다:
   `**요구 스킬:** superpowers 플러그인(brainstorming, systematic-debugging), eli5(선택)`
2. 에이전트 본문의 병기 지시에는 스킬 이름과 함께 **핵심 원칙 한 줄**을 적는다 —
   스킬이 없어도 방향은 유지된다:
   `- superpowers:systematic-debugging을 따른다 (없으면: 수정 전에 재현→가설→최소 검증 순서를 지킨다)`
3. Phase 6 구조 검증에서 `skills:`의 모든 이름이 실제 설치돼 있는지 확인한다

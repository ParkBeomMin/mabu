# 에이전트 설계 패턴

## 목차
1. 실행 모드 상세
2. 아키텍처 패턴과 Workflow 구현
3. 에이전트 분리 기준
4. 에이전트 frontmatter 전체 필드
5. 에이전트 정의 구조(템플릿)
6. 에이전트 재사용 설계

## 1. 실행 모드 상세

### Workflow (기본)

`Workflow` 도구에 JavaScript 스크립트를 넘기면 백그라운드에서 서브에이전트들을 결정론적으로
오케스트레이션한다. 스크립트 API:

- `agent(prompt, opts)` — 서브에이전트 1개. `opts.schema`(JSON Schema)를 주면 검증된 객체가
  반환된다(파싱 불필요, 불일치 시 자동 재시도). `opts.model`/`opts.effort`/`opts.label`/
  `opts.phase`/`opts.isolation:'worktree'`/`opts.agentType`(커스텀 에이전트 타입) 지원
- `pipeline(items, ...stages)` — 항목별로 스테이지를 통과시키되 **스테이지 간 바리어 없음**.
  항목 A가 3단계일 때 B는 1단계일 수 있다. 다단계의 기본형
- `parallel(thunks)` — 동시 실행 + **바리어**(전부 끝나야 반환). 전 단계 결과 전체가
  필요할 때만(전체 대상 dedup, 0건이면 조기 종료, "다른 발견과 비교" 프롬프트)
- `phase(title)` / `log(msg)` — 진행 표시
- 실패한 agent()는 `null` — `.filter(Boolean)` 후 진행
- 스크립트에서 `Date.now()`/`Math.random()` 불가(재개를 깨뜨림) — 타임스탬프는 args로

주의: Workflow 실행은 사용자 옵트인 대상이다. 하네스에서는 **오케스트레이터 스킬이 호출을
지시**하는 방식으로 충족한다(스킬/슬래시 커맨드 지시는 공식 옵트인 경로).

### 서브 에이전트

`Agent` 도구 직접 호출. 파라미터: `subagent_type`(빌트인 general-purpose/Explore/Plan 또는
커스텀 에이전트 이름, `"fork"`는 현재 컨텍스트 상속), `model`, `isolation: "worktree"`.
백그라운드로 돌면 완료 시 알림이 온다. 1~2회 위임, 단계 사이 사람 판단이 필요할 때.

### 에이전트 팀 (조건부)

`CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1`일 때만 존재한다. v2.1.178부터 팀 생성·정리는
자동이다 — `TeamCreate`/`TeamDelete`는 제거됐고 이 이름으로 설계하면 안 된다.
팀원 간 통신은 `SendMessage`, 작업 조율은 Task 도구(팀 환경에서만 제공).
팀원(in-process)에서는 에이전트 정의의 `skills`·`model` 일부 필드가 다르게 적용되므로
(모델 우선순위: 환경변수 > 스폰 프롬프트 > 정의 > 리더 모델) 정의 파일에만 의존하지 말고
스폰 프롬프트에 핵심 지시를 중복 기재한다.

**팀이 꺼진 환경에서 팀급 품질을 내는 법:** 실시간 토론 대신 Workflow로
생성 → 교차 검증(서로의 산출물을 반박하는 agent) → 종합의 다단계를 돌린다.
토론의 가치는 대부분 "다른 시각의 반박"이며, 그것은 비동기로도 구현된다.

## 2. 아키텍처 패턴과 Workflow 구현

| 패턴 | 언제 | Workflow 구현 |
|------|------|--------------|
| 파이프라인 | 순차 의존 | `agent(A)` → 결과를 다음 `agent(B)` 프롬프트에 |
| 팬아웃/팬인 | 병렬 독립 후 통합 | `parallel(items.map(i => () => agent(...)))` → 종합 agent |
| 전문가 풀 | 입력에 따라 다른 전문가 | 스크립트의 if/switch로 agentType 선택 |
| 생성-검증 | 품질 게이트 | `while (!pass && tries < N) { 생성 → 검증 }` — 루프 상한을 코드로 강제 |
| 감독자 | 동적 작업 분배 | 스크립트 자체가 감독자다. 모델 감독자가 필요하면 schema로 "다음 작업 목록"을 반환받아 루프 |

## 3. 에이전트 분리 기준

| 축 | 분리하라 | 합쳐라 |
|----|---------|--------|
| 전문성 | 요구 지식·프롬프트가 상이 | 같은 지식의 연속 작업 |
| 병렬성 | 동시 실행으로 시간 단축 | 순차 의존이 강함 |
| 컨텍스트 | 각자 큰 입력을 읽어야 함 | 공유 컨텍스트가 작음 |
| 재사용성 | 다른 하네스에서도 쓸 역할 | 이 파이프라인 전용 |

3명의 집중된 에이전트가 5명의 산만한 에이전트보다 낫다.

## 4. 에이전트 frontmatter 전체 필드 (.claude/agents/*.md)

| 필드 | 설명 |
|------|------|
| `name` (필수) | 소문자-하이픈 고유 식별자 |
| `description` (필수) | 언제 위임하는지 |
| `tools` / `disallowedTools` | 도구 allowlist / denylist |
| `model` | `sonnet`·`opus`·`haiku`·`fable`·전체 모델 ID·`inherit` |
| `effort` | `low`~`max` 추론 노력 |
| `skills` | 사전 로드 스킬 목록. **팀원·포크에서는 무시됨** — 본문 지시 병기 필수 |
| `permissionMode` | `default`·`acceptEdits`·`plan` 등 |
| `maxTurns` | 턴 상한 |
| `isolation` | `worktree` — 병렬 파일 수정 시 격리 (셋업 비용 있음, 필요할 때만) |
| `memory` | `user`·`project`·`local` — 실행 간 지속 메모리 |
| `background` | 백그라운드 유지 |
| `mcpServers` / `hooks` / `color` / `initialPrompt` | MCP·훅·표시색·자동 첫 프롬프트 |

관련 훅: `SubagentStart`/`SubagentStop`(matcher로 에이전트 타입 필터), 팀 환경에서는
`TeammateIdle`/`TaskCreated`/`TaskCompleted`.

## 5. 에이전트 정의 구조

```markdown
---
name: {이름}
description: {언제 위임하는지 — 트리거 상황 포함}
model: inherit
effort: {판단 작업이면 high, 기계 작업이면 low}
skills:
  - {필수 참조 스킬}
---

# {역할명}

## 핵심 역할
{한 문단. 무엇을 만들고 무엇은 만들지 않는지}

## 작업 원칙
- 작업 전 반드시 `.claude/skills/{스킬}/SKILL.md`를 읽는다  ← skills 필드와 중복이 아니라 보험
- {도메인 철칙 2~4개 — Why와 함께}

## 입력 / 출력
- 입력: {브리프 형식, 참조 파일}
- 출력: {파일 경로 규칙 또는 schema}. 산출물 본문을 대화로 길게 옮기지 않는다

## 에러 핸들링
- {모호한 입력 처리 — 되묻지 말고 결정 후 명시 / 도구 실패 시 행동}

## 재호출 지침
- 이전 산출물({경로})이 있으면 읽고 지적사항만 수정한다. 처음부터 다시 쓰지 않는다
```

## 6. 에이전트 재사용 설계

신규 요청이 기존 에이전트와 겹칠 때:
- **동일 역할** → 기존 것을 그대로 쓴다. 정의에 부족한 부분만 추가
- **부분 겹침** → 기존 정의를 일반화해 양쪽을 커버 (프로젝트 특화 내용은 스킬로 내림)
- **이름만 유사** → 새로 만들되 description에 경계를 명시해 트리거 충돌 방지

판단 기준: "두 에이전트의 프롬프트를 나란히 놓았을 때 70% 이상 같은가?" 같으면 하나다.

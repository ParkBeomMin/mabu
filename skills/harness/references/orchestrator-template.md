# 오케스트레이터 템플릿

## 목차
1. 구성: 진입 스킬 + Workflow 스크립트
2. Workflow 스크립트 템플릿
3. 진입 스킬 템플릿
4. 서브 에이전트 모드 템플릿
5. 에러 핸들링
6. 재개(resume)

## 1. 구성: 진입 스킬 + Workflow 스크립트

오케스트레이터는 두 파일로 나뉜다:

| 파일 | 담는 것 | 이유 |
|------|--------|------|
| `.claude/skills/{name}/SKILL.md` | 트리거, 컨텍스트 확인(초기/재개/부분), args 조립, Workflow 호출 지시 | 자연어 판단이 필요한 부분 |
| `workflows/{name}.workflow.mjs` | 순서·병렬·루프·스키마·재시도 | 결정론이 필요한 부분 |

흐름 제어를 스킬 산문으로 쓰면 모델이 매번 다르게 해석한다. 판단은 스킬에, 흐름은 코드에.

## 2. Workflow 스크립트 템플릿

```javascript
export const meta = {
  name: '{name}',
  description: '{한 줄}',
  phases: [{ title: '준비' }, { title: '생성' }, { title: '검증' }, { title: '종합' }],
}

// ---- 스키마: 산출물 형식을 도구 계층이 강제한다 ----
const DRAFT = { type: 'object', required: ['id', 'body'], properties: {
  id: { type: 'string' }, body: { type: 'string' } } }
const VERDICT = { type: 'object', required: ['pass', 'issues'], properties: {
  pass: { type: 'boolean' }, issues: { type: 'array', items: { type: 'string' } } } }

const AGENT = (n) => `프로젝트의 .claude/agents/${n}.md를 읽고 그 역할·원칙대로 작업하라.`

phase('생성')
// 팬아웃: 항목별 독립 작업은 pipeline (바리어 없음 — A의 검증과 B의 생성이 겹쳐 돈다)
const results = await pipeline(
  args.items,
  item => agent(`${AGENT('writer')}\n브리프: ${JSON.stringify(item)}`,
                { label: `write:${item.id}`, phase: '생성', schema: DRAFT, effort: 'high' }),
  // 생성-검증 루프: 상한을 코드로 강제한다. "만족할 때까지"를 모델 재량에 맡기지 않는다
  async (draft, item) => {
    for (let t = 0; t < 2; t++) {
      const v = await agent(`${AGENT('tester')}\n검증 대상: ${JSON.stringify(draft)}`,
                            { label: `test:${item.id}`, phase: '검증', schema: VERDICT })
      if (v.pass) return { draft, verdict: v }
      draft = await agent(
        `${AGENT('writer')}\n이전 초안과 지적사항을 반영해 수정하라.\n` +
        `초안: ${JSON.stringify(draft)}\n지적: ${v.issues.join('; ')}`,
        { label: `revise:${item.id}`, phase: '생성', schema: DRAFT })
    }
    return { draft, verdict: { pass: false, issues: ['2회 수정 후에도 미통과'] } }
  },
)

phase('종합')
const ok = results.filter(Boolean)                    // 실패(null)는 버리되
const dropped = args.items.length - ok.length          // 누락 개수는 반드시 보고한다
return { ok, dropped, failed: ok.filter(r => !r.verdict.pass) }
```

바리어가 정당한 경우만 `parallel`을 쓴다: 전체 결과의 dedup, 0건 조기 종료,
"다른 발견과 비교" 프롬프트. "코드가 깔끔해서"는 사유가 아니다.

## 3. 진입 스킬 템플릿

```markdown
---
name: {name}
description: {도메인} 제작·수정·재실행·보완 등 모든 관련 요청에 반드시 사용. ("다시", "~만 다시", "이전 결과 기반" 포함)
---

# {이름} 오케스트레이터

## Phase 0: 컨텍스트 확인
- `_workspace/` 확인: 산출물 있음+부분 수정 요청 → 해당 항목만 args로 / 새 입력 →
  기존을 `_workspace_prev/`로 이동 / 없음 → 초기 실행
- 직전 실행이 중단됐으면 runId를 찾아 `resumeFromRunId` 재개를 우선 검토

## Phase 1: 브리프 컴파일 — 요청 원문을 에이전트에게 그대로 넘기지 않는다
사용자의 요청은 대개 거칠다("재밌게 하나 만들어줘"). 에이전트 품질은 브리프 품질을
못 넘으므로, 원문을 구조화된 브리프로 번역한 뒤에야 에이전트를 부른다:
- **목표**: 측정 가능하게 ("재밌게" → "{도메인 기준으로 번역한 판정 가능 조건}")
- **맥락**: 정본 파일·기존 산출물에서 뽑은 관련 상태
- **제약/금지**: 명시된 것 + 도메인 규칙에서 오는 것
- **산출물 형식**: 경로·스키마
- **가정 목록**: 원문에 없어서 내가 정한 것들 — 반드시 사용자에게 보이게 남긴다
질문은 **진짜 갈림길일 때만 1문항** — 다르게 읽으면 결과물이 달라지는 지점만 묻고,
나머지는 합리적 기본값으로 정한 뒤 가정 목록에 적는다. 사소한 것까지 묻기 시작하면
오케스트레이터가 설문지가 된다.

## Phase 2: args 조립
- 컴파일된 브리프 + {정본 파일}의 작업 목록·설정을 args로
- 타임스탬프 등 비결정 값은 여기서 args에 넣는다 (스크립트 안에서 Date.now() 불가)

## Phase 3: 실행
- `Workflow({scriptPath: "workflows/{name}.workflow.mjs", args})` 호출
- 완료 알림을 기다린다. 그 사이 결과를 예단하지 않는다

## Phase 4: 반영·보고
- 반환값의 dropped/failed를 확인해 등록·보고. 누락을 숨기지 않는다
```

## 4. 서브 에이전트 모드 템플릿

호출 1~2회, 단계 사이 사람 판단이 필요할 때. 스킬 하나로 충분하다:

```
Phase 0 컨텍스트 확인 → Phase 1 브리프 컴파일(위와 동일) →
Phase 2 Agent(정의 파일 지시 + 컴파일된 브리프) →
Phase 3 산출물 검토 후 사용자 확인 → Phase 4 Agent(다음 단계) → 등록
```

Agent 호출 시 정의 파일을 읽으라는 지시와 산출물 경로 규칙을 프롬프트에 포함한다.
병렬이 필요하면 한 메시지에 여러 Agent 호출을 담는다(동시 실행).

## 5. 에러 핸들링

| 상황 | 전략 |
|------|------|
| agent() 실패(null) | filter 후 진행 + 결과에 누락 명시. 필수 단계면 1회 재시도 후 중단 보고 |
| 스키마 불일치 | 도구 계층이 자동 재시도 — 스크립트에서 추가 처리 불필요 |
| 생성-검증 루프 미수렴 | 상한(2회) 도달 시 failed로 반환. 소재 자체의 문제일 수 있으니 사람에게 |
| 상충 데이터 | 삭제하지 말고 출처 병기 |
| 결과가 비었거나 이상 | transcript 디렉토리의 `journal.jsonl`부터 읽는다(각 agent의 실제 반환값) |

## 6. 재개

- `Workflow({scriptPath, resumeFromRunId})` — 변경 없는 agent() 호출 프리픽스는 캐시로
  즉시 반환, 수정된 지점부터 실제 실행
- 이를 위해 스크립트에 비결정 값(Date.now 등)을 넣지 않는다 — args로 주입
- 중간 산출물을 `_workspace/`에 남기는 것도 같은 목적: 세션이 바뀌어도 이어서 간다

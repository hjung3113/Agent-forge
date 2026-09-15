# Agent Forge Architecture Review

## 1. 리뷰 목적

현재 설계가 단순히 그럴듯한 multi-agent 구조가 아니라 **실제 사내 환경에서 구현·운영 가능한지**를 적대적으로 검토한다.

리뷰 관점은 다섯 가지다.

1. 사내 제약 환경에서의 현실 가능성
2. Orchestrator/Controller 경계의 안정성
3. Agent/Harness 합성 모델의 복잡도와 품질
4. 보안·검증·완료 신뢰성
5. 운영·복구·확장성 및 과설계 위험

## 2. 관점 1 — 사내 환경 현실 가능성

### 공격 질문

- 외부 API key가 없어도 실제로 모든 실행이 가능한가?
- OpenCode가 필요한 자동화 인터페이스를 충분히 제공하지 않으면 전체 설계가 무너지지 않는가?
- Agent Forge 자체가 사내 정책상 새로운 우회 gateway처럼 보일 위험은 없는가?

### 판단

**실현 가능성은 높지만 OpenCode capability 검증이 선행 조건이다.**

Agent Forge는 모델 gateway를 새로 만드는 구조가 아니다. 이미 허용된 OpenCode를 subprocess/backend로 실행하고, 그 주변의 workflow, workspace, context, verification을 로컬 코드로 관리한다.

가장 큰 기술 리스크는 OpenCode의 다음 동작이다.

- non-interactive/headless 실행 안정성
- cwd binding
- output/exit code 수집
- cancel/timeout
- 동시 실행
- local/project instruction precedence

### 보강 결정

MVP Phase 0을 **OpenCode Runtime Spike**로 고정한다. 이 검증 전에 orchestration framework를 크게 구현하지 않는다.

## 3. 관점 2 — Orchestrator가 다시 control plane이 될 위험

### 공격 질문

- 총괄 Agent가 Agent를 자유롭게 호출하게 만들면 결국 LLM이 workflow state를 통제하지 않는가?
- recursive delegation이 runaway loop가 되지 않는가?
- LLM이 `DONE`이라고 하면 시스템도 완료로 오인하지 않는가?

### 판단

구조상 가장 위험한 지점이다. `Orchestrator`라는 이름 때문에 구현 중 자연스럽게 권한이 커질 가능성이 높다.

### 보강 결정

다음 불변조건을 강제한다.

```text
Orchestrator owns semantic decisions.
Controller owns executable state.
```

구체적으로:

- Orchestrator 출력은 `DelegateRequest` schema로 제한
- Agent spawn은 Controller만 수행
- max depth / active run / total run budget 강제
- Task state transition은 Controller만 기록
- DONE은 verification gate를 통과해야 함
- Worker의 추가 Agent 요구는 직접 spawn이 아니라 `delegate recommendation`으로 반환

이 경계를 깨는 기능은 구현 편의가 있어도 추가하지 않는다.

## 4. 관점 3 — Harness 합성이 너무 복잡해질 위험

### 공격 질문

- Base + Role + Project + Domain + Skill + Task를 모두 합치면 merge rule 자체가 새로운 framework가 되지 않는가?
- skill이 많아지면 context가 폭발하지 않는가?
- 같은 규칙이 여러 layer에 중복되어 충돌하지 않는가?

### 판단

장기적으로 가장 큰 유지보수 리스크다. 합성 모델 자체는 유효하지만 무제한 layer/override는 피해야 한다.

### 보강 결정

초기 합성 layer를 아래로 제한한다.

```text
System Policy
Role
Project
Domain(optional)
Skills(small set)
Task Contract
```

규칙:

- Agent definition은 참조만 갖고 내용을 복사하지 않음
- permission은 restrictive merge
- required checks/context는 union
- 명시적 모순은 fail-fast
- 실행 시 `resolved-agent.yaml` 저장
- skill 수를 최소화하고 자동으로 모든 skill을 주입하지 않음
- profile provenance 저장

추가 layer를 만들기 전에 기존 layer로 표현할 수 없는 이유를 입증해야 한다.

## 5. 관점 4 — 보안과 검증을 과대평가할 위험

### 공격 질문

- `read_only`라고 YAML에 적는 것만으로 실제 write를 막을 수 있는가?
- cwd가 worktree라고 해서 완전한 sandbox인가?
- Reviewer가 같은 모델이면 독립 검증이라고 할 수 있는가?

### 판단

Prompt 기반 정책을 실제 보안으로 착각하면 실패한다. ForgeRoom ADR-029에서 확인된 것처럼 provider가 지원하지 않는 permission을 runtime enforcement라고 부르면 안 된다.

### 보강 결정

- permission metadata와 hard enforcement를 문서상 명확히 분리
- command/path/spawn/workflow gate를 Controller가 소유
- pre/post git snapshot으로 write scope 검사
- secret/path policy fail-closed
- MVP를 "sandbox"라고 표현하지 않음
- Reviewer 독립성은 **다른 모델**이 아니라 **context isolation + evidence selection**으로 정의
- build/test/path checks는 LLM 밖에서 실행

추가로 더 강한 보안이 필요하면 container/OS sandbox는 별도 phase로 추가한다.

## 6. 관점 5 — 운영 복잡도와 과설계

### 공격 질문

- ForgeRoom과 비슷한 subsystem을 다시 너무 많이 만드는 것 아닌가?
- workflow DSL, database, UI, multi-worker를 한 번에 만들면 완성되지 못하는 것 아닌가?
- 장애 발생 시 어디를 봐야 하는가?

### 판단

가장 현실적인 실패 원인은 기술 난이도보다 **범위 팽창**이다.

### 보강 결정

초기에는 다음만 허용한다.

```text
local single process
OpenCode only
YAML config
JSON/SQLite state
Git worktree
bounded Agent delegation
file artifacts
optional Herdr
```

초기 제외:

- workflow DSL
- Redis/message broker
- distributed worker
- model router/gateway
- marketplace
- full RAG platform
- GUI
- automatic merge

관측은 항상 `Task -> Run -> Event -> Artifact` 경로로 가능해야 한다.

## 7. ForgeRoom 재사용에 대한 적대적 판단

### 그대로 옮기면 안 되는 것

ForgeRoom의 코드가 이미 존재한다는 이유만으로 다음을 Agent Forge에 넣으면 scope가 오염된다.

- Discord/GitHub product UX
- Mastra/workflow DSL
- OpenClaw session model
- Project Room UI semantics
- ForgeMap 전체 subsystem

### 적극 재사용할 것

다음은 Agent Forge 요구와 직접 일치한다.

- runtime provider boundary
- WorktreeManager 패턴
- full Harness 개념
- Project와 Harness 분리
- host-owned hard permission
- ApprovalGate/PolicyGate
- artifact/output contract
- rolling context summary 원칙

## 8. 발견된 추가 설계 결정

### 8.1 Git integration boundary

Agent Forge core의 완료는 기본적으로 다음까지다.

```text
validated worktree/branch
+ diff
+ verification evidence
```

remote push/PR/merge는 core 완료 조건이 아니다. 필요하면 `GitPublisher` 같은 adapter로 후속 추가한다.

자동 merge는 MVP에서 제외한다.

### 8.2 Herdr integration boundary

Herdr가 없어도 전체 task가 실행되어야 한다. Herdr integration failure는 core state를 바꾸지 않는다.

### 8.3 State source of truth

- task/run state: Controller store
- source change: Git worktree/diff
- execution evidence: run artifacts
- UI state: derived view

서로의 역할을 대체하지 않는다.

### 8.4 Same-model limitation

Agent를 여러 개 만든다고 reasoning diversity가 자동으로 생기지 않는다.

따라서 Agent 수보다 다음을 먼저 최적화한다.

- context quality
- skill selectivity
- evidence quality
- task contract
- reviewer isolation
- deterministic check

## 9. 최종 판정

### 현실 가능성

**높음.** 사내에서 OpenCode 실행이 안정적으로 자동화 가능하다는 전제에서 별도의 외부 API key 없이 구현 가능하다.

### 가장 큰 기술 리스크

1. OpenCode automation capability와 실제 사내 설정 차이
2. project-local instruction / compiled harness precedence
3. worktree 밖 write를 어느 수준까지 확실하게 탐지할 수 있는지

### 가장 큰 설계 리스크

1. Orchestrator에 control authority가 새는 것
2. Harness/Skill layer가 과도하게 복잡해지는 것
3. Agent 수를 품질 향상으로 착각하는 것
4. ForgeRoom 기능을 과도하게 이식하는 것

### 현재 설계 상태

현재 문서 구조는 위 리스크를 수용할 수 있는 경계를 갖고 있다. 구현 시 우선순위는 **OpenCode spike -> single worker -> worktree -> harness -> policy/verification -> orchestrator** 순서를 유지하는 것이다.

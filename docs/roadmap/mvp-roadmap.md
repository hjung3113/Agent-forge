# Agent Forge MVP Roadmap

## 1. 원칙

구현 순서는 **LLM orchestration보다 deterministic runtime foundation을 먼저** 만든다.

잘못된 순서:

```text
Orchestrator -> swarm -> Herdr UI -> 나중에 상태/권한/검증
```

권장 순서:

```text
OpenCode runtime spike
 -> single worker
 -> project/worktree
 -> role/harness composition
 -> policy + functional checks
 -> architecture fitness
 -> reviewer/verifier
 -> orchestrator delegation
 -> bounded multi-agent
 -> Herdr UI
```

Agent 수나 Skill 수를 먼저 늘리지 않는다.

## 2. Phase 0 — OpenCode Runtime Spike

목표: Agent Forge가 의존할 OpenCode 최소 실행 계약을 실제 환경에서 확인한다.

검증 항목:

- 특정 cwd에서 실행 가능
- non-interactive/headless 실행 가능
- agent/profile 지정 방식
- stdout/stderr capture
- exit code
- timeout/cancel
- structured/JSON output 지원 범위
- project-local AGENTS/instruction 동작
- tool/permission 표현 가능 범위
- concurrency 시 runtime 충돌 여부

산출물:

```text
OpenCodeAdapter prototype
runtime-capability.md
```

완료 조건:

- 임의의 테스트 repo에서 Controller 코드가 OpenCode를 실행하고 결과/로그를 수집할 수 있다.
- advisory permission과 실제 enforce 가능한 capability를 구분할 수 있다.

이 단계에서 실패하면 전체 아키텍처를 늘리지 않고 adapter 전략부터 수정한다.

## 3. Phase 1 — Single Agent Runtime

구현:

- Task Contract schema
- AgentRegistry 최소 버전
- `implementer` Role Contract
- Agent Environment capability contract
- OpenCodeAdapter
- RunRequest / RunResult
- timeout
- artifact directory
- task/run state

흐름:

```text
CLI request
 -> Controller
 -> implementer
 -> OpenCode
 -> result
```

아직 포함하지 않음:

- Orchestrator LLM
- parallel agents
- Herdr
- dynamic skill routing
- external Skill installation

완료 조건:

- 동일 task 입력으로 resolved role/harness, 실행 artifact, 상태를 재현 가능하게 남긴다.

## 4. Phase 2 — Project Registry + Worktree

구현:

- ProjectRegistry
- canonical repo cache
- clone/fetch
- WorktreeManager
- branch naming
- baseline/diff
- cleanup
- project build/test command
- allowed/denied path

흐름:

```text
Task(project=foo)
 -> repo resolve
 -> worktree
 -> OpenCode(cwd=worktree)
 -> diff
 -> checks
```

완료 조건:

- 등록된 프로젝트에 대해 source repo를 훼손하지 않고 task별 branch/worktree에서 작업한다.
- run 전후 변경을 구분해 scope 밖 write를 탐지할 수 있다.

## 5. Phase 3 — Role / Harness / Skill Composition

구현:

- RoleRegistry
- SkillRegistry
- HarnessRegistry
- HarnessCompiler
- capability/permission merge
- conflict validation
- resolved profile snapshot
- provenance

Canonical Role:

```text
orchestrator
project-expert
domain-expert
architect
implementer
debugger
reviewer
verifier
```

초기에는 실제 실행이 필요한 Role부터 구현하고 나머지는 schema/contract만 준비한다.

기본 Harness:

```text
implementation
analysis
review
verification
```

완료 조건:

- 같은 OpenCode backend가 서로 다른 role/project/skill 조합으로 실제 다른 실행 계약을 받는다.
- Role 전문성과 language/framework Skill이 중복 복제되지 않는다.
- resolved profile이 source provenance와 함께 artifact로 저장된다.

## 6. Phase 4 — Policy + Functional Verification

구현:

- CommandGate
- PathGate
- SpawnGate 기본 구조
- post-run diff validation
- CheckRunner
- OutputContractValidator
- acceptance evidence
- Reviewer/Verifier workflow 최소 구조

완료 조건:

- read-only Reviewer의 write가 탐지된다.
- forbidden path 변경이 차단된다.
- test failure가 있는데 DONE이 되는 경로가 없다.
- advisory instruction 실패와 hard policy violation을 구분한다.

## 7. Phase 4.5 — Architecture Fitness

기존 Verification subsystem 안에 추가한다. 별도 orchestration framework를 만들지 않는다.

구현:

- Project Profile의 `architecture.checks`
- blocking/non-blocking check
- architecture check result artifact
- baseline mode
- waiver reference
- CheckRunner integration

첫 적용 후보:

```text
C#     -> ArchUnitNET or project-specific architecture tests
Python -> import-linter or project-specific import checks
```

도구는 선택 사항이며 contract는 command/exit/evidence 중심으로 유지한다.

완료 조건:

- 샘플 C# 또는 Python 프로젝트에서 금지된 dependency를 deterministic하게 실패시킨다.
- legacy baseline이 있을 때 기존 위반과 신규 위반을 구분한다.
- blocking architecture failure가 DONE을 막는다.

## 8. Phase 5 — Reviewer / Verifier Hardening

구현:

- independent reviewer context assembly
- finding schema
- acceptance criteria -> evidence mapping
- architecture fitness evidence 연결
- unresolved blocking finding gate

완료 조건:

- Reviewer가 Implementer private reasoning 없이 diff/evidence만으로 검토한다.
- Verifier가 AC별 pass/fail/unknown과 근거를 반환한다.
- Reviewer의 근거 없는 `pass`만으로 DONE이 되지 않는다.

## 9. Phase 6 — Orchestrator Agent

이 단계에서 총괄 LLM Agent를 추가한다.

구현:

- Orchestrator prompt/harness
- Agent capability catalog
- DelegateRequest schema
- AgentResult schema
- dependency proposal
- delegation depth/run budget
- rolling task summary
- minimal-role selection guideline

흐름 예:

```text
Small fix
User
 -> Orchestrator
 -> debugger/implementer
 -> verifier

Architecture-sensitive change
User
 -> Orchestrator
 -> project-expert
 -> architect
 -> implementer
 -> reviewer
 -> verifier
```

모든 task에 긴 고정 chain을 적용하지 않는다.

완료 조건:

- Orchestrator가 subprocess/worktree를 직접 다루지 않고 structured request만으로 여러 Agent를 호출한다.
- task complexity에 따라 불필요한 Role을 생략할 수 있다.

## 10. Phase 7 — Skill Intake & Management

외부 marketplace가 아니라 **검증된 local Skill Registry**를 먼저 만든다.

구현:

```text
agent-forge skill list
agent-forge skill show <id>
agent-forge skill validate <path|id>
agent-forge skill audit <path>
agent-forge skill stage <path|source>
agent-forge skill approve <staged-id>
```

초기 intake:

- source revision/license/provenance
- metadata/schema validation
- script/network/filesystem inspection
- trigger/non-trigger
- role compatibility
- overlap/conflict
- integrity hash

완료 조건:

- unapproved external Skill은 실행 profile에 들어갈 수 없다.
- approved Skill은 pinned source와 local modification을 추적할 수 있다.
- broken reference/unsafe requirement를 fail-fast 한다.

## 11. Phase 8 — Parallel / Advanced Scheduling

필요성이 확인된 후 추가한다.

- dependency DAG
- bounded parallel workers
- separate writer worktrees
- merge/reconcile strategy
- cancellation propagation
- queue prioritization

병렬화를 성능을 위해 너무 일찍 넣지 않는다. 동일 모델/동일 repo task는 병렬화가 오히려 merge cost를 증가시킬 수 있다.

## 12. Phase 9 — Management CLI

```text
agent-forge agent add/list/show/enable/disable
agent-forge project add/list/validate/remove
agent-forge task show/cancel
agent-forge architecture check <project>
```

추가 시 모든 config는 Registry validation을 거친다.

## 13. Phase 10 — Herdr Integration

Herdr는 이 시점에 붙인다.

목표:

- Orchestrator/worker process 관측
- task/run state 표시
- logs tail
- operator cancel/intervention

Herdr 없이 core workflow가 정상 동작해야 한다.

## 14. Cross-harness 지원

MVP는 OpenCode만 지원한다.

다른 runtime이 실제로 필요해질 때:

```text
Canonical definitions
 -> Resolved Harness IR
 -> Runtime Adapter
 -> runtime-native generated artifact
```

순서로 추가한다.

새 adapter의 완료 조건:

- capability matrix 작성
- unsupported capability가 silent drop되지 않음
- 동일 canonical Role/Skill을 target runtime native form으로 생성 가능
- generated artifact가 canonical source가 아님

## 15. MVP 정의

첫 실사용 MVP는 다음을 권장한다.

```text
Phase 0 ~ Phase 6
+ Phase 7의 local Skill validate/audit 최소 기능
```

즉 MVP 기능:

- OpenCode backend
- Orchestrator 1개
- canonical Role contracts
- Agent Registry
- Project Registry
- Git worktree
- Harness + selected Skill composition
- PolicyGate
- Implementer / Reviewer / Verifier
- Architect optional path
- functional CheckRunner
- project-defined Architecture Fitness
- artifact/state/log
- deterministic completion gate
- external Skill auto-install 금지

Herdr, 병렬 DAG, multi-harness는 MVP 필수가 아니다.

## 16. 첫 실사용 시나리오

첫 end-to-end 목표는 복잡하게 잡지 않는다.

```text
User:
"등록된 샘플 프로젝트에서 작은 버그를 찾아 수정하고 검증해"

Orchestrator
 -> project-expert (필요 시)
 -> debugger/implementer
 -> reviewer (risk에 따라)
 -> verifier
```

Architecture-sensitive 샘플은 별도로 검증한다.

```text
"Domain이 Infrastructure를 참조하지 못하게 규칙을 추가하고 위반 예제를 검출해"

Architect
 -> project architecture rule
 -> CheckRunner
 -> Verifier
```

검증할 것:

- Agent 선택이 최소한인가
- context가 과하지 않은가
- worktree 격리가 실제로 되는가
- skill/harness가 의도대로 합성되는가
- reviewer가 구현 Agent를 맹목적으로 따라가지 않는가
- failed functional/architecture check가 DONE을 막는가
- 모든 근거가 artifact로 남는가

## 17. 후순위 항목

실제 병목이 확인되기 전에는 구현하지 않는다.

- distributed queue
- Redis
- multi-machine worker
- public marketplace
- runtime external Skill auto-install
- 자체 vector DB/RAG
- 복잡한 workflow DSL
- full GUI
- model routing gateway
- 자동 LLM skill router
- 대규모 statistical skill evaluation platform

## 18. 구현 선택 기준

새 기능을 추가하기 전 다음 질문에 답한다.

1. 현재 MVP 실행을 실제로 막고 있는가?
2. deterministic code로 해결할 수 있는가?
3. LLM Agent를 하나 더 만드는 것이 정말 필요한가?
4. 기존 Role/Project/Domain/Skill 조합으로 해결할 수 없는가?
5. 새로운 source of truth를 만드는가?
6. 실패 시 어떻게 관측하고 복구하는가?
7. 반복되는 review rule을 executable check로 내릴 수 있는가?
8. 외부 Skill/도구를 도입할 때 provenance와 제거 경로가 있는가?

5번이 yes인데 명확한 이유가 없다면 추가하지 않는 것이 기본이다.

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
 -> harness composition
 -> policy/checks
 -> orchestrator delegation
 -> multi-agent
 -> Herdr UI
```

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
- concurrency 시 runtime 충돌 여부

산출물:

```text
OpenCodeAdapter prototype
runtime-capability.md
```

완료 조건:

- 임의의 테스트 repo에서 Controller 코드가 OpenCode를 실행하고 결과/로그를 수집할 수 있다.

이 단계에서 실패하면 전체 아키텍처를 늘리지 않고 adapter 전략부터 수정한다.

## 3. Phase 1 — Single Agent Runtime

구현:

- Task Contract schema
- AgentRegistry 최소 버전
- `implementer` Role
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
- dynamic skills

완료 조건:

- 동일 task 입력으로 실행 artifact와 상태를 재현 가능하게 남긴다.

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

## 5. Phase 3 — Harness / Skill Composition

구현:

- RoleRegistry
- SkillRegistry
- HarnessRegistry
- HarnessCompiler
- permission merge
- conflict validation
- resolved profile snapshot

기본 Role:

```text
implementer
reviewer
verifier
debugger
project-expert
domain-expert
```

기본 Harness:

```text
implementation
analysis
review
verification
```

완료 조건:

- 같은 OpenCode backend가 서로 다른 role/project/skill 조합으로 실제 다른 실행 계약을 받는다.

## 6. Phase 4 — Policy + Verification

구현:

- CommandGate
- PathGate
- SpawnGate 기본 구조
- post-run diff validation
- CheckRunner
- OutputContractValidator
- acceptance evidence
- Reviewer/Verifier workflow

완료 조건:

- read-only Reviewer의 write가 탐지된다.
- forbidden path 변경이 차단된다.
- test failure가 있는데 DONE이 되는 경로가 없다.

## 7. Phase 5 — Orchestrator Agent

이 단계에서 총괄 LLM Agent를 추가한다.

구현:

- Orchestrator prompt/harness
- Agent capability catalog
- DelegateRequest schema
- AgentResult schema
- dependency proposal
- delegation depth/run budget
- rolling task summary

흐름:

```text
User
 -> Orchestrator
 -> delegate(log-expert)
 -> result
 -> delegate(implementer)
 -> result
 -> delegate(reviewer)
 -> verify
```

완료 조건:

- Orchestrator가 subprocess/worktree를 직접 다루지 않고 structured request만으로 여러 Agent를 호출한다.

## 8. Phase 6 — Parallel / Advanced Scheduling

필요성이 확인된 후 추가한다.

- dependency DAG
- bounded parallel workers
- separate writer worktrees
- merge/reconcile strategy
- cancellation propagation
- queue prioritization

병렬화를 성능을 위해 너무 일찍 넣지 않는다. 동일 모델/동일 repo task는 병렬화가 오히려 merge cost를 증가시킬 수 있다.

## 9. Phase 7 — Agent/Project Management CLI

```text
agent-forge agent add/list/show/enable/disable
agent-forge skill list/validate
agent-forge project add/list/validate/remove
agent-forge task show/cancel
```

추가 시 모든 config는 Registry validation을 거친다.

## 10. Phase 8 — Herdr Integration

Herdr는 이 시점에 붙인다.

목표:

- Orchestrator/worker process 관측
- task/run state 표시
- logs tail
- operator cancel/intervention

Herdr 없이 core workflow가 정상 동작해야 한다.

## 11. MVP 정의

다음까지를 첫 실사용 MVP로 권장한다.

```text
Phase 0 ~ Phase 5
+ 최소 agent/project CLI
```

즉 MVP 기능:

- OpenCode backend
- Orchestrator 1개
- role/project/domain expert 호출
- Agent Registry
- Project Registry
- Git worktree
- Harness + Skill composition
- PolicyGate
- Implementer / Reviewer / Verifier
- artifact/state/log
- deterministic completion gate

Herdr와 병렬 DAG는 MVP 필수가 아니다.

## 12. 첫 실사용 시나리오

첫 end-to-end 목표는 복잡하게 잡지 않는다.

```text
User:
"등록된 샘플 프로젝트에서 작은 버그를 찾아 수정하고 검증해"

Orchestrator
 -> project-expert
 -> implementer
 -> reviewer
 -> verifier
```

검증할 것:

- Agent 선택이 올바른가
- context가 과하지 않은가
- worktree 격리가 실제로 되는가
- skill/harness가 의도대로 합성되는가
- reviewer가 구현 Agent를 맹목적으로 따라가지 않는가
- failed test가 DONE을 막는가
- 모든 근거가 artifact로 남는가

## 13. 후순위 항목

실제 병목이 확인되기 전에는 구현하지 않는다.

- distributed queue
- Redis
- multi-machine worker
- marketplace
- automatic external skill installation
- 자체 vector DB/RAG
- 복잡한 workflow DSL
- full GUI
- model routing gateway

## 14. 구현 선택 기준

새 기능을 추가하기 전 다음 질문에 답한다.

1. 현재 MVP 실행을 실제로 막고 있는가?
2. deterministic code로 해결할 수 있는가?
3. LLM Agent를 하나 더 만드는 것이 정말 필요한가?
4. 기존 profile/skill 조합으로 해결할 수 없는가?
5. 새로운 source of truth를 만드는가?
6. 실패 시 어떻게 관측하고 복구하는가?

5번이 yes인데 명확한 이유가 없다면 추가하지 않는 것이 기본이다.

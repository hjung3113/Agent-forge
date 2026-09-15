# ForgeRoom Reuse Analysis

## 1. 목적

이 문서는 기존 `hjung3113/ForgeRoom`에서 Agent Forge에 재사용할 가치가 있는 설계와 구현 패턴을 정리한다.

핵심 원칙은 **ForgeRoom을 Agent Forge의 기반으로 간주하지 않는 것**이다. 두 프로젝트의 제품 목표가 다르므로 그대로 복사하지 않고 다음 세 범주로 판단한다.

```text
A. Directly Reuse Concept
B. Reuse After Adaptation
C. Do Not Carry Forward
```

ForgeRoom은 로컬 멀티에이전트 orchestration, worktree, agent runtime provider, harness, policy gate를 이미 상당 부분 다뤘기 때문에 Agent Forge의 설계를 검증하는 좋은 선행 자산이다.

## 2. 제품 경계 비교

### ForgeRoom

주요 목표:

- Discord/GitHub 기반 원격 task intake
- 로컬 multi-agent workflow
- Project Room
- worktree -> agent -> check -> PR
- OpenClaw 등 runtime provider 추상화
- workflow/Conductor/ForgeMap 중심의 제품 orchestration

참고:
- [ForgeRoom Overview](https://github.com/hjung3113/ForgeRoom/blob/main/Docs/overview.md)
- [ForgeRoom Architecture](https://github.com/hjung3113/ForgeRoom/blob/main/Docs/architecture.md)

### Agent Forge

주요 목표:

- 사내에서 허용된 OpenCode를 실행 backend로 활용
- 총괄 Orchestrator가 필요한 역할/프로젝트/도메인 Agent를 선택
- Agent별 Harness 조합
- project-bound 실행환경
- 안전한 spawn/runtime/verification control plane
- Agent/Skill/Profile을 확장할 수 있는 로컬 Agent Runtime

따라서 Agent Forge는 Discord/GitHub UX보다 **Agent Runtime composition과 orchestration core 자체**가 제품 중심이다.

## 3. 재사용 우선순위 요약

| ForgeRoom 자산 | Agent Forge 판단 | 이유 |
|---|---|---|
| AgentRuntimeProvider boundary | **A - 직접 개념 재사용** | OpenCode를 adapter 뒤로 숨기기 위한 핵심 경계 |
| AgentRunner request/result contract | **A/B** | 대부분 유효, OpenClaw/session 세부는 제거/수정 |
| Full Step Harness | **A - 직접 개념 재사용** | Agent별 환경/권한/skill/output 계약 요구와 정확히 일치 |
| Harness hard/advisory 분리 | **A - 직접 재사용** | prompt-only permission의 한계를 정확히 다룸 |
| Project Room vs Harness 분리 | **A - 직접 재사용** | project identity와 execution contract 분리에 유효 |
| ProjectRegistry | **B - 수정 재사용** | Discord/OpenClaw room metadata 제거, repo/workspace 중심으로 축소 |
| WorktreeManager | **A/B** | task isolation 패턴 재사용, `.forgeroom` 명칭/PR 정책은 수정 |
| ApprovalGate | **A - 직접 개념 재사용** | 사내 환경의 path/command safety에 중요 |
| File-based prompt/artifact protocol | **B - 수정 재사용** | 재현성/디버깅에 유효, OpenCode 실제 I/O 방식에 맞게 조정 |
| Conductor rolling summary | **B - 선택 재사용** | Orchestrator context 관리에 유용, 별도 meta-agent는 MVP 필수 아님 |
| ForgeMap | **B/C** | project context selection 아이디어는 유효, 전체 서브시스템 이식은 과함 |
| Workflow DSL/Mastra | **C - 초기 미채택** | Agent Forge MVP에는 과한 orchestration abstraction |
| Discord/GitHub Gateway | **C - 제외** | Agent Forge core 목적과 무관 |
| OpenClaw-specific session/agent | **C - 제외** | 초기 backend가 OpenCode이므로 provider-specific |

## 4. AgentRuntimeProvider -> RuntimeAdapter

ForgeRoom은 `AgentRunner`와 `AgentRuntimeProvider`를 분리한다.

참고:
- [AgentRunner + AgentRuntimeProvider](https://github.com/hjung3113/ForgeRoom/blob/main/Docs/modules/agent-runner.md)

이 경계는 Agent Forge에서 그대로 유효하다.

```text
Agent Forge Core
  -> RuntimeAdapter
       -> OpenCodeAdapter
```

### 그대로 가져올 원칙

- orchestration core가 CLI 실행 세부사항을 모르게 한다.
- runtime health와 run result를 공통 계약으로 변환한다.
- stdout/stderr를 artifact로 남긴다.
- timeout/failure kind를 공통 taxonomy로 변환한다.

### 수정할 부분

ForgeRoom의 다음 요소는 OpenClaw 전용이므로 제거한다.

- OpenClaw endpoint/token
- provider native session key
- OpenClaw role agent mapping
- gateway-specific resume semantics

Agent Forge MVP에서는 OpenCode subprocess/CLI contract만 구현한다.

## 5. Full Step Harness -> Agent Forge Harness

가장 직접적으로 재사용할 자산이다.

참고:
- [ADR-029: Full Step Harness](https://github.com/hjung3113/ForgeRoom/blob/main/Docs/decisions/2026-05-29-029-full-step-harness.md)

ForgeRoom은 Harness를 다음처럼 확장했다.

```text
harness.yaml
prompt-contract.md
output contract
permissions
tools
skills
hooks
AGENTS.md
```

이 구조는 Agent Forge의 요구인:

```text
Role
+ Project
+ Domain
+ Skills
+ Permissions
+ Output Contract
```

와 매우 잘 맞는다.

### 특히 유지할 결정

#### 1. Harness는 provider identity가 아니다

Harness는 실행 계약이고 provider/runtime identity는 별도다.

Agent Forge에서도:

```text
Harness != OpenCode Agent ID
Harness != Project
Harness != Model
```

로 유지한다.

#### 2. Permission은 soft와 hard를 구분

ForgeRoom ADR-029의 가장 중요한 판단이다.

```text
soft advisory
  -> prompt / AGENTS

hard enforcement
  -> host PolicyGate / path gate / diff validation
```

Agent Forge도 이 원칙을 그대로 채택한다.

#### 3. Output contract를 structured metadata로 둔다

reviewer/plan/verifier별 결과 형식을 코드에 하드코딩하기보다 Harness metadata로 일반화하는 방향을 가져온다.

## 6. Project Room -> Project Profile

ForgeRoom ADR-028은 Project Room과 Step Harness를 직교 개념으로 분리한다.

참고:
- [ADR-028: Project Room domain and seam](https://github.com/hjung3113/ForgeRoom/blob/main/Docs/decisions/2026-05-26-028-project-room-domain-and-seam.md)

핵심 교훈:

```text
Project Room
 = 이 task가 어느 project 운영공간에 속하는가

Harness
 = 이 실행이 어떤 환경/계약을 받는가
```

Agent Forge에서는 UI/Discord 의미의 Room은 필요 없지만 분리 원칙은 그대로 적용한다.

```text
Project Profile
 = repository / commands / context / project restrictions

Harness Profile
 = role execution environment / permissions / skills / output rules
```

따라서 `logwarehouse` project와 `implementation` harness를 한 YAML로 합치지 않는다.

## 7. ProjectRegistry 재사용

참고:
- [ProjectRegistry](https://github.com/hjung3113/ForgeRoom/blob/main/Docs/modules/project-registry.md)

재사용할 요소:

- config load/validate
- repository path/default branch
- project commands
- 시작 시 fail-fast validation
- project-specific allowed behavior

제외할 요소:

- Discord channel
- OpenClaw room/agent
- Mastra exposure
- maintainer UX 등 ForgeRoom product-specific metadata

Agent Forge Project Registry는 repository/workspace/context/check command 중심으로 더 작게 유지한다.

## 8. WorktreeManager 재사용

참고:
- [WorktreeManager](https://github.com/hjung3113/ForgeRoom/blob/main/Docs/modules/worktree-manager.md)

ForgeRoom에서 유효한 패턴:

- task별 branch/worktree
- runtime context directory bootstrap
- baseline/diff
- idempotent reuse
- allowed path 밖 변경 검증
- task 종료 cleanup policy

Agent Forge에서 개선할 점:

### canonical clone/cache 명확화

Agent Forge는 여러 project Agent를 반복 실행하므로:

```text
~/.agent-forge/repos/<project>
~/.agent-forge/worktrees/<task>
```

를 명시적으로 분리한다.

### workspace mode 지원

- shared read-only task worktree
- exclusive writer worktree
- parallel worker 별 worktree

를 Controller가 선택할 수 있게 한다.

## 9. ApprovalGate 재사용

참고:
- [ApprovalGate](https://github.com/hjung3113/ForgeRoom/blob/main/Docs/modules/approval-gate.md)

ForgeRoom에서 이미 다룬 다음 위험 카테고리는 그대로 유효하다.

- destructive git
- protected branch 직접 수정
- filesystem 위험 명령
- secret path
- migration/reset
- download-and-execute
- 외부 송출

Agent Forge에서는 이름을 `PolicyGate`로 일반화하고 spawn/skill/project permission까지 포함시킨다.

```text
CommandGate
PathGate
SpawnGate
WorkflowGate
```

## 10. Prompt File Protocol 재사용

참고:
- [Prompt File Protocol](https://github.com/hjung3113/ForgeRoom/blob/main/Docs/concepts/prompt-file-protocol.md)

장점:

- 긴 context를 CLI argument에 직접 넣지 않음
- prompt/output/debug artifact 재현 가능
- restart/retry 시 근거 보존
- 큰 diff/log를 경로로 참조 가능

Agent Forge에서도 runtime artifact protocol은 유지할 가치가 높다.

다만 중요한 교훈이 있다.

ForgeRoom ADR-029에서는 agent가 output 파일을 직접 쓰는지 provider가 response를 저장하는지 모호해져 output-channel bug가 발생했다. 따라서 Agent Forge는 초기부터 다음을 명확히 해야 한다.

```text
Source changes
  -> Agent writes to worktree

Agent response
  -> RuntimeAdapter captures response
  -> Controller persists output artifact
```

Agent에게 Agent Forge 내부 `outputs/` 파일을 직접 쓰게 하는 것과 response channel을 혼합하지 않는다.

## 11. Conductor에서 가져올 부분

참고:
- [Conductor](https://github.com/hjung3113/ForgeRoom/blob/main/Docs/modules/conductor.md)

ForgeRoom Conductor의 유용한 원칙:

- rolling task summary
- context 상한 관리
- step 사이 사용자 피드백 유지
- meta-agent가 직접 source code를 수정하지 않음
- 상태 전이는 코드가 소유

Agent Forge에서 Orchestrator context 관리에 재사용한다.

다만 MVP에서 별도 `Conductor Agent`를 만들 필요는 없다.

```text
Controller-maintained task state
+ optional LLM rolling summary
```

정도로 시작한다.

## 12. ForgeMap에서 가져올 부분

ForgeMap 전체 구현을 이식하는 것은 Agent Forge MVP에는 과하다.

가져올 아이디어:

- project canonical context
- task별 selected context
- context provenance
- 큰 context를 전부 prompt에 넣지 않음

초기 구현:

```text
Project Profile
+ AGENTS.md
+ explicit context paths
+ Agent-selected relevant files
```

로 충분하다.

필요가 확인되면 ContextProvider/ContextSelector 계층으로 발전시킨다.

## 13. 가져오지 않을 것

### Workflow DSL / Mastra

Agent Forge 초기 목표는 `Orchestrator -> structured delegation -> Controller`다. 별도 YAML workflow DSL과 workflow engine을 동시에 만들면 orchestration source of truth가 두 개가 된다.

MVP에서는 상태 머신과 dependency graph만 코드로 둔다.

### Discord/GitHub UX

Agent Forge는 execution runtime이 핵심이다. UI/TaskSource는 adapter로 나중에 추가한다.

### OpenClaw-specific layer

Agent Forge 초기 runtime은 OpenCode다. ForgeRoom의 OpenClaw-specific session/agent inventory를 가져오면 다시 provider 결합이 생긴다.

## 14. ForgeRoom에서 확인된 설계 리스크

### 14.1 Output channel ambiguity

agent response와 output file write의 authority를 혼합하지 않는다.

### 14.2 Provider capability를 과장하지 않기

provider가 permission flag를 제공하지 않으면 "런타임이 권한을 강제한다"고 간주하지 않는다.

### 14.3 Project와 Harness 결합 방지

project inventory와 execution contract는 직교해야 재사용성이 유지된다.

### 14.4 LLM state transition 방지

Pending -> Applied, DONE 같은 상태는 코드가 결정해야 한다.

### 14.5 UI/transport를 core에 섞지 않기

Discord/Herdr/OpenCode는 각각 adapter/surface이며 Controller core의 domain model이 아니다.

## 15. Agent Forge로 이전할 우선순위

```text
1. RuntimeAdapter boundary
2. WorktreeManager pattern
3. Harness schema / HarnessCompiler
4. PolicyGate
5. Artifact/output protocol
6. ProjectRegistry
7. rolling summary/context provenance
8. advanced context selection
```

반대로 다음은 처음부터 옮기지 않는다.

```text
Workflow DSL
Mastra
Discord orchestration
OpenClaw session model
ForgeMap full subsystem
```

## 16. 결론

ForgeRoom은 Agent Forge와 상당한 공통 기반을 갖고 있다. 특히 **Runtime Provider, Worktree, Harness, Project separation, PolicyGate**는 이미 한 차례 설계 검증을 거친 자산이므로 적극 재사용할 가치가 있다.

그러나 Agent Forge는 더 좁고 명확하게 다음에 집중해야 한다.

```text
Composable Agent Runtime
+ deterministic orchestration control
+ project-bound harness
+ OpenCode backend
+ verification/safety
```

ForgeRoom의 제품 기능을 가져오는 것이 아니라, **그 과정에서 검증된 execution boundary와 실패에서 얻은 교훈을 가져오는 것**이 적절하다.

# 02. System Architecture

## 1. 상위 구조

```text
                               User
                                |
                                v
                      +--------------------+
                      | Orchestrator Agent |
                      +---------+----------+
                                | semantic proposal
                                v
+-------------------------------------------------------------------+
|                    Agent Forge Controller                          |
|                                                                   |
|  TaskSpec / Amendment      Registry / Harness Compiler            |
|  Step / Attempt Scheduler  Policy / Verification Planner          |
|  State / Lease Manager     Workspace Manager                      |
|  Runtime Isolation         Runtime Adapter                        |
|  CheckRunner               Controller Artifact / Event Store      |
+-------------------------------+-----------------------------------+
                                |
                                v
                         OpenCode / Backend
                                |
              +-----------------+-----------------+
              |                 |                 |
          Project Expert     Implementer       Reviewer

                     Herdr = optional observer UI
```

## 2. 핵심 책임 경계

### Orchestrator

의미 판단을 담당한다.

- 목표 해석/분해 제안
- Agent/Role 선택 제안
- dependency/next action 제안
- amendment 제안

소유하지 않는다.

- process/worktree lifecycle
- frozen TaskSpec 변경 authority
- canonical state transition
- retry budget override
- permission grant
- final completion authority

### Controller

실행 가능성과 canonical state를 소유한다.

- TaskSpec normalize/freeze/hash
- TaskAmendment validation
- Step/RunAttempt lifecycle
- registry/profile resolve
- capability/policy validation
- workspace/lease
- runtime process lifecycle
- check/evidence collection
- recovery/reconciliation
- completion gate

Controller는 가능한 한 deterministic code로 구현한다.

### Cross-task Status Query (optional)

project를 지정하지 않은 진입을 지원하려면 Controller/Registry가 소유하는 read-only 조회 capability가 필요하다.

- LLM이 관여하지 않는다.
- 범위: 진행 중/대기/최근 완료 Task, RunAttempt 상태, 최근 event, 등록 project 목록.
- 상태를 변경하지 않으며 control plane이 아니다.

이 조회 결과는 [04-orchestration-runtime.md](04-orchestration-runtime.md)의 Orchestrator `user-briefing` mode 입력으로만 소비된다.

## 3. Controller를 trusted computing base로 본다

Agent Forge의 security/completion claim은 결국 Controller correctness에 의존한다.

따라서 Controller core는 작게 유지하고 다음을 분리한다.

```text
semantic LLM decision
  !=
state/policy/evidence authority
```

Controller가 읽는 untrusted input은 schema validation을 거친다.

## 4. Canonical subdomains

세부 책임은 다음 문서가 소유한다.

```text
TaskSpec / Amendment
  -> 11-task-contract-and-change-control.md

Runtime trust / capability enforcement
  -> 12-runtime-isolation-and-trust-boundaries.md

Task-Step-RunAttempt / recovery / artifact provenance
  -> 13-state-recovery-and-artifact-integrity.md

Verification floor / system eval
  -> 14-evaluation-and-conformance.md
```

## 5. Runtime Adapter

초기:

```text
RuntimeAdapter
  -> OpenCodeAdapter
```

core가 OpenCode CLI 세부 옵션을 알지 않게 한다.

개념 contract:

```text
run(request) -> runtime_result
cancel(attempt_id)
health() -> capability/status
```

`run`에는 최소:

- cwd/workspace id
- resolved harness
- frozen task reference
- environment profile
- timeout/output budget
- capability report

를 전달한다.

Process-tree containment과 capability level은 `12` 문서를 따른다.

## 6. Registry / Harness

```text
AgentRegistry
RoleRegistry
SkillRegistry
ProjectRegistry
HarnessRegistry
DomainRegistry(optional)
```

Agent는 고정 prompt가 아니라 composition 결과다.

```text
Base + Role + Project + Domain + Skills + Frozen TaskSpec
  -> Resolved Harness IR
  -> Runtime Adapter
```

Permission requirement와 grant는 별도로 resolve한다.

## 7. Workspace Manager

- canonical repo/cache
- exact base commit resolve
- branch/worktree
- writer lease
- baseline/diff
- allowed/denied/policy-sensitive paths
- cleanup

Git worktree는 source isolation이며 OS sandbox가 아니다.

## 8. State / Artifact

```text
Task
  -> Step
      -> RunAttempt
```

SQLite를 초기 canonical state store로 사용할 수 있다.

Controller Artifact Store에는 Controller가 수집/생성한 request/input/check/result를 기록한다.

단, Worker가 같은 OS identity로 전체 host filesystem에 접근할 수 있는 환경에서는 저장 위치만으로 tamper-proof라고 주장하지 않는다. 실제 storage integrity level은 Runtime Isolation capability와 함께 기록한다.

## 9. Verification

세 층을 결합한다.

```text
Output/schema validation
Deterministic CheckRunner
Semantic Reviewer/Verifier
```

CheckRunner가 실행한 command의 exit/result는 Controller-observed evidence다.

하지만 project command/test 자체가 임의 코드를 실행할 수 있으므로 **evidence authority와 execution safety는 다른 문제**다. CheckRunner도 Runtime Isolation policy를 적용받는다.

## 10. Control Plane / Execution Plane

### Control Plane

- TaskSpec/amendment
- registry/harness
- scheduler/state/lease
- policy/verification plan
- artifact/event metadata

### Execution Plane

- OpenCode/backend process
- child process
- Git worktree
- project build/test process
- staged context/generated runtime files

Execution Plane의 결과가 Control Plane authority를 직접 바꾸지 않는다.

## 11. 데이터 흐름

```text
1. User request
2. TaskSpec normalize/freeze + exact base SHA
3. Orchestrator delegate proposal
4. Controller Step/Attempt validation
5. Agent/Project/Harness resolve
6. workspace + lease
7. capability/policy preflight
8. Runtime Adapter execution
9. Controller output/diff collection
10. change classification
11. VerificationPlan required checks
12. CheckRunner / Reviewer / Verifier
13. artifact/event/state commit
14. completion gate
15. next Step or DONE
```

## 12. 초기 저장 구조

```text
agent-forge/                  # source
├─ src/
├─ agents/
├─ roles/
├─ projects/
├─ domains/
├─ skills/
├─ harnesses/
├─ config/
└─ evals/

~/.agent-forge/               # local runtime
├─ repos/
├─ worktrees/
├─ state/
├─ tasks/
└─ runs/
```

runtime state/artifact를 product source repo에 canonical data로 저장하지 않는다.

## 13. 기술 선택

MVP:

- local single Controller
- Python 또는 운영 편의 높은 단일 언어
- YAML config
- JSON artifacts
- SQLite state
- Git worktree
- OpenCode backend
- Herdr optional

실제 병목 전에는 Redis, distributed queue, workflow DSL을 추가하지 않는다.

## 14. 아키텍처 불변조건

1. Orchestrator가 executable state authority를 소유하지 않는다.
2. Frozen TaskSpec을 worker 자연어가 수정하지 않는다.
3. Worker output이 canonical state/evidence를 직접 기록하지 않는다.
4. Project workspace와 Controller state/artifact store를 분리한다.
5. capability claim은 actual enforcement level을 따른다.
6. CheckRunner 실행도 untrusted code execution 가능성을 고려한다.
7. Runtime/backend failure와 task/check failure를 구분한다.
8. DONE은 Controller verification gate를 통과해야 한다.
9. UI는 canonical protocol이 아니다.
10. OpenCode 세부사항은 adapter 뒤에 둔다.

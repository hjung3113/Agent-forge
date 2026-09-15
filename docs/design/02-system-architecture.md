# 02. System Architecture

## 1. 상위 구조

```text
                              User
                               |
                               v
                     +-------------------+
                     | Orchestrator Agent|
                     +---------+---------+
                               |
                      Spawn / Delegate Req
                               |
                               v
+----------------------------------------------------------------+
|                    Agent Forge Controller                       |
|                                                                |
|  +----------------+   +----------------+   +----------------+   |
|  | Agent Registry |   | Skill Registry |   |Project Registry|   |
|  +----------------+   +----------------+   +----------------+   |
|                                                                |
|  +----------------+   +----------------+   +----------------+   |
|  | Task Contract  |   | Scheduler /    |   | Policy / Gate  |   |
|  | Resolver       |   | State Machine  |   |                |   |
|  +----------------+   +----------------+   +----------------+   |
|                                                                |
|  +----------------+   +----------------+   +----------------+   |
|  | Harness        |   | Workspace      |   | Result / Event |   |
|  | Compiler       |   | Manager        |   | Store          |   |
|  +----------------+   +----------------+   +----------------+   |
|                                                                |
|                     +------------------+                       |
|                     | Runtime Adapter  |                       |
|                     +---------+--------+                       |
+-------------------------------|--------------------------------+
                                v
                           +----------+
                           | OpenCode |
                           +----+-----+
                                |
            +-------------------+-------------------+
            |                   |                   |
            v                   v                   v
       Project Expert      Implementer         Reviewer/Verifier
       Worktree A          Worktree B          scoped workspace

                     Herdr (optional UI)
                  observes state/process/logs
```

## 2. 책임 분리

### 2.1 Orchestrator Agent

의미 기반 판단을 담당한다.

- 사용자 목표 해석
- task decomposition 제안
- 어떤 Agent가 필요한지 선택
- Agent 호출 순서/의존성 제안
- 실패 결과를 보고 추가 분석 Agent 요청

Orchestrator는 다음을 직접 소유하지 않는다.

- subprocess lifecycle
- worktree 생성/삭제
- permission enforcement
- retry budget
- task state transition
- 완료 판정의 최종 authority

### 2.2 Controller

Agent Forge의 실제 control plane이다.

- 요청 validation
- Agent/Profile resolve
- dependency/concurrency 검사
- task/step state transition
- worktree lifecycle
- runtime process 실행
- timeout/retry/cancel
- policy enforcement
- artifact 수집
- verification gate

Controller는 가능하면 deterministic code로 구현한다.

### 2.3 Runtime Adapter

Controller와 실제 Agent backend의 경계다.

초기 구현:

```text
RuntimeAdapter
  -> OpenCodeAdapter
```

향후 backend가 추가되더라도 orchestration core가 backend-specific CLI 문법을 알지 않도록 한다.

예시 interface:

```text
run(request) -> run_result
cancel(run_id)
health() -> status
```

`run(request)`에는 최소한 다음이 포함된다.

- cwd
- resolved agent profile
- prompt/task contract
- runtime timeout
- output/log paths
- environment allowlist

### 2.4 Registry 계층

Registry는 문자열 이름을 실제 실행 계약으로 resolve한다.

```text
AgentRegistry
SkillRegistry
ProjectRegistry
DomainRegistry (optional initial / explicit later)
HarnessRegistry
```

Registry의 목적은 Agent 정의를 코드에 하드코딩하지 않는 것이다.

### 2.5 Harness Compiler

여러 profile을 최종 실행환경으로 합성한다.

```text
Base
 + Role
 + Project
 + Domain
 + Explicit Skills
 + Task Contract
        |
        v
Resolved Harness
```

merge precedence와 conflict rule은 [03-agent-composition-and-harness.md](03-agent-composition-and-harness.md)에서 정의한다.

### 2.6 Workspace Manager

- canonical repository clone/cache 관리
- fetch/update
- task branch/worktree 생성
- cwd 결정
- baseline snapshot
- diff 수집
- 허용 범위 밖 변경 탐지
- cleanup

### 2.7 Verification subsystem

LLM review와 deterministic check를 분리한다.

```text
LLM Reviewer
  -> correctness / design / risk review

Verifier
  -> acceptance criteria evidence aggregation

Deterministic Checks
  -> build / test / lint / typecheck / git diff / path policy
```

## 3. Control Plane과 Data Plane

### Control Plane

```text
Orchestrator request
Registry
Scheduler
State Machine
Policy Gate
Harness Compiler
Workspace Manager
```

### Data/Execution Plane

```text
OpenCode processes
Git worktrees
Project source
Prompt/context artifacts
stdout/stderr
outputs/diffs/check results
```

Herdr는 Control Plane 자체가 아니라 **관측 surface**다.

## 4. 내부 데이터 흐름

```text
1. User request
2. Task Contract 생성/정규화
3. Orchestrator가 delegate 요청 생성
4. Controller가 request schema 검증
5. AgentRegistry resolve
6. Project가 있으면 workspace resolve/create
7. Harness Compiler가 실행 profile 생성
8. Policy Gate preflight
9. OpenCodeAdapter.run(cwd=worktree)
10. stdout/stderr/result 수집
11. post-run path/diff policy 검사
12. output contract 검사
13. 필요 시 review/verify step
14. deterministic checks
15. state transition
16. Orchestrator에 structured result 반환
17. stop condition 충족 시 complete
```

## 5. 권장 저장 구조

초기 구현은 다음 정도면 충분하다.

```text
agent-forge/
├─ src/
│  ├─ controller/
│  ├─ orchestration/
│  ├─ runtime/
│  ├─ registry/
│  ├─ harness/
│  ├─ workspace/
│  ├─ policy/
│  └─ verification/
│
├─ agents/
├─ roles/
├─ projects/
├─ domains/
├─ skills/
├─ harnesses/
├─ config/
└─ .agent-forge/
   ├─ tasks/
   ├─ runs/
   ├─ events/
   ├─ results/
   └─ logs/
```

runtime data는 repository source와 구분한다. 프로젝트 source에는 Agent Forge 전용 상태를 최소한만 남긴다.

## 6. 기술 선택 가이드

MVP에서 필요한 것은 복잡한 인프라가 아니다.

- implementation language: Python 또는 현재 운영 편의가 높은 단일 언어
- process: local subprocess
- config: YAML
- contracts/events: JSON
- state: 처음에는 JSON 또는 SQLite
- repository isolation: Git worktree
- agent backend: OpenCode
- operator UI: Herdr optional

SQLite는 task 수, restart recovery, event query가 필요해지는 시점부터 권장한다. Redis/message broker는 멀티프로세스/멀티머신 요구가 생기기 전에는 추가하지 않는다.

## 7. 아키텍처 불변조건

1. Orchestrator가 OS process를 직접 관리하지 않는다.
2. Agent가 다른 Agent pane을 직접 조작하는 것이 필수 프로토콜이 아니다.
3. Project code 변경은 승인된 workspace root 안에서만 일어난다.
4. Reviewer의 입력은 구현 Agent의 private reasoning에 의존하지 않는다.
5. Runtime backend failure와 task logic failure를 구분한다.
6. Prompt permission은 hard permission으로 간주하지 않는다.
7. 완료 상태는 verification evidence 없이 선언하지 않는다.
8. Agent Forge core는 OpenCode CLI 세부 옵션에 직접 결합되지 않는다.

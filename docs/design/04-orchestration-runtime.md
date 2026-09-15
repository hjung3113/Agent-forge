# 04. Orchestration & Runtime

## 1. 목적

Agent Forge의 핵심 경계는 **Orchestrator Agent와 deterministic Controller를 분리하는 것**이다.

```text
User Goal
  -> Intake / TaskSpec
  -> Orchestrator semantic decision
  -> structured DelegateRequest
  -> Controller validation
  -> Step / RunAttempt execution
  -> structured result/evidence
  -> Controller completion gate
```

TaskSpec 변경 규칙은 [11-task-contract-and-change-control.md](11-task-contract-and-change-control.md), 상태/복구의 canonical 정의는 [13-state-recovery-and-artifact-integrity.md](13-state-recovery-and-artifact-integrity.md)를 따른다.

## 2. Orchestrator 책임

Orchestrator는 다음을 할 수 있다.

- 사용자 목표를 구조화할 초안 제안
- 필요한 전문 Role/Agent 선택
- task decomposition과 dependency 제안
- 결과를 보고 다음 Step 제안
- 추가 evidence/분석 Agent 요청
- 완료 후보 제안
- TaskAmendment 제안

다음은 직접 하지 않는다.

- subprocess/PID 관리
- worktree 생성/삭제
- policy override
- TaskSpec 직접 수정
- retry budget 임의 증가
- canonical task/run state 기록
- 검증 없이 DONE 선언

## 3. Controller 책임

Controller가 소유한다.

- TaskSpec normalize/freeze/hash
- DelegateRequest validation
- Step 생성/의존성 검증
- RunAttempt lifecycle
- concurrency/retry budget
- workspace lease
- capability/policy gate
- Runtime Adapter 호출
- CheckRunner
- evidence/artifact 저장
- VerificationPlan 계산
- completion state transition

의미 판단과 실행 authority를 한 컴포넌트에 섞지 않는다.

## 4. DelegateRequest

예:

```json
{
  "type": "delegate",
  "task_id": "T-001",
  "task_spec_hash": "...",
  "step_kind": "analysis",
  "agent": "log-expert",
  "project": "logwarehouse",
  "skills": ["carryover-analysis"],
  "depends_on_steps": [],
  "expected_output": "analysis"
}
```

Controller는 존재하는 Agent/Profile, dependency, budget, scope, capability requirement를 검증한다.

## 5. Task / Step / RunAttempt

```text
Task T-001
  |- Step S1 project grounding
  |    `- Attempt A1
  |- Step S2 implement
  |    |- Attempt A1 FAILED
  |    `- Attempt A2 SUCCEEDED
  `- Step S3 verify
       `- Attempt A1
```

- Task: 사용자 목표
- Step: 논리적 작업
- RunAttempt: 실제 runtime 실행 1회

Retry는 새 RunAttempt다.

## 6. State authority

- Orchestrator: 전이 제안
- Controller: canonical transition authority
- Reviewer/Verifier: evidence/finding 제공
- User/authorized policy: TaskAmendment/수동 gate 해제 가능

`OpenCode exit=0`, `worker says done`, `reviewer pass` 중 어느 것도 단독 상태 authority가 아니다.

## 7. Recursive delegation 제한

Worker는 다른 Worker를 직접 spawn하지 않는다.

```text
Worker
  -> delegate recommendation
  -> Orchestrator/Controller
  -> validated new Step/Attempt
```

권장 초기 budget:

```yaml
orchestration:
  max_depth: 2
  max_active_attempts_per_task: 3
  max_total_attempts_per_task: 12
  max_retry_per_step: 2
```

실제 값은 Project/System Policy에서 조정한다.

## 8. Planning 전략

항상 Planner/Architect를 호출하지 않는다.

```text
Small task
  -> Implement/Debug -> Verify

Architecture-sensitive
  -> Ground -> Architect -> Implement -> Review -> Verify
```

Role 수보다 Step의 목적과 evidence contract를 명확히 한다.

## 9. Reviewer / Verifier

Reviewer:

- semantic correctness/design/risk finding
- final diff와 selected evidence 중심
- Implementer private reasoning 기본 미제공

Verifier:

- AC별 evidence mapping
- required check 결과
- unresolved blocking finding
- scope/policy 상태

Controller CheckRunner가 만든 deterministic evidence가 Worker의 자기보고보다 우선한다.

## 10. Verification floor

Orchestrator가 reviewer를 생략하자고 제안해도 Project/System Policy가 요구하면 생략할 수 없다.

VerificationPlan은 [14-evaluation-and-conformance.md](14-evaluation-and-conformance.md)의 change classification과 required checks를 따른다.

예:

- architecture boundary 변경 -> architecture check/reviewer 강제 가능
- verification config 변경 -> stronger review/control check
- DB/security/public API 변경 -> project-defined mandatory path

## 11. Retry

Retry reason을 구분한다.

```text
runtime failure
task/check failure
review failure
output-contract failure
policy violation
configuration failure
```

Retry마다 무엇이 달라지는지 기록한다.

TaskSpec/AC를 약화시키는 것은 retry가 아니라 TaskAmendment 대상이며, hard policy는 amendment로도 완화할 수 없다.

## 12. Cancellation

Controller가 cancellation을 소유한다.

1. 신규 Attempt 생성 중지
2. Runtime Adapter에 process-tree cancel
3. stdout/stderr flush
4. Attempt state 기록
5. workspace lease 해제/보존 정책 수행
6. partial diff/artifact 보존

Process containment 세부는 [12-runtime-isolation-and-trust-boundaries.md](12-runtime-isolation-and-trust-boundaries.md)를 따른다.

## 13. Orchestrator context

총괄 Agent에 모든 원문을 누적하지 않는다.

```text
Frozen TaskSpec summary/hash
+ rolling task summary
+ authoritative Step results
+ selected findings/evidence references
```

과거 실패 Attempt의 원문은 필요할 때만 읽는다.

## 14. 완료 조건

Controller는 최소 다음을 확인한 뒤 DONE을 허용한다.

- FROZEN TaskSpec 존재
- required Step의 authoritative Attempt 성공
- 모든 blocking AC에 valid evidence 존재
- VerificationPlan의 required checks 성공
- unresolved blocking finding 없음
- scope/path/policy 위반 없음
- trusted artifact manifest complete
- recovery/lease 관점에서 active orphan Attempt 없음

## 15. 설계 불변조건

1. Orchestrator는 semantic decision을 하고 Controller가 executable state를 소유한다.
2. TaskSpec은 실행 중 자연어로 변경되지 않는다.
3. Retry는 Step 아래 새 Attempt다.
4. Worker가 다른 Agent를 직접 spawn하지 않는다.
5. verification floor는 Orchestrator가 낮출 수 없다.
6. DONE은 trusted evidence와 Controller gate를 요구한다.

# 04. Orchestration & Runtime

## 1. 목적

Agent Forge의 가장 중요한 경계는 **Orchestrator Agent와 deterministic Controller를 분리하는 것**이다.

Orchestrator는 의미를 판단하고, Controller는 실행 가능성과 상태를 결정한다.

```text
User Goal
  -> Orchestrator reasoning
  -> structured DelegateRequest
  -> Controller validation
  -> Agent execution
  -> structured AgentResult
  -> Orchestrator next decision
```

## 2. Orchestrator Agent 책임

Orchestrator는 다음을 할 수 있다.

- 사용자 목표를 Task Contract로 정리
- 필요한 전문 Agent 선택
- 작업을 작은 단계로 나눔
- 단계 간 dependency 제안
- 결과를 보고 추가 Agent 요청
- 실패 원인을 분석 Agent에 재위임
- 충분한 evidence가 모였을 때 완료 후보 제안

다음은 직접 하지 않는다.

- 다른 Herdr pane에 키 입력
- subprocess PID 관리
- timeout kill
- worktree 생성/삭제
- 정책 우회
- retry 횟수 임의 증가
- 검증 없이 DONE 상태 기록

## 3. DelegateRequest

Orchestrator와 Controller 사이에는 자연어가 아니라 구조화된 계약을 둔다.

예시:

```json
{
  "type": "delegate",
  "agent": "log-expert",
  "task": "Carryover 규칙과 현재 구현 차이를 분석",
  "project": "logwarehouse",
  "skills": ["standard-log", "carryover-analysis"],
  "depends_on": [],
  "scope": ["src/Parser/**", "docs/**"],
  "expected_output": "analysis"
}
```

Controller는 다음을 검증한다.

- 존재하는 Agent인가
- Project/Profile이 유효한가
- 요청 skills가 허용되는가
- dependency가 완료되었는가
- recursive delegation depth가 한도 내인가
- concurrency slot이 있는가
- scope와 permission이 충돌하지 않는가

## 4. AgentResult

Worker는 Orchestrator에 원시 터미널 화면을 반환하는 대신 구조화된 결과를 남긴다.

```json
{
  "run_id": "R-0014",
  "agent": "log-expert",
  "status": "succeeded",
  "summary": "...",
  "artifacts": ["results/R-0014.md"],
  "evidence": ["docs/spec/carryover.md", "src/Parser/..."],
  "warnings": [],
  "next_recommendations": []
}
```

stdout/stderr 원문은 별도 artifact로 보존하고, orchestration contract와 섞지 않는다.

## 5. Task State Machine

초기 상태 모델은 단순하게 유지한다.

```text
PENDING
  -> READY
  -> RUNNING
  -> VERIFYING
      -> DONE
      -> RETRY
      -> BLOCKED
      -> FAILED

RETRY -> READY
```

### 상태 authority

- Orchestrator: 상태 전환을 **제안** 가능
- Controller: 실제 상태를 기록하는 authority
- Reviewer/Verifier: evidence 제공
- User: 정책상 사람이 필요한 BLOCKED 상태 해제 가능

## 6. Run State

Task 안에서 각 Agent 호출은 별도 Run이다.

```text
Task T-001
  |- Run R-001 project-expert
  |- Run R-002 log-expert
  |- Run R-003 implementer
  |- Run R-004 reviewer
  `- Run R-005 verifier
```

이렇게 분리해야 실패/재시도/로그/비용을 개별 추적할 수 있다.

## 7. Recursive Delegation 제한

Orchestrator가 필요한 Agent를 동적으로 부르는 기능은 유용하지만 무제한 recursive swarm은 금지한다.

권장 기본 정책:

```yaml
orchestration:
  max_depth: 2
  max_active_runs_per_task: 3
  max_total_runs_per_task: 12
  max_retry_per_run: 2
```

전문 Agent가 또 다른 Agent를 직접 spawn하는 대신 기본적으로 **요청을 Controller/Orchestrator에 반환**하도록 한다.

```text
Worker A
  -> recommends_delegate(B)
  -> Controller/Orchestrator
  -> spawn B
```

이를 통해 누가 Agent를 생성했는지 추적 가능하고 runaway loop를 막는다.

## 8. Planning 전략

항상 Planner를 호출하지 않는다.

```text
Simple task
  -> Orchestrator -> Implementer -> Verify

Complex task
  -> Orchestrator -> Expert/Planner -> Implementer(s) -> Review -> Verify
```

Controller는 task complexity를 LLM 대신 완벽히 판단하려 하지 않는다. Orchestrator가 plan을 요청하되, 실행할 수 있는 최대 fan-out과 depth만 정책으로 통제한다.

## 9. Reviewer / Verifier 분리

### Reviewer

의미 기반 품질 검토.

입력:

- Task Contract
- relevant source
- git diff
- test/check outputs
- project/domain rules

기본적으로 제외:

- implementer의 chain-of-thought
- implementer의 자기평가
- 불필요한 전체 task transcript

### Verifier

Acceptance Criteria를 evidence에 매핑한다.

예:

```text
AC1: regression test exists
 -> evidence: tests/CarryoverTests.cs diff

AC2: test passes
 -> evidence: dotnet test exit=0

AC3: scope outside Parser unchanged
 -> evidence: changed-path validator pass
```

Reviewer가 `pass`라고 말해도 deterministic check가 실패하면 DONE이 아니다.

## 10. Retry 정책

Retry reason을 구분한다.

### Runtime retry

- temporary OpenCode failure
- timeout
- malformed/empty output

### Task retry

- test fail
- review fail
- output contract fail

### Non-retryable

- auth/runtime unavailable
- project config invalid
- permission conflict
- forbidden operation required

동일한 실패를 무작정 반복하지 않는다. retry마다 **무엇이 달라지는지** 있어야 한다.

## 11. Cancellation

Controller가 cancellation token을 소유한다.

취소 시:

1. 신규 Run spawn 중지
2. 실행 중 OpenCode process terminate
3. stdout/stderr flush
4. Run 상태 `cancelled`
5. worktree는 정책에 따라 keep/remove
6. partial diff와 artifact 보존

## 12. Orchestrator Context 관리

총괄 Agent가 모든 원문을 계속 누적하면 context가 폭발한다.

권장 방식:

```text
Task Contract
+ rolling task summary
+ latest RunResult summaries
+ selected evidence references
```

원본 stdout, diff, 큰 문서는 필요할 때 파일 경로로 읽게 한다.

ForgeRoom의 Conductor에서 유효했던 원칙처럼 **summary 상태 전이 자체는 가능한 한 코드가 소유하고 LLM은 서사 요약만 생성**하도록 한다.

## 13. 완료 조건

Controller는 최소 다음을 만족해야 DONE을 허용한다.

- 필수 Run 성공
- Task Contract의 acceptance criteria가 evidence와 연결됨
- required deterministic checks 성공
- unresolved blocking review finding 없음
- scope/path policy 위반 없음
- worktree 상태 수집 완료

Orchestrator의 "완료했습니다"라는 자연어만으로 DONE으로 바꾸지 않는다.

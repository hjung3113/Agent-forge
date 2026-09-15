# 11. Task Contract & Change Control

## 1. 목적

Task Contract는 Agent Forge에서 사용자 의도와 완료 조건을 고정하는 **canonical execution specification**이다.

Orchestrator가 자연어 요청을 구조화할 수는 있지만, 실행 중 실패를 해결하기 위해 scope나 acceptance criteria를 조용히 바꾸면 안 된다.

```text
User Request
  -> Intake / Grounding
  -> TaskSpec DRAFT
  -> Controller normalize / validate
  -> TaskSpec FROZEN
  -> task_spec_hash
  -> Steps / RunAttempts
```

이 문서는 TaskSpec의 lifecycle, 변경 권한, amendment, base revision pinning을 정의한다.

## 2. Canonical artifacts

### 2.1 TaskSpec

이번 task가 무엇을 해야 하는지를 정의한다.

최소 schema:

```yaml
schema_version: 1
task_id: T-001
objective: "Carryover validation bug 수정"
project: logwarehouse
base_revision: "<exact-commit-sha>"
scope:
  include:
    - src/Parser/**
  exclude:
    - db/**
constraints:
  - no database schema change
acceptance_criteria:
  - id: AC-1
    statement: regression test demonstrates the original failure
    evidence_kind: deterministic
  - id: AC-2
    statement: required project tests pass
    evidence_kind: deterministic
  - id: AC-3
    statement: no out-of-scope source change
    evidence_kind: deterministic
verification_hints:
  - dotnet test
stop_condition: all blocking acceptance criteria and controller-required verification pass
```

`verification_hints`는 사용자/Orchestrator의 의도다. 실제 최소 verification은 Controller가 Project Policy와 change classification으로 계산하며 TaskSpec보다 약해질 수 없다.

### 2.2 VerificationPlan

Controller가 다음을 합성해 만드는 파생 artifact다.

```text
TaskSpec
+ Project required checks
+ System hard policy
+ change/risk classification
+ architecture-sensitive detection
= VerificationPlan
```

Orchestrator나 Worker는 더 강한 verification을 제안할 수 있지만 required verification을 제거할 수 없다.

### 2.3 TaskAmendment

FROZEN TaskSpec을 바꾸는 유일한 경로다.

```yaml
schema_version: 1
amendment_id: TA-003
task_id: T-001
old_task_spec_hash: "..."
reason: "required fix crosses originally unknown module boundary"
requested_by: orchestrator
requested_change:
  scope_add:
    - src/Common/Carryover/**
authorization:
  type: user
  reference: "..."
new_task_spec_hash: "..."
```

Worker output이나 free-form summary가 TaskSpec을 변경한 것으로 간주되지 않는다.

## 3. TaskSpec lifecycle

```text
DRAFT
  -> NORMALIZED
  -> FROZEN
      -> execution
      -> amendment requested
          -> APPROVED -> new FROZEN revision
          -> REJECTED -> original spec remains
```

RunAttempt 시작 시 반드시 특정 `task_spec_hash`를 참조한다. FROZEN 이전의 intake/briefing 활동([13-state-recovery-and-artifact-integrity.md](13-state-recovery-and-artifact-integrity.md)의 IntakeSession)은 RunAttempt가 아니다.

## 4. 변경 권한

| 변경 | Orchestrator/Worker | Controller policy | User/authorized authority |
|---|---:|---:|---:|
| scope 확장 제안 | propose | No | approve |
| objective 의미 변경 | propose | No | approve |
| acceptance criteria 제거/완화 | propose | No | approve |
| security/policy 완화 | No | No | hard policy가 허용하지 않으면 불가 |
| 더 강한 deterministic check 추가 | propose | Yes | not required |
| 더 제한적인 permission 적용 | propose | Yes | not required |
| retry budget 증가 | propose | bounded policy only | policy 밖이면 approve |
| exact base revision 변경 | propose | re-ground required | policy/user authorization |

Controller가 자동으로 적용할 수 있는 것은 **의도를 바꾸지 않는 강화**다.

예:

- permission 축소
- 추가 test/check
- 추가 evidence requirement
- 더 좁은 runtime capability

반대로 목표, scope, AC를 실질적으로 바꾸는 것은 amendment가 필요하다.

## 5. Retry 불변조건

Retry는 TaskSpec 변경이 아니다.

```text
same Step
  Attempt 1 -> failed
  Attempt 2 -> changed execution strategy
```

허용:

- context 보강
- 다른 Role 선택
- correction prompt
- 구현 수정
- runtime retry

금지:

- 실패한 AC 삭제
- failing test를 required set에서 제거
- scope 밖 변경을 새 scope로 사후 합법화
- verification command를 더 약한 command로 교체

Retry마다 `retry_reason`과 `delta_from_previous_attempt`를 기록한다.

## 6. Base revision pinning

TaskSpec이 FROZEN될 때 project base는 branch 이름이 아니라 **exact commit SHA**로 고정한다.

```text
main
  -> resolve once
  -> 4ac9... exact SHA
  -> TaskSpec.base_revision
```

실행 중 remote `main`이 이동해도 현재 task baseline은 변하지 않는다.

새 base가 필요하면:

1. amendment 생성
2. repository re-ground
3. scope/AC 영향 확인
4. new TaskSpec revision freeze

을 거친다.

## 7. Acceptance Criterion 규칙

각 AC에는 stable id를 둔다.

권장 분류:

```text
deterministic
  -> test/build/static/path/check 결과

semantic
  -> reviewer finding/evidence

manual
  -> 사람이 확인해야 하는 외부 조건
```

`unknown`을 `pass`로 변환하지 않는다.

Manual evidence가 blocking이면 자동 DONE을 허용하지 않는다.

## 8. TaskSpec provenance

FROZEN artifact에는 최소 다음을 기록한다.

- schema version
- task id
- source request reference
- normalized timestamp
- project id
- exact base commit SHA
- objective/scope/constraints/AC
- authoring actor
- applied system/project policy versions
- content hash

Run artifact에는 현재 TaskSpec 내용을 복사하는 대신 immutable snapshot/hash reference를 남긴다.

## 9. Schema evolution

TaskSpec/Amendment schema는 version을 가진다.

원칙:

- 과거 artifact를 현재 schema로 조용히 재해석하지 않는다.
- migration은 explicit converter로 수행한다.
- migration 전/후 hash와 schema version을 기록한다.
- unsupported old schema는 실행이 아니라 inspect-only로 남길 수 있다.

## 10. 설계 불변조건

1. FROZEN TaskSpec은 직접 수정하지 않는다.
2. Orchestrator와 Worker는 amendment를 제안할 수 있지만 적용 authority가 아니다.
3. Retry가 실패한 요구사항을 약화시키는 수단이 되어서는 안 된다.
4. Controller-required verification은 TaskSpec hint보다 우선하며 더 강할 수 있다.
5. task base revision은 exact commit SHA로 고정한다.
6. 모든 RunAttempt는 TaskSpec hash를 참조한다.
7. AC에는 stable id와 evidence kind가 있어야 한다.
8. security hard policy는 amendment로 완화되지 않는다.

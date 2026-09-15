# Agent Forge

Agent Forge는 제약된 사내 환경에서 OpenCode를 실행 백엔드로 사용해 role·project·domain·skill별 agent를 조합·실행하는 로컬 멀티에이전트 control plane이다. 이 문서는 설계 논의에서 반복적으로 등장하는 용어를 정의한다.

## Language

**Controller**:
Task/Attempt/state/evidence/completion을 소유하는 deterministic 실행 권한. LLM이 아니라 코드다.
_Avoid_: Orchestrator, engine, runtime (역할이 다름)

**Orchestrator**:
어떤 agent와 작업이 필요한지 제안하는 semantic layer. spawn, 권한, 상태 전이, 완료 판정에 대한 권한은 없다.
_Avoid_: Controller, coordinator

**TaskSpec**:
하나의 작업 단위를 정의하는 frozen contract(objective, scope, AC, base revision). FROZEN 이후에는 amendment로만 바뀐다.
_Avoid_: task, spec, ticket

**TaskAmendment**:
FROZEN된 TaskSpec의 objective/scope/AC/base revision을 바꾸는 유일한 경로. Retry나 execution strategy 변경과는 다른 개념이다.
_Avoid_: scope change, edit

**Attempt (RunAttempt)**:
하나의 Step을 실행하려는 개별 시도. Retry는 새로운 Attempt를 만드는 것이지 기존 Attempt를 재사용하는 것이 아니다.
_Avoid_: run, execution, try

**Authoritative Attempt**:
Step의 완료를 대표하는 것으로 기록된 Attempt. 이 값이 바뀌거나 지워지는 것은 일반적인 상태 전이가 아니라 명시적인 operation이다.
_Avoid_: winning attempt, final attempt

**ExecutionRevocation**:
아직 실행 중이거나 시작되지 않은 Attempt를 향후 진행하지 못하게 막는 operation. cancel이 실행되기 전에 기록되며, 그 자체로는 이미 끝난 결과의 수용 여부를 바꾸지 않는다.
_Avoid_: cancellation (더 넓은 개념과 혼동됨), revoke

**ResultInvalidation**:
이미 EXITED/VALIDATING/SUCCEEDED 상태인 Attempt의 결과를 completion 근거로 더 이상 쓸 수 없게 만드는 operation. 원본 hash와 이력은 보존되며, 나중에 완화(loosening)로 다시 authoritative가 될 수 없다.
_Avoid_: rollback, undo

**TaskCompletionValidity**:
DONE으로 기록된 Task의 완료가 여전히 유효한지를 나타내는 별도 레코드. Task 상태 자체를 REOPENED로 되돌리지 않고, `DONE(voided)`처럼 완료 이후에도 무효화를 표현하기 위한 개념이다.
_Avoid_: reopen, undo done

**Safety Profile**:
어떤 실행(Worker Attempt 또는 CheckRunner 호출)이 시작되기 위해 만족해야 하는, 이름이 붙은 required claim의 집합. System policy의 재사용 가능한 조각이며 Project가 강화(tighten)만 할 수 있다.
_Avoid_: sandbox policy, security level

**Claim (capability claim)**:
어떤 property(예: worktree 밖 쓰기 탐지)에 대해 실제로 관찰된 enforcement level을 property_id, threat-model 참조, observer, coverage와 함께 기록한 evidence record. 하나의 property에 대해 여러 claim이 공존할 수 있다(예: 지속적 변경은 탐지되지만 일시적 변조는 탐지되지 않음).
_Avoid_: capability level (claim보다 좁은 의미로 혼용되기 쉬움)

**Admission**:
하나의 execution이 시작되기 직전, 측정된 capability report가 적용 중인 Safety Profile의 required claim을 모두 만족하는지 확인하는 결정. 실행을 시작하거나 거부하는 결정이며, 허용된 모든 제약이 실제로 ENFORCED라는 증명이 아니다.
_Avoid_: authorization (더 넓은 개념), sandboxing

**effective_enforcement**:
어떤 capability가 실제 환경에서 관찰된 enforcement 수준(UNSUPPORTED/ADVISORY/DETECTABLE/ENFORCED). backend가 이론적으로 지원하는 수준(backend_support)과는 다른 개념이다.
_Avoid_: capability level alone (measured vs supported 구분이 없어짐)

**TCB (Trusted Computing Base)**:
Controller executable과 그 schema/transition/admission/completion 로직, 그리고 그 상태를 쓰는 저장 경로. 이 범위 밖의 registry YAML, Project Profile, pinned Skill은 승인·검증된 뒤에만 신뢰되는 입력(T1)이며, 그 자체로 TCB는 아니다.
_Avoid_: trusted zone, core (모호함)

**Consumption lineage (consumes_artifacts / source_snapshot_ref)**:
어떤 evidence artifact가 어떤 이전 Attempt·artifact·source snapshot을 실제로 참조했는지를 Controller가 구성해 기록한 정보. 이 lineage가 없으면 그 artifact에 의존하는 검증은 자동으로 재검증 대상(needs_revalidation)이 된다.
_Avoid_: provenance alone (더 넓은 개념), dependency

**First vertical slice**:
Orchestrator 없이 operator가 직접 FROZEN TaskSpec을 제출/승인하고 하나의 등록된 agent를 지정해 Controller가 실행·검증까지 완료하는 최소 구성. 문서화된 MVP(Phase 0–6 + minimal Phase 7)보다 좁은, 먼저 배포 가능한 하위 집합이다.
_Avoid_: MVP (문서화된 MVP와 범위가 다름)

**Change class**:
관찰된 diff/untracked set을 Controller가 policy에 정의된 규칙(경로, 파일 유형 등)으로 분류한 결과. `ordinary`(특별한 규칙에 걸리지 않음), `unknown`(분류 자체가 실패함), semantic uncertainty(내용은 신뢰할 수 있으나 분류가 불확실함)는 서로 다른 실패 모드이며 같은 방식으로 처리하지 않는다.
_Avoid_: risk level, category

**Policy-sensitive change**:
architecture baseline, waiver, harness/test 설정처럼 그 자체가 검증 기준을 정의하는 파일에 대한 변경. 통상적인 source edit과 달리 별도의 authorization artifact 없이는 DONE의 근거가 될 수 없다.
_Avoid_: sensitive file, protected path

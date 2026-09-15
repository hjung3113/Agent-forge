# 13. State, Recovery & Artifact Integrity

## 1. 목적

Agent Forge가 신뢰 가능한 control plane이 되려면 LLM 결과보다 먼저 **상태 전이와 evidence provenance**가 안정적이어야 한다.

이 문서는 Task/Step/RunAttempt, crash recovery, lease, Controller Artifact Store를 정의한다.

## 2. Canonical hierarchy

```text
Task
  -> Step
      -> RunAttempt
```

Retry는 새 RunAttempt이며 과거 Attempt를 덮어쓰지 않는다.

## 3. 상태 모델

### Task

```text
DRAFT -> FROZEN -> READY -> RUNNING -> VERIFYING
                                    -> BLOCKED | FAILED | CANCELLED
VERIFYING -> DONE | BLOCKED | FAILED
```

### Step

```text
PENDING -> READY -> RUNNING
                  -> SUCCEEDED | FAILED | BLOCKED | CANCELLED | SUPERSEDED
```

### RunAttempt

```text
CREATED -> STARTING -> RUNNING -> EXITED -> VALIDATING
                                      -> SUCCEEDED | FAILED | POLICY_VIOLATION
RUNNING -> TIMED_OUT | CANCELLED | LOST
```

Controller만 canonical state transition을 기록한다.

TaskSpec lifecycle([11-task-contract-and-change-control.md](11-task-contract-and-change-control.md) §3, `DRAFT -> NORMALIZED -> FROZEN`)은 artifact 상태이며 위 Task/Step/RunAttempt 상태와 별개 entity다. 두 lifecycle이 각각 `DRAFT`/`FROZEN` 이름을 갖지만 독립적으로 움직이는 두 개의 mutable state source가 아니다: **`Task.FROZEN`은 `TaskSpec.FROZEN`의 derived projection**이다. Task는 자신의 `FROZEN` 상태를 별도로 설정하지 않으며, 연결된 TaskSpec이 `FROZEN`이 되는 순간 Task도 `FROZEN`으로 전이한다. Task가 `READY`가 되려면 `FROZEN` TaskSpec(task_spec_hash)과 VerificationPlan이 이미 존재해야 하고, RunAttempt `CREATED`는 참조하는 task_spec_hash를 요구한다.

### IntakeSession

TaskSpec이 아직 없는 project-미지정 진입(cross-project briefing, 후순위 기능)은 Task/Step/RunAttempt 어디에도 속하지 않는 별도 entity다. TaskSpec 없음, worktree 없음, Attempt-retry/budget 모델 적용 없음.

IntakeSession은 **ephemeral/non-canonical**이다. §12 Artifact producer/provenance의 `task/step/attempt id` 요구, canonical state store의 durability/recovery 보장, §4 atomic transition 규칙 중 어느 것도 IntakeSession에는 적용되지 않는다. 출력 `SuggestedTask[]`는 evidence가 아니라 제안이며, replay/audit 대상이 아니다. 이 범위 제한이 유지되는 한 IntakeSession은 별도 schema 확장 없이 존재할 수 있다 — canonical state/event/artifact 모델에 새 필드를 추가하지 않는다.

사용자 확인 후에만 이로부터 `11`의 DRAFT TaskSpec이 생성되고, 그 순간부터 canonical Task 상태 모델과 §12 provenance 요구가 시작된다.

## 4. Atomic transition

상태 전이는 SQLite transaction/CAS(version) 방식으로 중복/경합을 방지한다.

예:

```text
UPDATE attempts
SET state='RUNNING', version=version+1
WHERE id=? AND state='STARTING' AND version=?
```

cancel과 completion이 경합하면 명시된 transition rule로 하나만 canonical하게 승리한다.

## 5. Authoritative Attempt

Step에는 현재 authoritative result pointer가 있다.

```text
step.authoritative_attempt_id = A2
```

이전 실패/취소 artifact는 history로 보존한다.

## 6. Retry contract

기록:

- previous attempt id
- retry reason
- delta from previous
- TaskSpec hash
- input manifest
- context/runtime delta

AC를 약화시키는 retry는 금지한다.

## 7. State store

MVP는 SQLite면 충분하다.

개념 table:

```text
tasks
steps
run_attempts
events
artifacts
leases
```

복잡한 event-sourcing framework는 만들지 않는다.

## 8. Lease / heartbeat

- attempt id
- controller instance id
- process identity
- workspace lease
- started/heartbeat/timeout

Timeout 계산에는 가능한 한 monotonic clock을 사용하고 audit 표시에는 wall-clock timestamp를 사용한다.

## 9. Startup reconciliation

```text
STARTING/RUNNING 조회
 -> process/lease 확인
 -> workspace 확인
 -> partial artifacts 확인
 -> resume / LOST / cleanup
```

불명확한 상태를 SUCCEEDED로 복구하지 않는다.

## 10. Workspace lease

같은 worktree에 writer 하나가 기본이다.

Reviewer/read-only 병행은 실제 ENFORCED isolation이 가능할 때만 고려한다.

## 11. Controller Artifact Store

Canonical artifact namespace는 worktree와 분리한다.

```text
~/.agent-forge/
├─ state/agent-forge.db
└─ tasks/T-001/
   └─ steps/S2/
      └─ attempts/A2/
         ├─ request.json
         ├─ input-manifest.json
         ├─ resolved-agent.yaml
         ├─ resolved-harness.yaml
         ├─ policy.json
         ├─ prompt.md
         ├─ stdout.log
         ├─ stderr.log
         ├─ diff.patch
         ├─ checks/
         ├─ output.md
         ├─ artifact-manifest.json
         └─ result.json
```

경로는 Attempt id로 키를 잡는다. retry는 새 `attempts/A3/` 디렉터리를 만들 뿐 기존 Attempt의 evidence를 덮어쓰지 않는다. Step의 authoritative attempt 포인터(어떤 Attempt 결과가 유효한지)와 history(보존된 모든 과거 Attempt)는 서로 다른 개념이다 — 포인터는 state db가 가리키고, history는 이 경로 전체가 보존한다.

이 store가 canonical이라는 뜻은 **Controller가 어떤 artifact를 state/evidence로 인정할지 결정한다**는 의미다.

같은 OS user의 Worker가 파일시스템 전체에 접근할 수 있다면 경로 분리만으로 physical tamper-proof를 보장하지 않는다.

따라서 artifact manifest에:

```text
storage_integrity_level: enforced | detectable | advisory
```

를 기록한다. 이 필드는 [12-runtime-isolation-and-trust-boundaries.md](12-runtime-isolation-and-trust-boundaries.md) §4의 capability enforcement level과 별도 3단계 vocabulary다 — storage integrity는 "능력이 존재하는가"가 아니라 "저장이 얼마나 강하게 보호되는가"를 표현하므로 `UNSUPPORTED`가 적용되지 않는다.

## 12. Artifact producer / provenance

최소:

```text
artifact_id
schema_version
producer_type
producer_id
task/step/attempt id
created_at
content_sha256
input_manifest_hash
task_spec_hash
base_commit_sha
storage_integrity_level
```

producer 예:

```text
controller
runtime-worker
check-runner
reviewer
verifier
operator
```

Worker artifact는 유용한 input일 수 있지만 producer type만으로 deterministic evidence가 되지는 않는다.

## 13. Check evidence

CheckRunner가 직접 관찰한:

- registered command id
- exact args
- cwd
- environment/isolation profile
- tool version
- start/end
- exit code
- captured output
- source revision

을 기록한다.

`worker says tests passed`는 deterministic evidence가 아니다.

### Runner trust class

check도 구분한다.

```text
controller_builtin
  path/hash/static validation

external_tool
  trusted/pinned analyzer binary

project_command
  build/test/script from project environment
```

`project_command`는 exit code를 Controller가 관찰하더라도 command semantics나 실행 안전성이 project source에 영향을 받을 수 있다.

고위험 task에서는 controller-owned fixture/static check 또는 별도 sandbox를 추가할 수 있다.

## 14. Verification config tamper

policy-sensitive:

- test/build config
- architecture baseline/waiver
- verification script
- CI config
- Agent Forge project instruction

변경 시 stronger review/control check가 필요할 수 있다.

## 15. Input Manifest

```json
{
  "task_spec_hash": "...",
  "base_commit": "...",
  "resolved_harness_hash": "...",
  "skill_hashes": ["..."],
  "project_profile_hash": "...",
  "agent_forge_revision": "...",
  "runtime_adapter_version": "...",
  "runtime_backend_version": "..."
}
```

완전 deterministic replay가 아니라 input traceability를 목표로 한다.

## 16. Log safety / quotas

- environment secret 최소화
- redaction
- artifact access boundary
- retention
- stdout/stderr size limit
- per-attempt/task disk budget
- truncation metadata

## 17. Idempotency

side effect에는 idempotency key를 둔다.

```text
create-workspace:T-001:rev3
start-attempt:S2:A2
record-check:A2:test-main
```

## 18. Schema evolution

Task/state/event/artifact schema는 version을 가진다.

migration은 explicit하고 과거 artifact hash를 조용히 바꾸지 않는다.

## 19. 설계 불변조건

1. Task/Step/RunAttempt를 구분한다.
2. state transition은 transaction/version guard를 사용한다.
3. retry는 과거 evidence를 덮어쓰지 않는다.
4. Controller가 canonical artifact/evidence 인정 authority다.
5. 저장 경로 분리 자체를 tamper-proof로 과장하지 않는다.
6. CheckRunner evidence authority와 command execution safety를 구분한다.
7. Attempt는 exact TaskSpec/base/harness input manifest를 가진다.
8. restart 시 state를 reconcile한다.
9. verification config 변경은 policy-sensitive다.
10. log/artifact도 보안/quota/retention 대상이다.

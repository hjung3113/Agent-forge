# 06. Safety, Verification & Observability

## 1. 원칙

Agent Forge는 LLM 지시 준수와 security/completion authority를 분리한다.

세부 runtime trust model은 [12-runtime-isolation-and-trust-boundaries.md](12-runtime-isolation-and-trust-boundaries.md), artifact/recovery는 [13-state-recovery-and-artifact-integrity.md](13-state-recovery-and-artifact-integrity.md), verification floor는 [14-evaluation-and-conformance.md](14-evaluation-and-conformance.md)가 canonical이다.

## 2. Threat model의 정직성

MVP의 기본 목표는:

- LLM 실수
- scope creep
- false completion
- accidental/agent-driven source mutation
- policy drift

를 통제하는 것이다.

Worker가 Controller와 동일 OS identity로 unrestricted host access를 가진 상태에서 **hostile arbitrary code에 대한 완전한 tamper resistance**까지 자동 보장한다고 주장하지 않는다.

그 수준이 필요하면 Runtime Isolation이 ENFORCED 수준의 별도 OS user/container/sandbox를 제공해야 한다.

## 3. Policy Gate

### Command

- approved runner/command class
- destructive operation
- remote write
- migration/reset
- unexpected package/tool install
- worktree 내 git config 변경 또는 hook-path 지정 ([05-project-workspace-and-context.md](05-project-workspace-and-context.md) §10의 canonical cache gitdir 변조 차단)

문자열 denylist는 보조 방어다.

### Path

- task scope
- denied path
- policy-sensitive path
- symlink/canonical path

### Spawn

- Agent/Profile 존재
- capability compatibility
- depth/concurrency/budget

### Workflow

- TaskSpec hash
- dependency
- required verification
- completion bypass

## 4. Capability level

모든 security 관련 capability는:

```text
ENFORCED
DETECTABLE
ADVISORY
```

중 하나로 보고한다.

Prompt의 `read-only`는 자동으로 ENFORCED가 아니다.

## 5. Verification layers

### Output/schema

형식/schema 충족 여부.

### Deterministic CheckRunner

- build
- test
- lint/typecheck
- path/scope
- architecture fitness
- project-specific static/control check

### Semantic Review

- 요구사항 의미 충족
- edge/regression
- architecture risk
- 불필요한 scope expansion

### Verifier

AC별 evidence와 blocking 상태를 모은다.

## 6. Evidence authority와 execution safety는 다르다

Controller가 직접 관찰한:

```text
command X exited 0
```

는 Worker의 자기보고보다 강한 evidence다.

하지만 command X가 project test/script라면 그 과정에서 arbitrary project code가 실행될 수 있다.

따라서 CheckRunner도:

- environment allowlist
- filesystem/network capability
- process-tree timeout
- output budget

등 Runtime Isolation policy를 적용받는다.

고위험 task에서는 project-modifiable test만으로 충분하지 않을 수 있으며 controller-owned fixture/static check를 추가할 수 있다.

## 7. Controller Artifact Store

Canonical evidence는 Controller가 수집/생성한 artifact만 인정한다.

Worker가 임의 파일 경로를 제출했다고 trusted evidence로 승격하지 않는다.

다만 물리 저장소가 Worker와 같은 OS permission domain에 있으면 `tamper-proof`라고 표현하지 않는다.

artifact metadata에 storage/runtime integrity level을 기록하고 더 강한 보안이 필요하면 ENFORCED isolation backend를 요구한다.

## 8. Verification Evidence

예:

```json
{
  "criterion": "AC-2",
  "status": "pass",
  "evidence": ["check:test-main:artifact-id"],
  "producer": "check-runner"
}
```

`unknown`은 pass가 아니다.

## 9. Reviewer 독립성

Reviewer에게 기본 제공:

- Frozen TaskSpec
- final/current diff
- relevant source
- Controller check results
- project/domain/architecture rules

미제공:

- implementer private reasoning
- 자기평가 서사

동일 모델 호출은 완전한 독립 intelligence가 아니라 context isolation에 의한 검토다.

## 10. Retry / Recovery

공통 failure taxonomy 예:

```text
runtime_unavailable
timeout
auth_failed
agent_error
output_contract_failed
policy_violation
check_failed
review_failed
configuration_error
lost_attempt
```

상태/lease/reconcile 세부는 `13` 문서를 따른다.

## 11. Event / Observability

Canonical event 예:

```text
task.frozen
step.ready
attempt.started
attempt.exited
attempt.policy_failed
check.completed
verification.failed
task.completed
```

Event에는 task/step/attempt id, actor, timestamp, reason, schema version을 포함한다.

UI는 event/state의 derived view다.

## 12. Herdr

Herdr는:

- process/log 관측
- state 표시
- operator cancel/intervention

에 사용한다.

pane 화면 문자열이 canonical task state가 되어서는 안 된다.

## 13. Completion Integrity

DONE에는 최소:

```text
Frozen TaskSpec
+ authoritative Step/Attempt results
+ VerificationPlan required checks
+ valid AC evidence
+ unresolved blocking finding 없음
+ policy/scope pass
+ artifact manifest
+ active orphan/lease 없음
```

이 필요하다.

완료 근거가 아닌 것:

```text
"완료했습니다"
OpenCode exit 0
Worker의 test pass 주장
근거 없는 reviewer pass
```

## 14. 설계 불변조건

1. Prompt permission을 security boundary로 부르지 않는다.
2. capability enforcement level을 명시한다.
3. CheckRunner도 untrusted code execution 가능성을 고려한다.
4. Worker-generated claim과 Controller-observed check를 구분한다.
5. Controller artifact store의 물리 tamper resistance를 실제 isolation 이상으로 과장하지 않는다.
6. verification floor를 LLM이 낮출 수 없다.
7. UI는 canonical state authority가 아니다.

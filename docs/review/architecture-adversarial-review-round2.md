# Architecture Adversarial Review Round 2 — 2026-09-15

## 1. 목적

기존 보강안을 다시 공격적으로 검토했다.

관점:

1. authority leakage
2. runtime trust boundary
3. crash/retry correctness
4. evidence integrity
5. capability claim accuracy
6. context/prompt injection
7. verification bypass
8. over-engineering/operability

## 2. BLOCKER — Frozen TaskSpec 부재

실패 후 scope/AC를 조용히 변경할 수 있는 위험.

반영:

- TaskSpec lifecycle/hash
- TaskAmendment
- exact base SHA
- retry와 requirement change 분리

상태: **resolved in design**

## 3. BLOCKER — Task -> Run만으로 retry가 모호

반영:

```text
Task -> Step -> RunAttempt
```

- authoritative Attempt pointer
- retry delta/reason
- 과거 artifact immutable history

상태: **resolved in design**

## 4. BLOCKER — Worker writable workspace와 canonical evidence 혼합

반영:

- Controller Artifact Store 분리
- Worker local staging 비-canonical
- CheckRunner 직접 evidence 수집

상태: **resolved in design, physical isolation caveat remains**

## 5. HIGH — Capability 이름이 실제 enforcement보다 강함

반영:

```text
ENFORCED | DETECTABLE | ADVISORY
```

Phase 0 공격 테스트로 실제 수준을 기록.

## 6. HIGH — Task/Skill requirement가 permission grant로 오해될 수 있음

반영:

```text
effective grant = ceiling intersection
required capability = requirements union
required > grant => BLOCKED
```

## 7. HIGH — Orchestrator가 verification을 하향할 수 있음

반영:

Controller `VerificationPlan` + change classification.

## 8. HIGH — 단일 PID cancel로 child process가 남음

반영:

process group/tree, timeout, escalation, orphan cleanup.

## 9. HIGH — Repository content prompt injection

반영:

```text
CONTROL
TRUSTED_PROJECT_INSTRUCTION
REFERENCE_CONTENT
```

provider-native instruction precedence는 Phase 0 실험 대상.

## 10. HIGH — Verification 자체를 수정해 pass 가능

공격:

- test config 약화
- architecture baseline 증가
- waiver 추가
- verification script 변경

반영:

policy-sensitive change class + stronger review/control check.

## 11. HIGH — Controller crash/restart semantics

반영:

SQLite, lease/heartbeat, versioned transition, startup reconciliation, LOST, idempotency.

## 12. MEDIUM — Base branch drift

TaskSpec freeze 시 exact SHA pin, 변경은 amendment + re-ground.

## 13. MEDIUM — Artifact/log secret leakage

host env allowlist, credential 미주입, redaction, quota, retention.

## 14. MEDIUM — Schema evolution

Task/state/event/artifact schema version + explicit migration.

## 15. 재리뷰 추가 Finding O — Controller store를 밖에 두는 것만으로 tamper-proof가 아님

**Severity: HIGH**

초기 보강안도 한 번 더 공격해 보니 중요한 과장이 있었다.

Worker가 Controller와 같은 OS user로 unrestricted process를 실행한다면:

```text
~/.agent-forge/runs
~/.agent-forge/state/agent-forge.db
```

도 원칙적으로 접근 가능할 수 있다.

즉:

```text
outside worktree != security boundary
```

이다.

추가 반영:

- threat model 명시
- Controller Artifact Store는 logical authority 분리로 정의
- storage integrity도 ENFORCED/DETECTABLE/ADVISORY 수준 기록
- store path 은닉을 security mechanism으로 사용하지 않음
- hostile same-user code까지 막아야 하면 OS identity/container/ACL backend 필요
- Phase 0에 Controller store tamper attempt 추가

상태: **resolved as an honest boundary; actual enforcement remains Phase 0/environment dependent**

## 16. 재리뷰 추가 Finding P — CheckRunner도 arbitrary code executor

**Severity: HIGH**

초기 설계는 "LLM 밖에서 test를 실행하므로 deterministic"이라는 면을 강조했다.

하지만:

```text
dotnet test
pytest
build.sh
npm test
```

는 project-controlled code를 실행할 수 있다.

따라서 두 축을 분리해야 한다.

```text
Evidence integrity
  Controller가 command/exit/output을 직접 관찰했는가?

Execution safety / semantic independence
  command 자체가 project code에 얼마나 종속되는가?
```

추가 반영:

- CheckRunner도 Runtime Isolation 대상
- runner trust class: controller_builtin / external_tool / project_command
- high-risk change에는 controller-owned fixture/static check 등 독립 evidence source 추가 가능
- adversarial eval에 test-weakening, test host-side-effect 추가

상태: **resolved in design**

## 17. MEDIUM — Cancel vs completion race

재리뷰 중 state transition race도 확인했다.

반영:

- SQLite transaction/version guard
- illegal/double transition 방지
- monotonic timeout + wall-clock audit

상태: **resolved in design**

## 18. LOW — 보안 프레임워크 과설계 위험

MVP는 그대로 제한한다.

```text
local Controller
SQLite
Git worktree
Controller artifact namespace
runtime-specific enforcement
```

Redis/distributed worker/full sandbox framework/workflow DSL은 실제 요구 전 추가하지 않는다.

## 19. 남은 현실적 위험

### R1 OpenCode 실제 capability

사내 설치 환경에서 실험 전이다. **Phase 0 blocker.**

### R2 OS-level isolation availability

WSL/회사 정책에 따라 다르다. 문서로 해결할 수 없다.

### R3 Same-model correlated reasoning

Reviewer context isolation과 deterministic checks로 완화하되 제거되지는 않는다.

### R4 Parallel merge/reconcile

MVP 범위 밖. Phase 8.

### R5 Trusted computing base bugs

Controller 자체의 state/policy/check 코드 버그는 여전히 가능하다.

따라서 conformance/adversarial suite를 Controller 변경의 release gate로 사용해야 한다.

## 20. 최종 재판정

핵심 architecture를 갈아엎을 필요는 없다.

하지만 구현 우선순위는 Agent intelligence가 아니라 다음이다.

```text
1. OpenCode + CheckRunner adversarial runtime spike
2. Frozen TaskSpec / Amendment
3. Task-Step-RunAttempt / atomic state / recovery
4. Controller Artifact authority + integrity level
5. Capability / isolation contract
6. Single Worker
7. Project worktree / context trust
8. Harness / Policy / CheckRunner
9. Architecture Fitness
10. Reviewer / Verifier
11. System adversarial eval
12. Orchestrator
```

판정:

**MVP architecture: proceed, conditional on Phase 0 results.**

현재 남은 가장 큰 미확정 요소는 설계가 아니라 실제 OpenCode/host 환경에서 어느 capability까지 ENFORCED로 만들 수 있는지다.

# Agent Forge MVP Roadmap

## 1. 구현 원칙

LLM orchestration보다 **trust/state/runtime foundation**을 먼저 만든다.

```text
Runtime/Check capability spike
 -> TaskSpec/state/artifact authority
 -> single worker
 -> project/worktree
 -> harness composition
 -> policy/checks
 -> architecture fitness
 -> reviewer/verifier
 -> adversarial eval
 -> orchestrator
```

## 2. Phase 0 — OpenCode + Execution Isolation Spike

실험:

- headless/cwd/output/exit
- timeout/cancel/process-tree cleanup
- project instruction precedence
- env inheritance
- outside-worktree read/write
- Controller store access/tamper attempt
- symlink traversal
- HOME/SSH/Git credential
- network
- child/background process
- huge output
- project test/build script host side effect
- concurrency/runtime config collision

모든 capability를:

```text
ENFORCED | DETECTABLE | ADVISORY | UNSUPPORTED
```

로 기록한다.

산출물:

- OpenCodeAdapter spike
- CheckRunner execution spike
- runtime-capability.md
- adversarial results

이 단계 결과가 문서 가정과 다르면 다음 phase보다 먼저 architecture claim을 수정한다.

항목별 실패 대응:

- **headless/cwd/output/exit가 UNSUPPORTED**: 진짜 blocker. 아래 의사결정 순서를 따른다.
- **그 외 항목이 UNSUPPORTED/ADVISORY**: degrade — capability report에 반영하고 Controller-side enforcement(policy gate, post-run diff 등)로 보완한다. Phase를 막지 않는다.

의사결정 순서(blocker 발생 시):

```text
1. invocation mode 대체(headless flag, pipe/stdin mode 등)
2. degraded 등급 항목만 포기하고 진행
3. 사내 승인된 대체 runtime 조사 후 adapter 교체
4. 모두 불가하면 명시적 중단 조건과 함께 중단
```

외부 API나 model gateway로 조용히 전환하지 않는다(01 비목표).

## 3. Phase 0.5 — Task / State / Artifact Foundation

- TaskSpec schema/freeze/hash
- TaskAmendment
- exact base SHA
- Task/Step/RunAttempt
- SQLite state + versioned transition
- lease/heartbeat
- startup reconciliation
- Controller Artifact Store
- artifact producer/provenance/integrity level
- input manifest
- schema version
- log quota/retention

완료 조건:

- crash/restart reconcile
- retry history 보존
- Worker output만으로 canonical evidence가 되지 않음
- physical storage integrity 수준을 정직하게 보고함

## 4. Phase 1 — Single Agent Runtime

- AgentRegistry 최소
- implementer Role
- required capability / grant resolver
- OpenCodeAdapter
- Attempt execution
- timeout/cancel
- output contract

## 5. Phase 2 — Project Registry + Worktree

- repo cache
- exact base SHA worktree
- branch naming
- writer lease
- diff/untracked
- allowed/denied/policy-sensitive paths
- registered commands
- context trust/provenance

## 6. Phase 3 — Role / Harness / Skill Composition

- Role/Skill/Harness Registry
- HarnessCompiler
- grant ceiling intersection
- requirement union
- conflict validation
- resolved snapshot/input manifest

## 7. Phase 4 — Policy + Functional Verification

- Command/Path/Spawn/Workflow gate
- change classification
- VerificationPlan
- CheckRunner
- runner trust class
- check isolation profile
- AC evidence mapping

완료 조건:

- failing required check blocks DONE
- verification downgrade blocked
- verification/test config weakening detected

## 8. Phase 4.5 — Architecture Fitness

- project architecture checks
- baseline/waiver
- blocking result
- CheckRunner integration

## 9. Phase 5 — Reviewer / Verifier

- independent context assembly
- finding schema
- AC pass/fail/unknown
- unresolved blocking finding gate
- same-model limitation 명시

## 10. Phase 5.5 — System Conformance & Adversarial Eval

최소:

- false DONE
- scope creep
- malicious project instruction
- evidence/store tamper
- verification/test weakening
- project test host side effect
- architecture baseline tamper
- timeout orphan
- controller crash/recovery
- retry loop
- branch drift
- schema mismatch
- secret-like output

## 11. Phase 6 — Orchestrator

- Orchestrator harness
- capability catalog
- DelegateRequest
- decomposition proposal
- bounded budget
- rolling summary
- amendment proposal

금지:

- process/worktree/state 직접 관리
- frozen TaskSpec 수정
- permission grant
- verification floor 하향

cross-project user briefing(개인비서형 진입)은 Phase 6 완료 조건이 아니며 후순위다(§16 참조).

## 12. Phase 7 — Skill Intake

- validate/audit/stage/evaluate/approve ([10-skill-intake-portability-and-evaluation.md](../design/10-skill-intake-portability-and-evaluation.md) §6 Stage 5 — script/tool을 포함한 Skill은 evaluate(Behavioral/Runtime) 없이 approve하지 않는다)
- revision/license/provenance
- script/network/fs inspection
- trigger/non-trigger
- capability compatibility
- hash

## 13. Phase 8 — Parallel Scheduling

실제 필요 후:

- dependency DAG
- bounded parallel attempts
- separate writer worktrees
- cancellation propagation
- explicit merge/reconcile

자동 merge는 MVP 아님.

## 14. Phase 9 — Management CLI

```text
agent-forge agent ...
agent-forge project ...
agent-forge task show/cancel
agent-forge run show
agent-forge architecture check
agent-forge eval run
```

## 15. Phase 10 — Herdr

관측/수동 개입 UI. 없어도 core는 정상 동작해야 한다.

## 16. MVP 정의

```text
Phase 0 ~ Phase 6
+ Phase 7 local Skill validate/audit 최소 기능
```

제외:

- Redis/distributed queue
- public marketplace
- multi-machine
- auto merge
- full GUI
- external model gateway
- 장기 자가학습 memory/wiki (auto-extraction)
- cross-project user briefing / personal inbox agent

## 17. 첫 end-to-end 기준

작은 bug task에서:

- TaskSpec freeze
- base SHA pin
- 최소 Role
- Worker claim과 Controller evidence 분리
- storage/runtime integrity level 기록
- scope/policy-sensitive change 탐지
- test failure DONE 차단
- crash/retry history
- evidence traceability

## 18. 새 기능 질문

1. 실제 현재 실행을 막는가?
2. deterministic code로 해결 가능한가?
3. trust boundary가 변하는가?
4. 새 source of truth가 생기는가?
5. crash/retry canonical state는 무엇인가?
6. evidence producer와 runner trust class는 무엇인가?
7. enforcement level을 실제로 증명할 수 있는가?
8. adversarial eval로 회귀를 잡을 수 있는가?

명확한 답이 없으면 MVP에 넣지 않는다.

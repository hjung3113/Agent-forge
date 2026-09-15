# Agent Forge Documentation Guide

이 문서는 Agent Forge 문서의 **인덱스이자 읽기 가이드**다. 같은 사실을 여러 문서에서 중복 정의하지 않고 각 문서의 canonical responsibility를 고정한다.

## 문서 역할

| 문서 | 역할 | 핵심 질문 |
|---|---|---|
| [01-project-overview.md](design/01-project-overview.md) | 제품 목적/범위 | 왜 만드는가? |
| [02-system-architecture.md](design/02-system-architecture.md) | 상위 컴포넌트 경계 | 어떤 모듈로 나뉘는가? |
| [03-agent-composition-and-harness.md](design/03-agent-composition-and-harness.md) | Agent/Harness 합성 | Role/Project/Domain/Skill을 어떻게 합치는가? |
| [04-orchestration-runtime.md](design/04-orchestration-runtime.md) | Orchestrator/Controller/실행 흐름 | 누가 판단하고 누가 상태를 통제하는가? |
| [05-project-workspace-and-context.md](design/05-project-workspace-and-context.md) | Project/worktree/context | 어떤 repository에서 어떤 context로 실행하는가? |
| [06-safety-verification-and-observability.md](design/06-safety-verification-and-observability.md) | Policy/verification/observability | 완료와 안전성을 어떻게 확인하는가? |
| [07-agent-registry-and-extension.md](design/07-agent-registry-and-extension.md) | Registry lifecycle | Agent/Skill/Profile을 어떻게 확장하는가? |
| [08-role-contracts-and-agent-environment.md](design/08-role-contracts-and-agent-environment.md) | Role 책임/환경 | 각 Role의 authority와 capability requirement는 무엇인가? |
| [09-architecture-governance-and-fitness.md](design/09-architecture-governance-and-fitness.md) | Architecture Fitness | 구조 규칙을 어떻게 deterministic check로 내리는가? |
| [10-skill-intake-portability-and-evaluation.md](design/10-skill-intake-portability-and-evaluation.md) | Skill 공급망/portability | 외부 Skill을 어떻게 audit/pin/변환하는가? |
| [11-task-contract-and-change-control.md](design/11-task-contract-and-change-control.md) | TaskSpec/change control | 목표/scope/AC가 어떻게 freeze/변경되는가? |
| [12-runtime-isolation-and-trust-boundaries.md](design/12-runtime-isolation-and-trust-boundaries.md) | Threat/isolation/capability | 무엇을 실제 강제하고 무엇은 탐지만 가능한가? |
| [13-state-recovery-and-artifact-integrity.md](design/13-state-recovery-and-artifact-integrity.md) | State/recovery/evidence | retry/crash/evidence의 canonical model은 무엇인가? |
| [14-evaluation-and-conformance.md](design/14-evaluation-and-conformance.md) | System eval/conformance | Agent Forge 자체의 불변조건을 어떻게 검증하는가? |
| [forgeroom-reuse-analysis.md](reference/forgeroom-reuse-analysis.md) | ForgeRoom 재사용 분석 | 무엇을 가져오고 버리는가? |
| [architecture-review.md](review/architecture-review.md) | 1차 적대적 리뷰 | 초기 현실성/경계 위험은 무엇이었나? |
| [architecture-agent-environment-review.md](review/architecture-agent-environment-review.md) | 역할/Agent 환경 리뷰 | Role/Fitness/Skill 공급망을 어떻게 보강했나? |
| [architecture-adversarial-review-round2.md](review/architecture-adversarial-review-round2.md) | 신뢰경계 재리뷰 | TaskSpec/state/isolation/evidence의 남은 위험은 무엇인가? |
| [mvp-roadmap.md](roadmap/mvp-roadmap.md) | 구현 순서 | 어떤 foundation부터 구현하는가? |

## Canonical Responsibility

```text
System boundary / Controller TCB
  -> 02
Profile composition / merge semantics
  -> 03
Orchestration flow
  -> 04
Repository/worktree/context
  -> 05
General safety/completion/observability
  -> 06
Registry lifecycle
  -> 07
Role semantics / requirement
  -> 08
Architecture Fitness
  -> 09
External Skill trust/portability
  -> 10
TaskSpec lifecycle/amendment/base pinning
  -> 11
Threat model/runtime isolation/enforcement
  -> 12
Task-Step-Attempt/recovery/artifact provenance
  -> 13
Verification floor/adversarial eval/conformance
  -> 14
```

세부 내용이 이전 문서의 예시와 충돌하면 해당 책임의 canonical 문서를 따른다.

## 권장 읽기 순서

### 설계 전체

```text
01 -> 02 -> 11 -> 12 -> 13
   -> 03 -> 04 -> 05 -> 06
   -> 08 -> 09 -> 07 -> 10 -> 14
   -> latest adversarial review
```

### 구현 시작

```text
11 -> 13 -> 12 -> 02 -> 04 -> 05 -> 03 -> 06 -> 14 -> Roadmap
```

## Source-of-truth priority

```text
Accepted ADR
  -> current decision record
Current Design Specification
  -> canonical current specification
Implementation
  -> implementation of spec
Review / Reference
  -> findings/evidence/reference
```

Accepted ADR이 기존 결정을 supersede하면 design specification도 같은 변경에서 갱신한다.

ADR에는 가능하면 `status`, `supersedes`, `superseded_by`를 둔다.

## 문서 변경 규칙

- authority/state/trust boundary/public contract 변경은 design 문서와 같이 반영한다.
- 같은 사실을 여러 문서에서 재정의하지 않는다.
- security claim은 threat model과 `ENFORCED / DETECTABLE / ADVISORY / UNSUPPORTED`를 구분한다.
- `Controller-owned`와 `physically tamper-proof`를 동의어로 쓰지 않는다.
- CheckRunner도 project code를 실행할 수 있으므로 execution plane으로 본다.
- 외부 Skill은 승인 전 untrusted다.
- 반복 finding은 deterministic check/eval로 승격 가능한지 검토한다.

## 핵심 용어

- **TaskSpec**: frozen objective/scope/AC/base revision specification.
- **TaskAmendment**: frozen TaskSpec 변경 기록.
- **VerificationPlan**: Controller가 계산한 최소 검증 계획.
- **Step**: 논리 작업 단위.
- **RunAttempt**: Step의 runtime 실행 1회.
- **Controller Artifact Store**: Controller가 canonical evidence namespace로 관리하는 저장 영역. 실제 tamper resistance는 isolation level에 따름.
- **Capability Enforcement Level**: ENFORCED / DETECTABLE / ADVISORY / UNSUPPORTED ([12](design/12-runtime-isolation-and-trust-boundaries.md) §4가 canonical).
- **Workspace Lease**: writer concurrency 제어.
- **Runner Trust Class**: controller_builtin / external_tool / project_command.
- **Instruction Trust Class**: CONTROL / TRUSTED_PROJECT_INSTRUCTION / REFERENCE_CONTENT ([12](design/12-runtime-isolation-and-trust-boundaries.md) §6이 canonical).
- **Runtime Backend**: coding agent backend. 초기 OpenCode.

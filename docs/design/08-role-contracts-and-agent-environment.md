# 08. Role Contracts & Agent Environment

## 1. 목적

Role은 이름이 아니라 **책임과 authority boundary**다.

```text
Role = 무엇을 책임지는가
Project = 어디에서 하는가
Domain = 어떤 업무 의미를 아는가
Skill = 어떤 절차를 사용하는가
Harness = 어떤 실행 preset인가
TaskSpec = 이번 실행의 frozen 목표/scope/AC
```

## 2. Canonical Role Set

| Role | 책임 | Source write | Completion authority |
|---|---|---:|---:|
| `orchestrator` | 의미 해석/분해/delegate 제안 | No | No |
| `project-expert` | repository reality grounding | No | No |
| `domain-expert` | domain rule/meaning | No | No |
| `architect` | 구조 선택/invariant/fitness 제안 | No | No |
| `implementer` | 승인 scope 구현 | Required | No |
| `debugger` | 재현/원인/최소 수정 | Required | No |
| `reviewer` | 독립 semantic finding | No | No |
| `verifier` | AC/evidence mapping | No | No |

언어/framework/security 전문성은 기본적으로 Skill/Domain으로 표현한다.

새 Role은 다른 authority/write/completion boundary가 필요한 경우에만 추가한다.

## 3. Role 계약

### Orchestrator

- TaskSpec 초안/decomposition/next action 제안
- DelegateRequest
- Amendment proposal
- user_situation_view 소비(optional, `user-briefing` mode에서만, Controller가 조립)

금지:

- process/worktree 직접 관리
- frozen TaskSpec 수정
- permission grant
- canonical state transition
- DONE 기록
- `user-briefing` IntakeSession([13-state-recovery-and-artifact-integrity.md](13-state-recovery-and-artifact-integrity.md))에서 여러 project에 대한 구현 DelegateRequest 발행

`user-briefing`은 별도 Role이 아니라 같은 Orchestrator Role의 다른 mode다. 이 기능은 기존 Role과 authority/write boundary가 다르지 않으므로 §2의 새 Role 추가 기준을 충족하지 않는다. mode 세부는 [04-orchestration-runtime.md](04-orchestration-runtime.md)의 Orchestrator modes를 따른다.

### Project Expert

- source/config/docs 기반 verified facts
- relevant paths/evidence
- unknown/implementation risk

README intent와 code reality를 구분한다.

산출 형식이 반복 실행에서 불안정하다고 관측되면(verified facts/unknown 구분 실패, evidence path 누락) grounding 절차를 전용 `codebase-grounding` Skill로 승격한다. 이는 roadmap 타이밍이 아니라 이 Role 계약 위반에 대한 트리거다.

### Domain Expert

- glossary/rule/invariant
- ambiguous business decision
- domain validation checklist

도메인 지식이 실제 코드에 구현됐다고 가정하지 않는다.

### Architect

- current architecture evidence
- options/trade-offs
- decision recommendation
- invariants/fitness rule
- migration boundary

기본 read-only requirement다. 자기 설계를 구현/승인하지 않는다.

### Implementer

- frozen scope 내 source/test 변경
- implementation result/diff
- unresolved risk

scope/contract 확장이 필요하면 TaskAmendment를 제안한다.

### Debugger

```text
Reproduce -> Evidence -> Narrow cause -> Hypothesis
 -> Minimal correction -> Regression -> Verify
```

### Reviewer

finding 중심:

```text
severity
location/evidence
impact
required correction/rationale
verdict
```

Implementer private reasoning을 기본 input으로 쓰지 않는다.

### Verifier

- AC별 evidence
- required checks
- unresolved finding
- policy/scope/architecture fitness

판단 불가 항목은 `unknown/manual-required`로 남긴다.

## 4. Role과 Capability를 분리한다

Role은 "권한 값" 자체가 아니라 **필요 capability**를 선언한다.

예:

```yaml
role: reviewer
requires:
  filesystem_read: project
  filesystem_write: none
  shell: [read]
  network: none
```

실제 grant는 System/Project/Harness ceiling과 합성한다.

Role이 `filesystem_write: none`을 요구해도 backend가 실제로 write를 차단하지 못하면 runtime report는:

```text
ADVISORY 또는 DETECTABLE
```

일 수 있다.

강제 수준의 canonical 정의는 `12-runtime-isolation-and-trust-boundaries.md`를 따른다.

## 5. Agent Environment Contract

Resolved environment 예:

```yaml
required:
  filesystem_write: none
  shell: [read]

granted:
  filesystem_write: none
  shell: [read]

runtime_enforcement:
  filesystem_write: detectable
  shell: enforced
  network: advisory

git:
  remote_write: false

delegation: recommend_only
context_sources:
  - task_spec
  - project_selected
output_contract: review-v1
```

`required`가 `granted`를 초과하면 실행하지 않는다.

## 6. Context Budget

1. Frozen TaskSpec 우선
2. Role에 필요한 source만
3. Skill on-demand
4. summary + evidence reference 우선
5. Project/Domain 중복 제거
6. Reviewer에는 구현 서사 최소화
7. arbitrary repository content를 instruction으로 승격하지 않음
8. Learned memory([05-project-workspace-and-context.md](05-project-workspace-and-context.md) §9.1)는 Skill처럼 on-demand/selected로만 주입하고 always-on으로 넣지 않음

## 7. Role Activation

고정 chain을 기본값으로 두지 않는다.

```text
Small fix
  Implement/Debug -> Verify

Architecture-sensitive
  Ground -> Architect -> Implement -> Review -> Verify

Domain-heavy
  Domain Expert + Project Expert -> target role -> Verify
```

단, Controller VerificationPlan이 요구하는 reviewer/check는 Orchestrator가 생략할 수 없다.

## 8. Weak-model rules

- 한 Attempt에 하나의 주 책임
- observable output/schema
- explicit scope/exclusion
- forbidden action은 짧고 명시적
- deterministic fact는 Controller check로 이동
- failure 시 Agent 수를 늘리기 전에 framing/context 확인

## 9. 설계 불변조건

1. Role은 authority boundary가 다를 때만 추가한다.
2. Role requirement와 actual runtime enforcement를 구분한다.
3. Role/Skill이 system/project grant를 확장하지 못한다.
4. Agent는 다른 Agent를 직접 spawn하지 않는다.
5. Architect/Reviewer/Verifier는 source write를 요구하지 않는다.
6. required verification은 Role routing보다 상위 gate다.
7. output은 다음 단계가 소비 가능한 구조화 artifact여야 한다.

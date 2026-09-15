# 08. Role Contracts & Agent Environment

## 1. 목적

Agent Forge의 Role은 이름이 아니라 **실행 책임과 권한 경계를 정의하는 계약**이다.

Agent 수를 늘려 품질을 확보하지 않는다. 기본 Role 집합은 작게 유지하고, Project / Domain / Skill을 조합해 전문성을 만든다.

```text
Role = 무엇을 책임지는가
Project = 어디에서 작업하는가
Domain = 어떤 업무 의미를 알아야 하는가
Skill = 어떤 절차/능력을 사용하는가
Harness = 어떤 실행 환경과 제약을 적용하는가
Task Contract = 이번 실행에서 무엇을 끝내야 하는가
```

## 2. Canonical Role Set

MVP의 기본 Role은 다음으로 제한한다.

| Role | 핵심 책임 | 기본 write | 다른 Agent 직접 spawn | 완료 authority |
|---|---|---:|---:|---:|
| `orchestrator` | 목표 해석, 분해, delegate 제안 | No | No | No |
| `project-expert` | 저장소 구조/구현 사실 grounding | No | No | No |
| `domain-expert` | 업무 규칙/용어/도메인 판단 | No | No | No |
| `architect` | 구조적 선택, 경계, ADR, fitness rule 제안 | No | No | No |
| `implementer` | 승인 scope 내 구현 | Yes, worktree only | No | No |
| `debugger` | 재현 → 원인 규명 → 제한된 수정 | Yes, worktree only | No | No |
| `reviewer` | 의미 기반 독립 검토 | No | No | No |
| `verifier` | AC와 evidence 연결, 완료 증거 확인 | No | No | No |

`planner`, `researcher`, `security-reviewer`, `dotnet-expert` 같은 이름은 기본 Role로 늘리지 않는다.

- 일반 planning은 Orchestrator 책임이다.
- 외부 조사/리서치는 Skill 또는 explicit task mode로 표현한다.
- security, .NET, Python 같은 전문성은 Skill/Domain으로 표현한다.
- 새로운 Role은 **기존 Role과 다른 authority 또는 write boundary가 필요한 경우에만** 추가한다.

## 3. Role별 계약

### 3.1 Orchestrator

입력:

- User Goal
- Task Contract 또는 intake 결과
- Agent capability catalog
- RunResult summary
- selected evidence references

출력:

- `DelegateRequest`
- task decomposition 제안
- next action / stop recommendation

금지:

- subprocess 직접 관리
- worktree 생성/삭제
- policy override
- 임의 retry budget 증가
- worker filesystem 직접 수정
- 자연어만으로 DONE 선언

### 3.2 Project Expert

목적은 **repository reality를 grounding**하는 것이다.

입력:

- Project Profile
- repository source/config/docs
- 필요한 git history / build metadata

출력:

- verified facts
- relevant paths
- architecture/structure summary
- unknowns
- implementation risk

규칙:

- README의 의도와 실제 구현을 구분한다.
- 검증되지 않은 내용을 사실처럼 채우지 않는다.
- 큰 문서를 전부 복사하지 않고 evidence path를 반환한다.

### 3.3 Domain Expert

목적은 repository와 독립적인 업무 의미를 제공하는 것이다.

출력 예:

- domain rule
- glossary interpretation
- invariants
- ambiguous business decision
- domain-specific validation checklist

도메인 지식이 코드에서 실제로 구현됐다고 가정하지 않는다. Project Expert 또는 source evidence와 교차 검증한다.

### 3.4 Architect

Architect는 구현 Agent의 상위 권한자가 아니다. **구조적 선택과 경계의 전문 분석 Role**이다.

호출 조건:

- module/layer boundary 변경
- public contract 변경
- dependency direction 변경
- persistence/integration 구조 변경
- cross-cutting concern 도입
- 대규모 migration/refactor
- 신규 Architecture Fitness Rule 필요

출력:

```text
Context
Constraints
Observed Architecture
Options
Decision / Recommendation
Trade-offs
Invariants
Fitness Rules
Migration Boundary
Open Questions
```

기본은 read-only다. 직접 구현하거나 자기 설계를 스스로 승인하지 않는다.

단순 bug fix, 작은 CRUD, 국소 refactor에 항상 Architect를 넣지 않는다.

### 3.5 Implementer

입력:

- Task Contract
- relevant project/domain context
- approved architecture decision if required
- selected skills

출력:

- source diff
- tests
- implementation summary
- verification commands/results reference
- unresolved risk

규칙:

- 요청 범위 밖 정리/리팩터링을 하지 않는다.
- 기존 contract 변경이 필요하면 임의로 확장하지 않고 BLOCKED/decision request로 반환한다.
- test를 통과시키기 위해 verification을 약화하지 않는다.

### 3.6 Debugger

기본 순서:

```text
Reproduce
 -> Gather evidence
 -> Narrow cause
 -> State root-cause hypothesis
 -> Minimal correction
 -> Regression test
 -> Verify
```

증상만 가리는 변경이나 근거 없는 retry를 금지한다.

### 3.7 Reviewer

Reviewer의 목적은 구현자와 다른 권한을 갖는 것이 아니라 **다른 context와 책임으로 독립 검토**하는 것이다.

입력:

- Task Contract
- final diff
- relevant source
- test/check output
- architecture/project/domain rules

기본 미제공:

- Implementer private reasoning
- 구현자의 자기평가
- 불필요한 전체 transcript

출력은 finding 중심으로 한다.

```text
severity
location/evidence
why it matters
required correction or rationale
verdict
```

### 3.8 Verifier

Verifier는 새로운 설계를 만들거나 구현을 고치는 Role이 아니다.

책임:

- acceptance criterion별 evidence 존재 여부
- required check 결과
- unresolved blocking finding
- scope/path policy
- architecture fitness result

Verifier가 판단할 수 없는 항목은 `unknown` 또는 `manual evidence required`로 남긴다.

## 4. Agent Environment Contract

Role 이름만으로 실행환경을 암시하지 않는다. Controller가 다음 capability를 명시적으로 resolve한다.

```yaml
capabilities:
  filesystem: read_only | worktree_write
  shell: none | allowlisted
  network: disabled | allowlisted
  git:
    read: true
    local_commit: false
    remote_write: false
  delegation: recommend_only
  context_sources:
    - task_contract
    - project
    - domain
  checks:
    - project_required
  output_contract: <id>
```

이 값은 두 종류로 나뉜다.

- **Advisory**: prompt / AGENTS / runtime instruction
- **Enforced**: Controller / WorkspaceManager / PolicyGate / post-run check

문서와 코드에서 둘을 섞어 표현하지 않는다.

## 5. Context Budget 원칙

약한 모델일수록 많은 context가 품질을 보장하지 않는다.

기본 규칙:

1. Task Contract를 항상 먼저 제공한다.
2. Role에 필요한 context source만 선택한다.
3. Skill은 on-demand로 주입한다.
4. 원문 전체보다 summary + evidence reference를 우선한다.
5. Project와 Domain의 동일 설명을 중복 주입하지 않는다.
6. Reviewer에는 구현자의 불필요한 서사를 제거한다.
7. Architect에는 현재 구조와 제약을 우선 제공하고 구현 세부 전체를 무조건 넣지 않는다.

## 6. Role Activation

고정 workflow chain을 기본값으로 두지 않는다.

```text
Small fix
  Orchestrator -> Debugger/Implementer -> Reviewer(optional) -> Verifier

Architecture-sensitive change
  Orchestrator -> Project Expert -> Architect -> Implementer -> Reviewer -> Verifier

Domain-heavy analysis
  Orchestrator -> Domain Expert + Project Expert -> target Role -> Verifier
```

`Architecture Designer -> Fullstack Guardian -> Code Reviewer ...` 같은 긴 체인은 참고 패턴일 뿐 기본 프로토콜이 아니다.

Controller는 Role 호출 수보다 dependency, budget, permission을 통제한다.

## 7. Role Profile 최소 Schema

```yaml
id: architect
purpose: architecture decision and boundary analysis
write_mode: read_only
compatible_harnesses: [analysis, review]
default_skills:
  - architecture-analysis
required_output: architecture-decision
can_recommend_delegation: true
can_spawn: false
completion_authority: false
```

Role Profile은 특정 repository, language, business domain 내용을 포함하지 않는다.

## 8. 약한 모델 대응 규칙

Agent Forge는 모델이 충분히 영리하다고 가정하지 않는다.

- 한 Run에 하나의 주 책임만 부여한다.
- "적절히 알아서" 대신 observable output을 요구한다.
- scope/exclusion/stop condition을 Task Contract에 명시한다.
- role별 forbidden action을 짧고 명시적으로 둔다.
- output은 schema 또는 고정 section으로 제한한다.
- 검증 가능한 사실은 deterministic check로 이동한다.
- 실패 시 더 큰 Agent를 추가하기 전에 context/task framing을 먼저 고친다.

## 9. 설계 불변조건

1. Role은 authority/write boundary가 다를 때만 새로 만든다.
2. 언어/framework 전문성을 Role 증식으로 해결하지 않는다.
3. Agent는 다른 Agent를 직접 spawn하지 않는다.
4. Architect는 구현/완료 authority를 소유하지 않는다.
5. Reviewer/Verifier는 source write를 하지 않는다.
6. Role별 context는 최소 필요량만 주입한다.
7. Role 결과는 다음 단계가 소비 가능한 구조화된 artifact여야 한다.
8. Agent 수가 많다는 사실을 품질 지표로 사용하지 않는다.

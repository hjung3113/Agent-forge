# 03. Agent Composition & Harness

## 1. 핵심 모델

Agent는 고정 prompt가 아니라 실행 시 합성되는 runtime profile이다.

```text
Resolved Agent Runtime
  = Base
  + Role
  + Project
  + Domain(optional)
  + selected Skills
  + Frozen TaskSpec
```

목적은 project/language/domain 조합마다 Agent 정의를 복제하지 않는 것이다.

## 2. 각 Profile 책임

### Base

- 공통 실행 규칙
- artifact/output 기본 계약
- timeout/retry 기본값
- 공통 금지 행위

### Role

무슨 책임을 갖는지 정의한다.

예:

```text
project-expert
domain-expert
architect
implementer
debugger
reviewer
verifier
```

Role은 authority/write boundary를 정의하지만 실제 permission grant의 유일한 source는 아니다.

### Project

어디에서 작업하는지를 정의한다.

- repository/default branch
- registered build/test/check
- project instruction/context
- denied/policy-sensitive paths
- architecture checks
- capability ceiling

### Domain

repository와 독립적인 업무 의미를 제공한다.

### Skill

재사용 가능한 절차/능력이다.

Skill은 필요한 capability를 선언할 수 있지만 **권한을 부여하지 않는다.**

### Frozen TaskSpec

이번 실행의 objective/scope/AC/base revision을 제공한다.

TaskSpec lifecycle은 `11-task-contract-and-change-control.md`가 canonical이다.

## 3. Harness

Harness는 실행환경 preset이다.

```text
harnesses/
  implementation/
    harness.yaml
    prompt-contract.md
    output-contract.md
    AGENTS.md
```

예:

```yaml
id: implementation
capability_ceiling:
  filesystem_write: worktree
  shell: [read, test]
  network: none
output_contract: implementation-v1
```

Harness의 선언과 실제 runtime enforcement level은 다를 수 있으므로 `12-runtime-isolation-and-trust-boundaries.md`의 capability report를 함께 본다.

## 4. Merge를 한 종류의 precedence로 해결하지 않는다

필드 성격에 따라 merge semantics를 분리한다.

### 4.1 Hard policy

System hard policy는 override 불가다.

### 4.2 Grant ceiling

Permission/capability grant는 precedence가 아니라 **intersection**이다.

```text
effective_grant
 = System ceiling
 ∩ Project ceiling
 ∩ Harness ceiling
 ∩ Role ceiling
```

Task/Skill은 grant를 확장하지 않는다.

### 4.3 Requirement

Task/Role/Skill은 required capability를 합성한다.

```text
required = union(required capabilities)
```

그리고:

```text
required <= effective_grant
```

가 아니면 BLOCKED다.

### 4.4 Restriction

Scope/exclusion/denied path는 더 제한적인 조건이 우선한다.

Task가 Project deny를 해제할 수 없다.

### 4.5 Required checks/context

필수 check는 union을 기본으로 한다.

Context는 무조건 union하지 않고 budget/trust/relevance로 선택한다.

### 4.6 Conflict

명시적 모순은 자동 추정하지 않고 fail-fast 한다.

예:

```text
Task requires DB migration
Project hard policy denies migration
=> BLOCKED
```

## 5. Instruction precedence

Runtime instruction 의미상 우선순위는 대략 다음이다.

```text
System hard policy
Frozen TaskSpec
Project hard restrictions / approved project instruction
Role contract
Domain / selected Skill procedure
Base defaults
```

단, 이 precedence가 runtime provider-native instruction precedence와 완전히 같다고 가정하지 않는다.

OpenCode가 project-local `AGENTS.md`를 자동 로드한다면 실제 동작은 Phase 0 conformance에서 확인한다.

## 6. Capability 예

```yaml
role: implementer
requires:
  filesystem_write: true
  shell: [test]

project:
  grants:
    filesystem_write: worktree
    shell: [read, test]

system:
  grants:
    filesystem_write: worktree
    shell: [read, test]
    network: none
```

Resolved capability에는 값뿐 아니라:

```text
ENFORCED | DETECTABLE | ADVISORY
```

수준을 저장한다.

## 7. Skill 주입

모든 Skill을 항상 넣지 않는다.

1. Role default는 최소화
2. Project required만 자동 추가
3. Domain은 관련 있을 때만
4. explicit Skill은 Registry validation 후
5. 상충/중복 Skill은 fail 또는 smaller set

약한 모델에서는 context 양보다 trigger precision을 우선한다.

## 8. Project-local instruction

Project source의 모든 문서를 instruction으로 취급하지 않는다.

Context trust class:

```text
CONTROL
TRUSTED_PROJECT_INSTRUCTION
REFERENCE_CONTENT
```

arbitrary source/README/comment는 REFERENCE_CONTENT다.

## 9. Output Contract

exit code 0과 task success는 다르다.

Output Contract는 형식만 검증한다.

실제 완료는:

- deterministic checks
- findings
- AC evidence
- policy

를 Controller가 조합한다.

## 10. Resolved snapshot / provenance

RunAttempt 전에 최소:

```text
resolved-agent
resolved-harness
capability requirement/grant/report
selected skills + hashes
project profile hash
TaskSpec hash
```

를 input manifest에 기록한다.

LLM result 자체는 deterministic replay를 보장하지 않는다.

## 11. 설계 불변조건

1. Agent 정의에 project/domain 내용을 복사하지 않는다.
2. Skill은 permission grant authority가 아니다.
3. TaskSpec이 Project/System restriction을 완화하지 못한다.
4. grant는 intersection, requirement는 union으로 분리한다.
5. unsupported capability를 adapter가 조용히 버리지 않는다.
6. context와 instruction trust를 구분한다.
7. resolved composition/provenance를 실행 전에 snapshot한다.

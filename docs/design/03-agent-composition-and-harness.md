# 03. Agent Composition & Harness

## 1. 핵심 모델

Agent Forge에서 Agent는 고정 prompt가 아니라 **실행 시점에 여러 profile을 합성한 runtime profile**이다.

```text
Resolved Agent Runtime
  = Base Runtime
  + Role Profile
  + Project Profile
  + Domain Profile
  + Skill Set
  + Task Contract
```

이 구조의 목적은 `logwarehouse-implementer`, `logwarehouse-reviewer`, `standard-log-expert` 같은 Agent를 모두 별도 복사본으로 만들지 않고 필요한 요소를 조합해 생성하는 것이다.

## 2. Profile 책임

### 2.1 Base Runtime

모든 Agent에 공통 적용된다.

- OpenCode 실행 규칙
- 공통 작업 규칙
- 결과/output contract 기본값
- timeout/retry 기본값
- artifact/log protocol
- 금지되는 기본 행위

### 2.2 Role Profile

Agent가 **무슨 역할을 수행하는지** 정의한다.

예:

```text
planner
implementer
debugger
reviewer
verifier
researcher
```

Role Profile에는 다음이 들어갈 수 있다.

- role instruction
- default skills
- default tool capability
- default filesystem mode
- output contract
- evidence requirement

예:

```yaml
id: reviewer
skills:
  - code-review
  - regression-analysis
permissions:
  filesystem: read_only
output:
  required_sections:
    - Findings
    - Verdict
```

### 2.3 Project Profile

특정 프로그램/레포지토리에 관한 실행환경을 정의한다.

- repository URL 또는 local canonical clone
- default branch
- build/test/lint 명령
- project-specific AGENTS/context
- architecture docs
- allowed/denied paths
- project-specific skills
- project expert profile

Project Profile은 provider-specific agent identity를 소유하지 않는다. 프로젝트와 런타임 provider를 결합하지 않기 위함이다.

### 2.4 Domain Profile

코드 저장소와 독립적인 업무/도메인 지식을 제공한다.

예:

```text
standard-log
semiconductor-equipment
etl-pipeline
postgresql-migration
```

Domain Profile은 다음을 가질 수 있다.

- canonical domain docs
- glossary
- domain rules
- domain-specific validation checklist
- recommended skills

### 2.5 Skill

Skill은 재사용 가능한 **능력/절차 단위**다.

예:

```text
dotnet
python
fastapi
root-cause-debugging
minimal-change
architecture-review
standard-log
carryover-analysis
start-end-matching
```

Skill은 Agent 역할과 분리한다.

```text
Role = 무엇을 하는가
Skill = 어떻게 잘 하는가
Project = 어디에서 하는가
Domain = 어떤 업무 의미를 알아야 하는가
```

### 2.6 Task Contract

이번 실행에만 적용되는 계약이다.

최소 항목:

```yaml
objective: "Carryover validation bug 수정"
scope:
  - src/Parser/**
exclusions:
  - DB schema change
acceptance_criteria:
  - failing regression test added
  - existing tests pass
verification:
  - dotnet test
stop_condition:
  - acceptance criteria satisfied
```

Task Contract가 가장 구체적이므로 일반적인 role/skill 지침보다 task scope에 대한 우선순위가 높다. 단, security policy를 override할 수는 없다.

## 3. Harness Profile

Harness는 Agent가 역할을 수행하기 위해 필요한 환경·규칙·계약을 하나의 preset으로 묶는다.

권장 구조:

```text
harnesses/
  implementation/
    harness.yaml
    prompt-contract.md
    output-contract.md
    AGENTS.md
    skills/
    hooks/
```

예시:

```yaml
id: implementation
applies_to:
  roles: [implementer, debugger]
permissions:
  filesystem: worktree_only
  shell: gated
  network: disabled
tools:
  allow:
    - read
    - grep
    - edit
    - test
output:
  min_bytes: 50
  required_sections:
    - Summary
    - Verification
```

중요: `permissions`는 Runtime backend가 그대로 강제한다는 의미가 아니다. Harness Compiler가 다음 두 종류로 나눈다.

```text
Advisory
  -> prompt / AGENTS / OpenCode instruction

Enforced
  -> Controller PolicyGate / WorkspaceManager / post-run diff check
```

## 4. Merge Precedence

최종 profile 합성 우선순위는 다음을 권장한다.

```text
Hard System Policy              highest, override 불가
Task Contract scope/exclusions
Project restrictions
Role Profile
Domain Profile
Explicit Task Skills
Role default Skills
Base defaults                  lowest
```

단순히 YAML을 마지막 값으로 덮어쓰면 안 되는 필드가 있다.

### 4.1 Restrictive merge

permission은 더 제한적인 값이 우선한다.

```text
Role: filesystem = worktree_only
Project: filesystem = read_only
Result: read_only
```

### 4.2 Union merge

읽을 context source나 required checks는 합집합이 적합하다.

### 4.3 Conflict fail-fast

명시적으로 모순되는 경우 자동 추정하지 않는다.

예:

```text
Task requires DB migration
Project policy denies migration
=> configuration/policy conflict -> BLOCKED
```

## 5. Agent Profile 예시

### 5.1 LogWarehouse Implementer

```yaml
id: logwarehouse-implementer
role: implementer
project: logwarehouse
domains:
  - standard-log
skills:
  - dotnet
  - minimal-change
  - unit-testing
```

해석 결과:

```text
implementer role rules
+ logwarehouse repository/context/build commands
+ standard-log domain rules
+ dotnet/minimal-change/unit-testing skills
+ current Task Contract
```

### 5.2 Standard Log Expert

```yaml
id: log-expert
role: domain-expert
domains:
  - standard-log
skills:
  - carryover-analysis
  - start-end-matching
permissions:
  filesystem: read_only
```

이 Agent는 구현을 하지 않고 분석 evidence를 반환하도록 설계한다.

## 6. Skill 주입 원칙

모든 skill을 항상 넣지 않는다. context가 커질수록 약한 모델에서는 오히려 품질이 떨어질 수 있다.

권장 규칙:

1. Role default skill은 최소화한다.
2. Project가 반드시 요구하는 skill만 자동 추가한다.
3. Domain은 task와 관련 있을 때만 추가한다.
4. Orchestrator가 선택한 explicit skill은 Registry validation 후 추가한다.
5. 상충하는 skill 조합은 Harness Compiler가 차단한다.

예:

```text
implementer
 + dotnet
 + standard-log
 + bug-fix
```

이면 충분한데 여기에 frontend, security-review, database-design까지 무조건 넣지 않는다.

## 7. Project-local 지침과 Agent Forge Harness

프로젝트 repo 자체에 `AGENTS.md` 같은 지침이 있을 수 있다. 이를 무조건 복사하거나 덮어쓰지 않는다.

권장 precedence:

```text
System hard policy
Task Contract
Agent Forge compiled harness
Project-local AGENTS.md
General base instruction
```

다만 OpenCode가 project-local instruction을 어떤 순서로 읽는지는 runtime adapter가 확인해야 한다. Agent Forge가 통제할 수 없는 runtime-native precedence를 "강제 가능"하다고 문서화하지 않는다.

## 8. Output Contract

Agent 호출 성공(exit code 0)과 작업 성공은 다르다.

Harness는 최소 output contract를 가질 수 있다.

```yaml
output:
  min_bytes: 50
  first_line_regex: "^Review Result: (pass|fail)$"
  required_sections:
    - Findings
```

Output Contract Validator는 형식 충족 여부만 판단한다. 실제 코드 품질과 acceptance criteria는 Reviewer/Verifier/CheckRunner가 판단한다.

## 9. 설계 불변조건

1. Agent 정의에 repository-specific 내용을 중복 복사하지 않는다.
2. Harness와 runtime provider identity를 결합하지 않는다.
3. Skill은 role이 아니다.
4. Domain knowledge와 project source context를 구분한다.
5. permission metadata와 실제 enforcement를 구분한다.
6. 합성 결과는 실행 전에 저장해 재현 가능해야 한다.
7. 어떤 profile이 최종 규칙을 만들었는지 provenance를 남긴다.

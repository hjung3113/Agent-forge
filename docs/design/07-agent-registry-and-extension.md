# 07. Agent Registry & Extension

## 1. 목적

Agent Forge는 Agent를 코드에 하드코딩하지 않고 **Registry 기반으로 추가·수정·비활성화**할 수 있어야 한다.

Agent 추가 기능의 핵심은 새 프로그램을 만드는 것이 아니라 기존 profile을 조합한 선언을 등록하는 것이다.

## 2. 권장 디렉터리

```text
agents/
├─ log-expert/
│  └─ agent.yaml
├─ logwarehouse-expert/
│  └─ agent.yaml
└─ default-reviewer/
   └─ agent.yaml

roles/
├─ implementer/
├─ reviewer/
├─ verifier/
└─ debugger/

projects/
├─ logwarehouse/
│  ├─ project.yaml
│  └─ context.yaml
└─ standard-log-lifecycle/

domains/
└─ standard-log/

skills/
├─ dotnet/
├─ unit-testing/
├─ root-cause-debugging/
└─ standard-log/

harnesses/
├─ implementation/
├─ review/
└─ analysis/
```

## 3. Agent Definition

최대한 얇게 유지한다.

```yaml
id: logwarehouse-implementer
description: LogWarehouse implementation agent
role: implementer
project: logwarehouse
domains:
  - standard-log
skills:
  - dotnet
  - unit-testing
harness: implementation
enabled: true
```

Role/Project/Skill 내용을 agent.yaml에 다시 복사하지 않는다.

## 4. Agent Registry 책임

```text
load definitions
validate references
resolve inheritance/composition
check enabled state
produce ResolvedAgentProfile
```

개념 interface:

```text
list()
get(agent_id)
validate(agent_id)
resolve(agent_id, task_spec_hash)
```

`resolve()` 결과는 실행 전에 artifact로 저장한다.

## 5. 동적 Agent 선택

Orchestrator에게 전체 profile 내용을 넣을 필요는 없다. Registry가 **capability summary**를 제공한다.

예:

```json
{
  "id": "log-expert",
  "description": "Standard Log domain analysis",
  "capabilities": ["carryover", "context", "start-end-matching"],
  "project": null,
  "requires": {
    "filesystem_write": "none",
    "shell": ["read"],
    "network": "none"
  }
}
```

Orchestrator는 이 catalog를 보고 Agent를 선택하고, 실제 상세 profile은 Controller가 resolve한다.

## 6. Agent 추가 CLI

MVP 이후 또는 간단한 scaffolding으로 다음을 지원할 수 있다.

```text
agent-forge agent list
agent-forge agent show <id>
agent-forge agent add <id>
agent-forge agent validate <id>
agent-forge agent enable <id>
agent-forge agent disable <id>
agent-forge agent remove <id>
```

### `agent add` 흐름

```text
name
 -> role
 -> project(optional)
 -> domains(optional)
 -> explicit skills(optional)
 -> harness
 -> permission override(optional, only more restrictive)
 -> validate references
 -> create agents/<id>/agent.yaml
```

permission override는 [03-agent-composition-and-harness.md](03-agent-composition-and-harness.md) §4.2 grant ceiling을 더 제한하는 방향으로만 적용할 수 있다. Role/Harness ceiling을 넘어서는 확장은 override로 만들 수 없다.

Agent CLI가 skill이나 project 내용을 자동 복제하지 않는다.

## 7. Skill Registry

Skill에는 최소 메타데이터가 필요하다.

```yaml
id: root-cause-debugging
description: Evidence-first debugging procedure
compatible_roles:
  - debugger
  - implementer
conflicts_with: []
requires_tools:
  - read
  - grep
```

Skill의 실제 지침은 `SKILL.md` 같은 별도 파일에 둔다.

```text
skills/root-cause-debugging/
├─ skill.yaml
└─ SKILL.md
```

## 8. Skill 자동 선택

자동 선택은 처음부터 복잡한 semantic router로 만들 필요가 없다.

초기:

```text
Agent defaults
+ Project defaults
+ Domain defaults
+ Orchestrator explicit request
```

추후 task tags/capability matching을 추가할 수 있다.

예:

```text
Task: C# Parser null bug
Project: logwarehouse

Resolved skills:
  dotnet
  root-cause-debugging
  standard-log
```

## 9. Project Expert 자동 생성

Project가 등록되면 기본 Project Expert를 논리적으로 생성할 수 있다.

```text
<project-id>-expert
```

실제 별도 파일을 복제할 필요 없이 virtual profile로 만들 수 있다.

```text
role = project-expert
project = <project-id>
requires.filesystem_write = none
skills = project.default_analysis_skills
```

필요한 경우 사용자가 이를 명시적 agent.yaml로 override/확장한다.

## 10. Domain Expert 자동 생성

Domain도 동일하다.

```text
standard-log-expert
```

```text
role = domain-expert
domain = standard-log
requires.filesystem_write = none
```

이 방식이면 Agent 수를 늘려도 설정 중복이 작다.

## 11. Registry Validation

부팅 시 fail-fast 할 항목:

- 존재하지 않는 role 참조
- 존재하지 않는 project/domain/skill/harness 참조
- cyclic inheritance
- write-required role + read-only hard project policy 충돌
- malformed output contract
- duplicate id
- invalid repository config

실행 시 확인할 항목:

- project repository availability
- OpenCode health
- requested task scope
- current runtime capability

## 12. Versioning

Agent/Skill/Harness 정의가 변경되면 과거 RunAttempt를 재현하기 위해 실행 시 snapshot을 남긴다. snapshot 내용의 canonical 정의는 [13-state-recovery-and-artifact-integrity.md](13-state-recovery-and-artifact-integrity.md) §15 Input Manifest(`task_spec_hash`, `base_commit`, `resolved_harness_hash`, `skill_hashes`, `project_profile_hash`, capability report 등)를 따른다.

```text
attempt A-123
  input-manifest.json
  resolved-agent.yaml
  resolved-harness.yaml
```

Registry의 현재 파일만 보고 과거 실행을 해석하지 않는다.

## 13. 확장 포인트

초기부터 interface 경계만 두고 구현은 나중에 추가할 수 있다.

- Runtime backend plugin
- Context provider
- Project template
- Skill pack
- Harness pack
- Artifact exporter
- Observer UI

외부 marketplace 자동 설치는 사내 환경 특성상 기본 목표가 아니다. 관리자가 검증한 local pack을 등록하는 방식이 우선이다.

## 14. 설계 불변조건

1. Agent 추가가 core source 수정으로 이어지지 않아야 한다.
2. Agent definition은 composition reference 중심으로 얇게 유지한다.
3. Registry validation 실패는 실행 전에 발견한다.
4. 동적 Agent 선택과 실제 권한 enforcement를 분리한다.
5. 과거 RunAttempt는 resolved snapshot으로 재현 가능해야 한다.
6. 사용자 추가 profile은 system hard policy를 완화할 수 없다.

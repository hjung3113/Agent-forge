# 09. Architecture Governance & Fitness

## 1. 목적

Agent Forge에서 아키텍처 원칙은 문서로 끝나지 않는다.

가능한 규칙은 **실행 가능한 Architecture Fitness Check**로 내려서 Agent가 실수해도 CI/Controller가 구조 위반을 탐지하도록 한다.

```text
Architecture Decision
  -> Invariant
  -> Machine-checkable rule when feasible
  -> Controller CheckRunner
  -> Evidence
  -> Completion Gate
```

이 계층의 목적은 새로운 architecture framework를 만드는 것이 아니라 기존 Project Profile과 Verification subsystem에 **구조 검증 능력**을 추가하는 것이다.

## 2. Architecture Governance 책임

### Architect Role

- 현재 구조를 evidence로 분석
- 구조 변경 필요성 판단
- 선택지와 trade-off 정리
- architecture invariant 제안
- fitness rule 후보 제안

### Project Profile

- 해당 프로젝트가 채택한 architecture rule과 check command를 선언
- legacy baseline / exception을 참조

### CheckRunner

- 실제 rule 실행
- exit code / result parsing
- artifact 저장

### Reviewer

- machine check로 표현하기 어려운 구조적 문제 검토

### Verifier

- required architecture checks가 실행됐는지 확인
- blocking failure가 있으면 DONE 차단

## 3. Rule 분류

아키텍처 규칙을 세 종류로 나눈다.

### 3.1 Hard Fitness Rule

기계적으로 판정 가능하고 위반 시 기본적으로 완료를 막는다.

예:

- Domain -> Infrastructure dependency 금지
- 특정 namespace/package import 금지
- module dependency cycle 금지
- UI layer가 DB adapter를 직접 참조하지 않음
- read-only module에서 write adapter 참조 금지

### 3.2 Advisory Architecture Rule

자동 판정이 어렵거나 false positive가 큰 항목.

예:

- 책임이 과도하게 섞였는가
- public API가 불필요하게 넓어졌는가
- abstraction이 실제 변경축과 맞는가
- SOLID 위반이 실질적인 결합도 증가로 이어지는가

Reviewer가 evidence를 제시한다.

### 3.3 Migration Rule

현재 legacy violation이 존재하지만 신규 위반은 막아야 하는 규칙.

```text
existing debt = baseline
new violation = fail
removed violation = improvement
```

기존 대형 레거시 프로젝트에 Architecture Fitness를 도입할 때 전체 구조를 즉시 고치도록 강제하지 않는다.

## 4. Project Profile 예시

Architecture Policy를 별도 profile layer로 만들지 않는다. Project Profile의 검증 계약으로 둔다.

```yaml
architecture:
  mode: baseline
  checks:
    - id: dependency-direction
      runner: dotnet-test
      command: dotnet test tests/ArchitectureTests -c Debug
      blocking: true

    - id: python-import-boundaries
      runner: command
      command: lint-imports
      blocking: true

  baseline: .agent-forge/architecture-baseline.json
```

새 source of truth를 만들지 않고 project-owned rule과 test를 호출하는 것이 기본이다.

## 5. C# 권장 패턴 — ArchUnitNET

C#에서는 `ArchUnitNET` 같은 architecture test library를 활용할 수 있다.

적합한 규칙:

- namespace/assembly dependency direction
- forbidden dependency
- type/member access rule
- naming/placement rule
- layer boundary

예시 의도:

```text
Domain must not depend on Infrastructure
Application may depend on Domain
Infrastructure may implement Application ports
Presentation must not access Infrastructure internals directly
```

Agent Forge는 ArchUnitNET 자체를 core dependency로 강제하지 않는다.

프로젝트가 채택하면 Project Profile의 required check로 실행한다.

## 6. Python 권장 패턴 — import-linter

Python에서는 `import-linter`를 이용해 import boundary를 검사할 수 있다.

적합한 규칙:

- forbidden import
- layered dependency
- module independence
- acyclic package boundary

예:

```text
presentation -> application -> domain
infrastructure -> application/domain

domain -X-> infrastructure
application -X-> presentation
```

Python import graph가 architecture contract의 중요한 부분인 프로젝트에서 특히 유효하다.

## 7. Tool 독립성

Architecture Fitness의 contract는 특정 도구가 아니다.

```text
ArchitectureCheck
  -> command
  -> exit status
  -> machine-readable result(optional)
  -> evidence artifact
```

지원 후보:

- ArchUnitNET
- import-linter
- custom static script
- compiler/analyzer rule
- dependency graph checker
- project-specific test

Controller는 tool-specific semantics를 core domain에 하드코딩하지 않는다.

## 8. Baseline 전략

레거시 repository에는 baseline mode를 권장한다.

초기 도입:

1. 현재 violation을 수집한다.
2. 기존 violation을 baseline artifact로 고정한다.
3. 신규 violation만 blocking 처리한다.
4. violation이 감소하면 baseline을 갱신한다.
5. baseline 증가에는 명시적 승인 근거가 필요하다.

이 방식은 "Clean Architecture로 전면 재작성" 같은 rewrite cascade 없이 구조 악화를 멈추게 한다.

## 9. Exception / Waiver

정당한 예외가 필요할 수 있다. 코드에 임의 ignore를 흩뿌리지 않는다.

권장 record:

```yaml
id: ARCH-WAIVER-003
rule: dependency-direction
scope: src/LegacyBridge/**
reason: staged migration bridge
owner: project
expires: 2026-12-31
reference: ADR-014
```

원칙:

- 이유 없는 영구 waiver 금지
- scope를 좁게 유지
- 가능하면 expiry 또는 제거 조건 명시
- Architect가 제안할 수 있으나 자기 판단만으로 silent bypass하지 않음
- Verifier는 waiver 존재와 유효 범위를 evidence에 포함

## 10. Architecture Change Detection

다음 변경은 architecture-sensitive로 태깅할 수 있다.

- 새 top-level module/project/package
- project reference 변경
- dependency direction 변경
- public interface 추가/대규모 변경
- persistence/integration boundary 변경
- runtime/provider abstraction 변경
- shared/common module 확장

이 태그가 있다고 무조건 Architect를 호출하지는 않는다. Task Contract와 Project Policy가 필요한 review path를 결정한다.

## 11. Verification Artifact

권장 결과:

```json
{
  "check": "dependency-direction",
  "status": "pass",
  "blocking": true,
  "tool": "ArchUnitNET",
  "command": "dotnet test tests/ArchitectureTests -c Debug",
  "baseline_delta": {
    "new": 0,
    "removed": 2
  },
  "artifacts": ["checks/architecture/dependency-direction.log"]
}
```

Verifier는 이 결과를 acceptance evidence 또는 required project evidence에 연결한다.

## 12. Reviewer와 Fitness Check의 관계

둘은 대체 관계가 아니다.

```text
Fitness Check
  -> known structural invariants
  -> deterministic

Reviewer
  -> unknown/semantic architecture risk
  -> evidence-based judgment
```

LLM Reviewer에게 이미 자동화 가능한 dependency direction을 매번 추론시키는 것은 낭비다.

반대로 모든 설계 품질을 static rule로 환원하려는 것도 잘못이다.

## 13. Architecture Rule Lifecycle

```text
Observed recurring architecture defect
 -> Architect/Reviewer finding
 -> rule candidate
 -> false-positive 검토
 -> Project rule 추가
 -> baseline if needed
 -> CI/Controller execution
 -> evidence 축적
```

반복되는 리뷰 지적은 가능한 경우 자동 check로 승격한다.

## 14. 도입 원칙

- 모든 프로젝트에 동일 architecture pattern을 강제하지 않는다.
- 프로젝트의 실제 구조와 migration 전략을 기준으로 rule을 정의한다.
- architecture test 도입 때문에 production code를 대규모로 재작성하지 않는다.
- 도구 설치가 사내 정책에 막히면 동일 contract를 custom script/test로 대체할 수 있다.
- architecture check 자체의 실행 시간과 false-positive rate도 관리 대상이다.

## 15. 설계 불변조건

1. 아키텍처 규칙은 가능한 경우 executable check로 내린다.
2. Architecture Fitness는 Project Profile/Verification의 일부이며 별도 control plane이 아니다.
3. legacy violation과 new violation을 구분할 수 있어야 한다.
4. architecture check 실패를 LLM이 임의로 무시할 수 없다.
5. tool-specific 구현은 adapter/command 뒤에 둔다.
6. waiver는 명시적 artifact로 추적한다.
7. Reviewer의 반복 finding은 자동화 가능성을 검토한다.
8. 문서상 architecture와 실제 dependency graph가 충돌하면 실제 evidence를 우선하고 문서를 수정한다.

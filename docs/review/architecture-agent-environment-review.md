# Architecture & Agent Environment Review — 2026-09-15

## 1. 리뷰 목적

기존 Agent Forge 설계를 다시 검토하되 이번에는 **아키텍처 품질**과 **Agent 실행환경의 결정성/운영성**을 동시에 본다.

검토 대상은 현재 설계 문서와 다음 외부 패턴이다.

- wshobson/agents
- github/awesome-copilot
- alirezarezvani/claude-skills
- Jeffallan/claude-skills
- TNG/ArchUnitNET
- seddonym/import-linter

외부 프로젝트의 popularity나 Agent 수를 품질의 근거로 사용하지 않는다. Agent Forge의 제약과 목적에 맞는 패턴만 채택한다.

## 2. 리뷰 관점

1. **Architecture integrity** — control/data plane, source of truth, dependency boundary가 명확한가
2. **Role & autonomy boundary** — Agent가 늘어나며 책임/권한이 흐려지지 않는가
3. **Weak-model determinism** — 낮은 모델 성능에서도 task framing과 context가 안정적인가
4. **Enforceable verification** — 문서 규칙이 실제 check/gate로 내려가는가
5. **Portability & supply chain** — 외부 Skill과 multi-harness 확장이 통제 가능한가

Severity:

- `HIGH`: 구현 전에 구조 보강 필요
- `MEDIUM`: MVP에서 경계 명시 필요
- `LOW`: 운영 중 지표로 확인 가능

## 3. Finding 1 — Architecture rule이 문서에만 존재함

**Severity: HIGH**

### 관찰

기존 문서는 dependency direction, scope, architecture/SOLID review를 강조하지만 실제 구조 위반은 주로 Reviewer 판단에 의존한다.

### 위험

- 같은 실수를 반복해서 LLM에게 다시 리뷰시킴
- 약한 모델에서 dependency violation 탐지 품질이 흔들림
- 문서와 실제 코드 구조가 장기간 drift 가능

### 외부 패턴

- ArchUnitNET: C# dependency/type/member rule을 test로 실행
- import-linter: Python import boundary를 contract로 실행

### 결정

`09-architecture-governance-and-fitness.md`를 추가한다.

Architecture Fitness는 별도 control plane이 아니라 **Project Profile + CheckRunner + Verification Gate**로 구현한다.

Legacy repository에는 baseline mode를 지원해 existing violation과 new violation을 구분한다.

## 4. Finding 2 — Role taxonomy가 확장되면 Agent 폭증 가능

**Severity: HIGH**

### 관찰

기존 문서에 `planner`, `researcher`, `project-expert`, `domain-expert`, `implementer`, `reviewer`, `verifier`, `debugger` 등이 혼재하고 향후 `architect`, 언어별 expert가 추가될 여지가 있다.

### 위험

- 역할과 전문성이 섞임
- `dotnet-architect-reviewer` 같은 조합형 Agent 복제 증가
- Orchestrator routing 난이도 증가
- weak model에서 비슷한 Agent를 잘못 선택

### 결정

`08-role-contracts-and-agent-environment.md`를 추가하고 canonical Role을 다음으로 제한한다.

```text
orchestrator
project-expert
domain-expert
architect
implementer
debugger
reviewer
verifier
```

언어/framework/security/research 성격은 기본적으로 Skill/Domain으로 표현한다.

새 Role은 기존 Role과 다른 **authority 또는 write boundary**가 필요한 경우에만 추가한다.

## 5. Finding 3 — Agent environment가 Role 이름에 암묵적으로 묶일 위험

**Severity: MEDIUM**

### 관찰

`reviewer = read-only`, `implementer = write` 같은 규칙은 문서상 존재하지만 runtime capability contract로 더 명시할 필요가 있다.

### 위험

- backend별 permission 차이를 Role semantics로 오인
- advisory prompt와 enforced permission 혼동
- 새로운 runtime adapter 추가 시 보안 claim drift

### 결정

Role과 capability를 분리한다.

```text
Role Contract
  + Resolved Capability
  + Harness
  -> Runtime-native artifact
```

Capability에는 filesystem, shell, network, git, delegation, context source, output contract를 명시한다.

## 6. Finding 4 — Multi-agent workflow를 많이 호출하는 것이 품질이라는 오해

**Severity: MEDIUM**

### 관찰

Jeffallan/claude-skills 같은 프로젝트는 역할 체인을 설명하는 좋은 예를 제공하지만 긴 workflow chain을 Agent Forge가 그대로 도입하면 현재 설계 철학과 충돌한다.

### 위험

- context 전달 손실
- 같은 모델의 의견을 여러 번 반복
- latency 증가
- 책임 소재 불명확
- reviewer가 이전 Agent 서사를 그대로 따라감

### 결정

고정 chain은 참고용으로만 사용한다.

```text
small task -> 최소 Role
architecture-sensitive task -> Architect 추가
special domain task -> Expert 추가
```

Controller는 fan-out/depth budget을 유지하고 Orchestrator는 필요한 최소 Role만 요청한다.

## 7. Finding 5 — Skill Registry에 공급망 경계가 부족함

**Severity: HIGH**

### 관찰

기존 Skill Registry는 metadata/compatibility/conflict를 다루지만 외부 Skill을 가져오는 순간의 trust model이 충분히 정의되지 않았다.

### 위험

Skill은 다음을 포함할 수 있다.

- shell script
- network call
- package install
- filesystem mutation
- prompt-level policy override
- transitive reference

단순 Markdown으로 취급하면 안 된다.

### 외부 패턴

- alirezarezvani/claude-skills: pre-install skill security audit
- github/awesome-copilot: bundled script/reference와 progressive disclosure
- wshobson/agents: external payload pinning과 structural validation

### 결정

`10-skill-intake-portability-and-evaluation.md`를 추가한다.

외부 자산의 상태를:

```text
untrusted -> staged -> audited -> evaluated -> approved/pinned
```

으로 관리한다.

Runtime 중 unknown remote Skill 자동 설치는 기본 금지한다.

## 8. Finding 6 — Cross-harness 전략은 있지만 canonical/target 구분을 더 강화해야 함

**Severity: MEDIUM**

### 관찰

기존 Runtime Adapter 경계는 타당하다. 그러나 향후 OpenCode 외 Codex/Claude/Copilot을 지원할 때 source file을 어느 형식으로 관리할지 명확하지 않으면 각 provider 파일이 다시 source of truth가 될 수 있다.

### 외부 패턴

wshobson/agents의 핵심 장점은 하나의 source에서 target harness별 native artifact를 생성하고 capability 차이를 별도 관리하는 것이다.

### 결정

```text
Canonical Role/Project/Domain/Skill/Harness
  -> Resolved Harness IR
  -> Runtime Adapter
  -> harness-native generated artifact
```

을 기준으로 한다.

Generated artifact는 canonical이 아니며 수동 편집하지 않는다.

표현 불가능한 capability는 warning/fail로 드러내고 조용히 삭제하지 않는다.

## 9. Finding 7 — Skill routing에는 negative trigger가 필요함

**Severity: MEDIUM**

### 관찰

Skill을 "언제 사용하는가"만 적으면 weak model은 유사 task마다 과활성화할 수 있다.

### 외부 패턴

github/awesome-copilot의 Skill description은 trigger와 함께 "Do not trigger for..." 범위를 명시하는 사례가 있다.

### 결정

Skill metadata에 `triggers`와 `non_triggers`를 함께 둔다.

초기 routing은:

```text
Role defaults
+ Project required
+ relevant Domain
+ explicit Task/Orchestrator selection
```

정도로 제한한다.

semantic router/embedding은 실제 recall 문제가 확인될 때 추가한다.

## 10. Finding 8 — Codebase grounding을 독립 산출물로 다룰 가치가 있음

**Severity: LOW / ADOPT AS SKILL**

### 관찰

Project Expert가 repo 전체를 처음 접할 때 매번 자유형 분석을 하면 결과 형식과 evidence 품질이 흔들릴 수 있다.

### 외부 패턴

github/awesome-copilot의 `acquire-codebase-knowledge`는 stack/structure/architecture/testing/concerns를 evidence path와 함께 고정 산출물로 만든다.

### 결정

MVP core component를 추가하지 않는다.

대신 향후 `codebase-grounding` Skill 후보로 둔다.

원칙:

- verifiable fact only
- unknown은 TODO/unknown
- intent 문서와 실제 code reality 구분
- evidence path 필수

Project Expert의 optional bootstrap skill로 사용하는 것이 적합하다.

## 11. Finding 9 — Skill 품질 평가는 단계화해야 함

**Severity: LOW / DESIGN NOW, IMPLEMENT LATER**

### 관찰

wshobson/agents는 static + LLM judge + 반복 실행 평가를 분리한다. 방향은 좋지만 Agent Forge MVP에서 이를 모두 구현하면 과설계다.

### 결정

평가 계층만 정의하고 구현은 필요한 수준부터 시작한다.

```text
Level 1 Static
  schema / broken reference / unsafe script / compatibility

Level 2 Behavioral
  representative task / output contract

Level 3 Runtime
  actual OpenCode execution

Level 4 Statistical or LLM judge
  only when justified
```

## 12. 외부 프로젝트별 최종 판단

| Source | 채택 | 경계/비채택 |
|---|---|---|
| wshobson/agents | canonical source, harness-native generation, validation, capability matrix | 대규모 Agent catalog 자체는 목표 아님 |
| github/awesome-copilot | progressive disclosure, trigger/non-trigger, evidence contract, bundled resources | Skill 수 확대보다 필요한 것만 이식 |
| alirezarezvani/claude-skills | security audit, bounded harness, portability 아이디어 | 자체 프레임워크 전체 복제 안 함 |
| Jeffallan/claude-skills | context-aware activation, workflow 표현 | 고정 multi-agent chain 기본화 안 함 |
| ArchUnitNET | C# executable architecture rule | 모든 C# 프로젝트 강제 dependency 아님 |
| import-linter | Python import/layer fitness rule | Python 전체 architecture를 이것 하나로 판단하지 않음 |

## 13. 변경 후 권장 전체 구조

```text
User
  -> Orchestrator
      -> Task Contract
      -> minimal Role selection
  -> Controller
      -> Registry
      -> Harness Compiler
      -> Workspace
      -> Policy Gate
      -> Runtime Adapter
      -> CheckRunner
          -> Functional Checks
          -> Architecture Fitness Checks
      -> Reviewer
      -> Verifier
      -> Artifact / Event / State

Management path:
External Skill
  -> Intake / Audit / Evaluate / Pin
  -> Canonical Registry
  -> Harness Adapter
```

## 14. 최종 판정

### 유지해야 할 기존 설계

- Orchestrator / Controller 권한 분리
- Task Contract
- Worktree isolation
- Reviewer / Verifier 분리
- deterministic completion gate
- OpenCode adapter boundary
- Herdr optional UI

이 부분은 재설계할 이유가 없다.

### 이번에 보강해야 할 부분

1. Role contract와 Agent environment를 명시적으로 분리
2. Architect Role을 제한적으로 추가
3. Architecture Fitness / baseline / waiver 정의
4. External Skill intake/provenance/security 정의
5. Canonical IR -> harness-native generation 경계 정의
6. Skill trigger + non-trigger와 progressive disclosure 정의

### 현재 상태

**설계 방향은 유효하다.** 문제는 core architecture가 잘못된 것이 아니라, 실제 구현에 들어가면 흔들릴 가능성이 높은 governance 경계가 비어 있던 것이다.

이번 보강 후 Agent Forge의 핵심 흐름은 다음으로 정리된다.

```text
Grounding
 -> Task Contract
 -> Minimal Role Selection
 -> Controlled Execution
 -> Architecture/Functional Deterministic Checks
 -> Independent Review
 -> Evidence-based Completion
```

다음 구현 단계에서 우선 검증할 것은 Agent 수가 아니라 **OpenCode runtime capability, resolved harness 재현성, worktree enforcement, architecture check integration**이다.

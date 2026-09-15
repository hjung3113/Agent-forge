# 10. Skill Intake, Portability & Evaluation

## 1. 목적

Agent Forge는 외부 Agent/Skill 저장소를 참고할 수 있지만 **외부 Skill을 신뢰된 실행 자산으로 바로 취급하지 않는다.**

Skill은 prompt 조각이 아니라 실행 지침, script, reference, tool 요구사항을 포함할 수 있으므로 dependency와 유사한 공급망 관리가 필요하다.

또한 Agent Forge의 canonical 정의는 특정 harness 전용 문법에 종속되지 않아야 한다.

```text
External Source
  -> Stage
  -> Inspect / Audit
  -> Normalize
  -> Compatibility Check
  -> Evaluate
  -> Approve + Pin
  -> Canonical Skill Registry
  -> Harness Adapter
  -> OpenCode/Codex/Claude/... native artifact
```

## 2. 핵심 원칙

1. 외부 Skill/Agent는 처음에는 **untrusted content**다.
2. source URL만 저장하지 않고 revision/hash/provenance를 남긴다.
3. canonical source와 generated harness artifact를 구분한다.
4. 모든 Skill을 한꺼번에 context에 넣지 않는다.
5. 자동 semantic routing보다 명시적 trigger/capability matching을 먼저 사용한다.
6. harness portability는 lowest-common-denominator가 아니라 **canonical intent + harness-native rendering**으로 해결한다.
7. Skill 품질 평가는 "문법이 맞다"와 "실제로 도움이 된다"를 구분한다.

## 3. Canonical Skill Package

권장 구조:

```text
skills/<skill-id>/
├─ skill.yaml
├─ SKILL.md
├─ references/      # optional
├─ scripts/         # optional
├─ assets/          # optional
└─ tests/           # optional
```

`SKILL.md`는 사람/Agent가 읽는 절차 문서다.

`skill.yaml`은 Controller/Registry가 읽는 최소 metadata다.

예:

```yaml
id: root-cause-debugging
version: 1
source:
  type: local
  upstream: https://github.com/example/repo
  revision: <commit-sha>
  license: MIT
integrity:
  sha256: <package-hash>
compatible_roles:
  - debugger
  - implementer
triggers:
  - reproducible bug investigation
non_triggers:
  - architecture redesign
requires:
  tools: [read, grep, test]
  network: false
conflicts_with: []
```

## 4. Trigger와 Non-trigger

Skill description에는 "언제 쓰는가"뿐 아니라 **언제 쓰지 않는가**도 포함한다.

좋은 예:

```text
Use when:
- bug is reproducible
- root cause is unknown

Do not use when:
- user only asks for formatting
- architecture decision is the main task
```

약한 모델에서는 trigger만 있으면 비슷한 Skill을 과도하게 활성화할 수 있다.

## 5. Progressive Disclosure

Skill은 단계적으로 로드한다.

```text
Registry metadata
  -> selection
  -> SKILL.md
  -> required reference only
  -> script/asset only when used
```

전체 Skill repository를 prompt에 넣지 않는다.

권장:

- routing 단계: id, description, triggers, role compatibility만
- execution 단계: selected SKILL.md
- 세부 reference: task가 요구할 때만

## 6. External Skill Intake

외부 자산을 들여올 때 다음 순서를 사용한다.

### Stage 1 — Discover

기록:

- upstream repository
- exact path
- revision/commit
- license
- intended use
- owner/maintainer signal

Star 수는 참고 신호일 뿐 승인 기준이 아니다.

### Stage 2 — Static Inspection

검사:

- unexpected executable/script
- network call
- credential/secret access
- shell command
- filesystem write scope
- package install
- prompt injection성 지시
- destructive command
- hidden/generated binary
- symlink/path traversal

### Stage 3 — Semantic Review

확인:

- 실제 목적이 metadata와 일치하는가
- 역할 경계를 침범하는가
- "항상 실행", "정책 무시" 같은 과도한 지시가 있는가
- 기존 Skill과 내용이 중복되는가
- 검증 없이 완료를 선언하도록 유도하는가

### Stage 4 — Compatibility Check

확인:

- Agent Forge Role과 맞는가
- 요구 tool이 현재 runtime에서 가능한가
- network/filesystem requirement가 정책과 충돌하지 않는가
- harness adapter에서 손실되는 기능이 있는가

### Stage 5 — Evaluation

최소 세 층으로 나눈다.

```text
Static
  -> metadata/schema/reference integrity

Behavioral
  -> representative task에서 output contract/절차 준수

Runtime
  -> 실제 target harness에서 실행 가능한지
```

필요할 때만 LLM judge나 반복 통계 평가를 추가한다.

Level별 승인 요건:

| Level | 승인 필수 여부 | 관심 |
|---|---|---|
| Static | 항상 | 보안 |
| Behavioral | script/tool을 포함한 Skill에 필수 | 안전 + 품질 |
| Runtime | runtime tool을 요구하는 Skill에 필수 | 안전 + 품질 |
| LLM judge/통계 | 승인에 필수 아님 | 품질만, 사후 |

Behavioral 평가는 최소 1개 representative task를 실제 harness에서 실행하고 사람이 결과를 검토한다. 승인 시 이 평가 기록을 provenance에 포함한다.

### Stage 6 — Approve / Reject / Adapt

가능한 결과:

- `approved-as-is`
- `approved-adapted`
- `reference-only`
- `rejected`

외부 원문을 그대로 복사하는 것보다 필요한 원칙만 재작성하는 경우 `approved-adapted`로 provenance를 남긴다.

## 7. Provenance와 Pinning

승인된 외부 Skill에는 최소 다음을 남긴다.

```text
upstream repository
source path
source revision
license
import/adaptation date
local modifications summary
integrity hash
review status
```

upstream `main`을 런타임에 직접 따라가지 않는다.

업데이트는 새 revision을 stage한 뒤 재검증한다.

## 8. Skill Supply-chain Rule

기본 금지:

- runtime 중 임의 `git clone` 후 즉시 실행
- unknown script 자동 실행
- unpinned remote content include
- Skill이 자체적으로 Agent Forge policy 변경
- Skill이 승인 없이 다른 Skill을 설치

사내 환경에서는 특히 **approved local pack**을 기본 배포 단위로 둔다.

### 8.1 내부 생성 룰의 승격 경로

project memory([05-project-workspace-and-context.md](05-project-workspace-and-context.md) §9.1의 Learned Memory) entry는 승인 게이트를 통해서만 Skill/Domain/Project context로 승격된다.

- 승격은 사람 또는 Reviewer 승인을 거친다. Stage 6의 `approved-adapted` 패턴을 재사용한다.
- 자동 승격과 자동 전역화는 금지다.
- 자동 추출 자체는 MVP 범위 밖이다. MVP는 run artifact를 이후 수동 추출의 데이터 소스로 사용한다.

## 9. Canonical IR와 Harness Adapter

Agent Forge 내부에서는 provider/harness-neutral한 **Resolved Harness IR**을 만든다.

예:

```yaml
role: reviewer
instructions:
  - ...
skills:
  - code-review
capabilities:
  filesystem: read_only
  shell: allowlisted
output_contract: review-v1
context:
  - task_contract
  - final_diff
```

그 후 adapter가 각 harness의 native artifact로 변환한다.

```text
Resolved Harness IR
  |- OpenCodeAdapter -> .opencode/...
  |- CodexAdapter    -> .codex/...
  |- ClaudeAdapter   -> .claude/...
  `- CopilotAdapter  -> .github/... or native form
```

MVP는 OpenCode만 구현한다. 다른 adapter는 구조적 확장 포인트다.

## 10. Lowest-common-denominator 금지

모든 harness가 동일 capability를 제공한다고 가정하지 않는다.

각 adapter는 capability matrix를 가진다.

예:

| Capability | OpenCode | Harness B | 처리 |
|---|---:|---:|---|
| project instruction | Yes | Yes | native |
| tool allowlist | partial | Yes | Controller enforcement 유지 |
| subagent | Yes | No | Controller delegation 사용 |
| hook | partial | Yes | optional |

변환 시 손실이 있으면 조용히 버리지 않고 `compatibility warning`을 만든다.

## 11. Generated Artifact 원칙

canonical source:

```text
roles/
projects/
domains/
skills/
harnesses/
```

generated artifact:

```text
.agent-forge/generated/<runtime>/...
```

원칙:

- generated file을 사람이 직접 canonical하게 수정하지 않는다.
- source 변경 후 다시 생성 가능해야 한다.
- generated artifact에는 source revision/provenance를 넣는다.
- diff가 필요하면 source diff와 generated diff를 구분한다.

## 12. Skill Selection

MVP selection:

```text
Role defaults
+ Project required skills
+ Domain recommended skills when relevant
+ explicit Task/Orchestrator skills
```

향후 capability/tag matching을 추가할 수 있다.

초기부터 embedding/LLM semantic router를 필수화하지 않는다.

충돌 시:

- incompatible role -> reject
- tool requirement unavailable -> BLOCKED
- duplicate/overlap -> smaller set 우선
- permission relaxation request -> reject

## 13. Skill Quality Metrics

Skill 수가 아니라 다음을 본다.

- trigger precision
- task success contribution
- context cost
- output contract adherence
- false activation rate
- policy violation rate
- deterministic verification coverage
- maintenance/provenance freshness

실행 데이터가 쌓이기 전에는 복잡한 점수 모델을 만들지 않는다.

MVP는 metrics 자동화 없이 run artifact의 skill 주입 기록과 결과를 정기(예: 월 1회) 수동 리뷰로 대체한다. 승인된 skill은 연 1회, 또는 runtime/model 대량 변경 시 재평가한다.

## 14. 외부 프로젝트에서 채택할 패턴

### wshobson/agents

채택:

- single source of truth -> harness-native generation
- progressive loading
- structural validation
- cross-harness capability matrix

비채택:

- Agent/Skill 수 자체를 목표로 삼는 방식
- 대규모 catalog를 기본 context로 노출

### github/awesome-copilot

채택:

- `SKILL.md + references/scripts/assets`
- explicit trigger/non-trigger
- progressive disclosure
- evidence/output contract

### alirezarezvani/claude-skills

채택:

- external Skill security audit 관점
- bounded/self-verifying harness 관점
- cross-harness index/translation 아이디어

### Jeffallan/claude-skills

채택:

- context-aware activation
- workflow composition을 설명하는 방식

비채택:

- 여러 전문 Agent를 항상 순차 호출하는 고정 체인

### ArchUnitNET / import-linter

이들은 Skill source가 아니라 **deterministic Architecture Fitness 도구**로 취급한다. 자세한 내용은 [09-architecture-governance-and-fitness.md](09-architecture-governance-and-fitness.md)를 따른다.

## 15. 설계 불변조건

1. 외부 Skill은 audit/approval 전까지 untrusted다.
2. 승인 자산은 revision과 integrity를 추적한다.
3. canonical definition과 target-harness artifact를 분리한다.
4. Skill은 on-demand로 로드한다.
5. adapter가 표현하지 못하는 capability를 조용히 삭제하지 않는다.
6. Skill이 hard policy를 완화할 수 없다.
7. Skill 수보다 trigger precision과 verification 기여도를 우선한다.
8. 외부 marketplace 자동 설치는 기본 경로가 아니다.

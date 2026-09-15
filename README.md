# Agent Forge

Agent Forge는 **제약된 사내 환경에서 OpenCode를 실행 백엔드로 사용해 역할·프로젝트·도메인·스킬별 에이전트를 조합하고 실행하는 로컬 멀티에이전트 런타임/하네스**다.

목표는 Grok 계열 봇이나 Hermes 같은 외부 에이전트 런타임을 그대로 복제하는 것이 아니다. 별도 API key나 임의의 외부 gateway를 요구하지 않고, **사내에서 허용된 OpenCode를 실행 엔진으로 고정한 채 orchestration, project-aware harness, worktree isolation, verification, policy enforcement를 자체적으로 제공**하는 것이 목표다.

## 핵심 구조

```text
User
  |
  v
Orchestrator Agent
  |
  | structured delegation request
  v
Deterministic Controller
  |-- Agent / Skill / Project Registry
  |-- Scheduler / State Machine
  |-- Policy / Permission Gate
  |-- Workspace / Git Worktree Manager
  |-- Runtime Adapter (OpenCode first)
  |-- CheckRunner
  |     |-- Functional checks
  |     `-- Architecture Fitness checks
  |-- Result / Event Store
  |
  +--> Project Expert
  +--> Domain Expert
  +--> Architect
  +--> Implementer
  +--> Debugger
  +--> Reviewer
  +--> Verifier

External Skill
  -> Stage / Audit / Evaluate / Pin
  -> Canonical Skill Registry
  -> Harness Adapter

Herdr = optional observability / operator UI
```

Agent Forge의 기본 원칙은 다음과 같다.

1. **LLM이 판단하고 프로그램이 통제한다.** 총괄 에이전트는 필요한 에이전트와 작업을 제안하지만 spawn, 권한, 동시성, 상태 전이, retry, 완료 판정은 Controller가 소유한다.
2. **Agent는 단순 prompt가 아니다.** `Base Runtime + Role + Project + Domain + Skills + Task Contract`를 합성한 실행 프로파일이다.
3. **Role은 작고 명확하게 유지한다.** 언어/framework 전문성을 새 Agent Role로 복제하지 않고 Skill/Domain으로 표현한다.
4. **프로젝트 작업은 격리한다.** 프로젝트별 canonical clone/cache를 두고 task별 Git worktree를 만들어 그 디렉터리에서 OpenCode를 실행한다.
5. **권한은 prompt로만 보장하지 않는다.** filesystem, shell, path, git operation 같은 제약은 호스트 Controller가 검증하고 강제한다.
6. **검증은 구현 에이전트와 분리한다.** Reviewer/Verifier는 구현 reasoning이 아니라 Task Contract, diff, test 결과, 필요한 코드만 받아 독립적으로 검증한다.
7. **아키텍처 규칙은 가능한 경우 실행 가능하게 만든다.** C#/Python dependency/layer invariant는 project-owned architecture test/check로 내려 완료 gate에 연결한다.
8. **외부 Skill은 공급망 자산으로 취급한다.** audit/approval/pinning 전에는 untrusted이며, canonical source와 runtime별 generated artifact를 구분한다.
9. **Herdr는 control plane이 아니다.** pane 조작을 핵심 프로토콜로 삼지 않고 관측/수동 개입 UI로 사용한다.

## 문서 읽기

전체 문서 인덱스와 canonical responsibility는 **[docs/README.md](docs/README.md)** 를 기준으로 한다.

처음 읽는 경우:

1. [Project Overview](docs/design/01-project-overview.md)
2. [System Architecture](docs/design/02-system-architecture.md)
3. [Agent Composition & Harness](docs/design/03-agent-composition-and-harness.md)
4. [Orchestration & Runtime](docs/design/04-orchestration-runtime.md)
5. [Project Workspace & Context](docs/design/05-project-workspace-and-context.md)
6. [Safety, Verification & Observability](docs/design/06-safety-verification-and-observability.md)
7. [Role Contracts & Agent Environment](docs/design/08-role-contracts-and-agent-environment.md)
8. [Architecture Governance & Fitness](docs/design/09-architecture-governance-and-fitness.md)
9. [Agent Registry & Extension](docs/design/07-agent-registry-and-extension.md)
10. [Skill Intake, Portability & Evaluation](docs/design/10-skill-intake-portability-and-evaluation.md)
11. [Architecture & Agent Environment Review](docs/review/architecture-agent-environment-review.md)
12. [MVP Roadmap](docs/roadmap/mvp-roadmap.md)

## 현재 상태

현재 저장소는 **설계 단계**다. 문서가 구현의 기준이며, 구현 중 설계 경계가 바뀌면 관련 설계 문서를 함께 갱신한다.

다음 구현 우선순위는 Agent 수를 늘리는 것이 아니라:

```text
OpenCode runtime capability
 -> single-agent reproducibility
 -> worktree/policy enforcement
 -> role/harness resolution
 -> functional + architecture checks
 -> reviewer/verifier
 -> orchestrator delegation
```

순서로 검증하는 것이다.

# Agent Forge Documentation Guide

이 문서는 Agent Forge 문서의 **인덱스이자 읽기 가이드**다. 문서마다 책임을 분리해 같은 사실을 여러 문서에서 중복 정의하지 않는 것을 원칙으로 한다.

## 문서 역할

| 문서 | 역할 | 이 문서가 답하는 질문 |
|---|---|---|
| [01-project-overview.md](design/01-project-overview.md) | 제품 목적과 범위 | 왜 만드는가, 무엇을 대체하며 무엇은 대체하지 않는가? |
| [02-system-architecture.md](design/02-system-architecture.md) | 상위 아키텍처와 모듈 경계 | 전체 시스템은 어떤 컴포넌트로 나뉘는가? |
| [03-agent-composition-and-harness.md](design/03-agent-composition-and-harness.md) | Agent/Harness 합성 모델 | 역할·프로젝트·도메인·스킬을 어떻게 조합하는가? |
| [04-orchestration-runtime.md](design/04-orchestration-runtime.md) | 총괄 Agent, Controller, task lifecycle | 누가 판단하고 누가 실행·통제하는가? |
| [05-project-workspace-and-context.md](design/05-project-workspace-and-context.md) | 프로젝트 등록, clone/worktree, context | 특정 프로그램 Agent를 해당 프로젝트에서 어떻게 실행하는가? |
| [06-safety-verification-and-observability.md](design/06-safety-verification-and-observability.md) | 권한, 검증, 로그, Herdr | 안전성과 완료 신뢰성을 어떻게 확보하는가? |
| [07-agent-registry-and-extension.md](design/07-agent-registry-and-extension.md) | Agent/Skill 추가·수정·삭제 | 런타임에서 Agent를 어떻게 확장하는가? |
| [forgeroom-reuse-analysis.md](reference/forgeroom-reuse-analysis.md) | ForgeRoom 비교 및 재사용 판단 | 기존 ForgeRoom에서 무엇을 가져오고 무엇을 버리는가? |
| [architecture-review.md](review/architecture-review.md) | 5개 관점 적대적 설계 리뷰 | 현재 설계의 현실적 실패 지점과 보강 결정은 무엇인가? |
| [mvp-roadmap.md](roadmap/mvp-roadmap.md) | 구현 순서와 단계별 완료 조건 | 어떤 순서로 구현해야 과설계를 피할 수 있는가? |

## 권장 읽기 순서

### 처음 설계를 이해할 때

```text
01 Project Overview
  -> 02 System Architecture
  -> 03 Agent Composition & Harness
  -> 04 Orchestration & Runtime
  -> 05 Project Workspace & Context
  -> 06 Safety / Verification / Observability
  -> 07 Agent Registry & Extension
  -> Architecture Review
```

### 구현을 시작할 때

```text
02 System Architecture
  -> 04 Orchestration & Runtime
  -> 03 Agent Composition & Harness
  -> 05 Project Workspace & Context
  -> 06 Safety / Verification / Observability
  -> Architecture Review
  -> MVP Roadmap
```

### ForgeRoom 자산을 옮길 때

```text
ForgeRoom Reuse Analysis
  -> 대상 Agent Forge 설계 문서
  -> 기존 ForgeRoom 구현/ADR
```

## 문서 우선순위

충돌 시 다음 순서를 따른다.

1. 현재 Agent Forge 설계 문서
2. 향후 Agent Forge ADR
3. Agent Forge 구현
4. ForgeRoom 참고 문서

ForgeRoom은 **참고 구현/설계 자산**이지 Agent Forge의 canonical specification이 아니다.

## 문서 변경 규칙

- 모듈 책임이나 public contract가 바뀌면 관련 설계 문서를 함께 수정한다.
- 같은 사실을 여러 문서에 복사하지 않는다. 다른 문서에서는 링크로 참조한다.
- 구현 세부사항보다 **경계, 불변조건, 데이터 흐름, 실패 처리**를 우선 기록한다.
- 실험적 아이디어와 확정 설계를 섞지 않는다. 확정 전 아이디어는 roadmap 또는 향후 ADR에 둔다.
- `OpenCode`, `Herdr`, Git provider 등 외부 도구의 세부 동작은 adapter 뒤에 숨기고 문서에는 Agent Forge가 의존하는 최소 계약만 적는다.

## 핵심 용어

- **Orchestrator Agent**: 사용자 목표를 해석하고 필요한 Agent 호출을 제안하는 총괄 LLM Agent.
- **Controller**: spawn, 상태 전이, 권한, retry, concurrency, workspace, 결과를 결정적으로 통제하는 프로그램.
- **Agent Profile**: Role/Project/Domain/Skill 조합을 가리키는 논리적 Agent 정의.
- **Harness Profile**: Agent가 실행될 때 적용되는 지침, skills, tools, permissions, output contract 묶음.
- **Project Profile**: repository, build/test 명령, project context, 허용 경로 등을 정의하는 프로젝트 설정.
- **Task Contract**: objective, scope, exclusions, acceptance criteria, verification evidence, stop condition을 포함한 실행 계약.
- **Workspace**: task별 격리된 Git worktree.
- **Runtime Backend**: 실제 LLM coding agent를 실행하는 backend. 초기 구현은 OpenCode.

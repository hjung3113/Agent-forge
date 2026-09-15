# Agent Forge Working Guide

이 문서는 Agent Forge 저장소에서 작업하는 사람과 Agent의 진입점이다.

## First-entry order

1. [docs/README.md](docs/README.md) — 문서 인덱스와 읽기 순서
2. [docs/design/01-project-overview.md](docs/design/01-project-overview.md) — 제품 목적과 범위
3. [docs/design/02-system-architecture.md](docs/design/02-system-architecture.md) — 시스템 경계
4. 작업 대상과 직접 관련된 설계 문서
5. [docs/review/architecture-review.md](docs/review/architecture-review.md) — 현재 설계 리스크와 금지 패턴

## Architectural rules

- **Orchestrator는 판단하고 Controller가 통제한다.** LLM이 subprocess, worktree, task state authority를 직접 소유하게 만들지 않는다.
- **Herdr는 UI다.** pane screen scraping이나 agent-to-pane typing을 핵심 프로토콜로 만들지 않는다.
- **OpenCode는 adapter 뒤에 둔다.** core domain에 OpenCode CLI 세부 옵션을 퍼뜨리지 않는다.
- **Agent는 composition이다.** Role/Project/Domain/Skill 내용을 Agent 정의에 중복 복사하지 않는다.
- **Project와 Harness를 분리한다.** repository identity와 execution contract를 같은 profile로 합치지 않는다.
- **Prompt permission은 security boundary가 아니다.** 실제 권한은 host-side gate와 post-run validation으로 강제한다.
- **동시 writer는 worktree를 공유하지 않는다.** 병렬 source write는 별도 workspace를 사용한다.
- **DONE에는 evidence가 필요하다.** build/test/review/acceptance evidence 없이 완료 상태로 전이하지 않는다.
- **ForgeRoom은 reference다.** ForgeRoom의 product-specific 기능을 근거 없이 Agent Forge로 이식하지 않는다.

## Scope discipline

MVP에 기본 포함:

- local single-process Controller
- OpenCode runtime adapter
- Agent/Skill/Project/Harness registry
- Git worktree
- structured delegation
- PolicyGate
- Reviewer/Verifier
- artifact/event/state

명확한 필요가 생기기 전 제외:

- workflow DSL
- distributed queue
- Redis/message broker
- model gateway/router
- marketplace
- full RAG platform
- automatic merge
- GUI

## Documentation rule

모듈 책임, 상태 authority, profile merge rule, security boundary가 바뀌면 관련 design 문서를 같은 변경에서 갱신한다.

새 개념을 추가하기 전에 기존 `Role / Project / Domain / Skill / Harness / Task Contract` 중 하나로 표현할 수 없는지 먼저 확인한다.

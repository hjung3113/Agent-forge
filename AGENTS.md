# Agent Forge Working Guide

이 문서는 Agent Forge 저장소에서 작업하는 사람과 Agent의 진입점이다.

## First-entry order

1. [docs/README.md](docs/README.md) — 문서 인덱스와 canonical responsibility
2. [docs/design/01-project-overview.md](docs/design/01-project-overview.md) — 제품 목적과 범위
3. [docs/design/02-system-architecture.md](docs/design/02-system-architecture.md) — 시스템 경계
4. 작업 대상과 직접 관련된 설계 문서
5. [docs/design/08-role-contracts-and-agent-environment.md](docs/design/08-role-contracts-and-agent-environment.md) — Role/권한/context 경계
6. [docs/design/09-architecture-governance-and-fitness.md](docs/design/09-architecture-governance-and-fitness.md) — architecture rule/check 경계
7. [docs/review/architecture-agent-environment-review.md](docs/review/architecture-agent-environment-review.md) — 최신 아키텍처/Agent 환경 적대적 리뷰

## Architectural rules

- **Orchestrator는 판단하고 Controller가 통제한다.** LLM이 subprocess, worktree, task state authority를 직접 소유하게 만들지 않는다.
- **Herdr는 UI다.** pane screen scraping이나 agent-to-pane typing을 핵심 프로토콜로 만들지 않는다.
- **OpenCode는 adapter 뒤에 둔다.** core domain에 OpenCode CLI 세부 옵션을 퍼뜨리지 않는다.
- **Agent는 composition이다.** Role/Project/Domain/Skill 내용을 Agent 정의에 중복 복사하지 않는다.
- **Role은 authority 경계다.** language/framework/security 전문성을 이유로 Role을 증식하지 않는다.
- **Architect는 read-only 기본이다.** 구조 결정/불변조건/fitness rule을 제안하지만 구현·완료 authority를 소유하지 않는다.
- **Project와 Harness를 분리한다.** repository identity와 execution contract를 같은 profile로 합치지 않는다.
- **Prompt permission은 security boundary가 아니다.** 실제 권한은 host-side gate와 post-run validation으로 강제한다.
- **동시 writer는 worktree를 공유하지 않는다.** 병렬 source write는 별도 workspace를 사용한다.
- **DONE에는 evidence가 필요하다.** build/test/review/acceptance evidence 없이 완료 상태로 전이하지 않는다.
- **Architecture rule은 가능하면 executable check로 내린다.** 반복되는 dependency/layer 위반을 LLM review에만 맡기지 않는다.
- **Legacy architecture에는 baseline을 허용하되 신규 위반은 막는다.** architecture test 도입을 대규모 rewrite의 구실로 쓰지 않는다.
- **외부 Skill은 untrusted다.** audit/approval/pinning 없이 runtime에 자동 설치하거나 실행하지 않는다.
- **Canonical source와 generated harness artifact를 구분한다.** runtime별 생성 파일을 source of truth로 직접 수정하지 않는다.
- **ForgeRoom과 외부 Agent/Skill repo는 reference다.** popularity나 기존 구현을 이유로 기능을 그대로 이식하지 않는다.

## Canonical roles

MVP 기본 Role은 다음으로 제한한다.

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

새 Role을 추가하려면 기존 Role과 다른 **authority, write boundary, completion responsibility** 중 최소 하나가 필요함을 설명해야 한다.

## Scope discipline

MVP에 기본 포함:

- local single-process Controller
- OpenCode runtime adapter
- Agent/Skill/Project/Harness registry
- Git worktree
- structured delegation
- PolicyGate
- Reviewer/Verifier
- functional CheckRunner
- project-defined Architecture Fitness check contract
- artifact/event/state

명확한 필요가 생기기 전 제외:

- workflow DSL
- distributed queue
- Redis/message broker
- model gateway/router
- public marketplace
- runtime external skill auto-install
- full RAG platform
- automatic merge
- GUI
- generic semantic skill router

## External Skill rule

외부 Skill/Agent를 참고할 때:

1. upstream URL/path/revision/license 기록
2. script/network/filesystem/package-install 여부 확인
3. 역할/기존 Skill과 overlap 확인
4. 필요한 원칙만 최소 이식
5. 승인된 local canonical package로 고정
6. target harness artifact는 adapter로 생성

자세한 규칙은 [10-skill-intake-portability-and-evaluation.md](docs/design/10-skill-intake-portability-and-evaluation.md)를 따른다.

## Documentation rule

모듈 책임, 상태 authority, profile merge rule, Role contract, security boundary, architecture fitness rule이 바뀌면 관련 design 문서를 같은 변경에서 갱신한다.

새 개념을 추가하기 전에 기존 `Role / Project / Domain / Skill / Harness / Task Contract` 중 하나로 표현할 수 없는지 먼저 확인한다.

새 반복 리뷰 규칙을 발견하면 문서 지시로 끝내지 말고 deterministic check로 승격 가능한지 검토한다.

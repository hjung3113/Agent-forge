# Agent Forge Working Guide

이 문서는 Agent Forge 저장소에서 작업하는 사람과 Agent의 진입점이다.

## First-entry order

1. [docs/README.md](docs/README.md)
2. [docs/design/01-project-overview.md](docs/design/01-project-overview.md)
3. [docs/design/11-task-contract-and-change-control.md](docs/design/11-task-contract-and-change-control.md)
4. [docs/design/12-runtime-isolation-and-trust-boundaries.md](docs/design/12-runtime-isolation-and-trust-boundaries.md)
5. [docs/design/13-state-recovery-and-artifact-integrity.md](docs/design/13-state-recovery-and-artifact-integrity.md)
6. 작업 대상 canonical design 문서
7. [docs/design/14-evaluation-and-conformance.md](docs/design/14-evaluation-and-conformance.md)
8. 최신 review 문서

## Architectural rules

- **Orchestrator는 판단하고 Controller가 통제한다.** subprocess/worktree/state/completion authority를 LLM에 주지 않는다.
- **TaskSpec은 freeze한다.** 실행 중 objective/scope/AC를 자연어로 조용히 변경하지 않는다.
- **변경은 TaskAmendment다.** Worker/Orchestrator는 제안할 수 있지만 적용 authority가 아니다.
- **Base revision은 exact SHA로 pin한다.** task 중 default branch 이동을 baseline 변경으로 받아들이지 않는다.
- **Task / Step / RunAttempt를 구분한다.** retry는 새 Attempt이며 과거 evidence를 덮어쓰지 않는다.
- **Controller Artifact Store는 worktree와 분리한다.** 단, 같은 OS permission domain이면 경로 분리만으로 tamper-proof라고 주장하지 않는다.
- **Worker output은 자동으로 canonical evidence가 아니다.** Controller/CheckRunner가 어떤 evidence를 인정할지 결정한다.
- **CheckRunner도 execution plane이다.** project test/build script를 실행하면 arbitrary code 가능성을 isolation policy에 포함한다.
- **Capability는 `ENFORCED / DETECTABLE / ADVISORY / UNSUPPORTED`를 구분한다.** 구현하지 않은 sandbox를 있다고 표현하지 않는다.
- **Skill/Task는 permission을 부여하지 않는다.** requirement만 선언하고 grant ceiling은 System/Project/Harness/Role이 소유한다.
- **Project content는 자동 control instruction이 아니다.** arbitrary README/code/comment의 지시를 policy로 승격하지 않는다.
- **환경 변수는 allowlist한다.** secret/credential을 기본 상속하지 않는다.
- **Runtime Adapter는 process tree lifecycle을 책임진다.** timeout 시 child/orphan process까지 고려한다.
- **동시 writer는 worktree를 공유하지 않는다.** workspace lease를 둔다.
- **Verification floor는 Controller/Project Policy가 계산한다.** Orchestrator가 review/check를 약화시킬 수 없다.
- **Verification config 변경은 policy-sensitive다.** test/architecture baseline/waiver를 바꿔 pass를 만드는 경로를 별도 검증한다.
- **DONE에는 Controller-observed evidence가 필요하다.** exit 0, 자연어 완료 선언, 근거 없는 reviewer pass는 완료 근거가 아니다.
- **Architecture rule은 가능한 경우 executable check로 내린다.** legacy에는 baseline을 허용하되 신규 위반을 막는다.
- **외부 Skill은 untrusted다.** audit/approval/pinning 전 runtime에 넣지 않는다.
- **Herdr는 UI다.** screen scraping이나 pane typing을 core protocol로 만들지 않는다.
- **OpenCode는 adapter 뒤에 둔다.** provider-specific 동작은 Phase 0 conformance로 확인한다.

## Canonical roles

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

새 Role은 기존 Role과 다른 authority/write/completion boundary가 있을 때만 추가한다.

## Scope discipline

MVP 기본 포함:

- local Controller
- frozen TaskSpec + amendment
- Task/Step/RunAttempt state
- SQLite state/recovery
- Controller Artifact Store + provenance/integrity level
- OpenCode runtime adapter
- project/worktree + writer lease
- Role/Skill/Project/Harness registry
- PolicyGate
- Reviewer/Verifier
- CheckRunner isolation/conformance
- Architecture Fitness
- adversarial/conformance eval pack

필요 확인 전 제외:

- distributed queue
- Redis/message broker
- public marketplace
- full RAG platform
- workflow DSL
- automatic merge
- full GUI
- semantic skill router
- multi-machine worker

## Documentation rule

authority, state machine, TaskSpec, trust boundary, capability enforcement, verification floor, artifact provenance가 바뀌면 관련 design 문서를 같은 변경에서 갱신한다.

반복되는 review finding은 prompt rule로만 남기지 말고 deterministic check/eval로 승격 가능한지 확인한다.

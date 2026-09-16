# Architecture Grilling Session — 2026-09-16

## 1. 목적

기존 14개 design 문서 + roadmap + review 3편 + ForgeRoom reuse + agents 문서를 대상으로, 두 개의 독립 에이전트가 서로 마주 앉아 심문 형식으로 재검증했다.

- **Griller**: Grok 4.6 (reasoning effort: high) — 질문자. 매 라운드 design tree의 frontier를 계산해 질문 + 권장 답을 제시.
- **Answerer**: Codex (model `gpt-6-astra`, reasoning effort: medium) — 답변자. 문서를 인용해 동의/반박하고, 문서에 없는 제안은 명시적으로 "proposed, not existing spec"이라 표시.
- 오케스트레이터(Claude Code)가 herdr로 두 pane을 중계했다.

7 라운드, 총 60+ 문항. 최소 5라운드 요구 이후에는 griller가 스스로 frontier가 비었다고 판단할 때까지 계속했다.

## 2. 읽는 법

`(write-down)`로 표시된 항목은 **아직 문서에 없는 신규 제안**이다. Answerer가 여러 차례 확인시킨 원칙: 인터뷰에서 합의된 내용이라도 실제 design 문서를 조용히 고쳐 쓰지 않는다 — 구현 전에 명시적으로 기록해야 하는 구멍이라는 뜻이다.

## 3. 합의된 내용 (영역별)

### Product, index, SoT

**docs/README.md**
- Canonical responsibility table과 ENFORCED/DETECTABLE/ADVISORY/UNSUPPORTED vocabulary는 그대로 유지.
- 현재 source-of-truth는 design spec. ADR이 생기면 ADR-first가 steady-state rule.
- (write-down) consequential한 architectural choice를 채택하거나 이미 나온 결정을 supersede할 때 ADR을 기록한다. Phase 0 수치는 capability evidence로 남기고, 그것이 design claim을 바꿀 때만 design/ADR로 승격한다.

**01-project-overview.md**
- Agent Forge는 Controller가 소유하는 constrained coding-agent execution control plane이다. OpenCode는 environmental constraint이지 product 자체가 아니다. 언젠가 OpenCode가 contract/state/execution/completion guarantee를 자체 제공하면 별도 Controller가 불필요해질 수 있다는 것은 판단이지 재플랫폼 전략이 아니다.
- Quality는 model diversity가 아니라 구조(격리, contract, Controller-observed evidence)에서 나온다.
- first-slice ≠ MVP. First vertical slice = operator-as-orchestrator + Controller execute/verify. 문서화된 MVP는 여전히 Phase 0–6 + minimal Phase 7. 01의 Orchestrator success story는 product vision이지 Phase 1 gate가 아니다.

**docs/roadmap/mvp-roadmap.md**
- First slice에 반드시 포함: Phase 0 evidence, Phase 0.5 freeze/state/recovery/lease/provenance, 등록된 implementer 1개, workspace + Controller-observed source state, per-execution admission + envelope controls, 동일 admission 하의 CheckRunner, AC/check binding + completion gate, Herdr 없이 operator inspect/cancel.
- "OpenCode가 worktree에서 돌았다"가 아니다. Safety Profile은 제안된 vehicle이고, 그 밑의 policy/capability 결정이 문서가 실제로 요구하는 것.
- Headless/cwd/output/exit UNSUPPORTED는 여전히 product blocker. Orphan/child cleanup을 주장할 수 없으면 cancel/DONE을 주장하지 않는다. 그 외 gap은 capability report + admission으로 degrade, 정해진 decision sequence를 따른다. 승인되지 않은 model gateway로 조용히 갈아타지 않는다.

### Control plane & TCB

**02-system-architecture.md**
- Orchestrator = semantic proposal. Controller = executable state/policy/evidence/completion.
- TCB(T0)는 SQLite writer보다 넓다: harness compiler, schema, grant resolution, admission, transitions, evidence acceptance, completion logic, 그리고 SQLite/관련 OS·runtime dependency까지 포함. Pinning은 identity를 증명하지 harmlessness를 증명하지 않는다. YAML registry/Project Profile/pinned Skill은 T1 input. project_command 문자열은 T4 execution의 이름일 뿐.
- 로컬 단일 logical Controller. OpenCode는 Runtime Adapter 뒤. Herdr는 protocol이 아니다.
- (write-down) state DB당 mutating authority는 하나. inspect는 concurrent read-only 가능.

**04-orchestration-runtime.md**
- Spawn은 등록된 agent_id(explicit 또는 virtual) → resolve → snapshot. 임의의 role/project/skill tuple 요청은 나중 확장(agent disablement를 우회해서는 안 됨).
- Recursive worker는 추천만 한다 — spawn하지 않는다. Budget은 policy.
- Rolling summary는 context일 뿐 authority가 아니다. State/hash가 이긴다. LLM 요약은 worker provenance를 유지하고, template은 Controller가 생산한다. Summary는 버전 교체 가능하지만 canonical history를 그에 맞춰 고쳐쓰지 않는다.
- DelegateRequest validation은 TaskSpec hash/registry/budget/VerificationPlan을 쓴다 — summary를 쓰지 않는다.
- First E2E: CLI/operator가 DelegateRequest를 채운다. Orchestrator Worker는 Phase 6.

**07-agent-registry-and-extension.md**
- agent yaml은 optional alias. Execution contract는 resolved profile snapshot/hash. Catalog는 derived. Virtual project-expert/domain-expert는 resolve rule로 유지.
- Permission override는 ceiling을 좁히기만 한다.

**08-role-contracts-and-agent-environment.md**
- (write-down, rule 변경) Role은 distinct하고 stable한 책임, 별도로 평가 가능한 I/O나 evidence contract, 또는 다른 authority가 있어야 추가한다 — 기존 Role의 Skill/Domain으로 표현할 수 없을 때만.
- Debugger는 discriminated contract가 있어야만 존속: `reproduced_and_fixed | not_reproduced | blocked | diagnosis_incomplete`. Success-shaped field는 success일 때만 필수. 실패한 outcome이 write grant를 소급 취소하지 않는다(재현 테스트/instrumentation은 남을 수 있음). Controller는 identity/provenance/status/scope를 검증하지, 진짜 causality나 minimality를 검증하지 않는다.
- Verifier: mechanical AC ledger는 Controller. LLM Verifier는 semantic mapping에만. Manual AC는 지정된 human/external evidence가 필요 — LLM이 그 authority를 대신하지 않는다.
- First-slice는 implementer만 배포. Debugger는 contract가 생길 때, prompt alias로가 아니다.

### Contract, policy, completion

**11-task-contract-and-change-control.md**
- FROZEN TaskSpec + hash. Amendment만이 유일한 contract 변경 경로. Retry는 AC를 약화시킬 수 없다. Base는 정확한 SHA.
- Operator는 이미 제출한 필드가 아닌 한 normalized spec을 accept해야 한다. Pre-freeze Worker execution은 MVP 밖. Controller는 LLM 없이 SHA를 resolve/path를 validate/승인된 project context를 보여줄 수 있다. 11의 "Intake/Grounding" 표현은 operator 작업 + Controller inspection으로 한정한다.
- Scope는 authorization(glob)이지 모든 파일의 예측이 아니다. In-scope discovery는 amendment가 아니다; out-of-scope edit은 amendment다. Attempt grant를 좁히는 것 ≠ TaskSpec scope를 다시 쓰는 것; grant 부족은 block이지 spec을 다시 쓰지 않는다. 하나의 denied path만으로 AC-impossibility를 추정하지 않는다.
- (write-down) Freeze 시점의 TaskPolicySnapshot + append-only TaskPolicyDecision(`tighten_checks` | `tighten_grants` | `revoke_execution`). Attempt는 시작할 때의 policy hash를 유지한다; DONE은 effective policy를 쓴다. Loosening은 절대 silent하지 않다. `tighten_grants`는 이미 admit된 프로세스를 죽이지 않는다; 실행 계속이 금지된 경우가 `revoke_execution`이다. Extra check는 이후 Attempt가 없어도 VerificationPlan을 개정한다.

**13-state-recovery-and-artifact-integrity.md**
- Task / Step / RunAttempt. Retry = 새 Attempt. Authoritative pointer ≠ history. Worker가 쓴 파일이 자동으로 evidence가 되지 않는다.
- (write-down) 두 개의 operation: `ExecutionRevocation`(cancel 전에 기록; 동반하는 invalidation이 기록되지 않는 한 이전 completion을 자동으로 void하지 않음)과 `ResultInvalidation`(hash는 보존; authoritative_attempt_id를 atomically clear하고 Step을 ineligible로 만듦).
- (write-down) DONE 이후: REOPENED state를 추가하지 않는다. Sidecar `TaskCompletionValidity {valid|voided}`. Consumer는 `Task.state == DONE`을 유효한 완료로 취급하면 안 된다(`DONE(voided)`는 명시적이어야 함). Terminal DONE Task에는 새 Attempt가 없다; 구제는 새 Task. Authorizer = operator 또는 승인된 deterministic rule; LLM은 제안만 한다.
- (write-down) Lineage: Controller가 `consumes_artifacts` + `source_snapshot_ref` + input manifest를 구성한다. Worker가 자기 provenance를 권위 있게 선언하지 않는다. 빈 `consumes_artifacts`는 "이전 artifact 없음"이지 "input 없음"이 아니다. Snapshot identity는 관련 tracked/untracked content + metadata를 커버해야 한다. 실제 consumption을 따라 transitive `needs_revalidation`. 광범위한 "모두 재실행"은 accepted evidence나 coverage가 불확실할 때만 — 무관한 rejected artifact가 존재한다고 전체를 무효화하지 않는다. 새 Reviewer Attempt는 semantic evidence를 제공할 수 있다; 대화식 재승인은 안 된다.
- (write-down) Mutating Controller: DB 옆의 exclusive host-local lock. SQLite는 process/workspace side effect를 직렬화하지 않는다. Takeover는 이전 authority의 exclusion을 요구하지, heartbeat 만료만으로는 안 된다(A가 멈춰있을 수 있음). Kill은 Phase 0이 지지하는 가장 강한 OS process identity/containment로만; env cookie는 correlation일 뿐 kill-authorization이 아니다(same-UID가 복사/제거 가능, descendant까지 살아남는다는 보장 없음, group membership을 증명하지 않음). Identity가 모호하면 신호를 보내지 않는다; Attempt는 LOST; lease는 보존. Adopt-and-validate는 output/exit을 여전히 관측할 수 있을 때만; 아니면 cancel/LOST + 새 Attempt. Operator cancel = `CancellationRequest(reason=operator)`, 같은 machinery, 다른 authorization. FS가 lock을 못 걸면 mutating command는 fail closed.

**06-safety-verification-and-observability.md**
- Completion은 conjunction이다: required checks + valid blocking-AC evidence + no unresolved blocking findings + policy/scope + artifact/state integrity. Exit 0, "다 했다", 근거 없는 reviewer pass는 DONE이 아니다.
- Deterministic check는 semantic AC에 필요하지만 충분하지 않다. unknown ≠ pass. Manual blocking AC는 자동 DONE을 막는다.
- Command/Path/Spawn/Workflow는 Controller의 책임이지 4개의 별도 서비스가 아니다. WorkflowGate = hash + deps + VerificationPlan + completion.
- Herdr는 Controller cancel/inspect의 선택적 observer/client다. CLI가 operator path. Pane text는 state가 아니다.
- DONE deliverable = Controller가 묶은 worktree/branch + 검증된 결과를 복구하는 데 필요한 모든 것(validated state의 일부였다면 untracked도 포함). Clean commit은 필수가 아니다. Worker의 remote-write는 기본적으로 거부; 향후 publisher는 명시적 grant + 실제 enforcement가 필요.

### Isolation, admission, evidence

**12-runtime-isolation-and-trust-boundaries.md**
- Adversary: confused/goal-misaligned agent + untrusted content. Project/test/skill script를 Controller UID 아래서 악의적으로 실행하면 그 실행에 한해 사실상 hostile same-UID다 — 없는 containment를 있다고 하지 않는다. Worktree ≠ OS sandbox; path hiding ≠ protection.
- 4단계 + backend_support vs effective_enforcement. DETECTABLE은 이름 붙은 observer가 필요하고 그 observer가 실제로 증명하는 property만 증명한다(persistent hash ≠ 모든 intervening mutation; git status ≠ 모든 ignored write; process-group inspect ≠ escape한 descendant 없음).
- (write-down) Claim record: 버전 있는 property_id(예: `filesystem.persistent_worktree_changes.detect` ≠ `filesystem.outside_worktree_writes.prevent`) + threat-model ref + 해당 시 actor/persistence + level + mechanism ref + conformance test. Free-text coverage가 매칭을 조용히 좁히면 안 된다. 요구된 dimension이 빠지면 매칭 실패(fail closed). Admission은 요구사항에 맞는 claim을 선택한다; 좁은 pass가 넓은 요구를 만족시키지 않는다. Git/canonical-hash는 일반적인 outside-worktree write를 detect하지 못한다.
- Skill/Task는 절대 grant하지 않는다. required vs ceiling의 intersection; required가 UNSUPPORTED면 BLOCKED.
- Instruction class는 여전히 3개: CONTROL / TRUSTED_PROJECT_INSTRUCTION / REFERENCE_CONTENT. 선택되어 승인된 Skill procedure는 resolved-harness component(CONTROL bundle)이지만 여전히 hard policy/TaskSpec/Role 아래에 있다. 4번째 class는 없다. Compose-time phrase filter는 불완전하다 — static/schema + semantic intake + instruction compliance와 독립적인 runtime authority가 필요하다. Delegation 추천 ≠ spawn.
- CheckRunner는 T4다. Evidence authority ≠ execution safety.

**Admission / Safety Profile** — (write-down, System/Project policy의 fragment로, 새로운 precedence layer가 아니다)
- 이름 붙은 profile + conjunctive required claim + 명시적 `acceptable_levels` set(숫자 lattice 아님). Project는 requirement를 추가하거나 acceptable set을 좁힐 수만 있다 — System requirement를 더 쉬운 것으로 바꿀 수 없다. Profile 선택이 downgrade가 될 수 없다.
- Admission은 execution 단위(Worker와 CheckRunner를 각각, runner trust class 포함)다. "시작을 허용/거부하는 enforced 결정"이지 "허용된 모든 restriction이 ENFORCED라는 증명"이 아니다.
- 독립적인 check가 안전하지 않은 command를 안전하게 만들지 않는다. 필요한 containment가 없으면 실행하지 않는다.
- 제안된 기본 MVP profile: `accidental-persistent-trusted-internal`(제안, 문서화 필요). 더 강한 profile은 Project가 opt-in하는 named row.

**05-project-workspace-and-context.md**
- Worktree = source isolation. Clone-per-task는 공유 gitdir state를 줄이지만 same-UID security boundary는 아니다. Gitdir-link tamper: canonical repo config/hooks/refs의 pre/post hash는 persistent-accidental observer; hostile restore는 다른 claim이다.
- Writer lease: worktree당 writer 하나. Reviewer concurrency는 write가 실제로 ENFORCED일 때만.
- Worktree 안의 staging은 non-canonical; Controller가 input을 hash한다. Canonical copy는 Attempt store에 산다(authority ≠ 배타적 물리 위치).
- Dirty worktree도 evidence가 정확한 최종 내용에 묶여 있다면 validated DONE 결과가 될 수 있다.
- Learned Memory: (write-down) MVP writer = operator만. "Controller가 나중 승인을 위해 draft한다"도 auto-extraction이라 MVP에서 제외. REFERENCE_CONTENT + ADVISORY; grant/policy override 없음. TaskSpec에 이름 붙었지만 없거나 quarantine된 entry는 그 requirement를 blocked/manual로 만들지, 승격 backdoor가 아니다.

**14-evaluation-and-conformance.md**
- Verification floor는 Orchestrator가 낮출 수 없다. Change class: Controller-mandatory한 path/manifest detection + nonempty default floor; LLM은 제안만, Controller가 검증, detected class를 제거할 수 없다. Ordinary(검사됨, 특별 rule 없음) vs unknown(검사/coverage 실패). Semantic uncertainty(state는 신뢰 가능하지만 class가 불명확)는 block 대신 review를 추가할 수 있다. Project는 예를 들어 public API에 대해 conservative path/structural rule을 강제할 수 있다 — 불확실한 것은 불확실하다고 보고해야 한다.
- Inspection limitation은 하나의 버킷이 아니다: (a) required observer 실패 → 그 evidence를 보류 + completion block; (b) policy가 허용한 observer gap → 자동으로 unknown이 아님; (c) semantic uncertainty → 추가 review 가능. 실행 후 발견된 실패는 그 실행을 되돌릴 수 없다 — acceptance를 보류할 뿐이다. CheckRunner는 blocking AC를 만족시키지 않고도 diagnostic으로 실행될 수 있다; 그래도 자체 admission은 필요하다. Amendment는 System-required observer를 우회할 수 없다.
- Deterministic conformance는 각 invariant의 첫 구현부터 Controller를 gate한다(이미 14 §14/§17; Phase 5.5는 미루라는 licence가 아니다). 실행되지 않은 관련 검증은 조용히 건너뛰지 말고 보고해야 한다.

### Composition, fitness, skills

**03-agent-composition-and-harness.md**
- Grant = intersection; requirement = union; restriction = most restrictive; 명시적 모순은 fail-fast.
- 세 객체: `TaskSpec.scope`(intent), Attempt grant(ceiling), `VerificationPlan`(floor). Controller는 amendment 없이 grant/check를 조일 수 있다; expansion은 절대 "도움"이 아니다.
- 생성된 harness는 staging에 머문다; target `.opencode/`를 overlay하지 않는다.

**09-architecture-governance-and-fitness.md**
- Fitness는 Project Profile + CheckRunner + gate이지 새 control plane이 아니다. Runner class는 실제 구현을 따른다(`project_command` / `external_tool` / `controller_builtin`).
- Baseline/waiver/harness-test diff는 policy-sensitive다. (write-down) Authorization artifact는 rule+변경, content hash, scope/reason/owner/expiry, decision source를 묶는다. 이미 준 operator authorization이면 충분할 수 있다(추가 클릭 불필요). Expanded baseline에서 test가 통과한다고 그 확장이 승인됐다는 증거는 아니다. 만료된 waiver는 exception을 승인할 수 없다. 변경이 AC/scope/constraint 의미를 바꾸면 TaskSpec이 "waiver 금지"라고 안 했어도 amendment다.
- 약화된 config를 소비하는 수정된 analyzer/tool은 independent하지 않다. Fitness command가 없으면 policy가 허용할 때만 semantic review, universal Reviewer가 아니다.

**10-skill-intake-portability-and-evaluation.md**
- 외부 Skill은 stage/audit/eval/approve/pin 전까지 untrusted다. Pin ≠ execute. Script 실행은 PolicyGate + admission이 필요; interception이 UNSUPPORTED면 per-command ENFORCED는 거짓이고 enclosing admission이 적용된다(또는 profile이 per-command ENFORCED를 요구하면 BLOCK).
- (write-down) Stage 5 = 일반 Task + Controller가 검증한 `evaluation_only` candidate exception(hash-pinned, fixture-bounded, 일반적으로 승인되지 않음). Canonical eval record는 보존. Eval 성공은 approval provenance(또는 그 eval에 대한 AC)이지, 이후 product 변경이 통과한다는 증거가 아니다. 10 §6의 표는 유지: script/tool Skill은 Behavioral 필수; runtime tool이 필요하면 Runtime 필수. Prompt-only는 Static + Stage 6.

### Reviews, ForgeRoom, this-repo agents

**docs/review/architecture-review.md**
- 큰 orchestration framework 전에 Phase 0. Orchestrator가 control plane이 되면 안 된다. Composition layer는 bounded 상태 유지. MVP를 sandbox라 부르지 않는다. Scope expansion이 현실적인 failure mode. Core DONE ≠ push/PR; GitPublisher는 나중. Herdr 실패가 core state를 바꾸지 않는다.

**docs/review/architecture-agent-environment-review.md**
- Canonical 8개 Role; language/framework는 Skill/Domain으로. Fitness는 executable check로. Skill supply chain. Canonical IR → adapter → generated native. `triggers` + `non_triggers`. codebase-grounding은 core component가 아니라 Skill; project-expert grounding은 post-freeze.

**docs/review/architecture-adversarial-review-round2.md**
- Same-UID store에 대한 정직한 caveat(Finding O). CheckRunner는 임의 코드(Finding P). 남은 empirical 질문: OpenCode capability(R1), OS isolation 가용성(R2), same-model correlation(R3), parallel merge(R4), TCB bug(R5) — 마지막은 희망이 아니라 14로 gate한다.

**docs/reference/forgeroom-reuse-analysis.md**
- Seam을 재사용한다(adapter, harness, hard vs soft permission, Project≠Harness, PolicyGate category, worktree pattern, file artifact). Discord/Mastra/OpenClaw/ForgeMap을 product로 들여오지 않는다. ApprovalGate는 개념적으로만; 코드 재사용은 검토가 필요하다. Mapping 대상은 Agent Forge의 기존 책임(Controller, Registry, Adapter, Workspace, CheckRunner, artifact/state) 전체이지 3-box slogan이 아니다.

**docs/agents/issue-tracker.md, docs/agents/triage-labels.md, docs/agents/domain.md**
- GitHub issue + `gh`가 이 저장소를 만드는 human tracker다. PR은 request surface가 아니다. `ready-for-agent`는 "agent 작업에 충분히 specified"이지 immutable freeze가 아니다; human-only labeling은 문서화되어 있지 않다. Agent Forge가 존재한 뒤에는 issue가 TaskSpec의 source request가 될 수 있다; 이후 issue 수정이 조용히 amend하지 않는다; Controller DONE 자체가 issue를 닫지 않는다. 하나의 issue가 여러 task에 대응할 수 있다. Wayfinder map은 planning이고, 거기서 나온 승인된 implementation ticket은 TaskSpec이 될 수 있다.
- 아직 `CONTEXT.md` / `docs/adr/`가 없으므로 domain.md의 "조용히 진행" 규칙이 적용된다. Glossary가 생기면 쓰고, ADR이 생기면 충돌을 표시한다.

## 4. 명시적으로 미룬 것 / Post-MVP

| Item | Status |
| --- | --- |
| Orchestrator Worker, user-briefing, IntakeSession, cross-task status query | First-slice 이후; briefing은 MVP success에 없음 |
| 직접 role/project/skill tuple spawn | 확장 기능; agent disablement를 우회하면 안 됨 |
| Pre-freeze Worker grounding | MVP 밖; 미래 entity는 IntakeSession이 아님 |
| Learned Memory 자동 추출 / Controller draft lesson | MVP 밖 |
| Skill marketplace, semantic skill router | 제외 |
| Parallel writer, auto-merge, GitPublisher/PR | Phase 8 / later adapter |
| Herdr를 필수 경로로 | 절대 아님; 선택적 UI |
| Distributed queue, Redis, multi-machine, workflow DSL, full GUI | 제외 |
| OS/container ENFORCED sandbox를 기본값으로 | policy/task가 요구하고 backend가 있을 때만 |
| Hostile same-UID tamper-proof store | 주장하지 않음; ENFORCED isolation이거나 거부 |
| Universal ENFORCED network / universal untrusted-repo row | 문서화되지 않은 universal이 아님; named profile로 |
| Same-model independent intelligence | 인정된 한계; evidence diversity로 대응, 두 번째 모델이 아님 |
| Exhaustive TCB binary inventory | 원칙은 합의됨; inventory는 구현 시점에 |
| 정확한 kill primitive (cgroup / job object / …) | Phase 0가 가장 강한 지원되는 OS identity/containment를 선택; cookie는 그 mechanism이 아님 |
| CommandGate의 신뢰할 수 있는 interception | Phase 0; 없으면 ENFORCED 아님, DETECTABLE도 아닐 수 있음 |
| OpenCode instruction-precedence vs compiled harness | Phase 0 warn/fail-fast |
| IT-managed read-only policy를 별도 principal로 | Deployment variant; runtime은 여전히 완화할 수 없음 |

## 5. Write-down queue (구현 전 반드시 문서화해야 하는 구멍)

구현 순서 기준 우선순위:

1. Phase 0 capability report schema (버전 있는 property_id, threat-model, mechanism, tests) + ordinary vs unknown inspection
2. Exclusive Controller lock + kill/reconcile용 OS process identity + CancellationRequest
3. TaskPolicySnapshot / TaskPolicyDecision; ExecutionRevocation vs ResultInvalidation; DONE validity sidecar
4. Controller가 구성하는 consumes_artifacts + snapshot coverage
5. Admission을 System/Project policy fragment로 (acceptable_levels, per-execution, CheckRunner 포함)
6. Operator freeze-acceptance; 11의 intake wording
7. Role-addition rule; 그 Role이 나올 때 discriminated Debugger
8. Baseline/waiver authorization artifact
9. Skill evaluation_only candidate path
10. Roadmap에 first-slice vs MVP 명명

## 6. 결론

작은 deterministic TCB가 contract·process·evidence·completion을 소유하고, LLM은 제안만 하며, enforcement에 대한 정직함이 나중에 붙이는 disclaimer가 아니라 architecture 자체의 일부라는 것 — 이것이 7 라운드를 거쳐 griller와 answerer가 도달한 공유된 이해다.

세션 원문(질문/답변 전문)은 이 리뷰를 생성한 오케스트레이션 세션에 보존되어 있으며, 필요 시 재구성 가능하다.

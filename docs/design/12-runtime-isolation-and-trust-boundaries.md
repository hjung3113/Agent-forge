# 12. Runtime Isolation & Trust Boundaries

## 1. 목적

Agent Forge는 prompt 지시를 security boundary로 취급하지 않는다.

Git worktree는 source-change isolation에는 유용하지만 OS sandbox가 아니다.

이 문서는 실제 trust zone, threat model, capability enforcement 수준을 정의한다.

## 2. Threat model

### MVP가 우선 방어하는 것

- LLM/Agent의 실수
- scope creep
- 잘못된 shell/tool 사용
- false completion
- 정책상 금지된 source 변경
- accidental secret exposure

### 자동 보장하지 않는 것

Worker가 Controller와 **동일 OS user 권한으로 unrestricted arbitrary code**를 실행할 수 있다면, 단순히 경로를 분리하는 것만으로 hostile worker에 대한 완전한 host/storage tamper resistance를 보장할 수 없다.

그 수준을 요구할 때는:

- 별도 OS identity
- container/namespace/sandbox
- filesystem ACL
- network isolation

등 실제 ENFORCED boundary가 필요하다.

따라서 security claim은 threat model과 enforcement level을 함께 기록한다.

## 3. Trust Zones

```text
T0 Controller process / canonical state authority
T1 approved Agent Forge config, pinned Skills, Project Profiles
T2 project repository content / project-local instruction
T3 OpenCode/Worker and child processes
T4 project build/test/check subprocesses
T5 external network/systems
```

T4가 `CheckRunner`에 의해 시작됐다고 해서 안전한 코드는 아니다. project test/build script는 arbitrary code를 포함할 수 있다.

## 4. Capability enforcement level

### ENFORCED

기술적으로 차단/허용을 통제.

### DETECTABLE

완전 차단은 못하지만 Controller가 신뢰 가능한 방식으로 위반을 탐지.

### ADVISORY

prompt/runtime instruction 수준.

### UNSUPPORTED

backend가 capability 자체를 제공하지 않음. degrade가 아니라 부재다. 대체 수단(adapter 외부 구현, 기능 포기, 명시적 중단)을 별도로 결정해야 한다.

강한 이름을 쓰더라도 enforcement level을 별도로 저장한다. canonical enforcement level은 이 네 단계(`ENFORCED / DETECTABLE / ADVISORY / UNSUPPORTED`)이며 다른 문서는 이 절을 참조한다.

### backend_support와 effective_enforcement

capability report는 두 값을 분리해 기록한다.

- `backend_support`: runtime/adapter가 그 capability를 원래 제공하는지에 대한 raw 값.
- `effective_enforcement`: Controller-side 보완(policy gate, post-run diff/status 조회, env allowlist 등)을 적용한 뒤 실제로 성립하는 수준.

`backend_support = UNSUPPORTED`라도 Controller-side 수단으로 `DETECTABLE`/`ADVISORY` 수준을 만들 수 있으면 `effective_enforcement`는 그 값으로 올라간다. headless 실행처럼 Controller-side로 보완할 수단이 없는 capability는 `backend_support`와 `effective_enforcement`가 항상 같다. §5의 BLOCKED 판정은 `effective_enforcement` 기준이다.

## 5. Required vs Granted

Skill/Task는 permission을 부여하지 않는다.

```text
effective_grant
 = system ceiling
 ∩ project ceiling
 ∩ harness ceiling
 ∩ role ceiling
```

```text
required <= effective_grant
```

가 아니면 BLOCKED다. `required` capability의 `effective_enforcement`가 `UNSUPPORTED`인 경우도 동일하게 BLOCKED다. `backend_support`가 `UNSUPPORTED`라도 Controller-side 보완으로 `effective_enforcement`가 올라갔다면 BLOCKED가 아니다.

## 6. Instruction trust

```text
CONTROL
  hard policy, frozen TaskSpec, resolved harness

TRUSTED_PROJECT_INSTRUCTION
  Project Profile이 명시적으로 승인한 project instruction

REFERENCE_CONTENT
  source/docs/comments/logs/arbitrary markdown
```

REFERENCE_CONTENT의 "ignore policy"는 control instruction이 아니다.

Provider가 project-local instruction을 자동 탐색하면 실제 precedence를 Phase 0에서 검증한다.

## 7. Filesystem

최소 고려:

- canonical path resolution
- cwd/worktree
- symlink traversal
- denied/policy-sensitive paths
- HOME/TMP/config path
- environment allowlist
- pre/post source diff

`diff clean`은 workspace 밖 side effect가 없었다는 증거가 아니다.

### Controller Artifact Store

worktree 밖 store는 **logical authority 분리**에는 필요하다.

하지만 Worker가 동일 OS 권한으로 해당 경로를 수정할 수 있다면 physical tamper resistance는 ENFORCED가 아니다.

따라서:

- store path를 Worker input으로 불필요하게 노출하지 않음
- Worker가 제출한 파일을 곧바로 canonical evidence로 인정하지 않음
- Controller가 pipe/process result/diff를 직접 수집
- storage integrity level 기록
- stronger threat model이면 별도 OS/sandbox boundary 요구

를 적용한다.

경로 은닉만을 security mechanism으로 사용하지 않는다.

## 8. Environment / secrets

- host env 전체 상속 금지
- allowlist
- credentials 기본 미주입
- secret prompt/log 복사 금지
- redaction/retention
- SSH agent/Git credential helper 접근 가능성 테스트

## 9. Network

실제 차단이 없으면 `network_disabled=ENFORCED`라고 표현하지 않는다.

필요 시 firewall/container/backend를 추가한다.

## 10. Process containment

Runtime Adapter와 CheckRunner execution wrapper는:

- process group/job
- timeout
- terminate + kill escalation
- child/orphan cleanup
- output size budget
- temp cleanup

을 고려한다.

## 11. CheckRunner도 isolation 대상

`dotnet test`, `pytest`, build script는 deterministic **관찰 방식**일 수 있지만 실행 자체는 arbitrary code다.

따라서 두 질문을 분리한다.

```text
Evidence question:
  Controller가 command/exit/output을 직접 관찰했는가?

Safety question:
  그 command가 어떤 filesystem/network/process 권한으로 실행됐는가?
```

고위험/외부 repository에서는 controller-owned static check, fixture, sandboxed runner가 필요할 수 있다.

## 12. Command Gate 한계

문자열 denylist만으로 arbitrary command safety를 보장하지 않는다.

우선순위:

1. tool/runner allowlist
2. process/filesystem/network boundary
3. command class policy
4. textual denylist
5. post-run detection

## 13. Phase 0 adversarial spike

최소:

```text
outside-worktree write/read
Controller store path access/tamper attempt
canonical repo config/hook/ref tamper via worktree gitdir-link
symlink traversal
HOME/SSH/Git credential access
env leakage
network request
child/background process
timeout orphan cleanup
huge output
project instruction precedence
nested shell/interpreter bypass
project test script host side effect
concurrent runtime config collision
```

각 결과를:

```text
ENFORCED | DETECTABLE | ADVISORY | UNSUPPORTED
```

로 기록한다.

## 14. Sandbox claim

OS/container sandbox가 없으면 일반적으로 `sandboxed`라고 부르지 않는다.

정확한 표현:

```text
worktree-isolated source workspace
+ host-side policy/detection
+ capability-specific enforcement report
```

## 15. 설계 불변조건

1. threat model과 capability claim을 분리하지 않는다.
2. Task/Skill은 grant authority가 아니다.
3. worktree 밖 store도 같은 OS permission domain이면 tamper-proof가 아니다.
4. project content를 control instruction으로 자동 승격하지 않는다.
5. CheckRunner도 untrusted code execution 가능성을 가진다.
6. environment는 allowlist한다.
7. Runtime Adapter는 child process까지 lifecycle을 소유한다.
8. enforcement 없는 network/fs restriction을 ENFORCED라고 표시하지 않는다.

# 05. Project Workspace & Context

## 1. 목적

특정 Project Agent는 target repository의 격리 workspace에서 실행되어야 한다.

Git worktree는 source 변경을 분리하는 수단이며, trusted runtime state/evidence store가 아니다.

Runtime trust boundary는 [12-runtime-isolation-and-trust-boundaries.md](12-runtime-isolation-and-trust-boundaries.md), artifact/store 규칙은 [13-state-recovery-and-artifact-integrity.md](13-state-recovery-and-artifact-integrity.md)를 따른다.

## 2. Project Profile

예:

```yaml
id: logwarehouse
repository:
  url: git@internal.example.com/team/logwarehouse.git
  default_branch: main
workspace:
  strategy: git-worktree
context:
  include:
    - AGENTS.md
    - docs/architecture/**
commands:
  build: dotnet build LogWarehouse.sln
  test: dotnet test LogWarehouse.sln --no-build
permissions:
  deny_paths:
    - .env
    - secrets/**
policy_sensitive_paths:
  - .github/**
  - .agent-forge/architecture-baseline.json
  - tests/ArchitectureTests/**
```

Project Profile은 repository/context/registered commands/project policy를 소유하고 Role/Skill/runtime provider identity와 분리한다.

## 3. Local layout

```text
~/.agent-forge/
├─ repos/
│  ├─ logwarehouse.git/
│  └─ analyzer.git/
├─ worktrees/
│  ├─ T-001-logwarehouse/
│  └─ T-002-logwarehouse/
├─ state/
│  └─ agent-forge.db
├─ tasks/
└─ runs/
```

`tasks/`, `runs/`, `state/`는 Controller-owned 영역이다.

## 4. Workspace lifecycle

```text
Project resolve
 -> canonical repo/cache 확인 + fetch
 -> TaskSpec.base_revision exact SHA 확인
 -> task branch/worktree 생성
 -> workspace lease 획득
 -> runtime input staging
 -> baseline snapshot
 -> Agent 실행
 -> diff/untracked/policy 검사
 -> CheckRunner
 -> artifact store 수집
 -> lease release
 -> keep/remove/archive policy
```

remote default branch가 task 도중 이동해도 현재 Task는 pinned base SHA를 유지한다.

## 5. Branch/worktree naming

예:

```text
branch: agent-forge/T-001-carryover-fix
worktree: ~/.agent-forge/worktrees/T-001-logwarehouse
```

Controller가 생성한다.

Agent가 임의로 baseline branch/worktree를 바꾸지 않는다.

## 6. 여러 Agent의 workspace

### Sequential

```text
Implementer -> Reviewer -> Fix -> Verifier
```

동시 writer가 아니라면 동일 task worktree를 turn-taking으로 공유할 수 있다.

### Parallel writers

각 writer에 별도 worktree가 필요하다.

```text
Worker A -> worktree A
Worker B -> worktree B
```

MVP에서는 자동 merge/reconcile을 기본 기능으로 만들지 않는다.

### Workspace lease

같은 worktree에 동시에 하나의 writer만 허용한다.

Reviewer read-only 병행은 실제 filesystem enforcement가 가능한 경우에만 고려한다. 그렇지 않으면 순차 실행이 기본이다.

## 7. Runtime staging vs trusted artifact

Project-local staging이 필요한 runtime도 있다.

예:

```text
<worktree>/.agent-forge-runtime/
├─ context/
├─ generated/
└─ prompt-entry/
```

이 영역은 Worker가 접근할 수 있으므로 **canonical state/evidence가 아니다.**

Controller는 실행 전 staging input의 hash를 trusted store에 기록한다.

실행 결과의 canonical artifact는:

```text
~/.agent-forge/runs/<run-id>/
```

에 저장한다.

project repository 자체의 `.agent-forge/`가 architecture baseline 같은 source-controlled project config를 소유할 수 있지만, 그 변경은 policy-sensitive source change로 취급한다.

## 8. Context tier

```text
Tier 1 Always
  Frozen TaskSpec
  approved root/project instruction
  Project Profile summary

Tier 2 Selected
  relevant architecture/module docs
  relevant source/tests

Tier 3 On demand
  large logs/history/unrelated modules
```

모든 파일을 prompt에 넣지 않는다.

## 9. Context trust/provenance

Context source마다 다음을 기록한다.

```text
path/source
content hash
selection reason
trust class
```

Trust class 예:

```text
CONTROL
TRUSTED_PROJECT_INSTRUCTION
REFERENCE_CONTENT
```

Repository 안의 임의 문서/주석은 자동으로 control instruction이 되지 않는다.

## 10. Dirty baseline

canonical repo/cache는 작업용 dirty tree에 의존하지 않는다.

사용자가 특정 local dirty state를 기준으로 작업해야 한다면 explicit snapshot/import 기능으로 별도 취급한다.

TaskSpec freeze 시 exact base revision을 기록한다.

## 11. Post-run validation

최소:

- changed tracked files
- untracked files
- git diff
- task scope
- denied paths
- policy-sensitive paths
- unexpected submodule/worktree state

을 검사한다.

Symlink/path canonicalization과 workspace 밖 side effect 문제는 Git diff만으로 해결되지 않으므로 Runtime Isolation 정책과 함께 본다.

## 12. Build/Test command

Project Profile에 등록된 command가 기본 authority다.

CheckRunner는 registered command id와 실제 command/args를 artifact에 기록한다.

Worker가 test script/config 자체를 변경한 경우 검증 약화 가능성이 있으므로 change classification이 추가 review/control check를 요구할 수 있다.

## 13. Project add/validate

예:

```text
agent-forge project add <id> <repo-url>
agent-forge project validate <id>
agent-forge project list
agent-forge project remove <id>
```

validation에는 clone/fetch뿐 아니라:

- default branch resolve
- registered commands
- context paths
- denied/policy-sensitive paths
- architecture check references

를 포함한다.

## 14. 설계 불변조건

1. canonical repo/cache와 task worktree를 분리한다.
2. source workspace와 trusted state/artifact store를 분리한다.
3. task baseline은 exact commit SHA로 pin한다.
4. 동시에 쓰는 Agent는 같은 worktree를 공유하지 않는다.
5. worktree에는 writer lease를 둔다.
6. project content의 context trust class를 구분한다.
7. policy-sensitive verification/config 변경을 일반 source 변경으로 취급하지 않는다.
8. Git diff를 OS sandbox와 동일시하지 않는다.

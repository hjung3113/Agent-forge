# 05. Project Workspace & Context

## 1. 목적

특정 프로그램 Agent는 단순히 프로젝트 설명을 prompt로 받는 것이 아니라 **실제 target repository의 격리 workspace에서 실행**되어야 한다.

Agent Forge는 이를 위해 Project Registry + canonical repository cache + task worktree를 사용한다.

## 2. Project Profile

예시:

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
    - docs/domain/**
commands:
  build: dotnet build
  test: dotnet test
  lint: null
permissions:
  deny_paths:
    - .env
    - secrets/**
```

Project Profile은 **프로젝트 실행 메타데이터**를 소유한다.

다음은 Project Profile과 분리한다.

- role behavior
- generic coding skill
- runtime provider identity
- task-specific objective

## 3. Repository Layout

권장 local layout:

```text
~/.agent-forge/
├─ repos/
│  ├─ logwarehouse.git/       # canonical clone/cache
│  └─ analyzer.git/
│
├─ worktrees/
│  ├─ T-001-logwarehouse/
│  ├─ T-002-logwarehouse/
│  └─ T-003-analyzer/
│
├─ tasks/
├─ runs/
└─ logs/
```

매 task마다 remote에서 새로 clone하는 대신 canonical clone/cache를 유지하고 fetch 후 worktree를 만든다.

## 4. Workspace lifecycle

```text
Project resolve
  -> canonical repo 존재 확인
      -> 없으면 clone
      -> 있으면 fetch
  -> base ref 확인
  -> branch 생성
  -> git worktree add
  -> Agent Forge runtime context stage
  -> baseline snapshot
  -> Agent 실행
  -> diff/check/result 수집
  -> keep/remove/archive policy
```

### branch naming

예:

```text
agent-forge/<task-short-id>-<slug>
```

branch와 worktree path는 Controller가 생성하고 Agent가 임의로 결정하지 않는다.

## 5. Project Agent 실행

예:

```text
logwarehouse-expert
```

이 Agent를 실행할 때 Controller는 다음을 자동으로 resolve한다.

```text
Agent Profile
  role: project-expert
  project: logwarehouse

Controller
  -> ProjectRegistry(logwarehouse)
  -> workspace for current task
  -> cwd = worktree
  -> project context stage
  -> harness compile
  -> OpenCode run
```

따라서 Project Expert는 항상 해당 project source를 직접 읽을 수 있고, 임의의 다른 프로젝트 directory에서 실행되지 않는다.

## 6. 여러 Agent의 workspace 전략

모든 Agent가 반드시 별도 worktree를 가져야 하는 것은 아니다.

### 같은 task에서 turn-taking

```text
Implementer -> Reviewer -> Fix -> Verifier
```

동시에 쓰지 않는다면 같은 task worktree를 공유할 수 있다. Reviewer는 read-only policy로 실행한다.

### 병렬 구현

```text
Worker A  -> worktree A
Worker B  -> worktree B
```

동시에 source를 수정한다면 worktree를 분리한다.

### Domain Expert

project source를 읽을 필요가 없다면 workspace 없이 domain context만 사용할 수 있다. 프로젝트와 함께 분석해야 한다면 read-only project worktree 또는 동일 task worktree를 사용할 수 있다.

핵심 원칙은 **동시에 쓰는 두 Agent가 동일 working tree를 공유하지 않는 것**이다.

## 7. Runtime Context Directory

Project source와 Agent Forge artifact를 구분한다.

예:

```text
<worktree>/.agent-forge/
├─ context/
│  ├─ task.md
│  ├─ project.md
│  ├─ domain/
│  ├─ selected-docs/
│  └─ summary.md
├─ runtime/
│  └─ <run-id>/
│     ├─ resolved-agent.yaml
│     ├─ resolved-harness.yaml
│     └─ policy.json
├─ prompts/
├─ outputs/
├─ diffs/
└─ logs/
```

이 디렉터리는 실제 제품 source와 분리된 Agent Forge 실행 artifact다.

## 8. Context Source 계층

Project Agent에게 모든 파일을 처음부터 넣지 않는다.

```text
Tier 1 - Always
  Task Contract
  root AGENTS.md
  project profile

Tier 2 - Selected
  relevant architecture/module docs
  relevant source files
  relevant tests

Tier 3 - On demand
  large logs
  historical artifacts
  unrelated modules
```

Project Profile은 context를 직접 prompt에 모두 삽입하기보다 **어디에서 찾을지와 canonical source가 무엇인지** 정의하는 것이 좋다.

## 9. Context provenance

Agent가 어떤 근거를 사용했는지 추적할 수 있어야 한다.

`resolved-context.json` 예:

```json
{
  "project": "logwarehouse",
  "task": "T-001",
  "sources": [
    {"path": "AGENTS.md", "reason": "project root rule"},
    {"path": "docs/architecture/parser.md", "reason": "selected by project expert"},
    {"path": "src/Parser/Carryover.cs", "reason": "task scope"}
  ]
}
```

이는 stale/irrelevant context 문제를 디버깅할 때 중요하다.

## 10. Dirty baseline 정책

기존 canonical project가 dirty한 working directory에 의존해서는 안 된다. canonical clone/cache는 가능한 한 clean 상태를 유지한다.

worktree 생성 전:

- target base ref 확인
- uncommitted state와 무관한 canonical object DB 사용
- 예상하지 못한 local patch가 task baseline에 섞이지 않게 함

만약 사용자가 특정 local dirty state를 기준으로 작업해야 한다면 일반 Project Profile이 아니라 명시적 snapshot/import 기능으로 다뤄야 한다.

## 11. Post-run 검증

Agent 실행 전 baseline snapshot을 잡고 실행 후 다음을 계산한다.

- changed files
- untracked files
- git diff
- forbidden path changes
- source scope 밖 변경

허용 범위를 벗어난 변경을 자동 revert할지 task를 fail할지는 policy로 결정한다. 보안 민감 경로는 fail-closed가 기본이다.

## 12. Build/Test 명령

Project Profile의 명령은 Agent가 임의 생성하는 shell command보다 높은 신뢰도를 가진다.

```yaml
commands:
  build: dotnet build LogWarehouse.sln
  test: dotnet test LogWarehouse.sln --no-build
```

Verifier/CheckRunner는 가능한 한 등록된 명령을 사용한다. 임의 shell 명령이 필요하면 PolicyGate를 통과해야 한다.

## 13. 프로젝트 추가

초기 CLI 예:

```text
agent-forge project add <id> <repo-url>
agent-forge project validate <id>
agent-forge project list
agent-forge project remove <id>
```

`project add`는 repository를 등록하되 즉시 source를 수정하지 않는다. validation에서 clone/fetch 가능 여부, default branch, command, context path를 검사한다.

## 14. 설계 불변조건

1. Project Agent는 명시된 project workspace 밖에서 project write 작업을 하지 않는다.
2. canonical repo/cache와 task worktree 역할을 분리한다.
3. 병렬 writer는 동일 worktree를 공유하지 않는다.
4. runtime artifact와 product source를 구분한다.
5. context source의 provenance를 남긴다.
6. Project Profile은 runtime provider에 종속되지 않는다.

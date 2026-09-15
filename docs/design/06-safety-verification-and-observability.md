# 06. Safety, Verification & Observability

## 1. 원칙

Agent Forge는 LLM의 지시 준수를 보안 경계로 간주하지 않는다.

```text
Prompt says "do not write"
        !=
Filesystem write is technically prevented
```

따라서 안전성과 완료 신뢰성은 Controller가 가진 **hard gate + deterministic verification + artifact**로 확보한다.

## 2. Policy Gate

Policy Gate는 최소 다음을 검사한다.

### 2.1 Command Gate

차단 예:

- `git push --force`
- protected/default branch 직접 commit
- `rm -rf` 계열 위험 명령
- `curl | sh`, `wget | bash`
- 승인되지 않은 DB migration/reset
- 임의 외부 network 전송

### 2.2 Path Gate

- worktree root 밖 write
- secret file 접근
- task scope 밖 민감 경로 변경
- project profile의 denied path 접근

### 2.3 Spawn Gate

- 존재하지 않는 Agent
- recursive depth 초과
- concurrency 초과
- 비허용 skill/tool 요청
- project permission과 role permission 충돌

### 2.4 Workflow Gate

- 필수 verification step 누락
- reviewer가 구현 전/후 잘못된 dependency로 실행
- cyclic dependency
- 완료 gate bypass 시도

## 3. Fail-closed vs Fail-open

보안/무결성 관련 항목은 fail-closed를 기본으로 한다.

```text
secret access evaluation error -> deny
worktree root resolution failure -> deny
permission profile invalid -> deny
```

반면 관측 UI 전송 실패는 task 실행을 막지 않을 수 있다.

```text
Herdr display failure -> continue + log warning
```

## 4. Runtime permission과 Harness permission

Harness의 permission은 두 층으로 해석한다.

```text
Harness permission
   |
   +-> Advisory instruction
   |     "Reviewer는 파일을 수정하지 말 것"
   |
   `-> Enforced policy
         post-run diff must be empty
```

OpenCode가 provider-native permission 기능을 제공하는 경우 추가 방어선으로 사용할 수 있지만, Agent Forge의 security claim은 Controller가 실제 검증 가능한 범위만 대상으로 한다.

## 5. Baseline / Diff 기반 방어

각 Run 전후로 workspace를 비교한다.

```text
Before Run
  -> git status / snapshot

After Run
  -> changed set
  -> diff
  -> untracked files
  -> path policy validation
```

예를 들어 read-only reviewer가 source를 수정하면:

```text
reviewer run result = policy_violation
```

으로 처리한다. 필요하면 해당 Run이 만든 변경만 revert한다.

기존 dirty baseline과 Run이 만든 변경을 구분하지 못하는 구조는 피한다.

## 6. Verification 계층

완료 판단은 세 종류의 검증을 결합한다.

### 6.1 Output Contract Validation

Agent가 약속한 형식의 결과를 냈는지 검사한다.

예:

```text
Review Result: pass|fail
required section: Findings
minimum output size
structured JSON schema
```

### 6.2 Deterministic Checks

- build
- unit/integration test
- lint
- typecheck
- changed path validation
- git status
- static project-specific script

이 검사는 LLM Agent가 아니라 Controller/CheckRunner가 직접 실행한다.

### 6.3 Semantic Review

Reviewer가 판단한다.

- 요구사항 충족 여부
- 잘못된 가정
- edge case
- regression risk
- architecture/SOLID 위반
- 불필요한 scope expansion

## 7. Verification Evidence

Verifier는 acceptance criteria를 evidence에 연결한다.

권장 artifact:

```json
{
  "acceptance_criteria": [
    {
      "id": "AC-1",
      "status": "pass",
      "evidence": ["checks/dotnet-test.json", "diffs/final.diff"]
    }
  ]
}
```

모든 AC가 반드시 자동 검증 가능할 필요는 없다. 정성 항목은 Reviewer evidence로 연결하되 자동/수동인지 구분한다.

## 8. 독립 Reviewer 설계

동일 사내 LLM을 사용하더라도 다음으로 독립성을 높인다.

Reviewer에게 제공:

- Task Contract
- final diff
- relevant files
- check results
- project/domain rules

기본적으로 미제공:

- Implementer의 private reasoning
- "내 구현은 완벽하다" 같은 자기평가
- 불필요한 전체 transcript

필요 시 Reviewer가 추가 evidence를 직접 요청하게 한다.

## 9. Retry / Recovery

실패 원인을 공통 taxonomy로 둔다.

```text
runtime_unavailable
timeout
auth_failed
agent_error
output_contract_failed
policy_violation
check_failed
review_failed
configuration_error
```

Retry 가능 여부를 reason별로 정의한다.

- `timeout`: 제한적 retry 가능
- `output_contract_failed`: correction prompt로 retry 가능
- `check_failed`: debugger/implementer refine 가능
- `policy_violation`: 기본 retry 금지, 새 계획 필요
- `configuration_error`: retry 금지

## 10. Artifact Protocol

Run마다 최소 다음을 남긴다.

```text
runs/<run-id>/
├─ request.json
├─ resolved-agent.yaml
├─ resolved-harness.yaml
├─ policy.json
├─ prompt.md
├─ output.md
├─ stdout.log
├─ stderr.log
├─ diff.patch
├─ checks.json
└─ result.json
```

이 구조의 목적은 다음이다.

- 실패 재현
- 왜 특정 Agent가 선택되었는지 확인
- 어떤 skill/context가 들어갔는지 확인
- false completion 분석
- 향후 성능 비교

## 11. Event Model

관측은 process screen scraping보다 event를 중심으로 한다.

예:

```text
task.created
run.requested
run.started
run.output_received
run.policy_failed
run.completed
verification.started
verification.failed
task.completed
```

Event에는 `task_id`, `run_id`, timestamp, agent id, project id, state transition reason을 포함한다.

## 12. Herdr 역할

Herdr는 유용하지만 Agent Forge의 필수 control protocol은 아니다.

권장 역할:

```text
Herdr
├─ Orchestrator pane
├─ Worker panes
├─ Reviewer pane
├─ Verifier/check pane
└─ Controller/event/log pane
```

용도:

- 실행 상태 관측
- 특정 OpenCode process의 실시간 출력 확인
- operator의 수동 중단/개입
- debugging

비권장:

- pane 화면 문자열을 task authority로 사용
- Agent A가 pane B에 타이핑해야만 workflow가 진행되는 구조
- `working/done` UI 표기만 보고 task 상태 결정

Controller event/state가 canonical이고 Herdr는 그 상태를 보여주는 UI다.

## 13. Completion Integrity

다음은 완료 근거가 아니다.

```text
"완료했습니다"
"테스트가 통과할 것 같습니다"
OpenCode exit code 0
reviewer의 근거 없는 pass
```

DONE에는 최소 다음이 필요하다.

```text
Task Contract
+ expected artifacts
+ required checks
+ review outcome
+ policy pass
+ acceptance evidence
```

## 14. 보안 범위의 정직성

Agent Forge는 OS sandbox/container 자체를 제공하지 않는 MVP에서 "완전한 sandbox"라고 주장하지 않는다.

초기 방어는 다음 수준이다.

- cwd/worktree isolation
- allow/deny command policy
- path validation
- post-run diff validation
- environment allowlist
- network/tool restriction where enforceable

더 강한 격리가 필요해지면 container/sandbox backend를 별도 계층으로 추가한다.

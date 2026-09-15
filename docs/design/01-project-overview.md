# 01. Project Overview

## 1. 문제 정의

사내 환경에서는 외부 LLM/API 사용이 제한되고, 별도 API key 발급이나 임의의 base URL/gateway 사용도 승인 절차 또는 정책으로 막힐 수 있다. 반면 OpenCode처럼 사내에서 허용된 coding-agent 실행 경로는 사용할 수 있다.

Agent Forge는 이 제약 안에서 다음을 제공하기 위한 로컬 Agent Runtime이다.

- 한 명의 **Orchestrator Agent**가 사용자 목표를 해석하고 필요한 전문/역할 Agent를 호출
- `implementer`, `reviewer`, `verifier` 같은 역할 Agent뿐 아니라 `log-expert`, `project-expert` 같은 도메인/프로젝트 Agent 지원
- Agent별로 역할, 프로젝트, 도메인, skills, tools, permissions, context를 다르게 합성
- 프로젝트별 Git repository와 task worktree에 Agent를 바인딩
- OpenCode를 실행 backend로 사용하되 orchestration과 policy는 Agent Forge가 소유
- Herdr를 선택적 관측/운영 UI로 사용
- Agent/Skill/Project Profile을 나중에 추가·수정·비활성화할 수 있는 registry 제공

## 2. 목표

### 2.1 제품 목표

Agent Forge의 목표는 범용 외부 Agent 플랫폼을 완전히 복제하는 것이 아니라, 사내 환경에 필요한 핵심 기능을 **통제 가능하고 결정적인 형태로 재구성**하는 것이다.

```text
Approved OpenCode Runtime
        +
Deterministic Controller
        +
Composable Agent Harness
        +
Project-aware Workspace
        +
Verification / Safety
        =
Agent Forge
```

### 2.2 성공 기준

최소 성공 상태는 다음과 같다.

1. 사용자가 하나의 작업을 Orchestrator에 전달한다.
2. Orchestrator가 필요한 Agent를 구조화된 요청으로 선택한다.
3. Controller가 Agent Registry와 정책을 검증한다.
4. Project Agent라면 target repository의 격리 worktree를 준비한다.
5. Role/Project/Domain/Skill Harness를 합성한다.
6. OpenCode를 해당 cwd에서 실행한다.
7. 결과, diff, stdout/stderr, verification evidence를 수집한다.
8. Reviewer/Verifier 결과가 정책을 만족할 때만 완료한다.
9. 모든 상태와 판단 근거가 재현 가능한 artifact로 남는다.

project가 지정되지 않은 cross-project 대화형 진입은 후순위 기능이며 위 성공 기준에 포함되지 않는다.

## 3. 비목표

초기 단계에서 다음은 목표가 아니다.

- 외부 API 정책 우회
- 여러 외부 모델을 자유롭게 routing하는 model gateway
- LLM이 shell/pane을 직접 조작해 다른 Agent를 통제하는 구조
- 완전 자율적인 recursive swarm
- 분산 멀티머신 scheduler
- 대규모 workflow engine
- 자체 IDE 또는 Herdr 대체 terminal UI
- 사내 보안 정책을 우회하는 plugin/tool 설치

## 4. GrokBot/Hermes류와의 관계

Agent Forge는 다음 영역을 대체할 수 있다.

- Agent 역할 분리
- task delegation
- project-aware 실행
- multi-agent workflow
- code implementation/review/verification
- context/skill 주입
- 실행상태 관측

하지만 동일한 사내 LLM/OpenCode backend를 사용하는 경우 여러 Agent가 서로 다른 모델 지능을 제공하지는 않는다. 따라서 품질 향상의 핵심은 Agent 수가 아니라 다음이다.

- context isolation
- task contract 명확화
- role-specific evidence
- project/domain knowledge
- independent review context
- deterministic verification
- host-side permission enforcement

## 5. 핵심 설계 원칙

### P1. LLM decides, Controller governs

LLM은 의미 판단을 한다. Controller는 실행 가능 여부와 lifecycle을 통제한다.

### P2. Agent is a composed runtime profile

Agent를 prompt 파일 하나로 정의하지 않는다.

```text
Agent Runtime
 = Base Runtime
 + Role Profile
 + Project Profile
 + Domain Profile
 + Skill Set
 + Frozen TaskSpec
```

### P3. Project work is workspace-bound

특정 프로그램을 다루는 Agent는 target repository의 worktree에서만 실행한다.

### P4. Permission is enforced outside the model

Prompt의 "하지 마라"는 정책 설명일 뿐 hard security boundary가 아니다. 실제 차단은 Controller가 수행한다.

### P5. Verification is independent

구현 Agent의 자기평가를 완료 근거로 사용하지 않는다. Reviewer/Verifier에는 필요한 evidence만 제공한다.

### P6. UI is not the protocol

Herdr pane의 화면 텍스트나 키 입력은 핵심 IPC가 아니다. Agent 간 전달은 구조화된 request/result/event로 한다.

### P7. Start small

MVP는 OpenCode 단일 backend + 로컬 프로세스 + 파일/SQLite 수준으로 시작한다. 필요가 확인되기 전 Redis, message broker, distributed queue를 추가하지 않는다.

## 6. 대표 사용 예

```text
User: "LogWarehouse Carryover 처리 버그 수정해"

Orchestrator
  -> logwarehouse-expert: 관련 구조와 영향 범위 분석
  -> log-expert: Carryover 도메인 규칙 확인
  -> implementer: 코드 수정
  -> reviewer: diff 중심 독립 리뷰
  -> verifier: build/test/acceptance criteria 확인

Controller
  -> 모든 요청 검증
  -> worktree 생성
  -> harness 합성
  -> OpenCode 실행
  -> 결과/상태/재시도 관리
```

이때 `log-expert`와 `implementer`는 이름만 다른 Agent가 아니라 서로 다른 context, skill, permission, output contract를 가진 실행 프로파일이다.

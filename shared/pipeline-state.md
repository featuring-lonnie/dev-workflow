# 파이프라인 상태 관리

## 개요

`/dev-workflow:lfg` 자율 개발 파이프라인의 상태 관리 스키마 및 규칙을 정의합니다.

---

## 상태 파일 경로

```
.omc/state/lfg-state.json
```

---

## 상태 스키마

```json
{
  "version": "1.0",
  "ticketId": "PROJ-123",
  "pipeline": "lfg",
  "status": "RUNNING",
  "currentPhase": "WORKING",
  "startedAt": "2026-02-10T09:00:00Z",
  "updatedAt": "2026-02-10T09:30:00Z",
  "options": {
    "auto": false,
    "skipBrainstorm": false,
    "deepen": false,
    "skipReview": false,
    "skipCompound": false,
    "noNotify": false
  },
  "phases": {
    "INIT": {
      "status": "COMPLETED",
      "startedAt": "2026-02-10T09:00:00Z",
      "completedAt": "2026-02-10T09:02:00Z",
      "output": {
        "branchName": "feature/PROJ-123-user-auth",
        "projectPath": "~/projects/my-project"
      }
    },
    "PLANNING": {
      "status": "COMPLETED",
      "startedAt": "2026-02-10T09:02:00Z",
      "completedAt": "2026-02-10T09:15:00Z",
      "output": {
        "planDocPath": "docs/plans/PROJ-123-plan.md",
        "confluenceUrl": "https://company.atlassian.net/wiki/..."
      }
    },
    "WORKING": {
      "status": "IN_PROGRESS",
      "startedAt": "2026-02-10T09:15:00Z",
      "completedAt": null,
      "output": {
        "tasksTotal": 5,
        "tasksCompleted": 3,
        "commits": ["abc1234", "def5678"]
      }
    },
    "REVIEWING": {
      "status": "PENDING",
      "startedAt": null,
      "completedAt": null,
      "output": null
    },
    "FIXING": {
      "status": "PENDING",
      "startedAt": null,
      "completedAt": null,
      "output": null
    },
    "PR_CREATING": {
      "status": "PENDING",
      "startedAt": null,
      "completedAt": null,
      "output": null
    },
    "DOCUMENTING": {
      "status": "PENDING",
      "startedAt": null,
      "completedAt": null,
      "output": null
    },
    "NOTIFYING": {
      "status": "PENDING",
      "startedAt": null,
      "completedAt": null,
      "output": null
    },
    "COMPLETE": {
      "status": "PENDING",
      "startedAt": null,
      "completedAt": null,
      "output": null
    }
  },
  "errors": [],
  "metadata": {
    "jiraUrl": "https://company.atlassian.net/browse/PROJ-123",
    "ticketTitle": "사용자 인증 개선"
  }
}
```

---

## 상태 값 정의

### Pipeline Status

| 값 | 설명 |
|----|------|
| `RUNNING` | 파이프라인 실행 중 |
| `PAUSED` | 사용자 확인 대기 또는 에러로 일시 중지 |
| `COMPLETED` | 모든 Phase 완료 |
| `FAILED` | 복구 불가 에러로 중단 |
| `CANCELLED` | 사용자가 취소 |

### Phase Status

| 값 | 설명 |
|----|------|
| `PENDING` | 아직 시작되지 않음 |
| `IN_PROGRESS` | 현재 실행 중 |
| `COMPLETED` | 성공적으로 완료 |
| `SKIPPED` | 옵션에 의해 스킵됨 |
| `FAILED` | 에러로 실패 |

---

## Phase 목록 및 순서

| 순서 | Phase | 설명 | 스킵 가능 |
|------|-------|------|-----------|
| 1 | `INIT` | /start 로직 (Jira 상태 변경, 브랜치 생성) | 불가 |
| 2 | `PLANNING` | /brainstorm(선택) + /doc (병렬 리서치) | 불가 |
| 3 | `WORKING` | /work 로직 (태스크 루프 + 테스트 + 커밋) | 불가 |
| 4 | `REVIEWING` | /pr-review 로직 (병렬 전문 리뷰) | `--skip-review` |
| 5 | `FIXING` | P1/P2 자동 수정 (최대 3회) | 리뷰 결과에 따라 |
| 6 | `PR_CREATING` | /pr 로직 (PR 생성 + 링크) | 불가 |
| 7 | `DOCUMENTING` | /compound 로직 (지식 축적) | `--skip-compound` |
| 8 | `NOTIFYING` | /review 로직 (Slack 알림) | `--no-notify` |
| 9 | `COMPLETE` | 최종 보고 + 상태 정리 | 불가 |

---

## 사용자 확인 게이트

기본 모드(`--auto` 미사용)에서 다음 지점에서 사용자 확인을 요청합니다:

### Gate 1: PLANNING 완료 후

```
Tool: AskUserQuestion
Question: "계획이 준비되었습니다. 실행할까요?"
Options:
  - "실행" - 계획대로 WORKING 단계 시작
  - "계획 수정" - Plan 모드로 돌아가 수정
  - "중단" - 파이프라인 중단
```

### Gate 2: REVIEWING + FIXING 완료 후

```
Tool: AskUserQuestion
Question: "리뷰 통과. PR을 생성할까요?"
Options:
  - "PR 생성" - PR_CREATING 단계 시작
  - "추가 수정" - WORKING 단계로 돌아가 추가 작업
  - "중단" - 파이프라인 중단
```

**`--auto` 플래그 사용 시**: 모든 게이트 자동 통과 (완전 자율 실행)

---

## 재개 (Resume) 로직

`/dev-workflow:lfg --resume` 실행 시:

1. `.omc/state/lfg-state.json` 파일 확인
2. `status`가 `PAUSED` 또는 `RUNNING`인지 확인
3. `currentPhase`에서 이어서 실행
4. Phase의 `output`에서 이전 결과 참조

```
상태 파일 없음 → "재개할 파이프라인이 없습니다."
상태가 COMPLETED → "이미 완료된 파이프라인입니다. 새로 시작하시겠습니까?"
상태가 FAILED → "실패 지점: {currentPhase}. 이 단계부터 재시작하시겠습니까?"
상태가 PAUSED/RUNNING → "{currentPhase}부터 재개합니다."
```

---

## 에러 처리

### 에러 기록

```json
{
  "errors": [
    {
      "phase": "WORKING",
      "timestamp": "2026-02-10T09:25:00Z",
      "error": "Test failed: auth.test.ts",
      "retryCount": 1,
      "maxRetries": 3
    }
  ]
}
```

### 재시도 정책

| Phase | 최대 재시도 | 재시도 전략 |
|-------|-----------|------------|
| INIT | 1 | 전체 Phase 재실행 |
| PLANNING | 0 | 실패 시 PAUSED |
| WORKING | 3 (태스크별) | 개별 태스크 재시도 |
| REVIEWING | 1 | 전체 리뷰 재실행 |
| FIXING | 3 (라운드별) | P1/P2 항목 재수정 |
| PR_CREATING | 2 | PR 생성 재시도 |
| DOCUMENTING | 1 | 문서 생성 재시도 |
| NOTIFYING | 2 | 알림 재시도 |

### PAUSED 전환 조건

- 재시도 횟수 초과
- 사용자 확인 필요 (게이트)
- 복구 불가능한 에러 (인증 실패, 권한 없음)

---

## 상태 정리

파이프라인 완료 또는 취소 시:

1. `status`를 `COMPLETED` 또는 `CANCELLED`로 업데이트
2. 상태 파일은 **삭제하지 않음** (이력 보존)
3. 새 파이프라인 시작 시 기존 파일 덮어쓰기

**`/cancel` 실행 시:**
- `status`를 `CANCELLED`로 설정
- 실행 중인 에이전트는 자연 종료 대기

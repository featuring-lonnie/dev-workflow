---
description: 개발 작업 시작. Jira 티켓 상태를 In Progress로 변경하고 feature 브랜치 생성.
---

# /dev-workflow:start - 작업 시작

## 실행 단계

### 1. 티켓 번호 파싱
- 입력: `PROJ-123` 또는 Jira URL
- URL에서 티켓 ID 추출

### 2. Config에서 설정 로드

**먼저 `~/.claude/workflow/config.json`에서 필요한 설정을 읽습니다.**

```json
// 필요한 값들:
// - integrations.jira.cloudId (UUID 형식, 예: "2c7ce89f-01b8-412e-ba83-cb9d65b57537")
// - projects 배열 (Jira 프로젝트 → 로컬 경로 매핑)
```

### 3. Jira 티켓 조회
```
Tool: mcp__atlassian__getJiraIssue
Parameters:
  - cloudId: "{config.integrations.jira.cloudId}"  ← Config에서 읽은 UUID
  - issueIdOrKey: "{ticket_id}"
```

### 3. 프로젝트 매핑 (중요!)

**티켓의 프로젝트 키를 기반으로 로컬 프로젝트를 결정합니다.**

1. 티켓에서 프로젝트 키 추출 (예: `FS-533` → `FS`)
2. `~/.claude/workflow/config.json`의 `projects` 배열에서 해당 프로젝트 키를 `jiraProjects`에 포함하는 프로젝트 찾기
3. 해당 프로젝트의 `localPath`를 작업 디렉토리로 사용

```
config.json 프로젝트 매핑 예시:
{
  "projects": [
    { "name": "project-a", "localPath": "~/projects/project-a", "jiraProjects": ["BE", "FC"] },
    { "name": "project-b", "localPath": "~/projects/project-b", "jiraProjects": ["FS"] }
  ]
}

매핑 결과:
- FS-533 → FS → project-b (~/projects/project-b)
- BE-123 → BE → project-a (~/projects/project-a)
```

**매핑 실패 시:** 사용자에게 어느 프로젝트에서 작업할지 확인

### 4. Jira 상태 변경 → In Progress
```
Tool: mcp__atlassian__getTransitionsForJiraIssue
Parameters:
  - cloudId: "{config.integrations.jira.cloudId}"
  - issueIdOrKey: "{ticket_id}"
→ "In Progress" 또는 "진행 중" transition ID 찾기

Tool: mcp__atlassian__transitionJiraIssue
Parameters:
  - cloudId: "{config.integrations.jira.cloudId}"
  - issueIdOrKey: "{ticket_id}"
  - transitionId: "{in_progress_id}"
```

### 4-1. Jira 시작일 업데이트

**작업 시작 시 티켓의 시작일(Start Date)을 오늘 날짜로 설정합니다.**

```
Tool: mcp__atlassian__editJiraIssue
Parameters:
  - cloudId: "{config.integrations.jira.cloudId}"
  - issueIdOrKey: "{ticket_id}"
  - fields: { "customfield_10015": "{today_date_YYYY-MM-DD}" }
```

> **Note:** 시작일 필드 ID는 Jira 설정에 따라 다를 수 있습니다.
> `customfield_10015`가 작동하지 않으면 프로젝트 필드 메타데이터를 조회하여 확인하세요.

### 5. Git 브랜치 생성

**반드시 3단계에서 매핑된 프로젝트의 `localPath`에서 실행합니다.**

```bash
# 매핑된 프로젝트 디렉토리로 이동
cd {mapped_project_localPath}

# 현재 main/develop에서 분기
git fetch origin
git checkout -b feature/{ticket_id}-{short_description} origin/develop
```

브랜치 명명규칙:
- `feature/PROJ-123-user-signup` (기능)
- `fix/PROJ-123-login-error` (버그 수정)
- `hotfix/PROJ-123-critical-fix` (긴급 수정)

### 6. 작업 요약 출력

```
✅ 작업 시작 완료

📋 Jira: PROJ-123
   제목: 회원가입 API 구현
   상태: In Progress ← 변경됨
   시작일: 2026-02-05 ← 오늘 날짜로 설정됨

🌿 브랜치: feature/PROJ-123-user-signup
   기반: origin/develop

📝 다음 단계:
   1. 코드 작성
   2. /dev-workflow:commit 으로 커밋
   3. /dev-workflow:pr 로 PR 생성
```

## 에러 처리

| 상황 | 대응 |
|------|------|
| 티켓 없음 | "티켓을 찾을 수 없습니다: {ticket_id}" |
| 이미 In Progress | 상태 변경 스킵, 브랜치만 생성 |
| 브랜치 이미 존재 | 기존 브랜치로 checkout |
| Jira 권한 없음 | 권한 확인 안내 |

## 사용 예시

```
/dev-workflow:start PROJ-123
/dev-workflow:start PROJ-123 user-signup
/dev-workflow:start https://company.atlassian.net/browse/PROJ-123
```

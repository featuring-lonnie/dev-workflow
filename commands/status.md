---
description: 작업 상태 확인. Git, Jira, PR 상태 통합 조회.
---

# /dev-workflow:status - 상태 확인

## 출력 형식

```
작업 상태

Git
   브랜치: feature/PROJ-123-user-signup
   변경사항: 3 modified, 1 new
   커밋: 2 commits ahead of develop

Jira: PROJ-123
   제목: 회원가입 API 구현
   상태: In Progress
   담당자: lonnie

PR: #123
   상태: Open (리뷰 대기중)
   리뷰: 1/2 approved
   CI: All checks passed

타임라인
   시작: 2026-02-04 09:00
   경과: 4시간
```

## 실행 단계

### 0. 작업 디렉토리 확인 (선행 조건)

**현재 디렉토리가 올바른 프로젝트인지 확인합니다.**

```bash
# 현재 브랜치에서 티켓 ID 추출
git branch --show-current
# 예: feature/FS-533-http-only-cookie → FS

# config.json에서 해당 프로젝트 키의 localPath 확인
# FS → project-b (~/projects/project-b)
```

`/dev-workflow:start`로 작업을 시작했다면 이미 올바른 디렉토리에 있습니다.
독립적으로 실행 시 `~/.claude/workflow/config.json`의 `projects` 매핑을 참조하세요.

### 1. Git 상태 확인
```bash
# 현재 브랜치
git branch --show-current

# 변경사항
git status --short

# develop 대비 커밋 수
git rev-list --count origin/develop..HEAD
```

### 2. 티켓 ID 추출
브랜치명에서 파싱: `feature/PROJ-123-...` → `PROJ-123`

### 3. Jira 상태 조회
```
Tool: mcp__atlassian__getJiraIssue
Parameters:
  - issueIdOrKey: "PROJ-123"
```

### 4. PR 상태 조회 (있는 경우)
```bash
# PR 존재 여부
gh pr view --json number,state,reviews,statusCheckRollup 2>/dev/null

# CI 체크 상태
gh pr checks
```

### 5. 통합 상태 출력

## 상태별 아이콘

| 상태 | 아이콘 |
|------|--------|
| 성공/완료 | [OK] |
| 진행중 | [WIP] |
| 대기중 | [WAIT] |
| 실패/문제 | [FAIL] |
| 경고 | [WARN] |

## Jira 상태 매핑

| Jira 상태 | 표시 |
|-----------|------|
| To Do | To Do |
| In Progress | In Progress |
| In Review | In Review |
| Done | Done |

## PR 상태 매핑

| PR 상태 | 표시 |
|---------|------|
| Open + 리뷰 대기 | 리뷰 대기중 |
| Open + 리뷰 중 | 리뷰 중 |
| Open + 승인됨 | 머지 가능 |
| Merged | 머지됨 |
| Closed | 닫힘 |

## 사용 예시

```
/dev-workflow:status
# 현재 브랜치 기준 상태 확인

/dev-workflow:status PROJ-123
# 특정 티켓 상태 확인

/dev-workflow:status --all
# 내 모든 진행중 작업 확인
```

## --all 옵션

내가 담당한 모든 In Progress 티켓 조회:

```
Tool: mcp__atlassian__searchJiraIssuesUsingJql
Parameters:
  - jql: "assignee = currentUser() AND status = 'In Progress'"
```

출력:
```
내 진행중 작업 (3건)

1. PROJ-123 - 회원가입 API 구현
   상태: In Progress | PR: #123 (리뷰 대기)

2. PROJ-456 - 로그인 버그 수정
   상태: In Progress | PR: 없음

3. PROJ-789 - 결제 연동
   상태: In Review | PR: #125 (1/2 approved)
```

## 에러 상황

| 상황 | 표시 |
|------|------|
| 브랜치에 티켓 없음 | [WARN] 티켓 ID를 찾을 수 없습니다 |
| Jira 접근 불가 | [WARN] Jira 연결 확인 필요 |
| PR 없음 | [INFO] PR이 아직 생성되지 않았습니다 |

---
description: 개발 작업 시작. Jira 티켓 상태를 In Progress로 변경하고 feature 브랜치 생성.
---

# /dev-workflow:start - 작업 시작

## 실행 단계

### 1. 티켓 번호 파싱
- 입력: `PROJ-123` 또는 Jira URL
- URL에서 티켓 ID 추출

### 2. Jira 티켓 조회
```
Tool: mcp__atlassian__getJiraIssue
Parameters:
  - issueIdOrKey: "{ticket_id}"
```

### 3. Jira 상태 변경 → In Progress
```
Tool: mcp__atlassian__getTransitionsForJiraIssue
→ "In Progress" transition ID 찾기

Tool: mcp__atlassian__transitionJiraIssue
Parameters:
  - issueIdOrKey: "{ticket_id}"
  - transitionId: "{in_progress_id}"
```

### 4. Git 브랜치 생성
```bash
# 현재 main/develop에서 분기
git fetch origin
git checkout -b feature/{ticket_id}-{short_description} origin/develop
```

브랜치 명명규칙:
- `feature/PROJ-123-user-signup` (기능)
- `fix/PROJ-123-login-error` (버그 수정)
- `hotfix/PROJ-123-critical-fix` (긴급 수정)

### 5. 작업 요약 출력

```
✅ 작업 시작 완료

📋 Jira: PROJ-123
   제목: 회원가입 API 구현
   상태: In Progress ← 변경됨

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

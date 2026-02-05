---
description: 작업 완료. PR merge 및 Jira Done 처리.
---

# /dev-workflow:done - 작업 완료

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

### 1. PR 상태 확인
```bash
gh pr view --json state,reviews,statusCheckRollup,mergeable
```

체크 항목:
- [ ] 모든 리뷰 승인됨
- [ ] CI 체크 통과
- [ ] 충돌 없음

### 2. PR Merge
```bash
gh pr merge --squash --delete-branch
```

Merge 옵션:
- `--squash`: 커밋 스쿼시 (기본)
- `--merge`: 일반 머지
- `--rebase`: 리베이스 머지
- `--delete-branch`: 브랜치 삭제

### 3. Jira 상태 변경 → Done

**Config에서 cloudId 로드**: `~/.claude/workflow/config.json` → `integrations.jira.cloudId`

```
Tool: mcp__atlassian__getTransitionsForJiraIssue
Parameters:
  - cloudId: "{config.integrations.jira.cloudId}"
  - issueIdOrKey: "{ticket_id}"
→ "Done" 또는 "완료" transition ID 찾기

Tool: mcp__atlassian__transitionJiraIssue
Parameters:
  - cloudId: "{config.integrations.jira.cloudId}"
  - issueIdOrKey: "{ticket_id}"
  - transitionId: "{done_id}"
```

### 4. 완료 요약

```
✅ 작업 완료!

🔀 PR #123 머지됨
   - Squash merge to develop
   - 브랜치 feature/PROJ-123-user-signup 삭제됨

📋 Jira PROJ-123
   - 상태: Done ✓
   - 제목: 회원가입 API 구현

🚀 다음 단계:
   - deploy:staging 라벨 → 스테이징 자동 배포
   - 또는 수동 배포 진행
```

## Merge 전 검증

### 필수 조건
```bash
# 승인 수 확인
gh pr view --json reviews --jq '.reviews | map(select(.state=="APPROVED")) | length'

# CI 상태 확인
gh pr checks
```

### 실패 시 대응

| 상황 | 대응 |
|------|------|
| 승인 부족 | `/dev-workflow:review` 재요청 |
| CI 실패 | 로그 확인 후 수정 |
| 충돌 발생 | `git rebase develop` |
| Mergeable 아님 | 충돌 해결 필요 |

## 사용 예시

```
/dev-workflow:done
# 현재 브랜치의 PR 완료 처리

/dev-workflow:done PROJ-123
# 특정 티켓 완료 처리

/dev-workflow:done --merge
# squash 대신 일반 merge
```

## Jira 워크플로우 상태

```
To Do → In Progress → In Review → Done
                 ↑         ↓
                 └─ Reopened ←┘
```

## 완료 후 정리

- [x] PR 머지됨
- [x] 브랜치 삭제됨
- [x] Jira Done 처리됨
- [ ] 배포 확인 (별도)

---

## 예상치 못한 에러 처리

위 에러 외의 예상치 못한 에러가 발생한 경우:

1. **에러 분류 확인**: [공통 에러 핸들링 가이드](../shared/error-handling.md) 참조
2. **인증 에러가 아닌 경우**: GitHub 이슈 등록 여부 확인

```
Tool: AskUserQuestion
Question: "예상치 못한 에러가 발생했습니다. GitHub 이슈로 등록할까요?"
Options:
  - "이슈 등록" - dev-workflow 레포에 버그 리포트 생성
  - "건너뛰기" - 이슈 등록하지 않고 진행
  - "에러 상세 보기" - 에러 정보 확인 후 결정
```

**이슈 등록 시:**
```bash
gh issue create \
  --repo featuring-lonnie/dev-workflow \
  --title "[Bug] done: {에러 요약}" \
  --body "{이슈 본문 - shared/error-handling.md 템플릿 참조}" \
  --label "bug"
```

---
description: PR 생성. Jira 티켓 연동, deploy 라벨 추가.
---

# /dev-workflow:pr - PR 생성

## PR 템플릿

```markdown
## Summary
<!-- 작업 내용 1-3줄 요약 -->

## References
- Jira: [PROJ-123](https://company.atlassian.net/browse/PROJ-123)
- Doc: [Tech Spec](https://company.atlassian.net/wiki/spaces/.../pages/...)
<!-- Jira 티켓이나 문서가 없으면 해당 항목 생략 -->

## Changes
- 변경사항 1
- 변경사항 2
- 변경사항 3
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

### 1. 현재 브랜치 확인
```bash
git branch --show-current
# feature/PROJ-123-user-signup
```

### 2. 티켓 ID 추출
브랜치명에서 파싱: `feature/PROJ-123-...` → `PROJ-123`

### 3. 커밋 히스토리 분석
```bash
git log origin/develop..HEAD --oneline
```

### 4. 원격에 푸시
```bash
git push -u origin $(git branch --show-current)
```

### 5. References 조회

**Jira 티켓 링크 구성:**
- 브랜치명에서 추출한 티켓 ID로 Jira URL 생성
- `https://{org}.atlassian.net/browse/{ticket_id}`
- 티켓이 없는 브랜치면 생략

**Confluence 문서 링크 조회:**
- Jira 티켓의 remote links에서 Confluence 문서 탐색
```
Tool: mcp__atlassian__getJiraIssueRemoteIssueLinks
Parameters:
  - cloudId: "{config.integrations.jira.cloudId}"
  - issueIdOrKey: "{ticket_id}"
```
- Confluence 링크가 있으면 포함, 없으면 생략

### 6. PR 생성
```bash
gh pr create \
  --base develop \
  --title "[PROJ-123] 회원가입 API 구현" \
  --body "$(cat <<'EOF'
## Summary
회원가입 API 엔드포인트 추가

## References
- Jira: [PROJ-123](https://company.atlassian.net/browse/PROJ-123)
- Doc: [Tech Spec](https://company.atlassian.net/wiki/spaces/.../pages/...)

## Changes
- POST /api/v1/users/signup 엔드포인트 추가
- UserService.signup() 메서드 구현
- 입력값 검증 로직 추가

Written with Claude Code
EOF
)"
```

### 7. 라벨 추가
```bash
# 기본 라벨 (roles.developer.pr.defaultLabels)
gh pr edit --add-label "{roles.developer.pr.defaultLabels}"

# 배포 라벨 (사용자 선택 또는 자동)
gh pr edit --add-label "deploy:staging"
```

**설정 참조**: `~/.claude/workflow/config.json` → `roles.developer.pr.defaultLabels`

## 배포 라벨

| 라벨 | 의미 |
|------|------|
| `deploy:staging` | 스테이징 배포 |
| `deploy:production` | 프로덕션 배포 |
| `deploy:hotfix` | 긴급 핫픽스 |
| `no-deploy` | 배포 불필요 |

## PR 제목 규칙

```
[PROJ-123] 간결한 작업 설명
```

- 티켓 번호 prefix 필수
- 50자 이내
- 한글 OK

## 사용 예시

```
/dev-workflow:pr
# 자동으로 PR 생성

/dev-workflow:pr --label deploy:production
# 프로덕션 배포 라벨과 함께 생성

/dev-workflow:pr --draft
# Draft PR로 생성
```

## 에러 처리

| 상황 | 대응 |
|------|------|
| 푸시 안됨 | 먼저 push 실행 |
| PR 이미 존재 | 기존 PR URL 반환 |
| base 브랜치 충돌 | rebase 안내 |

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
  --title "[Bug] pr: {에러 요약}" \
  --body "{이슈 본문 - shared/error-handling.md 템플릿 참조}" \
  --label "bug"
```

---

## Attribution

PR 본문 마지막에 다음 attribution을 추가합니다:

```
Written with Claude Code
```

config.json의 `attribution.enabled`가 `false`인 경우 생략합니다.

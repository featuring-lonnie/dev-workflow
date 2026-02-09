---
description: 리뷰 요청. Slack 채널에 멘션으로 리뷰 요청 전송.
---

# /dev-workflow:review - 리뷰 요청

## Slack 메시지 형식

```
@백엔드 리뷰 부탁드립니다 🙏

📋 Jira: [PROJ-123] 회원가입 API 구현
🔗 PR: https://github.com/org/repo/pull/123

**변경 요약:**
- POST /api/v1/users/signup 엔드포인트 추가
- 입력값 검증 로직 구현

**리뷰 포인트:**
- 에러 핸들링 방식 확인 부탁드립니다

🤖 Written with Claude Code
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

### 1. PR 정보 조회
```bash
gh pr view --json url,title,number,body
```

### 2. Jira 티켓 정보 조회

**Config에서 cloudId 로드**: `~/.claude/workflow/config.json` → `integrations.jira.cloudId`

```
Tool: mcp__atlassian__getJiraIssue
Parameters:
  - cloudId: "{config.integrations.jira.cloudId}"
  - issueIdOrKey: "{ticket_id}"
```

### 3. Slack 메시지 전송

**중요:** `mentionGroup`에 `<!subteam^...>` 구문이 포함되어 있으므로 `content_type: text/plain`을 사용해야 Slack에서 멘션이 정상 동작합니다.

```
Tool: mcp__slack__conversations_add_message
Parameters:
  - channel_id: {roles.developer.slack.channelId}
  - content_type: text/plain
  - payload: "{roles.developer.slack.mentionGroup} 리뷰 부탁드립니다 🙏\n\n📋 Jira: ..."
```

**설정 참조**: `~/.claude/workflow/config.json`
- `roles.developer.slack.channelId` - 리뷰 요청 채널
- `roles.developer.slack.mentionGroup` - 멘션 그룹 (예: `<!subteam^S06FKUW4J92>`)
- `roles.developer.slack.mentionGroupDisplay` - 표시용 이름 (예: `@be`)

### 4. Jira 상태 업데이트 (선택)
```
Tool: mcp__atlassian__transitionJiraIssue
Parameters:
  - cloudId: "{config.integrations.jira.cloudId}"
  - issueIdOrKey: "{ticket_id}"
  - transitionId: "{in_review_id}"
- "In Review" 상태로 변경 (워크플로우에 있는 경우)
```

## 채널 및 멘션 설정

config.json에서 설정:
```json
{
  "roles": {
    "developer": {
      "slack": {
        "channelId": "...",
        "channelName": "...",
        "mentionGroup": "<!subteam^S06FKUW4J92>",
        "mentionGroupDisplay": "@be"
      }
    }
  }
}
```

## 옵션

### --urgent (긴급)
```
🚨 @백엔드 **긴급** 리뷰 부탁드립니다!

...
```

### --reviewer (특정 리뷰어)
```
@young @kim 리뷰 부탁드립니다 🙏

...
```

## 사용 예시

```
/dev-workflow:review
# 기본 리뷰 요청

/dev-workflow:review --urgent
# 긴급 리뷰 요청

/dev-workflow:review --reviewer young
# 특정 리뷰어 지정
```

## 리뷰 요청 후 상태

```
✅ 리뷰 요청 완료

📢 Slack: #개발_백엔드에 메시지 전송됨
🔗 PR: https://github.com/org/repo/pull/123
📋 Jira: PROJ-123 (In Review)

⏳ 리뷰 대기 중...
   /dev-workflow:status 로 상태 확인 가능
```

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
  --title "[Bug] review: {에러 요약}" \
  --body "{이슈 본문 - shared/error-handling.md 템플릿 참조}" \
  --label "bug"
```

---

## Attribution

Slack 메시지 마지막에 다음 attribution을 추가합니다:

```
🤖 Written with Claude Code
```

config.json의 `attribution.enabled`가 `false`인 경우 생략합니다.

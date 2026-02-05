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

### 1. PR 정보 조회
```bash
gh pr view --json url,title,number,body
```

### 2. Jira 티켓 정보 조회
```
Tool: mcp__atlassian__getJiraIssue
```

### 3. Slack 메시지 전송
```
Tool: mcp__slack__conversations_add_message
Parameters:
  - channel_id: {roles.developer.slack.channelId}
  - text: "{roles.developer.slack.mentionGroup} 리뷰 부탁드립니다 🙏\n\n📋 Jira: ..."
```

**설정 참조**: `~/.claude/workflow/config.json`
- `roles.developer.slack.channelId` - 리뷰 요청 채널
- `roles.developer.slack.mentionGroup` - 멘션 그룹 (@백엔드 등)

### 4. Jira 상태 업데이트 (선택)
```
Tool: mcp__atlassian__transitionJiraIssue
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
        "mentionGroup": "@백엔드"
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

## Attribution

Slack 메시지 마지막에 다음 attribution을 추가합니다:

```
🤖 Written with Claude Code
```

config.json의 `attribution.enabled`가 `false`인 경우 생략합니다.

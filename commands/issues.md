---
description: 멘션 수집 및 정리. Slack/Confluence에서 멘션을 수집하고 Jira 티켓 또는 TODO로 전환.
---

# /dev-workflow:issues - 멘션 수집 및 정리

## 개요

Slack과 Confluence에서 사용자 멘션을 수집하고, 맥락을 파악하여 TODO 또는 Jira 티켓으로 전환합니다.

---

## 실행 단계

### 1. Config 로드

**`~/.claude/workflow/config.json`에서 필요한 설정을 읽습니다.**

```json
// 필요한 값들:
// - currentUser.name
// - users.{username}.slack.userId
// - users.{username}.jira.accountId
// - integrations.jira.cloudId
// - integrations.confluence.cloudId
// - roles.developer.slack.channelId
// - roles.developer.slack.mentionGroup
// - roles.developer.jiraProjects
```

### 2. 옵션 파싱 및 시간 범위 결정

**커맨드 옵션:**
```
/dev-workflow:issues              # 기본값: 오늘 (00:00부터)
/dev-workflow:issues --today      # 오늘 00:00부터
/dev-workflow:issues --week       # 이번 주 월요일 00:00부터
/dev-workflow:issues --days N     # 지난 N일
```

**시간 범위 계산:**
```javascript
// 오늘 (기본값)
const today = new Date()
today.setHours(0, 0, 0, 0)

// 이번 주 월요일
const weekStart = new Date()
weekStart.setDate(weekStart.getDate() - weekStart.getDay() + 1)
weekStart.setHours(0, 0, 0, 0)

// N일 전
const nDaysAgo = new Date()
nDaysAgo.setDate(nDaysAgo.getDate() - N)
nDaysAgo.setHours(0, 0, 0, 0)
```

**날짜 포맷:**
- Slack: `YYYY-MM-DD` (예: "2026-02-05")
- Confluence CQL: `YYYY-MM-DD` 또는 `now("-7d")`

### 3. Slack 멘션 수집

#### 3.1 직접 멘션 검색

```
Tool: mcp__slack__conversations_search_messages
Parameters:
  - search_query: "<@{slack_user_id}>"
  - filter_date_after: "{start_date_YYYY-MM-DD}"
  - limit: 100
```

#### 3.2 그룹 멘션 검색 (역할 기반)

```
Tool: mcp__slack__conversations_search_messages
Parameters:
  - search_query: "{config.roles.developer.slack.mentionGroup}"  # 예: "@백엔드"
  - filter_date_after: "{start_date_YYYY-MM-DD}"
  - limit: 100
```

#### 3.3 메시지 정보 추출

수집된 메시지에서 추출:
- 메시지 텍스트 (text)
- 작성자 (user)
- 채널 (channel)
- 타임스탬프 (ts)
- 스레드 여부 (thread_ts)
- 메시지 URL (permalink)

### 4. Slack 스레드 맥락 수집

**스레드 메시지인 경우 전체 대화 맥락 수집:**

```
Tool: mcp__slack__conversations_replies
Parameters:
  - channel_id: "{channel_id}"
  - thread_ts: "{thread_ts}"
  - limit: "50"
```

- 스레드의 첫 메시지(부모)와 전체 대화 맥락 파악
- 답변 여부 확인 (hasReply)

### 5. Confluence 멘션 수집

#### 5.1 CQL로 멘션 검색

```
Tool: mcp__atlassian__searchConfluenceUsingCql
Parameters:
  - cloudId: "{config.integrations.confluence.cloudId}"
  - cql: "mention = currentUser() AND lastModified >= '{start_date_YYYY-MM-DD}'"
  - limit: 100
```

#### 5.2 페이지 내용 조회

검색 결과의 각 페이지에 대해:
```
Tool: mcp__atlassian__getConfluencePage
Parameters:
  - cloudId: "{config.integrations.confluence.cloudId}"
  - pageId: "{page_id}"
  - contentFormat: "markdown"
```

#### 5.3 멘션 맥락 파싱

- 페이지 제목
- 멘션이 포함된 섹션 (heading 기준)
- 멘션 전후 3문단
- 페이지 URL

### 6. 중복 제거 및 통합

**중복 제거 기준:**
- Slack: 같은 `thread_ts`의 멘션은 1개로 통합
- Confluence: 같은 `pageId`의 멘션은 1개로 통합

### 7. 멘션 분류 및 우선순위화

#### 7.1 카테고리 분류

| 카테고리 | 기준 |
|----------|------|
| Urgent | 긴급, 장애, 프로덕션, 에러 급증, 🚨 |
| Action Required | 해주세요, 부탁, 필요합니다, 요청, 확인 |
| Question | ?, 어떻게, 언제, 왜, 무엇을 |
| FYI | 정보 공유, 참고, 공유드립니다 |

#### 7.2 우선순위 점수 계산

| 기준 | 점수 |
|------|------|
| 직접 멘션 (@username) | +5 |
| 그룹 멘션 (@팀명) | +3 |
| 24시간 이내 | +3 |
| 미답변 스레드 (Slack) | +2 |
| 댓글 없음 (Confluence) | +2 |
| Urgent 키워드 | +5 |
| Action Required 키워드 | +4 |
| Question 키워드 | +3 |

점수 기준 내림차순 정렬

### 8. 수집 결과 요약 출력

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
📬 /dev-workflow:issues
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

⏱️  수집 기간: {time_range_description}

📊 수집 결과:
  Slack: {slack_count}건 (직접: {direct_count}, 그룹: {group_count})
  Confluence: {confluence_count}건
  총: {total_count}건 → 중복 제거: {unique_count}건

우선순위별:
  - 🔴 긴급 (Urgent): {urgent_count}건
  - 🟠 액션 필요 (Action Required): {action_count}건
  - 🟡 질문 (Question): {question_count}건
  - 🔵 정보 공유 (FYI): {fyi_count}건

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

### 9. 개별 멘션 처리

**각 멘션에 대해 순차적으로:**

#### 9.1 멘션 상세 표시

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
[{index}/{total}] {category_emoji} {category} | {source} | {time_ago}
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

From: {author} @ {channel_or_space}
{thread_title_if_slack}

> {message_text}

[스레드/페이지 맥락]
{context}

🔗 {source_url}

─────────────────────────────────────
```

#### 9.2 액션 선택

```
Tool: AskUserQuestion
Question: "액션을 선택하세요:"
Options:
  - "Jira 티켓 생성" - {jira_project} 프로젝트에 티켓 생성
  - "TODO 추가" - TaskCreate로 추가
  - "스킵" - 다음 멘션으로
```

### 10. Jira 티켓 생성 (선택 시)

```
Tool: mcp__atlassian__createJiraIssue
Parameters:
  - cloudId: "{config.integrations.jira.cloudId}"
  - fields:
      project:
        key: "{config.roles.developer.jiraProjects[0]}"
      summary: "[{source}] {ai_generated_title}"
      description: |
        # 출처
        {source_type} 멘션: {source_url}
        From: {author} @ {channel_or_space}
        Date: {timestamp}

        # 내용
        {message_text}

        # 맥락
        {context}

        # 분류
        Category: {category}
        Priority Score: {priority_score}
      issuetype:
        name: "Task"
      priority:
        name: "{priority_name}"
      assignee:
        accountId: "{config.users.{currentUser}.jira.accountId}"
```

**우선순위 매핑:**
| 카테고리 | Jira Priority |
|----------|---------------|
| Urgent | Highest |
| Action Required | High |
| Question | Medium |
| FYI | Low |

### 11. TODO 추가 (선택 시)

```
Tool: TaskCreate
Parameters:
  - subject: "{mention_summary_title}"
  - description: |
      출처: {source_type} ({source_url})
      From: {author}
      Date: {timestamp}

      {message_text}

      {context}
  - activeForm: "Processing mention from {source}"
```

### 12. 처리 결과 출력

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
✅ 멘션 처리 완료
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

📊 처리 결과:
  Jira 티켓: {jira_count}건
  TODO: {todo_count}건
  스킵: {skip_count}건

🎫 생성된 Jira 티켓:

  {ticket_key}: {ticket_summary}
  Priority: {priority}
  🔗 {ticket_url}

📝 TODO 목록 ({todo_count}건):
  ✓ Built-in TaskCreate로 추가됨
  {todo_list}

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

💡 다음 단계:
  - /dev-workflow:start {ticket_key} 으로 작업 시작
  - /dev-workflow:status 로 현재 작업 상태 확인
  - TaskList 로 TODO 확인

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

---

## 에러 처리

| 상황 | 대응 |
|------|------|
| Slack 인증 만료 | "Slack MCP 재인증 필요" 안내 |
| Confluence 권한 없음 | "Confluence 접근 권한 확인" 안내 |
| 멘션 0건 | "지정된 기간에 멘션이 없습니다" 안내 후 정상 종료 |
| Jira 프로젝트 없음 | config의 jiraProjects 확인 안내 |
| 기타 에러 | [공통 에러 핸들링 가이드](../shared/error-handling.md) 참조 |

---

## 사용 예시

```
/dev-workflow:issues
# 오늘의 멘션 수집 (기본값)

/dev-workflow:issues --today
# 오늘 00:00부터 현재까지

/dev-workflow:issues --week
# 이번 주 월요일부터 현재까지

/dev-workflow:issues --days 7
# 지난 7일간의 멘션 (휴가 복귀 후 사용 추천)

/dev-workflow:issues --days 30
# 지난 30일간의 멘션 (장기 휴가 복귀, 프로젝트 인수인계 시)
```

---

## 카테고리 이모지

| 카테고리 | 이모지 |
|----------|--------|
| Urgent | 🔴 |
| Action Required | 🟠 |
| Question | 🟡 |
| FYI | 🔵 |

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
  --title "[Bug] issues: {에러 요약}" \
  --body "{이슈 본문 - shared/error-handling.md 템플릿 참조}" \
  --label "bug"
```

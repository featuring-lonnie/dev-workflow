---
description: PR 리뷰 승인. gh CLI로 approve.
---

# /dev-workflow:approve - 리뷰 승인

## 실행 단계

### 1. PR 정보 확인
```bash
# PR 번호 또는 URL에서 파싱
gh pr view {pr_number} --json title,author,additions,deletions,files,reviews
```

### 2. 변경사항 요약 확인
```bash
gh pr diff {pr_number} --name-only
```

### 3. 리뷰 승인
```bash
gh pr review {pr_number} --approve --body "LGTM 👍"
```

### 4. 승인 완료 알림

```
✅ PR 승인 완료

🔗 PR: #123 [PROJ-123] 회원가입 API 구현
👤 작성자: @young
📊 변경: +150 -20 (5 files)

💬 승인 코멘트: LGTM 👍
```

## 승인 코멘트 옵션

### 기본
```bash
gh pr review --approve --body "$(cat <<'EOF'
LGTM 👍

🤖 Written with Claude Code
EOF
)"
```

### 코멘트와 함께
```bash
gh pr review --approve --body "LGTM 👍

몇 가지 제안:
- 추후 캐싱 적용 고려
- 에러 메시지 개선하면 좋을 것 같아요"
```

## 사용 예시

```
/dev-workflow:approve 123
# PR #123 승인

/dev-workflow:approve https://github.com/org/repo/pull/123
# URL로 승인

/dev-workflow:approve 123 --comment "에러 핸들링 잘 되어있네요!"
# 코멘트와 함께 승인
```

## Slack 알림 (선택)

리뷰 요청 스레드에 답글:
```
Tool: mcp__slack__conversations_add_message
Parameters:
  - channelId: {roles.developer.slack.channelId}
  - thread_ts: {original_message_ts}
  - text: "✅ 승인했습니다!"
```

**설정 참조**: `~/.claude/workflow/config.json` → `roles.developer.slack.channelId`

## 주의사항

- 본인 PR은 승인 불가
- 최소 1명 이상 승인 필요 (팀 정책에 따라)
- CI 통과 확인 후 승인 권장

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
  --title "[Bug] approve: {에러 요약}" \
  --body "{이슈 본문 - shared/error-handling.md 템플릿 참조}" \
  --label "bug"
```

---

## Attribution

리뷰 코멘트 마지막에 다음 attribution을 추가합니다:

```
🤖 Written with Claude Code
```

config.json의 `attribution.enabled`가 `false`인 경우 생략합니다.

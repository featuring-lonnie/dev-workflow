# 공통 에러 핸들링 가이드

## 개요

dev-workflow 커맨드 실행 중 에러가 발생했을 때의 표준 처리 절차입니다.

---

## 에러 분류

### 1. 인증 에러 (Auth Error)

**패턴:**
- Atlassian: `401 Unauthorized`, `Authentication required`
- Slack: `not_authed`, `invalid_auth`, `token_revoked`
- GitHub: `401`, `Bad credentials`

**처리:**
```
사용자에게 재인증 안내:
- Atlassian: "/mcp 명령어로 Atlassian 재인증 필요"
- Slack: "Slack MCP 토큰 갱신 필요"
- GitHub: "gh auth login으로 재인증 필요"
```

### 2. 권한 에러 (Permission Error)

**패턴:**
- `403 Forbidden`, `Permission denied`
- `You don't have permission to...`

**처리:**
```
사용자에게 권한 확인 안내 후 이슈 등록 제안
```

### 3. 설정 에러 (Config Error)

**패턴:**
- `config.json not found`
- `Missing required field: ...`
- `Invalid cloudId`

**처리:**
```
/dev-workflow:init 재실행 안내
```

### 4. 기타 에러 (Other Error)

**패턴:**
- 위 분류에 해당하지 않는 모든 에러
- MCP 도구 실패
- 예상치 못한 응답

**처리:**
```
→ GitHub 이슈 등록 여부 확인
```

---

## 이슈 등록 프로세스

### Step 1: 에러 감지 및 분류

```
에러 발생
  ↓
인증 에러인가? → Yes → 재인증 안내 (이슈 등록 X)
  ↓ No
권한 에러인가? → Yes → 권한 확인 안내 + 이슈 등록 제안
  ↓ No
설정 에러인가? → Yes → init 재실행 안내 (이슈 등록 X)
  ↓ No
기타 에러 → 이슈 등록 여부 확인
```

### Step 2: 사용자 확인 (AskUserQuestion)

**인증/설정 에러가 아닌 경우에만 실행:**

```
Tool: AskUserQuestion
Question: "예상치 못한 에러가 발생했습니다. GitHub 이슈로 등록할까요?"
Options:
  - "이슈 등록" - dev-workflow 레포에 버그 리포트 생성
  - "건너뛰기" - 이슈 등록하지 않고 진행
  - "에러 상세 보기" - 에러 정보 확인 후 결정
```

### Step 3: 이슈 생성 (사용자가 "이슈 등록" 선택 시)

```bash
gh issue create \
  --repo featuring-lonnie/dev-workflow \
  --title "[Bug] {커맨드명}: {에러 요약}" \
  --body "$(cat <<'EOF'
## 환경 정보

| 항목 | 값 |
|------|-----|
| OS | {os_info} |
| Claude Code | {claude_code_version} |
| dev-workflow | {plugin_version} |

## 실행한 커맨드

```
{실행한 커맨드}
```

## 에러 메시지

```
{에러 메시지 전문}
```

## 재현 단계

1. {단계 1}
2. {단계 2}
3. ...

## 추가 정보

{사용자가 추가한 정보 (있는 경우)}

---
Reported via dev-workflow error handler
EOF
)" \
  --label "bug"
```

---

## 이슈 템플릿

### 제목 형식

```
[Bug] {커맨드명}: {에러 요약 (50자 이내)}
```

예시:
- `[Bug] start: Jira transition ID not found`
- `[Bug] pr-review: Slack channel not accessible`
- `[Bug] doc: Confluence page creation failed`

### 본문 템플릿

```markdown
## 환경 정보

| 항목 | 값 |
|------|-----|
| OS | macOS 14.x / Ubuntu 22.04 / Windows 11 |
| Claude Code | vX.X.X |
| dev-workflow | 1.0.0 |

## 실행한 커맨드

```
/dev-workflow:{command} {arguments}
```

## 에러 메시지

```
{전체 에러 메시지}
```

## 재현 단계

1. config.json 설정 완료 상태
2. /dev-workflow:{command} 실행
3. 에러 발생

## 추가 정보

- 특이 사항이나 추가 컨텍스트

---
Reported via dev-workflow error handler
```

---

## 환경 정보 수집 방법

### OS 정보

```bash
# macOS
sw_vers -productVersion

# Linux
cat /etc/os-release | grep PRETTY_NAME

# 크로스 플랫폼
uname -a
```

### Claude Code 버전

```bash
claude --version
```

### 플러그인 버전

```
~/.claude/plugins/cache/lonnie-marketplace/dev-workflow/{version}/
또는 .claude-plugin/plugin.json의 version 필드
```

---

## 커맨드별 에러 핸들링 참조

각 커맨드 파일의 "에러 처리" 섹션에서 이 가이드를 참조합니다:

```markdown
## 에러 처리

| 상황 | 대응 |
|------|------|
| {커맨드별 특정 에러} | {특정 대응} |
| 기타 에러 | [공통 에러 핸들링 가이드](../shared/error-handling.md) 참조 |
```

---

## 이슈 등록 후 안내

이슈가 생성되면 사용자에게 다음을 안내합니다:

```
GitHub 이슈가 등록되었습니다.

이슈: #{issue_number}
URL: https://github.com/featuring-lonnie/dev-workflow/issues/{issue_number}

추가 정보가 있으면 이슈에 코멘트로 남겨주세요.
감사합니다!
```

---

## 에러 핸들링 비활성화

config.json에서 이슈 등록 제안을 비활성화할 수 있습니다:

```json
{
  "errorHandling": {
    "suggestIssue": false
  }
}
```

기본값: `true` (이슈 등록 제안 활성화)

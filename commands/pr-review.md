---
description: PR 리뷰 수행. Slack 리뷰 요청에서 PR 확인 후 로컬 체크아웃하여 코드 리뷰. P5 포맷 + 3자 에이전트 합의 리뷰.
---

# /dev-workflow:pr-review - PR 리뷰 수행

## 개요

Slack의 리뷰 요청 메시지에서 PR 링크를 확인하고, 로컬에서 코드를 체크아웃하여 리뷰를 수행합니다.

---

## 실행 단계

### 1. Slack 리뷰 요청 확인

```
Tool: mcp__slack__conversations_history
Parameters:
  - channelId: {roles.developer.slack.channelId}
  - limit: 20

# 또는 특정 메시지 검색
Tool: mcp__slack__conversations_search_messages
Parameters:
  - query: "리뷰 부탁"
```

**설정 참조**: `~/.claude/workflow/config.json` → `roles.developer.slack.channelId`

**파싱 대상:**
- PR URL: `https://github.com/{org}/{repo}/pull/{number}`
- Jira URL: `https://featuring-corp.atlassian.net/browse/{TICKET}`

### 2. PR 정보 조회

```bash
# PR 상세 정보
gh pr view {pr_number} --repo {org}/{repo} --json title,body,headRefName,additions,deletions,files,author

# 변경된 파일 목록
gh pr view {pr_number} --repo {org}/{repo} --json files --jq '.files[].path'

# PR diff 확인
gh pr diff {pr_number} --repo {org}/{repo}
```

### 3. 로컬 프로젝트 매핑

config.json의 projects 섹션에서 GitHub repo와 로컬 경로를 매핑:

```json
{
  "projects": [
    {
      "name": "project-a",
      "localPath": "~/projects/project-a",
      "github": {
        "org": "my-org",
        "repo": "project-a"
      }
    }
  ]
}
```

**매핑 로직:**
1. PR URL에서 org/repo 추출
2. config.projects에서 일치하는 항목 찾기
3. localPath로 이동

### 4. 로컬 체크아웃

```bash
# 프로젝트 디렉토리로 이동
cd {localPath}

# 최신 코드 fetch
git fetch origin

# PR 브랜치 체크아웃
gh pr checkout {pr_number}

# 또는 브랜치명으로 직접
git checkout {headRefName}
```

### 5. 비즈니스 컨텍스트 파악

**Jira 티켓에서 요구사항 확인:**
```
Tool: mcp__atlassian__getJiraIssue
Parameters:
  - issueIdOrKey: "{ticket_id}"
```

**확인 항목:**
- 티켓 설명 (description)
- 인수 조건 (acceptance criteria)
- 관련 티켓 링크
- 댓글의 추가 요구사항

**Confluence 기술 문서 확인 (있는 경우):**
```
Tool: mcp__atlassian__searchConfluenceUsingCql
Parameters:
  - cql: "text ~ '{ticket_id}' and type = page"
```

### 6. 관련 코드 분석

**변경된 파일의 기존 코드 이해:**
```bash
# 변경된 파일 목록
git diff origin/develop...HEAD --name-only

# 각 파일의 전체 컨텍스트 파악
# - 해당 파일이 속한 모듈/앱의 역할
# - 관련 모델, 서비스, 뷰 파일
# - 기존 비즈니스 로직 흐름
```

**호출 관계 분석:**
- 변경된 함수/메서드를 호출하는 코드 확인
- 변경으로 인한 사이드 이펙트 범위 파악

### 7. 코드 리뷰 수행

#### 기본 체크리스트

| 항목 | 확인 내용 |
|------|-----------|
| 기능 | 요구사항 충족 여부 |
| 코드 품질 | 가독성, 네이밍, 구조 |
| 보안 | 입력 검증, 인증/인가 |
| 성능 | 불필요한 연산, N+1 쿼리 |
| 에러 처리 | 예외 처리, 로깅 |

#### 비즈니스 로직 심층 리뷰 (CRITICAL)

| 카테고리 | 확인 항목 |
|----------|-----------|
| **요구사항 일치** | Jira 티켓의 인수 조건과 구현이 정확히 일치하는가? |
| **엣지케이스** | 빈 값, null, 경계값, 동시성 상황을 처리하는가? |
| **데이터 정합성** | DB 트랜잭션이 올바른가? 부분 실패 시 롤백되는가? |
| **도메인 규칙** | 비즈니스 도메인 규칙을 위반하지 않는가? |
| **기존 로직 호환** | 기존 기능에 영향을 주지 않는가? (regression) |
| **외부 연동** | 외부 API 실패 시 적절히 처리하는가? |
| **사용자 시나리오** | 실제 사용자 관점에서 올바르게 동작하는가? |

### 8. 3자 에이전트 합의 리뷰 (Multi-Perspective Review)

**3가지 관점에서 코드를 분석하고 합의:**

| 역할 | 관점 | 주요 체크포인트 |
|------|------|----------------|
| **리더급** | 아키텍처, 비즈니스 임팩트 | 설계 방향성, 확장성, 기술 부채, 비즈니스 리스크 |
| **시니어급** | 구현 품질, 유지보수성 | 코드 구조, 패턴 일관성, 에러 핸들링 |
| **주니어급** | 가독성, 학습 관점 | 코드 이해도, 문서화, 네이밍, 컨벤션 준수 |

**실행 방법:**
```
1. 각 관점에서 독립적으로 코드 분석
2. 발견한 이슈들을 P1-P5로 분류
3. 3자 의견 종합하여 최종 리뷰 작성
```

**리더급 관점 체크:**
- [ ] 이 변경이 시스템 전체 아키텍처에 미치는 영향은?
- [ ] 향후 확장성을 고려한 설계인가?
- [ ] 기술 부채를 늘리는가, 줄이는가?
- [ ] 비즈니스 요구사항과 정확히 일치하는가?
- [ ] 장애 발생 시 영향 범위는?

**시니어급 관점 체크:**
- [ ] 기존 코드 패턴과 일관성 있는가?
- [ ] 에러 핸들링이 충분한가?
- [ ] 성능 이슈가 예상되는가?
- [ ] 보안 취약점은 없는가?

**주니어급 관점 체크:**
- [ ] 코드를 처음 보는 사람이 이해할 수 있는가?
- [ ] 함수/변수명이 명확한가?
- [ ] 주석이 필요한 복잡한 로직이 있는가?
- [ ] 팀 컨벤션을 따르고 있는가?
- [ ] 비슷한 코드가 다른 곳에 있지 않은가?

---

### 9. P5 리뷰 포맷 적용

**우선순위 기반 피드백 분류:**

| 등급 | 의미 | 액션 | 예시 |
|------|------|------|------|
| **P1** | Blocker | 반드시 수정 후 머지 | 보안 취약점, 데이터 손실 가능성, 크리티컬 버그 |
| **P2** | Major | 머지 전 수정 권장 | 비즈니스 로직 오류, 성능 이슈 |
| **P3** | Medium | 수정 필요 (후속 PR 가능) | 에러 핸들링 개선, 코드 구조 개선 |
| **P4** | Minor | 권장 사항 | 네이밍 개선, 중복 코드 리팩토링 |
| **P5** | Nitpick | 선택 사항 | 스타일 선호도, 사소한 개선점 |

**리뷰 코멘트 작성 형식:**

```markdown
## PR Review: #{pr_number}

### P1 - Blocker
> 반드시 수정 필요

#### 1. {파일명}:{라인}
```{language}
{코드 스니펫}
```
**문제**: {설명}
**제안**: {해결 방안}

---

### P2 - Major
> 머지 전 수정 권장

#### 1. {파일명}:{라인}
**문제**: {설명}
**제안**: {해결 방안}

---

### P3 - Medium
> 수정 필요 (후속 PR 가능)

- `{파일}:{라인}`: {내용}

---

### P4 - Minor

- `{파일}:{라인}`: {내용}

---

### P5 - Nitpick

- {내용}

---

### Good Parts

- {칭찬할 부분}

---

### 리뷰 요약

| 등급 | 개수 |
|------|------|
| P1 | {n} |
| P2 | {n} |
| P3 | {n} |
| P4 | {n} |
| P5 | {n} |

**최종 판정**: {APPROVE / REQUEST_CHANGES / COMMENT}
**사유**: {판정 이유}

Written with Claude Code
```

---

### 10. 리뷰 코멘트 작성

```bash
# 리뷰 승인
gh pr review {pr_number} --approve --body "$(cat <<'EOF'
LGTM!

## 확인 사항
- [x] 기능 동작 확인
- [x] 코드 품질 양호

Written with Claude Code
EOF
)"

# 변경 요청
gh pr review {pr_number} --request-changes --body "$(cat <<'EOF'
## 변경 요청 사항

### 1. {파일명}:{라인}
{설명}

### 2. {파일명}:{라인}
{설명}

Written with Claude Code
EOF
)"

# 코멘트만 (승인/거절 없이)
gh pr review {pr_number} --comment --body "$(cat <<'EOF'
## 리뷰 코멘트

{코멘트 내용}

Written with Claude Code
EOF
)"
```

---

## 출력 형식

### 리뷰 시작 시

```
PR 리뷰 시작

PR: #{pr_number} - {title}
   URL: {pr_url}
작성자: {author}
변경: +{additions} -{deletions} ({files_count} files)

Jira: {ticket_id}
   URL: {jira_url}

로컬 경로: {localPath}
브랜치: {headRefName}

변경된 파일:
- {file1}
- {file2}
...
```

### 리뷰 완료 시

```
PR 리뷰 완료

PR: #{pr_number} - {title}
리뷰 결과: {APPROVED / CHANGES_REQUESTED / COMMENTED}

{리뷰 요약}

Written with Claude Code
```

---

## 사용 예시

```
/dev-workflow:pr-review
# Slack에서 최근 리뷰 요청 확인 후 진행

/dev-workflow:pr-review 123
# PR #123 직접 리뷰

/dev-workflow:pr-review https://github.com/featuring-corp/featuring-co/pull/123
# URL로 직접 리뷰

/dev-workflow:pr-review --slack-thread {thread_ts}
# 특정 Slack 스레드의 리뷰 요청 처리
```

---

## Slack 스레드 답글

리뷰 완료 후 요청 스레드에 답글:

```
Tool: mcp__slack__conversations_add_message
Parameters:
  - channelId: {roles.developer.slack.channelId}
  - text: "리뷰 완료했습니다\n{리뷰 요약}\n\n{attribution.text}"
  - thread_ts: "{original_message_ts}"  # 스레드 답글
```

---

## Attribution

모든 리뷰 코멘트, Slack 메시지에 다음 attribution 추가:

```
Written with Claude Code
```

config.json의 `attribution.enabled`가 `false`인 경우 생략합니다.

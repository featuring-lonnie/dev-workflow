---
description: PR 리뷰 수행. 3개 병렬 트랙 (보안 전문 + 성능 전문 + 3자 합의) 리뷰 후 P5 포맷 통합.
---

# /dev-workflow:pr-review - PR 리뷰 수행

## 개요

Slack의 리뷰 요청 메시지에서 PR 링크를 확인하고, 로컬에서 코드를 체크아웃하여 리뷰를 수행합니다. **3개 병렬 리뷰 트랙**으로 보안, 성능, 코드 품질을 동시에 분석하고 결과를 통합합니다.

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

**Config에서 cloudId 로드**: `~/.claude/workflow/config.json` → `integrations.jira.cloudId`

```
Tool: mcp__atlassian__getJiraIssue
Parameters:
  - cloudId: "{config.integrations.jira.cloudId}"
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
  - cloudId: "{config.integrations.confluence.cloudId}"
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

### 7. 3개 병렬 리뷰 트랙 실행 (CRITICAL)

**CRITICAL: 3개 리뷰 트랙을 병렬로 실행하여 보안, 성능, 코드 품질을 동시에 분석합니다.**

```
Tool: Task (3개 병렬)
```

#### Track 1: Security Sentinel (보안 전문 리뷰)

에이전트 정의: [security-sentinel](../agents/security-sentinel.md) 참조

```
subagent_type: "oh-my-claudecode:security-reviewer"
model: sonnet
prompt: |
  PR #{pr_number}의 보안 리뷰를 수행해주세요.

  에이전트 역할: security-sentinel (../agents/security-sentinel.md 참조)

  프로젝트 경로: {projectPath}
  변경된 파일: {changed_files}
  Diff: {diff_content}

  OWASP Top 10 기반으로 다음을 검사해주세요:
  1. 인증/인가 취약점
  2. 인젝션 (SQL, XSS, Command, SSRF)
  3. 민감 데이터 노출 (하드코딩 시크릿, 로그 노출)
  4. 입력 검증
  5. 설정 보안 (CORS, CSRF, Rate Limiting)

  결과를 P1-P5 등급으로 분류하여 보고해주세요.
```

#### Track 2: Performance Oracle (성능 전문 리뷰)

에이전트 정의: [performance-oracle](../agents/performance-oracle.md) 참조

```
subagent_type: "oh-my-claudecode:code-reviewer"
model: sonnet
prompt: |
  PR #{pr_number}의 성능 리뷰를 수행해주세요.

  에이전트 역할: performance-oracle (../agents/performance-oracle.md 참조)

  프로젝트 경로: {projectPath}
  변경된 파일: {changed_files}
  Diff: {diff_content}

  다음을 검사해주세요:
  1. N+1 쿼리
  2. 메모리 누수 패턴
  3. 불필요한 연산/호출
  4. 캐싱 전략
  5. 알고리즘 복잡도
  6. DB 인덱스 필요성

  결과를 P1-P5 등급으로 분류하여 보고해주세요.
```

#### Track 3: 3자 합의 리뷰 (기존 로직 유지)

```
subagent_type: "oh-my-claudecode:code-reviewer"
model: sonnet
prompt: |
  PR #{pr_number}의 코드 품질 리뷰를 3가지 관점에서 수행해주세요.

  프로젝트 경로: {projectPath}
  변경된 파일: {changed_files}
  Diff: {diff_content}
  Jira 티켓: {ticket_id} - {ticket_title}
  인수 조건: {acceptance_criteria}

  **3가지 관점에서 독립적으로 분석:**

  1. **리더급 관점**: 아키텍처, 비즈니스 임팩트
     - 설계 방향성, 확장성, 기술 부채
     - 비즈니스 요구사항 일치 여부
     - 장애 발생 시 영향 범위

  2. **시니어급 관점**: 구현 품질, 유지보수성
     - 코드 구조, 패턴 일관성
     - 에러 핸들링, 보안
     - 성능 이슈

  3. **주니어급 관점**: 가독성, 학습 관점
     - 코드 이해도, 문서화
     - 네이밍, 컨벤션 준수
     - 중복 코드

  3자 의견 종합하여 P1-P5로 분류해주세요.
```

### 8. 리뷰 결과 통합 (CRITICAL)

**3개 트랙의 결과를 통합하고 중복을 제거합니다.**

#### 8-1. 중복 제거 규칙

동일 파일/라인에 대한 발견사항이 여러 트랙에서 보고된 경우:
- **더 높은 심각도**를 채택
- **더 구체적인 설명**을 사용
- **출처 트랙**을 명시

#### 8-2. 보호 아티팩트 규칙

**다음 경로의 파일에 대한 삭제 권고는 자동으로 무시합니다:**
- `docs/plans/` — 계획 문서 (보호)
- `docs/solutions/` — 솔루션 문서 (보호)
- `docs/brainstorms/` — 브레인스토밍 문서 (보호)

### 9. P5 리뷰 포맷 적용

**통합된 리뷰 결과를 P5 포맷으로 정리합니다:**

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

### 리뷰 트랙 요약
| 트랙 | 판정 | P1 | P2 | P3+ |
|------|------|----|----|-----|
| Security Sentinel | {PASS/FAIL} | {n} | {n} | {n} |
| Performance Oracle | {PASS/WARN/FAIL} | {n} | {n} | {n} |
| 3자 합의 리뷰 | {PASS/FAIL} | {n} | {n} | {n} |

---

### P1 - Blocker
> 반드시 수정 필요

#### 1. [{track}] {파일명}:{라인}
```{language}
{코드 스니펫}
```
**문제**: {설명}
**제안**: {해결 방안}

---

### P2 - Major
> 머지 전 수정 권장

#### 1. [{track}] {파일명}:{라인}
**문제**: {설명}
**제안**: {해결 방안}

---

### P3 - Medium
> 수정 필요 (후속 PR 가능)

- [{track}] `{파일}:{라인}`: {내용}

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

{config.attribution.text}
```

### 10. 머지 차단 로직 (강화)

| 조건 | 판정 | 액션 |
|------|------|------|
| P1 1건 이상 | **REQUEST_CHANGES** | 반드시 수정 후 재리뷰 |
| P1 없음 + P2 3건 이상 | **REQUEST_CHANGES** | 수정 후 재리뷰 권장 |
| P1 없음 + P2 1-2건 | **COMMENT** | 수정 권장, 머지 가능 |
| P2 없음 | **APPROVE** | 머지 가능 |

### 11. 리뷰 코멘트 작성

```bash
# 리뷰 승인
gh pr review {pr_number} --approve --body "$(cat <<'EOF'
LGTM!

## 확인 사항
- [x] 기능 동작 확인
- [x] 코드 품질 양호
- [x] 보안 리뷰 통과
- [x] 성능 리뷰 통과

{config.attribution.text}
EOF
)"

# 변경 요청
gh pr review {pr_number} --request-changes --body "$(cat <<'EOF'
## 변경 요청 사항

### 1. {파일명}:{라인}
{설명}

### 2. {파일명}:{라인}
{설명}

{config.attribution.text}
EOF
)"

# 코멘트만 (승인/거절 없이)
gh pr review {pr_number} --comment --body "$(cat <<'EOF'
## 리뷰 코멘트

{코멘트 내용}

{config.attribution.text}
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

리뷰 트랙:
  1. Security Sentinel (보안 전문)
  2. Performance Oracle (성능 전문)
  3. 3자 합의 리뷰 (코드 품질)

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

트랙별 결과:
  Security: {PASS/FAIL} (P1: {n}, P2: {n})
  Performance: {PASS/WARN/FAIL} (P1: {n}, P2: {n})
  Code Quality: {PASS/FAIL} (P1: {n}, P2: {n})

통합: P1={n} P2={n} P3={n} P4={n} P5={n}

{리뷰 요약}

{config.attribution.text}
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
  - channel_id: {roles.developer.slack.channelId}
  - content_type: text/plain
  - payload: "리뷰 완료했습니다\n{리뷰 요약}\n\n{attribution.text}"
  - thread_ts: "{original_message_ts}"  # 스레드 답글
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
  --title "[Bug] pr-review: {에러 요약}" \
  --body "{이슈 본문 - shared/error-handling.md 템플릿 참조}" \
  --label "bug"
```

---

## Attribution

모든 리뷰 코멘트, Slack 메시지에 config의 attribution을 추가합니다:

- `~/.claude/workflow/config.json`의 `attribution.text` 값을 사용합니다 (하드코딩 금지)
- `attribution.enabled`가 `false`인 경우 생략합니다

```
# config.json 예시
"attribution": {
  "text": "🤖 Written with Claude Code"
}
```

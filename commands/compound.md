---
description: 지식 축적. 해결한 문제를 5개 병렬 에이전트로 분석하여 docs/solutions/에 솔루션 문서 생성.
---

# /dev-workflow:compound - 지식 축적

## 개요

해결한 문제의 맥락, 원인, 해결 방법을 체계적으로 분석하여 `docs/solutions/`에 솔루션 문서로 저장합니다. 축적된 솔루션은 `/doc` 실행 시 `learnings-researcher` 에이전트가 자동 참조하여 **지식의 복리 효과**를 실현합니다.

**자동 트리거 키워드**: "해결됐다", "고쳤다", "원인 찾았다", "it's fixed", "that worked"

---

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

### 1. 컨텍스트 수집

```bash
# 현재 브랜치 → 티켓 ID 추출
git branch --show-current

# 최근 커밋 이력
git log --oneline -20

# 변경사항 요약
git diff origin/develop...HEAD --stat

# 상세 변경 내용
git diff origin/develop...HEAD
```

### 2. Jira 티켓 정보 조회 (선택)

`--no-jira` 플래그가 없고, 티켓 ID가 추출된 경우:

**Config에서 cloudId 로드**: `~/.claude/workflow/config.json` → `integrations.jira.cloudId`

```
Tool: mcp__atlassian__getJiraIssue
Parameters:
  - cloudId: "{config.integrations.jira.cloudId}"
  - issueIdOrKey: "{ticket_id}"
```

### 3. 5개 병렬 에이전트 실행 (CRITICAL)

**CRITICAL: 모든 에이전트는 텍스트만 반환합니다. 파일 생성은 금지됩니다.**

**5개 에이전트를 병렬로 실행합니다:**

```
Tool: Task (5개 병렬)
```

#### Agent 1: Context Analyzer (컨텍스트 분석)
```
subagent_type: "oh-my-claudecode:explore"
model: haiku
prompt: |
  {projectPath}에서 다음 변경사항의 문제 타임라인을 재구성해주세요.

  브랜치: {branch_name}
  티켓: {ticket_id} - {ticket_title}
  커밋 로그: {git_log}

  분석해야 할 것:
  1. 문제가 언제 처음 발생/발견되었는가?
  2. 문제의 증상은 무엇이었는가?
  3. 어떤 순서로 조사가 진행되었는가?
  4. 최종 해결까지의 과정

  타임라인 형식으로 보고해주세요.
```

#### Agent 2: Solution Extractor (솔루션 추출)
```
subagent_type: "oh-my-claudecode:explore"
model: haiku
prompt: |
  {projectPath}에서 다음 변경사항의 근본 원인과 해결 방법을 분석해주세요.

  변경 파일: {changed_files}
  Diff: {diff_content}

  분석해야 할 것:
  1. 근본 원인 (root cause) - 한줄 요약 + 상세 설명
  2. 해결 방법 - 어떤 코드가 어떻게 변경되었는가
  3. 변경 전/후 코드 비교
  4. 왜 이 해결 방법이 적절한가
```

#### Agent 3: Related Docs Finder (관련 문서 탐색)
```
subagent_type: "oh-my-claudecode:explore"
model: haiku
prompt: |
  {projectPath}/docs/solutions/ 에서 현재 문제와 관련된 기존 솔루션 문서를 찾아주세요.

  현재 문제: {ticket_title}
  키워드: {scope_keywords}

  또한 다음에서 관련 문서를 탐색해주세요:
  - docs/plans/ (관련 계획 문서)
  - docs/brainstorms/ (관련 브레인스토밍 문서)

  관련 문서 목록과 각 문서와의 연관성을 보고해주세요.
```

#### Agent 4: Prevention Strategist (재발 방지)
```
subagent_type: "oh-my-claudecode:explore"
model: haiku
prompt: |
  다음 변경사항을 분석하여 재발 방지 전략을 제안해주세요.

  변경 파일: {changed_files}
  Diff: {diff_content}

  분석해야 할 것:
  1. 이 문제가 재발할 수 있는 시나리오
  2. 재발 방지를 위한 코드/프로세스 개선안
  3. 추가할 테스트 케이스
  4. 모니터링/알림 설정 제안
  5. 재발 위험도 (low / medium / high)
```

#### Agent 5: Category Classifier (분류)
```
subagent_type: "oh-my-claudecode:explore"
model: haiku
prompt: |
  다음 변경사항의 YAML frontmatter 값을 결정해주세요.

  티켓: {ticket_id} - {ticket_title}
  변경 파일: {changed_files}
  Diff 요약: {diff_stat}

  결정해야 할 값:
  1. category: bug | performance | security | architecture | integration | configuration | devops
  2. severity: low | medium | high | critical
  3. tags: 검색용 태그 3-7개 (영문 소문자)
  4. root_cause: 근본 원인 한줄 요약
  5. recurrence_risk: low | medium | high

  각 값에 대한 선택 근거를 포함해주세요.
```

### 4. 결과 통합 및 문서 생성

5개 에이전트의 결과를 통합하여 솔루션 문서를 생성합니다.

**문서 스키마**: [솔루션 문서 스키마](../shared/solution-schema.md) 참조

**디렉토리 확인:**
```bash
# docs/solutions/ 디렉토리가 없으면 생성
mkdir -p {projectPath}/docs/solutions
```

**파일 생성:**
```
파일명: docs/solutions/{today}-{category}-{descriptive-slug}.md
```

**문서 구조:**
```markdown
---
title: {Agent 5의 분류 기반}
date: {today}
category: {Agent 5}
tags: {Agent 5}
jira_ticket: {ticket_id 또는 null}
severity: {Agent 5}
project: {config.projects[].name}
root_cause: {Agent 2 + Agent 5}
recurrence_risk: {Agent 4 + Agent 5}
---

# {title}

## 문제 상황
{Agent 1의 타임라인 기반}

## 타임라인
{Agent 1}

## 근본 원인
{Agent 2}

## 해결 방법
{Agent 2}

## 재발 방지
{Agent 4}

## 관련 자료
{Agent 3}
- Jira: [{ticket_id}]({jira_url})

---

{config.attribution.text}
```

### 5. Jira 티켓에 솔루션 코멘트 추가 (선택)

`--no-jira` 플래그가 없고, 티켓 ID가 있는 경우:

```
Tool: mcp__atlassian__addCommentToJiraIssue
Parameters:
  - cloudId: "{config.integrations.jira.cloudId}"
  - issueIdOrKey: "{ticket_id}"
  - body: "## 솔루션 문서 생성됨\n\n- **근본 원인**: {root_cause}\n- **카테고리**: {category}\n- **재발 위험도**: {recurrence_risk}\n\n상세: `docs/solutions/{filename}`\n\n{config.attribution.text}"
```

### 6. 완료 요약

```
지식 축적 완료

솔루션 문서:
  파일: docs/solutions/{filename}
  카테고리: {category}
  심각도: {severity}

근본 원인: {root_cause}
재발 위험도: {recurrence_risk}

Jira 연동: {ticket_id에 코멘트 추가됨 / 스킵}

복리 효과:
  총 축적 솔루션: {total_solutions}건
  이 솔루션은 향후 /doc 실행 시 자동 참조됩니다.
```

---

## 옵션

| 옵션 | 설명 | 기본값 |
|------|------|--------|
| `--category {value}` | 카테고리 직접 지정 | 자동 분류 |
| `--ticket {PROJ-NNN}` | 티켓 ID 직접 지정 | 브랜치에서 추출 |
| `--no-jira` | Jira 코멘트 추가 스킵 | false |
| `--quick` | Agent 3,4 스킵 (빠른 기록) | false |

---

## 사용 예시

```
/dev-workflow:compound
# 현재 브랜치의 변경사항 분석 후 솔루션 문서 생성

/dev-workflow:compound --ticket BE-123
# 특정 티켓으로 솔루션 생성

/dev-workflow:compound --quick
# 빠른 기록 (핵심만 추출)

/dev-workflow:compound --no-jira
# Jira 코멘트 없이 로컬 문서만 생성

/dev-workflow:compound --category security
# 카테고리 직접 지정
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
  --title "[Bug] compound: {에러 요약}" \
  --body "{이슈 본문 - shared/error-handling.md 템플릿 참조}" \
  --label "bug"
```

---

## Attribution

솔루션 문서와 Jira 코멘트에 config의 attribution을 추가합니다:

- `~/.claude/workflow/config.json`의 `attribution.text` 값을 사용합니다 (하드코딩 금지)
- `attribution.enabled`가 `false`인 경우 생략합니다

```
# config.json 예시
"attribution": {
  "text": "🤖 Written with Claude Code"
}
```

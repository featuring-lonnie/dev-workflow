---
description: 구조화된 아이디어 정제. 5단계 질문으로 문제를 정의하고 대안을 비교하여 docs/brainstorms/에 문서 생성.
---

# /dev-workflow:brainstorm - 구조화된 아이디어 정제

## 개요

새로운 기능이나 문제 해결을 위한 아이디어를 5단계 구조화된 질문으로 정제합니다. 결과는 `docs/brainstorms/`에 저장되며, `/doc` 실행 시 자동으로 참조됩니다.

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

### 1. 입력 파싱

- 티켓 ID: 인자 또는 브랜치에서 추출
- 주제: 인자로 직접 전달 또는 티켓 제목 사용

Jira 티켓이 있는 경우 정보 조회:

**Config에서 cloudId 로드**: `~/.claude/workflow/config.json` → `integrations.jira.cloudId`

```
Tool: mcp__atlassian__getJiraIssue
Parameters:
  - cloudId: "{config.integrations.jira.cloudId}"
  - issueIdOrKey: "{ticket_id}"
```

### 2. 5단계 구조화 질문

**AskUserQuestion 도구를 사용하여 각 단계의 정보를 수집합니다.**

#### Stage 1: 문제 정의 (What & Why)

```
Tool: AskUserQuestion
Question: "해결하려는 문제가 무엇인가요?"
Options:
  - "버그/장애" - 기존 기능이 정상 동작하지 않음
  - "새 기능" - 아직 없는 기능을 추가
  - "개선" - 기존 기능의 성능/UX/구조 개선
  - "기술 부채" - 코드 품질/아키텍처 개선
```

이어서 상세 질문:
- 왜 이 문제를 해결해야 하는가? (비즈니스 가치)
- 해결하지 않으면 어떤 영향이 있는가?
- 누가 영향을 받는가? (사용자, 개발자, 운영팀)

#### Stage 2: 범위 및 제약 (Scope & Constraints)

```
Tool: AskUserQuestion
Question: "작업 범위는 어느 정도인가요?"
Options:
  - "소규모" - 단일 파일/함수 수정
  - "중규모" - 여러 파일, 단일 모듈
  - "대규모" - 여러 모듈, 시스템 전반
  - "불확실" - 탐색 후 결정 필요
```

추가 확인:
- 시간 제약이 있는가?
- 호환성 제약이 있는가? (기존 API, DB 스키마)
- 기술 제약이 있는가? (특정 프레임워크, 라이브러리)

#### Stage 3: 기존 코드/시스템 탐색

**자동으로 관련 코드와 기존 지식을 탐색합니다:**

```
Tool: Task (2개 병렬)
```

**A. 코드베이스 탐색:**
```
subagent_type: "oh-my-claudecode:explore"
model: haiku
prompt: |
  {projectPath}에서 "{topic}"과 관련된 기존 코드를 탐색해주세요.
  관련 파일, 현재 구현 방식, 확장 포인트를 보고해주세요.
```

**B. 기존 솔루션/문서 검색:**

Confluence 검색:
```
Tool: mcp__atlassian__searchConfluenceUsingCql
Parameters:
  - cloudId: "{config.integrations.confluence.cloudId}"
  - cql: "text ~ '{topic_keywords}' and type = page"
```

로컬 솔루션 검색:
```
Tool: Grep
Pattern: "{topic_keywords}"
Path: "{projectPath}/docs/solutions/"
```

#### Stage 4: 대안 탐색

탐색 결과를 기반으로 **최소 3개의 대안**을 도출하고 비교표를 생성합니다:

```markdown
| 기준 | 방법 A: {name} | 방법 B: {name} | 방법 C: {name} |
|------|---------------|---------------|---------------|
| 구현 복잡도 | {low/med/high} | {low/med/high} | {low/med/high} |
| 소요 시간 | {estimate} | {estimate} | {estimate} |
| 유지보수성 | {low/med/high} | {low/med/high} | {low/med/high} |
| 성능 영향 | {description} | {description} | {description} |
| 리스크 | {description} | {description} | {description} |
| 기존 코드 호환 | {yes/partial/no} | {yes/partial/no} | {yes/partial/no} |
```

**사용자에게 선호 대안 확인:**

```
Tool: AskUserQuestion
Question: "어떤 접근 방법을 선호하시나요?"
Options:
  - "방법 A: {name}" - {한줄 설명}
  - "방법 B: {name}" - {한줄 설명}
  - "방법 C: {name}" - {한줄 설명}
```

#### Stage 5: 성공 기준 (Definition of Done)

```
Tool: AskUserQuestion
Question: "성공 기준을 확인합니다. 필수 항목을 선택해주세요."
Options:
  - "기능 동작" - 요구사항대로 동작
  - "테스트 통과" - 단위/통합 테스트 포함
  - "성능 기준 충족" - 특정 성능 지표 달성
  - "문서화 완료" - Tech Spec 및 API 문서 포함
multiSelect: true
```

### 3. 브레인스토밍 문서 생성

**디렉토리 확인:**
```bash
mkdir -p {projectPath}/docs/brainstorms
```

**파일 생성:**
```
파일명: docs/brainstorms/{today}-{topic-slug}.md
```

**문서 구조:**
```markdown
# {topic} 브레인스토밍

## 메타 정보
| 항목 | 내용 |
|------|------|
| 날짜 | {today} |
| 티켓 | {ticket_id 또는 N/A} |
| 참여자 | {currentUser.name} |
| 상태 | Draft |

## 1. 문제 정의
### What
{Stage 1 결과}

### Why
{비즈니스 가치, 영향}

## 2. 범위 및 제약
{Stage 2 결과}

## 3. 현재 상태 분석
### 기존 코드
{Stage 3-A 결과}

### 관련 문서/솔루션
{Stage 3-B 결과}

## 4. 대안 비교
{Stage 4 비교표}

### 선택한 접근법: {method_name}
**선택 사유**: {reason}

## 5. 성공 기준 (Definition of Done)
{Stage 5 결과}
- [ ] {criterion_1}
- [ ] {criterion_2}
- [ ] {criterion_3}

## 다음 단계
1. `/dev-workflow:doc {ticket_id}` — 이 브레인스토밍을 기반으로 Tech Spec 작성
2. `/dev-workflow:deepen` — 특정 영역 심화 분석 (선택)

---

{config.attribution.text}
```

### 4. 완료 요약

```
브레인스토밍 완료

문서: docs/brainstorms/{filename}
주제: {topic}
선택한 접근법: {method_name}

다음 단계:
  1. /dev-workflow:doc {ticket_id} — Tech Spec 작성
  2. /dev-workflow:deepen — 특정 영역 심화 (선택)
```

---

## 사용 예시

```
/dev-workflow:brainstorm BE-123
# Jira 티켓 기반 브레인스토밍

/dev-workflow:brainstorm HTTP-only cookie 인증 방식 변경
# 주제로 직접 브레인스토밍

/dev-workflow:brainstorm
# 현재 브랜치 티켓 기반 브레인스토밍
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
  --title "[Bug] brainstorm: {에러 요약}" \
  --body "{이슈 본문 - shared/error-handling.md 템플릿 참조}" \
  --label "bug"
```

---

## Attribution

브레인스토밍 문서에 config의 attribution을 추가합니다:

- `~/.claude/workflow/config.json`의 `attribution.text` 값을 사용합니다 (하드코딩 금지)
- `attribution.enabled`가 `false`인 경우 생략합니다

```
# config.json 예시
"attribution": {
  "text": "🤖 Written with Claude Code"
}
```

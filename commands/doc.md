---
description: 기술 문서 작성. 병렬 리서치 + Confluence Tech Spec 생성 + 로컬 계획 파일 출력 + Jira 티켓 연동.
---

# /dev-workflow:doc - 기술 문서 작성

## Tech Spec 템플릿

```markdown
# [PROJ-123] {기능명} Tech Spec

## 개요
| 항목 | 내용 |
|------|------|
| Jira | [PROJ-123](https://company.atlassian.net/browse/PROJ-123) |
| 작성자 | {currentUser.name} |
| 작성일 | 2026-02-04 |
| 상태 | Draft / In Review / Approved |

## 목적
이 문서는 {기능명}의 기술적 설계를 정의합니다.

## 배경
- 왜 이 기능이 필요한가?
- 현재 상황과 문제점

## 기술적 설계

### 아키텍처
```
[Diagram or description]
```

### API 설계
| Method | Endpoint | Description |
|--------|----------|-------------|
| POST | /api/v1/... | ... |

### 데이터 모델
```sql
-- 테이블 스키마
```

### 시퀀스 다이어그램
```
User -> API -> Service -> DB
```

## 구현 계획
1. **Phase 1**: 기본 구조
2. **Phase 2**: 핵심 로직
3. **Phase 3**: 테스트 및 최적화

## 테스트 계획
- 단위 테스트:
- 통합 테스트:
- E2E 테스트:

## 롤백 계획
- 롤백 조건:
- 롤백 방법:

## 참고 자료
- 관련 문서 링크
- 외부 레퍼런스
```

## 실행 단계

### 0. Config 로드
**먼저 `~/.claude/workflow/config.json`에서 필요한 설정을 읽습니다:**
- `integrations.jira.cloudId` (UUID)
- `integrations.confluence.cloudId` (UUID)
- `roles.developer.confluence.techSpecSpaceKey`

### 0-1. 상세 수준 선택

```
Tool: AskUserQuestion
Question: "리서치 상세 수준을 선택해주세요."
Options:
  - "MINIMAL" - 코드베이스 탐색만 + 기본 Tech Spec
  - "MORE (권장)" - 코드베이스 + 과거 솔루션 학습 리서치
  - "A LOT" - 전체 리서치 (프레임워크 문서 + 업계 모범 사례 포함)
```

**`--detail` 옵션으로 직접 지정 가능:**
- `--detail minimal` → MINIMAL
- `--detail more` → MORE (기본값, 미지정 시)
- `--detail alot` → A LOT

### 1. Jira 티켓 정보 조회
```
Tool: mcp__atlassian__getJiraIssue
Parameters:
  - cloudId: "{config.integrations.jira.cloudId}"
  - issueIdOrKey: "PROJ-123"
```

### 1-0. 티켓 명확성 검증 (CRITICAL)

**CRITICAL: 코드베이스 탐색 전에 티켓의 작업 범위가 충분히 명확한지 검증합니다.**

티켓 내용이 불명확하면 코드베이스 탐색 방향도 불명확해지고, 결국 Tech Spec도 부정확해집니다.

#### 검증 체크리스트

| 항목 | 확인 내용 | 불명확 시 조치 |
|------|----------|---------------|
| **작업 목표** | 무엇을 달성해야 하는가? | 사용자에게 명확화 요청 |
| **변경 범위** | 어떤 시스템/컴포넌트가 영향받는가? | 사용자에게 범위 확인 |
| **입출력** | 어떤 데이터가 들어오고 나가는가? | 사용자에게 스펙 확인 |
| **제약 조건** | 성능, 보안, 호환성 요구사항이 있는가? | 사용자에게 제약 확인 |

#### 명확성 판단 기준

**명확한 티켓 예시:**
```
제목: [BE] access_token을 http-only cookie로 변경
내용:
- 목표: XSS 공격 방어를 위해 토큰을 cookie로 전달
- 범위: 로그인, 토큰 갱신, 로그아웃 API
- 제약: Authorization 헤더 방식 유지 (모바일 호환)
```

**불명확한 티켓 예시:**
```
제목: 인증 개선
내용: 보안을 강화해주세요
```

#### 불명확 시 처리

티켓이 불명확하면 **코드베이스 탐색 전에** 사용자에게 명확화를 요청합니다:

```
Tool: AskUserQuestion
Question: "티켓 내용을 명확히 해주세요"
Options:
  - 작업 목표가 불명확합니다. 구체적인 목표를 알려주세요.
  - 변경 범위가 불명확합니다. 어떤 API/시스템이 영향받나요?
  - 제약 조건이 불명확합니다. 호환성/성능 요구사항이 있나요?
```

명확화가 완료되면 다음 단계로 진행합니다.

### 1-0.5. 브레인스토밍 문서 참조 (자동)

동일 티켓의 브레인스토밍 문서가 있으면 자동으로 참조합니다:

```
Tool: Glob
Pattern: "{projectPath}/docs/brainstorms/*{ticket_id}*"
```

발견 시:
```
Tool: Read
file_path: "{brainstorm_doc_path}"
```

브레인스토밍 문서의 **선택한 접근법**, **성공 기준**, **범위** 정보를 리서치 및 Plan 모드의 컨텍스트로 활용합니다.

### 1-1. Phase R1: 병렬 로컬 리서치 (CRITICAL)

**CRITICAL: Tech Spec 초안 작성 전에 반드시 실제 코드베이스를 탐색하여 정확한 정보를 수집합니다.**

**ALL 레벨 (MINIMAL, MORE, A LOT) 공통:**

```
Tool: Task
subagent_type: "oh-my-claudecode:explore"  # repo-research-analyst 역할
model: sonnet
prompt: |
  {projectPath}에서 {티켓 주제}와 관련된 코드를 탐색해주세요.

  에이전트 역할: repo-research-analyst (../agents/repo-research-analyst.md 참조)

  찾아야 할 것:
  1. 관련 API 엔드포인트 경로와 뷰/컨트롤러
  2. 현재 구현 방식과 사용 중인 라이브러리
  3. 관련 데이터 모델/스키마
  4. 설정 파일 (settings, config)
  5. 기존 테스트 패턴

  실제 코드에서 확인한 정확한 정보를 보고해주세요.
```

**MORE 이상 레벨 (추가 병렬 실행):**

```
Tool: Task
subagent_type: "oh-my-claudecode:explore"  # learnings-researcher 역할
model: haiku
prompt: |
  {projectPath}/docs/solutions/ 에서 {티켓 주제}와 관련된 과거 솔루션을 탐색해주세요.

  에이전트 역할: learnings-researcher (../agents/learnings-researcher.md 참조)

  키워드: {scope_keywords}
  카테고리: {expected_category}

  관련 솔루션의 핵심 학습과 적용 가능한 패턴을 보고해주세요.
```

#### 탐색 결과 검증

- [ ] API 엔드포인트 경로가 실제 코드와 일치하는가?
- [ ] 요청/응답 필드명이 정확한가?
- [ ] 사용 중인 라이브러리/프레임워크가 맞는가?
- [ ] 기존 설정 구조를 파악했는가?

**이 단계를 건너뛰면 안 됩니다.** 코드베이스 탐색 없이 작성한 Tech Spec은 실제 구현과 맞지 않아 수정이 필요하게 됩니다.

### 1-2. Phase R2: 조건부 외부 리서치 (A LOT 레벨만)

**A LOT 레벨 선택 시에만 실행합니다. 2개 에이전트를 병렬 실행:**

```
Tool: Task (2개 병렬)
```

**A. framework-docs-researcher:**
```
subagent_type: "oh-my-claudecode:researcher"  # framework-docs-researcher 역할
model: sonnet
prompt: |
  다음 프로젝트에서 사용하는 프레임워크/라이브러리의 공식 문서를 조사해주세요.

  에이전트 역할: framework-docs-researcher (../agents/framework-docs-researcher.md 참조)

  프레임워크: {detected_framework} {version}
  주제: {ticket_topic}

  조사해야 할 것:
  1. 관련 API의 정확한 사양
  2. 공식 권장 패턴
  3. 주의사항 및 deprecated API
```

**B. best-practices-researcher:**
```
subagent_type: "oh-my-claudecode:researcher"  # best-practices-researcher 역할
model: haiku
prompt: |
  다음 주제의 업계 모범 사례를 조사해주세요.

  에이전트 역할: best-practices-researcher (../agents/best-practices-researcher.md 참조)

  주제: {ticket_topic}
  맥락: {project_context}

  조사해야 할 것:
  1. 업계 표준 접근법
  2. 보안 가이드라인
  3. 안티 패턴
```

### 1-3. Phase R3: 리서치 통합

모든 리서치 에이전트의 결과를 통합합니다:

```markdown
## 리서치 요약

### 코드베이스 분석 (repo-research-analyst)
{R1 결과}

### 과거 솔루션 참조 (learnings-researcher) — MORE 이상
{R1 추가 결과}

### 프레임워크 공식 문서 (framework-docs-researcher) — A LOT만
{R2-A 결과}

### 업계 모범 사례 (best-practices-researcher) — A LOT만
{R2-B 결과}

### 브레인스토밍 참조 — 해당 시
{brainstorm 문서 핵심 내용}
```

이 통합된 리서치 결과를 Plan 모드의 컨텍스트로 전달합니다.

### 2. Plan 모드 진입 - Tech Spec 내용 논의

**CRITICAL: 바로 문서를 생성하지 않고, Plan 모드로 진입하여 사용자와 내용을 논의합니다.**

```
Tool: EnterPlanMode
```

#### 2-1. SDD (Spec-Driven Development) 옵션 확인

**사용자에게 spec-kit 사용 여부를 먼저 확인합니다:**

```
Tool: AskUserQuestion
Question: "spec-kit을 사용하여 SDD 방식으로 진행할까요?"
Options:
  - "SDD 적용 (권장)" - spec-kit으로 specification 먼저 작성 후 Tech Spec 생성
  - "일반 방식" - 바로 Tech Spec 내용 논의
```

**SDD 적용 시 워크플로우:**

1. `/speckit.constitution` - 프로젝트 기본 원칙 확인/생성 (최초 1회)
2. `/speckit.specify` - 기능 명세 작성
3. `/speckit.plan` - 기술 계획 작성
4. 위 내용을 기반으로 Tech Spec 문서 생성

**SDD 장점:**
- 구조화된 specification으로 명확한 요구사항 정의
- 일관된 아키텍처 원칙 적용
- 구현 전 충분한 설계 검토

#### 2-2. Tech Spec 내용 논의

**리서치 결과를 기반으로** Plan 모드에서 다음 사항을 사용자와 논의:

1. **기술적 설계 범위** (탐색 결과 반영)
   - 어떤 API/시스템이 변경되는가? → **탐색에서 확인한 실제 엔드포인트 사용**
   - 아키텍처 다이어그램이 필요한가?
   - 데이터 모델 변경이 있는가? → **탐색에서 확인한 실제 모델 구조 참조**

2. **구현 계획**
   - Phase별 작업 분류
   - 우선순위 및 의존성

3. **테스트 계획**
   - 단위/통합/E2E 테스트 범위
   - 테스트 시나리오

4. **롤백 계획**
   - 롤백 조건
   - 롤백 방법

**AskUserQuestion 도구를 사용하여 각 섹션별로 사용자 의견을 수집합니다.**

논의가 완료되면 Plan 파일에 최종 Tech Spec 내용을 작성하고 `ExitPlanMode`로 사용자 승인을 요청합니다.

### 3. 승인 후 Confluence 페이지 생성 + 로컬 계획 파일 출력

**사용자가 Plan을 승인한 후에만 실행합니다.**

#### 3-0. Atlassian 인증 확인

**CRITICAL: API 호출 전에 인증 상태를 먼저 확인합니다.**

```
Tool: mcp__atlassian__atlassianUserInfo
```

인증 오류(401) 발생 시:
- 사용자에게 `/mcp` 명령어로 Atlassian 재인증 요청
- 재인증 완료 후 다음 단계 진행

#### 3-1. spaceId 조회

**CRITICAL: createConfluencePage API는 spaceKey가 아닌 spaceId를 요구합니다.**

config.json의 `techSpecSpaceKey`로 실제 `spaceId`를 조회합니다:

```
Tool: mcp__atlassian__getConfluenceSpaces
Parameters:
  - cloudId: "{config.integrations.confluence.cloudId}"
  - keys: "{roles.developer.confluence.techSpecSpaceKey}"
```

응답에서 `results[0].id`가 spaceId입니다.

#### 3-2. Confluence 페이지 생성

```
Tool: mcp__atlassian__createConfluencePage
Parameters:
  - cloudId: "{config.integrations.confluence.cloudId}"
  - spaceId: "{조회한 spaceId}"  # spaceKey가 아님!
  - title: "[{ticket_id}] {기능명} Tech Spec"
  - contentFormat: "markdown"  # 마크다운 형식 사용
  - body: "{plan_mode에서_확정된_내용}"
```

**설정 참조**: `~/.claude/workflow/config.json` → `roles.developer.confluence.*`

#### 3-3. 로컬 계획 파일 출력 (NEW)

**`/work` 커맨드가 소비할 수 있도록 로컬 계획 파일도 생성합니다.**

```bash
mkdir -p {projectPath}/docs/plans
```

```
파일명: docs/plans/{ticket_id}-plan.md
```

**로컬 계획 파일은 Tech Spec과 동일한 내용이지만 다음이 추가됩니다:**

```markdown
---
ticket: {ticket_id}
confluence_url: {confluence_page_url}
created: {today}
detail_level: {MINIMAL | MORE | A LOT}
---

{Tech Spec 전체 내용}

---

## 리서치 참조 (로컬 전용)

{Phase R1-R3 리서치 통합 결과}
```

### 4. Jira 티켓 본문에 문서 링크 추가

**CRITICAL: description 필드는 마크다운 문자열로 전달합니다 (ADF 형식 사용 X).**

```
Tool: mcp__atlassian__editJiraIssue
Parameters:
  - cloudId: "{config.integrations.jira.cloudId}"
  - issueIdOrKey: "PROJ-123"
  - fields: {
      "description": "{기존 description 마크다운}\n\n---\n\n## 관련 문서\n\n* [Tech Spec]({confluence_page_url})\n\n---\n\nUpdated with Claude Code"
    }
```

**주의사항:**
- `fields.description`에 마크다운 문자열 직접 전달
- ADF(Atlassian Document Format) JSON 형식은 오류 발생 가능
- 기존 description 내용 보존 필수

**구현 방법:**
1. `mcp__atlassian__getJiraIssue`로 기존 description 조회
2. 기존 description에 문서 링크 섹션 추가 (마크다운 형식)
3. `mcp__atlassian__editJiraIssue`로 업데이트

**주의사항:**
- 기존 description을 덮어쓰지 않도록 반드시 먼저 조회 필요
- 이미 "관련 문서" 섹션이 있으면 해당 섹션에 추가
- 마크다운 링크 형식 사용: `[링크 텍스트](URL)`

### 5. 완료 요약

```
Tech Spec 생성 완료

문서: [PROJ-123] 회원가입 API Tech Spec
   Confluence: https://company.atlassian.net/wiki/...
   로컬 계획: docs/plans/PROJ-123-plan.md

리서치 수준: {MINIMAL | MORE | A LOT}
   코드베이스 분석: 완료
   과거 솔루션 참조: {완료 / 스킵 (MINIMAL)}
   프레임워크 문서: {완료 / 스킵 (MINIMAL, MORE)}
   업계 모범 사례: {완료 / 스킵 (MINIMAL, MORE)}

Jira 연동:
   - PROJ-123 티켓 본문에 문서 링크 추가됨

다음 단계:
   1. /dev-workflow:deepen — 특정 영역 심화 (선택)
   2. /dev-workflow:work docs/plans/PROJ-123-plan.md — 계획 실행
   3. 또는 /dev-workflow:lfg PROJ-123 — 전체 자율 파이프라인
```

## 템플릿 옵션

### --template api
API 중심 템플릿 (엔드포인트, 요청/응답 스키마 강조)

### --template batch
배치 작업 템플릿 (스케줄, 처리 로직, 모니터링 강조)

### --template migration
DB 마이그레이션 템플릿 (스키마 변경, 롤백 계획 강조)

## 사용 예시

```
/dev-workflow:doc PROJ-123
# 기본 Tech Spec 생성 (MORE 레벨, Plan 모드로 논의 후 승인 시 생성)

/dev-workflow:doc PROJ-123 --detail alot
# 전체 리서치 (프레임워크 문서 + 업계 모범 사례 포함)

/dev-workflow:doc PROJ-123 --detail minimal
# 최소 리서치 (코드베이스 탐색만)

/dev-workflow:doc PROJ-123 --sdd
# SDD 방식으로 진행 (spec-kit 사용)

/dev-workflow:doc PROJ-123 --template api
# API 템플릿으로 생성

/dev-workflow:doc PROJ-123 --space BACKEND
# 특정 스페이스에 생성
```

## SDD (Spec-Driven Development) 워크플로우

spec-kit이 설치된 경우, SDD 방식으로 Tech Spec을 작성할 수 있습니다.

### SDD 진행 순서

```
1. /speckit.constitution  → 프로젝트 기본 원칙 (최초 1회)
2. /speckit.specify       → 기능 명세 작성
3. /speckit.plan          → 기술 계획 작성
4. /dev-workflow:doc      → Tech Spec 문서 생성 (위 내용 기반)
```

### SDD vs 일반 방식

| 구분 | SDD 방식 | 일반 방식 |
|------|----------|----------|
| 적합한 경우 | 대규모 기능, 신규 프로젝트 | 소규모 변경, 단순 기능 |
| 소요 시간 | 길지만 철저함 | 빠르지만 간략함 |
| 산출물 | Constitution + Spec + Plan + Tech Spec | Tech Spec |
| 일관성 | 높음 (원칙 기반) | 보통 |

## Confluence 스페이스

config.json에서 설정:
```json
{
  "roles": {
    "developer": {
      "confluence": {
        "techSpecSpaceKey": "...",
        "techSpecParentPageId": null
      }
    }
  },
  "integrations": {
    "confluence": {
      "spaces": {
        "backend": { "key": "...", "name": "..." },
        "personal": { "key": "...", "name": "..." }
      }
    }
  }
}
```

## Jira 연동 방식

**티켓 본문 방식 채택 이유:**
- 마크다운 링크 `[text](url)` 형식으로 클릭 가능한 링크 생성
- 티켓 조회 시 관련 문서 바로 확인 가능
- "관련 문서" 섹션으로 구조화된 관리

**구현 시 주의사항:**
- 기존 description 내용 보존 필수
- "관련 문서" 섹션이 이미 있으면 해당 섹션에 추가
- 마크다운 링크 형식 사용: `[Tech Spec](https://...)`

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
  --title "[Bug] doc: {에러 요약}" \
  --body "{이슈 본문 - shared/error-handling.md 템플릿 참조}" \
  --label "bug"
```

---

## Attribution

Confluence 문서와 Jira 코멘트에 config의 attribution을 추가합니다:

- `~/.claude/workflow/config.json`의 `attribution.text` 값을 사용합니다 (하드코딩 금지)
- `attribution.enabled`가 `false`인 경우 생략합니다

```
# config.json 예시
"attribution": {
  "text": "🤖 Written with Claude Code"
}
```

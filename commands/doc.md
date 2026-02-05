---
description: 기술 문서 작성. Confluence에 Tech Spec 생성 후 Jira 티켓 본문에 연동.
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

### 1-1. 코드베이스 탐색 (CRITICAL)

**CRITICAL: Tech Spec 초안 작성 전에 반드시 실제 코드베이스를 탐색하여 정확한 정보를 수집합니다.**

Jira 티켓의 내용만으로는 실제 구현 세부사항을 알 수 없습니다. 잘못된 API 엔드포인트, 데이터 모델, 기존 패턴 등을 문서에 기재하면 Tech Spec의 신뢰성이 떨어집니다.

#### 탐색 대상

Explore 에이전트를 사용하여 다음 항목을 확인합니다:

| 항목 | 탐색 내용 | 예시 |
|------|----------|------|
| **API 엔드포인트** | 실제 URL 경로, HTTP 메서드 | `/users/login/` vs `/api/auth/login` |
| **요청/응답 스키마** | 실제 필드명, 타입, 필수 여부 | `access_token` vs `accessToken` |
| **인증 미들웨어** | 현재 인증 방식, 사용 중인 라이브러리 | `JWTAuthentication`, `simplejwt` |
| **데이터 모델** | 관련 테이블, 필드, 관계 | User, Token 모델 구조 |
| **기존 패턴** | 프로젝트의 코딩 컨벤션, 설정 방식 | settings 구조, Response 포맷 |
| **관련 테스트** | 기존 테스트 케이스, 테스트 패턴 | pytest fixtures, test 구조 |

#### 탐색 방법

```
Tool: Task
Parameters:
  - subagent_type: "Explore"
  - prompt: |
      {프로젝트 경로}에서 {티켓 주제}와 관련된 코드를 탐색해주세요.

      찾아야 할 것:
      1. 관련 API 엔드포인트 경로와 뷰/컨트롤러
      2. 현재 구현 방식과 사용 중인 라이브러리
      3. 관련 데이터 모델/스키마
      4. 설정 파일 (settings, config)
      5. 기존 테스트 패턴

      실제 코드에서 확인한 정확한 정보를 보고해주세요.
```

#### 탐색 결과 검증

- [ ] API 엔드포인트 경로가 실제 코드와 일치하는가?
- [ ] 요청/응답 필드명이 정확한가?
- [ ] 사용 중인 라이브러리/프레임워크가 맞는가?
- [ ] 기존 설정 구조를 파악했는가?

**이 단계를 건너뛰면 안 됩니다.** 코드베이스 탐색 없이 작성한 Tech Spec은 실제 구현과 맞지 않아 수정이 필요하게 됩니다.

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

**코드베이스 탐색 결과를 기반으로** Plan 모드에서 다음 사항을 사용자와 논의:

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

### 3. 승인 후 Confluence 페이지 생성

**사용자가 Plan을 승인한 후에만 실행합니다.**

```
Tool: mcp__atlassian__createConfluencePage
Parameters:
  - cloudId: "{config.integrations.confluence.cloudId}"
  - spaceKey: {roles.developer.confluence.techSpecSpaceKey}
  - title: "[{ticket_id}] {기능명} Tech Spec"
  - parentPageId: {roles.developer.confluence.techSpecParentPageId}
  - body: "{plan_mode에서_확정된_내용}"
```

**설정 참조**: `~/.claude/workflow/config.json` → `roles.developer.confluence.*`

### 4. Jira 티켓 본문에 문서 링크 추가
```
Tool: mcp__atlassian__editJiraIssue
Parameters:
  - cloudId: "{config.integrations.jira.cloudId}"
  - issueIdOrKey: "PROJ-123"
  - description: "{기존 description}\n\n---\n\n## 관련 문서\n\n* [Tech Spec]({confluence_page_url})"
```

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
   URL: https://company.atlassian.net/wiki/...
   스페이스: lonnie's personal space

Jira 연동:
   - PROJ-123 티켓 본문에 문서 링크 추가됨

다음 단계:
   1. 문서 내용 작성/수정
   2. 팀 리뷰
   3. 상태를 "Approved"로 변경
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
# 기본 Tech Spec 생성 (Plan 모드로 논의 후 승인 시 생성)

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

## Attribution

Confluence 문서와 Jira 코멘트에 다음 attribution을 추가합니다:

```
Written with Claude Code
```

config.json의 `attribution.enabled`가 `false`인 경우 생략합니다.

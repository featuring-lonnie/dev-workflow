---
description: 자율 개발 파이프라인. 9단계 자동 실행 (start → doc → work → pr-review → fix → pr → compound → review → complete).
---

# /dev-workflow:lfg - 자율 개발 파이프라인

## 개요

Jira 티켓을 입력받아 **작업 시작부터 PR 생성, 지식 축적까지 전 과정을 자율 실행**하는 파이프라인입니다. 모든 dev-workflow 커맨드를 9단계로 연결하여 하나의 흐름으로 실행합니다.

**상태 관리 참조**: [파이프라인 상태 관리](../shared/pipeline-state.md)

---

## 9단계 파이프라인

```
1. INIT        → /start 로직
2. PLANNING    → /brainstorm(선택) + /doc (병렬 리서치)
3. WORKING     → /work 로직 (태스크 루프 + 테스트 + 자동 커밋)
4. REVIEWING   → /pr-review 로직 (3개 병렬 리뷰 트랙)
5. FIXING      → P1/P2 자동 수정 루프 (최대 3회)
6. PR_CREATING → /pr 로직 (PR 생성 + Jira/Confluence 링크)
7. DOCUMENTING → /compound 로직 (지식 축적)
8. NOTIFYING   → /review 로직 (Slack 리뷰 요청)
9. COMPLETE    → 최종 보고 + 상태 정리
```

---

## 실행 단계

### 0. Config 로드 및 상태 초기화

**Config 로드:**
```
~/.claude/workflow/config.json에서:
- integrations.jira.cloudId
- integrations.confluence.cloudId
- projects 배열 (프로젝트 매핑)
- roles.developer.* (Slack, Confluence, commit 설정)
```

**상태 파일 초기화:**
```
파일: .omc/state/lfg-state.json
스키마: shared/pipeline-state.md 참조
```

```bash
mkdir -p .omc/state
```

**--resume 플래그 확인:**
기존 상태 파일이 있고 `--resume` 플래그가 있으면 중단 지점에서 재개합니다.

### Phase 1: INIT — 작업 시작

**`/dev-workflow:start` 로직을 실행합니다.**

1. 티켓 번호 파싱 (인자 또는 URL)
2. Jira 티켓 조회
3. 프로젝트 매핑 (티켓 키 → localPath)
4. Jira 상태 변경 → In Progress
5. Jira 시작일 업데이트
6. Git 브랜치 생성

```
Tool: mcp__atlassian__getJiraIssue
Parameters:
  - cloudId: "{config.integrations.jira.cloudId}"
  - issueIdOrKey: "{ticket_id}"
```

```
Tool: mcp__atlassian__transitionJiraIssue
Parameters:
  - cloudId: "{config.integrations.jira.cloudId}"
  - issueIdOrKey: "{ticket_id}"
  - transitionId: "{in_progress_id}"
```

```bash
cd {mapped_project_localPath}
git fetch origin
git checkout -b feature/{ticket_id}-{short_description} origin/develop
```

**상태 업데이트:** `INIT → COMPLETED`

### Phase 2: PLANNING — 계획 수립

#### 2-1. Brainstorm (선택적)

**`--skip-brainstorm` 미지정 시 티켓 명확성을 확인합니다:**

티켓이 명확한 경우 (구체적 목표, 범위, 제약 조건 있음) → brainstorm 스킵
티켓이 불명확한 경우 → `/dev-workflow:brainstorm` 로직 실행

#### 2-2. Doc (병렬 리서치 + Tech Spec)

**강화된 `/dev-workflow:doc` 로직을 실행합니다:**

1. 상세 수준: MORE (기본값)
2. Phase R1: 병렬 로컬 리서치 (repo-research-analyst + learnings-researcher)
3. Phase R3: 리서치 통합
4. Plan 모드 진입 → Tech Spec 논의

**`--auto` 미사용 시 → Gate 1:**

```
Tool: AskUserQuestion
Question: "계획이 준비되었습니다. 실행할까요?"
Options:
  - "실행" - 계획대로 WORKING 단계 시작
  - "계획 수정" - Plan 모드로 돌아가 수정
  - "심화 분석" - /deepen 로직으로 특정 영역 심화
  - "중단" - 파이프라인 중단
```

**"심화 분석" 선택 시:** `/dev-workflow:deepen` 로직 실행 후 Gate 1으로 복귀

5. Confluence 페이지 생성
6. 로컬 계획 파일 출력 (`docs/plans/{ticket_id}-plan.md`)
7. Jira 티켓에 문서 링크 추가

**상태 업데이트:** `PLANNING → COMPLETED`, `planDocPath` 기록

### Phase 3: WORKING — 작업 실행

**`/dev-workflow:work` 로직을 실행합니다:**

1. 계획 문서 로드 (Phase 2에서 생성한 `docs/plans/{ticket_id}-plan.md`)
2. 태스크 분해
3. 실행 루프: 태스크 실행 → 테스트 → Phase별 자동 커밋
4. 전체 테스트 실행

**Swarm 모드:** 5개 이상 독립 태스크 시 자동 제안 (또는 `--auto` 시 자동 활성화)

**상태 업데이트:** `WORKING → COMPLETED`, 커밋 목록 기록

### Phase 4: REVIEWING — 코드 리뷰

**강화된 `/dev-workflow:pr-review` 로직을 셀프 리뷰로 실행합니다:**

**`--skip-review` 미지정 시:**

1. 변경사항 수집 (git diff)
2. 3개 병렬 리뷰 트랙 실행:
   - Track 1: Security Sentinel (보안)
   - Track 2: Performance Oracle (성능)
   - Track 3: 3자 합의 리뷰 (코드 품질)
3. 리뷰 결과 통합 (중복 제거 + 보호 아티팩트 규칙 적용)
4. P1-P5 분류

**상태 업데이트:** `REVIEWING → COMPLETED`, 리뷰 결과 기록

### Phase 5: FIXING — 자동 수정

**P1 또는 P2가 있을 경우 자동 수정을 시도합니다. (최대 3라운드)**

```
라운드 1:
  P1/P2 항목 수정 → 테스트 → 재리뷰
  ↓
통과? → Phase 6으로
  ↓ 실패
라운드 2:
  남은 P1/P2 수정 → 테스트 → 재리뷰
  ↓
통과? → Phase 6으로
  ↓ 실패
라운드 3:
  마지막 시도 → 테스트 → 재리뷰
  ↓
통과? → Phase 6으로
  ↓ 실패
PAUSED → 사용자 개입 요청
```

**재리뷰:** 전체 3트랙 재실행이 아닌, 수정된 파일만 대상으로 간이 리뷰

**P1/P2가 없으면 이 Phase는 자동 SKIPPED**

**상태 업데이트:** `FIXING → COMPLETED/SKIPPED`

### Phase 6: PR_CREATING — PR 생성

**`/dev-workflow:pr` 로직을 실행합니다:**

1. PR 생성

```bash
git push -u origin {branch_name}

gh pr create \
  --title "{gitmoji} [{ticket_id}] {ticket_title}" \
  --body "$(cat <<'EOF'
## Summary
{Tech Spec 기반 요약}

## Changes
{변경사항 목록}

## Test Plan
{테스트 계획}

## Related
- Jira: [{ticket_id}]({jira_url})
- Tech Spec: [{title}]({confluence_url})

{config.attribution.text}
EOF
)"
```

2. PR 라벨 추가
3. Jira 티켓에 PR 링크 추가

**`--auto` 미사용 시 → Gate 2:**

```
Tool: AskUserQuestion
Question: "리뷰 통과. PR을 생성할까요?"
Options:
  - "PR 생성" - PR 생성 진행
  - "추가 수정" - WORKING 단계로 돌아가 추가 작업
  - "중단" - 파이프라인 중단
```

**상태 업데이트:** `PR_CREATING → COMPLETED`, PR 번호/URL 기록

### Phase 7: DOCUMENTING — 지식 축적

**`/dev-workflow:compound` 로직을 실행합니다:**

**`--skip-compound` 미지정 시:**

1. 컨텍스트 수집 (git log, git diff)
2. 5개 병렬 에이전트 실행
3. 결과 통합 → `docs/solutions/` 솔루션 문서 생성
4. Jira 티켓에 솔루션 코멘트 추가

**상태 업데이트:** `DOCUMENTING → COMPLETED/SKIPPED`

### Phase 8: NOTIFYING — 리뷰 요청

**`/dev-workflow:review` 로직을 실행합니다:**

**`--no-notify` 미지정 시:**

```
Tool: mcp__slack__conversations_add_message
Parameters:
  - channel_id: {roles.developer.slack.channelId}
  - content_type: text/plain
  - payload: "{mention_group} 리뷰 부탁드립니다\n\nPR: {pr_url}\nJira: {jira_url}\n\n{pr_summary}\n\n{config.attribution.text}"
```

**상태 업데이트:** `NOTIFYING → COMPLETED/SKIPPED`

### Phase 9: COMPLETE — 최종 보고

**최종 보고 출력 및 상태 정리:**

```
LFG 파이프라인 완료

티켓: {ticket_id} - {ticket_title}
브랜치: {branch_name}
소요 시간: {elapsed}

결과:
  1. INIT        ✓ Jira In Progress, 브랜치 생성
  2. PLANNING    ✓ Tech Spec (Confluence + 로컬)
  3. WORKING     ✓ {n}개 태스크, {n}개 커밋
  4. REVIEWING   ✓ 보안/성능/품질 리뷰 통과
  5. FIXING      {✓ 완료 / ⊘ 스킵 (수정 불필요)}
  6. PR_CREATING ✓ PR #{pr_number}
  7. DOCUMENTING {✓ 솔루션 저장 / ⊘ 스킵}
  8. NOTIFYING   {✓ Slack 알림 / ⊘ 스킵}
  9. COMPLETE    ✓

산출물:
  PR: {pr_url}
  Tech Spec: {confluence_url}
  계획: docs/plans/{ticket_id}-plan.md
  솔루션: docs/solutions/{solution_filename}

다음 단계:
  - 팀 리뷰 대기
  - 리뷰 승인 후 /dev-workflow:done 으로 완료 처리
```

**상태 업데이트:** `pipeline.status → COMPLETED`

---

## 옵션

| 옵션 | 설명 | 기본값 |
|------|------|--------|
| `--auto` | 모든 확인 게이트 스킵, 완전 자율 실행 | false |
| `--resume` | 중단된 파이프라인 재개 | false |
| `--skip-brainstorm` | brainstorm 단계 스킵 | 티켓 명확 시 자동 스킵 |
| `--deepen` | PLANNING 후 deepen 실행 | false (기본 스킵) |
| `--skip-review` | pr-review 단계 스킵 | false |
| `--skip-compound` | compound 단계 스킵 | false |
| `--no-notify` | Slack 알림 스킵 | false |
| `--detail {level}` | 리서치 상세 수준 (minimal/more/alot) | more |

---

## 사용자 확인 게이트

### 기본 모드 (2개 게이트)

| 게이트 | 시점 | 선택지 |
|--------|------|--------|
| Gate 1 | PLANNING 완료 후 | 실행 / 계획 수정 / 심화 / 중단 |
| Gate 2 | REVIEWING 완료 후 | PR 생성 / 추가 수정 / 중단 |

### `--auto` 모드

모든 게이트 자동 통과. 완전 자율 실행.

---

## 상태 관리

### 상태 파일

```
.omc/state/lfg-state.json
```

상세 스키마: [파이프라인 상태 관리](../shared/pipeline-state.md) 참조

### 재개 (Resume)

```
/dev-workflow:lfg --resume
```

1. 상태 파일에서 `currentPhase` 확인
2. 해당 Phase부터 재개
3. 이전 Phase의 `output` 데이터 활용

### 에러 시 동작

| 상황 | 동작 |
|------|------|
| Phase 내 재시도 가능 에러 | 재시도 (Phase별 최대 횟수 적용) |
| 재시도 초과 | PAUSED 상태, 사용자 알림 |
| 인증 에러 | PAUSED, 재인증 안내 |
| 사용자 "중단" 선택 | CANCELLED 상태 |

---

## 스킵 가능 단계 요약

| 단계 | 플래그 | 기본 동작 |
|------|--------|-----------|
| Brainstorm | `--skip-brainstorm` | 티켓 명확 시 자동 스킵 |
| Deepen | `--deepen` | 기본 스킵, 플래그로 활성화 |
| PR Review | `--skip-review` | 실행 |
| Compound | `--skip-compound` | 실행 |
| Slack 알림 | `--no-notify` | 실행 |

---

## 사용 예시

```
/dev-workflow:lfg BE-123
# 기본 실행 (2개 확인 게이트)

/dev-workflow:lfg BE-123 --auto
# 완전 자율 실행 (확인 없이 전 과정 자동)

/dev-workflow:lfg BE-123 --auto --detail alot
# 자율 실행 + 전체 리서치 (프레임워크 문서 + 모범 사례)

/dev-workflow:lfg BE-123 --deepen
# PLANNING 후 심화 분석 포함

/dev-workflow:lfg BE-123 --skip-review --skip-compound
# 리뷰와 지식 축적 스킵 (빠른 실행)

/dev-workflow:lfg --resume
# 중단된 파이프라인 재개

/dev-workflow:lfg BE-123 --auto --no-notify
# 자율 실행, Slack 알림만 스킵
```

---

## 전체 워크플로우 다이어그램

```
/lfg BE-123
  │
  ├─ 1. INIT
  │    ├── Jira → In Progress
  │    └── git checkout -b feature/BE-123-...
  │
  ├─ 2. PLANNING
  │    ├── (brainstorm) ← 선택적
  │    ├── 병렬 리서치 (repo + learnings [+ framework + best-practices at A LOT])
  │    ├── Plan 모드 → Tech Spec 논의
  │    ├── ★ Gate 1: "실행할까요?"
  │    ├── Confluence 페이지 생성
  │    └── docs/plans/{ticket}-plan.md 생성
  │
  ├─ 3. WORKING
  │    ├── 계획 문서 → 태스크 분해
  │    ├── 태스크별: 실행 → 테스트 → (실패 시 수정 3회)
  │    ├── Phase별 자동 커밋 (Gitmoji)
  │    └── 전체 테스트 실행
  │
  ├─ 4. REVIEWING
  │    ├── Track 1: Security Sentinel (보안)
  │    ├── Track 2: Performance Oracle (성능)
  │    ├── Track 3: 3자 합의 리뷰 (품질)
  │    └── 결과 통합 (중복 제거, P1-P5)
  │
  ├─ 5. FIXING (P1/P2 있을 경우)
  │    ├── P1/P2 자동 수정
  │    ├── 테스트 → 간이 재리뷰
  │    └── 최대 3라운드
  │
  ├─ 6. PR_CREATING
  │    ├── ★ Gate 2: "PR을 생성할까요?"
  │    ├── git push + gh pr create
  │    └── Jira에 PR 링크 추가
  │
  ├─ 7. DOCUMENTING
  │    ├── 5개 병렬 에이전트 (compound)
  │    └── docs/solutions/ 솔루션 저장
  │
  ├─ 8. NOTIFYING
  │    └── Slack 리뷰 요청 메시지
  │
  └─ 9. COMPLETE
       └── 최종 보고
```

---

## 에러 처리

| 상황 | 대응 |
|------|------|
| 티켓 없음 | "티켓을 찾을 수 없습니다" → 중단 |
| 프로젝트 매핑 실패 | 사용자에게 프로젝트 선택 요청 |
| 테스트 전체 실패 | PAUSED → 사용자 개입 |
| 리뷰 P1 수정 불가 | PAUSED → 사용자 개입 |
| PR 생성 실패 | 2회 재시도 → PAUSED |
| 기타 에러 | [공통 에러 핸들링 가이드](../shared/error-handling.md) 참조 |

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
  --title "[Bug] lfg: {에러 요약}" \
  --body "{이슈 본문 - shared/error-handling.md 템플릿 참조}" \
  --label "bug"
```

---

## Attribution

PR, Confluence 문서, Slack 메시지, Jira 코멘트에 config의 attribution을 추가합니다:

- `~/.claude/workflow/config.json`의 `attribution.text` 값을 사용합니다 (하드코딩 금지)
- `attribution.enabled`가 `false`인 경우 생략합니다

```
# config.json 예시
"attribution": {
  "text": "🤖 Written with Claude Code"
}
```

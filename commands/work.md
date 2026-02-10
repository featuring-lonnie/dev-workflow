---
description: 구조화된 작업 실행. 계획 문서 기반 태스크 분해 후 실행-테스트-커밋 루프.
---

# /dev-workflow:work - 구조화된 작업 실행

## 개요

계획 문서(`docs/plans/`)를 기반으로 태스크를 분해하고, 우선순위순으로 **실행 → 테스트 → 커밋** 루프를 반복합니다. 논리적 단위(Phase)가 완료될 때마다 자동으로 커밋합니다.

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

### 1. 계획 문서 로드

**입력 소스 (우선순위):**

| 순위 | 소스 | 예시 |
|------|------|------|
| 1 | 명시 경로 | `/work docs/plans/BE-123-plan.md` |
| 2 | LFG 상태 파일 | `.omc/state/lfg-state.json`의 `planDocPath` |
| 3 | 자동 감지 | `docs/plans/`에서 현재 브랜치 티켓 매칭 |
| 4 | 즉석 생성 | Jira 티켓 + 코드 탐색으로 간이 계획 |

**자동 감지 로직:**
```
Tool: Glob
Pattern: "{projectPath}/docs/plans/*{ticket_id}*"
```

**즉석 생성 (소스 4):**

계획 문서가 없는 경우 Jira 티켓과 코드 탐색 결과로 간이 계획을 생성합니다:

```
Tool: mcp__atlassian__getJiraIssue
Parameters:
  - cloudId: "{config.integrations.jira.cloudId}"
  - issueIdOrKey: "{ticket_id}"
```

```
Tool: Task
subagent_type: "oh-my-claudecode:explore"
model: haiku
prompt: |
  {projectPath}에서 {ticket_title} 구현을 위해 변경이 필요한 파일과
  구현 순서를 분석해주세요.
```

### 2. 태스크 분해

계획 문서의 구현 계획 섹션을 태스크로 분해합니다:

```
Tool: TaskCreate (태스크별)
subject: "{Phase N}: {task_description}"
description: "{상세 구현 내용}"
```

**태스크 속성:**
| 속성 | 설명 |
|------|------|
| Phase | 논리적 단위 (커밋 단위) |
| 우선순위 | Phase 내 실행 순서 |
| 의존성 | 선행 태스크 ID |
| 예상 파일 | 변경 예상 파일 목록 |

### 3. 실행 루프

```
각 Phase에 대해:
  각 태스크에 대해:
    ├── 태스크 실행 (코드 작성/수정)
    ├── 테스트 실행
    │   ├── 통과 → 태스크 완료
    │   └── 실패 → 수정 시도 (최대 3회)
    │       ├── 수정 성공 → 태스크 완료
    │       └── 3회 실패 → 사용자에게 알림 후 계속/중단 선택
    └── 다음 태스크

  Phase 완료 시:
    ├── 관련 파일 스테이징
    └── 자동 커밋 (Gitmoji 컨벤션)
```

#### 3-1. 태스크 실행

각 태스크의 코드 작성/수정을 수행합니다. 계획 문서에 명시된 구현 방법을 따릅니다.

#### 3-2. 테스트 실행

**프로젝트 테스트 프레임워크 자동 감지:**

```bash
# Python
python -m pytest {test_path} -v

# JavaScript/TypeScript
npm test -- {test_path}
# 또는
pnpm test -- {test_path}

# Go
go test ./...

# Java
./gradlew test
```

**테스트 범위:**
- 변경된 코드의 기존 테스트
- 새로 작성한 테스트
- 관련 통합 테스트 (있는 경우)

#### 3-3. 실패 시 수정

```
테스트 실패
  ↓
에러 메시지 분석
  ↓
코드 수정 (시도 {n}/3)
  ↓
재테스트
  ↓
성공? → 계속 / 실패? → 다음 시도 또는 사용자 알림
```

**3회 실패 시:**
```
Tool: AskUserQuestion
Question: "태스크 '{task_name}' 테스트가 3회 실패했습니다. 어떻게 하시겠습니까?"
Options:
  - "건너뛰기" - 이 태스크를 스킵하고 다음으로
  - "수동 수정" - 작업을 중단하고 수동 개입
  - "전체 중단" - 워크 세션 종료
```

#### 3-4. Phase 커밋

Phase의 모든 태스크가 완료되면 자동 커밋합니다.

**커밋 컨벤션**: 기존 Gitmoji 컨벤션 사용 (`/dev-workflow:commit` 참조)

```bash
# Phase에서 변경된 파일만 스테이징
git add {changed_files_in_phase}

# Gitmoji 커밋
git commit -m "$(cat <<'EOF'
{gitmoji} [{app_name}] {phase_description}

Co-Authored-By: Claude Code <noreply@anthropic.com>
EOF
)"
```

### 4. 전체 테스트

모든 Phase 완료 후 전체 테스트 스위트를 실행합니다:

```bash
# 전체 테스트 실행
{project_test_command}
```

**전체 테스트 실패 시:**
- 실패한 테스트 분석
- 관련 코드 수정
- 재실행 (최대 3회)
- 3회 실패 시 사용자에게 알림

### 5. 완료 보고

```
작업 실행 완료

계획: {plan_doc_path}
실행 결과:
  총 Phase: {total_phases}
  총 태스크: {total_tasks}
  완료: {completed_tasks}
  스킵: {skipped_tasks}

커밋:
  1. {commit_hash} - {commit_message}
  2. {commit_hash} - {commit_message}
  ...

테스트:
  전체: {total_tests}
  통과: {passed}
  실패: {failed}

다음 단계:
  1. /dev-workflow:pr-review — 셀프 리뷰
  2. /dev-workflow:pr — PR 생성
```

---

## Swarm 모드 (병렬 실행)

**5개 이상의 독립 태스크**가 있을 경우 자동으로 Swarm 모드를 제안합니다:

```
Tool: AskUserQuestion
Question: "독립 태스크가 {n}개 감지되었습니다. 병렬 실행할까요?"
Options:
  - "병렬 실행 (Swarm)" - 독립 태스크를 병렬로 실행
  - "순차 실행" - 하나씩 순서대로 실행
```

**Swarm 모드 실행:**
```
Tool: Task (병렬, 독립 태스크별)
subagent_type: "oh-my-claudecode:executor"
model: sonnet
prompt: |
  {projectPath}에서 다음 태스크를 실행해주세요:
  {task_description}

  완료 후 변경된 파일 목록을 보고해주세요.
```

---

## 사용 예시

```
/dev-workflow:work docs/plans/BE-123-plan.md
# 특정 계획 문서로 실행

/dev-workflow:work
# 자동 감지된 계획 문서로 실행

/dev-workflow:work --swarm
# 강제 Swarm 모드

/dev-workflow:work --no-commit
# 자동 커밋 비활성화 (수동 커밋)
```

---

## 에러 처리

| 상황 | 대응 |
|------|------|
| 계획 문서 없음 | 즉석 계획 생성 또는 `/doc` 안내 |
| 테스트 프레임워크 미감지 | 사용자에게 테스트 명령어 확인 |
| 커밋 충돌 | `git stash` → 해결 → `git stash pop` |

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
  --title "[Bug] work: {에러 요약}" \
  --body "{이슈 본문 - shared/error-handling.md 템플릿 참조}" \
  --label "bug"
```

---

## Attribution

커밋 메시지에 다음 Co-Author를 추가합니다:

```
Co-Authored-By: Claude Code <noreply@anthropic.com>
```

config.json의 `attribution.enabled`가 `false`인 경우 생략합니다.

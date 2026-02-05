---
description: 워크플로우 초기 설정. 외부 연동 확인 후 프로젝트 설정.
---

# /dev-workflow:init - 워크플로우 초기 설정

## 개요

개발자 워크플로우 사용을 위한 초기 설정을 진행합니다.
단계별로 진행하며, 이전 단계가 완료되어야 다음 단계로 넘어갑니다.

---

## Phase 1: 외부 연동 확인

### 1.1 GitHub CLI (gh) 확인

```bash
# 설치 확인
gh --version

# 인증 확인
gh auth status
```

**체크 항목:**
- [ ] gh CLI 설치됨
- [ ] 인증 완료됨 (github.com 로그인)

**실패 시 안내:**
```bash
# 설치
brew install gh

# 인증
gh auth login
```

### 1.2 Slack MCP 확인

```
Tool: mcp__slack__channels_list
Parameters:
  - limit: 1
```

**체크 항목:**
- [ ] Slack MCP 연결됨
- [ ] 채널 목록 조회 가능

**실패 시 안내:**
- `~/.claude.json`에서 Slack MCP 설정 확인
- `SLACK_MCP_ADD_MESSAGE_TOOL: "true"` 환경변수 확인

### 1.3 Atlassian MCP 확인 (Jira)

**먼저 Cloud ID (UUID)를 조회합니다:**
```
Tool: mcp__atlassian__getAccessibleAtlassianResources
```

**반환 예시:**
```json
{
  "id": "2c7ce89f-01b8-412e-ba83-cb9d65b57537",  ← 이 UUID를 저장
  "url": "https://featuring-corp.atlassian.net",
  "name": "featuring-corp"
}
```

**중요:** `id` 필드의 UUID를 `config.integrations.jira.cloudId`와 `config.integrations.confluence.cloudId`에 저장합니다. URL이 아닌 UUID가 필요합니다.

**체크 항목:**
- [ ] Atlassian MCP 연결됨
- [ ] Cloud ID (UUID) 조회 가능

**실패 시 안내:**
- Atlassian MCP 설정 확인
- OAuth 인증 상태 확인

### 1.4 Atlassian MCP 확인 (Confluence)

```
Tool: mcp__atlassian__getConfluenceSpaces
Parameters:
  - cloudId: "{cloud_id}"
  - limit: 1
```

**체크 항목:**
- [ ] Confluence 접근 가능
- [ ] 스페이스 목록 조회 가능

---

### Phase 1 결과 출력

```
워크플로우 연동 상태

| 서비스          | 상태   | 비고                |
|-----------------|--------|---------------------|
| GitHub CLI      | OK/FAIL| gh v2.x.x / 미설치  |
| GitHub 인증     | OK/FAIL| 로그인됨 / 필요     |
| Slack MCP       | OK/FAIL| 연결됨 / 설정필요   |
| Slack 쓰기권한  | OK/FAIL| 활성화 / 비활성화   |
| Jira MCP        | OK/FAIL| 연결됨 / 설정필요   |
| Confluence MCP  | OK/FAIL| 연결됨 / 설정필요   |

{모두 OK인 경우}
모든 연동이 확인되었습니다.
Phase 2로 진행하시겠습니까? (Y/n)

{하나라도 FAIL인 경우}
일부 연동이 필요합니다.
위 안내에 따라 설정 후 다시 /dev-workflow:init을 실행해주세요.
```

---

## Phase 2: 역할 및 프로젝트 설정

Phase 1이 모두 통과한 경우에만 진행합니다.

### 2.0 역할 및 전문분야 설정 (v2.1)

#### Step 1: 역할 선택

**AskUserQuestion 사용:**
```
워크플로우 역할을 선택하세요:

1. Developer (개발자) - Git, PR, 코드 리뷰 중심 (/dev-workflow:* 명령어)
2. PM (프로젝트 매니저) - Jira 티켓, PRD, 릴리즈 노트 중심 (/pm-workflow:* 명령어)
3. Designer (디자이너) - Figma, 디자인 스펙, 핸드오프 중심 (/de-workflow:* 명령어)
```

#### Step 2: 전문분야 선택 (역할별)

**Developer 선택 시:**
```
개발 전문분야를 선택하세요:

1. Backend - Django/Python, API 설계, DB 최적화
2. Frontend - Next.js/React, UI/UX 구현
3. Fullstack - 전체 스택 개발
4. DevOps - 인프라, CI/CD, 모니터링
```

**PM 선택 시:**
```
PM 전문분야를 선택하세요:

1. Product - PRD, 기능 기획, 사용자 리서치
2. Project - 일정 관리, 리소스 조율
3. Technical - 기술 PM, 아키텍처 검토
```

**Designer 선택 시:**
```
디자인 전문분야를 선택하세요:

1. Product - UI/UX, 프로덕트 디자인
2. Brand - 브랜드 디자인, 마케팅 에셋
3. Motion - 모션 그래픽, 인터랙션 디자인
```

#### 전문분야별 설정 차이

| 항목 | Backend | Frontend | Fullstack | DevOps |
|------|---------|----------|-----------|--------|
| 기본 Jira 프로젝트 | BE | FS | BE, FS | BE |
| PR 라벨 | `backend` | `frontend` | `fullstack` | `infra` |
| Slack 채널 | 개발_백엔드 | 개발_프론트엔드 | 개발_백엔드 | 개발_인프라 |
| 기본 리뷰어 | 백엔드팀 | 프론트팀 | 혼합 | 인프라팀 |
| Confluence 스페이스 | Backend | Frontend | Backend | Infrastructure |
| 커밋 app_names | api, batch 등 | components 등 | 전체 | terraform 등 |

**역할별 명령어:**

| 역할 | 사용 가능 명령어 |
|------|-----------------|
| Developer | `/dev-workflow:start`, `/dev-workflow:commit`, `/dev-workflow:pr`, `/dev-workflow:review`, `/dev-workflow:pr-review`, `/dev-workflow:approve`, `/dev-workflow:done`, `/dev-workflow:doc`, `/dev-workflow:status` |
| PM | `/pm-workflow:start`, `/pm-workflow:ticket`, `/pm-workflow:track`, `/pm-workflow:prd`, `/pm-workflow:release`, `/pm-workflow:update`, `/pm-workflow:done`, `/pm-workflow:status` |
| Designer | `/de-workflow:start`, `/de-workflow:upload`, `/de-workflow:spec`, `/de-workflow:review`, `/de-workflow:handoff`, `/de-workflow:done`, `/de-workflow:status` |

**Config 업데이트:**
```json
{
  "currentUser": {
    "name": "{username}",
    "role": "developer",
    "specialization": "backend"
  }
}
```

---

### 2.1 프로젝트 매핑 설정

**로컬 프로젝트 경로와 GitHub 레포지토리 매핑**

```bash
# 로컬 프로젝트 디렉토리 탐색 (기본: ~/study)
ls -la ~/study/

# 각 프로젝트의 git remote 확인
cd {project_path} && git remote -v
```

**AskUserQuestion 사용:**
- 질문: "프로젝트가 있는 기본 디렉토리를 확인해주세요"
- 기본값: ~/study 또는 현재 디렉토리

**자동 수집:**
각 프로젝트별로:
1. 로컬 경로 (localPath)
2. Git remote URL에서 org/repo 추출
3. **연관된 Jira 프로젝트 키** (AskUserQuestion으로 확인)

**Jira 프로젝트 연결 (중요!):**
```
각 로컬 프로젝트에 대해 AskUserQuestion:
"이 프로젝트에서 사용할 Jira 프로젝트를 선택해주세요"
- 옵션: 2.3에서 조회한 Jira 프로젝트 목록 (BE, FS, FC 등)
- 복수 선택 가능 (multiSelect: true)
```

**이 연결이 중요한 이유:**
- `/dev-workflow:start FS-533` 실행 시
- FS 프로젝트 키 → jiraProjects에 "FS"가 포함된 로컬 프로젝트 찾기
- 해당 프로젝트의 localPath에서 Git 작업 수행

**매핑 형식:**
```json
{
  "projects": [
    {
      "name": "project-a",
      "localPath": "~/projects/project-a",
      "github": {
        "org": "my-org",
        "repo": "project-a"
      },
      "jiraProjects": ["BE", "FC"]
    },
    {
      "name": "project-b",
      "localPath": "~/projects/project-b",
      "github": {
        "org": "my-org",
        "repo": "project-b"
      },
      "jiraProjects": ["FS"]
    }
  ]
}
```

**매핑 예시:**
- `FS-533` → FS → project-b (~/projects/project-b)
- `BE-123` → BE → project-a (~/projects/project-a)

### 2.2 팀원 정보 수집

**Jira 팀원 정보:**
```
Tool: mcp__atlassian__lookupJiraAccountId
Parameters:
  - cloudId: "{config.integrations.jira.cloudId}"  // UUID
  - query: "{팀원 이름}"
```

**Slack 팀원 정보:**
```
Tool: mcp__slack__channels_list
- 채널 멤버에서 user_id 수집
```

### 2.3 Jira 설정

**프로젝트 목록 조회:**
```
Tool: mcp__atlassian__getVisibleJiraProjects
```

**AskUserQuestion 사용:**
- 질문: "사용할 Jira 프로젝트를 선택해주세요"
- 옵션: 프로젝트 목록 (BE, FS, FC 등)

### 2.4 Slack 설정

```
Tool: mcp__slack__channels_list
- 채널 목록 조회
```

**AskUserQuestion 사용:**
- 질문: "리뷰 요청에 사용할 Slack 채널을 선택해주세요"
- 옵션: 채널 목록

### 2.5 Confluence 설정

```
Tool: mcp__atlassian__getConfluenceSpaces
- 스페이스 목록 조회 (개인 스페이스 포함)
```

**AskUserQuestion 사용:**
- 질문: "Tech Spec 문서를 저장할 스페이스를 선택해주세요"
- 옵션: 스페이스 목록

### 2.6 Git 설정

```bash
# 기본 브랜치 확인
git symbolic-ref refs/remotes/origin/HEAD | sed 's@^refs/remotes/origin/@@'
```

**AskUserQuestion 사용:**
- 질문: "기본 브랜치를 선택해주세요"
- 옵션: main, develop, master

### 2.7 커밋/PR 설정

**AskUserQuestion 사용:**

1. **브랜치 prefix**
   - 기본값 제안: feature/, fix/, hotfix/

2. **기본 라벨**
   - 질문: "PR에 기본으로 붙일 라벨이 있나요?"
   - 예: backend, deploy:staging

3. **기본 리뷰어**
   - 질문: "기본 리뷰어를 지정하시겠습니까?"
   - 옵션: 팀원 목록

---

## 설정 파일 저장

모든 설정은 `~/.claude/workflow/config.json`에 저장 (v2.0 스키마):

```json
{
  "version": "2.0",
  "createdAt": "2026-02-04T18:30:00+09:00",
  "updatedAt": "2026-02-04T18:30:00+09:00",

  "team": {
    "name": "{team_name}",
    "prefix": "{team_prefix}"
  },

  "currentUser": {
    "name": "{username}",
    "role": "developer"
  },

  "roles": {
    "developer": {
      "enabled": true,
      "commands": ["start", "commit", "pr", "review", "pr-review", "approve", "done", "doc", "status"],
      "slack": {
        "channelId": "{channel_id}",
        "channelName": "{channel_name}",
        "mentionGroup": "{mention_group}"
      },
      "jiraProjects": ["BE", "FC", "FS"],
      "git": {
        "baseBranch": "develop",
        "branchPrefix": {
          "feature": "feature/",
          "fix": "fix/",
          "hotfix": "hotfix/"
        }
      },
      "commit": {
        "useGitmoji": true,
        "convention": "<gitmoji> [<app_name>] <주요 변경점>",
        "appNames": ["user-service", "auth", "payment", "campaign", "influencer", "common", "api", "batch", "admin"]
      },
      "pr": {
        "defaultLabels": ["backend"],
        "defaultReviewers": ["young", "moons"]
      },
      "confluence": {
        "techSpecSpaceKey": "{space_key}",
        "techSpecParentPageId": null
      },
      "templates": {
        "doc": "tech-spec"
      }
    },

    "pm": {
      "enabled": true,
      "commands": ["start", "ticket", "track", "prd", "release", "update", "done", "status"],
      "slack": { "channelId": "", "channelName": "", "mentionGroup": "@PM" },
      "jiraProjects": ["FC", "FS", "BE"],
      "confluence": { "prdSpaceKey": "", "releaseNotesSpaceKey": "", "meetingNotesSpaceKey": "" },
      "templates": { "doc": "prd", "releaseNotes": "release-notes", "meeting": "meeting-notes" }
    },

    "designer": {
      "enabled": true,
      "commands": ["start", "upload", "spec", "review", "handoff", "done", "status"],
      "slack": { "channelId": "", "channelName": "", "mentionGroup": "@디자인", "handoffChannelId": "" },
      "jiraProjects": ["DE", "FC"],
      "figma": { "teamId": "", "fileNamingConvention": "[{ticket_id}] {feature_name}" },
      "confluence": { "designSpecSpaceKey": "" },
      "templates": { "doc": "design-spec" }
    }
  },

  "integrations": {
    "jira": {
      "cloudId": "{UUID from getAccessibleAtlassianResources}",  // 예: "2c7ce89f-01b8-412e-ba83-cb9d65b57537"
      "projects": [
        { "key": "BE", "name": "Backend Engineering", "id": "{project_id}" },
        { "key": "FS", "name": "Frontend Studio", "id": "{project_id}" },
        { "key": "FC", "name": "Feature Core", "id": "{project_id}" }
      ],
      "defaultProject": "BE"
    },
    "slack": { "workspace": "{workspace_name}" },
    "confluence": {
      "cloudId": "{UUID from getAccessibleAtlassianResources}",  // Jira와 동일한 UUID 사용
      "spaces": {
        "team": { "key": "{team_space_key}", "name": "{team_space_name}", "id": "{space_id}" },
        "personal": { "key": "{personal_space_key}", "name": "{username}'s personal space" }
      }
    },
    "github": { "org": "{github_org}" }
  },

  "users": {
    "{username}": {
      "jira": { "accountId": "{jira_account_id}", "displayName": "{username}" },
      "slack": { "userId": "{slack_user_id}", "displayName": "{username}" },
      "role": "developer"
    }
  },

  "projects": [
    {
      "name": "project-a",
      "localPath": "~/projects/project-a",
      "github": { "org": "my-org", "repo": "project-a" },
      "jiraProjects": ["BE", "FC"]
    },
    {
      "name": "project-b",
      "localPath": "~/projects/project-b",
      "github": { "org": "my-org", "repo": "project-b" },
      "jiraProjects": ["FS"]
    }
  ],

  "attribution": {
    "enabled": true,
    "text": "Written with Claude Code",
    "coAuthor": "Co-Authored-By: Claude Code <noreply@anthropic.com>"
  }
}
```

---

## 사용 예시

```
/dev-workflow:init
# Phase 1: 연동 확인
# Phase 2: 역할 선택 + 프로젝트 설정 (Phase 1 통과 시)

/dev-workflow:init --check
# Phase 1만 실행 (연동 상태 확인만)

/dev-workflow:init --reconfigure
# 기존 설정 무시하고 재설정

/dev-workflow:init --add-project
# 새 프로젝트 매핑 추가
```

---

## --add-project 옵션

기존 설정에 새 프로젝트를 추가:

```
/dev-workflow:init --add-project

# 실행 순서:
1. 현재 디렉토리 또는 지정 경로의 git 정보 확인
2. GitHub org/repo 추출
3. 연관 Jira 프로젝트 선택
4. config.json의 projects 배열에 추가
```

**출력:**
```
새 프로젝트 추가

경로: ~/projects/new-project
GitHub: my-org/new-project
Jira: BE

projects 배열에 추가되었습니다.
```

---

## Attribution 설정

모든 생성/수정 작업에 attribution을 추가합니다:

| 대상 | Attribution 형식 |
|------|-----------------|
| PR 본문 | `Written with Claude Code` |
| 리뷰 코멘트 | `Written with Claude Code` |
| Slack 메시지 | `Written with Claude Code` |
| Confluence 문서 | `Written with Claude Code` |
| Git 커밋 | `Co-Authored-By: Claude Code <noreply@anthropic.com>` |

**비활성화:**
```json
{
  "attribution": {
    "enabled": false
  }
}
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
  --title "[Bug] init: {에러 요약}" \
  --body "{이슈 본문 - shared/error-handling.md 템플릿 참조}" \
  --label "bug"
```

---

## 설정 후 다음 단계

설정 완료 후 안내:

```
워크플로우 설정 완료!

설정 파일: ~/.claude/workflow/config.json

사용자: {username}
역할: {role}
전문분야: {specialization}

등록된 프로젝트:
   - project-a (BE, FC)
   - project-b (FS)

이제 다음 명령어를 사용할 수 있습니다:
   /dev-workflow:start {TICKET}  - 작업 시작
   /dev-workflow:commit          - 커밋
   /dev-workflow:pr              - PR 생성
   /dev-workflow:review          - 리뷰 요청
   /dev-workflow:pr-review       - PR 리뷰 수행
   /dev-workflow:approve         - 리뷰 승인
   /dev-workflow:done            - 작업 완료
   /dev-workflow:doc             - Tech Spec 작성
   /dev-workflow:status          - 상태 확인

팁:
   - 역할 변경: config.json의 currentUser.role 수정
   - 전문분야 변경: config.json의 currentUser.specialization 수정
   - 설정 변경: /dev-workflow:init --reconfigure
   - 프로젝트 추가: /dev-workflow:init --add-project
   - 연동 확인: /dev-workflow:init --check

관련 워크플로우:
   - /workflow-common:init    - 역할 선택 및 초기화
   - /workflow-common:status  - 통합 상태 확인
```

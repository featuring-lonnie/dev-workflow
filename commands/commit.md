---
description: 개발자 커밋. Gitmoji 컨벤션 적용. 형식 - <gitmoji> [<app_name>] <주요 변경점>
---

# /dev-workflow:commit - 커밋

## 커밋 메시지 형식

```
<gitmoji> [<app_name>] <주요 변경점>
```

예시:
- `✨ [user-service] 회원가입 API 추가`
- `🐛 [auth] 토큰 만료 버그 수정`
- `♻️ [payment] 결제 로직 리팩토링`

## Gitmoji 가이드

| Gitmoji | 코드 | 의미 |
|---------|------|------|
| ✨ | `:sparkles:` | 새 기능 추가 |
| 🐛 | `:bug:` | 버그 수정 |
| ♻️ | `:recycle:` | 리팩토링 |
| 🔧 | `:wrench:` | 설정 파일 변경 |
| 📝 | `:memo:` | 문서 추가/수정 |
| ✅ | `:white_check_mark:` | 테스트 추가/수정 |
| 🚀 | `:rocket:` | 배포 관련 |
| 🔥 | `:fire:` | 코드/파일 삭제 |
| 💄 | `:lipstick:` | UI/스타일 변경 |
| 🗃️ | `:card_file_box:` | DB 관련 변경 |
| 🔒 | `:lock:` | 보안 관련 |
| ⬆️ | `:arrow_up:` | 의존성 업그레이드 |
| 🏗️ | `:building_construction:` | 아키텍처 변경 |

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

### 1. 변경사항 확인
```bash
git status
git diff --staged
git diff  # unstaged 포함
```

### 2. 변경 분석
- 변경된 파일 목록 확인
- 주요 변경 모듈/서비스 파악 → app_name
- 변경 성격 파악 → gitmoji 선택

### 3. 스테이징
```bash
# 관련 파일만 스테이징 (전체 add 지양)
git add src/services/user/*.ts
git add src/controllers/UserController.ts
```

### 4. 커밋
```bash
git commit -m "$(cat <<'EOF'
✨ [user-service] 회원가입 API 추가

Co-Authored-By: Claude Code <noreply@anthropic.com>
EOF
)"
```

## app_name 규칙

config.json의 `roles.developer.commit.appNames`에서 참조:
```json
{
  "roles": {
    "developer": {
      "commit": {
        "appNames": ["user-service", "auth", "payment", ...]
      }
    }
  }
}
```

**설정 참조**: `~/.claude/workflow/config.json` → `roles.developer.commit.appNames`

## 자동 gitmoji 선택 로직

| 변경 패턴 | gitmoji |
|-----------|---------|
| 새 파일 + 기능 코드 | ✨ |
| 기존 파일 버그 수정 | 🐛 |
| 테스트 파일 변경 | ✅ |
| 설정 파일만 변경 | 🔧 |
| 파일 삭제 | 🔥 |
| import/구조 변경만 | ♻️ |
| README, docs 변경 | 📝 |

## 사용 예시

```
/dev-workflow:commit
# 자동으로 변경사항 분석 후 커밋 메시지 생성

/dev-workflow:commit 로그인 버그 수정
# 힌트 제공 시 해당 내용으로 커밋
```

---

## Attribution

커밋 메시지에 다음 Co-Author를 추가합니다:

```
Co-Authored-By: Claude Code <noreply@anthropic.com>
```

config.json의 `attribution.enabled`가 `false`인 경우 생략합니다.

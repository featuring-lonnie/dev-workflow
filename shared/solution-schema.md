# 솔루션 문서 스키마

## 개요

`/dev-workflow:compound` 커맨드로 생성되는 솔루션 문서의 표준 스키마 정의입니다.
모든 솔루션 문서는 이 스키마를 따라 `docs/solutions/` 디렉토리에 저장됩니다.

---

## YAML Frontmatter 스키마

```yaml
---
title: string             # 솔루션 제목 (한글, 명확한 문제 설명)
date: YYYY-MM-DD          # 작성일
category: string          # 카테고리 (아래 목록 참조)
tags: string[]            # 검색용 태그 (3-7개)
jira_ticket: string       # PROJ-NNN (nullable, Jira 연동 안 할 경우 null)
severity: string          # 심각도 (아래 목록 참조)
project: string           # config.projects[].name
root_cause: string        # 근본 원인 한줄 요약
recurrence_risk: string   # 재발 위험도 (아래 목록 참조)
---
```

---

## 필드 상세

### category (카테고리)

| 값 | 설명 | 예시 |
|----|------|------|
| `bug` | 버그 수정 | 로그인 실패, 데이터 누락 |
| `performance` | 성능 개선 | N+1 쿼리, 느린 응답 |
| `security` | 보안 수정 | XSS, CSRF, 인증 우회 |
| `architecture` | 아키텍처 결정/변경 | 모듈 분리, 패턴 도입 |
| `integration` | 외부 연동 | API 연동, 서드파티 라이브러리 |
| `configuration` | 설정 관련 | 환경변수, 배포 설정 |
| `devops` | 인프라/배포 | CI/CD, Docker, 모니터링 |

### severity (심각도)

| 값 | 설명 |
|----|------|
| `low` | 사소한 문제, 사용자 영향 없음 |
| `medium` | 일부 기능 영향, 우회 가능 |
| `high` | 핵심 기능 영향, 즉시 수정 필요 |
| `critical` | 서비스 중단 또는 데이터 손실 위험 |

### recurrence_risk (재발 위험도)

| 값 | 설명 |
|----|------|
| `low` | 일회성 문제, 재발 가능성 낮음 |
| `medium` | 유사 코드에서 재발 가능 |
| `high` | 패턴적 문제, 적극적 예방 필요 |

---

## 파일명 규칙

```
docs/solutions/YYYY-MM-DD-{category}-{descriptive-slug}.md
```

예시:
- `docs/solutions/2026-02-10-bug-login-token-expiry-handling.md`
- `docs/solutions/2026-02-08-performance-n-plus-one-user-list.md`
- `docs/solutions/2026-02-05-security-xss-comment-sanitization.md`

**규칙:**
- 날짜: ISO 8601 (YYYY-MM-DD)
- 카테고리: frontmatter의 category 값
- 슬러그: 영문 소문자, 하이픈 구분, 핵심 키워드 포함

---

## 본문 구조

```markdown
---
{YAML frontmatter}
---

# {title}

## 문제 상황
{문제가 발생한 상황과 증상 설명}

## 타임라인
| 시점 | 이벤트 |
|------|--------|
| {datetime} | {event} |

## 근본 원인
{상세한 근본 원인 분석}

### 관련 코드
```{language}
{problematic_code}
```

## 해결 방법
{적용한 해결 방법}

### 변경된 코드
```{language}
{fixed_code}
```

### 변경된 파일
- `{file_path}`: {변경 내용}

## 재발 방지
- {prevention_strategy_1}
- {prevention_strategy_2}

## 관련 자료
- Jira: [{ticket_id}]({jira_url})
- PR: [#{pr_number}]({pr_url})
- 관련 솔루션: [{related_solution}]({path})

---

Written with Claude Code
```

---

## 검색 활용

`learnings-researcher` 에이전트가 다음 방식으로 솔루션을 검색합니다:

| 검색 방법 | 대상 필드 |
|-----------|-----------|
| 카테고리 검색 | `category` |
| 태그 검색 | `tags` |
| 키워드 검색 | `title`, 본문 |
| 프로젝트별 검색 | `project` |
| 심각도별 검색 | `severity` |
| 티켓 연결 | `jira_ticket` |

---

## 디렉토리 구조

```
{project_root}/
└── docs/
    └── solutions/
        ├── 2026-02-10-bug-login-token-expiry.md
        ├── 2026-02-08-performance-n-plus-one.md
        ├── 2026-02-05-security-xss-comment.md
        └── ...
```

**주의:** `docs/solutions/` 디렉토리가 없으면 `/compound` 실행 시 자동 생성합니다.

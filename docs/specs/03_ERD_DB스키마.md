# Playwright Hub — ERD 및 DB 스키마

## 1. ER 다이어그램

```mermaid
erDiagram
    Org ||--o{ User : "has"
    Org ||--o{ Project : "owns"
    User ||--o{ Run : "executes"
    User ||--o{ Edit : "requests"
    Project ||--o{ Run : "contains"
    Project ||--o{ Edit : "contains"

    Org {
        UUID id PK
        String name
        DateTime created_at
        DateTime updated_at
    }

    User {
        UUID id PK
        UUID org_id FK
        String email UK
        String password_hash
        Role role "ADMIN | MEMBER"
        DateTime created_at
        DateTime updated_at
    }

    Project {
        UUID id PK
        UUID org_id FK
        String name
        String path
        String git_url
        JSON config
        DateTime created_at
        DateTime updated_at
    }

    Run {
        UUID id PK
        UUID user_id FK
        UUID project_id FK
        RunStatus status "QUEUED | RUNNING | PASSED | FAILED | CANCELLED"
        String test_filter
        String report_path
        Int duration_ms
        Int total_tests
        Int passed_tests
        Int failed_tests
        Int skipped_tests
        String error_message
        DateTime created_at
        DateTime finished_at
    }

    Edit {
        UUID id PK
        UUID user_id FK
        UUID project_id FK
        String prompt
        String diff
        JSON files_changed
        EditStatus status "STREAMING | WAITING_INPUT | PENDING | APPROVED | REJECTED"
        String commit_hash
        DateTime synced_at
        DateTime created_at
        String session_id "SDK 세션 ID"
        JSON messages "대화 이력"
        Float cost_usd "세션 비용"
        Int duration_ms "소요시간"
    }
```

## 2. 관계 요약

| 관계 | 설명 |
|------|------|
| Org → User | 1:N. 하나의 조직에 여러 사용자가 소속 |
| Org → Project | 1:N. 하나의 조직이 여러 프로젝트를 소유 |
| User → Run | 1:N. 하나의 사용자가 여러 실행을 수행 |
| Project → Run | 1:N. 하나의 프로젝트에 여러 실행 이력 |
| User → Edit | 1:N. 하나의 사용자가 여러 수정 요청 |
| Project → Edit | 1:N. 하나의 프로젝트에 여러 수정 이력 |

## 3. Prisma Schema

> **구현 참고**: Prisma(PostgreSQL 프로바이더)로 `Org`, `User`, `Project`, `Run`, `Edit` 모델과 `Role`, `RunStatus`, `EditStatus` 열거형을 정의한다. 엔티티 간 관계는 위 ER 다이어그램과 동일하며, `Project → Org`, `Run → Project`, `Edit → Project` 관계에 `onDelete: Cascade`가 적용된다. 실제 스키마는 `prisma/schema.prisma`에서 관리한다.

## 4. config JSON 구조 (Project.config)

> **구현 참고**: `Project.config`(JSON)는 `baseURL`, `env`(암호화 문자열 포함), `browser`, `timeout`, `retries`, `workers`, `reportOptions`(screenshots/video/trace 캡처 정책) 필드를 담는 Playwright 실행 설정이다.

## 5. 인덱스 전략

| 테이블 | 인덱스 | 용도 |
|--------|--------|------|
| users | org_id | 조직별 사용자 조회 |
| projects | org_id, name (unique) | 조직 내 프로젝트명 중복 방지 |
| runs | project_id, created_at DESC | 프로젝트별 최근 실행 조회 |
| runs | status | 상태별 필터 (대기 중 작업 조회) |
| runs | user_id | 사용자별 실행 이력 |
| edits | project_id, created_at DESC | 프로젝트별 최근 수정 이력 |
| edits | user_id | 사용자별 수정 이력 |

## 6. 상태 전이도

```mermaid
stateDiagram-v2
    state "Run 상태" as RunState {
        [*] --> QUEUED : 실행 요청
        QUEUED --> RUNNING : Worker 수신
        RUNNING --> PASSED : exit 0
        RUNNING --> FAILED : exit 1 / timeout
        QUEUED --> CANCELLED : 사용자 취소
        RUNNING --> CANCELLED : 사용자 취소
    }

    state "Edit 상태" as EditState {
        [*] --> STREAMING : 수정 요청
        STREAMING --> WAITING_INPUT : 턴 완료
        WAITING_INPUT --> STREAMING : 후속 메시지
        STREAMING --> PENDING : 세션 완료
        WAITING_INPUT --> PENDING : 대화 종료
        PENDING --> APPROVED : 승인 → 동기화 + Git commit
        PENDING --> REJECTED : 거부 → working 롤백
    }
```

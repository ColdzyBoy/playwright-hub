# Playwright Hub — API 명세서

## 1. 공통 사항

### Base URL

기본 Base URL은 `http://localhost:3001/api/v1`이다.

### 인증

모든 API 요청에 JWT 토큰을 `Authorization: Bearer <token>` 헤더로 포함해야 한다.

### 공통 응답 형식

> **구현 참고**: 성공 응답은 `{ "success": true, "data": ... }` 형태이며, 실패 응답은 `{ "success": false, "error": { "code", "message" } }` 형태로 통일한다.

### 공통 에러 코드

| HTTP | 코드 | 설명 |
|------|------|------|
| 400 | VALIDATION_ERROR | 요청 파라미터 검증 실패 |
| 401 | UNAUTHORIZED | 인증 실패 |
| 403 | FORBIDDEN | 권한 없음 |
| 404 | NOT_FOUND | 리소스 없음 |
| 409 | CONFLICT | 리소스 충돌 (락 등) |
| 429 | RATE_LIMIT | 요청 제한 초과 |
| 500 | INTERNAL_ERROR | 서버 내부 오류 |

---

## 2. 인증 (Auth)

```mermaid
sequenceDiagram
    actor 사용자
    participant API as API Server
    participant DB

    사용자->>API: POST /auth/register (email, password, orgName)
    API->>DB: Org + User 생성
    DB-->>API: user + org
    API-->>사용자: token (JWT) + user + org

    사용자->>API: POST /auth/login (email, password)
    API->>DB: 사용자 조회 + 비밀번호 검증
    DB-->>API: user
    API-->>사용자: token (JWT) + user

    Note over 사용자,API: 이후 모든 요청에 Bearer token 포함

    사용자->>API: POST /auth/invite (Admin 전용)
    API->>DB: 초대 생성
    API-->>사용자: inviteId
```

### POST /auth/register

회원가입.

> **구현 참고**: `email`, `password`, `orgName`을 요청 바디로 받아 Org와 첫 User(ADMIN)를 생성하고 `user`, `org`, JWT `token`을 201로 반환한다.

### POST /auth/login

로그인.

> **구현 참고**: `email`, `password`를 받아 검증 후 `user`(orgId 포함)와 JWT `token`을 200으로 반환한다.

### POST /auth/invite

사용자 초대 (Admin 전용).

> **구현 참고**: `email`, `role`을 받아 초대를 생성하고 `inviteId`를 201로 반환한다.

---

## 3. 프로젝트 (Projects)

### GET /projects

조직의 프로젝트 목록 조회.

> **구현 참고**: 각 항목은 `id`, `name`, `path`, `gitUrl`, `lastRun`(status/createdAt), `createdAt`을 포함한다.

### POST /projects

프로젝트 등록 (Admin 전용).

> **구현 참고**: `name`, `gitUrl`, `config`(baseURL·browser 등)를 받아 `id`, `name`, `path`를 201로 반환한다.

### GET /projects/:id

프로젝트 상세 조회.

### PATCH /projects/:id

프로젝트 설정 수정.

> **구현 참고**: `config`의 부분 필드(예: `baseURL`, `timeout`)를 받아 병합 갱신한다.

### DELETE /projects/:id

프로젝트 삭제 (Admin 전용).

### POST /projects/:id/sync

원격 저장소와 동기화 (git pull).

> **구현 참고**: 응답 `data`에 `commitHash`와 `filesChanged`(변경 파일 수)를 담아 반환한다.

### GET /projects/:id/tests

테스트 파일 및 케이스 목록 조회.

> **구현 참고**: 파일 단위로 `{ file, tests: [{ title, line }] }` 배열을 반환한다.

---

## 4. 실행 (Runs)

```mermaid
sequenceDiagram
    actor 사용자
    participant API as API Server
    participant QR as Queue Router
    participant Redis as BullMQ
    participant Worker
    participant Docker

    사용자->>API: POST /projects/:id/runs
    API->>API: Run(QUEUED) 생성
    API->>QR: 큐 선택 요청
    QR->>Redis: 작업 등록
    API-->>사용자: runId + eventsUrl (202)

    사용자->>API: GET /runs/:runId/events (SSE 연결)
    Redis->>Worker: 작업 수신
    Worker->>Docker: 컨테이너 실행

    loop 실행 중
        Docker-->>Worker: stdout/stderr
        Worker-->>API: 로그 및 진행도 전달 (Redis Pub/Sub)
        API-->>사용자: log 이벤트 (SSE)
        API-->>사용자: progress 이벤트 (SSE)
    end

    Docker-->>Worker: exit code
    Worker->>API: Run 상태 업데이트
    API-->>사용자: done 이벤트 (SSE)

    사용자->>API: GET /runs/:runId/report
    API-->>사용자: 리포트 데이터 (JSON)
```

### POST /projects/:id/runs

테스트 실행 요청.

> **구현 참고**: `testFilter`와 `env`(환경변수 오버라이드)를 받아 Run을 QUEUED로 생성한 뒤 `runId`, `status`, `eventsUrl`(예: `/api/v1/runs/{runId}/events`)을 202로 반환한다.

실행 화면의 실시간 진행도 값은 SSE `progress` 이벤트를 기준으로 갱신한다. `GET /runs/:runId`와 `GET /projects/:id/runs`는 마지막 저장 상태와 완료 후 집계값 조회에 사용한다.

### GET /projects/:id/runs

실행 이력 조회. 쿼리스트링으로 `page`, `limit`, `status`를 받는다.

> **구현 참고**: 응답은 `{ runs: [...], total, page, limit }` 형태이며 각 run은 `id`, `status`, `testFilter`, 테스트 집계(`totalTests`, `passedTests`, `failedTests`, `skippedTests`), `durationMs`, `userId/userName`, `createdAt`, `finishedAt`을 포함한다.

### GET /runs/:runId

실행 상세 조회.

### DELETE /runs/:runId

실행 중단 (status가 QUEUED 또는 RUNNING인 경우).

> **구현 참고**: 성공 시 `{ status: "CANCELLED" }`를 반환한다. QUEUED면 BullMQ `job.remove()`, RUNNING이면 Redis Pub/Sub 채널 `run:{runId}:cancel`로 워커에 취소 신호를 보내 컨테이너를 stop한다.

### GET /runs/:runId/events

테스트 실행 로그 및 진행도 SSE(Server-Sent Events) 스트림.

> **구현 참고**: Fastify + `fastify-sse-v2` 플러그인으로 구현한다. 연결 시 JWT를 쿼리(`?token=...`) 또는 `Authorization` 헤더로 검증하고 Redis Pub/Sub 채널 `run:{runId}:events`를 구독하여 수신되는 이벤트를 그대로 SSE 스트림으로 재전송한다. 이벤트 형식과 타입은 본 문서 §7 참조. 이미 종료된 Run은 최종 상태를 단발 전송 후 스트림을 닫는다. `Last-Event-ID` 헤더가 있으면 해당 시점 이후 이벤트만 리플레이한다(Phase 2+).

---

## 5. 리포트 (Reports)

### GET /runs/:runId/report

리포트 요약 데이터 (파싱된 JSON).

> **구현 참고**: 응답 `data`는 `summary`(total/passed/failed/skipped/duration)와 `suites` 배열을 포함하며, 각 suite는 `title`, `file`, `tests` 배열을 갖는다. 각 test는 `title`, `status`, `duration`, `retries`를 기본으로 하고, 실패 시 `error`(`message`, `stack`)와 `attachments`(스크린샷·trace·비디오의 name/path/contentType)를 포함한다.

### GET /runs/:runId/report/files/:path

리포트 첨부 파일(스크린샷, trace, 비디오) 정적 서빙.

> **구현 참고**: 경로 규칙은 `/runs/{runId}/report/files/{path}`이며 확장자별로 `image/png`, `application/zip`, `video/webm` 등의 `Content-Type`을 적절히 반환한다.

---

## 6. 수정 (Edits)

```mermaid
sequenceDiagram
    actor 사용자
    participant API as API Server
    participant SDK as Claude Agent SDK
    participant Git

    사용자->>API: POST /projects/:id/edits (prompt)
    API->>API: Edit(STREAMING) 생성
    API-->>사용자: editId + eventsUrl (202)

    사용자->>API: GET /edits/:editId/events (SSE 연결)
    API->>SDK: query({ prompt, options })

    loop 실시간 스트리밍
        SDK-->>API: SDKMessage (tool_use, text, stream_event)
        API-->>사용자: edit:tool_start / edit:text_delta / edit:text_done (SSE)
    end

    SDK-->>API: 턴 완료
    API-->>사용자: edit:turn_complete { filesChanged } (SSE)

    alt 후속 대화
        사용자->>API: POST /edits/:editId/messages { message }
        API->>SDK: sendFollowUp(message)
        SDK-->>API: 추가 스트리밍...
        API-->>사용자: edit:turn_complete (SSE)
    end

    사용자->>API: POST /edits/:editId/end
    SDK-->>API: result (세션 완료)
    API-->>사용자: edit:result { costUsd, durationMs } (SSE)

    alt 승인
        사용자->>API: POST /edits/:editId/approve
        API->>API: working → stable 동기화
        API->>Git: git commit
        API-->>사용자: APPROVED + commitHash
    else 거부
        사용자->>API: POST /edits/:editId/reject
        API->>API: working/ 롤백
        API-->>사용자: REJECTED
    end
```

### POST /projects/:id/edits

수정 세션 시작.

> **구현 참고**: `prompt`를 받아 Edit을 STREAMING 상태로 생성하고 `editId`, `status`, `eventsUrl`(예: `/api/v1/edits/{editId}/events`)을 202로 반환한다.

### GET /edits/:editId

수정 상세 조회 (대화 이력 + diff 포함).

> **구현 참고**: 응답 `data`에 `id`, `prompt`, `status`, `diff`(unified diff), `filesChanged`, SDK 세션 정보(`sessionId`, `messages[]`, `costUsd`, `durationMs`), `createdAt`을 담는다.

### GET /edits/:editId/sessions

SDK 세션 목록 조회 (세션 재개 시 사용).

> **구현 참고**: `sessionId`, `summary`, `lastModified`(epoch ms)로 이루어진 세션 배열을 반환한다.

### POST /edits/:editId/resume

이전 세션 재개.

> **구현 참고**: `sessionId`와 후속 `prompt`를 받아 해당 세션을 재개하고 새 스트리밍 세션(`editId`, `status`, `eventsUrl`)을 202로 반환한다.

### POST /edits/:editId/messages

진행 중(STREAMING/WAITING_INPUT) 세션에 후속 메시지 전송.

> **구현 참고**: `message` 문자열을 받아 `claude.service.sendFollowUp(editId, message)` 호출 후 202를 반환한다. 응답 스트림은 기존 SSE 연결을 통해 전달된다.

### POST /edits/:editId/interrupt

진행 중인 수정 세션 즉시 중단.

> **구현 참고**: 바디 없음. `claude.service.interruptSession(editId)`를 호출하여 `queryInstance.interrupt()`를 트리거한다. 중단 완료는 SSE `edit:result { subtype: 'interrupted' }`로 전달된다.

### POST /edits/:editId/rewind

특정 메시지 시점으로 파일 변경 되돌리기.

> **구현 참고**: `messageId`와 선택적 `dryRun`을 받아 `queryInstance.rewindFiles({ messageId, dryRun })` 결과를 200으로 반환한다. `dryRun: true`는 예상 결과만 계산해 반환한다.

### POST /edits/:editId/end

대화 종료(PENDING 전환).

> **구현 참고**: 현재 WAITING_INPUT 상태의 세션을 명시적으로 종료하여 승인/거부 대기 상태(PENDING)로 전환한다. 내부적으로 `claude.service`에 종료 신호를 보내 `edit:result`를 방출하도록 한다.

### POST /edits/:editId/approve

수정 승인 (stable 동기화 + Git commit).

> **구현 참고**: 응답 `data`에 `status: "APPROVED"`, `commitHash`, `syncedAt`을 담는다.

### POST /edits/:editId/reject

수정 거부 (working 롤백).

> **구현 참고**: 응답은 `{ status: "REJECTED" }`.

### GET /projects/:id/edits

프로젝트별 수정 이력 조회. 쿼리스트링으로 `page`, `limit`을 받는다.

---

## 7. SSE (Server-Sent Events)

실시간 스트리밍은 WebSocket 대신 **단방향 SSE**를 사용한다. 클라이언트는 브라우저 native `EventSource` API로 스트림을 구독하고, 반대 방향(취소·후속 메시지·중단·되돌리기·종료)은 별도의 HTTP REST 엔드포인트로 전송한다.

서버 구현은 Fastify + [`fastify-sse-v2`](https://github.com/NodeFactoryIo/fastify-sse-v2) 플러그인을 사용한다. 메시지 프레임 형식:

```
event: <type>
id: <monotonic-id>
data: <JSON string>

```

### GET /runs/:runId/events

테스트 실행 로그 및 진행도 SSE 스트림.

서버→클라이언트 이벤트 타입:
- `log` — 실행 로그 라인 문자열
- `status` — 상태 문자열(`QUEUED`/`RUNNING`/`PASSED`/`FAILED`/`CANCELLED`)
- `progress` — 집계 스냅샷(`status`, `totalTests`, `completedTests`, `passedTests`, `failedTests`, `skippedTests`, `progressPercent`)
- `test_result` — 개별 테스트 결과(`title`, `status`, `duration`)
- `done` — 최종 상태 및 `reportPath`
- `error` — 실패 원인 메시지

`QUEUED` 상태에서는 별도의 큐 위치/예상 대기 시간 표시를 유지하고, `RUNNING`부터 `progress` 이벤트를 사용해 화면의 진행도 바와 집계 수치를 갱신한다. 취소는 `DELETE /runs/:runId`로 요청한다(§4).

### GET /edits/:editId/events

Claude Agent SDK 수정 세션 SSE 스트림. 이벤트 타입 전체는 `08_Claude_Agent_SDK_연동명세서.md` §6 참조.

클라이언트→서버 조작은 아래 REST 엔드포인트를 사용한다(§6에서 이미 정의):
- `POST /edits/:editId/messages` — 후속 메시지 전송
- `POST /edits/:editId/interrupt` — 중단
- `POST /edits/:editId/rewind` — 파일 변경 되돌리기
- `POST /edits/:editId/end` — 대화 종료(PENDING 전환)
- `POST /edits/:editId/approve` / `/reject` — 최종 승인/거부

### 재연결 및 복원

- SSE는 브라우저 `EventSource`가 네트워크 단절 시 자동 재연결한다.
- 서버는 이벤트마다 단조 증가 `id:`를 부여하며, 재연결 시 클라이언트가 보낸 `Last-Event-ID` 헤더 이후 이벤트만 리플레이한다(Phase 2에서 Redis Stream으로 버퍼링, Phase 1은 최종 상태만 재전송).
- 이미 종료된 Run/Edit에 연결 시도 시 저장된 최종 상태만 전송하고 스트림을 닫는다.

---

## 8. 조직/사용자 관리

### GET /org

현재 조직 정보 조회

### PATCH /org

조직 정보 수정 (Admin 전용)

### GET /org/members

조직 멤버 목록

### PATCH /org/members/:userId

멤버 역할 변경 (Admin 전용)

### DELETE /org/members/:userId

멤버 제거 (Admin 전용)

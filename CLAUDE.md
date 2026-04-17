# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 저장소 현재 상태 (2026-04-17 기준)

Greenfield 명세 전용 저장소다. 아직 애플리케이션 코드는 없다. `package.json`, 소스 트리, 빌드/린트/테스트 설정 모두 아직 존재하지 않는다. 저장소에 존재하는 것은 한 줄짜리 `README.md`, `LICENSE`, `docs/`, 그리고 명세 외 메타 파일(`skills-lock.json`, `.gitignore`)뿐이다. 명세는 확정 완료되었고, 구현은 마일스톤 **M1(모노레포 스캐폴딩)** 부터 시작한다.

## 먼저 읽을 것

코드 없이 문서만 있는 단계이므로, 어떤 작업이든 아래 순서를 따른다.

1. `docs/implementation/README.md` — 구현 문서 전체 체계·읽기 순서·기술 스택 확정 요약
2. `docs/implementation/00_개요_문서_사용법.md` — 스펙↔마일스톤 매핑, FR 매핑, **구현 10원칙**, 용어 정의
3. `docs/implementation/01_Phase_로드맵.md` — Phase 0~4 로드맵, 크리티컬 패스, 주차별 순서

특정 기능을 구현할 때는 **대응하는 스펙 문서 + 대응하는 마일스톤 문서**를 한 쌍으로 참조한다. 매핑 표는 `docs/implementation/00_개요_문서_사용법.md` §4~§5.

## 이 프로젝트가 만드는 것

Playwright 테스트 프로젝트를 조직 단위로 등록·실행·리포트 조회하는 웹 플랫폼(Playwright Hub). 특징:

- Docker 컨테이너로 테스트 격리 실행
- BullMQ 큐 + SSE(`fastify-sse-v2`)로 실시간 로그·진행도 스트리밍
- `@anthropic-ai/claude-agent-sdk` 기반 자연어 코드 수정 + Diff 승인 워크플로우
- 조직/역할(Admin/Member) 기반 멀티테넌시

Phase 1(단일 서버) → Phase 2(다중 워커/NFS) → Phase 3(다중 Redis/S3) 확장을 전제로 설계되어 있다.

## 개발 명령

M1(스캐폴딩) **완료 전까지는 실행 가능한 개발 스크립트가 없다.** M1에서 `apps/web`(Next.js), `apps/api`(Fastify), `packages/shared-types`, `prisma/`, `deploy/docker-compose.yml`이 갖춰진 뒤 아래 명령을 사용한다.

| 명령 | 용도 |
|------|------|
| `pnpm install` | 모노레포 의존성 설치 (pnpm 8+ 필수, **npm/yarn 금지**) |
| `pnpm --filter web dev` | 프론트엔드 개발 서버 |
| `pnpm --filter api dev` | API 개발 서버 (`tsx watch` 기반 예정) |
| `pnpm --filter api exec prisma migrate dev` | Prisma 스키마 마이그레이션 |
| `pnpm --filter api exec prisma studio` | Prisma Studio |
| `docker compose -f deploy/docker-compose.yml up -d` | Phase 1 풀스택 기동 (web/api/worker/postgres/redis) |

lint/format/test 스크립트는 M1에서 확정된다. 단일 테스트 실행 방법은 테스트 러너(Vitest 예정) 설정이 들어온 시점에 본 문서를 갱신할 것.

## 아키텍처 정신 모델

단일 파일 탐색으로는 드러나지 않는 크로스-커팅 개념. 코드 작성 전에 반드시 숙지한다.

- **`projects/stable/` vs `projects/working/`** — Docker가 `:ro`로 마운트하여 테스트를 실행하는 안정본은 `stable/`, Claude Agent SDK가 파일을 편집하는 작업본은 `working/`. Edit 승인 시 `sync.service`가 분산 락을 잡고 `working/` → `stable/` 복사 후 git commit 한다. 두 경로를 **혼동하지 말 것**.
- **Run 상태 전이** — `QUEUED → RUNNING → PASSED|FAILED|CANCELLED`. 흐름: `POST /projects/:id/runs` → BullMQ enqueue → Worker가 Docker 컨테이너 기동 → stdout/stderr·progress를 Redis Pub/Sub → SSE로 릴레이 → `prisma.$transaction`으로 상태 확정.
- **Edit 상태 전이** — `STREAMING → WAITING_INPUT → PENDING → APPROVED|REJECTED`. Claude Agent SDK `query()`가 반환하는 AsyncGenerator를 서버가 수신하면서 SSE 이벤트로 브라우저에 중계한다. `sessionId`로 재개 가능하며, `cwd`는 `projects/working/:project`로 반드시 샌드박싱.
- **Queue Router 추상화** — API 레이어에서 작업을 "어느 Redis의 어느 큐로 보낼지" 결정하는 계층. Phase 1은 단일 Redis로 직행, Phase 2는 worker group 기반, Phase 3는 project-based 또는 load-based 라우팅. 라우팅 전략 변경이 라우트/서비스 상단으로 새어나오지 않게 유지한다.
- **SSE 스택** — 서버는 `fastify-sse-v2`, 클라이언트는 브라우저 native `EventSource`. 별도 SSE 라이브러리 도입 금지.
- **공유 이벤트 타입** — 서버/클라이언트의 Run/Edit SSE 이벤트 타입은 `packages/shared-types`에 단일 원본을 두고 양쪽이 import 한다. 서버 라우트의 Zod 스키마도 같은 패키지에서 `z.infer<>`로 타입 추출.

## 확정 기술 스택 (재논의 금지, 변경은 스펙 PR 경유)

- **Frontend**: Next.js 14 (App Router), React 18, Tailwind CSS 3, shadcn/ui, TanStack Query v5, Zustand
- **Backend**: Node.js 20 LTS, Fastify 5, 공식 플러그인(`@fastify/cors`·`helmet`·`compress`·`jwt`·`sensible`·`rate-limit`·`static`·`type-provider-zod`), `fastify-sse-v2` 4+, BullMQ 5, Prisma 5, ioredis, dockerode, simple-git, bcryptjs, zod
- **AI**: `@anthropic-ai/claude-agent-sdk`
- **Data**: PostgreSQL 16, Redis 7
- **Infra**: Docker 24+, Docker Compose v2
- **테스트 런타임**: Playwright 1.50+ (mcr.microsoft.com/playwright 이미지)
- **공통**: TypeScript 5+, **pnpm 8+**

세부 스펙은 `docs/specs/07_기술스택_프로젝트구조.md`, 목표 디렉터리 구조는 `docs/implementation/README.md` "디렉토리 구조 (목표)"와 `docs/implementation/09_파일단위_체크리스트.md`.

## 구현 10원칙 (출처: `docs/implementation/00_개요_문서_사용법.md` §7)

1. 스펙↔구현 일관성: 모순 발견 시 **스펙 PR 먼저**, 그 다음 구현 문서.
2. 레이어 분리: `route → service → db/external`. 라우트에서 Prisma 직접 호출 금지.
3. Zod-first: HTTP body/query/params, env, 외부 메시지 모두 Zod로 선검증.
4. 조직 격리 불변조건: 모든 Prisma `where`에 `orgId` 포함, `orgScope` 미들웨어 + 서비스 레이어 이중 체크.
5. 에러 규격화: API 응답은 항상 `{ success, data | error: { code, message } }`.
6. WS/SSE 이벤트 타입 공유: `packages/shared-types` 단일 원본.
7. 원자적 상태 전이: Run/Edit 상태 변경은 `prisma.$transaction` + 분산 락.
8. 보안 기본값: Docker 리소스 제한, path traversal 방지, Claude SDK `cwd` 샌드박싱, `projects/working/` 밖 접근 차단.
9. 관찰성: 모든 비동기 작업에 correlation ID(`runId`, `editId`, `request-id`) 부여.
10. 배포 재현성: 매 마일스톤 종료 시 clean state에서 `docker compose up -d` 기동 검증.

## 문서·커밋 규약

- 저장소 내 문서는 **한국어 + GFM Markdown**, 이모지 사용 금지(`docs/implementation/00_개요_문서_사용법.md` §8).
- 각 문서 상단에 "최종 수정일"과 "관련 스펙" 메타 유지.
- 외부 라이브러리 API(특히 Playwright, Fastify 5, BullMQ, Prisma, Next.js 14, Claude Agent SDK)를 확인할 때는 **`context7` MCP로 최신 문서 조회 권장** — 본 문서 내 버전 표기보다 실제 SDK 문서 우선.
- 구현 중 스펙과 모순이 드러나면 그 자리에서 코드만 바꾸지 말고 스펙 PR을 먼저 연다.

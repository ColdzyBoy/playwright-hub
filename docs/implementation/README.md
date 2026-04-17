# docs/implementation/ — Playwright Hub 구현 계획 문서

- 최종 수정일: 2026-04-17
- 대상 레포: `/home/superstart/projects/playwright-hub`
- 기반 명세: `../specs/` 8종 문서

## 이 디렉토리가 무엇인가

`docs/specs/`에 정리된 제품 명세를 **실제 코드 구현으로 옮기기 위한 단계별·파일 단위 실행 계획**이다. greenfield 상태의 레포에서 시작해 Phase 3 대규모 확장까지 도달하기 위한 모든 작업 항목을 담았다.

## 문서 목록

| # | 파일 | 내용 |
|---|------|------|
| 00 | [00_개요_문서_사용법.md](./00_개요_문서_사용법.md) | 문서 목적·독자·스펙 매핑·용어 정의·구현 10원칙 |
| 01 | [01_Phase_로드맵.md](./01_Phase_로드맵.md) | Phase 0~4 전체 로드맵 + 크리티컬 패스 + 주차별 개발 순서 |
| 02 | [02_마일스톤_M1_스캐폴딩.md](./02_마일스톤_M1_스캐폴딩.md) | 모노레포, Prisma, Docker Compose, 공통 타입 |
| 03 | [03_마일스톤_M2_인증_조직.md](./03_마일스톤_M2_인증_조직.md) | JWT 인증, 조직/역할 관리, 프론트 로그인 |
| 04 | [04_마일스톤_M3_프로젝트_관리.md](./04_마일스톤_M3_프로젝트_관리.md) | Git 연동, stable/working, 테스트 탐지, 분산 락 |
| 05 | [05_마일스톤_M4_테스트_실행_엔진.md](./05_마일스톤_M4_테스트_실행_엔진.md) | **BullMQ + Docker + SSE** (가장 큰 마일스톤) |
| 06 | [06_마일스톤_M5_리포트_조회.md](./06_마일스톤_M5_리포트_조회.md) | HTML 리포트 파싱·서빙, 스크린샷/Trace/비디오 |
| 07 | [07_마일스톤_M6_Claude_Agent_SDK.md](./07_마일스톤_M6_Claude_Agent_SDK.md) | **Claude Agent SDK 통합** (두 번째로 큰 마일스톤) |
| 08 | [08_마일스톤_M7_대시보드_폴리싱.md](./08_마일스톤_M7_대시보드_폴리싱.md) | 대시보드, E2E, 배포 문서, 베타 체크리스트 |
| 09 | [09_파일단위_체크리스트.md](./09_파일단위_체크리스트.md) | 전체 파일 인덱스 (약 190개) 레이어별 정리 |
| 10 | [10_검증_시나리오.md](./10_검증_시나리오.md) | 마일스톤별 실행 가능한 검증 명령어 + E2E 통합 시나리오 |
| 11 | [11_리스크_불확실성.md](./11_리스크_불확실성.md) | Top 10 리스크 + 스택별 불확실성 + SDK 스파이크 체크리스트 |
| 12 | [12_Phase2_Phase3_확장.md](./12_Phase2_Phase3_확장.md) | 다중 워커/NFS/다중 Redis/S3/Phase 4 로드맵 |

## 읽는 순서

### 처음 읽는 사람
1. 본 README
2. `00_개요_문서_사용법.md` — 전체 문서 체계 이해
3. `01_Phase_로드맵.md` — 프로젝트의 큰 그림
4. `02_~08_` 마일스톤 문서 — 구현 시 단계별로 참조
5. `11_리스크_불확실성.md` — 착수 전 리스크 숙지

### 지금 구현 중인 마일스톤이 있는 사람
1. 해당 마일스톤 문서(M1~M7)의 **파일 체크리스트** 섹션을 체크박스로 진행
2. 실제 파일 경로·exports는 `09_파일단위_체크리스트.md`에서 재확인
3. 마일스톤 종료 시 `10_검증_시나리오.md`로 판정

### 리뷰어/QA
1. `10_검증_시나리오.md` — 검증 명령 실행
2. `01_Phase_로드맵.md` §9 — 마일스톤 완료 1줄 판정 기준

### PO / 기획자
1. `00_개요_문서_사용법.md` — FR 매핑 테이블
2. `01_Phase_로드맵.md` — 주차별 일정
3. `11_리스크_불확실성.md` — 리스크 관리

## 핵심 정보 요약

### 기술 스택 (확정)
- **Frontend**: Next.js 14 App Router, React 18, Tailwind, shadcn/ui, TanStack Query v5, zustand, 브라우저 native `EventSource`(SSE 수신)
- **Backend**: Node.js 20, Fastify 5, `@fastify/cors`/`@fastify/helmet`/`@fastify/compress`/`@fastify/jwt`/`@fastify/sensible`/`@fastify/rate-limit`/`@fastify/static`/`@fastify/type-provider-zod`, `fastify-sse-v2`, BullMQ, Prisma, dockerode, simple-git, `@anthropic-ai/claude-agent-sdk`, bcryptjs, zod
- **DB / 캐시**: PostgreSQL 16, Redis 7
- **인프라**: Docker Compose (Phase 1) → 다중 워커 + NFS (Phase 2) → 다중 Redis + S3 (Phase 3)

### 디렉토리 구조 (목표)
```
playwright-hub/
├── apps/{web,api}/
├── packages/shared-types/
├── docker/playwright-runner/
├── deploy/
├── prisma/
├── projects/{stable,working}/
├── reports/
├── tests/e2e/
├── scripts/
└── docs/{specs,implementation}/
```

### 예상 일정
- **1인 fullstack**: 6~10주 (Phase 1 MVP)
- **2~3인 팀 (M4·M6 병렬)**: 4~6주
- **Phase 2**: +2~3주
- **Phase 3**: +4~6주

### 크리티컬 패스
```
Phase 0 → M1(Prisma) → M2(auth) → M3(projects)
       → M4(Worker+WS) → M5(Report) → M7(Dashboard+Deploy+E2E)
```
M6(Claude SDK)은 M3 이후 병렬 가능.

### 최고 위험 마일스톤
- **M4 (테스트 실행 엔진)** — BullMQ/Docker/WS 결합 복잡도
- **M6 (Claude SDK)** — SDK API 변동·AsyncGenerator·샌드박싱·비용 관리

## 문서 유지보수 규칙

- 구현 중 스펙과 모순 발견 시 **`docs/specs/` PR이 먼저**, 그 다음 `docs/implementation/` 갱신
- 마일스톤 종료 시 해당 문서에 "실제 발견 이슈 / 스파이크 결과" 섹션 추가
- 리스크 발생 시 `11_리스크_불확실성.md` 표 갱신
- 각 문서 상단 "최종 수정일" 유지

## 작업 시작

1. `01_Phase_로드맵.md` 에서 주차별 순서 확인
2. `02_마일스톤_M1_스캐폴딩.md` 에서 Phase 0 → M1 착수
3. 각 파일 체크박스를 실제 커밋과 매핑하여 진행률 추적

## 참조

- 제품 명세: [../specs/](../specs/)
- Claude Code 작업 컨텍스트: 본 문서 집합 + `docs/specs/` 전부
- 외부 레퍼런스: Playwright, BullMQ, Prisma, Next.js 공식 문서 (context7 MCP로 최신 확인 권장)
